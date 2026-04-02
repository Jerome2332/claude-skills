# Testing Strategy (LiteSVM / Mollusk / Surfpool)

## Testing Pyramid

1. **Unit tests (fast)**: LiteSVM or Mollusk
2. **Integration tests (realistic state)**: Surfpool
3. **Cluster smoke tests**: devnet/testnet/mainnet as needed

## Required Testing Framework

**TypeScript tests MUST use `node:test`** with `tsx` runner. Do not use Jest, Mocha, or Vitest.

```typescript
// CORRECT
import { before, describe, test } from 'node:test';
import assert from 'node:assert';

// WRONG - Do not use
import { describe, it, expect } from 'vitest';
import { describe, it } from 'mocha';
```

---

## LiteSVM

A lightweight Solana Virtual Machine that runs directly in your test process. Created by Aursen from Exotic Markets.

### When to Use LiteSVM

- Fast execution without validator overhead
- Direct account state manipulation
- Built-in performance profiling
- Multi-language support (Rust, TypeScript, Python)

### Rust Setup

```bash
cargo add --dev litesvm
```

```rust
use litesvm::LiteSVM;
use solana_sdk::{pubkey::Pubkey, signature::Keypair, transaction::Transaction};

#[test]
fn test_deposit() {
    let mut svm = LiteSVM::new();

    // Load your program
    let program_id = pubkey!("YourProgramId11111111111111111111111111111");
    svm.add_program_from_file(program_id, "target/deploy/program.so");

    // Create accounts
    let payer = Keypair::new();
    svm.airdrop(&payer.pubkey(), 1_000_000_000).unwrap();

    // Build and send transaction
    let transaction = Transaction::new_signed_with_payer(
        &[/* instructions */],
        Some(&payer.pubkey()),
        &[&payer],
        svm.latest_blockhash(),
    );

    let result = svm.send_transaction(transaction);
    assert!(result.is_ok());
}
```

### TypeScript Setup

```bash
pnpm add --save-dev litesvm
```

```typescript
import { describe, test, before } from 'node:test';
import assert from 'node:assert';
import { LiteSVM } from 'litesvm';
import { PublicKey, Transaction, Keypair } from '@solana/web3.js';

describe('MyProgram', () => {
  let svm: LiteSVM;
  let payer: Keypair;

  before(() => {
    const programId = new PublicKey('YourProgramId11111111111111111111111111111');
    svm = new LiteSVM();
    svm.addProgramFromFile(programId, 'target/deploy/program.so');

    payer = Keypair.generate();
    svm.airdrop(payer.publicKey, 1_000_000_000n);
  });

  test('should execute deposit instruction', () => {
    const transaction = new Transaction();
    transaction.recentBlockhash = svm.latestBlockhash();
    transaction.add(/* instructions */);
    transaction.sign(payer);

    // Simulate first (optional)
    const simulation = svm.simulateTransaction(transaction);

    // Execute
    const result = svm.sendTransaction(transaction);
    assert.ok(result);
  });
});
```

### Account Types in LiteSVM

**System Accounts:**
- Payer accounts (contain lamports)
- Uninitialized accounts (empty, awaiting setup)

**Program Accounts:**
- Serialize with `borsh`, `bincode`, or `solana_program_pack`
- Calculate rent-exempt minimum balance

**Token Accounts:**
- Use `spl_token::state::Mint` and `spl_token::state::Account`
- Serialize with Pack trait

### Advanced LiteSVM Features

```rust
// Modify clock sysvar
svm.set_sysvar(&Clock { slot: 1000, .. });

// Warp to slot
svm.warp_to_slot(5000);

// Configure compute budget
svm.set_compute_budget(ComputeBudget { max_units: 400_000, .. });

// Toggle signature verification (useful for testing)
svm.with_sigverify(false);

// Check compute units used
let result = svm.send_transaction(transaction)?;
println!("CUs used: {}", result.compute_units_consumed);
```

---

## Mollusk

A lightweight test harness providing direct interface to program execution without full validator runtime. Best for Rust-only testing with fine-grained control.

### When to Use Mollusk

- Fast execution for rapid development cycles
- Precise account state manipulation for edge cases
- Detailed performance metrics and CU benchmarking
- Custom syscall testing

### Setup

