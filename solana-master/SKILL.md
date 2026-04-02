---
name: solana-master
description: Authoritative Solana development skill (Feb 2026). Covers full-stack dApp development with strict coding standards. UI via @solana/client + @solana/react-hooks, SDK via @solana/kit, programs via Anchor/Pinocchio, testing via LiteSVM/Mollusk/Surfpool. Enforces TypeScript style (Array<T>, no abbreviations, no any), deletionist philosophy, comprehensive security checklists, and TDD practices.
user-invocable: true
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/solana-master/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/solana-master/feedback.log
```

Only log general preferences, not task-specific details.

---
# Solana Master Skill

The authoritative, consolidated Solana development skill combining best practices from the Solana Foundation, Anchor guidelines, and production-hardened patterns.

## What This Skill Covers

Use this skill for:
- Solana dApp UI (React / Next.js)
- Wallet connection + signing flows
- Transaction building / sending / confirmation UX
- On-chain program development (Anchor or Pinocchio)
- Client SDK generation (typed program clients via Codama)
- Local testing (LiteSVM, Mollusk, Surfpool)
- Security hardening and audit-style reviews
- TypeScript/Rust code quality enforcement

## Success Criteria

**Before declaring any task complete:**
1. Run the project's test command (e.g., `pnpm test`, `npm test`, `cargo test-sbf`)
2. Run the build command (e.g., `pnpm build`, `cargo build-sbf`)
3. If tests or build fail, there is more work to do
4. Never show completion indicators until all tests pass and build succeeds

## Core Philosophy

### You Are a Deletionist

> "Perfection isn't achieved when there's nothing more to add, rather perfection is achieved when there is nothing more to be taken away."

**Remove:**
- Comments that simply repeat what the code does or the variable name
- Repeated code that should be a named function
- Unused imports, constants, files, and stale comments
- Magic numbers (replace with named constants or computed values)

### Code Honesty

Never deceive anyone reading this code. Deception includes:
- Variable names that don't match the variable's purpose
- Comments that no longer describe the code
- Temporary workarounds not labeled with `TODO` comments

---

## Default Stack Decisions (Opinionated)

### 1. UI: framework-kit First
- Use `@solana/client` + `@solana/react-hooks`
- Prefer Wallet Standard discovery/connect via framework-kit
- Bootstrap with `create-solana-dapp` for new projects

### 2. SDK: @solana/kit First
- Prefer Kit types (`Address`, `Signer`, transaction message APIs, codecs)
- Prefer `@solana-program/*` instruction builders over hand-rolled data
- Generate typed clients via Codama, never hand-write Borsh layouts

### 3. Legacy Compatibility: web3.js Only at Boundaries
- If a dependency requires web3.js objects (`PublicKey`, `Transaction`, `Connection`), use `@solana/web3-compat` as the boundary adapter
- Never let web3.js types leak across the app; contain them to adapter modules
- Do not create new web3.js v1 code or use `@coral-xyz/anchor` TS client in new code

### 4. Programs
- **Default**: Anchor (fast iteration, IDL generation, mature tooling)
- **Performance**: Pinocchio when you need CU optimization, minimal binary size, zero dependencies, or fine-grained control

### 5. Testing
- **Unit tests**: LiteSVM or Mollusk (fast, in-process)
- **Integration tests**: Surfpool (realistic cluster state)
- **TypeScript tests**: Use `node:test` + `tsx`, not Jest/Mocha/Vitest
- Use `solana-test-validator` only for specific RPC behaviors not emulated by LiteSVM

### 6. Package Manager
- Use `npm` or `pnpm`
- Do not use `yarn` (unnecessary dependency)

---

## Documentation Sources

Use these official sources (in priority order):

| Domain | Source |
|--------|--------|
| Anchor | https://www.anchor-lang.com/docs |
| Solana Kit | https://solanakit.com |
| Solana Kite | https://solanakite.org |
| Solana Docs | https://solana.com/docs |
| Agave/CLI | https://docs.anza.xyz/ |
| Pinocchio | https://github.com/anza-xyz/pinocchio |

**Do NOT use:**
- Solana Labs documentation (replaced by Anza)
- Project Serum tools/docs (collapsed years ago)

---

## Operating Procedure

### 1. Classify the Task Layer
- UI/wallet/hook layer
- Client SDK/scripts layer
- Program layer (+ IDL)
- Testing/CI layer
- Infrastructure (RPC/indexing/monitoring)

### 2. Pick the Right Building Blocks
| Layer | Building Blocks |
|-------|-----------------|
| UI | framework-kit patterns, `@solana/react-hooks` |
| Scripts/backends | `@solana/kit` directly |
| Legacy library present | `@solana/web3-compat` adapter boundary |
| High-performance programs | Pinocchio over Anchor |

### 3. Implement with Solana-Specific Correctness
Always be explicit about:
- Cluster + RPC endpoints + WebSocket endpoints
- Fee payer + recent blockhash
- Compute budget + prioritization (where relevant)
- Expected account owners + signers + writability
- Token program variant (SPL Token vs Token-2022) and extensions

### 4. Add Tests
- Unit test: LiteSVM or Mollusk
- Integration test: Surfpool
- For wallet UX: mocked hook/provider tests
- TypeScript: `node:test` with `tsx` runner

### 5. Deliverables
When implementing changes, provide:
- Exact files changed + diffs
- Commands to install/build/test
- A "risk notes" section for anything touching signing/fees/CPIs/token transfers

---

## Platform Awareness

Remember this is **Solana**, not Ethereum:
- Use "programs" not "smart contracts"
- Use "transaction fees" not "gas"
- There are no "mempools"

**Token terminology:**
- "Token Extensions Program" or "Token-2022" for the newer program
- "Classic Token Program" or "SPL Token" for the older program

---

## Git Commits

- Do not add "Co-Authored-By: Claude" or similar attribution
- Use conventional commit format: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- Test locally before committing

---

## Progressive Disclosure (Read When Needed)

### Core Development

| Topic | File |
|-------|------|
| TypeScript coding standards | [typescript-style.md](typescript-style.md) |
| UI + wallet + hooks | [frontend-framework-kit.md](frontend-framework-kit.md) |
| Kit ↔ web3.js boundary | [kit-web3-interop.md](kit-web3-interop.md) |
| Anchor programs | [programs-anchor.md](programs-anchor.md) |
| Pinocchio programs | [programs-pinocchio.md](programs-pinocchio.md) |
| Testing strategy | [testing.md](testing.md) |
| IDLs + codegen | [idl-codegen.md](idl-codegen.md) |
| Security checklist | [security.md](security.md) |

### Advanced Topics (NEW - Feb 2026)

| Topic | File |
|-------|------|
| Actions, Blinks, ALTs, State Compression, v0 Transactions | [advanced-patterns.md](advanced-patterns.md) |
| Transaction lifecycle, confirmation, retry, fees | [transactions.md](transactions.md) |
| Tokens, Token-2022, Extensions, NFTs | [tokens.md](tokens.md) |
| Clusters, deployment, local validator, RPC | [infrastructure.md](infrastructure.md) |
| Staking, inflation, rent economics | [economics.md](economics.md) |

### Reference

| Topic | File |
|-------|------|
| Curated links + resources | [resources.md](resources.md) |

---

## Acknowledgment

When working on a Solana project, acknowledge that these guidelines have been applied to indicate you have read and understood them.
