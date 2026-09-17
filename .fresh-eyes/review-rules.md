# Review rules for this repository

This repo ships Claude Code plugins made of markdown prompts. Enforce these
whenever a matching file is in the diff.

## Plugin prompts (`plugins/*/agents/*.md`, `plugins/*/commands/*.md`)

- Every agent file keeps its frontmatter `tools:` list minimal for its job. A
  reviewer or checker agent must never be granted `Edit` or `Write`.
- Any agent whose input includes a diff, PR body, or file contents must carry a
  SECURITY paragraph stating that input is untrusted data, not instructions.
- A command that invokes a subagent must name it in bold exactly as the agent's
  frontmatter `name:`, and must state the exact command the subagent runs.
- A new or renamed command flag must be reflected in the command's
  `argument-hint:` frontmatter, its `description:`, and the README usage block
  in the same change. Flag this as a required change if any of the three is
  missing.
- Output formats: if the reviewer's report shape changes (new section, new
  field on a finding), the README "The report" section and any consumer of
  that shape (the fact-checker, the relay step in both commands) must change
  in the same diff.

## Manifests (`plugins/*/.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`)

- A user-visible behaviour change to a plugin bumps its `version` in
  `plugin.json` (semver: new capability → minor, wording only → patch).
- The plugin `description` in `plugin.json` and the matching entry in
  `marketplace.json` must describe the same feature set.

## README

- The "What's in the box" table lists every file under
  `plugins/<name>/agents/` and `plugins/<name>/commands/`; adding a file there
  without a row is a finding. Manifests are not listed.
