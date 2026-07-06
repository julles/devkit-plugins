# pr-review

Review a **GitHub pull request** from its URL against the priorities Security >
Performance > Simplicity. `pr-review` fetches the diff remotely via the `gh` CLI,
so no local checkout is needed, and can optionally post its findings back to the
PR as review comments.

## Prerequisite

The `gh` CLI must be installed and authenticated:

```
gh auth status   # if this fails, run: gh auth login
```

## Invocation

Paste the PR link (or `owner/repo#123`):

```
/devkit:pr-review https://github.com/acme/payments/pull/482
/devkit:pr-review acme/payments#482
```

## Flow

1. **Fetch** the PR context (`gh pr view`) and diff (`gh pr diff`), and read the
   title/description to understand intent.
2. **Repo context** — if the PR is in the current repository, read its
   `CLAUDE.md`; otherwise infer conventions from the changed files.
3. **Review** the diff against the SPS ruleset (same rules as
   [review-sps](review-sps.md)). If the diff touches money code, it recommends a
   [pay-check](pay-check.md) pass.
4. **Present numbered findings**, grouped Security → Performance → Simplicity, by
   severity, with an overall approve / request-changes recommendation. Nothing is
   posted or changed.
5. **Post or apply** only what you approve:
   - **Post to the PR** — a summary review comment, plus optional inline
     line-level comments, via `gh`. It never approves or requests changes on your
     behalf unless you say so.
   - **Apply locally** — if you have checked out the PR branch
     (`gh pr checkout`), it edits the files for the findings you approve.

## Output language

English by default. Ask for Indonesian ("bahasa Indonesia") and the findings —
and any comments posted to the PR — are written in Indonesian. Code, identifiers,
file paths, and severity labels stay unchanged.

## Example — reviewing a payment PR

```
/devkit:pr-review https://github.com/acme/payments/pull/482
```

pr-review fetches PR #482 ("Add partial refunds"), reads the description, and
reviews the diff:

```
#1 [Critical] internal/refund/service.go:61
    Problem:    Partial refund amount is validated on the client but trusted
                server-side; a caller can refund more than was captured.
    Suggestion: Re-validate amount <= captured_amount from the authoritative
                charge record before issuing the refund.

#2 [High] internal/refund/repository.go:44
    Problem:    Refund list endpoint loads all rows for the merchant with no
                pagination.
    Suggestion: Add limit/offset (or keyset) pagination and an index on
                (merchant_id, created_at).

Summary: 1 Critical (security), 1 High (performance). Recommendation: request
changes. This PR touches money — run /devkit:pay-check on the branch too.
Post to the PR, apply locally, or skip?
```

You reply `post #1 #2`, and pr-review adds a review with those two inline
comments. #1 is left as "request changes" only when you explicitly ask.
