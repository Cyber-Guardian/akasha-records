---
date: 2026-02-02T00:00:00-05:00
researcher: Claude
git_commit: 206f8473b221e6fd8865a56201f079817232b03c
branch: main
repository: filescience
topic: "Bedrock Test Migration to Filescience Monorepo (Throttling + DynamoDB)"
tags: [research, codebase, test-migration, throttling, dynamodb, bedrock]
status: complete
last_updated: 2026-02-02
last_updated_by: Claude
---

# Research: Bedrock Test Migration to Filescience Monorepo

**Date**: 2026-02-02T00:00:00-05:00
**Researcher**: Claude
**Git Commit**: 206f8473b221e6fd8865a56201f079817232b03c
**Branch**: main
**Repository**: filescience

## Research Question

Migrate bedrock tests from `/Users/jordanmesches/Documents/work/filescience/bedrock/tests/unit/test_throttling` and `/Users/jordanmesches/Documents/work/filescience/bedrock/tests/unit/test_dynamo` to the corresponding throttling and dynamodb components in the filescience monorepo.

## Summary

The bedrock repository contains comprehensive unit tests for two components:
1. **Throttling** - ~30 test files covering ValkeyFunctions, ThrottleService, and M365 policies
2. **DynamoDB** - ~25 test files covering query, scan, bidirectional query, parallel scan, and delete operations

Both test suites need migration to the filescience monorepo's component structure. Currently, **neither component has any test files** in the monorepo.

## Detailed Findings

### Source: Bedrock Throttling Tests

**Location**: `/Users/jordanmesches/Documents/work/filescience/bedrock/tests/unit/test_throttling/`

**Structure**:
```
test_throttling/
├── __init__.py
├── conftest.py                      # Shared fixtures (mock_glide_client, mock_valkey_functions)
├── test_keys.py                     # Key generation tests (valkey_tree_meta_key, etc.)
├── test_policies.py                 # M365 policies, RateGateSpec, gate tiers
├── test_valkey_functions/
│   ├── __init__.py
│   ├── conftest.py                  # Async/sync mock client fixtures
│   ├── test_assertions.py           # Shared assertion helpers
│   ├── test_decode_response.py      # _decode_response tests
│   ├── test_tree_scoring.py         # get_optimal_throttle_context, etc.
│   ├── test_async_tree_scoring.py
│   ├── test_context_operations.py   # try_prune_throttle_context, penalize_throttle_context
│   ├── test_async_context_operations.py
│   ├── test_redis_helpers.py        # tree_exists, gate_exists, throttle_context_exists
│   ├── test_async_redis_helpers.py
│   ├── test_gate_functions.py       # register_gate, check_gate, calculate_gate_hierarchy_penalty
│   ├── test_async_gate_functions.py
│   ├── test_tree_functions.py       # register_tree, register_node, deregister_tree
│   └── test_async_tree_functions.py
└── test_service/
    ├── __init__.py
    ├── conftest.py                  # Mock fixtures for ThrottleService
    ├── test_assertions.py           # Shared assertion helpers
    ├── test_admission.py            # admit() tests with policy flow
    ├── test_async_admission.py
    ├── test_registration.py         # register_tree, register_gate, register_node
    ├── test_async_registration.py
    ├── test_orchestration.py        # request_capacity_by_context, penalize_by_context
    ├── test_async_orchestration.py
    ├── test_context_retrieval.py    # get_throttle_context, get_optimal_throttle_context
    └── test_async_context_retrieval.py
```

**Import Patterns** (must be updated):
- `from fsbedrock.throttling.valkey.functions import ValkeyFunctions`
- `from fsbedrock.throttling.valkey.async_functions import AsyncValkeyFunctions`
- `from fsbedrock.throttling.service import ThrottleService`
- `from fsbedrock.throttling.async_service import AsyncThrottleService`
- `from fsbedrock.throttling import RateGateSpec, RequestPolicy`
- `from fsbedrock.throttling.policies.m365 import ...`

**Test Count**: ~30 test files, 150+ test functions

---

### Source: Bedrock DynamoDB Tests

**Location**: `/Users/jordanmesches/Documents/work/filescience/bedrock/tests/unit/test_dynamo/`

