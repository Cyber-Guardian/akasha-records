---
date: 2026-02-03T17:00:00-05:00
researcher: Claude
git_commit: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
branch: main
repository: filescience
topic: "Moving API Client into Components"
tags: [research, codebase, api-client, components, polylith, refactoring]
status: complete
last_updated: 2026-02-03
last_updated_by: Claude
---

# Research: Moving API Client into Components

**Date**: 2026-02-03T17:00:00-05:00
**Researcher**: Claude
**Git Commit**: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
**Branch**: main
**Repository**: filescience

## Research Question

How is the API client currently structured, what files need to move, and what patterns should the new component follow?

## Summary

The API client code is currently located inside `bases/filescience/discover/clouds/` and consists of two layers:
1. **Generic API client infrastructure** (`clouds/api/`) - 5 files with no cloud-specific dependencies
2. **Microsoft Graph client** (`clouds/microsoft/api/`) - 3 files implementing Microsoft-specific logic

These files are candidates for extraction into `components/filescience/cloud_api/` to break the dependency chain that currently prevents `entity-discovery-trigger` from importing just the API client without pulling in the entire discover base (which requires throttling, Valkey, DynamoDB).

## Detailed Findings

### Files to Extract

#### Generic API Layer (`bases/filescience/discover/clouds/api/`)

| File | Purpose | Dependencies |
|------|---------|--------------|
| `client.py` | `ApiClient`, `ApiComponent`, `PaginationStrategy`, `ResponseTemplate`, `ResponseIterator` | httpx, orjson, pydantic, suppression.py |
| `oauth.py` | OAuth2 auth implementations for Microsoft, Box, Clio, NetDocs | httpx, pydantic |
| `session.py` | `SessionManager` - httpx client factory with OAuth auth | httpx, httpx_retries, `filescience.models`, `filescience.discover.aws` |
| `suppression.py` | `SuppressionRule`, rate limit error handling | httpx, pydantic |
| `__init__.py` | Package exports | All above |

#### Microsoft-Specific Layer (`bases/filescience/discover/clouds/microsoft/api/`)

| File | Purpose | Dependencies |
|------|---------|--------------|
| `client.py` | `GraphClient`, `GraphResponse`, components (User, Group, Site, Onedrive, etc.) | httpx, pydantic, generic API layer, models.py |
| `models.py` | Pydantic models for Graph API responses (Drive, DriveItem, User, Group, Site, etc.) | pydantic |
| `__init__.py` | Package exports | client.py, models.py |

### Current Dependency Chain Problem

The discover base's `__init__.py` exports:
```python
from filescience.discover.manager import DiscoveryManager
from filescience.discover.dispatcher import DispatcherV2
from filescience.discover.queue import WorkQueue, WorkItem
```

Import chain:
1. `manager.py:29` imports `from filescience.throttling import ThrottleService`
2. `queue.py:12-17` imports throttling models (`AdmissionRequest`, `RequestPolicyType`, etc.)
3. Throttling requires `valkey-glide` (Valkey client)

Result: Any import from the discover base triggers throttling imports → requires Valkey client → fails in entity-discovery-trigger Lambda which only needs GraphClient.

### API Client Class Structure

#### Base `ApiClient` (`clouds/api/client.py:199-411`)

```
ApiClient
├── __init__(base_url, client, headers, default_params, default_pagination_strategy)
├── raw_request(method, endpoint, params, data, json_payload, suppression_rules)
├── request_from_spec(request_spec)
├── request(method, endpoint, response_model, params, data, json_payload)
├── get/post/put/delete/patch() - HTTP method shortcuts
├── paginate(endpoint, response_template, pagination_strategy, params)
└── iter_items(endpoint, response_template, pagination_strategy, params)
```

#### `GraphClient` (`clouds/microsoft/api/client.py:27-46`)

```
GraphClient(ApiClient)
├── __init__(client: httpx.AsyncClient)  # Sets base_url="https://graph.microsoft.com/v1.0"
├── .user = User(self)
├── .group = Group(self)
├── .site = Site(self)
├── .onedrive = Onedrive(self)
├── .outlook = Outlook(self)
├── .contacts = Contacts(self)
├── .todo = Todo(self)
├── .calendar = Calendar(self)
├── .planner = Planner(self)
└── .teams = Teams(self)
```

Each component (User, Group, Site, etc.) inherits from `ApiComponent[GraphClient]` and provides methods like:
- `user.list()` → yields `models.User`
- `group.list()` → yields `models.Group`
- `site.list()` → yields `models.Site`
- `onedrive.get_drives(entity)` → yields `models.Drive`

### Existing Component Patterns

The monorepo has 4 existing components with these patterns:

**Pattern 1 - Simple (models, domain_backup)**:
```
component/
├── __init__.py     # Imports from core.py, defines __all__
└── core.py         # Main implementation
```

