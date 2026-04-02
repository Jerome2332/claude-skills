# Programs with Anchor (Default Choice)

## When to Use Anchor

Use Anchor by default when:
- You want fast iteration with reduced boilerplate
- You want an IDL and TypeScript client story out of the box
- You want mature testing and workspace tooling
- You need built-in security through automatic account validation

## Core Advantages

- **Reduced Boilerplate**: Abstracts repetitive account management, instruction serialization, and error handling
- **Built-in Security**: Automatic account-ownership verification and data validation
- **IDL Generation**: Automatic interface definition for client generation

## Anchor Version

- Write all code for the latest stable Anchor (currently 0.32.x)
- Do not use unnecessary macros that are not needed in the latest version
- Use AVM (Anchor Version Manager) for reproducible builds

## Anchor Has Silly Defaults - Fix Them

Every project needs an IDL. Add to `Cargo.toml`:

```toml
[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
```

If using SPL Tokens (almost every Anchor project):

```toml
[dependencies]
anchor-lang = "0.32.1"
anchor-spl = "0.32.1"
```

---

## Core Macros

### `declare_id!()`

Declares the onchain address where the program resides.

```rust
declare_id!("YourProgramId11111111111111111111111111111");
```

**Never modify the program ID** in `lib.rs` or `Anchor.toml` when making changes unless you're redeploying to a new address.

### `#[program]`

Marks the module containing every instruction entrypoint and business-logic function.

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(context: Context<Initialize>, data: u64) -> Result<()> {
        context.accounts.account.data = data;
        Ok(())
    }
}
```

### `#[derive(Accounts)]`

Lists accounts an instruction requires and enforces constraints:
- Declares all necessary accounts for specific instructions
- Enforces constraint checks automatically to block bugs and exploits
- Generates helper methods for safe account access and mutation

### `#[error_code]`

Enables custom, human-readable error types:

```rust
#[error_code]
pub enum MyError {
    #[msg("Custom error message")]
    CustomError,
    #[msg("Value too large: {0}")]
    ValueError(u64),
}
```

---

## Account Types

| Type | Purpose |
|------|---------|
| `Signer<'info>` | Verifies the account signed the transaction |
| `SystemAccount<'info>` | Confirms System Program ownership |
| `Program<'info, T>` | Validates executable program accounts |
| `Account<'info, T>` | Typed program account with automatic validation |
| `UncheckedAccount<'info>` | Raw account requiring manual validation |

---

## Space Calculation (CRITICAL - NO MAGIC NUMBERS)

**Do not use magic numbers anywhere.** Never write `8 + 32` or similar.

### Correct Pattern

```rust
#[derive(InitSpace)]
#[account]
pub struct UserProfile {
    pub authority: Pubkey,

    #[max_len(50)]
    pub username: String,

    pub bump: u8,
}

#[derive(Accounts)]
pub struct InitializeProfile<'info> {
    #[account(
        init,
        payer = authority,
        space = UserProfile::DISCRIMINATOR.len() + UserProfile::INIT_SPACE,
        seeds = [b"profile", authority.key().as_ref()],
        bump
    )]
    pub profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

### Rules

- All structs should have `#[derive(InitSpace)]` to get the `INIT_SPACE` trait
- Use `space = SomeStruct::DISCRIMINATOR.len() + SomeStruct::INIT_SPACE`
- Do NOT make constants for sizes of data structures
- When making structs, ensure Strings and Vectors have a `max_len` attribute
- Vectors have two numbers for `max_len`: max vector length, then max item length

---

## Account Constraints

### Initialization

```rust
#[account(
    init,
    payer = payer,
    space = CustomAccount::DISCRIMINATOR.len() + CustomAccount::INIT_SPACE
)]
pub account: Account<'info, CustomAccount>,
```

### PDA Validation

```rust
#[account(
    seeds = [b"vault", owner.key().as_ref()],
    bump
)]
pub vault: SystemAccount<'info>,
```