**Structure**:
```
test_dynamo/
├── conftest.py                        # aiobotocore compatibility patch for moto
├── test_query_table/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # query_table sync tests
│   └── test_async.py                  # aquery_table async tests
├── test_iter_query_pages/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # iter_query_pages sync tests
│   └── test_async.py                  # aiter_query_pages async tests
├── test_scan_table/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # scan_table sync tests
│   └── test_async.py                  # ascan_table async tests
├── test_iter_scan_pages/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # iter_scan_pages sync tests
│   └── test_async.py                  # aiter_scan_pages async tests
├── test_bidirectional_query/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # BiDirectionalQueryExecutor.query()
│   ├── test_async.py                  # BiDirectionalQueryExecutor.aquery()
│   └── test_facade.py                 # query_table_bidirectionally, aquery_table_bidirectionally
├── test_parallel_scan/
│   ├── __init__.py
│   ├── assertions.py                  # Shared assertion helpers
│   ├── test_sync.py                   # scan_table_in_parallel sync tests
│   ├── test_async.py                  # ParallelScanExecutor.ascan() tests
│   └── test_facade.py                 # ascan_table_in_parallel, scan_table_in_parallel
└── test_delete/
    ├── __init__.py
    ├── assertions.py                  # Shared assertion helpers
    ├── test_sync.py                   # delete_items sync tests
    └── test_async.py                  # adelete_items async tests
```

**Import Patterns** (must be updated):
- `from fsbedrock.persistence.dynamodb import query_table, aquery_table`
- `from fsbedrock.persistence.dynamodb.read.query import iter_query_pages, aiter_query_pages, BiDirectionalQueryExecutor`
- `from fsbedrock.persistence.dynamodb.read.scan import iter_scan_pages, aiter_scan_pages, scan_table, ParallelScanExecutor`
- `from fsbedrock.persistence.dynamodb.delete import delete_items, adelete_items`

**Test Dependencies**:
- `moto` (AWS mocking)
- `boto3` (sync DynamoDB)
- `aioboto3` (async DynamoDB)
- `aiobotocore` (compatibility patch)
- `pytest-asyncio`
- `pytest-mock` (mocker fixture)

**Test Count**: ~25 test files, 60+ test functions

---

### Target: Filescience Monorepo Components

#### Throttling Component

**Location**: `/Users/jordanmesches/Documents/work/filescience/filescience/components/filescience/throttling/`

**Current Structure**:
```
components/filescience/throttling/
├── __init__.py
├── _base.py
├── models.py
├── service.py
├── async_service.py
├── policies/
│   ├── __init__.py
│   └── m365/
│       ├── __init__.py
│       ├── constants.py
│       ├── gate_specs.py
│       ├── policies.py
│       └── registry.py
└── valkey/
    ├── __init__.py
    ├── client.py
    ├── constants.py
    ├── keys.py
    ├── functions.py
    └── async_functions.py
```

**Test Directory**: **Does not exist** - must be created

**New Import Pattern**:
- `from filescience.throttling.valkey.functions import ValkeyFunctions`
- `from filescience.throttling.service import ThrottleService`
- etc.

#### DynamoDB Component

**Location**: `/Users/jordanmesches/Documents/work/filescience/filescience/components/filescience/dynamodb/`

**Current Structure**:
```
components/filescience/dynamodb/
├── __init__.py
├── _types.py
├── session.py
├── models.py
├── mapper.py
├── serializers.py
├── utils.py
├── delete.py
├── read/
│   ├── __init__.py
│   ├── get.py
│   ├── query.py
│   └── scan.py
├── write/
│   ├── __init__.py
│   ├── put.py
│   └── transaction.py
└── mappers/
    ├── __init__.py
    └── backup.py
```

**Test Directory**: **Does not exist** - must be created

**New Import Pattern**:
- `from filescience.dynamodb import query_table, aquery_table`
- `from filescience.dynamodb.read.query import iter_query_pages, BiDirectionalQueryExecutor`
- etc.

---

## Migration Requirements

### 1. Create Test Directories

```bash
# Throttling tests
mkdir -p components/filescience/throttling/tests/test_valkey_functions
mkdir -p components/filescience/throttling/tests/test_service

# DynamoDB tests
mkdir -p components/filescience/dynamodb/tests/test_query_table
mkdir -p components/filescience/dynamodb/tests/test_iter_query_pages
mkdir -p components/filescience/dynamodb/tests/test_scan_table
mkdir -p components/filescience/dynamodb/tests/test_iter_scan_pages
mkdir -p components/filescience/dynamodb/tests/test_bidirectional_query
mkdir -p components/filescience/dynamodb/tests/test_parallel_scan
mkdir -p components/filescience/dynamodb/tests/test_delete
```

