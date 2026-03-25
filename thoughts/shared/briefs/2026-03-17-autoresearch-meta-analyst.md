---
type: event
created: 2026-03-17
status: active
related:
  - thoughts/shared/briefs/2026-03-16-autoresearch-research-quality.md
  - thoughts/shared/plans/2026-03-16-autoresearch-research-quality.md
---
# Idea Brief: Autoresearch Meta-Analyst — Self-Improving R&D Lab

**Date:** 2026-03-17
**Status:** Shaped -> Planning

## Problem
The autoresearch harness generates rich per-experiment data (30+ fields: costs, durations, scores, failure categories, personalities, novelty scores, dimension coordinates, researcher turns, progress notes) but nobody systematically mines it for patterns about the lab's own behavior. The harness evolves only when a human notices an issue and writes a plan. The data to identify improvements already exists — there's no machinery to analyze it and reason about what it means for the harness's code, architecture, and prompts.

## Constraints
- Open loop only — meta-analyst proposes, human reviews. No autonomous code mutation. "Slop compiles" risk is real.
- Must work against any campaign's data (not coupled to a specific workspace)
- Recommendations must be grounded in data (cite experiment IDs, distributions, correlations)
- Output should be actionable — briefs or proposals that feed into `/create_plan` or direct implementation
- The meta-analyst targets the harness code itself (`bases/filescience/tool_autoresearch/`), not just campaign config knobs
- Should compute structured analytics as a side effect (useful for dashboard, future automation)

## Options Considered

### Periodic Retrospective Agent (CLI command)
A CLI command (`autoresearch meta`) that spawns a Claude agent to analyze experiments + harness source. Produces markdown reports.
- Gains: Simple. Runs against any campaign retroactively.
- Costs: No real-time observation. User must remember to run it.
- Complexity: Low

### Live Sidecar Daemon
Second daemon alongside coordinator. Watches experiments.jsonl via byte-offset polling. Runs LLM analysis every N experiments. Accumulates a meta-journal.
- Gains: Real-time. Sees temporal patterns. Alerts mid-campaign.
- Costs: Complex lifecycle. Own budget tracking. Adds LLM cost per N experiments.
- Complexity: Medium

### Claude Code Skill (slash command)
A `/lab-retrospective` skill invoked on-demand in Claude Code. Full codebase access — reads experiment data AND harness source. Computes analytics, correlates data patterns with code, produces improvement briefs.
- Gains: Zero infrastructure. Full codebase reasoning. Fits existing workflow (`/whatsnext` pattern). Can do web research for technique improvements. Human is in the loop by design.
- Costs: Not automated — requires human to invoke. No persistent meta-journal across invocations.
- Complexity: Low-Medium

### Event Bus + Pluggable Analyzers
Coordinator emits structured events. Meta-analyst is one consumer among many.
- Gains: Clean extensibility.
- Costs: Over-engineered. No second consumer exists.
- Complexity: High

## Chosen Approach
**Claude Code Skill** — on-demand `/lab-retrospective` command that mines experiment data against harness source and proposes code/architecture improvements. Paired with a Python analytics module that computes derived metrics (reusable by dashboard and future automation).

Reasons:
- Fits the "in-loop codev" pattern — highest-leverage decisions (what to change about the lab) stay human-in-the-loop
- Full codebase access means it can correlate "personality X has 5% keep rate" with the actual prompt text and propose specific edits
- Analytics module is a durable artifact — useful beyond just the skill (dashboard display, future auto-tuning foundation)
- Can evolve: start as a skill, promote analytics to dashboard, promote safe recommendations to auto-tune later

## Key Context Discovered During Shaping
- ExperimentRecord has 30+ fields per experiment — enough data for meaningful meta-analysis without adding new instrumentation
- The lab leader already gets stale detection nudges (`consecutive_discards >= threshold/2`) but it's hardcoded heuristic, not data-driven — meta-analyst could ground these
- Knowledge base (`knowledge.json`) accumulates domain findings but not meta-findings about the lab's behavior — separate concern
- The watcher polling pattern in `watcher.py` already demonstrates incremental JSONL reading — analytics module can reuse this
- Adjacent work: the research quality plan ([[2026-03-16-autoresearch-research-quality|Research Quality Plan]]) adds sweep + knowledge layer — the meta-analyst sits above that, evaluating whether those improvements actually helped

## Concrete examples of what the meta-analyst would propose
- "Experiments with `failure_category=import_error` always come from the radical personality trying to import unavailable libraries -> add an allowed-imports list to the researcher prompt" (prompt change)
- "The evaluator metadata has `memory_peak_mb` but the lab leader never sees it -> pipe it into the leader prompt" (data flow change)
- "70% of researcher turns are spent reading champion code that's already in the prompt -> reduce allowed tools or restructure the prompt" (architecture insight)
- "The adaptive backoff formula doesn't account for cost — it sleeps the same after a $0.02 crash as a $0.50 discard" (coordinator logic change)
- "Personality X has 40% keep rate at $0.12/experiment vs Y at 5% keep rate at $0.35/experiment -> adjust personality weighting or drop Y" (strategy recommendation)

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-autoresearch-meta-analyst.md`
