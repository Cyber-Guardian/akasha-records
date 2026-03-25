---
date: 2026-02-04T10:00:00-05:00
researcher: Claude
git_commit: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
branch: main
repository: filescience
topic: "Adding DynamoDB stream trigger to valkey-queue-processor"
tags: [research, codebase, dynamodb, streams, lambda, valkey-queue-processor, infrastructure]
status: complete
last_updated: 2026-02-04
last_updated_by: Claude
---

# Research: Adding DynamoDB Stream Trigger to valkey-queue-processor

**Date**: 2026-02-04T10:00:00-05:00
**Researcher**: Claude
**Git Commit**: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
**Branch**: main
**Repository**: filescience

## Research Question
How do we add a DynamoDB stream trigger to the valkey_queue_processor Lambda function?

## Summary

The valkey-queue-processor Lambda handler is already designed to process DynamoDB stream events, but **the infrastructure code does not configure the stream or event source mapping**. To add the trigger, you need to:

1. Enable streams on the `work-queue` DynamoDB table (modify the DynamoDB module)
2. Create an `aws_lambda_event_source_mapping` resource (modify the Lambda module or create a new module)
3. Add IAM permissions for DynamoDB stream access (already covered by `AmazonDynamoDBFullAccess`)

## Detailed Findings

### 1. Handler Code Is Already Stream-Ready

**Location**: `bases/filescience/valkey_queue_processor/handler.py`

The handler is fully implemented for DynamoDB stream processing:

- **Batch Processor** (line 51): `BatchProcessor(event_type=EventType.DynamoDBStreams)`
- **Event Types** (lines 221-228): Processes only `INSERT` and `MODIFY` events
- **Partial Failure Support** (line 326): Uses `process_partial_response()` for partial batch failures
- **Idempotency** (lines 54-58): DynamoDB-backed idempotency using `eventID` as key

```python
# From handler.py:51
processor = BatchProcessor(event_type=EventType.DynamoDBStreams)

# From handler.py:326-331
return process_partial_response(
    event=event,
    record_handler=record_handler,
    processor=processor,
    context=context,
)
```

### 2. DynamoDB Module Does NOT Support Streams

**Location**: `infrastructure/modules/dynamodb/main.tf`

The current DynamoDB module has **no stream configuration**:

- No `stream_enabled` attribute
- No `stream_view_type` attribute
- No `stream_arn` output

**Required additions to enable streams:**

```hcl
# In variables.tf - add:
variable "stream_enabled" {
  description = "Enable DynamoDB streams"
  type        = bool
  default     = false
}

variable "stream_view_type" {
  description = "Stream view type: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES"
  type        = string
  default     = "NEW_AND_OLD_IMAGES"
}

# In main.tf - add to aws_dynamodb_table.main:
stream_enabled   = var.stream_enabled
stream_view_type = var.stream_enabled ? var.stream_view_type : null

# In outputs.tf - add:
output "stream_arn" {
  description = "DynamoDB stream ARN"
  value       = var.stream_enabled ? aws_dynamodb_table.main.stream_arn : null
}
```

### 3. Lambda Module Does NOT Support Event Source Mappings

**Location**: `infrastructure/modules/lambda/main.tf`

The Lambda module creates the function but has **no event source mapping resource**.

**Required addition for DynamoDB stream trigger:**

```hcl
# In variables.tf - add:
variable "dynamodb_stream_arn" {
  description = "DynamoDB stream ARN to trigger Lambda"
  type        = string
  default     = null
}

variable "dynamodb_stream_batch_size" {
  description = "Batch size for DynamoDB stream"
  type        = number
  default     = 100
}

variable "dynamodb_stream_starting_position" {
  description = "Starting position: TRIM_HORIZON or LATEST"
  type        = string
  default     = "LATEST"
}

# In main.tf - add:
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  count = var.dynamodb_stream_arn != null ? 1 : 0

  event_source_arn  = var.dynamodb_stream_arn
  function_name     = aws_lambda_function.this.arn
  starting_position = var.dynamodb_stream_starting_position
  batch_size        = var.dynamodb_stream_batch_size

  # Enable partial batch failure reporting
  function_response_types = ["ReportBatchItemFailures"]
}
```

### 4. IAM Permissions Already Sufficient

**Location**: `infrastructure/modules/lambda/main.tf:35-39`

The Lambda module attaches `AmazonDynamoDBFullAccess` when `enable_dynamodb_access = true`:

```hcl
resource "aws_iam_role_policy_attachment" "dynamodb" {
  count      = var.enable_dynamodb_access ? 1 : 0
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}
```

This managed policy includes the required stream permissions:
- `dynamodb:DescribeStream`
- `dynamodb:GetRecords`
- `dynamodb:GetShardIterator`
- `dynamodb:ListStreams`

### 5. Live Configuration Changes Needed

**work-queue table** (`infrastructure/live/non-prod/us-east-1/dev/dynamodb/terragrunt.hcl`):

Current configuration (lines 9-47) has no stream settings. Add:

```hcl
inputs = {
  # ... existing config ...

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"  # Handler accesses new_image
}
```

