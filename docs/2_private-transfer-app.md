# Building a Private Transfer App on Horizen CCE

This document walks through the implementation of a real WASM application for the Horizen CCE platform: a **private transfer app** that supports deposits, account-to-account transfers, withdrawals to external addresses, and regulatory deanonymization. It serves as a practical reference for developers building their own applications.
It shows some code snippets in GO language.

> **Prerequisite**: Read `1_summary.md` first for the full system architecture (smart contracts, Manager, Executor, cryptographic design). This document focuses on the guest application — the WASM module you write and deploy.

App code is available in the following repository:

https://github.com/HorizenOfficial/horizen-pes-nova <br>
*(this guide is based on 0.0.18 tag)*



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
    types.go           # Application-specific types (state, instructions, events, local type replacements)
    uint256.go         # Custom 256-bit unsigned integer implementation
    app.go             # Core business logic
  utils/
    utils.go           # Memory management (allocate/deallocate) and data translation
    logger.go          # Logging functions for TinyGo WASI
  Makefile             # Build targets (dev and production)
  go.mod               # Dependencies
  integration_test.go  # Tests against the WASM runtime
  system_tests/        # Full end-to-end tests (deploy, encrypt, blockchain)
```

The key insight: **`main.go` is a thin bridge** that converts raw WASM pointers into Go types and delegates to the `app` package. All business logic lives in `app/`. Memory management and WASM utilities live in `utils/`.

---

## Step 1: Set Up the Module

### go.mod

The app references main horizen-pes repo for some common code:

```
require (
    github.com/horizen-pes v0.0.18
)
```

The WASM guest module defines all its types locally — it does **not** import types from a common library. This is a deliberate design choice: the guest is a sandboxed TinyGo environment that cannot import host packages (e.g., `go-ethereum`). Communication between host and guest happens via JSON serialization, with both sides defining compatible types independently.

| Package | Provides |
|---------|----------|
| `app` (local) | Business logic, local type definitions (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result types), serialization helpers (`SerializeAndWriteResult`, `PtrToAddress`, `PtrToUint256`) |
| `utils` (local) | Memory management (`allocate`/`deallocate`, `StringToPtr`), pointer conversion (`PtrToString`), logging (`LogError/Warn/Info/Debug/Trace`) |


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
    AppID    int64                    `json:"appId"`
    Accounts map[string]*AccountState `json:"accounts"`
    Nonce    uint64                   `json:"nonce"`
}

// AccountState represents a single user account.
type AccountState struct {
    Address Address  `json:"address"`
    Balance *Uint256 `json:"balance"`
}
```

Design choices:
- **Accounts keyed by hex string** (e.g., `"0xadd...01"`). This avoids issues with binary map keys in JSON serialization.
- **Global nonce** incremented on every state-changing operation. Included in events so clients can verify ordering.
- **`Uint256`** for balances — a custom 256-bit unsigned integer (`[4]uint64`) that matches Ethereum's native precision. Supports `Add`, `Sub`, `Cmp`, `AddOverflow`, and `IsZero`. This is a local implementation because TinyGo does not support `math/big`.
- **`Address`** — a local `[20]byte` type, not `go-ethereum`'s `common.Address`. JSON serialization produces the same `"0x..."` format, ensuring compatibility with the host.

---

## Step 3: Implement the WASM Exports

Your module must export these functions: `load_module`, `deposit`, `process_request`, `generate_deanonymization_report`, plus `allocate`/`deallocate` (implemented in `utils/`).

### main.go — The Bridge Layer

