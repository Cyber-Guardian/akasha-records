# Idea Brief: Testing Pyramid for Agent Code

**Date:** 2026-03-05
**Status:** Shaped -> Planning

## Problem
The tools/ workspace has no CI coverage, tests that mock every integration seam ("lasagna mocking"), and no test gate before deployment. The triage agent rewrite hit 4 sequential runtime failures in production -- all in wiring between layers, all preventable with one construction test.

## Constraints
- tools/ is a separate Polylith workspace with its own pyproject.toml -- not covered by main CI
- PydanticAI provides TestModel, FunctionModel, ALLOW_MODEL_REQUESTS=False as first-party testing primitives
- respx already handles Linear HTTP transport mocking (better than unittest.mock.patch)
- Triage agent deploys to Lambda via local `make deploy`, not through CI deploy workflows

## Options Considered

### A. Plug the Holes
Fix exactly what broke: test gate in Makefile, one construction test, fix MagicMock spec.
- Gains: Fast, directly prevents repeat
- Costs: No systemic discipline -- next rewrite hits different gaps
- Complexity: Low

### B. Testing Pyramid for Agent Code (chosen)
Structured test layers using PydanticAI-native primitives: construction tests, TestModel/FunctionModel unit tests, spec'd mocks, CI enforcement.
- Gains: Reusable discipline, catches wiring bugs structurally, PydanticAI-native
- Costs: 3-4 sessions investment
- Complexity: Medium

### C. Eval-Driven Development
Everything in B plus pydantic-evals regression datasets, LLM-as-judge, online monitoring.
- Gains: Catches behavioral regressions, not just wiring
- Costs: Significant investment, overkill for current agent complexity
- Complexity: High

## Chosen Approach
**Testing Pyramid for Agent Code** -- it directly prevents the incident class (construction tests catch all 4 failures), establishes reusable discipline, and uses PydanticAI's own testing philosophy rather than fighting the framework.

## Key Context Discovered During Shaping
- All 4 triage failures were in integration wiring, not business logic ([[2026-03-05-triage-agent-deployment-failures|Post-Mortem]])
- PydanticAI docs recommend ALLOW_MODEL_REQUESTS=False globally + TestModel as the primary testing pattern
- Industry consensus (Simon Willison, Tweag, Lincoln Loop): fakes over mocks -- mocks couple to call signatures, fakes couple to contracts
- respx is already a transport-level fake (good) -- no need for VCR cassettes yet
- test_handler.py already has 2 integration smoke tests (TestAgentIntegration) added during the fix PR -- these need to be formalized and expanded
- MockCheckpointContext is duplicated in test_handler.py and test_runner.py -- needs consolidation

## Deferred (not in this plan)
- VCR cassettes (pytest-recording) -- respx sufficient for now
- inline-snapshot + dirty-equals -- no complex message assertions needed yet
- Helm CLI testing -- doesn't deploy to Lambda, different failure class
- pydantic-evals -- graduate to this when agent complexity warrants it

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-05-agent-testing-pyramid.md`
