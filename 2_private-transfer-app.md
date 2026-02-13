# Building a Private Transfer App on Horizen CCE

This document walks through the implementation of a real WASM application for the Horizen CCE platform: a **private transfer app** that supports deposits, account-to-account transfers, withdrawals to external addresses, and regulatory deanonymization. It serves as a practical reference for developers building their own applications.
It shows some code snippets in GO language.

> **Prerequisite**: Read `1_summary.md` first for the full system architecture (smart contracts, Manager, Executor, cryptographic design). This document focuses on the guest application — the WASM module you write and deploy.

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

## Suggested Project Structure

```
runtime/wasm-go/
  main.go              # WASM exports (bridge layer)
  app/
    types.go           # Application-specific types (state, instructions, events)
    app.go             # Core business logic
  Makefile             # Build targets (dev and production)
  go.mod               # Dependencies
  integration_test.go  # Tests against the WASM runtime
  system_tests/        # Full end-to-end tests (deploy, encrypt, blockchain)
```

The key insight: **`main.go` is a thin bridge** that converts raw WASM pointers into Go types and delegates to the `app` package. All business logic lives in `app/`.

---

## Step 1: Set Up the Module

### go.mod

```go
module github.com/horizen-pes-nova/payment-app

go 1.24.0

require (
    github.com/horizen-cce-common-go v0.0.20  // shared types and utilities
)
```

Through horizen-cce-common-go you will have accesso to some common libraries:

| Package | Used By | Provides |
|---------|---------|----------|
| `github.com/horizen-cce-common-go/wasm/types` | WASM guest code | `Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result types, serialization |
| `github.com/horizen-cce-common-go/wasm/utils` | WASM guest code | Pointer conversion (`PtrToString`), logging, memory management (`allocate`/`deallocate` auto-exported) |


### Makefile

```makefile
# Development build (with debug symbols)
tinygo build -o build/payment_app.wasm -target=wasi .

