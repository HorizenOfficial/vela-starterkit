# Horizen Vela - Developer Strter Kit

This repository contains the resources needed to implement and test a Proof of Concept based on Horizen Vela — a platform that executes WebAssembly (WASM) modules inside AWS Nitro Enclaves (TEE), with encrypted state management and blockchain-based coordination.

## Documentation

The [docs/](docs/) folder contains the following guides (recommended reading order):

| # | Document | Description |
|---|----------|-------------|
| 1 | [Architecture Summary](docs/1_summary.md) | System overview, on-chain/off-chain flow, and key components |
| 2 | [Private Transfer App](docs/2_private-transfer-app.md) | Step-by-step walkthrough for building a WASM app (Go code snippets) |
| 3 | [TypeScript Client Library](docs/3_typescript-client.md) | Browser-side integration: key derivation, encryption, request submission |

## Local Test Environment

The [dockerfiles/](dockerfiles/) folder contains a Docker Compose setup that spins up a complete local environment:

- **Anvil** dev chain (Foundry)
- Automatic smart-contract deployment
- Graph Node (subgraph) infrastructure
- Processor Manager and Authority Service

See the [dockerfiles README](dockerfiles/README.md) for setup instructions and configuration details.


