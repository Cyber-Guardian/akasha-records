# Idea Brief: Unified Monorepo — Single Namespace + Terragrunt Convergence + CD

**Date:** 2026-03-12
**Status:** Shaped -> Planning

## Problem

The triage agent has no continuous deployment — deploys are manual `make deploy` from a laptop. The root cause is three architectural divergences: a separate `tooling` namespace that prevents `uv build --wheel` from working, standalone OpenTofu instead of Terragrunt for IaC, and no GitHub Actions workflow for deployment. Fixing CD requires unifying all three.

## Constraints

- `hatch-polylith-bricks` is single-namespace per wheel — the `tooling` namespace must merge into `filescience` for unified builds
- The triage agent's cross-namespace dependency on `filescience.telemetry` is currently a deferred import invisible to import-linter — flattening fixes this
- Triage agent has 4 Lambdas + SQS FIFO + DynamoDB + API Gateway — more complex than core services (1 Lambda each)
- Memory-bank sync is logically independent from code deploy — changes on every CI commit
- Terragrunt v0.78+ Stacks is GA but implicit stacks (directory-of-units) is sufficient at current scale
- `tool_` prefix on tooling bricks preserves cognitive signaling after namespace merge

## Options Considered

### Automate-in-Place
Keep standalone Terraform, add a GHA workflow that runs existing `make deploy` in CI. Keep `pip install -t` build.
- Gains: Fastest to ship, no migration risk
- Costs: Two build systems, two IaC tools, no multi-env support
- Complexity: Low

### S3-Mediated Code Split
Keep standalone Terraform for infra shell, switch Lambda code updates to S3 content-addressed pattern + `aws lambda update-function-code`.
- Gains: Code deploys consistent with core services, infra stays simple
- Costs: Terraform state drift on source_code_hash, two mechanisms updating same Lambda
- Complexity: Medium

### Unified Monorepo (chosen)
Flatten namespace, converge IaC onto Terragrunt, unify builds on `uv build --wheel`, add CD via path-scoped workflows.
- Gains: One build system, one IaC tool, CD for everything, `poly` sees all bricks
- Costs: Largest lift, Terraform state migration, namespace boundary weakens (mitigated by guards)
- Complexity: High

## Chosen Approach

**Unified Monorepo** — the other options solve symptoms (no CD) while cementing the root cause (architectural divergence). This is the right time to converge because the triage agent is new enough that migration is bounded, and the guards (import-linter contracts, pre-commit hooks, brick allowlist script, separate CI test jobs) adequately compensate for the lost namespace isolation.

## Key Context Discovered During Shaping

- `hatch-polylith-bricks` path mapping doesn't validate namespace roots — cross-namespace mapping works mechanically but is undocumented (`tools/pyproject.toml:34`)
- The `tooling -> filescience` import-linter contract already has a hole: deferred imports in `handler.py:211,224` are invisible to static analysis. Flattening actually fixes this.
- Terragrunt v0.78+ Stacks (GA May 2025) introduces catalog/live split pattern — not needed now but is the upgrade path
- The canonical Terragrunt pattern for mixed service types: `services/` vs `tooling/` category directories within each environment, separate state files per component
- Every production Python Polylith example uses a single namespace — the dual-namespace pattern was a workaround, not idiomatic
- `poly` CLI only discovers bricks under the single `workspace.toml` namespace — tooling bricks are currently invisible to `poly check`, `poly deps`, `poly info`
- Community consensus: internal tooling gets its own CD workflow (path-scoped), separate from production service CD
- Guards to compensate for lost namespace isolation: pre-commit import-linter hook, forbidden-import contracts between product/tooling bricks, brick allowlist CI script, separate CI test jobs per category

## Structural Decisions

### Namespace
- Merge `tooling` -> `filescience` with `tool_` prefix
- `tools/components/tooling/config` -> `components/filescience/tool_config`
- `tools/components/tooling/durable_harness` -> `components/filescience/tool_durable_harness`
- `tools/components/tooling/linear_client` -> `components/filescience/tool_linear_client`
- `tools/components/tooling/slack_client` -> `components/filescience/tool_slack_client`
- `tools/bases/tooling/triage_agent` -> `bases/filescience/tool_triage_agent`
- `tools/bases/tooling/helm_cli` -> `bases/filescience/tool_helm_cli`

### Projects
- Flat `projects/` — `tools/projects/triage-agent` -> `projects/triage-agent`
- `tools/projects/helm` -> `projects/helm`
- Each project uses `hatch-polylith-bricks` with `[tool.polylith.bricks]` table

### IaC
- Migrate triage agent Terraform into `infrastructure/live/non-prod/<region>/<env>/tooling/triage-agent/`
- Split into separate Terragrunt units: `dynamodb/`, `sqs/`, `lambdas/`, `api-gateway/`
- Create new shared modules as needed (SQS FIFO, API Gateway) or inline
- S3 content-addressed artifacts via `artifact-versions.json` (same as core services)
- State migration from standalone backend to Terragrunt-managed backend

### CD Workflows
- `deploy-testing.yml` — update path filters to include tooling paths
- `sync-memory-bank.yml` — new workflow, triggered by `memory-bank/**` changes
- Separate CI test jobs: `test-product` and `test-tooling`

### Guards
- Pre-commit hook: import-linter
- Import-linter forbidden contract: product bases cannot import `tool_*` bricks
- `scripts/check_brick_allowlists.py`: validates `[tool.polylith.bricks]` tables in product projects
- PostToolUse hook: already runs import-linter on every .py write (Claude Code)

## Next Step

Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-12-unified-monorepo-cd.md`
