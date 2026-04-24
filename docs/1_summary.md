# Vela - Architecture Summary (v0.1.0)

## 1. System Overview

Vela  is a platform that executes WebAssembly (WASM) modules inside AWS Nitro Enclaves (TEE), with encrypted state management and blockchain-based coordination. Users submit requests on-chain; the system processes them securely off-chain and posts signed results back to the blockchain.

```
                                 ┌──────────────────────────────────┐
                                 │         Blockchain (EVM)         │
                                 │  Set of smart contracts:         │
                                 │  ProcessorEndpoint (requests,    │
                                 │    state updates, fees)          │
                                 │  TeeAuthenticator (TEE identity  │
                                 │    and signature verification)   │
                                 │  AuthorityRegistry (authority    │
                                 │    permissions)                  │
                                 └────────┬──────────▲─────────────┘
                                          │          │
                              fetch requests    submit state updates
                                          │          │
┌─────────────────────────────────────────▼──────────┴──────────────────┐
│                    Secure Processor Manager                           │
│                    (runs outside enclave)                             │
│                                                                       │
│  ┌──────────────┐   ┌────────────────┐   ┌────────────────────────┐   │
│  │  Blockchain   │   │  Executor      │   │  Storage (LevelDB)     │  │
│  │  Client       │   │  Client        │   │  - versioned app state │  │
│  │  (RPC)        │   │  (TCP/VSock)   │   │  - WASM bytecode       │  │
│  └──────────────┘   └───────┬────────┘    │  - keyset recovery     │  │
│                              │            └────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Log Server (centralized logging with optional file rotation)   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Admin Interface (MANAGER_ADMIN_PORT)                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┼────────────────────────────────────────┘
                               │ bidirectional messages
                               │ (JSON over VSock/TCP)
┌──────────────────────────────▼────────────────────────────────────────┐
│                    WASM Executor (AWS Nitro Enclave)                  │
│                                                                       │
│  ┌──────────────┐   ┌───────────────────┐   ┌─────────────────────┐   │
│  │  Enclave      │   │  Wasmtime Runtime │   │  Crypto Operations  │  │
│  │  KeySet       │   │  (WASM execution) │   │  - AES-256 state    │  │
│  │  - AES state  │   │                   │   │  - P521 ECDH events │  │
│  │  - secp256k1  │   │  ┌─────────────┐  │   │  - secp256k1 sign   │  │
│  │    signing    │   │  │ WASM Guest  │  │   └─────────────────────┘  │
│  │  - P521 comm  │   │  │ Application │  │                            │
│  └──────────────┘   │  └─────────────┘  │                             │
│                      └───────────────────┘                            │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 2. Main Components

### 2.1 Smart Contracts (Blockchain Layer)

The on-chain layer coordinates request submission, TEE attestation, and state management.

#### ProcessorEndpoint

The primary contract. Manages a FIFO request queue and processes state updates.

- **Request submission** (`submitRequest`): Users enqueue `PROCESS`, `DEANONYMIZATION` or `ASSOCIATEKEY` requests. Each request carries an application ID, request type, encrypted payload, a token address (`0x0` = native ETH), an asset amount, and a max fee. `DEPLOYAPP` is rejected here — deployments use a separate entry point.
- **Meta-transaction submission** (`submitRequestFor`): A facilitator submits a `PROCESS` or `ASSOCIATEKEY` request on behalf of an end user. The user signs an EIP-712 authorization (`REQUEST_AUTHORIZATION_TYPEHASH`) covering `sender`, protocol version, application ID, request type, payload hash, token address, asset amount, nonce, and deadline; the facilitator pays gas and the fee via `msg.value`. Per-user nonces (`facilitatorNonces[sender]`) provide replay protection and are readable via `getFacilitatorNonce(user)`. ERC-20 asset transfer uses EIP-2612 `permit` + `transferFrom` — the `depositPermit` field decodes to `(uint8 v, bytes32 r, bytes32 s)`. The resulting `PendingRequest.sender` is the end user; `PendingRequest.facilitator` is the caller.
- **Deploy submission** (`submitDeployRequest`): Restricted to addresses with `DEPLOYER_ROLE`. Takes a protocol version and a deploy descriptor payload referencing the WASM artifact by its `sha256`. A unique `applicationId` is derived from the request ID for each deploy.
- **State update** (`stateUpdate`): Called by the Manager after execution. Verifies the TEE signature, updates the on-chain state root, emits encrypted events, processes withdrawals and refunds via pull-payment claims. The TEE also reports success/failure inline via `errorCode` + `errorMsg`.
- **Claims** (`claim`, `pendingClaims`): Withdrawal and refund amounts are credited to per-token pull-payment balances; recipients call `claim(tokenAddress, payee)` to collect.
- **ERC-20 support**: For non-ETH tokens the caller must `approve` the `ProcessorEndpoint` up to `assetAmount` before calling `submitRequest`; tokens are pulled via `safeTransferFrom`. Only tokens in the `TokenAllowlist` are accepted.
- **Access control**: `UPDATE_STATUS_ROLE` for state updates; `DEPLOYER_ROLE` for deploy submissions; `ADMIN` for configuration.

Key data structures:
```solidity
struct PendingRequest {
    uint256 timestamp;
    address tokenAddress;     // 0x0 = ETH
    uint256 assetAmount;      // business asset amount (deposit for PROCESS / 0 for others)
    uint256 maxFeeValue;
    bytes32 requestId;
    bytes   payload;
    address sender;
    address facilitator;      // address(0) for direct submissions, facilitator address for meta-tx
    uint64  applicationId;
    uint8   protocolVersion;
    RequestType requestType;  // DEPLOYAPP | PROCESS | DEANONYMIZATION | ASSOCIATEKEY
}

