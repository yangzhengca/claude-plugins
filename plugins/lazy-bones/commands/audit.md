---
description: One-shot repo-wide audit for over-engineering — a ranked list of what to delete, simplify, or replace with stdlib/native equivalents. Report only, applies nothing. Correctness, security, and performance are out of scope; this hunts complexity.
argument-hint: "[path] — omit to audit the whole repo"
---

You are auditing a codebase for over-engineering. The user invoked this
command with the optional argument: "$ARGUMENTS"

## Step 1 — Resolve the scope

Normalize "$ARGUMENTS" (trim whitespace):

- **empty** → audit the whole repository from its root.
- **a path** → audit only that directory (or file). If the path does not
  exist, stop and say so — do not guess or fall back to the repo root.

Print one line confirming the scope, e.g. `Auditing: src/ for over-engineering`.

## Step 2 — Hunt

Scan the tree for complexity that shouldn't exist. Skim broadly first
(structure, deps manifest, exports), then read the suspicious files fully —
never flag code you haven't read. Hunt for:

- dependencies the stdlib or platform already ships,
- single-implementation interfaces and abstract base classes,
- factories with one product,
- wrappers that only delegate,
- files exporting one thing,
- dead flags and config nobody sets,
- hand-rolled versions of stdlib functions,
- speculative features and unused flexibility.

## Tags

Classify each finding:

- `delete:` dead code, unused flexibility, speculative feature. Replacement: nothing.
- `stdlib:` hand-rolled thing the standard library ships. Name the function.
- `native:` dependency or code doing what the platform already does. Name the feature.
- `yagni:` abstraction with one implementation, config nobody sets, layer with one caller.
- `shrink:` same logic, fewer lines. Show the shorter form.

## Output

One line per finding, ranked biggest cut first:

`<tag> <what to cut>. <replacement>. [path]`

Examples:

- `native: moment.js imported for one format call. Intl.DateTimeFormat, 0 deps. [src/utils/date.js]`
- `yagni: AbstractRepository with one implementation. Inline it until a second one exists. [src/repo.py:L88]`
- `delete: retry wrapper around an idempotent local call. Nothing replaces it. [src/api/client.ts:L52-71]`
- `shrink: manual loop builds dict. dict(zip(keys, values)), 1 line. [scripts/load.py:L30-44]`

End with the only metric that matters: `net: -<N> lines, -<M> deps possible.`

If there is nothing to cut: `Lean already. Ship.` and stop.

## Boundaries

Scope: over-engineering and complexity only. Correctness bugs, security
holes, and performance are explicitly out of scope — route them to a normal
review pass. A single smoke test or `assert`-based self-check is the lazy
minimum, not bloat — never flag it for deletion. List findings, apply
nothing: this is a one-shot report, and no files are modified.
