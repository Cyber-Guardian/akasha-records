---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Jig — Rust Agent Core with Provider Mesh

**Date:** 2026-03-20
**Status:** Shaped → Planning

## Problem
FileScience engineering is dependent on Claude Code — a closed, single-vendor CLI that's slow (Opus at 20-30 tok/s), expensive, and architecturally locked to Anthropic. Meanwhile, the team has already built most orchestration primitives (Helm for swarm coordination, workspace containers for isolation, autoresearch for autonomous experimentation, MCP for tool interfaces). The missing piece is a fast, provider-agnostic execution layer that ties it all together, routes intelligently across models, and eventually generates training data for fine-tuned models.

## Constraints
- Must be a full Claude Code replacement, not a wrapper
- Company internal tool under Engineering Leverage initiative
- Rust for the core (speed, single binary, memory safety)
- Must support existing MCP servers (9 currently configured)
- Must support existing workflow patterns (hooks, skills, agents, workspace containers)
- Provider mesh must route at the task level, not just individual API calls
- Telemetry must capture (prompt, tool_calls, result, feedback) tuples for future training
- RTK (Rust Token Killer) patterns and code should carry forward where possible

## Options Considered

### Forge — Full Rust Agent Core
Full Rust implementation of the agent loop, TUI, file operations, MCP client, context management, streaming. Provider mesh routes per-call based on task classification.
- Gains: Maximum speed, full control, single binary, RTK code carries forward
- Costs: 3-6 months to daily-driver parity. Agent loop quality depends on model cooperation.
- Complexity: High

### Conductor — Rust Orchestrator over SDK Workers
Rust binary handles TUI, file ops, routing. Agent loop runs through existing SDKs (Claude Agent SDK, OpenAI Agents SDK, Ollama HTTP) as workers.
- Gains: Leverages battle-tested SDKs. Faster to daily-driver status.
- Costs: Polyglot runtime. Subprocess overhead. Context sync between workers is lossy.
- Complexity: Medium

### Anvil — Fork OpenCode, Rewrite Hot Path in Rust
Fork MIT-licensed OpenCode (TS/Bun), replace hot path with Rust via FFI. Add provider mesh as custom router.
- Gains: Fastest path to daily-driver. 75 providers already wired. LSP built-in.
- Costs: Bun dependency. Fork maintenance burden. Architecture may not align.
- Complexity: Low-Medium

## Chosen Approach
**Forge (renamed Jig)** — Full Rust agent core with provider mesh. Own the entire stack. The agent loop is well-understood (prompt → tool calls → observe → repeat). The differentiators worth building from scratch: task-aware provider mesh, context management, and telemetry-as-training-data pipeline.

## Key Context Discovered During Shaping

- Claude Code speed: Sonnet is ~2-3x faster than Opus (exact tok/s figures are blog-sourced, not official Anthropic numbers). Codex on Cerebras Spark: 1,000+ tok/s (confirmed by Cerebras announcement), but with accuracy tradeoff — Terminal-Bench drops from 77.3% to 58.4% on Spark tier.
- OpenCode (sst/opencode): 127K stars, MIT, TS/Bun, 75 providers. Claude Code ~45% faster on wall-clock in morphllm.com benchmark (methodology caveats: different validation depths, OpenCode ran more tests).
- RTK already proves Rust can intercept and optimize the CLI layer (60-90% token savings)
- AlphaEvolve pattern: Flash for volume, Pro for depth — 57-80% cost reduction via model tiering
- No existing tool does task-aware model routing at the agent loop level
- Autoresearch generates (hypothesis, code_diff, metric_score) tuples — natural fine-tuning data
- Session telemetry generates (user_intent, tool_sequence, file_changes, outcome) — RLHF-style data
- Agent Runtime Substrate brief exists for ECS Fargate execution — self-hosted models could run there

## Related Prior Work
- [[2026-03-19-agent-runtime-substrate|Agent Runtime Substrate (ECS Fargate)]]
- [[2026-03-17-universal-dev-harness|Universal Dev Harness]]
- [[2026-03-20-akasha-capability-platform|Akasha Capability Platform]]
- [[2026-03-18-deep-autoresearch-architectures-survey|Autoresearch Architectures Survey]]
- [[2026-03-12-agent-harness-composition|Agent Harness Composition]]

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-jig-rust-agent-core.md`
