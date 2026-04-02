---
name: carbon-decoder
description: Build correct Carbon framework decoders for Solana programs. Use when implementing a new decoder crate, debugging silent parse failures, or adding instruction/account support to an existing Carbon decoder. Covers codegen CLI, trait implementation, discriminators, account vs instruction decoding, ArrangeAccounts, events, and pipeline wiring.
metadata:
  filePattern: "**/decoders/**/*.rs,**/carbon-*-decoder/**/*.rs"
  priority: 90
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/carbon-decoder/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/carbon-decoder/feedback.log
```

Only log general preferences, not task-specific details.

---

# Carbon Decoder Builder

You are an expert in the [Carbon](https://github.com/sevenlabs-hq/carbon) Solana indexing framework. You build decoder crates that are correct, complete, and integrate cleanly into the Carbon pipeline.

**Framework version:** `carbon-core` v0.12.0+
**Solana SDK:** Agave v3.x (`solana-account`, `solana-pubkey`, `solana-instruction` as separate crates — NOT the monolithic `solana-sdk` 1.x)

---

## Step 0: Always Use the CLI Codegen First

Before writing any Rust by hand, check if an IDL exists. The CLI generates the entire decoder crate structure from an Anchor or Codama IDL.

```bash
# From on-chain address (preferred):
npx @sevenlabs-hq/carbon-cli parse \
  --program-id <PROGRAM_ADDRESS> \
  --rpc <RPC_URL> \
  --output ./decoders/my-program-decoder

# From local IDL file:
npx @sevenlabs-hq/carbon-cli parse \
  --idl ./my-program.json \
  --output ./decoders/my-program-decoder

# Scaffold a full indexer project:
npx @sevenlabs-hq/carbon-cli scaffold
```

**When no IDL is available** (native programs, closed-source programs, programs with no on-chain IDL): write the decoder manually using the patterns below.

---

## Canonical Decoder Crate Structure

Every decoder follows this layout exactly. No exceptions.

```
decoders/<program-name>-decoder/
├── Cargo.toml
├── README.md
└── src/
    ├── lib.rs                    # Decoder struct + PROGRAM_ID const + mod declarations
    ├── accounts/
    │   ├── mod.rs                # AccountDecoder impl + XAccount enum
    │   └── <account_name>.rs    # One file per account type
    ├── instructions/
    │   ├── mod.rs                # InstructionDecoder impl + XInstruction enum
    │   └── <instruction_name>.rs # One file per instruction
    ├── events/
    │   ├── mod.rs                # CpiEvent variant + event enum
    │   └── <event_name>.rs      # One file per event type
    └── types/
        └── mod.rs                # Shared custom types (enums, sub-structs)
```

`graphql/` is optional and gated behind a `graphql` crate feature.

---

## Cargo.toml Template

```toml
[package]
name = "carbon-<program-name>-decoder"
version = "0.1.0"
edition = "2021"
license = "MIT"

[lib]
crate-type = ["rlib"]  # REQUIRED — do not omit

[features]
default = []
graphql = ["async-graphql"]

[dependencies]
carbon-core = { version = "0.12", features = [] }
borsh = { version = "1", features = ["derive"] }
solana-pubkey = "3"
solana-instruction = "3"
solana-account = "3"
serde = { version = "1", features = ["derive"], optional = true }

[dev-dependencies]
carbon-test-utils = "0.12"
```

---

## lib.rs — Always Minimal

```rust
use solana_pubkey::Pubkey;

pub struct ProgramNameDecoder;

pub const PROGRAM_ID: Pubkey =
    solana_pubkey::Pubkey::from_str_const("PROGRAM_ADDRESS_BASE58_HERE");

pub mod accounts;
pub mod events;
pub mod instructions;
pub mod types;

#[cfg(feature = "graphql")]
pub mod graphql;
```

The decoder struct is a zero-sized type (ZST). It holds no state. The same instance is passed to both `.instruction()` and `.account()` in the pipeline.

---

## Instructions

### instructions/mod.rs — InstructionDecoder impl

```rust
use carbon_core::instruction::{DecodedInstruction, InstructionDecoder};
use solana_instruction::Instruction;

use super::ProgramNameDecoder;
pub mod buy;
pub mod sell;
pub mod initialize;

