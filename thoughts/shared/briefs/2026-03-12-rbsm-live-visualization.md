---
type: event
created: 2026-03-12
status: active
---
# Idea Brief: RBSM Live State Space Visualization

**Date:** 2026-03-12
**Status:** Shaped -> Planning

## Problem
When the Hypothesis RBSM runs, it explores the state space across hundreds of examples with 30-100 steps each, but there's no way to see what it actually explored. You can't tell which transitions are hot vs. cold, whether Hypothesis is stuck in shallow loops, or how deep the exploration goes. The `target()` labels guide Hypothesis internally but aren't surfaced. `--hypothesis-show-statistics` only shows example-level aggregates, not per-step or per-transition data.

The result: you can't answer "is the oracle actually covering interesting territory?" without reading Hypothesis debug output or guessing from pass/fail.

## Constraints
- Hypothesis has no per-step structured hook -- step sequences are only available as pre-formatted strings in the observability `representation` field
- The observability callback (`add_observability_callback`, v6.137.0+) fires once per example, not per rule
- No existing tools or plugins visualize RBSM runs as graphs -- the gap is real
- Our RBSM already tracks state in the `ThreadOracle` dataclass -- we know state after each rule, just don't emit it
- Hypothesis is timing-sensitive -- heavy per-step overhead could trigger health checks or distort `target()` guidance
- Test environment mocks OTel (`_setup_otel`/`_flush_otel` are patched out) -- no collector running during tests

## Options Considered

### Custom Capture + Websocket Graph
Instrument RBSM rules with a lightweight decorator or base class method that emits `(from_state, rule_name, to_state)` tuples. Transport via websocket to a browser page rendering a live force-directed graph (D3/vis.js). Nodes = states, edges = transitions, edge weight/thickness = frequency.
- Gains: Purpose-built data shape, zero infrastructure deps, minimal per-step overhead (append to list + websocket send), live feedback during dev runs
- Costs: Custom visualization frontend to build and maintain, websocket server to run alongside pytest
- Complexity: Medium

### OpenTelemetry Spans
Each rule execution as an OTel span with `from_state`/`to_state` attributes. Aggregate spans into a graph via a custom Grafana panel or standalone frontend.
- Gains: Reuses existing telemetry component, could extend to production state machine observability later
- Costs: OTel spans model timelines not graphs -- need custom projection anyway. Collector adds latency. Thousands of spans per test run. Test env doesn't run a collector. Still need a custom frontend since Jaeger/Tempo render timelines, not node-edge graphs
- Complexity: High -- impedance mismatch between OTel's data model and state graph visualization

### Hypothesis Observability JSONL Post-Processing
Use `HYPOTHESIS_EXPERIMENTAL_OBSERVABILITY=1` to write JSONL, then parse `representation` strings post-run to extract step sequences and build a static graph.
- Gains: Zero instrumentation -- uses built-in Hypothesis feature. No runtime overhead beyond file I/O
- Costs: Not live -- post-run only. Requires parsing pre-formatted pseudo-Python strings. No structured per-step data
- Complexity: Low

## Chosen Approach
**Custom Capture + Websocket Graph** -- The user wants live visualization (watching the graph build in real-time as Hypothesis explores), which rules out post-processing. OTel's data model is a poor fit for state graphs and adds infrastructure overhead in the test environment. Custom capture gives purpose-built data at minimal cost, with a websocket transport that enables real-time rendering.

## Key Context Discovered During Shaping
- Hypothesis `add_observability_callback` (v6.137.0+, `hypothesis.internal.observability`) is internal API -- not stable across versions
- `_stateful_repr_parts` collects step strings during a run but isn't exposed as structured data
- The `representation` field in observability output joins all steps as a newline-delimited pseudo-Python string -- parseable but fragile
- Our `ThreadOracle` dataclass already tracks `.state` per thread -- instrumentation can read it directly after each rule
- `target()` scores appear in observation `features` dict -- could overlay on the graph as metadata
- Production state machine visualization (real Lambda invocations) is a separate future concern -- OTel would be the right tool for that, but doesn't need to influence this test-time tool

## Related
- [[2026-03-11-model-based-testing-state-machines|Model-Based Testing with State Machines Brief]]
- [[2026-03-11-testing-infrastructure-portfolio|Testing Infrastructure Portfolio Brief]]
- `.claude/rules/model-based-testing.md` -- process rule for RBSM development
- `.claude/reference/state-machine-testing-guide.md` -- full RBSM pattern guide
- `tools/test/bases/triage_agent/test_thread_state_machine.py` -- existing RBSM (first consumer)

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-12-rbsm-live-visualization.md`
