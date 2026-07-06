---
name: codegen-create-spec
description: Step 2 of spec-driven codegen. Turn the exploration notes into a lightweight Markdown spec at specs/<feature>.md for the developer to review and approve. Writes NO code. Use when the user says "codegen-create-spec <feature>". Follow with codegen-apply once the developer approves the spec.
---

# codegen-create-spec

**Step 2 of 4** (explore → create-spec → apply → archive). This step writes the
spec file. It does **not** generate code.

## Input

The same `<feature>` slug used in step 1, e.g. `codegen-create-spec merchant-payout`.

## What to do

1. Read `specs/<feature>.explore.md`. If it doesn't exist, tell the developer to
   run `/devkit:codegen-explore <feature>` first (or do a quick inline
   exploration if they ask you to proceed anyway).
2. Write `specs/<feature>.md`, grounded in the exploration (real entity fields,
   real conventions, real endpoint shapes):

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
- <path of the existing feature this will be modeled on (from exploration)>
```

3. If an essential is genuinely unknowable from the code (a business rule, an
   error case), ask — don't invent it.
4. Show the spec and tell the developer: **review and edit it, then run
   `/devkit:codegen-apply <feature>` to generate the code.** Running apply is the
   approval — do not generate anything here.
