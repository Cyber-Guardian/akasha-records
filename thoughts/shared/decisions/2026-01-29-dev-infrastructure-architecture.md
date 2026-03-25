---
date: 2026-01-29T18:30:00-05:00
author: claude
git_commit: pending
branch: main
repository: filescience
topic: "Dev Infrastructure Architecture (OpenTofu/Terragrunt)"
tags: [decisions, infrastructure, aws, opentofu, terragrunt]
status: complete
last_updated: 2026-01-29
last_updated_by: claude
---

# Decision: Dev Infrastructure Architecture (OpenTofu/Terragrunt)

## Context

- FileScience backend requires infrastructure for work queue (DynamoDB), caching (Valkey), and networking (VPC)
- Greenfield setup with no existing IaC
- Need cost-effective dev environment with path to staging/prod

## Decision

Deploy dev environment in AWS us-east-1 using:
- **IaC**: Terragrunt + OpenTofu (not Terraform)
- **State**: CloudFormation-bootstrapped S3 + DynamoDB lock + KMS
- **VPC**: Custom VPC (10.1.0.0/16), no NAT gateway, bastion for private subnet access
- **DynamoDB**: `work-queue` table with pk/sk keys, GSI on throttle_context_id/next_visible_time, TTL on expires_time
- **Valkey**: Engine 8.2, cluster mode enabled, cache.t4g.micro, 1 shard + 1 replica

## Options Considered

### IaC Tool
- A: **Terraform** - Mature, large community, but licensing concerns (BSL)
- B: **OpenTofu** - OSS fork, MPL-2.0 license, compatible providers (Chosen)
- C: **Pulumi** - Code-first, but different paradigm and learning curve

### State Management
- A: **CloudFormation bootstrap + S3/DynamoDB** - Simple, self-managed (Chosen)
- B: **Terraform Cloud** - Managed but vendor lock-in
- C: **Local state** - Not viable for team collaboration

### NAT Gateway
- A: **NAT Gateway** - ~$32/month, enables private subnet internet access
- B: **SSM Bastion only** - ~$3/month, sufficient for dev, manual access to Valkey (Chosen)

## Tradeoffs

**Gains**:
- OSS tooling (no licensing concerns)
- Low dev environment cost (~$10-15/month total)
- Cluster mode Valkey ready for prod patterns

**Gives up**:
- No automatic internet access from private subnets (requires bastion hop)
- Single replica adds ~$3/month vs no-replica option

## Consequences

- Bastion required for Valkey access during development
- Must use SSM port forwarding for local dev connections
- Staging/prod will need NAT or VPC endpoints for Lambda internet access

## References

- Plan: `memory-bank/thoughts/shared/plans/2026-01-29-opentofu-infrastructure-setup.md`
- Research: `memory-bank/thoughts/shared/research/2026-01-29-infrastructure-architecture-decisions.md`
- Research: `memory-bank/thoughts/shared/research/2026-01-29-opentofu-dynamodb-elasticache.md`
