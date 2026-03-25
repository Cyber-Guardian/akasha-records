---
date: 2026-01-30T12:00:00-05:00
researcher: Claude
git_commit: fc5d9342bdb294d487fdebec8a7d1003fd3b5a70
branch: main
repository: filescience
topic: "DynamoDB Utils Migration Analysis - Bedrock to Monorepo"
tags: [research, codebase, dynamodb, migration, discover, fsbedrock]
status: complete
last_updated: 2026-01-30
last_updated_by: Claude
---

# Research: DynamoDB Utils Migration Analysis

**Date**: 2026-01-30T12:00:00-05:00
**Researcher**: Claude
**Git Commit**: fc5d9342bdb294d487fdebec8a7d1003fd3b5a70
**Branch**: main
**Repository**: filescience

## Research Question

We need to migrate DynamoDB utils from `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock`, understand how they were used in the old discover (`/Users/jordanmesches/Documents/work/filescience-discover-v2/src`), and evaluate the current state of the migrated discover (`bases/filescience/discover`).

## Summary

The DynamoDB utilities in `fsbedrock` provide a sophisticated 8-layer abstraction for DynamoDB operations supporting both sync and async patterns. The old discover codebase uses these utilities extensively for persisting backup discovery metadata (entities, resources, artifacts, delta states) and managing a work queue. The migrated discover in the monorepo has **stubbed out all DynamoDB persistence operations** - this is the primary gap that needs to be addressed.

## Detailed Findings

### 1. fsbedrock DynamoDB Utilities Architecture

**Location**: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/persistence/dynamodb/`

The utilities implement an 8-layer architecture:

| Layer | Purpose | Key Files |
|-------|---------|-----------|
| 1. Types | Type definitions | `_types.py` |
| 2. Session | Optimized aioboto3 sessions | `session.py` |
| 3. Read Ops | get, query, scan | `read/get.py`, `read/query.py`, `read/scan.py` |
| 4. Write Ops | put, transaction, delete | `write/put.py`, `write/transaction.py`, `delete.py` |
| 5. Serialization | Item ↔ Pydantic models | `serializers.py` |
| 6. Data Models | Persistence models | `models.py` |
| 7. High-Level Utils | Model CRUD operations | `utils.py` |
| 8. Domain Mapping | Persistence ↔ Domain models | `domain/backup/mappers.py` |

#### Key Type Definitions (`_types.py`)
- `Item = dict[str, TableAttributeType]` - Single DynamoDB item
- `ItemCollection = list[Item]` - Multiple items

#### Session Management (`session.py`)
- `OptimizedAioSession` - Custom session with optimized JSON parser (bypasses shape validation)
- `aioboto3_config` - Pre-configured with 1000 max connections, adaptive retries (10 attempts), 3s connect / 60s read timeouts
- `create_session(use_optimized_session=True)` - Factory function

#### Read Operations

**Get** (`read/get.py`):
- `get_item(table, key)` / `aget_item(table, key)` - Single item retrieval

**Query** (`read/query.py`):
- `query_table(table, **kwargs)` / `aquery_table(table, **kwargs)` - Standard query
- `BiDirectionalQueryExecutor` - Queries forward and backward simultaneously, merges at collision point (~18% speedup on large queries)
- `query_table_bidirectionally(table_name, session, **kwargs)` - Convenience wrapper

**Scan** (`read/scan.py`):
- `scan_table(table, **kwargs)` / `ascan_table(table, **kwargs)` - Standard scan
- `ParallelScanExecutor` - Splits table into segments, scans in parallel workers
- `scan_table_in_parallel(table_name, concurrency, session, **kwargs)` - Convenience wrapper

#### Write Operations

**Put** (`write/put.py`):
- `put_item(table, **kwargs)` / `aput_item(table, **kwargs)` - Single item write

**Transaction** (`write/transaction.py`):
- `put_items_in_transaction(table, items, **kwargs)` / `aput_items_in_transaction(...)` - Atomic multi-item write

**Delete** (`delete.py`):
- `delete_items(table_name, items, concurrency=10, session)` / `adelete_items(...)` - Batch delete with workers

#### Serialization (`serializers.py`)

**`PydanticSerializer[PersistenceModel]`**:
- `deserialize_item(item)` - Single item to Pydantic model
- `deserialize_stream(stream)` / `adeserialize_stream(stream)` - Stream of items to models
- `serialize(model)` - Model to item(s)

**`PydanticNodeSerializer`** - For multi-row node pattern (Resource/Artifact):
- Handles IDENTITY + LINK item pairs
- Validates node type consistency during deserialization

**Pre-configured Serializers**:
- `DeltaStateSerializer = PydanticSerializer(DeltaStateItem)`
- `EntitySerializer = PydanticSerializer(EntityItem)`
- `ResourceSerializer = PydanticNodeSerializer("RES")`
- `ArtifactSerializer = PydanticNodeSerializer("ART")`

#### Persistence Models (`models.py`)

```
BaseItem (pk, sk, tenant_id)
├── LinkableNodeItem (parent_ref, parent_type, parent_id)
│   ├── NodeLinkItem (created_at, name, id)
│   └── DeltaStateItem (state, created_at)
├── NodeIdentityItem (cloud_external_id, cloud_id, service_id, id)
└── EntityItem (active, cloud_external_id, name, type, tenant_id, org_id, cloud_id, id)
```

#### High-Level Utilities (`utils.py`)

**Read Operations**:
- `get_model(table_name, serializer, model_mapper, key, *, session, resource)` / `aget_model(...)`
- `get_models(table_name, serializer, model_mapper, executor, *, session, resource)` / `aget_models(...)`
- `get_model_from_table(table, serializer, model_mapper, key)` / `aget_model_from_table(...)`
- `get_models_from_table(table, serializer, model_mapper, executor)` / `aget_models_from_table(...)`

**Write Operations**:
- `write_model(table_name, serializer, model_mapper, model, *, session, resource)` / `awrite_model(...)`
- `write_model_to_table(table, serializer, model_mapper, model)` / `awrite_model_to_table(...)`
- `transactional_write_model(...)` / `atransactional_write_model(...)`
- `transactional_write_model_to_table(...)` / `atransactional_write_model_to_table(...)`

**Executor Pattern**:
- `Query(kwargs)` - Query executor
- `Scan(kwargs)` - Scan executor
- Both implement `ReadExecutor` abstract base with `execute(table)` / `aexecute(table)`

#### Domain Model Mappers (`domain/backup/mappers.py`)

**`EntityDynamoDBModelMapper`**:
- PK: `TENANT#{tenant_id}`
- SK: `ENTITY#{id}`

