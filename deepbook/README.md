# DeepBook Skill

A Claude skill for integrating [DeepBook v3](https://docs.sui.io/onchain-finance/deepbookv3/deepbook) — Mysten Labs' onchain central limit order book on Sui — and the two products built on top of it: prediction markets and margin trading. Covers the Move side (`packages/{deepbook,predict,deepbook_margin}`) and the `@mysten/deepbook-v3` TypeScript SDK.

## What's covered

- **Mental model**: three shared objects (`Pool<Base,Quote>`, `BalanceManager`, `Registry`); Book → State → Vault flow; the `TradeProof` / `TradeCap` authorization pattern; why `$DEEP` exists.
- **BalanceManager**: one-per-user-across-pools, owner vs. trader (1000 `TradeCap` slots), `generate_proof_as_owner` / `generate_proof_as_trader`, common create-and-deposit flow.
- **Move API surface**: `create_permissionless_pool`, `place_limit_order`, `place_market_order`, `swap_exact_base_for_quote` / `_quote_for_base`, `modify_order`, `cancel_order(s)`, `cancel_all_orders`, `withdraw_settled_amounts`, `claim_rebates`, `stake` / `unstake` / `submit_proposal` / `vote`, `borrow_flashloan_*` / `return_flashloan_*`, plus the read-only `get_quote_quantity_out`, `get_level2_range`, `mid_price`, `account`, `locked_balance`.
- **Order type / self-matching constants**: pulled from `deepbook::constants` as `u8`-returning functions (`no_restriction`, `immediate_or_cancel`, `fill_or_kill`, `post_only`, `self_matching_allowed`, `cancel_taker`, `cancel_maker`).
- **TypeScript SDK**: `DeepBookClient` setup, `env: 'mainnet' | 'testnet'`, `deepBook.placeLimitOrder` / `swapExactBaseForQuote` / `cancelOrder` / `claimRebates`, `flashLoans.borrowBaseAsset` / `returnBaseAsset`, `balanceManager.createAndShareBalanceManager` / `depositIntoManager` / `withdrawAllFromManager`.
- **Lot / tick / min size**: pool-defined scales, common `EInvalidLotSize` / `EInvalidTickSize` / `EOrderBelowMinimumSize` failure modes.
- **Fees, rebates, governance**: taker bps by pool type, maker rebate gating on staked DEEP + 28-day phaseout, stake activation lag (epoch `N+1`), governance voting power formula, quorum, `pay_with_deep` vs. whitelisted pools.
- **Off-chain reads**: SDK queries (`getLevel2Range`, `midPrice`, `checkManagerBalance`) plus Sui event streams (`OrderPlaced`, `OrderFilled`, `OrderCanceled`, `BalanceEvent`).
- **Flash loans**: `FlashLoan` hot potato, same-PTB return requirement, arb pattern.
- **Predict (testnet only)**: `Predict` shared object, `OracleSVI`, `PredictManager` wrapping a `BalanceManager`, `MarketKey(oracle_id, expiry, strike, direction)`, `mint` / `redeem` / `mint_range` / `redeem_range`, LP `supply` / `withdraw`, predict-server endpoints, testnet package ids.
- **Margin (testnet only)**: `MarginManager<Base, Quote>` wrapping a `BalanceManager`, `MarginPool<Asset>` for lenders, `borrow_base` / `borrow_quote` / `repay_*` / `liquidate`, `risk_ratio`, conditional orders and TPSL via `pool_proxy.move` / `tpsl.move`.
- **Anti-patterns**: shared `BalanceManager`s, hardcoded package ids, mainnet-only predict calls, raw `u8` constants, missing `pay_with_deep`, float prices in Move, expecting rebates without staking, dropped `FlashLoan` potatoes.

## Status

- **Order book**: mainnet (DeepBook v3 contract id `0x337f4f...` at v6 Jan 2026; 18 supported coins, 24 pools).
- **Predict** + **margin**: testnet only, branch `predict-testnet-4-16`. Package ids rotate — verify before use.

## Relationship with other skills

- **sui-ts-sdk** — DeepBook PTBs are ordinary Sui PTBs; wallet signing, gas, batching follow that skill.
- **sui-frontend** — for wallet-driven DeepBook trades in browsers via dApp Kit hooks.
- **move** — for writing Move modules that call DeepBook (e.g., a router that holds a `BalanceManager`, or a strategy contract).
- **walrus / seal** — orthogonal; only relevant if you also need encrypted off-chain state.

## Usage

Reference `SKILL.md` in your project's `CLAUDE.md` — see the [root README](../README.md) for install options.