```bash
cargo add --dev mollusk-svm
cargo add --dev mollusk-svm-programs-token  # For SPL token helpers
cargo add --dev solana-sdk solana-program
```

### Basic Usage

```rust
use mollusk_svm::Mollusk;
use mollusk_svm::result::Check;
use solana_sdk::{account::Account, pubkey::Pubkey, instruction::Instruction};

#[test]
fn test_instruction() {
    let program_id = Pubkey::new_unique();
    let mollusk = Mollusk::new(&program_id, "target/deploy/program");

    // Create accounts
    let payer = (
        Pubkey::new_unique(),
        Account {
            lamports: 1_000_000_000,
            data: vec![],
            owner: solana_sdk::system_program::ID,
            executable: false,
            rent_epoch: 0,
        },
    );

    // Build instruction
    let instruction = Instruction {
        program_id,
        accounts: vec![/* account metas */],
        data: vec![/* instruction data */],
    };

    // Execute with validation
    mollusk.process_and_validate_instruction(
        &instruction,
        &[payer],
        &[
            Check::success(),
            Check::compute_units(50_000),
        ],
    );
}
```

### Token Program Helpers

```rust
use mollusk_svm_programs_token::token;

// Add token program to test environment
token::add_program(&mut mollusk);

// Create pre-configured token accounts
let mint_account = token::mint_account(decimals, supply, mint_authority);
let token_account = token::token_account(mint, owner, amount);
```

### CU Benchmarking

```rust
use mollusk_svm::MolluskComputeUnitBencher;

let bencher = MolluskComputeUnitBencher::new(mollusk)
    .must_pass(true)
    .out_dir("../target/benches");

bencher.bench(
    "deposit_instruction",
    &instruction,
    &accounts,
);
// Generates markdown report with CU usage and deltas
```

### Advanced Configuration

```rust
// Set compute budget
mollusk.set_compute_budget(200_000);

// Enable all feature flags
mollusk.set_feature_set(FeatureSet::all_enabled());

// Customize sysvars
mollusk.sysvars.clock = Clock {
    slot: 1000,
    epoch: 5,
    unix_timestamp: 1700000000,
    ..Default::default()
};
```

---

## Surfpool

SDK and tooling suite for integration testing with realistic cluster state. Surfnet is the local network component (drop-in replacement for solana-test-validator).

### When to Use Surfpool

- Complex CPIs requiring mainnet programs (e.g., Jupiter with 40+ accounts)
- Testing against realistic account state
- Time travel and block manipulation
- Account/program cloning between environments

### Setup

```bash
# Install Surfpool CLI
cargo install surfpool

# Start local Surfnet
surfpool start
```

### Connection Setup

```typescript
import { Connection } from '@solana/web3.js';

const connection = new Connection('http://localhost:8899', 'confirmed');
```

### System Variable Control

```typescript
// Time travel to specific slot
await connection._rpcRequest('surfnet_timeTravel', [{
    absoluteSlot: 250000000
}]);

// Pause/resume block production
await connection._rpcRequest('surfnet_pauseClock', []);
await connection._rpcRequest('surfnet_resumeClock', []);
```

### Account Manipulation

```typescript
// Set account state
await connection._rpcRequest('surfnet_setAccount', [{
    pubkey: accountPubkey.toString(),
    lamports: 1000000000,
    data: Buffer.from(accountData).toString('base64'),
    owner: programId.toString(),
}]);

// Set token account
await connection._rpcRequest('surfnet_setTokenAccount', [{
    pubkey: ownerPubkey.toString(),
    mint: mintPubkey.toString(),
    owner: ownerPubkey.toString(),
    amount: '1000000',
}]);

// Clone account from another program
await connection._rpcRequest('surfnet_cloneProgramAccount', [{
    source: sourceProgramId.toString(),
    destination: destProgramId.toString(),
    account: accountPubkey.toString(),
}]);
```

### SOL Supply Configuration

```typescript
// Configure supply for economic edge case testing
await connection._rpcRequest('surfnet_setSupply', [{
    circulating: '500000000000000000',
    nonCirculating: '100000000000000000',
    total: '600000000000000000',
}]);
```

---

## Test Layout Recommendation

