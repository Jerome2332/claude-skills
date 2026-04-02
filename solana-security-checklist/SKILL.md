---
name: solana-security-checklist
description: Run the Solana client-side security checklist against TypeScript code that interacts with Solana programs. Use before merging any code that touches swap execution, wallet operations, fee calculations, or account data reading.
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/solana-security-checklist/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/solana-security-checklist/feedback.log
```

Only log general preferences, not task-specific details.

---
# Solana Security Checklist

You are running a security checklist audit on TypeScript client code that interacts with Solana programs. This checklist is adapted from Solana program security best practices, applied to the TypeScript client layer where the same classes of bugs appear.

The user will point you to a file or directory. Review it against every item below.

## The Checklist

### 1. Account Owner Validation

When receiving account data from RPC, is `account.owner` checked before trusting or deserializing the data?

**Why it matters**: An attacker can create an account with any data at any address. Without owner validation, your code may deserialize attacker-controlled data as a legitimate program account.

**What to look for**:
- Any call to `getAccountInfo` or `getMultipleAccountsInfo` followed by `data` parsing without checking `account.owner`
- Any deserializer that accepts raw bytes without first asserting the account is owned by the expected program
- Bonding curve reads that don't verify the account is owned by `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P`

**Correct pattern**:
```typescript
const account = await rpc.getAccountInfo(address).send();
if (account.value?.owner !== EXPECTED_PROGRAM_ID) {
  throw new Error('Unexpected account owner');
}
// Safe to deserialize now
```

---

### 2. Signer Verification

Are signatures verified before executing any writes or mutations?

**What to look for**:
- Swap execution endpoints that accept a `signedTransaction` parameter — is the transaction actually signed before submission?
- Any place where a wallet address from user input is used to authorize an action without verifying the user controls that wallet
- In the Scope project: does every write endpoint call `fastify.authenticate` middleware?

**Correct pattern**: The signed transaction carries its own signature verification — the RPC node rejects unsigned transactions. BUT: the API must verify that the `walletId` in the request body actually belongs to the authenticated user (per CLAUDE.md invariant: explicit `walletId` required on all writes).

---

### 3. PDA Re-Derivation Verification

When a PDA is received as input (from a client request or from an account's data fields), is it re-derived from known seeds to verify it is the correct account?

**Why it matters**: An attacker can pass a different account at a PDA-expected slot. If the program doesn't re-derive and compare, it will operate on the wrong account.

**What to look for**:
- Any route handler that accepts a `bondingCurve` address from the request body and uses it without re-deriving from `mint`
- Any swap handler that accepts pool addresses from client input without verifying they correspond to the expected mints

---

### 4. Integer Overflow Safety

All arithmetic on lamport and token amounts must use `BigInt`. No intermediate `number` coercions.

**What to look for**:
- Any `* 0.01`, `/ 100`, or similar floating-point math on lamport or token amounts
- Any `Number(bigintValue)` where the value could exceed `Number.MAX_SAFE_INTEGER` (9 quadrillion lamports ≈ 9000 SOL — easily exceeded for token amounts with 9 decimals)
- Any `parseInt` or `parseFloat` on balance or fee strings from RPC responses

**Correct pattern**:
```typescript
// WRONG: precision loss
const fee = Number(amount) * 0.01;

