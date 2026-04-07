---
name: pr-description
description: Use when creating a pull request, writing a PR description, or running gh pr create. Triggers on "create PR", "open PR", "ship", "push and create PR", or any workflow that ends with a pull request.
---

# PR Descriptions

Write PR descriptions that a human teammate would write. Reviewers skip slop.

## Rules

1. **WHY over HOW** — Lead with motivation. The diff shows what changed; the description explains why.
2. **Short** — 2-5 sentences. If you need more, the PR is too big.
3. **No AI attribution** — Never add "Generated with Claude Code", "Codex", or similar.
4. **No test plan checklists** — No `- [x] verified X` lists unless the repo convention uses them.
5. **No markdown headers** — No `## Summary`, `## Changes`, `## Test Plan` sections. Just write naturally.
6. **Match repo voice** — Before writing, check the last 5 merged PRs (`gh pr list --state merged --limit 5 --json title,body`) and match their style.

## Examples

**Bad — AI slop:**
```
## Summary
- Added user authentication module with JWT tokens
- Updated database schema to support user sessions
- Added login/logout API endpoints

## Test Plan
- [x] Unit tests pass
- [x] Manual testing of login flow
- [x] Verified token expiration

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**Good — human:**
```
Users couldn't stay logged in across page refreshes. Added JWT-based auth
with session persistence. Login/logout endpoints at /api/auth/*.
```

**Bad — narrating the diff:**
```
## Changes
Modified `src/components/Header.tsx` to add mobile navigation menu.
Updated `globals.css` with responsive breakpoints. Added hamburger
icon component in `src/components/MenuIcon.tsx`. Updated tests.
```

**Good — context for reviewer:**
```
Nav was unusable on mobile — menu items overflowed off-screen.
Added a hamburger menu that collapses below 768px.
```

## Process

1. Run `gh pr list --state merged --limit 5 --json title,body` to check repo conventions
2. Read your own diff: `git diff main...HEAD --stat` (what changed), `git log main..HEAD --oneline` (why)
3. Write 2-5 sentences: why this change exists, what it does at a high level, anything the reviewer should know
4. Use the repo's PR title convention (e.g., conventional commits: `feat:`, `fix:`)
5. Include TODOs only if they are critical to shipping this feature(e.g. some secrets need to be added to GH actions, or the db needs to be migrated manually)
