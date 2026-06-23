# Fresh Eyes 👀

**Get fresh eyes on what you just built — without leaving the session that built it.**

A Claude Code plugin: the `/fresh-eyes` command delegates to a read-only
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

```bash
# 1. Add this repo as a plugin marketplace
/plugin marketplace add yangzhengca/claude-plugins

# 2. Install the plugin
/plugin install fresh-eyes@yangzhengca
```

## Usage

```bash
/fresh-eyes            # review the whole branch (vs. its merge-base)
/fresh-eyes staged     # review only staged changes  (git diff --staged)
/fresh-eyes changes    # review only unstaged changes (git diff)
```

The reviewer returns a structured report grouped by severity
(**Critical / Warning / Nit**), each finding citing `file:line` with a concrete
suggested fix and a verdict (`ship` / `fix-before-merge` / `needs-discussion`).
Nothing is modified — ask your main session to apply any fixes you want.

## What's in the box

| Path | What it is |
|------|------------|
| `plugins/fresh-eyes/commands/fresh-eyes.md` | The `/fresh-eyes` slash command (scope resolution + delegation) |
| `plugins/fresh-eyes/agents/fresh-eyes-reviewer.md` | The read-only reviewer subagent |

## License

MIT
