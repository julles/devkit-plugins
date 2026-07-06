---
name: codegen
description: Spec-driven backend code generation. First generate a lightweight Markdown spec for the developer to review and approve; only then generate a vertical slice (handler, service, repository, DTO/model, migration) that mirrors the repo's existing conventions; then archive the spec. Use when the user says "codegen", "generate <feature/endpoint>", "scaffold <resource>", or "implement this spec". Generates migration; defers tests. Does NOT produce OpenAPI.
---

# codegen

Spec-driven generation for backend features. The flow is linear and one-shot:

**create spec → developer approves → generate code → archive spec.**

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

## Step 1 — Create the spec

Whatever the developer gives you (a one-line request, a rough description, or an
existing spec file), produce a spec in this shape and save it as the active spec
at `specs/<feature>.md`:

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
```

If essentials are missing (a field type, an error case, an auth rule), ask
before finalizing — don't guess.

## Step 2 — Developer approves (gate)

Show the full spec and ask the developer to review it. They may edit, add rules,
or approve. **Stop here until they approve.** Do not write any code yet.

## Step 3 — Generate code from the approved spec

### Learn the repo's conventions (mirror, don't invent)
Read the target repo's `CLAUDE.md` and manifests (`go.mod`, `package.json`) for
stack and conventions. Then **find one existing feature/slice to mirror** (an
analogous resource/module). Copy its structure, naming, error handling, DI,
validation approach, and query style — the code must read as if the same team
wrote it.

### Generate the slice
Briefly list the files you'll create/modify, then generate:
- **DTO/model** from the spec's entity fields + constraints.
- **Handler** validates the request against the spec; maps errors to the spec's
  status codes.
- **Service** holds the spec's business rules.
- **Repository** with parameterized queries.
- **Migration** for the entity (money columns as integer minor-units or decimal,
  indexes on filtered/joined columns, sensible constraints; reversible).
- **Wiring**: register routes/DI/module.

For money/payment operations, generate code that already satisfies `pay-check`
(enforced idempotency key with a DB unique constraint, atomic balance updates,
no float money, no PAN/PII in logs). Keep it simple — no speculative abstraction.

Do **not** generate tests or OpenAPI.

### Wire, verify, report
- Run the repo's build/typecheck (`go build ./...`, `npm run build`) and fix
  compile errors. Do not run or write tests.
- Report what was generated (per file) and what's left manual (e.g. provider
  credentials in config, env vars). Offer to run `/devkit:pay-check` on the new
  money code.

## Step 4 — Archive the spec

Once code generation succeeds, move the spec out of the active folder into the
archive as a historical record:

```
specs/<feature>.md  →  specs/archive/<YYYY-MM-DD>-<feature>.md
```

Use today's date. Create `specs/archive/` if it doesn't exist. The active
`specs/` folder should only hold specs not yet implemented. A later change is a
**new** spec through this same flow — not an edit of an archived one.
