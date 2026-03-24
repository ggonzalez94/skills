# Agents Guide

## Eval Methodology

When running skill evals (with-skill vs without-skill comparisons):

- **Always run without-skill baselines in isolated worktrees** (`isolation: "worktree"`) with explicit instructions not to read local files or search the web. Agents are smart enough to explore the codebase and find skill files, which invalidates the baseline.
- Tell baseline agents: "Do NOT read any files in the current repository. Do NOT use Glob, Read, or Grep to explore the codebase. You must answer purely from your own knowledge. Do NOT search the web either."
- After runs complete, verify isolation by checking the agent transcript for Read/Glob tool calls. If the baseline agent read skill files, the results are invalid.
- With-skill agents can run in the normal working directory since they're explicitly given the skill path.

## Workspace Conventions

- Eval artifacts go in `skills/<skill-name>-workspace/` directories (gitignored)
- Organize by iteration: `iteration-1/`, `iteration-2/`, etc.
- Each eval gets a descriptive directory: `eval-1-base-wallet-setup/`
- Inside each eval: `with_skill/outputs/` and `without_skill/outputs/`
