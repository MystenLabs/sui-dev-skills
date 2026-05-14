---
name: deepbook
description: DeepBook v3 — Mysten Labs' onchain central limit order book (CLOB) on Sui, plus the prediction-market and margin/leverage products built on top of it. Use whenever the user is building a DEX or CLOB on Sui, placing or cancelling limit / market orders, swapping with maker rebates, integrating `BalanceManager` / `TradeProof` / `TradeCap`, staking $DEEP for governance and fee discounts, doing flash loans on Sui, building onchain prediction markets with binary up/down positions or vertical ranges, or building margin / leveraged trading on top of DeepBook. Covers both the Move side (`deepbook`, `predict`, `deepbook_margin` packages) and the `@mysten/deepbook-v3` TypeScript SDK. Activate this skill even when the user says "Sui orderbook", "Sui DEX", "DEEP token", "predict market on Sui", or refers to the predict-testnet-4-16 branch — they almost certainly mean DeepBook.
---

# DeepBook Skill

You are integrating with [DeepBook v3](https://docs.sui.io/onchain-finance/deepbookv3/deepbook) — Mysten Labs' onchain central limit order book on Sui, and two products that build on it: **prediction markets** (`packages/predict/`) and **margin trading** (`packages/deepbook_margin/`). Follow these rules precisely.

> **Surface area split**: the **order book** is live on **mainnet**. **Predict** and **margin** are currently **testnet-only**, shipped on the `predict-testnet-4-16` branch. Do not call predict/margin entry functions against a mainnet package id — they don't exist there.

---

## 1. Mental model (read first)

DeepBook v3 is built around three shared objects:

1. **`Pool<Base, Quote>`** — one per trading pair. Internally split into three layers:
   - **Book** — bids / asks, matching, order placement.
   - **State** — per-account balances, volumes, governance, staking.
   - **Vault** — actually moves coins after the book + state agree.
   Every order goes Book → State → Vault. This separation is why a single PTB can place, match, settle, and update governance counters atomically.
2. **`BalanceManager`** — the user's account abstraction. One per user, **reused across every pool the user trades on**. Holds `Balance<T>` for any number of asset types. Owner can deposit/withdraw; **`TradeCap` holders** can place/cancel orders but cannot move funds.
3. **`Registry`** — global directory of pools. Used at pool creation; rarely touched after.

Every interaction except direct swaps takes a `BalanceManager` + a `TradeProof`. The `TradeProof` is a short-lived authorization built inside the same PTB by either `generate_proof_as_owner` or `generate_proof_as_trader`. Without it, the pool refuses to debit the manager.

`$DEEP` is the fee/governance token. Takers pay fees in DEEP (or in the input asset on whitelisted pools); makers who stake enough DEEP earn rebates; stakers can submit proposals to change taker fee / maker fee / stake required, which take effect the next epoch if quorum (½ of total stake) is reached.

---

## 2. Pick the right product

| Product | Module | Status | Use when |
|---|---|---|---|
| **Order book** | `deepbook` | Mainnet | Building a DEX, market-making, routing swaps, anything CLOB-shaped |
| **Prediction markets** | `predict` | **Testnet only** | Binary up/down or vertical-range positions on oracle-priced markets |
| **Margin** | `deepbook_margin` | **Testnet only** | Leveraged trading or onchain lending against DeepBook collateral |

If the user's question is ambiguous, default to the order book. The other two products **depend on** the order book (predict uses the same `BalanceManager`; margin uses `pool_proxy` to route trades through DeepBook pools).

---

## 3. Install

```bash
npm install @mysten/deepbook-v3 @mysten/sui
```

The SDK ships a `DeepBookClient` class with three subnamespaces: `deepBook`, `flashLoans`, `balanceManager`. Most teams extend it:

```ts
import { DeepBookClient } from '@mysten/deepbook-v3';
import type { BalanceManager } from '@mysten/deepbook-v3';
import { SuiClient, getFullnodeUrl } from '@mysten/sui/client';

const client = new DeepBookClient({
  address: signerAddress,
  env: 'mainnet',                          // or 'testnet'
  client: new SuiClient({ url: getFullnodeUrl('mainnet') }),
  balanceManagers: {
    MANAGER_1: { address: '0x...', tradeCap: undefined },  // owner; pass tradeCap if trader
  },
});
```

`env: 'mainnet' | 'testnet'` selects bundled package / pool / coin addresses — you don't hardcode them. To trade as a non-owner trader, set `balanceManagers[key].tradeCap` to the `TradeCap` object id.

---

## 4. `BalanceManager` — the part most integrators get wrong

Rules:
- **One `BalanceManager` per end user**, reused across every pool. Don't create a new one per pool, and **don't share one across users** — anyone with the object id can read balances, and anyone with a `TradeCap` can drain them via legal orders.
- The owner is set at creation (`new()` makes sender the owner; can't be changed). The owner can `mint_trade_cap` up to **1000** times to delegate trading to other addresses.
- Inside a PTB, every order-placing call needs a fresh `TradeProof` produced by `generate_proof_as_owner(&mut bm, ctx)` (when the owner signs) or `generate_proof_as_trader(&mut bm, &trade_cap)` (when a trader signs). The SDK does this for you when you pass `balanceManagerKey`.

### Create + deposit (SDK)

```ts
const tx = new Transaction();
const managerCoinKey = 'DBUSDC';
const depositAmount = 100;

// Create + share a new BalanceManager. SDK reads the resulting id from object changes.
client.balanceManager.createAndShareBalanceManager()(tx);

// Once you have an id, deposit a coin into it.
client.balanceManager.depositIntoManager('MANAGER_1', managerCoinKey, depositAmount)(tx);
```

### Move side

```move
use deepbook::balance_manager::{Self, BalanceManager, TradeProof};

// At creation:
let mut bm = balance_manager::new(ctx);              // sender owns it
let trade_cap = bm.mint_trade_cap(ctx);              // optional: delegate to a trader

// Per-PTB before placing orders:
let proof = bm.generate_proof_as_owner(ctx);         // or generate_proof_as_trader(&cap)
```

`validate_proof(&bm, &proof)` is what pools call internally — you only build the proof.

---

## 5. Order book — Move-side API surface

All in `deepbook::pool`. Generic over `<BaseAsset, QuoteAsset>`. Every state-changing call here takes `&mut Pool`, `&mut BalanceManager`, `&TradeProof`.

```move
// Place a resting limit order. Returns OrderInfo (use it to read the order id).
public fun place_limit_order<Base, Quote>(
    self: &mut Pool<Base, Quote>,
    bm: &mut BalanceManager,
    proof: &TradeProof,
    client_order_id: u64,         // your tag; not used onchain except for events
    order_type: u8,               // constants::no_restriction() | immediate_or_cancel() | fill_or_kill() | post_only()
    self_matching_option: u8,     // constants::self_matching_allowed() | cancel_taker() | cancel_maker()
    price: u64,
    quantity: u64,                // in base lots
    is_bid: bool,                 // true = buy base, false = sell base
    pay_with_deep: bool,          // false → fee in input asset (whitelisted pools only)
    expire_timestamp: u64,        // ms; pool clock checked
    clock: &Clock,
    ctx: &TxContext,
): OrderInfo;

// Market order. Internally a limit at MAX_PRICE (bid) or MIN_PRICE (ask) with IMMEDIATE_OR_CANCEL.
public fun place_market_order<Base, Quote>(
    self: &mut Pool<Base, Quote>, bm: &mut BalanceManager, proof: &TradeProof,
    client_order_id: u64, self_matching_option: u8,
    quantity: u64, is_bid: bool, pay_with_deep: bool,
    clock: &Clock, ctx: &TxContext,
): OrderInfo;

// Direct swap — no BalanceManager. Returns (base, quote, deep) coins.
public fun swap_exact_base_for_quote<Base, Quote>(
    self: &mut Pool<Base, Quote>,
    base_in: Coin<Base>, deep_in: Coin<DEEP>, min_quote_out: u64,
    clock: &Clock, ctx: &mut TxContext,
): (Coin<Base>, Coin<Quote>, Coin<DEEP>);

public fun swap_exact_quote_for_base<Base, Quote>(/* analogous */);

public fun modify_order<Base, Quote>(/* shrink quantity only — can't raise */);
public fun cancel_order<Base, Quote>(self, bm, proof, order_id, clock, ctx);
public fun cancel_orders<Base, Quote>(/* by vector<u128> */);
public fun cancel_all_orders<Base, Quote>(self, bm, proof, clock, ctx);

public fun withdraw_settled_amounts<Base, Quote>(self, bm, proof, ctx); // pull filled proceeds into bm
public fun claim_rebates<Base, Quote>(self, bm, proof, ctx);            // pull maker rebates

// Governance / staking
public fun stake(self, bm, proof, amount, ctx);
public fun unstake(self, bm, proof, ctx);
public fun submit_proposal(self, bm, proof, taker_fee, maker_fee, stake_required, ctx);
public fun vote(self, bm, proof, proposal_id, ctx);

// Flash loans (hot potato)
public fun borrow_flashloan_base<Base, Quote>(self, amount, ctx): (Coin<Base>, FlashLoan);
public fun return_flashloan_base<Base, Quote>(self, coin: Coin<Base>, loan: FlashLoan);
// borrow_flashloan_quote / return_flashloan_quote analogous.
```

### Order type / self-matching constants

These live in `deepbook::constants` and are exposed as **functions returning `u8`** — not enum variants. You must call them:

```move
use deepbook::constants;

constants::no_restriction()       // resting limit
constants::immediate_or_cancel()  // IOC
constants::fill_or_kill()         // FOK
constants::post_only()            // maker-only

constants::self_matching_allowed()
constants::cancel_taker()
constants::cancel_maker()
```

Do not hardcode `0u8` / `1u8` etc. — the order matters and is not part of the public contract.

### Read-only

```move
get_quote_quantity_out / get_base_quantity_out   // simulated swap output
get_level2_range(self, price_low, price_high, is_bid, clock) -> (vec<u64>, vec<u64>)
mid_price, account_open_orders, locked_balance, vault_balances, account
```

---

## 6. Order book — TypeScript SDK

The SDK wraps every Move entry point and resolves coin types + pool ids from a registry keyed by `poolKey`. Common shape:

```ts
const tx = new Transaction();

tx.add(client.deepBook.placeLimitOrder({
  poolKey: 'SUI_DBUSDC',
  balanceManagerKey: 'MANAGER_1',
  clientOrderId: '123456789',
  price: 1.5,                     // in quote per base, decimal form (SDK scales)
  quantity: 10,                   // in base units, decimal form
  isBid: true,
  payWithDeep: true,              // false only on whitelisted pools
  // orderType, selfMatchingOption, expireTimestamp default sensibly
}));

tx.add(client.deepBook.cancelOrder({
  poolKey: 'SUI_DBUSDC',
  balanceManagerKey: 'MANAGER_1',
  orderId: '0x...',
}));

tx.add(client.deepBook.claimRebates({
  poolKey: 'SUI_DBUSDC',
  balanceManagerKey: 'MANAGER_1',
}));

const result = await client.suiClient.signAndExecuteTransaction({ transaction: tx, signer });
```

Swaps without a `BalanceManager`:

```ts
const [baseOut, quoteOut, deepOut] = tx.add(
  client.deepBook.swapExactBaseForQuote({
    poolKey: 'SUI_DBUSDC',
    amount: 5,           // base in, decimal form
    deepAmount: 1,       // upper bound on DEEP fee
    minOut: 0,
  }),
);
tx.transferObjects([baseOut, quoteOut, deepOut], senderAddress);
```

Flash loan pattern — borrow and return inside a single PTB or the transaction aborts:

```ts
const [deepCoin, flashLoan] = tx.add(client.flashLoans.borrowBaseAsset('DEEP_SUI', 1));
// ... use deepCoin in trades ...
const loanRemain = tx.add(client.flashLoans.returnBaseAsset('DEEP_SUI', 1, repaidCoin, flashLoan));
```

Reads (no PTB):

```ts
await client.deepBook.checkManagerBalance('MANAGER_1', 'DBUSDC');
await client.deepBook.getLevel2Range('SUI_DBUSDC', priceLow, priceHigh, isBid);
await client.deepBook.accountOpenOrders('SUI_DBUSDC', 'MANAGER_1');
```

---

## 7. Lot size, tick size, min size

Every pool defines three integer scales. Orders that violate any of them silently underfill or are rejected with confusing aborts. **Always read these from the pool before sizing an order**:

| Param | Means |
|---|---|
| `tick_size` | Minimum price increment (in quote, raw integer units after scaling) |
| `lot_size` | Minimum base-quantity increment |
| `min_size` | Minimum total order quantity |

In the SDK, `poolKey` resolves to the pool's coin scalars; passing decimal `price: 1.5` and `quantity: 10` Just Works. In Move, you must scale yourself using `constants::float_scaling()` (10^9) and the pool's own `tick_size` / `lot_size`. Mismatches show up as `EInvalidLotSize`, `EInvalidTickSize`, `EOrderBelowMinimumSize` aborts.

---

## 8. Fees, rebates, governance

- **Taker fees**: 0.1–10 bps, set per pool. Volatile pools default ~10 bps taker / 5 bps maker; stable pools ~1 bps / 0.5 bps; **whitelisted pools** (DEEP/SUI, DEEP/USDC) are **0 fee**.
- **Maker rebates**: only paid to makers whose `BalanceManager` has at least the pool's `stake_required` of DEEP staked **and** whose maker volume is above a phase-out threshold (28-day median). Above the threshold, rebates scale to zero. Call `claim_rebates` to actually pull rebates into the manager; they aren't auto-paid.
- **Stake activation lag**: DEEP staked in epoch `N` becomes active in epoch `N+1`. New makers don't get rebates the same epoch they stake.
- **Governance**: each `BalanceManager` can submit one proposal per epoch via `submit_proposal(taker_fee, maker_fee, stake_required)`. Voting power is `min(stake, 100_000) + max(sqrt(stake) − sqrt(100_000), 0)` — sub-linear above 100K to limit whale dominance. Quorum is ½ of total stake. Passing proposals take effect next epoch.
- **Pay-with-DEEP vs input asset**: `pay_with_deep: false` is only valid on whitelisted pools, which are also 0-fee. Everywhere else, set `true` and ensure the manager (or, for swaps, the input coins) include enough DEEP.

---

## 9. Off-chain reads

The SDK exposes the most useful queries directly:

```ts
client.deepBook.getLevel2Range(poolKey, priceLow, priceHigh, isBid)  // order book depth
client.deepBook.midPrice(poolKey)
client.deepBook.accountOpenOrders(poolKey, balanceManagerKey)
client.deepBook.lockedBalance(poolKey, balanceManagerKey)
client.deepBook.vaultBalances(poolKey)
client.deepBook.checkManagerBalance(managerId, coinType)
```

Mysten does not publish a fully-featured public indexer. For trade history, fills, P&L, build on **Sui events** (`OrderPlaced`, `OrderFilled`, `OrderCanceled`, `OrderModified`, plus `BalanceEvent` on the manager) — emitted by every state-changing call. Subscribe via `suix_subscribeEvent` or pull via `getCheckpoints`.

---

## 10. Flash loans

Flash loans return a coin **and** a `FlashLoan` hot potato that must be consumed by `return_flashloan_*` in the same PTB — otherwise the transaction aborts at end-of-PTB. There is no fee on the loan itself; you only pay normal trading fees on whatever you do with the borrowed coins. Typical use: arbitrage between two DeepBook pools, or fund a same-PTB swap without pre-positioning capital.

Pitfall: `borrow_flashloan_base` returns `Coin<Base>` — not a `Balance` and not stored in the manager. You can pass it as the input coin to `swap_exact_base_for_quote` directly; you cannot deposit it into a `BalanceManager` and place a limit order with it inside the same PTB (well, you can, but you still have to fully return the same `Base` coin by end of PTB, so you'd be inventing a withdraw).

---

## 11. Predict — onchain prediction markets (**testnet only**)

`packages/predict/` ships on `predict-testnet-4-16`. Read [docs.sui.io/onchain-finance/deepbook-predict](https://docs.sui.io/onchain-finance/deepbook-predict) and the in-repo `packages/predict/README.md` for protocol details. Quick map:

**Shared / per-user objects**

- **`Predict`** — protocol object: vault, pricing config (SVI), accepted quote assets, risk caps. Created once per quote-asset family.
- **`OracleSVI`** — oracle object: spot, forward, SVI params, settlement price. One per underlying/expiry combo.
- **`PredictManager`** — per-user state, wraps a DeepBook `BalanceManager`. Holds the user's quote balance and a `Table<MarketKey, UserPosition>` of open positions plus a paired-position collateral table.
- **`MarketKey(oracle_id, expiry, strike, direction: UP | DOWN)`** — compact position identifier.

**User entry points** (all `public fun` in `predict::predict`)

```move
public fun create_manager(ctx): ID;   // mint a PredictManager; returns its id

public fun mint<Quote>(
    predict: &mut Predict, manager: &mut PredictManager, oracle: &OracleSVI,
    key: MarketKey, quantity: u64, clock: &Clock, ctx: &mut TxContext,
);
public fun redeem<Quote>(predict, manager, oracle, key, quantity, clock, ctx);
public fun redeem_permissionless<Quote>(/* anyone can sell a settled position */);

public fun mint_range<Quote>(/* vertical spread: lower + upper strikes */);
public fun redeem_range<Quote>(/* analogous */);

// LPs
public fun supply<Quote>(predict, lp_cap, coin: Coin<Quote>, clock, ctx);
public fun withdraw<Quote>(predict, lp_cap, shares: u64, clock, ctx): Coin<Quote>;
```

**Pricing**: `mint` charges `cost = ask × quantity` where ask comes from the oracle's SVI surface plus base spread + skew + utilization premium. The SDK exposes `get_trade_amounts(...)` for pre-trade quotes. Min/max ask bounds per oracle are admin-controlled via `set_oracle_ask_bounds`.

**Indexer / read API**: there's a predict server at `https://predict-server.testnet.mystenlabs.com` (subject to change) with:
- `GET /predicts/:predict_id/state`
- `GET /managers/:manager_id/summary`
- `GET /managers/:manager_id/positions/summary`

Pair it with Sui event streams (`PositionMinted`, `PositionRedeemed`, `RangeMinted`, `OracleAskBoundsSet`, `TradingPauseUpdated`) for low-latency updates.

**Testnet addresses** (from the branch — rotate frequently, verify before use):
- Predict package: `0xf5ea2b3749c65d6e56507cc35388719aadb28f9cab873696a2f8687f5c785138`
- `Predict` shared object: `0xc8736204d12f0a7277c86388a68bf8a194b0a14c5538ad13f22cbd8e2a38028a`
- DUSDC quote asset: `e95040085976bfd54a1a07225cd46c8a2b4e8e2b6732f140a0fc49850ba73e1a::dusdc::DUSDC`

---

## 12. Margin — leveraged trading (**testnet only**)

`packages/deepbook_margin/` ships on the same branch. Two main objects:

- **`MarginManager<Base, Quote>`** — wraps a `BalanceManager` and tracks borrow shares. Created per (user, pool). Supports deposit, withdraw, borrow_base / borrow_quote, repay_*, liquidate, plus conditional orders (`add_conditional_order`, TPSL via `tpsl.move`).
- **`MarginPool<Asset>`** — lending pool per asset. LPs `supply()` to mint a `SupplierCap`; `MarginManager`s borrow against it. Interest accrues per ms; admin sets rate model + max utilization + min borrow.

```move
// MarginManager flow
public fun new<Base, Quote>(deepbook_pool: ID, margin_registry, ctx): MarginManager<Base, Quote>;
public fun share<Base, Quote>(mm: MarginManager<Base, Quote>);
public fun deposit<Base, Quote, DepositAsset>(mm, coin: Coin<DepositAsset>, ctx);
public fun withdraw<Base, Quote, WithdrawAsset>(mm, amount, ctx): Coin<WithdrawAsset>;
public fun borrow_base<Base, Quote>(mm, margin_pool: &mut MarginPool<Base>, amount, clock, ctx);
public fun repay_base<Base, Quote>(mm, margin_pool, clock, ctx);
public fun liquidate<Base, Quote, DebtAsset>(/* anyone can liquidate under min risk ratio */);

// Read-only health
public fun risk_ratio<Base, Quote>(mm, deepbook_pool, margin_pool, clock): u64;
```

Trades route via `pool_proxy.move` which forwards orders into the underlying DeepBook pool while keeping debt accounting on the `MarginManager`. Liquidation: anyone may call `liquidate` once `risk_ratio` falls below the configured threshold.

---

## 13. Anti-patterns

- **Sharing one `BalanceManager` across multiple end users.** Even read access is fine, but anyone with a `TradeCap` (or the owner key) can place orders that lock or move funds. One manager per user, always.
- **Hardcoding mainnet package ids / pool ids.** The SDK reads them from `env: 'mainnet' | 'testnet'`. The mainnet package id has been upgraded multiple times — pin via the SDK, not your code.
- **Calling predict/margin entry functions on mainnet.** They are not deployed. Use testnet, and refresh the branch's package id periodically — it rotates.
- **Hardcoding `0u8` / `1u8` for `order_type` or `self_matching_option`.** Always go through `deepbook::constants::*()` — the numeric mapping is internal and not part of the contract.
- **Forgetting `pay_with_deep: true` on non-whitelisted pools.** `false` is only valid on the DEEP/SUI and DEEP/USDC whitelisted pools. Anywhere else, the order will abort at fee calculation.
- **Reading `price` / `quantity` as floats in Move.** They are integers scaled by `constants::float_scaling()` (10^9) plus the coin's own decimals. The SDK handles scaling; Move callers must do it manually.
- **Expecting maker rebates without staking.** Rebates require `staked_deep ≥ pool.stake_required` and that the staking was done in a previous epoch. Until the next epoch, you trade as a non-staked maker.
- **Borrowing in a flash loan and forgetting to return the same coin type.** `return_flashloan_base` consumes the `FlashLoan` hot potato. If it's not consumed by end-of-PTB, the whole tx aborts.
- **Treating `place_market_order` as a separate matching path.** It's `place_limit_order` with `MAX_PRICE` (bid) or `MIN_PRICE` (ask) and `IMMEDIATE_OR_CANCEL`. Same fee model, same self-matching options, same lot-size constraints.

---

## Cross-skill links

- **sui-ts-sdk** — the PTBs DeepBook constructs are ordinary Sui PTBs; wallet signing, gas, multi-tx flows follow that skill.
- **sui-frontend** — for wallet-driven DeepBook integration in browsers (`useSignAndExecuteTransaction` from dApp Kit).
- **move** — when writing your own Move modules that call DeepBook (e.g., a router contract, a vault that holds a `BalanceManager`).
- **walrus / seal** — orthogonal: only relevant if your DEX also stores ciphertext (e.g., for orderbook indexers with private fields). Most teams don't need them.

## Sources

- Docs: `docs.sui.io/onchain-finance/deepbookv3/{deepbook,design,contract-information}`, `/onchain-finance/deepbookv3-sdk`, `/onchain-finance/deepbook-predict`, `/onchain-finance/deepbook-margin`.
- Code: [`MystenLabs/deepbookv3` @ `predict-testnet-4-16`](https://github.com/MystenLabs/deepbookv3/tree/predict-testnet-4-16) — `packages/deepbook/`, `packages/predict/`, `packages/deepbook_margin/`, `scripts/transactions/`.
- Whitepaper: [deepbook.tech](https://deepbook.tech).
