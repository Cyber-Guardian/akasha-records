# Idea Brief: Observability — VM Full Stack + Harness Assertions

**Date:** 2026-02-25
**Status:** Shaped → Planning

## Problem
FileScience has partial observability (OTel SDK in discover only, local Jaeger+Prometheus, no logs backend, no unified dashboard) and no machine-verifiable telemetry assertions. Lambda handlers (VQP, EDT) don't emit to the local stack. Production debugging relies on manual CloudWatch log trawling. There is no way to write tests that assert against runtime behavior ("p95 < 2s", "no ERROR spans").

## Constraints
- All three polylith projects must be instrumented (discover, valkey-queue-processor, entity-discovery-trigger)
- Python OTel SDK already vendored — keep OTLP as the export protocol
- OTel Collector stays as pipeline layer (OTLP-native, already configured)
- No Kubernetes — local stack runs via Docker Compose
- Lambda cold-start sensitivity — instrumentation must be lightweight
- Cardinality guardrails from Feb 12 research: no tenant IDs, resource IDs, or user IDs as default metric labels
- `make otel-local-up/down` targets referenced in docs but missing from Makefile

## Options Considered

### Fill Gaps in Current Stack (Loki + Grafana)
Keep Jaeger + Prometheus. Add Loki for logs, Grafana for unified view. Extend instrumentation.
- Gains: Simplest change, well-documented LGTM stack
- Costs: Loki heavier operationally (needs chunk storage), 4 separate backends, resource-hungry — works against ephemeral spin-up
- Complexity: Low

### VM Full Stack — Swap Backends Only
Replace Jaeger → VictoriaTraces, Prometheus → VictoriaMetrics, add VictoriaLogs + Grafana. Keep OTel Collector. Extend telemetry to all projects.
- Gains: All three signal pillars, lighter footprint, PromQL/LogQL/Jaeger APIs, foundation for Harness patterns
- Costs: Learn VM operational model, swap 2 containers + add 2
- Complexity: Medium

### VM Full Stack + Harness Assertions (chosen)
Same as above, plus: pytest fixtures that spin up VM stack ephemerally and assert against PromQL/LogQL queries. Machine-verifiable acceptance criteria for all projects.
- Gains: Telemetry as test contract. Agents can self-verify. Per-branch CI observability possible. Full Harness Engineering adoption (runtime enforcement layered on top of existing structural enforcement via import-linter).
- Costs: More upfront work — assertion framework + Docker orchestration in tests. VM stack must boot fast (~5s).
- Complexity: High

## Chosen Approach
**VM Full Stack + Harness Assertions** — The lighter VM footprint is what makes ephemeral spin-up practical. OTel Collector stays (no OTLP friction). Instrumentation extends to all three projects. Grafana unifies the view. The assertion framework turns telemetry into a testable contract — the same shift OpenAI made with Harness Engineering.

Vector was explicitly rejected: it solves a problem FileScience doesn't have (massive log transformation at scale) and creates one it does (OTLP incompatibility with existing SDK).

## Key Context Discovered During Shaping
- `development/observability/local/docker-compose.yaml` — current 3-container stack (OTel Collector, Jaeger, Prometheus)
- `bases/filescience/discover/telemetry.py` — existing OTel helper module (spans, counters, histograms, log correlation, no-op when disabled)
- `bases/filescience/discover/manager.py:27` — telemetry imports already wired (start_span, increment_counter, record_duration)
- OTel Collector logs pipeline exists but exports to `debug` only (no queryable backend)
- `make otel-local-up/down` referenced in `docs/observability/discover-local-telemetry.md` but missing from Makefile
- Feb 12 research (`memory-bank/thoughts/shared/reference/2026-02-12-opentelemetry-cloudwatch-prometheus-grafana-integration.md`) defined canonical resource attributes, cardinality policy, and 4-phase rollout — Phase 0 partially complete
- VictoriaMetrics suite: single-binary, no external storage deps, 3.7x less RAM than Tempo (VictoriaTraces), PromQL/LogQL/Jaeger-compatible queries
- OpenAI Harness Engineering: ephemeral per-worktree VM stacks, agents query own telemetry, machine-verifiable acceptance criteria via PromQL/LogQL
- Lambda Powertools already provides structured JSON logging in VQP/EDT — OTel integration layer needed, not a rewrite

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-25-observability-vm-stack-harness-assertions.md`
