---
date: 2026-02-03T17:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-03-api-client-component-extraction.md
last_generated: 2026-02-03T20:17:16.594414+00:00
---

# Research Brief: 2026-02-03-api-client-component-extraction

## TL;DR

The API client code is currently located inside `bases/filescience/discover/clouds/` and consists of two layers:
1. **Generic API client infrastructure** (`clouds/api/`) - 5 files with no cloud-specific dependencies
2. **Microsoft Graph client** (`clouds/microsoft/api/`) - 3 files implementing Microsoft-specific logic

These files are candidates for extraction into `components/filescience/cloud_api/` to break the dependency chain that currently prevents `entity-discovery-trigger` from importing just the API client without pulling in the entire discover base (which requires throttling, Valkey, DynamoDB).

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **AWS dependency in session.py** - Should `aws.py` move with the API client, or should credential injection be refactored?
2. **Other cloud providers** - Only Microsoft is currently implemented. Box, Clio, NetDocs OAuth exists but no full clients. Plan for future cloud clients?
3. **Testing** - Tests exist in `test/bases/filescience/discover/clouds/microsoft/api/test_site.py`. Move to `test/components/filescience/cloud_api/`?
4. **pyproject.toml registration** - New component needs entry in `[tool.polylith.bricks]` section

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-03-api-client-component-extraction.md`
