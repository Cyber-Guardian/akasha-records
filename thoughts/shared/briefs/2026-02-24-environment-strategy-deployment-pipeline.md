# Idea Brief: Environment Strategy & Deployment Pipeline

**Date:** 2026-02-24
**Status:** Shaped -> Planning
**Linear Issue:** ENG-2133

## Problem
Today only a `dev` environment exists and deployment is fully manual (`make build` -> `make upload-sha256` -> `terragrunt run-all apply`). There's no automated path from "merge to main" to "running in a real environment," and no pre-production gate to catch issues before they hit customers. The agent orchestration system (ENG-2128 family) also needs automated deploy triggers to function.

## Constraints
- Single AWS account (`non-prod`) for now — prod will be a separate account later
- Terragrunt structure is clean: `live/<account>/<region>/<env>/` — adding envs is copy + adjust
- No OIDC provider exists in AWS — needs creation for GHA
- VPC CIDRs must be non-overlapping per env (dev = `10.1.0.0/16`)
- State backend (S3 + DynamoDB locks) already supports multi-env via key pattern
- Staging is us-west-2 (different AZs)
- Production is deferred to a separate card (separate account, couple months out)
- `make upload-sha256` already supports `ARTIFACT_BUCKET` and `ENV` params

## Options Considered

### A. One Mega Workflow
Single `deploy.yml` with `if:` conditionals per trigger event (push to main, pre-release, release). All deploy logic in one file.
- Gains: DRY, single file to maintain
- Costs: Complex conditionals, hard to read/debug, hard to independently trigger
- Complexity: Medium

### B. Separate Workflows Per Trigger
Individual `deploy-testing.yml`, `deploy-staging.yml`, `deploy-prod.yml` each self-contained.
- Gains: Clear separation, easy to understand, easy to debug independently
- Costs: Duplicates build/upload/terragrunt steps across files
- Complexity: Low (but wasteful)

### C. Reusable Workflow + Thin Callers
One `_deploy.yml` reusable workflow parameterized by (environment, region, OIDC role ARN, artifact bucket). Thin caller workflows per trigger.
- Gains: DRY where it matters (deploy logic), clear where it matters (triggers), each caller independently re-runnable, OIDC role ARN as input works naturally with per-env roles
- Costs: Slightly more setup than option B
- Complexity: Medium

## Chosen Approach
**Reusable Workflow + Thin Callers (Option C)** — standard GHA pattern for multi-environment deployment. DRY core logic with clear per-trigger entry points. Per-env OIDC roles pass naturally as workflow inputs.

## Key Decisions Made During Shaping
- **Testing env:** same infra as dev, us-east-1, VPC CIDR `10.2.0.0/16`, no bastion (CI-only)
- **Staging env:** same infra as dev, us-west-2, no bastion initially (add later if needed)
- **OIDC roles:** per-environment (least privilege)
- **Bastions:** neither Testing nor Staging gets one — bastion is for manual Valkey debugging (dev-only concern)
- **Prod:** separate Linear card, couple months out, will be a dedicated AWS account
- **Deployment strategy:** Strategy 1 (approval per release) — merge freely, explicit release creation triggers prod deploy

## Key Context Discovered During Shaping
- Bastion module is purely SSM + valkey-cli for manual debugging — not needed in automated envs
- `artifact-versions.json` is read by Terragrunt units via `jsondecode(file(...))` — each env gets its own copy
- Modules are fully parameterized by environment — VPC, Lambda, DynamoDB, Valkey all take `environment` variable
- `release.yml` already outputs `released`, `version`, `tag` per project — deploy workflows can consume these
- State backend key pattern `<account>/<region>/<env>/<resource>/terraform.tfstate` already supports multi-env
- Staging in us-west-2 needs a new `region.hcl` with `us-west-2` AZs

## Scope for ENG-2133 (adjusted)
1. OIDC provider + per-env IAM roles (new Terraform module)
2. Testing environment provisioned (`infrastructure/live/non-prod/us-east-1/testing/`)
3. Staging environment provisioned (`infrastructure/live/non-prod/us-west-2/staging/`)
4. Reusable `_deploy.yml` workflow + `deploy-testing.yml` (on merge to main) + `deploy-staging.yml` (on pre-release)
5. Environment hierarchy + service routing + ephemeral design documented
6. **NOT** Production (separate card)

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-02-24-environment-strategy-deployment-pipeline.md`
