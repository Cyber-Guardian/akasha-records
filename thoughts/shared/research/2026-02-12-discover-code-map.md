---
date: 2026-02-12T14:31:22Z
researcher: Codex (Cursor)
git_commit: 350e143
branch: main
repository: filescience
topic: "Code map of discover"
tags: [research, codebase, discover, dispatcher, manager, queue, microsoft-graph]
status: complete
last_updated: 2026-02-12
last_updated_by: Codex (Cursor)
---

# Research: Code map of discover

**Date**: 2026-02-12T14:31:22Z
**Researcher**: Codex (Cursor)
**Git Commit**: 350e143
**Branch**: main
**Repository**: filescience

## Research Question

Generate a code map of `discover`.

## Summary

`discover` is an async traversal pipeline that:
- starts from tenant root `Entity` nodes,
- routes work by cloud/service (`CloudRouter`/`ServiceRouter`),
- emits `Resource`/`Artifact`/`DeltaState` nodes from service handlers (OneDrive + Outlook),
- enqueues traversable `Resource` nodes into DynamoDB work queue with throttle metadata,
- leases and executes queued work via Valkey-backed admission control and throttle context selection.

Primary orchestration is in `DiscoveryManager` and `DispatcherV2`, with queue semantics in `WorkQueue`, and service-specific traversal in `bases/filescience/discover/clouds/microsoft/services/`.

## Code Map

### 1) Public surface and entrypoints

- `bases/filescience/discover/__init__.py`
  - Re-exports `DiscoveryManager`, `DispatcherV2`, `WorkQueue`, `WorkItem`.
- `projects/discover/sandbox.py`
  - Local runnable entrypoint that loads tenant entities, dispatches them in parallel, waits for throttle tree readiness, then runs queue processing loop.

### 2) Runtime orchestration

- `bases/filescience/discover/manager.py`
  - `DiscoveryManager`: async context manager owning long-lived DynamoDB + Valkey/throttle clients.
  - Wires `WorkQueue` + `DispatcherV2`.
  - `run()`: main lease/process loop with task group, maintenance accounting, and idle-stop logic.
  - `wait_for_tree_ready()`: polls throttle context tree availability before processing.
  - `RunMetrics`: shared mutable counters for status/stop logging.

### 3) Dispatch and persistence boundary

- `bases/filescience/discover/dispatcher.py`
  - `DispatcherV2.dispatch_entity()`: explodes a root `Entity` into per-service `ServiceEntity`.
  - `_dispatch_task()`: gets API session, selects cloud router, executes handler stream.
  - `_handle_resource_item()`: saves `Resource`, enqueues traversal work item with service-specific `request_policy_type`, optionally hydrates delta state.
  - `traverse_resource()`: executes traversal for queued work then calls queue `finish_work()`.
  - Service policy map currently includes:
    - OneDrive -> `onedrive_traversal`
    - Outlook -> `outlook_traversal`

### 4) Queue, leasing, and throttle integration

- `bases/filescience/discover/queue.py`
  - `enqueue()`: idempotent put into DynamoDB queue table (`pk/sk` deterministic by tenant + operation + cloud external id).
  - `lease_work()`: obtains optimal throttle context from Valkey, queries visible queue work, leases an item, reconstructs `Resource`, optionally performs admission check and returns `WorkItem`.
  - Empty-context maintenance paths:
    - prune throttle context when ref-count indicates no in-flight work,
    - penalize context when work is still in-flight but no visible item.
  - `finish_work()`: DynamoDB transaction to tombstone queue item + decrement singleton ref-count; then release admission token.

### 5) Cloud/service routing model

- `bases/filescience/discover/clouds/router.py`
  - `ServiceRouter`: handler registry keyed by `(node_type, on_initial_backup)`.
  - `CloudRouter`: maps service -> `ServiceRouter`, computes backup mode (`initial` vs `delta`) from latest backup row, builds execution `Context`, streams emitted nodes.
  - `Context`: cloud/service/tenant/session + optional `work_id` and metrics.
- `bases/filescience/discover/clouds/models.py`
  - `ServiceEntity`: extends `Entity` with explicit `service`.

### 6) Microsoft cloud composition

- `bases/filescience/discover/clouds/microsoft/cloud.py`
  - Defines `O365Router` and mounts:
    - `onedrive.router`
    - `outlook.router`

### 7) OneDrive traversal handlers

- `bases/filescience/discover/clouds/microsoft/services/onedrive.py`
  - `user` -> drives
  - `drive` (initial/delta paths) -> folders + files + delta token (`DeltaState`)
  - `folder` -> subfolders + files
  - `file` (initial) -> artifact versions
  - Uses `GraphClient(...).onedrive.*` iterators and emits `Resource`/`Artifact`/`ArtifactVersion`/`DeltaState`.

