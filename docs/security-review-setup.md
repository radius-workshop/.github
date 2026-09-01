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

## Creating a new repo

Create every new repo **from the org template** so its default branch is
initialized at creation *and* it ships with the security-gate guidance every
AI coding agent reads:

```bash
gh repo create radius-workshop/<name> \
  --template radius-workshop/repo-template \
  --clone \
  --private            # or --public — one is required
```

`--template radius-workshop/repo-template` seeds commit zero from the template
repo, which carries `README.md`, `CLAUDE.md`, `AGENTS.md`, and `GEMINI.md`. So
whichever agent a developer points at the repo — Claude Code (`CLAUDE.md`),
Codex (`AGENTS.md`), Gemini (`GEMINI.md`) — reads the same rules from the repo's
first commit. Those files entered `repo-template` through a scanned PR, so
commit zero is a **reviewed baseline** rather than arbitrary unscanned code.
`--clone` checks the new repo out locally so you can start working immediately.

Then do all real work through pull requests, as normal.

### If a push to the default branch is rejected

Putting your changes on a feature branch, pushing that, and opening a PR
is the **normal workflow** — the PR gets scanned and
merges once it passes. A `GH013 ... Required workflow 'Claude Security
Review (required)' is not satisfied` rejection on a direct push to the
default branch just means "open a PR instead."

Do not use any strategy that allows bypassing the security check and getting code onto the default branch
**without a scanned PR**. This includes pushing it to a side branch and then 
renaming that branch onto the default, repointing the default branch at it, or 
an admin bypass merge. Each lands unscanned code on a protected branch while 
the ruleset still reports every rule active — the repo *looks* governed, but that
code never was scanned. Do not do this.

### The one case branch-and-PR can't cover: commit zero

The ruleset protects a default branch that already exists. A brand-new
**empty** repo has no default branch yet, and a PR needs an existing base
branch to target — so the very first commit cannot go through a PR at all
(chicken-and-egg). This is exactly the gap the rename/repoint trick
abuses to smuggle a whole initial codebase in unscanned.

The fix is to never let commit zero carry unreviewed code: create the repo
from the template (above) so the first commit is a **reviewed** server-side
baseline, the default branch exists and is protected from that point on, and
every real change after it arrives through a scanned PR.

> This holds for any tool driving the repo — a person, or an AI coding
> agent (Claude Code, Codex, Gemini, …). A rejected push is a
> stop-and-report signal: open a PR, or (for a new repo) recreate it from
> the template — never route the code onto the default branch some
> other way.

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

## Troubleshooting

The failure mode that matters most here is **silent**: the workflow files, the
secret, and the docs can all look healthy while no scan is actually running.
Diagnose in the order below.

### No PR anywhere is getting scanned (org-wide)

If *no* PR in any repo is getting a `Claude Security Review (required)` check
(empty status checks, `mergeStateStatus: CLEAN` with no Claude check), the cause
is almost always the **ruleset**, not the workflow files. Enforcement hinges
entirely on the required-workflow rule inside `protect-main`, and that rule has
been silently dropped before — which turned scanning off org-wide while every
workflow file still looked fine.

Check the ruleset's `rules` array **first**:

```bash
gh api /orgs/radius-workshop/rulesets/16146745 --jq '.rules[].type'
```

You should see `deletion`, `non_fast_forward`, `pull_request`, **and**
`workflows`. If `workflows` is missing, that is the outage — re-add the
required-workflow rule pointing at
`.github/workflows/security-review-required.yml` @ `refs/heads/main`. To see
when and by whom a rule changed:

```bash
gh api /orgs/radius-workshop/rulesets/16146745/history
```

The per-repo caller workflows were deliberately removed — the ruleset is the
**sole** trigger, so nothing in an individual repo will start a scan on its own.
That is by design, and it is why the ruleset is the first thing to check.

### One PR has no scan, or a stale result

A config change does **not** retroactively re-scan already-open PRs — the scan
runs on `pull_request` events. An existing PR keeps its last result (or absence
of one) until a new event fires. Re-trigger it:

- **Normal PR:** push a commit, or close and reopen the PR.
- **Dependabot PR:** comment `@dependabot recreate` (force-pushes a fresh commit
  and fires the event; `rebase` is a no-op if the branch is already current).

### The scan check appears but merge isn't blocked

A check showing on a PR is not the same as it being *required*. "Required"
enforcement comes from the ruleset rule, which matches the workflow by name. If
a green or red check shows but the PR merges anyway, confirm the `workflows` rule
is present (above) and that it references the correct workflow file and ref.

### A new repo comes up empty after `--template` create

If `gh repo create --template` exits successfully but leaves the repo with no
`main` and no files, the `protect-main` ruleset is rejecting commit zero. The
`--template` create writes the initial commit **server-side, at creation**, and
the require-workflows rule must have `do_not_enforce_on_create: true` to allow
it. When that field is `false`, GitHub rejects the initial commit —
`Required workflow 'Claude Security Review (required)' is not satisfied` — and
the repo is left empty.

A required workflow can't run on a bare ref creation, so enforcing it on create
blocks *all* bootstrapping while adding no scan. **This field must stay `true`.**
Check it first whenever a create produces an empty repo:

```bash
gh api /orgs/radius-workshop/rulesets/16146745 \
  --jq '.rules[] | select(.type=="workflows") | .parameters.do_not_enforce_on_create'
```

### Expected non-failures (not bugs)

- **PRs in `radius-workshop/.github` itself are not scanned.** A required
  workflow is exempt on the repo that *stores* it (source-repo exemption), so
  this repo's own PRs show all-green with no Claude check and are not blocked by
  the gate. That is GitHub behavior, not a broken gate — but it means changes to
  the scan config here land without being scanned, so review them with
  corresponding care.

### Where to change scan behavior

All scan behavior lives in **this repo** (`radius-workshop/.github`): workflow
logic under `.github/workflows/`, scanner tuning under `security/`. Never add a
scan workflow to an individual repo to "fix" it — that reintroduces the per-repo
drift the central setup was built to eliminate.

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
