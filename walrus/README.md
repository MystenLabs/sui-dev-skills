# Walrus Skill

A Claude skill for integrating [Walrus](https://docs.wal.app) — Mysten Labs' decentralized blob storage coordinated by Sui — into Sui apps. Covers the `@mysten/walrus` TypeScript SDK, the publisher/aggregator HTTP API, the `walrus` CLI, and the Move side.

## What's covered

- **Mental model**: blobs as immutable byte arrays; register → upload → certify lifecycle; `BlobId` as content hash; epochs (Testnet 1d / Mainnet 14d); 13.6 GiB max; ~4.5× RAM overhead; **all blobs are public**.
- **Integration paths**: HTTP gateway (publisher/aggregator) vs. SDK+relay vs. SDK direct vs. CLI — which to pick and why.
- **HTTP API**: `PUT /v1/blobs` with `epochs`, `deletable`, `permanent`, `send_object_to`; `GET /v1/blobs/:id` and `/by-object-id/:id`; Quilt endpoints; response shapes (`newlyCreated` vs `alreadyCertified`).
- **TypeScript SDK**:
  - Setup: `SuiGrpcClient.$extend(walrus(...))`, package config, fetch customization.
  - `WalrusFile` — `getFiles`, `writeFiles`, `bytes()/text()/json()`, identifiers & tags.
  - `WalrusBlob` — read a Quilt, query files by id/identifier/tag.
  - `writeFilesFlow` — the browser popup-blocker workaround (split register + certify across clicks).
  - `writeBlobFlow` / `onStep` / `resume` — crash-recoverable uploads.
  - `readBlob` / `writeBlob` — raw bytes.
  - Upload relay config (`sendTip`: const vs linear).
  - Error handling (`RetryableWalrusClientError` + `client.walrus.reset()`, `onError` debug hook).
  - WASM loading (Vite `?url`, CDN fallback, Next.js `serverExternalPackages`).
- **Quilts**: when to use (many small blobs), limits (666 per quilt, ~4 GiB per file), cost savings.
- **Blob management**: `extend`, `delete`, `burn-blobs`, `set-blob-attribute`, shared blobs.
- **Security**: Walrus = availability + integrity, **not confidentiality**. Envelope-encrypt with Seal.
- **Large uploads**: memory budgeting, persistence between steps, rate-ramping.
- **Networks**: Mainnet vs Testnet, `walrus get-wal`.
- **Move integration**: accepting `Blob` objects on-chain, `wrapped_blob.move` pattern.
- **Anti-patterns**: direct SDK in browsers, single event handler for both sigs, deletion-as-privacy, hardcoded public gateways, retries without `reset()`, and more.

## Relationship with other skills

- **seal** — the canonical pairing for confidential data. Walrus stores ciphertext; Seal gates the decryption key via on-chain policy.
- **sui-ts-sdk** — the transactions Walrus generates (register / certify) are ordinary Sui PTBs; wallet signing and execution follow the patterns from this skill.
- **sui-frontend** — for wiring `writeFilesFlow` into React/dApp Kit with proper wallet UX.
- **move** — for Move modules that accept, store, or verify `Blob` Sui objects.

## Usage

Reference `SKILL.md` in your project's `CLAUDE.md` — see the [root README](../README.md) for install options.
