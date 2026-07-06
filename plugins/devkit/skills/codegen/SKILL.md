---
name: codegen
description: Spec-driven backend code generation. First explore the repo, then auto-write a lightweight Markdown spec named after the feature for the developer to review/enhance and approve; only then generate a vertical slice (handler, service, repository, DTO/model, migration) that mirrors the repo's existing conventions; archive the spec when the developer says so. Use when the user says "codegen", "generate <feature/endpoint>", "scaffold <resource>", or "implement this spec". Generates migration; defers tests. Does NOT produce OpenAPI.
---

# codegen

Spec-driven generation for backend features. The flow is linear and one-shot:

**explore → create spec → developer approves/enhances → generate code → developer archives.**

This skill runs the whole flow end-to-end. Each step is **also available as its
own command** for granular control (state is handed off through files in
`specs/`):

- `/devkit:codegen-explore <feature>` → writes `specs/<feature>.explore.md`
- `/devkit:codegen-create-spec <feature>` → writes `specs/<feature>.md`
- `/devkit:codegen-apply <feature>` → generates code from the approved spec
- `/devkit:codegen-archive <feature>` → archives the spec

Use the step commands when you want to pause between stages; use this umbrella
command to run straight through.

The spec is a **single-use instruction**, not a living contract. Each new feature
or change is a fresh spec. Generated code **mirrors an existing feature in the
repo** — it is not a generic template.

Rules of the road:
- **Never generate code before the spec is approved.**
- **Migration is generated; tests are not** (leave tests to a separate step).
- **Do not create OpenAPI/Swagger** or any API-contract artifact.
- **Generate code that already passes the SPS review**, in priority order
  Security > Performance (non-negotiable) > Simplicity:
  - **Security**: validate input at the boundary, parameterized queries, no
    hardcoded secrets, permission checks on protected operations.
  - **Performance**: no N+1 (load related data in one query), paginate list
    endpoints, index columns the queries filter/join/sort on, no needless work
    on hot paths.
  - **Simplicity**: no speculative abstraction — mirror the existing slice, keep
    it something a junior can follow.
  - For money paths, also apply the `pay-check` rules (idempotency key with a DB
    unique constraint, money as integer minor-units, atomic balance updates).

## Step 1 — Explore

Before writing anything, understand the repo. Read `CLAUDE.md` and manifests
(`go.mod`, `package.json`) for stack and conventions, then explore the code
(use the `Explore` agent or codegraph on larger repos) to find:

- **A mirror slice**: an existing analogous feature/module to copy the structure,
  naming, DI, error handling, validation, and query style from.
- **Neighbours of the entity**: related models, existing tables/columns, enums,
  and how similar endpoints are shaped.
- **Cross-cutting conventions**: auth/permission approach, error/response format,
  pagination style, migration tooling.

Summarize what you found briefly — this grounds the spec in the real codebase,
not assumptions.

## Step 2 — Create the spec (auto)

From the exploration, **automatically write** the spec file, named after the
feature, at `specs/<feature>.md` (kebab-case, e.g. `specs/refund.md`). Ground it
in what you found (real entity fields, real conventions, real endpoint shapes):

```markdown
# <Feature name>

## Entity: <Name>
| field | type | constraints |
| ----- | ---- | ----------- |
| id    | uuid | pk          |
| ...   | ...  | ...         |

## Operations
### <METHOD> <path> — <summary>
- Request: <fields / body>
- Response: <fields / status codes>
- Rules: <business rules; e.g. idempotent via Idempotency-Key, amount > 0>

## Business rules
- <cross-cutting rules: money is minor-units, state transitions, auth, ...>

## Mirror slice
- <path of the existing feature this will be modeled on>
```

If essentials are genuinely unknowable from the code (a business rule, an error
case), ask — don't invent them.

## Step 3 — Developer approves / enhances (gate)

Show the spec file and ask the developer to review it. They may edit it directly,
have you enhance it, add rules, or approve. **Stop here until they approve.** Do
not write any code yet.

## Step 4 — Generate code from the approved spec

Following the mirror slice from Step 1, briefly list the files you'll
create/modify, then generate:
- **DTO/model** from the spec's entity fields + constraints.
- **Handler** validates the request against the spec; maps errors to the spec's
  status codes.
- **Service** holds the spec's business rules.
- **Repository** with parameterized queries.
- **Migration** for the entity (money columns as integer minor-units or decimal,
  indexes on filtered/joined columns, sensible constraints; reversible).
- **Wiring**: register routes/DI/module.

For money/payment operations, generate code that already satisfies `pay-check`.
Keep it simple — no speculative abstraction. Do **not** generate tests or OpenAPI.

Then run the repo's build/typecheck (`go build ./...`, `npm run build`) and fix
compile errors. Report what was generated (per file) and what's left manual
(e.g. provider credentials in config). Offer to run `/devkit:pay-check` on the
new money code.

## Step 5 — Developer archives

**Do not archive automatically.** Once the developer is satisfied and asks to
archive (or confirms the feature is done), move the spec into the archive:

```
specs/<feature>.md  →  specs/archive/<YYYY-MM-DD>-<feature>.md
```

Use today's date; create `specs/archive/` if needed. The active `specs/` folder
then only holds specs not yet implemented. A later change is a **new** spec
through this same flow — not an edit of an archived one.