### Ownership and Relationships

```rust
#[account(
    has_one = authority @ CustomError::InvalidAuthority,
    constraint = account.is_active @ CustomError::AccountInactive
)]
pub account: Account<'info, CustomAccount>,
```

### Reallocation

```rust
#[account(
    mut,
    realloc = new_space,
    realloc::payer = payer,
    realloc::zero = true  // Clear old data when shrinking
)]
pub account: Account<'info, CustomAccount>,
```

### Closing Accounts

```rust
#[account(
    mut,
    close = destination
)]
pub account: Account<'info, CustomAccount>,
```

### Constraint Formatting

Use a newline after each key in account constraints:

```rust
// CORRECT - Readable
#[account(
    init,
    payer = authority,
    space = Config::DISCRIMINATOR.len() + Config::INIT_SPACE,
    seeds = [b"config"],
    bump
)]
pub config: Account<'info, Config>,

// WRONG - Hard to read
#[account(init, payer = authority, space = Config::DISCRIMINATOR.len() + Config::INIT_SPACE, seeds = [b"config"], bump)]
pub config: Account<'info, Config>,
```

---

## Bumps

Use `context.bumps.foo` not `context.bumps.get("foo").unwrap()`:

```rust
// CORRECT
pub fn initialize(context: Context<Initialize>) -> Result<()> {
    context.accounts.vault.bump = context.bumps.vault;
    Ok(())
}

// WRONG - Outdated pattern
pub fn initialize(context: Context<Initialize>) -> Result<()> {
    context.accounts.vault.bump = *context.bumps.get("vault").unwrap();
    Ok(())
}
```

---

## PDA Management

- Add `pub bump: u8` to every struct stored in a PDA
- Save the bump when the struct is created
- Use stored bump for CPIs to avoid recalculation

```rust
#[account]
#[derive(InitSpace)]
pub struct Vault {
    pub authority: Pubkey,
    pub bump: u8,
}

pub fn initialize(context: Context<Initialize>) -> Result<()> {
    let vault = &mut context.accounts.vault;
    vault.authority = context.accounts.authority.key();
    vault.bump = context.bumps.vault;
    Ok(())
}
```

---

## Project Structure

```
programs/my-program/
├── src/
│   ├── lib.rs
│   ├── state/
│   │   ├── mod.rs
│   │   └── user_profile.rs
│   ├── instructions/          # or handlers/
│   │   ├── mod.rs
│   │   ├── initialize.rs
│   │   └── admin/            # Admin-only handlers
│   │       ├── mod.rs
│   │       └── update_config.rs
│   └── errors.rs
├── Cargo.toml
└── Xargo.toml
```

### Naming Conventions

- Account constraint structs should end with `AccountConstraints`:

```rust
// CORRECT
pub struct InitializeAccountConstraints<'info> { ... }

// WRONG - Same name as function
pub struct Initialize<'info> { ... }
```

---

## Cross-Program Invocations (CPIs)

### Basic CPI

```rust
let cpi_accounts = Transfer {
    from: context.accounts.from.to_account_info(),
    to: context.accounts.to.to_account_info(),
};
let cpi_program = context.accounts.system_program.to_account_info();
let cpi_context = CpiContext::new(cpi_program, cpi_accounts);

transfer(cpi_context, amount)?;
```

### PDA-Signed CPIs

```rust
let seeds = &[b"vault".as_ref(), &[context.bumps.vault]];
let signer = &[&seeds[..]];
let cpi_context = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer);
```

---

## Token Accounts

### SPL Token

```rust
#[account(
    mint::decimals = 9,
    mint::authority = authority,
)]
pub mint: Account<'info, Mint>,

#[account(
    mut,
    associated_token::mint = mint,
    associated_token::authority = owner,
)]
pub token_account: Account<'info, TokenAccount>,
```

### Token-2022 Compatibility

