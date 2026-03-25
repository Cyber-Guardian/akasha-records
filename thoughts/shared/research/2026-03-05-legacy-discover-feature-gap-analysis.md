---
date: 2026-03-05T12:00:00-05:00
researcher: Claude
git_commit: 6042b27
branch: main
repository: filescience
topic: "Legacy discover service feature gap analysis"
tags: [research, codebase, discover, migration, cloud-providers]
status: complete
last_updated: 2026-03-05
last_updated_by: Claude
---

# Research: Legacy Discover Feature Gap Analysis

**Date**: 2026-03-05
**Git Commit**: 6042b27
**Branch**: main

## Research Question

What features exist in the legacy discover service (`filescience-discover/src`) that are missing from the current monorepo discover base (`bases/filescience/discover/`)?

## Summary

The current monorepo discover service supports **2 of 7 Microsoft services** (OneDrive, Outlook) and **0 of 6 non-Microsoft cloud providers** that exist in the legacy. Beyond cloud coverage, there are significant architectural differences: the legacy uses multi-threaded synchronous processing with SQS output, while the current uses async processing with a DynamoDB work queue and Valkey throttle tree. Several legacy internal features (version backfill, stall detection, Step Functions integration, org stats) have no equivalent in the current codebase.

## Feature Gap Matrix

### Cloud Providers

| Provider | Legacy | Current | Status |
|----------|--------|---------|--------|
| Microsoft 365 | 7 services | 2 services | Partial |
| Clio | 2 services | 0 | Missing (supported cloud) |
| Box | 1 service | 0 | Missing (supported cloud) |
| Google Workspace | 5 services | 0 | Missing (not supported) |
| Zoom | 1 service | 0 | Missing (not supported) |
| Smartsheet | 3 services | 0 | Missing (not supported) |
| NetDocs | 1 service | 0 | Missing (not supported) |
| QuickBooks | 0 (commented out) | 0 | N/A |

### Microsoft 365 Services

| Service | Legacy | Current | Gap Details |
|---------|--------|---------|-------------|
| OneDrive | Full (drives, files, versions, delta) | Full (drives, folders, files, versions, delta) | Parity |
| Outlook | Full (folders, hidden folders, recoverable items, delta, EML/JSON) | Full (folders, hidden folders, recoverable items, delta, EML/JSON) | Parity |
| Teams | Full (teams, channels, chats, messages, replies) | Models defined, graph client built | No router handlers |
| Calendar | Full (calendars, events, attachments, beta delta API) | Models defined, graph client built | No router handlers |
| Contacts | Full (folders, contacts, delta) | Models defined, graph client built | No router handlers |
| Todo | Full (lists, tasks, attachments, malformed JSON parser) | Models defined, graph client built | No router handlers |
| Planner | Full (plans, buckets, tasks, dedup) | Models defined, graph client built | No router handlers |

**Note**: The current codebase has Pydantic models and `GraphClient` service components for ALL 7 Microsoft services (`cloud_api/microsoft/client.py` and `cloud_api/microsoft/models.py`). Only the `ServiceRouter` handlers in `bases/filescience/discover/clouds/microsoft/services/` are missing for Teams, Calendar, Contacts, Todo, and Planner.

### Clio (Supported Cloud - Missing Entirely)

Legacy services:
- **ClioContacts** (`CLIO_CONTACTS`) -- discovers contacts, fetches matters per contact with full field expansion, yields matter + contact detail artifacts as inlined JSON
- **ClioDocuments** (`CLIO_DOCUMENTS`) -- recursive folder tree walk, document metadata artifacts, document version artifacts with download URLs

Current state: No Clio cloud_api client, no OAuth, no models, no router handlers.

### Box (Supported Cloud - Missing Entirely)

Legacy service:
- **Storage** (`BX_CORE`) -- BFS folder tree from root, file artifacts, version backfill via 5 background workers, user impersonation via `As-User` header

Current state: `BoxOAuth2TokenAuth` exists in `cloud_api/oauth.py` but nothing else -- no Box API client, no models, no router handlers.

### Google Workspace (Not a Supported Cloud)

Legacy services:
- **Gdrive** (`GS_GDRIVE`) -- all Drive files, Google Workspace export (Docs->ODT, Sheets->ODS, Slides->ODP)
- **Gmail** (`GS_GMAIL`) -- all messages including spam/trash
- **Calendar** (`GS_CALENDAR`) -- calendar list, events
- **Contacts** (`GS_CONTACTS`) -- People API connections
- **Photos** (`GS_PHOTOS`) -- media items with estimated sizes

### Zoom (Not a Supported Cloud)

