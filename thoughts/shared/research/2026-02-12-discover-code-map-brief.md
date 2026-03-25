---
date: 2026-02-12T14:31:22Z
source_research: memory-bank/thoughts/shared/research/2026-02-12-discover-code-map.md
last_generated: 2026-02-12T14:32:35.618677+00:00
---

# Research Brief: 2026-02-12-discover-code-map

## TL;DR

`discover` is an async traversal pipeline that:
- starts from tenant root `Entity` nodes,
- routes work by cloud/service (`CloudRouter`/`ServiceRouter`),
- emits `Resource`/`Artifact`/`DeltaState` nodes from service handlers (OneDrive + Outlook),
- enqueues traversable `Resource` nodes into DynamoDB work queue with throttle metadata,
- leases and executes queued work via Valkey-backed admission control and throttle context selection.

Primary orchestration is in `DiscoveryManager` and `DispatcherV2`, with queue semantics in `WorkQueue`, and service-specific traversal in `bases/filescience/discover/clouds/microsoft/services/`.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- None for the requested mapping scope.

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-12-discover-code-map.md`
