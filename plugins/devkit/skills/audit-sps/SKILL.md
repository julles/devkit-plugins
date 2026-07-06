---
name: audit-sps
description: Audit an ENTIRE existing codebase with priorities Security > Performance > Simplicity, then apply fixes only when the developer asks. Use when the user says "audit-sps", "audit the whole codebase", "scan the whole source", "full SPS audit", or wants a security/performance/simplicity review of a whole repo rather than one feature or diff. For a single feature/diff, use review-sps instead.
---

# audit-sps

Scan the **whole existing source tree** and rank what matters. Same priorities
and ruleset as `review-sps`, but repo-wide: **1. Security → 2. Performance
(non-negotiable) → 3. Simplicity**.

Required flow: **audit → present a ranked report → DO NOT apply.** Only edit
files when the developer explicitly asks ("implement", "apply #2", "apply all
security"). Apply only what they approve — never auto-apply silently.

A whole repo is too big for one pass and produces too many findings to dump
raw. The two jobs unique to this skill are **fan-out** (cover it all) and
**noise control** (surface what matters, don't drown it).

## 1. Determine scope

Get the source files, not the noise:

- Prefer `git ls-files` (respects `.gitignore`) to list tracked files.
- **Exclude**: dependencies (`node_modules`, `vendor`, `.git`), build/generated
  output (`dist`, `build`, `*.pb.go`, generated clients), lockfiles, and binary
  assets. Tests are secondary — scan them only for hardcoded secrets.
- If the developer names a subtree (e.g. `internal/`), audit only that.

Report the file/line count you're about to cover so the scope is explicit.

## 2. Read repo context

Read the target repo's `CLAUDE.md` (root and per-module) plus manifests
(`go.mod`, `package.json`, etc) to learn the language, framework, and
conventions. Use it so findings fit the repo, not a generic checklist.

## 3. Fan out the review

- **Small repo** (roughly < 30 source files): review directly here in one pass.
- **Larger repo**: split files into groups by directory/module and spawn
  parallel `Explore` (or `general-purpose`) subagents — one per group. Give each
  the SPS ruleset below and have it return findings as structured data
  (`file:line`, category, severity, problem, suggested fix). This keeps the main
  context lean. Then collect, dedup, and rank here.

Say how many groups/agents you used, and flag anything you deliberately skipped
(sampled dirs, skipped generated code) — no silent truncation.

## 4. The SPS ruleset

Check **only** what's actually in the code. Don't invent findings.

### Security (top priority)
- Input not validated at trust boundaries (user input, request params, uploads, headers)
- Injection: SQL/NoSQL/command/template — queries not parameterized
- Hardcoded secrets (API keys, passwords, tokens, connection strings)
- Broken authz/authn: endpoints without permission checks, IDOR
- Sensitive data logged or exposed in responses/errors
- Dangerous operations on untrusted input (deserialization, eval, path traversal, SSRF)

### Performance (non-negotiable)
- N+1 queries / queries inside loops
- Queries without an index on filtered/joined/sorted columns
- Loading large datasets without pagination/limit
- Repeated work that could be cached or computed once
- Blocking I/O on hot paths; avoidable O(n²)+ algorithms
- Excessive allocation/serialization on hot paths

### Simplicity (so a junior can follow it)
- Over-engineering: abstraction/interface/factory for a single implementation, config for a value that never changes
- Reinventing the standard library or an existing native feature
- Deep nesting, over-long functions, unclear naming
- Dead code, unused flags/params

## 5. Present a ranked report (DO NOT apply)

Rank ALL findings globally by category (Security → Performance → Simplicity),
then by severity (**Critical / High / Medium / Low**) within each.

**Noise control** — a whole-repo audit can surface hundreds of items:
- List every **Security** and **Performance** finding (these are the point).
- For **Simplicity**, list Medium+ individually; collapse repetitive Low items
  into counts by pattern (e.g. "12× dead code, 8× deep nesting — expand?").

Number findings globally (#1, #2, ...) so the developer can point to them.

Format per listed finding:

```
#N [SEVERITY] path/file.ext:line
    Problem:    <one sentence>
    Suggestion: <concrete fix; include a code snippet if useful>
```

Close with a summary table: count per category × severity, then ask:
*"Which do you want to apply? (e.g. 'apply #1 #3', 'apply all security', 'expand simplicity', or 'skip')"*.

If the codebase is clean against SPS, say so and stop.

## 6. Apply (only when asked)

- Apply **only** the findings the developer names (or "all" / "all <category>").
- Edit the affected files; keep the repo's style and conventions.
- After applying, report briefly per finding what changed. If a finding needs a
  larger design decision, ask first — don't guess.
- Don't touch findings they didn't approve.
