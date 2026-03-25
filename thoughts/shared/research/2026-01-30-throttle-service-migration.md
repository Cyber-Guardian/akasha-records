---
date: 2026-01-30T12:00:00-05:00
researcher: Claude
git_commit: a9c4eca99026b1a655109edac3a0383532c46aab
branch: main
repository: filescience
topic: "ThrottleService Migration - What needs to be migrated"
tags: [research, codebase, throttling, polylith, migration]
status: complete
last_updated: 2026-01-30
last_updated_by: Claude
---

# Research: ThrottleService Migration

**Date**: 2026-01-30T12:00:00-05:00
**Researcher**: Claude
**Git Commit**: a9c4eca99026b1a655109edac3a0383532c46aab
**Branch**: main
**Repository**: filescience

## Research Question
What is needed to complete the ThrottleService migration to the Polylith monorepo?

## Summary

The throttling component is **partially migrated**. The data models, policies, and synchronous `ValkeyFunctions` wrapper are already present. Three files are missing and need to be copied from the bedrock source:

1. `service.py` - Sync `ThrottleService` class (977 lines)
2. `async_service.py` - Async `AsyncThrottleService` class (900 lines)
3. `valkey/async_functions.py` - Async `AsyncValkeyFunctions` class (818 lines)

## Detailed Findings

### Current Throttling Component State

**Already migrated** (`components/filescience/throttling/`):
- `__init__.py` - Exports with try/except fallback for missing services
- `models.py` - All data models (RateGateSpec, ConcurrencyGateSpec, GateCharge, Lease, AdmissionToken, AdmissionResult, RequestPolicy, ThrottleContext)
- `_base.py` - Validation utilities
- `valkey/__init__.py` - Valkey module exports
- `valkey/client.py` - Client factory functions (create_valkey_client, create_async_valkey_client)
- `valkey/constants.py` - Valkey key constants
- `valkey/keys.py` - Key generation functions
- `valkey/functions.py` - Sync ValkeyFunctions class (already migrated)
- `policies/m365/` - M365 policy implementations (OnedriveTraversalPolicy, M365ThrottleContext)

**Missing** (indicated by try/except ImportError patterns):
- `service.py` - `ThrottleService` is set to `None` on import failure
- `async_service.py` - `AsyncThrottleService` is set to `None` on import failure
- `valkey/async_functions.py` - `AsyncValkeyFunctions` is set to `None` on import failure

### Source Files to Migrate

All source files are located at: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/throttling/`

#### 1. service.py (ThrottleService)

**Source**: `fsbedrock/throttling/service.py` (977 lines)
**Target**: `components/filescience/throttling/service.py`

**Import changes required**:
```python
# FROM:
from glide_sync import GlideClient
from fsbedrock.throttling._base import (...)
from fsbedrock.throttling.models import (...)
from fsbedrock.throttling.valkey import (...)

# TO:
from glide_sync import GlideClient
from filescience.throttling._base import (...)
from filescience.throttling.models import (...)
from filescience.throttling.valkey import (...)
```

**Key API**:
- `__init__(self, client: GlideClient)` - Takes GlideClient, wraps in ValkeyFunctions
- `.functions` property - Escape hatch to ValkeyFunctions
- Registration: `register_tree()`, `register_gate()`, `register_node()`, `register_root_node()`, `deregister_tree()`
- Context retrieval: `get_throttle_context()`, `get_optimal_throttle_context()`, `get_throttle_context_tree_id()`
- Capacity: `request_capacity_by_context()`, `request_capacity_by_policy()`, `penalize_by_context()`
- Admission: `admit()`, `extend_token()`, `release_token()`

#### 2. async_service.py (AsyncThrottleService)

**Source**: `fsbedrock/throttling/async_service.py` (900 lines)
**Target**: `components/filescience/throttling/async_service.py`

**Import changes required**:
```python
# FROM:
from glide import GlideClient as AsyncGlideClient  # try/except
from fsbedrock.throttling._base import (...)
from fsbedrock.throttling.models import (...)
from fsbedrock.throttling.valkey import (...)