// Variant names match the instruction struct names exactly
#[derive(Debug, Clone)]
pub enum ProgramNameInstruction {
    Buy(buy::Buy),
    Sell(sell::Sell),
    Initialize(initialize::Initialize),
    CpiEvent(events::ProgramNameEvent),  // if the program emits CPI log events
}

impl<'a> InstructionDecoder<'a> for ProgramNameDecoder {
    type InstructionType = ProgramNameInstruction;

    fn decode_instruction(
        &self,
        instruction: &'a Instruction,
    ) -> Option<DecodedInstruction<Self::InstructionType>> {
        // Always check program_id first
        if instruction.program_id != super::PROGRAM_ID {
            return None;
        }

        // Try each instruction in sequence — first discriminator match wins
        if let Some(decoded) = buy::Buy::decode(&instruction.data) {
            return Some(DecodedInstruction {
                program_id: instruction.program_id,
                data: ProgramNameInstruction::Buy(decoded),
                accounts: instruction.accounts.clone(),
            });
        }
        if let Some(decoded) = sell::Sell::decode(&instruction.data) {
            return Some(DecodedInstruction {
                program_id: instruction.program_id,
                data: ProgramNameInstruction::Sell(decoded),
                accounts: instruction.accounts.clone(),
            });
        }
        // ... repeat for all variants
        None
    }
}
```

### instructions/<instruction_name>.rs — One Per Instruction

#### For Anchor programs (8-byte SHA256 discriminator):

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use carbon_core::deserialize::ArrangeAccounts;
use solana_instruction::AccountMeta;
use solana_pubkey::Pubkey;

// Instruction data struct — field names and types MUST match the IDL exactly
#[derive(Debug, Clone, BorshSerialize, BorshDeserialize, PartialEq)]
pub struct Buy {
    pub amount: u64,
    pub max_sol_cost: u64,
}

// Named accounts struct — one field per positional account in the IDL
#[derive(Debug, Clone)]
pub struct BuyAccounts {
    pub global: Pubkey,
    pub fee_recipient: Pubkey,
    pub mint: Pubkey,
    pub bonding_curve: Pubkey,
    pub associated_bonding_curve: Pubkey,
    pub associated_user: Pubkey,
    pub user: Pubkey,
    pub system_program: Pubkey,
    pub token_program: Pubkey,
    pub rent: Pubkey,
    pub event_authority: Pubkey,
    pub program: Pubkey,
    pub remaining: Vec<AccountMeta>,  // captures extra CPI accounts
}

impl Buy {
    // Discriminator bytes: SHA256("global:buy")[0..8]
    // Compute with: sha256("global:<snake_case_name>") then take first 8 bytes
    // NEVER compute at runtime — hardcode the bytes from the IDL or CLI output
    pub fn decode(data: &[u8]) -> Option<Self> {
        if data.len() < 8 {
            return None;
        }
        let discriminator = &data[0..8];
        if discriminator != [102, 6, 61, 18, 1, 218, 235, 234] {
            return None;
        }
        let mut data_slice = &data[8..];
        borsh::BorshDeserialize::deserialize(&mut data_slice).ok()
    }
}

impl ArrangeAccounts for Buy {
    type ArrangedAccounts = BuyAccounts;

    fn arrange_accounts(accounts: &[AccountMeta]) -> Option<Self::ArrangedAccounts> {
        let mut iter = accounts.iter();
        Some(BuyAccounts {
            global:                   iter.next()?.pubkey,
            fee_recipient:            iter.next()?.pubkey,
            mint:                     iter.next()?.pubkey,
            bonding_curve:            iter.next()?.pubkey,
            associated_bonding_curve: iter.next()?.pubkey,
            associated_user:          iter.next()?.pubkey,
            user:                     iter.next()?.pubkey,
            system_program:           iter.next()?.pubkey,
            token_program:            iter.next()?.pubkey,
            rent:                     iter.next()?.pubkey,
            event_authority:          iter.next()?.pubkey,
            program:                  iter.next()?.pubkey,
            remaining: iter.cloned().collect(),
        })
    }
}
```

#### For native (non-Anchor) programs (1-byte opcode dispatch):

```rust
impl SomeInstruction {
    pub fn decode(data: &[u8]) -> Option<Self> {
        if data.is_empty() || data[0] != 0x03 {  // opcode byte, not 8-byte discriminator
            return None;
        }
        let mut data_slice = &data[1..];
        borsh::BorshDeserialize::deserialize(&mut data_slice).ok()
    }
}
```

