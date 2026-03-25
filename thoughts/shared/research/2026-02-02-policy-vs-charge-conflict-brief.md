---
date: 2026-02-02T14:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-02-policy-vs-charge-conflict.md
last_generated: 2026-02-02T18:35:12.885911+00:00
---

# Research Brief: 2026-02-02-policy-vs-charge-conflict

## TL;DR

The ThrottleService currently has **two parallel admission systems** that co-exist but serve different purposes:

1. **Legacy Policy-Based System** (`request_capacity_by_context()`, `request_capacity_by_policy()`) - Uses `gate_specs`, `limiter_configs`, and tree hierarchies stored in Valkey. This is the **currently active production path** used by `valkey_queue_processor`.

2. **New Charge-Based System** (`admit()`) - Uses `GateCharge` objects and is designed for the THROTTLE_GATES plan. This path exists but is **not integrated** into the queue processor yet.

The THROTTLE_GATES plan (`filescience-infrastructure/aws/application/valkey/THROTTLE_GATES_PLAN.md`) describes the migration from the legacy "limiter" terminology to the new "gate" terminology with support for both rate gates and concurrency gates (leases).

**Current conflict**: The tests were migrated and fixed to use `policy.build_charges()` → `admit(charges, now)`, but the actual `valkey_queue_processor` still uses the legacy `request_capacity_by_context()` path. The new `admit()` method is implemented but not wired into the queue processor.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Deployment Sequencing**: Should the Lua refactoring happen before or after deploying the current code to AWS?
2. **Backwards Compatibility**: Should the legacy `request_capacity_by_*` methods be deprecated or removed?
3. **Concurrency Gates**: Are concurrency gates (leases) needed for the initial deployment, or can they come later?
4. **NOT_REGISTERED Handling**: Should auto-registration be implemented, or should gates be pre-registered?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-02-policy-vs-charge-conflict.md`
