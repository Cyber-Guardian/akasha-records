# Idea Brief: Plugin Architecture + Helm Intelligence Allocation

**Date:** 2026-03-09
**Status:** Shaped → Parked

## Problem
Two connected bottlenecks limit scaling as a solo developer with agents:

1. **Integration development is manual and fragile.** Adding a new cloud provider touches 12+ files across 6 bricks, with 9 hardcoded seams (if/elif chains, enum values, dict registries). The DDD plan improves internal layering but doesn't address pluggability. An agent can't scaffold a new integration today — there's no contract to follow.

2. **Intelligence allocation is mostly human-decided.** Model selection is static, autonomous/co-dev labeling is a manual 6-criterion judgment, phase promotion/merge timing is manual. Helm automates execution of those decisions, but the decisions themselves are still human.

## Constraints
- 9 hardcoded seams for cloud integrations (session.py, dispatcher.py, queue.py, entities.py, policy registries)
- Helm V1 complete, V2 (execute + promote) planned but not built yet
- Agents cold-start every invocation — no shared state or learned allocation preferences
- Model selection is static (Opus for implementation, Sonnet for review, Haiku for triage)
- `autonomous` vs `co-dev` labeling is a 6-criterion manual judgment call
- Generic framework layer (ApiClient, ServiceRouter, CloudRouter, OAuth2TokenAuth) is solid
- DDD plan gives internal structure but doesn't close the pluggability gap

## Options Considered

### Thread A: Plugin Architecture for Cloud Integrations

#### Convention-over-Configuration ("Drop a Directory")
Replace all 9 hardcoded seams with auto-discovery based on directory structure. Filesystem IS the registry. Walk directories at import time, collect objects implementing Protocols.
- Gains: Zero-touch extension. No wiring code to forget. Convention is documentation.
- Costs: Implicit registration harder to debug. Needs Protocol contracts for type safety.
- Complexity: Medium

#### Explicit Registry with Protocol Contracts ("CloudPlugin ABC")
Formal `CloudPlugin` Protocol every integration implements. Central `PluginRegistry`. Dispatcher, session manager, throttle registry read from registry.
- Gains: Explicit, type-safe, IDE-friendly, easy to validate at startup.
- Costs: More boilerplate per plugin. Protocol becomes coupling surface.
- Complexity: Medium

#### Hybrid — Convention + Protocol + Scaffold Template
Convention-based discovery with Protocol validation, plus a scaffold template (`make new-cloud name=google`) that generates stub files implementing the Protocol with test stubs as specification.
- Gains: Convention = no wiring. Protocol = type safety. Scaffold = agents have a starting point. Test stubs ARE the spec.
- Costs: Three systems to maintain. Scaffold can drift.
- Complexity: Medium

### Thread B: Helm Intelligence Allocation

#### Rule Engine ("Deterministic Allocation")
Encode every allocation decision as deterministic rules. Automate autonomous/co-dev 6-criteria check via filesystem/plan parsing. Model routing by task type. Phase parallelism detection from file scopes.
- Gains: Transparent, debuggable, no LLM overhead. Consistent.
- Costs: Rules are brittle — can't capture "politically sensitive" or subtle judgment.
- Complexity: Low-Medium

#### Blueprint Profiles ("Stripe Model")
Pre-defined YAML workflow templates for common task types. Directed graphs with deterministic + agentic nodes. Helm selects template from issue metadata.
- Gains: Reusable, composable, customizable. Blueprint IS the allocation decision.
- Costs: Blueprint authoring is new skill. Over-engineering risk with few task types.
- Complexity: Medium

#### Adaptive Allocation ("Learning Helm")
Track (task attributes, configuration, outcome) tuples. Statistical analysis over 50+ tasks reveals patterns for model selection, mode assignment.
- Gains: Gets smarter over time. Data-driven.
- Costs: Need volume before patterns emerge. Cold start. "Outcome" hard to define.
- Complexity: Medium-High

## Chosen Approach
**Thread A: Hybrid Plugin Architecture** (convention + protocol + scaffold) — the scaffold template is the key differentiator. Turns "add Google Workspace" from a 12-file scavenger hunt into `make new-cloud name=google` → agent fills stubs → tests pass → PR.

