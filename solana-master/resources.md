# Curated Resources (Source-of-Truth First)

## Quick Reference

| Need | Go To |
|------|-------|
| Anchor docs | https://www.anchor-lang.com/docs |
| Solana Kit | https://solanakit.com |
| Solana Kite | https://solanakite.org |
| Core Solana docs | https://solana.com/docs |
| Agave/CLI | https://docs.anza.xyz/ |
| RPC API Reference | https://solana.com/docs/rpc |
| Actions/Blinks | https://solana.com/docs/advanced/actions |

## LLM-Friendly Docs (Markdown Format)

The Solana Foundation provides markdown versions of docs for LLM consumption:

```
# Pattern: Add raw GitHub URL prefix
https://raw.githubusercontent.com/solana-foundation/developer-content/main/docs/

# Examples:
/docs/core/accounts.md
/docs/advanced/actions.md
/docs/rpc/http/sendTransaction.mdx
```

**Note**: The developer-content repo was archived Jan 2025; content moved to solana-com repo.

---

## Learning Platforms

| Resource | Description |
|----------|-------------|
| [Blueshift](https://learn.blueshift.gg/) | Free, open-source Solana learning platform |
| [Blueshift GitHub](https://github.com/blueshift-gg) | Course content and tools |
| [Solana Cookbook](https://solanacookbook.com/) | Practical recipes and examples |

---

## Core Solana Documentation

| Resource | Description |
|----------|-------------|
| [Solana Documentation](https://solana.com/docs) | Core, RPC, Frontend, Programs |
| [Next.js + Solana React Hooks](https://solana.com/docs/frontend/nextjs-solana) | Official frontend guide |
| [@solana/web3-compat](https://solana.com/docs/frontend/web3-compat) | Legacy interop guide |
| [RPC API Reference](https://solana.com/docs/rpc) | All RPC methods |

---

## Modern JS/TS SDK

| Resource | Description |
|----------|-------------|
| [@solana/kit Repository](https://github.com/anza-xyz/kit) | Source code |
| [Solana Kit Docs](https://solana.com/docs/clients/kit) | Installation, upgrade guide |
| [Solana Kit Website](https://solanakit.com) | Quick reference |

---

## UI and Wallet Infrastructure

| Resource | Description |
|----------|-------------|
| [framework-kit Repository](https://github.com/solana-foundation/framework-kit) | @solana/client, @solana/react-hooks |
| [ConnectorKit](https://github.com/civic-io/connector-kit) | Headless Wallet Standard connector |
| [Solana Kite](https://solanakite.org) | Kit utilities and helpers |

---

## Scaffolding

| Resource | Description |
|----------|-------------|
| [create-solana-dapp](https://github.com/solana-developers/create-solana-dapp) | Official project scaffolder |

---

## Program Frameworks

### Anchor

| Resource | Description |
|----------|-------------|
| [Anchor Repository](https://github.com/coral-xyz/anchor) | Source code |
| [Anchor Documentation](https://www.anchor-lang.com/) | Official docs |
| [Anchor Version Manager (AVM)](https://www.anchor-lang.com/docs/avm) | Version management |

### Pinocchio

| Resource | Description |
|----------|-------------|
| [Pinocchio Repository](https://github.com/anza-xyz/pinocchio) | Source code |
| [pinocchio-system](https://crates.io/crates/pinocchio-system) | System program helpers |
| [pinocchio-token](https://crates.io/crates/pinocchio-token) | Token program helpers |
| [Pinocchio Guide](https://github.com/vict0rcarvalh0/pinocchio-guide) | Community guide |
| [How to Build with Pinocchio (Helius)](https://www.helius.dev/blog/pinocchio) | Tutorial |

---

## Testing

### LiteSVM

| Resource | Description |
|----------|-------------|
| [LiteSVM Repository](https://github.com/LiteSVM/litesvm) | Source code |
| [litesvm crate](https://crates.io/crates/litesvm) | Rust package |
| [litesvm npm](https://www.npmjs.com/package/litesvm) | TypeScript package |

### Mollusk

| Resource | Description |
|----------|-------------|
| [Mollusk Repository](https://github.com/buffalojoec/mollusk) | Source code |
| [mollusk-svm crate](https://crates.io/crates/mollusk-svm) | Rust package |

### Surfpool

| Resource | Description |
|----------|-------------|
| [Surfpool Documentation](https://docs.surfpool.dev/) | Official docs |
| [Surfpool Repository](https://github.com/txtx/surfpool) | Source code |

---

## IDLs and Codegen

| Resource | Description |
|----------|-------------|
| [Codama Repository](https://github.com/codama-idl/codama) | Universal IDL format |
| [Codama Generating Clients](https://solana.com/docs/programs/codama-generating-clients) | Official guide |
| [Shank (Metaplex)](https://github.com/metaplex-foundation/shank) | Native Rust IDL generation |
| [Kinobi (Metaplex)](https://github.com/metaplex-foundation/kinobi) | Client generation |

---

## Tokens and NFTs

| Resource | Description |
|----------|-------------|
| [SPL Token Documentation](https://spl.solana.com/token) | Classic token program |
| [Token-2022 Documentation](https://spl.solana.com/token-2022) | Token Extensions |
| [Metaplex Documentation](https://developers.metaplex.com/) | NFT standards |

---

## Payments

| Resource | Description |
|----------|-------------|
| [Commerce Kit Repository](https://github.com/solana-foundation/commerce-kit) | Payment components |
| [Commerce Kit Documentation](https://commercekit.solana.com/) | Official docs |
| [Kora Documentation](https://docs.kora.network/) | Payment infrastructure |

---

## Security

| Resource | Description |
|----------|-------------|
| [Blueshift Program Security Course](https://learn.blueshift.gg/en/courses/program-security) | Free security course |
| [Solana Security Best Practices](https://solana.com/docs/programs/security) | Official guide |
| [Neodyme Security Workshop](https://workshop.neodyme.io/) | Hands-on security |

---

## Performance and Optimization

| Resource | Description |
|----------|-------------|
| [Solana Optimized Programs](https://github.com/Laugharne/solana_optimized_programs) | Optimization patterns |
| [sBPF Assembly SDK](https://github.com/blueshift-gg/sbpf) | Low-level optimization |
| [Doppler Oracle (21 CU)](https://github.com/blueshift-gg/doppler) | Ultra-optimized example |

---

## RPC Providers

| Provider | Description |
|----------|-------------|
| [Helius](https://helius.dev/) | Full-featured RPC + webhooks |
| [QuickNode](https://quicknode.com/) | Multi-chain RPC |
| [Triton](https://triton.one/) | High-performance RPC |
| [Alchemy](https://alchemy.com/) | Enterprise RPC |

---

## DO NOT USE (Deprecated/Outdated)

| Source | Reason |
|--------|--------|
| Solana Labs documentation | Company replaced by Anza |
| Project Serum tools/docs | Collapsed years ago |
| @solana/web3.js v1 docs | Use @solana/kit instead |
| Old wallet-adapter docs | Use framework-kit instead |

---

## Community

| Resource | Description |
|----------|-------------|
| [Solana Stack Exchange](https://solana.stackexchange.com/) | Q&A |
| [Solana Discord](https://discord.gg/solana) | Community chat |
| [Anchor Discord](https://discord.gg/anchor) | Anchor-specific help |
| [Solana GitHub](https://github.com/solana-labs) | Core repos |
| [Anza GitHub](https://github.com/anza-xyz) | Validator + Kit repos |
