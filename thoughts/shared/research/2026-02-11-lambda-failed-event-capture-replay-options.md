---
date: 2026-02-11T17:30:00-05:00
researcher: Claude
git_commit: f5261b3713f3f094831096f5cff45397a50f50e3
branch: main
repository: filescience
topic: "Lambda Failed Event Capture & Local Replay System Design"
tags: [research, lambda, event-capture, replay, debugging, infrastructure, extensions]
status: complete
last_updated: 2026-02-11
last_updated_by: Claude
---

# Research: Lambda Failed Event Capture & Local Replay System

**Date**: 2026-02-11T17:30:00-05:00
**Researcher**: Claude
**Git Commit**: f5261b3713f3f094831096f5cff45397a50f50e3
**Branch**: main
**Repository**: filescience

## Research Question
How can we set up a system to capture Lambda event payloads for failed invocations (via layer, extension, or wrapper), store them durably (S3 or DynamoDB), and provide a CLI tool or local harness for replaying those events against a Lambda handler locally?

## Summary

Three viable approaches exist, ranked by complexity and fit for this repo:

1. **AWS-native ESM OnFailure → S3** (recommended first step) — Zero code changes, Terraform-only. Full event payload retained in S3 for stream-triggered Lambdas. Requires adding `destination_config`, retry bounds, and bisect controls to the Lambda module.

2. **Custom Python replay harness** — A ~50-line script that reads captured events from S3/local JSON and invokes `lambda_handler` or `record_handler` directly, with optional `debugpy` attachment. No Docker, no SAM, no heavy tooling.

3. **Lambda extension/layer wrapper** — Possible but overkill. Extensions API cannot intercept event payloads directly (metadata only). A Runtime API proxy extension could, but requires Go/Rust and significant complexity. A Python decorator on `record_handler` is simpler and achieves the same goal.

**Bottom line**: The AWS-native path + a thin replay script covers 95% of the need. A custom extension is not warranted.

## Detailed Findings

### 1. Current Infrastructure State

**Lambda Module** (`infrastructure/modules/lambda/main.tf:134-145`):
- DynamoDB event source mapping exists with `ReportBatchItemFailures` enabled
- **Missing**: `destination_config`, `maximum_retry_attempts`, `maximum_record_age_in_seconds`, `bisect_batch_on_function_error`
- Currently: unbounded retries until stream retention expires (24h)

**Handler Pattern** (`bases/filescience/valkey_queue_processor/handler.py`):
- Uses Powertools `BatchProcessor` + `process_partial_response()`
- Per-record failures are returned as `batchItemFailures`, NOT raised as exceptions from `lambda_handler`
- This means wrapping `lambda_handler` would NOT capture per-record failures — only total batch crashes
- To capture per-record failures, must wrap `record_handler` (line 302)

**Existing test fixtures**:
- `test/events/valkey_queue_processor_live_failure_event.json` — real DynamoDB stream INSERT event that triggers `KeyError: 'data'`
- `.vscode/launch.json` — debugpy configs for discover sandbox (no Lambda-specific configs yet)

**No failure destination infrastructure exists** — no S3 bucket, no SQS queue, no IAM policies for failure capture.

### 2. AWS-Native: ESM OnFailure → S3 (Recommended Primary Path)

**How it works**: When DynamoDB Streams event source mapping exhausts retries, AWS writes the full event to an S3 destination.

**S3 object includes**:
```json
{
  "requestContext": { "requestId": "...", "condition": "RetryAttemptsExhausted" },
  "responseContext": { "statusCode": 200, "functionError": "Unhandled" },
  "DDBStreamBatchInfo": { "shardId": "...", "startSequenceNumber": "...", "batchSize": 1 },
  "payload": "<FULL DynamoDB Stream event JSON>"
}
```

**S3 key format**: `aws/lambda/<ESM-UUID>/<shardID>/YYYY/MM/DD/YYYY-MM-DDTHH.MM.SS-<UUID>`

