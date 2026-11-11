# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **vela-starterkit** — a developer onboarding kit for Vela CCE (Confidential Compute Environment). It contains:
- Documentation (`docs/`) describing the platform architecture, WASM app development, and TypeScript client usage
- A Docker Compose local environment (`dockerfiles/`) that runs the full Vela stack locally
- No application source code — the actual platform components live in sibling repositories

**Current version: v0.3.0** (all Docker images tagged `v0.3.0`).

## Related Repositories

| Repository | Path | Purpose |
|------------|------|---------|
| `vela` | `https://github.com/HorizenOfficial/vela` | Core platform (Manager, Executor, smart contracts) — Go |
| `vela-nova` | `https://github.com/HorizenOfficial/vela-nova` | Reference WASM app (private transfer) — TinyGo |
| `vela-common-go` | `https://github.com/HorizenOfficial/vela-common-go` | Shared WASM types library — Go |
| `vela-common-ts` | `https://github.com/HorizenOfficial/vela-common-ts` | TypeScript client library for browsers |

When updating documentation, always verify against the actual source code in these sibling repos at the matching version tag.

## Local Environment

```bash
cd dockerfiles
cp .env.dev .env
docker compose up
```

Services: chain (Anvil), deployer, executor, manager, authorityservice, subgraph stack. All use internal Docker network `10.10.40.0/24`. Exposed ports: 8545 (RPC), 8081 (authority service), 8000 (subgraph).

Volume prefix: `vela-skit-*`. Container prefix: `vela-skit-*`.

## Documentation Structure

| File | Content |
|------|---------|
| `docs/1_summary.md` | Full architecture: smart contracts, Manager, Executor, WASM interface, crypto, deployment, config reference |
| `docs/2_private-transfer-app.md` | Step-by-step WASM app development guide with Go code from vela-nova |
| `docs/3_typescript-client.md` | Browser client: VelaClient API, key derivation, encryption, subgraph queries |
| `docs/4_trigger-contract-app.md` | Step-by-step guide for a trigger-contract app (`TRUSTPROCESS`/`trusted_request`); |

## Key Architecture Facts (v0.3.0)

- **WASM exports**: `deploy` (called once at app deployment with constructor params), `deposit`, `process_request` (with `requestType int32` param), `trusted_request` (called for `TRUSTPROCESS` requests), `allocate`/`deallocate` and `memory`. There is no `load_module` export (removed; cache warm-up only compiles the module) and no `generate_deanonymization_report` export — deanonymization is a case inside `process_request` routed via `requestType`.
- **Guest execution constraints**: every guest call runs on a fresh instance (no state in globals between calls); only a closed set of WASI imports is accepted (e.g. code linking `time.Sleep` imports `poll_oneoff` and is refused at deploy); each operation is time-bounded by `EXECUTOR_GUEST_EXECUTION_TIMEOUT_MS` (default 10 s); guest memory is capped by `EXECUTOR_MAX_GUEST_MEMORY_BYTES` (default and max 2 GiB); app state is capped by `EXECUTOR_MAX_APP_STATE_SIZE` (default 10 MiB; failure `APP_STATE_TOO_LARGE`, reported on-chain as `REQUEST_FUNC_FAILED`). The guest timeout plus 3 s must fit within `MANAGER_COMMUNICATION_PARAMS_REQUEST_TIMEOUT_SEC`, or the executor refuses the manager at handshake.
- **RequestType** has five values: `DEPLOYAPP=0`, `PROCESS=1`, `DEANONYMIZATION=2`, `ASSOCIATEKEY=3`, `TRUSTPROCESS=4`. `submitRequest` rejects `DEPLOYAPP` and `TRUSTPROCESS` with `InvalidRequestType`.
- **External triggers**: `submitDeployRequestWithTrigger`, the `ITrigger`/`AbstractTrigger` contracts, and `TriggerExecuted`/`TriggerWithdraw` events back the `TRUSTPROCESS` / `trusted_request` flow.
- **Upgradeable contracts**: `ProcessorEndpoint`, `TeeAuthenticator` and `AuthorityRegistry` run behind UUPS proxies, configured via `initialize(...)` instead of constructors; clients and config must use the **proxy** addresses. Off-hot-path `ProcessorEndpoint` entry points live in `ProcessorEndpointExtension` (reached by `delegatecall`, same ABI). In dev mode (`TEE_NO_ATTESTATION=true`) the deployer uses a non-proxied `NoAttestationTeeAuthenticator`.
- **Chain target**: Base (Base Sepolia for testing); Horizen L3 is no longer supported. Contracts target the `cancun` EVM version, so chains must be Cancun or later.
- **Request queues**: per-application queues served round-robin, with global trigger (`TRUSTPROCESS`) and deploy queues served first. The Manager reads them via `getPendingRequestsWithStateRoot(maxCount)` (`getNextPendingRequest` was removed). `stateUpdate` enforces the selected application's turn (`ApplicationNotSelected`, `PriorityQueueNotServed`), ignoring queue heads younger than `selectionGrace` (default 60 s).
- **Batch execution**: one poll settles up to `MAX_BATCH_SIZE` (default 5) requests of one application via `batchStateUpdate`, with one TEE signature over all entry hashes (`checkBatchSignature`) and a `BatchProcessed` event; each request still emits its own `RequestCompleted`/`UserEvent`/`AppEvent`. Trigger requests, deploys and apps with a trigger contract are limited to one entry.
- **ErrorCode**: `APP_NOT_DEPLOYED` was removed, so every code from `WRONG_KEY_SENT` onward shifted down by one (`WRONG_KEY_SENT=7` … `INSUFFICIENT_FUEL=11`).
- **Facilitator nonces** are read via the public mapping getter `facilitatorNonces(user)`; `getFacilitatorNonce` was removed.
- **TokenAllowlist** is a standalone contract (passed to `ProcessorEndpoint.initialize`), not internal to the endpoint. The `RESET_OPERATOR` role adds `adminReset()`/`adminResetApps()`, which are testnet/dev only — the role can only be granted at initialization and is permanently disabled in production (`address(0)`).
- **ProcessResult** has an optional `Report []byte` field for deanonymization data.
- **Common types** (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result structs `DeployResult`/`DepositResult`/`ProcessResult`) come from `vela-common-go/wasm/types`, not defined locally per app. `LoadModuleResult` no longer exists.
- **TypeScript client** class is `VelaClient` (renamed from `HorizenCCEClient`). Package is `@horizen/vela-common-ts` (renamed from `horizen-cce-common-ts`). It exports `RequestType` (including `TRUSTPROCESS`) and `ErrorCode`; `RequestResult.errorCode` is always a `bigint` (`0n` on success).
- **Executor keyset** controlled by `EXECUTOR_KEYSET_RECOVERY_TYPE` (value `0` = fixed dev keys from env vars).

## Naming Conventions

The project was renamed from "Horizen CCE" / "horizen-pes" to "Vela CCE" / "vela" in v0.0.25. When editing docs, use "Vela" branding. Docker images still use `horizen/cce-*` prefix.
