---
description: Get fresh eyes on what you just built — review changes (staged, unstaged, or whole branch) via the read-only fresh-eyes-reviewer subagent, then fix findings in this session
argument-hint: "[staged|changes] — omit to review the whole branch"
---

You are orchestrating a code review. The user invoked this command with the
optional argument: "$1"

## Step 1 — Resolve the review scope

Normalize "$1" (trim whitespace, case-insensitive) to one of three scopes:

- **staged** ← `staged`, `cached`, `index`
  Diff command: `git diff --staged`

- **unstaged** ← `changes`, `unstaged`, `working`, `wt`
  Diff command: `git diff`

- **branch** ← empty (no argument), `branch`, `all`
  Detect the base branch by trying these in order, using the first that works:
  1. Upstream of the current branch: `git rev-parse --abbrev-ref --symbolic-full-name @{u}`
  2. The repo default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (e.g. `origin/main`)
  3. Fall back to `main`, then `master`.
  Diff command: `git diff <base>...HEAD`  (three-dot / merge-base form — shows only what this branch introduced).

If "$1" is non-empty but matches none of the keywords above, do NOT guess and do
NOT silently default to a branch review. Stop and ask which scope was meant,
listing the valid options (`staged` / `changes`, or omit the argument for a full
branch review).

## Step 2 — Confirm scope, then delegate

First print one line confirming the resolved scope and the exact command, e.g.
`Reviewing: staged changes — git diff --staged`.

Then invoke the **fresh-eyes-reviewer** subagent. In the delegation prompt, state explicitly:
- which scope was selected (staged / unstaged / branch),
- the EXACT git command it must run to obtain the diff,
- that it must read surrounding file context as needed and end with a structured findings report.

Do NOT review the code yourself — delegate to fresh-eyes-reviewer and wait for its
report. The point of this command is that the reviewer works in its OWN clean
context window, with no memory of how the code was built, so its review is
unbiased. You keep the build context, so you can act on the findings accurately.

## Step 3 — Relay the findings

When the subagent returns, present its findings report to me as-is. Do not modify
any files unless I explicitly ask you to afterward. Because you still hold the full
implementation context from this session, you can fix any findings I choose to
address without the drift you'd get from a context-blind session.
