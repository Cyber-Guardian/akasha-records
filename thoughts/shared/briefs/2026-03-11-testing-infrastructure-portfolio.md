# Idea Brief: Testing Infrastructure Portfolio — Holistic Verification System

**Date:** 2026-03-11
**Status:** Shaped → Planning

## Problem
FileScience is scaling from 3 Lambda services to a distributed system — 5+ container services, dozens of Lambdas, DynamoDB tables, Dashboard + API. Agents will build most of it. Agents can only work autonomously when their output is machine-verifiable. Every gap in verification is a gap where a human must review. The testing strategy IS the agent autonomy strategy — the testing harness is integral to agentic engineering leverage.

## Constraints
- Team: Jordan on V2 infra, Niklas + Harsh on v1 legacy
- Polylith monorepo — all services share a codebase (Pact-style CDC is overkill for internal services; Protocol-typed contracts suffice)
- Dashboard + API will be the first real HTTP boundary (different stack — contract testing matters there)
- Ruff `select = ["ALL"]` already runs flake8-bandit S-prefix rules on every edit — standalone Bandit is redundant
- HIPAA pivot means security and supply chain scanning are not optional
- Spiral pass model — each pass must unlock the next level of agent autonomy
- Existing milestones: Mutation Gate Live, E2E Contract Framework, Full Service Coverage, Agent Validation Protocol
- Existing in-flight: model-based testing (ENG-2439/2440), mutation pipeline fix (ENG-2424), contract framework (ENG-2425/2426)

## Options Considered

### A: Enrich Existing Milestones
Keep 4 milestones, expand scope. Minimal restructuring but milestones become bloated and lose clarity.
- Gains: No Linear restructuring
- Costs: Milestones too large for bounded agent tasks, names misleading
- Complexity: Low

### B: Verification Tiers Aligned to Passes
3 tiers mapped to spiral passes. Clear "why this pattern now" logic. Each tier unlocks next level of agent autonomy.
- Gains: Clean sequencing, exit tests per tier, maps to passes
- Costs: Replaces 4-milestone structure
- Complexity: Medium

### C: Verification Platform + Pattern Library
Build reusable platform first (fixtures, CI templates, generators). Maximum agent leverage but highest upfront investment.
- Gains: Agents create tests from recipes, scales to 10+ services
- Costs: Risk of over-engineering before knowing real needs
- Complexity: High

## Chosen Approach
**B with elements of C** — Tier the portfolio across passes (B's sequencing), but within each tier build reusable infrastructure first, then per-service application (C's platform instinct). The model-based testing plan already follows this internal pattern (tooling → fixtures → generator skill).

### Milestone restructure (4 → 6):

| # | Current | Proposed | Pass |
|---|---------|----------|------|
| 1 | Mutation Gate Live | **Mutation Gate Live** (no change) | 1 |
| 2 | *(new)* | **Security & Supply Chain Gates** | 1 |
| 3 | *(new)* | **Model-Based Testing** | 1 |
| 4 | E2E Contract Framework | **Interface Verification** (rename+expand) | 1→2 |
| 5 | Full Service Coverage | **Verification Platform** (redefine) | 2 |
| 6 | Agent Validation Protocol | **Production Confidence** (rename+expand) | 3 |

### Pass exit tests:
- **Pass 1:** Can an agent implement a feature and the gates catch real bugs?
- **Pass 2:** Can agents work on different services in parallel without breaking each other?
- **Pass 3:** Can we ship agent-built code to production with confidence?

## Key Context Discovered During Shaping

**Grounding corrections (validated via `/ground`):**
- Ruff S-rules already provide SAST — don't add standalone Bandit (redundant). Semgrep adds deeper semantic analysis (92% vs 88% detection) as a CI-only complement
- Pact CDC is overkill for monorepo internal services — Protocol-typed contracts (ENG-2425) suffice. Reserve Pact for Dashboard↔API boundary only
- pip-audit (dependency scanning) and detect-secrets (secrets detection) are missing from all milestones — HIPAA-relevant, low-effort
- pytest-testmon for test selection has known over-selection issues on module-level changes — Pass 3 optimization, not critical path
- Schemathesis is strongly validated (1.4-4.5x more defects than alternatives, built on Hypothesis)
- Locust preferred over k6 for Python team (native Python, distributed architecture)
- Syrupy is the leading Python snapshot testing library (zero-dependency, pytest-native)

**Research source:** Deep research report on testing infrastructure (`/Users/jordanmesches/Downloads/deep-research-report.md`) — comprehensive survey of ~20 testing patterns with tool recommendations and CI/CD integration guidance.

**"Verification = agent autonomy" thesis strongly grounded:**
- Anthropic 2026 Agentic Coding Trends Report: agents iterate until quality gates pass
- FeatureBench: 74% → 11% success between bounded/unbounded tasks — quality gates ARE boundaries
- FileScience's own backpressure research and judge-agent brief independently arrived at same thesis

## Related
- [[2026-03-09-platform-v2-roadmap-strategy|Platform V2 Roadmap Strategy]]
- [[2026-03-11-model-based-testing-state-machines|Model-Based Testing Brief]]
- [[2026-02-06-ai-agent-backpressure-tools|Agent Backpressure Tools Research]]
- [[2026-02-28-judge-taste-agents|Judge / Taste Agents Brief]]
- [[2026-02-25-eng-2135-ci-quality-gates-pr-checks|CI Quality Gates Plan]]
- ENG-2414: Design testing strategy for agent factory model (this brief resolves it)

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-11-testing-infrastructure-portfolio.md`
