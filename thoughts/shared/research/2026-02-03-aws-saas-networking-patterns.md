---
date: 2026-02-03
researcher: Claude
topic: "AWS SaaS Networking Patterns and Best Practices"
tags: [research, aws, networking, vpc, lambda, saas, multi-tenant]
status: complete
trigger: "entity-discovery-trigger needed internet access from VPC - prompted broader research"
---

# Research: AWS SaaS Networking Patterns and Best Practices

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

## VPC Endpoints

### Gateway Endpoints (FREE)
- **Supported services**: S3, DynamoDB only
- **How it works**: Route table entry directs traffic to service
- **Recommendation**: Always enable for S3/DynamoDB when using VPC

### Interface Endpoints (~$7/month per AZ per service)
- **Supported services**: 100+ (SSM, Secrets Manager, SQS, SNS, Lambda, etc.)
- **How it works**: Creates ENI in your subnet with private IP
- **Cost warning**: 3-AZ setup = ~$21/month per service
- **Use when**: Lambda in VPC needs AWS services beyond S3/DynamoDB

## NAT Gateway

- **Cost**: ~$32/month base + $0.045/GB data processed
- **Use when**: VPC resources need internet access (can't be avoided)
- **Alternatives**:
  - Split into VPC and non-VPC Lambdas
  - NAT Instance (cheaper but self-managed)
  - IPv6 with Egress-Only Internet Gateway

## Multi-Tenant Network Isolation Models

| Model | Isolation | Cost | Complexity | When to Use |
|-------|-----------|------|------------|-------------|
| **VPC-per-Tenant** | Strongest | Highest | High | Compliance, enterprise |
| **Subnet-per-Tenant** | Medium | Medium | High | <50 tenants |
| **Pool Model** | Lowest | Lowest | Low | Most SaaS |
| **Hybrid** | Mixed | Medium | Medium | High-value tenants siloed |

### FileScience Context
We use **Pool Model** - single VPC, tenants isolated at application layer (tenant_id in data). This is appropriate for:
- Cost efficiency
- Simpler operations
- Application-level isolation sufficient for backup SaaS

## Recommended Pattern for FileScience

```
Lambda Classification:
├── VPC Lambdas (need Valkey/internal resources)
│   └── valkey-queue-processor ✓
│
└── Non-VPC Lambdas (need external APIs)
    └── entity-discovery-trigger (Graph API)
    └── Future: any Lambda calling external services
```

**Rationale**:
- entity-discovery-trigger needs Graph API (internet) but NOT Valkey
- DynamoDB accessible from both VPC (Gateway Endpoint) and non-VPC (IAM)
- SSM Parameter Store accessible from non-VPC (IAM)
- No cost for non-VPC Lambda internet access

## Current Infrastructure State

```
dev VPC:
├── NAT Gateway: DISABLED
├── Internet Gateway: ENABLED (public subnets only)
├── Gateway Endpoints:
│   ├── DynamoDB ✓ (free)
│   └── S3 ✓ (free)
├── Interface Endpoints: NONE
└── Private subnets: NO internet access
```

## Decision: entity-discovery-trigger

**Decision**: Run entity-discovery-trigger **outside VPC**

**Reasoning**:
- Needs: Graph API (internet), DynamoDB, SSM
- Doesn't need: Valkey or other VPC resources
- All AWS services accessible via IAM from non-VPC Lambda
- No cost for internet access

**Alternative considered**: Enable NAT Gateway
- Rejected: $32+/month ongoing cost for something achievable for free
- Would only choose this if Lambda needed both VPC resources AND internet

## Resources

### Essential Reading
- [Building a Scalable and Secure Multi-VPC AWS Network Infrastructure](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure
/welcome.html)
- [SaaS Tenant Isolation Strategies (PDF)](https://d1.awsstatic.com/whitepapers/saas-tenant-isolation-strategies.pdf)
- [Guidance for Multi-Tenant Architectures on AWS](https://aws.amazon.com/solutions/guidance/multi-tenant-architectures-on-aws/)

### Lambda + VPC
- [Three ways to use AWS services from Lambda in VPC](https://www.alexdebrie.com/posts/aws-lambda-vpc/) - Alex DeBrie
- [Lambda VPC + Internet without NAT Gateway](https://serverlessfirst.com/lambda-vpc-internet-access-no-nat-gateway/)
- [Mixing VPC and Non-VPC Lambda Functions](https://www.jeremydaly.com/mixing-vpc-and-non-vpc-lambda-functions-for-higher-performing-microservices/)
- [You Don't Need a NAT: A $30 to $0 AWS story](https://medium.com/@nikeshkazi/you-dont-need-a-nat-a-30-to-0-aws-story-a80c2f65acd1)
- [AWS Lambda VPC Configuration](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html)

### PrivateLink (for future reference)
- [Building SaaS Services with PrivateLink](https://aws.amazon.com/blogs/architecture/building-saas-services-for-aws-customers-with-privatelink/)
- [AWS PrivateLink Concepts](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [VPC Lattice for Multi-Tenant SaaS](https://aws.amazon.com/blogs/networking-and-content-delivery/secure-customer-resource-access-in-multi-tenant-saas-with-amazon-vpc-lattice/)

### Multi-Tenant Architecture
- [Full Stack Isolation Strategies](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/full-stack-isolation.html)
- [Architectural Design Patterns for Multi-Tenancy](https://www.nagarro.com/en/blog/architectural-design-patterns-aws-multi-tenancy)

## Action Items

1. **Update entity-discovery-trigger plan**: Remove `vpc_config` from Terragrunt
2. **Document Lambda classification**: Which Lambdas need VPC vs not
3. **Future consideration**: If we ever need VPC + internet, evaluate:
   - NAT Gateway (simplest, costs money)
   - Proxy Lambda pattern (complex, free)
   - Interface endpoints for specific AWS services

## Related Documents

- Plan: `memory-bank/thoughts/shared/plans/2026-02-03-entity-discovery-trigger-migration.md`
- Research: `memory-bank/thoughts/shared/research/2026-02-03-entity-discovery-trigger-migration.md`
