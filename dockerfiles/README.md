This folder contains an example docker compose to be used for debug/demo.<br>
It starts a dev chain using [Foundry Anvil](https://getfoundry.sh/anvil/overview), automatically deploys the smart contracts, and runs all the components of Horizen CCE locally.

#### Limitations of this dev enviroment:

- the TEE is only emulated (no real AWS Nitro Enclave is used, just a separate process)
- only one single app deployment is supported - the app must have appId 1
- based on horizen-pes  0.0.18 tag 

## Instructions:

1) Create an .env file. For local development you can copy the ready-made dev defaults:

    ```
    cp .env.dev .env
    ```

2) Start the environment with:

    ```
    docker compose up
    ```

    The startup sequence is:
    1. **chain** (Anvil) starts and becomes available
    2. **subgraph-postgres**, **subgraph-ipfs** start (Graph Node infrastructure)
    3. **deployer** connects to the chain, deploys all smart contracts, writes the deployed addresses to a shared volume, and exits
    4. **subgraph-node** (Graph Node) starts, connects to the chain, and becomes healthy
    5. **subgraph-deployer** reads the deployed contract addresses, generates a local subgraph manifest, and deploys the subgraph to Graph Node, then exits
    6. **manager** and **authorityservice** start, reading the deployed contract addresses and querying the subgraph automatically

## Additional info:

- the manager database and chain data are persisted in docker volumes (`horizen-cce-manager-data` for the DB, `horizen-cce-chain-data` for chain data).<br>
  To start from scratch, delete the volumes.
- deployed contract addresses are stored in the `horizen-cce-deploy-data` volume. The deployer checks this on startup and skips deployment if contracts are already present on the chain.
- deanonymization reports are stored in `horizen-cce-manager-reports`; the authority service shares this reports volume so it can read the same outputs.
- to connect to the chain from Metamask, use the following parameters:
   - rpc url: http://localhost:8545
   - chainid: 31337


## Restarting and volume management

- **Restart without deleting volumes**: the deployer detects existing contracts and skips deployment. Fast restart.
- **Chain data deleted** (`docker volume rm dockerfiles_horizen-cce-chain-data`): the deployer detects contracts are missing from the chain and redeploys.
- **Deploy data deleted** (`docker volume rm dockerfiles_horizen-cce-deploy-data`): the deployer redeploys (same addresses since Anvil is deterministic with the same nonce).
- **Contracts modified**: rebuild the deployer image, delete both volumes, and restart.

## Where to go next
The system is up and running, but you need to deploy an app inside it.

Currently only  a single-app manual deployment is supported:
- the app wasm *must* be named *1.wasm* and put manually into the wasms/ folder before launching the deploy app command
- launch a deploy app command with app id = 1 to initialize it

### Practical how-to for the horizen-pes-nova test app (Private transfer):
- go to https://github.com/HorizenOfficial/horizen-pes-nova/releases/tag/v0.0.18
*(look for 0.0.18 tag for a compatible version with this dev enviroment)*
- use payment_app.wasm (remember to rename to 1.wasm)
- use the nova-linux wallet executable to launch the deploy command and interact with the app.

    Use wallet.conf.template as wallet config file, with the following properties set to connect to this dev environment:
    
    ```
    rpcUrl=http://localhost:8545
    ProcessorAddress=0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
    TeeAuthenticatorAddress=0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
    AuthorityServiceURL=http://localhost:8081
    SubgraphURL=http://localhost:8000/subgraphs/name/hcce
    ```

### Other useful repositories to look into:

https://github.com/HorizenOfficial/horizen-cce-common-ts

TypeScript library for interacting with Horizen CCE (Confidential Computing Environment) smart contracts.
Provides P-521 ECDH encryption and a blockchain client optimized for browser applications.

*(look for 0.0.18 tag for a compatible version with this dev enviroment)*








