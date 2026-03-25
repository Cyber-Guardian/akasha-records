---
topic: CI Pipeline
status: active
touched: 2026-03-05
related:
  - thoughts/shared/plans/2026-02-24-ENG-2133-environment-strategy-deployment-pipeline.md
---

# CI Pipeline

## Current State
11 workflow files in `.github/workflows/`. Key workflows:
- `lint.yml` — ruff, architecture (import-linter), pip-audit, detect-secrets, scope-check, extensions, complexity-report
- `test.yml` — pytest + diff-cover (80% threshold)
- `release.yml` — python-semantic-release versioning
- `_deploy.yml` — reusable parameterized deploy (env/region/role)
- `deploy-testing.yml` / `deploy-staging.yml` — thin callers for `_deploy.yml`
- `ci-failure-linear.yml` — creates/updates Linear issues on CI failures
- `agent-ci-feedback.yml` — posts @mention to Linear on agent PR CI failures (3-round escalation)
- `helm-monitor.yml` — Helm orchestration monitoring
- `semantic-pr.yml` — enforces conventional PR titles + scope enforcement + agent PR metadata
- `mutation.yml` — nightly mutmut mutation testing
- `extension-smoke.yml` — extension validation pipeline

**Local/CI parity gap identified (2026-03-05):** CI runs 7 PR checks but only 3 have automatic local triggers (ruff/import-linter via Claude Code hooks, detect-secrets via .githooks/pre-commit). No pre-push hook exists. Research recommends expanding `.githooks/` with ruff pre-commit + test/import-linter pre-push, plus new `make lint` and `make check` targets with CI calling make targets for parity. See [[2026-03-05-deep-local-vs-ci-testing-divergence-brief|Local vs CI Divergence Research]].

## Key Decisions
- Deploy to testing on every push to main; staging on pre-release tags (`*-v*-rc.*`)
- Per-env artifact buckets, per-env OIDC IAM roles
- `[skip ci]` in PSR release commits prevents deploy loop
- scope-check validates agent PR file changes against plan file
- 80% diff-cover threshold for new code

## Open Questions
- ENG-2133 testing/staging deploys awaiting `terragrunt apply`

## Artifacts
- `.github/workflows/` — all workflow files
- `infrastructure/modules/github-oidc-provider/` and `github-oidc-role/` — CI IAM
