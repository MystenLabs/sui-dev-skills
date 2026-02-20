# Sui Frontend Skill

A Claude skill for building Sui dApps with `@mysten/dapp-kit` — provider setup, wallet connection, on-chain queries, and transaction signing in React. Fixes common AI coding mistakes like using `SuiGrpcClient` instead of `SuiJsonRpcClient` in `SuiClientProvider`, missing CSS imports, wrong provider order, and invalidating queries before waiting for indexing.

## What's covered

- Package installation (`@mysten/dapp-kit`, `@mysten/sui`, `@tanstack/react-query`)
- Provider hierarchy (`QueryClientProvider` → `SuiClientProvider` → `WalletProvider`)
- Network config with `SuiJsonRpcClient` (correct client for dApp Kit)
- Wallet connection (`ConnectButton`, `useWallets`, `useConnectWallet`, `useDisconnectWallet`)
- Wallet state (`useCurrentAccount`, `useCurrentWallet`, connection status)
- Raw client access (`useSuiClient`)
- On-chain queries (`useSuiClientQuery` with `enabled` guard)
- Paginated queries (`useSuiClientInfiniteQuery`)
- Transaction signing and execution (`useSignAndExecuteTransaction`)
- Signing without executing (`useSignTransaction`) for sponsored flows
- Personal message signing (`useSignPersonalMessage`) for auth
- Network switching (`useSuiClientContext`)
- Cache invalidation after transactions (`useQueryClient` + `waitForTransaction`)
- Wallet-gated UI patterns
- Anti-patterns to avoid (wrong client type, missing CSS, wrong provider order, stale queries)

## Relationship with sui-ts-sdk skill

The `Transaction` class and PTB construction patterns from **sui-ts-sdk** work identically in the browser. This skill covers the React/dApp Kit layer on top. Use both skills together when building a frontend — the frontend skill for hooks and providers, the SDK skill for building transactions.

## Usage

Copy or reference `SKILL.md` in your project's `CLAUDE.md`:

```bash
cp SKILL.md ~/my-sui-dapp/CLAUDE.md
```

Or via the monorepo path — see the [root README](../README.md).

## Evals

`evals/evals.json` contains four test cases covering provider setup, portfolio queries, transaction execution with cache invalidation, and sign-in + network switching. See the [root README](../README.md) for how to run them.
