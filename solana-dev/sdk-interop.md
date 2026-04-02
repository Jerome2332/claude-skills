# SDK Interoperability Patterns

## Core Principle

> New code: Kit types and Kit-first APIs.
> Legacy dependencies: isolate web3.js-shaped types behind an adapter boundary.

## Architectural Separation

### Kit-Native Zone (`src/solana/kit/`)
- Address types (`Address`)
- Instruction builders
- Transaction assembly
- Typed codecs
- Generated clients (Codama)

### Adapter Zone (`src/solana/web3/`)
- Legacy library integrations (Anchor TS client, etc.)
- Explicit conversions at module boundaries only
- Compatibility shims

## Directory Structure
```
src/
├── solana/
│   ├── kit/              # Kit-native code
│   │   ├── client.ts     # Codama-generated
│   │   ├── instructions/ # Kit instruction builders
│   │   └── types.ts      # Kit types
│   └── web3/             # Legacy adapter zone
│       ├── anchor.ts     # Anchor client wrapper
│       └── compat.ts     # Conversion utilities
└── components/           # UI (uses kit/ only)
```

## @solana/web3-compat

The bridge package for legacy library integration.

### Installation
```bash
npm install @solana/web3-compat
```

### Key Conversion Utilities

```typescript
import {
  toAddress,       // PublicKey → Address
  toPublicKey,     // Address → PublicKey
  toWeb3Instruction,
  toKitInstruction,
  toKitSigner,
  toWeb3Keypair,
} from '@solana/web3-compat';
```

### Examples

#### Address Conversion
```typescript
import { PublicKey } from '@solana/web3.js';
import { toAddress, toPublicKey } from '@solana/web3-compat';
import type { Address } from '@solana/kit';

// web3.js → Kit
const publicKey = new PublicKey('...');
const address: Address = toAddress(publicKey);

// Kit → web3.js
const address: Address = '...';
const publicKey: PublicKey = toPublicKey(address);
```

#### Instruction Conversion
```typescript
import { toWeb3Instruction, toKitInstruction } from '@solana/web3-compat';

// Kit instruction → web3.js (for legacy libraries)
const kitIx = createTransferInstruction(...);
const web3Ix = toWeb3Instruction(kitIx);

// web3.js instruction → Kit (from legacy libraries)
const anchorIx = anchorProgram.methods.doSomething().instruction();
const kitIx = toKitInstruction(await anchorIx);
```

#### Signer Conversion
```typescript
import { toKitSigner, toWeb3Keypair } from '@solana/web3-compat';
import { Keypair } from '@solana/web3.js';

// web3.js Keypair → Kit Signer
const keypair = Keypair.generate();
const kitSigner = toKitSigner(keypair);

// Kit Signer → web3.js Keypair (when needed)
const web3Keypair = toWeb3Keypair(kitSigner);
```

## Anti-Patterns to Avoid

### ❌ Mixing Types Across Layers
```typescript
// BAD: Address and PublicKey mixed
function transfer(from: Address, to: PublicKey) {
  // Confusing, error-prone
}

// GOOD: Consistent types
function transfer(from: Address, to: Address) {
  // Clear, Kit-native
}
```

### ❌ Cross-Framework Transaction Building
```typescript
// BAD: Building with one framework, signing with another
const tx = new Transaction();  // web3.js
tx.add(kitInstruction);        // Kit instruction
await wallet.signTransaction(tx);  // ??

// GOOD: Convert at boundaries
const web3Ix = toWeb3Instruction(kitInstruction);
const tx = new Transaction().add(web3Ix);
```

### ❌ Passing Connection to Kit Code
```typescript
// BAD: web3.js Connection in Kit-native code
import { Connection } from '@solana/web3.js';
function fetchData(connection: Connection) { ... }

// GOOD: Use Kit client
import { createClient } from '@solana/kit';
const client = createClient({ endpoint: rpcUrl });
```

## Migration Decision Flow

Before adding web3.js dependencies, ask:

```
Is a Kit-native alternative available?
    ├── Yes → Use Kit
    └── No → Is web3-compat sufficient?
              ├── Yes → Use adapter pattern
              └── No → Can codegen replace it?
                        ├── Yes → Generate client
                        └── No → Add to adapter zone
```

## Common Integration Patterns

### Anchor Client Integration
```typescript
// src/solana/web3/anchor.ts
import { Program, AnchorProvider } from '@coral-xyz/anchor';
import { toKitInstruction, toAddress } from '@solana/web3-compat';
import { IDL, MyProgram } from '../idl';

export class AnchorAdapter {
  private program: Program<MyProgram>;

  constructor(provider: AnchorProvider) {
    this.program = new Program(IDL, provider);
  }

  // Expose Kit-native interface
  async createInitializeInstruction(params: {
    user: Address;
    data: Address;
  }) {
    const ix = await this.program.methods
      .initialize()
      .accounts({
        user: toPublicKey(params.user),
        data: toPublicKey(params.data),
      })
      .instruction();

    return toKitInstruction(ix);
  }
}

// Usage in Kit-native code
const adapter = new AnchorAdapter(provider);
const ix = await adapter.createInitializeInstruction({
  user: userAddress,  // Kit Address
  data: dataAddress,  // Kit Address
});
```

### Wallet Adapter Bridge
```typescript
// Bridge wallet-adapter to Kit signer
import { WalletContextState } from '@solana/wallet-adapter-react';
import { KeyPairSigner } from '@solana/kit';

function walletToKitSigner(wallet: WalletContextState): KeyPairSigner | null {
  if (!wallet.publicKey || !wallet.signTransaction) {
    return null;
  }

  return {
    address: toAddress(wallet.publicKey),
    signTransactions: async (txs) => {
      // Convert and sign
      const signed = await wallet.signAllTransactions!(
        txs.map(toWeb3Transaction)
      );
      return signed.map(toKitTransaction);
    },
  };
}
```

## Best Practices Summary

1. **Keep Kit as the default** - Only use web3.js when required by dependencies
2. **Isolate adapters** - All conversions in dedicated adapter modules
3. **Convert at boundaries** - Never mix types in business logic
4. **Document integrations** - Note which libraries require web3.js
5. **Prefer codegen** - Generate Kit clients instead of using web3.js
