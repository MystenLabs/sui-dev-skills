---
name: sui-move
description: Sui Move 2024 smart contract development. Use when writing, reviewing, or debugging Sui Move code, Move.toml configuration, or Sui object model patterns.
---

# Sui Move Development Skill

You are writing Sui Move smart contracts. Follow these rules precisely. Sui Move is **not** Aptos Move and is **not** Rust — do not apply patterns from those languages.

---

## 1. Package Setup

Always use Move 2024 edition. Every new package's `Move.toml` must include:

```toml
[package]
name = "my_package"
edition = "2024.beta"
```

**Implicit framework dependencies (Sui 1.45+)** — do not list `Sui`, `MoveStdlib`, `Bridge`, or `SuiSystem` in `[dependencies]`. They are implicit:

```toml
# ✅ Sui 1.45+
[dependencies]
# no framework entries needed

# ❌ Outdated
[dependencies]
Sui = { git = "...", subdir = "crates/sui-framework/packages/sui-framework", rev = "..." }
```

**Named addresses** — prefix named addresses with your project name to avoid conflicts:

```toml
# ✅
[addresses]
my_protocol_amm = "0x0"

# ❌ Too generic — prone to collisions
[addresses]
amm = "0x0"
```

Run `sui move build` after any significant change to verify the code compiles before proceeding.

---

## 2. Module Layout (Move 2024)

Use the new single-line module declaration without braces:

```move
// ✅ Move 2024
module my_package::my_module;

// ❌ Legacy — do not use
module my_package::my_module {
    ...
}
```

Standard section order within a module:
1. `use` imports
2. Constants (`const`)
3. Structs / Enums
4. `fun init` (if needed)
5. Public functions
6. `public(package)` functions
7. Private functions
8. Test module (`#[test_only]`)

Use `=== Section Title ===` comments to delimit sections for readability.

### `use` import rules

Don't use a lone `{Self}` — just import the module directly:

```move
// ✅
use my_package::my_module;

// ❌ Redundant braces
use my_package::my_module::{Self};
```

When importing both the module and members, group them with `Self`:

```move
// ✅
use sui::coin::{Self, Coin};

// ❌ Separate imports
use sui::coin;
use sui::coin::Coin;
```

---

## 3. Structs

All structs must be declared `public`. Ability declarations go **after** the fields:

```move
// ✅ Move 2024
public struct Pool has key {
    id: UID,
    balance_x: Balance<SUI>,
    balance_y: Balance<USDC>,
}

public struct PoolCap has key, store {
    id: UID,
    pool_id: ID,
}

// ❌ Legacy — no public keyword
struct Pool has key {
    id: UID,
}
```

**Object rule**: Any struct with the `key` ability **must** have `id: UID` as its first field. Use `object::new(ctx)` to create UIDs — never reuse or fabricate them.

### Naming conventions

**Capabilities** must be suffixed with `Cap`:

```move
// ✅
public struct AdminCap has key, store { id: UID }

// ❌ Unclear it's a capability
public struct Admin has key, store { id: UID }
```

**No `Potato` suffix** — a struct's lack of abilities already communicates it's a hot potato:

```move
// ✅
public struct Promise {}

// ❌
public struct PromisePotato {}
```

**Events named in past tense** — they describe something that already happened:

```move
// ✅
public struct LiquidityAdded has copy, drop { ... }
public struct FeesCollected has copy, drop { ... }

// ❌
public struct AddLiquidity has copy, drop { ... }
public struct CollectFees has copy, drop { ... }
```

**Dynamic field keys** — use positional structs (no named fields):

```move
// ✅
public struct BalanceKey() has copy, drop, store;

// ⚠️ Acceptable but not canonical
public struct BalanceKey has copy, drop, store {}
```

### Constants naming

Error constants use `EPascalCase`. All other constants use `ALL_CAPS`:

```move
// ✅
const ENotAuthorized: u64 = 0;
const MAX_FEE_BPS: u64 = 10_000;

// ❌
const NOT_AUTHORIZED: u64 = 0;   // error should be EPascalCase
const MaxFeeBps: u64 = 10_000;   // non-error should be ALL_CAPS
```

---

## 4. Object Abilities Cheat Sheet

