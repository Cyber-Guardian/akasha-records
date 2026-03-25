---
date: 2026-02-04T10:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-04-dynamodb-stream-trigger-valkey-queue-processor.md
last_generated: 2026-02-04T15:44:18.946380+00:00
---

# Research Brief: 2026-02-04-dynamodb-stream-trigger-valkey-queue-processor

## TL;DR

The valkey-queue-processor Lambda handler is already designed to process DynamoDB stream events, but **the infrastructure code does not configure the stream or event source mapping**. To add the trigger, you need to:

1. Enable streams on the `work-queue` DynamoDB table (modify the DynamoDB module)
2. Create an `aws_lambda_event_source_mapping` resource (modify the Lambda module or create a new module)
3. Add IAM permissions for DynamoDB stream access (already covered by `AmazonDynamoDBFullAccess`)

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Starting Position**: Should the stream trigger use `TRIM_HORIZON` (process all existing records) or `LATEST` (only new records)? `LATEST` is safer for initial deployment.

2. **Batch Size**: Default of 100 is reasonable, but could be tuned based on processing time and memory usage.

3. **Parallelization Factor**: AWS supports up to 10 concurrent batches per shard. Not configured in the proposed solution.

4. **Error Handling**: The handler has retry via partial batch failures. Consider whether to add a dead-letter queue (DLQ) for persistently failing records.

5. **Idempotency Table**: The `dev-idempotency` table is referenced but not defined in Terraform. Should it be added to IaC?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-04-dynamodb-stream-trigger-valkey-queue-processor.md`
