---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Agent Runtime Substrate — Universal Agent Platform on ECS Fargate

**Date:** 2026-03-19
**Status:** Shaped → Planning

## Problem
No hosted platform for running agent workloads. Every agent system (autoresearch, triage-bot, Helm) has built its own bespoke dispatch, lifecycle, and integration layer locally. Helm (~3,500 LOC, 11 commands, dual state machine) was over-engineered for the wrong abstraction — a local CLI orchestrator when what's needed is a hosted queued work provisioner. Cyrus (third-party, not Linear-built) has an unresolved wiring gap since March 2 and is a managed black box. Out-loop autonomous work has no execution leg.

## Constraints
- Must run on infrastructure (AWS — existing VPC, DynamoDB patterns, deploy pipeline)
- Agent runtime is pluggable: Claude Agent SDK now, FS-1 later
- Multiple input interfaces, not just Linear (Slack, GitHub webhooks, internal/API)
- Linear Agent SDK (Developer Preview, May 2025) provides first-class agent primitives — use it, don't rebuild it
- No Python SDK for Linear Agent SDK — GraphQL direct calls required
- Claude Agent SDK requires Node.js in every container image

## Chosen Approach
**Event-driven ECS Fargate with DynamoDB job store** — ephemeral container-per-task, multiple input adapters (Linear Agent SDK, Slack, GitHub, API), pluggable agent runtime.

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-19-agent-runtime-substrate.md`
