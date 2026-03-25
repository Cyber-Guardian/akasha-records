---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Health Check Convention

**Date:** 2026-03-16
**Status:** Shaped → Planning

## Problem
Each Lambda has runtime dependencies (secrets, APIs, DynamoDB, S3, Valkey) but post-deploy validation only checks that code loads — not that it can talk to its dependencies. A malformed secret (JSON-wrapped OAuth token) caused a production outage that no test caught. No shared convention or reusable infrastructure exists for defining and running deep health checks.

## Constraints
- 5 Lambdas with different dependency profiles and invocation patterns (HTTP, SQS, DynamoDB stream, async invoke)
- HTTP-triggered Lambdas (receiver) already handle empty body → 200; need a different health check entry
- Polylith architecture: shared health check logic should live in a component, per-Lambda checks in the base
- Must work with Terragrunt/direct-deploy model (no CodeDeploy aliases)
- Checks must be read-only probes (no mutations)
- No AWS-native readiness probe for Lambda — teams build their own (AWS Builders Library: "deep health check" pattern)
- AWS Lambda Powertools has no health check utility

## Options Considered

### Convention-only (docs + pattern)
Define a convention: every Lambda checks for `event.get("source") == "health-check"` and calls a local `_health_check()`. Each Lambda implements its own checks. No shared code.
- Gains: Zero coupling, each Lambda owns its checks
- Costs: Duplication, inconsistent response formats, no shared test helpers
- Complexity: Low

### Shared component (`lambda_utils.health`)
Create a `HealthCheckRegistry` in the `lambda_utils` component. Composable check primitives: `check_secret()`, `check_dynamodb_table()`, `check_s3_doc()`. Each handler registers its checks and calls `run_health_check(event)`.
- Gains: Consistent response format, reusable primitives, one test suite, declarative dependency registration
- Costs: New code in lambda_utils, coupling to component
- Complexity: Medium

### Dedicated health check Lambda
A single Lambda that invokes all others and aggregates results.
- Gains: Single point of truth, can run as scheduled canary
- Costs: Centralized coupling, new infrastructure, violates Polylith independence
- Complexity: High

## Chosen Approach
**Shared component (`lambda_utils.health`)** — best balance of reusability and ownership. Each Lambda declares its dependencies, the framework runs checks and formats responses. Convention-only is too loose (inconsistent formats). Dedicated Lambda adds infrastructure for no gain.

## Key Context Discovered During Shaping
- AWS Builders Library distinguishes "shallow" (does it load?) from "deep" (can it talk to deps?) health checks
- Common convention in the wild: `event.get("source") == "health-check"` as the trigger field
- Triage processor already has an ad-hoc `_health_check()` at handler.py:248-290 — serves as the reference implementation
- VQP has a per-invocation `_health_check(client)` that pings Valkey (handler.py:87-94) — different purpose but related
- `lambda_utils` is a "pure leaf" component (no component deps) — health check primitives fit its role
- Deploy workflows already have the CI smoke test pattern: invoke + assert on response

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-health-check-convention.md`
