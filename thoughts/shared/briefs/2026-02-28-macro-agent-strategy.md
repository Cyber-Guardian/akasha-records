# Idea Brief: Macro Agent Strategy (Tournament Ideation + Execution + Visual Review)

**Date:** 2026-02-28
**Status:** Shaped → Parked

## Problem
The agent development strategy has four independent seeds (helm orchestration, swarm harness, judge/taste agents, complexity-as-moat) plus significant visualization work — all circling the same gap but not connected. There's no macro view that shows how these pieces compose into a unified system. Each seed gets shaped in isolation, but the real leverage comes from how they interact.

The full development loop today is: human thinks → agent executes → human reviews prose. Each leg has a bottleneck:
- **Ideation:** Single-shot — one agent, one approach, no systematic comparison
- **Execution:** Serial — manual orchestration breaks at 2-3 concurrent agents
- **Review:** Prose-heavy — human reads diffs/plans/docs, slow and error-prone

## Constraints
- One-person team leveraging AI — solutions must amplify, not add overhead
- Cyrus (Linear agent) is the primary execution engine — influence via CLAUDE.md, hooks, Linear API
- Claude Agent SDK available for custom agent sessions
- Existing guardrails (import-linter, ruff, ty, scope discipline) provide structural quality floor
- Token cost matters — can't tournament-rank everything, need selective triggering
- Visualization pipeline already shaped (Excalidraw, `/visualize` skill, C4 methodology)

## The Three Legs

### Leg 1: Tournament Ideation
Generate N approaches for a task, have judge agents evaluate and rank them, execute the winner.

**Pattern source:** Gemini Enterprise Idea Generation — multi-agent tournament generates ~100 ideas in ~40 minutes, ranks by quality via structured competition. Each idea gets overview, detailed description, review summary, and tournament performance report.

**Existing seeds:**
- Judge/Taste Agents (2026-02-28) — separate evaluator agents that build taste over time
- Deterministic Guardrail Layer (2026-02-25) — the structural/deterministic evaluation floor that judge agents build on top of

**Key design choice:** Tournament-on-Judgment-Calls — only trigger tournament for tasks above a complexity threshold or where multiple viable approaches exist. Simple/obvious tasks skip straight to execution. Evolution path: manual threshold → automated threshold → judges self-trigger.

### Leg 2: Agent Execution
Decompose work, delegate to parallel agents, manage lifecycle and merge ordering.

**Pattern source:** Entire.io/Checkpoints — capture agent reasoning alongside code (provenance). Universal semantic reasoning layer for agents.

**Existing seeds:**
- Agent Helm Orchestration (2026-02-27) — decompose → delegate → monitor → merge. Hybrid approach: Claude Code `/helm` skill for thinking + GH Action webhook monitor for async tracking.
- Agent Swarm Harness (2026-02-28) — own the execution environment for agent swarms. Dev server for lifecycle management. "Project agent" as resident context holder.
- Nightly Scheduled Agents (ENG-2183) — consumer of this leg, currently parked

**Key design choice:** Hybrid helm — Claude Code for decomposition + judgment, lightweight automation for monitoring + conflict detection.

### Leg 3: Visual Review
Humans evaluate agent output through visual devices (diagrams, architecture views, graph navigation) instead of prose.

**Pattern source:** Google Stitch — prompt/image to multiple UI design variants with code export. Visual-first artifacts that collapse the designer→developer handoff.

**Existing work (extensive):**
- Visualization in Shaping & Planning (2026-02-27) — brief + full implementation plan for forward-design diagrams. YAML graph spec → Python layout engine → Excalidraw + SVG.
- C4 Diagram Methodology (2026-02-12) — decided on C4 mental model with tiered tooling
- AI Visual Whiteboarding Integration (2026-02-16) — research on tldraw vs Excalidraw for AI integration
- `/visualize` skill design (2026-02-16) — handoff document for skill implementation
- Memory-Bank OS (2026-02-28) — Obsidian vault as the meta-level visual review surface for agent thinking

**Key design choice:** Excalidraw as primary editable format, SVG for agent iteration loops, Obsidian graph view for meta-navigation of agent thinking.

## How The Legs Compose

```
  Human sets goal
        │
        ▼
  ┌─────────────┐     Leg 2: Helm decomposes into tasks
  │    Helm     │────────────────────────────────┐
  └─────┬───────┘                                │
        │                                        │
        │ Complex task?                    Simple task?
        │ Yes                              Skip to worker
        ▼                                        │
  ┌─────────────┐     Leg 1: Tournament          │
  │  N agents   │     Generate N approaches      │
  │  (parallel) │                                 │
  └─────┬───────┘                                 │
        │                                         │
        ▼                                         │
  ┌─────────────┐     Leg 1: Judge agents         │
  │  Tournament │     Score + rank approaches     │
  │  evaluation │                                 │
  └─────┬───────┘                                 │
        │                                         │
        │ Winner                                  │
        ▼                                         ▼
  ┌─────────────┐     Leg 2: Swarm executes
  │   Worker    │     Agent writes code
  │   agent(s)  │
  └─────┬───────┘
        │
        ▼
  ┌─────────────┐     Leg 3: Visual review
  │  Visual     │     Diagrams, graph view,
  │  artifacts  │     architecture diffs
  └─────┬───────┘
        │
        ▼
  Human reviews visually → approve / reject / iterate
```

