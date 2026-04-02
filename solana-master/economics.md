# Solana Economics: Staking, Inflation, and Rent

## Staking Overview

Staking SOL tokens helps secure the network while earning rewards. It operates as a **shared-risk shared-reward financial model** where token holders and validators align incentives.

### How Staking Works

1. **Token holders** delegate SOL to validators
2. **Validators** process transactions and maintain network
3. **Rewards** are distributed proportionally to stake
4. **Validators** charge commission (percentage of rewards)

Validators with more delegated stake are selected more frequently to write transactions, earning proportionally greater rewards.

---

## Stake Accounts

### What is a Stake Account?

A stake account is a special account type that enables token delegation. Unlike system accounts that only send/receive SOL, stake accounts manage complex delegation operations.

### Account Structure

| Field | Description |
|-------|-------------|
| Address | Unique identifier (not necessarily controlled by keypair) |
| Stake Authority | Controls delegation, splitting, merging |
| Withdraw Authority | Controls withdrawals, liquidation (more privileged) |
| Delegated Stake | Amount delegated to a validator |
| Voter Pubkey | The validator's vote account |
| Activation Epoch | When delegation becomes active |

### Two Signing Authorities

| Authority | Capabilities |
|-----------|--------------|
| **Stake Authority** | Delegate, deactivate, split, merge, change authorities |
| **Withdraw Authority** | Withdraw tokens, change authorities, close account |

**Important**: The withdraw authority is more powerful as it can liquidate tokens. Secure it carefully.

### Key Operations

**Delegation**
- Each stake account delegates to **one validator only**
- To delegate to multiple validators, create multiple stake accounts

**Splitting**
- Divide a stake account into two
- Useful for partial undelegation

**Merging**
- Combine two stake accounts
- Must share identical authorities and lockup terms

**Warmup/Cooldown**
- Delegation changes take multiple epochs to fully activate
- Prevents network destabilization from sudden stake shifts

### Lockups

Stake accounts can have lockups preventing withdrawals until:
- A specific date
- A specific epoch

Delegation and splitting remain permitted while locked.

### Lifecycle

1. **Create** stake account with initial SOL
2. **Delegate** to chosen validator
3. **Wait** for warmup period
4. **Earn** rewards each epoch
5. **Deactivate** when ready to withdraw
6. **Wait** for cooldown period
7. **Withdraw** SOL back to wallet

Accounts with zero balance are removed from tracking.

---

## Creating Stake Accounts

### Using CLI

```bash
# Create stake account
solana create-stake-account stake-keypair.json 10

# Delegate to validator
solana delegate-stake stake-keypair.json <VALIDATOR_VOTE_PUBKEY>

# Check stake account
solana stake-account stake-keypair.json
```

### Using TypeScript

```typescript
import {
  Connection,
  Keypair,
  StakeProgram,
  Authorized,
  Lockup,
  LAMPORTS_PER_SOL,
} from '@solana/web3.js';

// Create stake account
const stakeAccount = Keypair.generate();
const authorizedStaker = wallet.publicKey;
const authorizedWithdrawer = wallet.publicKey;

const createStakeAccountTx = StakeProgram.createAccount({
  fromPubkey: wallet.publicKey,
  stakePubkey: stakeAccount.publicKey,
  authorized: new Authorized(authorizedStaker, authorizedWithdrawer),
  lockup: new Lockup(0, 0, wallet.publicKey), // No lockup
  lamports: 10 * LAMPORTS_PER_SOL,
});

// Delegate to validator
const delegateTx = StakeProgram.delegate({
  stakePubkey: stakeAccount.publicKey,
  authorizedPubkey: authorizedStaker,
  votePubkey: validatorVoteAccount,
});
```

---

## Inflation Schedule

### Key Parameters

| Parameter | Value |
|-----------|-------|
| Initial Inflation Rate | 8% |
| Disinflation Rate | -15% per year |
| Long-term Inflation Rate | 1.5% |

### How It Works

1. Inflation starts at 8% annually
2. Decreases by 15% each year
3. Eventually stabilizes at 1.5%

### Staking Yield Factors

