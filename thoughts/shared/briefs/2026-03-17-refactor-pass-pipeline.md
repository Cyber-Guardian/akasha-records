---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Separate Refactor Pass Pipeline

**Date:** 2026-03-17
**Status:** Shaped → Planning
**Supersedes:** [[2026-03-05-refactor-agent-pipeline|Multi-Agent Write → Test → Refactor Pipeline]] (was parked)

## Problem
Claude Code writes functional Python that passes all guardrails but stops there. Senior engineers iterate — write it, test it, then refactor with the safety net of passing tests. The March 5 brief identified this gap and chose "Baked-in Polish" (a refactor phase inside implement_plan). This brief refines that: refactoring must be a **completely separate flow** from implementation, with its own agent that plans its own work under stricter backpressure.

## Constraints
- Tests must exist and pass before refactoring starts (non-negotiable gate)
- Refactor agent must not change behavior — only structure
- Refactor agent creates its own mini-plan (what to change, why, in what order) before touching code
- The two flows must be independently invocable — refactor old code, not just fresh implementations
- CodeScene's 63% failure rate on ungated AI refactoring applies — the refactor plan + test gate is the mitigation
- Stricter backpressure than implementation: anti-pattern checking, complexity analysis, Beck's four rules as evaluation criteria
- Worktree quality parity ([[2026-03-17-worktree-agent-quality-parity|brief]]) should land first or concurrently — ensures refactor agent gets real-time lint/type feedback

## Options Considered

### Separate Refactor Skill (/refactor)
Standalone skill, always opt-in. User invokes after implementation or on any existing code. Reads changed files + tests → runs analysis → generates refactor plan → user approves → executes → re-runs tests.
- Gains: Flexible, works on any code, no overhead on simple tasks
- Costs: Relies on user remembering to invoke
- Complexity: Medium

### Implement-then-Refactor Pipeline (auto with escape hatch)
Modify plan execution: after all phases complete and tests pass, auto-dispatch refactor agent. Plans can mark refactor: false to skip.
- Gains: Every implementation gets polish by default
- Costs: Adds time/tokens to every execution, must opt out not in
- Complexity: Medium-High

### Both: Skill + Auto-trigger
Build /refactor as standalone skill. Wire into implement_plan as post-phase auto-trigger. Plans can opt out with refactor: skip. User can also invoke independently.
- Gains: Best of both — automatic for plans, available for ad-hoc use. One agent serves both triggers.
- Costs: Slightly more wiring
- Complexity: Medium

## Chosen Approach
**Both: Skill + Auto-trigger** — Build the refactor agent as a standalone /refactor skill first (core artifact, independently testable). Then wire into implement_plan as a post-phase hook. Auto vs opt-in becomes configuration, not architecture.

## Key Context Discovered During Shaping
- Prior brief exists: [[2026-03-05-refactor-agent-pipeline|March 5 brief]] chose "Baked-in Polish" but was parked. This brief evolves the idea with stronger separation.
- CodeScene research: AI breaks code 63% of the time refactoring without behavioral equivalence checks
- Beck's four rules as evaluation order: passes tests, reveals intention, no duplication, fewest elements
- Radon already installed for CC/MI analysis
- Deep research on Python code quality: [[2026-03-04-deep-best-possible-python-code|Python Code Quality Research]]
- Copilot smell research: Multiply-Nested Container (40%), Long Parameter List (22%), Long Method (14%) are dominant AI-generated smells
- adversarial-code-reviewer agent exists as a pattern for post-implementation analysis agents
- Worktree quality parity work (today) is fixing backpressure in worktree agents — prerequisite for stricter refactor-agent backpressure

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-refactor-pass-pipeline.md`
