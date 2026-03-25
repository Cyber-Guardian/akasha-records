---
date: 2026-03-02T15:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state.md
last_generated: 2026-03-02T15:55:32.782025+00:00
---

# Research Brief: 2026-03-02-multi-agent-orchestration-and-macro-strategy-state

## TL;DR

There are two distinct but deeply interrelated threads, and they have **diverged in maturity and concreteness**:

**Thread 1 — Plan-Execute-Review Workflow** is **operational and battle-tested**. The full lifecycle (`/shape` → `/create_plan` → `adversarial-plan-reviewer` → `plan-implementer` subagent → `/validate_plan` → `/complete_plan`) is implemented and in active use. The `/resolve-linear-issue` Tech Lead Orchestrator pattern (PR #7) chains these together for Linear issues. Cyrus (autonomous agent) has completed 10+ issues using this workflow.

**Thread 2 — Macro Agent Strategy** is a **5-phase strategic roadmap** (brief + plan dated 2026-02-28) that subsumes and extends thread 1. It envisions three "legs" — tournament ideation, parallel agent execution (helm orchestration), and visual human review — that compose into a development system. Phase 1 has two ready-to-execute implementation plans, but no phases have started. The revised helm orchestration plan (2026-03-02) incorporates multi-agent architecture research.

**The divergence**: Thread 1 is the working reality. Thread 2 is the strategic vision that builds on thread 1 but has not started executing. The macro strategy references the existing workflow as infrastructure but layers on helm decomposition, concurrent Cyrus agents, scope isolation validation, tournament ideation, and visual review — none of which exist yet beyond plans/briefs.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **ENG-2129/2130 vs /helm** — The alignment checker → orchestrator agent progression (Linear issues) and the `/helm` skill (macro strategy) aim at overlapping outcomes. Which is the actual next step? Are they complementary or competing?
2. **Scope validation gap** — The scope enforcement chain (`cyrus-setup.sh` → `generate_scope.py` → `scope_guard.py`) exists but has never been validated end-to-end in a production Cyrus session. This is Phase 1 of the revised helm plan and is a hard gate.
3. **When does Phase 1 start?** — The macro strategy and both implementation plans (visualization + helm validation) are marked "ready to execute" but no phase has started. The brief says "Parked."
4. **Swarm harness vs helm** — The macro plan includes an "Agent Swarm Harness" (dev server for agent lifecycle) in Phase 3, but the revised helm plan operates entirely through Linear/Cyrus delegation. Is the swarm harness still relevant, or does the helm approach replace it?
5. **ENG-2128 completion** — The parent execution discipline issue is "In Progress" with 5/6 sub-issues done. ENG-2136 (prod environment) is deferred. Should ENG-2128 be closed?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-02-multi-agent-orchestration-and-macro-strategy-state.md`