| Factor | Impact |
|--------|--------|
| % of SOL staked | Higher % = lower yield per staker |
| Validator uptime | Better performance = more rewards |
| Commission rate | Lower = more to delegators |
| Epoch duration | Affects compounding frequency |

### Supply Considerations

Projected supply increases are **upper limits** - actual supply may be lower due to:
- Transaction fee burning (50% burned)
- Rent burning
- Slashing (future implementation)
- Lost tokens

---

## Rent

### What is Rent?

Rent is the cost of storing data on-chain. Accounts must maintain a minimum balance to remain **rent-exempt**.

### Rent-Exempt Minimum

```typescript
const minBalance = await connection.getMinimumBalanceForRentExemption(
  accountDataSize
);
```

Formula: `(account_size + 128) × rent_per_byte_per_epoch × 2_years_of_epochs`

### Current Rent Rate

- ~0.00000348 SOL per byte per epoch
- Rent-exempt threshold requires ~2 years of rent

### Common Account Sizes

| Account Type | Size | ~Rent-Exempt |
|--------------|------|--------------|
| System Account (0 data) | 0 bytes | ~0.00089 SOL |
| Token Account | 165 bytes | ~0.002 SOL |
| Mint Account | 82 bytes | ~0.0015 SOL |
| Stake Account | 200 bytes | ~0.0023 SOL |

### Reclaiming Rent

When closing accounts, rent-exempt SOL is returned:

```typescript
// Anchor: close constraint
#[account(mut, close = destination)]
pub account: Account<'info, Data>,

// Transfer remaining lamports to destination
```

---

## Transaction Fees

### Fee Structure

| Component | Amount |
|-----------|--------|
| Base fee | 5,000 lamports (~$0.0000075) per signature |
| Priority fee | Variable (micro-lamports per CU) |

### Fee Distribution

- **50% burned** - Permanently removed from supply
- **50% to validator** - Block producer incentive

### Priority Fee Calculation

```
priority_fee = compute_unit_limit × compute_unit_price
```

Example: 200,000 CU × 1,000 micro-lamports = 200,000,000 micro-lamports = 0.0002 SOL

---

## Validator Selection

### Finding Validators

```bash
# List all validators
solana validators

# Check block production
solana block-production

# View specific validator
solana validator-info get <VALIDATOR_PUBKEY>
```

### Selection Criteria

| Factor | Why It Matters |
|--------|----------------|
| **Commission** | Lower = more rewards to you |
| **Uptime** | Higher = more consistent rewards |
| **Stake concentration** | Decentralization is healthier |
| **Vote account age** | Established track record |
| **Skip rate** | Lower = better performance |

### Resources

- [Solana Beach](https://solanabeach.io/validators)
- [Validators.app](https://validators.app)
- [StakeView](https://stakeview.app)

---

## Economic Security Model

### Proof of Stake

Validators stake SOL as collateral. Malicious behavior can result in:
- Lost rewards
- Reputation damage
- (Future) Slashing of staked SOL

### Network Security

Total staked SOL creates economic security:
- More stake = higher cost to attack
- Distributed stake = censorship resistance
- Aligned incentives = honest behavior

### Slashing (Future)

Not currently implemented, but planned:
- Penalties for double-signing
- Penalties for extended downtime
- Penalties for malicious behavior

---

## Quick Reference

### Staking Commands

```bash
# Create stake account
solana create-stake-account <KEYPAIR> <AMOUNT>

# Delegate
solana delegate-stake <STAKE_ACCOUNT> <VOTE_ACCOUNT>

# Deactivate (begin cooldown)
solana deactivate-stake <STAKE_ACCOUNT>

# Withdraw (after cooldown)
solana withdraw-stake <STAKE_ACCOUNT> <RECIPIENT> <AMOUNT>

# View stake account
solana stake-account <STAKE_ACCOUNT>

# View stake history
solana stake-history
```

### Key Epochs

| Event | Epochs |
|-------|--------|
| Warmup period | ~1 epoch (~2-3 days) |
| Cooldown period | ~1 epoch (~2-3 days) |
| Epoch duration | ~2-3 days |
| Rewards distribution | End of each epoch |