## The Moat Thesis
Complexity-as-Moat (2026-02-28) is the strategic rationale for this entire stack. The three legs compound:
- Tournament ideation → higher quality designs than single-shot competitors
- Parallel agent execution → higher velocity than serial competitors
- Visual review → faster human verification than prose-review competitors

A competitor would need all three legs to match output quality + velocity. Each leg alone is useful but not a moat. The composition is.

## Options Considered

### Build All Three Legs in Parallel
Shape and plan each leg independently, execute in parallel. Fastest time to full system.
- Gains: Maximum speed to complete stack
- Costs: Integration risk — legs designed independently may not compose cleanly. High cognitive load.
- Complexity: High

### Sequential: Visual Review → Execution → Ideation
Build on what's furthest along. Visual review has a plan. Execution has a shaped brief. Ideation is seeds.
- Gains: Each leg informs the next. Visual review improves ability to evaluate tournament results. Execution infra needed before tournament can run.
- Costs: Slowest time to full system. Tournament ideation (arguably highest leverage) comes last.
- Complexity: Medium

### Sequential: Execution → Visual Review → Ideation
Execution infrastructure (helm + swarm) is the prerequisite for everything. Can't run tournaments without parallel agents. Can't generate visual artifacts at scale without execution infra.
- Gains: Foundation-first. Each subsequent leg has infrastructure to build on.
- Costs: Visual review work (already shaped + planned) sits idle. Delay on the leg with most existing momentum.
- Complexity: Medium

### Interleaved: Execution + Visual Review together, then Ideation
Ship the visualization plan (already exists). Build helm orchestration in parallel. Once both work, add tournament layer on top.
- Gains: Leverages existing momentum on visualization. Helm brief is already shaped → planning. Tournament only meaningful once you can run parallel agents AND review results visually.
- Costs: Two active fronts, but both are well-defined.
- Complexity: Medium

## Chosen Approach
**Interleaved: Execution + Visual Review together, then Ideation** — the visualization plan is already written, execute it. Shape and plan helm orchestration in parallel. Tournament ideation is the capstone that depends on both. The judge/taste agent seed evolves as the tournament design crystallizes.

## Key Context Discovered During Shaping

### External signals
- **Entire.io** (Thomas Dohmke, ex-GitHub CEO, $60M seed) — building developer platform for agent-era. Checkpoints captures agent reasoning alongside code. Three pillars: semantic reasoning layer, git-compatible database, AI-native SDLC.
- **Google Stitch** (Google Labs) — prompt/image → multiple UI design variants → Figma/code. Gemini 2.5 Pro / Gemini 3 multimodal. Collapses designer→developer handoff.
- **Gemini Enterprise Idea Generation** — multi-agent tournament system. Generates ~100 ideas in ~40 minutes. Tournament-style competition ranks by user-defined quality criteria. Each idea gets overview + review + tournament performance report. Preview feature.

### Existing work inventory
- 4 seed briefs: helm orchestration, swarm harness, judge/taste agents, complexity-as-moat
- 1 shaped + planned: visualization in shaping/planning (7-phase implementation plan)
- 1 shaped: helm orchestration (hybrid approach chosen, ready for planning)
- 20+ visualization-related documents (research, plans, decisions, handoffs)
- Memory-Bank OS brief (2026-02-28) — Obsidian vault as infrastructure for this macro strategy

### Key architectural insight
The three legs share infrastructure:
- **Shared rendering layer**: Excalidraw pipeline serves tournament result visualization, execution progress dashboards, and human review surfaces
- **Shared memory layer**: Obsidian vault / MOC structure makes the macro plan navigable for both Claude and humans
- **Shared evaluation primitives**: Deterministic guardrails (import-linter, ruff, ty) are the floor; judge agents add taste on top

## Relationship to Other Work
- **Memory-Bank OS** (2026-02-28) — infrastructure for making this macro plan navigable
- **Agent Helm Orchestration** (2026-02-27) — Leg 2 component, shaped → ready for planning
- **Agent Swarm Harness** (2026-02-28) — Leg 2 component, seed
- **Judge/Taste Agents** (2026-02-28) — Leg 1 component, seed
- **Complexity-as-Moat** (2026-02-28) — strategic rationale for the whole stack
- **Visualization in Shaping & Planning** (2026-02-27) — Leg 3 component, shaped + planned
- **Deterministic Guardrail Layer** (2026-02-25) — shared evaluation floor across all legs
- **Nightly Scheduled Agents** (ENG-2183) — consumer of Leg 2, currently parked

## Next Step
- [Parked] — this brief is the macro reference document linking the strategic pieces. Individual legs proceed through their own shape → plan → implement lifecycle. The brief serves as the "MOC" connecting them until the Memory-Bank OS is in place.
