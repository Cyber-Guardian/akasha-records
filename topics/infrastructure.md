---
topic: Infrastructure
status: active
touched: 2026-03-04
related:
  - thoughts/shared/plans/2026-02-24-ENG-2133-environment-strategy-deployment-pipeline.md
  - thoughts/shared/research/2026-02-04-dynamodb-stream-trigger-valkey-queue-processor.md
---

# Infrastructure

## Current State
Dev environment fully provisioned in us-east-1. Testing and staging environments code-complete in Terraform/Terragrunt (ENG-2133), awaiting `terragrunt apply`.

Dev resources: VPC + bastion, DynamoDB (work-queue, resources, dev-idempotency), Valkey (standalone GLIDE mode), Lambda artifacts via content-addressed S3, all 3 service Lambdas deployed.

Module inventory: artifact-bucket, bastion, dynamodb, github-oidc-provider, github-oidc-role, lambda, lambda-layer, valkey, vpc (9 modules).

Toolchain: OpenTofu (not Terraform CLI), Terragrunt for DRY config and orchestration.

## Key Decisions
- OpenTofu + Terragrunt (not bare Terraform)
- Per-env artifact buckets and OIDC IAM roles
- Content-addressed Lambda artifacts (`artifact-versions.json`)
- Standalone GLIDE mode for Valkey (cluster mode broken)

## Open Questions
- DynamoDB stream trigger + event source mapping for VQP not yet applied
- Testing env provisioning pending `terragrunt apply`
- Staging env (us-west-2) needs its own artifact bucket
- `idempotency` table not yet in IaC

## Artifacts
- `infrastructure/modules/` — 9 reusable Terraform modules
- `infrastructure/live/non-prod/` — Terragrunt environment configs
