# Transaction Lifecycle, Confirmation, and Fees

## Transaction Fees

### Fee Structure

| Component | Description |
|-----------|-------------|
| Base Fee | 5,000 lamports per signature |
| Priority Fee | Optional: `compute_units × micro_lamports` |
| Total | Base + Priority |

### Fee Payer

The first signer that is also writable pays all fees. Fees are collected **even for failed transactions** after instruction execution begins.

### Fee Distribution

- **50%** burned (removed from circulation)
- **50%** to the block-producing validator

### Setting Priority Fees

```typescript
import { ComputeBudgetProgram } from '@solana/web3.js';

const instructions = [
  // Set compute unit limit
  ComputeBudgetProgram.setComputeUnitLimit({ units: 200_000 }),

  // Set priority fee (micro-lamports per CU)
  ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 1000 }),

  // Your actual instructions...
  yourInstruction,
];
```

### Estimating Priority Fees

```typescript
const recentFees = await connection.getRecentPrioritizationFees({
  lockedWritableAccounts: [accountAddress],
});

// Use median or percentile based on urgency
const medianFee = calculateMedian(recentFees.map((f) => f.prioritizationFee));
```

---

## Compute Budget

### Limits

| Constraint | Limit |
|------------|-------|
| Max CU per transaction | 1,400,000 |
| Default CU per instruction | 200,000 |
| Max call stack depth | 64 frames |
| Max CPI depth | 4 levels |

### Compute Unit Costs (Approximate)

| Operation | Cost |
|-----------|------|
| Instruction invocation | ~100 CU |
| Cross-program invocation | ~1,000 CU |
| SHA256 (per 64 bytes) | ~100 CU |
| Ed25519 signature verify | ~1,800 CU |
| Secp256k1 signature verify | ~2,000 CU |

### Requesting Optimal Compute

Always request the minimum CU needed:

```typescript
// Simulate to get actual CU usage
const simulation = await connection.simulateTransaction(transaction);
const unitsUsed = simulation.value.unitsConsumed;

// Add 10-20% buffer
const requestedUnits = Math.ceil(unitsUsed * 1.1);
```

---

## Commitment Levels

### Levels (Most to Least Finalized)

| Level | Description | Use Case |
|-------|-------------|----------|
| `finalized` | Supermajority confirmed, max lockout reached | Default, highest safety |
| `confirmed` | Supermajority votes received | Balance of speed and safety |
| `processed` | Most recent block (may be skipped) | Progress reporting only |

### Commitment Selection Guide

| Scenario | Recommended |
|----------|-------------|
| Financial transactions | `finalized` |
| User-facing balance updates | `confirmed` |
| Transaction sequence | `confirmed` |
| Real-time progress | `processed` |
| Dependent transactions | `confirmed` or `finalized` |

---

## Confirmation Strategies

### Blockhash Fetching

```typescript
// Recommended: Use confirmed commitment
const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash('confirmed');
```

| Commitment | Pros | Cons |
|------------|------|------|
| `processed` | Maximum prep time | ~5% fork risk |
| `confirmed` | Good balance | Few slots behind |
| `finalized` | Zero fork risk | ~32 slots behind |

### Preflight Commitment

Match preflight commitment to blockhash fetch commitment:

```typescript
const signature = await connection.sendTransaction(transaction, {
  preflightCommitment: 'confirmed',  // Match blockhash commitment
});
```

### Tracking Expiration

```typescript
const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash('confirmed');

// Poll until expired
while (true) {
  const currentHeight = await connection.getBlockHeight('confirmed');
  if (currentHeight > lastValidBlockHeight) {
    console.log('Transaction expired');
    break;
  }
  await sleep(500);
}
```

---

## Retry Strategies

### When Transactions Get Dropped

**Before Processing:**
- UDP packet loss
- Network congestion
- Validator overflow
- RPC pool desync
- Minority fork blockhash

**After Processing:**
- Minority fork consensus loss

### Using maxRetries

```typescript
const signature = await connection.sendTransaction(transaction, {
  maxRetries: 5,  // RPC will retry up to 5 times
});
```

### Manual Rebroadcast Pattern

