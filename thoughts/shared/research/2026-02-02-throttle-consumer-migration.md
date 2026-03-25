---
date: 2026-02-02T14:30:00-05:00
researcher: Claude
git_commit: 3ebc5d3b07b739def343f6cb8d4eca22158ea014
branch: main
repository: filescience
topic: "Throttle Service Consumer Migration - Discover Queue and Valkey Lambda"
tags: [research, codebase, throttling, discover, valkey-lambda, admit-api]
status: complete
last_updated: 2026-02-02
last_updated_by: Claude
---

# Research: Throttle Service Consumer Migration

**Date**: 2026-02-02T14:30:00-05:00
**Researcher**: Claude
**Git Commit**: 3ebc5d3b07b739def343f6cb8d4eca22158ea014
**Branch**: main
**Repository**: filescience

## Research Question

After completing the throttle service migration to the new `admit(AdmissionRequest)` API, what changes are needed in the consumers (discover queue and valkey lambda) to use this new API?

## Summary

The throttle service migration (THROTTLE_GATES) is complete with the new `admit(AdmissionRequest)` API. There are **two consumers** that need updates:

1. **Discover Queue** (`bases/filescience/discover/queue.py`) - Has a TODO stub at line 315-320 for calling `admit()`. Currently skips admission checking entirely. **Needs architecture work** to get `RequestPolicy` access.

2. **Valkey Lambda** (`bases/filescience/valkey_queue_processor/`) - Already registers gates and throttle contexts correctly. **No changes needed** for the admission API since admission happens at work-leasing time (in the queue), not at work-submission time (in the lambda).

## Detailed Findings

### The New Admit API

**Location**: `components/filescience/throttling/service.py:641-738`

**Signature**:
```python
def admit(
    self,
    request: AdmissionRequest,
    now: int,
) -> AdmissionResult:
```

**AdmissionRequest** (`models.py:135-154`):
```python
@dataclass(frozen=True)
class AdmissionRequest:
    policy: RequestPolicy      # Defines gate specs via gate_specs() method
    tenant_id: int             # Tenant identifier
    usage: RequestUsage        # Per-unit usage (defaults to 1.0 for all units)
    attempt_id: str | None     # Optional unique identifier
```

**RequestUsage** (`models.py:104-132`):
```python
@dataclass(frozen=True)
class RequestUsage:
    units: dict[ThrottleUnit, float]  # Empty dict = all units default to 1.0

    def for_unit(self, unit: ThrottleUnit) -> float:
        return self.units.get(unit, 1.0)

    @classmethod
    def default(cls) -> "RequestUsage":
        return cls(units={})

    @classmethod
    def of(cls, **kwargs: float) -> "RequestUsage":
        # e.g., RequestUsage.of(REQUESTS=1, GIGABYTES=5.2)
        units = {ThrottleUnit[k]: v for k, v in kwargs.items()}
        return cls(units=units)
```

**AdmissionResult** (`models.py:187-201`):
```python
@dataclass
class AdmissionResult:
    allowed: bool              # Whether admission was allowed
    next_allowed_at: int       # Unix timestamp for next allowed attempt
    blocked_by: str | None     # Gate ID that blocked if denied
    admission_token: AdmissionToken | None  # Token for concurrency gates
```

**Key Features**:
- Auto-registers missing gates on first admission (NOT_REGISTERED handling)
- Builds GateCharge internally from policy + usage
- Single retry after auto-registration

---

### Consumer 1: Discover Queue

**Location**: `bases/filescience/discover/queue.py`

**Current State**: The queue has a **TODO stub** for admission at lines 315-320:

```python
# TODO: Replace with admit(AdmissionRequest) once queue architecture is refactored
# to provide RequestPolicy. Currently the policy is only available in Valkey metadata
# and the admit() API requires a RequestPolicy object.
# See: memory-bank/thoughts/shared/plans/2026-02-02-throttle-gates-migration.md Phase 4
# self._throttle_client.admit(request, now)
pass
```

**The Problem**: The queue cannot call `admit()` because:
1. `admit()` requires an `AdmissionRequest` with a `RequestPolicy` object
2. The `RequestPolicy` is only stored in **Valkey metadata** (as JSON gate_configs)
3. The queue only has access to `throttle_context_id`, not the policy

