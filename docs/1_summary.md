# Horizen CCE - Architecture Summary

## 1. System Overview

Horizen CCE is a platform that executes WebAssembly (WASM) modules inside AWS Nitro Enclaves (TEE), with encrypted state management and blockchain-based coordination. Users submit requests on-chain; the system processes them securely off-chain and posts signed results back to the blockchain.

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
└──────────────────────────────┼────────────────────────────────────────┘
                               │ bidirectional messages
                               │ (JSON over VSock)
┌──────────────────────────────▼────────────────────────────────────────┐
│                    WASM Executor (AWS Nitro Enclave)                  │
│                                                                      │
│  ┌──────────────┐   ┌───────────────────┐   ┌─────────────────────┐  │
│  │  Enclave      │   │  Wasmtime Runtime │   │  Crypto Operations  │  │
│  │  KeySet       │   │  (WASM execution) │   │  - AES-256 state    │  │
│  │  - AES state  │   │                   │   │  - P521 ECDH events │  │
│  │  - secp256k1  │   │  ┌─────────────┐  │   │  - secp256k1 sign   │  │
│  │    signing    │   │  │ WASM Guest  │  │   └─────────────────────┘  │
│  │  - P521 comm  │   │  │ Application │  │                            │
│  └──────────────┘   │  └─────────────┘  │                            │
│                      └───────────────────┘                            │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 2. Main Components

### 2.1 Smart Contracts (Blockchain Layer)

The on-chain layer coordinates request submission, TEE attestation, and state management.

#### ProcessorEndpoint

The primary contract. Manages a FIFO request queue and processes state updates.

- **Request submission** (`submitRequest`): Users enqueue requests with a deposit and max fee. Each request includes an application ID, request type, and encrypted payload.
- **State update** (`stateUpdate`): Called by the Manager after execution. Verifies the TEE signature, updates the on-chain state root, emits encrypted events, processes withdrawals and refunds via pull-payment.
- **Failure handling** (`markRequestFailed`): Marks a request as failed and refunds the sender.
- **Access control**: `UPDATE_STATUS_ROLE` for state updates; `ADMIN` for configuration.

Key data structures:
```solidity
struct PendingRequest {
    uint256 timestamp;
    uint256 depositAmount;
    uint256 maxFeeValue;
    bytes32 requestId;
    bytes   payload;
    address sender;
    uint64  applicationId;
    uint8   protocolVersion;
    RequestType requestType;  // DEPLOYAPP | PROCESS | DEANONYMIZATION | ASSOCIATEKEY
}
```

Events: `RequestSubmitted`, `RequestCompleted`, `StateRootUpdate`, `UserEvent` (encrypted), `Refund`, `Withdrawal`.

#### TeeAuthenticator

Manages TEE identity. Verifies AWS Nitro Enclave attestation documents against expected PCR0 values and registers the TEE's signing address and P521 public key.

- **Single-step attestation** (`updateTee`): For small attestations.
- **Multi-step attestation** (`updateTeeStep1`..`4`): Breaks large attestation verification across multiple transactions to fit gas limits.
- **Signature verification** (`checkSignature`): Called by ProcessorEndpoint during `stateUpdate`. Recovers the signer from an ECDSA signature and verifies it matches the registered TEE address.

The signed message covers: `applicationId`, `prevStateRoot`, `newStateRoot`, `requestId`, `hash(events)`, `hash(eventSubTypes)`, `hash(withdrawals)`, `refundAmount`, `applicationFee`.

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
6. Start polling loop

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
  6. On failure: rollback state, mark request failed on-chain
