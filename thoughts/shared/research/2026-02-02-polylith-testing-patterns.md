---
date: 2026-02-02T00:00:00-05:00
researcher: Claude
git_commit: 206f8473b221e6fd8865a56201f079817232b03c
branch: main
repository: filescience
topic: "Optimal Testing Patterns for Polylith-Style Repositories"
tags: [research, polylith, testing, pytest, architecture]
status: complete
last_updated: 2026-02-02
last_updated_by: Claude
---

# Research: Optimal Testing Patterns for Polylith-Style Repositories

**Date**: 2026-02-02T00:00:00-05:00
**Researcher**: Claude
**Git Commit**: 206f8473b221e6fd8865a56201f079817232b03c
**Branch**: main
**Repository**: filescience

## Research Question

What are the optimal testing patterns for Polylith-style monorepo architectures? Where should tests live, how should they be organized, and what are the best practices from the Polylith community?

## Summary

Polylith has **opinionated testing patterns** that differ between the original Clojure and Python implementations. For Python Polylith, the recommended approach is the **"loose" theme** with tests in a root-level `test/` directory that mirrors the brick structure. This enables:
- Better tooling compatibility (MyPy, Ruff, etc.)
- Incremental testing based on changed bricks
- Clear separation between fast brick tests and slow project tests

## Detailed Findings

### 1. Where Tests Should Live

#### Python Polylith: Loose Theme (Recommended)

**Official recommendation**: Tests at workspace root in a `test/` folder with mirrored structure.

```
workspace-root/
├── components/
│   └── filescience/
│       └── throttling/
│           └── core.py
├── bases/
│   └── filescience/
│       └── discover/
│           └── core.py
├── test/                          # ← Root-level test directory
│   ├── components/
│   │   └── filescience/
│   │       └── throttling/
│   │           └── test_core.py
│   └── bases/
│       └── filescience/
│           └── discover/
│               └── test_core.py
├── projects/
│   └── my_project/
│       └── test/                  # ← Optional: slow/integration tests
└── pyproject.toml
```

**Why NOT inside components** (TDD theme):
- Poor Python tooling integration
- MyPy, Ruff, and other tools have issues with nested package structures
- Not recommended for Python despite being the Clojure convention

#### Clojure Polylith: Tests Inside Bricks

The original Clojure implementation places tests inside each brick:
```
components/
└── parser/
    ├── src/
    └── test/     # ← Tests inside component
```

This works well for Clojure's namespace system but doesn't translate cleanly to Python.

---

### 2. Test Organization: Two-Level Approach

Polylith recommends a **two-level testing strategy**:

#### Level 1: Brick Tests (Fast)

**Purpose**: Quick feedback loop during development
**Location**: `test/components/` and `test/bases/`
**Characteristics**:
- Run in < 100ms
- Unit tests
- Integration tests using in-memory databases
- No external dependencies

**Key quote from docs**: "Does that mean we recommend only putting unit tests in your bricks? No. As long as the tests are fast (e.g., by using in-memory databases), you should put them in the bricks they belong to."

#### Level 2: Project Tests (Slow)

**Purpose**: Deployment-specific and slow integration tests
**Location**: `projects/{name}/test/`
**Characteristics**:
- > 100ms execution time
- Require external services
- Specialized setup/teardown
- Deployment-specific scenarios

**When to use**:
- Tests that require real database connections
- External API integration tests
- End-to-end tests for specific deployments

---

### 3. Pytest Configuration for Polylith

