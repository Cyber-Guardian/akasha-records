---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Autonomous Experiment Harness v3 (Hybrid Agent Topology)

**Date:** 2026-03-16
**Status:** Shaped -> Planning

## Problem
PR #107 implemented the autoresearch harness using `claude -p` subprocess invocations for both the lab leader and researcher roles. The subprocess approach is too unstructured — fragile stdout JSON parsing, no typed tool use, imprecise token tracking. The core loop (research -> hypothesize -> implement -> measure -> select winner -> iterate) is sound, but the agent invocation layer needs to match the role: structured orchestration for the lab leader, free-form coding for the researchers.

## Constraints
- Lab leader needs typed, validated outputs (`LabLeaderDecision`) — a decision-making agent, not a coder
- Researchers need full coding agent capabilities (file ops, web search, codebase navigation) — reimplementing these as custom tools would be worse than Claude Code's battle-tested implementations
- PydanticAI is the established agent framework in this codebase (triage-agent patterns)
- Claude Agent SDK is new to the repo but runs locally (prior Lambda rejection doesn't apply — that was about 12s cold start + no checkpointing in durable Lambda)
- Evaluator must remain outside the agent boundary as an immutable subprocess
- First use case: dedup/compression optimization

## Options Considered

### PydanticAI Leader + Claude Agent SDK Workers (Hybrid Native)
PydanticAI for the structured lab leader, Agent SDK for free-form researcher coding agents. Minimal MCP integration.
- Gains: Best runtime per role. Leader gets Pydantic validation + FunctionModel testing. Researchers get full Claude Code.
- Costs: Two dependencies. Agent SDK is new. Researcher testing requires mocking `query()`.
- Complexity: Medium

### PydanticAI for Everything (Homogeneous)
Both roles use PydanticAI. Researchers get custom `@agent.tool` functions for file I/O.
- Gains: Single framework. FunctionModel testing for both. Consistent patterns.
- Costs: Must reimplement file ops as custom tools — worse than Claude Code's native tools. Researcher becomes "PydanticAI agent that writes files" not "coding agent."
- Complexity: Medium-High

### PydanticAI Leader + Agent SDK Workers with MCP Bridge
Same hybrid as option 1, but researchers get additional in-process MCP tools for experiment discipline: structured result capture, strategy doc serving, file-write boundary enforcement.
- Gains: Everything from option 1, plus tool-enforced guardrails. Leader can lazily load context via tools. MCP bridge reusable for future harnesses.
- Costs: Two dependencies. More upfront MCP tool work.
- Complexity: Medium-High

### Raw Anthropic SDK + Agent SDK (Minimal)
Skip PydanticAI for the leader. Raw `messages.create()` with manual tool loop.
- Gains: No PydanticAI dependency for a simple role.
- Costs: Loses FunctionModel testing. Hand-rolls boilerplate. Diverges from triage-agent patterns.
- Complexity: Low-Medium

## Chosen Approach
**PydanticAI Leader + Claude Agent SDK Workers with MCP Bridge** — the natural split validated by every production multi-agent system studied. Structured framework for the structured role, coding agent for the coding role. MCP bridge enforces experiment discipline at the tool level.

## Key Context Discovered During Shaping

**Grounded patterns from production systems:**
- Anthropic's multi-agent research system: Opus lead (extended thinking, prompt-constrained) + Sonnet workers (free search). 90.2% improvement over single-agent.
- AlphaEvolve: Non-LLM evolutionary controller + Gemini Flash/Pro code generators. Evaluator fleet runs separately.
- AI Scientist v2: Experiment Manager (BFTS, config-driven) + implementation agents (sandboxed code execution). Structured orchestrator decides search topology, workers execute freely within scoped briefs.
- Darwin Godel Machine: Archive + MAP-Elites selection logic + foundation model code mutators.

**MCP-as-guardrail is an emerging pattern:**
- PolicyLayer/Intercept: YAML policy proxy at MCP transport layer (default-deny, argument validation, rate limits)
- mcp-scan: Content filtering and code analysis proxy
- No published in-process MCP guardrail examples yet — our approach would be novel at the in-process level

**Codebase compatibility:**
- PydanticAI already in use (triage-agent): `Agent()`, `@register_tool`, `RunContext`, `FunctionModel`
- Agent SDK not yet in repo but compatible — both depend on `anthropic` SDK (0.84.0 resolved)
- Prior Agent SDK rejection was Lambda-specific; autoresearch runs locally as a daemon
- Agent SDK gives structured output (`output_format`), token tracking (`ResultMessage.usage`), custom MCP (`create_sdk_mcp_server`)

**Related systems in the autonomous code optimization space:**
- pi-autoresearch: Living context doc (`autoresearch.md`) for agent continuity across context resets
- AutoKernel: 5-stage correctness harness before benchmarking, ~40 experiments/hour
- OpenAI self-evolving agents: Metaprompt agent generates improved prompts using failure reasoning

## References
- [[2026-03-14-autonomous-experiment-harness|Original Brief]]
- [[thoughts/shared/plans/2026-03-14-autonomous-experiment-harness|Original Plan (v2)]]
- [[2026-03-04-deep-advanced-agent-architectural-patterns-brief|Agent Architectural Patterns Research]]
- [[2026-03-04-deep-advanced-agent-harness-techniques-brief|Agent Harness Techniques Research]]
- PR #107: https://github.com/Cyber-Guardian/filescience/pull/107
- Anthropic multi-agent research: https://www.anthropic.com/engineering/multi-agent-research-system
- AlphaEvolve: https://arxiv.org/abs/2506.13131
- AI Scientist v2: https://github.com/SakanaAI/AI-Scientist-v2
- Darwin Godel Machine: https://sakana.ai/dgm/
- PolicyLayer/Intercept: https://github.com/PolicyLayer/Intercept
- Claude Agent SDK: https://platform.claude.com/docs/en/agent-sdk/overview

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-autonomous-experiment-harness-v3.md`
