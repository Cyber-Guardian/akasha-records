---
date: 2026-03-04T14:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-dedicated-concern-scoped-agents.md
last_generated: 2026-03-04T20:36:02.616632+00:00
---

# Research Brief: 2026-03-04-deep-dedicated-concern-scoped-agents

## TL;DR

The industry has converged on **specialist decomposition + judge/aggregator** as the dominant pattern for AI code review (Qodo 2.0, diffray, CodeRabbit). Empirical evidence strongly favors specialists over generalists: security-focused reviewers score 9.61% vs 5.30% I-Score; multi-agent architectures report 3x bug detection and 87% false-positive reduction. Three concerns warrant dedicated agents — **security** (needs CWE/OWASP knowledge + SAST integration), **architecture** (needs cross-file dependency context), and **performance** (needs algorithmic complexity focus) — while correctness stays generalist, observability folds into generalist prompting, and style belongs to deterministic linters. The optimal agent count is **3-5 specialists** before coordination overhead dominates (power-law growth beyond 4 agents). A judge/aggregator agent is essential to prevent the primary anti-patterns: alert fatigue (21% nitpicking observed), context fragmentation, and contradictory advice.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How should specialist agents share cross-cutting context (PR intent, plan criteria) without redundant prompt stuffing? Shared context header vs. each agent reads the plan independently?
- Should the judge agent be a separate .md agent definition, or a Python function in review.py that does deterministic aggregation?
- What's the right model tier for specialists? Security may warrant opus; architecture may be fine on sonnet.
- How to handle the case where a specialist agent's findings change the relevance of another specialist's pass (e.g., security finding reveals an architecture issue)?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-dedicated-concern-scoped-agents.md`
