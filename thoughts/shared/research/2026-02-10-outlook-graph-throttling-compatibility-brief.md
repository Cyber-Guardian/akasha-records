---
date: 2026-02-10T10:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-10-outlook-graph-throttling-compatibility.md
last_generated: 2026-02-10T17:28:08.507343+00:00
---

# Research Brief: 2026-02-10-outlook-graph-throttling-compatibility

## TL;DR

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

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **User rate gate**: Should Outlook get its own `outlook_user_rate_gate(target_id, 10000, 600)` or use the conservative `user_gate(3000, 300)`?
   - Conservative approach means ~40% lower throughput but safer
   - Separate gate allows full Outlook throughput but adds another gate to check per admission
2. **Upload bandwidth gate**: Is 150MB/5min upload limit relevant for discover (traversal/enumeration)? Likely not unless downloading attachments.
3. **Email sending limit**: 150 emails/15min per tenant — relevant only if discover writes back. Not needed for read-only backup.
4. **September 2025 per-user-per-tenant change**: The halved limit may make the existing `user_gate` even more appropriate as a conservative proxy.

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-10-outlook-graph-throttling-compatibility.md`
