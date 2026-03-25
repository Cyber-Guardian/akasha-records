---
date: 2026-02-03
source_research: memory-bank/thoughts/shared/research/2026-02-03-aws-saas-networking-patterns.md
last_generated: 2026-02-03T18:56:51.475508+00:00
---

# Research Brief: 2026-02-03-aws-saas-networking-patterns

## TL;DR

## Context
While planning the entity-discovery-trigger Lambda migration, we discovered the dev VPC has NAT Gateway disabled. This Lambda needs to call Microsoft Graph API (external internet) but was configured to run in VPC private subnets. This prompted research into AWS networking patterns for SaaS applications.
## Key Insight: Lambda VPC Trade-offs
**Lambda functions outside VPC get free internet access** via AWS-managed NAT. Once you attach a Lambda to a VPC, you lose this and must explicitly configure internet access.
| Lambda Needs | Recommended Approach | Cost |
|--------------|---------------------|------|
| Only external APIs (Graph API, Stripe, etc.) | **Don't attach to VPC** | Free |
| Only S3 + DynamoDB | VPC + Gateway Endpoints | Free |
| VPC resources (RDS, ElastiCache, Valkey) | VPC required | Free (no internet) |
| VPC resources + external APIs | NAT Gateway **or** split Lambdas | $32+/month |

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-03-aws-saas-networking-patterns.md`
