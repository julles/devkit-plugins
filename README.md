# devkit-plugins

A Claude Code plugin marketplace with **devkit** — a developer toolkit.

Current skills:

| Skill | Command | What it does |
| ----- | ------- | ------------ |
| review-sps | `/devkit:review-sps` | Reviews a **single feature/change** (diff, files, or named feature) with priorities **Security > Performance > Simplicity**, then applies fixes only when you approve them. |
| audit-sps | `/devkit:audit-sps` | Scans the **whole existing codebase** against the same SPS ruleset — fans out across modules, ranks findings globally, then applies fixes only when you approve them. |
| pay-check | `/devkit:pay-check` | **Payment-domain review** for money-handling backends — idempotency, money representation, atomic balance updates, webhook verification, transaction state machines, PAN/PII exposure, provider-call resilience. Same review→approve→apply flow. |
| codegen | `/devkit:codegen` | **Spec-driven scaffolding** — from a lightweight Markdown spec, generates a vertical slice (handler, service, repository, DTO/model, migration) that mirrors the repo's conventions. Migration included, tests deferred, no OpenAPI. |

## Install

From GitHub (after you push this repo):

```
/plugin marketplace add julles/devkit-plugins
/plugin install devkit@devkit-plugins
```

Or from a local clone:

```
/plugin marketplace add /path/to/devkit-plugins
/plugin install devkit@devkit-plugins
```

Then restart Claude Code (or run `/reload-plugins`).

### Try it without installing

```
claude --plugin-dir ./plugins/devkit
```

## Usage

Both skills follow the same flow: **review → numbered findings → you choose what
to apply.** Nothing is changed until you approve it.

1. Reads the target repo's `CLAUDE.md` for context (language, conventions).
2. Lists **numbered findings**, grouped Security → Performance → Simplicity, each
   with a severity (Critical / High / Medium / Low), `file:line`, the problem,
   and a suggested fix.
3. You decide:

   ```
   apply #1 #3
   apply all security
   skip
   ```

### review-sps — one feature/change

```
/devkit:review-sps                    # reviews your uncommitted git diff
/devkit:review-sps internal/auth/     # reviews a folder
/devkit:review-sps login flow         # finds the feature's files and reviews them
```

### audit-sps — the whole codebase

```
/devkit:audit-sps                     # scans the entire tracked source tree
/devkit:audit-sps internal/           # scans one subtree
```

It fans out across modules (parallel subagents on larger repos), ranks all
findings globally, and collapses repetitive low-severity simplicity noise into
counts so the important items stay visible.

### pay-check — payment-domain review

```
/devkit:pay-check                     # reviews payment code in your diff
/devkit:pay-check internal/payment/   # reviews a payment module
/devkit:pay-check refund flow         # finds and reviews the refund feature
```

Adds payment-specific rules on top of the generic review: money as integer
minor-units/decimal (never float), enforced idempotency keys, atomic balance
updates, webhook signature + replay protection, valid transaction state
transitions, no PAN/PII in logs, and resilient provider calls (timeout after a
charge → reconcile, don't blindly retry).

### codegen — spec-driven scaffolding

```
/devkit:codegen specs/refund.md       # generate from a spec file
/devkit:codegen a refund endpoint     # drafts the spec first, then generates
```

Two gates:

1. **Spec first** — codegen writes a lightweight Markdown spec (entity, fields,
   operations, business rules) and shows it for review. You edit/approve. It
   generates **no code** until the spec is approved, then saves it under `specs/`.
2. **Then code** — after approval it mirrors an existing feature slice in your
   repo (structure, naming, DI, error handling) and generates handler → service
   → repository → DTO/model → migration, with the house rules and payment rules
   baked in.

The **Markdown spec is the source of truth**. It does **not** generate tests or
OpenAPI. Re-running against an updated spec re-confirms the spec, then syncs the
code.

## Repo layout

```
devkit-plugins/
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog
├── plugins/
│   └── devkit/
│       ├── .claude-plugin/
│       │   └── plugin.json        # plugin manifest
│       └── skills/
│           ├── review-sps/
│           │   └── SKILL.md        # single feature/diff review
│           ├── audit-sps/
│           │   └── SKILL.md        # whole-codebase scan
│           ├── pay-check/
│           │   └── SKILL.md        # payment-domain review
│           └── codegen/
│               └── SKILL.md        # spec-driven scaffolding
└── README.md
```

## License

[MIT](https://reza.mit-license.org/) — see [LICENSE](LICENSE).
