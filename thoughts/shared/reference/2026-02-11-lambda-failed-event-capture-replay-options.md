# Lambda Failed Event Capture + Replay Options (2026-02-11)

## Question
Do tools already exist to record/store failed Lambda events and replay them locally, or should we build custom?

## Short Answer
Yes, tools exist. For AWS-hosted workloads, native AWS features cover most needs.  
For this repo's DynamoDB-stream Lambda, the strongest built-in path is **event source mapping OnFailure -> S3** (full failed payload retained), then replay locally with a small script/debug harness.

## Existing Options

### 1) Lambda Event Source Mapping OnFailure Destinations (DynamoDB/Kinesis/Kafka)
- Relevant to this repo (`filescience-dev-valkey-queue-processor` is DynamoDB stream triggered).
- Configure on the event source mapping with:
  - `destination_config.on_failure.destination`
  - `maximum_retry_attempts`
  - `maximum_record_age_in_seconds`
  - optional `bisect_batch_on_function_error`
- Destination choices:
  - **S3**: includes metadata **and full invocation payload** (`payload`) for replay.
  - **SQS/SNS**: metadata only (no full stream payload), so replay requires re-fetching from stream before retention expires.
- This is currently the best native "capture + durable store + later replay" option for stream-triggered Lambdas.

### 2) Lambda Async Invocation Destinations / DLQ (for async invokes)
- Works for asynchronously invoked Lambdas (not stream poller semantics).
- Destinations support SQS/SNS/S3/EventBridge/Lambda; record includes request/response context.
- DLQ exists but is less rich than destinations.

### 3) EventBridge Archive + Replay
- Useful when events already flow through EventBridge.
- Can archive and replay events by time range/rules.
- Less direct for DynamoDB stream-triggered Lambda unless architecture routes through EventBridge.

### 4) LocalStack Event Studio (third-party)
- Provides event inspection/edit/replay in local emulation.
- Useful for local event-driven debugging workflows.
- Not a native production capture mechanism for live AWS failures.

### 5) OSS point solutions
- Some repos exist for narrow pipelines (for example S3/SNS/Lambda or Kinesis replay), but no widely adopted universal tool that cleanly handles all trigger types + storage + replay orchestration.

## Fit For This Repo
- Current infra (`infrastructure/modules/lambda/main.tf`) defines DynamoDB event source mapping, but does **not** yet expose:
  - `destination_config` (OnFailure target)
  - bounded retries/record age
  - batch bisection controls
- Therefore, the capture/replay capability is not currently wired in IaC, even though AWS supports it natively.

## Recommendation
Do **not** build a large custom platform first.

Use a thin approach:
1. Configure DynamoDB stream ESM OnFailure destination to **S3**.
2. Add bounded retry controls + optional bisection.
3. Add a tiny local replay utility that reads S3 failed payload objects and invokes `lambda_handler` under `debugpy`.
4. Keep Toolkit remote debug for final cloud-only validation.

This yields durable capture, deterministic replay, and fast local breakpoints with minimal custom code.

## Sources
- AWS Lambda docs: Retain discarded records for DynamoDB event source  
  https://docs.aws.amazon.com/lambda/latest/dg/services-dynamodb-errors.html
- AWS Lambda docs: Capturing async invocation records  
  https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-retain-records.html
- AWS Database Blog: S3 as on-failure destination for DynamoDB stream consumers  
  https://aws.amazon.com/blogs/database/gracefully-handle-failed-aws-lambda-events-from-amazon-dynamodb-streams/
- EventBridge archive + replay docs  
  https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive.html
- LocalStack Event Studio docs  
  https://docs.localstack.cloud/user-guide/tools/event-studio/
