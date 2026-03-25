---
topic: Discover Service
status: active
touched: 2026-03-04
related:
  - thoughts/shared/research/2026-02-12-discover-code-map.md
  - thoughts/shared/plans/2026-02-10-outlook-discover-integration.md
---

# Discover Service

## Current State
Fully operational async traversal pipeline for Microsoft 365 (OneDrive + Outlook). Entry point: `projects/discover/sandbox.py`.

Architecture: `DiscoveryManager` + `DispatcherV2` for orchestration. `WorkQueue` backed by DynamoDB (queue table) + Valkey (admission control, throttle context selection via Lua). `CloudRouter` / `ServiceRouter` handler registry routes by node type + backup mode.

OneDrive: users -> drives -> folders/files -> artifact versions + delta tokens.
Outlook: users -> mail folders -> messages (EML or JSON depending on attachments).

## Key Decisions
- DispatcherV2 replaced original dispatcher for better async orchestration
- CloudRouter/ServiceRouter pattern for extensible cloud support
- Delta tokens for incremental sync (OneDrive)

## Open Questions
- 36 poison DynamoDB stream records with `ENQUEUED_WORK_ITEM` shape missing required fields — causes silent `batchItemFailures` (active blocker)
- "Discover Debug Test Runs (Phase 3)" backlogged — handle poison records first, then test runs

## Artifacts
- `bases/filescience/discover/` — full base
- `projects/discover/sandbox.py` — entry point
- `research/2026-02-12-discover-code-map.md` — authoritative code map
