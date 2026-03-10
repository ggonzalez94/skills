# Skills

A collection of agent skills I find useful. These are designed to work with any [Agent Skills](https://agentskills.io)-compatible tool — Claude Code, Cursor, OpenClaw, Gemini CLI, and others.

## Install

Pick the skills you want:

```bash
npx skills add ggonzalez94/skills --skill agent-cli-design
```

## Skills

| Skill | Description |
|---|---|
| [agent-cli-design](skills/agent-cli-design) | Principles for building CLIs that AI agents can use reliably. Adapted from [Justin Poehnelt's "Rewrite your CLI for AI Agents"](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/), based on patterns from Google's Workspace CLI. Covers machine-readable output, schema introspection, input hardening, safety rails, and multi-surface architecture. |
| [crypto-wallet-testing](skills/crypto-wallet-testing) | Pre-funded crypto wallet for on-chain testing. Provides agents with access to a funding source for deploying contracts, sending transactions, funding sub-wallets, and bridging across EVM chains. Includes private key resolution, safety rules, and a chain reference with verified token addresses. |

## License

MIT