### 2. Import Pattern Transformations

| Bedrock Import | Filescience Import |
|---------------|-------------------|
| `fsbedrock.throttling` | `filescience.throttling` |
| `fsbedrock.throttling.valkey` | `filescience.throttling.valkey` |
| `fsbedrock.throttling.policies.m365` | `filescience.throttling.policies.m365` |
| `fsbedrock.persistence.dynamodb` | `filescience.dynamodb` |
| `fsbedrock.persistence.dynamodb.read` | `filescience.dynamodb.read` |
| `fsbedrock.persistence.dynamodb.delete` | `filescience.dynamodb.delete` |

### 3. Test Dependencies to Add

For `pyproject.toml` of each component (or project using them):
```toml
[project.optional-dependencies]
test = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-mock>=3.0",
    "moto[dynamodb]>=5.0",  # DynamoDB only
    "aioboto3>=13.0",       # DynamoDB only
]
```

### 4. Relative Import Updates

Update relative imports in test assertion files:
- `from tests.unit.test_throttling.test_service.test_assertions import ...`
- Becomes: `from .test_assertions import ...` or absolute import path

---

## Code References

### Bedrock Throttling Test Files
- `/bedrock/tests/unit/test_throttling/conftest.py:1-40` - Shared fixtures
- `/bedrock/tests/unit/test_throttling/test_keys.py:1-47` - Key generation tests
- `/bedrock/tests/unit/test_throttling/test_policies.py:1-366` - Policy tests
- `/bedrock/tests/unit/test_throttling/test_valkey_functions/conftest.py:1-32` - Mock client fixtures
- `/bedrock/tests/unit/test_throttling/test_service/conftest.py:1-86` - Service fixtures

### Bedrock DynamoDB Test Files
- `/bedrock/tests/unit/test_dynamo/conftest.py:1-25` - aiobotocore patch
- `/bedrock/tests/unit/test_dynamo/test_iter_query_pages/test_sync.py:1-122` - Full query page tests
- `/bedrock/tests/unit/test_dynamo/test_bidirectional_query/test_async.py:1-139` - BiDirectional tests
- `/bedrock/tests/unit/test_dynamo/test_parallel_scan/test_async.py:1-87` - Parallel scan tests

### Target Components
- `/filescience/components/filescience/throttling/` - Throttling component (13 source files)
- `/filescience/components/filescience/dynamodb/` - DynamoDB component (14 source files)

---

## Architecture Documentation

### Test Pattern: Sync/Async Parity

Both test suites follow a consistent pattern:
1. **Shared assertion helpers** (`test_assertions.py` or `assertions.py`) that work for both sync/async
2. **Separate test files** for sync (`test_sync.py`) and async (`test_async.py`) variants
3. **Shared fixtures** in `conftest.py` with `@pytest.fixture` and `@pytest_asyncio.fixture`

### Test Pattern: Mocking Strategy

**Throttling Tests**:
- Mock `GlideClient` for ValkeyFunctions tests
- Mock `ValkeyFunctions` for ThrottleService tests
- Use `MagicMock` for sync, `AsyncMock` for async

**DynamoDB Tests**:
- Use `moto` `@mock_aws` decorator for real DynamoDB behavior
- Create actual tables with `boto3`/`aioboto3`
- Use `mocker.patch` for testing specific code paths (e.g., worker isolation)

### Key Configuration: aiobotocore/moto Compatibility

The DynamoDB tests require a compatibility patch (in conftest.py):
```python
@pytest.fixture(autouse=True)
def patch_aiobotocore_for_moto(mocker):
    # Patches aiobotocore.endpoint.convert_to_response_dict
    # to handle moto's synchronous response bodies
```

---

## Historical Context

This migration is part of the ongoing consolidation of the bedrock library into the filescience monorepo. Recent commits show:
- `206f847` - Add DynamoDB persistence utilities and type definitions
- `fc5d934` - Migrate Throttle Service + Discover

The throttling and dynamodb components have been migrated, but their tests have not yet been moved over.

---

## Open Questions

1. **Test location preference**: Should tests live in `components/filescience/throttling/tests/` or in a top-level `tests/unit/test_throttling/` directory?
2. **Monorepo test runner**: How are component tests run in the monorepo? Is there a central pytest configuration?
3. **CI/CD integration**: Are there existing GitHub Actions or similar that need to be updated to include the new tests?
4. **Coverage requirements**: Are there coverage thresholds that need to be met for new test additions?
