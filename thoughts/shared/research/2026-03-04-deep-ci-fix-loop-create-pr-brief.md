---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-ci-fix-loop-create-pr.md
last_generated: 2026-03-04T20:14:35.539144+00:00
---

# Research Brief: 2026-03-04-deep-ci-fix-loop-create-pr

## TL;DR

The pure-skill approach is **feasible but must abandon `run_in_background`** as its core CI-watching primitive due to known bugs (notification regression #21048, infinite token drain #11716). The validated alternative is a **foreground polling loop** using `gh run list --commit <SHA> --json status,conclusion` with an extended Bash timeout — this costs zero tokens while waiting and avoids the 13K-line context overflow from `gh run watch`. The skill design should follow the existing helm skill pattern (multi-step conditional loop, 50-65 lines) with a hard 3-round fix cap matching both the cloud-side `agent-ci-feedback.yml` and industry consensus. Key risk: autonomous fix loops produce 75% more logic errors than human fixes — human checkpoint after each fix round is strongly recommended over fully autonomous operation.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **gh run view --log-failed reliability**: Should the skill have a fallback for when --log-failed returns empty? (e.g., `gh api` to fetch logs directly, or `gh run view --log` + grep)
2. **Human checkpoint granularity**: Should the skill pause after EVERY fix round for human review, or only after the final round? The brief's "chosen approach" implies fully autonomous, but research suggests per-round checkpoints are safer.
3. **Semantic PR failures**: The cloud system treats semantic PR failures differently (title-only fixes). Should the local skill handle this case explicitly, or just treat all failures uniformly?
4. **Polling interval tuning**: 30-second polling is a starting point. CI runs in this repo typically take 2-5 minutes — what's the optimal interval to balance responsiveness vs. API rate limits?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-ci-fix-loop-create-pr.md`
