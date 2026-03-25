---
date: 2026-03-02T11:34:34-05:00
author: Claude
git_commit: 3476c2632fbe5eb1d5f87fe51108414ef2c60f0a
branch: main
repository: filescience
topic: "Helm Orchestrator Project Setup Handoff"
tags: [handoff, orchestration, helm, multi-agent, linear-project]
status: complete
last_updated: 2026-03-02
last_updated_by: Claude
---

# Handoff: Helm Orchestrator — research, shaping, and Linear project setup

## Task(s)

1. **Research: multi-agent orchestration & macro strategy state** — COMPLETED
   - Reconciled two diverging threads: (1) operational plan-execute-review workflow, (2) planned 5-phase macro agent strategy
   - Surveyed 35 memory-bank docs, Linear issues, recent git PRs/commits, Claude Agent SDK capabilities

2. **Shape: Helm Orchestrator CLI (Option C)** — COMPLETED
   - Decision: own the macro (deterministic Python CLI), let Cyrus own the micro (agent runtime)
   - Layered approach: V1 = execution layer, V2 = design/tournament layer, V3 = custom execution runtime
   - Chosen over: building on Claude Code markdown skills (Option A), monolithic orchestrator service

3. **Linear project setup** — COMPLETED
   - Created "Helm Orchestrator" project with 3 milestones and 6 V1 issues with dependency ordering

## Critical References

- **Brief**: `memory-bank/thoughts/shared/briefs/2026-03-02-helm-orchestrator-cli.md` — the shaped decision and rationale
- **Research**: `memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state.md` — full state-of-the-world analysis with 5 open questions
- **Multi-agent architecture survey**: `memory-bank/thoughts/shared/research/2026-03-01-multi-agent-architecture-research.md` — 14 systems surveyed, key findings (CooperBench, Gas Town)

## Recent changes

No code changes — this session was research, shaping, and Linear project setup. Files written:
- `memory-bank/thoughts/shared/briefs/2026-03-02-helm-orchestrator-cli.md` (new)
- `memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state.md` (new)
- `memory-bank/durable/01-active/current_work.md` (updated — Helm Orchestrator now top of Current Focus)

## Learnings

1. **The macro split**: We control the macro (deterministic Python orchestration in `tools/helm/`), Cyrus/Claude Code controls the micro (agent runtime, worktrees, hooks, plan-implementer). This was the key decision.

2. **Claude Agent SDK**: `settingSources: ["project"]` is how Cyrus loads CLAUDE.md + agents + hooks. SDK spawns CLI subprocess. One-level subagent nesting is a hard limit. API key billing only (no Max plan). Agent Teams is a separate experimental CLI feature, not SDK.

3. **Existing scope chain is built but unvalidated**: `cyrus-setup.sh` -> `generate_scope.py` -> `scope_guard.py` exists and is wired, but has never been tested end-to-end in a real Cyrus session. This is the Phase 1 hard gate (ENG-2228).

4. **Helm has two modes**: Design mode (local, SDK-powered tournament ideation — V2) and Execution mode (Linear-mediated decompose/delegate/monitor/merge — V1). The user wants the helm to eventually own both thinking and coordination.

5. **V3 milestone added**: Custom execution runtime to replace Cyrus when its constraints become the bottleneck (token visibility, session control, model choice, cost tracking).

6. **Old Agent Orchestration project**: Still exists on Linear (paused). The Helm Orchestrator project supersedes it but the old project's issues (ENG-2128 chain, ENG-2129/2130) are still relevant — especially ENG-2129 (alignment checker) which may complement or be subsumed by the helm.

## Artifacts

### Written this session
- `memory-bank/thoughts/shared/briefs/2026-03-02-helm-orchestrator-cli.md` — shaped brief
- `memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state.md` — full research doc
- `memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state-brief.md` — auto-generated brief (via hook)
- `memory-bank/durable/01-active/current_work.md` — updated

### Linear artifacts created
- **Project**: [Helm Orchestrator](https://linear.app/filescience/project/helm-orchestrator-50bee88b78f9)
- **Milestones**: V1: Execution Layer, V2: Design Layer, V3: Custom Execution Runtime
- **Issues**:
  - [ENG-2228](https://linear.app/filescience/issue/ENG-2228) — Validate scope enforcement chain e2e (High, co-dev)
  - [ENG-2229](https://linear.app/filescience/issue/ENG-2229) — Build helm CLI foundation + plan template (High, co-dev)
  - [ENG-2230](https://linear.app/filescience/issue/ENG-2230) — Build decomposition engine (High, co-dev, blocked by 2228+2229)
  - [ENG-2231](https://linear.app/filescience/issue/ENG-2231) — Build status dashboard + conflict detection (Medium, co-dev, blocked by 2230)
  - [ENG-2232](https://linear.app/filescience/issue/ENG-2232) — Build async GH Action monitor (Medium, co-dev, blocked by 2230)
  - [ENG-2233](https://linear.app/filescience/issue/ENG-2233) — Build merge orchestration (Medium, co-dev, blocked by 2231)

### Key pre-existing artifacts referenced
- `memory-bank/thoughts/shared/plans/2026-03-02-agent-helm-orchestration.md` — revised helm plan (execution architecture carries over, runtime changes to Python CLI)
- `memory-bank/thoughts/shared/plans/2026-02-28-macro-agent-strategy.md` — 5-phase strategic roadmap
- `memory-bank/thoughts/shared/briefs/2026-02-28-macro-agent-strategy.md` — macro strategy brief (three legs)
- `memory-bank/thoughts/shared/plans/2026-02-25-resolve-linear-issue-orchestrator.md` — Tech Lead Orchestrator pattern (operational)
- `memory-bank/thoughts/shared/research/2026-03-01-multi-agent-architecture-research.md` — 14-system survey

## Action Items & Next Steps

1. **Commit artifacts to main** — the brief, research doc, and current_work.md updates are uncommitted
2. **Start ENG-2228 or ENG-2229** — these can run in parallel:
   - ENG-2228: Validate scope chain e2e (requires real Cyrus delegation — co-dev)
   - ENG-2229: Build helm CLI foundation (architecture decisions needed — co-dev, good candidate for `/create_plan`)
3. **Consider `/create_plan` for ENG-2229** — the CLI foundation needs architectural decisions (CLI framework, Linear client design, config schema, manifest format). The revised helm plan (2026-03-02) has detailed phase designs that can be adapted.
4. **Decide on old Agent Orchestration project** — pause/archive it, or keep ENG-2129/2130 as separate future work that the helm may subsume

## Other Notes

- All V1 issues are labeled `co-dev` — the user wants to be in the loop for architectural decisions
- The user explicitly chose to defer symbolic verification (Z3) and tournament ideation to V2 — don't pull those forward
- The user's framing: "the helm is a human CLI tool, mostly used by Claude Code via a skill/command" — design for both interfaces
- The research doc has 5 open questions worth reviewing before planning starts (ENG-2129 overlap, scope validation gap, swarm harness relevance, etc.)