**Current Flow** (`lease_work()` at lines 210-346):
1. Get optimal throttle context: `throttle_client.get_optimal_throttle_context(tree_id, now)` → returns `(context_id, fencing_token)`
2. Query DynamoDB for visible work items with that context
3. Lease work item with conditional update
4. **MISSING**: Call `admit()` to check rate limits
5. Return `WorkItem` to dispatcher

**What the Queue Has**:
- `throttle_context_id` (string)
- `tree_id` (string)
- `tenant_id` (int, from constructing tree_id)
- `operation_type` (string, from constructing tree_id)

**What the Queue Needs**:
- `RequestPolicy` instance with `gate_specs()` method

**Options to Get Policy**:

**Option A: Reconstruct policy from Valkey metadata**
- Throttle context metadata is stored in Valkey with `gate_configs` JSON
- Could add method to `ThrottleService` to retrieve and reconstruct policy
- Drawback: Requires new Valkey call and policy reconstruction logic

**Option B: Store policy type in DynamoDB work item**
- Work items already have `request_policy_type` field (removed during processing)
- Could keep it or add a separate field to store the policy type
- Use policy registry to reconstruct: `build_throttle_context(policy_type, ...)`
- Drawback: Requires schema change or different field handling

**Option C: Store policy in DynamoDB throttle context metadata**
- Metadata singleton at `THROTTLE_CONTEXT#{id}#METADATA` already exists
- Could add `request_policy_type` field during work registration
- Drawback: Requires migration of existing data

**Recommendation**: Option B or C - keep policy type accessible from DynamoDB since the queue already queries DynamoDB for work items.

---

### Consumer 2: Valkey Lambda

**Location**: `bases/filescience/valkey_queue_processor/`

**Current State**: The lambda **does not need changes** for the admit API.

**Why**: The lambda processes DynamoDB stream events for **work submission**, not work leasing. Its job is to:
1. Register throttle contexts in Valkey
2. Register gates in Valkey
3. Update work items with `throttle_context_id`
4. Manage ref_count for pruning

**Admission happens later** - when the dispatcher calls `queue.lease_work()` to actually process the work.

**Lambda Flow** (`handlers/submitted_work_item_handler.py`):
1. Parse `SubmittedWorkItem` from DynamoDB stream event
2. Build `ThrottleContext` from `request_policy_type` via registry
3. Call `ThrottleRegistrationService.register_or_get_throttle_context()`
4. Call `WorkRegistrationService.register_work_to_throttle_context_index()`

**ThrottleRegistrationService** (`services/throttle_service.py`):
- Generates throttle_context_id
- Registers tree if needed via `ThrottleService.register_tree()`
- Registers gates from `policy.gate_specs()` via `ThrottleService.register_gate()`
- Registers node hierarchy via `ThrottleService.register_node()`

**Key Point**: The lambda already uses the policy's `gate_specs()` to register gates at lines 442-457 of `services/throttle_service.py`. This ensures gates exist **before** any admission attempt.

---

### Data Flow Diagram

