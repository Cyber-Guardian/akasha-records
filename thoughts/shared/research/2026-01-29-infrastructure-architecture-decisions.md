---
date: 2026-01-29T14:30:00-05:00
researcher: Claude
git_commit: 491e7136fbd1b0adc2ca5aaff20de734ceecdbed
branch: main
repository: filescience
topic: "Infrastructure Architecture Decisions: VPC, State Backend, Valkey, IaC Structure"
tags: [research, infrastructure, opentofu, terragrunt, vpc, dynamodb, elasticache, valkey, multi-account, aws]
status: complete
last_updated: 2026-01-29
last_updated_by: Claude
---

# Research: Infrastructure Architecture Decisions

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

## Environment Requirements

- **Dev**: us-east-2, AWS Account A
- **Staging**: us-east-1, AWS Account A (same as dev)
- **Prod**: us-east-1, AWS Account B (separate account)
- **Resources**: 1 DynamoDB table with GSIs, Valkey cluster mode

---

## Summary of Recommendations

| Decision | Recommendation |
|----------|----------------|
| VPC | Custom VPCs per environment (not default) |
| IaC Tool | Terragrunt + OpenTofu |
| State Location | Central admin account with cross-account access |
| State Structure | One file per environment-region combo |
| Valkey | Cluster mode enabled with `num_node_groups` |
| Directory | `infrastructure/` at repo root, account > region > env hierarchy |

---

## 1. VPC Strategy

### Decision: Use Custom VPCs

**Do NOT use default VPC for production workloads.**

**Reasons:**
- Default VPC lacks security best practices
- No workload isolation between environments
- Cannot customize network architecture for multi-tier apps
- Compliance requirements often mandate custom VPCs

### Recommended Architecture

```
Dev Account (123456789012):
  └── us-east-1: Dev VPC (10.1.0.0/16)

Non-Prod Account (123456789012):
  └── us-west-2: Staging VPC (10.2.0.0/16)

Prod Account (987654321098):
  └── us-east-1: Prod VPC (10.0.0.0/16)
```

### CIDR Planning (No Overlaps)

| Environment | CIDR | Available IPs |
|-------------|------|---------------|
| Prod | 10.0.0.0/16 | 65,536 |
| Dev | 10.1.0.0/16 | 65,536 |
| Staging | 10.2.0.0/16 | 65,536 |

### Per-VPC Components

- 2+ private subnets (multi-AZ for HA)
- 2+ public subnets (for load balancers if needed)
- Internet Gateway
- NAT Gateway (1 for dev/staging, 2 for prod HA)
- **DynamoDB VPC Gateway Endpoint (FREE)**
- **S3 VPC Gateway Endpoint (FREE)**
- ElastiCache subnet group

### OpenTofu Provider Configuration (Multi-Account)

```hcl
# Provider aliases for each environment
provider "aws" {
  region = "us-east-1"
  alias  = "dev"
}

provider "aws" {
  region = "us-west-2"
  alias  = "staging"
}

provider "aws" {
  region = "us-east-1"
  alias  = "prod"

  assume_role {
    role_arn     = "arn:aws:iam::987654321098:role/OpenTofuRole"
    session_name = "OpenTofuSession"
  }
}
```

### VPC Peering vs Transit Gateway

**Start with VPC Peering** for 3 VPCs:
- Free (only data transfer costs)
- Simpler setup
- Sufficient for small scale

**Migrate to Transit Gateway** when:
- 5+ VPCs
- Need centralized egress
- Multi-region disaster recovery

---

## 2. State Backend Strategy

### Decision: Central Administrative Account

Store all state in a **dedicated admin account** (or the dev/staging account), not in the account where resources are deployed.

### Architecture

```
ADMIN ACCOUNT (or Dev Account)
├── S3: filescience-tofu-state
│   └── environments/
│       ├── dev/us-east-1/terraform.tfstate
│       ├── staging/us-west-2/terraform.tfstate
│       └── prod/us-east-1/terraform.tfstate
├── DynamoDB: filescience-tofu-locks
└── KMS: State encryption key
```

### State File Structure

**One state file per environment-region combination** (not per resource type):

```
s3://filescience-tofu-state/
└── environments/
    ├── dev/
    │   └── us-east-1/
    │       └── terraform.tfstate
    ├── staging/
    │   └── us-west-2/
    │       └── terraform.tfstate
    └── prod/
        └── us-east-1/
            └── terraform.tfstate
```

### Bootstrap Process (Chicken-Egg Solution)

**Use CloudFormation to bootstrap** the state backend:

1. Deploy CloudFormation stack with S3 bucket + DynamoDB table + KMS key
2. Create IAM roles in target accounts
3. Configure OpenTofu/Terragrunt to use the backend