struct WithdrawalRequest {
    address tokenAddress;     // 0x0 = ETH
    address payable receiver;
    uint256 amount;
}
```

Events: `RequestSubmitted` (carries `facilitator`), `RequestCompleted`, `DeployRequestSubmitted`, `DeployRequestCompleted`, `StateRootUpdate`, `UserEvent` (encrypted, per-user), `AppEvent` (plaintext, application-level), `Refund` (with `tokenAddress`), `Withdrawal` (with `tokenAddress`), `ReportGenerated`, `PaymentWithdrawn`.

#### TeeAuthenticator

Manages TEE identity. Verifies AWS Nitro Enclave attestation documents against expected PCR0 values and registers the TEE's signing address and P521 public key. In development mode, a `NoAttestationTeeAuthenticator` variant is used to bypass Nitro attestation requirements.

- **Multi-step attestation** (`updateTeeStep1`..`4`): Breaks large attestation verification across multiple transactions to fit gas limits.
- **Signature verification** (`checkSignature`): Called by ProcessorEndpoint during `stateUpdate`. Recovers the signer from an ECDSA signature and verifies it matches the registered TEE address.

The signed message covers: `applicationId`, `prevStateRoot`, `newStateRoot`, `processedRequestId`, `hash(userEvents)`, `hash(userEventSubTypes)`, `hash(appEvents)`, `hash(appEventSubTypes)`, `hash(withdrawalRequests)`, `refundAmount`, `applicationFee`, `errorCode`, `errorMsg`. UserEvents and AppEvents are hashed separately, and each kind contributes a data hash plus a subtype hash.

#### AuthorityRegistry & DefaultAuthority

Controls which addresses can submit deanonymization requests per application. Uses a registry/strategy pattern: each application can have a custom authority checker, with a default allowlist fallback.

### 2.2 Secure Processor Manager

The Manager runs outside the enclave and orchestrates the entire execution flow.

**Startup sequence:**
1. Load configuration (env vars / `.conf` file)
2. Connect to blockchain (Ethereum RPC)
3. Initialize versioned LevelDB storage
4. Connect to Executor (TCP or VSock)
5. Complete keyset handshake with Executor
6. Start log server (centralized log collection with optional file rotation)
7. Start admin interface (if `MANAGER_ADMIN_PORT` is configured)
8. Start polling loop

**Main processing loop** (`pollBlockchain`):
```
every N seconds:
  1. Fetch next pending request from ProcessorEndpoint
  2. Compare on-chain state root with local state root
     - If mismatch: detect chain reorg → rollback or fatal
  3. Route request by type:
     - Deploy  → send WASM to Executor, store initial state
     - Process / Deanonymize / AssociateKey → load state + WASM, send to Executor
  4. Store new encrypted state in LevelDB
  5. Submit signed UpdatePayload to blockchain
  6. On failure: rollback state, call stateUpdate with errorCode + errorMsg to mark the request failed on-chain
