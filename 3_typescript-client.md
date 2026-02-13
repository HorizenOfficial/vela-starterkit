# Horizen CCE TypeScript Client Library

The `horizen-cce-common-ts` library provides everything a browser application needs to interact with the Horizen CCE platform: key derivation from a wallet, payload encryption, request submission, event decryption, and withdrawal collection. It is designed for browser environments using the Web Crypto API.

> **Prerequisite**: Read `1_summary.md` for the full system architecture. This document focuses on the client side — how a user's browser application communicates with the platform.

---

## Installation

```bash
npm install horizen-cce-common-ts ethers
```

`ethers` v6 is a peer dependency (used for wallet integration and contract interaction).

---

## Quick Overview

A typical user interaction follows this flow:

```
1. Connect wallet           →  ethersSignerFromBrowser()
2. Initialize client        →  new HorizenCCEClient(signer, ...)
3. Register encryption key  →  submitRequest(ASSOCIATEKEY, ...)
4. Encrypt & send request   →  encryptForTee(payload) → submitRequest(PROCESS, ...)
5. Wait for completion      →  getRequestCompletedEvent(requestId, ...)
6. Decrypt response events  →  getCurrentUserEvents(...) or fetchAndDecryptUserEvents(...)
7. Collect withdrawals      →  getPendingPayments() → withdrawPayments()
```

---

## Connecting a Wallet

```typescript
import { ethersSignerFromBrowser } from "horizen-cce-common-ts";

// Get signer from MetaMask (or any injected wallet)
const signer = await ethersSignerFromBrowser();
```

This uses `window.ethereum` to create an ethers.js `BrowserProvider` and returns a `Signer`. Works with MetaMask, WalletConnect, and other injected providers.

---

## Initializing the Client

```typescript
import { HorizenCCEClient } from "horizen-cce-common-ts";

const client = new HorizenCCEClient(
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
       ▼ 528-bit output
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
  deriveKeyPairFromSeed,            // derive from a seed
  importPublicKeyFromHex,           // import a public key from hex
  exportPublicKeyToHex,             // export a public key to hex
  P521KeyPair,
} from "horizen-cce-common-ts";

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
```

---

## Registering Your Key (Associate Key)

Before you can receive encrypted events or submit encrypted payloads, your P-521 public key must be registered on-chain via an `ASSOCIATEKEY` request.

```typescript
import { RequestType, exportPublicKeyToHex } from "horizen-cce-common-ts";

const keyPair = await client.getSignerKeyPair();
const publicKeyHex = await exportPublicKeyToHex(keyPair.publicKey);
const publicKeyBytes = hexToBytes(publicKeyHex);

const receipt = await client.submitRequestAndWaitForRequestId(
  1,                         // protocol version
  applicationId,             // your application ID
  RequestType.ASSOCIATEKEY,  // request type
  publicKeyBytes,            // payload = raw public key bytes
  0n,                        // no deposit
  maxFee                     // max fee in wei
);

console.log("Key registered, requestId:", receipt.requestId);
```

The Executor stores your public key in the application's user key store. All future events targeted at your address will be encrypted with this key.

---

## Submitting Requests

### Encrypt and Submit

```typescript
import { RequestType, stringToBytes } from "horizen-cce-common-ts";

// 1. Build your payload (application-specific JSON)
const payload = JSON.stringify({
  type: "transfer",
  transfer: {
    to: "0x1234567890abcdef1234567890abcdef12345678",
    amount: "0x6f05b59d3b20000"  // 0.5 ETH in hex
  }
});

// 2. Encrypt for TEE (ECDH with TEE's public key)
const encryptedPayload = await client.encryptForTee(
  stringToBytes(payload)
);

// 3. Submit on-chain
const receipt = await client.submitRequestAndWaitForRequestId(
  1,                       // protocol version
  applicationId,           // application ID
  RequestType.PROCESS,     // request type
  encryptedPayload,        // encrypted payload
  depositAmount,           // deposit in wei (bigint)
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

The transaction value sent to the contract is `depositAmount + maxFee`.

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

To deposit funds into your application account, submit a `PROCESS` request with a `depositAmount`:

```typescript
const receipt = await client.submitRequestAndWaitForRequestId(
  1,
  applicationId,
  RequestType.PROCESS,
  encryptedPayload,      // can be an empty encrypted payload
  parseEther("1.0"),     // deposit 1 ETH
  maxFee
);
```

The smart contract holds the deposit. The WASM application's `deposit()` function credits the user's internal account.

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
  console.log("Status:", result.status);         // 0 = success
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
} from "horizen-cce-common-ts";

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

`fetchAndDecryptUserEvents` handles pagination automatically — it fetches pages of events from the subgraph, attempts decryption on each, and accumulates results until the limit is reached.

### Subgraph Types

```typescript
interface UserEvent {
  applicationId: number;
  requestId: string;
  eventSubType: string;
  encryptedData: Uint8Array;  // raw encrypted bytes
  blockNumber: number;
  logIndex: number;
  sortKey: bigint;            // for pagination cursor
}

interface RequestCompleted {
  requestId: string;
  status: number;
  errorCode: number;
  errorMessage: string;
  applicationFees: bigint;
  blockNumber: number;
}
```
--

## Low-Level Encryption / Decryption

For cases where you need direct control over encryption (e.g., encrypting a deanonymization report, or working outside the `HorizenCCEClient`):

```typescript
import {
  encrypt,
  decrypt,
  importPublicKeyFromHex,
  stringToBytes,
  bytesToString,
} from "horizen-cce-common-ts";

// Encrypt a message for a known public key
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

---

## Utility Functions

```typescript
import {
  hexToBytes,     // "0xabcd" → Uint8Array
  bytesToHex,     // Uint8Array → "abcd" (no 0x prefix)
  stringToBytes,  // "hello" → Uint8Array (UTF-8)
  bytesToString,  // Uint8Array → "hello" (UTF-8)
} from "horizen-cce-common-ts";
```

---
