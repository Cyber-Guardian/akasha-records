---
date: 2026-02-02T14:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-02-throttle-consumer-migration.md
last_generated: 2026-02-02T21:43:23.005674+00:00
---

# Research Brief: 2026-02-02-throttle-consumer-migration

## TL;DR

The throttle service migration (THROTTLE_GATES) is complete with the new `admit(AdmissionRequest)` API. There are **two consumers** that need updates:

1. **Discover Queue** (`bases/filescience/discover/queue.py`) - Has a TODO stub at line 315-320 for calling `admit()`. Currently skips admission checking entirely. **Needs architecture work** to get `RequestPolicy` access.

2. **Valkey Lambda** (`bases/filescience/valkey_queue_processor/`) - Already registers gates and throttle contexts correctly. **No changes needed** for the admission API since admission happens at work-leasing time (in the queue), not at work-submission time (in the lambda).

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Which option for getting policy in queue?** - Options A, B, or C described above
2. **Should admission happen before or after leasing?** - Currently leases first, then admits. Could invert to avoid unnecessary lease attempts.
3. **What happens on admission denial?** - Queue needs to handle `AdmissionResult.allowed=False` - retry later, penalize context, or return to dispatcher for scheduling.

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-02-throttle-consumer-migration.md`
