---
name: token2022-audit
description: Audit the codebase for Token 2022 vs original SPL Token handling gaps. Use when reviewing code that reads mint accounts, token accounts, or token balances to ensure both TOKEN_PROGRAM_ID and TOKEN_2022_PROGRAM_ID are handled correctly.
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/token2022-audit/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/token2022-audit/feedback.log
```

Only log general preferences, not task-specific details.

---
# Token 2022 Audit

You are performing a targeted audit of the codebase to find every place that reads or processes Solana token data and checking whether it correctly handles both the original SPL Token program and Token 2022.

## Critical Facts — Memorize Before Searching

### Program IDs
- Original SPL Token: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- Token 2022: `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`

### Account Size Differences
- Original SPL Token account: **165 bytes fixed**
- Token 2022 token account: **165 bytes base + variable extension data** — never assume 165 bytes is the full account
- Token 2022 mint account: base mint data + variable extension data — never assume a fixed mint size

### The Metadata Pointer Extension
- Token 2022 mints can carry their own metadata via the `MetadataPointer` extension, meaning metadata lives on the mint account itself rather than at a separate Metaplex PDA
- Before falling back to Metaplex `getMetadata(mint)`, check whether the mint has a `MetadataPointer` extension pointing to itself
- If it does, decode the `TokenMetadata` extension directly from the mint account data instead of fetching a separate Metaplex account

### Correct API Usage
- `fetchMint(rpc, mintAddress)` — WRONG if the token might be Token 2022; you must pass the program ID: `fetchMint(rpc, mintAddress, { programAddress: TOKEN_2022_PROGRAM_ID })`
- To handle both: fetch the account first, check `account.owner`, then call `fetchMint` / `fetchTokenAccount` with the correct program address
- `getAssociatedTokenAddressSync(mint, owner)` — defaults to original SPL Token; for Token 2022, pass `TOKEN_2022_PROGRAM_ID` as the fourth argument: `getAssociatedTokenAddressSync(mint, owner, false, TOKEN_2022_PROGRAM_ID)`
- `createAssociatedTokenAccountInstruction` — same issue, must pass the correct token program ID

### Extension Helper Functions (from `@solana/spl-token`)
- `getExtensionTypes(mint.extensions)` — returns the list of extension type tags on a mint
- `getExtensionData(ExtensionType.MetadataPointer, mintData)` — extracts raw extension bytes
- `isSome(option)` / `isNone(option)` — for Rust-style `Option<T>` fields in extension data
- `isExtension(extensionType, extension)` — type guard for extension structs

## Audit Procedure

### Step 1: Find All Token-Reading Code

Search the codebase for:
- `fetchMint`
- `getMint`
- `fetchTokenAccount`
- `getTokenAccountBalance`
- `getTokenAccountsByOwner`
- `getMultipleAccountsInfo` when used on mint or token accounts
- `TOKEN_PROGRAM_ID` — find every usage
- `TOKEN_2022_PROGRAM_ID` — find every usage (to understand what IS handled)
- `165` as a numeric literal near token account code (hardcoded size assumption)
- `getAssociatedTokenAddress` / `getAssociatedTokenAddressSync`
- `createAssociatedTokenAccountInstruction`
- `getAccount` from `@solana/spl-token`
- Any Borsh deserialization of mint or token account data with a fixed-size schema

### Step 2: Check Each Site

For each location found, answer:

1. **Does it pass the program ID?** — If calling `fetchMint` or similar without specifying the program, it will fail for Token 2022 mints.
2. **Does it check `account.owner` first?** — The correct pattern is to read the account, inspect `.owner`, and dispatch to the right program.
3. **Does it hardcode 165?** — Any `data.length === 165`, `data.slice(0, 165)`, or similar is broken for Token 2022 accounts.
4. **Does it derive ATAs correctly?** — Token 2022 ATAs are derived with `TOKEN_2022_PROGRAM_ID`, not the default.
5. **Does it handle the metadata pointer extension?** — If fetching token metadata, does it check for the `MetadataPointer` extension before going to Metaplex?

### Step 3: Check the Scope Project Specifically

In this project, pay special attention to:
- `apps/api/src/services/data/token-metadata.service.ts` — metadata fetching cold path
- `apps/api/src/services/portfolio/position.service.ts` — token balance reading
- Any Birdeye or Helius data transforms that re-parse raw account data
- Any place that creates ATAs for users during swap execution
- `apps/api/src/lib/triton-transaction-parser.ts` — does it correctly identify Token 2022 mints from transaction data?

## Output Format

Produce a table with one row per finding:

| File:Line | Handles Token 2022? | Issue | Recommended Fix |
|-----------|--------------------|----|-----------------|
| `path/to/file.ts:42` | No | Calls `fetchMint` without program ID | Pass `programAddress` argument based on `account.owner` |
| `path/to/file.ts:88` | Partial | ATA derived with default program | Pass `TOKEN_2022_PROGRAM_ID` when mint owner is Token 2022 |

After the table, add a **Summary** section:
- Count of files audited
- Count of confirmed gaps (critical: will fail at runtime)
- Count of partial gaps (warning: will silently produce wrong data)
- Count of clean sites (pass)

Use severity labels:
- `critical` — code will throw or return wrong data for any Token 2022 token
- `warning` — code degrades gracefully but misses extension data or metadata
- `note` — defensive improvement, not currently broken

## Correct Patterns to Reference When Writing Fixes

```typescript
// WRONG: assumes original SPL Token
const mint = await fetchMint(rpc, mintAddress);

// CORRECT: dispatch on owner
const rawAccount = await rpc.getAccountInfo(mintAddress).send();
const programId = rawAccount.value.owner; // either TOKEN_PROGRAM_ID or TOKEN_2022_PROGRAM_ID
const mint = await fetchMint(rpc, mintAddress, { programAddress: programId });

// WRONG: ATA for original SPL only
const ata = getAssociatedTokenAddressSync(mint, owner);

// CORRECT: ATA respects token program
const ata = getAssociatedTokenAddressSync(mint, owner, false, tokenProgramId);

// CORRECT: check for metadata pointer before going to Metaplex
import { getExtensionData, ExtensionType } from '@solana/spl-token';
const metadataPointerData = getExtensionData(ExtensionType.MetadataPointer, mint.tlvData);
if (metadataPointerData !== null) {
  // decode TokenMetadata extension directly, skip Metaplex PDA fetch
} else {
  // fall back to Metaplex metadata PDA
}
```
