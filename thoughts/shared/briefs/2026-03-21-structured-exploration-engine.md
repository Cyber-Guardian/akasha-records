---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Structured Exploration Engine (MCP)

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
Every agent in the system that needs context does the same thing poorly: ad-hoc search, grab whatever comes back, dump it into the prompt. Deep research sessions, sub-agents, the autoresearch thinker, general Claude Code sessions — they all need the same capability: search broadly and thoughtfully across a large corpus, then distill what matters from what doesn't. Nobody does this well because it's treated as a throwaway step rather than a first-class operation.

The current `/deep_research` skill is a 345-line prompt that can't enforce its own rules. Prior research (March 5th) identified 6 structural weaknesses: 3-iteration convergence ceiling, binary gap tracking, no self-critique, no QMD integration, thin distilled output, over-engineered mid-run steering. This is a program-shaped problem implemented as a prompt.

## Constraints
- Must be an MCP server — composable with everything in the ecosystem
- Must work against multiple corpora: codebase (files), memory-bank (QMD/graph-memory), web, experiment logs
- Must return curated output — not raw search results, but a distilled context package with provenance
- Must be fast enough to be a building block, not a 10-minute ceremony
- Standalone — no dependency on Claude Code session context
- Lives in `tools/` following existing MCP patterns (FastMCP like workspace-mcp, graph-memory, dev-harness)
- Registered in `.mcp.json`
- The existing `/deep_research` skill becomes a thin shim calling this MCP

## Three-Phase Architecture

### Phase 1: Strategize (evolving)
"What angles do we need to explore, and why?" — A living exploration plan that gets re-evaluated as findings come in. New angles emerge. Old angles get deprioritized. The plan gets grounded and alignment-checked against the original question after every exploration cycle. Forces the system to think at the meta level — not "what query should I run" but "is my exploration strategy still correct given what I've learned?"

### Phase 2: Explore (multi-corpus)
"Go search for X from angle Y using tools Z." — Structured tactical search across multiple corpora with multiple query strategies per angle. Deterministic backbone ensures you don't just keyword-match and stop.

### Phase 3: Curate (distill)
"Of everything we found, what matters?" — Separate signal from noise. Produce a curated context package with provenance. Not a summary — a selection of the most relevant findings with enough context to be useful.

The loop between Phase 1 and Phase 2 is continuous. Every time Phase 2 returns findings, Phase 1 re-evaluates. The deterministic backbone enforces that you can't explore without a plan, and you can't curate without having reflected on what you found.

## Callers
- `/deep_research` skill -> thin shim over this tool
- Sub-agents (plan-implementer, researcher) -> call it to gather context before acting
- Autoresearch thinker -> survey experiment history + codebase before planning
- Any agent -> "I need to understand X before I act"

## Options Considered

### Upgrade the prompt skill to leverage new MCP tools
Rewrite the 345-line prompt to call QMD, graph-memory, and use the agent taxonomy better. Keep everything as prompt engineering.
- Gains: No new infrastructure, works within Claude Code natively
- Costs: Still a prompt that can't enforce its own rules. Can't be called by other systems (autoresearch, sub-agents). The 3-iteration ceiling and distillation lossiness are structural.
- Complexity: Low

### Standalone PydanticAI harness (prior brief, March 5th)
Build a Python harness using PydanticAI agents + pydantic-graph. STORM's two-stage architecture. Multi-model, typed state machine.
- Gains: Full programmatic control. Model-agnostic. Typed state.
- Costs: Needs its own API key for internal LLM calls. Runs outside Claude Code context. Two-layer LLM problem. Doesn't compose as a building block for other agents.
- Complexity: High

### MCP server with three-phase deterministic backbone (chosen)
FastMCP server implementing strategize/explore/curate as a composable primitive. The MCP owns state and phase enforcement. Claude (or any caller) provides judgment within each phase. No internal LLM calls — the caller's LLM does the thinking, the MCP enforces the structure.
- Gains: Composable (any MCP client can call it). Deterministic enforcement of the three phases. Integrates with QMD/graph-memory as library calls. No separate API key. Fast — it's infrastructure, not a second brain. Reusable across deep research, sub-agents, autoresearch.
- Costs: The caller's LLM still drives judgment — but now within enforced structure. More complex than a prompt rewrite.
- Complexity: Medium

## Chosen Approach
**MCP server with three-phase deterministic backbone** — the structured exploration engine is a research primitive, not a deep research replacement. Deep research composes on top of it. So does everything else that needs context.

## Key Context Discovered During Shaping
- Prior deep research meta-analysis identified 6 structural weaknesses — all addressable by moving to programmatic control ([[2026-03-05-deep-improve-the-deep-research|Deep Research Meta-Analysis]])
- Prior brief chose PydanticAI harness but never planned/built it ([[2026-03-05-deep-research-pydantic-harness|PydanticAI Harness Brief]])
- Autoresearch thinker architecture has the exact pattern needed: deterministic coordinator forcing strategic/tactical alternation ([[2026-03-19-autoresearch-wave3-thinker-architecture|Thinker Architecture Brief]])
- Multi-experiment research planning deep research provides the theoretical grounding for the three-phase model ([[2026-03-19-deep-multi-experiment-research-planning|Multi-Experiment Research Planning]])
- New infrastructure since March 5th: workspace MCP (parallel execution), graph-memory (structural discovery), QMD (522-doc semantic search), 12 specialized agents, helm orchestration
- The three phases map to: autoresearch thinker (strategize), autoresearch researcher (explore), distillation protocol (curate)

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-structured-exploration-engine.md`