**Terraform changes needed**:

```hcl
# infrastructure/modules/lambda/main.tf — extend event source mapping
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  # ... existing config ...

  destination_config {
    on_failure {
      destination_arn = var.on_failure_destination_arn  # NEW
    }
  }

  maximum_retry_attempts          = var.maximum_retry_attempts          # NEW
  maximum_record_age_in_seconds   = var.maximum_record_age_in_seconds   # NEW
  bisect_batch_on_function_error  = var.bisect_batch_on_function_error  # NEW
}
```

**New infrastructure needed**:
- S3 bucket for failed events (or prefix in existing artifacts bucket)
- IAM policy: `s3:PutObject` + `s3:ListBucket` on destination bucket
- Note: IAM policy goes on the **event source mapping role** (Lambda execution role), not a separate role

**Pros**: Zero handler code changes, AWS-managed, full payload retained, S3 lifecycle for retention
**Cons**: Only captures after ALL retries exhausted (not per-record partial failures)

**Important nuance for this repo**: Since `valkey-queue-processor` uses `ReportBatchItemFailures`, individual record failures are retried by the stream poller (not the Lambda service). The OnFailure destination triggers when the *entire batch* fails after max retries — which happens when the same records keep failing across retries. This IS the behavior we want for poison records.

### 3. Lambda Extensions API — Why It's Not the Right Tool

**Lifecycle**: INIT → INVOKE → SHUTDOWN. Extensions receive events at each phase.

**INVOKE event** (what an extension receives):
```json
{
  "eventType": "INVOKE",
  "deadlineMs": 1234567890,
  "requestId": "316aa6d0-...",
  "invokedFunctionArn": "arn:aws:lambda:...",
  "tracing": { "type": "X-Amzn-Trace-Id", "value": "Root=..." }
}
```

**No event payload**. Extensions get metadata only. Cannot see the handler's input or output.

**SHUTDOWN event** includes `shutdownReason` (`FAILURE`, `TIMEOUT`, `SPINDOWN`) but no exception details.

**Runtime API Proxy pattern**: An extension CAN intercept the `/next` and `/response` Runtime API calls between the Lambda service and the runtime, which would give access to the full payload. But:
- AWS recommends implementing in Go/Rust (not Python) for cross-runtime compatibility
- Significant complexity (must proxy all HTTP calls between Runtime API and runtime)
- Maintenance burden for a custom extension
- Deployed as a Lambda layer, consuming one of 5 layer slots

**Verdict**: Way too much complexity for what we need. The AWS-native S3 destination does the same thing with zero code.

### 4. Lambda Wrapper Scripts (`AWS_LAMBDA_EXEC_WRAPPER`)

**How it works**: Set `AWS_LAMBDA_EXEC_WRAPPER=/opt/wrapper.sh` env var. Lambda executes wrapper script before starting the Python runtime.

**What it can do**: Modify runtime startup, add instrumentation flags, initialize tools.
**What it cannot do**: Intercept event payloads (operates at process level, not request level).

**Used by**: AWS Distro for OpenTelemetry (ADOT) for trace instrumentation, but doesn't capture payloads.

**Verdict**: Not useful for event capture. Better suited for observability instrumentation.

### 5. Python Decorator on `record_handler` (Code-Based Alternative)

If you want per-record failure capture WITHOUT waiting for all retries to exhaust:

```python
# Placement: outside @idempotent_function, inside Powertools decorators
@tracer.capture_method
@capture_failed_record       # <-- NEW
@idempotent_function(...)
def record_handler(record: DynamoDBRecord) -> dict[str, Any]:
    ...
```

