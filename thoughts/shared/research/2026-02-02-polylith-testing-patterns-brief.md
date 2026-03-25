---
date: 2026-02-02T00:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-02-polylith-testing-patterns.md
last_generated: 2026-02-02T15:54:36.853721+00:00
---

# Research Brief: 2026-02-02-polylith-testing-patterns

## TL;DR

Polylith has **opinionated testing patterns** that differ between the original Clojure and Python implementations. For Python Polylith, the recommended approach is the **"loose" theme** with tests in a root-level `test/` directory that mirrors the brick structure. This enables:
- Better tooling compatibility (MyPy, Ruff, etc.)
- Incremental testing based on changed bricks
- Clear separation between fast brick tests and slow project tests

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Test markers vs directories**: Should slow tests use `@pytest.mark.slow` or live in `projects/{name}/test/`?
2. **Coverage thresholds**: What coverage percentage should be required for components?
3. **Moto vs LocalStack**: For DynamoDB tests, is moto sufficient or should LocalStack be considered?
4. **Parallel test execution**: How to configure `pytest-xdist` with Polylith namespace packages?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-02-polylith-testing-patterns.md`