# Production build (optimized, no debug symbols)
tinygo build -o build/payment_app.wasm -opt=s -no-debug -target=wasi .
```

The target is always `wasi`. TinyGo compiles your Go code to a WebAssembly module that the Executor loads and runs inside the enclave.

---

## Step 2: Design Your State

All application state is serialized to JSON, encrypted by the Executor (AES-256), and stored between requests. You receive it as a string, deserialize it, modify it, and return the updated version.

### State Schema

```go
// ApplicationInternalState is the root state container.
type ApplicationInternalState struct {
    AppID    int64                    `json:"appId"`
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
- **`types.Uint256`** for balances — a 256-bit unsigned integer that matches Ethereum's native precision. Supports `Add`, `Sub`, `Mul64`, `Cmp`, and overflow checks.

---

## Step 3: Implement the WASM Exports

Your module must export four functions. The common library auto-exports `allocate` and `deallocate` when imported — you only implement the application-specific ones.

### main.go — The Bridge Layer

```go
package main

import (
    "github.com/horizen-cce-common-go/wasm/types"
    "github.com/horizen-cce-common-go/wasm/utils"
    "github.com/horizen-pes-nova/payment-app/app"
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
    sender := types.PtrToAddress(senderPtr, senderLen)
    value := types.PtrToUint256(valuePtr, valueLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.DepositFunds(sender, value, stateJSON)
    return types.SerializeAndWriteResult(result)
}

//export process_request
func process_request(appId int64, senderPtr *byte, senderLen int32,
                     requestType int32,
                     payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    sender := types.PtrToAddress(senderPtr, senderLen)
    payloadJSON := utils.PtrToString(payloadPtr, payloadLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.ProcessRequest(sender, requestType, payloadJSON, stateJSON)
    return types.SerializeAndWriteResult(result)
}

func main() {} // Required by Go, not used in WASM
```

Key patterns:
- Use `types.PtrToAddress`, `types.PtrToUint256`, `utils.PtrToString` to convert WASM pointers into Go types.
- Always return via `types.SerializeAndWriteResult(result)` — this serializes your result to JSON and writes it into WASM linear memory for the host to read.
- The `//export` directive is required for TinyGo to expose the function to the host.

---

## Step 4: Implement Business Logic

### LoadModule — Initialization

Called once when the application is deployed on-chain. Returns the initial empty state.

```go
func LoadModule(appId int64) types.LoadModuleResult {
    initialState := &ApplicationInternalState{
        AppID:    appId,
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
func DepositFunds(senderPtr *types.Address, value *types.Uint256, stateJSON string) types.DepositResult {
    // 1. Validate inputs
    if senderPtr == nil {
        return types.DepositResult{Error: "Sender address is nil"}
    }

    // 2. Deserialize state
    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return types.DepositResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    var events []types.PlainEvent

    // 3. Process deposit (skip zero-value deposits)
    if !value.IsZero() {
        senderHex := senderPtr.Hex()

        // Create account if it doesn't exist
        if currentState.Accounts[senderHex] == nil {
            currentState.Accounts[senderHex] = &AccountState{
                Address: *senderPtr,
                Balance: types.NewUint256(0),
            }
        }

        // Add to balance with overflow check
        oldBalance := *currentState.Accounts[senderHex].Balance
        if currentState.Accounts[senderHex].Balance.AddOverflow(
            *currentState.Accounts[senderHex].Balance, *value) {
            *currentState.Accounts[senderHex].Balance = oldBalance
            return types.DepositResult{Error: "Overflow"}
        }

        currentState.Nonce++

        // 4. Emit event to the depositor
        eventData, _ := json.Marshal(DepositEvent{
            Type: "deposit", Amount: value,
            Balance: currentState.Accounts[senderHex].Balance,
            Nonce: currentState.Nonce,
        })
        events = append(events, types.PlainEvent{
            UserID:       *senderPtr,
            EventSubType: "deposit",
            Data:         eventData,
        })
    }

    // 5. Return updated state
    newStateBytes, _ := json.Marshal(&currentState)
    return types.DepositResult{
        State:  newStateBytes,
        Events: events,
        Fuel:   types.NewUint256(35),
    }
}
```

### ProcessRequest — Transfers, Withdrawals, Deanonymization

This is the main entry point. The `requestType` parameter (set by the on-chain request) takes priority over the payload for determining the operation.

```go
func ProcessRequest(senderPtr *types.Address, requestType int32,
                    payloadJSON, stateJSON string) types.ProcessResult {
    // Parse state and payload...

    // Determine instruction type
    var instructionType string
    if requestType == int32(common.Deanonymize) {
        instructionType = "deanonymize"   // requestType wins
    } else {
        instructionType = instructions.Type  // from payload JSON
    }

    switch instructionType {
    case "transfer":  // ...
    case "withdraw":  // ...
    case "deanonymize":  // ...
    }
}
```

#### Transfer

Moves funds between two accounts within the app. Both sender and recipient receive encrypted events.

```go
case "transfer":
    // Validate sender has sufficient balance
    if currentState.Accounts[senderHex].Balance.Cmp(*instructions.Transfer.Amount) < 0 {
        return types.ProcessResult{Error: "Insufficient balance for transfer"}
    }

    // Create recipient account if needed
    if currentState.Accounts[recipientHex] == nil {
        currentState.Accounts[recipientHex] = &AccountState{
            Address: instructions.Transfer.To,
            Balance: types.NewUint256(0),
        }
    }

    // Execute transfer (with overflow protection)
    currentState.Accounts[senderHex].Balance.Sub(...)
    currentState.Accounts[recipientHex].Balance.AddOverflow(...)
    currentState.Nonce++

    // Two events: one for sender, one for recipient
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
    currentState.Accounts[senderHex].Balance.Sub(...)
    currentState.Nonce++

    // Withdrawal instruction — processed by the smart contract
    withdrawals = append(withdrawals, types.Withdrawal{
        DestinationAddress: instructions.Withdraw.To,
        Amount:             instructions.Withdraw.Amount,
    })

    // Event for the sender
    events = append(events, types.PlainEvent{
        UserID: sender, EventSubType: "withdrawal", Data: ...,
    })
```

`types.Withdrawal` entries are included in the signed update payload. The smart contract verifies the TEE signature and credits the destination address via pull-payment.

#### Deanonymization

Returns a full state snapshot as an encrypted report. Does **not** modify state.

```go
case "deanonymize":
    report := DeanonymizationReport{
        Accounts: currentState.Accounts,
        Nonce:    currentState.Nonce,
    }
    reportBytes, _ := json.Marshal(report)

    return types.ProcessResult{
        State:  []byte(stateJSON),  // unchanged — skip re-marshaling
        Report: reportBytes,
        Fuel:   types.NewUint256(20),
    }
```

The Executor encrypts the report with the requesting authority's P-521 public key. Only authorized auditors (registered in the `AuthorityRegistry` contract) can submit deanonymization requests.

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
    "amount": "0x6f05b59d3b20000"
  }
}
```

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
For deanonymization, the payload can be empty — the `requestType` parameter (set to `DEANONYMIZATION` on-chain) is what triggers it.

---

## Step 6: Events

Events are the only way to communicate results back to users. 
Each event targets a specific user and is encrypted with their registered P-521 key.

### Event Types

| EventSubType | Recipient | Data Fields |
|--------------|-----------|-------------|
| `deposit` | Depositor | `type`, `amount`, `balance`, `nonce` |
| `transfer_sent` | Sender | `type`, `to`, `amount`, `balance`, `nonce` |
| `transfer_received` | Recipient | `type`, `from`, `amount`, `balance`, `nonce` |
| `withdrawal` | Withdrawer | `type`, `to`, `amount`, `balance`, `nonce` |

### Emitting an Event

```go
eventData, _ := json.Marshal(DepositEvent{
    Type:    "deposit",
    Amount:  value,
    Balance: currentState.Accounts[senderHex].Balance,
    Nonce:   currentState.Nonce,
})

