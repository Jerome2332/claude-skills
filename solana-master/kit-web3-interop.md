# Kit ↔ web3.js Interop (Boundary Patterns)

## The Rule

- **New code**: Kit types and Kit-first APIs
- **Legacy dependencies**: Isolate web3.js-shaped types behind an adapter boundary

## When to Use @solana/web3-compat

Use `@solana/web3-compat` when:
- A dependency expects `PublicKey`, `Keypair`, `Transaction`, `VersionedTransaction`, `Connection`
- You are migrating an existing web3.js codebase incrementally
- You need to interface with Anchor's TypeScript client

### Why This Approach Works

- web3-compat re-exports web3.js-like types and delegates to Kit where possible
- It includes helper conversions to move between web3.js and Kit representations
- Prevents type pollution across your codebase

## Practical Boundary Layout

Keep these modules separate:

```
src/
├── solana/
│   ├── kit/                    # Kit-first code (preferred)
│   │   ├── addresses.ts
│   │   ├── instructions.ts
│   │   ├── transactions.ts
│   │   └── codecs.ts
│   └── web3/                   # Legacy adapters only
│       ├── anchor-client.ts    # Anchor TS client wrapper
│       ├── converters.ts       # Type conversions
│       └── legacy-sdk.ts       # Old SDK integrations
```

### Kit Directory (Preferred)

All Kit-first code belongs here:
- Addresses using `Address` type
- Instruction builders from `@solana-program/*`
- Transaction assembly using Kit's message APIs
- Typed codecs for program data
- Generated clients from Codama

### Web3 Directory (Adapters Only)

Legacy lib adapters go here:
- Anchor TS client wrappers
- Conversions between `PublicKey` and Kit `Address`
- Conversions between web3 `TransactionInstruction` and Kit instruction shapes
- **Only at edges, never in core domain**

## Conversion Helpers

Use web3-compat helpers:

```typescript
import {
  toAddress,
  toPublicKey,
  toWeb3Instruction,
  toKitSigner,
} from '@solana/web3-compat';

// Kit Address to web3.js PublicKey
const publicKey = toPublicKey(kitAddress);

// web3.js PublicKey to Kit Address
const address = toAddress(publicKey);

// Kit instruction to web3.js TransactionInstruction
const web3Instruction = toWeb3Instruction(kitInstruction);

// web3.js Keypair to Kit Signer
const signer = toKitSigner(keypair);
```

## Anchor Client Integration

When using Anchor's TypeScript client with a Kit-first app:

```typescript
// src/solana/web3/anchor-client.ts
import { Program, AnchorProvider } from '@coral-xyz/anchor';
import { toAddress, toPublicKey } from '@solana/web3-compat';
import type { MyProgram } from '../../../target/types/my_program';

export class AnchorClientAdapter {
  private program: Program<MyProgram>;

  constructor(provider: AnchorProvider) {
    this.program = new Program<MyProgram>(IDL, provider);
  }

  // Expose Kit-friendly interface
  async initialize(authority: Address): Promise<string> {
    const authorityPubkey = toPublicKey(authority);

    const signature = await this.program.methods
      .initialize()
      .accounts({ authority: authorityPubkey })
      .rpc();

    return signature;
  }
}
```

## When You Still Need @solana/web3.js Directly

Some methods outside web3-compat's compatibility surface may fall back to legacy web3.js.

If that happens:
1. Keep `@solana/web3.js` as an explicit dependency
2. Isolate fallback usage to adapter modules only
3. **Never let `PublicKey` bleed into your core domain types**

```typescript
// WRONG - PublicKey in domain types
interface User {
  id: string;
  wallet: PublicKey;  // web3.js type leaking!
}

// CORRECT - Kit types in domain
interface User {
  id: string;
  wallet: Address;  // Kit type
}

// Conversion happens at adapter boundary
function toWeb3User(user: User): Web3User {
  return {
    id: user.id,
    wallet: toPublicKey(user.wallet),
  };
}
```

## Common Mistakes to Prevent

### 1. Mixing `Address` and `PublicKey`

```typescript
// WRONG - Type drift and confusion
function transfer(from: Address, to: PublicKey, amount: bigint) { ... }

// CORRECT - Consistent types
function transfer(from: Address, to: Address, amount: bigint) { ... }
```

### 2. Building in One Stack, Signing in Another

```typescript
// WRONG - Building with Kit, signing with web3.js without conversion
const kitTransaction = buildTransaction(instructions);
const web3Signer = Keypair.generate();
await web3Signer.sign(kitTransaction);  // Type mismatch!

// CORRECT - Explicit conversion at boundary
const kitTransaction = buildTransaction(instructions);
const web3Transaction = toWeb3Transaction(kitTransaction);
await web3Signer.sign(web3Transaction);
```

### 3. Multiple Connection Sources

```typescript
// WRONG - Passing web3.js Connection into Kit code
async function fetchBalance(connection: Connection, address: Address) {
  // Mixing paradigms!
}

// CORRECT - Single source of truth
async function fetchBalance(client: SolanaClient, address: Address) {
  return client.getBalance(address);
}
```

## Decision Checklist

Before adding web3.js:

1. **Is there a Kit-native equivalent?** → Prefer Kit
2. **Is the only reason a dependency?** → Use web3-compat at the boundary
3. **Can you generate a Kit-native client (Codama)?** → Prefer codegen
4. **Is it a one-time migration?** → Create adapter, plan to remove

## Migration Strategy

If migrating an existing web3.js codebase:

### Phase 1: Add Boundaries
```typescript
// Create adapter layer
src/solana/web3/adapters.ts
```

### Phase 2: New Code Uses Kit
```typescript
// All new features use Kit types
src/features/new-feature/
├── hooks.ts      // Uses Kit
├── api.ts        // Uses Kit
└── types.ts      // Uses Address, not PublicKey
```

### Phase 3: Incremental Migration
```typescript
// Replace web3.js usage file by file
// Delete adapter functions as they become unused
```

### Phase 4: Remove web3.js
```bash
pnpm remove @solana/web3.js
# Keep only @solana/web3-compat if still needed
```

## Package.json Structure

```json
{
  "dependencies": {
    "@solana/kit": "^2.x",
    "@solana/client": "^1.x",
    "@solana/react-hooks": "^1.x",
    "@solana/web3-compat": "^1.x",
    "@solana-program/system": "^1.x"
  },
  "devDependencies": {
    "@solana/web3.js": "^1.x"
  }
}
```

Note: `@solana/web3.js` in devDependencies if only needed for tests or build-time operations.
