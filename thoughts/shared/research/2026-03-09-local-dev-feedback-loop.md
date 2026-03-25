---
date: 2026-03-09T12:00:00-05:00
researcher: Claude
git_commit: bbd6181c214a1a3d5fa9f5d3a24b5ed9f806306b
branch: main
repository: filescience
topic: "Local development infrastructure and feedback loop patterns"
tags: [research, codebase, ephemeral, observability, testing, local-dev]
status: complete
last_updated: 2026-03-09
last_updated_by: Claude
---

# Research: Local Development Infrastructure & Feedback Loop Patterns

**Date**: 2026-03-09
**Researcher**: Claude
**Git Commit**: bbd6181
**Branch**: main

## Research Question
How does the existing ephemeral environment, test harness, and local development infrastructure work together — and what patterns exist for local Lambda invocation, observability querying, and developer feedback loops?

## Summary

The codebase has a mature local dev stack with two independent capabilities that have never been connected: (1) an ephemeral AWS mock environment (moto + Valkey + Terragrunt-provisioned DynamoDB), and (2) a full OTel observability stack (VictoriaMetrics/Logs/Traces + Grafana + assertion helpers). Two local runners exist (`event_pump.py` for VQP, `sandbox.py` for discover) but the triage agent has none. No handler has ever been run locally with OTel instrumentation against the ephemeral stack — the two capabilities exist in parallel.

## Detailed Findings

### 1. Ephemeral Stack Provisioning

**Docker services** (`development/ephemeral/docker-compose.yaml`):
- `moto` (port 5050) — AWS mock server
- `valkey` (port 6399) — Redis-compatible store
- `valkey-bootstrap` — loads Lua throttle script
- `bootstrap` — creates S3 bucket + SSM params via `init.sh`
- Observability stack included via compose `include`

**DynamoDB tables** provisioned via Terragrunt (`development/ephemeral/infrastructure/`):
- `work-queue` (pk/sk, GSI for throttle context, DynamoDB Streams enabled)
- `resources` (pk/sk, GSI for parent link)
- `dev-idempotency` (pk only, TTL)

**Not provisioned**: triage agent tables (`triage-bot-threads`, `triage-bot-callbacks`, `triage-bot-idempotency`), triage S3 memory bucket/index, triage secrets.

**Extensibility pattern**: Add a subdirectory to `development/ephemeral/infrastructure/<table>/` with a `terragrunt.hcl` pointing at `infrastructure/modules//dynamodb`. The `terragrunt run --all` in `make ephemeral-up` picks it up automatically. S3/SSM resources go in `bootstrap/init.sh`.

### 2. Observability Harness

**Stack** (`development/observability/local/docker-compose.yaml`):
- OTel Collector (gRPC:4317, HTTP:4318, health:13133)
- VictoriaMetrics (PromQL:8428)
- VictoriaLogs (LogsQL:9428)
- VictoriaTraces (Jaeger API:10428)
- vmagent (scrapes collector + VM self-metrics)
- Grafana (port 3000, anonymous admin, 4 pre-built dashboards)

**Assertion helpers** (`test/harness/assertions.py`):
- `assert_metric(vm_url, query)` — PromQL with polling
- `assert_log_contains(vl_url, query)` — LogsQL with polling
- `assert_trace_exists(vt_url, service_name, operation)` — Jaeger API with polling

**Pytest fixture** (`test/harness/conftest.py`): Session-scoped `otel_stack` starts Docker, waits for health, yields endpoint dict, tears down with `docker compose down -v`.

**Pre-built dashboards**: home (service overview), discover, EDT, VQP. No triage agent dashboard.

### 3. Existing Base Testing Patterns

**Unit test pattern** (all bases): All external I/O replaced with `AsyncMock`/`MagicMock`. No moto at the base level. DynamoDB calls are mocked at the aioboto3 API surface. Only the `dynamodb` component tests use in-process `mock_aws`.

**No integration tests**: `@pytest.mark.integration` is registered but never used anywhere.

**Triage agent specifics**: `MockCheckpointContext` for durable execution, `FunctionModel` for LLM-free agent loop execution, stacked `@patch` decorators for all external deps.

### 4. Local Runner Patterns

