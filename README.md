<img width="1254" height="1254" alt="fresheyes" src="https://github.com/user-attachments/assets/4a1333bd-2df2-4ae2-b2dd-2391bdb6f1fc" />

# Fresh Eyes 👀

**Get fresh eyes on what you just built — without leaving the session that built it.**

A Claude Code plugin: the `/fresh-eyes:review` command delegates to a read-only
`fresh-eyes-reviewer` subagent that reviews your diff in its **own clean context
window**. The reviewer has no memory of how the code was written, so its review
carries no confirmation bias. Only its report comes back to your main session —
where you still hold every instruction and decision — so you can fix the findings
without drift.

## The problem it solves

How do you review code you just built with Claude Code? The two obvious ways each
have a flaw:

- **Review in the same session** — the AI just wrote this code with your
  implementation instructions fresh in context. Ask it to review its own work and
  it's biased: it already "decided" the changes were reasonable. You get a rubber
  stamp, not a review.
- **Review in a separate, clean session** — now it's objective, but the moment you
  ask it to _fix_ the findings it's working blind. That session never saw your
  instructions or the decisions behind the code, so the fixes drift from what you
  actually asked for.

So you're stuck: biased review, or context-blind fixes.

**The command → subagent pattern gives you both.** The subagent reviews in a clean
context (unbiased), but it's launched from your build session and only its report
returns — so you fix the findings right where you wrote the code, with the AI that
still knows everything you asked for.

> Unbiased review. Context-aware fixes. One session.

## Install

### 1. Add this repo as a plugin marketplace

```bash
/plugin marketplace add yangzhengca/claude-plugins
```

### 2. Install the plugin

```bash
/plugin install fresh-eyes@yangzhengca
```

## Usage

```bash
/fresh-eyes:review            # review the whole branch (vs. its merge-base)
/fresh-eyes:review staged     # review only staged changes  (git diff --staged)
/fresh-eyes:review changes    # review only unstaged changes (git diff)
/fresh-eyes:review --blind    # skip the task brief — pure code-only review
/fresh-eyes:review --no-check # skip the fact-check pass — reviewer's report as-is

/fresh-eyes:review-pr 123                                  # review a PR (yours or a teammate's)
/fresh-eyes:review-pr 123 is the cache invalidation right? # + get a question answered
/fresh-eyes:review-pr 123 --issue AI-42                    # + check coverage vs a Linear issue
/fresh-eyes:review-pr 123 --issue AI-42 is retry handled?  # both extras together — no quotes needed
/fresh-eyes:review-pr 123 --no-check                       # skip the fact-check pass
```

> Plugin commands are namespaced `/<plugin>:<command>`, so the command is
> `/fresh-eyes:review` (plugin `fresh-eyes`, command `review`).

### The task brief

Before delegating, your build session distills a **task brief** from the
conversation — what you were trying to accomplish and why: the goal,
requirements, decisions you made (including options you rejected), and
constraints. The reviewer checks the diff _against the brief_, so it catches
unmet requirements and scope creep, not just bugs.

The brief carries only the **what and why — never the how**. Implementation
narrative ("I did X, it works") is deliberately excluded, because passing it
along would reimport the confirmation bias the clean context exists to remove.
Pass `--blind` (combinable with a scope, e.g. `/fresh-eyes:review staged --blind`)
to skip the brief entirely and get a pure code-only review.

### The report

The reviewer returns a structured report grouped by severity
(**Critical / Warning / Nit**). Every finding is numbered (`F1`, `F2`, …),
tagged with a category (`bug` / `security` / `performance` / `maintainability` /
`test` / `style` / `docs`), and anchored two ways: `file:line` **plus a verbatim
quote of the added lines it targets**, unique within the diff — so a
hallucinated line number can't send you to the wrong place, and your main
session can grep straight to it. Each finding ends with a concrete suggested fix.

The summary carries a **coverage line** (every changed file ends the review as
`reviewed` or `skipped: <reason>` — a header, config, or docs counterpart is
never silently omitted), a brief-fulfillment assessment
(`met` / `partially met` / `not met`) and a verdict
(`ship` / `fix-before-merge` / `needs-discussion`). PR reviews add a
**PR Conclusions** section with three direct verdicts, each backed by evidence
from the diff: coding standards, patterns, and best practices
(`follows` / `does not follow` / `cannot determine`); necessity and project
value (`yes` / `no` / `cannot determine`); and edge cases and error handling
(`handled` / `not handled` / `cannot determine`).
Nothing is modified — ask your main session to apply any fixes you want.