events = append(events, types.PlainEvent{
    UserID:       *senderPtr,     // who receives this event
    EventSubType: "deposit",      // plaintext filter (visible on-chain)
    Data:         eventData,      // encrypted by the Executor
})
```

`EventSubType` is emitted in plaintext on-chain (for filtering/indexing). `Data` is encrypted — only the target user can read it. Design your sub-types to not leak sensitive information.

---

## Step 7: Fuel

Every operation must return a `Fuel` value representing computation cost. The fee charged to the user is `fuel * weiPerFuelUnit` (configured on-chain). Any remaining deposit is refunded.

| Operation | Fuel |
|-----------|------|
| `LoadModule` | 5 |
| `Deposit` | 35 |
| `Transfer` | 50 |
| `Withdrawal` | 50 |
| `Deanonymize` | 20 |

Fuel values are application-defined. Set them proportional to the computational cost of each operation.

---

## Step 8: Error Handling

Return errors via the `Error` field of the result struct. When an error is returned, the Executor marks the request as failed on-chain and refunds the user.

```go
// Input validation
if senderPtr == nil {
    return types.ProcessResult{Error: "Sender address is missing"}
}

// Business rule validation
if currentState.Accounts[senderHex].Balance.Cmp(*amount) < 0 {
    return types.ProcessResult{Error: "Insufficient balance for transfer"}
}

// Overflow protection
oldBalance := *account.Balance
if account.Balance.AddOverflow(*account.Balance, *value) {
    *account.Balance = oldBalance  // revert
    return types.ProcessResult{Error: "Overflow"}
}
```

Important: when returning an error, **do not modify state**. Either return early before mutations, or revert changes before returning.

---

## TinyGo Constraints

Since your code compiles with TinyGo (not standard Go), keep these limitations in mind:

| Constraint | Detail |
|------------|--------|
| **No `reflect`-heavy code** | Avoid packages that rely on deep reflection |
| **Limited stdlib** | `net`, `os` (beyond basics), and some other packages are unavailable |
| **Determinism required** | No `time.Now()`, no random numbers — same inputs must produce same outputs |
| **Stateless execution** | No global mutable state between invocations; everything goes through the state parameter |
| **JSON serialization** | `encoding/json` works. Keep state structures simple |
| **Logging** | Use `utils.LogError/Warn/Info/Debug/Trace` — captured via WASI pipes, rate-limited to 1000 logs/sec |
| **Memory** | `allocate`/`deallocate` are auto-exported by importing the common library — do not implement them yourself |

---

## Summary

Building an application on Horizen CCE comes down to:

1. **Define your state** — a JSON-serializable struct that holds all application data.
2. **Implement three exports** — `load_module` (init), `deposit` (fund accounts), `process_request` (everything else).
3. **Return results** — updated state, events for users, withdrawals for on-chain transfers, and fuel for fee calculation.
4. **Handle errors** — return early with an `Error` string; never leave state partially modified.
5. **Compile with TinyGo** — `tinygo build -target=wasi .`

The Executor handles all cryptography (state encryption, event encryption, payload decryption, signing). Your app works with plaintext data and lets the platform handle the rest.
