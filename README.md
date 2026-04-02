# Claude Code Skills

Custom skills for Claude Code built for my Solana development, security auditing, and developer workflows. Each skill is a standalone SKILL.md that provides specialized context to Claude Code when working on relevant tasks.

## Solana Development

| Skill | Description |
|-------|-------------|
| [solana-master](./solana-master) | Full-stack Solana dApp development with strict coding standards |
| [pinocchio-development](./pinocchio-development) | Build high-performance Solana programs with Pinocchio (88-95% compute unit reduction) |
| [carbon-decoder](./carbon-decoder) | Build Carbon framework decoders for Solana program indexing |
| [borsh-schema-gen](./borsh-schema-gen) | Generate TypeScript Borsh codecs from Rust struct definitions |
| [discriminator-verify](./discriminator-verify) | Verify Anchor instruction and account discriminators |
| [pda-audit](./pda-audit) | Audit PDA derivations for correctness |
| [solana-dev](./solana-dev) | End-to-end Solana development playbook |
| [solana-security-checklist](./solana-security-checklist) | Client-side security checklist for Solana TypeScript |
| [token2022-audit](./token2022-audit) | Audit Token 2022 vs SPL Token handling gaps |

## Security

| Skill | Description |
|-------|-------------|
| [vulnhunter](./vulnhunter) | Vulnerability detection and variant analysis |
| [zz-code-recon](./zz-code-recon) | Deep architectural context building for security audits |

## Developer Workflow

| Skill | Description |
|-------|-------------|
| [tla-precheck](./tla-precheck) | State machine verification using TLA+ TypeScript DSL |
| [improve-claude-md](./improve-claude-md) | Optimize CLAUDE.md files for better AI instruction adherence |

## Installation

Copy the skill directory into your project's `.claude/skills/` directory:

```bash
# Clone the repo
git clone https://github.com/Jerome2332/claude-skills.git

# Copy a skill to your project
cp -r claude-skills/solana-master .claude/skills/solana-master
```

Or reference individual SKILL.md files directly in your Claude Code configuration.

## License

MIT
