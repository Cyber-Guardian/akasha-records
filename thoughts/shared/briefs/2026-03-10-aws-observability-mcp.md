# Idea Brief: AWS Observability Stack with MCP Support

**Date:** 2026-03-10
**Status:** Shaped → Planning

## Problem

The dev harness MCP observability tools (`query_logs`, `query_metrics`, `query_traces`, `check_stack`) only work against the local docker-compose Victoria* stack. When debugging the triage agent running on AWS, there's no MCP-powered observability — you're stuck with the CloudWatch console. As the platform grows to 40+ Lambdas, ECS services, and Step Functions, this gap becomes a significant developer experience bottleneck.

## Constraints

- Zero container orchestration on AWS today — entire footprint is serverless (Lambda, DynamoDB, SQS, S3)
- NAT Gateway disabled in dev VPC — private subnets have no internet access
- Triage agent Lambdas run outside the VPC
- No X-Ray tracing enabled on any Lambda
- Team already has ECS experience from v1 platform (40 Lambdas, 5 ECS services, 10 Step Functions)
- V2 platform will grow to similar or larger scale
- Must work for both current triage agent and future v2 services

## Options Considered

### AWS Native (AMP + AMG + CloudWatch + X-Ray)
Use AWS managed services for all observability pillars.
- Gains: Zero operational overhead, fully managed, cheapest at small scale (~$29-32/mo for 5 Lambdas)
- Costs: Different query languages from local dev (CloudWatch Insights vs LogsQL, CW Metrics vs PromQL). Costs scale linearly with ingestion volume — ~$200-400+/mo at v2 platform scale. Vendor lock-in.
- Complexity: Low

### EC2 + docker-compose
Run the same Victoria* stack on a single EC2 instance using the same compose file.
- Gains: Cheapest self-hosted option (~$26-69/mo). Same compose file as local dev (in theory).
- Costs: "Same compose file" argument is weak — Docker's ECS integration retired 2023, AWS Copilot end-of-support June 2026. Still need Terraform for EC2/ASG/SG. No auto-recovery without ASG wrapper. Not a "grow into" solution — teams consistently migrate away as they scale past ~10 services.
- Complexity: Low initially, migration cost later

### ECS EC2 Launch Type
Run Victoria* stack as ECS services on EC2 instances with EBS volumes.
- Gains: Same Victoria* APIs as local dev (LogsQL, PromQL, Jaeger). EBS performance (no EFS penalty). Scales in place — single-node to cluster mode without re-architecture. Operational consistency with v1 and future v2 ECS services. Auto-restart, health checks, rolling deploys. Proven at scale (Zomato case study).
- Costs: More Terraform than EC2+compose (task definitions, services, cluster). Slightly higher learning curve for initial setup.
- Complexity: Medium

### EKS
Run Victoria* on Kubernetes.
- Gains: Most flexible orchestration. Rich ecosystem (Helm charts for VM).
- Costs: $73/mo control plane tax. Operational overhead disproportionate for team size. Internal research graded C (not recommended) for 1-10 devs. No existing K8s infrastructure.
- Complexity: High

## Chosen Approach

**ECS EC2 Launch Type** — because the team already knows ECS from v1, it scales in place to v2 platform size, avoids a future migration from EC2+compose, and Zomato proves the pattern works at hundreds-of-services scale. The dev/prod parity that matters (Victoria* APIs) holds regardless of orchestrator — docker-compose locally, ECS on AWS, same containers and queries.

## Key Context Discovered During Shaping

- Docker Compose → ECS bridge tooling is all deprecated/dead (Docker integration retired 2023, Copilot EOL June 2026)
- Zomato runs VictoriaMetrics on ECS EC2 with spot instances, EBS via rexray, GitOps config sync — proven production pattern
- VictoriaMetrics single-node handles millions of time series — cluster mode only needed at very large scale
- The MCP tools only need an env var to switch between local and AWS endpoints — the Victoria* APIs are identical
- Platform v2 roadmap brief already graded Kubernetes C (not recommended) for this team size
- ECS EC2 launch type gives EBS performance; Fargate would force EFS ($0.30/GB vs $0.08/GB, higher latency)

## Next Step

- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-10-aws-observability-mcp.md`