```

**Key interfaces:**

| Interface | Purpose |
|-----------|---------|
| `blockchain.Client` | Fetch requests, submit state updates (success or error) |
| `communication.ExecutorClient` | Send process/deploy requests, handle keyset handshake |
| `storage.DataLayer` | Store/retrieve versioned state, WASM bytecode, keyset recovery |

**Storage layer** (Versioned LevelDB):
- Each state update creates a new version (ID = SHA256 of encrypted state)
- Retains last N versions for rollback on chain reorganization
- Non-versioned storage for enclave keyset recovery data

**Log server** (new in v0.0.25):
- Centralized log collection from both Manager and Executor
- Optional file rotation with configurable parameters:
  - `LOG_SERVER_FILE_ROTATION` — enable/disable rotation
  - `LOG_SERVER_FILE_MAX_SIZE_MB` — max size per file
  - `LOG_SERVER_FILE_MAX_BACKUPS` — number of rotated files to keep
  - `LOG_SERVER_FILE_MAX_AGE_DAYS` — max age before deletion
  - `LOG_SERVER_FILE_COMPRESS` — compress rotated files

**Admin interface** (new in v0.0.25):
- Exposed on `MANAGER_ADMIN_PORT`
- Configurable request timeout via `MANAGER_ADMIN_COMMUNICATION_PARAMS_REQUEST_TIMEOUT_SEC`

### 2.3 WASM Executor

The Executor runs inside the AWS Nitro Enclave (TEE) and handles all cryptographic operations and WASM execution.

**Enclave KeySet** — three keys generated on first start and recovered on restart:

| Key | Type | Purpose |
|-----|------|---------|
| `StateKey` | AES-256 | Encrypt/decrypt application state |
| `SigningKey` | secp256k1 | Sign update payloads (verified on-chain) |
| `CommunicationKey` | P-521 (ECDH) | Encrypt events, payloads, and reports |

**Keyset recovery** — controlled by `EXECUTOR_KEYSET_RECOVERY_TYPE`:

| Value | Mode | Description |
|-------|------|-------------|
| `0` | Fixed keys | Use keys from environment variables (`EXECUTOR_FIXED_SIGNING_KEY`, `EXECUTOR_FIXED_COMMUNICATION_KEY`, `EXECUTOR_FIXED_STATE_KEY`). Used for local development. |
| Other | Recovery handshake | Restore keys from encrypted recovery data stored by the Manager (production mode). |

**Keyset handshake** (on connection):
1. Executor asks Manager for stored recovery data
2. If found: restore keys from encrypted recovery blob
3. If not found: generate new keyset, send encrypted recovery to Manager for storage

**Request processing pipeline** (`HandleProcessRequest`):
```
1. Decrypt application state (AES-256 with StateKey)
2. Deserialize into AppData (state + user key store)
3. If assetAmount > 0: call WASM deposit()
4. If AssociateKey: register sender's P521 public key (+ optional encrypted subtype seed)
5. Otherwise (Process or Deanonymize):
   - decrypt payload (ECDH)
   - call WASM process_request() with requestType parameter
   - the WASM app uses requestType to route internally
6. Encrypt new state (AES-256), compute state root (SHA256)
7. For each `PlainEvent` entry, if the target user has provided a subtype seed, compute eventSubType
8. Encrypt `PlainEvent` entries using recipient P521 public keys (ECDH → AES-GCM); `AppEvent` entries pass through as plaintext
9. If Deanonymize: validate that report data is present in the response
10. Sign UpdatePayload with secp256k1 SigningKey
11. Return: UpdatePayload + encrypted ApplicationState + optional DeanonymizationReport
```

---

## 3. Application Flow (End-to-End)

### 3.1 Deploy Application

A deploy has two phases: off-chain WASM artifact upload, then on-chain `submitDeployRequest`.

1. The deployer (must hold `DEPLOYER_ROLE`) uploads the WASM binary to the Authority Service, which stores it in the shared artifact folder keyed by `sha256(wasm)`.
2. The deployer calls `submitDeployRequest(protocolVersion, payload)` on `ProcessorEndpoint`, where `payload` is a JSON deploy descriptor: `{ "mode": "artifact_ref", "artifactId": "sha256:<hex>", "wasmSha256": "<hex>", "constructorParams": … }`.
3. The contract derives a fresh `applicationId` from the request hash and enqueues the deploy.

```
Deployer                 Blockchain              Manager                Executor
  │                          │                      │                      │
  │─ (upload WASM to         │                      │                      │
  │   Authority Service)     │                      │                      │
  │                          │                      │                      │
  │─ submitDeployRequest ───>│                      │                      │
  │   (deploy descriptor     │                      │                      │
  │    payload, maxFee)      │                      │                      │
  │                          │<── poll ─────────────│                      │
  │                          │── PendingRequest ───>│                      │
  │                          │                      │── load WASM by sha256│
  │                          │                      │── SendDeployApp ────>│
  │                          │                      │                      │─ compile WASM
  │                          │                      │                      │─ call load_module()
  │                          │                      │                      │─ encrypt initial state
  │                          │                      │                      │─ sign payload
  │                          │                      │<─ UpdatePayload ─────│
  │                          │                      │<─ ApplicationState ──│
  │                          │                      │                      │
  │                          │                      │── store state+WASM   │
  │                          │<── stateUpdate ──────│   in LevelDB         │
  │                          │── verify sig ────────│                      │
  │                          │── update state root  │                      │
  │<── DeployRequestCompleted│                      │                      │
