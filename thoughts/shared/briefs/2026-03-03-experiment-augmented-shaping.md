# Idea Brief: Experiment-Augmented Shaping

**Date:** 2026-03-03
**Status:** Shaped → Planning

## Problem
During shaping sessions, you regularly hit moments where the right answer is "I don't know — we'd need to try it." Today, that kills momentum: you either assume and move on (risky) or park and come back later (slow). There's no way to say "go test this real quick" and continue the conversation with real data.

## Constraints
- Experiments must run in full isolation — worktrees for code, ephemeral infra for integration tests. Cannot pollute the real repo or environments.
- Needs to work within the shape conversation flow — experiments are inline side quests, not separate workflows.
- Helm is the execution/monitoring layer — R&D experiments should use Helm for orchestration where appropriate, not reinvent it.
- Some experiments are fast (perf benchmarks, code comparisons) and some are slow (integration feasibility, infra-dependent). The design must handle both gracefully.
- Agent needs deep repo context to design meaningful experiments (compare against current approach, not abstract).

## Options Considered

### Experiment-Augmented Shaping
Extend `/shape` with an `/experiment` escape hatch. Mid-conversation, define an experiment → agent runs it in a worktree/ephemeral env → findings doc lands → shaping resumes with data. Single conversation, experiments are inline side quests.
- Gains: Lowest friction, findings naturally in context, matches real workflow
- Costs: Long-running experiments could block conversation, needs async execution
- Complexity: Medium

### R&D Campaign Runner
Standalone `/rnd` skill for multi-experiment campaigns. Define a hypothesis with several angles, run a battery of experiments in parallel, produce a comprehensive findings report. Shape/plan sessions reference the report.
- Gains: Supports larger-scale R&D, parallelism, doesn't block conversation
- Costs: Heavier orchestration, leaves shaping conversation, may be overbuilt for v1
- Complexity: High
- **Backlogged** — Linear issue to be created

### Hypothesis Queue with Continuous Learning
Persistent hypothesis backlog. Drop hypotheses during shape/plan/build, R&D agents pick them up asynchronously (nightly or on-demand), findings accumulate in a knowledge base. Shape sessions query knowledge base for relevant prior experiments.
- Gains: Non-blocking, builds institutional knowledge, integrates with nightly agents (ENG-2183), prevents re-testing
- Costs: Decoupled from conversation, needs knowledge retrieval, more complex lifecycle
- Complexity: High
- **Backlogged** — Linear issue to be created

## Chosen Approach
**Experiment-Augmented Shaping** — it matches the actual workflow: you're in a shaping conversation, you hit an empirical question, you need data now. The other approaches are valuable evolution but not the starting point. Start inline, let the pattern mature, then grow into campaigns and persistent knowledge.

## Key Context Discovered During Shaping
- Helm V1 execution layer is fully implemented — experiments can delegate execution to Helm for orchestration and monitoring
- Nightly scheduled agents (ENG-2183) are parked — the Hypothesis Queue approach would layer on top of this when ready
- Worktree support exists in Claude Code (`EnterWorktree`) — provides code-level isolation out of the box
- Ephemeral infra is the real infrastructure investment — needed regardless of which approach, and the existing Terragrunt dev/testing/staging pattern provides a template
- The experiment lifecycle (design → execute → collect → interpret) maps naturally to shape's iterative loop — experiments are just a new tool available within that loop

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-03-experiment-augmented-shaping.md`
