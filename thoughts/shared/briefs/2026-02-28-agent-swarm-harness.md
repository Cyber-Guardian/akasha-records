# Idea Brief: Agent Swarm Harness (Dev Server)

**Date:** 2026-02-28
**Status:** Shaped → Planning

## Problem
Agent execution currently runs through Cyrus (Linear's managed Claude Agent SDK). We can't pick models, control cost, run persistent agents, enforce hard guardrails programmatically, or build management hierarchies. We have a beefy Windows server in the office (Spectrum Business static IP: 208.105.25.26, Docker + WSL2 + Linux dual-boot) that could run the entire agent stack at near-zero marginal cost per session. The goal is to own compute, orchestration, guardrails, and quality control — replacing Cyrus entirely.

## Constraints
- Dev server is Windows with Docker, WSL2, and Linux dual-boot — Claude Agent SDK runs natively on Linux/macOS, so WSL2 or Docker is the execution environment
- Server is on Spectrum Business internet (static IP) — needs to reach Anthropic API, GitHub, Linear API
- Anthropic API billing replaces Linear's Cyrus billing — cost model shifts from "included in Linear" to "pay per token"
- All existing guardrails (CLAUDE.md, hooks, scope_guard, import-linter) must work in the new execution environment
- FileScience-specific initially, with intent to generalize later
- Server specs unknown — need to validate capacity (cores, RAM) before sizing concurrent agent count

## Options Considered

### CLI Swarm (shell orchestrator)
Use `claude -p` (Claude Code CLI non-interactive mode) as the agent runtime. Script on dev server spawns multiple `claude` processes in separate worktrees, each with different `--model` flags and injected system prompts. Coordination via filesystem and git.
- Gains: Reuses everything built (CLAUDE.md, hooks, skills, scope_guard). Minimal new code. Helm plan directly usable.
- Costs: Less programmatic control than SDK. Model selection per-session not per-turn. Inter-agent communication indirect.
- Complexity: Medium

### Python SDK Orchestrator
Build a Python service using Claude Agent SDK directly. Orchestrator spawns/manages agent sessions programmatically — model selection, token budgets, context injection, inter-agent communication all in code.
- Gains: Total control. Inline guardrails. Per-turn model selection. Direct inter-agent communication.
- Costs: Building an agent framework. Crash handling, retries, state persistence.
- Complexity: Medium-High

### Docker Compose Agent Fleet
Each agent type (worker, judge, manager) is a Docker container. Workers use SDK. Judges use lighter models. Manager orchestrates via Docker API.
- Gains: Clean isolation. Resource limits per agent. Can mix LLM providers.
- Costs: Docker orchestration complexity. Overkill at current scale (5-10 agents, not 50).
- Complexity: High

## Chosen Approach
**CLI Swarm (Phase A) → SDK Orchestrator (Phase B)** — Start with `claude -p` on the dev server because it reuses all existing hooks, skills, CLAUDE.md, and scope_guard immediately. The dev server just becomes the machine running parallel Claude Code sessions in worktrees. As CLI limitations are hit (per-turn model selection, inline guardrails, inter-agent communication), graduate to Python SDK orchestrator. Docker fleet is premature.

## Architectural Vision (full stack, phased)
```
JUDGES    — taste/opinion agents reviewing high-leverage outputs
MANAGERS  — helm + overviewer, pick optimal LLM per task, invoke judges
GUARDRAILS — deterministic symbolic reasoning, hard rules, fail-closed
WORKERS   — Claude Code sessions in worktrees, scoped to file sets
COMPUTE   — dev server (Windows + WSL2/Docker)
```

Phase A (this plan) covers: COMPUTE + WORKERS + existing GUARDRAILS.
Phase B (future) covers: MANAGERS with per-turn model selection + inline guardrails.
Separate briefs exist for JUDGES and enhanced GUARDRAILS.

## Key Context Discovered During Shaping
- Cyrus runs Claude Agent SDK with `settingSources: ["project"]` — all hooks fire, CLAUDE.md loads. Self-hosted CLI should behave identically.
- `claude -p` supports `--model` flag for model selection per session
- Claude Agent SDK Python/TS supports programmatic session creation
- Existing helm orchestration plan (2026-02-27) covers decomposition, delegation, monitoring, merge ordering — designed for Cyrus but portable to self-hosted
- Nightly scheduled agents (ENG-2183, parked) is a direct consumer of this infrastructure
- Server hostname: `syn-208-105-025-026.biz.spectrum.com` (Spectrum Business, Charter Communications)

## Related Briefs
- [[thoughts/shared/briefs/2026-02-27-agent-helm-orchestration|Agent Helm Orchestration]] — workflow layer (decompose, delegate, monitor, merge). Absorbs into Manager layer.
- [[2026-02-28-judge-taste-agents|Judge/Taste Agents]] — quality review layer. Becomes Judges layer.
- [[2026-02-28-complexity-as-moat|Complexity-as-Moat]] — strategic rationale for this investment.
- [[2026-02-25-deterministic-guardrail-layer|Deterministic Guardrail Layer]] — hard guardrail infrastructure. Becomes Guardrails layer.

## Next Step
- Plan → [2026-02-28-agent-swarm-harness.md](../../plans/2026-02-28-agent-swarm-harness.md) ✅ CREATED
- Macro strategy cross-ref: [2026-02-28-macro-agent-strategy.md](../../plans/2026-02-28-macro-agent-strategy.md) Phase 3 — update "decisions pending" to resolved