```

**Key interfaces:**

| Interface | Purpose |
|-----------|---------|
| `blockchain.Client` | Fetch requests, submit state updates, mark failures |
| `communication.ExecutorClient` | Send process/deploy requests, handle keyset handshake |
| `storage.DataLayer` | Store/retrieve versioned state, WASM bytecode, keyset recovery |

**Storage layer** (Versioned LevelDB):
- Each state update creates a new version (ID = SHA256 of encrypted state)
- Retains last N versions for rollback on chain reorganization
- Non-versioned storage for enclave keyset recovery data

### 2.3 WASM Executor

The Executor runs inside the AWS Nitro Enclave (TEE) and handles all cryptographic operations and WASM execution.

**Enclave KeySet** — three keys generated on first start and recovered on restart:

| Key | Type | Purpose |
|-----|------|---------|
| `StateKey` | AES-256 | Encrypt/decrypt application state |
| `SigningKey` | secp256k1 | Sign update payloads (verified on-chain) |
| `CommunicationKey` | P-521 (ECDH) | Encrypt events, payloads, and reports |

**Keyset handshake** (on connection):
1. Executor asks Manager for stored recovery data
2. If found: restore keys from encrypted recovery blob
3. If not found: generate new keyset, send encrypted recovery to Manager for storage

**Request processing pipeline** (`HandleProcessRequest`):
```
1. Decrypt application state (AES-256 with StateKey)
2. Deserialize into AppData (state + user key store)
3. If deposit: call WASM deposit()
4. If AssociateKey: register sender's P521 public key
5. If Deanonymize: decrypt payload (ECDH), call WASM generate_deanonymization_report()
6. Otherwise: decrypt payload (ECDH), call WASM process_request()
7. Encrypt new state (AES-256), compute state root (SHA256)
8. Encrypt events using recipient P521 public keys (ECDH → AES-GCM)
9. Sign UpdatePayload with secp256k1 SigningKey
10. Return: UpdatePayload + encrypted ApplicationState + optional DeanonymizationReport
```

---

## 3. Application Flow (End-to-End)

### 3.1 Deploy Application

```
User                     Blockchain              Manager                Executor
  │                          │                      │                      │
  │─ submitRequest(DEPLOY) ──>│                      │                      │
  │   (WASM bytecode in       │                      │                      │
  │    payload, deposit+fee)  │                      │                      │
  │                          │<── poll ──────────────│                      │
  │                          │── PendingRequest ────>│                      │
  │                          │                      │── SendDeployApp ─────>│
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
  │<── RequestCompleted ─────│                      │                      │
```

### 3.2 Process Request (Standard)

```
User                     Blockchain              Manager                Executor
  │                          │                      │                      │
  │─ submitRequest(PROCESS) ─>│                      │                      │
  │   (encrypted payload,     │                      │                      │
  │    deposit+fee)           │                      │                      │
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

Before a user can receive encrypted events or submit encrypted payloads, they register their P-521 public key on-chain via an `ASSOCIATEKEY` request. The Executor stores this key in the user key store (part of AppData). Subsequent event encryption uses this key via ECDH.

### 3.4 Deanonymization

An authorized authority submits a `DEANONYMIZATION` request. The Executor calls the WASM module's dedicated `generate_deanonymization_report()` export (separate from `process_request()`). The WASM application generates a report (e.g., all account balances). The Executor encrypts this report with the authority's P-521 public key. The Manager stores the encrypted report on the filesystem. The authority retrieves it via the Authority Service HTTP API (`GET /nonce` + `POST /getreport`).

---

## 4. WASM Guest Interface

To develop a new WASM application for Horizen PES, you implement a TinyGo program that exports specific functions, compile it to WASM with `tinygo build -target=wasi`, and deploy it on-chain.

### 4.1 Required Exports

The WASM module must export these functions:

#### `load_module(appId: i64) -> *byte`

Called once when the application is deployed. Returns the initial state.