---

## Accounts

### accounts/mod.rs — AccountDecoder impl

```rust
use carbon_core::account::{AccountDecoder, DecodedAccount};
use solana_account::Account;

use super::ProgramNameDecoder;
pub mod bonding_curve;
pub mod global;

#[derive(Debug, Clone)]
pub enum ProgramNameAccount {
    BondingCurve(Box<bonding_curve::BondingCurve>),  // Box large structs to avoid stack overflow
    Global(global::Global),
}

impl<'a> AccountDecoder<'a> for ProgramNameDecoder {
    type AccountType = ProgramNameAccount;

    fn decode_account(
        &self,
        account: &'a Account,
    ) -> Option<DecodedAccount<Self::AccountType>> {
        // Always check owner first
        if account.owner != super::PROGRAM_ID {
            return None;
        }

        // Try each account type — first discriminator match wins
        if let Some(decoded) = bonding_curve::BondingCurve::decode(&account.data) {
            return Some(DecodedAccount {
                lamports: account.lamports,
                data: ProgramNameAccount::BondingCurve(Box::new(decoded)),
                owner: account.owner,
                executable: account.executable,
                rent_epoch: account.rent_epoch,
            });
        }
        if let Some(decoded) = global::Global::decode(&account.data) {
            return Some(DecodedAccount {
                lamports: account.lamports,
                data: ProgramNameAccount::Global(decoded),
                owner: account.owner,
                executable: account.executable,
                rent_epoch: account.rent_epoch,
            });
        }
        None
    }
}
```

### accounts/<account_name>.rs — One Per Account Type

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_pubkey::Pubkey;

#[derive(Debug, Clone, BorshSerialize, BorshDeserialize, PartialEq)]
pub struct BondingCurve {
    pub virtual_token_reserves: u64,
    pub virtual_sol_reserves: u64,
    pub real_token_reserves: u64,
    pub real_sol_reserves: u64,
    pub token_total_supply: u64,
    pub complete: bool,
}

impl BondingCurve {
    // Discriminator: SHA256("account:BondingCurve")[0..8]
    // PascalCase account type name — matches the Rust struct name
    pub fn decode(data: &[u8]) -> Option<Self> {
        if data.len() < 8 {
            return None;
        }
        let discriminator = &data[0..8];
        if discriminator != [23, 183, 248, 55, 96, 216, 172, 96] {
            return None;
        }
        let mut data_slice = &data[8..];
        borsh::BorshDeserialize::deserialize(&mut data_slice).ok()
    }
}
```

---

## Events (CPI Log Events)

Anchor programs emit events as CPI calls to the event authority program. Carbon surfaces these as a `CpiEvent` instruction variant — NOT as a separate pipe.

### events/mod.rs

```rust
pub mod trade_event;
pub mod create_event;
pub mod complete_event;

#[derive(Debug, Clone)]
pub enum ProgramNameEvent {
    TradeEvent(trade_event::TradeEvent),
    CreateEvent(create_event::CreateEvent),
    CompleteEvent(complete_event::CompleteEvent),
}
```

### events/<event_name>.rs

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use carbon_core::deserialize::CarbonDeserialize;
use solana_pubkey::Pubkey;

// CarbonDeserialize proc macro generates the discriminator check from the attribute
#[derive(Debug, Clone, BorshSerialize, BorshDeserialize, CarbonDeserialize, PartialEq)]
#[carbon(discriminator = "0xe445a52e51cb9a1dbddb7fd34ee661ee")]
pub struct TradeEvent {
    pub mint: Pubkey,
    pub sol_amount: u64,
    pub token_amount: u64,
    pub is_buy: bool,
    pub user: Pubkey,
    pub timestamp: i64,
    pub virtual_sol_reserves: u64,
    pub virtual_token_reserves: u64,
    pub real_sol_reserves: u64,
    pub real_token_reserves: u64,
}
```

### Decoding events in a processor

Events are accessed through `InstructionMetadata`, not as separate decoded structs:

```rust
// Inside your instruction processor's process() method:
async fn process(
    &mut self,
    (metadata, instruction, nested, raw): Self::InputType,
    metrics: Arc<MetricsCollection>,
) -> CarbonResult<()> {
    match instruction.data {
        ProgramNameInstruction::CpiEvent(event) => {
            match event {
                ProgramNameEvent::TradeEvent(e) => {
                    // handle trade event
                }
                _ => {}
            }
        }
        _ => {}
    }
    Ok(())
}
```

