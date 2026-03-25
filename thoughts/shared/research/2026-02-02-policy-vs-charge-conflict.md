---
date: 2026-02-02T14:30:00-05:00
researcher: claude-opus-4-5
git_commit: 206f8473b221e6fd8865a56201f079817232b03c
branch: main
repository: filescience
topic: "Policy vs Charge System Conflict - Post-Migration State"
tags: [research, throttling, gates, policy, charges, admission]
status: complete
last_updated: 2026-02-02
last_updated_by: claude-opus-4-5
---

# Research: Policy vs Charge System Conflict - Post-Migration State

**Date**: 2026-02-02T14:30:00-05:00
**Researcher**: claude-opus-4-5
**Git Commit**: 206f8473b221e6fd8865a56201f079817232b03c
**Branch**: main
**Repository**: filescience

## Research Question

The handoff document mentions a conflict between "policy vs charge system" that was half-implemented before the bedrock test migration. What is the current state of the throttling system and what needs to be resolved?

## Summary

The ThrottleService currently has **two parallel admission systems** that co-exist but serve different purposes:

1. **Legacy Policy-Based System** (`request_capacity_by_context()`, `request_capacity_by_policy()`) - Uses `gate_specs`, `limiter_configs`, and tree hierarchies stored in Valkey. This is the **currently active production path** used by `valkey_queue_processor`.

2. **New Charge-Based System** (`admit()`) - Uses `GateCharge` objects and is designed for the THROTTLE_GATES plan. This path exists but is **not integrated** into the queue processor yet.

The THROTTLE_GATES plan (`filescience-infrastructure/aws/application/valkey/THROTTLE_GATES_PLAN.md`) describes the migration from the legacy "limiter" terminology to the new "gate" terminology with support for both rate gates and concurrency gates (leases).

**Current conflict**: The tests were migrated and fixed to use `policy.build_charges()` → `admit(charges, now)`, but the actual `valkey_queue_processor` still uses the legacy `request_capacity_by_context()` path. The new `admit()` method is implemented but not wired into the queue processor.

## Detailed Findings

### Current ThrottleService API Surface

The service at `components/filescience/throttling/service.py` exposes three admission methods:

| Method | Lines | Input | Used By |
|--------|-------|-------|---------|
| `request_capacity_by_context()` | 536-622 | `tree_id`, `throttle_context_id`, `req_units_list` | valkey_queue_processor (legacy) |
| `request_capacity_by_policy()` | 624-708 | `gate_specs[]`, `req_units_list` | Direct API calls (legacy) |
| `admit()` | 814-893 | `list[GateCharge]` | Tests (new) |

### Legacy System: How valkey_queue_processor Uses Throttling

The queue processor at `bases/filescience/valkey_queue_processor/` currently uses the **registration-first** approach:

**Registration Flow** (`services/throttle_service.py:380-501`):
1. Build `ThrottleContext` from `RequestPolicy` via `build_throttle_context()`
2. Generate `throttle_context_id` = `{tree_id}:{target_id}`
3. Register tree in Valkey if not exists
4. For each `gate_spec` from `policy.gate_specs()`:
   - Register limiter with `{dimension}:{name}:{tenant_id}` ID
   - Build `limiter_configs` list with `id`, `rate_limit`, `window_size`, `counter_window_size`
5. Register node hierarchy (root → children → leaf)
6. Store metadata on leaf node including serialized `limiter_configs`

**Admission Flow** (not currently implemented in queue processor):
The queue processor registers contexts but does not yet call admission. The `request_capacity_by_context()` method is available but not wired in.

### New Charge-Based System: THROTTLE_GATES Design

Per `THROTTLE_GATES.md`, the new system standardizes on:

**Vocabulary Changes**:
- `limiter` → `gate`
- `hierarchy` → `series`
- `ThrottleRule` → `RateGateSpec` / `ConcurrencyGateSpec`

**API Changes**:
- Old: `admit(policy, tenant_id, now, attempt_id)` (policy-first)
- New: `admit(charges, now, attempt_id)` (charge-first)

**Key Insight**: The `RequestPolicy.build_charges(tenant_id)` method (at `models.py:144-159`) is the **bridge** between the two systems:
```python
def build_charges(self, tenant_id: int, **kwargs) -> list[GateCharge]:
    return [
        GateCharge(gate_id=f"{spec.gate_id}:{tenant_id}", units=1.0)
        for spec in self.gate_specs()
    ]
```

### Test Migration Fix

The handoff documents the test fix applied:
```python
# Before (broken - policy-first signature that never existed):
result = throttle_service.admit(policy, TENANT_ID, NOW)

# After (fixed - charge-first signature):
charges = policy.build_charges(TENANT_ID)
result = throttle_service.admit(charges, NOW)
```

This fix aligns tests with the **implemented** `admit()` method signature, not a proposed future API.

### What's Missing for THROTTLE_GATES Completion

Per `THROTTLE_GATES_PLAN.md`, these objectives remain:

**1. Lua Refactoring** (`throttle_service.lua`):
- [ ] Rename `limiter` → `gate`, `hierarchy` → `series`
- [ ] Remove `ensure_and_check_limiter_hierarchy`
- [ ] Add `admit_gate_series` function accepting `GateCharge[]`
- [ ] Add gate type dispatch (`rate` vs `concurrency`)
- [ ] Add `NOT_REGISTERED` fail-fast with missing gate IDs list
- [ ] Add concurrency gate lease support (ZSET per gate)
- [ ] Add `AdmissionToken` with extend/release

