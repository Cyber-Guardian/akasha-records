# Idea Brief: Deep Research PydanticAI Harness

**Date:** 2026-03-05
**Status:** Shaped -> Planning

## Problem
Deep research output documents are thin — summaries of summaries rather than substantive analytical documents. The root cause is architectural: a 345-line prompt skill can't enforce multi-phase workflows, the 150-word distillation protocol loses nuance needed for rich synthesis, and all sessions converge at 3 iterations regardless of depth budget. No amount of prompt engineering fixes a program-shaped problem implemented as a prompt.

## Constraints
- Needs `ANTHROPIC_API_KEY` (or other provider key) in environment — same pattern as helm CLI
- Must integrate with QMD MCP server for internal memory-bank search
- Must support codebase search (not just web search) — our research is often internal
- Should produce documents compatible with current memory-bank format (markdown with YAML frontmatter)
- Lives in `tools/deep-research/` following the helm project pattern
- The existing `/deep_research` Claude Code skill becomes a thin shim invoking the harness

## Options Considered

### Adapt STORM's pattern into the existing Claude Code skill
Rewrite the 345-line prompt to add a conversation phase, outline generation, and section-by-section writing. Keep everything as prompt engineering.
- Gains: No new dependencies, no API key requirement, works within Claude Code natively
- Costs: Still fighting the fundamental constraint — a prompt can't enforce its own rules. The 3-iteration ceiling and distillation lossyness are structural, not prompt-level problems.
- Complexity: Medium

### Claude Agent SDK harness
Build a Python harness using Anthropic's Agent SDK. Implement STORM's two-stage architecture with programmatic control.
- Gains: Native to Claude, direct API access, Anthropic-maintained
- Costs: Locked to Anthropic models, no built-in graph/state machine (hand-roll it), no structured output validation with retry
- Complexity: Medium-High

### PydanticAI harness (chosen)
Build a Python harness using PydanticAI agents + pydantic-graph. STORM's two-stage architecture as the blueprint. Multi-model support, typed state machine, MCP integration, structured output validation.
- Gains: Model-agnostic, pydantic-graph gives typed state machine (research -> verify -> write nodes), structured output with validation/retry, MCP integration for QMD, Logfire observability, FileStatePersistence for resume-across-sessions, multi-model cost optimization (cheap for research, expensive for writing)
- Costs: New dependency (pydantic-ai), needs API key in environment, runs outside Claude Code context
- Complexity: Medium

### GPT-Researcher or LangGraph as substrate
Use an existing framework as the foundation and adapt it.
- Gains: Battle-tested (25K+ stars), existing web search pipelines
- Costs: Web-search-only by default, significant adaptation needed for codebase/QMD research, LangChain ecosystem baggage, fighting their architecture rather than building ours
- Complexity: High

## Chosen Approach
**PydanticAI harness** — best balance of architectural control, type safety, and ecosystem fit. pydantic-graph is the key differentiator: it makes STORM's multi-phase architecture a typed state machine rather than prompt instructions the model might ignore. Multi-model support enables STORM's cost optimization (cheap models for research conversations, expensive for synthesis). MCP integration connects our QMD and codebase tools directly.

## Key Context Discovered During Shaping
- STORM's two-stage separation (research conversations -> outline-driven writing) is the gold standard for rich output — the conversation phase builds deep understanding before any writing happens
- Our current system's 3-iteration ceiling is caused by binary gap tracking and gap-on-first-encounter closure — graduated confidence in typed state fixes this
- Subagent distillation (150 words) is the #1 cause of thin output — isolated agent conversations with full context don't need it
- PydanticAI v1.66 (March 2026), 15.3K stars, 402 contributors, shipping fast
- pydantic-graph supports FileStatePersistence for pause/resume — replaces our manifest markdown files
- Weizhena/Deep-Research-skills is the only other Claude Code deep research extension, but lacks gap tracking or iteration loops
- [[2026-03-05-deep-improve-the-deep-research|Meta-research on improving deep research]] identified 6 weaknesses that this harness addresses

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-05-deep-research-pydantic-harness.md`
