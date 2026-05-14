---
name: walrus
description: Walrus — decentralized blob storage coordinated by Sui. Use when storing files/blobs/assets in a Sui app via the `@mysten/walrus` TypeScript SDK, publisher/aggregator HTTP API, or `walrus` CLI. Covers blob lifecycle (register → upload → certify), Quilts for small blobs, upload relay, crash-recoverable uploads, and the security model (public by default — encrypt with Seal if confidentiality matters).
---

# Walrus Skill

You are integrating [Walrus](https://docs.wal.app) — Mysten Labs' decentralized blob storage layer. Walrus stores immutable byte arrays ("blobs"); Sui handles coordination, payments, metadata, and object ownership. Storage is provided by erasure-coded slivers across ~1000 shards with Byzantine fault tolerance up to 1/3 malicious nodes. Follow these rules precisely.

> **Critical**: All blobs on Walrus are **public and discoverable by anyone**. Deletion does not remove cached copies. If your data is sensitive, **encrypt before uploading** (see the `seal` skill for the canonical pattern).

---

## 1. Mental model (read first)

A successful Walrus upload is **three Sui transactions plus distributed storage work**:

1. **Register** the blob on Sui (reserve storage, mint a `Blob` object).
2. **Upload** slivers (erasure-coded fragments) to ~2/3 of storage nodes — roughly 2200 HTTP requests direct.
3. **Certify** availability on Sui once enough nodes have acknowledged — this is the "Point of Availability" (PoA).

Reading a blob is ~335 requests to reconstruct from slivers. These numbers are why most apps **do not call storage nodes directly** — they use one of:

- **Publishers / Aggregators** (public or self-hosted HTTP gateways) — simplest; a single HTTP PUT/GET.
- **Upload relay** — the SDK writes via one request to a relay that fans out to nodes. Recommended for browsers and mobile.
- **Direct SDK** (`@mysten/walrus`) — full control; you pay the request overhead. Best for backends with custom retry logic.

Blob identity and lifecycle:

- Blobs are identified by a **`BlobId`** (content hash of the encoding). Identical content → identical `BlobId` → no duplicate storage cost.
- Storage is **per epoch**. Testnet epoch = **1 day**; Mainnet epoch = **2 weeks**. Max 53 epochs per registration (~2 years on Mainnet).
- A `Blob` object on Sui carries the `BlobId` plus ownership/deletability flags. Burn the object to reclaim a small SUI rebate; burning does **not** delete storage.
- **Max blob size**: ~13.6 GiB. **Max per-file inside a Quilt**: ~4 GiB.
- **Memory**: uploads need ~2–3× blob size in RAM; reads need ~1.5–2× (erasure-coding overhead).

Two costs on every write: **SUI** (gas for up to 3 transactions) and **WAL** (storage + write fee). Use [costcalculator.wal.app](https://costcalculator.wal.app/) or `walrus info`.

---

## 2. Pick the right integration path

| Path | When | Tradeoffs |
|---|---|---|
| **HTTP API via publisher/aggregator** | Scripts, demos, apps where a trusted gateway is fine | Simplest. Publisher holds the signing keys and pays SUI/WAL, so you pick who you trust. Public publishers on Testnet; self-host for Mainnet. |
| **SDK + upload relay** | Browsers, mobile, any client that can't open thousands of outbound connections | One request per upload. User still pays SUI/WAL directly. Relay may charge a tip. |
| **SDK direct** | Server-side services that want full control, custom retry, or to avoid relay fees | You open ~2200 connections per write. Fine on servers, awful in browsers. |
| **`walrus` CLI** | One-off ops, debugging, scripting from a shell | Not an integration path — a tool. |

Default recommendation for a Sui dApp that lets users upload: **SDK + upload relay**. For read-only browsing: **aggregator HTTP API**.

---

## 3. HTTP API (publishers & aggregators)

### 3.1 Store a blob (publisher)

```bash
# Raw bytes inline
curl -X PUT "$PUBLISHER/v1/blobs" -d "some string"

# File upload
curl -X PUT "$PUBLISHER/v1/blobs?epochs=5" --upload-file ./photo.jpg

# Permanent (non-deletable), transfer Blob object to an address
curl -X PUT "$PUBLISHER/v1/blobs?epochs=10&permanent=true&send_object_to=0xabc..." --upload-file ./archive.tar
```

Query parameters:

| Param | Default | Meaning |
|---|---|---|
| `epochs` | `1` | Number of epochs to store. |
| `deletable` | `true` | Mints a deletable blob (owner can delete early). |
| `permanent` | `false` | Mints a permanent blob. Mutually exclusive with `deletable=true`. |
| `send_object_to` | caller | Recipient address for the `Blob` Sui object. |

Response shape: `{ newlyCreated: { blobObject, ... } }` for new blobs, or `{ alreadyCertified: { blobId, endEpoch, ... } }` when a certified blob with the same id already exists. **Always handle both cases.**

### 3.2 Read a blob (aggregator)

```bash
# By BlobId
curl "$AGGREGATOR/v1/blobs/$BLOB_ID" -o file.bin

# By Sui object id — lets you set HTTP headers via blob attributes
curl "$AGGREGATOR/v1/blobs/by-object-id/$OBJECT_ID"
```

Aggregators limit content-type sniffing to prevent scripts/stylesheets from being served with inferred MIME types. Set headers via `walrus set-blob-attribute` (key-value pairs like `content-type`, `content-disposition`) — they are returned when reading by object id.

For v1.35+: add `strict_consistency_check=true` to verify slivers cryptographically. For v1.36+ with a trusted uploader: `skip_consistency_check=true` for a speedup.

### 3.3 Quilt HTTP

Store many small blobs as one Quilt; retrieve individual files by identifier or by `QuiltPatchId`. The aggregator returns raw bytes with metadata in HTTP headers. Identifiers must be unique within a quilt and must not start with `_` (reserved for Walrus metadata).

### 3.4 Public endpoints

Do **not** hardcode specific public endpoints in production code — they rotate. Current lists:
- [Public aggregators & publishers](https://docs.wal.app/system-overview/public-aggregators-and-publishers)
- Self-host in production via the operator guide.

---

## 4. TypeScript SDK: `@mysten/walrus`

### 4.1 Install & initialize

```bash
npm install @mysten/walrus @mysten/sui
```

The SDK extends a `SuiGrpcClient`:

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { walrus } from '@mysten/walrus';

const client = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
}).$extend(walrus());

// Now: client.walrus.getFiles({ ... }), client.walrus.writeFiles({ ... }), etc.
```

- `walrus()` with no args uses the built-in Testnet package config. For Mainnet or a pinned deployment, import `MAINNET_WALRUS_PACKAGE_CONFIG` / `TESTNET_WALRUS_PACKAGE_CONFIG` or pass `packageConfig: { systemObjectId, stakingPoolId }`.
- All Walrus methods live under the `.walrus` namespace on the extended client.
- Imports come from the package root — no subpaths.

Useful exports:
```typescript
import {
  walrus, WalrusClient,
  WalrusFile, WalrusBlob,
  RetryableWalrusClientError,
  blobIdFromInt, blobIdToInt,
  encodeQuilt,
  TESTNET_WALRUS_PACKAGE_CONFIG, MAINNET_WALRUS_PACKAGE_CONFIG,
} from '@mysten/walrus';
```

### 4.2 `WalrusFile` — the high-level API

**Read** (works for both plain blobs and files inside a Quilt — always batch):

```typescript
const [readme, logo] = await client.walrus.getFiles({ ids: [blobIdOrQuiltId1, id2] });

const bytes: Uint8Array = await readme.bytes();
const text:  string     = await readme.text();
const json:  unknown    = await logo.json();

const identifier: string | null        = await readme.getIdentifier();
const tags:       Record<string,string> = await readme.getTags();
```

**Write** (files are bundled into a single Quilt by the SDK):

```typescript
const file = WalrusFile.from({
  contents: new TextEncoder().encode('Hello from Walrus\n'),
  identifier: 'README.md',
  tags: { 'content-type': 'text/plain' },
});

const results = await client.walrus.writeFiles({
  files: [file],
  epochs: 3,
  deletable: true,
  signer: keypair, // must own enough SUI for tx + WAL for storage
});
// results: Array<{ id: string; blobId: string; blobObject }>
```

The Quilt encoding is inefficient for a single file — prefer writing multiple files together. For a single file, consider `writeBlob` (§4.5) instead.

### 4.3 `WalrusBlob` — read a Quilt and query its contents

```typescript
const blob = await client.walrus.getBlob({ blobId });
const all = await blob.files();
const [readme] = await blob.files({ identifiers: ['README.md'] });
const textFiles = await blob.files({ tags: [{ 'content-type': 'text/plain' }] });
```

### 4.4 `writeFilesFlow` — the **browser workflow** (important)

Browsers block wallet-popup prompts that aren't in direct response to a click. A Walrus write needs **two** signatures (register + certify), so you must split them into separate click handlers. Use `writeFilesFlow`:

```typescript
// On file-select: start encoding immediately.
const flow = client.walrus.writeFilesFlow({
  files: [WalrusFile.from({ contents: fileBytes, identifier: 'upload.bin' })],
});
await flow.encode();

// Click 1 — register + upload slivers
async function handleRegister() {
  const registerTx = flow.register({
    epochs: 3,
    owner: currentAccount.address,
    deletable: true,
  });
  const result = await signAndExecuteTransaction({ transaction: registerTx });
  if (result.$kind === 'FailedTransaction') throw new Error(result.FailedTransaction.status.error?.message);
  await flow.upload({ digest: result.Transaction.digest });
}

// Click 2 — certify
async function handleCertify() {
  const certifyTx = flow.certify();
  const result = await signAndExecuteTransaction({ transaction: certifyTx });
  if (result.$kind === 'FailedTransaction') throw new Error(result.FailedTransaction.status.error?.message);
  const files = await flow.listFiles(); // [{ id, blobId, blobObject }]
}
```

Always check `result.$kind === 'FailedTransaction'` after each signed step. Don't collapse both signatures into one handler — the second popup will be blocked in Safari and some Chrome profiles.

### 4.5 `writeBlob` / `writeBlobFlow` — raw bytes, with crash recovery

For a single blob without Quilt packaging:

```typescript
const { blobId } = await client.walrus.writeBlob({
  blob: fileBytes,
  deletable: false,
  epochs: 10,
  signer: keypair,
});
```

For resumable uploads (long-running backends, unreliable networks):

```typescript
const saved = await db.load(fileId); // undefined on first attempt

const { blobId } = await client.walrus.writeBlob({
  blob: fileBytes,
  epochs: 3,
  deletable: true,
  signer: keypair,
  onStep: (step) => db.save(fileId, step), // persist each step
  resume: saved,                            // skip completed steps on retry
});
```

Or use the async-iterator flow when you want per-step control:

```typescript
const flow = client.walrus.writeBlobFlow({ blob: fileBytes });
for await (const step of flow.run({ signer, epochs: 3, deletable: true })) {
  await db.save(fileId, step);
}
```

`flow.executeRegister(...)` and `flow.executeCertify(...)` are the typed step-by-step equivalents.

### 4.6 `readBlob` — raw bytes

```typescript
const bytes = await client.walrus.readBlob({ blobId });
```

`readBlob` already handles storage-node failures and retries internally. Implement additional retry only for `RetryableWalrusClientError` (§4.8).

---

## 5. Upload relay

When to use: **browsers, mobile, any client that cannot comfortably open thousands of outbound HTTP requests.** Without a relay the SDK opens ~2200 connections to storage nodes per write.

Configure at `walrus({})` init:

```typescript
const client = new SuiGrpcClient({ network: 'testnet', baseUrl: '...' }).$extend(
  walrus({
    uploadRelay: {
      host: 'https://upload-relay.testnet.walrus.space',
      sendTip: { max: 1_000 }, // max MIST per blob; SDK auto-discovers actual tip
    },
  }),
);
```

For explicit tip modes instead of auto-discovery:

```typescript
// Constant tip — flat MIST per blob.
sendTip: { address: '0x...', kind: { const: 105 } }

// Linear tip — base + per-encoded-KiB.
sendTip: { address: '0x...', kind: { linear: { base: 105, perEncodedKib: 10 } } }
```

The relay fetches the tip config from `$host/v1/tip-config`. Failures manifest as `NotEnoughBlobConfirmationsError`.

---

## 6. Quilts — the pattern for many small blobs

If you are storing lots of files under ~1 MiB each (NFT thumbnails, avatars, small JSON), use a Quilt, not individual blobs. A Quilt packs up to **666 blobs (QuiltV1)** into one storage unit and amortizes the fixed metadata cost. Reported savings: ~409× for 10 KiB files and ~238× on Sui fees vs. individual blobs.

Tradeoffs:
- Individual files cannot be deleted, extended, or shared separately — those operations apply to the whole Quilt.
- Files are identified by `QuiltPatchId`, which changes if moved to another quilt.
- Identifiers must be unique within a Quilt; `_`-prefixed names are reserved.

`writeFiles` already packs into a Quilt. To encode one offline:

```typescript
import { encodeQuilt } from '@mysten/walrus';
```

---

## 7. Managing blobs after upload

### Via the `walrus` CLI

- `walrus store file.txt --epochs 2 --context testnet`
- `walrus read <BLOB_ID> --out file.txt`
- `walrus extend --blob-obj-id <ID> --epochs-ahead 10` — extends a blob's life. Anyone can extend a **shared** blob; only the owner can extend an owned one.
- `walrus delete --blob-id <BLOB_ID>` — only the owner, only if `deletable`, does not remove cached copies (blob is public).
- `walrus burn-blobs --blob-obj-id <ID>` — destroys the Sui object (no storage refund); you lose the ability to delete/extend but free some Sui storage rebate.
- `walrus set-blob-attribute --blob-obj-id <ID> --attr content-type text/html` — set HTTP headers for aggregator responses.

### Shared blobs

A **shared blob** is a Walrus `Blob` object wrapped into a Sui shared object. Shared blobs must be **permanent** (cannot be deleted before expiry), but anyone can fund extending them. Use for community-maintained assets.

### `walrus info`

Print live network state — system object id, current epoch, price per unit, active committee. Always verify integrations against `walrus info` output rather than hardcoded numbers.

---

## 8. Data security (the Seal pairing)

Walrus provides **availability and integrity** (blob bytes returned match what was uploaded, verifiable via content hash). It does **not** provide confidentiality — blobs are public.

For confidentiality, combine with Seal using envelope encryption:

```typescript
// 1. Generate a data-encryption key and encrypt locally with AES-GCM.
const dek = crypto.getRandomValues(new Uint8Array(32));
const ciphertext = await aesGcmEncrypt(dek, plaintext);

// 2. Seal-wrap the DEK against your seal_approve* policy.
const { encryptedObject: wrappedDek } = await seal.encrypt({
  threshold: 2, packageId, id: keyIdBytes, data: dek,
});

// 3. Upload ciphertext + wrappedDek to Walrus as two files in a quilt,
//    or upload ciphertext to Walrus and store wrappedDek on Sui.
await client.walrus.writeFiles({
  files: [
    WalrusFile.from({ contents: ciphertext, identifier: 'content.bin' }),
    WalrusFile.from({ contents: wrappedDek, identifier: 'wrapped.dek' }),
  ],
  epochs: 10, deletable: false, signer,
});
```

Why envelope encryption: you can rotate Seal key servers by re-encrypting only the small DEK — the large ciphertext on Walrus stays put. See the `seal` skill for details.

For TEE-based off-chain computation over Walrus data, see Nautilus (out of scope for this skill).

---

## 9. Error handling

```typescript
import { RetryableWalrusClientError } from '@mysten/walrus';

try {
  await client.walrus.writeBlob({ /* ... */ });
} catch (err) {
  if (err instanceof RetryableWalrusClientError) {
    client.walrus.reset();  // clear cached committee/epoch state
    // retry — most often fires during epoch transitions
  } else {
    throw err;
  }
}
```

`RetryableWalrusClientError` is the critical one — cached committee data is invalidated at epoch boundaries. Retry after `reset()` is not guaranteed to succeed but usually does.

For deeper debugging, surface individual node errors:

```typescript
walrus({
  storageNodeClientOptions: {
    onError: (e) => console.warn('node error:', e),
    timeout: 60_000,
  },
})
```

High-level methods (`readBlob`, `writeBlob`, `writeFiles`) already tolerate some node failures — they only throw when **quorum cannot be reached**. Do not implement loop-retries inside these calls yourself.

---

## 10. WASM loading (Vite / Next.js)

The SDK requires WASM bindings to encode/decode blobs. Node, Bun, and many bundlers work out of the box. When they don't:

**Vite** — import the wasm file as a URL and pass it:
```typescript
import walrusWasmUrl from '@mysten/walrus-wasm/web/walrus_wasm_bg.wasm?url';

walrus({ wasmUrl: walrusWasmUrl })
```

**CDN fallback** (any bundler that can't resolve the wasm path):
```typescript
walrus({ wasmUrl: 'https://unpkg.com/@mysten/walrus-wasm@latest/web/walrus_wasm_bg.wasm' })
```

**Next.js API routes** — exclude from server bundling:
```typescript
// next.config.ts
const nextConfig: NextConfig = {
  serverExternalPackages: ['@mysten/walrus', '@mysten/walrus-wasm'],
};
```

---

## 11. Large uploads (> 1 GiB)

Memory is the main constraint: upload requires **~4.5× blob size** in RAM. Checklist:

- [ ] Confirm host has enough RAM: `blob_size × 4.5 × concurrent_uploads`.
- [ ] Persist `{blobId, txDigest}` between steps; use `onStep`/`resume` for crash recovery.
- [ ] Don't run many large uploads in parallel on one box — ramp traffic up.
- [ ] Estimate cost via `walrus info` or [costcalculator.wal.app](https://costcalculator.wal.app/); Mainnet epoch = 14 days, not 7.
- [ ] For migrations over ~1 TiB, coordinate with the Walrus team on Discord.
- [ ] Availability is only guaranteed **after** the certify tx is on-chain — don't let clients read until then.

---

## 12. Networks

| | Mainnet | Testnet |
|---|---|---|
| Sui network | Mainnet | Testnet |
| Epoch | 14 days | 1 day |
| Max epochs per registration | 53 (~2 years) | 53 (~53 days) |
| Tokens | WAL + SUI (real value) | tWAL + tSUI (no value) |
| Reset? | No | **Can be wiped without warning** |

Get Testnet WAL after funding a Sui Testnet wallet: `walrus get-wal` (default swaps 0.5 SUI → 0.5 WAL).

---

## 13. Move-side integration

`Blob` is a Sui object. Your Move package can accept, store, and verify `Blob` objects — e.g., an NFT module that requires an attached Walrus blob for its metadata. Reference: `wrapped_blob.move` in the Walrus repo. Load the `move` skill for object handling (abilities, transfer, shared objects).

To verify that a given blob's content matches expected bytes on-chain, use the content-addressed `blob_id` field — anyone who has the bytes can recompute the hash and compare.

---

## 14. Routing: what else to load

| Task | Also load |
|---|---|
| Encrypting before upload | `seal` skill |
| Building transactions that handle `Blob` objects | `sui-ts-sdk` |
| React/browser upload UI (wallet popups, state) | `sui-frontend` + `sui-ts-sdk` |
| A Move package that accepts Walrus `Blob` objects | `move` skill |

---

## 15. Anti-patterns to avoid

- ❌ Calling the SDK directly from a browser without an upload relay — ~2200 fanout requests will get you blocked or rate-limited.
- ❌ Doing both `register` and `certify` signing inside a single event handler in a browser — wallet popup blocked on the second.
- ❌ Storing many small files as individual blobs — use a Quilt.
- ❌ Assuming deletion makes a blob private. Walrus blobs are **public forever** once uploaded; copies may be cached indefinitely.
- ❌ Hardcoding public publisher/aggregator URLs in production. They rotate; self-host or configure from env.
- ❌ Retrying a `RetryableWalrusClientError` without calling `client.walrus.reset()` — you'll retry against stale committee data.
- ❌ Wrapping high-level `writeBlob` / `readBlob` in custom retry loops — they already retry. Only retry at the `Retryable*` boundary.
- ❌ Ignoring `result.$kind === 'FailedTransaction'` after signing register/certify — a failed on-chain step silently breaks the flow.
- ❌ Running large parallel uploads without budgeting 4.5× RAM each — OOMs appear as allocation errors.
- ❌ Using `epochs: 1` for production data — Testnet gives you a day; Mainnet two weeks; factor in renewal cost and risk.
- ❌ Forgetting the WASM URL config in Vite or Next.js — produces a runtime error deep inside `encode`. Set `wasmUrl` up front.
- ❌ Calling `writeFiles` with just one file when `writeBlob` would be cheaper — Quilt encoding overhead dominates for single small files.
- ❌ Using Walrus for mutable state. Blobs are immutable; to update, write a new blob and update a pointer stored on Sui.
