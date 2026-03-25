# Idea Brief: Ephemeral Agent Environments

**Date:** 2026-02-27
**Status:** Shaped → Planning

## Problem
Agents and devs share a single `dev` environment in AWS. Deploying and provisioning takes minutes, agents can step on each other's state, and integration bugs that mocks miss only surface against real AWS. There's no isolated, fast feedback loop for running the full pipeline locally.

## Constraints
- Two Lambda handlers with different trigger patterns (DynamoDB Streams ESM vs direct invocation)
- Valkey uses GLIDE client with TLS — local Valkey Docker doesn't have TLS
- DynamoDB Streams → Lambda ESM can't be emulated locally without a custom event pump
- moto already proven in test suite for DynamoDB
- `sandbox.py` exists as a local harness but hits real AWS
- `VALKEY_USE_TLS` env var already exists for TLS toggle
- `AWS_ENDPOINT_URL` supported natively by boto3 for endpoint swap
- LocalStack killed Community Edition (March 2026), paid for ElastiCache ($39/seat)

## Options Considered

### LocalStack Full Emulation
Run LocalStack per agent with DynamoDB, S3, SSM, Lambda emulation + Valkey Docker.
- Gains: Near-zero app code changes, DynamoDB Streams ESM included (in theory)
- Costs: $39/seat minimum (ElastiCache + Lambda layers), fidelity gaps in Streams→Lambda, licensing uncertainty post-March 2026, Lambda layers silently no-op on free tier
- Complexity: Medium

### Docker Compose + Moto Server + Real Valkey
Run moto server (DynamoDB + S3 + SSM), real Valkey container, direct handler invocation. Build event pump for Streams→handler flow.
- Gains: Proven moto fidelity, real Valkey (Lua scripts, RESP), no vendor lock-in, fastest startup, endpoint-swap config only
- Costs: No automatic Streams→Lambda trigger (need event pump), TLS config branching for GLIDE
- Complexity: Medium

### Port/Adapter Abstraction Layer
Extract interfaces for every infrastructure dependency with in-memory test implementations.
- Gains: Zero infrastructure per agent, pure Python, instant startup
- Costs: Massive refactoring, two implementations to maintain, loses fidelity on Valkey Lua/GLIDE/DynamoDB query semantics — the exact bugs we want to catch
- Complexity: High

## Chosen Approach
**Docker Compose + Moto Server + Real Valkey** — Best balance of fidelity (real Valkey, proven moto), speed (seconds to start, no Terraform), and isolation (compose project per agent). No application code abstraction needed — endpoint swap via `AWS_ENDPOINT_URL` env var. Event pump bridges the DynamoDB Streams → handler gap with ~50-100 lines of deterministic code.

## Key Context Discovered During Shaping
- `boto3` natively supports `AWS_ENDPOINT_URL` (added 2023) — no wrapper needed
- `VALKEY_USE_TLS` already exists in VQP config — `handler.py:21`
- `DISCOVER_VALKEY_USE_TLS` already exists in discover — `manager.py:73`
- Community consensus: moto for unit tests, real containers for integration, neither replaces staging
- moto `ThreadedMotoServer` runs in-process on random port (no Docker needed for AWS)
- Existing OTel docker-compose at `development/observability/local/docker-compose.yaml` can be extended
- SSM params follow `/clouds/{cloud_name}/cloudsmith` path — `session.py:53`
- DynamoDB table schemas: work-queue (pk/sk + GSI throttle-context-visible-index + streams NEW_AND_OLD_IMAGES), resources (pk/sk + GSI ParentLinkCreatedIndex), dev-idempotency (id only + TTL)

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-27-ephemeral-agent-environments.md`
