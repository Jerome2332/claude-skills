# IDLs + Client Generation (Codama / Shank / Kinobi)

## Goal

Never hand-maintain multiple program clients by manually re-implementing serializers.
Prefer an IDL-driven, code-generated workflow.

## The Golden Rule

**Do not write IDLs by hand** unless you have no alternative.
**Do not hand-write Borsh layouts** for programs you own; use the IDL/codegen pipeline.

---

## Codama (Preferred)

Use Codama as the "single program description format" to generate:
- TypeScript clients (including Kit-friendly output)
- Rust clients (when available/needed)
- Documentation artifacts

### Why Codama?

- Universal format that works across frameworks
- Generates Kit-native TypeScript clients
- Supports both Anchor and native Rust programs
- Active development by Solana ecosystem

---

## Anchor → Codama Pipeline

If the program is Anchor:

### 1. Produce Anchor IDL from the build

```bash
anchor build
# IDL generated at: target/idl/my_program.json
```

### 2. Convert Anchor IDL to Codama nodes

```bash
npx @codama/nodes-from-anchor target/idl/my_program.json -o codama/my_program.json
```

### 3. Render a Kit-native TypeScript client

```bash
npx @codama/renderers-js codama/my_program.json -o clients/ts/my_program
```

### Complete Script

```bash
#!/bin/bash
set -e

PROGRAM_NAME="my_program"

# Build program and generate Anchor IDL
anchor build

# Convert to Codama format
npx @codama/nodes-from-anchor \
  "target/idl/${PROGRAM_NAME}.json" \
  -o "codama/${PROGRAM_NAME}.json"

# Generate TypeScript client
npx @codama/renderers-js \
  "codama/${PROGRAM_NAME}.json" \
  -o "clients/ts/${PROGRAM_NAME}"

echo "Client generated at clients/ts/${PROGRAM_NAME}"
```

---

## Native Rust → Shank → Codama Pipeline

If the program is native Rust (no Anchor):

### 1. Annotate Rust code with Shank macros

```rust
use shank::ShankAccount;
use shank::ShankInstruction;

#[derive(ShankAccount)]
pub struct MyAccount {
    pub authority: Pubkey,
    pub data: u64,
}

#[derive(ShankInstruction)]
pub enum MyInstruction {
    #[account(0, writable, signer, name = "authority", desc = "The account authority")]
    #[account(1, writable, name = "account", desc = "The account to initialize")]
    Initialize { data: u64 },
}
```

### 2. Generate Shank IDL

```bash
cargo install shank-cli
shank idl -o idl/my_program.json programs/my_program/src/lib.rs
```

### 3. Convert Shank IDL to Codama

```bash
npx @codama/nodes-from-shank idl/my_program.json -o codama/my_program.json
```

### 4. Generate clients via Codama renderers

```bash
npx @codama/renderers-js codama/my_program.json -o clients/ts/my_program
```

---

## Repository Structure Recommendation

```
my-project/
├── programs/
│   └── my_program/
│       ├── src/
│       │   └── lib.rs
│       └── Cargo.toml
├── idl/
│   └── my_program.json         # Anchor or Shank IDL
├── codama/
│   └── my_program.json         # Codama IDL
├── clients/
│   ├── ts/
│   │   └── my_program/         # Generated TypeScript client
│   │       ├── index.ts
│   │       ├── instructions/
│   │       ├── accounts/
│   │       └── types/
│   └── rust/
│       └── my_program/         # Generated Rust client (if needed)
├── tests/
└── Anchor.toml
```

---

## Generation Guardrails

### When to Check Generated Code into Git

Check codegen outputs into git if:
- You need deterministic builds
- You want users to consume the client without running codegen
- CI/CD needs the client without build tools

### When to Keep Codegen in CI Only

Keep codegen out of git if:
- You want to ensure clients always match latest program
- You publish clients to npm/crates.io separately
- You have complex multi-program dependencies

### Recommended: Hybrid Approach

```yaml
# .github/workflows/codegen.yml
name: Validate Codegen

on: [push, pull_request]

jobs:
  codegen:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build programs
        run: anchor build

      - name: Generate clients
        run: ./scripts/generate-clients.sh

      - name: Check for uncommitted changes
        run: |
          if [[ -n $(git status --porcelain clients/) ]]; then
            echo "Generated clients are out of date!"
            git diff clients/
            exit 1
          fi
```

---

## Using Generated Clients

### TypeScript (Kit-native)

```typescript
import { createMyProgramClient, getInitializeInstruction } from './clients/ts/my_program';
import { createSolanaRpc } from '@solana/kit';

const rpc = createSolanaRpc('https://api.devnet.solana.com');
const client = createMyProgramClient(rpc);

// Type-safe instruction building
const instruction = getInitializeInstruction({
  authority: authorityAddress,
  account: accountAddress,
  data: 42n,
});

// Type-safe account fetching
const accountData = await client.account.myAccount.fetch(accountAddress);
console.log(accountData.data); // Typed as bigint
```

### With Anchor TS Client (Legacy Boundary)

If you must use Anchor's TS client in a Kit-first app:

```typescript
// src/solana/web3/anchor-adapter.ts
import { Program, AnchorProvider } from '@coral-xyz/anchor';
import { toAddress, toPublicKey } from '@solana/web3-compat';
import type { MyProgram } from '../../target/types/my_program';
import IDL from '../../target/idl/my_program.json';

export function createAnchorAdapter(provider: AnchorProvider) {
  const program = new Program<MyProgram>(IDL as any, provider);

  return {
    // Expose Kit-friendly interface
    async initialize(authority: Address, data: bigint): Promise<string> {
      return program.methods
        .initialize(data)
        .accounts({ authority: toPublicKey(authority) })
        .rpc();
    },
  };
}
```

---

## Quick Client Generation

For projects already using Anchor, the fastest path:

```bash
# Install Codama tools
pnpm add -D @codama/nodes-from-anchor @codama/renderers-js

# One-liner to generate client
npx create-codama-clients
```

This interactive CLI will:
1. Detect your Anchor IDL
2. Generate Codama nodes
3. Render TypeScript client
4. Set up the output directory

---

## Troubleshooting

### IDL Not Generated

```bash
# Ensure idl-build feature is enabled in Cargo.toml
[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
```

### Type Mismatches in Generated Client

- Regenerate after any program changes
- Clear cached IDL: `rm -rf target/idl/ codama/`
- Ensure Anchor version matches IDL format

### Client Doesn't Match Program

- Always regenerate clients after program changes
- Add CI check to verify generated code is committed
- Use version tags to track program/client compatibility

---

## Best Practices

1. **Automate generation**: Add to build scripts and CI
2. **Version together**: Keep program and client versions in sync
3. **Test generated code**: Include generated client in test suite
4. **Document breaking changes**: Note when IDL changes break clients
5. **Prefer Kit output**: Use Codama's Kit-native renderers for new projects
