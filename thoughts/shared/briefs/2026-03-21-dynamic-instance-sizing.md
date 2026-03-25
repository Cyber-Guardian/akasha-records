---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Dynamic Instance Sizing for Compute Host

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
The compute host is fixed at t3.medium. Agents needing heavy compute must either run slow on 2 vCPU or context-switch to Fargate (40-70s, different machine, different UX). No way to scale the dev workspace machine itself up/down based on workload.

## Constraints
- EC2 resize requires instance stop (~90s total for stop + resize + start)
- Stopping kills all Docker containers — workspace state must be preserved via snapshot
- EIP stays attached — no IP/TLS change across resize
- Snapshot/fork (Phase 3) and auto-start/stop already exist as building blocks

## Options Considered

### Snapshot-Resize-Restore
Snapshot all workspaces, stop instance, change type, start, restore from snapshots. Time-bounded auto-scale-down.
- Gains: Same machine, same IP, same workspace. ~90s pause.
- Costs: Two resizes per burst. Small state drift risk from snapshot.
- Complexity: Medium

### Fargate-Only (Status Quo)
Use existing Fargate burst. Accept different-machine UX.
- Gains: Zero new code.
- Costs: 40-70s + ECR push, different machine, can't use dev workspace directly.
- Complexity: None

### Second Instance Pool
Two instances (cheap + beefy), move workspaces between them.
- Gains: No downtime, can overlap.
- Costs: Double infrastructure, two TLS setups, high complexity.
- Complexity: High

## Chosen Approach
**Profile-Based Resize with deferred implicit sugar** — agent declares a workload profile (dev/test/heavy/burst), system maps to instance type, snapshots workspaces, resizes, restores, auto-scales-down. Follows the proven AWS Batch pattern (explicit declaration, system picks infrastructure). Implicit auto-detection from commands deferred — hooks left for future addition.

## Key Context Discovered During Shaping
- EC2 `modify-instance-attribute` requires stopped instance — can't hot-resize
- [[2026-03-21-elastic-agent-compute|Elastic Agent Compute Plan]] Phase 3 snapshot/fork solves the "containers die on resize" problem
- Auto-start/stop plan already has the instance lifecycle management pattern
- Web research: NO agent platform or CI system does implicit compute detection. Universal pattern is explicit declaration (K8s resource requests, AWS Batch job specs, CI runner labels, GitHub Codespaces machine types). Building implicit detection is novel/unproven — defer it.

## Workload Profiles
| Profile | Instance Type | vCPU | Memory | Use Case |
|---------|--------------|------|--------|----------|
| dev | t3.medium | 2 | 4 GiB | Editing, light commands (default) |
| test | c7a.xlarge | 4 | 8 GiB | Test suite, moderate builds |
| heavy | c7a.2xlarge | 8 | 16 GiB | Full test matrix, RBSM exploration |
| burst | c7a.4xlarge | 16 | 32 GiB | Parallel test forks, heavy builds |

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-dynamic-instance-sizing.md`
