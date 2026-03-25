---
date: 2026-02-10T10:30:00-05:00
researcher: Claude
git_commit: f5261b3713f3f094831096f5cff45397a50f50e3
branch: main
repository: filescience
topic: "Microsoft Graph Outlook Throttling: Compatibility Analysis for Discover Integration"
tags: [research, throttling, microsoft-graph, outlook, discover, compatibility, rate-limiting, concurrency]
status: complete
last_updated: 2026-02-10
last_updated_by: Claude
---

# Research: Microsoft Graph Outlook Throttling — Compatibility with Existing System

**Date**: 2026-02-10T10:30:00-05:00
**Researcher**: Claude
**Git Commit**: f5261b3713f3f094831096f5cff45397a50f50e3
**Branch**: main
**Repository**: filescience

## Research Question

How does Microsoft Graph handle throttling for Outlook endpoints, and is our existing throttle system compatible or does it need extension to support Outlook in the discover service?

## Summary

**The existing throttle system is highly compatible with Outlook's requirements. The concurrency gates (implemented in the 2026-02-05 plan) already handle the most complex Outlook-specific constraint (4 concurrent requests/mailbox). Only minor additions are needed to wire Outlook into the discover service.**

### Compatibility Matrix

| Outlook Throttle Dimension | Existing Support | Gap? |
|---|---|---|
| 10,000 req/10min per app+mailbox | `user_gate(target_id)` — 3,000/300s | Rate gate exists but **limit value differs** (see below) |
| 4 concurrent req per mailbox | `outlook_user_concurrency_gate(target_id)` — 4 slots, 30s TTL | **Fully implemented** |
| 130,000 req/10s global | `GLOBAL_GATE` — 130,000/10s | **Fully implemented** |
| Tenant-level (license-tiered) | `tenant_gate(license_count)` | **Fully implemented** |
| Per-app-per-tenant (1min + 24h) | `per_app_per_tenant_*` gates | **Fully implemented** |
| 150MB upload/5min per mailbox | No upload bandwidth gate | **Gap — needs new gate** |
| Retry-After 429 handling | `penalize_by_context` with `retry_after` param | **Fully implemented** |
| Delta query throttling | Same base limits apply (no special exemption) | **No special handling needed** |
| Batch request throttling | Each request counts individually | **No special handling needed** |

### Key Finding: `user_gate` Rate Limit Discrepancy

The existing `user_gate` is set to **3,000 requests per 300 seconds** (10 req/sec). Microsoft documents **10,000 requests per 10 minutes** (~16.7 req/sec) for Outlook endpoints. These are different rates — need to decide whether to:
- (A) Create a separate `outlook_user_rate_gate` with the correct 10,000/600s limit
- (B) Use the existing `user_gate` as a conservative proxy (covers Outlook safely since 3,000/300s = 10 req/sec < 16.7 req/sec)

## Detailed Findings

### 1. Complete Outlook Throttling Limits (from Microsoft Docs)

| Limit Type | Value | Scope | Notes |
|---|---|---|---|
| API Requests | 10,000 per 10 minutes | Per app + mailbox | ~16.7 req/sec sustained |
| Concurrent Requests | 4 | Per app + mailbox | Not time-based; hard platform constraint |
| Upload Size | 150MB per 5 minutes | Per mailbox | Attachment uploads |
| Bandwidth | 15 Mbits per 30 seconds | Per mailbox | Upload throttling |
| Email Sending | 150 emails per 15 minutes | Per tenant | Unique emails |
| Batch Size | 20 requests max | Per batch | Each counts individually |
| Batch Concurrency | 4 requests at a time | Outlook service | Microsoft Graph batch handler |
| Global | 130,000 per 10 seconds | Per app across all tenants | All Graph services |

**Operations covered uniformly**: Mail, Calendar, Contacts, Attachments — all subject to the same base limits.

**No per-operation differentiation**: Reads, writes, deletes all count the same against the 10,000/10min budget.

