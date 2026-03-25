---
date: 2026-02-05T12:00:00-05:00
researcher: Claude
git_commit: a1f1c929be1210043ac52fe2d080a8074f5920dd
branch: main
repository: filescience
topic: "Outlook Graph API Rate Limits and Concurrency Gate Requirements"
tags: [research, throttling, microsoft-graph, outlook, concurrency, rate-limiting]
status: complete
last_updated: 2026-02-05
last_updated_by: Claude
---

# Research: Outlook Graph API Rate Limits and Concurrency Gate Requirements

**Date**: 2026-02-05T12:00:00-05:00
**Researcher**: Claude
**Git Commit**: a1f1c929be1210043ac52fe2d080a8074f5920dd
**Branch**: main
**Repository**: filescience

## Research Question

What are Microsoft Graph API rate limits for Outlook endpoints, and what extra building blocks do we need to add concurrency gates to support Outlook API throttling rules?

## Summary

Microsoft Graph imposes **two distinct types of limits** for Outlook/Exchange endpoints:

1. **Rate Gates** (already supported): 10,000 requests per 10 minutes per app-mailbox combination
2. **Concurrency Gates** (NOT yet supported): **Maximum 4 concurrent requests per mailbox** (MailboxConcurrency)

The existing throttle service handles rate gates well but has **no support for concurrency/slot-based gates**. Adding concurrency gates requires new Lua functions for slot acquisition/release and new Python abstractions to manage slot lifecycle.

## Detailed Findings

### 1. Microsoft Graph Outlook Rate Limits

| Limit Type | Value | Scope | Status |
|------------|-------|-------|--------|
| **Request Rate** | 10,000 requests / 10 min | Per app + mailbox | ✅ Supported (rate gates) |
| **Concurrent Requests** | 4 maximum | Per mailbox | ❌ NOT supported |
| **Upload Bandwidth** | 150 MB / 5 min | Per mailbox | ⚠️ Partially supported (GB gates exist) |

**Critical Finding**: The **MailboxConcurrency limit of 4** is a **hard platform constraint** that cannot be increased. Exceeding this triggers immediate 429 responses regardless of remaining rate quota.

### 2. Existing Throttle Service Architecture

**Location**: `components/filescience/throttling/`

The current system implements rate gates using sliding window counters:

```
ThrottleService.admit()
→ _build_charges() (constructs GateCharge from policy + usage)
→ ValkeyFunctions.admit_charges()
→ Lua admit_charges function
→ Sliding window counter check
→ Returns AdmissionResult
```

**Key Files**:
- `service.py:650-747` - Admission logic
- `models.py:40-66` - `RateGateSpec` (rate-based only)
- `valkey/functions.py` - Lua function wrappers
- `infrastructure/modules/valkey/files/throttle_service.lua` - Atomic operations

**Supported ThrottleUnits** (`models.py:16-27`):
```python
class ThrottleUnit(Enum):
    REQUESTS = "requests"
    GIGABYTES = "gb"
    MEGABYTES = "mb"
    SLOTS = "slots"        # Defined but NOT implemented
    CONNECTIONS = "connections"  # Defined but NOT implemented
```

The `SLOTS` and `CONNECTIONS` units exist in the enum but have **no implementation** in the Lua layer.

### 3. Current M365 Gate Specs

**Location**: `components/filescience/throttling/policies/m365/gate_specs.py`

| Gate | Window | Limit | Unit |
|------|--------|-------|------|
| `GLOBAL_GATE` | 10 sec | 130,000 | requests |
| `tenant_gate` | 5 min | 18,750-93,750 (by license tier) | requests |
| `per_app_per_tenant_24h_gate` | 24 hr | 1.2M-6M | requests |
| `per_app_per_tenant_1min_gate` | 1 min | 1,250-6,250 | requests |
| `user_gate` | 5 min | 3,000 | requests |

**Missing**: No `user_concurrency_gate` for the 4-slot MailboxConcurrency limit.

### 4. Gap Analysis: What's Missing for Concurrency Gates

#### 4.1 Lua Layer Gaps

The Lua script (`throttle_service.lua`) only implements sliding window counters. It lacks:

1. **Slot acquisition**: Atomically claim one of N available slots
2. **Slot release**: Return a slot when request completes (or times out)
3. **Slot TTL**: Auto-release slots if holder crashes/times out
4. **Slot usage tracking**: Track current slot occupancy for scoring

**Conceptual Lua structure needed**:
```lua
-- Key structure for concurrency gates
-- {global}:cgate:{gate_id}          -- Hash: {max_slots, ttl_seconds}
-- {global}:cgate:{gate_id}:slots    -- Sorted set: {slot_token -> expiry_time}

local function acquire_slot(gate_id, now, holder_id)
    -- 1. Cleanup expired slots
    -- 2. Check if slots available (ZCARD < max_slots)
    -- 3. Add new slot with TTL
    -- 4. Return token or DENIED
end

local function release_slot(gate_id, holder_id)
    -- Remove slot from sorted set
end
```

#### 4.2 Python Layer Gaps

**Models** (`models.py`):
- Need `ConcurrencyGateSpec` parallel to `RateGateSpec`
- Need `SlotToken` for tracking acquired slots
- `AdmissionResult` needs to include slot tokens for release

