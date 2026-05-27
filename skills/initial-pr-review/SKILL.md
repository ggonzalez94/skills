---
name: initial-pr-review
description: Use this skill whenever the user wants a first read, eyeball, first look, first pass, sanity check, or high-level take on a pull request — they want framing and judgment (why it exists, does it fit the direction, is the approach sound, anything scary), NOT a line-by-line code review. Trigger on phrases like "what does this PR do", "explain this PR", "give me context on PR", "take a look at PR #N", "should we ship this", "is this heading the right way", "quick read before I approve", or any time the user shares a PR number/URL/branch with low context (teammate's PR, boss dropped a link, need to approve in N minutes) and asks about purpose, direction, architecture, or risk. Especially trigger when the user explicitly disclaims depth ("no nits", "skip the typos", "not a detailed review", "just the shape", "just the framing"). Prefer this over code-review, security-review, pr-description, babysit, or pr-summary skills. Infer PR from current branch; ask if none.
---

# Initial PR Review

A first pass on a pull request. The goal is to help the reviewer decide: should this exist at all, and is this roughly the right shape?

**Investigate deeply. Report tersely.** Do the homework — read the full diff, the linked issue, the surrounding code, recent merged work in the area. Then compress what you learned into five short answers. Depth is in the *thinking*, not the *output*.

This is not a correctness review, a style review, or a test-coverage review — line-by-line nits belong in a deeper pass.

## Step 1: Find the PR

Run `gh pr view --json number,title,body,headRefName,baseRefName,author,url`.

- If a PR exists on the current branch, use its number.
- If it fails (no PR for this branch), ask the user: "I couldn't find a PR for this branch. What PR number or URL should I review?" Wait for an answer before continuing.

## Step 2: Investigate (do real work here)

Take your time. The output is short, but the analysis behind it should be thorough — without depth, the five answers in Step 3 are just guesses.

In parallel, pull:

- PR body, title, linked issues, labels: `gh pr view <N> --json title,body,closingIssuesReferences,labels,reviews,comments`
- File-level stats: `gh pr diff <N> --name-only`
- The full diff: `gh pr diff <N>` — **read it carefully**, not just the file list
- Recent merged PRs near the same area, to read the product direction: `gh pr list --state merged --limit 15 --json title,labels,mergedAt`
- Any issue the PR closes or references: `gh issue view <issue-N> --json title,body,state,comments`. If the PR body mentions an issue informally ("fixes #42", "for ticket FOO-1"), look it up.

Then go beyond the PR itself:

- **Read the changed files in context.** For non-trivial changes, fetch the PR locally (`gh pr checkout <N>` in a worktree, or use `gh api` to grab files at the head commit) and read the surrounding code, not just the diff hunks. A diff hunk that looks fine in isolation can be wrong in context.
- **Look at what calls into the changed code.** Grep for callers of modified functions/classes. A change is "right" only if its callers stay correct.
- **Check tests and CI.** Are there new tests? Do existing tests still cover the changed paths? What does CI say? (`gh pr checks <N>`)
- **Sample the open-questions surface.** Look at PR review comments and discussion — if reviewers are already debating something, that's a signal about what to focus on.

Spend the tokens. A confident "yes/no" from a careful read beats a hedged answer from a skim.

## Step 3: Answer the five questions — tersely

Output exactly this structure. Keep each answer to **1-3 short sentences**, max. No extra headers. No long paragraphs unless the user explicitly asks for more depth — your job is to compress everything you learned into a quick read.

```
**Why does this PR exist?**
<What it does, from the user's perspective. Plain language a non-engineer could follow.>

**Does it fit the product direction?**
<Aligned / unclear / off. Reference the linked issue or recent work. Say "no linked issue" if there isn't one.>

**Is this the right architecture?**
<Would you build it this way from scratch? If not, give the one-sentence alternative.>

**Obvious regressions?**
<Yes (list briefly) / No / Unclear (say what to check). Only call out things you have actual evidence for from Step 2 — not speculation.>

**Big security issues?**
<Yes (list briefly) / No / Unclear (say what to check). Only flag high-confidence, high-impact issues.>
```

If you found specific bugs while reading the diff, **don't dump them here.** Hold them — at the end of the output, add one line: "Found N specific issues during review; run `/code-review` for the full pass." That respects the format while not losing the work.

## Step 4 (optional): Render as interactive HTML

Only do this if the user explicitly asks ("show me as html", "render this", "make me a page", "give me a visual report", etc.). Default output is plain markdown — don't generate HTML unless asked.

When asked:

1. Copy `assets/report-template.html` from this skill to a temp path (e.g. `/tmp/initial-pr-review-<PR_NUMBER>.html`).
2. Fill in the placeholders (`{{PR_TITLE}}`, `{{Q1_BODY}}`, `{{Q2_PILL_CLASS}}` etc.). Pill classes are `good`, `warn`, or `bad`; pill text is short (`Aligned`, `Unclear`, `Risk`, `No`, `Yes`).
3. Open it with `open /tmp/initial-pr-review-<PR_NUMBER>.html` on macOS, or print the path for the user to open.

**Always use a light background.** The template ships with one — don't switch to a dark theme even if the user prefers dark mode elsewhere. PR reviews get shared and screenshotted; light backgrounds render predictably everywhere.

If the user asks for a tweak (different color, extra section), edit the copy in `/tmp/`, not the template in the skill.

## Style rules

- **Simple terms.** Someone outside the team should follow "why this PR exists." Avoid jargon when a plain word works.
- **Concise output, deep analysis.** Short doesn't mean shallow — short means you did the work and compressed it. If you can't justify a stance, go back to Step 2.
- **Pick a stance.** Don't hedge every question. "Unclear" is a real answer, but only when it's earned — say what would resolve it.
- **No line-by-line nits in the output.** Hold them for `/code-review`.

## When to use something else

This skill is for the *first* read. If the user wants:

- A **detailed**, line-by-line code review → `/code-review`
- To **create** a PR (write the title/body) → `pr-description`
- To **monitor and merge** a PR over time → `babysit`
- To **summarize** what a PR did (for a changelog or handoff) → `pr-summary`
- A **security audit** of contracts or sensitive code → `security-review`

If the user's intent is somewhere between "give me context" and "do a full review", default to this skill — depth-in / brevity-out is the right tradeoff for a first pass.
