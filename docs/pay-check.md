# pay-check

A **payment-domain review** for backend code that moves money. `pay-check` is the
domain layer on top of [review-sps](review-sps.md): it looks specifically for the
mistakes that lose real money — incorrect money representation, missing
idempotency, non-atomic balance updates, unverified webhooks, and leaked card
data.

Because money bugs are financial incidents, `pay-check` treats correctness of
money and idempotency as **Critical**, at the same tier as security.

## Invocation

```
/devkit:pay-check                     # reviews payment code in your diff
/devkit:pay-check internal/payment/   # reviews a payment module
/devkit:pay-check "refund flow"       # locates and reviews the refund feature
```

It focuses on code paths that touch money — charges, authorizations, captures,
refunds, transfers, payouts, ledgers, balances, wallets, and payment webhooks —
and ignores unrelated code (that is [review-sps](review-sps.md)'s job).

## Flow

Same shape as the other review skills: resolve scope → read context (including
which payment **provider** is in use, since signature schemes and idempotency
support differ) → review against the payment ruleset → present numbered findings
→ apply only what you approve.

## What it checks

### Money representation — _Critical when wrong_
- Amounts held as `float`/`double` instead of integer minor units (cents) or a
  decimal type.
- Implicit or inconsistent rounding.
- Arithmetic that mixes or drops currency.
- Serialization that loses precision.

### Idempotency — _Critical for mutating endpoints_
- Charge/refund/transfer endpoints that do not accept or enforce an idempotency
  key.
- Idempotency enforced only in application code, not by a database unique
  constraint — a race then lets a double-charge through.
- Replays that re-run the side effect instead of returning the original result.

### Atomicity & concurrency
- Balance/ledger read-modify-write without a transaction or lock (lost update,
  double-spend).
- No optimistic (version) or pessimistic locking on the balance row.
- Multi-step money movements not wrapped in a single transaction.

### Webhooks
- Inbound webhooks processed before the signature (HMAC) is verified.
- Signatures compared with `==` instead of a constant-time comparison.
- No replay protection (timestamp tolerance + event-id de-duplication).
- Outbound webhooks lacking retries/backoff or a signed payload.

### Transaction state machine
- Illegal state transitions (for example, capturing after a refund, or mutating a
  terminal state).
- Status changes made without an audit record.

### Provider calls & unknown state
- Provider calls without a timeout and bounded retry/backoff.
- A network timeout _after_ a charge treated as a plain failure and blindly
  retried, causing a double-charge; unknown outcomes require reconciliation, not
  naive retry.

### Data protection — _Critical when leaked_
- PAN/CVV/full card data logged, stored unmasked, or returned in responses.
- Hardcoded payment secrets (gateway keys, webhook secrets).
- Over-logging of PII around payment events.

### Amount & input validation
- Negative or zero amounts accepted where invalid; missing min/max/overflow
  checks.
- Amount or currency trusted from the client instead of authoritative
  server-side data.

## Example — reviewing a refund handler

```
/devkit:pay-check "refund flow"
```

pay-check reads `CLAUDE.md`, sees the service integrates Midtrans, and reviews the
refund handler, service, and ledger update. It reports:

```
#1 [Critical] internal/refund/service.go:57
    Problem:    Refund amount is a float64; multiplying by the exchange rate
                introduces rounding error (e.g. 10.00 -> 9.999999).
    Suggestion: Store and compute amounts as int64 minor units; apply a single,
                explicit rounding mode at the boundary only.

#2 [Critical] internal/refund/handler.go:23
    Problem:    POST /refunds does not accept an Idempotency-Key; a client retry
                after a timeout issues a second refund.
    Suggestion: Require an Idempotency-Key header, persist (key -> refund_id) with
                a UNIQUE constraint, and return the original result on replay.

#3 [Critical] internal/refund/ledger.go:41
    Problem:    Balance is read, decremented in Go, then written back — a
                concurrent refund can double-spend.
    Suggestion: Use an atomic UPDATE ... SET balance = balance - $1 WHERE ...
                inside the same transaction, or optimistic locking on a version.

#4 [High] internal/webhook/midtrans.go:15
    Problem:    The refund-status webhook is processed before verifying the
                signature.
    Suggestion: Verify the Midtrans signature (constant-time) and reject on
                mismatch before any state change; de-duplicate by event id.

Summary: 3 Critical, 1 High (money & idempotency). Which do you want to apply?
(e.g. 'apply #1 #3', 'apply all idempotency', or 'skip')
```

You reply `apply #1 #2 #3`. pay-check fixes the money type, adds idempotency
enforcement (flagging that it needs a migration for the unique constraint), and
converts the balance update to an atomic statement, then reports each change. #4
is left for a follow-up because it involves the shared webhook layer.

## Recommended usage

Run `pay-check` alongside `review-sps` on every payment endpoint: `review-sps`
for generic quality, `pay-check` for the money path. When scaffolding with
[codegen](codegen.md), the generated money code already satisfies these rules,
so `pay-check` on new code should come back clean.
