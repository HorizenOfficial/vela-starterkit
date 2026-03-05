# Building a Private Transfer App on Vela (v0.0.25)

This document walks through the implementation of a real WASM application for the Vela CCE platform: a **private transfer app** that supports deposits, account-to-account transfers, withdrawals to external addresses, and regulatory deanonymization. It serves as a practical reference for developers building their own applications.
It shows some code snippets in GO language.

> **Prerequisite**: Read `1_summary.md` first for the full system architecture (smart contracts, Manager, Executor, cryptographic design). This document focuses on the guest application — the WASM module you write and deploy.

App code is available in the following repository:

https://github.com/HorizenOfficial/vela-nova <br>
*(this guide is based on v0.0.25 tag)*



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
    github.com/HorizenOfficial/vela v0.0.25
    github.com/HorizenOfficial/vela-common-go v0.0.25
)
```

In v0.0.25, common WASM types (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result structs) and helper functions are provided by the shared library `vela-common-go`. The guest still communicates with the host via JSON serialization, but type definitions are now shared across WASM applications rather than duplicated in each one.

| Package | Provides |
|---------|----------|
| `vela-common-go/wasm/types` | Core types (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`), result types (`LoadModuleResult`, `DepositResult`, `ProcessResult`), serialization helpers (`SerializeAndWriteResult`, `PtrToAddress`, `PtrToUint256`) |
| `vela-common-go/wasm/utils` | Memory management (`allocate`/`deallocate`, `StringToPtr`), pointer conversion (`PtrToString`), logging (`LogError/Warn/Info/Debug/Trace`) |
| `vela/pkg/common` | Request type constants (`Process`, `Deanonymize`, etc.) used for routing inside `process_request` |
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
    AppID    uint64                   `json:"appId"`
    Accounts map[string]*AccountState `json:"accounts"`
    Nonce    uint64                   `json:"nonce"`
}

// AccountState represents a single user account.
type AccountState struct {
    Address types.Address  `json:"address"`
    Balance *types.Uint256 `json:"balance"`
}
```

Design choices:
- **Accounts keyed by hex string** (e.g., `"0xadd...01"`). This avoids issues with binary map keys in JSON serialization.
- **Global nonce** incremented on every state-changing operation. Included in events so clients can verify ordering.
- **`types.Uint256`** for balances — provided by `vela-common-go/wasm/types`. A 256-bit unsigned integer (`[4]uint64`) that matches Ethereum's native precision. Supports `Add`, `Sub`, `Cmp`, `AddOverflow`, and `IsZero`.
- **`types.Address`** — provided by `vela-common-go/wasm/types`. A `[20]byte` type (not `go-ethereum`'s `common.Address`). JSON serialization produces the same `"0x..."` format, ensuring compatibility with the host.

---

## Step 3: Implement the WASM Exports

Your module must export these functions: `load_module`, `deposit`, `process_request`, plus `allocate`/`deallocate` (provided by `vela-common-go/wasm/utils`).

> **Change from v0.0.18**: The `generate_deanonymization_report` export has been removed. Deanonymization is now handled inside `process_request` via the new `requestType` parameter.

### main.go — The Bridge Layer

```go
package main

import (
    "github.com/HorizenOfficial/vela-common-go/wasm/types"
    "github.com/HorizenOfficial/vela-common-go/wasm/utils"
    "github.com/HorizenOfficial/vela-nova/payment-app/app"
)

//export load_module
func load_module(appId int64) *byte {
    result := app.LoadModule(appId)
    return types.SerializeAndWriteResult(result)
}

