---

## name: sui-dev-skills
description: Sui blockchain development — smart contracts, TypeScript SDK, and frontend dApps. Use when writing, reviewing, or debugging any Sui project. Activates the appropriate sub-skill(s) based on context.

# Sui Dev Skills

This is a collection of composable skills for Sui blockchain development. Each sub-skill covers a different layer of the stack. Read the relevant sub-skill(s) before writing any Sui code.

## Sub-skills


| Skill            | Path                    | When to activate                                                                                                                                         |
| ---------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **sui-move**     | `sui-move/SKILL.md`     | Writing, reviewing, or debugging Sui Move smart contracts, `Move.toml` config, or object model patterns                                                  |
| **sui-ts-sdk**   | `sui-ts-sdk/SKILL.md`   | Writing TypeScript that interacts with Sui — PTB construction, `@mysten/sui` client setup, transaction execution, on-chain queries (backend or frontend) |
| **sui-frontend** | `sui-frontend/SKILL.md` | Building browser dApps with `@mysten/dapp-kit-react` or `@mysten/dapp-kit-core` — wallet connection, React hooks, Web Components                         |


## How to use these skills

1. **Identify the layers involved.** A task may touch one or more layers:
  - Move contracts only → read `sui-move/SKILL.md`
  - TypeScript backend or scripts → read `sui-ts-sdk/SKILL.md`
  - Frontend dApp → read `sui-frontend/SKILL.md` **and** `sui-ts-sdk/SKILL.md` (the `Transaction` API is the same in browser and Node)
  - Full-stack → read all three
2. **Read the sub-skill(s) before generating code.** Each SKILL.md contains API patterns, correct import paths, and common mistakes to avoid. Sui Move is not Aptos Move, and the SDK has undergone major renames — the details matter.
3. **Follow the rules in each sub-skill exactly.** They override general knowledge about Sui, Move, or the TypeScript SDK. If a sub-skill says to use a specific import path or API, use it even if your training data suggests otherwise.

## Key cross-cutting rules

- **Package name**: The TypeScript SDK is `@mysten/sui` (not `@mysten/sui.js`). The frontend kit is `@mysten/dapp-kit-react` or `@mysten/dapp-kit-core` (not `@mysten/dapp-kit`).
- **Move edition**: Always use `edition = "2024.beta"` in `Move.toml`.
- **PTB patterns**: The `Transaction` class from `@mysten/sui/transactions` is the same whether you're in a Node script or a React component. The sui-ts-sdk skill is the authority on PTB construction.
- **Client**: Use `SuiGrpcClient` for new code. The JSON-RPC client (`SuiClient`) is deprecated.

