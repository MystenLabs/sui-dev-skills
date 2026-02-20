---
name: sui-frontend
description: Sui frontend dApp development with @mysten/dapp-kit. Use when building React apps that connect to Sui wallets, query on-chain data, or execute transactions from the browser. Use alongside the sui-ts-sdk skill for PTB construction patterns.
---

# Sui Frontend Skill

You are building a React frontend that interacts with the Sui blockchain using `@mysten/dapp-kit`. This skill covers provider setup, wallet connection, on-chain queries, and transaction signing. For PTB construction details (splitCoins, moveCall, coinWithBalance, etc.), apply the **sui-ts-sdk** skill alongside this one — the `Transaction` API is identical in browser and Node contexts.

---

## 1. Package Installation

Three packages are required:

```bash
npm install @mysten/dapp-kit @mysten/sui @tanstack/react-query
```

| Package | Purpose |
|---------|---------|
| `@mysten/dapp-kit` | Wallet adapters, React hooks, ConnectButton |
| `@mysten/sui` | Sui TypeScript SDK (Transaction class, client) |
| `@tanstack/react-query` | Required peer dependency for data fetching and caching |

---

## 2. Provider Setup

Wrap your app in three providers in this exact order:

```tsx
// ✅ app.tsx — correct provider nesting order
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { SuiClientProvider, WalletProvider } from '@mysten/dapp-kit';
import '@mysten/dapp-kit/dist/index.css'; // required for ConnectButton styling
import { networkConfig } from './network';

const queryClient = new QueryClient();

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <SuiClientProvider networks={networkConfig} defaultNetwork="testnet">
        <WalletProvider autoConnect>
          <YourApp />
        </WalletProvider>
      </SuiClientProvider>
    </QueryClientProvider>
  );
}
```

```tsx
// ❌ Wrong order — WalletProvider needs SuiClientProvider as ancestor
<WalletProvider>
  <SuiClientProvider networks={networkConfig}>
    <QueryClientProvider client={queryClient}>
      <YourApp />
    </QueryClientProvider>
  </SuiClientProvider>
</WalletProvider>
```

**Import the dApp Kit CSS.** The `ConnectButton` and wallet modal depend on it. Forgetting this is the most common cause of broken wallet UI.

**`autoConnect`** — reconnects the last-used wallet on page load. Omit if you want the user to manually connect each session.

---

## 3. Network Configuration & Client

dApp Kit's `SuiClientProvider` requires a `networks` map. **Use `SuiJsonRpcClient` here, not `SuiGrpcClient`** — the provider is built for the JSON-RPC API. `SuiGrpcClient` is for backend/Node.js contexts and is not compatible with dApp Kit.

```typescript
// ✅ network.ts
import { SuiJsonRpcClient, getJsonRpcFullnodeUrl } from '@mysten/sui/jsonRpc';

export const networkConfig = {
  mainnet: {
    url: getJsonRpcFullnodeUrl('mainnet'),
    createClient: (config: { url: string }) =>
      new SuiJsonRpcClient({ url: config.url, network: 'mainnet' }),
  },
  testnet: {
    url: getJsonRpcFullnodeUrl('testnet'),
    createClient: (config: { url: string }) =>
      new SuiJsonRpcClient({ url: config.url, network: 'testnet' }),
  },
};
```

```typescript
// ❌ Do not use SuiGrpcClient in SuiClientProvider
import { SuiGrpcClient } from '@mysten/sui/grpc'; // wrong client type for dApp Kit
```

**Both `url` and `network` are required** in `SuiJsonRpcClient`. Omitting either causes wallet display issues or silent connection errors.

For a single-network app, the config can be simplified:

```tsx
// ✅ Single-network shorthand
<SuiClientProvider
  networks={{ mainnet: { url: getJsonRpcFullnodeUrl('mainnet') } }}
  defaultNetwork="mainnet"
>
```

---

## 4. Wallet Connection

### ConnectButton

The simplest approach — renders a "Connect Wallet" button that opens a modal listing all detected wallets:

```tsx
import { ConnectButton } from '@mysten/dapp-kit';

function Header() {
  return (
    <header>
      <ConnectButton />
    </header>
  );
}
```

`ConnectButton` automatically shows the connected address once a wallet is linked. It requires the CSS import from section 2.

### Supported wallets

