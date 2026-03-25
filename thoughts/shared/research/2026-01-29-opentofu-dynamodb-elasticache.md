---
date: 2026-01-29T12:00:00-05:00
researcher: Claude
git_commit: 491e7136fbd1b0adc2ca5aaff20de734ceecdbed
branch: main
repository: filescience
topic: "OpenTofu for DynamoDB and ElastiCache Infrastructure"
tags: [research, opentofu, terraform, dynamodb, elasticache, aws, infrastructure-as-code]
status: complete
last_updated: 2026-01-29
last_updated_by: Claude
---

# Research: OpenTofu for DynamoDB and ElastiCache Infrastructure

**Date**: 2026-01-29T12:00:00-05:00
**Researcher**: Claude
**Git Commit**: 491e7136fbd1b0adc2ca5aaff20de734ceecdbed
**Branch**: main
**Repository**: filescience

## Research Question

How to use OpenTofu to define a DynamoDB table and an ElastiCache instance for the FileScience backend.

## Summary

This codebase is a **greenfield project** with no existing infrastructure-as-code. OpenTofu is an open-source fork of Terraform (from v1.5.6/1.6.x) that maintains **full compatibility** with Terraform's 3,900+ providers and 23,600+ modules. The AWS provider works identically in both tools. Key benefits of OpenTofu include permissive open-source licensing and native state encryption.

## Detailed Findings

### Current Codebase State

- **No IaC exists**: No Terraform, OpenTofu, CloudFormation, CDK, or other infrastructure files
- **No infrastructure directories**: No `terraform/`, `infra/`, `infrastructure/`, or `deploy/` directories
- **No AWS references**: No DynamoDB, ElastiCache, or other AWS resource configurations
- **Project structure**: Python Polylith monorepo in early/scaffolded state

### OpenTofu Overview

OpenTofu is a community-governed fork of Terraform with:
- **Full provider compatibility**: Uses same provider protocol as Terraform
- **Licensing**: Permissive open-source (vs Terraform's Business Source License)
- **State encryption**: Native support for encrypting state files
- **Migration**: Can read Terraform state files directly
- **CLI**: Uses `tofu` command instead of `terraform`

### AWS Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.47.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**Authentication** (recommended order):
1. Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`
2. AWS CLI shared credentials (`~/.aws/credentials`)
3. IAM roles (for EC2/AWS services)

### DynamoDB Table Definition

#### Basic Example (On-Demand Billing)

```hcl
resource "aws_dynamodb_table" "example" {
  name         = "my-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"
  range_key    = "timestamp"  # Optional

  attribute {
    name = "id"
    type = "S"  # S=String, N=Number, B=Binary
  }

  attribute {
    name = "timestamp"
    type = "N"
  }

  tags = {
    Environment = "production"
  }
}
```

#### With Global Secondary Index

```hcl
resource "aws_dynamodb_table" "game_scores" {
  name           = "GameScores"
  billing_mode   = "PROVISIONED"
  read_capacity  = 20
  write_capacity = 20
  hash_key       = "UserId"
  range_key      = "GameTitle"

  attribute {
    name = "UserId"
    type = "S"
  }

  attribute {
    name = "GameTitle"
    type = "S"
  }

  attribute {
    name = "TopScore"
    type = "N"
  }

  global_secondary_index {
    name               = "GameTitleIndex"
    hash_key           = "GameTitle"
    range_key          = "TopScore"
    write_capacity     = 10
    read_capacity      = 10
    projection_type    = "INCLUDE"
    non_key_attributes = ["UserId"]
  }

  tags = {
    Name        = "game-scores-table"
    Environment = "production"
  }
}
```

**Key Points**:
- `billing_mode`: `PAY_PER_REQUEST` (on-demand) or `PROVISIONED`
- Only define attributes used as keys (hash, range, GSI/LSI keys)
- Attribute types: `S` (String), `N` (Number), `B` (Binary)

### ElastiCache Definition

#### Option 1: aws_elasticache_cluster (Simple)

For Memcached or single-node Redis:

```hcl
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "my-redis-cluster"
  engine               = "redis"
  engine_version       = "7.1"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name  = aws_elasticache_subnet_group.example.name
  security_group_ids = [aws_security_group.redis.id]

  tags = {
    Environment = "dev"
  }
}
```

#### Option 2: aws_elasticache_replication_group (Production Redis with HA)

For Redis with replication and automatic failover:

```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "my-redis-cluster"
  description                = "Redis cluster with replication"

  engine               = "redis"
  engine_version       = "7.1"
  node_type            = "cache.m5.large"
  port                 = 6379
  parameter_group_name = "default.redis7"

  automatic_failover_enabled = true
  multi_az_enabled           = true
  num_cache_clusters         = 3  # 1 primary + 2 replicas

  snapshot_retention_limit = 5
  snapshot_window          = "03:00-05:00"

  subnet_group_name  = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]

  tags = {
    Environment = "production"
  }
}
```

#### Required: Subnet Group

```hcl
resource "aws_elasticache_subnet_group" "redis" {
  name       = "redis-subnet-group"
  subnet_ids = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
  ]
  description = "Subnet group for Redis cluster"
}
```

**Node Types** (cost hierarchy low to high):
- `cache.t3.*`: Burstable, cost-effective (dev/test)
- `cache.m5.*`: General purpose, balanced
- `cache.r5.*`: Memory-optimized (production)

### Recommended File Structure

```
infrastructure/
├── main.tf           # Provider config, backend
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── dynamodb.tf       # DynamoDB table definitions
├── elasticache.tf    # ElastiCache definitions
├── networking.tf     # VPC, subnets, security groups
└── README.md         # Documentation
```

**Best Practices**:
- Separate state files per environment (dev/staging/prod)
- Use S3 backend with DynamoDB lock table for state
- OpenTofu 1.8+ supports variables in backend blocks

## Code References

No existing IaC files in codebase. Relevant project files:
- `pyproject.toml` - Python project configuration
- `workspace.toml` - Polylith workspace configuration

## Architecture Documentation

This is a greenfield infrastructure setup. No existing patterns to document.

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/reference/2026-01-28-filescience-product-context-expanded-notes.md` - Product context mentions cloud integrations but no infrastructure decisions yet

