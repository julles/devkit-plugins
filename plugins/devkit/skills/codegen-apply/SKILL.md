---
name: codegen-apply
description: Step 3 of spec-driven codegen. Generate the vertical slice (handler, service, repository, DTO/model, migration) from the approved spec at specs/<feature>.md, mirroring the repo's conventions. Generates migration; no tests; no OpenAPI. Use when the user says "codegen-apply <feature>". Running this means the spec is approved. Follow with codegen-archive when done.
---

# codegen-apply

**Step 3 of 4** (explore → create-spec → apply → archive). Running this command
means the developer has reviewed and **approved** `specs/<feature>.md`. Generate
the code from it.

## Input

The `<feature>` slug, e.g. `codegen-apply merchant-payout`.

## Rules

- **Migration is generated; tests are not** (leave tests to a separate step).
- **Do not create OpenAPI/Swagger.**
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
    unique constraint, money as integer minor-units, atomic balance updates,
    no PAN/PII in logs).

## What to do

1. Read the approved `specs/<feature>.md`. If it's missing, tell the developer to
   run `/devkit:codegen-create-spec <feature>` first.
2. Open the **mirror slice** named in the spec (and `specs/<feature>.explore.md`
   if present) and copy its structure, naming, DI, error handling, and query
   style.
3. List the files you'll create/modify, then generate the slice:
   - **DTO/model** from the spec's entity fields + constraints.
   - **Handler** that validates against the spec and maps errors to the spec's
     status codes.
   - **Service** holding the spec's business rules.
   - **Repository** with parameterized queries.
   - **Migration** for the entity (money columns as integer minor-units or
     decimal, indexes on filtered/joined columns, sensible constraints;
     reversible).
   - **Wiring**: register routes/DI/module.
4. Run the repo's build/typecheck (`go build ./...`, `npm run build`) and fix
   compile errors. Do not run or write tests.
5. Report what was generated (per file) and what's left manual (e.g. provider
   credentials in config). Offer to run `/devkit:pay-check` on the new money
   code. Tell the developer to run `/devkit:codegen-archive <feature>` once the
   feature is verified.
