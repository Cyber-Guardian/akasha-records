---
date: 2026-01-29T14:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-01-29-infrastructure-architecture-decisions.md
last_generated: 2026-01-29T16:43:18.929626+00:00
---

# Research Brief: 2026-01-29-infrastructure-architecture-decisions

## TL;DR

**Date**: 2026-01-29T14:30:00-05:00
**Researcher**: Claude
**Git Commit**: 491e7136fbd1b0adc2ca5aaff20de734ceecdbed
**Branch**: main
**Repository**: filescience
## Research Questions
1. VPC strategy: default vs custom for multi-account/multi-region
2. State backend: optimal patterns for multi-account/multi-region
3. ElastiCache: Valkey cluster mode for functions support
4. IaC structure: directory layout in monorepo

## Key facts / constraints

- **Dev**: us-east-2, AWS Account A
- **Staging**: us-east-1, AWS Account A (same as dev)
- **Prod**: us-east-1, AWS Account B (separate account)
- **Resources**: 1 DynamoDB table with GSIs, Valkey cluster mode

---

## Key recommendations

- **VPC**: Custom VPCs per environment (not default)
- **IaC Tool**: Terragrunt + OpenTofu
- **State Location**: Central admin account with cross-account access
- **State Structure**: One file per environment-region combo
- **Valkey**: Cluster mode enabled with `num_node_groups`
- **Directory**: `infrastructure/` at repo root, account > region > env hierarchy

## Open questions

1. **DynamoDB Table Design**: What are the specific access patterns for the table and GSIs?
2. **Valkey Use Case**: Caching, sessions, pub/sub, or all?
3. **CI/CD**: GitHub Actions or other platform for deployments?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-01-29-infrastructure-architecture-decisions.md`