dApp Kit auto-detects any wallet implementing the [Sui Wallet Standard](https://docs.sui.io/standards/wallet-standard). No manual registration is needed — the user's installed wallets appear automatically. Common wallets: Sui Wallet, Suiet, Ethos, Martian, Morphis, Surf.

### Custom connection UI

When the built-in button doesn't fit your design:

```tsx
import { useWallets, useConnectWallet, useDisconnectWallet } from '@mysten/dapp-kit';

function WalletMenu() {
  const wallets = useWallets();
  const { mutate: connect } = useConnectWallet();
  const { mutate: disconnect } = useDisconnectWallet();

  return (
    <div>
      {wallets.map((wallet) => (
        <button key={wallet.name} onClick={() => connect({ wallet })}>
          {wallet.name}
        </button>
      ))}
      <button onClick={() => disconnect()}>Disconnect</button>
    </div>
  );
}
```

### Wallet connection state

```tsx
import { useCurrentWallet } from '@mysten/dapp-kit';

function ConnectionStatus() {
  const { currentWallet, connectionStatus } = useCurrentWallet();
  // connectionStatus: 'disconnected' | 'connecting' | 'connected'

  if (connectionStatus === 'connecting') return <p>Connecting...</p>;
  if (connectionStatus === 'connected') return <p>Connected: {currentWallet?.name}</p>;
  return <p>Disconnected</p>;
}
```

---

## 5. Current Account

`useCurrentAccount` gives you the connected address and public key:

```tsx
import { useCurrentAccount } from '@mysten/dapp-kit';

function Profile() {
  const account = useCurrentAccount();

  if (!account) {
    return <p>No wallet connected</p>;
  }

  return (
    <div>
      <p>Address: {account.address}</p>
      <p>Public key: {account.publicKey}</p>
    </div>
  );
}
```

`useCurrentAccount()` returns `null` when no wallet is connected. Always null-check before using `account.address` — TypeScript enforces this.

---

## 6. Accessing the Raw Client

`useSuiClient` returns the active `SuiJsonRpcClient` for imperative calls — typically inside event handlers or after a transaction completes. Use this instead of creating your own client instance:

```tsx
import { useSuiClient } from '@mysten/dapp-kit';

function SomeComponent() {
  const client = useSuiClient();

  const handleSuccess = async (digest: string) => {
    // Always wait for indexing before follow-up reads (see sui-ts-sdk §11)
    await client.waitForTransaction({ digest });
    // client methods are the same as SuiJsonRpcClient — see sui-ts-sdk for the full API
  };
}
```

Do not instantiate `new SuiJsonRpcClient(...)` inside components — use `useSuiClient` so it stays in sync with the active network.

---

## 7. Querying On-Chain Data

`useSuiClientQuery` wraps React Query and automatically uses the active network client. The first argument is any `SuiJsonRpcClient` method name (see sui-ts-sdk §14 for the full list of query methods and their argument shapes):

```tsx
import { useSuiClientQuery, useCurrentAccount } from '@mysten/dapp-kit';

// SUI balance for the connected wallet
function Balance() {
  const account = useCurrentAccount();
  const { data, isPending, error } = useSuiClientQuery(
    'getBalance',
    { owner: account?.address ?? '', coinType: '0x2::sui::SUI' },
    { enabled: !!account }, // ✅ skip the query until a wallet is connected
  );

  if (isPending) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  const sui = Number(data.totalBalance) / 1_000_000_000;
  return <p>Balance: {sui.toFixed(4)} SUI</p>;
}
```

```tsx
// Other common method names — argument shapes are in sui-ts-sdk §14
useSuiClientQuery('getOwnedObjects', { owner: account?.address ?? '' }, { enabled: !!account });
useSuiClientQuery('getObject', { id: '0x...', options: { showContent: true } });
useSuiClientQuery('multiGetObjects', { ids: ['0xA', '0xB'], options: { showContent: true } });
useSuiClientQuery('getAllBalances', { owner: account?.address ?? '' }, { enabled: !!account });
```

**Always pass `enabled: !!account`** for queries that require a connected wallet. Without it, the hook fires immediately with an empty owner string and returns an error.

---

## 8. Paginated Queries

For queries that return pages of results, use `useSuiClientInfiniteQuery`:

```tsx
import { useSuiClientInfiniteQuery, useCurrentAccount } from '@mysten/dapp-kit';

function OwnedNFTs() {
  const account = useCurrentAccount();
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useSuiClientInfiniteQuery(
      'getOwnedObjects',
      {
        owner: account?.address ?? '',
        filter: { StructType: '0xPKG::nft::NFT' },
        options: { showContent: true },
      },
      {
        enabled: !!account,
        getNextPageParam: (lastPage) =>
          lastPage.hasNextPage ? lastPage.nextCursor : null,
      },
    );

  const allObjects = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div>
      {allObjects.map((obj) => (
        <NFTCard key={obj.data?.objectId} object={obj} />
      ))}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load more'}
        </button>
      )}
    </div>
  );
}
```

`data.pages` is an array of page responses. Flatten it with `.flatMap` to get a single list of items. `fetchNextPage` fetches the next cursor automatically.

---

## 9. Signing and Executing Transactions

Use `useSignAndExecuteTransaction` to sign and submit a PTB through the connected wallet. Build the `Transaction` exactly as you would in any other context — see **sui-ts-sdk** for PTB construction patterns (`splitCoins`, `moveCall`, `coinWithBalance`, etc.):

```tsx
import { useSignAndExecuteTransaction, useSuiClient, useCurrentAccount } from '@mysten/dapp-kit';
import { Transaction } from '@mysten/sui/transactions';

function ActionButton() {
  const account = useCurrentAccount();
  const client = useSuiClient();
  const { mutate: signAndExecute, isPending } = useSignAndExecuteTransaction();

  const handleAction = () => {
    if (!account) return;

    const tx = new Transaction();
    // ... build PTB using sui-ts-sdk patterns ...

    signAndExecute(
      { transaction: tx },
      {
        onSuccess: async (result) => {
          // result.digest is available immediately after wallet confirms
          await client.waitForTransaction({ digest: result.digest });
          // now safe to re-query updated on-chain state
        },
        onError: (error) => {
          console.error('Transaction failed:', error.message);
        },
      },
    );
  };

  return (
    <button onClick={handleAction} disabled={!account || isPending}>
      {isPending ? 'Waiting for wallet...' : 'Submit'}
    </button>
  );
}
```

### Requesting effects and events

```tsx
signAndExecute(
  {
    transaction: tx,
    options: { showEffects: true, showEvents: true },
  },
  {
    onSuccess: (result) => {
      console.log('Effects:', result.effects);
      console.log('Events:', result.events);
    },
  },
);
```

The hook routes signing through the user's wallet — never call `client.signAndExecuteTransaction()` directly in React components.

---

## 10. Signing Without Executing

When you need the user's signature but execution happens elsewhere — for example, a sponsored transaction where your backend attaches gas and submits:

```tsx
import { useSignTransaction } from '@mysten/dapp-kit';
import { Transaction } from '@mysten/sui/transactions';

function SponsoredMint() {
  const { mutate: signTransaction } = useSignTransaction();

  const handleSign = () => {
    const tx = new Transaction();
    // ... build PTB ...

    signTransaction(
      { transaction: tx },
      {
        onSuccess: ({ bytes, signature }) => {
          // bytes + signature go to the backend; the sponsor attaches gas and executes
          fetch('/api/sponsor', {
            method: 'POST',
            body: JSON.stringify({ bytes, signature }),
          });
        },
        onError: (error) => console.error('Signing rejected:', error),
      },
    );
  };

  return <button onClick={handleSign}>Mint (Gasless)</button>;
}
```

For the server-side sponsored execution flow (attaching the sponsor's gas, collecting both signatures, submitting), see **sui-ts-sdk §15**.

---

## 11. Personal Message Signing

Sign arbitrary bytes with the connected wallet. Useful for authentication challenges where the server issues a nonce and verifies the resulting signature:

```tsx
import { useSignPersonalMessage } from '@mysten/dapp-kit';

function AuthButton() {
  const { mutate: signMessage } = useSignPersonalMessage();

  const handleAuth = () => {
    const message = new TextEncoder().encode('Sign in to MyApp: nonce=abc123');

    signMessage(
      { message },
      {
        onSuccess: ({ bytes, signature }) => {
          // POST bytes + signature to your backend for verification
          verifyOnServer({ bytes, signature });
        },
        onError: (error) => console.error('Signing rejected:', error),
      },
    );
  };

  return <button onClick={handleAuth}>Sign In</button>;
}
```

The message must be a `Uint8Array` — use `TextEncoder` to convert strings. The wallet displays the raw bytes to the user before signing.

---

## 12. Network Switching

Read and change the active network at runtime using `useSuiClientContext`. Only networks included in `SuiClientProvider`'s `networks` prop are available:

```tsx
import { useSuiClientContext } from '@mysten/dapp-kit';

function NetworkSwitcher() {
  const { network, selectNetwork } = useSuiClientContext();

  return (
    <select value={network} onChange={(e) => selectNetwork(e.target.value)}>
      <option value="mainnet">Mainnet</option>
      <option value="testnet">Testnet</option>
    </select>
  );
}
```

Switching network automatically updates the client returned by `useSuiClient` and re-runs any active `useSuiClientQuery` hooks against the new network.

---

## 13. Cache Invalidation After Transactions

After a successful transaction, on-chain state has changed. Invalidate relevant queries so the UI reflects the update. Always wait for indexing first — invalidating immediately after `signAndExecute` resolves will still return stale data:

```tsx
import { useQueryClient } from '@tanstack/react-query';
import { useSignAndExecuteTransaction, useSuiClient } from '@mysten/dapp-kit';

function MintButton() {
  const queryClient = useQueryClient();
  const client = useSuiClient();
  const { mutate: signAndExecute } = useSignAndExecuteTransaction();

  const handleMint = () => {
    const tx = new Transaction();
    // ... build PTB ...

    signAndExecute(
      { transaction: tx },
      {
        onSuccess: async (result) => {
          await client.waitForTransaction({ digest: result.digest }); // ✅ wait first
          await queryClient.invalidateQueries({ queryKey: ['getBalance'] });
          await queryClient.invalidateQueries({ queryKey: ['getOwnedObjects'] });
        },
      },
    );
  };

  return <button onClick={handleMint}>Mint NFT</button>;
}
```

```tsx
// ❌ Invalidating before waitForTransaction — indexer hasn't caught up yet
onSuccess: async (result) => {
  await queryClient.invalidateQueries({ queryKey: ['getBalance'] }); // stale!
  await client.waitForTransaction({ digest: result.digest });
},
```

---

## 14. Wallet-Gated UI

Conditionally render content based on connection state:

```tsx
import { useCurrentAccount, ConnectButton } from '@mysten/dapp-kit';

function ProtectedPage() {
  const account = useCurrentAccount();

  if (!account) {
    return (
      <div>
        <p>Connect your wallet to continue.</p>
        <ConnectButton />
      </div>
    );
  }

  return <Dashboard address={account.address} />;
}
```

For a reusable guard component:

```tsx
function WalletGuard({ children }: { children: React.ReactNode }) {
  const account = useCurrentAccount();
  if (!account) return <ConnectButton />;
  return <>{children}</>;
}

// Usage
<WalletGuard>
  <Dashboard />
</WalletGuard>
```

---

## 15. What dApp Kit is NOT

| Mistake | Correct approach |
|---------|-----------------|
| Using `SuiGrpcClient` in `SuiClientProvider` | Use `SuiJsonRpcClient` — dApp Kit requires the JSON-RPC client |
| Omitting `network` in `SuiJsonRpcClient` | Both `url` and `network` are required; omitting either breaks wallet display |
| Forgetting `@mysten/dapp-kit/dist/index.css` | Required for `ConnectButton` and wallet modal styling |
| Registering wallet providers manually | dApp Kit auto-detects all Wallet Standard wallets; no registration needed |
| Calling `client.signAndExecuteTransaction()` directly in a component | Use `useSignAndExecuteTransaction` — it routes signing through the connected wallet |
| Reading `account.address` without null check | `useCurrentAccount()` returns `null` before connection; always guard |
| `useSuiClientQuery` without `enabled: !!account` | Without `enabled`, the query fires with an empty owner string and returns errors |
| Invalidating queries before `waitForTransaction` | Indexer may not have processed the tx yet; always wait first |
| Placing providers in the wrong order | Order must be: `QueryClientProvider` → `SuiClientProvider` → `WalletProvider` |
| Creating `new SuiJsonRpcClient()` inside a component | Use `useSuiClient()` — stays in sync with the active network |
| Encoding message as a plain string for `useSignPersonalMessage` | Encode with `new TextEncoder().encode(str)` — the hook expects `Uint8Array` |