```typescript
async function sendWithRetry(
  connection: Connection,
  transaction: VersionedTransaction,
  maxAttempts: number = 3
): Promise<string> {
  const { blockhash, lastValidBlockHeight } =
    await connection.getLatestBlockhash('confirmed');

  let attempts = 0;
  let signature: string;

  while (attempts < maxAttempts) {
    signature = await connection.sendTransaction(transaction, {
      skipPreflight: false,
      maxRetries: 0,  // We handle retries
    });

    // Check if confirmed
    const confirmation = await connection.confirmTransaction({
      signature,
      blockhash,
      lastValidBlockHeight,
    }, 'confirmed');

    if (!confirmation.value.err) {
      return signature;
    }

    // Check if blockhash expired
    const currentHeight = await connection.getBlockHeight();
    if (currentHeight > lastValidBlockHeight) {
      throw new Error('Blockhash expired, need to rebuild transaction');
    }

    attempts++;
    await sleep(500 * attempts);  // Exponential backoff
  }

  throw new Error('Max retry attempts reached');
}
```

### Re-Signing Safety

**Critical**: Only re-sign a transaction after confirming the blockhash has expired:

```typescript
// WRONG: Re-signing while blockhash still valid can cause double-execution
if (failed) {
  transaction.sign([payer]);  // Dangerous!
}

// CORRECT: Verify expiration first
const currentHeight = await connection.getBlockHeight();
if (currentHeight > lastValidBlockHeight) {
  // Safe to create new transaction with fresh blockhash
  const newTransaction = buildTransaction(newBlockhash);
  newTransaction.sign([payer]);
}
```

---

## Preflight Checks

### What Preflight Validates

1. **Signature validity** - All required signatures present
2. **Blockhash recency** - Within 150 blocks
3. **Simulation** - Transaction simulates against bank slot

### When to Skip Preflight

```typescript
// Generally keep preflight enabled
const signature = await connection.sendTransaction(transaction, {
  skipPreflight: false,  // Default, recommended
});

// Skip only for specific scenarios:
// - Time-sensitive transactions where you've already simulated
// - When RPC node is known to be behind
const signature = await connection.sendTransaction(transaction, {
  skipPreflight: true,
});
```

---

## Durable Transactions (Nonce Accounts)

For offline signing or cluster instability, use nonce accounts to bypass standard expiration.

### Creating a Nonce Account

```typescript
import {
  SystemProgram,
  NONCE_ACCOUNT_LENGTH,
  NonceAccount,
} from '@solana/web3.js';

const nonceAccount = Keypair.generate();

const createNonceInstruction = SystemProgram.createNonceAccount({
  fromPubkey: payer.publicKey,
  noncePubkey: nonceAccount.publicKey,
  authorizedPubkey: authority.publicKey,
  lamports: await connection.getMinimumBalanceForRentExemption(NONCE_ACCOUNT_LENGTH),
});
```

### Using Nonce in Transaction

```typescript
import { SystemProgram } from '@solana/web3.js';

// Advance nonce must be first instruction
const advanceNonceInstruction = SystemProgram.nonceAdvance({
  noncePubkey: nonceAccount.publicKey,
  authorizedPubkey: authority.publicKey,
});

// Fetch current nonce value
const nonceAccountInfo = await connection.getAccountInfo(nonceAccount.publicKey);
const nonceAccountData = NonceAccount.fromAccountData(nonceAccountInfo.data);
const nonce = nonceAccountData.nonce;

// Build transaction with nonce as blockhash
const transaction = new Transaction({
  recentBlockhash: nonce,  // Use nonce instead of blockhash
  feePayer: payer.publicKey,
});

transaction.add(advanceNonceInstruction);
transaction.add(yourInstruction);
```

---

## sendTransaction Best Practices

### Parameters

```typescript
interface SendTransactionConfig {
  skipPreflight?: boolean;        // Default: false
  preflightCommitment?: string;   // Default: finalized
  maxRetries?: number;            // RPC retry attempts
  minContextSlot?: number;        // Minimum slot for preflight
  encoding?: 'base58' | 'base64'; // Default: base58
}
```

### Recommended Pattern

```typescript
async function sendSafeTransaction(
  connection: Connection,
  transaction: VersionedTransaction
): Promise<string> {
  // 1. Simulate first
  const simulation = await connection.simulateTransaction(transaction, {
    commitment: 'confirmed',
  });

  if (simulation.value.err) {
    throw new Error(`Simulation failed: ${JSON.stringify(simulation.value.err)}`);
  }

  // 2. Send with recommended settings
  const signature = await connection.sendTransaction(transaction, {
    skipPreflight: false,
    preflightCommitment: 'confirmed',
    maxRetries: 3,
  });

  // 3. Confirm
  const confirmation = await connection.confirmTransaction(signature, 'confirmed');

  if (confirmation.value.err) {
    throw new Error(`Transaction failed: ${JSON.stringify(confirmation.value.err)}`);
  }

  return signature;
}
```
