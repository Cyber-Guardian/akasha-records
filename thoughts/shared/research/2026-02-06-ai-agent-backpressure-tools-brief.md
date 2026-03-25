---
date: 2026-02-06T17:49:33Z
source_research: memory-bank/thoughts/shared/research/2026-02-06-ai-agent-backpressure-tools.md
last_generated: 2026-02-06T17:51:46.198491+00:00
---

# Research Brief: 2026-02-06-ai-agent-backpressure-tools

## TL;DR

We identified **four distinct axes** of backpressure for AI agents, not just two:

1. **Correctness** — Is the code doing what it should? (tests, mutation testing)
2. **Fitness** — Does the code maintain architectural integrity? (import rules, dependency boundaries, complexity)
3. **Safety** — Is the code free from security vulnerabilities and secrets? (SAST, secrets scanning, dependency auditing)
4. **Discipline** — Is the code staying within complexity/maintainability budgets? (cyclomatic complexity, diff-coverage, resource budgets)

The first two (correctness, fitness) are about *what the code does*. The latter two (safety, discipline) are about *what the code avoids*. All four can be automated and integrated into the existing Claude Code hook infrastructure at different stages.

---

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **import-linter + Polylith**: The multi-package setup (`root_packages`) may need PYTHONPATH tuning for Polylith. Needs hands-on verification.
2. **xenon thresholds**: What letter grades (A/B/C) are appropriate for this codebase? Need to baseline current complexity first.
3. **detect-secrets baseline**: Need to generate initial `.secrets.baseline` and tune for false positives.
4. **mutmut on this codebase**: How long does a full mutation run take for 329 tests across components/bases? Need to time a trial run.
5. **Hook ordering**: With xenon and detect-secrets added, the PostToolUse hook chain would be 4 tools. Need to verify cumulative latency stays under ~30s.
6. **Hypothesis integration**: Worth exploring for DynamoDB serializers and throttle service edge cases, but separate from backpressure tooling.

---

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-06-ai-agent-backpressure-tools.md`