Legacy service:
- **Meetings** (`ZM_MEETINGS`) -- 30-day window iteration back to 2011, meeting recordings

### Smartsheet (Not a Supported Cloud)

Legacy services:
- **Contacts** (`ST_CONTACTS`), **Discussions** (`ST_DISCUSSIONS`), **Sheets** (`ST_SHEETS`)
- Folder tree walk, sheet download as Excel, embedded discussions, user impersonation via `Assume-User` header

### NetDocs (Not a Supported Cloud)

Legacy service:
- **Documents** (`ND_CORE`) -- workspaces, collabspaces, recursive folder tree, document versions, approvals, links, attachments, 40 concurrent consumer threads

## Internal / Infrastructure Gaps

### 1. Version Backfill (Missing)

**Legacy**: Both OneDrive and Box run 5 background `ThreadPoolExecutor` workers that fetch file version history. OneDrive triggers for files >= 5MB. Each older version is yielded as a separate `ArtifactVersion` with a version-specific download URL.

**Current**: The OneDrive router handler (`file_versions_from_file`) yields `ArtifactVersion` objects, but `DispatcherV2._dispatch_task` has a `continue` statement that **skips** `ArtifactVersion` items (`dispatcher.py`). Version discovery is wired but not persisted.

### 2. SQS Transfer Queue (Architecturally Different)

**Legacy**: `TransferQueue` manages dual-lane SQS output -- "express" queue (pre-resolved artifacts with inlined data) and "standard" queue (artifacts needing download). 20 background workers batch artifacts up to SQS message size limits, with 3-retry backoff on send failures. X-Ray trace propagation into SQS messages.

**Current**: No SQS output. Artifacts are saved directly to DynamoDB via transactional writes in `DispatcherV2._save_artifact`. The downstream pipeline (VQP) reads from DynamoDB.

### 3. Stall Detection (Missing)

**Legacy**: `DiscoverManager._check_stalled` cross-references three signals: artifact size progress, resource heartbeat (`ResourceStore.check_resource_heartbeat`), and active retry count (`DiscoverAPI.retried_requests`). Declares stall after 600s of no progress across all three.

**Current**: No equivalent. The `run()` loop has a 10s idle timeout (no work and no in-flight tasks) but no multi-signal stall detection for stuck API calls.

### 4. Step Functions Integration (Missing)

**Legacy**: `DiscoverManager` accepts an `sf_task_token`, sends heartbeats every 60 ticks, and reports `send_task_success`/`send_task_failure` on completion.

**Current**: No Step Functions integration. The manager is invoked directly as a Lambda handler.

### 5. Org Stats / Usage Reports (Missing)

**Legacy**: Microsoft provider calls Graph Reports API (`getMailboxUsageDetail`, `getOneDriveUsageAccountDetail`) to get per-user count/size estimates before discovery starts. Google calls Admin Reports API. These stats feed into `Entity.total_count`/`total_size` for progress reporting.

**Current**: No usage stats pre-fetch. Entity count/size tracking does not exist.

### 6. Concurrent Processing Model (Architecturally Different)

**Legacy**: `ConcurrentProcessor` runs N workers (default 15) on a `ThreadPoolExecutor`. `DiscoverQueue` uses a `PriorityQueue` with per-service spinlocks for fair scheduling across entities/services. `halt_all` flag for coordinated shutdown.

**Current**: Single async task group in `DiscoveryManager.run()`. Work distribution is handled by the DynamoDB work queue + Valkey throttle tree, which provides fairness and rate limiting at the infrastructure level rather than in-process.

### 7. Artifact Store / Dedup Cache (Missing)

**Legacy**: `ArtifactStoreCache` bulk-loads existing artifacts from DynamoDB at startup. `ArtifactStore.get_artifact_entry` enables dedup during discovery. `ResyncFilter` uses the artifact store to skip already-known artifacts during delta resync.

**Current**: No in-memory artifact cache. Artifact dedup is handled by the DynamoDB conditional writes (idempotent enqueue via `ConditionExpression=Attr("pk").not_exists()`).

### 8. Resource Store / Resource Tracking (Architecturally Different)

**Legacy**: `ResourceStoreCache` bulk-loads resources from DynamoDB. `ResourceStore` maintains in-memory entries with heartbeat counters for stall detection. `_get_resource` creates or updates resources with parent change detection.

**Current**: Resources are saved via transactional DynamoDB writes in `DispatcherV2._save_resource`. No in-memory cache or heartbeat tracking.

### 9. Malformed JSON Parser (Missing)

**Legacy**: `response_parsers.malformed_json` -- bracket-balancing repair for truncated/broken JSON responses. Used by the Todo service specifically.