```python
import json, traceback, boto3
from datetime import datetime, timezone

_s3_client = None

def capture_failed_record(fn):
    """Save failing DynamoDB stream records to S3 for later replay."""
    def wrapper(record):
        try:
            return fn(record)
        except Exception as e:
            global _s3_client
            if _s3_client is None:
                _s3_client = boto3.client("s3")
            _s3_client.put_object(
                Bucket=os.environ.get("FAILED_EVENTS_BUCKET", ""),
                Key=f"failed-records/valkey-queue-processor/{datetime.now(timezone.utc):%Y-%m-%d}/{record.event_id}.json",
                Body=json.dumps({
                    "event_id": record.event_id,
                    "event_name": str(record.event_name),
                    "sequence_number": record.dynamodb.sequence_number,
                    "new_image": record.dynamodb.new_image,
                    "old_image": record.dynamodb.old_image,
                    "exception": str(e),
                    "traceback": traceback.format_exc(),
                    "captured_at": datetime.now(timezone.utc).isoformat(),
                }),
            )
            raise  # CRITICAL: preserve batchItemFailures behavior
    return wrapper
```

**Pros**: Captures every failing record immediately (not just after retries exhausted), includes exception + traceback
**Cons**: Code change, requires S3 permissions, adds latency on failure path, must not swallow exceptions

**Decorator stacking matters**:
- Outside `@idempotent_function` → captures every attempt including retries
- Inside `@idempotent_function` → only captures first attempt (retries return cached result)
- For debugging, outside is better (captures all failures)

### 6. Local Replay Harness Options

#### Option A: Custom Script (Recommended — Simplest)

```python
#!/usr/bin/env python
"""Replay a failed Lambda event locally with optional debugpy."""
import json, sys, os

def main():
    event_file = sys.argv[1]
    debug = "--debug" in sys.argv

    if debug:
        import debugpy
        debugpy.listen(("0.0.0.0", 5678))
        print("Waiting for debugger on port 5678...")
        debugpy.wait_for_client()

    # Set env vars to match Lambda
    os.environ.setdefault("VALKEY_HOST", "localhost")
    os.environ.setdefault("VALKEY_PORT", "6380")
    os.environ.setdefault("VALKEY_USE_TLS", "false")
    os.environ.setdefault("QUEUE_TABLE_NAME", "work-queue")
    os.environ.setdefault("IDEMPOTENCY_TABLE_NAME", "dev-idempotency")
    os.environ.setdefault("ENVIRONMENT", "dev")

    with open(event_file) as f:
        raw = json.load(f)

    # Handle S3 on-failure wrapper (has "payload" key) or raw event
    event = json.loads(raw["payload"]) if "payload" in raw else raw

    # Mock Lambda context
    class MockContext:
        function_name = "filescience-dev-valkey-queue-processor"
        aws_request_id = "replay-local"
        memory_limit_in_mb = 512
        def get_remaining_time_in_millis(self): return 120_000

    from filescience.valkey_queue_processor.handler import lambda_handler
    result = lambda_handler(event, MockContext())

    failures = result.get("batchItemFailures", [])
    print(f"Result: {len(event.get('Records', []))} records, {len(failures)} failures")
    if failures:
        print(f"Failed items: {[f['itemIdentifier'] for f in failures]}")

if __name__ == "__main__":
    main()
```

**Usage**:
```bash
# Download failed event from S3
aws s3 cp s3://filescience-dev-failed-events/aws/lambda/.../event.json failed_event.json

# Replay locally
uv run python scripts/replay_lambda_event.py failed_event.json

# Replay with debugger
uv run python scripts/replay_lambda_event.py failed_event.json --debug
# Then attach VS Code debugger to port 5678
```

**Prerequisites**: SSM tunnel to Valkey (port 6380), AWS credentials for DynamoDB

#### Option B: python-lambda-local (Pip Package)

```bash
pip install python-lambda-local
python-lambda-local -f lambda_handler -t 120 \
  bases/filescience/valkey_queue_processor/handler.py \
  test/events/valkey_queue_processor_live_failure_event.json
```

**Pros**: Zero custom code, handles context mocking
**Cons**: May not support Python 3.13+, limited env var control, no debugpy integration

#### Option C: SAM Local Invoke (Docker-Based)