**Delta queries**: Subject to same limits. No special exemptions. Recommended for reducing API call volume.

**Application vs Delegated permissions**: Same throttle limits apply regardless of permission type.

### 2. September 2025 Change

Starting September 30, 2025, the per-app/per-user per-tenant throttling limit was reduced to **half of the total per-tenant limit**. This prevents a single user or app from consuming all quota within a tenant. The exact numeric impact on the 10,000/10min base limit is not fully documented but may reduce effective per-mailbox rates.

### 3. Existing Throttle Infrastructure — What Already Works

#### Rate Gates (fully functional)
- `GLOBAL_GATE`: 130,000 req/10s — matches Graph global limit exactly
- `tenant_gate(license_count)`: Tiered 18,750–93,750 req/300s — matches Graph tenant limits
- `per_app_per_tenant_24h_gate`: Tiered 1.2M–6M req/24h — matches Graph daily limit
- `per_app_per_tenant_1min_gate`: Tiered 1,250–6,250 req/60s — matches Graph per-minute limit
- `user_gate(target_id)`: 3,000 req/300s — conservative vs Outlook's 10,000/600s

#### Concurrency Gates (fully functional)
- `outlook_user_concurrency_gate(target_id)`: 4 slots, 30s TTL — exactly matches MailboxConcurrency limit
- Lua `admit_charges` handles mixed rate + concurrency in single atomic call
- `release_token()` / TTL expiry for slot cleanup
- `extend_token()` for long-running operations

#### OutlookTraversalPolicy (fully functional)
- Defined in `components/filescience/throttling/policies/m365/policies.py:37-52`
- Returns 6 gates: GLOBAL + tenant + 24h + 1min + user_rate + outlook_concurrency
- `build_request_policy(RequestPolicyType.OUTLOOK_TRAVERSAL, ...)` factory works
- `RequestPolicyType.OUTLOOK_TRAVERSAL` defined in throttling models

#### Penalty System (fully functional)
- `penalize_by_context()` supports `retry_after` parameter for 429 handling
- Exponential backoff: `window/8 * 2^(count-1)`, capped at `window_size`
- Diminishing returns formula prevents penalty pileup
- Tree score propagation blocks over-penalized contexts from being selected

### 4. Existing Discover Integration — What the Pattern Looks Like

The OneDrive service provides the reference implementation:

**Service handler** (`bases/filescience/discover/clouds/microsoft/services/onedrive.py`):
- `ServiceRouter(Cloud.OFFICE_365.ONEDRIVE)` creates service-level router
- `@router.register(node_type, on_initial_backup=bool)` decorators for handler functions
- Handlers call Graph API, yield discovered resources to dispatcher
- Dispatcher calls `enqueue()` with `request_policy_type="onedrive_traversal"`

**Queue integration** (`bases/filescience/discover/queue.py`):
- `enqueue()` writes work item to DynamoDB `work-queue` table
- Stream processor assigns `throttle_context_id` and registers in Valkey
- `lease_work()` calls `build_request_policy(policy_type, target_id)` to reconstruct policy
- `admit()` checks all gates atomically, returns `AdmissionToken` with leases
- `finish_work()` releases token after processing

**Cloud router** (`bases/filescience/discover/clouds/microsoft/cloud.py`):
- `O365Router = CloudRouter(Cloud.OFFICE_365)`
- `O365Router.mount_service_router(onedrive.router)`
- Outlook service would be mounted here: `O365Router.mount_service_router(outlook.router)`

### 5. What Doesn't Exist Yet (Gaps for Discover Integration)

#### 5.1 No Outlook Service Handler
- No file at `bases/filescience/discover/clouds/microsoft/services/outlook.py`
- Needs: ServiceRouter, handler functions for mailbox/folder/message discovery
- Pattern: Follow `onedrive.py` structure

#### 5.2 No Outlook Service Mount
- `cloud.py` only mounts `onedrive.router`
- Needs: `O365Router.mount_service_router(outlook.router)`

