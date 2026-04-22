# Vela TypeScript Client Library (v0.0.25)

The `@horizen/vela-common-ts` library provides everything a browser application needs to interact with the Vela CCE platform: key derivation from a wallet, payload encryption, request submission, event decryption, and withdrawal collection. It is designed for browser environments using the Web Crypto API.

> **Prerequisite**: Read `1_summary.md` for the full system architecture. This document focuses on the client side — how a user's browser application communicates with the platform.

Source code: https://github.com/HorizenOfficial/vela-common-ts

---

## Installation

```bash
npm install @horizen/vela-common-ts ethers
```

The package is published under the `@horizen/` scope. `ethers` v6 is a peer dependency (used for wallet integration and contract interaction).

---

## Quick Overview

A typical user interaction follows this flow:

```
1. Connect wallet           →  ethersSignerFromBrowser()
2. Initialize client        →  new VelaClient(signer, ...)
3. Register encryption key  →  submitRequest(ASSOCIATEKEY, ...)
4. Encrypt & send request   →  encryptForTee(payload) → submitRequest(PROCESS, ...)
5. Wait for completion      →  getRequestCompletedEvent(requestId, ...)
6. Decrypt response events  →  getCurrentUserEvents(...) or fetchAndDecryptUserEvents(...)
7. Collect withdrawals      →  getPendingClaims(token, addr) → claim(token, addr)
```