### How the reviewer works

The reviewer is tuned for **precision over recall**: a false positive costs
trust in every other finding. Concretely, it

- reads the change's shape first (`--stat`, `--name-status`) and builds a
  coverage checklist before opening a single hunk;
- writes a short risk plan on large diffs, works it, then does a second
  **plan-free pass** so the plan never becomes a coverage ceiling;
- establishes non-local claims (concurrency, attacker control, ownership,
  callers) by reading call sites, never from a name or an import;
- doesn't report what your compiler, type checker, linter, or formatter
  already catches;
- files findings only against added lines — deleted code is context, unchanged
  code and files outside the diff are evidence, not targets;
- looks for cross-file breakage: an interface changed without its
  implementations, a config key without its reader, one locale file updated
  and not the others;
- skips generated and vendored files (lockfiles, snapshots, protobuf output,
  `vendor/`, `dist/`) but keeps tests in scope, and **never opens
  credential-shaped files** (`.env`, `id_rsa`, `.netrc`, `.npmrc`, …) even for
  context — a committed one is reported as Critical by path only.

### The fact-check pass

After the reviewer returns, a second, deliberately narrow subagent
(`fresh-eyes-fact-checker`) sees **only the diff** and the numbered findings —
with severity withheld so it can't filter by value — and removes a finding only
when the diff _proves_ it wrong: the code it describes isn't in that file's
diff, or a specific diff line literally contradicts its central claim. Anything
unverifiable, low-value, or merely disputed is approved; findings about memory
safety, concurrency, declaration consistency, behaviour changes, unused
parameters, or committed secrets are **never** removed. Nothing is dropped
silently — removed findings are listed in a `Fact-check` section with the
refuting diff line. Pass `--no-check` to skip the pass.

### Project rules

Drop a `.fresh-eyes/review-rules.md` at your repo root and both commands pass it
to the reviewer verbatim as a checklist to enforce: naming conventions, field
orders, "if this file changes, those docs must change too", required regression
tests — the kind of project knowledge a fresh-context reviewer can't infer from
a diff. Rules that don't apply to a changed file are ignored; violations become
findings (Warning unless the rule says otherwise). `review-pr` reads the file
from **your** checkout, never from the PR's, so a PR can't rewrite its own
review rules. This repo's own [`.fresh-eyes/review-rules.md`](.fresh-eyes/review-rules.md)
is a small example.

### Reviewing a PR

`/fresh-eyes:review-pr <pr-number>` reviews a GitHub pull request — yours or a
teammate's — instead of your local diff. The PR is fetched with `gh` and checked out **in an isolated
git worktree** — your own checkout is never touched — then handed to the same
reviewer subagent, with the PR's title and body as the task brief (treated as
author-written claims to verify, not trust). The same fact-check pass and
project rules apply. The worktree is cleaned up afterwards, and nothing is ever
posted to GitHub without your explicit approval, even in Auto mode. The PR
report concludes whether the change follows project standards and patterns,
is necessary and valuable, and handles edge cases and errors. If you later
request PR comments, you choose which findings to post and approve their exact
text and locations; inline comments are the default.

Two optional extras:

- **A question or concern** — append it as free text
  (`/fresh-eyes:review-pr 123 not sure the retry logic handles timeouts`).
  The reviewer investigates it in the code and answers it explicitly in a
  dedicated section of the report, alongside the regular review.
- **A Linear issue** — pass `--issue <id>` and the issue's requirements are
  fetched and given to the reviewer, which then rates every requirement
  `met` / `partial` / `missing` with `file:line` evidence, so coverage gaps
  surface even when the code that _does_ exist is correct.

## What's in the box

| Path                                                   | What it is                                                                                                 |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `plugins/fresh-eyes/commands/review.md`                | The `/fresh-eyes:review` slash command (scope resolution + delegation)                                     |
| `plugins/fresh-eyes/commands/review-pr.md`             | The `/fresh-eyes:review-pr` slash command (PR fetch, worktree isolation, Linear requirements + delegation) |
| `plugins/fresh-eyes/agents/fresh-eyes-reviewer.md`     | The read-only reviewer subagent shared by both commands                                                    |
| `plugins/fresh-eyes/agents/fresh-eyes-fact-checker.md` | The diff-only fact-checker subagent that runs after the reviewer (skip with `--no-check`)                  |
| `.fresh-eyes/review-rules.md`                          | This repo's own project rules — an example of the file both commands pick up                               |

