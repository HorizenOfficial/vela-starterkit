# Building a Private Transfer App on Vela (v0.1.0)

This document walks through the implementation of a real WASM application for the Vela CCE platform: a **private transfer app** that supports deposits, account-to-account transfers, withdrawals to external addresses, and regulatory deanonymization. It serves as a practical reference for developers building their own applications.
It shows some code snippets in GO language.

> **Prerequisite**: Read `1_summary.md` first for the full system architecture (smart contracts, Manager, Executor, cryptographic design). This document focuses on the guest application — the WASM module you write and deploy.

App code is available in the following repository:

https://github.com/HorizenOfficial/vela-nova <br>
*(this guide is based on v0.1.0 tag)*



---

## What This App Does

The private transfer app implements a simple account-based ledger running inside a TEE. Users interact with it through on-chain requests; all balances and transfers are hidden from external observers.

| Operation | Description |
|-----------|-------------|
| **Deposit** | Fund an account from an on-chain transaction |
| **Transfer** | Move funds between accounts within the app |
| **Withdraw** | Send funds from an account back to an external address on-chain |
| **Deanonymize** | Generate an encrypted report of all account balances for an authorized auditor |

---

## Project Structure

```
runtime/wasm-go/
  main.go              # WASM exports (bridge layer)
  app/
    types.go           # Application-specific types (state, instructions, events)
    app.go             # Core business logic
  Makefile             # Build targets (dev and production)
  go.mod               # Dependencies (vela, vela-common-go)
  integration_test.go  # Tests against the WASM runtime
  system_tests/        # Full end-to-end tests (deploy, encrypt, blockchain)
```

The key insight: **`main.go` is a thin bridge** that converts raw WASM pointers into Go types and delegates to the `app` package. All business logic lives in `app/`. Core types (`Address`, `Uint256`, `PlainEvent`, result structs), memory management (`allocate`/`deallocate`), and WASM utilities are provided by the shared library `vela-common-go`.

---

## Step 1: Set Up the Module

### go.mod

The app references the main vela repository and the shared WASM types library:

```
require (
    github.com/HorizenOfficial/vela v0.1.0
    github.com/HorizenOfficial/vela-common-go v0.1.0
)
```

In v0.0.25, common WASM types (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result structs) and helper functions are provided by the shared library `vela-common-go`. The guest still communicates with the host via JSON serialization, but type definitions are now shared across WASM applications rather than duplicated in each one.

| Package | Provides |
|---------|----------|
| `vela-common-go/wasm/types` | Core types (`Address`, `Uint256`, `PlainEvent`, `AppEvent`, `Withdrawal`), result types (`LoadModuleResult`, `DepositResult`, `ProcessResult`, `DeployResult`), serialization helpers (`SerializeAndWriteResult`, `PtrToAddress`, `PtrToUint256`) |
| `vela-common-go/wasm/utils` | Memory management (`allocate`/`deallocate`, `BytesToPtr`, `get_allocated_memory_stats`), pointer conversion (`PtrToString`), logging (`LogError/Warn/Info/Debug/Trace`) |
| `vela/pkg/common` | Request type constants (`Deploy`, `Process`, `Deanonymize`, `AssociateKey`) used for routing inside `process_request` |
| `app` (local) | Application-specific business logic, state types, instruction types, event types |


### Makefile

```makefile
# Development build (with debug symbols)
tinygo build -o build/payment_app.wasm -target=wasi .

# Production build (optimized, no debug symbols)
tinygo build -o production_build/payment_app.wasm -opt=s -no-debug -target=wasi .
```

The target is always `wasi`. TinyGo compiles your Go code to a WebAssembly module that the Executor loads and runs inside the enclave.

---

## Step 2: Design Your State

All application state is serialized to JSON, encrypted by the Executor (AES-256), and stored between requests. You receive it as a string, deserialize it, modify it, and return the updated version.

### State Schema

