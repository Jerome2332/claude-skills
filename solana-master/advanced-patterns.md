# Advanced Solana Patterns

## Solana Actions and Blinks

**Solana Actions** are specification-compliant APIs that return transactions for user preview and signing across QR codes, buttons, widgets, and websites. **Blinks** (blockchain links) transform any Action into shareable, metadata-rich links.

### Action URL Scheme

```
solana-action:https://actions.example.com/donate
```

### GET Request/Response

```typescript
// GET Response Structure
interface ActionGetResponse {
  type: 'action' | 'completed';
  icon: string;           // SVG, PNG, or WebP URL
  title: string;          // Source identifier
  description: string;    // Action explanation
  label: string;          // Button text (verb-based, ≤5 words)
  disabled?: boolean;
  links?: {
    actions: Array<LinkedAction>;
  };
  error?: ActionError;
}

interface LinkedAction {
  href: string;
  label: string;
  parameters?: Array<ActionParameter>;
}
```

### POST Request/Response

```typescript
// POST Request
interface ActionPostRequest {
  account: string;  // Base58-encoded public key
}

// POST Response
interface ActionPostResponse {
  transaction: string;    // Base64-encoded serialized transaction
  message?: string;       // UTF-8 description
  links?: {
    next: NextActionLink;
  };
}
```

### Required CORS Headers

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET,POST,PUT,OPTIONS
```

### actions.json Configuration

Place at domain root to map routes to Action APIs:

```json
{
  "rules": [
    {
      "pathPattern": "/donate/*",
      "apiPath": "/api/actions/donate/*"
    }
  ]
}
```

Wildcards:
- `*` - Single path segment
- `**` - Multiple segments (must be last)

### Action Identity (Verification)

Include signed memo instruction for attribution:

```
protocol:identity:reference:signature
```

---

## Address Lookup Tables (ALTs)

ALTs compress transaction size by referencing addresses via 1-byte indices instead of 32-byte pubkeys, expanding capacity from 32 to 64 addresses per transaction.

### Creating a Lookup Table

```typescript
import {
  AddressLookupTableProgram,
  Connection,
  Keypair,
  PublicKey,
} from '@solana/web3.js';

const [createInstruction, lookupTableAddress] =
  AddressLookupTableProgram.createLookupTable({
    authority: payer.publicKey,
    payer: payer.publicKey,
    recentSlot: await connection.getSlot(),
  });
```

### Extending with Addresses

```typescript
const extendInstruction = AddressLookupTableProgram.extendLookupTable({
  payer: payer.publicKey,
  authority: payer.publicKey,
  lookupTable: lookupTableAddress,
  addresses: [address1, address2, address3],
});
```

Due to transaction size limits, add ~20 addresses maximum per transaction.

### Using ALTs in Transactions

```typescript
import { TransactionMessage, VersionedTransaction } from '@solana/web3.js';

// Fetch the lookup table
const lookupTableAccount = await connection
  .getAddressLookupTable(lookupTableAddress)
  .then((result) => result.value);

// Build v0 transaction
const messageV0 = new TransactionMessage({
  payerKey: payer.publicKey,
  recentBlockhash: blockhash,
  instructions: instructions,
}).compileToV0Message([lookupTableAccount]);

const transaction = new VersionedTransaction(messageV0);
transaction.sign([payer]);
```

### Key Constraints

- **Requires v0 transactions** - Legacy transactions cannot use ALTs
- **Up to 256 addresses** per table
- **Sign before sending** for VersionedTransaction

---

## State Compression

State compression stores cryptographic hashes of offchain data onchain, enabling massive cost savings (240x for NFTs).

### How It Works

1. Data is hashed to create a `leaf`
2. Adjacent leaves hash together into `branches`
3. Branches hash upward through the tree
4. Final `root hash` is stored onchain

### Concurrent Merkle Trees

Solana's enhanced merkle trees maintain a changelog buffer allowing up to `maxBufferSize` concurrent changes within a single slot.

### Tree Sizing Parameters

| Parameter | Purpose |
|-----------|---------|
| `maxDepth` | Maximum nodes: `2^maxDepth` |
| `maxBufferSize` | Concurrent changes before root invalidation |
| `canopyDepth` | Proof nodes cached onchain |

### Size Examples

| Nodes | Canopy | Approximate Cost |
|-------|--------|------------------|
| 16,384 | 0 | ~0.22 SOL |
| 16,384 | 11 | ~1.13 SOL |
| 1,048,576 | 10 | ~1.67 SOL |
| 1,048,576 | 15 | ~15.81 SOL |

### Cost Calculation

```typescript
import { getConcurrentMerkleTreeAccountSize } from '@solana/spl-account-compression';