| Ability | Meaning in Sui |
|---------|---------------|
| `key` | Struct is an on-chain object; requires `id: UID` as first field |
| `store` | Can be embedded inside other objects; enables `public_transfer`, `public_share_object`, `public_freeze_object` |
| `copy` | Value can be duplicated (not valid on objects with `key`) |
| `drop` | Value can be silently discarded |

**Object ownership model:**

```move
// Transfer to an address (owned object)
transfer::transfer(obj, recipient);             // key only — module-restricted
transfer::public_transfer(obj, recipient);      // key + store — usable anywhere

// Share (accessible by anyone, goes through consensus)
transfer::share_object(obj);                    // key only — module-restricted
transfer::public_share_object(obj);             // key + store

// Freeze (immutable forever)
transfer::freeze_object(obj);                   // key only — module-restricted
transfer::public_freeze_object(obj);            // key + store
```

Only call `transfer`, `share_object`, and `freeze_object` (the non-`public_` variants) inside the module that **defines** that object's type.

**Never** construct an object struct literal outside of its defining module.

---

## 5. Mutability — `mut` is Required

All variables that are reassigned or mutably borrowed must be declared `let mut`:

```move
// ✅ Move 2024
let mut pool = Pool { id: object::new(ctx), ... };
let mut balance = balance::zero<SUI>();

// ❌ Legacy
let pool = Pool { id: object::new(ctx), ... };
```

Function parameters that are mutably borrowed must also be `mut`:

```move
public fun deposit(mut pool: Pool, coin: Coin<SUI>): Pool { ... }
```

---

## 6. Visibility

| Keyword | Scope |
|---------|-------|
| `public` | Callable from any module |
| `public(package)` | Callable only within the same package |
| *(none)* | Private — callable only within the same module |

`public(friend)` and `friend` declarations are **deprecated**. Use `public(package)` instead.

```move
// ✅
public(package) fun internal_logic(pool: &mut Pool) { ... }

// ❌ Deprecated
friend my_package::other_module;
public(friend) fun internal_logic(pool: &mut Pool) { ... }
```

**Never use `public entry`** — use one or the other. `public` functions are composable in PTBs and return values; `entry` functions are transaction endpoints only and cannot return values:

```move
// ✅ Composable — can be chained in PTBs
public fun mint(ctx: &mut TxContext): NFT { ... }

// ✅ Transaction endpoint — no return value needed
entry fun mint_and_transfer(recipient: address, ctx: &mut TxContext) { ... }

// ❌ Redundant combination — avoid
public entry fun mint(ctx: &mut TxContext): NFT { ... }
```

### Function parameter ordering

Always order parameters: mutable objects → immutable objects → capabilities → primitive types → `&Clock` → `&mut TxContext`:

```move
// ✅
public fun call(
    app: &mut App,
    config: &Config,
    cap: &AdminCap,
    amount: u64,
    is_active: bool,
    clock: &Clock,
    ctx: &mut TxContext,
) { }

// ❌ Wrong order
public fun call(
    amount: u64,
    app: &mut App,
    cap: &AdminCap,
    config: &Config,
    ctx: &mut TxContext,
) { }
```

### Getter naming

Name getters after the field. Do not use a `get_` prefix:

```move
// ✅
public fun fee_bps(pool: &Pool): u64 { pool.fee_bps }
public fun fee_bps_mut(pool: &mut Pool): &mut u64 { &mut pool.fee_bps }

// ❌
public fun get_fee_bps(pool: &Pool): u64 { pool.fee_bps }
```

---

## 7. Method Syntax

In Move 2024, functions whose first argument matches a type are automatically callable as methods:

```move
// Given:
public fun value(coin: &Coin<SUI>): u64 { coin.value() }

// Both are valid, prefer the method form:
let v = coin.value();       // ✅ method syntax — prefer this
let v = coin::value(&coin); // ✅ also valid, but more verbose
```

Use method syntax wherever it improves readability. Declare `use fun` aliases for functions defined outside the owning module:

```move
use fun my_module::pool_value as Pool.value;
```

---

## 8. Enums (Move 2024)

Use enums for types with multiple variants. Enums **cannot** have the `key` ability (they cannot be top-level objects), but they can be stored inside objects:

```move
public enum OrderStatus has copy, drop, store {
    Pending,
    Filled { amount: u64 },
    Cancelled,
}
```

