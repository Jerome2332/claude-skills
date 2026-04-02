# Frontend with framework-kit (Next.js / React)

## Goals

- One Solana client instance for the app (RPC + WebSocket + wallet connectors)
- Wallet Standard-first discovery/connect
- Minimal "use client" footprint in Next.js (hooks only in leaf components)
- Transaction sending that is observable, cancelable, and UX-friendly

## Recommended Dependencies

```bash
pnpm add @solana/client @solana/react-hooks @solana/kit
pnpm add @solana-program/system @solana-program/token  # Only what you need
```

## Bootstrap Recommendation

Prefer `create-solana-dapp` and pick a kit/framework-kit compatible template for new projects:

```bash
npx create-solana-dapp@latest my-app
```

## Provider Setup (Next.js App Router)

Create a single client and provide it via SolanaProvider.

### `app/providers.tsx`

```tsx
'use client';

import React from 'react';
import { SolanaProvider } from '@solana/react-hooks';
import { autoDiscover, createClient } from '@solana/client';

const endpoint =
  process.env.NEXT_PUBLIC_SOLANA_RPC_URL ?? 'https://api.devnet.solana.com';

// Some environments prefer an explicit WebSocket endpoint
const websocketEndpoint =
  process.env.NEXT_PUBLIC_SOLANA_WS_URL ??
  endpoint.replace('https://', 'wss://').replace('http://', 'ws://');

export const solanaClient = createClient({
  endpoint,
  websocketEndpoint,
  walletConnectors: autoDiscover(),
});

export function Providers({ children }: { children: React.ReactNode }) {
  return <SolanaProvider client={solanaClient}>{children}</SolanaProvider>;
}
```

### `app/layout.tsx`

```tsx
import { Providers } from './providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## Hook Usage Patterns

Prefer framework-kit hooks before writing your own store/subscription logic:

| Hook | Purpose |
|------|---------|
| `useWalletConnection()` | Connect/disconnect and wallet discovery |
| `useBalance(...)` | Lamport balance |
| `useSolTransfer(...)` | SOL transfers |
| `useSplToken(...)` | Token balances/transfers |
| `useTransactionPool(...)` | Managing send + status + retry flows |

### Example: Wallet Connection

```tsx
'use client';

import { useWalletConnection } from '@solana/react-hooks';

export function WalletButton() {
  const { connected, connect, disconnect, wallets, selectedWallet } = useWalletConnection();

  if (connected) {
    return (
      <button onClick={disconnect}>
        Disconnect {selectedWallet?.name}
      </button>
    );
  }

  return (
    <div>
      {wallets.map((wallet) => (
        <button key={wallet.name} onClick={() => connect(wallet)}>
          Connect {wallet.name}
        </button>
      ))}
    </div>
  );
}
```

### Example: Balance Display

```tsx
'use client';

import { useBalance } from '@solana/react-hooks';

