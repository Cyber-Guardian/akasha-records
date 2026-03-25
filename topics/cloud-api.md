---
topic: Cloud API
status: active
touched: 2026-03-04
related: []
---

# Cloud API

## Current State
Fully extracted as `components/filescience/cloud_api/`. This extraction unblocked entity-discovery-trigger from the discover base dependency chain.

Contents:
- `client.py` — Microsoft GraphClient (generic + Microsoft-specific)
- `session.py` — SessionManager for OAuth2 token management
- `oauth.py` — OAuth2 flows
- `aws.py` — AWS credential helpers
- `suppression.py` — API request suppression/retry logic
- `microsoft/` — Microsoft-specific API sub-components

Component depends on `models` only (enforced by import-linter). No discover, throttling, or DynamoDB dependencies.

## Key Decisions
- Extracted from discover to break circular dependency chain
- Pure leaf component — depends only on `models`
- GraphClient handles both generic HTTP and Microsoft Graph specifics

## Open Questions
- None currently — component is stable

## Artifacts
- `components/filescience/cloud_api/` — full component
