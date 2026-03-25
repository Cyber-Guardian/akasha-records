---
date: 2026-02-11T10:20:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-11-valkey-queue-processor-batch-failure-hidden-errors.md
last_generated: 2026-02-11T14:58:44.412846+00:00
---

# Research Brief: 2026-02-11-valkey-queue-processor-batch-failure-hidden-errors

## TL;DR

- Lambda is operating in partial batch failure mode and returns `batchItemFailures`, so invocation-level status remains successful.
- The failing items are DynamoDB stream sequence numbers, not event IDs.
- All 36 failed sequence numbers in the investigated batch resolve to `INSERT` records with `NewImage` containing only:
  - `pk`, `sk`, `throttle_context_id`, `last_updated_time`
- Handler classifies these as `ENQUEUED_WORK_ITEM`, then validation fails with `KeyError: 'data'` because required fields are missing.
- This exception path is not logged with `logger.exception`, so CloudWatch only shows "Processing record" and "Batch completed with failures".

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-11-valkey-queue-processor-batch-failure-hidden-errors.md`