| Handler | Local Runner | How It Works |
|---------|-------------|--------------|
| VQP | `event_pump.py` via `make ephemeral-pump` | Polls DynamoDB Streams, calls `lambda_handler` in-process with `MockLambdaContext` |
| Discover | `sandbox.py` via `make ephemeral-sandbox` | Instantiates `DiscoveryManager` directly (no Lambda handler) |
| EDT | `replay_lambda_event.py` | Replays captured S3 failure events with `MockLambdaContext` |
| Triage | **None** | No local runner exists |

**`MockLambdaContext`** exists in two places (`event_pump.py:34`, `replay_lambda_event.py:37`) with `function_name`, `aws_request_id`, `memory_limit_in_mb`, `invoked_function_arn`, and `get_remaining_time_in_millis()`.

**`replay_lambda_event.py`** (`scripts/replay_lambda_event.py`) uses a handler alias registry and `importlib` dynamic loading. Currently covers VQP and EDT only.

### 5. Handler Context Requirements

| Handler | Uses `context.function_name` | Uses `get_remaining_time_in_millis()` | Special context |
|---------|-----|-----|------|
| EDT | Via `@capture_failed_events` | No | Powertools `inject_lambda_context` |
| VQP | Via `@capture_failed_events` | Yes (logging + idempotency) | Powertools `inject_lambda_context` |
| Triage | No | No | `_adapt_context()` → `PassthroughContext` if no `.step` attr |

### 6. Configuration Patterns

- **Main bases**: `os.environ` at import time, `AWS_ENDPOINT_URL` for moto redirection (boto3 reads it natively)
- **Triage agent**: `get_secret()` via Secrets Manager (no local mode), `load_linear_config()` from YAML file, `os.environ` for bucket/prefix/model
- **OTel**: `setup_telemetry()` reads `OTEL_ENABLED`, `OTEL_EXPORTER_OTLP_ENDPOINT` env vars

## Architecture Documentation

### Two Unconnected Capabilities

The ephemeral stack and the observability stack are included together in Docker Compose but have never been used together for handler execution:
- `event_pump.py` and `sandbox.py` run handlers against moto but don't call `setup_telemetry()`
- The harness tests send synthetic OTel data but don't invoke handlers
- No handler in the codebase calls `setup_telemetry()` — it's only used in the telemetry component's own tests and the harness validation tests

### The Telemetry Component Is Ready But Unused by Handlers

`components/filescience/telemetry/core.py` provides `setup_telemetry()`, `capture_method`, `start_span`, `increment_counter`, `record_histogram`, `flush_telemetry`, `enable_log_correlation`. All no-op when `OTEL_ENABLED` is not set. The EDT handler uses `capture_method` and `flush_telemetry` from powertools, not the custom telemetry component. The VQP handler uses neither.

## Code References

- `development/ephemeral/docker-compose.yaml` — ephemeral stack definition
- `development/ephemeral/.env.ephemeral.defaults` — env var defaults
- `development/ephemeral/bootstrap/init.sh` — S3/SSM provisioning
- `development/ephemeral/infrastructure/root.hcl` — Terragrunt moto provider config
- `development/ephemeral/event_pump.py` — VQP local runner
- `development/ephemeral/run_event_pump.sh` — VQP runner shell wrapper
- `projects/discover/sandbox.py` — discover local runner
- `scripts/replay_lambda_event.py` — event replay script (VQP + EDT)
- `test/harness/conftest.py` — otel_stack session fixture
- `test/harness/assertions.py` — PromQL/LogsQL/Jaeger assertion helpers
- `test/harness/test_infra_validation.py` — harness validation tests
- `development/observability/local/docker-compose.yaml` — OTel stack
- `development/observability/local/otel-collector-config.yaml` — collector pipelines
- `components/filescience/telemetry/core.py` — OTel telemetry module
- `tools/bases/tooling/triage_agent/handler.py:56` — triage handler entry
- `tools/test/bases/triage_agent/test_handler.py` — MockCheckpointContext + FunctionModel patterns

## Historical Context
- [[2026-03-05-triage-agent-memory-access|Triage Agent Memory Access Plan]] — the plan that added memory tools to the triage agent
- [[2026-03-03-memory-mcp-server|Memory MCP Server Plan]] — prior MCP server design (Lambda-hosted, different pattern)

## Open Questions
- Should the existing `event_pump.py` and `replay_lambda_event.py` patterns be generalized rather than creating a new tool?
- Is `replay_lambda_event.py`'s handler alias registry the right extension point for adding triage agent support?
- Should `setup_telemetry()` be wired into handlers at the production level, or only in local dev runners?
