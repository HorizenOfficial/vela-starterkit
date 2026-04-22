# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **vela-starterkit** — a developer onboarding kit for Vela CCE (Confidential Compute Environment). It contains:
- Documentation (`docs/`) describing the platform architecture, WASM app development, and TypeScript client usage
- A Docker Compose local environment (`dockerfiles/`) that runs the full Vela stack locally
- No application source code — the actual platform components live in sibling repositories

**Current version: v0.0.25** (all Docker images tagged `v0.0.25`).

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

## Key Architecture Facts (v0.0.25)

- **WASM exports**: `load_module`, `deploy` (called once at app deployment with constructor params), `deposit`, `process_request` (with `requestType int32` param). The old `generate_deanonymization_report` export was removed — deanonymization is now a case inside `process_request` routed via `requestType`.
- **ProcessResult** has an optional `Report []byte` field for deanonymization data.
- **Common types** (`Address`, `Uint256`, `PlainEvent`, `Withdrawal`, result structs) come from `vela-common-go/wasm/types`, not defined locally per app.
- **TypeScript client** class is `VelaClient` (renamed from `HorizenCCEClient`). Package is `vela-common-ts` (renamed from `horizen-cce-common-ts`).
- **Executor keyset** controlled by `EXECUTOR_KEYSET_RECOVERY_TYPE` (value `0` = fixed dev keys from env vars).

## Naming Conventions

The project was renamed from "Horizen CCE" / "horizen-pes" to "Vela CCE" / "vela" in v0.0.25. When editing docs, use "Vela" branding. Docker images still use `horizen/cce-*` prefix.
