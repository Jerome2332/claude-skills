# Solana Frontend Development

## Architecture Patterns

### Single Client Instance
Establish one Solana client per application with centralized configuration.

```typescript
// app/providers.tsx
'use client';

import { SolanaProvider, createClient } from '@solana/framework-kit';

const client = createClient({
  endpoint: process.env.NEXT_PUBLIC_RPC_URL!,
  wsEndpoint: process.env.NEXT_PUBLIC_WS_URL,
});

export function Providers({ children }: { children: React.ReactNode }) {
  return <SolanaProvider client={client}>{children}</SolanaProvider>;
}
```

### App Router Integration
```typescript
// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## React Hooks

### useWalletConnection
```typescript
import { useWalletConnection } from '@solana/framework-kit';

function WalletButton() {
  const {
    wallets,           // Available wallets
    selectedWallet,    // Currently connected
    connect,           // Connect function
    disconnect,        // Disconnect function
    isConnecting,      // Loading state
  } = useWalletConnection();

  // Auto-discover available wallets
  const availableWallets = wallets.filter(w => w.readyState === 'Installed');

  if (selectedWallet) {
    return (
      <button onClick={disconnect}>
        {selectedWallet.publicKey.toBase58().slice(0, 8)}...
      </button>
    );
  }

  return (
    <div>
      {availableWallets.map(wallet => (
        <button key={wallet.name} onClick={() => connect(wallet)}>
          Connect {wallet.name}
        </button>
      ))}
    </div>
  );
}
```

### useBalance
```typescript
import { useBalance } from '@solana/framework-kit';

function BalanceDisplay({ address }: { address: string }) {
  const { balance, isLoading, error, refetch } = useBalance(address);

  if (isLoading) return <span>Loading...</span>;
  if (error) return <span>Error: {error.message}</span>;

  return (
    <span>
      {(balance / 1_000_000_000).toFixed(4)} SOL
      <button onClick={refetch}>Refresh</button>
    </span>
  );
}
```

### useSolTransfer
```typescript
import { useSolTransfer } from '@solana/framework-kit';

function SendSol() {
  const { transfer, isPending, signature, error } = useSolTransfer();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');

  const handleSend = async () => {
    await transfer({
      to: recipient,
      lamports: parseFloat(amount) * 1_000_000_000,
    });
  };

  return (
    <form onSubmit={e => { e.preventDefault(); handleSend(); }}>
      <input
        value={recipient}
        onChange={e => setRecipient(e.target.value)}
        placeholder="Recipient address"
        disabled={isPending}
      />
      <input
        type="number"
        value={amount}
        onChange={e => setAmount(e.target.value)}
        placeholder="Amount (SOL)"
        disabled={isPending}
      />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Sending...' : 'Send'}
      </button>
      {signature && <p>TX: {signature}</p>}
      {error && <p>Error: {error.message}</p>}
    </form>
  );
}
```

### useSplToken
```typescript
import { useSplToken } from '@solana/framework-kit';

function TokenBalance({ mint, owner }: { mint: string; owner: string }) {
  const { balance, decimals, transfer, isLoading } = useSplToken(mint, owner);

  const displayBalance = balance / Math.pow(10, decimals);

  return <span>{displayBalance.toLocaleString()} tokens</span>;
}
```

### useTransactionPool
```typescript
import { useTransactionPool } from '@solana/framework-kit';

