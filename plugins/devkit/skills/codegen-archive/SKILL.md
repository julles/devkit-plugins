---
name: codegen-archive
description: Step 4 of spec-driven codegen. Archive the implemented spec — move specs/<feature>.md to specs/archive/<YYYY-MM-DD>-<feature>.md and remove the exploration scratch file. Use when the user says "codegen-archive <feature>" after verifying the generated feature.
---

# codegen-archive

**Step 4 of 4** (explore → create-spec → apply → archive). Run this once the
developer has verified the generated feature.

## Input

The `<feature>` slug, e.g. `codegen-archive merchant-payout`.

## What to do

1. Move the implemented spec into the archive, using today's date:

   ```
   specs/<feature>.md  →  specs/archive/<YYYY-MM-DD>-<feature>.md
   ```

   Create `specs/archive/` if it doesn't exist.
2. Delete the exploration scratch file `specs/<feature>.explore.md` if present —
   it has served its purpose.
3. Confirm the move.

The active `specs/` folder then holds only specs not yet implemented. A later
change is a **new** feature through the same flow (explore → create-spec → apply
→ archive) — not an edit of an archived spec.
