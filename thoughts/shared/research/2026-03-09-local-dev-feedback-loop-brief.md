---
date: 2026-03-09T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-09-local-dev-feedback-loop.md
last_generated: 2026-03-09T15:55:17.990569+00:00
---

# Research Brief: 2026-03-09-local-dev-feedback-loop

## TL;DR

The codebase has a mature local dev stack with two independent capabilities that have never been connected: (1) an ephemeral AWS mock environment (moto + Valkey + Terragrunt-provisioned DynamoDB), and (2) a full OTel observability stack (VictoriaMetrics/Logs/Traces + Grafana + assertion helpers). Two local runners exist (`event_pump.py` for VQP, `sandbox.py` for discover) but the triage agent has none. No handler has ever been run locally with OTel instrumentation against the ephemeral stack — the two capabilities exist in parallel.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Should the existing `event_pump.py` and `replay_lambda_event.py` patterns be generalized rather than creating a new tool?
- Is `replay_lambda_event.py`'s handler alias registry the right extension point for adding triage agent support?
- Should `setup_telemetry()` be wired into handlers at the production level, or only in local dev runners?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-09-local-dev-feedback-loop.md`
