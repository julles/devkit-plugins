# codegen

**Spec-driven** backend scaffolding. `codegen` explores the repository, writes a
lightweight Markdown spec, waits for your approval, then generates a vertical
slice that mirrors the codebase's existing conventions — and archives the spec
when you say so.

The spec is a **single-use instruction**, not a living contract. Each feature or
change is a fresh spec through the same flow.

## Invocation

```
/devkit:codegen "refund endpoint"     # explores, drafts a spec, then generates
/devkit:codegen specs/payout.md       # starts from an existing spec file
```

## The five-step flow

**explore → create spec → approve/enhance → generate → archive.**

### 1. Explore
Before writing anything, `codegen` reads `CLAUDE.md` and the manifests, then
explores the code to find:
- a **mirror slice** — an existing analogous feature to copy structure, naming,
  dependency injection, error handling, and query style from;
- the **entity's neighbours** — related models, tables, columns, and enums;
- **cross-cutting conventions** — auth, error/response format, pagination,
  migration tooling.

This grounds the spec in the real codebase instead of assumptions.

### 2. Create the spec (automatic)
`codegen` writes the spec file itself, named after the feature, at
`specs/<feature>.md`:

```markdown
# Payout

## Entity: Payout
| field         | type   | constraints                    |
| ------------- | ------ | ------------------------------ |
| id            | uuid   | pk                             |
| merchant_id   | uuid   | fk -> merchants, indexed       |
| amount_minor  | bigint | > 0                            |
| currency      | char(3)| ISO 4217                       |
| status        | text   | pending|processing|paid|failed |
| idempotency_key | text | unique                         |
| created_at    | timestamptz | default now()             |

## Operations
### POST /payouts — request a payout
- Request: { merchant_id, amount_minor, currency }, header Idempotency-Key
- Response: 201 { id, status } | 409 on duplicate key | 422 on validation error
- Rules: amount_minor > 0; merchant must have sufficient available balance

## Business rules
- Money is stored and computed in integer minor units.
- Balance is debited atomically within the payout transaction.
- Status transitions: pending -> processing -> paid | failed (terminal).

## Mirror slice
- internal/refund (closest existing money-movement feature)
```

### 3. Approve / enhance (gate)
`codegen` shows the spec and stops. You can edit it directly, ask `codegen` to
enhance it, add rules, or approve. **No code is generated until you approve.**

### 4. Generate
Following the mirror slice, `codegen` lists the files it will create/modify, then
generates the vertical slice:
- **DTO/model** from the entity;
- **handler** that validates against the spec and maps errors to the spec's
  status codes;
- **service** holding the business rules;
- **repository** with parameterized queries;
- **migration** for the entity (money columns as integer minor units, indexes on
  filtered/joined columns, reversible);
- **wiring** (routes, DI, module registration).

It then builds/typechecks the project and fixes compile errors. It does **not**
generate tests or OpenAPI.

The generated code already satisfies the SPS priorities and, for money paths, the
[pay-check](pay-check.md) rules — enforced idempotency with a database unique
constraint, atomic balance updates, integer money, no card data in logs.

### 5. Archive (developer-triggered)
`codegen` does **not** archive automatically. When you confirm the feature is
done, it moves the spec to the archive as a historical record:

```
specs/payout.md  →  specs/archive/2026-07-07-payout.md
```

The active `specs/` folder then holds only specs not yet implemented.

## What it does and does not do

| Does | Does not |
| ---- | -------- |
| Explore and mirror existing conventions | Use a generic template |
| Generate handler, service, repository, DTO/model | Generate tests |
| Generate a reversible migration | Produce OpenAPI/Swagger |
| Bake in SPS + payment rules | Write code before spec approval |
| Archive the spec on request | Archive automatically or keep a living spec |

## Example — scaffolding a payout feature

```
/devkit:codegen "merchant payout"
```

1. **Explore.** `codegen` finds `internal/refund` as the closest money-movement
   slice, notes the project uses Go, `sqlx`, chi, and golang-migrate, and sees
   the existing idempotency helper in `internal/idempotency`.
2. **Create spec.** It writes `specs/merchant-payout.md` (as above) and shows it.
3. **Approve.** You add a rule — payouts above a threshold require manual
   approval — and approve.
4. **Generate.** It creates `internal/payout/{model,handler,service,repository}.go`,
   `migrations/000123_create_payouts.up.sql` / `.down.sql`, registers the route,
   and confirms `go build ./...` passes. It notes that the payout provider
   credentials still need to be added to config, and offers to run `pay-check`.
5. **Archive.** After you verify the feature, you say "archive", and the spec
   moves to `specs/archive/2026-07-07-merchant-payout.md`.
