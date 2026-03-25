---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Typed Linear GraphQL Client

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
Hand-written GraphQL query strings in `tool_linear_client/queries.py` are validated against nothing at build or test time. Tests mock the HTTP layer, so field-name mismatches (like `state { type }` vs `status { name }`) are invisible until production. This caused a silent bot crash that affected all triage bot users.

## Constraints
- `tool_linear_client` is a Polylith component used by both async Lambda handlers and sync CLI tools
- 8 queries + 5 mutations, ~450 lines of client code with manual dict→Pydantic parsing
- `update_issue` builds dynamic mutations at runtime (f-string GraphQL) — codegen can't cover this directly
- Component is a leaf in the Polylith graph — no filescience component deps
- Linear API must support introspection for codegen to fetch the schema

## Options Considered

### Ariadne-codegen Full Replacement
Replace queries.py, models.py, and manual parsing entirely with ariadne-codegen output. Generated client becomes the public API.
- Gains: Eliminates entire class of field-name bugs, type checker validates full chain, no manual parsing
- Costs: High migration — rewrite all callsites, refactor dynamic mutations, unknown Polylith interaction
- Complexity: High

### Ariadne-codegen Behind Existing Interface (Facade)
Generate typed client internally, keep existing LinearClient/AsyncLinearClient API unchanged. Generated client is an implementation detail — our clients delegate to it. Hand-written models stay as public interface.
- Gains: Same schema validation guarantee, zero callsite changes, dynamic update_issue stays as-is
- Costs: Thin adapter layer, two model sets (generated + ours), codegen build step
- Complexity: Medium

### Schema Validation Test Only
Single integration test that fetches Linear's schema and validates all query strings against it. No codegen, no new types.
- Gains: Catches the bug class with ~30 lines, zero migration, no new deps
- Costs: No typed responses, manual parsing stays, only catches at CI time not write time
- Complexity: Low

## Chosen Approach
**Ariadne-codegen Behind Existing Interface** — best balance of structural safety and pragmatism. Schema validation at codegen time eliminates the bug class. Facade pattern means zero callsite migration. Dynamic mutations stay as-is. Can incrementally expose generated types to callsites later.

## Key Context Discovered During Shaping
- ariadne-codegen v0.18.0 (Mar 2026) generates Pydantic v2 models + async httpx client, validates queries against schema at generation time
- Linear API Project type has `status { name }`, not `state { type }` — Issues have `state { name }`, confusing similarity
- The `update_issue` method builds GraphQL dynamically via f-strings — must stay hand-written or be refactored into discrete operations
- `tool_linear_client` is used by: `tool_triage_agent` (async), `helm` CLI (sync), and tests

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-typed-linear-graphql-client.md`
