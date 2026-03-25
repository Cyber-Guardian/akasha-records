---
type: event
created: 2026-03-17
status: active
tags: [observability, dev-harness, polylith, testing, mcp]
---
# Idea Brief: Universal Dev Harness (Component Dyno + Macro Process Harness)

**Date:** 2026-03-17
**Status:** Shaped -> Planning

## Problem
The dev-harness MCP server (`tools/dev-harness/`) is welded to the triage agent. `invoke_triage` is hardcoded, `_setup_env` sets triage-specific tables, `_reset_caches` clears triage-specific module caches. To dyno a different service (discover, VQP, EDT) or test a component in isolation with observability, there's no generic rig. The observability backend (VictoriaMetrics/Logs/Traces + Grafana + OTel Collector) is already service-agnostic, but the invocation and instrumentation layer is not.

## Constraints
- Handler signatures vary: Lambda `(event, context)` for EDT/VQP/triage, async process for discover
- Components are libraries with no natural invocation entry points — they're tested, not invoked
- The observability query tools (`query_traces`, `query_logs`, `query_metrics`, `check_stack`) are already generic
- Only `invoke_triage` and `_setup_env`/`_reset_caches` are service-specific
- `filescience.telemetry` already provides `capture_method`, `setup_telemetry`, `flush_telemetry` — any brick using it emits to OTel automatically
- `with_health_check` is shared across Lambda handlers
- Polylith's testing philosophy: test bricks in project context, not standalone — components don't run in isolation naturally
- MCP is the right abstraction because Claude Code is the primary consumer; human developers use `make` targets

## Options Considered

### Two-Mode Convention Harness
Each brick gets an optional `__harness__.py` with a standard protocol — two variants for components (test/function entry points) vs bases (handler invocation).
- Gains: Single convention, auto-discoverable, each brick opts in independently
- Costs: Two protocol variants to maintain. Component `functions()` export could become maintenance burden
- Complexity: Medium

### Layered — Tests-With-Telemetry (Dyno) + Generalized Invoke (Harness)
Component dyno: pytest plugin / make target that runs any brick's tests with OTel enabled pointing at local observability stack. No new convention needed. Macro harness: refactor `invoke_triage` into generic `invoke(service, event)` with per-service adapter modules in `tools/dev-harness/adapters/`.
- Gains: Component dyno requires zero per-brick work (tests already exist). Macro harness is a straightforward refactor. No new conventions to learn
- Costs: Two different mechanisms. Component dyno is tests-only (no ad-hoc function calls)
- Complexity: Low-Medium

### MCP-Native Process Runner
Both modes use subprocess execution. Spawns pytest or a handler CLI shim with OTel env vars injected.
- Gains: True isolation, no import-time side effects, no cache reset needed
- Costs: Subprocess overhead, loses in-process introspection, harder to debug failures
- Complexity: Low

## Chosen Approach
**Layered — Tests-With-Telemetry (Dyno) + Generalized Invoke (Harness)** — leans into the natural grain discovered during grounding: components and bases have fundamentally different invocation surfaces. Fighting that with a unified convention adds complexity without proportional value.

Two instruments feeding the same observability backend:

| Instrument | Analogy | What it does | Granularity |
|---|---|---|---|
| **Component Dyno** | Engine on a stand | Runs a brick's tests with full OTel instrumentation, exposes performance/trace/memory data via existing MCP query tools | Single component |
| **Macro Process Harness** | Full car on a rolling road | Invokes a complete base handler with realistic events, captures full request lifecycle | Full service |

## Key Context Discovered During Shaping

**Grounding findings:**
- Convention-based plugin discovery is validated for internal monorepos (PyPA guide, pytest conftest, prior plugin architecture brief [[2026-03-09-plugin-architecture-and-helm-intelligence|Plugin Architecture Brief]])
- MCP as instrumentation layer is validated specifically because Claude Code is the primary consumer (Sentry MCP monitoring, Honeycomb hosted MCP, MCP 2026 roadmap tasks feature)
- Component-level *invocation* through MCP is weak (no natural entry points), but component-level *instrumentation through tests* is strong
- Nobody is building exactly this — OpenAI Codex ephemeral-observability-per-worktree is the closest analog
- Polylith's own testing philosophy tests bricks in project context, not standalone

**Existing infrastructure that carries forward unchanged:**
- `development/observability/local/` Docker stack (VM suite + Grafana + OTel Collector)
- MCP query tools: `query_traces`, `query_logs`, `query_metrics`, `check_stack`
- `filescience.telemetry` component (`capture_method`, `setup_telemetry`, `flush_telemetry`)
- `development/ephemeral/` sandbox infra (moto, DynamoDB tables)

**What needs to change:**
- `tools/dev-harness/server.py` — refactor `invoke_triage` into generic `invoke(service, event)` with adapter pattern
- New: `tools/dev-harness/src/dev_harness/adapters/` — per-service adapter modules (triage first, then EDT, VQP, discover)
- New: `dyno(brick)` MCP tool — runs component tests with OTel env vars
- New: `make dyno brick=<name>` — human-facing equivalent
- Existing local runners (`event_pump.py`, `sandbox.py`) could become adapter implementations

**Handler signatures to abstract over:**
- EDT: `lambda_handler(event, context)` — event: `{"CloudId", "TenantId", "OrgId"}`
- VQP: `lambda_handler(event, context)` — event: DynamoDB Streams `{"Records": [...]}`
- Triage processor: `handler(event, context)` via `durable_execution` — event: `{"channel", "thread_ts", "text", "user"}`
- Discover: `DiscoveryManager.run()` — long-running async process, no event

## References
- [[2026-03-09-local-dev-feedback-loop|Local Dev Feedback Loop Research]]
- [[2026-03-09-plugin-architecture-and-helm-intelligence|Plugin Architecture Brief]]
- [[thoughts/shared/plans/2026-03-09-local-dev-harness|Local Dev Harness Plan (triage-specific)]]
- [[2026-03-17-agent-debug-toolkit|Agent Debug Toolkit Brief]]
- [[2026-03-16-autonomous-experiment-harness-v3|Autonomous Experiment Harness v3]]

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-universal-dev-harness.md`
