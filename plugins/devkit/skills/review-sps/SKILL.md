---
name: review-sps
description: Review a specific feature or change with priorities Security > Performance > Simplicity, then apply fixes only when the developer asks. Use when the user says "review-sps", "review feature X", "review with SPS rules", or asks for a security/performance/simplicity-focused code review of a diff, files, or a named feature.
---

# review-sps

Review the code for one specific feature/change. Priorities, in **strict order**:
**1. Security → 2. Performance (non-negotiable) → 3. Simplicity**.

Required flow: **review first → present findings + suggestions → DO NOT apply.**
Only edit the affected files when the developer explicitly asks ("implement",
"apply #2", "apply all security"). Apply only what they approve — never
auto-apply silently.

## Output language

Write findings in **English by default**. If the developer asks for Indonesian
(e.g. "bahasa Indonesia", "in Indonesian", or an `id` / `--lang id` argument),
write the Problem, Suggestion, and summary text in Indonesian. Always keep code,
identifiers, file paths, severity labels, and command names unchanged.

## 1. Determine scope (from arguments)

Auto-detect what the developer passed:

- **Empty / "diff"** → `git diff` (unstaged + staged). If clean, try
  `git diff <base>...HEAD` against the main branch (`main`/`master`).
- **File/folder path** → review the contents of that path.
- **Feature name** (e.g. "login flow") → find related files by grepping/globbing
  that name; if ambiguous, confirm which files are in scope before proceeding.

If the scope is large, delegate the reading to the `Explore` agent to keep the
main context lean — but keep the review and apply steps here.

## 2. Read repo context

Read the target repo's `CLAUDE.md` (root, and the nearest one to the reviewed
files) to learn the language, conventions, and local rules. If there is none,
infer the language/framework from the files and manifests (`go.mod`,
`package.json`, etc). Use this context so suggestions fit the repo's style
instead of being generic.

## 3. Review against the SPS ruleset

Check **only** what is actually present in the reviewed code. Do not invent
problems to fill a report. If it is clean, say it is clean.

### Security (top priority)
- Input not validated at trust boundaries (user input, request params, uploads, headers)
- Injection: SQL/NoSQL/command/template — queries not parameterized
- Hardcoded secrets (API keys, passwords, tokens, connection strings)
- Broken authz/authn: endpoints without permission checks, IDOR (accessing another user's object)
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
- Over-engineering: an abstraction/interface/factory for a single implementation, config for a value that never changes
- Reinventing the standard library or an existing native feature
- Deep nesting, over-long functions, unclear naming
- Dead code, unused flags/params

## 4. Present findings (DO NOT apply)

Group by category in the order **Security → Performance → Simplicity**. Within a
category, sort by highest severity first. Give each finding a global number (#1,
#2, ...) so the developer can point to them easily.

Format per finding:

```
#N [SEVERITY] path/file.ext:line
    Problem:    <one sentence>
    Suggestion: <concrete fix; include a code snippet if useful>
```

Severity: **Critical / High / Medium / Low**. Exploitable security = Critical/High.
Minor simplicity = Low.

Close with a one-line summary: count per category + the highest severity, then
ask: *"Which ones do you want to apply? (e.g. 'apply #1 #3', 'apply all security', or 'skip')"*.

If there are no findings at all: say the code is clean against SPS and stop.

## 5. Apply (only when asked)

When the developer asks to implement:
- Apply **only** the findings they name (or "all" / "all <category>").
- Edit the affected files; keep the repo's style and conventions.
- After applying, report briefly per finding what changed. If a finding needs a
  larger design decision, ask first — do not guess.
- Do not touch findings they did not approve.