**Required configuration** in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = [
    "--import-mode=importlib",
]
consider_namespace_packages = true
pythonpath = ["components", "bases", "test"]
```

**Why this matters**: Without `consider_namespace_packages = true`, pytest will fail to discover tests in namespace packages.

**Additional recommended options**:
```toml
[tool.pytest.ini_options]
testpaths = ["test"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
asyncio_mode = "auto"  # If using pytest-asyncio
```

---

### 4. Running Tests

#### Standard Test Execution

```bash
# With uv (your toolchain)
uv run pytest

# Run specific component tests
uv run pytest test/components/filescience/throttling/

# Run with coverage
uv run pytest --cov=components --cov=bases
```

#### Incremental Testing (Changed Code Only)

Use `poly diff` to identify changed bricks, then run only affected tests:

```bash
# Get changed bricks
poly diff --bricks --short

# Output: throttling,dynamodb

# Convert to pytest filter
uv run pytest -k "throttling or dynamodb"
```

**CI/CD pattern**:
```bash
# In CI, mark stable points with tags
git tag stable-$(date +%Y%m%d)

# poly diff compares against last stable tag
CHANGED=$(poly diff --bricks --short)
if [ -n "$CHANGED" ]; then
    uv run pytest -k "${CHANGED//,/ or }"
fi
```

---

### 5. Dependency Management for Tests

#### Centralized Dev Dependencies

All test tools go in the root `pyproject.toml`:

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-mock>=3.0",
    "pytest-cov>=4.0",
    "moto[dynamodb]>=5.0",
    "mypy>=1.0",
    "ruff>=0.1",
]
```

**Key principle**: Development `pyproject.toml` has ALL dependencies (all bricks + all third-party libraries + all dev tools). Project-specific `pyproject.toml` files have only production dependencies.

#### Shared Fixtures

Use `conftest.py` at the workspace root for shared fixtures:

```
test/
├── conftest.py              # ← Shared fixtures for all tests
├── components/
│   └── filescience/
│       └── throttling/
│           ├── conftest.py  # ← Component-specific fixtures
│           └── test_core.py
```

---

### 6. Testing Component Interactions

**Key insight**: Polylith's architecture minimizes the need for mocking.

> "New tests are easy to write, and you can avoid mocking in most cases because you can access all components from your workspace."

**How it works**:
- Tests run from **project context** - all bricks are available
- Tests can access implementing namespaces directly
- Only mock external systems (APIs, databases not using in-memory alternatives)

**When mocking IS needed**:
- External HTTP APIs
- Real databases (when in-memory alternatives aren't sufficient)
- Time-dependent code
- Random/non-deterministic behavior

---

### 7. Real-World Examples

#### Python Polylith Examples

| Repository | Toolchain | Notes |
|------------|-----------|-------|
| [python-polylith-example](https://github.com/DavidVujic/python-polylith-example) | Poetry | Official example from maintainer |
| [python-polylith-example-uv](https://github.com/DavidVujic/python-polylith-example-uv) | uv | Modern uv-based example |
| [python-polylith-microservices-example](https://github.com/ttamg/python-polylith-microservices-example) | Various | Multi-service architecture |

#### Clojure Polylith Examples

| Repository | Notes |
|------------|-------|
| [clojure-polylith-realworld-example-app](https://github.com/furkan3ayraktar/clojure-polylith-realworld-example-app) | Full CRUD app with auth |

---

## Recommendations for Filescience

Based on this research, here's the recommended test structure:

### Proposed Structure

```
filescience/
├── components/
│   └── filescience/
│       ├── throttling/
│       └── dynamodb/
├── bases/
│   └── filescience/
│       └── discover/
├── test/                              # ← NEW: Root test directory
│   ├── conftest.py                    # Shared fixtures
│   ├── components/
│   │   └── filescience/
│   │       ├── throttling/
│   │       │   ├── conftest.py
│   │       │   ├── test_keys.py
│   │       │   ├── test_policies.py
│   │       │   ├── test_valkey_functions/
│   │       │   │   ├── conftest.py
│   │       │   │   ├── test_assertions.py
│   │       │   │   ├── test_decode_response.py
│   │       │   │   └── ...
│   │       │   └── test_service/
│   │       │       ├── conftest.py
│   │       │       ├── test_assertions.py
│   │       │       └── ...
│   │       └── dynamodb/
│   │           ├── conftest.py
│   │           ├── test_query_table/
│   │           ├── test_scan_table/
│   │           └── ...
│   └── bases/
│       └── filescience/
│           └── discover/
│               └── test_*.py
├── projects/
│   └── discover/
│       └── test/                      # Slow integration tests (optional)
└── pyproject.toml
```

### Pytest Configuration

Add to root `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = [
    "--import-mode=importlib",
    "-ra",
    "--strict-markers",
]
testpaths = ["test"]
pythonpath = ["components", "bases", "test"]
consider_namespace_packages = true
asyncio_mode = "auto"
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-mock>=3.0",
    "pytest-cov>=4.0",
    "moto[dynamodb]>=5.0",
    "aioboto3>=13.0",
]
```

### Import Pattern for Tests

```python
# In test/components/filescience/throttling/test_keys.py
from filescience.throttling.valkey.keys import (
    valkey_tree_meta_key,
    valkey_node_key,
)

def test_valkey_tree_meta_key():
    assert valkey_tree_meta_key("m365:123:backup456:traversal") == "{m365:123:backup456:traversal}:meta"
```

---

## Code References

- Python Polylith Docs - Testing: https://davidvujic.github.io/python-polylith-docs/testing/
- Polylith Official - Testing: https://polylith.gitbook.io/poly/workflow/testing
- Polylith - Testing Incrementally: https://polylith.gitbook.io/polylith/introduction/testing-incrementally
- Python Polylith - Dependencies: https://davidvujic.github.io/python-polylith-docs/dependencies/

---

## Open Questions

1. **Test markers vs directories**: Should slow tests use `@pytest.mark.slow` or live in `projects/{name}/test/`?
2. **Coverage thresholds**: What coverage percentage should be required for components?
3. **Moto vs LocalStack**: For DynamoDB tests, is moto sufficient or should LocalStack be considered?
4. **Parallel test execution**: How to configure `pytest-xdist` with Polylith namespace packages?