Pattern match with `match`:

```move
match (order.status) {
    OrderStatus::Pending => { ... },
    OrderStatus::Filled { amount } => { /* use amount */ },
    OrderStatus::Cancelled => { ... },
}
```

---

## 9. Macros

Use macro functions for higher-order patterns instead of manual loops.

### Vector macros

```move
// Do something N times
32u8.do!(|_| do_action());

// Build a new vector from an index range
let v = vector::tabulate!(32, |i| i);

// Iterate by immutable reference
vec.do_ref!(|e| process(e));

// Iterate by mutable reference
vec.do_mut!(|e| *e = *e + 1);

// Consume vector, calling a function on each element
vec.destroy!(|e| handle(e));

// Fold into a single value
let sum = vec.fold!(0u64, |acc, x| acc + x);

// Filter (requires T: drop)
let big = vec.filter!(|x| *x > 100);
```

All of these replace verbose manual `while` loops. Use them whenever you iterate over a vector.

### Option macros

```move
// Execute a function if Some, then drop
opt.do!(|value| process(value));

// Unwrap with a default (or abort)
let value = opt.destroy_or!(default);
let value = opt.destroy_or!(abort ECannotBeEmpty);
```

These replace verbose `if (opt.is_some())` / `destroy_some()` patterns:

```move
// ❌ Verbose
if (opt.is_some()) {
    let inner = opt.destroy_some();
    process(inner);
};

// ✅
opt.do!(|inner| process(inner));
```

---

## 10. Common Standard Library Patterns

```move
// Strings — use method syntax, don't import utf8
let s: String = b"hello".to_string();
let ascii: ascii::String = b"hello".to_ascii_string();

// Coin and Balance
use sui::coin::{Self, Coin};
use sui::balance::{Self, Balance};

let balance: Balance<SUI> = coin.into_balance();
let coin: Coin<SUI> = balance.into_coin(ctx);  // ✅ method syntax
let amount: u64 = coin.value();

// Split a payment
let exact = payment.split(amount, ctx);        // ✅
let exact = payment.balance_mut().split(amount); // ✅ avoids ctx

// Consuming values without `drop` — the @0x0 burn pattern
//
// Sui Move's linear type system requires every non-`drop` value to be
// explicitly consumed. The `_` prefix only suppresses warnings for values
// that *do* have `drop` — it won't help for Balance<T>, Coin<T>, or your
// own structs that lack `drop`.
//
// To permanently destroy any `key + store` object, transfer it to @0x0
// (an address no one controls, equivalent to Solidity's address(0)):
transfer::public_transfer(my_obj, @0x0);       // ✅ permanent burn
//
// Balance<T> has neither `drop` nor `key`, so it cannot be transferred
// directly, and `balance::destroy_zero` only works on empty balances.
// Wrap it in a Coin first:
//
//   let _locked = supply.increase_supply(MINIMUM_LIQUIDITY); // ❌ compile error
//
let locked = supply.increase_supply(MINIMUM_LIQUIDITY).into_coin(ctx);
transfer::public_transfer(locked, @0x0);       // ✅ burns the minimum liquidity
//
// Hot potatoes (structs with no abilities at all) cannot use this pattern —
// they must be destructured and each field consumed individually.

// Option
let opt: Option<u64> = option::some(42);
let val = opt.destroy_or!(default_value);      // ✅ macro form
let val = opt.borrow();

// Address and IDs
let id: ID = object::id(&my_obj);
let addr: address = id.to_address();

// UID deletion
id.delete();                                   // ✅
// object::delete(id);                         // ❌ verbose

// TxContext sender
ctx.sender()                                   // ✅
// tx_context::sender(ctx)                     // ❌ verbose

// Vector literals and index syntax
let mut v = vector[1, 2, 3];                   // ✅ literal
let first = v[0];                              // ✅ index syntax
assert!(v.length() == 3);                      // ✅ method syntax
// let mut v = vector::empty();               // ❌ verbose
// vector::push_back(&mut v, 1);              // ❌ verbose

// Struct unpack — use .. to ignore fields you don't need
let MyStruct { id, .. } = value;               // ✅
// let MyStruct { id, field_a: _, field_b: _ } = value; // ❌ verbose
```

---

## 11. Events

Emit events for all state-changing operations that clients need to observe:

