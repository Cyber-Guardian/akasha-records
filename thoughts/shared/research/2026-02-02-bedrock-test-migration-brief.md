---
date: 2026-02-02T00:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-02-bedrock-test-migration.md
last_generated: 2026-02-02T15:49:09.506132+00:00
---

# Research Brief: 2026-02-02-bedrock-test-migration

## TL;DR

The bedrock repository contains comprehensive unit tests for two components:
1. **Throttling** - ~30 test files covering ValkeyFunctions, ThrottleService, and M365 policies
2. **DynamoDB** - ~25 test files covering query, scan, bidirectional query, parallel scan, and delete operations

Both test suites need migration to the filescience monorepo's component structure. Currently, **neither component has any test files** in the monorepo.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Test location preference**: Should tests live in `components/filescience/throttling/tests/` or in a top-level `tests/unit/test_throttling/` directory?
2. **Monorepo test runner**: How are component tests run in the monorepo? Is there a central pytest configuration?
3. **CI/CD integration**: Are there existing GitHub Actions or similar that need to be updated to include the new tests?
4. **Coverage requirements**: Are there coverage thresholds that need to be met for new test additions?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-02-bedrock-test-migration.md`