```go
// ApplicationInternalState is the root state container.
type ApplicationInternalState struct {
    AppID         uint64                   `json:"appId"`
    Accounts      map[string]*AccountState `json:"accounts"`
    AllowedTokens map[string]bool          `json:"allowedTokens"` // token address hex -> allowed
    Nonce         uint64                   `json:"nonce"`
    Transactions  []TransactionRecord      `json:"transactions,omitempty"`
}

// AccountState represents a single user account with per-token balances.
type AccountState struct {
    Address  types.Address             `json:"address"`
    Balances map[string]*types.Uint256 `json:"balances"` // token address hex -> balance
}

// TransactionRecord is an in-state audit log entry appended on every
// state-changing operation. A bounded ring buffer (`MaxTransactions = 50`)
// is used to cap state growth.
type TransactionRecord struct {
    Type         string         `json:"type"` // "deposit", "transfer", "withdrawal"
    From         types.Address  `json:"from"`
    To           types.Address  `json:"to"`
    TokenAddress types.Address  `json:"tokenAddress"`
    Amount       *types.Uint256 `json:"amount"`
    Nonce        uint64         `json:"nonce"`
    Timestamp    int64          `json:"timestamp"`
    InvoiceID    string         `json:"invoice_id,omitempty"`
}

// DeployParams carries the constructor parameters the app receives via
// the `deploy` export — in this app, the initial ERC-20 token allowlist.
// ETH (address(0)) is always allowed implicitly.
type DeployParams struct {
    AllowedTokens []string `json:"allowedTokens"`
}
```

