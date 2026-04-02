# Tokens on Solana

## Token Fundamentals

### Token Types

| Type | Description | Example |
|------|-------------|---------|
| **Fungible** | Interchangeable, divisible | USDC, SOL |
| **Non-Fungible (NFT)** | Unique, indivisible | Artwork, collectibles |
| **Semi-Fungible** | Fungible within a class | Game items |

### Token Programs

| Program | ID | Use Case |
|---------|-----|----------|
| Token Program (SPL) | `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` | Classic tokens |
| Token-2022 | `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` | Extended features |

**Recommendation**: Use Token-2022 for new tokens to access extensions.

---

## Core Components

### Mint Account

The **mint account** is the global identifier for a token type. It stores:

| Field | Description |
|-------|-------------|
| `supply` | Total tokens in existence |
| `decimals` | Precision (0-9) |
| `mintAuthority` | Can create new tokens (optional) |
| `freezeAuthority` | Can freeze token accounts (optional) |

### Token Account

**Token accounts** hold token balances for specific owners:

| Field | Description |
|-------|-------------|
| `mint` | Which token this holds |
| `owner` | Who controls this account |
| `amount` | Current balance |
| `delegate` | Optional delegated authority |
| `state` | Initialized, frozen, or uninitialized |
| `delegatedAmount` | Amount delegate can spend |

### Associated Token Account (ATA)

An **ATA** is a deterministically-derived token account using PDAs:

```
seeds = [owner, TOKEN_PROGRAM_ID, mint]
```

Benefits:
- Predictable addresses (no registry needed)
- One canonical account per owner-mint pair
- Simplified token transfers

---

## Token Operations

### Creating Tokens

```bash
# Create new token (CLI)
spl-token create-token

# With specific decimals
spl-token create-token --decimals 6

# Using Token-2022
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
```

### Creating Token Accounts

```bash
# Create associated token account
spl-token create-account <MINT_ADDRESS>

# For another owner
spl-token create-account <MINT_ADDRESS> --owner <OWNER_ADDRESS>
```

### Minting Tokens

```bash
# Mint to your token account
spl-token mint <MINT_ADDRESS> <AMOUNT>

# Mint to specific account
spl-token mint <MINT_ADDRESS> <AMOUNT> <TOKEN_ACCOUNT>
```

### Transferring Tokens

```bash
# Transfer
spl-token transfer <MINT_ADDRESS> <AMOUNT> <RECIPIENT>

# With fee payer
spl-token transfer <MINT_ADDRESS> <AMOUNT> <RECIPIENT> --fee-payer <KEYPAIR>

# Allow creating recipient ATA
spl-token transfer <MINT_ADDRESS> <AMOUNT> <RECIPIENT> --fund-recipient
```

---

## TypeScript Implementation

### Using @solana/spl-token

```typescript
import {
  createMint,
  getOrCreateAssociatedTokenAccount,
  mintTo,
  transfer,
  getMint,
  getAccount,
  TOKEN_PROGRAM_ID,
  TOKEN_2022_PROGRAM_ID,
} from '@solana/spl-token';
import { Connection, Keypair, PublicKey } from '@solana/web3.js';

// Create a new mint
const mint = await createMint(
  connection,
  payer,           // Fee payer
  mintAuthority,   // Mint authority
  freezeAuthority, // Freeze authority (or null)
  9,               // Decimals
  undefined,       // Keypair (optional)
  undefined,       // Confirm options
  TOKEN_2022_PROGRAM_ID  // Use Token-2022
);

// Get or create ATA
const tokenAccount = await getOrCreateAssociatedTokenAccount(
  connection,
  payer,
  mint,
  owner,
  false,           // Allow owner off curve (PDA)
  'confirmed',
  undefined,
  TOKEN_2022_PROGRAM_ID
);

// Mint tokens
await mintTo(
  connection,
  payer,
  mint,
  tokenAccount.address,
  mintAuthority,
  1000000000n,  // Amount (bigint)
  [],           // Multi-signers
  undefined,
  TOKEN_2022_PROGRAM_ID
);

// Transfer tokens
await transfer(
  connection,
  payer,
  sourceTokenAccount,
  destinationTokenAccount,
  owner,
  500000000n,
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID
);
```

### Fetching Token Data

```typescript
// Get mint info
const mintInfo = await getMint(
  connection,
  mintAddress,
  'confirmed',
  TOKEN_2022_PROGRAM_ID
);

console.log('Supply:', mintInfo.supply);
console.log('Decimals:', mintInfo.decimals);
console.log('Mint Authority:', mintInfo.mintAuthority?.toBase58());

// Get token account info
const accountInfo = await getAccount(
  connection,
  tokenAccountAddress,
  'confirmed',
  TOKEN_2022_PROGRAM_ID
);

console.log('Balance:', accountInfo.amount);
console.log('Owner:', accountInfo.owner.toBase58());
```

### Finding Token Accounts

```typescript
import { getAssociatedTokenAddress } from '@solana/spl-token';

// Get ATA address (doesn't create it)
const ataAddress = await getAssociatedTokenAddress(
  mint,
  owner,
  false,  // Allow owner off curve
  TOKEN_2022_PROGRAM_ID
);

// Get all token accounts for owner
const tokenAccounts = await connection.getTokenAccountsByOwner(
  ownerAddress,
  { programId: TOKEN_2022_PROGRAM_ID }
);
```

---

## Token-2022 Extensions

