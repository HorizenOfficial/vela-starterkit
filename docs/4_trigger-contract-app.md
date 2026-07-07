# Building a Trigger-Contract App on Vela (v0.2.0)

This document walks through the implementation of a WASM application that drives an **external trigger contract** — the `TRUSTPROCESS` / `trusted_request` flow. The example is an **anonymous execution pool** ("mixer"): users deposit funds into a shared on-chain pool, then ask the app to perform an arbitrary on-chain call on their behalf. The call is executed by a trigger contract from the *pool's* address (not the user's), and whatever is left unspent is credited back to the user's private balance inside the TEE.
It shows code snippets in the GO language.

> **Prerequisite**: Read `1_summary.md` first — especially [§3.3 Process Request with Trigger Contract](1_summary.md#33-process-request-with-trigger-contract), which describes the on-chain trigger callbacks (`execute` → `withdraw` → `getTrustProcessPayload`) and the separate high-priority trigger queue. This guide focuses on the guest application and its companion trigger contract. If you have not built a plain Vela app yet, read `2_private-transfer-app.md` first — this guide assumes the basics (state, exports, events, fuel) and only details what a trigger app adds.

---

## What This App Does

The app implements a shared custody pool inside a TEE. Users fund the pool, then submit an *execute* request describing an on-chain call. The app hands the call to its trigger contract, which performs it from the pool; the unspent remainder round-trips back through `trusted_request` and is re-credited to the user privately.

| Operation | Export | Description |
|-----------|--------|-------------|
| **Deposit** | `deposit` | Fund your private balance from an on-chain transaction. |
| **Execute** | `process_request` | Lock funds, instruct the trigger contract to run an on-chain call from the pool, and emit the `AppEvent` that fires the trigger. |
| **Reconcile** | `trusted_request` | (Automatic, `TRUSTPROCESS`.) Credit the unspent leftover back to the user's private balance after the trigger has run. |

The **privacy property**: an observer sees a public deposit *into* the pool and a call *from* the pool, but the link between them lives only inside the TEE ledger — there is no ZK proof, the trust anchor is the enclave attestation plus the on-chain trigger's attested result.

---

## How the Trigger Round-Trip Works

A trigger app differs from a normal app in one structural way: a user request does not finish when `process_request` returns. It kicks off a second, platform-driven request.

```
1. User → process_request({"command":"execute", ...})
      app locks the funds, returns:
        - a Withdrawal to the trigger contract address
        - an AppEvent carrying ABI-encoded call parameters
2. (on-chain, inside stateUpdate) ProcessorEndpoint:
        - claims the withdrawal into the trigger
        - trigger.execute(appEventData)      → performs the on-chain call from the pool
        - trigger.withdraw()                 → sweeps the leftover back to the endpoint
        - trigger.getTrustProcessPayload(...) → returns ABI-encoded (lockId, remain, outcome)
        - if that payload is non-empty → enqueues a TRUSTPROCESS request
3. (automatic) Manager picks up the TRUSTPROCESS (trigger queue has priority) →
      Executor → trusted_request(payload, state)
        app decodes (lockId, remain, outcome), credits `remain` back to the owner,
        emits NO AppEvents → the trigger returns "" → the loop terminates.
```

The two rules that keep this safe and finite:
- **The trigger only enqueues a `TRUSTPROCESS` when `getTrustProcessPayload` returns non-empty bytes.**
- **A `TRUSTPROCESS` carries no `AppEvent`s.** The trigger guards on `appEventData.events.length == 0` and returns `""`, so a trusted request can never spawn another one. This is the loop terminator — every trigger you write must honor it.

---

## Project Structure

```
trigger-app/
  main.go              # WASM exports (bridge layer) — includes trusted_request
  app/
    types.go           # Application state, deploy params, event bodies
    app.go             # Core business logic (Deploy, Deposit, ProcessRequest, TrustedRequest)
    abi.go             # Minimal ABI encode/decode for the trigger wire formats
  contracts/
    MixerTrigger.sol   # The companion trigger contract (extends AbstractTrigger)
  Makefile             # TinyGo build targets
  go.mod               # Dependencies (vela-common-go)
```

As with any Vela app, **`main.go` is a thin bridge** that converts raw WASM pointers into Go types and delegates to the `app` package. The only new export is `trusted_request`.

---

## Step 1: Set Up the Module

### go.mod

A standalone trigger app needs only the shared WASM library. (The in-tree `app/trigger` reference lives inside the platform module `github.com/HorizenOfficial/vela`; a standalone app declares its own module.)

```
module github.com/yourorg/vela-mixer

go 1.24

require github.com/HorizenOfficial/vela-common-go v0.2.0
```

| Package | Provides |
|---------|----------|
| `vela-common-go/wasm/types` | Core types (`Address`, `Uint256`, `PlainEvent`, `AppEvent`, `Withdrawal`), result types (`DeployResult`, `LoadModuleResult`, `DepositResult`, `ProcessResult`), helpers (`SerializeAndWriteResult`, `PtrToAddress`, `PtrToUint256`, `HexToAddress`, `NewUint256`) |
| `vela-common-go/wasm/utils` | Memory management (`allocate`/`deallocate`, `BytesToPtr`, `get_allocated_memory_stats`), `PtrToString`, logging (`LogError/Warn/Info/Debug/Trace`) |

### Makefile

```makefile
APP := mixer_app

build:
	tinygo build -o build/$(APP).wasm -target=wasi .

production_build:
	tinygo build -o production_build/$(APP).wasm -opt=s -no-debug -target=wasi .
```

The target is always `wasi`. The trigger app compiles exactly like any other Vela WASM app — exporting `trusted_request` requires nothing special in the build.

---

## Step 2: Design Your State

The state must remember, for each in-flight execution, **who owns the locked funds**, so that `trusted_request` can credit the leftover back to the right account. This is the core piece a trigger app adds on top of a normal ledger: a **lock table**.

`app/types.go`:

```go
package app

import "github.com/HorizenOfficial/vela-common-go/wasm/types"

// AccountState holds one account's per-token balances (ETH only in this tutorial).
type AccountState struct {
    Address  types.Address             `json:"address"`
    Balances map[string]*types.Uint256 `json:"balances"` // token hex -> balance
}

// Lock records funds removed from an account and handed to the trigger for an
// in-flight execution. trusted_request looks the lock up by ID to know whom to
// credit the unspent leftover back to.
type Lock struct {
    Owner  types.Address  `json:"owner"`
    Amount *types.Uint256 `json:"amount"` // ETH locked for this execution
}

// ApplicationInternalState is the persisted app state.
type ApplicationInternalState struct {
    AppID          uint64                   `json:"appId"`
    Accounts       map[string]*AccountState `json:"accounts"`       // account hex -> balances
    Locks          map[string]*Lock         `json:"locks"`          // lockId hex -> lock
    LockNonce      uint64                   `json:"lockNonce"`      // monotonic; derives deterministic lockIds
    TriggerAddress string                   `json:"triggerAddress"` // registered trigger contract (hex)
}

// DeployParams carries the constructor parameters. The trigger contract address
// MUST be provided here as well as on-chain (see "Deploying with a Trigger").
type DeployParams struct {
    TriggerContract string `json:"triggerContract"`
}

// --- event bodies (JSON, encrypted per-user by the Executor) ---

type depositConfirmation struct {
    Type       string         `json:"type"` // "deposit"
    BalanceAfter *types.Uint256 `json:"balanceAfter"`
}

type executionOutcome struct {
    Type    string         `json:"type"`    // "execution_outcome"
    LockID  string         `json:"lockId"`  // hex of the 16-byte lock id
    Outcome string         `json:"outcome"` // "success" | "failure"
    Remain  *types.Uint256 `json:"remain"`  // unspent amount credited back
    Balance *types.Uint256 `json:"balance"` // owner's new balance
}
```

Design notes:
- **Lock IDs are deterministic**, derived from `LockNonce` (an incrementing counter), *not* random — TinyGo forbids non-determinism (see constraints). A `bytes16` lock id is the counter in the low 8 bytes.
- **`TriggerAddress`** is seeded from the deploy params. `process_request` stamps it into every `Withdrawal.DestinationAddress`, so the platform routes locked funds to the trigger.
- Balances use `types.Uint256` (256-bit, `[4]uint64`) to match Ethereum precision — `math/big` is unavailable in TinyGo.

---

## Step 3: Implement the WASM Exports

The only difference from a standard app is the extra `trusted_request` export. Note its signature — **no `sender`, no `requestType`** — the trusted path never reads a sender and the request type is implicit.

`main.go`:

```go
package main

import (
    "github.com/HorizenOfficial/vela-common-go/wasm/types"
    "github.com/HorizenOfficial/vela-common-go/wasm/utils"
    "github.com/yourorg/vela-mixer/app"
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
    value := types.PtrToUint256(valuePtr, valueLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
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

//export trusted_request
func trusted_request(appId int64, payloadPtr *byte, payloadLen int32,
                     statePtr *byte, stateLen int32) *byte {
    _ = appId
    payload := utils.PtrToString(payloadPtr, payloadLen)
    stateJSON := utils.PtrToString(statePtr, stateLen)
    result := app.TrustedRequest(payload, stateJSON)
    return types.SerializeAndWriteResult(result)
}

//export get_memory_stats
func get_memory_stats() *byte {
    result := utils.GetAllocatedMemoryStats()
    return types.SerializeAndWriteResult(result)
}

func main() {} // Required by Go, not used in WASM
```

Key points:
- **`trusted_request` receives only `(appId, payload, state)`** — the payload is **clear text** (the Executor does *not* ECDH-decrypt trusted payloads) and is produced by the trigger contract, not by a user.
- `allocate`/`deallocate` are provided by `vela-common-go/wasm/utils` and auto-exported by TinyGo.
- If a `TRUSTPROCESS` request reaches an app that does not export `trusted_request`, the Executor fails it with `trusted_request function not found` — so exporting it is what opts your app into the trigger flow.

---

## Step 4: Implement Business Logic

`app/app.go`. `ethTokenHex` is the hex of the zero address (native ETH sentinel).

```go
package app

import (
    "encoding/hex"
    "encoding/json"
    "fmt"

    "github.com/HorizenOfficial/vela-common-go/wasm/types"
)

var ethTokenHex = (types.Address{}).Hex()

const (
    subtypeExecuteRequested = "execute_requested"
    subtypeDeposit          = "deposit"
    subtypeExecutionOutcome = "execution_outcome"
)
```

### Deploy — Store the Trigger Address

`deploy` records the trigger contract address into state. The app will stamp it into withdrawals so the platform knows where to send locked funds.

```go
func Deploy(appId int64, paramsJSON string) types.DeployResult {
    var trigger string
    if paramsJSON != "" {
        var params DeployParams
        if err := json.Unmarshal([]byte(paramsJSON), &params); err != nil {
            return types.DeployResult{Error: fmt.Sprintf("failed to parse deploy params: %v", err)}
        }
        if params.TriggerContract != "" {
            addr, err := types.HexToAddress(params.TriggerContract)
            if err != nil {
                return types.DeployResult{Error: fmt.Sprintf("invalid trigger address %q: %v", params.TriggerContract, err)}
            }
            trigger = addr.Hex()
        }
    }

    stateJSON, _ := json.Marshal(newInitialState(appId, trigger))
    return types.DeployResult{State: stateJSON, Fuel: types.NewUint256(5)}
}

func newInitialState(appId int64, trigger string) *ApplicationInternalState {
    return &ApplicationInternalState{
        AppID:          uint64(appId),
        Accounts:       make(map[string]*AccountState),
        Locks:          make(map[string]*Lock),
        TriggerAddress: trigger,
    }
}
```

### LoadModule — Cache Warm-Up Fallback

Same minimal state, with no trigger configured. The Executor calls this only to rebuild its default-state cache on restart; apps never call it themselves.

```go
func LoadModule(appId int64) types.LoadModuleResult {
    stateJSON, _ := json.Marshal(newInitialState(appId, ""))
    return types.LoadModuleResult{State: stateJSON, Fuel: types.NewUint256(5)}
}
```

### Deposit — Funding the Pool

Standard credit of the sender's private balance (ETH only in this example).

```go
func DepositFunds(senderPtr, tokenPtr *types.Address, value *types.Uint256, stateJSON string) types.DepositResult {
    if senderPtr == nil || tokenPtr == nil || value == nil {
        return types.DepositResult{Error: "deposit: nil argument"}
    }
    if tokenPtr.Hex() != ethTokenHex {
        return types.DepositResult{Error: "deposit: only ETH is supported"}
    }

    var state ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &state); err != nil {
        return types.DepositResult{Error: fmt.Sprintf("failed to parse state: %v", err)}
    }

    var events []types.PlainEvent
    if !value.IsZero() {
        acc := state.Accounts[senderPtr.Hex()]
        if acc == nil {
            acc = &AccountState{Address: *senderPtr, Balances: make(map[string]*types.Uint256)}
            state.Accounts[senderPtr.Hex()] = acc
        }
        bal := acc.Balances[ethTokenHex]
        if bal == nil {
            bal = types.NewUint256(0)
            acc.Balances[ethTokenHex] = bal
        }
        old := *bal
        if bal.AddOverflow(*bal, *value) {
            *bal = old
            return types.DepositResult{Error: "deposit: balance overflow"}
        }

        balCopy := *bal
        data, _ := json.Marshal(depositConfirmation{Type: subtypeDeposit, BalanceAfter: &balCopy})
        events = append(events, types.PlainEvent{UserID: *senderPtr, Data: data})
    }

    newState, _ := json.Marshal(&state)
    return types.DepositResult{State: newState, Events: events, Fuel: types.NewUint256(35)}
}
```

### ProcessRequest — Lock Funds and Fire the Trigger

This is the heart of a trigger app. On an `execute` command the app:
1. checks the sender's balance,
2. deducts and records a **lock** (so the leftover can later be returned),
3. emits a **`Withdrawal`** to the trigger contract (the platform moves the locked funds into the trigger), and
4. emits an **`AppEvent`** whose `Data` is the ABI-encoded call parameters — this is what fires the trigger.

```go
// ExecuteInstruction is the decrypted process_request payload for command "execute".
type PayloadInstructions struct {
    Command string           `json:"command"`
    Execute *ExecuteRequest  `json:"execute,omitempty"`
}

type ExecuteRequest struct {
    Target types.Address  `json:"target"` // contract/EOA to call from the pool
    Value  *types.Uint256 `json:"value"`  // ETH to forward with the call
    Data   string         `json:"data"`   // 0x-prefixed calldata for the call
}

func ProcessRequest(senderPtr *types.Address, requestType int32, payloadJSON, stateJSON string) types.ProcessResult {
    _ = requestType // this app has no deanonymization path
    if senderPtr == nil {
        return types.ProcessResult{Error: "process: missing sender"}
    }

    var state ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &state); err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("failed to parse state: %v", err)}
    }

    var instr PayloadInstructions
    if payloadJSON != "" && payloadJSON != "{}" {
        if err := json.Unmarshal([]byte(payloadJSON), &instr); err != nil {
            return types.ProcessResult{Error: fmt.Sprintf("failed to parse payload: %v", err)}
        }
    }

    switch instr.Command {
    case "execute":
        return handleExecute(*senderPtr, instr.Execute, &state)
    default:
        return types.ProcessResult{Error: fmt.Sprintf("unsupported command: [%s]", instr.Command)}
    }
}

func handleExecute(sender types.Address, req *ExecuteRequest, state *ApplicationInternalState) types.ProcessResult {
    if req == nil || req.Value == nil {
        return types.ProcessResult{Error: "execute: missing parameters"}
    }
    if state.TriggerAddress == "" {
        return types.ProcessResult{Error: "execute: no trigger contract configured"}
    }
    triggerAddr, err := types.HexToAddress(state.TriggerAddress)
    if err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("execute: bad trigger address: %v", err)}
    }

    // 1. Check and debit the sender's balance.
    acc := state.Accounts[sender.Hex()]
    if acc == nil || acc.Balances[ethTokenHex] == nil {
        return types.ProcessResult{Error: "execute: no balance"}
    }
    bal := acc.Balances[ethTokenHex]
    if bal.Cmp(*req.Value) < 0 {
        return types.ProcessResult{Error: "execute: insufficient balance"}
    }
    bal.Sub(*bal, *req.Value)

    // 2. Record a lock so trusted_request can credit the leftover back.
    state.LockNonce++
    lockID := lockIDFromNonce(state.LockNonce)
    lockKey := hex.EncodeToString(lockID[:])
    amt := *req.Value
    state.Locks[lockKey] = &Lock{Owner: sender, Amount: &amt}

    // 3. Decode calldata (0x-hex) for the on-chain call.
    callData, err := decodeHex(req.Data)
    if err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("execute: bad calldata: %v", err)}
    }

    // 4. Withdrawal: move the locked ETH into the trigger contract.
    withdrawals := []types.Withdrawal{{
        TokenAddress:       types.Address{}, // ETH
        DestinationAddress: triggerAddr,
        Amount:             req.Value,
    }}

    // 5. AppEvent: ABI-encoded (bytes16 lockId, address target, uint256 value, bytes data).
    //    The trigger contract abi.decode()s this to perform the call.
    appEvents := []types.AppEvent{{
        EventSubType: subtypeToBytes32(subtypeExecuteRequested),
        Data:         abiEncodeExecuteRequested(lockID, req.Target, req.Value, callData),
    }}

    newState, _ := json.Marshal(state)
    return types.ProcessResult{
        State:       newState,
        AppEvents:   appEvents,
        Withdrawals: withdrawals,
        Fuel:        types.NewUint256(60),
    }
}
```

> **Authentication note.** This example trusts the on-chain `sender` (the request's `msg.sender`) as the account owner. A production app that supports **facilitators / meta-transactions** (see `submitRequestFor` in `1_summary.md`) should additionally verify an EIP-712 signature over the execute payload and burn a per-user nonce before locking — this is exactly what the Zen Way reference does. secp256k1 recovery is heavy under TinyGo; keep it out of the hot path unless you need it.

### TrustedRequest — Reconcile the Result

The `TRUSTPROCESS` request re-enters here with the **clear-text, ABI-encoded** payload the trigger produced: `(bytes16 lockId, uint256 remain, uint8 outcome)`. The app decodes it, credits `remain` back to the lock owner, clears the lock, and emits an encrypted outcome event to the owner.

Crucially, it returns **no `AppEvent`s** — that is what makes the trigger return `""` next time and terminates the round-trip.

```go
func TrustedRequest(payloadString, stateJSON string) types.ProcessResult {
    lockID, remain, outcome, err := abiDecodeActionExecuted([]byte(payloadString))
    if err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("trusted_request: %v", err)}
    }

    var state ApplicationInternalState
    if err := json.Unmarshal([]byte(stateJSON), &state); err != nil {
        return types.ProcessResult{Error: fmt.Sprintf("failed to parse state: %v", err)}
    }

    lockKey := hex.EncodeToString(lockID[:])
    lock := state.Locks[lockKey]
    if lock == nil {
        return types.ProcessResult{Error: "trusted_request: unknown or already-resolved lock"}
    }
    delete(state.Locks, lockKey) // idempotency: a replayed TRUSTPROCESS finds no lock

    owner := lock.Owner
    acc := state.Accounts[owner.Hex()]
    if acc == nil {
        acc = &AccountState{Address: owner, Balances: make(map[string]*types.Uint256)}
        state.Accounts[owner.Hex()] = acc
    }
    bal := acc.Balances[ethTokenHex]
    if bal == nil {
        bal = types.NewUint256(0)
        acc.Balances[ethTokenHex] = bal
    }
    if bal.AddOverflow(*bal, *remain) {
        return types.ProcessResult{Error: "trusted_request: credit overflow"}
    }

    outcomeStr := "success"
    if outcome != 0 {
        outcomeStr = "failure"
    }
    remainCopy, balCopy := *remain, *bal
    data, _ := json.Marshal(executionOutcome{
        Type:    subtypeExecutionOutcome,
        LockID:  lockKey,
        Outcome: outcomeStr,
        Remain:  &remainCopy,
        Balance: &balCopy,
    })
    events := []types.PlainEvent{{
        UserID:       owner,
        EventSubType: subtypeToBytes32(subtypeExecutionOutcome),
        Data:         data,
    }}

    newState, _ := json.Marshal(&state)
    // NOTE: no AppEvents -> trigger returns "" -> TRUSTPROCESS loop terminates.
    return types.ProcessResult{State: newState, Events: events, Fuel: types.NewUint256(50)}
}
```

On **failure** (`outcome == 1`), the trigger's `execute` reverted, its `withdraw()` swept the *full* claimed amount back, so `remain` equals the whole locked amount — the user is made whole. On **success**, `remain` is exactly what the on-chain call did not spend.

---

## Step 5: ABI Helpers (Guest Side)

The WASM guest cannot import `go-ethereum`, so it hand-rolls the small amount of ABI encoding needed to talk to the trigger contract. The wire formats are deliberately kept to fixed shapes so the code stays short and correct under TinyGo.

`app/abi.go`:

```go
package app

import (
    "encoding/binary"
    "encoding/hex"
    "fmt"
    "strings"

    "github.com/HorizenOfficial/vela-common-go/wasm/types"
)

// lockIDFromNonce derives a deterministic bytes16 lock id from the monotonic
// LockNonce. No randomness — TinyGo requires determinism.
func lockIDFromNonce(n uint64) [16]byte {
    var id [16]byte
    binary.BigEndian.PutUint64(id[8:], n)
    return id
}

// subtypeToBytes32 packs a short ASCII label left-aligned into a bytes32 topic.
func subtypeToBytes32(s string) [32]byte {
    var b [32]byte
    copy(b[:], s)
    return b
}

func decodeHex(s string) ([]byte, error) {
    s = strings.TrimPrefix(s, "0x")
    if s == "" {
        return nil, nil
    }
    return hex.DecodeString(s)
}

// abiEncodeExecuteRequested encodes abi.encode(bytes16 lockId, address target,
// uint256 value, bytes data). Head = 4 words; the dynamic `bytes` tail follows.
func abiEncodeExecuteRequested(lockID [16]byte, target types.Address, value *types.Uint256, data []byte) []byte {
    out := make([]byte, 0, 32*5+len(data))

    var w0 [32]byte // bytes16 lockId — left-aligned
    copy(w0[:16], lockID[:])
    out = append(out, w0[:]...)

    var w1 [32]byte // address — right-aligned (last 20 bytes)
    copy(w1[12:], target[:])
    out = append(out, w1[:]...)

    out = append(out, value.Bytes()...) // uint256 — 32-byte big-endian

    var w3 [32]byte // offset to the bytes tail = 4 head words * 32 = 128
    binary.BigEndian.PutUint64(w3[24:], 128)
    out = append(out, w3[:]...)

    var wlen [32]byte // tail: bytes length
    binary.BigEndian.PutUint64(wlen[24:], uint64(len(data)))
    out = append(out, wlen[:]...)

    out = append(out, data...) // tail: bytes payload, right-padded to 32
    if rem := len(data) % 32; rem != 0 {
        out = append(out, make([]byte, 32-rem)...)
    }
    return out
}

// abiDecodeActionExecuted decodes abi.encode(bytes16 lockId, uint256 remain,
// uint8 outcome) — three static words, 96 bytes.
func abiDecodeActionExecuted(payload []byte) (lockID [16]byte, remain *types.Uint256, outcome uint8, err error) {
    if len(payload) != 96 {
        return lockID, nil, 0, fmt.Errorf("malformed payload: expected 96 bytes, got %d", len(payload))
    }
    copy(lockID[:], payload[0:16]) // bytes16 — left-aligned in the first word
    r := &types.Uint256{}
    r.SetBytes(payload[32:64]) // uint256 remain
    return lockID, r, payload[95], nil // uint8 outcome — last byte of the third word
}
```

`types.Uint256.Bytes()` returns the 32-byte big-endian form, which *is* the ABI encoding of a single `uint256`; `SetBytes` is its inverse. That property is what keeps these helpers tiny.

---

## Step 6: The Trigger Contract

The companion contract extends `AbstractTrigger` from the platform (`contracts/contracts/trigger/AbstractTrigger.sol`). You override two hooks; the base class handles the rest (the `_onlyProcessorEndpoint` guard, the non-overridable `withdraw()` sweep, and event emission).

`contracts/MixerTrigger.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {AbstractTrigger} from "vela/contracts/trigger/AbstractTrigger.sol";
import {IProcessorEndpoint} from "vela/contracts/interfaces/IProcessorEndpoint.sol";
import {Structs} from "vela/contracts/Structs.sol";

contract MixerTrigger is AbstractTrigger {
    error CallFailed();

    constructor(IProcessorEndpoint _processorEndpoint) AbstractTrigger(_processorEndpoint) {}

    // Perform the user's on-chain call from the pool's (this contract's) balance.
    // The ProcessorEndpoint has already claimed the withdrawn ETH into this contract.
    function _execute(Structs.EventData calldata appEventData) internal override {
        if (appEventData.events.length == 0) return; // TRUSTPROCESS re-entry: no events
        (, address target, uint256 value, bytes memory data) =
            abi.decode(appEventData.events[0], (bytes16, address, uint256, bytes));

        (bool ok, ) = target.call{value: value}(data);
        if (!ok) revert CallFailed(); // revert => executeSuccess=false => full amount swept back
    }

    // Build the trusted payload the WASM `trusted_request` consumes:
    // (bytes16 lockId, uint256 remain, uint8 outcome). `remain` is the ETH that
    // withdraw() swept back (returnedTokens has ETH last). Returning "" enqueues
    // nothing — the guard below is what terminates the TRUSTPROCESS loop.
    function _getTrustProcessPayload(
        Structs.EventData calldata appEventData,
        bool executeSuccess,
        bool /*withdrawSuccess*/,
        Structs.TokenAndAmount[] calldata returnedTokens,
        Structs.TokenAndAmount[] calldata /*failedTokens*/
    ) internal pure override returns (bytes memory) {
        if (appEventData.events.length == 0) return ""; // no follow-up for a TRUSTPROCESS
        (bytes16 lockId, , , ) =
            abi.decode(appEventData.events[0], (bytes16, address, uint256, bytes));

        uint256 remain = 0;
        if (returnedTokens.length > 0) {
            remain = returnedTokens[returnedTokens.length - 1].amount; // ETH is last
        }
        return abi.encode(lockId, remain, uint8(executeSuccess ? 0 : 1));
    }
}
```

Contract-side rules to internalize:
- **`_execute` runs the call from the pool.** If it reverts, the endpoint's try/catch sets `executeSuccess = false`, the base `withdraw()` sweeps the full claimed amount back, and the user is refunded in full.
- **`_getTrustProcessPayload` decodes the `AppEvent`** your WASM emitted (`appEventData.events[0]`) and builds the trusted payload. A **non-empty** return enqueues the `TRUSTPROCESS`; the `events.length == 0` guard makes trusted requests non-recursive.
- You never write `execute`/`withdraw`/`getTrustProcessPayload` (the external entrypoints) or the `_onlyProcessorEndpoint` guard — those come from `AbstractTrigger`. You must **not** implement `ITrigger` directly.

---

## Step 7: Payload & Wire Formats

Three distinct encodings cross the boundary — get these exactly right or the round-trip breaks:

| Direction | Format | Encoding |
|-----------|--------|----------|
| User → `process_request` | `{"command":"execute","execute":{...}}` | JSON (ECDH-encrypted in transit) |
| WASM `AppEvent.Data` → trigger `_execute`/`_getTrustProcessPayload` | `(bytes16 lockId, address target, uint256 value, bytes data)` | **ABI** (Solidity decodes on-chain) |
| Trigger `_getTrustProcessPayload` → WASM `trusted_request` | `(bytes16 lockId, uint256 remain, uint8 outcome)` | **ABI** (clear text, not decrypted) |

**Execute payload** (decrypted by the Executor, then handed to `process_request`):

```json
{
  "command": "execute",
  "execute": {
    "target": "0x1234567890abcdef1234567890abcdef12345678",
    "value": "0x6f05b59d3b20000",
    "data": "0xa9059cbb..."
  }
}
```

`value` is the ETH forwarded with the call; `data` is the calldata (empty `"0x"` for a plain transfer). The app locks `value`, so the user must have deposited at least that much.

The two trigger-facing formats are ABI-encoded because Solidity decodes them on-chain, where `abi.decode` is far cheaper than JSON parsing. The WASM produces/consumes them with the helpers in Step 5.

---

## Step 8: Events

A trigger app emits both kinds of Vela events, for different purposes:

**`AppEvent` (plaintext) — the trigger channel.** `process_request` emits exactly one `AppEvent` per execute. Its `Data` is the ABI-encoded call parameters, and the ProcessorEndpoint hands it to the trigger's `execute` and `getTrustProcessPayload` callbacks during `stateUpdate` (see `1_summary.md` §4.5). This is how a WASM app passes parameters to its trigger contract.

**`PlainEvent` (per-user, encrypted) — user notifications.** `deposit` emits a `deposit` confirmation; `trusted_request` emits an `execution_outcome` to the lock owner once the round-trip settles. The Executor encrypts these with the recipient's registered P-521 key.

| Event | Kind | Emitted by | Recipient | Data |
|-------|------|-----------|-----------|------|
| `execute_requested` | `AppEvent` (plaintext) | `process_request` | trigger contract | ABI `(lockId, target, value, data)` |
| `deposit` | `PlainEvent` (encrypted) | `deposit` | depositor | `{type, balanceAfter}` |
| `execution_outcome` | `PlainEvent` (encrypted) | `trusted_request` | lock owner | `{type, lockId, outcome, remain, balance}` |

> **Loop invariant, restated where it matters:** in this app `trusted_request` returns **zero `AppEvent`s**, and the trigger's empty-events guard turns that into a `""` payload — so the round-trip stops after one trusted request. Emitting an `AppEvent` from the trusted path is allowed and chains another `TRUSTPROCESS`, but then *you* own termination: without a provable stopping condition (a step counter, a shrinking amount, a bounded depth) it becomes a never-ending `TRUSTPROCESS` loop. See Step 10.

---

## Step 9: Fuel

| Operation | Fuel |
|-----------|------|
| `Deploy` / `LoadModule` | 5 |
| `Deposit` | 35 |
| `ProcessRequest` (execute) | 60 |
| `TrustedRequest` | 50 |

Fuel is application-defined; set it proportional to computational cost. Note that `TRUSTPROCESS` requests are **not** charged an application fee (the Executor sets it to `0`) and skip the minimum-fee check — their authenticity is established on-chain, not by a user payment — but you still return a `Fuel` value.

---

## Step 10: Error Handling & Loop Safety

Standard rules apply — return early via the `Error` field and never leave state partially mutated — plus two trigger-specific concerns:

- **Idempotency of `trusted_request`.** Resolve each lock exactly once. Deleting the lock on first handling (as above) means a replayed or duplicated `TRUSTPROCESS` finds no lock and errors out instead of double-crediting.
- **Mind recursion when a `TRUSTPROCESS` emits an `AppEvent`.** It is *not* forbidden for `trusted_request` to emit an `AppEvent` — a trusted request can legitimately fire the trigger again, chaining one `TRUSTPROCESS` into the next. But every `AppEvent` from the trusted path is a new turn of the loop, so you must be very careful to guarantee it terminates. In this app the simplest safe choice is taken: `trusted_request` emits **no** `AppEvent`s, and the trigger's `appEventData.events.length == 0` guard returns `""`, so the round-trip always stops after exactly one trusted request. If you deliberately do chain (e.g. a multi-step settlement), make the termination condition explicit and provable — a monotonic step counter in state, a decreasing amount, or a bounded depth — and gate the `AppEvent` on it, or an app-level bug becomes a never-ending `TRUSTPROCESS` loop.

A failed `process_request` (e.g. insufficient balance) reverts before emitting any withdrawal or app event, so **no** trigger fires and **no** `TRUSTPROCESS` is enqueued — the failure is contained to that request.

---

## Deploying with a Trigger

A trigger app requires the trigger address to be registered in **two** places (see `1_summary.md` §3.1):

1. **On-chain**, at deploy time: the trigger contract must be **deployed first**, then the app is deployed with `submitDeployRequestWithTrigger(protocolVersion, payload, trigger)` (instead of `submitDeployRequest`), passing the already-deployed trigger address. This wires `triggerContracts[applicationId]` in the `ProcessorEndpoint`, so the endpoint invokes the trigger during `stateUpdate`.
2. **In the WASM state**, via the deploy descriptor's `constructorParams`: the same address is passed as `{"triggerContract":"0x…"}`, so the app can stamp it into `Withdrawal.DestinationAddress`.

```
1. Deploy MixerTrigger(processorEndpoint)                     → triggerAddr
2. Upload mixer_app.wasm to the Authority Service             → sha256
3. submitDeployRequestWithTrigger(
       protocolVersion,
       { "mode": "artifact_ref", "artifactId": "sha256:…", "wasmSha256": "…",
         "constructorParams": { "triggerContract": "<triggerAddr>" } },
       triggerAddr)
```

Both must point at the same contract. If only the on-chain side is set, the app never emits withdrawals to the trigger; if only the WASM side is set, the endpoint never calls the trigger.

---

## TinyGo Constraints

The same limits as any Vela app (see `2_private-transfer-app.md`), with two that bite trigger apps specifically:

| Constraint | Detail |
|------------|--------|
| **Determinism required** | Lock IDs must be derived from in-state counters, **not** `crypto/rand` or UUIDs — the Rust reference uses UUIDs, but a TinyGo guest must stay deterministic. |
| **No `go-ethereum`** | ABI encoding/decoding for the trigger wire formats must be hand-rolled (Step 5); you cannot import `github.com/ethereum/go-ethereum/accounts/abi`. |
| **Limited stdlib** | `math/big` is unavailable — use `types.Uint256`. `encoding/binary`, `encoding/hex`, and `encoding/json` are available. |
| **Shared types** | Use `Address`, `Uint256`, `PlainEvent`, `AppEvent`, `Withdrawal` from `vela-common-go/wasm/types`. |

---

## Summary

Building a trigger-contract app on Vela adds four things on top of a normal app:

1. **Export `trusted_request`** — the handler for `TRUSTPROCESS` requests (no sender, no requestType, clear-text payload).
2. **Track locks in state** — remember who owns each in-flight execution's funds so the leftover can be returned.
3. **From `process_request`, emit a `Withdrawal` to the trigger + an ABI-encoded `AppEvent`** — this moves funds into the trigger and passes it the call parameters.
4. **Write a trigger contract** extending `AbstractTrigger`, overriding `_execute` (perform the on-chain action) and `_getTrustProcessPayload` (build the trusted payload, guarding on empty events to terminate the loop).

Register the trigger both on-chain (`submitDeployRequestWithTrigger`) and in the WASM (`constructorParams.triggerContract`). Keep `trusted_request` idempotent and AppEvent-free, and the round-trip is safe and finite.