**`ResourceDynamoDBModelMapper`**:
- Returns tuple: `(NodeIdentityItem, NodeLinkItem)`
- PK: `TENANT#{tenant_id}#RES#{id}`
- SK: `IDENTITY` or `LINK#{created_at_timestamp}`

**`ArtifactDynamoDBModelMapper`**:
- Same pattern as Resource but with `ART` prefix

**`DeltaStateDynamoDBModelMapper`**:
- PK: `TENANT#{tenant_id}#RES#{parent.id}`
- SK: `DELTA`

### 2. Old Discover DynamoDB Usage

**Location**: `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/`

#### Tables Used

| Table | Purpose |
|-------|---------|
| `resources` | Backup metadata (entities, resources, artifacts, delta states) |
| `v2-queue` | Work queue with throttle context tracking |

#### Imports from fsbedrock

```python
# From dispatcher.py / dispatcherv2.py
from fsbedrock.persistence.dynamodb.utils import (
    aget_model_from_table,
    atransactional_write_model_to_table,
    awrite_model_to_table,
)
from fsbedrock.persistence.dynamodb.serializers import (
    ResourceSerializer,
    ArtifactSerializer,
    DeltaStateSerializer,
)
from fsbedrock.domain.backup.mappers import (
    ResourceDynamoDBModelMapper,
    ArtifactDynamoDBModelMapper,
    DeltaStateDynamoDBModelMapper,
)
from fsbedrock.persistence.dynamodb.session import create_session
```

#### Data Flow

**Initialization** (`manager.py:62-69`):
```python
self._dynamodb_client = await self._session.client("dynamodb", region_name="us-east-1")
self._dynamodb_resource = await self._session.resource("dynamodb", region_name="us-east-1")
self._dynamodb_table = await self._dynamodb_resource.Table("resources")
self._queue_table = await self._dynamodb_resource.Table("v2-queue")
```

**Resource Persistence** (`dispatcherv2.py:48-49`):
```python
async def _save_resource(self, resource: Resource):
    await atransactional_write_model_to_table(
        self._dynamodb_table, ResourceSerializer, ResourceDynamoDBModelMapper(), resource
    )
```

