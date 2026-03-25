---
type: event
created: 2026-03-18
status: active
---
# Idea Brief: Recording Fake Strategy for Protocol-Backed Components

**Date:** 2026-03-18
**Status:** Shaped → Planning

## Problem
We have Protocol contracts for 6 interfaces but no general pattern for creating recording fakes that implement them. The existing RecordingSlackClient is hand-rolled, incomplete (3/6 methods), and has no conformance enforcement. Two duplicate files (recording_fakes.py, fixtures.py) across test/shared/ and tools/test/shared/ create a latent divergence bug.

## Constraints
- RecordingSlackClient exists in two identical copies (test/shared/ and tools/test/shared/)
- MockCheckpointContext exists in two DIVERGENT copies — tools/ version is a superset with call tracking
- DynamoDB tests all use moto (16 files) — no recording fake needed
- SessionManager tests just MagicMock it to avoid AWS SSM — no recording fake consumer exists
- ThrottleService is deeply stateful (14 methods, tree/gate state machine) — too complex for a simple recording fake

## Options Considered

### Hand-Written Recording Fakes (per component)
Each component gets a tailored recording fake: commands record to lists, queries serve canned responses, command+query does both.
- Gains: Maximum control, easy to understand
- Costs: Boilerplate per interface, manual maintenance
- Complexity: Low

### Generic RecordingProxy Base Class
Auto-record all calls via base class, override for special behavior.
- Gains: Eliminates boilerplate
- Costs: Indirection, harder to read, premature abstraction for 3-4 fakes
- Complexity: Medium

### Tiered Fakes (chosen)
Tier 1 recording fakes for simple interfaces (SlackClient, CheckpointContext). Tier 2 stateful fakes for complex interfaces (ThrottleService) deferred. DynamoDB and SessionManager skipped.
- Gains: Right tool for the job, doesn't over-engineer simple cases
- Costs: Two patterns to understand
- Complexity: Low for Tier 1, High for Tier 2 (deferred)

## Chosen Approach
**Tiered Fakes** — Tier 1 now (ENG-2593), Tier 2 deferred to separate issue.

## Key Context Discovered During Shaping
- I/O classification: 6 command methods, 16 query methods, 15 command+query methods across all interfaces
- SessionManager has no test consumer for a recording fake — `test_dispatcher.py:49` and `test_telemetry.py:21` just use MagicMock
- `test/shared/fixtures.py` MockCheckpointContext is stale — missing callback_replies, step_names, callback_names that the tools/ version has
- `recording_fakes.py` files are identical (confirmed via diff), but `fixtures.py` files are divergent
- Moto covers all 16 DynamoDB test files — no recording fake needed
- [[2026-03-17-cross-brick-protocol-contracts|Protocol Contracts Plan]] shipped Protocols for cloud_api, dynamodb, throttling

## Next Step
- [Plan] → `/create_plan` for ENG-2593 with narrowed scope: SlackClientProtocol + RecordingSlackClient upgrade + CheckpointContext conformance + file consolidation
