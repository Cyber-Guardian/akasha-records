---
date: 2026-01-30T20:22:52.037004+00:00
source_research: memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md
last_generated: 2026-01-30T20:22:52.037004+00:00
---

# Research Brief: 2026-01-30-valkey-queue-processor-dependency-analysis

## TL;DR

Scope: `bases/filescience/valkey_queue_processor` (the Lambda “function code” base) and how it’s packaged via `projects/valkey-queue-processor`.
## Executive summary
- The base has **one cross-brick (components/) dependency**: `filescience.throttling`.
- The `valkey-queue-processor` project **packages additional components** (e.g. `cloudbridge_model`, `config`, `dynamodb`, `domain_backup`) into the function wheel/zip **even if not imported** by the base. This is driven by Polylith brick mapping in the project’s `pyproject.toml`.
- There is a **potential runtime mismatch** between:
  - the Lambda handler’s Valkey client import (`glide.GlideClient`), and
  - the throttling component’s expected client type (`glide_sync.GlideClient`).
- Lambda layer build currently installs `valkey-glide` but **not** `valkey-glide-sync`; if the throttling component is used (it is), the layer may need to include `valkey-glide-sync` too.
## Cross-brick dependencies (components/)
Only `filescience.throttling` is imported from within the base:

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md`