### Credits

The fact-check pass, quote-anchored findings, coverage checklist, and
precision-over-recall bar are adapted from techniques in
[alibaba/open-code-review](https://github.com/alibaba/open-code-review)
(Apache-2.0); the fact-checker prompt in particular is a close adaptation of its
review-filter prompt.

---

<img width="1254" height="1254" alt="ChatGPT Image Jul 25, 2026, 09_50_02 AM" src="https://github.com/user-attachments/assets/e55381ba-63bd-4881-8179-b8e0e317ecca" />

# Lazy Bones 🦴

**The best code is the code never written.**

A second plugin in this marketplace: a YAGNI enforcer adapted from
[ponytail](https://github.com/DietrichGebert/ponytail) (MIT) — "the laziest
senior dev in the room". Lazy means efficient, not careless: it shrinks the
_solution_, never the reading, and it is never lazy about input validation,
error handling, security, or accessibility.

## Install

```bash
/plugin marketplace add yangzhengca/claude-plugins   # if not already added
/plugin install lazy-bones@yangzhengca
```

## Usage

```bash
/lazy-bones:yagni            # lazy senior dev mode, intensity: full (default)
/lazy-bones:yagni lite       # build what's asked, but name the lazier alternative
/lazy-bones:yagni ultra      # YAGNI extremist — challenge the requirement itself
/lazy-bones:yagni off        # back to normal

/lazy-bones:review           # over-engineering review of the whole branch (vs. merge-base)
/lazy-bones:review staged    # review only staged changes  (git diff --staged)
/lazy-bones:review changes   # review only unstaged changes (git diff)
/lazy-bones:review --pr 123  # review a GitHub PR's diff for YAGNI (via gh pr diff)

/lazy-bones:audit            # scan the whole repo for over-engineering
/lazy-bones:audit src/       # scan just one directory
```

### `/lazy-bones:yagni` — the mode

Switches the session into lazy senior dev mode until turned off. Before any
code is written, a decision ladder runs — stop at the first rung that holds:

1. Does this need to exist at all? (YAGNI)
2. Already in this codebase? Reuse it.
3. Stdlib does it? Use it.
4. Native platform feature covers it? Use it.
5. Already-installed dependency solves it? Use it.
6. Can it be one line? One line.
7. Only then: the minimum code that works.

No unrequested abstractions, no scaffolding "for later", deletion over
addition, shortest working diff wins. Deliberate corner-cuts are marked with
a `lazy-bones:` comment naming the ceiling and the upgrade path.

### `/lazy-bones:review` — the diff review

Reviews a diff for over-engineering **only** — the complement of a
correctness review (that's `/fresh-eyes:review`). Same scopes as fresh-eyes
(`staged`, `changes`, or omit for the whole branch), plus `--pr <number>` to
review a GitHub PR's diff fetched with `gh pr diff` — no checkout, no
worktree. Findings come back one tagged line each
(`delete:` / `stdlib:` / `native:` / `yagni:` / `shrink:`) with location,
what to cut, and what replaces it, ending with `net: -N lines possible.` —
or `Lean already. Ship.` Report only; nothing is modified or posted.

### `/lazy-bones:audit` — the bloat scan

One-shot, repo-wide (or path-scoped) hunt for over-engineering: hand-rolled
stdlib, dependencies the platform already covers, single-implementation
abstractions, dead flags, wrappers that only delegate. Findings come back one
line each, ranked biggest cut first and tagged
`delete:` / `stdlib:` / `native:` / `yagni:` / `shrink:`, ending with
`net: -N lines, -M deps possible.` — or `Lean already. Ship.` Report only;
nothing is modified.

## What's in the box

| Path                                    | What it is                                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `plugins/lazy-bones/commands/yagni.md`  | The `/lazy-bones:yagni` slash command (the session-wide lazy mode + intensity levels)                   |
| `plugins/lazy-bones/commands/review.md` | The `/lazy-bones:review` slash command (over-engineering diff review — local scopes or `--pr <number>`) |
| `plugins/lazy-bones/commands/audit.md`  | The `/lazy-bones:audit` slash command (repo-wide over-engineering report)                               |

## Credits

Lazy Bones is adapted from
[DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail),
MIT-licensed. The decision ladder, tag taxonomy, and safety boundaries are
ponytail's; this plugin repackages the core as Claude Code slash commands.

---

# License

Both plugins are MIT-licensed — see [LICENSE](LICENSE), which also carries the
copyright notice for the portions adapted from ponytail.
