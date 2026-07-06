---
name: codegen-explore
description: Step 1 of spec-driven codegen. Explore the repository for a feature and write exploration notes to specs/<feature>.explore.md — a mirror slice, the entity's neighbours, and cross-cutting conventions. Writes NO spec and NO code. Use when the user says "codegen-explore <feature>". Follow with codegen-create-spec.
---

# codegen-explore

**Step 1 of 4** in the spec-driven codegen flow
(explore → create-spec → apply → archive). This step only explores and records
what it finds. It does **not** write a spec and does **not** generate code.

## Input

A feature name, e.g. `codegen-explore merchant payout`. Derive a kebab-case
`<feature>` slug (e.g. `merchant-payout`) and use it for the output file. Reuse
the same slug in the later steps.

## What to do

1. Read the target repo's `CLAUDE.md` and manifests (`go.mod`, `package.json`)
   for stack, framework, and conventions.
2. Explore the code (use the `Explore` agent or codegraph on larger repos) to
   find:
   - **A mirror slice** — an existing analogous feature/module to copy structure,
     naming, DI, error handling, validation, and query style from.
   - **The entity's neighbours** — related models, existing tables/columns,
     enums, and how similar endpoints are shaped.
   - **Cross-cutting conventions** — auth/permission approach, error/response
     format, pagination style, migration tooling.
3. Write the findings to `specs/<feature>.explore.md`:

```markdown
# Exploration: <Feature name>

## Mirror slice
- <path of the closest existing feature to model on, and why>

## Entity neighbours
- <related models / tables / columns / enums found>

## Conventions
- Auth: <how protected endpoints check permission>
- Errors: <response/error format>
- Pagination: <style used for list endpoints>
- Migrations: <tool and layout>

## Open questions
- <anything the spec will need that isn't answerable from the code>
```

4. Summarize what you found in the chat, then tell the developer the next step:
   `/devkit:codegen-create-spec <feature>`.