export function BalanceDisplay({ address }: { address: string }) {
  const { balance, isLoading, error } = useBalance(address);

  if (isLoading) return <span>Loading...</span>;
  if (error) return <span>Error loading balance</span>;

  const solBalance = Number(balance) / 1e9;
  return <span>{solBalance.toFixed(4)} SOL</span>;
}
```

### Custom Instructions

When you need custom instructions, build them using `@solana-program/*` and send via framework-kit transaction helpers:

```tsx
'use client';

import { useTransactionPool, useWalletConnection } from '@solana/react-hooks';
import { createTransferInstruction } from '@solana-program/system';

export function useCustomTransfer() {
  const { selectedWallet } = useWalletConnection();
  const { sendTransaction } = useTransactionPool();

  const transfer = async (destination: string, lamports: bigint) => {
    if (!selectedWallet) throw new Error('Wallet not connected');

    const instruction = createTransferInstruction({
      source: selectedWallet.address,
      destination,
      lamports,
    });

    const signature = await sendTransaction([instruction]);
    return signature;
  };

  return { transfer };
}
```

## Data Fetching and Subscriptions

- Prefer watchers/subscriptions over manual polling
- Clean up subscriptions with abort handles returned by watchers
- For Next.js: keep server components server-side; only leaf components that call hooks should be client components

### Subscription Cleanup Pattern

```tsx
'use client';

import { useEffect } from 'react';
import { solanaClient } from './providers';

export function AccountWatcher({ address }: { address: string }) {
  useEffect(() => {
    const abortController = new AbortController();

    const subscription = solanaClient.watchAccount(address, {
      signal: abortController.signal,
      onUpdate: (accountInfo) => {
        console.log('Account updated:', accountInfo);
      },
    });

    return () => {
      abortController.abort();
    };
  }, [address]);

  return null;
}
```

## Transaction UX Checklist

- [ ] Disable inputs while a transaction is pending
- [ ] Provide signature immediately after send
- [ ] Track confirmation states (processed/confirmed/finalized) based on UX need
- [ ] Show actionable errors:

| Error Type | User Message |
|------------|--------------|
| User rejected signing | "Transaction cancelled" |
| Insufficient SOL for fees/rent | "Insufficient SOL balance" |
| Blockhash expired/dropped | "Transaction expired, please retry" |
| Account already in use/initialized | "Account already exists" |
| Program error (custom code) | Parse and display meaningful message |

### Error Handling Example

```tsx
async function handleTransaction() {
  setIsLoading(true);
  setError(null);

  try {
    const signature = await sendTransaction(instructions);
    setSignature(signature);

    // Track confirmation
    const confirmation = await waitForConfirmation(signature);
    setConfirmed(true);
  } catch (thrownObject) {
    const error = ensureError(thrownObject);

    if (error.message.includes('User rejected')) {
      setError('Transaction cancelled');
    } else if (error.message.includes('insufficient')) {
      setError('Insufficient SOL balance');
    } else if (error.message.includes('blockhash')) {
      setError('Transaction expired, please retry');
    } else {
      setError(`Transaction failed: ${error.message}`);
    }
  } finally {
    setIsLoading(false);
  }
}
```

## When to Use ConnectorKit (Optional)

If you need a headless connector with composable UI elements and explicit state control, use ConnectorKit.

**Typical reasons:**
- You want a headless wallet connection core (useful across frameworks)
- You want more control over wallet/account state than a single provider gives
- You need production diagnostics/health checks for wallet sessions

```bash
pnpm add @civic-io/connector-kit
```

## Server Components vs Client Components

### Keep Server Components Server-Side

```tsx
// app/page.tsx (Server Component)
import { TokenList } from './components/token-list';

export default async function Page() {
  // Server-side data fetching
  const tokens = await fetchPopularTokens();

  return (
    <main>
      <h1>Popular Tokens</h1>
      <TokenList initialTokens={tokens} />
    </main>
  );
}
```

### Client Components for Hooks Only

```tsx
// app/components/token-list.tsx
'use client';

import { useState } from 'react';
import { useWalletConnection } from '@solana/react-hooks';

interface TokenListProps {
  initialTokens: Array<Token>;
}

export function TokenList({ initialTokens }: TokenListProps) {
  const [tokens] = useState(initialTokens);
  const { connected } = useWalletConnection();

  return (
    <ul>
      {tokens.map((token) => (
        <li key={token.mint}>
          {token.name}
          {connected && <TradeButton mint={token.mint} />}
        </li>
      ))}
    </ul>
  );
}
```

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.mainnet-beta.solana.com
NEXT_PUBLIC_SOLANA_WS_URL=wss://api.mainnet-beta.solana.com

# For Helius (recommended for production)
NEXT_PUBLIC_SOLANA_RPC_URL=https://mainnet.helius-rpc.com/?api-key=YOUR_KEY
```

## Performance Tips

1. **Lazy load wallet adapters** - Don't include all wallet code in initial bundle
2. **Use suspense boundaries** - Show loading states while wallet connects
3. **Debounce balance updates** - Don't re-render on every lamport change
4. **Cache token metadata** - Token info rarely changes, cache aggressively
