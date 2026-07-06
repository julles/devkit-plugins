# devkit — Documentation

`devkit` is a Claude Code plugin that packages a backend developer's review and
scaffolding workflow into a set of skills. It is opinionated: every skill shares
one set of house rules so the toolkit behaves consistently whether it is
reviewing, auditing, or generating code.

## Skills

| Skill | Command | Purpose | Docs |
| ----- | ------- | ------- | ---- |
| review-sps | `/devkit:review-sps` | Review a single feature or change | [review-sps.md](review-sps.md) |
| audit-sps | `/devkit:audit-sps` | Audit an entire codebase | [audit-sps.md](audit-sps.md) |
| pay-check | `/devkit:pay-check` | Payment-domain review of money-handling code | [pay-check.md](pay-check.md) |

## Shared model

Every skill is built on the same three principles.

### 1. Priority order: Security → Performance → Simplicity

Findings and generated code are ordered by a fixed priority:

1. **Security** — the highest priority. Anything exploitable is treated as
   blocking.
2. **Performance** — non-negotiable. Payment and other transactional backends
   fail in production on N+1 queries, missing indexes, and unbounded result sets
   long before they fail on anything else.
3. **Simplicity** — the code must be understandable by a junior engineer.
   "Simplicity" here means the absence of accidental complexity
   (over-engineering, speculative abstraction), not the absence of necessary
   logic.

### 2. Review first, apply on consent

The review skills never modify your code as a side effect. They present a
numbered list of findings and stop. You then decide what to apply:

```
apply #1 #3
apply all security
skip
```

Only the findings you name are applied.

### 3. Severity: Critical / High / Medium / Low

Every finding carries a severity so you can triage:

| Severity | Meaning |
| -------- | ------- |
| Critical | Exploitable security hole, guaranteed data/money loss, or double-charge. Fix before shipping. |
| High | Serious performance or correctness problem under realistic load. |
| Medium | Real issue with limited blast radius, or a maintainability problem. |
| Low | Minor simplicity or style issue. |

## Context awareness

Before reviewing or generating, the skills read the target repository's
`CLAUDE.md` and manifests (`go.mod`, `package.json`, …) to learn the stack,
framework, conventions, and — for payment work — the provider(s) in use
(Stripe, Midtrans, Xendit, etc). Suggestions and generated code follow the
repository's own conventions rather than a generic template.

## Installation

```
/plugin marketplace add julles/devkit-plugins
/plugin install devkit@devkit-plugins
```

See the top-level [README](../README.md) for local development and testing.