```

### 3.2 Process Request (Standard)

```
User                     Blockchain              Manager                Executor
  │                          │                      │                      │
  │─ submitRequest(PROCESS) ─>│                      │                      │
  │   (encrypted payload,     │                      │                      │
  │    tokenAddress,          │                      │                      │
  │    assetAmount, maxFee)   │                      │                      │
  │                          │<── poll ──────────────│                      │
  │                          │── PendingRequest ────>│                      │
  │                          │                      │── load state+WASM    │
  │                          │                      │   from LevelDB       │
  │                          │                      │── SendProcessReq ───>│
  │                          │                      │                      │─ decrypt state (AES)
  │                          │                      │                      │─ if deposit: call WASM deposit()
  │                          │                      │                      │─ decrypt payload (ECDH)
  │                          │                      │                      │─ call WASM process_request()
  │                          │                      │                      │─ encrypt new state
  │                          │                      │                      │─ encrypt events
  │                          │                      │                      │─ sign payload
  │                          │                      │<─ UpdatePayload ─────│
  │                          │                      │<─ ApplicationState ──│
  │                          │                      │                      │
  │                          │                      │── store new state    │
  │                          │<── stateUpdate ──────│   in LevelDB         │
  │                          │── emit UserEvent(s)  │                      │
  │                          │── process withdrawals│                      │
  │<── RequestCompleted ─────│                      │                      │
  │<── UserEvent(encrypted) ─│                      │                      │
```

### 3.3 Associate Key

Before a user can receive encrypted events or submit encrypted payloads, they register their P-521 public key on-chain via an `ASSOCIATEKEY` request. The payload is either the raw 133-byte uncompressed public key, or 226 bytes = public key || encrypted subtype seed (when the user opts in to privacy-preserving event sub-types derived from an HMAC'd seed). The Executor stores the key (and, if provided, the seed) in the user key store (part of AppData). Subsequent event encryption uses this key via ECDH.

### 3.4 Deanonymization

An authorized authority submits a `DEANONYMIZATION` request. The Executor passes the request to `process_request()` with `requestType = Deanonymize`. The WASM application uses the `requestType` parameter to determine that a deanonymization report is needed and returns the report data in the `ProcessResult.Report` field. The Executor validates that report data is present, encrypts it with the authority's P-521 public key, and stores it. The Manager stores the encrypted report on the filesystem. The authority retrieves it via the Authority Service HTTP API (`GET /nonce` + `POST /getreport`).

---

## 4. WASM Guest Interface

To develop a new WASM application for Vela CCE, you implement a TinyGo program that exports specific functions, compile it to WASM with `tinygo build -target=wasi`, and deploy it on-chain.

### 4.1 Required Exports

The WASM module must export these functions:

#### `deploy(appId: i64, paramsPtr: *byte, paramsLen: i32) -> *byte`

Called once when the application is deployed on-chain. Receives the JSON-encoded constructor parameters extracted by the Executor from the deploy descriptor's `constructorParams` field, and returns the initial state.

```go
//export deploy
func deploy(appId int64, paramsPtr *byte, paramsLen int32) *byte {
    paramsJSON := utils.PtrToString(paramsPtr, paramsLen)
    result := myapp.Deploy(appId, paramsJSON)
    return types.SerializeAndWriteResult(result)
}
```

Returns `DeployResult`:
```go
type DeployResult struct {
    State []byte   `json:"state"`   // JSON-serialized initial state
    Fuel  *Uint256 `json:"fuel"`    // Fuel units consumed
    Error string   `json:"error,omitempty"`
}
```

#### `load_module(appId: i64) -> *byte`

Retained for cache warm-up on Executor restart (the host calls it to rebuild the default-initial-state cache entry without re-running the constructor). New deployments use `deploy` — apps should treat `load_module` as a fallback that produces a minimal default state with no constructor input.

```go
//export load_module
func load_module(appId int64) *byte {
    result := myapp.LoadModule(appId)
    return types.SerializeAndWriteResult(result)
}
```

Returns `LoadModuleResult`:
```go
type LoadModuleResult struct {
    State []byte   `json:"state"`   // JSON-serialized initial state
    Fuel  *Uint256 `json:"fuel"`    // Fuel units consumed
    Error string   `json:"error,omitempty"`
}
```

#### `deposit(appId: i64, senderPtr: *byte, senderLen: i32, tokenPtr: *byte, tokenLen: i32, valuePtr: *byte, valueLen: i32, statePtr: *byte, stateLen: i32) -> *byte`

Called when a user sends funds (deposit) with their request. `tokenPtr`/`tokenLen` carry the token address (`0x0` for native ETH, otherwise an allowlisted ERC-20).

```go
//export deposit
func deposit(appId int64, senderPtr *byte, senderLen int32,
             tokenPtr *byte, tokenLen int32,
             valuePtr *byte, valueLen int32,
             statePtr *byte, stateLen int32) *byte {
    sender := types.PtrToAddress(senderPtr, senderLen)
    token  := types.PtrToAddress(tokenPtr, tokenLen)
    value  := types.PtrToUint256(valuePtr, valueLen)
    state  := utils.PtrToString(statePtr, stateLen)
    result := myapp.DepositFunds(sender, token, value, state)
    return types.SerializeAndWriteResult(result)
}
```

Returns `DepositResult`:
```go
type DepositResult struct {
    State     []byte       `json:"state"`               // Updated state
    Events    []PlainEvent `json:"events"`              // Per-user encrypted events
    AppEvents []AppEvent   `json:"appEvents"`           // Application-level plaintext events
    Fuel      *Uint256     `json:"fuel"`                // Fuel consumed
    Error     string       `json:"error,omitempty"`
}
```

#### `process_request(appId: i64, senderPtr: *byte, senderLen: i32, requestType: i32, payloadPtr: *byte, payloadLen: i32, statePtr: *byte, stateLen: i32) -> *byte`

Main processing function. Called for `PROCESS` and `DEANONYMIZATION` requests. The Executor passes the `requestType` parameter so the WASM app can route internally. `ASSOCIATEKEY` is handled entirely by the Executor and does not reach this function.

```go
//export process_request
func process_request(appId int64, senderPtr *byte, senderLen int32,
                     requestType int32,
                     payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    sender  := types.PtrToAddress(senderPtr, senderLen)
    payload := utils.PtrToString(payloadPtr, payloadLen)
    state   := utils.PtrToString(statePtr, stateLen)
    result  := myapp.ProcessRequest(sender, requestType, payload, state)
    return types.SerializeAndWriteResult(result)
}
```

Returns `ProcessResult`:
```go
type ProcessResult struct {
    State       []byte       `json:"state"`              // Updated state
    Events      []PlainEvent `json:"events"`              // Per-user encrypted events
    AppEvents   []AppEvent   `json:"appEvents"`           // Application-level plaintext events
    Withdrawals []Withdrawal `json:"withdrawals"`         // Fund transfers
    Report      []byte       `json:"report,omitempty"`    // Deanonymization report (only for DEANONYMIZATION requests)
    Fuel        *Uint256     `json:"fuel"`                // Fuel consumed
    Error       string       `json:"error,omitempty"`
}
```

When `requestType` is `DEANONYMIZATION` (value 2), the application **must** populate the `Report` field. The Executor validates this: if the request is a deanonymization but `Report` is empty, or if `Report` is non-empty for a non-deanonymization request, the Executor returns an error.

#### `allocate` and `deallocate`

Memory management functions. These must be implemented in the WASM module's `utils` package. They manage memory allocation and deallocation for data exchange between host and guest.

```go
//export allocate
func allocate(size int32) int32        // allocates memory in WASM linear memory