```go
package main

import (
    "github.com/horizen-pes-nova/payment-app/app"
    "github.com/horizen-pes-nova/payment-app/utils"
)

//export load_module
func load_module(appId int64) *byte {
    result := app.LoadModule(appId)
    return app.SerializeAndWriteResult(result)
}

//export deposit
func deposit(appId int64, senderPtr *byte, senderLen int32,
             valuePtr *byte, valueLen int32,
             statePtr *byte, stateLen int32) *byte {
    sender := app.PtrToAddress(senderPtr, senderLen)
    value := app.PtrToUint256(valuePtr, valueLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.DepositFunds(sender, value, stateJSON)
    return app.SerializeAndWriteResult(result)
}

//export process_request
func process_request(appId int64, senderPtr *byte, senderLen int32,
                     payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    sender := app.PtrToAddress(senderPtr, senderLen)
    payloadJSON := utils.PtrToString(payloadPtr, payloadLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.ProcessRequest(sender, payloadJSON, stateJSON)
    return app.SerializeAndWriteResult(result)
}

//export generate_deanonymization_report
func generate_deanonymization_report(payloadPtr *byte, payloadLen int32,
                                     statePtr *byte, stateLen int32) *byte {
    payloadJSON := utils.PtrToString(payloadPtr, payloadLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.GenerateDeanonymizationReport(payloadJSON, stateJSON)
    return app.SerializeAndWriteResult(result)
}

func main() {} // Required by Go, not used in WASM
```

Key patterns:
- Use `app.PtrToAddress`, `app.PtrToUint256`, `utils.PtrToString` to convert WASM pointers into Go types.
- Always return via `app.SerializeAndWriteResult(result)` — this serializes your result to JSON and writes it into WASM linear memory for the host to read (4-byte little-endian length prefix + data).
- The `//export` directive is required for TinyGo to expose the function to the host.
- **`process_request` does not receive a `requestType` parameter** — the Executor routes requests by type and calls the appropriate export directly. Only `PROCESS` requests go through `process_request`.
- **`generate_deanonymization_report` is a separate export** with no `appId` or `sender` parameters.

---

## Step 4: Implement Business Logic

### LoadModule — Initialization

Called once when the application is deployed on-chain. Returns the initial empty state.

