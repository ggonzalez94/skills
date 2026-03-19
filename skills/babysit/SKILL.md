---
name: babysit
description: "Monitor and shepherd a GitHub PR to merge: review comments, fix CI, address feedback, and merge when ready. Triggers on: babysit this PR, watch the PR, monitor PR, keep an eye on the PR, shepherd this PR, babysit-prs."
---

# Babysit PR

Continuously monitor a GitHub PR, address review comments, fix CI failures, and merge when everything is green.

## CRITICAL: How to invoke

**Your ONLY action when this skill is triggered is to invoke `/loop 10m` with the prompt below.** Do NOT manually execute the steps yourself. Do NOT run any `gh` commands directly. Just call the `loop` skill and pass it the prompt between the `BEGIN PROMPT` and `END PROMPT` markers.

Tell the user: "Starting babysit loop — I'll check PR #N every 10 minutes, address comments, fix CI, and merge when ready."

Then invoke `/loop 10m` with this prompt:

<!-- BEGIN PROMPT -->

You are babysitting a GitHub PR. Your job is to shepherd it to merge AUTONOMOUSLY — do not ask the user for confirmation at any step, including merging.

**Critical rule: merge ONLY at the top of an iteration, NEVER after making fixes.** When you fix code or address comments, you must wait for the next iteration before merging. This gives reviewers time to review your changes.

### Step 1: Identify the PR

Run `gh pr view --json number,title,state,url,headRefName,statusCheckRollup,reviewDecision,isDraft` to get the current PR for this branch.

If no PR exists on the current branch, report this and stop.

If the PR is already merged or closed, report this and cancel the loop by saying: "PR is already merged/closed. Stopping babysit loop." then use the tool to cancel.

### Step 2: Check CI and comments (read-only assessment)

Gather the full picture before taking any action:

**CI status:** Run `gh pr checks` to see the status of all CI checks.

**Review comments:** Run `gh api repos/{owner}/{repo}/pulls/{number}/comments --jq '.[] | select(.in_reply_to_id == null) | {id, path, line, body, user: .user.login}'` to get open review comments. Also check for PR-level review comments: `gh api repos/{owner}/{repo}/pulls/{number}/reviews --jq '.[] | select(.state != "APPROVED" and .state != "DISMISSED") | {id, state, body, user: .user.login}'`

**Draft status:** Check if the PR is in draft state.

### Step 3: Merge gate — if everything is already green, merge now

If ALL of these conditions are true **and you did NOT make any changes in this iteration**, merge IMMEDIATELY:

- All CI checks pass (no pending, no failures)
- No unresolved review comments remain
- PR is not in draft state
- No merge conflicts

Merge command:
```bash
gh pr merge --squash --auto
```

Then report: "PR merged successfully. Stopping babysit loop." and cancel the loop.

**You have explicit permission to merge. Do NOT ask for confirmation.**

If any condition is not met, proceed to Step 4.

### Step 4: Fix issues (if any)

**If CI checks are pending/in-progress:** Report status and wait for next iteration. Do not proceed further.

**If CI checks failed:** Investigate the failure. Run `gh run view <run-id> --log-failed` to get failure logs. Fix the issue if it's a code problem (test failure, lint error, type error, build failure). Commit and push the fix.

**If there are unresolved review comments**, evaluate each one:

1. **Is the issue real?** Does the comment point to an actual bug, security issue, or code quality problem?
2. **Is the proposed fix optimal?** If the reviewer suggests a specific change, is it the best approach?

Then act:

- **Real issue, good suggestion**: Implement the fix. Commit with message `fix(scope): address review — <what was fixed>`.
- **Real issue, better alternative exists**: Implement the better fix. Reply to the comment explaining your approach.
- **Not a real issue / style preference / misunderstanding**: Reply to the comment explaining why the current code is correct or why the change isn't needed. Be respectful and specific.
- **Question (not a change request)**: Reply with the answer.

Use `gh api repos/{owner}/{repo}/pulls/{number}/comments/{id}/replies -f body="<reply>"` to reply to comments.

After addressing all comments, push any fixes.

**DO NOT merge after making fixes.** Wait for the next iteration — this gives reviewers a chance to review your changes before merge.

### Step 5: Status Report

At the end of each iteration, output a brief status:
- CI: passing/failing/pending
- Comments: N addressed, M remaining
- Merge readiness: ready/blocked (reason)
- Action taken: what you did this iteration

If nothing changed since the last check, just report "No changes. Waiting." to keep output minimal.

<!-- END PROMPT -->

## Notes

- The skill uses `gh` CLI throughout — ensure it's authenticated.
- Owner/repo are inferred from the git remote. Use `gh repo view --json nameWithOwner -q .nameWithOwner` if needed.
- When fixing code, follow existing project conventions (check CLAUDE.md).
- Don't force-push or rewrite history — always add new commits.
- If you encounter a merge conflict, do your best to resolve it. If unsure, report it back.
