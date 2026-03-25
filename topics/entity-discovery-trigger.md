---
topic: Entity Discovery Trigger
status: active
touched: 2026-03-04
related:
  - thoughts/shared/plans/2026-02-24-ENG-2133-environment-strategy-deployment-pipeline.md
---

# Entity Discovery Trigger

## Current State
Code complete (Phases 1-4 done), deployed to dev. Syncs users, groups, and sites from Microsoft Graph API to DynamoDB `resources` table.

Handler uses OTel (`setup_telemetry`, `capture_method` from `filescience.telemetry`), Powertools Logger, `SessionManager` from `cloud_api` component. Event receives `{CloudId, TenantId, OrgId}`, fetches remote entities, diffs against DynamoDB, saves new/reactivated, marks removed as inactive.

Historical blocker (resolved): Migration was blocked on `cloud_api` component extraction — importing from discover pulled in throttling/Valkey deps. Resolved by extracting `cloud_api` as a standalone component.

## Key Decisions
- Uses OTel instrumentation (not X-Ray/Powertools Tracer) — on the new telemetry stack
- Depends on `cloud_api` and `models` components only

## Open Questions
- None currently — service is stable and deployed

## Artifacts
- `bases/filescience/entity_discovery_trigger/` — full base
- `infrastructure/live/non-prod/us-east-1/dev/entity-discovery-trigger/` — Terragrunt config
