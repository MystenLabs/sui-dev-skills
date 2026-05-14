---
name: seal
description: Seal — decentralized secrets management on Sui. Identity-based encryption with onchain access policies enforced by threshold key servers. Use when building apps that encrypt user data (stored on Walrus, Sui objects, or off-chain) and gate decryption on Move-defined policies. Covers the Move side (`seal_approve*` patterns) and the `@mysten/seal` TypeScript SDK.
---

# Seal Skill

You are integrating [Seal](https://seal-docs.wal.app) — Mysten Labs' decentralized secrets management service. Seal encrypts data client-side with **identity-based encryption (IBE)** and delegates decryption-key release to a **threshold of off-chain key servers** that evaluate **onchain access policies** written in Move. Storage is orthogonal: ciphertext lives wherever you want (Walrus, Sui objects, S3). Follow these rules precisely.

> **Not for**: wallet private keys, regulated PHI, classified/government secrets. Seal's trust model assumes `t-of-n` key servers are honest — it is not suitable when a single compromised server must never leak plaintext. See the Security section.

This skill covers both sides of a Seal integration: the Move package that defines who can decrypt, and the TypeScript SDK that performs encryption and decryption.

---

## 1. Mental model (read first)

Seal is not "encrypt with a key and store the key somewhere." The flow is:

1. Your Move package defines one or more `seal_approve*` entry functions. Each takes an identity `id: vector<u8>` plus any objects it needs, and **aborts** if access is denied.
2. You **encrypt** data client-side against a chosen identity `id` and a set of key servers with a threshold `t`. The SDK produces ciphertext and a symmetric key.
3. To **decrypt**, the client builds a PTB that calls your `seal_approve*` function and asks `t` key servers to release a derived decryption key. Each key server dry-runs the PTB on a full node; if the function does not abort, the server returns its share.
4. The client reconstructs the decryption key from `t` shares and decrypts locally.

Key consequences:

- **Access policy changes take effect immediately** for future decryptions — no re-encryption needed. You just edit your Move package (or the on-chain state it reads).
- **`seal_approve*` is evaluated via `dry_run_transaction_block` on a full node**. It must be side-effect free, cannot use `Random`, and must not rely on just-created objects (propagation delay).
- The identity namespace is scoped by **package ID**: Seal prepends the full package ID to every key-id, so packages cannot forge each other's identities. Within your package, you design the rest of the id bytes.
- **Once a user has fetched a decryption key, it cannot be revoked** — it is a cryptographic object the user holds locally. Plan access-control changes around this.

---

## 2. Install

```bash
npm install @mysten/seal @mysten/sui
```

The SDK depends on `@mysten/sui` (v2+). Imports all come from the package root:

```typescript
import {
  SealClient,
  SessionKey,
  EncryptedObject,
  NoAccessError,
  type KeyServerConfig,
  type SealClientOptions,
  type EncryptOptions,
  type DecryptOptions,
} from '@mysten/seal';
```

There are no subpath exports — do not write `@mysten/seal/client` or similar.

---

## 3. The Move side: `seal_approve*` functions

### 3.1 Function shape

Every access policy is a Move function whose name starts with `seal_approve`. Hard rules:

- **Must be `entry`**, not `public` and not `public entry`. (Public would lock the signature against future SDK changes.)
- **First parameter must be `id: vector<u8>`** — the identity the client is decrypting against.
- **Must abort when access is denied**; must not return a value. Return is `()`.
- **Must be side-effect free**: no `transfer`, no mutating shared state, no event emission. The function is evaluated via `dry_run_transaction_block` — effects would be discarded anyway, and key servers may reject functions that appear to mutate.
- **Must not use `sui::random::Random`**: its output is non-deterministic across dry-runs on different servers and a leaked random value can defeat policy.
- The `Clock` object is fine for time checks.

```move
module my_pkg::policies;

const ENoAccess: u64 = 0;

// ✅ Correct shape.
entry fun seal_approve(id: vector<u8>, policy: &MyPolicy, ctx: &TxContext) {
    assert!(check(id, policy, ctx.sender()), ENoAccess);
}

// ❌ Wrong: public means you can never change the signature without breaking callers.
public entry fun seal_approve_bad(id: vector<u8>, /* ... */) { /* ... */ }

// ❌ Wrong: returning a bool — must assert + abort.
entry fun seal_approve_bool(id: vector<u8>): bool { true }
```

### 3.2 Key-id conventions

Seal prepends the **full package id** to the client-supplied `id` bytes when evaluating policies. That is, the effective identity is `[package_id][your_id_bytes]`. Your `seal_approve*` therefore only needs to validate `your_id_bytes`.

A common pattern is `[policy_object_id][nonce]` or `[creator_address][nonce]`. Design the id so that:

- A caller cannot reuse another user's id (prefix with owner / policy object id).
- Multiple encryptions under the same policy are distinguishable (append a nonce).
- The prefix the `seal_approve*` function checks is unforgeable — an attacker encrypting under a different id must be detected.

### 3.3 The five canonical patterns

Reference implementations live at [MystenLabs/seal/move/patterns/sources](https://github.com/MystenLabs/seal/tree/main/move/patterns/sources). Use them as starting points:

| Pattern | File | Use case |
|---|---|---|
| **Private data** | `private_data.move` | Only the object's current owner can decrypt. Personal storage, private NFTs. |
| **Whitelist / Allowlist** | `whitelist.move` | An admin maintains a `Table<address, bool>`; anyone on it can decrypt. Subscriptions, early access. |
| **Subscription** | `subscription.move` | Users mint a time-limited pass; decryption allowed while pass is valid. Paid media, SaaS. |
| **Time-lock encryption (TLE)** | `tle.move` | Identity encodes a timestamp; anyone can decrypt once `clock.timestamp_ms() >= t`. Sealed-bid auctions, scheduled reveals. |
| **Voting** | `voting.move` | Encrypt ballots, reveal via **onchain decryption** once voting ends. Pairs with `seal::bf_hmac_encryption`. |

Whitelist / private-data patterns both enforce a **key-id prefix check**:

```move
// whitelist.move (simplified): id must start with the whitelist's object id.
fun check_policy(id: vector<u8>, wl: &Whitelist, caller: address): bool {
    // 1. The id's prefix must equal this whitelist's object id — otherwise the caller
    //    could try to use this approve function to unlock a *different* whitelist's data.
    let wl_id = object::id(wl).to_bytes();
    if (!has_prefix(&id, &wl_id)) return false;
    // 2. The caller must be on the list.
    wl.addresses.contains(caller)
}
```

### 3.4 Testing policies

`seal_approve*` functions are ordinary Move entry functions — test them with the standard Move test framework. Dry-run them via `sui client dry-run` to confirm behavior against real on-chain state before shipping.

---

## 4. The TypeScript side

### 4.1 `SealClient` setup

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { SealClient } from '@mysten/seal';

const suiClient = new SuiGrpcClient({ url: 'https://fullnode.testnet.sui.io:443' });

const seal = new SealClient({
  suiClient,
  serverConfigs: [
    { objectId: '0x...mysten-server-1', weight: 1 },
    { objectId: '0x...ruby-nodes',      weight: 1 },
    { objectId: '0x...nodeinfra',       weight: 1 },
  ],
  verifyKeyServers: false, // see below
});
```

**Picking servers.** Seal is permissionless; anyone can run a key server. The verified-provider list (Testnet and Mainnet) lives at the [Pricing page](https://seal-docs.wal.app/Pricing/). You choose the set and the threshold `t`. **Changing the server set requires re-encrypting the data** (different public keys) — use envelope encryption (§5) if you need to rotate without re-encrypting blobs.

**`verifyKeyServers`**. Anyone can register an on-chain `KeyServer` object with any URL. Setting `verifyKeyServers: true` makes the SDK fetch `/v1/service` on each server at startup and confirm the object id matches. Do this **once at startup in long-lived services**; in short-lived contexts (serverless, frontend) prefer `false` for latency, and verify out-of-band.

**Weights.** `weight` biases which servers are asked first but every server in the set is a valid source. For decentralized (committee) key servers, you pass the single committee object id and one `aggregatorUrl`.

**API keys.** Some permissioned servers require an HTTP header:

```typescript
{ objectId: '0x...', weight: 1, apiKeyName: 'x-api-key', apiKey: process.env.SEAL_API_KEY }
```

### 4.2 Encryption

```typescript
const { encryptedObject, key } = await seal.encrypt({
  threshold: 2,                 // t in t-of-n
  packageId: '0x...my_pkg',     // the package that owns the seal_approve* functions
  id: keyIdBytes,               // Uint8Array — the policy-specific part of the identity
  data: new TextEncoder().encode('hello'),
});
```

- `encryptedObject` is a `Uint8Array` — store it wherever (Walrus is the obvious fit).
- `key` is the symmetric key used under the hood. **Discard it** unless you have an explicit disaster-recovery reason. If you keep it, anyone who gets it decrypts the data with no policy check.
- `encrypt` does **not** hide the message size. Pad if size is sensitive.
- To inspect a stored blob later: `EncryptedObject.parse(bytes)` returns the threshold, package id, id, and ciphertext metadata.

### 4.3 Decryption: `SessionKey`

Decryption requires a `SessionKey` — a short-lived ephemeral keypair the user signs once to authorize the SDK to fetch multiple decryption keys without re-prompting.

```typescript
import { SessionKey } from '@mysten/seal';

const session = await SessionKey.create({
  address: currentAccount.address,
  packageId: '0x...my_pkg',    // session is scoped to a single package
  ttlMin: 10,                   // minutes; max is set by the key servers
  suiClient,
});

// Ask the wallet to sign the session's personal message.
const { signature } = await signPersonalMessage({ message: session.getPersonalMessage() });
await session.setPersonalMessageSignature(signature);
```

Notes:
- The session is **scoped to one `packageId`**. You cannot reuse it against another package's `seal_approve*`.
- You can pass a `Signer` in the constructor (backend/CLI flows) to skip the personal-message round-trip.
- `SessionKey` is serializable via `session.export()` and can be persisted (e.g., IndexedDB) across tabs.

Once the session is initialized, build the PTB that calls your policy and decrypt:

```typescript
import { Transaction } from '@mysten/sui/transactions';

const tx = new Transaction();
tx.moveCall({
  target: `${PKG}::policies::seal_approve`,
  arguments: [tx.pure.vector('u8', keyIdBytes), tx.object(policyObjectId)],
});

const plaintext = await seal.decrypt({
  data: encryptedObject,
  sessionKey: session,
  txBytes: await tx.build({ client: suiClient, onlyTransactionKind: true }),
});
```

Critical:
- **All `moveCall`s in the PTB must target the same package id** as the session.
- Inside `seal_approve*`, `TxContext::sender()` returns the session key's address, not a wallet address — design policies accordingly.
- **Fully specify objects in the PTB** (include versions via `tx.objectRef(...)` when you have them). Key servers dry-run the PTB and freshly-created or just-mutated objects may not be visible yet.

### 4.4 Batch decryption: `fetchKeys`

If you need to decrypt many blobs encrypted under different identities, do **not** call `decrypt` in a loop — that's `N × t` key-server requests. Instead, build one PTB with multiple `seal_approve*` calls and use `fetchKeys`:

```typescript
const tx = new Transaction();
for (const idBytes of allIds) {
  tx.moveCall({
    target: `${PKG}::policies::seal_approve`,
    arguments: [tx.pure.vector('u8', idBytes), tx.object(policyObjectId)],
  });
}
await seal.fetchKeys({
  ids: allIds,
  sessionKey: session,
  txBytes: await tx.build({ client: suiClient, onlyTransactionKind: true }),
  threshold: 2,
});

// Now per-blob decrypts are local — no more key-server traffic.
for (const { encryptedObject } of blobs) {
  const pt = await seal.decrypt({ data: encryptedObject, sessionKey: session, txBytes: /* same PTB */ });
}
```

`SealClient` caches fetched keys internally, so **reuse the same instance** across requests in your app.

### 4.5 Error handling

Common errors:
- `NoAccessError` — the `seal_approve*` function aborted. The user is not authorized.
- `InvalidParameter` — often means an object referenced in the PTB isn't visible to the full node yet. Wait a few seconds and retry.
- `RetryableKeyServerError` — retry on the same or a different server.

Wrap `decrypt` in a retry with bounded backoff for transient errors; don't retry `NoAccessError`.

---

## 5. Envelope encryption (the pattern that matters with Walrus)

`Seal.encrypt` works well for small payloads but has two limits for large/long-lived data:
1. IBE encryption over many MB is slow.
2. To rotate key servers, you must re-encrypt every blob.

**Envelope encryption** (a.k.a. layered / DEK-KEK):

```typescript
// 1. Encrypt the data with a fresh symmetric key (fast, symmetric).
const dek = crypto.getRandomValues(new Uint8Array(32));
const ciphertext = await aesGcmEncrypt(dek, plaintext);   // your preferred AES-GCM helper

// 2. Wrap the DEK with Seal — small payload, policy-gated.
const { encryptedObject: wrappedDek } = await seal.encrypt({
  threshold: 2,
  packageId: PKG,
  id: keyIdBytes,
  data: dek,
});

// 3. Store `ciphertext` on Walrus, store `wrappedDek` alongside (or on Sui).
```

To rotate key servers later: re-encrypt only the small `wrappedDek`; the large `ciphertext` on Walrus stays put. This is the recommended pattern for any Walrus-backed storage.

Default to **AES-256-GCM** for the data-encryption step. Reserve **HMAC-CTR** for onchain decryption (§6) — it is slower and more memory-hungry.

---

## 6. Onchain decryption (voting, sealed-bid auctions)

For use cases where the **Move contract itself** needs to decrypt (e.g., tally encrypted votes on-chain), Seal publishes a companion Move package:

- Testnet: `0x4016869413374eaa71df2a043d1660ed7bc927ab7962831f8b07efbc7efdb2c3`
- Mainnet: `0xcb83a248bda5f7a0a431e6bf9e96d184e604130ec5218696e3f1211113b447b7`

Module: `seal::bf_hmac_encryption`. You must encrypt with **HMAC-CTR** (not AES) for the ciphertext to be decryptable on-chain.

Three-step flow:
1. **Initialization** — fetch key-server public keys via `client.getPublicKeys`, convert with `bf_hmac_encryption::new_public_key`, and store them as a shared object.
2. **Derived-key delivery** — client calls `client.getDerivedKeys` against the post-reveal condition and submits the keys + proofs to the contract, which verifies them via `verify_derived_keys`.
3. **Decryption** — contract calls `bf_hmac_encryption::decrypt(ciphertext, derived_keys, public_keys)` which returns `Option<vector<u8>>`.

Only reach for this when plaintext truly needs to be visible on-chain. For everything else, decrypt client-side.

---

## 7. Security best practices

1. **Threshold choice**. Pick `t` to balance availability vs. confidentiality. `2-of-3` tolerates one offline server; `3-of-5` tolerates two. A too-high `t` risks **permanent data loss** if servers disappear — not a hypothetical, because policies are the only source of truth for access but the servers are the only source of keys.

2. **Vet operators**. Key servers are permissionless. For production, pick operators with real SLAs, and check whether their full-node dependency is self-hosted, third-party, or public — key fetches can hang if the full node does. Use the curated [verified providers list](https://seal-docs.wal.app/Pricing/) as a starting point.

3. **Client-side decryption is final**. Once the SDK hands the plaintext to your app, Seal has no visibility into what happens next. A compromised frontend or a user copy-pasting the plaintext is out of scope. Add **application-level audit logging** if you need a trail — key servers do not log deliveries on-chain.

4. **Don't use Seal for**:
   - Wallet private keys, seed phrases, or anything whose compromise is unrecoverable.
   - Regulated PHI, PII under HIPAA/GDPR Article 9, or anything requiring certified HSM-level guarantees.
   - Secrets that must survive a compromise of `t` operators colluding.

5. **Symmetric key returned by `encrypt`**. The `key` field is the data-encryption key. If you persist it for disaster recovery, treat it like a root secret. Default: drop it.

6. **Package upgrades are visible on-chain**. If your `seal_approve*` policy changes via a package upgrade, that change is public. Design around it: don't rely on policies you'd be embarrassed to publish.

---

## 8. Performance checklist

- [ ] Reuse a single `SealClient` instance — it caches derived keys per session.
- [ ] Reuse a single `SessionKey` for the user's session (persist in IndexedDB for multi-tab).
- [ ] Use `fetchKeys` with a multi-call PTB when decrypting many blobs.
- [ ] `verifyKeyServers: false` in hot paths; verify out-of-band.
- [ ] Always include full object refs (with version) in PTBs submitted to key servers.
- [ ] Use AES-256-GCM (default), not HMAC-CTR, unless you need onchain decryption.
- [ ] Use envelope encryption for payloads >100 KB or for Walrus-backed storage.

---

## 9. Seal CLI (one-off ops, key server operators)

`seal-cli` (from the `MystenLabs/seal` Rust crate) is useful for debugging and key-server operators — not typically called from app code. Common commands:

- `seal-cli genkey` — produce a masterkey / publickey pair.
- `seal-cli encrypt-aes --threshold N --public-keys ... --package-id ... --id ...`
- `seal-cli parse <hex>` — decode an `EncryptedObject` for inspection. This is the easiest way to verify what a blob on Walrus was encrypted against.
- `seal-cli fetch-keys` — exercise a live key server from the command line.

Encryption is randomized: re-running `encrypt-aes` on the same input produces different ciphertext. That is expected, not a bug.

---

## 10. Routing: what else to load

| Task | Also load |
|---|---|
| Writing the `seal_approve*` Move module | `move` skill (especially `objects`, `patterns`) |
| Building PTBs that call `seal_approve*` for decryption | `sui-ts-sdk` skill |
| Wiring encryption/decryption into a React dApp | `sui-frontend` + `sui-ts-sdk` |
| Storing the ciphertext on Walrus | `walrus` skill (envelope-encryption pattern is canonical there) |

---

## 11. Anti-patterns to avoid

- ❌ `public entry fun seal_approve` — use `entry`, not `public entry`. Locks the signature.
- ❌ `seal_approve` that returns `bool` — must abort, not return.
- ❌ Using `sui::random` inside `seal_approve*` — non-deterministic across servers, may also be insecure in a dry-run context.
- ❌ Calling `transfer::*`, emitting events, or writing to shared state inside `seal_approve*`.
- ❌ Encrypting a large blob directly with `seal.encrypt` instead of envelope-encrypting it.
- ❌ Calling `seal.decrypt` in a loop — use `fetchKeys` with a batched PTB.
- ❌ Re-creating `SealClient` / `SessionKey` per request — destroys caching.
- ❌ Trusting a `KeyServer` object id without verifying the URL → public-key binding (set `verifyKeyServers: true` during setup at least once).
- ❌ Assuming access can be revoked after the key is fetched — it cannot.
- ❌ Storing Seal's returned symmetric `key` without a concrete recovery reason.
- ❌ Using HMAC-CTR for app data instead of AES — HMAC-CTR is reserved for onchain decryption.