```go
//export load_module
func load_module(appId int64) *byte {
    result := myapp.LoadModule(appId)
    return myapp.SerializeAndWriteResult(result)
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

#### `deposit(appId: i64, senderPtr: *byte, senderLen: i32, valuePtr: *byte, valueLen: i32, statePtr: *byte, stateLen: i32) -> *byte`

Called when a user sends funds (deposit) with their request.

```go
//export deposit
func deposit(appId int64, senderPtr *byte, senderLen int32,
             valuePtr *byte, valueLen int32,
             statePtr *byte, stateLen int32) *byte {
    sender := myapp.PtrToAddress(senderPtr, senderLen)
    value  := myapp.PtrToUint256(valuePtr, valueLen)
    state  := utils.PtrToString(statePtr, stateLen)
    result := myapp.DepositFunds(sender, value, state)
    return myapp.SerializeAndWriteResult(result)
}
```

Returns `DepositResult`:
```go
type DepositResult struct {
    State  []byte       `json:"state"`            // Updated state
    Events []PlainEvent `json:"events"`            // Emitted events
    Fuel   *Uint256     `json:"fuel"`              // Fuel consumed
    Error  string       `json:"error,omitempty"`
}
```

#### `process_request(appId: i64, senderPtr: *byte, senderLen: i32, payloadPtr: *byte, payloadLen: i32, statePtr: *byte, stateLen: i32) -> *byte`

Main processing function. Called for `PROCESS` requests. The Executor routes requests by type — only standard processing goes through this function (deanonymization has its own export, and `ASSOCIATEKEY` is handled by the Executor directly).

```go
//export process_request
func process_request(appId int64, senderPtr *byte, senderLen int32,
                     payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    sender  := myapp.PtrToAddress(senderPtr, senderLen)
    payload := utils.PtrToString(payloadPtr, payloadLen)
    state   := utils.PtrToString(statePtr, stateLen)
    result  := myapp.ProcessRequest(sender, payload, state)
    return myapp.SerializeAndWriteResult(result)
}
```

Returns `ProcessResult`:
```go
type ProcessResult struct {
    State       []byte       `json:"state"`              // Updated state
    Events      []PlainEvent `json:"events"`              // Emitted events
    Withdrawals []Withdrawal `json:"withdrawals"`         // Fund transfers
    Fuel        *Uint256     `json:"fuel"`                // Fuel consumed
    Error       string       `json:"error,omitempty"`
}
```

#### `generate_deanonymization_report(payloadPtr: *byte, payloadLen: i32, statePtr: *byte, stateLen: i32) -> *byte`

Called when the Executor receives a `DEANONYMIZATION` request. This is a separate export from `process_request`. Note: no `appId` or `sender` parameters — the Executor handles authorization.

```go
//export generate_deanonymization_report
func generate_deanonymization_report(payloadPtr *byte, payloadLen int32,
                                     statePtr *byte, stateLen int32) *byte {
    payload := utils.PtrToString(payloadPtr, payloadLen)
    state   := utils.PtrToString(statePtr, stateLen)
    result  := myapp.GenerateDeanonymizationReport(payload, state)
    return myapp.SerializeAndWriteResult(result)
}
```

Returns `DeanonymizationResult`:
```go
type DeanonymizationResult struct {
    Report []byte   `json:"report"`            // Encrypted report data
    Fuel   *Uint256 `json:"fuel"`              // Fuel consumed
    Error  string   `json:"error,omitempty"`
}
```

#### `allocate` and `deallocate`

Memory management functions. These must be implemented in the WASM module's `utils` package. They manage memory allocation and deallocation for data exchange between host and guest.

```go
//export allocate
func allocate(size int32) int32        // allocates memory in WASM linear memory

