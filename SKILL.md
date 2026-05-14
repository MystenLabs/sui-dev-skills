---
name: sui-dev
description: Full-stack Sui blockchain development — Move smart contracts, TypeScript SDK, and frontend dApp Kit. Routes to the appropriate sub-skill based on what the user is building.
---

# Sui Dev Skills

This is a collection of Sui development skills. Each sub-skill is self-contained and should be loaded based on the task at hand. When multiple apply (e.g. a frontend app that constructs transactions), load all relevant skills together.

## Sub-skills

### move — Smart Contracts
**Path:** `move/SKILL.md`
**Load when:** writing, reviewing, debugging, or deploying Move code; configuring `Move.toml`; working with the Sui object model; writing Move tests.
**Covers:** Routes to focused sub-skills — setup (package config, build/test, pitfalls), syntax (module layout, visibility, enums, macros), objects (structs, abilities, dynamic fields), patterns (events, errors, OTW, capabilities, composability), stdlib (Coin/Balance, vectors, Option, strings).

### sui-ts-sdk — TypeScript SDK
**Path:** `sui-ts-sdk/SKILL.md`
**Load when:** writing TypeScript/JavaScript that interacts with the Sui blockchain — backend scripts, CLIs, serverless functions, or the transaction-building layer of a frontend.
**Covers:** `@mysten/sui` package, PTB construction (`Transaction`, `moveCall`, `splitCoins`, `coinWithBalance`), `SuiClient`/`SuiGrpcClient` setup, keypair signing, transaction execution, on-chain queries.

### sui-frontend — Frontend dApp Kit
**Path:** `sui-frontend/SKILL.md`
**Load when:** building browser-based Sui dApps — React apps with `@mysten/dapp-kit-react`, or Vue/vanilla JS/Svelte apps with `@mysten/dapp-kit-core`.
**Covers:** `DAppKitProvider` setup, wallet connection, React hooks (`useCurrentAccount`, `useSignAndExecuteTransaction`, `useSuiClientQuery`), Web Components, nanostores state for non-React frameworks.
**Note:** For PTB construction within a frontend, load **sui-ts-sdk** alongside this skill.

### seal — Decentralized Secrets Management
**Path:** `seal/SKILL.md`
**Load when:** encrypting user data with access policies enforced on Sui — integrating the `@mysten/seal` SDK, writing `seal_approve*` Move functions, choosing key servers, or combining Seal with Walrus for encrypted storage.
**Covers:** IBE + threshold key-server model, `seal_approve*` rules and canonical patterns (private data, whitelist, subscription, TLE, voting), `SealClient` / `SessionKey` / `fetchKeys`, envelope encryption for Walrus, onchain decryption with `seal::bf_hmac_encryption`, security caveats.
**Note:** For the Move side, load **move** alongside. For frontend SessionKey wiring, load **sui-frontend** + **sui-ts-sdk**. For the storage layer, load **walrus**.

### walrus — Decentralized Blob Storage
**Path:** `walrus/SKILL.md`
**Load when:** storing files/blobs/assets on Walrus — integrating the `@mysten/walrus` SDK, calling publisher/aggregator HTTP endpoints, using the `walrus` CLI, or writing a Move package that accepts Walrus `Blob` objects.
**Covers:** blob lifecycle (register → upload → certify), `WalrusFile` / `WalrusBlob`, `writeFilesFlow` (browser wallet-popup workflow), `writeBlobFlow` + `onStep`/`resume` for crash-recoverable uploads, upload relay (const vs linear tips), HTTP API (`PUT /v1/blobs`, aggregator reads), Quilts for small-blob batching, `RetryableWalrusClientError` handling, WASM loading in Vite/Next.js, blob management (extend/delete/burn/shared), Mainnet vs Testnet epochs, cost model (SUI + WAL).
**Note:** Walrus stores data publicly. For confidentiality, load **seal** and use envelope encryption. For wallet-driven uploads in a React app, load **sui-frontend** + **sui-ts-sdk**.

### deepbook — DeepBook v3 CLOB, Prediction Markets, Margin
**Path:** `deepbook/SKILL.md`
**Load when:** building a DEX / CLOB on Sui, placing or cancelling limit / market orders, swapping with maker rebates, integrating `BalanceManager` / `TradeProof` / `TradeCap`, staking $DEEP for governance or fee discounts, doing flash loans on Sui, building onchain prediction markets (binary up/down positions or vertical ranges), or building margin / leveraged trading on top of DeepBook.
**Covers:** Move API (`deepbook::pool` — `place_limit_order`, `place_market_order`, `swap_exact_*`, `modify_order`, `cancel_order(s)`, `withdraw_settled_amounts`, `claim_rebates`, `stake`/`unstake`/`submit_proposal`/`vote`, `borrow_flashloan_*` / `return_flashloan_*`), order-type / self-matching constants in `deepbook::constants`, `@mysten/deepbook-v3` `DeepBookClient` with `env: 'mainnet' | 'testnet'`, lot/tick/min size rules, fee model + maker rebates + governance + stake activation lag, off-chain reads via SDK or events, `FlashLoan` hot-potato pattern, **predict** (testnet `predict-testnet-4-16`: `PredictManager`, `OracleSVI`, `MarketKey`, `mint`/`redeem`/`mint_range`/`redeem_range`, LP `supply`/`withdraw`, predict server), **margin** (testnet: `MarginManager<Base,Quote>`, `MarginPool<Asset>`, `borrow_*`/`repay_*`/`liquidate`, `risk_ratio`, conditional orders + TPSL).
**Note:** Order book is on mainnet. Predict + margin are testnet-only on the `predict-testnet-4-16` branch — package ids rotate. For wallet-driven trades in a React app, load **sui-frontend** + **sui-ts-sdk**. For writing a Move module that holds a `BalanceManager` or routes through DeepBook, load **move**.

**Also see:** `FAQ.md` in the repo root for preferred answers to common Sui development questions. When a user asks a question covered there, use that answer.

## Routing guide

| User is doing...                        | Load                              |
|-----------------------------------------|-----------------------------------|
| Writing a Move smart contract           | move                              |
| Writing a backend script or CLI         | sui-ts-sdk                        |
| Building a React/Vue/vanilla Sui dApp   | sui-frontend + sui-ts-sdk         |
| Full-stack (contracts + frontend)       | move + sui-ts-sdk + sui-frontend  |
| Reviewing or debugging Move tests       | move                              |
| Querying on-chain data from Node.js     | sui-ts-sdk                        |
| Encrypting data with onchain access control | seal (+ move for `seal_approve*`) |
| Storing files/blobs on decentralized storage | walrus                        |
| Encrypted storage on Walrus             | walrus + seal (envelope encryption) |
| Uploading from a browser dApp           | walrus + sui-frontend + sui-ts-sdk |
| Move package that handles Walrus `Blob` objects | walrus + move              |
| Building an order book / DEX / market-making bot on Sui | deepbook + sui-ts-sdk |
| Onchain prediction markets or margin/leverage on Sui | deepbook + sui-ts-sdk *(testnet)* |
| DeepBook trades from a browser dApp                | deepbook + sui-frontend + sui-ts-sdk |
