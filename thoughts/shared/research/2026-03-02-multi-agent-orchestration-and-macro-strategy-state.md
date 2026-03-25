---
date: 2026-03-02T15:00:00-05:00
researcher: Claude
git_commit: 3476c2632fbe5eb1d5f87fe51108414ef2c60f0a
branch: main
repository: filescience
topic: "Multi-Agent Orchestration & Macro Development Strategy — Current State"
tags: [research, codebase, multi-agent, orchestration, macro-strategy, plan-execute-review, helm, cyrus]
status: complete
last_updated: 2026-03-02
last_updated_by: Claude
---

# Research: Multi-Agent Orchestration & Macro Development Strategy — Current State

**Date**: 2026-03-02T15:00:00-05:00
**Researcher**: Claude
**Git Commit**: 3476c26
**Branch**: main
**Repository**: filescience

## Research Question
Where do the two threads — (1) multi-agent orchestration and (2) the macro development strategy (plan → execute → review) — currently stand, and how do they relate?

## Summary

There are two distinct but deeply interrelated threads, and they have **diverged in maturity and concreteness**:

**Thread 1 — Plan-Execute-Review Workflow** is **operational and battle-tested**. The full lifecycle (`/shape` → `/create_plan` → `adversarial-plan-reviewer` → `plan-implementer` subagent → `/validate_plan` → `/complete_plan`) is implemented and in active use. The `/resolve-linear-issue` Tech Lead Orchestrator pattern (PR #7) chains these together for Linear issues. Cyrus (autonomous agent) has completed 10+ issues using this workflow.

**Thread 2 — Macro Agent Strategy** is a **5-phase strategic roadmap** (brief + plan dated 2026-02-28) that subsumes and extends thread 1. It envisions three "legs" — tournament ideation, parallel agent execution (helm orchestration), and visual human review — that compose into a development system. Phase 1 has two ready-to-execute implementation plans, but no phases have started. The revised helm orchestration plan (2026-03-02) incorporates multi-agent architecture research.

**The divergence**: Thread 1 is the working reality. Thread 2 is the strategic vision that builds on thread 1 but has not started executing. The macro strategy references the existing workflow as infrastructure but layers on helm decomposition, concurrent Cyrus agents, scope isolation validation, tournament ideation, and visual review — none of which exist yet beyond plans/briefs.

## Detailed Findings

### Thread 1: Plan-Execute-Review Workflow (Operational)

#### The Skill Chain

The plan-execute-review lifecycle is implemented as 6 slash commands and 1 subagent:

| Step | Skill/Agent | File | Purpose |
|------|------------|------|---------|
| 1. Shape | `/shape` | `.claude/commands/shape.md` | Explore problem space, frame approaches |
| 2. Plan | `/create_plan` | `.claude/commands/create_plan.md` | Interactive research → implementation plan |
| 3. Review | `adversarial-plan-reviewer` | `.claude/agents/adversarial-plan-reviewer.md` | PASS/BLOCK verdict (score ≥9 required) |
| 4. Execute | `plan-implementer` | `.claude/agents/plan-implementer.md` | Opus subagent, one phase at a time |
| 5. Validate | `/validate_plan` | `.claude/commands/validate_plan.md` | Verify implementation against plan |
| 6. Close | `/complete_plan` | `.claude/commands/complete_plan.md` | Memory-bank hygiene, Linear closeout |

Supporting skills: `/iterate_plan` (revise existing plans), `/implement_plan` (standalone execution escape hatch).

#### The Orchestrator Pattern (ENG-2152, PR #7)

`/resolve-linear-issue` is the "Tech Lead Orchestrator" that chains the above:

```
Fetch issue → Understand → Branch → Shape/Plan/Direct decision
  → /create_plan (with ## Linear Context)
  → "In Progress" (after plan approval)
  → For each phase: dispatch plan-implementer subagent
  → Manual verification per phase
  → Create PR (parent context)
  → "In Review"
  → /complete_plan
```

Key design decisions (from brief `2026-02-25-resolve-linear-issue-orchestrator.md`):
- Main context owns understanding + planning + closeout (interactive work)
- Implementation phases delegated to `plan-implementer` Opus subagent
- Plan file carries Linear metadata as state handoff medium
- Sequential phase dispatch (not parallel) — phases are interdependent
- "In Progress" set after plan approval, not after understanding
- Simple issues can implement directly without formal planning

#### Enforcement Infrastructure

| Layer | Mechanism | Status |
|-------|-----------|--------|
| Soft guardrails | CLAUDE.md route table + anti-rationalization rules | Active |
| Agent discipline | `.claude/reference/agent-discipline.md` | Active |
| Scope guard | `.claude/hooks/scope_guard.py` (PreToolUse) | Dormant (`["**"]`) |
| Code quality | ruff + ty + import-linter (PostToolUse) | Active |
| Task gate | `task_completed_gate.sh` (TaskCompleted) | Active |
| CI feedback | `.github/workflows/agent-ci-feedback.yml` | Active, Phase 1.1 fix deployed |

#### Cyrus Track Record

Cyrus has completed 10+ issues autonomously using this workflow:
- ENG-2172 through ENG-2178 (observability phases 1-7)
- ENG-2180, ENG-2181, ENG-2184, ENG-2185, ENG-2187, ENG-2203, ENG-2216 (CI fixes)

Notable incident: ENG-2176 — Cyrus bundled 3 issues into 1 PR ("scope creep"). This led to the Agent Scope Discipline addition to CLAUDE.md (commit `8684a3d`).

---

### Thread 2: Macro Agent Strategy (Planned, Not Started)

#### The Three Legs

The macro strategy (brief: `2026-02-28-macro-agent-strategy.md`, plan: same date) defines a three-legged system:

**Leg 1 — Tournament Ideation**: Generate N approaches, judge agents rank them, execute the winner.
- Status: **Seed briefs only** (judge-taste-agents, complexity-as-moat)
- Research finding: Pairwise comparison + execution-based evaluation works; LLM debate doesn't
- Deferred to Phase 4 of roadmap

**Leg 2 — Agent Execution (Helm)**: Decompose → delegate → monitor → merge.
- Status: **Shaped, planned, revised** (3 plan iterations: 02-27, 02-28 macro, 03-02 revised)
- Key research: CooperBench shows 2 coordinating agents = half the success of 1 → agents must NOT coordinate, helm coordinates
- Revised plan (03-02) discovered `cyrus-setup.sh` + `generate_scope.py` already exist for scope auto-population
- 6-phase implementation plan ready to execute

**Leg 3 — Visual Review**: Humans evaluate via diagrams/graphs, not prose.
- Status: **Shaped + 7-phase plan** (visualization-in-shaping-planning, 02-27)
- Components: YAML graph spec → Python layout → Excalidraw + SVG
- Memory-Bank OS (Obsidian vault + MOCs) is the meta-level surface

#### The 5-Phase Roadmap

| Phase | Name | Dependencies | Status |
|-------|------|-------------|--------|
| 1 | Visual Pipeline + Helm Validation | None | Plans ready, not started |
| 2 | Helm Decomposition + Visual Status | Phase 1 | Planned |
| 3 | Autonomous Execution + Swarm Harness | Phase 2 | Swarm harness is seed brief only |
| 4 | Tournament Ideation | Phases 1+3 | Seeds only, research done |
| 5 | Integration + Memory Infrastructure | Phases 1-4 | Brief only (Memory-Bank OS) |

#### The Moat Thesis

The strategic rationale ("complexity-as-moat") argues the three legs compound:
- Tournament → higher quality designs than single-shot
- Parallel execution → higher velocity than serial
- Visual review → faster human verification than prose
- A competitor needs all three legs to match; each alone is not a moat

---

### How The Threads Relate

```
Thread 1 (operational)          Thread 2 (planned)
========================        ========================
/shape                          ← unchanged, used as-is
/create_plan                    ← adds ### Files section for helm parsing
adversarial-plan-reviewer       ← unchanged
plan-implementer                ← unchanged, dispatched by helm too
/resolve-linear-issue           ← continues for single issues
                                + /helm for multi-agent decomposition
scope_guard.py (dormant)        ← activated by cyrus-setup.sh scope auto-pop
agent-ci-feedback.yml           ← extended by helm-monitor.yml
                                + Tournament layer (Phase 4)
                                + Visualization pipeline (Phase 1)
                                + Memory-Bank OS (Phase 5)
```

Thread 2 **extends** thread 1 — it doesn't replace it. The existing plan-execute-review chain remains the single-issue path. Thread 2 adds the multi-issue orchestration layer on top.

---

### Linear Issue Landscape

#### The Dependency Chain

```
ENG-2128 (execution discipline) [In Progress — 5/6 sub-issues done]
  ├── ENG-2131 (branching/commits) [Done]
  ├── ENG-2132 (versioning) [Done]
  ├── ENG-2133 (environments) [Done]
  ├── ENG-2134 (agent discipline) [Done]
  ├── ENG-2135 (CI quality gates) [Done]
  └── ENG-2136+ (deferred)
       │
       ▼
ENG-2129 (alignment checker) [Triage] — passive monitoring of agent PRs
       │
       ▼
ENG-2130 (orchestrator agent) [Triage] — graduates checker to active orchestrator
```

The alignment checker (ENG-2129) → orchestrator agent (ENG-2130) progression is an **independent evolution path** from the macro strategy's helm. ENG-2130 envisions a persistent Linear-native orchestrator agent; the macro strategy's helm is a Claude Code skill. Both aim for similar outcomes through different mechanisms.

#### Related Issues

| Issue | Title | Status | Thread |
|-------|-------|--------|--------|
| ENG-2152 | Tech Lead Orchestrator | Done | Thread 1 |
| ENG-2183 | Nightly Scheduled Agents | Investigation | Thread 2 consumer |
| ENG-2225 | Harness Self-Improvement | Investigation | Thread 2 consumer |
| ENG-2226 | Z3 Plan Verification | Investigation | Cross-cutting |
| ENG-2227 | Commit Lineage Indexing | Investigation | Thread 2 enabler |

---

### Recent Git Activity (Orchestration-Related)

Key commits (newest first):
- `3476c26` — bulk commit of macro strategy artifacts (briefs, plans, research)
- `ea65e8e` — deterministic plan verification brief (ENG-2226)
- `e259758` — CLAUDE.md restructure for routing reliability (PR #39)
- `9d9b5f4` — ephemeral agent environment stack (PR #36)
- `77b4a87` — defense-in-depth scope enforcement (scope_guard.py)
- `8684a3d` — agent scope discipline to CLAUDE.md
- `c0ee3a8` — session-aware guards for CI feedback loop
- `bcea5ac` — resolve-linear-issue rewritten as Tech Lead Orchestrator (PR #7)
- `6d26bff` — agent execution discipline conventions (PR #8)

---

### Document Inventory

35 relevant documents found across `memory-bank/thoughts/`:
- **20 briefs** covering orchestration, scope enforcement, helm, swarm harness, judge agents, nightly agents, memory-bank OS, deterministic guardrails, etc.
- **10 plans** including macro strategy roadmap, helm orchestration (original + revised), resolve-linear-issue orchestrator, CI feedback loop, scope enforcement, ephemeral environments, swarm harness
- **10 research docs** including multi-agent architecture survey (14 systems), engineering execution discipline, deterministic layer claims, workflow frameworks analysis, backpressure tools
- **2 reference docs** on harness engineering and spec-driven development
- **1 handoff** documenting Cyrus scope creep on ENG-2171

## Architecture Documentation

### Current Architecture (Thread 1 — operational)

```
Human → CLAUDE.md routing → /resolve-linear-issue
                                   │
                         ┌─────────┤
                         │         │
                    /shape    /create_plan
                         │         │
                         └────┬────┘
                              │
                    adversarial-plan-reviewer (PASS/BLOCK)
                              │
                    plan-implementer (per phase)
                              │
                    /complete_plan + PR
```

### Planned Architecture (Thread 2 — macro strategy)

```
Human → /helm ENG-XXXX (decompose)
              │
       ┌──────┼───────┐
       │      │        │
    Sub-1   Sub-2    Sub-3    (file-scoped, non-overlapping)
       │      │        │
    Cyrus   Cyrus    Cyrus    (worktree-isolated)
       │      │        │
    scope.json per worktree (via cyrus-setup.sh + generate_scope.py)
       │      │        │
       PR     PR       PR
       │      │        │
    helm-monitor.yml (async progress → Linear comments)
       │      │        │
    /helm merge (serial, dependency-ordered)
```

## Historical Context (from memory-bank/thoughts/)

The evolution follows a clear arc:

1. **Feb 9** — Workflow frameworks analysis + Claude Code features overlap study → identified 10 improvements
2. **Feb 23-24** — CI failure orchestration brief + Linear-centric agent orchestration brief → early framing
3. **Feb 25** — Resolve-linear-issue orchestrator shaped + planned + implemented (PR #7) → Thread 1 established
4. **Feb 25-26** — Agent scope creep prevention, defense-in-depth scope enforcement, CI feedback loop → enforcement layer
5. **Feb 27** — Helm orchestration shaped + planned, visualization shaped + planned → Thread 2 components ready
6. **Feb 28** — Macro agent strategy brief + plan → Thread 2 unified into strategic roadmap
7. **Mar 1** — Multi-agent architecture research (14 systems) → revised helm plan with research-informed decisions
8. **Mar 2** — Revised helm plan (03-02), deterministic plan verification brief (ENG-2226), harness self-improvement brief (ENG-2225)

## Related Research
- `memory-bank/thoughts/shared/research/2026-03-01-multi-agent-architecture-research.md` — 14-system survey informing helm design
- `memory-bank/thoughts/shared/research/2026-02-24-engineering-execution-discipline-research.md` — Execution discipline foundations
- `memory-bank/thoughts/shared/research/2026-02-09-optimal-workflow-frameworks-analysis.md` — Workflow framework comparison

## Open Questions

1. **ENG-2129/2130 vs /helm** — The alignment checker → orchestrator agent progression (Linear issues) and the `/helm` skill (macro strategy) aim at overlapping outcomes. Which is the actual next step? Are they complementary or competing?
2. **Scope validation gap** — The scope enforcement chain (`cyrus-setup.sh` → `generate_scope.py` → `scope_guard.py`) exists but has never been validated end-to-end in a production Cyrus session. This is Phase 1 of the revised helm plan and is a hard gate.
3. **When does Phase 1 start?** — The macro strategy and both implementation plans (visualization + helm validation) are marked "ready to execute" but no phase has started. The brief says "Parked."
4. **Swarm harness vs helm** — The macro plan includes an "Agent Swarm Harness" (dev server for agent lifecycle) in Phase 3, but the revised helm plan operates entirely through Linear/Cyrus delegation. Is the swarm harness still relevant, or does the helm approach replace it?
5. **ENG-2128 completion** — The parent execution discipline issue is "In Progress" with 5/6 sub-issues done. ENG-2136 (prod environment) is deferred. Should ENG-2128 be closed?