## Related Research

No prior infrastructure research documents exist.

## External Resources

### Official Documentation
- [OpenTofu Official Documentation](https://opentofu.org/docs/)
- [OpenTofu Registry](https://search.opentofu.org/)
- [OpenTofu Installation Guide](https://opentofu.org/docs/intro/install/)

### AWS Resources
- [OpenTofu AWS Provider](https://search.opentofu.org/provider/opentofu/aws/latest)
- [Terraform aws_dynamodb_table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table)
- [Terraform aws_elasticache_cluster](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticache_cluster)
- [Terraform aws_elasticache_replication_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticache_replication_group)

### Guides and Tutorials
- [OpenTofu vs Terraform - Spacelift](https://spacelift.io/blog/opentofu-vs-terraform)
- [DynamoDB + Terraform Guide - Dynobase](https://dynobase.dev/dynamodb-terraform/)
- [How to Manage DynamoDB Tables With Terraform - Spacelift](https://spacelift.io/blog/terraform-dynamodb)
- [OpenTofu Tutorial - Spacelift](https://spacelift.io/blog/opentofu-tutorial)

### Modules
- [Terraform AWS DynamoDB Module](https://github.com/terraform-aws-modules/terraform-aws-dynamodb-table)
- [Terraform AWS ElastiCache Module](https://github.com/terraform-aws-modules/terraform-aws-elasticache)

## Open Questions

1. **VPC Strategy**: Will FileScience use an existing VPC or create a new one for backend services?
2. **Environment Separation**: How many environments (dev/staging/prod) and should they share state?
3. **State Backend**: Where to store OpenTofu state (S3 recommended)?
4. **DynamoDB Table Design**: What tables are needed and what are their access patterns?
5. **ElastiCache Use Case**: Redis for caching, sessions, or both? Single-node vs replication group?
6. **Infrastructure Directory**: Where should IaC live - root `infrastructure/` or separate repo?
