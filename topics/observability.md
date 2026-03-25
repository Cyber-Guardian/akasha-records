---
topic: Observability
status: active
touched: 2026-03-17
related:
  - thoughts/shared/plans/2026-02-25-observability-vm-stack-harness-assertions.md
  - thoughts/shared/plans/2026-03-17-universal-dev-harness.md
---

# Observability

## Current State
Local observability stack is operational: VictoriaMetrics (metrics:8428), VictoriaLogs (logs:9428), VictoriaTraces (traces:10428), Grafana (:3000), OTel Collector (gRPC:4317, HTTP:4318). Managed via `make otel-local-up/down`. Pre-built Grafana dashboards for discover, EDT, VQP, home.

Dev-harness MCP server (`tools/dev-harness/`) exposes observability queries to Claude Code: `query_traces`, `query_logs`, `query_metrics`, `check_stack`. Currently triage-agent-specific for invocation — universal harness plan in progress.

Instrumentation varies by service:
- `discover`: OTel SDK via `filescience.telemetry`
- `valkey-queue-processor`: Powertools Tracer (X-Ray) — needs migration to OTel
- `entity-discovery-trigger`: OTel (`capture_method` from `filescience.telemetry`)
- `triage-agent`: OTel via `_setup_otel()` in handler, gated on `OTEL_ENABLED`

Shared component: `components/filescience/telemetry/` provides `setup_telemetry`, `capture_method`, `flush_telemetry`.

**Planned:** Universal dev harness (2026-03-17) — component dyno (instrumented pytest runs) + macro process harness (generic service invocation via adapter pattern). 4 phases.

## Key Decisions
- VictoriaMetrics suite over LGTM stack (lighter footprint, single-binary, OTLP-native)
- Keep Powertools Logger (only removing Tracer from VQP)
- Production ADOT layer deferred — local dev + test harness only
- `filescience.telemetry` as shared component

## Open Questions
- VictoriaMetrics stack plan is backlogged
- VQP still on X-Ray/Powertools Tracer — needs migration to OTel

## Artifacts
- `components/filescience/telemetry/` — shared telemetry component
- `development/observability/local/docker-compose.yaml` — VM suite + Grafana + OTel Collector
- `tools/dev-harness/` — MCP dev-harness server (observability queries + service invocation)
- `development/ephemeral/` — moto + Valkey + DynamoDB sandbox
