# Idea Brief: Concern-Scoped Review Agents

**Date:** 2026-03-04
**Status:** Shaped → Parked (brainstorm phase — not yet committed to an approach)

## Problem
The current adversarial-code-reviewer is a single generalist agent checking 7 concern categories in one pass. Empirically, prompt overloading degrades accuracy (45-point swings from scope variation), and specialist reviewers consistently outperform generalists (9.61% vs 5.30% I-Score on security; 3x bug detection in multi-agent setups).

## Constraints
- Must compose with existing helm merge pipeline (review.py / merge.py)
- Agent format is established (.claude/agents/ YAML frontmatter + markdown prompt)
- Token overhead must stay reasonable (independent parallel dispatch adds ~58%)
- Must work both in PR review (helm) and on-demand (slash commands)
- Each agent produces its own verdict (user preference)

## Options Considered

### Parallel Specialists + Judge (full Qodo pattern)
3 specialist agents (security, architecture, performance) + refactored generalist + judge agent. Helm dispatches all via ThreadPoolExecutor, judge aggregates before merge decision.
- Gains: Maximum detection quality (28-62% improvement); industry-validated; dedup built in
- Costs: 5 agents to maintain; judge adds latency; ~58% token overhead
- Complexity: High

### Specialist Agents Only (no judge)
Same 3 specialists + generalist, each produces independent verdict. Deterministic veto-based aggregation in review.py (any BLOCK → halt).
- Gains: Simpler; lower latency; no judge prompt to engineer
- Costs: No dedup; overlapping findings surface raw; contradictory advice possible
- Complexity: Medium

### Incremental Specialist Extraction (one at a time)
Start with security agent only (strongest empirical case). Ship alongside existing generalist. Add architecture and performance later. Judge when N>2.
- Gains: Lowest risk; validates pattern before full investment; immediate security depth
- Costs: Slower to full coverage; generalist still overloaded initially
- Complexity: Low per step

## Chosen Approach
**Not yet decided** — parked in brainstorm phase. Recommendation leans toward Incremental Specialist Extraction (Option 3) based on research, but no commitment made.

## Key Context Discovered During Shaping
- Industry has converged on specialist + judge pattern (Qodo 2.0, diffray, CodeRabbit)
- 3 concerns warrant specialist agents: security, architecture, performance
- Correctness stays generalist; observability folds in; style belongs to linters
- Sweet spot: 3-5 agents total (coordination overhead power-law beyond 4)
- Judge agent is mandatory when N>2 specialists to prevent alert fatigue (21% nitpicking without)
- RAG over CWE/CVE yields 16-24% accuracy gain for security agents
- Helm review.py can parallelize via ThreadPoolExecutor (currently serial subprocess.run)
- Deep research doc: `memory-bank/thoughts/shared/research/2026-03-04-deep-dedicated-concern-scoped-agents.md`

## Next Step
- [Parked] → Linear backlog issue with brief as context. Resume when ready to commit to an approach.