**DeltaState Persistence** (`dispatcherv2.py:56-57`):
```python
async def _save_delta_state(self, delta_state: DeltaState):
    await awrite_model_to_table(
        self._dynamodb_table, DeltaStateSerializer, DeltaStateDynamoDBModelMapper(), delta_state
    )
```

**DeltaState Retrieval** (`dispatcherv2.py:59-67`):
```python
async def _retrieve_delta_state(self, resource: Resource):
    key = {
        "PK": f"TENANT#{resource.tenant_id}#RES#{resource.id}",
        "SK": f"DELTA"
    }
    item = await aget_model_from_table(
        self._dynamodb_table, DeltaStateSerializer, DeltaStateDynamoDBModelMapper(), key
    )
    return item
```

#### Work Queue Schema (`queue.py`)

**Work Items**:
- pk: `TENANT#{tenant_id}#QUEUE#RES`
- sk: SHA-256 hash of `tenant_id:operation_type:cloud_external_id`
- Attributes: `created_time`, `data`, `operation_type`, `request_policy_type`, `backup_id`, `target_id`, `next_visible_time`, `throttle_context_id`

**Throttle Context Singleton**:
- pk: `THROTTLE_CONTEXT#{throttle_context_id}`
- sk: `METADATA`
- Attributes: `ref_count`, `fencing_token`

**GSI**: `throttle-context-visible-index` on `throttle_context_id` (PK) and `next_visible_time` (SK)

### 3. Migrated Discover Current State

**Location**: `bases/filescience/discover/`

#### DynamoDB Persistence Status: STUBBED

All DynamoDB persistence operations are currently stubbed in `dispatcher.py:12-38`:

```python
# TODO: Migrate DynamoDB persistence utilities when ready
# These will be imported from filescience.dynamodb:
# - aget_model_from_table
# - atransactional_write_model_to_table
# - awrite_model_to_table
# - ResourceSerializer, ArtifactSerializer, DeltaStateSerializer
# - ResourceDynamoDBModelMapper, ArtifactDynamoDBModelMapper, DeltaStateDynamoDBModelMapper

async def _stub_write_resource(resource):
    """Stub: Write resource to DynamoDB"""
    pass

async def _stub_write_artifact(artifact):
    """Stub: Write artifact to DynamoDB"""
    pass

async def _stub_write_delta_state(delta_state):
    """Stub: Write delta state to DynamoDB"""
    pass

async def _stub_get_delta_state(resource):
    """Stub: Get delta state from DynamoDB"""
    return None
```

#### What IS Implemented

1. **Session Management** (`session.py`) - Optimized aioboto3 session (copied from fsbedrock)
2. **WorkQueue** (`queue.py`) - Full DynamoDB queue operations using boto3 directly
3. **Manager** (`manager.py`) - DynamoDB client/resource/table initialization
4. **Cloud Router** (`clouds/router.py`) - Backup state queries using boto3 directly

#### What IS NOT Implemented (Needs Migration)

1. **High-level utilities**: `aget_model_from_table`, `atransactional_write_model_to_table`, `awrite_model_to_table`
2. **Serializers**: `ResourceSerializer`, `ArtifactSerializer`, `DeltaStateSerializer`
3. **Mappers**: `ResourceDynamoDBModelMapper`, `ArtifactDynamoDBModelMapper`, `DeltaStateDynamoDBModelMapper`
4. **Persistence models**: `NodeIdentityItem`, `NodeLinkItem`, `DeltaStateItem`, `EntityItem`

### 4. Existing DynamoDB in Monorepo

#### Placeholder Component
- `components/filescience/dynamodb/__init__.py` - Empty placeholder

#### Valkey Queue Processor Base
The `bases/filescience/valkey_queue_processor/` already has DynamoDB code but uses **direct boto3 calls**, not the fsbedrock abstraction layer:

- `clients/dynamodb_client.py` - Singleton client/resource pattern
- `services/work_registration_service.py` - TransactWriteItems for work registration
- `services/pruning_service.py` - Conditional deletes for throttle context pruning
- `models/schema.py` - Pydantic models for DynamoDB stream records

#### Infrastructure
- `infrastructure/modules/dynamodb/` - Terraform module for table creation
- `infrastructure/live/non-prod/us-east-1/dev/dynamodb/` - Dev environment config for `work-queue` table

## Code References

