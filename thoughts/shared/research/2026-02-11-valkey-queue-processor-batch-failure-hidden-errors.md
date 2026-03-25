---
date: 2026-02-11T10:20:00-05:00
researcher: gpt-5.3-codex-high
git_commit: f5261b3
branch: main
repository: filescience
topic: "Valkey queue processor batch failures without visible error logs"
tags: [research, valkey-queue-processor, dynamodb-streams, logging]
status: complete
last_updated: 2026-02-11
last_updated_by: gpt-5.3-codex-high
---

# Research: Valkey queue processor batch failures without visible error logs

## Research Question
Why does `filescience-dev-valkey-queue-processor` show batch failures but Lambda invocations do not show as failed, and why are error details missing in CloudWatch logs?

## Summary
- Lambda is operating in partial batch failure mode and returns `batchItemFailures`, so invocation-level status remains successful.
- The failing items are DynamoDB stream sequence numbers, not event IDs.
- All 36 failed sequence numbers in the investigated batch resolve to `INSERT` records with `NewImage` containing only:
  - `pk`, `sk`, `throttle_context_id`, `last_updated_time`
- Handler classifies these as `ENQUEUED_WORK_ITEM`, then validation fails with `KeyError: 'data'` because required fields are missing.
- This exception path is not logged with `logger.exception`, so CloudWatch only shows "Processing record" and "Batch completed with failures".

## Evidence Collected
- CloudWatch stream investigated:
  - `/aws/lambda/filescience-dev-valkey-queue-processor`
  - `2026/02/11/[$LATEST]a63b6a651b4f42febc94b3a20d4b3768`
- Observed 6 invocations in this stream:
  - each with `record_count=40`
  - each with `failure_count=36`
  - no `ERROR`/`Exception` lines in the stream
- Failed item IDs from logs were resolved via DynamoDB Streams API and all 36 were:
  - `eventName=INSERT`
  - `NewImage` keys exactly `['last_updated_time', 'pk', 'sk', 'throttle_context_id']`

## Reproduction of Hidden Error
- Fetch one failed sequence number from stream and deserialize `NewImage`.
- Run classifier + model parse:
  - `identify_record_type(...) -> RecordType.ENQUEUED_WORK_ITEM`
  - `EnqueuedWorkItem.from_dynamodb_record(...) -> KeyError: 'data'`

## Relevant Code Paths
- `lambda_handler` logs only aggregate failures and returns partial response:
  - `bases/filescience/valkey_queue_processor/handler.py`
- `record_handler` logs "Processing record" and routes by record type, but does not wrap route execution in a catch+log:
  - `bases/filescience/valkey_queue_processor/handler.py`
- Enqueued work item parser requires fields missing in these records:
  - `bases/filescience/valkey_queue_processor/models/schema.py`
- Work registration update can set `throttle_context_id` and `last_updated_time` without a condition check:
  - `bases/filescience/valkey_queue_processor/services/work_registration_service.py`

## Key Takeaways
1. Invocation success with batch failures is expected behavior for partial batch response.
2. Missing error details are due to an exception path that currently lacks explicit logging.
3. The poison records in this stream are minimal enqueued-image inserts that cannot satisfy `EnqueuedWorkItem` schema requirements.