// CORRECT: integer basis points
const fee = (amount * 100n) / 10000n;
```

**Scope-specific invariant**: Fee calculations must use integer basis points (100 bps = 1%). Never floating-point. This is a non-negotiable invariant per CLAUDE.md.

---

### 5. Input Validation

All external inputs — mint addresses, amounts, slippage values, wallet IDs — must be validated with Zod (or equivalent runtime schema validation) before use.

**What to look for**:
- Route handlers that read `request.body.mint` and use it as a `PublicKey` without first validating it is a valid base58 address
- Slippage values used directly without bounding to a sane range (e.g., 0–5000 bps)
- Amount fields used without checking they are positive integers
- Referral codes used without matching against `REFERRAL_CODE_REGEX` from `packages/shared/src/constants/referrals.ts`

---

### 6. Account Initialization Check

Are accounts checked for existence and initialization before reads?

**Why it matters**: Reading an uninitialized account returns zero-filled data that will deserialize to plausible-looking but wrong values (e.g., a bonding curve with all zeros looks like a valid account with no reserves).

**What to look for**:
- Any `getAccountInfo` result used without checking `account.value !== null`
- Any deserialization that proceeds when `data.length === 0`
- Bonding curve reads that don't verify `complete === false` before attempting a buy

---

### 7. No Floating-Point in Financial Math

No `*` or `/` operations on JavaScript `number` type for any lamport, token, fee, or basis-point calculation.

**What to look for**: Same as item 4 — a separate pass focusing specifically on financial arithmetic expressions, not just BigInt conversion.

**This is a non-negotiable invariant per CLAUDE.md.** Any violation is `critical`.

---

### 8. Error Messages Do Not Leak Sensitive Data

Error messages returned to the client must not include:
- Private keys or seed phrases
- Raw session tokens or JWTs
- Internal Turnkey sub-organization IDs (beyond what the user already knows)
- Internal database IDs used as security boundaries
- Full stack traces in production (file paths, line numbers)

**What to look for**:
- Any `catch (e) { throw e }` that re-throws raw errors to the HTTP response
- Any error handler that includes `error.stack` in the response body
- Log lines that include `privateKey`, `seed`, or `token` fields

---

### 9. Idempotency Keys on Swap Execution

Every swap execution must include an idempotency key to prevent double-spend on retry.

**This is a non-negotiable invariant per CLAUDE.md.**

**What to look for**:
- `POST /api/swap/execute` — does the handler require an `idempotencyKey` in the request body?
- Is the idempotency key checked in Redis (or equivalent) before processing?
- Is the key stored atomically with the swap result to cover the race condition?

---

### 10. Wallet ID Explicit on All Write Operations

Every write operation (swap, order create, order cancel) must use an explicit `walletId` — never implicit wallet selection.

**This is a non-negotiable invariant per CLAUDE.md.**

**What to look for**:
- Any trading endpoint that falls back to the user's primary wallet without requiring `walletId` in the request
- Any order creation that infers the wallet from context rather than requiring the caller to specify it
- Frontend components that submit swaps without including the selected `walletId` from `useUserWallets()`

---

## Scope-Project-Specific Checks

These items are specific to this project's architecture:

- **No direct Solana RPC calls outside `src/solana/`** — per CLAUDE.md forbidden patterns, all RPC must go through the RPC Manager
- **No `any` type for external API responses** — must use `unknown` + runtime validation
- **Fee calculations use 100 bps (1%)** — never a different hardcoded value
- **Rankings reset at midnight UTC** — any date/time logic in the ranking pipeline must use UTC, not local timezone
- **Soft-delete for wallets via `disconnected_at`** — no hard-deletes on wallet records

## Output Format

Produce a checklist table:

| # | Check | Status | File:Line (if fail) | Severity | Recommended Fix |
|---|-------|--------|---------------------|----------|-----------------|
| 1 | Account owner validation | pass | — | — | — |
| 4 | Integer overflow safety | fail | `apps/api/src/services/trading/swap.service.ts:88` | critical | Replace `Number(amount) * 0.01` with `(amount * 100n) / 10000n` |
| 9 | Idempotency keys | n/a | — | — | Not applicable to this file |

Severity:
- `critical` — must fix before merge; exploitable or invariant violation
- `warning` — should fix; degrades correctness or safety
- `note` — optional defensive improvement
- `pass` — no issue found
- `n/a` — check not applicable to the code reviewed

After the table, summarize:
- Total critical findings
- Total warning findings
- Overall verdict: PASS (zero criticals), CONDITIONAL (warnings only), FAIL (any critical)