function TransactionManager() {
  const {
    transactions,      // All tracked transactions
    pending,           // Pending transactions
    addTransaction,    // Track new transaction
    retryTransaction,  // Retry failed
    clearCompleted,    // Cleanup
  } = useTransactionPool();

  return (
    <div>
      {transactions.map(tx => (
        <div key={tx.signature}>
          <span>{tx.signature.slice(0, 8)}...</span>
          <span>{tx.status}</span>
          {tx.status === 'failed' && (
            <button onClick={() => retryTransaction(tx.signature)}>
              Retry
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

## Transaction UX Best Practices

### Confirmation States
```typescript
type TxStatus = 'pending' | 'processed' | 'confirmed' | 'finalized' | 'failed';

function TransactionStatus({ signature }: { signature: string }) {
  const [status, setStatus] = useState<TxStatus>('pending');

  useEffect(() => {
    const connection = new Connection(rpcUrl);

    // Subscribe to confirmation
    connection.onSignature(signature, (result) => {
      if (result.err) {
        setStatus('failed');
      } else {
        setStatus('confirmed');
      }
    }, 'confirmed');

    // Check for finalized
    connection.onSignature(signature, () => {
      setStatus('finalized');
    }, 'finalized');
  }, [signature]);

  return (
    <div>
      {status === 'pending' && <Spinner />}
      {status === 'processed' && <span>Processing...</span>}
      {status === 'confirmed' && <span>Confirmed ✓</span>}
      {status === 'finalized' && <span>Finalized ✓✓</span>}
      {status === 'failed' && <span>Failed ✗</span>}
    </div>
  );
}
```

### Error Handling
```typescript
function parseTransactionError(error: unknown): string {
  if (error instanceof Error) {
    // User rejection
    if (error.message.includes('User rejected')) {
      return 'Transaction cancelled by user';
    }

    // Insufficient funds
    if (error.message.includes('insufficient lamports')) {
      return 'Insufficient SOL balance';
    }

    // Account conflicts
    if (error.message.includes('already in use')) {
      return 'Account already exists';
    }

    // Program errors
    const customError = error.message.match(/custom program error: 0x(\w+)/);
    if (customError) {
      const code = parseInt(customError[1], 16);
      return `Program error: ${errorCodeToMessage(code)}`;
    }

    return error.message;
  }

  return 'Unknown error occurred';
}
```

### Form Disable Pattern
```typescript
function TransferForm() {
  const [isPending, setIsPending] = useState(false);

  const handleSubmit = async () => {
    setIsPending(true);
    try {
      const signature = await sendTransaction(tx);
      // Don't re-enable until confirmed
      await confirmTransaction(signature);
    } finally {
      setIsPending(false);
    }
  };

  return (
    <form>
      <input disabled={isPending} />
      <button disabled={isPending}>
        {isPending ? 'Processing...' : 'Submit'}
      </button>
    </form>
  );
}
```

## Data Management

### Subscriptions over Polling
```typescript
function useAccountSubscription(address: string) {
  const [account, setAccount] = useState<AccountInfo<Buffer> | null>(null);

  useEffect(() => {
    const connection = new Connection(rpcUrl, 'confirmed');

    const subscriptionId = connection.onAccountChange(
      new PublicKey(address),
      (accountInfo) => {
        setAccount(accountInfo);
      },
      'confirmed'
    );

    // Cleanup subscription
    return () => {
      connection.removeAccountChangeListener(subscriptionId);
    };
  }, [address]);

  return account;
}
```

### Abort Handles
```typescript
function useAccountData(address: string) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const abortController = new AbortController();

    async function fetchData() {
      try {
        const result = await fetch(`/api/account/${address}`, {
          signal: abortController.signal,
        });
        setData(await result.json());
      } catch (e) {
        if (!abortController.signal.aborted) {
          console.error(e);
        }
      }
    }

    fetchData();

    return () => abortController.abort();
  }, [address]);

  return data;
}
```

## Component Best Practices

### Minimal Client Components
```typescript
// Keep server components where possible
// page.tsx (Server Component)
export default async function Page() {
  const staticData = await fetchStaticData();

  return (
    <div>
      <h1>{staticData.title}</h1>
      {/* Only wallet interaction needs client */}
      <WalletButton />
    </div>
  );
}

// WalletButton.tsx
'use client';
export function WalletButton() {
  // Client-side wallet logic
}
```

### Custom Instruction Building
```typescript
import { createTransferInstruction } from '@solana-program/token';

function buildCustomTransaction() {
  const instruction = createTransferInstruction({
    source: fromAta,
    destination: toAta,
    owner: wallet.publicKey,
    amount: BigInt(1000000),
  });

  return new Transaction().add(instruction);
}
```

## Environment Configuration

```env
# .env.local
NEXT_PUBLIC_RPC_URL=https://api.devnet.solana.com
NEXT_PUBLIC_WS_URL=wss://api.devnet.solana.com
NEXT_PUBLIC_NETWORK=devnet
NEXT_PUBLIC_PROGRAM_ID=YourProgramId...
```

```typescript
// lib/constants.ts
export const NETWORK = process.env.NEXT_PUBLIC_NETWORK as 'devnet' | 'mainnet-beta';
export const RPC_URL = process.env.NEXT_PUBLIC_RPC_URL!;
export const PROGRAM_ID = new PublicKey(process.env.NEXT_PUBLIC_PROGRAM_ID!);
```
