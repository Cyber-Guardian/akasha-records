# Entity Discovery Trigger - Deployment Issues Handoff

**Date**: 2026-02-03
**Status**: Blocked on architectural refactor

## What Was Accomplished

### Phase 4 Tests - COMPLETE
- Created test directory structure: `test/bases/filescience/discover/clouds/microsoft/api/` and `test/bases/filescience/entity_discovery_trigger/`
- **26 tests written and passing**:
  - `test_site.py` (6 tests): Site component and model tests
  - `test_entities.py` (15 tests): RemoteEntity, check_entity_active, get_entities
  - `test_handler_integration.py` (5 tests): Lambda handler import, make_entity_id
- Full test suite passes: 319 tests

### Lambda Deployed But Not Working
- Lambda deployed to `filescience-dev-entity-discovery-trigger` in us-east-1
- Core layer updated multiple times to add missing dependencies

## The Problem: Dependency Chain Issues

When trying to invoke the Lambda, we hit a cascade of missing module errors:

1. `aioboto3` - Added to core layer
2. `types_boto3_dynamodb` / `types_aiobotocore_dynamodb` - Added (runtime type imports in `_types.py`)
3. `httpx` / `anyio` - Added
4. `httpx-retries` - Added
5. `orjson` - Added
6. **`filescience.throttling`** - This is where it gets problematic

### Root Cause: Discover Base Has Throttling Imports

The `bases/filescience/discover/__init__.py` imports:
```python
from filescience.discover.manager import DiscoveryManager
from filescience.discover.dispatcher import DispatcherV2
from filescience.discover.queue import WorkQueue, WorkItem
```

Both `manager.py` and `queue.py` import from `filescience.throttling`:
- `manager.py:29`: `from filescience.throttling import ThrottleService`
- `queue.py:12`: `from filescience.throttling import (...)`

The entity-discovery-trigger only needs:
- `filescience.discover.clouds.api.session.SessionManager`
- `filescience.discover.clouds.microsoft.api.client.GraphClient`

But including ANY part of the discover base pulls in the __init__.py which pulls in throttling.

### Attempted Workarounds

1. **Selective brick inclusion** - Tried including only `discover/clouds` and `discover/aws.py` without `discover/__init__.py`
   - Problem: Python needs `__init__.py` to treat `discover` as a package for imports to work

2. **Manual __init__.py stub** - About to create a minimal stub when interrupted
   - This is hacky and fragile

## Proposed Solution: Extract Cloud API to Separate Component

Create a new component: `components/filescience/cloud_api/` containing:
- OAuth2 auth implementations (Microsoft, Box, Clio, NetDocs)
- SessionManager
- Generic API client base classes
- GraphClient and other cloud-specific clients

This would:
1. Eliminate the circular dependency between discover (queue/manager) and the API layer
2. Allow entity-discovery-trigger to import only what it needs
3. Follow the Polylith principle of small, focused components

### Files to Move

From `bases/filescience/discover/clouds/`:
```
api/
├── __init__.py
├── client.py      # ApiClient, ApiComponent base classes
├── oauth.py       # OAuth2 implementations
├── session.py     # SessionManager
└── suppression.py # Rate limit suppression

microsoft/api/
├── __init__.py
├── client.py      # GraphClient, Site, User, Group, etc.
└── models.py      # Pydantic models
```

### Current Layer Requirements (for reference)

`infrastructure/layers/core-requirements.txt` now contains:
```
boto3==1.37.3
aioboto3==14.2.0
httpx
httpx-retries
anyio
orjson
types-boto3-dynamodb
types-aiobotocore-dynamodb
pydantic==2.12.5
pydantic-settings==2.12.0
```

## Files Changed (Uncommitted)

- `test/bases/filescience/discover/clouds/microsoft/api/test_site.py` - NEW
- `test/bases/filescience/entity_discovery_trigger/test_entities.py` - NEW
- `test/bases/filescience/entity_discovery_trigger/test_handler_integration.py` - NEW
- `infrastructure/layers/core-requirements.txt` - Updated with new deps
- `infrastructure/live/non-prod/us-east-1/dev/entity-discovery-trigger/terragrunt.hcl` - Removed AWS_REGION env var
- `projects/entity-discovery-trigger/pyproject.toml` - Tried selective brick inclusion

## Next Steps

1. **Create `components/filescience/cloud_api/` component**
   - Move API client base classes
   - Move OAuth implementations
   - Move SessionManager
   - Move GraphClient and models

2. **Update discover base** to import from cloud_api component
   - `manager.py`, `queue.py`, etc. import from `filescience.cloud_api`

3. **Update entity-discovery-trigger** to depend on cloud_api component instead of discover base

4. **Rebuild and redeploy** Lambda with clean dependency tree

## References

- Plan: `memory-bank/thoughts/shared/plans/2026-02-03-entity-discovery-trigger-migration.md`
- Research: `memory-bank/thoughts/shared/research/2026-02-03-entity-discovery-trigger-migration-brief.md`