//export deposit
func deposit(appId int64, senderPtr *byte, senderLen int32,
             valuePtr *byte, valueLen int32,
             statePtr *byte, stateLen int32) *byte {
    _ = appId
    sender := types.PtrToAddress(senderPtr, senderLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    value := types.PtrToUint256(valuePtr, valueLen)
    result := app.DepositFunds(sender, value, stateJSON)
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
- **`process_request` receives a `requestType int32` parameter** — the app uses this to determine the operation (e.g., standard processing vs. deanonymization). Both `PROCESS` and `DEANONYMIZATION` requests go through this single export.
- **`ASSOCIATEKEY` requests** are handled entirely by the Executor and never reach the WASM module.

---

## Step 4: Implement Business Logic

### LoadModule — Initialization

Called once when the application is deployed on-chain. Returns the initial empty state.

```go
func LoadModule(appId int64) types.LoadModuleResult {
    initialState := &ApplicationInternalState{
        AppID:    uint64(appId),
        Accounts: make(map[string]*AccountState),
    }
    stateJSON, err := json.Marshal(initialState)
    if err != nil {
        return types.LoadModuleResult{
            Error: fmt.Sprintf("failed to marshal initial state: %v", err),
        }
    }
    return types.LoadModuleResult{
        State: stateJSON,
        Fuel:  types.NewUint256(5),
    }
}
```

### Deposit — Funding Accounts

Called when a user submits a request with a deposit amount. The Executor calls `deposit()` before `process_request()` if the request includes funds.

```go
func DepositFunds(senderPtr *Address, value *Uint256, stateJSON string) DepositResult {
    // 1. Validate inputs
    if senderPtr == nil {
        return DepositResult{Error: "Sender address is nil"}
    }
    if value == nil {
        return DepositResult{Error: "value is nil"}
    }

    senderHex := senderPtr.Hex()

    // 2. Deserialize state
    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return DepositResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    var events []PlainEvent

    // 3. Process deposit (skip zero-value deposits)
    if !value.IsZero() {
        // Create account if it doesn't exist
        if currentState.Accounts[senderHex] == nil {
            currentState.Accounts[senderHex] = &AccountState{
                Address: *senderPtr,
                Balance: NewUint256(0),
            }
        }

        // Add to balance with overflow check
        oldBalance := *currentState.Accounts[senderHex].Balance
        if currentState.Accounts[senderHex].Balance.AddOverflow(
            *currentState.Accounts[senderHex].Balance, *value) {
            *currentState.Accounts[senderHex].Balance = oldBalance
            return DepositResult{Error: fmt.Sprintf("Overflow while adding amount %s to balance: %s",
                value, oldBalance)}
        }

        currentState.Nonce++

        // 4. Emit event to the depositor
        eventData, _ := json.Marshal(DepositEvent{
            Type: "deposit", Amount: value,
            Balance: currentState.Accounts[senderHex].Balance,
            Nonce: currentState.Nonce,
        })
        events = append(events, PlainEvent{
            UserID:       *senderPtr,
            EventSubType: "deposit",
            Data:         eventData,
        })
    }

    // 5. Return updated state
    newStateBytes, _ := json.Marshal(&currentState)
    return DepositResult{
        State:  newStateBytes,
        Events: events,
        Fuel:   NewUint256(35),
    }
}
```

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

Moves funds between two accounts within the app. Both sender and recipient receive encrypted events. Transfers support an optional `InvoiceID` field (max 100 characters) for payment tracking.

```go
case "transfer":
    // Validate sender has sufficient balance
    if currentState.Accounts[senderHex] == nil {
        return types.ProcessResult{Error: fmt.Sprintf("Account %s does not exist!", senderHex)}
    }
    if currentState.Accounts[senderHex].Balance.Cmp(*instructions.Transfer.Amount) < 0 {
        return types.ProcessResult{Error: "Insufficient balance for transfer"}
    }

    recipientHex := instructions.Transfer.To.Hex()

    // Create recipient account if needed
    if currentState.Accounts[recipientHex] == nil {
        currentState.Accounts[recipientHex] = &AccountState{
            Address: instructions.Transfer.To,
            Balance: types.NewUint256(0),
        }
    }

    // Execute transfer (save balances for revert on overflow)
    oldSenderBalance := *currentState.Accounts[senderHex].Balance
    oldRecipientBalance := *currentState.Accounts[recipientHex].Balance

    currentState.Accounts[senderHex].Balance.Sub(
        *currentState.Accounts[senderHex].Balance, *instructions.Transfer.Amount)
    if currentState.Accounts[recipientHex].Balance.AddOverflow(
        *currentState.Accounts[recipientHex].Balance, *instructions.Transfer.Amount) {
        // Revert both balances on overflow
        *currentState.Accounts[senderHex].Balance = oldSenderBalance
        *currentState.Accounts[recipientHex].Balance = oldRecipientBalance
        return types.ProcessResult{Error: "Overflow while adding transfer amount"}
    }
    currentState.Nonce++

    // Two events: one for sender, one for recipient (InvoiceID included if present)
    events = append(events,
        types.PlainEvent{UserID: sender, EventSubType: "transfer_sent", Data: ...},
        types.PlainEvent{UserID: instructions.Transfer.To, EventSubType: "transfer_received", Data: ...},
    )
```

The Executor encrypts each event with the target user's P-521 public key. The sender sees `transfer_sent`; the recipient sees `transfer_received`. External observers see nothing — they only know a request was processed.

#### Withdrawal

Removes funds from the app and instructs the smart contract to release them on-chain.

```go
case "withdraw":
    if currentState.Accounts[senderHex] == nil {
        return ProcessResult{Error: fmt.Sprintf("Account %s does not exist", senderHex)}
    }
    if currentState.Accounts[senderHex].Balance.Cmp(*instructions.Withdraw.Amount) < 0 {
        return ProcessResult{Error: fmt.Sprintf("Insufficient balance %s for withdrawal %s",
            currentState.Accounts[senderHex].Balance, *instructions.Withdraw.Amount)}
    }

    currentState.Accounts[senderHex].Balance.Sub(
        *currentState.Accounts[senderHex].Balance, *instructions.Withdraw.Amount)
    currentState.Nonce++

    // Withdrawal instruction — processed by the smart contract
    withdrawals = append(withdrawals, Withdrawal{
        DestinationAddress: instructions.Withdraw.To,
        Amount:             instructions.Withdraw.Amount,
    })

    // Event for the sender
    events = append(events, PlainEvent{
        UserID: sender, EventSubType: "withdrawal", Data: ...,
    })
```

`Withdrawal` entries are included in the signed update payload. The smart contract verifies the TEE signature and credits the destination address via pull-payment.

#### Deanonymization (inside ProcessRequest)

When `requestType == common.Deanonymize`, the app generates a report and returns it in `ProcessResult.Report`. The state is **not modified** — the original state JSON is returned as-is to avoid unnecessary marshalling.

```go
case "deanonymize":
    report := DeanonymizationReport{
        Accounts: currentState.Accounts,
        Nonce:    currentState.Nonce,
    }
    reportBytes, err := json.Marshal(report)
    if err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("Failed to serialize deanonymization report: %v", err)}
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
    "amount": "0x6f05b59d3b20000",
    "invoice_id": "INV-2025-001"
  }
}
```
The `invoice_id` field is optional (max 100 characters). It is included in both sender and recipient events for payment tracking.

**Withdrawal:**
```json
{
  "type": "withdraw",
  "withdraw": {
    "to": "0x1234567890abcdef1234567890abcdef12345678",
    "amount": "0x6f05b59d3b20000"
  }
}
```

**Deanonymization:**
```json
{}
```
For deanonymization, the payload can be an empty JSON object. The `requestType` parameter (set by the Executor to `2`) determines the routing — the app checks `requestType == common.Deanonymize` and generates a report regardless of payload content.

---

## Step 6: Events

Events are the only way to communicate results back to users.
Each event targets a specific user and is encrypted with their registered P-521 key.

Every app must define its own event. <br>
Here are the payload formats your app defines:

### Event Types

| EventSubType | Recipient | Data Fields |
|--------------|-----------|-------------|
| `deposit` | Depositor | `type`, `amount`, `balance`, `nonce` |
| `transfer_sent` | Sender | `type`, `to`, `amount`, `balance`, `nonce`, `invoice_id` (optional) |
| `transfer_received` | Recipient | `type`, `from`, `amount`, `balance`, `nonce`, `invoice_id` (optional) |
| `withdrawal` | Withdrawer | `type`, `to`, `amount`, `balance`, `nonce` |

### Emitting an Event

```go
eventData, _ := json.Marshal(DepositEvent{
    Type:    "deposit",
    Amount:  value,
    Balance: currentState.Accounts[senderHex].Balance,
    Nonce:   currentState.Nonce,
})

events = append(events, PlainEvent{
    UserID:       *senderPtr,     // who receives this event
    EventSubType: "deposit",      // plaintext filter (visible on-chain)
    Data:         eventData,      // encrypted by the Executor
})
```

`EventSubType` is emitted in plaintext on-chain (for filtering/indexing). `Data` is encrypted — only the target user can read it. Design your sub-types to not leak sensitive information.

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

1. Download `payment_app.wasm` from https://github.com/HorizenOfficial/vela-nova/releases/tag/v0.0.25
2. Copy it into `dockerfiles/wasms/` and rename to `1.wasm`
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
    ./nova-linux deployapp 1
    ./nova-linux registeruser
    ./nova-linux getpublicbalance
    ./nova-linux deposit -a "1 ETH"
    ./nova-linux getprivatebalance
    ```

---

## Summary

Building an application on Vela CCE comes down to:

1. **Define your state** — a JSON-serializable struct that holds all application data.
2. **Import shared types** — use `Address`, `Uint256`, `PlainEvent`, `Withdrawal`, and result types from `vela-common-go/wasm/types`.
3. **Implement three exports** — `load_module` (init), `deposit` (fund accounts), `process_request` (transfers, withdrawals, and deanonymization via `requestType` routing).
4. **Memory management** — `allocate`/`deallocate` are provided by `vela-common-go/wasm/utils`.
5. **Return results** — updated state, events for users, withdrawals for on-chain transfers, fuel for fee calculation, and optional report for deanonymization.
6. **Handle errors** — return early with an `Error` string; never leave state partially modified.
7. **Compile with TinyGo** — `tinygo build -target=wasi .`

The Executor handles all cryptography (state encryption, event encryption, payload decryption, signing). Your app works with plaintext data and lets the platform handle the rest.

### Related Resources

| Resource | URL |
|----------|-----|
| Vela Nova (test app source) | https://github.com/HorizenOfficial/vela-nova/releases/tag/v0.0.25 |
| Vela Common TS (browser client) | https://github.com/HorizenOfficial/vela-common-ts |
| Local dev environment | `dockerfiles/` in this repository |