#### 5.3 Queue Processor Missing OUTLOOK_TRAVERSAL
- `bases/filescience/valkey_queue_processor/models/policy_types.py` only has `ONEDRIVE_TRAVERSAL`
- `bases/filescience/valkey_queue_processor/policies/registry.py` only handles OneDrive case
- Needs: Add `OUTLOOK_TRAVERSAL` to both

#### 5.4 No Upload Bandwidth Gate
- Microsoft enforces 150MB/5min upload limit per mailbox
- No corresponding gate exists
- Relevance: Only needed if discover fetches large attachments. For traversal/enumeration, this may not matter.

#### 5.5 No Outlook-Specific User Rate Gate
- `user_gate` is 3,000/300s (OneDrive-tuned)
- Outlook is 10,000/600s
- Options: (A) Create `outlook_user_rate_gate(target_id)`, or (B) keep using `user_gate` as conservative limit

### 6. 429 Response Handling

**Retry-After header**: Always included in Outlook 429 responses. The existing penalty system's `retry_after` parameter passes this directly to `calculate_gate_series_penalty()`.

**MailboxConcurrency 429s**: These return `Retry-After: 1` which means "wait for 1 concurrent call to complete" not "wait 1 second." The concurrency gate with explicit release handles this correctly — when a slot is released, next admission succeeds immediately.

**Proactive throttle monitoring**: `x-ms-throttle-limit-percentage` header appears at 80% capacity. Not currently used but could feed into scoring.

### 7. Batch Request Considerations

- Each request in a batch counts individually against throttle limits
- Microsoft Graph sends max 4 batch requests to Outlook at a time (aligns with MailboxConcurrency)
- Batching does NOT reduce throttling — it's for code efficiency only
- If we use batching in discover, we'd need to charge the full request count against rate gates

## Architecture Documentation

### Current Throttle Flow (Outlook-Ready)

```
1. Outlook handler discovers resource
   └─ dispatcher.enqueue(request_policy_type="outlook_traversal", target_id=mailbox_id)

2. DynamoDB Stream → valkey_queue_processor
   └─ registry.build_throttle_context(OUTLOOK_TRAVERSAL, ...)
   └─ OutlookTraversalPolicy → 6 gates (including concurrency)
   └─ Registers throttle context in Valkey tree

3. lease_work() → get_optimal_throttle_context()
   └─ Picks lowest-scoring leaf in tree
   └─ Fetches metadata → builds OutlookTraversalPolicy
   └─ admit_charges() → atomic check of all 6 gates
   └─ Returns AdmissionToken with concurrency lease

4. Process work item
   └─ GraphClient.outlook.get_messages() / get_folders() etc.
   └─ Token lease auto-expires after 30s if not released

5. finish_work()
   └─ DynamoDB transaction (tombstone + ref_count--)
   └─ release_token() → frees concurrency slot immediately
```

### Key Data Structures

- **Concurrency sorted set**: `{global}:concurrency:slots:outlook_user_concurrency_{mailbox_id}:{tenant_id}` — members are lease_ids, scores are expiry timestamps
- **Admission token hash**: `{global}:admission_token:{attempt_id}` — fields: leases JSON, expires_at
- **Rate gate counters**: `{global}:counters:requests:user_{mailbox_id}:{tenant_id}` — sliding window counters

## Code References

### Throttle Infrastructure
- `components/filescience/throttling/service.py:686-783` — `admit()` method
- `components/filescience/throttling/service.py:884-915` — `release_token()` method
- `components/filescience/throttling/models.py:67-90` — `ConcurrencyGateSpec`
- `components/filescience/throttling/models.py:180-192` — `AdmissionToken`
- `components/filescience/throttling/valkey/functions.py:320-347` — `admit_charges()` wrapper
- `infrastructure/modules/valkey/files/throttle_service.lua:509-628` — `admit_charges` Lua (3-phase)
- `infrastructure/modules/valkey/files/throttle_service.lua:251-281` — concurrency check/commit
- `infrastructure/modules/valkey/files/throttle_service.lua:634-694` — extend/release token