Design choices:
- **Accounts keyed by hex string** (e.g., `"0xadd...01"`). This avoids issues with binary map keys in JSON serialization.
- **Per-token balances** — an account holds one balance per token address (`"0x00…00"` for ETH, ERC-20 contract address otherwise). The set of accepted tokens is narrowed further by the app's own `AllowedTokens` map, seeded by constructor params at deploy time.
- **Global nonce** incremented on every state-changing operation. Included in events so clients can verify ordering.
- **In-state transaction log** (`Transactions`) — a bounded ring buffer (cap `MaxTransactions = 50`) maintained so a deanonymization report can include recent activity. Old entries are dropped on overflow.
- **`types.Uint256`** for balances — provided by `vela-common-go/wasm/types`. A 256-bit unsigned integer (`[4]uint64`) that matches Ethereum's native precision. Supports `Add`, `Sub`, `Cmp`, `AddOverflow`, `SubOverflow`, and `IsZero`.
- **`types.Address`** — provided by `vela-common-go/wasm/types`. A `[20]byte` type (not `go-ethereum`'s `common.Address`). JSON serialization produces the same `"0x..."` format, ensuring compatibility with the host.

---

## Step 3: Implement the WASM Exports

Your module must export these functions: `deploy`, `load_module`, `deposit`, `process_request`, plus `allocate`/`deallocate` (provided by `vela-common-go/wasm/utils`).

> **Change from v0.0.18**: The `generate_deanonymization_report` export has been removed. Deanonymization is now handled inside `process_request` via the `requestType` parameter.
>
> **Change from v0.0.25**: A new `deploy` export receives JSON-encoded constructor parameters (the `constructorParams` field of the deploy descriptor). `load_module` is retained for cache warm-up on Executor restart — new deployments go through `deploy`.

### main.go — The Bridge Layer

```go
package main

import (
    "github.com/HorizenOfficial/vela-common-go/wasm/types"
    "github.com/HorizenOfficial/vela-common-go/wasm/utils"
    "github.com/HorizenOfficial/vela-nova/payment-app/app"
)

//export deploy
func deploy(appId int64, paramsPtr *byte, paramsLen int32) *byte {
    paramsJSON := utils.PtrToString(paramsPtr, paramsLen)
    result := app.Deploy(appId, paramsJSON)
    return types.SerializeAndWriteResult(result)
}

//export load_module
func load_module(appId int64) *byte {
    result := app.LoadModule(appId)
    return types.SerializeAndWriteResult(result)
}

//export deposit
func deposit(appId int64, senderPtr *byte, senderLen int32,
             tokenPtr *byte, tokenLen int32,
             valuePtr *byte, valueLen int32,
             statePtr *byte, stateLen int32) *byte {
    _ = appId
    sender := types.PtrToAddress(senderPtr, senderLen)
    token := types.PtrToAddress(tokenPtr, tokenLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    value := types.PtrToUint256(valuePtr, valueLen)
    result := app.DepositFunds(sender, token, value, stateJSON)
    return types.SerializeAndWriteResult(result)
}

//export process_request
func process_request(appId int64, senderPtr *byte, senderLen int32,
                     requestType int32,
                     payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    _ = appId
    sender := types.PtrToAddress(senderPtr, senderLen)
    payloadJSON := utils.PtrToString(payloadPtr, payloadLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.ProcessRequest(sender, requestType, payloadJSON, stateJSON)
    return types.SerializeAndWriteResult(result)
}

//export get_memory_stats
func get_memory_stats() *byte {
    result := app.GetAllocatedMemoryStats()
    return types.SerializeAndWriteResult(result)
}

func main() {} // Required by Go, not used in WASM
```

Key patterns:
- Use `types.PtrToAddress`, `types.PtrToUint256`, `utils.PtrToString` to convert WASM pointers into Go types. These are now imported from `vela-common-go`.
- Always return via `types.SerializeAndWriteResult(result)` — this serializes your result to JSON and writes it into WASM linear memory for the host to read (4-byte little-endian length prefix + data).
- The `//export` directive is required for TinyGo to expose the function to the host.
- **`deploy` receives constructor params** as JSON. The app unmarshals them into a `DeployParams` struct and builds the initial state (here, seeding the ERC-20 token allowlist).
- **`deposit` receives a `tokenPtr`** identifying which token is being deposited (`0x0` for native ETH, otherwise the ERC-20 contract address that the on-chain `TokenAllowlist` has whitelisted).
- **`process_request` receives a `requestType int32` parameter** — the app uses this to determine the operation (e.g., standard processing vs. deanonymization). Both `PROCESS` and `DEANONYMIZATION` requests go through this single export.
- **`ASSOCIATEKEY` requests** are handled entirely by the Executor and never reach the WASM module.

---

## Step 4: Implement Business Logic

### Deploy — Initialization with Constructor Params

Called once when the application is deployed on-chain (via `submitDeployRequest`). Receives the JSON-encoded `constructorParams` from the deploy descriptor and returns the initial state.

```go
func Deploy(appId int64, paramsJSON string) types.DeployResult {
    allowedTokens := map[string]bool{ethTokenHex: true} // ETH always allowed

    if paramsJSON != "" {
        var params DeployParams
        if err := json.Unmarshal([]byte(paramsJSON), &params); err != nil {
            return types.DeployResult{Error: fmt.Sprintf("failed to parse deploy params: %v", err)}
        }
        for _, tokenHex := range params.AllowedTokens {
            if _, err := types.HexToAddress(tokenHex); err != nil {
                return types.DeployResult{Error: fmt.Sprintf("invalid token address %q: %v", tokenHex, err)}
            }
            allowedTokens[tokenHex] = true
        }
    }

    initialState := &ApplicationInternalState{
        AppID:         uint64(appId),
        Accounts:      make(map[string]*AccountState),
        AllowedTokens: allowedTokens,
    }
    stateJSON, _ := json.Marshal(initialState)
    return types.DeployResult{
        State: stateJSON,
        Fuel:  types.NewUint256(5),
    }
}
```

### LoadModule — Cache Warm-Up Fallback

Called by the Executor on restart to rebuild the default-initial-state cache entry without re-running the constructor. Returns the same minimal state as `Deploy` with no constructor input (ETH-only allowlist). Apps never need to call this themselves.

```go
func LoadModule(appId int64) types.LoadModuleResult {
    initialState := &ApplicationInternalState{
        AppID:         uint64(appId),
        Accounts:      make(map[string]*AccountState),
        AllowedTokens: map[string]bool{ethTokenHex: true},
    }
    stateJSON, _ := json.Marshal(initialState)
    return types.LoadModuleResult{
        State: stateJSON,
        Fuel:  types.NewUint256(5),
    }
}
```

### Deposit — Funding Accounts

Called when a user submits a request with a non-zero `assetAmount`. The Executor calls `deposit()` before `process_request()` if the request includes funds. The `tokenPtr` argument identifies which token is being deposited (ETH sentinel `0x0` or an ERC-20 address).

```go
func DepositFunds(senderPtr, tokenPtr *types.Address, value *types.Uint256, stateJSON string) types.DepositResult {
    // 1. Validate inputs
    if senderPtr == nil {
        return types.DepositResult{Error: "Sender address is nil"}
    }
    if tokenPtr == nil {
        return types.DepositResult{Error: "Token address is nil"}
    }
    if value == nil {
        return types.DepositResult{Error: "value is nil"}
    }

    senderHex := senderPtr.Hex()
    tokenHex := resolveTokenHex(*tokenPtr) // "0x00…00" for ETH

    // 2. Deserialize state
    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return types.DepositResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    // 3. Enforce the app-level allowlist (ETH always, ERC-20 only if deploy-time opted in)
    if !currentState.AllowedTokens[tokenHex] {
        return types.DepositResult{Error: fmt.Sprintf("Token %s not allowed by this application", tokenHex)}
    }

    var events []types.PlainEvent

    // 4. Process deposit (skip zero-value deposits)
    if !value.IsZero() {
        // Create account if it doesn't exist
        if currentState.Accounts[senderHex] == nil {
            currentState.Accounts[senderHex] = &AccountState{
                Address:  *senderPtr,
                Balances: make(map[string]*types.Uint256),
            }
        }

        balance := getOrCreateTokenBalance(currentState.Accounts[senderHex], tokenHex)

        // Add to balance with overflow check
        oldBalance := *balance
        if balance.AddOverflow(*balance, *value) {
            *balance = oldBalance
            return types.DepositResult{Error: fmt.Sprintf("Overflow while adding amount %s to balance: %s",
                value, oldBalance)}
        }

        currentState.Nonce++
        recordTransaction(&currentState, "deposit", *senderPtr, *senderPtr, *tokenPtr, value, "")

        // 5. Emit event to the depositor (DepositEvent fields include tokenAddress + balance)
        eventData, _ := json.Marshal(DepositEvent{
            Type: "deposit", TokenAddress: *tokenPtr, Amount: value,
            Balance: balance, Nonce: currentState.Nonce,
        })
        // EventSubType is intentionally left unset. The payment app assumes
        // users register a subtype seed at ASSOCIATEKEY time, and the Executor
        // overrides PlainEvent subtypes with an HMAC-derived value from that
        // seed (any value set here would be discarded). The Type discriminator
        // is carried inside the encrypted Data payload.
        events = append(events, types.PlainEvent{
            UserID: *senderPtr,
            Data:   eventData,
        })
    }

    // 6. Return updated state
    newStateBytes, _ := json.Marshal(&currentState)
    return types.DepositResult{
        State:  newStateBytes,
        Events: events,
        Fuel:   types.NewUint256(35),
    }
}
```

> **`EventSubType` is a `[32]byte`**, not a string. If the target user registered a subtype seed during `ASSOCIATEKEY` (226-byte payload), the Executor overrides whatever the app writes here with an HMAC-derived opaque subtype drawn from the user's seed — so the private transfer app deliberately leaves `EventSubType` zero. If no seed is registered, the Executor preserves the app-supplied value, and apps can pack short ASCII labels into the first bytes (`[32]byte{'d','e','p','o','s','i','t',0,…}`). The plaintext `Type` discriminator (`"deposit"`, `"transfer_sent"`, …) is carried inside the encrypted event data. See `SUBTYPE_KEY_MESSAGE` in the TypeScript client guide for the seed derivation.

### ProcessRequest — Transfers, Withdrawals, and Deanonymization

This is the main entry point for `PROCESS` and `DEANONYMIZATION` requests. The `requestType` parameter (passed by the Executor) takes precedence: when it equals `Deanonymize` (value 2), the app generates a report regardless of the payload. For standard requests, the operation type is determined by the payload's `type` field.

```go
func ProcessRequest(senderPtr *types.Address, requestType int32, payloadJSON, stateJSON string) types.ProcessResult {
    if senderPtr == nil {
        return types.ProcessResult{Error: "Sender address is missing"}
    }

    sender := *senderPtr
    senderHex := sender.Hex()

    // Parse state
    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    var events []types.PlainEvent
    var withdrawals []types.Withdrawal
    var instructions PayloadInstructions

    // Parse payload if present
    if payloadJSON != "" && payloadJSON != "{}" {
        if err := json.Unmarshal([]byte(payloadJSON), &instructions); err != nil {
            return types.ProcessResult{Error: fmt.Sprintf("Failed to parse payload instructions: %v", err)}
        }
    }

    // requestType takes precedence over payload type
    var instructionType string
    if requestType == int32(common.Deanonymize) {
        instructionType = "deanonymize"
    } else {
        instructionType = instructions.Type
    }

    switch instructionType {
    case "transfer":       // ...
    case "withdraw":       // ...
    case "deanonymize":    // generate report and return early
    default:
        return types.ProcessResult{Error: fmt.Sprintf("Unsupported instruction type: [%s]", instructionType)}
    }

    newStateBytes, _ := json.Marshal(currentState)
    return types.ProcessResult{
        State: newStateBytes, Events: events, Withdrawals: withdrawals, Fuel: types.NewUint256(50),
    }
}
```

#### Transfer

Moves funds between two accounts within the app, for a specific token. Both sender and recipient receive encrypted events. Transfers support an optional `InvoiceID` field (max 100 characters) for payment tracking and an optional `TokenAddress` (defaults to ETH `0x0`).

```go
case "transfer":
    tokenHex := resolveTokenHex(instructions.Transfer.TokenAddress)
    if !currentState.AllowedTokens[tokenHex] {
        return types.ProcessResult{Error: fmt.Sprintf("Token %s not allowed", tokenHex)}
    }

    // Validate sender has sufficient balance in the chosen token
    if currentState.Accounts[senderHex] == nil {
        return types.ProcessResult{Error: fmt.Sprintf("Account %s does not exist!", senderHex)}
    }
    senderBalance := getOrCreateTokenBalance(currentState.Accounts[senderHex], tokenHex)
    if senderBalance.Cmp(*instructions.Transfer.Amount) < 0 {
        return types.ProcessResult{Error: "Insufficient balance for transfer"}
    }

    recipientHex := instructions.Transfer.To.Hex()
    if currentState.Accounts[recipientHex] == nil {
        currentState.Accounts[recipientHex] = &AccountState{
            Address:  instructions.Transfer.To,
            Balances: make(map[string]*types.Uint256),
        }
    }
    recipientBalance := getOrCreateTokenBalance(currentState.Accounts[recipientHex], tokenHex)

    // Execute transfer (save balances for revert on overflow)
    oldSenderBalance := *senderBalance
    oldRecipientBalance := *recipientBalance

    senderBalance.Sub(*senderBalance, *instructions.Transfer.Amount)
    if recipientBalance.AddOverflow(*recipientBalance, *instructions.Transfer.Amount) {
        // Revert both balances on overflow
        *senderBalance = oldSenderBalance
        *recipientBalance = oldRecipientBalance
        return types.ProcessResult{Error: "Overflow while adding transfer amount"}
    }
    currentState.Nonce++
    recordTransaction(&currentState, "transfer", sender, instructions.Transfer.To,
        instructions.Transfer.TokenAddress, instructions.Transfer.Amount, instructions.Transfer.InvoiceID)

    // Two events: one for sender, one for recipient (InvoiceID included if present).
    // EventSubType is left unset — see the note in DepositFunds; the Executor
    // fills it in from each recipient's registered seed.
    events = append(events,
        types.PlainEvent{UserID: sender,                    Data: ...},
        types.PlainEvent{UserID: instructions.Transfer.To,  Data: ...},
    )

    // Optional: if the transfer carries an InvoiceID, emit a plaintext
    // AppEvent whose EventSubType is a deterministic Keccak256 hash over
    // (lenPrefix || InvoiceID || sender || tokenAddress || amount || recipient).
    // Data is nil — the subtype itself is the verifiable receipt anyone can
    // recompute and look up on-chain (indexed bytes32 topic).
    if instructions.Transfer.InvoiceID != "" {
        invoiceIDBytes := []byte(instructions.Transfer.InvoiceID)
        var lenPrefix [4]byte
        binary.BigEndian.PutUint32(lenPrefix[:], uint32(len(invoiceIDBytes)))

        h := sha3.NewLegacyKeccak256()
        h.Write(lenPrefix[:])
        h.Write(invoiceIDBytes)
        h.Write(sender[:])
        h.Write(instructions.Transfer.TokenAddress[:])
        h.Write(instructions.Transfer.Amount.Bytes())
        h.Write(instructions.Transfer.To[:])

        var receiptSubType [32]byte
        copy(receiptSubType[:], h.Sum(nil))

        appEvents = append(appEvents, types.AppEvent{
            EventSubType: receiptSubType,
            Data:         nil,
        })
    }
```

The Executor encrypts each event with the target user's P-521 public key. The sender sees `transfer_sent`; the recipient sees `transfer_received` (both discriminated by the `Type` field inside the encrypted data). External observers see nothing about the private transfer itself — they only know a request was processed, plus the optional receipt topic when an `InvoiceID` is present. The receipt hash is collision-resistant on its inputs (length-prefixed `InvoiceID`, fixed 20-byte addresses, fixed 32-byte big-endian `Amount`) and lets a third party verify "this transfer happened" by recomputing the subtype from the transfer parameters.

#### Withdrawal

Removes funds from the app and instructs the smart contract to release them on-chain. The optional `TokenAddress` in the instruction selects which token to withdraw (ETH by default).

```go
case "withdraw":
    tokenHex := resolveTokenHex(instructions.Withdraw.TokenAddress)
    if !currentState.AllowedTokens[tokenHex] {
        return types.ProcessResult{Error: fmt.Sprintf("Token %s not allowed", tokenHex)}
    }
    if currentState.Accounts[senderHex] == nil {
        return types.ProcessResult{Error: fmt.Sprintf("Account %s does not exist", senderHex)}
    }
    senderBalance := getOrCreateTokenBalance(currentState.Accounts[senderHex], tokenHex)
    if senderBalance.Cmp(*instructions.Withdraw.Amount) < 0 {
        return types.ProcessResult{Error: fmt.Sprintf("Insufficient balance %s for withdrawal %s",
            senderBalance, *instructions.Withdraw.Amount)}
    }

    senderBalance.Sub(*senderBalance, *instructions.Withdraw.Amount)
    currentState.Nonce++
    recordTransaction(&currentState, "withdrawal", sender, instructions.Withdraw.To,
        instructions.Withdraw.TokenAddress, instructions.Withdraw.Amount, "")

    // Withdrawal instruction — processed by the smart contract.
    // TokenAddress selects the custody pool; the contract credits the
    // destination via pull-payment per token (`claim(tokenAddress, payee)`).
    withdrawals = append(withdrawals, types.Withdrawal{
        TokenAddress:       instructions.Withdraw.TokenAddress, // 0x0 = ETH
        DestinationAddress: instructions.Withdraw.To,
        Amount:             instructions.Withdraw.Amount,
    })

    // Event for the sender (EventSubType left unset — see DepositFunds)
    events = append(events, types.PlainEvent{
        UserID: sender, Data: ...,
    })
```

`Withdrawal` entries are included in the signed update payload. The smart contract verifies the TEE signature and credits the destination address via pull-payment per token.

#### Deanonymization (inside ProcessRequest)

When `requestType == common.Deanonymize`, the app generates a report and returns it in `ProcessResult.Report`. The state is **not modified** — the original state JSON is returned as-is to avoid unnecessary marshalling.

The reference app supports two report types, selected by the `reportType` field inside the payload (defaults to `"balances"` when the payload is empty or omits the field):

| `reportType` | Shape | Contents |
|---|---|---|
| `"balances"` (default) | `DeanonymizationReport{Accounts, Nonce}` | Full snapshot of all accounts' per-token balances and the current state nonce. |
| `"tx_history"` | `TxHistoryReport{Address, Balances, Transactions}` | The target `Address`'s current per-token balances plus all in-state transaction records involving it, optionally filtered to the `[FromTimestamp, ToTimestamp]` window. |

```go
case "deanonymize":
    reportType := "balances"
    if instructions.Deanonymize != nil && instructions.Deanonymize.ReportType != "" {
        reportType = instructions.Deanonymize.ReportType
    }

    var reportBytes []byte
    switch reportType {
    case "balances":
        reportBytes, _ = json.Marshal(DeanonymizationReport{
            Accounts: currentState.Accounts,
            Nonce:    currentState.Nonce,
        })
    case "tx_history":
        if instructions.Deanonymize.Address.IsZero() {
            return types.ProcessResult{Error: "tx_history report requires a non-zero address"}
        }
        addr := instructions.Deanonymize.Address
        fromTs := instructions.Deanonymize.FromTimestamp
        toTs   := instructions.Deanonymize.ToTimestamp

        filtered := []TransactionRecord{}
        for _, tx := range currentState.Transactions {
            if tx.From != addr && tx.To != addr {
                continue
            }
            if fromTs > 0 && tx.Timestamp < fromTs {
                continue
            }
            if toTs > 0 && tx.Timestamp > toTs {
                break
            }
            filtered = append(filtered, tx)
        }

        balances := map[string]*types.Uint256{}
        if acc := currentState.Accounts[addr.Hex()]; acc != nil && acc.Balances != nil {
            balances = acc.Balances
        }

        reportBytes, _ = json.Marshal(TxHistoryReport{
            Address:      addr,
            Balances:     balances,
            Transactions: filtered,
        })
    default:
        return types.ProcessResult{Error: fmt.Sprintf("Unsupported report type: %s", reportType)}
    }

    return types.ProcessResult{
        State:  []byte(stateJSON), // no state modification, reuse original
        Report: reportBytes,
        Fuel:   types.NewUint256(20),
    }
```

The Executor validates that the `Report` field is populated for deanonymization requests (and empty for non-deanonymization requests). It encrypts the report with the requesting authority's P-521 public key. Only authorized auditors (registered in the `AuthorityRegistry` contract) can submit deanonymization requests.

---

## Step 5: Payload Format

Users submit encrypted JSON payloads. After decryption, the Executor passes them to `process_request`.
Every app must define its own payloads.
Here are the payload formats your app defines:

**Transfer:**
```json
{
  "type": "transfer",
  "transfer": {
    "to": "0x1234567890abcdef1234567890abcdef12345678",
    "tokenAddress": "0x0000000000000000000000000000000000000000",
    "amount": "0x6f05b59d3b20000",
    "invoice_id": "INV-2025-001"
  }
}
```
The `tokenAddress` field is optional and defaults to ETH (`0x0`) if omitted; for an ERC-20 transfer, set it to the token contract address (must be in the app's `AllowedTokens`). The `invoice_id` field is optional (max 100 characters): it is included in both sender and recipient events for payment tracking, and also triggers emission of a plaintext `AppEvent` whose indexed `eventSubType` is a Keccak256 receipt over `(len(invoice_id) || invoice_id || sender || tokenAddress || amount || recipient)` — a public, verifiable proof that the transfer happened, without revealing balances.

**Withdrawal:**
```json
{
  "type": "withdraw",
  "withdraw": {
    "to": "0x1234567890abcdef1234567890abcdef12345678",
    "tokenAddress": "0x0000000000000000000000000000000000000000",
    "amount": "0x6f05b59d3b20000"
  }
}
```
`tokenAddress` is optional; defaults to ETH if omitted.

**Deanonymization:**

Empty payload (defaults to a full `"balances"` snapshot):
```json
{}
```

Explicit balances report (same as the default):
```json
{
  "deanonymize": { "reportType": "balances" }
}
```

Transaction-history report for one address, optionally time-windowed:
```json
{
  "deanonymize": {
    "reportType": "tx_history",
    "address": "0x1234567890abcdef1234567890abcdef12345678",
    "fromTimestamp": 1700000000,
    "toTimestamp":   1710000000
  }
}
```

The `requestType` parameter (set by the Executor to `2`) determines routing — the app checks `requestType == common.Deanonymize` and generates a report regardless of payload content. The payload fields above just select **which** report. `address` is required for `tx_history`; `fromTimestamp`/`toTimestamp` are optional (`0` = unbounded).

---

## Step 6: Events

Events are the only way to communicate results back to users.
Each event targets a specific user and is encrypted with their registered P-521 key.

Every app must define its own event. <br>
Here are the payload formats your app defines:

### Event Types

All four per-user events leave `PlainEvent.EventSubType` as the zero `[32]byte`. The `Type` string inside the (encrypted) `Data` is what distinguishes them at decryption time. On-chain, the indexed `eventSubType` topic is filled in by the Executor: if the recipient registered a subtype seed at `ASSOCIATEKEY` time, the Executor derives an HMAC-based opaque value from that seed so external observers can't correlate events across users; if no seed is registered, whatever the app wrote (zero, in this app) is emitted as-is.

| `Type` (inside Data) | Recipient | Data Fields |
|---|---|---|
| `deposit` | Depositor | `type`, `tokenAddress`, `amount`, `balance`, `nonce` |
| `transfer_sent` | Sender | `type`, `to`, `tokenAddress`, `amount`, `balance`, `nonce`, `invoice_id` (optional) |
| `transfer_received` | Recipient | `type`, `from`, `tokenAddress`, `amount`, `balance`, `nonce`, `invoice_id` (optional) |
| `withdrawal` | Withdrawer | `type`, `to`, `tokenAddress`, `amount`, `balance`, `nonce` |

In addition, a transfer that carries an `InvoiceID` emits one plaintext `AppEvent` with `Data = nil` and `EventSubType = keccak256(lenPrefix || InvoiceID || sender || tokenAddress || amount || recipient)` — a third party who knows the transfer parameters can recompute the hash and find the event on-chain as proof of execution.

### Emitting an Event

```go
eventData, _ := json.Marshal(DepositEvent{
    Type:         "deposit",
    TokenAddress: *tokenPtr,
    Amount:       value,
    Balance:      balance,
    Nonce:        currentState.Nonce,
})

// Per-user encrypted event. EventSubType is intentionally unset — see above.
events = append(events, types.PlainEvent{
    UserID: *senderPtr, // who receives this event
    Data:   eventData,  // encrypted by the Executor with the recipient's P-521 key
})
```

`EventSubType` is a `bytes32` value emitted as an **indexed** on-chain log topic (for filtering/indexing). For `PlainEvent`s targeted at seed-registered users, the Executor overrides whatever the app writes with an HMAC-derived subtype drawn from the user's seed (see `SUBTYPE_KEY_MESSAGE` and `generateSubtypeSet` in the TypeScript client guide). `Data` is encrypted — only the target user can read it.

If you need to emit a public, application-wide signal (not targeted at any single user), append to the `AppEvents` slice on `DepositResult` / `ProcessResult` instead. `AppEvent.Data` is **not** encrypted and is emitted on-chain as `AppEvent(applicationId, requestId, eventSubType, data)` — with `eventSubType` as an indexed `bytes32` topic that the Executor does **not** override (the app-supplied value is used verbatim).

---

## Step 7: Fuel

Every operation must return a `Fuel` value representing computation cost. The fee charged to the user is `fuel * weiPerFuelUnit` (configured on-chain). Any remaining deposit is refunded. A minimum fee per request (`MIN_FEE_PER_REQUEST`) is enforced at the contract level.

| Operation | Fuel |
|-----------|------|
| `LoadModule` | 5 |
| `Deposit` | 35 |
| `ProcessRequest` (transfer or withdraw) | 50 |
| `ProcessRequest` (deanonymize) | 20 |

Fuel values are application-defined. Set them proportional to the computational cost of each operation.

---

## Step 8: Error Handling

Return errors via the `Error` field of the result struct. When an error is returned, the Executor marks the request as failed on-chain and refunds the user.

```go
// Input validation
if senderPtr == nil {
    return ProcessResult{Error: "Sender address is missing"}
}

// Business rule validation
if currentState.Accounts[senderHex].Balance.Cmp(*amount) < 0 {
    return ProcessResult{Error: "Insufficient balance for transfer"}
}

// Overflow protection
oldBalance := *account.Balance
if account.Balance.AddOverflow(*account.Balance, *value) {
    *account.Balance = oldBalance  // revert
    return DepositResult{Error: fmt.Sprintf("Overflow while adding amount %s to balance: %s",
        value, oldBalance)}
}
```

Important: when returning an error, **do not modify state**. Either return early before mutations, or revert changes before returning.

---

## TinyGo Constraints

Since your code compiles with TinyGo (not standard Go), keep these limitations in mind:

| Constraint | Detail |
|------------|--------|
| **No `reflect`-heavy code** | Avoid packages that rely on deep reflection |
| **Limited stdlib** | `net`, `os` (beyond basics), `math/big`, and some other packages are unavailable |
| **Determinism required** | No `time.Now()`, no random numbers — same inputs must produce same outputs |
| **Stateless execution** | No global mutable state between invocations; everything goes through the state parameter |
| **JSON serialization** | `encoding/json` works. Keep state structures simple |
| **Logging** | Use `utils.LogError/Warn/Info/Debug/Trace` — writes to stdout with level prefixes (ERR/WRN/INF/DBG/TRC), captured by host via WASI pipe |
| **Memory** | `allocate`/`deallocate` are provided by `vela-common-go/wasm/utils` — auto-exported by TinyGo |
| **Shared types** | Use `Address`, `Uint256`, etc. from `vela-common-go/wasm/types` — do not import from the host (`go-ethereum`) |

---

## Local Development Setup

To test this app with the local dev environment:

1. Download `payment_app.wasm` from https://github.com/HorizenOfficial/vela-nova/releases/tag/v0.1.0
3. Start the environment: `cd dockerfiles && cp .env.dev .env && docker compose up`
4. Download the `nova-linux` wallet from the same release page
5. Configure `wallet.conf` (from `wallet.conf.template`):
    ```
    rpcUrl=http://localhost:8545
    ProcessorAddress=0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
    TeeAuthenticatorAddress=0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
    AuthorityServiceURL=http://localhost:8081
    SubgraphURL=http://localhost:8000/subgraphs/name/hcce
    ```
6. Set wallet keys:
    - `keySecp256k1`: use an Anvil default private key (accounts are pre-funded with 1000 ETH)
    - `keyP521`: generate with `./nova-linux generatekeys`
7. Deploy and interact:
    ```bash
    ./nova-linux deployapp --wasm /absolute/path/to/payment_app.wasm --max-value-fee "100 wei" (Use an account with `DEPLOYAPP` role)
    ./nova-linux registeruser
    ./nova-linux getpublicbalance
    ./nova-linux deposit -a "1 ETH"
    ./nova-linux getprivatebalance
    ```

---

## Summary

Building an application on Vela CCE comes down to:

1. **Define your state** — a JSON-serializable struct that holds all application data.
2. **Import shared types** — use `Address`, `Uint256`, `PlainEvent`, `AppEvent`, `Withdrawal`, and result types from `vela-common-go/wasm/types`.
3. **Implement the exports** — `deploy` (init with constructor params), `load_module` (cache warm-up fallback), `deposit` (fund accounts per token), `process_request` (transfers, withdrawals, and deanonymization via `requestType` routing).
4. **Memory management** — `allocate`/`deallocate` are provided by `vela-common-go/wasm/utils`.
5. **Return results** — updated state, events for users (subtype as `[32]byte`), withdrawals for on-chain transfers (with per-token `TokenAddress`), fuel for fee calculation, and optional report for deanonymization.
6. **Handle errors** — return early with an `Error` string; never leave state partially modified.
7. **Compile with TinyGo** — `tinygo build -target=wasi .`

The Executor handles all cryptography (state encryption, event encryption, payload decryption, signing). Your app works with plaintext data and lets the platform handle the rest.

### Related Resources

| Resource | URL |
|----------|-----|
| Vela Nova (test app source) | https://github.com/HorizenOfficial/vela-nova/releases/tag/v0.1.0 |
| Vela Common TS (browser client) | https://github.com/HorizenOfficial/vela-common-ts |
| Local dev environment | `dockerfiles/` in this repository |