**valkey-queue-processor** (`infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor/terragrunt.hcl`):

Add dependency on DynamoDB stream ARN and pass to module:

```hcl
inputs = {
  # ... existing config ...

  dynamodb_stream_arn            = dependency.dynamodb.outputs.stream_arn
  dynamodb_stream_batch_size     = 100
  dynamodb_stream_starting_position = "LATEST"
}
```

### 6. Current Infrastructure Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                         Current State                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                    ┌─────────────────────┐   │
│  │  work-queue  │                    │ valkey-queue-proc   │   │
│  │  DynamoDB    │ ───── ? ─────────▶ │     Lambda          │   │
│  │  (no stream) │   (no trigger)     │ (stream-ready code) │   │
│  └──────────────┘                    └─────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        Target State                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────┐   ┌─────────────────────┐   │
│  │  work-queue  │     │  Event   │   │ valkey-queue-proc   │   │
│  │  DynamoDB    │────▶│  Source  │──▶│     Lambda          │   │
│  │  (stream on) │     │  Mapping │   │ (stream-ready code) │   │
│  └──────────────┘     └──────────┘   └─────────────────────┘   │
│         │                                       │               │
│         └───────────────────────────────────────┘               │
│              Processes INSERT/MODIFY events                     │
└─────────────────────────────────────────────────────────────────┘
```

### 7. Handler Record Routing

The handler routes records by type (lines 245-306):

| Record Type | Handler | Action |
|-------------|---------|--------|
| `THROTTLE_CONTEXT_METADATA` | `ThrottleContextMetadataHandler` | Registers throttle context in Valkey |
| `SUBMITTED_WORK_ITEM` | `SubmittedWorkItemHandler` | Registers work item, creates throttle context |
| `ENQUEUED_WORK_ITEM` | (validates schema) | Skips - already processed |
| `TOMBSTONE` | (validates schema) | Skips - already processed |
| `UNKNOWN` | (logs) | Skips silently |

### 8. Stream View Type Requirement

The handler accesses `new_image` from stream records:

```python
# handler.py:232-233
new_image = record.dynamodb.new_image if record.dynamodb else None
```

This requires either:
- `NEW_IMAGE` - Only the new item state (sufficient for INSERT/MODIFY)
- `NEW_AND_OLD_IMAGES` - Both old and new states (more flexible)

## Code References

| File | Line(s) | Description |
|------|---------|-------------|
| `bases/filescience/valkey_queue_processor/handler.py` | 51 | BatchProcessor for DynamoDB streams |
| `bases/filescience/valkey_queue_processor/handler.py` | 221-228 | INSERT/MODIFY event filtering |
| `bases/filescience/valkey_queue_processor/handler.py` | 326-331 | Partial batch failure response |
| `infrastructure/modules/dynamodb/main.tf` | 1-61 | DynamoDB module (no stream config) |
| `infrastructure/modules/lambda/main.tf` | 35-39 | DynamoDB IAM policy attachment |
| `infrastructure/modules/lambda/main.tf` | 89-122 | Lambda function resource (no ESM) |
| `infrastructure/live/non-prod/us-east-1/dev/dynamodb/terragrunt.hcl` | 9-47 | work-queue table config |
| `infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor/terragrunt.hcl` | 1-66 | Lambda config |

## Architecture Documentation

### Existing Patterns

1. **Content-Addressed Deployments**: Lambda artifacts use SHA256-hashed S3 keys for immutable deploys
2. **Terragrunt Dependencies**: Modules reference outputs from other modules via `dependency` blocks
3. **Conditional Resources**: Lambda module uses `count` for conditional VPC/DynamoDB/SSM policy attachment
4. **Environment Variables**: Configuration passed to Lambda via environment variables from Terragrunt

### No Existing Event Source Mapping Pattern

The codebase has no existing `aws_lambda_event_source_mapping` resources. This is the first Lambda that needs a trigger beyond direct invocation.

## Historical Context (from memory-bank/thoughts/)

| Document | Relevance |
|----------|-----------|
| `memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md` | Dependency analysis of the Lambda |
| `memory-bank/thoughts/shared/handoffs/general/2026-02-03_valkey-queue-processor-layer-and-tls-deploy.md` | Recent deployment notes, mentions direct invoke fails because handler expects stream events |
| `memory-bank/thoughts/shared/research/2026-01-29-opentofu-dynamodb-elasticache.md` | DynamoDB infrastructure patterns |

## Open Questions

1. **Starting Position**: Should the stream trigger use `TRIM_HORIZON` (process all existing records) or `LATEST` (only new records)? `LATEST` is safer for initial deployment.

2. **Batch Size**: Default of 100 is reasonable, but could be tuned based on processing time and memory usage.

3. **Parallelization Factor**: AWS supports up to 10 concurrent batches per shard. Not configured in the proposed solution.

4. **Error Handling**: The handler has retry via partial batch failures. Consider whether to add a dead-letter queue (DLQ) for persistently failing records.

5. **Idempotency Table**: The `dev-idempotency` table is referenced but not defined in Terraform. Should it be added to IaC?
