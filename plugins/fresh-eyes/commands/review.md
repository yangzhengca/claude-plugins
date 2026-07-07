---
description: Get fresh eyes on what you just built — review changes (staged, unstaged, or whole branch) via the read-only fresh-eyes-reviewer subagent, then fix findings in this session. Distills the session's goal into a task brief for the reviewer; pass --blind to skip it.
argument-hint: "[staged|changes] [--blind] — omit scope to review the whole branch"
---

You are orchestrating a code review. The user invoked this command with the
optional arguments: "$ARGUMENTS"

## Step 1 — Parse the arguments

Split "$ARGUMENTS" on whitespace. Extract:

- **blind mode**: present if any token is `--blind`. When set, skip Step 2
  entirely — no task brief is produced or passed.
- **scope keyword**: the remaining token, if any. If MORE than one token
  remains, do not guess — stop and ask which scope was meant.

Normalize the scope keyword (trim whitespace, case-insensitive) to one of three
scopes:

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

If the scope keyword is non-empty but matches none of the keywords above, do NOT
guess and do NOT silently default to a branch review. Stop and ask which scope was
meant, listing the valid options (`staged` / `changes`, or omit the argument for a
full branch review).

## Step 2 — Distill the task brief (skip if --blind)

From this session's conversation, distill a short **task brief**: what the user
was trying to accomplish and why. Include:

- the goal of the task, in one or two sentences,
- concrete requirements and acceptance criteria that were stated or agreed,
- decisions the user made along the way — including options they explicitly
  rejected or reversed,
- constraints (compatibility, performance, style, scope limits).

STRICTLY the *what* and *why* — NEVER the *how*. Do not describe how the code was
implemented, which files were touched, what approach you took, or assert that
anything works or was tested. Passing implementation narrative to the reviewer
reimports the confirmation bias this command exists to eliminate.

If this session contains no meaningful task context (e.g. the command was run in
a fresh session), produce no brief and proceed as a plain fresh-eyes review — do
not invent one.

## Step 3 — Confirm scope, then delegate

First print one line confirming the resolved scope and the exact command, e.g.
`Reviewing: staged changes — git diff --staged`, noting whether a task brief is
included (`— with task brief`, `— blind`, or `— no brief (no task context)`).

Then invoke the **fresh-eyes-reviewer** subagent. In the delegation prompt, state explicitly:
- which scope was selected (staged / unstaged / branch),
- the EXACT git command it must run to obtain the diff,
- the task brief from Step 2 under a heading `## Task brief` (omit the section
  entirely in blind mode or when there is no brief),
- that it must read surrounding file context as needed and end with a structured findings report.

Do NOT review the code yourself — delegate to fresh-eyes-reviewer and wait for its
report. The point of this command is that the reviewer works in its OWN clean
context window, with no memory of how the code was built, so its review is
unbiased. You keep the build context, so you can act on the findings accurately.

## Step 4 — Relay the findings

When the subagent returns, present its findings report to me as-is. Do not modify
any files unless I explicitly ask you to afterward. Because you still hold the full
implementation context from this session, you can fix any findings I choose to
address without the drift you'd get from a context-blind session.
