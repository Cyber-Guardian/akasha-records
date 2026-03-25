# Idea Brief: Helm — Swarm Orchestration Harness (Option C — Own the Macro)

**Date:** 2026-03-02
**Status:** Shaped → Planning

## Problem
The current helm design is a Claude Code markdown skill — decomposition, delegation, monitoring, and merge logic is prose instructions interpreted non-deterministically. For the macro layer coordinating swarms of concurrent agents and $100+/hr of compute, the orchestration logic must be deterministic Python code you can test, debug, and version. Additionally, the design phase (tournament ideation, judging with reasoning models + symbolic systems) is a local process that needs to orchestrate multiple agents — which requires SDK-level control, not markdown skills.

**Important framing:** This is a swarm orchestration harness, not a Claude Code replacement. Claude Code / Cyrus is the agent runtime — it's excellent at micro-level execution (worktrees, hooks, plan following). The helm doesn't replace that. It sits above the agent layer, coordinating *swarms* of agents working in parallel. Think air traffic control: it doesn't fly the planes, it coordinates which planes go where, tracks their progress, detects conflicts, and sequences landings.

## Constraints
- Cyrus is the proven micro-execution runtime — worktrees, hooks, CLAUDE.md, plan-implementer — 10+ issues shipped
- Claude Agent SDK spawns CLI subprocess — needs binary installed wherever helm runs
- One-level subagent nesting is a hard limit (SDK and Cyrus both)
- Linear remains the control plane for execution (issue state, delegation, session lifecycle)
- API key billing only for SDK (no Max plan)
- Existing scope chain (`cyrus-setup.sh` → `generate_scope.py` → `scope_guard.py`) is built but unvalidated end-to-end
- `generate_scope.py` already has plan parsing logic the helm can import
- Tournament ideation is a research bet — execution layer must work independently
- One-person team — the system must amplify, not add overhead

## Options Considered

### Thin Harness + SDK Design Engine
Python harness with subcommands (decompose, design, status, merge). SDK for tournament design mode, Linear GraphQL for execution mode. Tournament opt-in.
- Gains: Clean separation, testable, debuggable. Tournament additive.
- Costs: Two integration surfaces. Tournament still fuzzy.
- Complexity: Medium

### Monolithic Orchestrator Service
Long-running daemon watching Linear for helm-tagged issues, running tournaments, monitoring Cyrus.
- Gains: Fully autonomous, real-time event reaction.
- Costs: Infrastructure overkill for 3-5 agents. Daemon debugging harder than CLI. Building a service to manage a service.
- Complexity: High

### Layered Harness: Orchestrate Now, Tournament Later
Same as Thin Harness but explicitly staged. V1 = execution layer (decompose + delegate + monitor + merge). V2 = design mode with SDK tournament. Architecture accommodates tournament from day 1 (subcommand exists, data model ready).
- Gains: Ships value immediately. Validates swarm orchestration before investing in design. Escape hatch built in.
- Costs: Slight over-architecture in V1. Risk of V2 never shipping.
- Complexity: Medium (V1), Medium-High (V2)

## Chosen Approach
**Layered Swarm Harness** — `tools/helm/` Python harness with CLI interface. V1 is the execution layer: decompose plans into file-scoped sub-issues, delegate to Cyrus agents via Linear, monitor swarm progress, orchestrate serial merges. V2 adds design mode: SDK-powered tournament ideation with reasoning models + symbolic judges. The `/helm` Claude Code skill wraps the harness — usable by humans and agents alike. Plan files are the contract between design and execution.

The macro split: **the harness owns the macro** (deterministic swarm orchestration in Python), **agents own the micro** (Cyrus/Claude Code handles agent runtime, worktrees, hooks, plan-implementer). The harness never reaches into agent-level concerns — it coordinates, it doesn't execute.

## Key Context Discovered During Shaping
- Multi-agent architecture research (2026-03-01) surveyed 14 systems: decomposition quality is #1 bottleneck (Gas Town), agents must NOT coordinate (CooperBench), serial merge queue is most reliable
- Revised helm plan (2026-03-02) defines the 6-phase execution architecture — validated design, runtime changes to Python CLI
- Claude Agent SDK offers `query()`, `ClaudeSDKClient`, programmatic subagents, session resume/fork, `max_budget_usd`, custom MCP tools — full control surface
- Cyrus uses `settingSources: ["project"]` to load CLAUDE.md + agents + hooks — our existing infrastructure works inside Cyrus sessions
- `generate_scope.py` has `parse_plan()` and can be extended with `parse_plan_by_phase()` — helm reuses this directly
- ENG-2129/2130 (alignment checker → orchestrator agent) is a parallel evolution path on Linear — the helm CLI may subsume or complement it

## Relationship to Other Work
- **Macro Agent Strategy** (2026-02-28) — the harness is the implementation vehicle for the strategy. V1 = Leg 2 (execution). V2 = Leg 1 (ideation). Leg 3 (visual review) remains independent.
- **Revised Helm Plan** (2026-03-02) — the execution architecture carries over; runtime changes from markdown skill to Python harness
- **Z3 Plan Verification** (ENG-2226) — V2 judge infrastructure. Z3 becomes one symbolic judge alongside LLM judges in tournament evaluation.
- **Nightly Scheduled Agents** (ENG-2183) — consumer of the harness. Once helm works, nightly agents become a cron that creates Linear issues and invokes `helm decompose`.
- **Harness Self-Improvement** (ENG-2225) — consumer of nightly infrastructure

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-02-helm-orchestrator-cli.md`
- Linear project: "Helm Orchestrator" with V1 and V2 milestone breakdown
