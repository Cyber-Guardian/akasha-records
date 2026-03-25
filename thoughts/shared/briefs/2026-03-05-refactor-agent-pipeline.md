---
type: event
created: 2026-03-05
status: superseded
---
# Idea Brief: Multi-Agent Write → Test → Refactor Pipeline

**Date:** 2026-03-05
**Status:** Superseded by [[2026-03-17-refactor-pass-pipeline|Separate Refactor Pass Pipeline]]

## Problem
Claude Code writes functional Python that passes all guardrails but stops there. Senior engineers iterate — write it, test it, then refactor for elegance with a safety net. Claude lacks this refactor pass. AI refactoring without guardrails fails 63% of the time (CodeScene research), so the design must be test-gated and surgical.

## Constraints
- Refactor agent MUST have tests covering target behavior before touching anything
- Refactor agent should know plan intent (not just code-focused)
- Must propose numbered surgical changes, not open-ended rewrites
- Must be able to conclude "no refactoring needed"
- Radon already installed — can detect candidates symbolically (CC >= 11, MI < 10)
- CodeScene found AI breaks code in 63% of refactoring attempts without behavioral equivalence checks

## Options Considered

### Baked-in Polish (refactor phase in implement_plan)
Add a refactor phase to every plan implementation. After plan-implementer finishes and tests pass, a refactor-agent subagent runs against changed files with plan intent + radon output, proposes numbered changes, gets approval, applies, re-runs tests.
- Gains: Every implementation gets polish automatically. Consistent quality.
- Costs: Adds time/tokens to every phase. May feel forced on quick changes.
- Complexity: Medium

### Standalone /polish Skill
A new /polish skill invoked on demand. Runs radon to find hot spots, dispatches refactor agent, proposes changes, re-runs tests.
- Gains: Flexible, no overhead on simple tasks. Can polish old code too.
- Costs: Relies on user remembering to invoke. Won't become habit.
- Complexity: Low

### Signal-Driven Polish (radon hook)
PostToolUse hook runs radon on every written .py file. Warnings (not blocks) when metrics exceed thresholds. Main context handles refactoring with heuristics from a rule file.
- Gains: Zero overhead for clean code. Only triggers when metrics say something's off.
- Costs: Doesn't catch "correct but inelegant" code with fine metrics.
- Complexity: Low

### Hybrid: Signal Gate + Opt-in Agent
Combine radon hook (early warning) with refactor agent (as implement_plan phase when flagged or user opts in).
- Gains: Automatic when warranted, opt-in otherwise. Intent-aware. Safe.
- Costs: Most infrastructure to build.
- Complexity: High

## Chosen Approach
**Baked-in Polish** — add refactor phase to implement_plan, layer radon hook on top later. Gives core value (every implementation gets a polish pass) with medium complexity.

## Key Context Discovered During Shaping
- CodeScene "Refactoring vs Refuctoring" paper: AI breaks code 63% of the time when refactoring without behavioral equivalence validation
- Beck's four rules as evaluation order: passes tests, reveals intention, no duplication, fewest elements
- Sandi Metz limits: 100 lines/class, 5 lines/method, 4 params/method
- Rule of Three: tolerate duplication twice, extract on third
- Python-specific heuristics: single-method class -> function, repeated constructor args -> class, nested loops -> comprehension, data-only class -> dataclass
- Copilot code smell research: Multiply-Nested Container (40%), Long Parameter List (22%), Long Method (14%) are dominant AI-generated smells
- Radon provides CC grading (A-F, C+ = refactor candidate) and MI grading (A/B/C, C = refactor now)
- The refactor agent instruction set should encode Beck's rules, Metz limits, Python heuristics, and CodeScene's "propose first, apply surgically" discipline
- Deep research on Python code quality also done: `memory-bank/thoughts/shared/research/2026-03-04-deep-best-possible-python-code.md`

## Next Step
- [Parked] → Linear backlog issue (pending)