```go
func LoadModule(appId int64) LoadModuleResult {
    initialState := &ApplicationInternalState{
        AppID:    appId,
        Accounts: make(map[string]*AccountState),
    }
    stateJSON, err := json.Marshal(initialState)
    if err != nil {
        return LoadModuleResult{
            Error: fmt.Sprintf("failed to marshal initial state: %v", err),
        }
    }
    return LoadModuleResult{
        State: stateJSON,
        Fuel:  NewUint256(5),
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

### ProcessRequest — Transfers and Withdrawals

This is the main entry point for `PROCESS` requests. The operation type is determined by the payload's `type` field. Deanonymization is handled by a separate export (`generate_deanonymization_report`).

```go
func ProcessRequest(senderPtr *Address, payloadJSON, stateJSON string) ProcessResult {
    if senderPtr == nil {
        return ProcessResult{Error: "Sender address is missing"}
    }

    sender := *senderPtr
    senderHex := sender.Hex()

    // Parse state
    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return ProcessResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    var events []PlainEvent
    var withdrawals []Withdrawal

    if payloadJSON != "" {
        var instructions PayloadInstructions
        if err := json.Unmarshal([]byte(payloadJSON), &instructions); err != nil {
            return ProcessResult{Error: fmt.Sprintf("Failed to parse payload instructions: %v", err)}
        }

        switch instructions.Type {
        case "transfer":  // ...
        case "withdraw":  // ...
        default:
            return ProcessResult{Error: fmt.Sprintf("Unsupported instruction type: [%s]", instructions.Type)}
        }
    }

    newStateBytes, _ := json.Marshal(currentState)
    return ProcessResult{
        State: newStateBytes, Events: events, Withdrawals: withdrawals, Fuel: NewUint256(50),
    }
}
```

#### Transfer

Moves funds between two accounts within the app. Both sender and recipient receive encrypted events.

```go
case "transfer":
    // Validate sender has sufficient balance
    if currentState.Accounts[senderHex] == nil {
        return ProcessResult{Error: fmt.Sprintf("Account %s does not exist!", senderHex)}
    }
    if currentState.Accounts[senderHex].Balance.Cmp(*instructions.Transfer.Amount) < 0 {
        return ProcessResult{Error: "Insufficient balance for transfer"}
    }

    recipientHex := instructions.Transfer.To.Hex()

    // Create recipient account if needed
    if currentState.Accounts[recipientHex] == nil {
        currentState.Accounts[recipientHex] = &AccountState{
            Address: instructions.Transfer.To,
            Balance: NewUint256(0),
        }
    }

    // Execute transfer
    currentState.Accounts[senderHex].Balance.Sub(
        *currentState.Accounts[senderHex].Balance, *instructions.Transfer.Amount)
    currentState.Accounts[recipientHex].Balance.Add(
        *currentState.Accounts[recipientHex].Balance, *instructions.Transfer.Amount)
    currentState.Nonce++

    // Two events: one for sender, one for recipient
    events = append(events,
        PlainEvent{UserID: sender, EventSubType: "transfer_sent", Data: ...},
        PlainEvent{UserID: instructions.Transfer.To, EventSubType: "transfer_received", Data: ...},
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

### GenerateDeanonymizationReport — Separate Export

This is a dedicated WASM export, **not** handled through `process_request`. The Executor calls it directly when the request type is `DEANONYMIZATION`. It does **not** modify state and returns a `DeanonymizationResult` (not a `ProcessResult`).

```go
func GenerateDeanonymizationReport(payloadJSON, stateJSON string) DeanonymizationResult {
    var payload ReportPayloadInstructions
    if err := json.Unmarshal([]byte(payloadJSON), &payload); err != nil {
        return DeanonymizationResult{Error: fmt.Sprintf("Failed to parse payload: %s, err: %v",
            payloadJSON, err)}
    }

    var currentState ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &currentState); err != nil {
        return DeanonymizationResult{Error: fmt.Sprintf("Failed to parse application state: %v", err)}
    }

    report := UnencryptedDeanonymizationReportData{
        Accounts: currentState.Accounts,
        Nonce:    currentState.Nonce,
    }
    reportBytes, _ := json.Marshal(report)

    return DeanonymizationResult{Report: reportBytes, Fuel: NewUint256(20)}
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

**Deanonymization (passed to `generate_deanonymization_report`):**
```json
{}
```
For deanonymization, the payload is an empty JSON object. The Executor calls the dedicated `generate_deanonymization_report()` export directly — the request type routing is handled by the Executor, not the payload.

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

events = append(events, PlainEvent{
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
| `ProcessRequest` (transfer or withdraw) | 50 |
| `GenerateDeanonymizationReport` | 20 |

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
| **Memory** | `allocate`/`deallocate` must be implemented in your `utils` package — they are auto-exported by TinyGo |
| **Local types** | Define your own `Address` (`[20]byte`), `Uint256` (`[4]uint64`), etc. — do not import from the host |

---

## Summary

Building an application on Horizen CCE comes down to:

1. **Define your state** — a JSON-serializable struct that holds all application data.
2. **Define local types** — `Address`, `Uint256`, `PlainEvent`, `Withdrawal`, and result types compatible with the host's JSON format.
3. **Implement four exports** — `load_module` (init), `deposit` (fund accounts), `process_request` (transfers, withdrawals), `generate_deanonymization_report` (auditing).
4. **Implement memory management** — `allocate`/`deallocate` in a `utils` package for WASM memory exchange.
5. **Return results** — updated state, events for users, withdrawals for on-chain transfers, and fuel for fee calculation.
6. **Handle errors** — return early with an `Error` string; never leave state partially modified.
7. **Compile with TinyGo** — `tinygo build -target=wasi .`

The Executor handles all cryptography (state encryption, event encryption, payload decryption, signing). Your app works with plaintext data and lets the platform handle the rest.