```move
use sui::event;

public struct LiquidityAdded has copy, drop {
    pool_id: ID,
    amount_x: u64,
    amount_y: u64,
    lp_minted: u64,
}

// Inside function:
event::emit(LiquidityAdded {
    pool_id: object::id(pool),
    amount_x,
    amount_y,
    lp_minted,
});
```

---

## 12. Error Handling

Error constants use `EPascalCase` and `u64` values:

```move
const EInsufficientLiquidity: u64 = 0;
const EZeroAmount: u64 = 1;

assert!(amount > 0, EZeroAmount);
```

### Clever errors (Move 2024)

Annotating a constant with `#[error]` allows it to carry a human-readable message. The value can be any valid constant type — `vector<u8>` is most common for string messages:

```move
#[error]
const EInsufficientLiquidity: vector<u8> = b"Insufficient liquidity in pool";

assert!(reserves > 0, EInsufficientLiquidity);
abort EInsufficientLiquidity
```

At runtime, the Sui CLI and GraphQL server automatically decode these into a readable message:
```
Error from '0x2::amm::swap' (line 42), abort 'EInsufficientLiquidity': "Insufficient liquidity in pool"
```

**Gotcha**: clever error abort codes encode the source line number, so their `u64` value can change if the file is reformatted or lines shift. Don't hardcode clever error abort codes in tests or off-chain tooling — match by constant name instead.

**`assert!` without an abort code** is also valid and auto-derives a clever abort code from the source line:

```move
// ✅ Valid — line number is embedded automatically
assert!(amount > 0);
```

This is fine for internal invariants where the line number alone is enough context.

---

## 13. One-Time Witness (OTW) Pattern

Use the OTW pattern for modules that need a unique, uncopyable proof-of-publication (e.g., coin types, publisher objects):

```move
public struct MY_MODULE has drop {}

fun init(otw: MY_MODULE, ctx: &mut TxContext) {
    // The OTW name must exactly match the module name in ALL_CAPS
    let publisher = package::claim(otw, ctx);
    transfer::public_transfer(publisher, ctx.sender());
}
```

---

## 14. Capability Pattern

Use capability objects to gate privileged functions instead of checking `ctx.sender()`. This is more composable and testable — the capability can be held by a contract, not just a wallet:

```move
// ✅ Capability-gated
public struct AdminCap has key, store { id: UID }

public fun set_fee(_: &AdminCap, pool: &mut Pool, new_fee: u64) {
    pool.fee_bps = new_fee;
}

// ❌ Sender check — not composable with other contracts
public fun set_fee(pool: &mut Pool, ctx: &TxContext) {
    assert!(ctx.sender() == pool.admin, ENotAdmin);
}
```

Note the parameter order: the object (`pool`) comes before the primitive (`new_fee`), and `_: &AdminCap` follows the objects-then-capabilities ordering from section 6.

---

## 15. Pure Functions and Composability

Keep core logic functions **pure** — they take objects by reference/value and return values. Do not call `transfer::transfer` inside core logic functions:

```move
// ✅ Pure — composable with other protocols
public fun swap<X, Y>(
    pool: &mut Pool<X, Y>,
    coin_in: Coin<X>,
    ctx: &mut TxContext,
): Coin<Y> {
    // ... swap logic
}

// ❌ Transfer inside core logic breaks composability
public fun swap<X, Y>(pool: &mut Pool<X, Y>, coin_in: Coin<X>, ctx: &mut TxContext) {
    let coin_out = /* ... */;
    transfer::public_transfer(coin_out, ctx.sender()); // ❌
}
```

Return excess coins even if their value is zero — let the caller decide what to do with them.

---

## 16. Dynamic Fields

Use dynamic fields for extensible storage or when the key set is not known at compile time:

```move
use sui::dynamic_field as df;
use sui::dynamic_object_field as dof;

// Add a dynamic field (value stored inline with parent)
df::add(&mut parent.id, key, value);

// Add a dynamic object field (value is an independent object)
dof::add(&mut parent.id, key, child_obj);

// Access
let val: &MyType = df::borrow(&parent.id, key);
let val: &mut MyType = df::borrow_mut(&mut parent.id, key);

// Remove
let val: MyType = df::remove(&mut parent.id, key);
```

---

## 17. Comments

