# Sui Move Skill

A Claude skill for idiomatic Sui Move 2024 development. Fixes common AI coding mistakes like using outdated Move patterns, confusing Sui Move with Aptos Move, and violating Sui's object model.

## What's covered

- Move 2024 edition requirements (`edition = "2024.beta"`, implicit deps)
- Sui object model (key/store/copy/drop, ownership modes)
- Modern syntax (method syntax, enums, macros like `do!`, `tabulate!`)
- Standard library patterns (`balance.into_coin`, `ctx.sender()`, `id.delete()`)
- Naming conventions (Cap suffix, past-tense events, EPascalCase errors)
- Pure function pattern (return values, don't call transfer internally)
- Testing conventions (no `test_` prefix, `assert_eq!`, `test_utils::destroy`)
- Anti-patterns to avoid (Aptos patterns, `public(friend)`, `public entry`)

## Usage

Copy or reference `SKILL.md` in your project's `CLAUDE.md`:

```bash
cp SKILL.md ~/my-sui-project/CLAUDE.md
```

Or via the monorepo path — see the [root README](../README.md).

## Evals

The AMM benchmark in `evals/evals.json` tests whether the skill produces idiomatic Move 2024 code for a Uniswap-style AMM. See the [root README](../README.md) for how to run it.