# TO:
from glide import GlideClient as AsyncGlideClient  # try/except
from filescience.throttling._base import (...)
from filescience.throttling.models import (...)
from filescience.throttling.valkey import (...)
```

**Key API**: Same as ThrottleService but all methods are async.

#### 3. valkey/async_functions.py (AsyncValkeyFunctions)

**Source**: `fsbedrock/throttling/valkey/async_functions.py` (818 lines)
**Target**: `components/filescience/throttling/valkey/async_functions.py`

**Import changes required**:
```python
# FROM:
from glide import GlideClient as AsyncGlideClient  # try/except
from fsbedrock.throttling.valkey.constants import (...)
from fsbedrock.throttling.valkey.keys import (...)

# TO:
from glide import GlideClient as AsyncGlideClient  # try/except
from filescience.throttling.valkey.constants import (...)
from filescience.throttling.valkey.keys import (...)
```

**Key API**: Same as ValkeyFunctions but all methods are async.

### Dependencies Already Satisfied

The `valkey-glide-sync>=2.0.0` package was added to all pyproject.toml files in the previous session:
- Root `pyproject.toml`
- `projects/discover/pyproject.toml`
- `projects/valkey-queue-processor/pyproject.toml`

This provides:
- `glide_sync` module (sync client)
- `glide` module (async client)

### Current Export Pattern

The throttling `__init__.py` uses try/except to gracefully handle missing services:

```python
# components/filescience/throttling/__init__.py (lines 16-24)
try:
    from filescience.throttling.service import ThrottleService
except ImportError:
    ThrottleService = None  # type: ignore

try:
    from filescience.throttling.async_service import AsyncThrottleService
except ImportError:
    AsyncThrottleService = None  # type: ignore
```

This means once the files are added, they will automatically be exported.

### Valkey Module Export Pattern

Similarly, `valkey/__init__.py` has:

```python
# components/filescience/throttling/valkey/__init__.py (lines 7-10)
try:
    from filescience.throttling.valkey.async_functions import AsyncValkeyFunctions
except ImportError:
    AsyncValkeyFunctions = None  # type: ignore
```

### Usage in Codebase

The ThrottleService is used in the valkey-queue-processor Lambda:

**`bases/filescience/valkey_queue_processor/handler.py:103-126`**:
```python
valkey_client = get_valkey_client()
throttle_service = ThrottleService(valkey_client)
throttle_registration_service = create_throttle_registration_service(throttle_service)
```

**`bases/filescience/valkey_queue_processor/services/throttle_service.py`**:
Uses ThrottleService's `.functions` property to access low-level ValkeyFunctions methods.

## Code References

| File | Description |
|------|-------------|
| `components/filescience/throttling/__init__.py:16-24` | Try/except exports for missing services |
| `components/filescience/throttling/valkey/__init__.py:7-10` | Try/except for AsyncValkeyFunctions |
| `fsbedrock/throttling/service.py` | Source ThrottleService (977 lines) |
| `fsbedrock/throttling/async_service.py` | Source AsyncThrottleService (900 lines) |
| `fsbedrock/throttling/valkey/async_functions.py` | Source AsyncValkeyFunctions (818 lines) |

## Migration Steps

1. **Copy** `fsbedrock/throttling/service.py` → `components/filescience/throttling/service.py`
2. **Copy** `fsbedrock/throttling/async_service.py` → `components/filescience/throttling/async_service.py`
3. **Copy** `fsbedrock/throttling/valkey/async_functions.py` → `components/filescience/throttling/valkey/async_functions.py`
4. **Update imports** in all three files: `fsbedrock.throttling` → `filescience.throttling`
5. **Verify** with `uv run poly check` - should pass without errors
6. **Test** with `uv run python -c "from filescience.throttling import ThrottleService; print(ThrottleService)"`

## Historical Context (from memory-bank/thoughts/)

- Plan: `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md` (Phase 3 specifies throttling component structure)
- Handoff: `memory-bank/thoughts/shared/handoffs/general/2026-01-29_16-50-00_polylith-migration-phase4.md` (notes ThrottleService pending)

## Open Questions

None - all information needed for migration has been identified.
