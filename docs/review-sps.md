# review-sps

Review a **single feature or change** against the priorities Security >
Performance > Simplicity, then apply fixes only when you approve them.

Use `review-sps` for targeted review — an uncommitted diff, a set of files, or a
named feature. For a whole-repository sweep, use [audit-sps](audit-sps.md); for
money-handling code, layer [pay-check](pay-check.md) on top.

## Invocation

```
/devkit:review-sps                    # reviews your uncommitted git diff
/devkit:review-sps internal/billing/  # reviews a folder
/devkit:review-sps "checkout flow"    # locates the feature's files, then reviews
```

### Scope resolution

| Argument | Scope |
| -------- | ----- |
| _(none)_ or `diff` | `git diff` of unstaged + staged changes; if clean, the diff against the main branch |
| a file or folder path | the contents of that path |
| a feature name | files located by searching for that name (you are asked to confirm if ambiguous) |

## Flow

1. **Resolve scope** from the argument.
2. **Read repository context** (`CLAUDE.md`, manifests) to align with the repo's
   language and conventions.
3. **Review** the code against the SPS ruleset, reporting only issues that are
   actually present.
4. **Present numbered findings**, grouped Security → Performance → Simplicity and
   sorted by severity. Nothing is changed.
5. **Apply** only the findings you approve.

## What it checks

**Security** — unvalidated input at trust boundaries; SQL/NoSQL/command/template
injection; hardcoded secrets; broken authentication/authorization and IDOR;
sensitive data in logs or responses; dangerous operations on untrusted input
(deserialization, `eval`, path traversal, SSRF).

**Performance** — N+1 queries and queries inside loops; missing indexes on
filtered/joined/sorted columns; unbounded result sets (no pagination); repeated
work that should be cached or computed once; blocking I/O on hot paths; avoidable
super-linear algorithms.

**Simplicity** — abstractions, interfaces, or factories with a single
implementation; configuration for values that never change; reinvented standard
library or native features; deep nesting, over-long functions, unclear naming;
dead code and unused parameters.

## Output format

```
#1 [Critical] internal/billing/charge_repository.go:42
    Problem:    Query built with string concatenation of `customerID` — SQL injection.
    Suggestion: Use a parameterized query: db.Query("... WHERE customer_id = $1", customerID).

#2 [High] internal/billing/charge_service.go:88
    Problem:    Customer is loaded per transaction inside the loop — N+1 (1 + N queries).
    Suggestion: Batch-load customers by ID before the loop, or JOIN in the transaction query.

Summary: 1 Critical (security), 1 High (performance). Which do you want to apply?
(e.g. 'apply #1 #3', 'apply all security', or 'skip')
```

## Example — reviewing a new "create charge" endpoint

You have implemented a `POST /charges` endpoint and want it reviewed before
opening a pull request:

```
/devkit:review-sps "create charge"
```

review-sps locates the handler, service, and repository, reads `CLAUDE.md` to
confirm the stack (Go, sqlx, chi router), and reports:

- **#1 [Critical] Security** — the gateway API key is read from a hardcoded
  constant instead of configuration; move it to an environment variable / secrets
  store.
- **#2 [High] Performance** — the endpoint fetches the customer's full
  transaction history to compute a running total on every charge; replace with an
  aggregate query and an index on `(customer_id, created_at)`.
- **#3 [Low] Simplicity** — a `PricingStrategy` interface has a single
  implementation; inline it until a second pricing model actually exists.

You reply `apply all security`, and only finding #1 is fixed. #2 and #3 are left
for you to schedule.

> Note: `review-sps` covers generic security/performance/simplicity. Payment-
> specific concerns — idempotency, money representation, webhook verification —
> are the job of [pay-check](pay-check.md). Run both on payment endpoints.
