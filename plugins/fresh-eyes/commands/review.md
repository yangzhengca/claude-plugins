---
description: Get fresh eyes on what you just built — review changes (staged, unstaged, or whole branch) via the read-only fresh-eyes-reviewer subagent, fact-check its findings against the diff with a second narrow subagent, then fix findings in this session. Distills the session's goal into a task brief for the reviewer; enforces the repo's .fresh-eyes/review-rules.md if present. Pass --blind to skip the brief, --no-check to skip the fact-check pass.
argument-hint: "[staged|changes] [--blind] [--no-check] — omit scope to review the whole branch"
---

You are orchestrating a code review. The user invoked this command with the
optional arguments: "$ARGUMENTS"

## Step 1 — Parse the arguments

Split "$ARGUMENTS" on whitespace. Extract:

- **blind mode**: present if any token is `--blind`. Remove it. When set,
  skip Step 2 entirely — no task brief is produced or passed.
- **no-check mode**: present if any token is `--no-check`. Remove it. When
  set, skip Step 5 (the fact-check pass) and relay the reviewer's report
  directly.
- **scope keyword**: the single remaining token, if any, after both flags are
  removed. If MORE than one token remains, do not guess — stop and ask which
  scope was meant.

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

## Step 3 — Load the project's review rules (if any)

Look for `.fresh-eyes/review-rules.md` at the repository root
(`git rev-parse --show-toplevel`). If it exists, Read it: it is the repo's own
review checklist — conventions, invariants, and "if X changes, Y must change
too" contracts the maintainers want every review to enforce. It will be passed
to the reviewer verbatim under a `## Project rules` heading. If the file does
not exist, skip this step silently; do not create it and do not invent rules.

## Step 4 — Confirm scope, then delegate

First print one line confirming the resolved scope and the exact command, e.g.
`Reviewing: staged changes — git diff --staged`, noting whether a task brief is
included (`— with task brief`, `— blind`, or `— no brief (no task context)`),
`— with project rules` when Step 3 found a file, and `— no fact-check` when
`--no-check` was given.

Then invoke the **fresh-eyes-reviewer** subagent. In the delegation prompt, state explicitly:
- which scope was selected (staged / unstaged / branch),
- the EXACT git command it must run to obtain the diff,
- the task brief from Step 2 under a heading `## Task brief` (omit the section
  entirely in blind mode or when there is no brief),
- the project rules from Step 3 under a heading `## Project rules` (omit the
  section entirely when there is no rules file),
- that it must read surrounding file context as needed and end with its
  structured findings report — numbered findings, each with a verbatim quote of
  the added code, plus the coverage line.

Do NOT review the code yourself — delegate to fresh-eyes-reviewer and wait for its
report. The point of this command is that the reviewer works in its OWN clean
context window, with no memory of how the code was built, so its review is
unbiased. You keep the build context, so you can act on the findings accurately.

## Step 5 — Fact-check the findings (skip if --no-check)

If the reviewer reported no findings, skip this step.

Otherwise invoke the **fresh-eyes-fact-checker** subagent. It sees only the
diff and removes a finding only when the diff proves it wrong. In the
delegation prompt give it:

- the EXACT diff command from Step 1 (the same one the reviewer ran),
- the numbered findings list: for each finding, its id (`F<n>`), path,
  `file:line`, the verbatim quoted code, and the claim text.
  Do NOT include severity or category — they are withheld on purpose so the
  checker cannot filter by value.

Wait for its verdict (`APPROVE ALL`, or `REMOVE: ...` with a refuting diff line
per removed id).

## Step 6 — Relay the findings

Present the reviewer's report to me as-is, with one adjustment when the
fact-checker removed anything: take the removed findings out of the Findings
section and append a short section:

### Fact-check
- `<n>` findings checked against the diff, `<m>` removed.
- **F<n>** — removed: <the checker's ground and refuting diff line, quoted>.

Never drop a finding silently. If the reviewer's verdict rested solely on
findings that were removed, say so in one line under the verdict rather than
changing the reviewer's words. If `--no-check` was given, or there were no
findings, omit the Fact-check section entirely.

Do not modify any files unless I explicitly ask you to afterward. Because you
still hold the full implementation context from this session, you can fix any
findings I choose to address without the drift you'd get from a context-blind
session. The verbatim quote on each finding is there so you can locate it
exactly — grep for the quote rather than trusting the line number alone.