```
Work Submission (Lambda):
┌─────────────────────────────────────────────────────────────────────┐
│  DynamoDB Stream Event (INSERT with request_policy_type)            │
│                              ↓                                      │
│  Lambda: SubmittedWorkItemHandler                                   │
│    1. Build ThrottleContext from policy registry                    │
│    2. Register tree/gates/nodes in Valkey                          │
│    3. Update work item with throttle_context_id                     │
│    4. Increment ref_count metadata                                  │
└─────────────────────────────────────────────────────────────────────┘

Work Leasing (Queue):
┌─────────────────────────────────────────────────────────────────────┐
│  Dispatcher calls queue.lease_work(tenant_id, operation_type)       │
│                              ↓                                      │
│  Queue: WorkQueue.lease_work()                                      │
│    1. Get optimal throttle context from Valkey                      │
│    2. Query DynamoDB for visible work                               │
│    3. Lease work item                                               │
│    4. *** MISSING: Call admit(AdmissionRequest) ***                 │
│    5. Return WorkItem                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Throttle Methods Used by Each Consumer

**Discover Queue** (`queue.py`):
| Method | Location | Purpose |
|--------|----------|---------|
| `get_optimal_throttle_context()` | line 225 | Select best context from tree |
| `functions.try_prune_throttle_context()` | line 150-154 | Remove idle contexts |
| `functions.penalize_throttle_context()` | line 192-198 | Temporary penalty |
| `admit()` | line 315-320 | **TODO - not yet called** |

**Valkey Lambda** (`valkey_queue_processor/`):
| Method | Location | Purpose |
|--------|----------|---------|
| `functions.throttle_context_exists()` | throttle_service.py:139 | Check context exists |
| `functions.update_fencing_token()` | throttle_service.py:180 | Update token on existing context |
| `functions.tree_exists()` | throttle_service.py:209 | Check tree exists |
| `register_tree()` | throttle_service.py:216 | Create tree |
| `functions.gate_exists()` | throttle_service.py:238 | Check gate exists |
| `register_gate()` | throttle_service.py:244 | Create gate |
| `register_root_node()` | throttle_service.py:304 | Create root node |
| `register_node()` | throttle_service.py:340 | Create child nodes |
| `functions.try_prune_throttle_context()` | pruning_service.py:167 | Prune on ref_count=0 |

## Code References

- `bases/filescience/discover/queue.py:315-320` - TODO stub for admit() call
- `bases/filescience/discover/queue.py:210-346` - lease_work() method
- `bases/filescience/valkey_queue_processor/handlers/submitted_work_item_handler.py:48-106` - Work submission handler
- `bases/filescience/valkey_queue_processor/services/throttle_service.py:380-501` - Throttle registration orchestration
- `components/filescience/throttling/service.py:641-738` - admit() implementation
- `components/filescience/throttling/models.py:135-154` - AdmissionRequest model
- `components/filescience/throttling/models.py:104-132` - RequestUsage model

## Architecture Documentation

### Current Throttling Architecture

```
Components:
├── components/filescience/throttling/
│   ├── models.py          - ThrottleUnit, RateGateSpec, AdmissionRequest, etc.
│   ├── service.py         - ThrottleService with admit(), register_gate(), etc.
│   ├── async_service.py   - AsyncThrottleService (same API, async)
│   └── valkey/
│       ├── functions.py   - ValkeyFunctions (Lua function wrappers)
│       └── keys.py        - Valkey key helpers

Consumers:
├── bases/filescience/discover/
│   ├── queue.py           - WorkQueue (needs admit() integration)
│   └── dispatcher.py      - Calls queue.lease_work()
│
└── bases/filescience/valkey_queue_processor/
    ├── handler.py         - Lambda entry point
    ├── handlers/          - Record type handlers
    └── services/          - Registration and pruning services
```

### Policy Registry

The policy registry at `bases/filescience/valkey_queue_processor/policies/registry.py` maps policy type strings to ThrottleContext builders:

```python
def build_throttle_context(
    request_policy_type: str,
    tenant_id: int,
    backup_id: str,
    target_id: str,
    operation_type: str,
) -> ThrottleContext:
    # Returns ThrottleContext with appropriate RequestPolicy
```

This is currently only used by the lambda. The queue would need access to this (or similar) to reconstruct policies.

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/plans/2026-02-02-throttle-gates-migration.md` - Completed migration plan, notes architecture gap in Phase 4
- `memory-bank/thoughts/shared/research/2026-02-02-policy-vs-charge-conflict.md` - Design decision for AdmissionRequest API

## Related Research

- [[2026-01-30-throttle-service-migration-brief|2026-01-30-throttle-service-migration-brief.md]] - Original migration research
- [[2026-01-30-valkey-queue-processor-dependency-analysis|2026-01-30-valkey-queue-processor-dependency-analysis.md]] - Lambda dependencies

## Open Questions

1. **Which option for getting policy in queue?** - Options A, B, or C described above
2. **Should admission happen before or after leasing?** - Currently leases first, then admits. Could invert to avoid unnecessary lease attempts.
3. **What happens on admission denial?** - Queue needs to handle `AdmissionResult.allowed=False` - retry later, penalize context, or return to dispatcher for scheduling.
