# Claude Security Review Setup

Automated security review that runs on every pull request using Claude.

## Quick Setup

From your repo's root directory:

```bash
mkdir -p .github/workflows && curl -o .github/workflows/security-review.yml \
  https://raw.githubusercontent.com/radius-workshop/.github/main/docs/security-review-caller.yml
```

Then commit and push to your default branch.

## What It Does

- Scans PR diffs for security vulnerabilities using Claude
- Posts findings as PR comments
- Optionally fails the build when findings are detected (off by default)
- Uses org-wide custom scan instructions tuned for demo/example apps

## Prerequisites

The workflow needs the `ANTHROPIC_API_KEY` secret. On the Free GitHub plan,
org-level secrets are only available to **public** repos.

For **public repos**: The org secret is automatically available. No extra setup needed.

For **private repos**: The reusable workflow approach won't have access to the
org secret. Either:
- Add `ANTHROPIC_API_KEY` as a repo-level secret
  (Settings > Secrets and variables > Actions > New repository secret)
- Or make the repo public

### Permissions

The caller workflow grants the `pull-requests: write`, `issues: write`, and
`id-token: write` permissions the reusable workflow needs to post findings.
GitHub's default `GITHUB_TOKEN` permissions are read-only on newer orgs/repos,
and a reusable workflow cannot elevate beyond the caller's permissions — so
the `permissions:` block in the caller is load-bearing. If you hand-edit the
caller, don't remove it, or the job will be rejected before it runs.

## Org-Level Setup (One-Time)

These steps only need to be done once by an org admin.

### 1. Add the Anthropic API key as an org secret

Go to Org Settings > Secrets and variables > Actions > New organization secret.
Add `ANTHROPIC_API_KEY` with visibility set to "All repositories".

### 2. Require approval for fork PRs

This prevents untrusted PRs from triggering the security review (which is
not hardened against prompt injection).

```bash
gh auth refresh -h github.com -s admin:org

gh api orgs/radius-workshop/actions/permissions \
  --method PUT \
  --field allowed_actions=all \
  --field enabled_repositories=all \
  --field fork_pull_request_workflows_approval_policy=require-approval-for-all-outside-collaborators
```

## Options

| Parameter | Default | Description |
|-----------|---------|-------------|
| `fail-on-findings` | `false` | Set to `true` to fail the build when security findings are detected |

To enable, add a `with` block in your caller workflow:

```yaml
jobs:
  security-review:
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
    uses: radius-workshop/.github/.github/workflows/security-review.yml@main
    with:
      fail-on-findings: true
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Security Categories

In addition to standard security checks (SQL injection, XSS, command injection, etc.), the org-wide configuration adds checks for:

- **Insecure defaults** — debug mode, wildcard CORS, hardcoded secrets
- **Copy-paste hazards** — credentials in code/comments, string-concatenated queries
- **Missing security fundamentals** — unauthenticated endpoints, no input validation
- **Dependency risks** — unpinned versions, deprecated packages
- **AI tool security** — MCP servers, skills, and plugins without auth, sandboxing, or input validation

## More Information

- [Custom scan instructions](../security/custom-scan-instructions.txt)
- [False positive filtering](../security/false-positive-filtering.txt)
- [Upstream action docs](https://github.com/anthropics/claude-code-security-review)
