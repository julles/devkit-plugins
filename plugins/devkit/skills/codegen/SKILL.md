---
name: codegen
description: Spec-driven backend code generation. From a lightweight Markdown spec, generate a vertical slice (handler, service, repository, DTO/model, migration) that mirrors the repo's existing conventions. Use when the user says "codegen", "generate <feature/endpoint>", "scaffold <resource>", or "implement this spec". Generates migration; defers tests. Does NOT produce OpenAPI.
---

# codegen

Spec-driven generation for backend features. The **Markdown spec is the source of
truth**; code is generated to match it and stays in sync on re-runs. Output
**mirrors an existing feature in the repo** — it is not a generic template.

Rules of the road:
- **Migration is generated; tests are not** (leave tests to a separate step).
- **Do not create OpenAPI/Swagger** or any API-contract artifact.
- **Plan before writing:** show the file list and get approval first.
- Bake in the house rules from the start: input validation, parameterized
  queries, no over-engineering. For money paths, apply the `pay-check` rules
  (idempotency key, money as integer minor-units, atomic balance updates).

## 1. Get the spec

If the developer points to a spec file (e.g. `specs/<feature>.md`), use it. If
they only describe the feature, draft the spec in this shape, show it, and get
confirmation before generating:

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

Keep the spec in the repo (e.g. `specs/`) so it stays the source of truth.

## 2. Parse the spec

Extract entities, fields+constraints, operations (method/path/request/response),
and business rules. If something essential is missing (a field type, an error
case, an auth rule), ask — don't guess.

## 3. Learn the repo's conventions (mirror, don't invent)

Read the target repo's `CLAUDE.md` and manifests (`go.mod`, `package.json`) for
stack and conventions. Then **find one existing feature/slice to mirror** (an
analogous resource/module). Copy its structure, naming, error handling, DI,
validation approach, and query style. The generated code must read as if the
same team wrote it.

## 4. Plan the files (gate — confirm before writing)

Map each spec operation to the files to create/modify, one line each — typically:
handler/controller, service, repository, DTO/model (from the spec's entity),
migration, route/DI wiring. List modified files (router registration, module)
too. **Write nothing until the developer approves or adjusts the plan.**

## 5. Generate

Following the mirrored slice:
- **DTO/model** derived from the spec's entity fields + constraints.
- **Handler** validates the request against the spec; maps errors to the spec's
  status codes.
- **Service** holds the business rules from the spec.
- **Repository** with parameterized queries.
- **Migration** for the entity (money columns as integer minor-units or decimal,
  indexes on filtered/joined columns, sensible constraints; reversible).
- **Wiring**: register routes/DI/module.

For money/payment operations, generate code that already satisfies `pay-check`
(enforced idempotency key with a DB unique constraint, atomic balance updates,
no float money, no PAN/PII in logs). Keep it simple — no speculative abstraction.

Do **not** generate tests or OpenAPI.

## 6. Wire, verify, report

- Run the repo's build/typecheck (`go build ./...`, `npm run build`) and fix
  compile errors. Do not run or write tests.
- Report what was generated (per file) and what's left manual (e.g. provider
  credentials in config, env vars). Offer to run `/devkit:pay-check` on the new
  money code.

## Re-run = sync

On a later run against an updated spec, diff the spec against the existing
generated code and apply only the changes (new fields, new operations, changed
constraints). Preserve hand-written business logic — don't clobber it; flag
conflicts instead.