Use `///` for doc comments. JavaDoc-style `/** */` is not supported in Move:

```move
/// Returns the current fee in basis points.
public fun fee_bps(pool: &Pool): u64 { pool.fee_bps }

// ❌ Not supported
/** Returns the current fee in basis points. */
public fun fee_bps(pool: &Pool): u64 { pool.fee_bps }
```

Use regular `//` comments to explain non-obvious logic, potential edge cases, and TODOs:

```move
// Note: can underflow if reserve is smaller than minimum_liquidity.
// TODO: add assert! guard before production use.
let lp_supply = math::sqrt(reserve_x * reserve_y);
```

---

## 18. Building and Testing

Always verify code compiles and tests pass using the Sui CLI:

```bash
# Build
sui move build

# Run all tests
sui move test

# Run a specific test by name
sui move test swap_exact_input
```

### Test conventions

**Naming** — do not prefix test functions with `test_`. The `#[test]` attribute already signals intent:

```move
// ✅
#[test] fun create_pool() { }
#[test] fun swap_returns_correct_amount() { }

// ❌
#[test] fun test_create_pool() { }
```

**Merge attributes** — combine `#[test]` and `#[expected_failure]` on one line:

```move
// ✅
#[test, expected_failure(abort_code = EInsufficientLiquidity)]
fun swap_with_zero_input() { ... }

// ❌
#[test]
#[expected_failure(abort_code = EInsufficientLiquidity)]
fun swap_with_zero_input() { ... }
```

**Don't clean up in `expected_failure` tests** — let them abort naturally, don't add `scenario.end()` or other teardown:

```move
// ✅
#[test, expected_failure(abort_code = EInsufficientLiquidity)]
fun swap_with_zero_input() {
    let mut ctx = tx_context::dummy();
    let pool = create_pool(&mut ctx);
    pool.swap(coin::zero(&mut ctx)); // aborts here — done
}

// ❌ — don't clean up after expected failure
#[test, expected_failure(abort_code = EInsufficientLiquidity)]
fun swap_with_zero_input() {
    let mut scenario = test_scenario::begin(@0xA);
    // ... test body ...
    scenario.end(); // unnecessary, misleading
}
```

**Use `tx_context::dummy()` for simple tests** — only reach for `test_scenario` when you genuinely need multi-transaction or multi-sender behaviour:

```move
// ✅ Simple test — no scenario needed
#[test]
fun create_pool() {
    let mut ctx = tx_context::dummy();
    let pool = new_pool(&mut ctx);
    assert_eq!(pool.fee_bps(), 30);
    sui::test_utils::destroy(pool);
}

// ✅ Multi-sender test — scenario is appropriate
#[test]
fun only_admin_can_set_fee() {
    let mut scenario = test_scenario::begin(@admin);
    // ...
    scenario.end();
}
```

**Assertions** — prefer `assert_eq!` over `assert!` for value comparisons (shows both sides on failure), and never pass abort codes to `assert!`:

```move
// ✅
assert_eq!(pool.fee_bps(), 30);
assert!(pool.is_active());

// ❌
assert!(pool.fee_bps() == 30);   // doesn't show the actual value on failure
assert!(pool.is_active(), 0);    // abort code conflicts with app error codes
```

**Destroying objects in tests** — use `sui::test_utils::destroy`, never write custom `destroy_for_testing` functions:

```move
// ✅
use sui::test_utils::destroy;
destroy(pool);

// ❌
pool.destroy_for_testing();
```

---

## 19. What Sui Move is NOT

| Pattern | Source | Do NOT use in Sui Move |
|---------|--------|------------------------|
| `acquires`, `move_to`, `move_from`, `borrow_global` | Aptos / Core Move | Sui has no global storage |
| `signer` type | Aptos / Core Move | Use `&mut TxContext` and `ctx.sender()` |
| `Script` functions | Aptos | Use `entry` functions instead |
| `public(friend)` | Legacy Sui Move | Use `public(package)` |
| Struct without `public` keyword | Legacy Sui Move | All structs must be `public` in 2024 |
| `let x = ...` for mutable vars | Legacy Sui Move | Use `let mut x = ...` |
| `use` inside function bodies for module-level imports | Style issue | Put `use` at the top of the module |
| `&signer` | Rust / Aptos | Does not exist in Sui Move |