---

## Discriminator Reference

### Computing Discriminators (for verification — hardcode results, never compute at runtime)

**TypeScript (run once to get the bytes):**
```typescript
import { createHash } from 'crypto';

// Anchor instruction discriminator
const ixDisc = (name: string) =>
  [...createHash('sha256').update(`global:${name}`).digest()].slice(0, 8);

// Anchor account discriminator
const acctDisc = (name: string) =>
  [...createHash('sha256').update(`account:${name}`).digest()].slice(0, 8);

console.log('buy:', ixDisc('buy'));          // [102, 6, 61, 18, 1, 218, 235, 234]
console.log('BondingCurve:', acctDisc('BondingCurve')); // [23, 183, 248, 55, 96, 216, 172, 96]
```

**Rust (for one-off verification):**
```rust
use sha2::{Digest, Sha256};
let mut hasher = Sha256::new();
hasher.update(b"global:buy");
let result = hasher.finalize();
println!("{:?}", &result[..8]);
```

### Native program dispatch
Non-Anchor programs use `data[0]` as the opcode, not an 8-byte discriminator:
```rust
if data[0] != OPCODE_BUY { return None; }
```
Raydium AMM V4, Orca Whirlpool (some instructions), and most older SPL programs use this pattern.

---

## Pipeline Integration

### Single-program indexer

```rust
use carbon_core::{
    metrics::LogMetrics,
    pipeline::{Pipeline, ShutdownStrategy},
};
use std::sync::Arc;

let pipeline = Pipeline::builder()
    .datasource(datasource)
    .metrics(Arc::new(LogMetrics::new()))
    .instruction(MyDecoder, MyInstructionProcessor)  // decoder + processor pair
    .account(MyDecoder, MyAccountProcessor)          // same ZST decoder, reused
    .shutdown_strategy(ShutdownStrategy::Immediate)
    .build()?;

pipeline.run().await?;
```

### Multi-program indexer with `instruction_decoder_collection_fast!`

**Always use `_fast!` — the non-fast version does sequential O(N) scan through all programs.**

```rust
use carbon_core::collection::instruction_decoder_collection_fast;

instruction_decoder_collection_fast!(
    AllInstructions,    // enum name for all instruction variants
    AllInstructionTypes, // data-free mirror enum (for filtering/logging)
    AllPrograms,         // enum of program IDs
    Pumpfun  => carbon_pumpfun_decoder::PROGRAM_ID
             => carbon_pumpfun_decoder::PumpfunDecoder
             => carbon_pumpfun_decoder::instructions::PumpfunInstruction,
    Jupiter  => carbon_jupiter_swap_decoder::PROGRAM_ID
             => carbon_jupiter_swap_decoder::JupiterSwapDecoder
             => carbon_jupiter_swap_decoder::instructions::JupiterSwapInstruction,
    RaydiumAmm => carbon_raydium_amm_v4_decoder::PROGRAM_ID
               => carbon_raydium_amm_v4_decoder::RaydiumAmmV4Decoder
               => carbon_raydium_amm_v4_decoder::instructions::RaydiumAmmV4Instruction,
);

// Use in pipeline:
.transaction(MyMultiProgramProcessor, None::<TransactionSchema<AllInstructions>>)
```

### Processor input types

**Instruction processor:**
```rust
type Input = (
    InstructionMetadata,
    DecodedInstruction<XInstruction>,
    NestedInstructions,
    solana_instruction::Instruction,  // raw instruction, for fallback parsing
);
```

**Account processor:**
```rust
type Input = (
    AccountMetadata,
    DecodedAccount<XAccount>,
    solana_account::Account,  // raw account
);
```

### Filters (optional)

```rust
.instruction_with_filters(decoder, processor, vec![
    Box::new(MyProgramIdFilter),
])
```

`Filter` trait has default-`true` methods for all five filter points. Override only what you need.

---

## Using the `CarbonDeserialize` Derive Macro (Alternative to Manual `decode()`)

Instead of writing `decode()` by hand, use the proc macro:

```rust
use carbon_core::deserialize::CarbonDeserialize;

#[derive(Debug, Clone, BorshSerialize, BorshDeserialize, CarbonDeserialize, PartialEq)]
#[carbon(discriminator = "0x6606_3d12_01da_ebea")]  // hex representation of the 8 bytes
pub struct Buy {
    pub amount: u64,
    pub max_sol_cost: u64,
}
```

