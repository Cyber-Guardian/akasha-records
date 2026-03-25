---
type: event
created: 2026-03-21
status: active
tags: [dev-harness, agent-runtime, observability, pydantic-ai]
---
# Idea Brief: Generic Agent Runner for Dev Harness

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
Every new pydantic-ai agent (triage bot, Caroline, future Akasha agents) requires bespoke infrastructure wiring — OTEL setup, tool registry resolution, error handling, channel integration. The dev harness has per-agent adapters that duplicate boilerplate. Debugging agents means tailing log files instead of seeing structured traces in Grafana.

## Constraints
- `AgentConfig` in `handler.py:81-105` is already 90% declarative (system_prompt, tools, model, max_turns)
- `build_agent()` resolves tools by string name from a registry — the pattern exists
- `run_agent()` in the durable harness is fully platform-agnostic
- Existing `ServiceAdapter` ABC handles Lambda services (discover, EDT, VQP) with unique infrastructure needs — those stay unchanged
- pydantic-ai has first-class OTEL via `logfire.instrument_pydantic_ai()` — one call instruments everything
- Channel choreography (typing indicators, pacing, friction) requires real code, not YAML config — the manifesto proved this
- The dev harness registry auto-discovers adapters by convention — extensible

## Options Considered

### Thin Runner + Fat Channel Adapters
Minimal runner loads config, constructs agent, hands result to channel adapter. Channel adapter owns all channel-specific behavior.
- Gains: Runner stays simple. Channel feel preserved. New agent = YAML file.
- Costs: Channel adapters still ~200 lines each. Config doesn't capture choreography well.
- Complexity: Medium

### Config-All-The-Way (CrewAI-style)
Agent YAML + Channel YAML. Even choreography is config.
- Gains: Maximum declarative surface. Zero Python for new agents.
- Costs: Timing choreography becomes a DSL. Manifesto anti-patterns require contextual judgment. CrewAI tried this and it's still incomplete.
- Complexity: High

### Instrumented Base Class + Convention (Chosen)
`BaseAgent` Python class with auto-OTEL, tool registry, conversation state, channel hooks. New agents subclass and override 3 methods. Dev harness discovers by convention.
- Gains: Python is the config — IDE support, type checking. OTEL wires once. Channel choreography stays real code. Follows existing ServiceAdapter pattern.
- Costs: ~50 lines per agent vs ~20 lines YAML. But the code is meaningful.
- Complexity: Low-Medium

## Chosen Approach
**Instrumented Base Class + Convention** — any pydantic-ai agent gets auto-OTEL traces in Grafana, the dev harness can invoke it, and channel output is a separate plug. Code over config because interaction feel requires contextual logic.

## Key Context Discovered During Shaping
- Ground verdict: idea holds for pydantic-ai agents, doesn't replace Lambda service adapters
- No mainstream framework has config-driven construction + tool registry + pluggable channels + auto-OTEL combined
- pydantic-ai `logfire.instrument_pydantic_ai()` is one-call instrumentation following GenAI semantic conventions
- Caroline proved the channel adapter pattern: `LinqClient` + `Choreographer` IS the channel adapter
- Triage bot swap analysis: 4 clean swap points (receiver, normalizer, result poster, human-wait callback)
- Forge plan (2026-03-12) is a different layer (multi-agent composition) that would consume this

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-generic-agent-runner.md`