Use `InterfaceAccount` for dual compatibility:

```rust
use anchor_spl::token_interface::{Mint, TokenAccount};

pub mint: InterfaceAccount<'info, Mint>,
pub token_account: InterfaceAccount<'info, TokenAccount>,
pub token_program: Interface<'info, TokenInterface>,
```

---

## System Functions

Use `Clock::get()?;` not `anchor_lang::solana_program::clock`:

```rust
// CORRECT
let clock = Clock::get()?;
let timestamp = clock.unix_timestamp;

// WRONG
let clock = anchor_lang::solana_program::clock::Clock::get()?;
```

---

## Account Discriminators

Default discriminators use `sha256("account:<StructName>")[0..8]`. Custom discriminators (Anchor 0.31+):

```rust
#[account(discriminator = 1)]
pub struct Escrow { ... }
```

**Constraints:**
- Discriminators must be unique across your program
- Using `[1]` prevents using `[1, 2, ...]` which also start with `1`
- `[0]` conflicts with uninitialized accounts

---

## LazyAccount (Anchor 0.31+)

Heap-allocated, read-only account access for efficient memory usage:

```toml
# Cargo.toml
anchor-lang = { version = "0.32.1", features = ["lazy-account"] }
```

```rust
pub account: LazyAccount<'info, CustomAccountType>,

pub fn handler(context: Context<MyInstruction>) -> Result<()> {
    let value = context.accounts.account.get_value()?;
    Ok(())
}
```

**Note:** LazyAccount is read-only. After CPIs, use `unload()` to refresh cached values.

---

## Zero-Copy Accounts

For accounts exceeding stack/heap limits:

```rust
#[account(zero_copy)]
pub struct LargeAccount {
    pub data: [u8; 10000],
}
```

Accounts under 10,240 bytes use `init`; larger accounts require external creation then `zero` constraint initialization.

---

## Error Handling

Return useful error messages:

```rust
#[error_code]
pub enum MyError {
    #[msg("Insufficient funds for this operation")]
    InsufficientFunds,
    #[msg("Invalid parameter: value must be between 1 and 100")]
    InvalidParameter,
    #[msg("Account not initialized")]
    NotInitialized,
}

// Usage
require!(value > 0, MyError::InvalidParameter);
require!(balance >= amount, MyError::InsufficientFunds);
```

---

## Compatibility Notes for Anchor 0.32.0

To resolve build conflicts:

```bash
cargo update base64ct --precise 1.6.0
cargo update constant_time_eq --precise 0.4.1
cargo update blake3 --precise 1.5.5
```

If you encounter `solana-program` conflicts, add to your program's `Cargo.toml`:

```toml
[dependencies]
solana-program = "3"
```

---

## Security Best Practices

### Account Validation
- Use typed accounts (`Account<'info, T>`) over `UncheckedAccount` when possible
- Always validate signer requirements explicitly
- Use `has_one` for ownership relationships
- Validate PDA seeds and bumps

### CPI Safety
- Use `Program<'info, T>` to validate CPI targets (prevents arbitrary CPI attacks)
- Never pass extra privileges to CPI callees
- Prefer explicit program IDs for known CPIs

### Common Gotchas
- **Avoid `init_if_needed`**: Permits reinitialization attacks
- **Legacy IDL formats**: Ensure tooling agrees on format (pre-0.30 vs new spec)
- **PDA seeds**: Ensure all seed material is stable and canonical

---

## Testing

- Use `anchor test` for end-to-end tests
- Prefer Mollusk or LiteSVM for fast unit tests
- Use Surfpool for integration tests with mainnet state

See [testing.md](testing.md) for detailed testing patterns.

---

## IDL and Clients

- Treat the program's IDL as a product artifact
- Prefer generating Kit-native clients via Codama
- If using Anchor TS client in Kit-first app, put it behind web3-compat boundary

See [idl-codegen.md](idl-codegen.md) for the client generation workflow.