Requires a minimal `template.yaml`:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  ValkeyQueueProcessor:
    Type: AWS::Serverless::Function
    Properties:
      Handler: filescience.valkey_queue_processor.handler.lambda_handler
      Runtime: python3.12
      Timeout: 120
      MemorySize: 512
```

```bash
sam build && sam local invoke ValkeyQueueProcessor \
  -e test/events/valkey_queue_processor_live_failure_event.json \
  --env-vars env.json \
  -d 5678  # debug port
```

**Pros**: Most accurate runtime emulation, official AWS tool
**Cons**: Requires Docker, slower startup, SAM template maintenance, Docker networking for Valkey

#### Comparison

| Approach | Docker | Setup | Debug | Speed | Accuracy |
|----------|--------|-------|-------|-------|----------|
| Custom script | No | Minimal | debugpy | Fastest | Good (same Python, real AWS calls) |
| python-lambda-local | No | pip install | Limited | Fast | Good |
| SAM local | Yes | template.yaml | Excellent | Slow | Best (Lambda runtime) |

### 7. Partial Batch Failure Nuance

**Critical for this repo**: The `valkey-queue-processor` uses `ReportBatchItemFailures`. This means:

- **Individual record failures**: Returned in `batchItemFailures` response. Stream poller retries only those records. `lambda_handler` invocation shows as SUCCESS in CloudWatch.
- **Total batch failure**: Lambda invocation itself fails (unhandled exception, timeout, OOM). Stream poller retries entire batch.

**OnFailure destination fires when**: After `maximum_retry_attempts` exhausted, the batch (or sub-batch after bisection) is discarded. Both total failures AND persistent partial failures eventually trigger this.

**`bisect_batch_on_function_error = true`**: On total batch failure, splits batch in half and retries each half. Helps isolate poison records. Eventually a single poison record ends up alone in a batch, fails all retries, and goes to OnFailure destination.

## Code References

- `infrastructure/modules/lambda/main.tf:134-145` — DynamoDB event source mapping (missing destination_config)
- `infrastructure/modules/lambda/variables.tf:99-116` — Stream-related variables (no retry/destination vars)
- `infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor/terragrunt.hcl:76-79` — Stream trigger config
- `bases/filescience/valkey_queue_processor/handler.py:439-475` — `lambda_handler` (batch entry point)
- `bases/filescience/valkey_queue_processor/handler.py:296-435` — `record_handler` (per-record processor, wrapper target)
- `bases/filescience/valkey_queue_processor/handler.py:422-434` — Exception logging (recently added)
- `bases/filescience/valkey_queue_processor/handler.py:60` — `BatchProcessor` singleton
- `bases/filescience/valkey_queue_processor/handler.py:62-67` — Idempotency config
- `test/events/valkey_queue_processor_live_failure_event.json` — Existing test fixture
- `.vscode/launch.json` — Existing debugpy configs (discover only)
- `infrastructure/modules/artifact-bucket/main.tf` — Artifacts bucket (no failure bucket)

## Architecture Documentation

### Current Lambda Event Flow (No Failure Capture)
```
DynamoDB work-queue → Stream (NEW_AND_OLD_IMAGES)
  → Event Source Mapping (batch=100, ReportBatchItemFailures)
    → lambda_handler → process_partial_response → record_handler (per record)
      → Success: idempotency cached, work registered in Valkey
      → Failure: exception logged, added to batchItemFailures
    → Return batchItemFailures to stream poller
    → Stream poller retries failed records (unbounded)
    → Records eventually expire from stream (24h retention)
```

### Proposed Flow (With Failure Capture)
```
DynamoDB work-queue → Stream
  → Event Source Mapping (batch=100, max_retries=3, bisect=true)
    → lambda_handler → process_partial_response → record_handler
      → Failure: exception logged, batchItemFailures returned
    → Stream poller retries (up to 3 times, with bisection)
    → After max retries exhausted:
      → S3 bucket: full event payload + metadata
      → S3 Event Notification (optional): alert or auto-process
    → Developer workflow:
      → aws s3 cp s3://bucket/path event.json
      → python scripts/replay_lambda_event.py event.json [--debug]
      → Fix issue, deploy, records naturally replay from stream