Deployers additionally use `submitDeployRequest*` / `getDeployRequestCompletedEvent` — see the [Deploying a WASM Application](#deploying-a-wasm-application) section.

---

## Connecting a Wallet

```typescript
import { ethersSignerFromBrowser } from "@horizen/vela-common-ts";

// Get signer from MetaMask (or any injected wallet)
const signer = await ethersSignerFromBrowser();
```

This uses `window.ethereum` to create an ethers.js `BrowserProvider` and returns a `Signer`. Works with MetaMask, WalletConnect, and other injected providers.

---

## Initializing the Client

```typescript
import { VelaClient } from "@horizen/vela-common-ts";

const client = new VelaClient(
  signer,
  false,                          // useAlternativeSign (true for eth_sign RPC)
  "0x<TeeAuthenticator address>", // TEE Authenticator contract
  "0x<ProcessorEndpoint address>"  // Processor Endpoint contract
);
```

The client wraps both smart contracts and handles key derivation, encryption, and decryption internally.

| Parameter | Description |
|-----------|-------------|
| `signer` | ethers.js Signer from the user's wallet |
| `useAlternativeSign` | Use `eth_sign` RPC method instead of `signMessage()`. Set `true` if the wallet doesn't support EIP-191 personal_sign |
| `teeAuthenticatorAddress` | On-chain address of the TEE Authenticator contract |
| `processorEndpointAddress` | On-chain address of the Processor Endpoint contract |

---

## Key Derivation

Every user needs a P-521 key pair for encrypted communication with the TEE. The library derives this deterministically from the user's wallet signature — the same wallet always produces the same key pair, with no extra seed phrase to manage.

```typescript
// Automatic — the client derives keys internally when needed
const keyPair = await client.getSignerKeyPair();
// keyPair.privateKey  → CryptoKey (for decrypting events)
// keyPair.publicKey   → CryptoKey (registered on-chain for receiving events)
```

### How It Works

1. The library asks the wallet to sign a challenge message
2. The signature bytes become the Input Keying Material (IKM) for HKDF
3. HKDF derives a P-521 private key using rejection sampling
4. The public key is computed from the private key

```
wallet.signMessage("horizen0x1234...")
       │
       ▼ signature bytes (IKM)
    HKDF-SHA256(ikm, salt=[], info=[])
       │
       ▼ 528-bit output (MSB masked to 521 bits)
    rejection sampling → valid P-521 private key
       │
       ▼
    P521KeyPair { privateKey, publicKey }
```

Since the derivation is deterministic, the user doesn't need to store or back up the key — it can always be re-derived from the same wallet.

### Low-Level Key Functions

For advanced use cases, the library also exports lower-level key operations:

```typescript
import {
  deriveP521PrivateKeyFromSigner,  // derive from any ethers Signer
  deriveKeyPairFromHKDF,            // derive from raw IKM bytes
  deriveKeyPairFromSeed,            // derive from a seed (uses SHA-512)
  generateKeyPair,                  // generate a random P-521 key pair
  importPublicKeyFromHex,           // import a public key from hex
  exportPublicKeyToHex,             // export a public key to hex
  importPrivateKeyFromHex,          // import a private key from hex (d value)
  importPrivateKeyFromJWK,          // import a private key from JWK
  exportPrivateKeyToJWK,            // export a private key to JWK
  P521KeyPair,
} from "@horizen/vela-common-ts";

// Derive with custom challenge/salt/info
const keyPair = await deriveP521PrivateKeyFromSigner(
  signer,
  false,
  "custom-challenge",     // override default "horizen"
  new Uint8Array([1, 2]), // custom HKDF salt
  new Uint8Array([3, 4])  // custom HKDF info
);

// Import a public key received from elsewhere
const teePubKey = await importPublicKeyFromHex("04abcdef...");

// Export for display or storage
const hexPubKey = await exportPublicKeyToHex(keyPair.publicKey);

// Import/export private keys
const privKey = await importPrivateKeyFromHex("abcdef...");
const jwk = await exportPrivateKeyToJWK(keyPair.privateKey);
```

---

## Registering Your Key (Associate Key)

Before you can receive encrypted events or submit encrypted payloads, your P-521 public key must be registered on-chain via an `ASSOCIATEKEY` request.

The `ASSOCIATEKEY` payload is either:
- **133 bytes** — raw uncompressed P-521 public key only, or
- **226 bytes** — public key (133 B) concatenated with an encrypted subtype seed (93 B). Opt in to this form if you want privacy-preserving event sub-types derived from the seed via HMAC (see `generateSubtypeSet`).

```typescript
import {
  RequestType,
  exportPublicKeyToHex,
  hexToBytes,
  ETH_TOKEN,
  PROTOCOL_VERSION,
  buildAssociateKeyPayload,
} from "@horizen/vela-common-ts";

const keyPair = await client.getSignerKeyPair();
const publicKeyHex = await exportPublicKeyToHex(keyPair.publicKey);

// Option A: 133-byte payload (public key only)
const publicKeyBytes = hexToBytes(publicKeyHex);

// Option B: 226-byte payload (public key + encrypted subtype seed)
// const publicKeyBytes = await buildAssociateKeyPayload(
//   keyPair.publicKey,
//   await client.getTeePublicKey(),
//   seed,
// );

const receipt = await client.submitRequestAndWaitForRequestId(
  PROTOCOL_VERSION,          // protocol version
  applicationId,             // your application ID
  RequestType.ASSOCIATEKEY,  // request type
  publicKeyBytes,            // payload = public key (+ optional encrypted seed)
  ETH_TOKEN,                 // token address (0x0 for ETH)
  0n,                        // asset amount (no deposit for ASSOCIATEKEY)
  maxFee                     // max fee in wei
);

console.log("Key registered, requestId:", receipt.requestId);
```

The Executor stores your public key (and optional subtype seed) in the application's user key store. All future events targeted at your address will be encrypted with this key.

---

## Submitting Requests

### Encrypt and Submit

```typescript
import { RequestType, stringToBytes, ETH_TOKEN, PROTOCOL_VERSION } from "@horizen/vela-common-ts";

// 1. Build your payload (application-specific JSON)
const payload = JSON.stringify({
  type: "transfer",
  transfer: {
    to: "0x1234567890abcdef1234567890abcdef12345678",
    amount: "0x6f05b59d3b20000",  // 0.5 ETH in hex
    invoice_id: "INV-2025-001"     // optional tracking ID
  }
});

// 2. Encrypt for TEE (ECDH with TEE's public key)
const encryptedPayload = await client.encryptForTee(
  stringToBytes(payload)
);

// 3. Submit on-chain (ETH example)
const receipt = await client.submitRequestAndWaitForRequestId(
  PROTOCOL_VERSION,        // protocol version
  applicationId,           // application ID
  RequestType.PROCESS,     // request type
  encryptedPayload,        // encrypted payload
  ETH_TOKEN,               // token address (0x0 = native ETH)
  assetAmount,             // asset amount in wei (bigint)
  maxFee                   // max fee in wei (bigint)
);

console.log("Request submitted, id:", receipt.requestId);
```

`encryptForTee()` handles the full encryption flow internally:
1. Fetches the TEE's P-521 public key from the `TeeAuthenticator` contract
2. Derives the user's P-521 private key from the wallet
3. Performs ECDH to compute a shared secret
4. Derives an AES-256 key via HKDF
5. Encrypts the payload with AES-256-GCM

For ETH (`tokenAddress === ETH_TOKEN`) the transaction value sent to the contract is `assetAmount + maxFeeValue`. For ERC-20 tokens, the transaction value is only `maxFeeValue` — the asset is pulled via `transferFrom`, so the caller must pre-approve the `ProcessorEndpoint`:

```typescript
// One-time approval per token (or per session) before ERC-20 submits
await (await client.approveToken(erc20Address, assetAmount)).wait();

const receipt = await client.submitRequestAndWaitForRequestId(
  PROTOCOL_VERSION,
  applicationId,
  RequestType.PROCESS,
  encryptedPayload,
  erc20Address,            // token contract address
  assetAmount,             // amount to pull via transferFrom
  maxFee,
);
```

Only tokens registered in the on-chain `TokenAllowlist` are accepted.

### Request Types

```typescript
enum RequestType {
  DEPLOYAPP = 0,       // deploy a new WASM application
  PROCESS = 1,         // standard request (transfer, withdraw, etc.)
  DEANONYMIZATION = 2, // request a deanonymization report
  ASSOCIATEKEY = 3     // register a P-521 public key
}
```

### Deposit (Funding Your Account)

To deposit funds into your application account, submit a `PROCESS` request with a non-zero `assetAmount`:

```typescript
const receipt = await client.submitRequestAndWaitForRequestId(
  PROTOCOL_VERSION,
  applicationId,
  RequestType.PROCESS,
  encryptedPayload,      // can be an empty encrypted payload
  ETH_TOKEN,             // or an ERC-20 address (must be approved first)
  parseEther("1.0"),     // deposit 1 ETH
  maxFee
);
```

The smart contract takes custody of the asset (native transfer for ETH; `transferFrom` for ERC-20). The WASM application's `deposit()` function credits the user's internal account.

---

## Waiting for Completion

After submitting a request, poll for its completion:

```typescript
const result = await client.getRequestCompletedEvent(
  receipt.requestId,
  currentBlock,   // fromBlock (latest end of range)
  deployBlock     // toBlock (earliest end of range)
);

if (result) {
  console.log("Status:", result.status);         // 0n = success
  console.log("Error:", result.errorMessage);     // undefined if success
}
```

> Note: `fromBlock` is the upper bound and `toBlock` is the lower bound of the block range to search.

---

## Decrypting Events

Events are the primary way the application communicates results back to users. Each event is encrypted with the target user's P-521 public key; only the intended recipient can decrypt it.

### Option A: Direct Contract Query

```typescript
const events = await client.getCurrentUserEvents(
  currentBlock,     // fromBlock (upper bound)
  deployBlock,      // toBlock (lower bound)
  applicationId,
  "deposit",        // eventSubType filter (or undefined for all)
  (data) => true,   // filter function on decrypted payload
  false             // stopAtFirst
);

for (const eventBytes of events) {
  const event = JSON.parse(bytesToString(eventBytes));
  console.log(event);
  // { type: "deposit", amount: "0x...", balance: "0x...", nonce: 1 }
}
```

The client internally:
1. Queries `UserEvent` logs from the `ProcessorEndpoint` contract
2. Attempts to decrypt each event with the user's private key
3. Silently skips events intended for other users (decryption fails)
4. Applies the filter function on successfully decrypted payloads

### Option B: Subgraph Query (Recommended for Production)

For better performance and pagination, use the subgraph client:

```typescript
import {
  createSubgraphClient,
  fetchAndDecryptUserEvents,
  importPublicKeyFromHex,
  bytesToString,
} from "@horizen/vela-common-ts";

// Create subgraph client
const subgraph = createSubgraphClient(
  "https://your-subgraph-endpoint/graphql",
  10_000  // timeout in ms
);

// Health check
await subgraph.healthCheck();

// Fetch TEE public key (needed for decryption)
const teePublicKeyHex = await client.getTeePublicKey();
const teePublicKey = await importPublicKeyFromHex(teePublicKeyHex);

// Get user's private key
const keyPair = await client.getSignerKeyPair();

// Fetch and decrypt events
const events = await fetchAndDecryptUserEvents(
  subgraph,
  teePublicKey,
  keyPair.privateKey,
  applicationId,
  "transfer_received",   // eventSubType filter
  10,                     // limit (0 = no limit)
  (data) => true          // optional filter on decrypted payload
);

for (const eventBytes of events) {
  const event = JSON.parse(bytesToString(eventBytes));
  console.log(event);
}
```

`fetchAndDecryptUserEvents` handles pagination automatically — it fetches pages of events from the subgraph (up to 1000 per page), attempts decryption on each, and accumulates results until the limit is reached.

### Subgraph Types

```typescript
interface UserEvent {
  applicationId: bigint;
  requestId: string;
  eventSubType: string;
  encryptedData: Uint8Array;   // raw encrypted bytes
  blockNumber: number;
  logIndex: number;
  sortKey: bigint;             // for pagination cursor
}

interface AppEvent {                // plaintext app-level event
  applicationId: bigint;
  requestId: string;
  eventSubType: string;
  data: Uint8Array;
  blockNumber: number;
  logIndex: number;
  sortKey: bigint;
}

interface RequestCompleted {
  applicationId: bigint;
  requestId: string;
  status: number;
  errorCode: number;
  errorMessage: string;
  applicationFees: bigint;
  blockNumber: number;
}

interface DeployRequestCompleted {  // emitted for submitDeployRequest completions
  applicationId: bigint;
  requestId: string;
  applicationFees: bigint;
  status: number;
  errorCode: number;
  errorMessage: string;
  blockNumber: number;
}

interface OnChainRefund {
  applicationId: bigint;
  requestId: string;
  to: string;
  tokenAddress: string;           // 0x0 = ETH
  amount: bigint;
  blockNumber: number;
}

interface OnChainWithdrawal {
  applicationId: bigint;
  requestId: string;
  to: string;
  tokenAddress: string;           // 0x0 = ETH
  amount: bigint;
  blockNumber: number;
}

interface ClaimExecuted {           // emitted when a pending claim is pulled
  tokenAddress: string;
  payee: string;
  amount: bigint;
  blockNumber: number;
}
```

---

## Deploying a WASM Application

Deploys go through a dedicated endpoint (`submitDeployRequest`) — not `submitRequest`. The caller must hold `DEPLOYER_ROLE` on the `ProcessorEndpoint` contract. The payload is a JSON deploy descriptor that references the WASM artifact by its `sha256` (the artifact itself is uploaded out-of-band to the Authority Service beforehand):

```typescript
import { PROTOCOL_VERSION } from "@horizen/vela-common-ts";

const wasmBytes = await fetch("/payment_app.wasm").then(r => r.arrayBuffer());
const wasmSha256 = new Uint8Array(
  await crypto.subtle.digest("SHA-256", wasmBytes)
);

const receipt = await client.submitDeployRequestAndWaitForRequestId(
  PROTOCOL_VERSION,
  maxFee,
  wasmSha256,
  { /* optional constructor params, included in the deploy descriptor */ },
);

console.log("Deploy requestId:", receipt.requestId);

// Wait for completion (uses the DeployRequestCompleted event, not RequestCompleted)
const deployResult = await client.getDeployRequestCompletedEvent(
  undefined,            // applicationId (unknown until completion)
  receipt.requestId,
  currentBlock,
  deployBlock,
);
console.log("Assigned applicationId:", deployResult?.applicationId);
```

The contract derives a fresh `applicationId` from the request hash; you learn the value from the `DeployRequestCompleted` event.

---

## Application-level (Plaintext) Events

Some apps emit non-encrypted, app-wide `AppEvent`s alongside per-user `UserEvent`s. These are queried separately:

```typescript
const appEvents = await client.getAppEvents(
  currentBlock,                 // fromBlock (upper bound)
  deployBlock,                  // toBlock (lower bound)
  applicationId,
  undefined,                    // requestId filter (or undefined)
  "global_state_snapshot",      // eventSubType filter (or undefined)
);

for (const ev of appEvents) {
  console.log(ev.eventSubType, ev.data);
}
```

---

## Withdrawals (Pull Payments / Claims)

After a withdrawal is processed by the WASM application, the smart contract credits the destination address via pull-payment, tracked per token. The recipient must explicitly claim their funds:

```typescript
import { ETH_TOKEN } from "@horizen/vela-common-ts";

// Check pending claims for an address (per token)
const pending = await client.getPendingClaims(ETH_TOKEN, userAddress);
console.log("Pending withdrawal:", pending, "wei");

// Claim funds
if (pending > 0n) {
  const tx = await client.claim(ETH_TOKEN, userAddress);
  await tx.wait();
  console.log("Claim completed");
}

// For an ERC-20 token, pass its contract address instead of ETH_TOKEN
// const erc20Pending = await client.getPendingClaims(erc20Address, userAddress);
// await (await client.claim(erc20Address, userAddress)).wait();
```

---

## Low-Level Encryption / Decryption

For cases where you need direct control over encryption (e.g., encrypting a deanonymization report, or working outside the `VelaClient`):

```typescript
import {
  encrypt,
  decrypt,
  encryptWithAES,
  decryptWithAES,
  importPublicKeyFromHex,
  stringToBytes,
  bytesToString,
} from "@horizen/vela-common-ts";

// Encrypt a message for a known public key (ECDH + AES-256-GCM)
const receiverPubKey = await importPublicKeyFromHex("04abcdef...");
const myKeyPair = await client.getSignerKeyPair();

const ciphertext = await encrypt(
  myKeyPair.privateKey,
  receiverPubKey,
  stringToBytes("hello world")
);
// ciphertext = [12-byte nonce || AES-GCM ciphertext + auth tag]

// Decrypt a received message
const plaintext = await decrypt(
  myKeyPair.privateKey,
  senderPublicKey,
  ciphertext
);
console.log(bytesToString(plaintext)); // "hello world"
```

The encryption scheme:
1. **ECDH** on P-521 curve → 528-bit shared secret
2. **HKDF-SHA256** → 256-bit AES key
3. **AES-256-GCM** with random 12-byte nonce and 128-bit auth tag
4. Output format: `[nonce (12 bytes) || ciphertext || auth tag]`

This format is compatible with the Go implementation in the Executor — messages encrypted in TypeScript can be decrypted by the TEE, and vice versa.

Lower-level AES functions are also available for direct use:

```typescript
// Encrypt/decrypt with a raw AES-256-GCM CryptoKey
const encrypted = await encryptWithAES(aesKey, plainBytes);
const decrypted = await decryptWithAES(aesKey, encrypted);
```

---

## Utility Functions

```typescript
import {
  hexToBytes,     // "0xabcd" → Uint8Array
  bytesToHex,     // Uint8Array → "abcd" (no 0x prefix)
  stringToBytes,  // "hello" → Uint8Array (UTF-8)
  bytesToString,  // Uint8Array → "hello" (UTF-8)
} from "@horizen/vela-common-ts";
```

---

## Full API Reference

### VelaClient Methods

| Method | Description |
|--------|-------------|
| `submitRequest(protocolVersion, applicationId, requestType, payload, tokenAddress, assetAmount, maxFeeValue)` | Submit a PROCESS / DEANONYMIZATION / ASSOCIATEKEY request (returns transaction response) |
| `submitRequestAndWaitForRequestId(...)` | Same signature as `submitRequest`; waits for receipt and returns `RequestReceipt` |
| `submitDeployRequest(protocolVersion, maxFeeValue, wasmSha256, constructorParams?)` | Submit a DEPLOYAPP request (caller must hold `DEPLOYER_ROLE`) |
| `submitDeployRequestAndWaitForRequestId(...)` | Same as `submitDeployRequest`; returns `RequestReceipt` |
| `approveToken(tokenAddress, amount)` | ERC-20 `approve` for the ProcessorEndpoint (required before non-ETH `submitRequest`) |
| `encryptForTee(data)` | Encrypt data for the TEE using ECDH |
| `getTeePublicKey()` | Get the TEE's P-521 public key from the contract |
| `getSignerKeyPair()` | Derive the user's P-521 key pair from their wallet |
| `getRequestCompletedEvent(requestId, fromBlock, toBlock)` | Query for PROCESS/DEANONYMIZATION/ASSOCIATEKEY completion (returns `RequestResult`) |
| `getDeployRequestCompletedEvent(applicationId?, requestId?, fromBlock, toBlock)` | Query for DEPLOYAPP completion |
| `getCurrentUserEvents(fromBlock, toBlock, applicationId, eventSubType, filter, stopAtFirst)` | Get and decrypt per-user `UserEvent`s |
| `getAppEvents(fromBlock, toBlock, applicationId, requestId?, eventSubType?)` | Get plaintext application-level `AppEvent`s |
| `decryptAndFilterEvents(events, filter, stopAtFirst)` | Decrypt and filter raw contract events |
| `getPendingClaims(tokenAddress, payee)` | Check pending withdrawal balance per token for an address |
| `claim(tokenAddress, payee)` | Execute pull-payment claim of pending balance |

### Crypto Exports

| Export | Description |
|--------|-------------|
| `encrypt(privateKey, publicKey, message)` | ECDH + AES-256-GCM encryption |
| `decrypt(privateKey, publicKey, ciphertext)` | ECDH + AES-256-GCM decryption |
| `encryptWithAES(key, message)` | Direct AES-256-GCM encryption |
| `decryptWithAES(key, ciphertext)` | Direct AES-256-GCM decryption |
| `deriveP521PrivateKeyFromSigner(signer, useAltSign, ...)` | Derive P-521 from wallet |
| `deriveKeyPairFromHKDF(ikm, salt, info)` | Derive P-521 from HKDF |
| `deriveKeyPairFromSeed(seed)` | Derive P-521 from seed (SHA-512) |
| `generateKeyPair()` | Generate random P-521 key pair |
| `importPublicKeyFromHex(hex)` | Import public key from hex |
| `exportPublicKeyToHex(key)` | Export public key to hex |
| `importPrivateKeyFromHex(hex)` | Import private key from hex (d value) |
| `importPrivateKeyFromJWK(jwk)` | Import private key from JWK |
| `exportPrivateKeyToJWK(key)` | Export private key to JWK |

### Subgraph Exports

| Export | Description |
|--------|-------------|
| `createSubgraphClient(endpoint, timeoutMs)` | Create a subgraph client instance |
| `fetchAndDecryptUserEvents(client, teePubKey, privKey, appId, subType, limit, filter)` | Fetch and decrypt events with pagination |
| `userEventSortKey(event)` | Compute sort key for pagination cursor |
| `MockSubgraphClient` | Mock implementation for testing |

### Constants

| Export | Value | Description |
|--------|-------|-------------|
| `PROTOCOL_VERSION` | `0` | Current on-chain protocol version expected by `submitRequest` / `submitDeployRequest` |
| `ETH_TOKEN` | `ZeroAddress` | Sentinel `tokenAddress` value representing native ETH |
| `CHALLENGE` | `"horizen"` | Default challenge prefix for key derivation |
| `HKDF_SALT` | `Uint8Array(0)` | Default HKDF salt (empty) |
| `HKDF_INFO` | `Uint8Array(0)` | Default HKDF info (empty) |
| `SUBTYPE_KEY_MESSAGE` | `"subtype-key-v1"` | Message signed to derive the seed for privacy-preserving event sub-types |
| `DEFAULT_SUBTYPE_N` | `50` | Number of HMAC-derived sub-types produced from a seed |

### Seed / Subtype Helpers

| Export | Description |
|--------|-------------|
| `generateSeed(signer, useAltSign?)` | Derive a seed from the wallet (uses `SUBTYPE_KEY_MESSAGE`) |
| `encryptSeed(seed, teePublicKey, userPrivateKey)` | ECDH-encrypt the seed for inclusion in the 226-byte `ASSOCIATEKEY` payload |
| `buildAssociateKeyPayload(userPublicKey, teePublicKey, seed)` | Build the 226-byte public-key + encrypted-seed payload |
| `generateSubtypeSet(seed, n?)` | Produce `n` HMAC-derived sub-type strings from the seed |