//export deallocate
func deallocate(ptr *byte, size int32) // frees previously allocated memory
```

The `utils` package also provides `BytesToPtr` for returning results (4-byte little-endian length prefix + data) and `PtrToString` for reading host-provided data.

#### `get_memory_stats() -> *byte` (optional)

Returns memory allocation statistics. Useful for debugging.

### 4.2 Types

In v0.0.25, common WASM types are provided by the shared library `github.com/HorizenOfficial/vela-common-go/wasm/types`. This library defines the core types (`Address`, `Uint256`, `PlainEvent`, `AppEvent`, `Withdrawal`, result structs) and helper functions (`SerializeAndWriteResult`, `PtrToAddress`, `PtrToUint256`). The utility functions (`PtrToString`, `BytesToPtr`, `allocate`/`deallocate`, `get_allocated_memory_stats`, logging) are in `vela-common-go/wasm/utils`.

The WASM guest module still communicates with the host via JSON serialization — both sides define compatible types independently. The guest cannot import host packages (e.g., `go-ethereum`), but it now shares type definitions with other WASM applications through the common library.

**Core types** (defined in `vela-common-go/wasm/types`):

| Type | Description |
|------|-------------|
| `Address` | 20-byte Ethereum address (`[20]byte`). JSON: `"0x..."` hex string. |
| `Uint256` | 256-bit unsigned integer (`[4]uint64`). JSON: `"0x..."` hex string. Supports `Add`, `Sub`, `Cmp`, `AddOverflow`, `SubOverflow`, `Mul64`, `Add64`, `IsZero`, `Eq`, `SetHex`, `ToHex`. Custom implementation (no `math/big` in TinyGo). |
| `PlainEvent` | Per-user encrypted event: `UserID` (Address), `EventSubType` ([32]byte — the on-chain `bytes32` indexed subtype), `Data` ([]byte). Executor encrypts `Data` with the target user's P-521 key. |
| `AppEvent` | Application-level plaintext event (not encrypted, visible on-chain): `EventSubType` ([32]byte — the on-chain `bytes32` indexed subtype), `Data` ([]byte). Emitted via the `AppEvent` contract event. |
| `Withdrawal` | Withdrawal instruction: `TokenAddress` (Address — `0x0` = ETH), `DestinationAddress` (Address), `Amount` (*Uint256). |

**Helper functions** (defined in `vela-common-go/wasm/types`):

| Function | Description |
|----------|-------------|
| `SerializeAndWriteResult(result)` | Serialize result to JSON and allocate in WASM memory |
| `PtrToAddress(ptr, len)` | Convert WASM pointer to `*Address` |
| `PtrToUint256(ptr, len)` | Convert WASM pointer to `*Uint256` |

**Utility functions** (defined in `vela-common-go/wasm/utils`):

| Function | Description |
|----------|-------------|
| `PtrToString(ptr, len)` | Convert WASM pointer to Go string |
| `BytesToPtr(data)` | Convert Go byte slice to WASM pointer (4-byte length prefix + data) |
| `LogError/Warn/Info/Debug/Trace(fmt, args...)` | Logging with prefixes (ERR/WRN/INF/DBG/TRC), captured by host via WASI stdout pipe |

### 4.3 Request Types

The Executor routes requests by type to the appropriate WASM export:

| Value | Type | Submitted via | WASM Export Called |
|-------|------|--------------|-------------------|
| 0 | `DEPLOYAPP` | `submitDeployRequest` (DEPLOYER_ROLE only) | `load_module` |
| 1 | `PROCESS` | `submitRequest` | `deposit` (if `assetAmount > 0`) + `process_request(requestType=1)` |
| 2 | `DEANONYMIZATION` | `submitRequest` (AuthorityRegistry-gated) | `process_request(requestType=2)` — app returns report in `ProcessResult.Report` |
| 3 | `ASSOCIATEKEY` | `submitRequest` | None (handled entirely by the Executor) |

The `requestType` is passed to `process_request` as an `int32` parameter. The WASM application uses it to determine which logic to execute (e.g., standard processing vs. deanonymization report generation). This is a change from earlier versions where deanonymization had a separate WASM export.

`submitRequest` rejects `DEPLOYAPP` with `InvalidRequestType` — deploys must go through `submitDeployRequest`, which derives a fresh `applicationId` from the request hash.

### 4.4 State Management

- State is an opaque `[]byte` from the host's perspective — your app defines its own format (typically JSON).
- State is passed as a JSON string to each function and returned as `[]byte` in the result.
- The Executor encrypts state with AES-256 before storing; your app never handles encryption.
- State root = SHA256(encrypted state). Used for on-chain consistency.
- State is versioned in LevelDB, enabling rollback on chain reorganizations.

### 4.5 Events

An application can emit two kinds of events from `deposit` / `process_request`:

**`PlainEvent` (per-user, encrypted):**
- Targets a specific user (`UserID`).
- The Executor encrypts `Data` using the target user's registered P-521 public key (ECDH → AES-GCM).
- Emitted on-chain as `UserEvent(appId, requestId, eventSubType, encryptedData)`.
- Users decrypt off-chain using their P-521 private key.

**`AppEvent` (application-level, plaintext):**
- No recipient — intended for public/app-wide signals (e.g. global state notifications).
- `Data` is **not** encrypted and is visible on-chain.
- Emitted as `AppEvent(appId, requestId, eventSubType, data)`.

`EventSubType` is a `bytes32` value emitted as an **indexed** log topic (usable as an indexer/subgraph filter). For `PlainEvent`s, the value the app writes depends on whether the target user registered a subtype seed at `ASSOCIATEKEY` time: if a seed is present, the Executor **overrides** any app-supplied subtype with an HMAC-derived opaque value drawn from the user's seed (privacy-preserving, unlinkable across users), so apps typically leave `EventSubType` as the zero `[32]byte`; if no seed is registered, the app-supplied value is preserved and can be a short ASCII label packed into the first bytes (`[32]byte{'d','e','p','o','s','i','t',0,…}`). For `AppEvent`s the app's value is always used as-is. Design any app-supplied sub-types so they don't leak sensitive information.

### 4.6 Withdrawals

Return `Withdrawal` entries to transfer funds from the contract to a destination address. Each withdrawal specifies a `TokenAddress` (`0x0` for native ETH, or an allowlisted ERC-20 address). The smart contract validates the app's custody balance per token, then credits the destination via pull-payment — recipients call `claim(tokenAddress, payee)` to collect.

### 4.7 Fuel

Each operation must return a `Fuel` value representing computation cost. Fuel is priced at a configurable Wei-per-unit rate. The total fee (fuel cost) plus refund must equal the user's `maxFeeValue`. A minimum fee per request (`MIN_FEE_PER_REQUEST`) is enforced at the contract level.

### 4.8 Deanonymization Reports

When the request type is `DEANONYMIZATION` (value 2), the Executor calls `process_request()` with `requestType=2`. The application **must** populate the `Report` field in the returned `ProcessResult`. The Executor validates that report data is present and encrypts it with the requesting authority's P-521 public key. The authority retrieves it via the Authority Service.

### 4.9 Application-side Constraints

- **TinyGo limitations**: Not all Go stdlib packages are available. Avoid `reflect`-heavy code, `net`, `os` (beyond basics), etc.
- **Stateless execution**: All state must be serialized/deserialized each call. No global mutable state persists between invocations.
- **Determinism**: Results must be deterministic for the same inputs. Avoid `time.Now()`, random number generation, or other non-deterministic operations.
- **Memory**: Implement `allocate`/`deallocate` in your `utils` package. The host manages memory lifecycle via these exports.
- **Logging**: Use `utils.LogError/Warn/Info/Debug/Trace` — output is written to stdout with level prefixes (ERR/WRN/INF/DBG/TRC) and captured by the host via WASI pipes.

---

## 5. Cryptographic Design

| Primitive | Key Type | Purpose |
|-----------|----------|---------|
| AES-256-GCM | Symmetric (32 bytes) | Encrypt application state at rest |
| NIST P-521 (ECDH) | Asymmetric (133 bytes pub) | Encrypt events, payloads, reports (ECDH → HKDF-SHA256 → AES-GCM) |
| secp256k1 (ECDSA) | Asymmetric | Sign update payloads for on-chain verification |
| HKDF-SHA256 | KDF | Derive AES keys from ECDH shared secrets |
| SHA256 | Hash | State root computation |

**Trust boundary**: All key material and cryptographic operations live inside the Nitro Enclave. The Manager only handles encrypted blobs and signed payloads.

---

## 6. Resilience

### Chain Reorganization Handling

1. Manager detects state root mismatch between chain and local storage
2. Scans stored versions for matching state root → confirms reorg
3. Waits `REORG_TIMEOUT` (default 3 min) for chain to stabilize
4. Rolls back LevelDB to matching version
5. Resumes processing from rolled-back state

### Keyset Recovery

On Executor restart, the keyset recovery mechanism (controlled by `EXECUTOR_KEYSET_RECOVERY_TYPE`) restores the keyset. In production, the handshake protocol recovers the keyset from encrypted recovery data stored by the Manager. In development, fixed keys from environment variables are used. This ensures the same signing address and encryption keys are used across restarts, maintaining continuity with on-chain identity.

### Error Handling

- **Transient errors** (WASM failure, invalid payload): Mark request failed on-chain, refund user, continue polling.
- **Fatal errors** (storage corruption, unrecoverable state mismatch): Shutdown for manual intervention.
- **Blockchain submission errors**: Detect reorg vs. permanent failure; rollback or mark failed accordingly.

---

## 7. Deployment Architecture (Docker Compose)

The local development environment runs the full stack via Docker Compose. All images use the `v0.1.0` tag.

### 7.1 Services

| Service | Image | Purpose |
|---------|-------|---------|
| `executor` | `horizen/cce-executor:v0.1.0` | WASM execution inside emulated TEE |
| `manager` | `horizen/cce-manager:v0.1.0` | Orchestration, blockchain polling, state storage |
| `authorityservice` | `horizen/cce-authorityservice:v0.1.0` | Deanonymization report retrieval API |
| `chain` | `horizen/cce-chain:v0.1.0` | Anvil dev chain (Foundry) |
| `deployer` | `horizen/cce-deployer:v0.1.0` | Smart contract deployment (runs once) |
| `subgraph-deployer` | `horizen/cce-subgraph-deployer:v0.1.0` | Subgraph deployment (runs once) |
| `subgraph-node` | `graphprotocol/graph-node` | The Graph indexing |
| `subgraph-postgres` | `postgres:14` | Graph Node database |
| `subgraph-ipfs` | `ipfs/kubo:v0.17.0` | IPFS for subgraph storage |

### 7.2 Networking

Services communicate over an internal Docker network (`pes_network`) with fixed IP addresses:

| Service | IP Address |
|---------|------------|
| Executor | `EXECUTOR_IP_HOST` (default: `10.10.40.10`) |
| Manager | `MANAGER_IP_HOST` (default: `10.10.40.20`) |
| Chain | `CHAIN_RPC_ADDRESS` (default: `10.10.40.30`) |
| Authority Service | `AUTHORITY_SERVICE_IP_ADDRESS` (default: `10.10.40.40`) |

Only the chain RPC (port 8545), authority service (port 8081), and subgraph query endpoint (port 8000) are exposed to the host.

### 7.3 Volumes

| Volume | Purpose |
|--------|---------|
| `vela-skit-manager-data` | Manager LevelDB (versioned state) |
| `vela-skit-manager-reports` | Deanonymization reports (shared with authority service) |
| `vela-skit-chain-data` | Anvil blockchain data |
| `vela-skit-logs` | Centralized log files |
| `vela-skit-deploy-data` | Deployed contract addresses (shared across services) |
| `vela-skit-subgraph-postgres` | Graph Node PostgreSQL data |
| `vela-skit-subgraph-ipfs` | Graph Node IPFS data |

### 7.4 Startup Sequence

1. **chain** (Anvil) starts and becomes available
2. **subgraph-postgres**, **subgraph-ipfs** start (Graph Node infrastructure)
3. **deployer** connects to the chain, deploys all smart contracts, writes deployed addresses to `vela-skit-deploy-data`, and exits
4. **subgraph-node** starts, connects to the chain, and becomes healthy
5. **subgraph-deployer** reads deployed contract addresses, generates a local subgraph manifest, deploys the subgraph, and exits
6. **executor** starts and waits for Manager connection
7. **manager** starts, reads deployed contract addresses, connects to the Executor, performs keyset handshake, and begins polling
8. **authorityservice** starts, reads deployed contract addresses, connects to the subgraph

---

## 8. Configuration Reference

### 8.1 Executor Environment Variables

| Variable | Description |
|----------|-------------|
| `CHANNEL_TYPE` | Communication channel: `tcp` (Docker) or `vsock` (Nitro Enclave) |
| `EXECUTOR_IP_HOST` | Executor IP address (used when `CHANNEL_TYPE=tcp`) |
| `EXECUTOR_PORT` | Executor listening port |
| `EXECUTOR_KEYSET_RECOVERY_TYPE` | Keyset recovery mode: `0` = fixed keys from env vars |
| `EXECUTOR_FIXED_SIGNING_KEY` | secp256k1 private key (dev mode only) |
| `EXECUTOR_FIXED_COMMUNICATION_KEY` | P-521 private key (dev mode only) |
| `EXECUTOR_FIXED_STATE_KEY` | AES-256 key (dev mode only) |
| `EXECUTOR_FUEL_PRICE_PER_UNIT` | Wei per fuel unit |
| `EXECUTOR_MIN_FEE_PER_REQUEST` | Minimum fee enforced per request |
| `EXECUTOR_COMMUNICATION_PARAMS_REQUEST_TIMEOUT_SEC` | Executor-side request timeout |
| `LOG_SERVER_IP_HOST` | Log server IP (points to Manager) |
| `EXECUTOR_LOG_*` | Logging configuration (kind, console, file, level, color) |

### 8.2 Manager Environment Variables

| Variable | Description |
|----------|-------------|
| `MANAGER_IP_HOST` | Manager IP address |
| `MANAGER_ADMIN_PORT` | Admin interface port |
| `MANAGER_KEY_SECP256` | Manager's secp256k1 private key for blockchain transactions |
| `MANAGER_DATA_FOLDER` | Path for LevelDB storage (default: `/data`) |
| `MANAGER_REPORTS_FOLDER` | Path for deanonymization reports (default: `/reports`) |
| `MANAGER_ADMIN_COMMUNICATION_PARAMS_REQUEST_TIMEOUT_SEC` | Admin request timeout |
| `MANAGER_COMMUNICATION_PARAMS_REQUEST_TIMEOUT_SEC` | Executor communication timeout |
| `BLOCKCHAIN_POLLING_INTERVAL` | Seconds between blockchain polls |
| `REORG_TIMEOUT` | Wait time for chain reorganization stabilization |
| `HANDSHAKE_TIMEOUT` | Keyset handshake timeout |
| `CHAIN_RPC_PROTOCOL` | RPC protocol (`http` or `https`) |
| `CHAIN_RPC_ADDRESS` | Chain RPC IP address |
| `CHAIN_RPC_PORT` | Chain RPC port |
| `CHAIN_PROCESSOR_ADDRESS` | ProcessorEndpoint contract address |
| `CHAIN_TEEAUTHENTICATOR_ADDRESS` | TeeAuthenticator contract address |
| `MANAGER_LOG_*` | Logging configuration |
| `LOG_SERVER_*` | Log server configuration (including file rotation) |

### 8.3 Deployer Environment Variables

| Variable | Description |
|----------|-------------|
| `DEPLOYER_PRIVATE_KEY` | Account used for contract deployment |
| `DEPLOYER_ADMIN` | Admin address for contract access control |
| `UPDATE_STATUS_OPERATOR` | Address authorized for state updates (Manager's address) |
| `TEE_NO_ATTESTATION` | `true` for dev mode (skip Nitro attestation) |
| `TEE_SIGNER_ADDRESS` | TEE signing address (derived from Executor signing key) |
| `TEE_PUB_P521` | TEE P-521 public key (derived from Executor communication key) |
| `MIN_FEE_PER_REQUEST` | Minimum fee per request (in Wei) |

