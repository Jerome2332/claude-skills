---
name: pda-audit
description: Audit PDA derivations in the codebase for correctness. Use when reviewing swap execution, token account creation, metadata fetching, or any code that derives program-derived addresses from known seeds.
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/pda-audit/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/pda-audit/feedback.log
```

Only log general preferences, not task-specific details.

---
# PDA Audit

You are auditing every PDA derivation in the codebase to verify that the seeds and program IDs are correct. Incorrect PDA derivations produce wrong addresses silently — transactions will fail at runtime or, worse, resolve to an account owned by a different program.

## Reference: Correct PDA Derivations

These are the canonical seed patterns. Any deviation is a finding.

### Associated Token Account (ATA)
```
seeds: [owner_pubkey, token_program_id, mint_pubkey]
program: ASSOCIATED_TOKEN_PROGRAM_ID (ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJe8bY)
```
**Critical detail**: The second seed is `token_program_id`, NOT the mint. And `token_program_id` differs:
- Original SPL Token: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- Token 2022: `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`

An ATA for a Token 2022 mint is derived with `TOKEN_2022_PROGRAM_ID` in the seeds. Using `TOKEN_PROGRAM_ID` for a Token 2022 mint produces the wrong address.

### Metaplex Token Metadata PDA
```
seeds: [b"metadata", METAPLEX_PROGRAM_ID, mint_pubkey]
program: METAPLEX_PROGRAM_ID (metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s)
```
**Critical detail**: `METAPLEX_PROGRAM_ID` appears in BOTH the seeds AND as the program used for derivation. This is intentional and correct. Many implementations incorrectly omit it from the seeds or use a different program.

### Metaplex Master Edition PDA
```
seeds: [b"metadata", METAPLEX_PROGRAM_ID, mint_pubkey, b"edition"]
program: METAPLEX_PROGRAM_ID (metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s)
```

### Pump.fun Bonding Curve
```
seeds: [b"bonding-curve", mint_pubkey]
program: Pump.fun program (6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P)
```

### Pump.fun Associated Bonding Curve (vault token account)
```
seeds: [b"associated-bonding-curve", mint_pubkey]
program: Pump.fun program (6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P)
```

### Pump.fun Creator Vault
```
seeds: [b"creator-vault", creator_pubkey]
program: Pump.fun program (6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P)
```

### Pump.fun Global Config
```
seeds: [b"global"]
program: Pump.fun program (6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P)
```

### PumpSwap Pool
```
seeds: [b"pool", index_as_u16_le, creator_pubkey, base_mint_pubkey, quote_mint_pubkey]
program: PumpSwap AMM (pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA)
```

### PumpSwap LP Mint
```
seeds: [b"pool_lp_mint", pool_pubkey]
program: PumpSwap AMM (pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA)
```

### PumpSwap Pool Token Account
```
seeds: [b"pool_token_account", pool_pubkey, mint_pubkey]
program: PumpSwap AMM (pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA)
```

### PumpSwap Creator Vault Authority
```
seeds: [b"creator_vault", coin_creator_pubkey]
program: PumpSwap AMM (pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA)
```

## Common Bug Patterns to Look For

1. **Missing program ID in seeds** — e.g., Metaplex PDA derived without `METAPLEX_PROGRAM_ID` in seeds
2. **Wrong token program for ATA** — using `TOKEN_PROGRAM_ID` for a Token 2022 mint's ATA derivation
3. **Wrong program for derivation** — e.g., calling `findProgramAddressSync` with `SystemProgram.programId` instead of the owning program
4. **Pubkey encoded as base58 string bytes** — seeds should be `.toBuffer()` (32 raw bytes), not the base58 string encoded as bytes
5. **String seeds missing the `b""` equivalent** — `Buffer.from("bonding-curve")` is correct; `new PublicKey("bonding-curve").toBuffer()` is wrong
6. **Wrong seed order** — order matters; `[mint, owner]` is wrong for ATA (should be `[owner, token_program_id, mint]`)
7. **Using `findProgramAddress` (async) when `findProgramAddressSync` is available** — not a bug, but unnecessary async in hot paths

## Audit Procedure

### Step 1: Find All PDA Derivations

Search for:
- `PublicKey.findProgramAddressSync(`
- `findProgramAddress(`
- `PublicKey.createProgramAddressSync(`
- `createProgramAddress(`
- Any custom PDA utility functions (search for `pda`, `findPda`, `getPda`, `derivePda` in the codebase)

Also search for hardcoded PDA results (strings that are expected PDA addresses used as constants) — these bypass derivation entirely and should be flagged for verification.

### Step 2: Verify Each Derivation

For each derivation call found:
1. What PDA type is being derived? (ATA, metadata, bonding curve, etc.)
2. What seeds are used?
3. What program ID is used?
4. Does it match the canonical pattern above?
5. If it's an ATA derivation: does it account for Token 2022 mints?

## Output Format

Produce a table:

| File:Line | PDA Type | Seeds Used | Program Used | Correct? | Issue / Fix |
|-----------|----------|------------|--------------|----------|-------------|
| `apps/api/src/services/data/token-metadata.service.ts:88` | Metaplex metadata | `["metadata", METAPLEX_PROGRAM_ID, mint]` | `METAPLEX_PROGRAM_ID` | Yes | — |
| `apps/web/src/hooks/use-bonding-curve-cache.ts:42` | Pump.fun bonding curve | `["bonding-curve", mint]` | `PUMP_PROGRAM_ID` | Yes | — |
| `apps/api/src/services/trading/swap.service.ts:130` | ATA | `[owner, TOKEN_PROGRAM_ID, mint]` | `ASSOCIATED_TOKEN_PROGRAM_ID` | Partial | Must use TOKEN_2022_PROGRAM_ID for T22 mints |

After the table:

**Critical Findings** — wrong PDAs that will produce wrong addresses at runtime
**Warning Findings** — PDAs that work for legacy tokens but break for Token 2022
**Notes** — defensive improvements (using async when sync is available, etc.)
