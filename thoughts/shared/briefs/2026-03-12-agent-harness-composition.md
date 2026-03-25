---
type: event
created: 2026-03-12
status: active
---
# Idea Brief: Agent Harness Composition via Cognitive Personalities

**Date:** 2026-03-12
**Status:** Shaped → Planning

## Problem
Current agent frameworks (LangGraph, CrewAI, OpenAI Agents SDK) compete on task routing and state management — "who gets the task next?" None encode cognitive diversity: different thinking styles (convergent vs divergent, constrained vs free-form) that productively tension each other. Agents today either follow instructions linearly or operate as homogeneous swarms. There's no composition primitive for "these two agents think differently and must agree before the work is done."

Meanwhile, the core agent loop (tool-use iteration + structured outputs) has commoditized. MCP solved tool connectivity. Memory patterns are well-known. The next competitive surface is **how you compose agents with different cognitive profiles into a harness that achieves outcomes humans couldn't efficiently supervise.**

## Constraints
- Must bootstrap using existing tools (Claude Code, Agent SDK, helm) — can't build from scratch
- Personality configs must be runtime-agnostic — valuable as an abstraction, not tied to one agent runtime
- Need machine-checkable goal definitions, not vibes-based "is this done?"
- Consensus protocols must have deadlock resolution (escalate to human)
- Feedback loops are task-type-dependent — visual work needs screenshots, code needs tests, infra needs health checks

## Options Considered

### Claude Code Personality Configs (Bootstrap)
Build personality definitions as Claude Code configuration bundles (rules, tools, hooks, skills). Use helm as the composition layer to wire personalities together. Swap runtimes later.
- Gains: Leverages everything that exists today. Immediate. Testable with real tasks.
- Costs: Coupled to Claude Code initially. Helm needs evolution to support cognitive roles + consensus.
- Complexity: Medium

### Custom Agent SDK Harness (Build from Scratch)
Build a standalone Python harness using Claude Agent SDK with custom orchestration for cognitive roles, goals, and consensus.
- Gains: Clean architecture. Full control. Not constrained by Claude Code's config model.
- Costs: Significant upfront investment. Duplicates working infrastructure. Delays practical use.
- Complexity: High

### Framework Plugin (LangGraph/CrewAI Extension)
Build the cognitive composition layer as a plugin on an existing framework.
- Gains: Large ecosystem. Community. Existing state management.
- Costs: Coupled to framework's opinions. Limited by their primitives. Can't differentiate.
- Complexity: Medium

## Chosen Approach
**Claude Code Personality Configs (Bootstrap)** — start with what works. Define personalities as composable Claude Code configuration bundles. Evolve helm (or build a thin layer) to support cognitive role composition, goal evaluation, and consensus protocols. Prove the concept with a concrete use case (e.g., website-building with creative + pragmatist). The personality config format and harness definitions are the portable value — the runtime is swappable.

## Key Context Discovered During Shaping
- Agent harness landscape has converged on 4 topologies (hierarchical, mesh, DAG, hybrid) — none encode cognitive diversity
- Claude Code's config system (CLAUDE.md, rules, tools, hooks, skills, agents/) already provides the levers for personality differentiation
- Helm already decomposes work into phases and dispatches to subagents — needs evolution from task-routing to cognitive-role-routing
- Docker analogy: personality config = Dockerfile, harness definition = docker-compose, runtime = swappable
- Existing brief on CI feedback loops ([[2026-02-26-agent-ci-feedback-loop|CI Feedback Loop]]) covers the reactive case; this idea covers proactive goal-based verification
- Prior agent harness research ([[2026-03-04-deep-advanced-agent-harness-techniques-brief|Harness Techniques Brief]]) confirms the gap in cognitive composition

## Key Primitives Identified
1. **Agent personalities** — cognitive profiles as configuration bundles (rules, tools, constraints, thinking style)
2. **Goal definitions** — machine-checkable acceptance criteria attached to the harness, not individual agents
3. **Rail sets** — scoped action spaces per personality (what the agent can and cannot do)
4. **Consensus protocols** — how differently-configured agents resolve disagreement
5. **Feedback loop auto-detection** — task type → appropriate verification mechanism

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-12-agent-harness-composition.md`