**Thread B: Rule Engine → Blueprint Profiles** — start with deterministic rules (fastest to remove human allocation decisions), extract into blueprint profiles as patterns stabilize. `new-integration` blueprint is the first specialized one, tying directly to the plugin architecture.

## Combined Vision
```
HELM (smarter)
  ├─ Rule engine: "Is this autonomous-eligible?" → YES
  ├─ Blueprint selector: "touches cloud_api/clouds/ → use new-integration blueprint"
  │
  ├─ BLUEPRINT: new-integration
  │   ├─ [deterministic] make new-cloud name=google    ← PLUGIN SCAFFOLD
  │   ├─ [agentic] implement stubs (Opus)              ← agent fills contracts
  │   ├─ [deterministic] make lint && make lint-imports ← protocol validation
  │   ├─ [agentic] fix lint (Sonnet, max 2)
  │   ├─ [deterministic] make test                     ← contract tests
  │   ├─ [agentic] fix tests (Sonnet, max 2)
  │   └─ [deterministic] git push → PR
  │
  └─ Outcome tracking → feeds back into allocation rules
```

## Key Context Discovered During Shaping

### Hardcoded seams (complete inventory)
| Location | Hardcoded Assumption |
|---|---|
| `cloud_api/session.py:35-46` | `_bind_auth()` if/elif chain — only 4 clouds |
| `discover/dispatcher.py:30-33` | `SERVICE_POLICY_MAP` — only OneDrive + Outlook |
| `discover/dispatcher.py:44` | `self._routers` — only `Cloud.OFFICE_365 → O365Router` |
| `discover/queue.py:248` | `tree_id = f"m365:{tenant_id}:..."` — M365 prefix |
| `entity_discovery_trigger/entities.py:20-81` | `get_entities()` always calls GraphClient |
| `valkey_queue_processor/models/policy_types.py:10-14` | `RequestPolicyType` enum — only 2 values |
| `valkey_queue_processor/policies/registry.py:36-57` | `build_throttle_context()` match — only 2 cases |
| `throttling/models.py:28-37` | Parallel `RequestPolicyType` enum — must stay in sync |
| `throttling/policies/m365/` | All gate specs are M365-specific constants |

### What's generic (already works for any cloud)
- `ApiClient`, `PaginationStrategy`, `ResponseTemplate`, `ResponseIterator`
- `OAuth2TokenAuth` base class
- `CloudModel` / `CloudService` / `Cloud` registry (extensible via subclassing)
- `ServiceRouter` decorator pattern + `CloudRouter.execute()` routing
- Queue serialization using `Cloud.from_id()` / `CloudService.from_id()`

### Helm allocation decisions (currently manual)
- autonomous vs co-dev: 6-criteria manual judgment
- Model per agent: static (Opus/Sonnet/Haiku)
- Phase parallelism: human writes `blocked_by` in plan
- Promotion timing: manual `helm status` check
- Merge timing: human runs `helm merge`

### Inspiration sources
- [[2026-03-09-stripe-minions-brainstorm-compilation|Stripe Minions Brainstorm]] — blueprint engine, devbox pool, shift feedback left
- Stripe Minions Part 2 blog post — directed graph of deterministic + agentic nodes
- [[2026-02-28-agent-swarm-harness|Agent Swarm Harness]] — compute layer
- [[2026-03-02-helm-orchestrator-cli|Helm CLI]] — execution layer
- [[2026-02-27-ddd-micro-architecture|DDD Micro-Architecture]] — internal layering (prerequisite for clean plugin boundaries)

## Related Work
- [[2026-02-27-ddd-micro-architecture|DDD Brief]] — internal layering, doesn't address pluggability
- [[2026-03-02-helm-orchestrator-cli|Helm CLI Brief]] — V1 execution, V2 planned
- [[2026-02-28-agent-swarm-harness|Swarm Harness Brief]] — compute layer
- [[2026-02-24-linear-agent-orchestration-system|Linear Orchestration Brief]] — autonomous/co-dev framing
- [[2026-02-05-agentic-back-pressure|Backpressure Research]] — shift feedback left

## Next Step
- [Parked] → ENG-2393: Add plugin architecture for cloud integrations (Discover V2)
- [Parked] → ENG-2395: Add rule-based intelligence allocation to Helm (Helm Orchestrator)
