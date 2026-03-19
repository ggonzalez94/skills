# Skills

A collection of agent skills I find useful. These are designed to work with any [Agent Skills](https://agentskills.io)-compatible tool — Claude Code, Cursor, OpenClaw, Gemini CLI, and others.

## Install

Pick the skills you want:

```bash
npx skills add ggonzalez94/skills --skill agent-cli-design
```

## Skills

| Skill | Description | Compatibility |
|---|---|---|
| [agent-cli-design](skills/agent-cli-design) | Principles for building CLIs that AI agents can use reliably. Adapted from [Justin Poehnelt's "Rewrite your CLI for AI Agents"](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/), based on patterns from Google's Workspace CLI. Covers machine-readable output, schema introspection, input hardening, safety rails, and multi-surface architecture. | All platforms |
| [babysit](skills/babysit) | Autonomous PR shepherd. Monitors a GitHub PR on a 10-minute loop — checks CI status, addresses review comments (fixes real issues, replies to style nits), resolves merge conflicts, and squash-merges when all checks pass. Safety rule: never merges in the same iteration as a code fix, giving reviewers time to respond. Requires `gh` CLI and the `/loop` skill. | Claude Code only |
| [crypto-wallet-testing](skills/crypto-wallet-testing) | Pre-funded crypto wallet for on-chain testing. Provides agents with access to a funding source for deploying contracts, sending transactions, funding sub-wallets, and bridging across EVM chains. Includes private key resolution, safety rules, and a chain reference with verified token addresses. | All platforms |

## License

MIT
