---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Elastic Agent Compute — Base + Burst Workspace Infrastructure

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
Agent workspaces are single-threaded, single-homed, and constrained by local laptop resources. An agent can't fork its workspace into multiple copies, run tests with different configs in parallel, or scale up compute for intensive work. Local Docker can't handle many concurrent containers, and always-on beefy instances waste money during idle/light-dev periods.

## Constraints
- Existing workspace MCP server uses local Docker (`docker run` / `docker exec` via subprocess)
- `container.py` already abstracts container lifecycle — remote Docker is a transport-level change
- Agent runtime substrate brief (2026-03-19) chose ECS Fargate — reuse that infra
- Existing AWS VPC, ECR, Terraform patterns available
- Token costs dwarf compute costs — optimize for speed, not compute savings
- Sub-5s latency desired for workspace forking on the static box; ~30s acceptable for burst fan-out

## Options Considered

### Always-On Beefy EC2
Single large EC2 instance (c7a.4xlarge, 16 vCPU) running all containers.
- Gains: Simplest, sub-second container ops, no cold starts
- Costs: ~$360/mo even when idle, no elastic scaling
- Complexity: Low

### Pure Fargate (No Static)
Everything runs on ECS Fargate tasks, no persistent instance.
- Gains: True pay-per-use, infinite scale
- Costs: 15-30s cold start on every workspace creation, no local image cache
- Complexity: Medium

### Base + Burst (Chosen)
Small always-on EC2 for daily dev + ECS Fargate fan-out for heavy/parallel work.
- Gains: Best cost/latency tradeoff — cheap baseline, elastic burst, sub-second local ops on static box, pay-per-second burst
- Costs: Two substrates to manage (EC2 + Fargate)
- Complexity: Medium

## Chosen Approach
**Base + Burst** — small always-on EC2 (t3.medium, ~$30/mo) handles normal dev workspaces and serves as Docker image cache + ECR push point. Heavy work (parallel test matrices, large suites) snapshots via `docker commit` -> ECR push -> N Fargate tasks with different configs -> collect results -> tear down.

## Key Context Discovered During Shaping
- `WorkspaceContainer` in `components/filescience/workspace_runtime/container.py` uses local Docker subprocess calls — pointing `DOCKER_HOST` at a remote daemon requires zero code changes
- `entrypoint.sh` does full `git clone` on every start — workspace snapshotting (`docker commit`) bypasses this entirely for forked containers
- Workspace MCP server (`tools/workspace-mcp/src/workspace_mcp/server.py`) manages single workspace state — needs to become a workspace pool for parallel fan-out
- Agent runtime substrate brief ([[2026-03-19-agent-runtime-substrate|Agent Runtime Substrate]]) already chose ECS Fargate — burst tasks reuse that cluster/task-def pattern
- Cost math: static ~$30/mo + 50 burst sessions/day at ~$0.07 each = ~$130/mo total vs $360/mo always-on beefy

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-elastic-agent-compute.md`