### fsbedrock (Source)
- `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/persistence/dynamodb/utils.py` - High-level CRUD utilities
- `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/persistence/dynamodb/serializers.py` - Serializer implementations
- `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/persistence/dynamodb/models.py` - Persistence models
- `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/domain/backup/mappers.py` - Domain model mappers
- `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/persistence/dynamodb/session.py` - Session management

### Old Discover (Usage Reference)
- `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/dispatcherv2.py` - Shows how utilities are used
- `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/queue.py` - Work queue implementation
- `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/manager.py` - Initialization pattern

### Migrated Discover (Target)
- `bases/filescience/discover/dispatcher.py:12-38` - Stubs that need implementation
- `bases/filescience/discover/session.py` - Already migrated session utilities
- `bases/filescience/discover/queue.py` - Already migrated queue (uses direct boto3)

## Architecture Documentation

### Single Table Design Pattern
Both `resources` and queue tables use single-table design with composite keys:
- Partition key encodes entity type and tenant: `TENANT#{tenant_id}#TYPE#{id}`
- Sort key encodes sub-type or relationship: `IDENTITY`, `LINK#{timestamp}`, `DELTA`

### Multi-Row Node Pattern
Resources and Artifacts use two DynamoDB items per domain object:
1. **IDENTITY row**: Cloud metadata (cloud_external_id, cloud_id, service_id)
2. **LINK row**: Relationship and temporal data (parent_ref, created_at, name)

This is handled by `PydanticNodeSerializer` which aggregates/splits during serialization.

### Mapper-Serializer-Utility Pattern
Three-layer abstraction:
1. **Domain Models** (dataclasses) - Business logic
2. **Persistence Models** (Pydantic) - Match DynamoDB structure
3. **DynamoDB Items** (dict) - Raw format

Utilities compose all layers for CRUD operations.

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/durable/01-active/next_up.md:26` - Notes "DynamoDB persistence utilities stubbed in discover dispatcher (Phase 4 handoff)"
- `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md` - Parent migration plan
- `memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md` - Related dependency analysis

## Related Research

- `memory-bank/thoughts/shared/research/2026-01-29-opentofu-dynamodb-elasticache.md` - DynamoDB infrastructure setup
- `memory-bank/thoughts/shared/research/2026-01-30-components-filescience-usage-report.md` - Component usage patterns

## Migration Checklist

### Files to Migrate from fsbedrock

1. **Core Utilities** → `components/filescience/dynamodb/`
   - [ ] `_types.py` - Type definitions
   - [ ] `session.py` - Already in discover, consolidate to component
   - [ ] `read/get.py` - Get operations
   - [ ] `read/query.py` - Query operations (include BiDirectionalQueryExecutor)
   - [ ] `read/scan.py` - Scan operations (include ParallelScanExecutor)
   - [ ] `write/put.py` - Put operations
   - [ ] `write/transaction.py` - Transaction operations
   - [ ] `delete.py` - Delete operations
   - [ ] `serializers.py` - PydanticSerializer, PydanticNodeSerializer
   - [ ] `models.py` - Persistence models
   - [ ] `utils.py` - High-level CRUD utilities

2. **Domain Mappers** → `components/filescience/domain_backup/mappers.py` (or similar)
   - [ ] `EntityDynamoDBModelMapper`
   - [ ] `ResourceDynamoDBModelMapper`
   - [ ] `ArtifactDynamoDBModelMapper`
   - [ ] `DeltaStateDynamoDBModelMapper`

3. **Update Discover** → `bases/filescience/discover/dispatcher.py`
   - [ ] Replace stubs with actual imports from `filescience.dynamodb`
   - [ ] Wire up persistence calls

### Dependencies to Add

```toml
# In components/filescience/dynamodb/pyproject.toml
dependencies = [
    "boto3>=1.40.0",
    "aioboto3>=15.5.0",
    "aiobotocore>=2.23.0",
    "types-boto3>=1.40.0",
    "types-aiobotocore>=2.23.0",
    "pydantic>=2.12.4",
    "anyio>=4.11.0",
]
```

## Open Questions

1. **Component vs Base**: Should DynamoDB utilities be a `component` (shared library) or stay embedded in bases?
2. **Async-Only**: Should we migrate both sync and async versions, or just async?
3. **BiDirectionalQueryExecutor**: Is this optimization needed in the new system?
4. **ParallelScanExecutor**: Is parallel scan functionality required?
5. **Domain Mappers Location**: Should mappers live with `domain_backup` component or with `dynamodb` component?
