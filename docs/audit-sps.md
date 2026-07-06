# audit-sps

Audit an **entire existing codebase** against the priorities Security >
Performance > Simplicity, then apply fixes only when you approve them.

`audit-sps` is the repository-wide counterpart of [review-sps](review-sps.md). It
applies the same ruleset but adds the two capabilities a whole-repo scan
requires: **fan-out** (covering everything) and **noise control** (surfacing what
matters instead of drowning it).

## Invocation

```
/devkit:audit-sps            # scans the entire tracked source tree
/devkit:audit-sps internal/  # scans a single subtree
```

## Flow

1. **Determine scope.** Uses `git ls-files` (which respects `.gitignore`) and
   excludes dependencies (`node_modules`, `vendor`), build/generated output,
   lockfiles, and binaries. Tests are scanned only for hardcoded secrets. The
   file/line count to be covered is reported up front.
2. **Read repository context** (`CLAUDE.md`, manifests) per module.
3. **Fan out.** Small repositories are reviewed in a single pass. Larger ones are
   split by directory/module and reviewed by parallel sub-agents, one per group,
   which keeps the review fast and thorough. The number of groups used, and
   anything deliberately skipped, is reported — no silent truncation.
4. **Rank globally.** All findings are ranked by category (Security → Performance
   → Simplicity), then by severity within each category.
5. **Present the report**, then apply only what you approve.

## Noise control

A whole-repository audit can surface hundreds of items. To keep the signal
visible:

- **Every Security and Performance finding is listed** — these are the point of
  the audit.
- **Simplicity** findings of Medium severity and above are listed individually;
  repetitive Low-severity items are collapsed into counts by pattern, which you
  can expand on request.

```
Simplicity (Low): 12× dead code, 8× deep nesting, 5× single-implementation interface — reply "expand simplicity" to list.
```

## Example — auditing a payment gateway service

You have inherited a payment service and want a baseline health check:

```
/devkit:audit-sps
```

audit-sps reports that it will cover 214 source files (~31k lines) across
`internal/charge`, `internal/refund`, `internal/webhook`, `internal/ledger`, and
`internal/gateway`, using six parallel review groups. It then produces a ranked
report and a summary table:

| Area | Critical | High | Medium | Low |
| ---- | -------- | ---- | ------ | --- |
| Security | 2 | 3 | 4 | — |
| Performance | 1 | 5 | 6 | — |
| Simplicity | — | — | 3 | 27 (collapsed) |

Highlights:

- **#1 [Critical] Security — `internal/webhook/handler.go`** — inbound provider
  webhooks are processed before the signature is verified; an attacker can post a
  forged `charge.succeeded` event.
- **#2 [Critical] Security — `internal/gateway/client.go`** — the provider secret
  key is logged at debug level.
- **#3 [High] Performance — `internal/ledger/report.go`** — the settlement report
  loads every ledger entry into memory with no pagination; it will OOM as volume
  grows.

You triage from the top: `apply all security`, then work through Performance.

> For a deep, payment-specific pass on the money paths this audit surfaces, run
> [pay-check](pay-check.md) on the relevant modules.
