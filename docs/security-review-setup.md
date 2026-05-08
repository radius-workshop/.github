# Org-Wide Security Review

Every pull request opened against the default branch of any repository in
the `radius-workshop` organization runs an automated security scan. PRs
cannot be merged until the scan passes.

This document describes the configuration. There is **nothing to set up
per repository** — new repos are covered automatically.

For a non-engineer-friendly summary, see the user-facing doc on Notion:
**[Github Security Scanning with Claude](https://www.notion.so/Github-Security-Scanning-with-Claude-35a42ab48031809c8279ce2c9ff2a5e8)**.

## What's enforced

The org-level ruleset `protect-main` targets every repository's default
branch and applies these rules:

- **Pull request required.** Direct pushes to `main` are rejected.
- **Conversation resolution required.** Open review threads block merge.
- **Required workflow.** `Claude Security Review (required)` must finish
  cleanly. With `fail-on-findings: true` hardcoded, any finding fails the
  workflow and blocks merge.
- **Force-push and deletion blocked** on the default branch.
- **Required approvals: 0.** The PR + scan gates apply at this level;
  raise later if peer review becomes a requirement.

Org admins appear in the ruleset's bypass list (pull requests only) as a
safety valve for genuine emergencies.

## How it's wired together

```
GitHub PR event
        │
        ▼
  org ruleset (protect-main)
        │ requires
        ▼
  security-review-required.yml ─── on: pull_request, fail-on-findings: true
        │ uses (reusable workflow)
        ▼
  security-review.yml ─── checks out, runs the action with org secrets
        │
        ▼
  anthropics/claude-code-security-review
        + security/custom-scan-instructions.txt
        + security/false-positive-filtering.txt
```

| File | Role |
|---|---|
| `.github/workflows/security-review-required.yml` | Org-required entrypoint referenced by the ruleset. Triggers on `pull_request`, hardcodes the strict policy. |
| `.github/workflows/security-review.yml` | Reusable workflow. Takes `fail-on-findings` and `ANTHROPIC_API_KEY` as inputs; runs the action. |
| `security/custom-scan-instructions.txt` | Categories the scanner adds on top of standard checks. |
| `security/false-positive-filtering.txt` | Patterns the scanner should treat as expected and not flag. |

## Secrets

`ANTHROPIC_API_KEY` lives as an **org-level secret** with repository
access set to "All repositories." The required workflow forwards it via
`secrets: inherit`, so private and public repos resolve it the same way.

There should be no repo-level copies of `ANTHROPIC_API_KEY` — they are
redundant and create drift.

## Adding a new repo

Nothing. The ruleset targets every repo in the org by default, so new
repos are protected as soon as they're created.

## Bypassing in an emergency

If a fix genuinely cannot wait for the scan (e.g. credential rotation),
an org admin can bypass via the `protect-main` ruleset's bypass list. The
bypass scope is "Allow for pull requests only," so direct-to-`main`
pushes are still blocked. Document the reason in the PR body.

## Handling findings

Findings appear as PR comments from the Claude action, each with a file,
line, severity, and recommendation. To clear the gate:

- **Real issue**: fix the code, push again. The scan re-runs.
- **False positive**: leave a comment with reasoning. If the pattern is
  recurring, update `security/false-positive-filtering.txt` in this repo
  so future runs suppress it.

## Custom categories

In addition to standard security checks (SQL injection, XSS, command
injection, etc.), the org-wide configuration adds:

- **Insecure defaults** — debug mode, wildcard CORS, hardcoded secrets
- **Copy-paste hazards** — credentials in code/comments, string-concatenated queries
- **Missing security fundamentals** — unauthenticated endpoints, no input validation
- **Dependency risks** — unpinned versions, deprecated packages
- **AI tool security** — MCP servers, skills, and plugins without auth, sandboxing, or input validation

## More information

- [Custom scan instructions](../security/custom-scan-instructions.txt)
- [False positive filtering](../security/false-positive-filtering.txt)
- [Upstream action](https://github.com/anthropics/claude-code-security-review)