**Pattern 2 - Hierarchical (throttling)**:
```
component/
├── __init__.py     # Re-exports from all submodules
├── service.py
├── async_service.py
├── models.py
├── valkey/         # Subpackage with own __init__.py
│   ├── __init__.py
│   └── ...
└── policies/       # Nested subpackage
    └── m365/
```

**Pattern 3 - Organized (dynamodb)**:
```
component/
├── __init__.py     # Comprehensive exports grouped by category
├── _types.py       # Private types (underscore prefix)
├── session.py
├── models.py
├── read/           # Subpackage for related operations
│   └── __init__.py
└── write/
    └── __init__.py
```

**Key conventions**:
- `__init__.py` defines public API via `__all__`
- Private modules use underscore prefix (`_types.py`, `_base.py`)
- Subpackages have their own `__init__.py` with re-exports
- Optional dependencies wrapped in try/except

### Proposed Component Structure

Based on existing patterns, `cloud_api` component could follow the hierarchical pattern:

```
components/filescience/cloud_api/
├── __init__.py             # Re-exports from submodules
├── client.py               # ApiClient, ApiComponent, ResponseTemplate, etc.
├── oauth.py                # OAuth2 implementations
├── session.py              # SessionManager
├── suppression.py          # SuppressionRule, error handling
└── microsoft/              # Subpackage for Microsoft Graph
    ├── __init__.py         # Re-exports GraphClient, models
    ├── client.py           # GraphClient and components
    └── models.py           # Pydantic response models
```

### Files Currently Importing API Client Code

#### Within Discover Base

| File | Import |
|------|--------|
| `dispatcher.py:5` | `from filescience.discover.clouds.api.session import SessionManager` |
| `microsoft/cloud.py:1` | `from filescience.discover.clouds.router import CloudRouter` |
| `microsoft/services/onedrive.py:9` | `from filescience.discover.clouds.microsoft.api.client import GraphClient` |

#### External Consumer

| File | Import |
|------|--------|
| `entity_discovery_trigger/entities.py:8` | `from filescience.discover.clouds.microsoft.api.client import GraphClient` |

### SessionManager AWS Dependency

`session.py` imports `from filescience.discover.aws import AWS` for SSM parameter access (OAuth credentials). Options:
1. Move `aws.py` into cloud_api component
2. Keep AWS access separate, inject credentials via constructor
3. Accept AWS dependency in cloud_api component

## Code References

### Generic API Client
- `bases/filescience/discover/clouds/api/client.py:199-411` - ApiClient class
- `bases/filescience/discover/clouds/api/client.py:86-139` - ResponseTemplate
- `bases/filescience/discover/clouds/api/client.py:141-197` - ResponseIterator
- `bases/filescience/discover/clouds/api/client.py:32-84` - PaginationStrategy implementations
- `bases/filescience/discover/clouds/api/client.py:413-418` - ApiComponent base

### Microsoft Graph Client
- `bases/filescience/discover/clouds/microsoft/api/client.py:27-46` - GraphClient
- `bases/filescience/discover/clouds/microsoft/api/client.py:19-25` - GraphResponse template
- `bases/filescience/discover/clouds/microsoft/api/client.py:49-247` - Graph components (User, Group, Site, Onedrive, etc.)

### Import Chain Files
- `bases/filescience/discover/__init__.py:3-7` - Discover base exports
- `bases/filescience/discover/manager.py:29` - ThrottleService import
- `bases/filescience/discover/queue.py:12-17` - Throttling model imports

### Existing Component Patterns
- `components/filescience/throttling/__init__.py` - Hierarchical pattern example
- `components/filescience/dynamodb/__init__.py` - Organized exports example
- `components/filescience/models/__init__.py` - Simple pattern example

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/handoffs/general/2026-02-03_entity-discovery-trigger-deployment-issues.md` - Documents the blocking issue that necessitates this extraction
- `memory-bank/thoughts/shared/plans/2026-02-03-entity-discovery-trigger-migration.md` - Parent plan for entity discovery trigger work
- `memory-bank/thoughts/shared/research/2026-02-03-entity-discovery-trigger-migration-brief.md` - Related migration analysis

## Related Research

- [[2026-01-30-components-filescience-usage-report|2026-01-30-components-filescience-usage-report.md]] - Existing component usage analysis
- [[2026-01-29-polylith-migration-source-analysis|2026-01-29-polylith-migration-source-analysis.md]] - Polylith migration patterns

## Open Questions

1. **AWS dependency in session.py** - Should `aws.py` move with the API client, or should credential injection be refactored?
2. **Other cloud providers** - Only Microsoft is currently implemented. Box, Clio, NetDocs OAuth exists but no full clients. Plan for future cloud clients?
3. **Testing** - Tests exist in `test/bases/filescience/discover/clouds/microsoft/api/test_site.py`. Move to `test/components/filescience/cloud_api/`?
4. **pyproject.toml registration** - New component needs entry in `[tool.polylith.bricks]` section