**Current**: No equivalent. All responses parsed via `orjson.loads` in `ResponseIterator`.

### 10. Provider/Service Registry (Architecturally Different)

**Legacy**: Decorator-based `ProviderRegistry` and `ServiceRegistry` with `BaseSingleton` pattern. `@ProviderRegistry.register(Cloud.X)` auto-registers providers at import time.

**Current**: Router-based. `CloudRouter` holds `_service_routers`, `ServiceRouter` uses `@router.register(node_type)` decorator for handler functions. Mount pattern: `O365Router.mount_service_router(onedrive.router)`.

## Architectural Comparison

| Aspect | Legacy | Current |
|--------|--------|---------|
| Concurrency | ThreadPoolExecutor (15 workers) | asyncio/anyio task groups |
| Work distribution | In-memory PriorityQueue + spinlocks | DynamoDB work queue + Valkey throttle tree |
| Output pipeline | SQS (dual-lane: express + standard) | Direct DynamoDB writes |
| HTTP client | requests (synchronous) | httpx (async) with RetryTransport |
| State caching | In-memory singleton caches (bulk hydrate at startup) | No in-memory caching |
| API retry | Custom retry with decorrelated jitter (5 retries, 10s base, 3min max) | httpx RetryTransport (3 retries, 0.5s backoff) |
| Delta tracking | DeltaStoreCache singleton, DynamoDB-backed | DeltaState objects yielded from handlers, saved by dispatcher |
| Auth | Per-provider API classes (GraphAPI, BoxAPI, etc.) | Shared OAuth2TokenAuth with httpx auth flow |
| Tracing | X-Ray (propagated into SQS messages) | Not visible in discover base |
| Monitoring | Stall detection, progress logging, Step Functions heartbeat | 10s idle timeout, structured logging |

## Code References

### Legacy (filescience-discover/src/filescience/)
- `internal/abstract.py` -- BaseProvider, BaseService (generator-based discovery)
- `internal/api.py` -- DiscoverAPI with retry/backoff, per-cloud subclasses
- `internal/artifacts.py` -- ArtifactStore/Cache for dedup
- `internal/concurrent_processor.py` -- ThreadPoolExecutor worker
- `internal/delta.py` -- DeltaStore/Cache for change tokens
- `internal/discover_queue.py` -- PriorityQueue + spinlock fair scheduling
- `internal/manager.py` -- DiscoverManager orchestrator (stall detection, SF heartbeat)
- `internal/transfer_queue.py` -- Dual-lane SQS pipeline
- `internal/resources.py` -- ResourceStore/Cache with heartbeat
- `clouds/clio/` -- Clio provider (contacts, documents)
- `clouds/box/` -- Box provider (storage with version backfill)
- `clouds/microsoft/` -- M365 provider (7 services)
- `clouds/google/` -- Google provider (5 services)
- `clouds/zoom/` -- Zoom provider (meetings/recordings)
- `clouds/smartsheet/` -- Smartsheet provider (contacts, discussions, sheets)
- `clouds/netdocs/` -- NetDocs provider (documents, versions, approvals)

### Current (bases/filescience/discover/)
- `manager.py` -- DiscoveryManager (async, Valkey throttle)
- `dispatcher.py` -- DispatcherV2 (router dispatch, DynamoDB writes)
- `queue.py` -- WorkQueue (DynamoDB + Valkey throttle tree)
- `clouds/router.py` -- ServiceRouter/CloudRouter handler registry
- `clouds/microsoft/services/onedrive.py` -- OneDrive handlers
- `clouds/microsoft/services/outlook.py` -- Outlook handlers

### Shared Components (components/filescience/)
- `cloud_api/microsoft/client.py` -- GraphClient with ALL 7 service components
- `cloud_api/microsoft/models.py` -- Pydantic models for ALL 7 services
- `cloud_api/oauth.py` -- OAuth for Microsoft, Box, Clio, NetDocs
- `models/core.py` -- Cloud/CloudService enum definitions (ALL services defined)

## Open Questions

1. **Prioritization**: Clio and Box are listed as supported clouds in identity.md. What's the timeline for adding their discover handlers?
2. **ArtifactVersion handling**: The current OneDrive handler yields versions, but the dispatcher skips them. Is this intentional (deferred) or a bug?
3. **Missing M365 services**: Models and graph client methods exist for Teams, Calendar, Contacts, Todo, and Planner. Are router handlers planned?
4. **SQS vs DynamoDB**: Was the shift from SQS output to direct DynamoDB writes an intentional architectural decision, or is SQS integration planned?
5. **Stall detection**: Is the 10s idle timeout sufficient, or does the Valkey throttle tree provide equivalent protection?