### M365 Policies
- `components/filescience/throttling/policies/m365/policies.py:37-52` — `OutlookTraversalPolicy` (6 gates)
- `components/filescience/throttling/policies/m365/gate_specs.py:121-138` — `outlook_user_concurrency_gate`
- `components/filescience/throttling/policies/m365/policies.py:80-112` — `build_request_policy` factory

### Discover Integration Points
- `bases/filescience/discover/queue.py:222-402` — `lease_work()` flow
- `bases/filescience/discover/queue.py:404-521` — `finish_work()` flow
- `bases/filescience/discover/clouds/microsoft/cloud.py:7-9` — O365 cloud router (mount point)
- `bases/filescience/discover/clouds/microsoft/services/onedrive.py` — reference implementation
- `bases/filescience/discover/dispatcher.py:36-40` — router registry

### Graph Client
- `components/filescience/cloud_api/microsoft/client.py:168-200` — `Outlook` class (get_folders, get_messages, etc.)
- `components/filescience/cloud_api/microsoft/models.py:122-163` — MailFolder, MailMessage models

### Queue Processor (needs OUTLOOK_TRAVERSAL)
- `bases/filescience/valkey_queue_processor/models/policy_types.py:10-12` — PolicyType enum
- `bases/filescience/valkey_queue_processor/policies/registry.py:36-48` — policy factory

### Cloud/Service Model
- `components/filescience/models/core.py:59` — `Cloud.OFFICE_365.OUTLOOK` (service ID: 2)

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/research/2026-02-05-outlook-graph-concurrency-gates.md` — Original concurrency gate research (rate limits, gap analysis, design options)
- `memory-bank/thoughts/shared/plans/2026-02-05-concurrency-gates-outlook.md` — Complete implementation plan (all 5 phases done, 240 tests passing)
- `memory-bank/thoughts/shared/research/2026-01-30-throttle-service-migration.md` — Original throttle service migration

## Related Research

- `memory-bank/thoughts/shared/research/2026-02-05-outlook-graph-concurrency-gates.md`

## Open Questions

1. **User rate gate**: Should Outlook get its own `outlook_user_rate_gate(target_id, 10000, 600)` or use the conservative `user_gate(3000, 300)`?
   - Conservative approach means ~40% lower throughput but safer
   - Separate gate allows full Outlook throughput but adds another gate to check per admission
2. **Upload bandwidth gate**: Is 150MB/5min upload limit relevant for discover (traversal/enumeration)? Likely not unless downloading attachments.
3. **Email sending limit**: 150 emails/15min per tenant — relevant only if discover writes back. Not needed for read-only backup.
4. **September 2025 per-user-per-tenant change**: The halved limit may make the existing `user_gate` even more appropriate as a conservative proxy.

## Sources

- [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
- [Microsoft Graph service-specific throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits)
- [Throttling changes coming to beta Outlook APIs (Sep 2025)](https://devblogs.microsoft.com/microsoft365dev/throttling-changes-coming-to-beta-outlook-apis-and-outlook-related-apis-in-microsoft-graph/)
- [Combine multiple HTTP requests using JSON batching](https://learn.microsoft.com/en-us/graph/json-batching)
- [MailboxConcurrency Limit Q&A](https://learn.microsoft.com/en-us/answers/questions/734143/mailboxconcurrency-limit-throttling-exception)
- [Rate limit on calendar Graph APIs Q&A](https://learn.microsoft.com/en-us/answers/questions/1343166/rate-limit-on-calendar-graph-apis)
- [Managing 429 Errors in Microsoft Graph API - Practical Guide](https://mjfnet.com/p/managing-429-errors-in-microsoft-graph-api-a-practical-guide-for-developers/)