const space = getConcurrentMerkleTreeAccountSize(
  maxDepth,
  maxBufferSize,
  canopyDepth
);

const rentLamports = await connection.getMinimumBalanceForRentExemption(space);
```

---

## Versioned Transactions

### Transaction Versions

| Version | Features |
|---------|----------|
| `legacy` | Original format, no ALT support |
| `0` | Address Lookup Tables support |

### RPC Configuration

**Critical**: Set `maxSupportedTransactionVersion` in RPC requests:

```typescript
// web3.js
const transaction = await connection.getTransaction(signature, {
  maxSupportedTransactionVersion: 0,
});

const block = await connection.getBlock(slot, {
  maxSupportedTransactionVersion: 0,
});
```

Without this, requests returning v0 transactions will fail.

### Creating v0 Transactions

```typescript
import {
  TransactionMessage,
  VersionedTransaction,
  Keypair,
} from '@solana/web3.js';

// Build message
const messageV0 = new TransactionMessage({
  payerKey: payer.publicKey,
  recentBlockhash: blockhash,
  instructions: instructions,
}).compileToV0Message();

// Create and sign transaction
const transaction = new VersionedTransaction(messageV0);
transaction.sign([payer]);

// Send
const signature = await connection.sendTransaction(transaction);
```

### Key Difference from Legacy

VersionedTransaction requires signing **before** calling `sendTransaction()`, not passing signers as a parameter.

---

## Token Extensions (Token-2022)

The Token Extensions Program adds features beyond the original Token Program:

### Key Extensions

| Extension | Purpose |
|-----------|---------|
| Metadata | Store name, symbol, URI directly on mint |
| Transfer Fees | Automatic fee collection on transfers |
| Interest-Bearing | Accrue interest over time |
| Non-Transferable | Soulbound tokens |
| Permanent Delegate | Revocation authority |
| Confidential Transfers | Privacy-preserving balances |
| Transfer Hook | Custom logic on transfers |
| Default Account State | Initialize accounts frozen |
| Mint Close Authority | Allow closing mint accounts |

### Creating Token with Metadata

```bash
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-metadata

spl-token initialize-metadata <MINT_ADDRESS> <NAME> <SYMBOL> <URI>
```

### Detecting Token Program

```typescript
import { TOKEN_PROGRAM_ID, TOKEN_2022_PROGRAM_ID } from '@solana/spl-token';

const accountInfo = await connection.getAccountInfo(mintAddress);
const owner = accountInfo?.owner;

const isToken2022 = owner?.equals(TOKEN_2022_PROGRAM_ID);
```

---

## Best Practices

### Optimize Compute Usage

Every transaction has a compute budget. Exceeding it fails the transaction but still collects fees.

```typescript
import { ComputeBudgetProgram } from '@solana/web3.js';

// Set compute unit limit
const modifyComputeUnits = ComputeBudgetProgram.setComputeUnitLimit({
  units: 200_000,
});

// Set priority fee
const addPriorityFee = ComputeBudgetProgram.setComputeUnitPrice({
  microLamports: 1000,
});
```

### Save Bumps in Account State

Store PDA bumps to avoid recomputation:

```rust
#[account]
pub struct Vault {
    pub authority: Pubkey,
    pub bump: u8,  // Save the bump
}
```

### Payer-Authority Pattern

Separate the account funder (payer) from the account manager (authority) for flexibility.

### Use Invariants

Call assertion functions at instruction completion to verify program properties.

### Optimize Indexing Structure

Place static-sized fields at the beginning of structs, variable-sized fields at the end.
