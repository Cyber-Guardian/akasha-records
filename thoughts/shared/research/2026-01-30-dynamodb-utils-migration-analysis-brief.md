---
date: 2026-01-30T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-01-30-dynamodb-utils-migration-analysis.md
last_generated: 2026-01-30T20:40:07.092504+00:00
---

# Research Brief: 2026-01-30-dynamodb-utils-migration-analysis

## TL;DR

The DynamoDB utilities in `fsbedrock` provide a sophisticated 8-layer abstraction for DynamoDB operations supporting both sync and async patterns. The old discover codebase uses these utilities extensively for persisting backup discovery metadata (entities, resources, artifacts, delta states) and managing a work queue. The migrated discover in the monorepo has **stubbed out all DynamoDB persistence operations** - this is the primary gap that needs to be addressed.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Component vs Base**: Should DynamoDB utilities be a `component` (shared library) or stay embedded in bases?
2. **Async-Only**: Should we migrate both sync and async versions, or just async?
3. **BiDirectionalQueryExecutor**: Is this optimization needed in the new system?
4. **ParallelScanExecutor**: Is parallel scan functionality required?
5. **Domain Mappers Location**: Should mappers live with `domain_backup` component or with `dynamodb` component?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-01-30-dynamodb-utils-migration-analysis.md`
