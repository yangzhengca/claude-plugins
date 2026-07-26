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
  ask it to *fix* the findings it's working blind. That session never saw your
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

/fresh-eyes:review-pr 123                                  # review a PR (yours or a teammate's)
/fresh-eyes:review-pr 123 is the cache invalidation right? # + get a question answered
/fresh-eyes:review-pr 123 --issue AI-42                    # + check coverage vs a Linear issue
/fresh-eyes:review-pr 123 --issue AI-42 is retry handled?  # both extras together — no quotes needed
```

> Plugin commands are namespaced `/<plugin>:<command>`, so the command is
> `/fresh-eyes:review` (plugin `fresh-eyes`, command `review`).

### The task brief

Before delegating, your build session distills a **task brief** from the
conversation — what you were trying to accomplish and why: the goal,
requirements, decisions you made (including options you rejected), and
constraints. The reviewer checks the diff *against the brief*, so it catches
unmet requirements and scope creep, not just bugs.

The brief carries only the **what and why — never the how**. Implementation
narrative ("I did X, it works") is deliberately excluded, because passing it
along would reimport the confirmation bias the clean context exists to remove.
Pass `--blind` (combinable with a scope, e.g. `/fresh-eyes:review staged --blind`)
to skip the brief entirely and get a pure code-only review.

The reviewer returns a structured report grouped by severity
(**Critical / Warning / Nit**), each finding citing `file:line` with a concrete
suggested fix, plus a summary carrying a brief-fulfillment assessment
(`met` / `partially met` / `not met`) and a verdict
(`ship` / `fix-before-merge` / `needs-discussion`).
Nothing is modified — ask your main session to apply any fixes you want.

### Reviewing a PR

`/fresh-eyes:review-pr <pr-number>` reviews a GitHub pull request — yours or a
teammate's — instead of your local diff. The PR is fetched with `gh` and checked out **in an isolated
git worktree** — your own checkout is never touched — then handed to the same
reviewer subagent, with the PR's title and body as the task brief (treated as
author-written claims to verify, not trust). The worktree is cleaned up
afterwards, and nothing is ever posted to GitHub unless you ask.

Two optional extras:

- **A question or concern** — append it as free text
  (`/fresh-eyes:review-pr 123 not sure the retry logic handles timeouts`).
  The reviewer investigates it in the code and answers it explicitly in a
  dedicated section of the report, alongside the regular review.
- **A Linear issue** — pass `--issue <id>` and the issue's requirements are
  fetched and given to the reviewer, which then rates every requirement
  `met` / `partial` / `missing` with `file:line` evidence, so coverage gaps
  surface even when the code that *does* exist is correct.

## What's in the box

| Path | What it is |
|------|------------|
| `plugins/fresh-eyes/commands/review.md` | The `/fresh-eyes:review` slash command (scope resolution + delegation) |
| `plugins/fresh-eyes/commands/review-pr.md` | The `/fresh-eyes:review-pr` slash command (PR fetch, worktree isolation, Linear requirements + delegation) |
| `plugins/fresh-eyes/agents/fresh-eyes-reviewer.md` | The read-only reviewer subagent shared by both commands |

---

<img width="1254" height="1254" alt="ChatGPT Image Jul 25, 2026, 09_50_02 AM" src="https://github.com/user-attachments/assets/e55381ba-63bd-4881-8179-b8e0e317ecca" />

# Lazy Bones 🦴

**The best code is the code never written.**

A second plugin in this marketplace: a YAGNI enforcer adapted from
[ponytail](https://github.com/DietrichGebert/ponytail) (MIT) — "the laziest
senior dev in the room". Lazy means efficient, not careless: it shrinks the
*solution*, never the reading, and it is never lazy about input validation,
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

| Path | What it is |
|------|------------|
| `plugins/lazy-bones/commands/yagni.md` | The `/lazy-bones:yagni` slash command (the session-wide lazy mode + intensity levels) |
| `plugins/lazy-bones/commands/review.md` | The `/lazy-bones:review` slash command (over-engineering diff review — local scopes or `--pr <number>`) |
| `plugins/lazy-bones/commands/audit.md` | The `/lazy-bones:audit` slash command (repo-wide over-engineering report) |

## Credits

Lazy Bones is adapted from
[DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail),
MIT-licensed. The decision ladder, tag taxonomy, and safety boundaries are
ponytail's; this plugin repackages the core as Claude Code slash commands.

---

# License

Both plugins are MIT-licensed — see [LICENSE](LICENSE), which also carries the
copyright notice for the portions adapted from ponytail.