//export deallocate
func deallocate(ptr *byte, size int32) // frees previously allocated memory
```

The `utils` package also provides `StringToPtr` for returning results (4-byte little-endian length prefix + data) and `PtrToString` for reading host-provided data.

#### `get_memory_stats() -> *byte` (optional)

Returns memory allocation statistics. Useful for debugging.

### 4.2 Types

The WASM guest module defines its own types locally (not imported from the host). This is a deliberate design choice: the guest is a separate sandboxed environment compiled with TinyGo, which cannot import host packages (e.g., `go-ethereum`). Communication between host and guest happens via JSON serialization — both sides define compatible types independently.

**Core types** (defined in the guest `app` package):

| Type | Description |
|------|-------------|
| `Address` | 20-byte Ethereum address (`[20]byte`). JSON: `"0x..."` hex string. |
| `Uint256` | 256-bit unsigned integer (`[4]uint64`). JSON: `"0x..."` hex string. Supports `Add`, `Sub`, `Cmp`, `AddOverflow`, `IsZero`. Custom implementation (no `math/big` in TinyGo). |
| `PlainEvent` | Event to emit: `UserID` (Address), `EventSubType` (string), `Data` ([]byte). |
| `Withdrawal` | Withdrawal instruction: `DestinationAddress` (Address), `Amount` (*Uint256). |

**Helper functions** (defined in the guest `app` package):

| Function | Description |
|----------|-------------|
| `SerializeAndWriteResult(result)` | Serialize result to JSON and allocate in WASM memory |
| `PtrToAddress(ptr, len)` | Convert WASM pointer to `*Address` |
| `PtrToUint256(ptr, len)` | Convert WASM pointer to `*Uint256` |

**Utility functions** (defined in the guest `utils` package):

| Function | Description |
|----------|-------------|
| `PtrToString(ptr, len)` | Convert WASM pointer to Go string |
| `StringToPtr(data)` | Convert Go byte slice to WASM pointer (4-byte length prefix + data) |
| `LogError/Warn/Info/Debug/Trace(fmt, args...)` | Logging with prefixes (ERR/WRN/INF/DBG/TRC), captured by host via WASI stdout pipe |

### 4.3 Request Types

The Executor routes requests by type to the appropriate WASM export:

| Value | Type | WASM Export Called |
|-------|------|-------------------|
| 0 | `DEPLOYAPP` | `load_module` |
| 1 | `PROCESS` | `deposit` (if funds included) + `process_request` |
| 2 | `DEANONYMIZATION` | `generate_deanonymization_report` |
| 3 | `ASSOCIATEKEY` | None (handled entirely by the Executor) |

Note: the `requestType` is **not** passed to the WASM module. The Executor determines which export to call based on the request type and invokes the correct function directly.

### 4.4 State Management

- State is an opaque `[]byte` from the host's perspective — your app defines its own format (typically JSON).
- State is passed as a JSON string to each function and returned as `[]byte` in the result.
- The Executor encrypts state with AES-256 before storing; your app never handles encryption.
- State root = SHA256(encrypted state). Used for on-chain consistency.
- State is versioned in LevelDB, enabling rollback on chain reorganizations.

### 4.5 Events

Events are the primary mechanism to communicate results to users.

- Each `PlainEvent` targets a specific user (`UserID`).
- The Executor encrypts each event using the target user's registered P-521 public key (ECDH → AES-GCM).
- Events are emitted on-chain as `UserEvent(appId, requestId, eventSubType, encryptedData)`.
- Users decrypt events off-chain using their P-521 private key.
- `EventSubType` is a plaintext string for event filtering (not encrypted).

### 4.6 Withdrawals

Return `Withdrawal` entries to transfer funds from the contract to a destination address. The smart contract validates that total withdrawals do not exceed the contract balance, and credits are distributed via pull-payment.

### 4.7 Fuel

Each operation must return a `Fuel` value representing computation cost. Fuel is priced at a configurable Wei-per-unit rate. The total fee (fuel cost) plus refund must equal the user's `maxFeeValue`.

### 4.8 Deanonymization Reports

When the request type is `DEANONYMIZATION`, the Executor calls the dedicated `generate_deanonymization_report()` export (not `process_request()`). The application **must** return a non-nil `Report` in the `DeanonymizationResult`. The Executor encrypts this report with the requesting authority's P-521 public key. The authority retrieves it via the Authority Service.


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

On Executor restart, the handshake protocol recovers the keyset from encrypted recovery data stored by the Manager. This ensures the same signing address and encryption keys are used across restarts, maintaining continuity with on-chain identity.

### Error Handling

- **Transient errors** (WASM failure, invalid payload): Mark request failed on-chain, refund user, continue polling.
- **Fatal errors** (storage corruption, unrecoverable state mismatch): Shutdown for manual intervention.
- **Blockchain submission errors**: Detect reorg vs. permanent failure; rollback or mark failed accordingly.
