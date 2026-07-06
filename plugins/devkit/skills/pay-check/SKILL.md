---
name: pay-check
description: Review payment/money-handling backend code for domain-specific correctness and safety — idempotency, money representation, atomic balance updates, webhook verification, transaction state machines, PAN/PII exposure, and provider-call resilience. Use when the user says "pay-check", "review payment code", "check idempotency/webhooks/money handling", or reviews anything touching charges, refunds, transfers, ledgers, balances, or payment webhooks. Complements review-sps (generic Security/Performance/Simplicity) with payment-domain rules.
---

# pay-check

Review backend code that moves money. This is the **payment-domain** layer on top
of `review-sps`: money bugs lose real money, so treat correctness of money and
idempotency as **Critical**, at the same tier as security.

Required flow: **review first → present numbered findings + suggestions → DO NOT
apply.** Only edit files when the developer explicitly asks ("implement", "apply
#2", "apply all money"). Apply only what they approve — never auto-apply.

## Output language

Write findings in **English by default**. If the developer asks for Indonesian
(e.g. "bahasa Indonesia", "in Indonesian", or an `id` / `--lang id` argument),
write the Problem, Suggestion, and summary text in Indonesian. Always keep code,
identifiers, file paths, severity labels, and command names unchanged.

## 1. Determine scope (from arguments)

- **Empty / "diff"** → `git diff` (unstaged + staged); if clean, `git diff
  <base>...HEAD` vs the main branch.
- **File/folder path** → review that path.
- **Feature name** (e.g. "refund flow", "payout") → find related files by
  grepping/globbing; confirm scope if ambiguous.

Focus on code paths that touch money: charges, authorizations, captures,
refunds, transfers, payouts, ledgers, balances, wallets, and payment webhooks.
Ignore unrelated code — that's `review-sps`'s job.

## 2. Read repo context

Read the target repo's `CLAUDE.md` and manifests (`go.mod`, `package.json`) for
language, framework, and the payment provider(s) in use (Stripe, Midtrans,
Xendit, etc). Provider matters: signature schemes, idempotency support, and
webhook formats differ.

## 3. Review against the payment ruleset

Check **only** what's actually in the reviewed code. Don't invent findings. If
clean, say so.

### Money representation (Critical when wrong)
- Amounts as `float`/`double` → must be integer minor units (cents) or a decimal type
- Rounding is implicit or inconsistent → require an explicit, single rounding mode
- Arithmetic mixes or drops currency → every amount carries a currency; never add across currencies
- JSON/serialization loses precision (large integers, float encoding)

### Idempotency (Critical for mutating endpoints)
- Charge/refund/transfer endpoints don't accept or enforce an idempotency key
- Idempotency enforced only in app code, not at the DB (unique constraint) → race lets a double-charge through
- Replays don't return the original result / re-run the side effect

### Atomicity & concurrency
- Balance/ledger read-modify-write without a transaction or lock → lost update / double-spend
- No optimistic (version) or pessimistic locking on the balance row
- Multi-step money moves not wrapped in one transaction (partial commit)

### Webhooks
- Inbound webhook processed without verifying the signature (HMAC) first
- Signature compared with `==` instead of a constant-time compare
- No replay protection (timestamp tolerance + event-id dedup) → duplicate delivery re-applies effects
- Outbound webhooks lack retries/backoff or a signed payload

### Transaction state machine
- Illegal state transitions allowed (e.g. capture after refund, mutate a terminal state)
- Status changed without an audit record

### Provider calls & unknown state
- Provider calls without timeout + bounded retry/backoff
- Network timeout after a charge treated as failure and blindly retried → double-charge; unknown outcomes need reconciliation, not naive retry

### Data protection (Critical when leaked)
- PAN/CVV/full card data logged, stored unmasked, or returned in responses → mask/tokenize
- Payment secrets (gateway keys, webhook secrets) hardcoded
- PII over-logged around payment events

### Amount & input validation
- Negative/zero amounts accepted where invalid; missing min/max/overflow checks
- Amount or currency trusted from the client instead of authoritative server-side data

## 4. Present findings (DO NOT apply)

Group by the areas above, sorted by severity (**Critical / High / Medium /
Low**). Money-loss, double-charge, missing idempotency, float money, and leaked
card data are Critical/High. Number findings globally (#1, #2, ...).

Format per finding:

```
#N [SEVERITY] path/file.ext:line
    Problem:    <one sentence>
    Suggestion: <concrete fix; include a code snippet if useful>
```

Close with a one-line summary (counts + highest severity), then ask:
*"Which do you want to apply? (e.g. 'apply #1 #3', 'apply all idempotency', or 'skip')"*.

If the payment code is clean, say so and stop.

## 5. Apply (only when asked)

- Apply **only** the findings the developer names (or "all" / "all <area>").
- Edit the affected files; keep the repo's style and conventions. Money and
  idempotency fixes often need a migration or a lock — flag that, don't silently
  half-fix.
- After applying, report briefly per finding what changed. If a fix needs a
  larger design decision (schema change, provider config), ask first.
- Don't touch findings they didn't approve.
