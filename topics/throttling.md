---
topic: Throttling
status: active
touched: 2026-03-04
related:
  - thoughts/shared/research/2026-01-30-throttle-service-migration.md
  - thoughts/shared/plans/2026-01-30-throttle-service-migration.md
  - thoughts/shared/plans/2026-02-02-throttle-gates-migration.md
  - thoughts/shared/research/2026-02-10-outlook-graph-throttling-compatibility.md
---

# Throttling

## Current State
Fully migrated to `components/filescience/throttling/`. Sync and async service classes, Valkey-backed functions (including Lua scripts), M365/OneDrive policies, models, and client factories are all in place. Component is a pure leaf — no component dependencies (enforced by import-linter).

Uses `GlideClient` in standalone mode (not cluster) due to known GLIDE cluster connection bug. `AsyncThrottleService` for async paths, `ThrottleService` for sync.

## Key Decisions
- 2026-01: Extracted throttling from discover into standalone component
- 2026-02: Gates migration completed (throttle gates moved to component)
- Standalone GLIDE mode is a permanent workaround for cluster connection issues

## Open Questions
- Lua throttle script not yet deployed to dev Valkey (backlog item)
- `glide` vs `glide_sync` client type mismatch between VQP handler and throttling component — known sharp edge, unresolved

## Artifacts
- `components/filescience/throttling/` — full component
- `components/filescience/throttling/valkey/` — Valkey functions and client
- `components/filescience/throttling/policies/m365/` — M365 throttle policies