```
tests/
├── unit/
│   ├── deposit.rs          # LiteSVM or Mollusk
│   ├── withdraw.rs
│   └── mod.rs
├── integration/
│   ├── full_flow.rs        # Surfpool
│   └── mod.rs
├── typescript/
│   ├── deposit.test.ts     # node:test + LiteSVM
│   └── integration.test.ts
└── fixtures/
    └── accounts.rs         # Shared test account setup
```

---

## TypeScript Test Structure

```typescript
// tests/typescript/deposit.test.ts
import { describe, test, before, after } from 'node:test';
import assert from 'node:assert';

describe('Deposit', () => {
  let testContext: TestContext;

  before(async () => {
    testContext = await setupTestEnvironment();
  });

  after(async () => {
    await testContext.cleanup();
  });

  test('should deposit SOL successfully', async () => {
    const { svm, payer, vault } = testContext;

    const balanceBefore = svm.getBalance(vault);
    await executeDeposit(svm, payer, vault, 1_000_000_000n);
    const balanceAfter = svm.getBalance(vault);

    assert.strictEqual(
      balanceAfter - balanceBefore,
      1_000_000_000n,
      'Vault balance should increase by deposit amount'
    );
  });

  test('should fail with insufficient funds', async () => {
    const { svm, poorPayer, vault } = testContext;

    await assert.rejects(
      async () => executeDeposit(svm, poorPayer, vault, 999_000_000_000n),
      /insufficient funds/i,
      'Should reject with insufficient funds error'
    );
  });
});
```

### Running TypeScript Tests

```bash
# Run all tests
npx tsx --test tests/**/*.test.ts

# Run specific test file
npx tsx --test tests/typescript/deposit.test.ts

# With coverage (if using c8)
npx c8 tsx --test tests/**/*.test.ts
```

### package.json Scripts

```json
{
  "scripts": {
    "test": "tsx --test tests/**/*.test.ts",
    "test:unit": "tsx --test tests/unit/**/*.test.ts",
    "test:integration": "tsx --test tests/integration/**/*.test.ts",
    "test:coverage": "c8 tsx --test tests/**/*.test.ts"
  }
}
```

---

## CI Guidance

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-action@stable

      - name: Build program
        run: cargo build-sbf

      - name: Run Rust unit tests
        run: cargo test-sbf

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: pnpm install

      - name: Run TypeScript unit tests
        run: pnpm test:unit

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4

      - name: Install Surfpool
        run: cargo install surfpool

      - name: Start Surfpool
        run: surfpool start --background

      - name: Run integration tests
        run: pnpm test:integration
```

---

## Best Practices

- Keep unit tests as the default CI gate (fast feedback)
- Use deterministic PDAs and seeded keypairs for reproducibility
- Minimize fixtures; prefer programmatic account creation
- Profile CU usage during development to catch regressions
- Run integration tests in separate CI stage to control runtime
- Use `test()` not `it()` in TypeScript tests
- Always clean up test resources in `after()` hooks
- Test error paths, not just happy paths
- Use descriptive assertion messages

---

## Common Testing Patterns

### Testing PDAs

```typescript
test('should derive correct PDA', () => {
  const [derivedPda, bump] = PublicKey.findProgramAddressSync(
    [Buffer.from('vault'), owner.publicKey.toBuffer()],
    programId
  );

  assert.strictEqual(
    derivedPda.toBase58(),
    expectedPda.toBase58(),
    'PDA should match expected address'
  );
});
```

### Testing Token Transfers

```typescript
test('should transfer tokens correctly', async () => {
  const sourceBefore = await getTokenBalance(sourceAccount);
  const destinationBefore = await getTokenBalance(destinationAccount);

  await executeTransfer(amount);

  const sourceAfter = await getTokenBalance(sourceAccount);
  const destinationAfter = await getTokenBalance(destinationAccount);

  assert.strictEqual(sourceBefore - sourceAfter, amount);
  assert.strictEqual(destinationAfter - destinationBefore, amount);
});
```

### Testing Error Conditions

```typescript
test('should reject unauthorized access', async () => {
  const unauthorizedSigner = Keypair.generate();

  await assert.rejects(
    async () => executeWithSigner(unauthorizedSigner),
    (error: Error) => {
      assert.match(error.message, /unauthorized|permission|signer/i);
      return true;
    }
  );
});
```