The macro generates an impl that checks the discriminator then Borsh-deserializes. `BorshDeserialize` is a **required supertrait** — `CarbonDeserialize` will not compile without it.

---

## Critical Gotchas

**1. Discriminator bytes are hardcoded, never computed at runtime.**
Compute them once (TypeScript script above), paste the array. If the program upgrades and changes its discriminators, you must regenerate.

**2. `ArrangeAccounts` is positional — order is everything.**
`next_account(&mut iter)` extracts accounts in order. If the on-chain program adds an account at position 3, all fields at position 4+ shift. Always verify against the current IDL or a live transaction.

**3. `remaining` in `XAccounts` is `Vec<AccountMeta>`, not `Vec<Pubkey>`.**
It captures extra accounts beyond the named ones. Anchor CPI calls often append extra accounts.

**4. Events live inside `CpiEvent` instruction variant — there is no separate event pipe.**
Anchor CPI events are emitted as calls to the event authority. Handle them by matching `XInstruction::CpiEvent(e)` inside the instruction processor.

**5. Account discriminator ordering matters when two types have the same prefix.**
The first `decode()` that succeeds wins. Keep more specific (longer-data) types before more general ones.

**6. Box large account structs in the enum variant.**
`XAccount::SomeLargeAccount(Box<SomeLargeAccount>)` — prevents stack overflow from large enum discriminant trees.

**7. `[lib] crate-type = ["rlib"]` is required in Cargo.toml.**
Without it the crate cannot be used as a dependency.

**8. `instruction_decoder_collection!` (non-fast) is deprecated.**
Always use `instruction_decoder_collection_fast!` with the 4-part `Variant => PROGRAM_ID => DecoderExpr => InstructionType` syntax.

**9. Solana SDK is v3 (Agave), not v1.**
Import `solana-account`, `solana-pubkey`, `solana-instruction` as separate crates. The monolithic `solana-sdk` 1.x is not compatible.

**10. `decode_log_events::<T>()` adjusts for precompile instructions.**
It handles Ed25519/Secp256k1/Secp256r1 precompile programs that execute without emitting invoke logs. Use `InstructionMetadata::decode_log_events::<TradeEvent>()` inside a processor to extract Anchor events from logs as an alternative to the `CpiEvent` pattern.

---

## Verification Checklist

Before marking a decoder complete:

- [ ] `PROGRAM_ID` verified against on-chain `programData` (not docs)
- [ ] All discriminator bytes computed from IDL / SHA256 and hardcoded
- [ ] Discriminators verified against a real captured transaction (use `carbon-test-utils`)
- [ ] `ArrangeAccounts` verified against current IDL — account count and order match
- [ ] `remaining: Vec<AccountMeta>` present in every `XAccounts` struct
- [ ] Account `decode()` checks `account.owner == PROGRAM_ID` in `mod.rs`, not per-file
- [ ] `[lib] crate-type = ["rlib"]` in Cargo.toml
- [ ] `BorshDeserialize` derived on every struct that uses `CarbonDeserialize`
- [ ] Large structs are `Box`ed in account enum variants
- [ ] Using `instruction_decoder_collection_fast!` (not the deprecated non-fast version)
- [ ] Unit tests added using `carbon-test-utils` with real transaction fixtures

---

## Scope-Specific: Existing Carbon Integration

Scope uses Carbon for transaction parsing in the streaming pipeline. The relevant decoders and integration points are in:

- `apps/api/src/lib/helius-laserstream-client.ts` — raw transaction bytes flow into the carbon parse layer
- `apps/api/src/services/scope/stream-source-factory.ts` — routes to correct decoder based on program ID
- `apps/api/src/lib/triton-transaction-parser.ts` — legacy parser (Triton fallback path)
- `apps/api/src/lib/dex-swap-parser.ts` — DEX swap events from parsed Carbon output

When adding a new program decoder for Scope:
1. Build the crate under `decoders/<program>-decoder/`
2. Add it as a workspace member in root `Cargo.toml`
3. Wire it into `stream-source-factory.ts` via the `instruction_decoder_collection_fast!` macro
4. Add a contract doc to `docs/api-contracts/` before integrating external API calls
5. Capture a fixture transaction and add a unit test in `apps/api/__tests__/`