This avoids the "IaC can't create its own state storage" problem.

### Partial Backend Configuration

Use `.tfbackend` files for each environment:

```hcl
# backend-configs/dev.s3.tfbackend
bucket         = "filescience-tofu-state"
key            = "environments/dev/us-east-1/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "filescience-tofu-locks"
encrypt        = true

assume_role = {
  role_arn    = "arn:aws:iam::123456789012:role/TofuDeploymentRole"
  external_id = "filescience-tofu"
}
```

**Usage:**
```bash
tofu init -backend-config=backend-configs/dev.s3.tfbackend
tofu apply -var-file=environments/dev.tfvars
```

---

## 3. ElastiCache Valkey Cluster Mode

### Decision: Valkey (not Redis) with Cluster Mode Enabled

**Why Valkey over Redis:**
- 33% cheaper serverless, 20% cheaper node-based
- BSD license (fully open-source, no licensing issues)
- AWS's strategic direction for ElastiCache
- 100% API compatible with Redis

### Cluster Mode Configuration

**CRITICAL**: Use `num_node_groups` + `replicas_per_node_group`, NOT `num_cache_clusters`:

```hcl
resource "aws_elasticache_replication_group" "valkey" {
  replication_group_id = "filescience-valkey"
  description          = "Valkey cluster mode enabled"

  engine               = "valkey"           # NOT "redis"
  engine_version       = "7.2"
  node_type            = "cache.m7g.large"

  # MUST use .cluster.on parameter group
  parameter_group_name = "default.valkey7.cluster.on"

  # Cluster mode settings
  num_node_groups         = 3    # Number of shards
  replicas_per_node_group = 2    # Replicas per shard (= 9 total nodes)

  # REQUIRED for cluster mode
  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Security
  transit_encryption_enabled = true
  at_rest_encryption_enabled = true
  auth_token                 = random_password.valkey_auth.result

  # Networking
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.elasticache.id]
}
```

### Functions/Scripting Support

**Lua scripting IS supported** in cluster mode, but with a constraint:

- All keys accessed in a script **must hash to the same slot**
- Use **hash tags** in key names: `{tenant_id}:key_name`
- Example: `{firm123}:backup`, `{firm123}:status` → same slot

### Environment Sizing

| Environment | Node Type | Shards | Replicas | Total Nodes |
|-------------|-----------|--------|----------|-------------|
| Dev | cache.t4g.micro | 1 | 0 | 1 |
| Staging | cache.t4g.small | 2 | 1 | 4 |
| Prod | cache.m7g.large | 3 | 2 | 9 |

---

## 4. IaC Directory Structure

### Decision: Terragrunt with Account > Region > Env Hierarchy

**Why Terragrunt:**
- DRY configuration across 3 environments
- Automatic backend state management
- Hierarchical config inheritance
- 42% performance improvement in v0.80+

### Recommended Structure

```
filescience/                           # Root monorepo
├── components/                        # Python Polylith components
├── bases/
├── projects/
├── infrastructure/                    # IaC root
│   ├── modules/                       # Shared reusable modules
│   │   ├── dynamodb-table/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── elasticache-valkey/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── vpc/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   │
│   ├── bootstrap/                     # CloudFormation for state backend
│   │   ├── bootstrap-backend.yaml
│   │   └── bootstrap.sh
│   │
│   └── live/                          # Live environment configs
│       ├── terragrunt.hcl             # Root: backend, provider generation
│       │
│       ├── non-prod/                  # AWS Account A
│       │   ├── account.hcl            # account_id, aws_profile
│       │   │
│       │   ├── us-east-1/
│       │   │   ├── region.hcl         # aws_region = "us-east-1"
│       │   │   └── dev/
│       │   │       ├── env.hcl        # environment = "dev"
│       │   │       ├── vpc/
│       │   │       │   └── terragrunt.hcl
│       │   │       ├── dynamodb/
│       │   │       │   └── terragrunt.hcl
│       │   │       └── elasticache/
│       │   │           └── terragrunt.hcl
│       │   │
│       │   └── us-west-2/
│       │       ├── region.hcl         # aws_region = "us-west-2"
│       │       └── staging/
│       │           ├── env.hcl        # environment = "staging"
│       │           ├── vpc/
│       │           │   └── terragrunt.hcl
│       │           ├── dynamodb/
│       │           │   └── terragrunt.hcl
│       │           └── elasticache/
│       │               └── terragrunt.hcl
│       │
│       └── prod/                      # AWS Account B
│           ├── account.hcl            # Different account_id
│           └── us-east-1/
│               ├── region.hcl
│               └── prod/
│                   ├── env.hcl        # environment = "prod"
│                   ├── vpc/
│                   │   └── terragrunt.hcl
│                   ├── dynamodb/
│                   │   └── terragrunt.hcl
│                   └── elasticache/
│                       └── terragrunt.hcl
│
├── pyproject.toml
└── workspace.toml
```

