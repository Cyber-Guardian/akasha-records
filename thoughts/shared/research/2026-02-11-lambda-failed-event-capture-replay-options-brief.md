---
date: 2026-02-11T17:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-11-lambda-failed-event-capture-replay-options.md
last_generated: 2026-02-11T17:49:31.685150+00:00
---

# Research Brief: 2026-02-11-lambda-failed-event-capture-replay-options

## TL;DR

Three viable approaches exist, ranked by complexity and fit for this repo:

1. **AWS-native ESM OnFailure → S3** (recommended first step) — Zero code changes, Terraform-only. Full event payload retained in S3 for stream-triggered Lambdas. Requires adding `destination_config`, retry bounds, and bisect controls to the Lambda module.

2. **Custom Python replay harness** — A ~50-line script that reads captured events from S3/local JSON and invokes `lambda_handler` or `record_handler` directly, with optional `debugpy` attachment. No Docker, no SAM, no heavy tooling.

3. **Lambda extension/layer wrapper** — Possible but overkill. Extensions API cannot intercept event payloads directly (metadata only). A Runtime API proxy extension could, but requires Go/Rust and significant complexity. A Python decorator on `record_handler` is simpler and achieves the same goal.

**Bottom line**: The AWS-native path + a thin replay script covers 95% of the need. A custom extension is not warranted.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Failure bucket**: Dedicated `filescience-dev-failed-events` bucket, or prefix under existing `filescience-dev-artifacts`?
2. **Retry bounds**: What should `maximum_retry_attempts` be? Suggested: 2-3 (fast poison record isolation with bisect)
3. **Record age**: What should `maximum_record_age_in_seconds` be? Suggested: 3600 (1 hour) to avoid stale retries
4. **Alerting**: Should S3 Event Notification trigger an alert (SNS → email/Slack) when failed events land?
5. **Replay scope**: Should replay harness support single-record replay (extract one record from batch) or full-batch only?
6. **Entity-discovery-trigger**: Should the same capture/replay system apply? (Simpler case — direct invocation, no batch processing)

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-11-lambda-failed-event-capture-replay-options.md`
