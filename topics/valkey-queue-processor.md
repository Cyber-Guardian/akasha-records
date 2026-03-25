---
topic: Valkey Queue Processor
status: active
touched: 2026-03-04
related:
  - thoughts/shared/research/2026-02-04-dynamodb-stream-trigger-valkey-queue-processor.md
  - thoughts/shared/research/2026-02-11-valkey-queue-processor-batch-failure-hidden-errors.md
  - thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md
---

# Valkey Queue Processor

## Current State
Lambda handler code-complete for DynamoDB stream processing. Uses Powertools BatchProcessor, partial batch failure response, DynamoDB-backed idempotency. Routes stream records by type: `THROTTLE_CONTEXT_METADATA`, `SUBMITTED_WORK_ITEM`, `ENQUEUED_WORK_ITEM`, `TOMBSTONE`.

Infrastructure gap: DynamoDB stream trigger (event source mapping) NOT yet wired in Terraform. The DynamoDB module lacks `stream_enabled`; the Lambda module lacks `aws_lambda_event_source_mapping`. Research from 2026-02-04 documents all required Terraform changes — still pending.

## Key Decisions
- `glide_sync.GlideClient` for throttling; `glide.GlideClient` in handler — potential mismatch noted
- Standalone GLIDE mode (not cluster) as permanent workaround
- `LATEST` starting position preferred for initial stream deployment
- Powertools Tracer (X-Ray) instrumentation — 5 files with `@capture_lambda_handler` + 11x `@capture_method`

## Open Questions
- 36 poison stream records cause silent `KeyError: 'data'` — exception path lacks logging
- Stream trigger Terraform changes not yet applied (blocked on ENG-2133 env strategy)
- `idempotency` table not yet in IaC

## Artifacts
- `bases/filescience/valkey_queue_processor/` — full base
- `projects/valkey-queue-processor/` — deployment project
