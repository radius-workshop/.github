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
- Fails the workflow if vulnerabilities are found
- Uses org-wide custom scan instructions tuned for demo/example apps

## Prerequisites

The `ANTHROPIC_API_KEY` org-level secret must be configured. If PRs aren't triggering the review, contact an org admin to verify the secret is set.

## What Gets Scanned

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
