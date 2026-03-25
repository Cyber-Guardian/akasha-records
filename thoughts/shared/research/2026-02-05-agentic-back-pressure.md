---
title: Back pressure for agentic engineering (codebase feedback)
date: 2026-02-05
status: draft
---

# Back pressure for agentic engineering

Goal: define automated validation and feedback signals that regulate agent throughput
so the codebase stays within desired quality, architecture, and operational bounds.

## Working definition
Back pressure for agentic engineering is a deliberate set of automated feedback
loops (gates, budgets, and signals) that slow or stop change throughput when
quality, architectural integrity, or operational risk degrades. It is the
software-delivery analogue of system backpressure: keep work flowing, but never
faster than the system can safely absorb.

## Signal taxonomy (from fastest to most expensive)

1. Static, fast, deterministic
   - Format, lint, type check (e.g., Ruff + ty)
   - Import/layer boundaries, forbidden deps, cycles
   - Security hygiene: secrets scan, vulnerable deps, license policy
   - Config correctness: schema validation, IaC lint, runtime config checks

2. Unit and focused integration tests
   - Test selection based on changed files or dependency graph
   - Coverage thresholds (use cautiously; pair with mutation testing)

3. Architecture fitness functions (automated checks for desired traits)
   - Dependency rules (allowed layers, bounded contexts)
   - Service/API contract checks (schema, compatibility)
   - Data invariants (migration safety, key cardinality bounds)

4. Performance and reliability regression
   - Micro/benchmarks with regression baselines
   - Load or soak tests for critical paths
   - Resource budgets (CPU, memory, cold start, package size)

5. Runtime and progressive delivery signals
   - Canary analysis: error rate, latency, saturation metrics
   - SLO budget consumption or anomaly detection

6. Test quality signals
   - Mutation testing to confirm tests fail on meaningful behavior changes

## Fitness functions (why they matter for back pressure)
Architectural fitness functions are objective, automated checks that verify
architectural characteristics you want to preserve. They enable continuous
feedback so architecture does not silently degrade during incremental change.
This makes them a natural "back pressure" input for agentic systems.

## Back pressure ladder (example policy)
Use a graded ladder of checks. Failures increase friction or halt.

Stage 0 (local, < 30s):
  - Format + lint + type check
  - Fast unit tests or targeted test selection

Stage 1 (PR CI, minutes):
  - Full unit suite
  - Key integration tests
  - Fitness functions (dependency rules, contract checks)

Stage 2 (post-merge or nightly):
  - Performance baselines
  - Broader integration / e2e
  - Mutation testing (scheduled)

Stage 3 (deploy):
  - Canary analysis / progressive delivery gates
  - Rollback on SLO regression

## Back pressure design rules for agentic systems
1. Prefer high-signal, low-noise checks early. False positives erode trust.
2. Separate gating vs advisory signals. Only gate on stable checks.
3. Budget-aware: test selection and staged depth based on change size and risk.
4. Explicit failure handling: on fail, agent must stop, summarize, and propose
   remediation steps before continuing.
5. Keep signals explainable: show the failing check, scope, and next action.

## Sources
- Architectural fitness functions: https://www.continuous-architecture.org/practices/fitness-functions/
- Evolutionary architecture context: https://evolutionaryarchitecture.com/precis.html
- AWS: fitness functions in cloud architecture: https://aws.amazon.com/blogs/architecture/using-cloud-fitness-functions-to-drive-evolutionary-architecture/
- Feedback loops in delivery: https://www.datadoghq.com/blog/feedback-loops-progressive-delivery/
- CI signal improvements: https://docs.ci.openshift.org/docs/release-oversight/improving-ci-signal/
- Mutation testing: https://cosmic-ray.readthedocs.io/
- Mutation testing (mutmut): https://mutmut.readthedocs.io/en/latest/
- Ruff: https://docs.astral.sh/ruff/
- ty type checker: https://docs.astral.sh/ty/type-checking/
