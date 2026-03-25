# Idea Brief: Grounded Task Estimation for Claude Code

**Date:** 2026-03-09
**Status:** Shaped → Planning

## Problem
Claude Code systematically overestimates task durations because its mental model is calibrated to human-solo development timelines from training data, not AI-agentic execution speeds. This bias leaks into plans, briefs, Linear issue scoping, and verbal estimates — making work seem bigger than it is and dampening ambition. The bias is well-documented in LLM research (anchoring effects in effort estimation) and confirmed by direct user experience.

## Constraints
- Must work across all estimation surfaces: session estimates, plans, briefs, Linear issues
- Must account for the agentic-vs-assistive distinction — this codebase is optimized for agentic execution (Polylith, hooks, skills, memory-bank, architectural contracts)
- No published prior art for "agentic task estimation calibration" — this is novel
- Should not require manual time tracking — the system should be low-friction
- Must coexist with Claude's default "avoid giving time estimates" instruction — reframe rather than contradict

## Options Considered

### Calibration Rule (lightweight)
A `.claude/rules/` file instructing Claude to adjust estimates downward for AI-executed work with heuristic multipliers (divide intuitive estimate by 4-8x for agentic, 2-3x for AI-assisted).
- Gains: Zero infrastructure, instant, easy to tune
- Costs: Static multiplier — no learning, no evidence, crude
- Complexity: Low

### Scope-Based Sizing (structural)
Replace time estimates with concrete scope metrics (files touched, functions changed, tests needed, external dependencies). Map to calibrated t-shirt sizes with empirically observed durations from git history.
- Gains: Grounded in observable facts, decomposition forces honesty
- Costs: Needs initial calibration pass over git history, more upfront work
- Complexity: Medium

### Feedback Loop Skill (adaptive)
An `/estimate` skill that produces scope-based estimates, logs them, records actual duration after completion, and builds a calibration table in memory-bank.
- Gains: Self-correcting over time, gets better with use
- Costs: Most complex, needs session timing infrastructure, takes N tasks to calibrate
- Complexity: High

### Hybrid: Rule Now + Scope Conventions + Feedback Later
Ship calibration rule immediately. Bake scope decomposition into plans/briefs as convention. Collect estimation-vs-actual data to build toward adaptive calibration.
- Gains: Immediate improvement + growth path
- Costs: Incremental complexity over time
- Complexity: Low → Medium (phased)

## Chosen Approach
**Hybrid** — The calibration rule gives an instant fix for the systematic bias. Scope-based sizing baked into plans/briefs forces decomposition (where real accuracy comes from). Feedback collection is the long game but needs data first — start collecting now.

## Key Context Discovered During Shaping
- LLMs exhibit anchoring bias in effort estimation, anchoring to training-data norms (ACM study on LLM effort estimation)
- METR RCT found assistive AI made experienced devs 19% slower, but agentic workflows show 2.5-5x speedup — the distinction matters
- This repo's infrastructure (Polylith boundaries, hooks, skills, memory-bank) puts it squarely in the "agentic" camp where speedups are real
- Claude's default system prompt says "avoid giving time estimates" — the bias operates even without explicit numbers, through complexity labels and phase counts
- No published prior art for agentic task estimation calibration — this would be pioneering

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-09-grounded-task-estimation.md`
