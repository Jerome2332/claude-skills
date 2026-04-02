---
name: discriminator-verify
description: Verify or compute Anchor instruction and account discriminators. Use when auditing transaction parsers, checking for silent parse failures, or adding support for a new instruction. Cross-checks discriminators in triton-transaction-parser.ts and dex-swap-parser.ts against computed values.
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/discriminator-verify/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/discriminator-verify/feedback.log
```

Only log general preferences, not task-specific details.

---
# Discriminator Verifier

You are verifying that Anchor instruction and account discriminators in the codebase match their cryptographically correct computed values. Discriminator mismatches cause silent parse failures — transactions are silently dropped rather than failing visibly.

## How Discriminators Are Computed

### Anchor Instruction Discriminator
```
sha256("global:<instruction_name>")[0..8]
```
- `instruction_name` is the snake_case instruction name as declared in the Anchor IDL
- Take the first 8 bytes of the SHA-256 digest
- Example: `sha256("global:buy")[0..8]`

### Anchor Account Discriminator
```
sha256("account:<AccountTypeName>")[0..8]
```
- `AccountTypeName` is the PascalCase account struct name
- Example: `sha256("account:BondingCurve")[0..8]`

### TypeScript Computation
```typescript
import { createHash } from 'crypto';

function instructionDiscriminator(name: string): Buffer {
  return Buffer.from(
    createHash('sha256').update(`global:${name}`).digest()
  ).slice(0, 8);
}

function accountDiscriminator(name: string): Buffer {
  return Buffer.from(
    createHash('sha256').update(`account:${name}`).digest()
  ).slice(0, 8);
}

// Example usage
const buyDisc = instructionDiscriminator('buy');
console.log(buyDisc.toString('hex')); // should match parser constant
```

### Closed Account Sentinel
- `[0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF]`
- Anchor sets this when an account is closed — parsers must check for this and skip such accounts

## Critical: Pump.fun Is NOT an Anchor Program

Pump.fun uses its own discriminators that do NOT follow the `global:` pattern. Its discriminators come directly from its IDL and were derived from the program's internal dispatch table. Do not attempt to compute Pump.fun discriminators using the `global:` formula — they will not match.

For Pump.fun discriminators, use the values from:
- The official Pump.fun IDL files at `https://github.com/pump-fun/pump-public-docs/tree/main/idl`
- The existing constants in `apps/api/src/lib/triton-transaction-parser.ts` (treat these as the source of truth unless the IDL contradicts them)

## Audit Procedure

### Step 1: Read the Parser Files

Read both files completely:
- `apps/api/src/lib/triton-transaction-parser.ts`
- `apps/api/src/lib/dex-swap-parser.ts`

Extract every discriminator constant you find. These are typically:
- `Buffer.from([...])` with 8 hex-like numbers
- Hex strings like `'66063d1201daebea'`
- Named constants like `BUY_DISCRIMINATOR`

### Step 2: Identify the Program for Each Discriminator

For each discriminator, determine:
1. Which program does it belong to? (Pump.fun, Raydium AMM v4, Raydium CLMM, Orca Whirlpool, Meteora DLMM, etc.)
2. Is that program Anchor-based or native?
3. What instruction or account type does it represent?

### Step 3: Compute Expected Values

For Anchor programs, compute the expected discriminator using the formula above.

For native/non-Anchor programs (including Pump.fun), cross-reference against the IDL. If an IDL file exists in the project under `docs/` or is referenced in the codebase, use it. If not, note that manual verification against the IDL is required.

### Step 4: Compare

For each discriminator:
- Write out the instruction/account name
- Write out the expected hex (computed or from IDL)
- Write out the actual hex in the parser
- Mark as MATCH or MISMATCH

### Step 5: Check for Missing Instructions

Review the IDL (if available) for instructions that exist on-chain but have no corresponding discriminator in the parser. These are silent coverage gaps — the parser silently ignores those transaction types.

## Output Format

Produce a verification table:

| Instruction/Account | Program | Anchor? | Expected (hex) | Actual in Parser (hex) | Status |
|---------------------|---------|---------|----------------|------------------------|--------|
| `buy` | Pump.fun | No | `66063d1201daebea` | `66063d1201daebea` | MATCH |
| `sell` | Pump.fun | No | `33e685a4017f83ad` | `33e685a4017f83ad` | MATCH |
| `swap` | Raydium AMM v4 | Yes | `(computed)` | `(actual)` | ? |

After the table, add:

**Coverage Gaps** — instructions in the IDL with no discriminator in the parser:
- List each gap with instruction name and expected discriminator

**Mismatches** — discriminators in the parser that do not match computed/IDL values:
- These are `critical` findings — the parser will silently drop matching transactions

**Notes** — any discriminators that could not be verified (IDL not available, non-Anchor program with no reference):
- These are `warning` findings — manual verification required

## Known Reference Values

These are the known-correct discriminators for the DEX programs this project parses. Use these to cross-check `dex-swap-parser.ts`:

### Raydium AMM v4 (non-Anchor)
- The swap instruction discriminator comes from the program's own encoding, not Anchor's formula
- Check `dex-swap-parser.ts` for the current value and flag if it cannot be verified against a known source

### Orca Whirlpool (Anchor)
- `swap`: `sha256("global:swap")[0..8]`
- `swap_v2`: `sha256("global:swap_v2")[0..8]`
- `two_hop_swap`: `sha256("global:two_hop_swap")[0..8]`

### Raydium CLMM (Anchor)
- `swap`: `sha256("global:swap")[0..8]`
- `swap_v2`: `sha256("global:swap_v2")[0..8]`

### Meteora DLMM (Anchor)
- `swap`: `sha256("global:swap")[0..8]`

**Note:** When multiple Anchor programs share the same instruction name (`swap`), their discriminators are IDENTICAL because the formula only depends on the name, not the program. The program ID is what differentiates them at the routing layer — the parser must check the program ID first, then the discriminator.