### Configuration Hierarchy

Each level defines only what's unique to that level:

**Root `terragrunt.hcl`**: Backend config, provider generation
```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "filescience-tofu-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "filescience-tofu-locks"
  }
}
```

**`account.hcl`**: Account-specific vars
```hcl
locals {
  account_id   = "123456789012"
  account_name = "non-prod"
}
```

**`region.hcl`**: Region-specific vars
```hcl
locals {
  aws_region = "us-east-1"
}
```

**`env.hcl`**: Environment-specific vars
```hcl
locals {
  environment = "dev"
}
```

**Resource `terragrunt.hcl`**: Resource-specific inputs
```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../../../modules//dynamodb-table"
}

inputs = {
  table_name   = "filescience-dev-data"
  billing_mode = "PAY_PER_REQUEST"
}
```

---

## Implementation Checklist

### Phase 1: Bootstrap State Backend
- [ ] Create CloudFormation template for S3 + DynamoDB + KMS
- [ ] Deploy to admin/non-prod account
- [ ] Create IAM roles in both accounts

### Phase 2: Create Modules
- [ ] VPC module with subnets, NAT, endpoints
- [ ] DynamoDB module with GSI support
- [ ] ElastiCache Valkey module with cluster mode

### Phase 3: Deploy Dev
- [ ] Configure dev environment
- [ ] Deploy VPC
- [ ] Deploy DynamoDB
- [ ] Deploy Valkey
- [ ] Test connectivity

### Phase 4: Deploy Staging
- [ ] Configure staging (different region)
- [ ] Deploy all resources
- [ ] Validate state isolation

### Phase 5: Deploy Prod
- [ ] Configure prod (different account)
- [ ] Deploy all resources
- [ ] Set up monitoring/alerts

---

## Cost Estimates (Monthly)

| Component | Dev | Staging | Prod |
|-----------|-----|---------|------|
| VPC (NAT Gateway) | $32 | $32 | $64 (HA) |
| VPC Endpoints | FREE | FREE | FREE |
| DynamoDB (on-demand) | $5-20 | $5-20 | $50-200 |
| Valkey | $12 | $48 | $500+ |
| **Total** | ~$50 | ~$100 | ~$600+ |

---

## Sources

### VPC Strategy
- [AWS Building Scalable Multi-VPC Infrastructure](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html)
- [AWS VPC Peering vs Transit Gateway](https://ably.com/blog/aws-vpc-peering-vs-transit-gateway-and-beyond)

### State Backend
- [OpenTofu S3 Backend](https://opentofu.org/docs/language/settings/backends/s3/)
- [Bootstrap S3 Remote Backend for OpenTofu](https://github.com/aws-samples/bootstrap-amazon-s3-remote-backend-for-open-tofu)
- [Solving Terraform Chicken-Egg Problem](https://cloudchronicles.blog/blog/Solving-the-Terraform-Backend-Chicken-and-Egg-Problem/)

### Valkey
- [AWS ElastiCache Creating Cluster Mode](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Replication.CreatingReplGroup.NoExistingCluster.Cluster.html)
- [Terraform aws_elasticache_replication_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticache_replication_group)
- [Terraform AWS ElastiCache Module](https://registry.terraform.io/modules/terraform-aws-modules/elasticache/aws/latest/examples/valkey-replication-group)

### IaC Structure
- [Scalr: Structuring Terraform and OpenTofu](https://scalr.com/learning-center/structuring-terraform-and-opentofu-a-platform-engineers-four-part-guide-3/)
- [Gruntwork Infrastructure Live Example](https://github.com/gruntwork-io/terragrunt-infrastructure-live-example)
- [Spacelift: Terraform Monorepo](https://spacelift.io/blog/terraform-monorepo)
- [Why I Use Terragrunt Over Terraform in 2025](https://www.axelmendoza.com/posts/terraform-vs-terragrunt/)

---

## Related Research

- [[2026-01-29-opentofu-dynamodb-elasticache|2026-01-29-opentofu-dynamodb-elasticache.md]] - Initial OpenTofu research

## Open Questions

1. **DynamoDB Table Design**: What are the specific access patterns for the table and GSIs?
2. **Valkey Use Case**: Caching, sessions, pub/sub, or all?
3. **CI/CD**: GitHub Actions or other platform for deployments?