```

### Layer Slot Budget
Current layers (3/5 used): Core, Valkey, Utils
Available: 2 slots — sufficient for any future layers, no extension layer needed since we're not building a custom extension.

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/reference/2026-02-11-lambda-failed-event-capture-replay-options.md` — Prior research covering AWS-native options (ESM OnFailure, async destinations, EventBridge archive, LocalStack Event Studio)
- `memory-bank/thoughts/shared/reference/2026-02-11-lambda-local-remote-debugging.md` — Remote debug (AWS Toolkit) + local debug (SAM) workflows
- `memory-bank/thoughts/shared/research/2026-02-11-valkey-queue-processor-batch-failure-hidden-errors.md` — Investigation that triggered this research: 36 poison records with minimal INSERT images missing `data` field
- `memory-bank/thoughts/shared/research/2026-02-04-dynamodb-stream-trigger-valkey-queue-processor.md` — Original stream trigger analysis
- `memory-bank/thoughts/shared/plans/2026-02-04-dynamodb-stream-trigger.md` — Stream trigger implementation plan

## Related Research

- `memory-bank/thoughts/shared/reference/2026-02-11-lambda-local-remote-debugging.md` — Complements this with breakpoint debugging workflows
- `memory-bank/thoughts/shared/research/2026-02-04-valkey-connection-failure-analysis.md` — Valkey connection issues that would be debuggable with replay

## External Sources

### AWS Documentation
- [Retain discarded records for DynamoDB event source](https://docs.aws.amazon.com/lambda/latest/dg/services-dynamodb-errors.html)
- [Gracefully handle failed Lambda events from DynamoDB Streams (AWS Blog)](https://aws.amazon.com/blogs/database/gracefully-handle-failed-aws-lambda-events-from-amazon-dynamodb-streams/)
- [Lambda Extensions API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html)
- [Lambda Telemetry API](https://docs.aws.amazon.com/lambda/latest/dg/telemetry-api.html)
- [Modifying the runtime environment (wrapper scripts)](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-modify.html)
- [Runtime API proxy extension (AWS Blog)](https://aws.amazon.com/blogs/compute/enhancing-runtime-security-and-governance-with-the-aws-lambda-runtime-api-proxy-extension/)
- [sam local invoke](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html)
- [Batch Processing — Powertools for AWS Lambda (Python)](https://docs.powertools.aws.dev/lambda/python/latest/utilities/batch/)

### Third-Party Tools
- [python-lambda-local (PyPI)](https://pypi.org/project/python-lambda-local/)
- [aws-lambda-context (PyPI)](https://pypi.org/project/aws-lambda-context/)
- [aws-samples/aws-lambda-extensions (GitHub)](https://github.com/aws-samples/aws-lambda-extensions)
- [LocalStack Event Studio](https://docs.localstack.cloud/user-guide/tools/event-studio/)

### Terraform
- [aws_lambda_event_source_mapping (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_event_source_mapping)

## Open Questions

1. **Failure bucket**: Dedicated `filescience-dev-failed-events` bucket, or prefix under existing `filescience-dev-artifacts`?
2. **Retry bounds**: What should `maximum_retry_attempts` be? Suggested: 2-3 (fast poison record isolation with bisect)
3. **Record age**: What should `maximum_record_age_in_seconds` be? Suggested: 3600 (1 hour) to avoid stale retries
4. **Alerting**: Should S3 Event Notification trigger an alert (SNS → email/Slack) when failed events land?
5. **Replay scope**: Should replay harness support single-record replay (extract one record from batch) or full-batch only?
6. **Entity-discovery-trigger**: Should the same capture/replay system apply? (Simpler case — direct invocation, no batch processing)