**Service** (`service.py`):
- `admit()` needs to handle both rate and concurrency gates
- Need `release_slots()` method for explicit slot release
- Need slot TTL refresh for long-running operations

**Policy** (`policies/m365/`):
- Need `user_concurrency_gate(target_id, max_slots=4)` spec
- `OnedriveTraversalPolicy` needs to include concurrency gate

#### 4.3 Queue Integration Gaps

**Location**: `bases/filescience/discover/queue.py`

The `lease_work()` flow needs to:
1. Acquire concurrency slot as part of admission
2. Track slot token with work item
3. Release slot in `finish_work()` (success or failure)

### 5. Microsoft Graph 429 Handling Best Practices

**Retry-After Header** (required):
```
HTTP/1.1 429 Too Many Requests
Retry-After: 5
```

**Existing Support**: The penalty system (`penalize_by_context`) already supports `retry_after` parameter.

**Proactive Throttle Monitoring** (not implemented):
- Header `x-ms-throttle-limit-percentage` indicates usage (0.0-1.8)
- Could use to preemptively back off before hitting 429

### 6. Design Considerations

#### Option A: Separate Concurrency Gates (Recommended)

Add concurrency gates as a parallel system to rate gates:
- New `ConcurrencyGateSpec` type
- Separate Lua functions (`acquire_slot`, `release_slot`)
- Policies include both rate and concurrency gates
- Clear separation of concerns

**Pros**: Clean abstraction, independent scaling, easier to reason about
**Cons**: More code, two gate types to manage

#### Option B: Unified Gate Type with Mode Flag

Single `GateSpec` with `mode: "rate" | "concurrency"`:
- Reuse existing infrastructure where possible
- Mode determines which Lua function to call

**Pros**: Less code duplication
**Cons**: Muddier abstraction, mode switching complexity

#### Option C: Concurrency as Pre-Admission Check

Check concurrency before rate admission:
1. Try acquire slot (fail fast if unavailable)
2. If slot acquired, proceed to rate admission
3. If rate denied, release slot

**Pros**: Minimal changes to rate gate flow
**Cons**: Two round-trips to Valkey, slot acquired but rate denied wastes slot briefly

### 7. Recommended Building Blocks

**Phase 1: Core Concurrency Gate Infrastructure**
1. Add `ConcurrencyGateSpec` to `models.py`
2. Add Lua functions: `register_concurrency_gate`, `acquire_slot`, `release_slot`, `get_slot_usage`
3. Add `ValkeyFunctions` wrappers for new Lua functions
4. Add `ThrottleService.acquire_slot()` and `release_slot()` methods

**Phase 2: Policy Integration**
1. Add `user_concurrency_gate(target_id, max_slots=4)` to `gate_specs.py`
2. Update `RequestPolicy` to support mixed gate types
3. Update `OnedriveTraversalPolicy` to include concurrency gate

**Phase 3: Queue Integration**
1. Update `AdmissionRequest` to track slot tokens
2. Update `lease_work()` to acquire slots
3. Update `finish_work()` to release slots
4. Add slot TTL refresh for long operations

**Phase 4: Outlook-Specific Gates**
1. `outlook_user_concurrency_gate(target_id)` - 4 slots per mailbox
2. `outlook_upload_bandwidth_gate(target_id)` - 150 MB / 5 min (if needed)

## Code References

### Existing Throttle Infrastructure
- `components/filescience/throttling/service.py:650-747` - Admission logic
- `components/filescience/throttling/models.py:16-27` - ThrottleUnit enum (SLOTS exists)
- `components/filescience/throttling/models.py:40-66` - RateGateSpec
- `components/filescience/throttling/valkey/functions.py:74-113` - register_gate wrapper
- `infrastructure/modules/valkey/files/throttle_service.lua:451-517` - admit_charges Lua

### M365 Gate Specs
- `components/filescience/throttling/policies/m365/gate_specs.py:92-99` - user_gate (rate)
- `components/filescience/throttling/policies/m365/policies.py:19-33` - OnedriveTraversalPolicy

### Queue Integration
- `bases/filescience/discover/queue.py:216-384` - lease_work flow
- `bases/filescience/discover/queue.py:386-481` - finish_work transaction

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/research/2026-01-30-throttle-service-migration.md` - Original throttle service migration
- `memory-bank/thoughts/shared/plans/2026-02-02-throttle-gates-migration.md` - Gate migration plan
- `memory-bank/thoughts/shared/research/2026-02-02-policy-vs-charge-conflict.md` - Policy design decisions

## Open Questions

1. **Slot TTL duration**: What's the appropriate TTL for Outlook operation slots? (Suggested: 30-60 seconds with refresh)
2. **Slot acquisition order**: FIFO vs priority-based slot allocation?
3. **Batch request handling**: Should a JSON batch of 4 requests acquire 4 slots or 1?
4. **Cross-service isolation**: Outlook concurrency limits are per-mailbox across all apps - do we need to coordinate with other services?
5. **Upload bandwidth tracking**: Is 150 MB/5 min worth implementing as a separate gate type?

## Sources

- [Microsoft Graph service-specific throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits)
- [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
- [MailboxConcurrency Limit - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/734143/mailboxconcurrency-limit-throttling-exception)
- [Graph API rate limits - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5589984/graph-api-rate-limits)
