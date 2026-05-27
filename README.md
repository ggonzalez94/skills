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
| [crypto-wallet-testing](skills/crypto-wallet-testing) | Pre-funded crypto wallet for on-chain testing. Provides agents with access to a funding source for deploying contracts, sending transactions, funding sub-wallets, and bridging across EVM chains. Builds on the open-wallet skill when OWS is available, falls back to raw private key resolution. Includes safety rules and a chain reference with verified token addresses. | All platforms |
| [gh-image-video](skills/gh-image-video) | Host images, videos, and other binary attachments on a public Google Cloud Storage bucket (`gh-evidence`) so they can be embedded in GitHub PR descriptions, comments, and reviews without committing them to the repo. Preflight checks (gcloud installed / logged in / bucket reachable), first-run bucket setup, sensitivity heuristics that pause before uploading anything that looks private, and per-file-type markdown snippets (`![](...)` for images, bare URL for videos so GitHub auto-embeds, plain link for everything else). Requires `gcloud` CLI. | All platforms |
| [initial-pr-review](skills/initial-pr-review) | First-pass PR review focused on context and direction, not line-by-line nits. Answers five short questions: why does this exist, does it fit the product direction, is the architecture sound, any obvious regressions, any big security issues. Investigates deeply (reads the full diff, linked issues, surrounding code, CI status, reviewer discussion) but reports tersely — 1-3 sentences per answer. Optional interactive HTML report with a light theme. Requires `gh` CLI. | All platforms |
| [open-wallet](skills/open-wallet) | Open Wallet Standard (OWS) — the universal wallet layer for AI agents. Local-first encrypted storage, policy-gated signing, and scoped API keys so agents never see raw private keys. Supports EVM, Solana, Bitcoin, Cosmos, Tron, TON, Sui, Filecoin, and Spark from a single mnemonic. Covers CLI, Node.js SDK, Python SDK, policy engine, and multi-chain reference. | All platforms |
| [pr-description](skills/pr-description) | Concise, human-sounding PR descriptions. Enforces WHY-over-HOW writing, kills AI slop (no "Generated with Claude Code", no test checklists, no `## Summary` headers), and checks repo conventions before writing. Includes bad/good examples. | All platforms |

## License

MIT
