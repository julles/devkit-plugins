# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **plugin marketplace**, not an application. There is no build,
test runner, or lint step. It ships one plugin, `devkit`, whose "code" is
Markdown skill definitions (`SKILL.md`) plus JSON manifests. The only tooling is
the validator.

## Validate (run after any manifest or skill change)

```
claude plugin validate .                 # marketplace catalog (.claude-plugin/marketplace.json)
claude plugin validate ./plugins/devkit  # plugin manifest (plugins/devkit/.claude-plugin/plugin.json)
```

Test skills live without installing: `claude --plugin-dir ./plugins/devkit`, then
`/reload-plugins` to pick up edits.

## Structure and the manifest chain

Three files must agree, or install/validation breaks:

- `.claude-plugin/marketplace.json` — catalog. Its `plugins[].source` is a
  relative path (`"./plugins/devkit"`), **not** the `pluginRoot`-shortened form
  (the validator rejects a bare `"devkit"`).
- `plugins/devkit/.claude-plugin/plugin.json` — the plugin. `name` (`devkit`) is
  the skill namespace: a skill folder `review-sps/` becomes `/devkit:review-sps`.
- `plugins/devkit/skills/<name>/SKILL.md` — one skill per folder. The folder name
  is the invocation name.

`.claude-plugin/` holds only manifests. All other dirs (`skills/`) sit at the
plugin root, never inside `.claude-plugin/`.

Bump `version` in `plugin.json` on every release — installed users only get
updates when it changes.

## Skill conventions (shared across all SKILL.md)

Every skill in `devkit` encodes the same house rules — keep new skills
consistent with them:

- **Priority order is fixed and identical everywhere:** Security > Performance
  (non-negotiable) > Simplicity. Simplicity means a junior can follow the code —
  flag over-engineering, not clever-but-clear code.
- **Review-first, apply-on-consent:** present numbered findings and stop; only
  edit files when the developer explicitly approves specific ones. Never
  auto-apply.
- **Severity:** Critical / High / Medium / Low.
- **Read the target repo's `CLAUDE.md`** for language/convention context before
  reviewing, so findings fit the repo instead of being generic.

Current skills: `review-sps` (one feature/diff), `audit-sps` (whole-codebase
scan — fans out to subagents per module, ranks globally, collapses repetitive
low-severity noise), `pay-check` (payment-domain review — idempotency, money
representation, atomic balance updates, webhook verification; layered on top of
the generic SPS review), and `codegen` (spec-driven scaffolding — generates a
vertical slice from a lightweight Markdown spec by mirroring an existing repo
slice; migration included, no tests, no OpenAPI). `codegen` runs end-to-end;
the same flow is also split into step commands `codegen-explore`,
`codegen-create-spec`, `codegen-apply`, `codegen-archive`, which hand off state
through files in `specs/` (`<feature>.explore.md` → `<feature>.md` →
`specs/archive/<date>-<feature>.md`). When adding a skill, restate the shared
ruleset in its `SKILL.md` — skills are loaded independently and must be
self-contained.
