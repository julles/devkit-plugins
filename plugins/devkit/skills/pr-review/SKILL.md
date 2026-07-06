---
name: pr-review
description: Review a GitHub pull request from its URL with priorities Security > Performance > Simplicity. Fetches the PR diff via the gh CLI, reviews it, and presents numbered findings; can optionally post them back to the PR as review comments. Use when the user pastes a pull request link or says "pr-review", "review this PR", "review pull request <url>".
---

# pr-review

Review a **GitHub pull request** against the priorities Security > Performance
(non-negotiable) > Simplicity. Same ruleset as `review-sps`, but the input is a
PR URL and the diff is fetched remotely — no local checkout required.

Required flow: **review → present numbered findings → do not change anything.**
Only post to the PR or apply fixes when the developer explicitly asks.

## Output language

Write findings in **English by default**. If the developer asks for Indonesian
(e.g. "bahasa Indonesia", "in Indonesian", or an `id` / `--lang id` argument),
write the Problem, Suggestion, and summary text in Indonesian. Always keep code,
identifiers, file paths, severity labels, and command names unchanged. Comments
posted to the PR follow the same language choice.

## Prerequisite

The `gh` CLI must be installed and authenticated (`gh auth status`). If it
isn't, tell the developer to run `gh auth login` and stop.

## 1. Fetch the PR

From the pasted URL (or `owner/repo#123`):

- **Context**: `gh pr view <url> --json title,body,author,headRefName,baseRefName,files`
- **Diff**: `gh pr diff <url>`

Read the PR title/description to understand intent. If the PR is large, review
the highest-risk files first (auth, payment, data access) and say what you
prioritized.

## 2. Repo context

If the PR belongs to the current working repository, read its `CLAUDE.md` for
conventions. Otherwise infer the stack from the changed files and file paths.
Use it so findings fit the repo's conventions.

## 3. Review the diff against the SPS ruleset

Check **only** what the diff actually changes. Don't invent findings. Review the
new/changed lines, but flag when a change breaks an assumption elsewhere.

### Security (top priority)
- Unvalidated input at trust boundaries; injection (SQL/NoSQL/command/template);
  hardcoded secrets; broken authz/authn and IDOR; sensitive data in logs or
  responses; dangerous operations on untrusted input.

### Performance (non-negotiable)
- N+1 queries / queries in loops; missing indexes on filtered/joined/sorted
  columns; unbounded result sets; repeated work that should be cached; blocking
  I/O on hot paths; avoidable super-linear algorithms.

### Simplicity (so a junior can follow it)
- Single-implementation abstractions; reinvented stdlib/native features; deep
  nesting, over-long functions, unclear naming; dead code and unused params.

If the diff touches money/payment code (charges, refunds, transfers, ledgers,
balances, webhooks), note that and recommend running `/devkit:pay-check` for the
payment-specific rules.

## 4. Present findings (do not change anything)

Group by category (Security → Performance → Simplicity), sorted by severity
(**Critical / High / Medium / Low**). Number findings globally.

```
#N [SEVERITY] path/file.ext:line
    Problem:    <one sentence>
    Suggestion: <concrete fix; include a code snippet if useful>
```

Close with a one-line summary (counts + highest severity, plus an overall
recommendation: approve / request changes) and ask what to do next:
*"Post to the PR, apply locally, or skip? (e.g. 'post all', 'post #1 #2', 'skip')"*.

If the PR is clean against SPS, say so and recommend approval.

## 5. Post or apply (only when asked)

- **Post to the PR** — use `gh` to add a review with the requested findings:
  a summary comment via `gh pr review <url> --comment --body "..."`, and, when
  the developer wants line-level comments, inline comments via
  `gh api repos/{owner}/{repo}/pulls/{n}/comments`. Post only the findings they
  approve; never approve or request-changes on their behalf unless they say so.
- **Apply locally** — only if the PR branch is checked out locally
  (`gh pr checkout <url>`). Then apply the approved findings by editing the
  files, keeping the repo's conventions. Do not push without being asked.
- Report what was posted/changed. Leave unapproved findings untouched.