### Available Extensions

| Extension | Purpose |
|-----------|---------|
| **Metadata** | Name, symbol, URI on-chain |
| **Transfer Fee** | Automatic fee on transfers |
| **Interest-Bearing** | Accrue interest over time |
| **Non-Transferable** | Soulbound tokens |
| **Permanent Delegate** | Revocation authority |
| **Confidential Transfers** | Privacy-preserving balances |
| **Transfer Hook** | Custom logic on transfers |
| **Default Account State** | Initialize accounts frozen |
| **Mint Close Authority** | Allow closing mint accounts |
| **Immutable Owner** | Prevent owner reassignment |
| **Memo Required** | Require memo on transfers |
| **CPI Guard** | Restrict CPI access |

### Creating Token with Metadata

```bash
# Create token with metadata extension
spl-token create-token \
  --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-metadata

# Initialize metadata
spl-token initialize-metadata <MINT> "Token Name" "SYMBOL" "https://metadata.uri"

# Update metadata field
spl-token update-metadata <MINT> name "New Name"
```

### Creating Token with Transfer Fee

```typescript
import {
  createInitializeTransferFeeConfigInstruction,
  createInitializeMintInstruction,
  ExtensionType,
  getMintLen,
} from '@solana/spl-token';

// Calculate space with extension
const mintLen = getMintLen([ExtensionType.TransferFeeConfig]);

// Initialize transfer fee config
const initTransferFeeInstruction = createInitializeTransferFeeConfigInstruction(
  mint,
  transferFeeConfigAuthority,  // Can update fee
  withdrawWithheldAuthority,   // Can collect fees
  100,    // Fee basis points (1%)
  BigInt(1000000),  // Max fee
  TOKEN_2022_PROGRAM_ID
);
```

### Detecting Token Program

```typescript
// Check which program owns an account
const accountInfo = await connection.getAccountInfo(mintAddress);

if (accountInfo?.owner.equals(TOKEN_PROGRAM_ID)) {
  console.log('Classic Token Program');
} else if (accountInfo?.owner.equals(TOKEN_2022_PROGRAM_ID)) {
  console.log('Token-2022 Program');
}
```

---

## Anchor Token Integration

### SPL Token

```rust
use anchor_spl::token::{self, Mint, Token, TokenAccount, Transfer};

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,

    #[account(mut)]
    pub to: Account<'info, TokenAccount>,

    pub authority: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

// Transfer CPI
let cpi_accounts = Transfer {
    from: ctx.accounts.from.to_account_info(),
    to: ctx.accounts.to.to_account_info(),
    authority: ctx.accounts.authority.to_account_info(),
};
let cpi_program = ctx.accounts.token_program.to_account_info();
let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);

token::transfer(cpi_ctx, amount)?;
```

### Token-2022 Compatibility

```rust
use anchor_spl::token_interface::{Mint, TokenAccount, TokenInterface};

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    #[account(mut)]
    pub from: InterfaceAccount<'info, TokenAccount>,

    #[account(mut)]
    pub to: InterfaceAccount<'info, TokenAccount>,

    pub mint: InterfaceAccount<'info, Mint>,

    pub authority: Signer<'info>,

    pub token_program: Interface<'info, TokenInterface>,
}
```

### Mint Constraints

```rust
#[account(
    mint::decimals = 9,
    mint::authority = authority,
)]
pub mint: Account<'info, Mint>,

#[account(
    init,
    payer = payer,
    associated_token::mint = mint,
    associated_token::authority = owner,
)]
pub token_account: Account<'info, TokenAccount>,
```

---

## NFTs and Metadata

### Metaplex Token Metadata

For NFTs, use Metaplex Token Metadata standard:

```typescript
import { createNft, mplTokenMetadata } from '@metaplex-foundation/mpl-token-metadata';
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';

const umi = createUmi(rpcEndpoint).use(mplTokenMetadata());

await createNft(umi, {
  mint: mintSigner,
  name: 'My NFT',
  symbol: 'MNFT',
  uri: 'https://arweave.net/metadata.json',
  sellerFeeBasisPoints: 500, // 5% royalty
}).sendAndConfirm(umi);
```

### Compressed NFTs (State Compression)

For large collections, use compressed NFTs:

```typescript
import { createTree, mintV1 } from '@metaplex-foundation/mpl-bubblegum';

// Create merkle tree
await createTree(umi, {
  merkleTree,
  maxDepth: 14,
  maxBufferSize: 64,
}).sendAndConfirm(umi);

// Mint compressed NFT
await mintV1(umi, {
  leafOwner: owner,
  merkleTree,
  metadata: {
    name: 'Compressed NFT',
    uri: 'https://arweave.net/metadata.json',
    // ...
  },
}).sendAndConfirm(umi);
```

---

## Best Practices

### Token Account Management

1. **Always check if ATA exists** before transferring
2. **Use `getOrCreateAssociatedTokenAccount`** for convenience
3. **Consider `--fund-recipient`** flag for transfers to new users

### Program Selection

1. **New tokens**: Use Token-2022 for future-proofing
2. **DeFi integration**: Check protocol compatibility
3. **Extensions needed**: Must use Token-2022

### Security Considerations

1. **Revoke mint authority** for fixed supply tokens
2. **Revoke freeze authority** if not needed
3. **Validate token program** in CPIs (prevent arbitrary program attacks)
4. **Check mint/account relationships** in instructions
