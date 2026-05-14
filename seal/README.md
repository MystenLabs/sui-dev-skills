# Seal Skill

A Claude skill for integrating [Seal](https://seal-docs.wal.app) — Mysten Labs' decentralized secrets management service — into Sui apps. Covers both the Move side (`seal_approve*` access policies) and the `@mysten/seal` TypeScript SDK.

## What's covered

- Mental model: identity-based encryption + threshold key servers + Move-defined policies
- Move side:
  - `seal_approve*` function shape and rules (`entry`, `id: vector<u8>` first, must abort, no side effects, no `Random`)
  - Key-id design (package-id prefix, policy-object prefix, nonces)
  - Five canonical patterns: private data, whitelist, subscription, time-lock, voting
- TypeScript SDK:
  - `SealClient` setup, key-server selection, `verifyKeyServers`, API-key auth
  - `encrypt` with threshold/package-id/identity
  - `SessionKey` flow (personal-message signing, scoping, persistence)
  - `decrypt` and batched `fetchKeys`
  - Error handling (`NoAccessError`, `InvalidParameter`, retryable errors)
- Envelope encryption — the correct pattern for Walrus-backed storage
- Onchain decryption via `seal::bf_hmac_encryption` (voting, sealed-bid auctions)
- Security best practices: threshold choice, operator vetting, what Seal is NOT for
- Performance checklist (caching, `fetchKeys`, AES vs HMAC-CTR)
- `seal-cli` for debugging
- Anti-patterns

## Relationship with other skills

- **move** — for writing the `seal_approve*` Move module (object model, abilities, testing).
- **sui-ts-sdk** — for building the PTBs that the Seal SDK submits to key servers.
- **sui-frontend** — for wiring wallet-based `SessionKey` signing into React / dApp Kit apps.
- **walrus** — Seal's ciphertext is storage-agnostic, but envelope encryption + Walrus is the canonical pairing. The Walrus skill covers upload flows, Quilts, and the `@mysten/walrus` SDK.

## Usage

Reference `SKILL.md` in your project's `CLAUDE.md` — see the [root README](../README.md) for install options.