**2. Python Service Changes**:
- [ ] Implement `NOT_REGISTERED` handling in `admit()` (auto-register missing gates, retry once)
- [ ] Wire `admit()` into queue processor flow
- [ ] Add `extend_token()` and `release_token()` integration

**3. Queue Processor Integration**:
- [ ] Replace `request_capacity_by_context()` with `admit()` calls
- [ ] Add token lifecycle management (release on completion/failure)
- [ ] Update metadata schema (remove `limiter_configs`, store only policy metadata)

### Current State of Skipped Tests

10 tests are skipped with reason: `"NOT_REGISTERED auto-registration not implemented in service"`

These tests validate the planned behavior where:
1. `admit()` returns `ADMISSION_STATUS_NOT_REGISTERED` with missing gate IDs
2. Service automatically registers missing gates from `policy.gate_specs()`
3. Service retries admission once

The constant `ADMISSION_STATUS_NOT_REGISTERED = -1` exists at `service.py:49` but no handling logic is implemented.

## Code References

### ThrottleService Methods
- `components/filescience/throttling/service.py:536-622` - `request_capacity_by_context()` (legacy)
- `components/filescience/throttling/service.py:624-708` - `request_capacity_by_policy()` (legacy)
- `components/filescience/throttling/service.py:814-893` - `admit()` (new, charge-based)

### Models
- `components/filescience/throttling/models.py:11-36` - `RateGateSpec`
- `components/filescience/throttling/models.py:39-62` - `ConcurrencyGateSpec`
- `components/filescience/throttling/models.py:68-78` - `GateCharge`
- `components/filescience/throttling/models.py:111-125` - `AdmissionResult`
- `components/filescience/throttling/models.py:128-160` - `RequestPolicy` (with `build_charges()`)

### Queue Processor
- `bases/filescience/valkey_queue_processor/services/throttle_service.py:380-501` - `register_or_get_throttle_context()`
- `bases/filescience/valkey_queue_processor/policies/registry.py:11-44` - `build_throttle_context()`

### Infrastructure Docs
- `filescience-infrastructure/aws/application/valkey/THROTTLE_GATES.md` - Vocabulary and contract
- `filescience-infrastructure/aws/application/valkey/THROTTLE_GATES_PLAN.md` - Implementation plan

## Architecture Documentation

### Two Parallel Admission Paths

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LEGACY PATH (Active)                        │
│                                                                     │
│  RequestPolicy ──► gate_specs() ──► limiter_configs[]              │
│        │                                   │                        │
│        ▼                                   ▼                        │
│  register_or_get_throttle_context()   request_capacity_by_context()│
│        │                                   │                        │
│        ▼                                   ▼                        │
│  [Valkey: tree + nodes + metadata]    ensure_and_check_gate_hierarchy│
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         NEW PATH (Implemented but not wired)        │
│                                                                     │
│  RequestPolicy ──► gate_specs() ──► build_charges(tenant_id)       │
│                                            │                        │
│                                            ▼                        │
│                                     list[GateCharge]               │
│                                            │                        │
│                                            ▼                        │
│                                     admit(charges, now)            │
│                                            │                        │
│                                            ▼                        │
│                                     admit_charges (Lua)            │
│                                            │                        │
│                                            ▼                        │
│                                     AdmissionResult + Token        │
└─────────────────────────────────────────────────────────────────────┘
```

### GateCharge Flow

```
GateSpec                    GateCharge                   Lua
┌───────────────┐          ┌──────────────────┐         ┌─────────────────┐
│ dimension     │          │ gate_id          │         │ gate_id         │
│ name          │ ───────► │   = "dim:name:   │ ──JSON─►│                 │
│ rate_limit    │ build_   │      tenant_id"  │         │ units           │
│ time_window   │ charges()│ units = 1.0      │         └─────────────────┘
└───────────────┘          └──────────────────┘
```

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/plans/2026-01-30-throttle-service-migration.md` - Original migration plan
- `memory-bank/thoughts/shared/research/2026-01-30-throttle-service-migration-brief.md` - Migration research
- `memory-bank/thoughts/shared/handoffs/general/2026-02-02_13-15-23_bedrock-test-migration-complete.md` - Test migration completion noting the API mismatch fix

## Related Research

- `memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md` - Queue processor dependencies

## Open Questions

1. **Deployment Sequencing**: Should the Lua refactoring happen before or after deploying the current code to AWS?
2. **Backwards Compatibility**: Should the legacy `request_capacity_by_*` methods be deprecated or removed?
3. **Concurrency Gates**: Are concurrency gates (leases) needed for the initial deployment, or can they come later?
4. **NOT_REGISTERED Handling**: Should auto-registration be implemented, or should gates be pre-registered?

## Recommendations for Next Steps

Based on the research, the work breakdown would be:

1. **Phase A: Deploy Current State** - The existing code (registration + legacy admission path) is functional. Deploy it to validate infrastructure.

2. **Phase B: Lua Refactoring** - Implement `admit_gate_series` with `NOT_REGISTERED` support and gate type dispatch.

3. **Phase C: Python Integration** - Wire `admit()` into queue processor, implement auto-registration retry logic.

4. **Phase D: Concurrency Gates** - Add lease support for concurrency control (optional, can be deferred).