### 8) Outlook traversal handlers

- `bases/filescience/discover/clouds/microsoft/services/outlook.py`
  - `user` -> top-level mail folders + folder delta state
  - `mail_folder` -> child folders
  - `mail_folder` (initial/delta) -> messages as `Artifact` plus updated delta state
  - Message artifact strategy:
    - if attachments or inline CID images: `application/eml`, `/$value` download URL,
    - otherwise: `application/json` message URL.

### 9) Session abstraction

- `bases/filescience/discover/session.py`
  - Thin re-export of DynamoDB component session helpers (`create_session`, config, optimized session class).

### 10) Tests that document behavior

- `test/bases/filescience/discover/test_dispatcher.py`
  - Service-to-policy map behavior, queue policy assignment, and per-item handling expectations.
- `test/bases/filescience/discover/clouds/microsoft/services/test_outlook.py`
  - Outlook handler registration, folder/message traversal outputs, delta token extraction, EML/JSON message artifact behavior.

## End-to-end flow (current implementation)

1. `projects/discover/sandbox.py` loads tenant entities and calls `DiscoveryManager.dispatch(entity)`.
2. `DispatcherV2.dispatch_entity()` fans out one entity into per-service `ServiceEntity` tasks.
3. `_dispatch_task()` calls `O365Router.execute(...)`.
4. `CloudRouter` chooses initial-vs-delta mode and delegates to mounted `ServiceRouter`.
5. Service handler yields nodes (`Resource`, `Artifact`, `DeltaState`, `ArtifactVersion`).
6. `DispatcherV2` saves/enqueues traversable `Resource` nodes via `WorkQueue.enqueue(...)`.
7. `DiscoveryManager.run()` loops on `WorkQueue.lease_work(...)`.
8. Leased `WorkItem` is traversed by `DispatcherV2.traverse_resource(...)`.
9. `WorkQueue.finish_work(...)` tombstones queue row, decrements context ref-count, and releases admission token.

## Code References

- `projects/discover/sandbox.py:61-103` - local execution flow (collect entities, dispatch, run manager)
- `bases/filescience/discover/manager.py:66-150` - manager setup and dispatch wiring
- `bases/filescience/discover/manager.py:252-281` - main queue-processing loop
- `bases/filescience/discover/dispatcher.py:41-73` - entity fanout into service entities
- `bases/filescience/discover/dispatcher.py:117-150` - resource save/enqueue path
- `bases/filescience/discover/dispatcher.py:151-175` - router execution and item-type branching
- `bases/filescience/discover/queue.py:223-305` - throttle-context selection and maintenance when no work
- `bases/filescience/discover/queue.py:307-411` - leasing visible work item and return `WorkItem`
- `bases/filescience/discover/queue.py:413-550` - finish-work transaction + retry behavior
- `bases/filescience/discover/clouds/router.py:33-59` - service router registration/execution
- `bases/filescience/discover/clouds/router.py:94-113` - cloud router context + execute path
- `bases/filescience/discover/clouds/microsoft/cloud.py:7-10` - OneDrive/Outlook router mount
- `bases/filescience/discover/clouds/microsoft/services/onedrive.py:15-211` - OneDrive handlers
- `bases/filescience/discover/clouds/microsoft/services/outlook.py:15-203` - Outlook handlers
- `test/bases/filescience/discover/test_dispatcher.py:16-223` - dispatcher behavior tests
- `test/bases/filescience/discover/clouds/microsoft/services/test_outlook.py:145-618` - Outlook handler tests

## Architecture Documentation

- Pattern: `Entity` seeds -> service-specific traversal -> queue-backed recursive traversal.
- Routing model:
  - cloud-level router (`CloudRouter`) chooses service router,
  - service-level router (`ServiceRouter`) chooses handlers by node type + backup mode.
- Queue model:
  - DynamoDB table stores queued work and throttle context singletons,
  - Valkey functions select/score contexts and enforce admission constraints,
  - completion is transactional with tombstoning + ref-count decrement.
- Data model usage:
  - `Resource` used for traversable containers,
  - `Artifact` used for leaf or content objects,
  - `DeltaState` used for incremental checkpointing.

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/thoughts/shared/plans/2026-02-10-outlook-discover-integration.md`
  - documents Outlook discovery rollout (folder + message handlers, service-aware policy mapping).
- `memory-bank/thoughts/shared/research/2026-02-10-outlook-graph-throttling-compatibility.md`
  - describes Outlook throttling compatibility with existing discover queue/throttle architecture.

## Related Research

- `memory-bank/thoughts/shared/research/2026-02-10-outlook-graph-throttling-compatibility.md`
- `memory-bank/thoughts/shared/research/2026-02-05-outlook-graph-concurrency-gates.md`

## Open Questions

- None for the requested mapping scope.
