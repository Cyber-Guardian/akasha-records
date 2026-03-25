# Idea Brief: DDD Micro-Architecture (Polylith Internals)

**Date:** 2026-02-27
**Status:** Shaped -> Planning

## Problem
Polylith gives clean macro boundaries (import-linter DAG), but inside bricks the structure is ad-hoc. No consistent vocabulary for "domain model," "application service," "infrastructure adapter." Some bricks do it well implicitly (throttling has clean value objects + domain services + infra adapters), others mix concerns (WorkQueue is ~710 LOC of DynamoDB transactions + retry logic + throttle coordination). Agents lack predictable internal structure to pattern-match against, reducing code quality and increasing review burden.

## Constraints
- ~5 bricks have enough mass to benefit (throttling ~3500 LOC, cloud_api ~1500, dynamodb ~1200, discover base ~2600, VQP base ~2800)
- ~5 bricks are too small for DDD overhead (models 141 LOC, domain_backup 92, lambda_utils 103, telemetry 340)
- import-linter already enforces inter-brick DAG — same tool can enforce intra-brick layering
- Agents (Cyrus) work autonomously via CLAUDE.md conventions + plan-implementer subagent
- Must not break existing 407+ tests or 7 import-linter contracts
- Python 3.13+, frozen dataclasses as value objects (already used in throttling)

## Options Considered

### Convention-Only ("Document, Don't Restructure")
Document DDD vocabulary in CLAUDE.md, retrofit opportunistically. Zero migration cost.
- Gains: Immediately usable, no risk
- Costs: No enforcement, relies on discipline, agents can still violate conventions
- Complexity: Low

### Folder Convention Only
Standardize `domain/` / `application/` / `infrastructure/` inside complex bricks. Import-linter enforces layering.
- Gains: Structural signal for agents, enforceable via import-linter
- Costs: Migration effort on ~5 bricks, import path changes, test updates
- Complexity: Low-Medium

### Hybrid (Folder Convention + Lightweight Building Blocks)
Folder convention + thin `ddd` component with marker base classes (ValueObject, Entity, AggregateRoot, DomainEvent, Repository Protocol). Not a framework — canonical examples agents copy from.
- Gains: Structural + semantic signal. Agents get both *where* (folder) and *what* (typed block). Self-documenting.
- Costs: Migration on ~5 bricks, new thin component. Incremental retrofit.
- Complexity: Medium

### Full Cosmic Python (Folder + Blocks + Events + UoW)
Complete DDD with domain events, in-process message bus, Unit of Work pattern.
- Gains: Decoupled side effects, testable event flows, richest agent vocabulary
- Costs: Significant rearchitecture, event indirection, co-existing patterns during migration
- Complexity: High

## Chosen Approach
**Hybrid (Folder Convention + Lightweight Building Blocks)** — gives agents both structural and semantic signal. The folder convention provides *where*, the building blocks provide *what*. Import-linter enforces layering. The building blocks component stays deliberately thin (marker classes + docstring examples, not a framework). Full domain events can be layered in later when imperative side effects cause pain.

## Key Context Discovered During Shaping

### Existing DDD patterns (already in codebase)
- Value objects: `throttling/models.py` — `RateGateSpec`, `ConcurrencyGateSpec`, `GateCharge` (frozen dataclasses)
- Entities: `domain_backup/models.py` — `Entity`, `Node`, `Resource`, `Artifact` (uuid5 identity)
- Repository pattern: `dynamodb/mapper.py` — `ModelMapper[Persistence, Domain]` + `Serializer` ABCs
- Strategy/Policy: `RequestPolicy` ABC with concrete M365 policies
- Anti-corruption layer: DynamoDB mappers, cloud_api Graph response translation

### Research findings
- Cosmic Python is the canonical Python DDD reference (frozen dataclasses, ABC repos, in-process event bus)
- Knowledge graph code structure improved agent pass@1 by 75% (arxiv 2505.14394)
- Context-rot empirically proven — bounded contexts reduce token surface area (Chroma research)
- Building blocks constrain agent search space: "one example seen -> all subsequent ones generated correctly"
- Kraken Technologies: 27k Python modules, import-linter enforcement — production precedent
- No one has publicly combined Polylith + DDD for AI agents — unexplored niche

### Import-linter enforcement (existing infrastructure)
- 7 contracts in `pyproject.toml` enforcing Polylith boundaries
- Same tool can enforce `domain/` cannot import `infrastructure/` within bricks
- Kraken Technologies uses identical approach at 27k-module scale

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-02-27-ddd-micro-architecture.md`
