---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Cross-Brick Protocol Contracts

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
Three components (`cloud_api`, `dynamodb`, `throttling`) have cross-brick interfaces consumed by bases, but those interfaces are implicit — defined by whatever consumers happen to import. No machine-checkable contract exists for the API surface, test doubles can't be validated against the real interface, and interface drift is only caught by import errors at runtime.

## Constraints
- PEP 544 warns against "too large, complex, or implementation-oriented" Protocols — stdlib Protocols are 1-7 methods
- `typing.IO` (15 methods) is being actively deprecated in favor of smaller Protocols — this is a counter-example, not precedent
- `@runtime_checkable` only checks method existence, not signatures — behavioral tests required alongside
- The codebase has zero Protocols in components today (one trivial one in bases: `DiscoveryMetrics`)
- dynamodb already has ABCs for pluggable parts (`Serializer`, `ModelMapper`, `ReadExecutor`)
- Polylith convention: interface definitions live inside the component (provider-side)
- Project decision: "Python Protocols for internal service boundaries" (pass-1 foundation sketch)

## Options Considered

### Full API Mirror
One Protocol per component mirroring the entire public API. Dead simple, but ThrottleService (14 methods + 7 nested) pushes into the `typing.IO` anti-pattern zone.
- Gains: Simple mental model, no judgment calls
- Costs: Ignores PEP 544 guidance on Protocol size
- Complexity: Low

### Consumer-Side Protocols (Go-style)
Each base defines its own narrow Protocol for what it needs. Maximally narrow but creates artifact sprawl (up to 9 Protocol files) and verbose naming.
- Gains: True ISP, each Protocol 3-6 methods
- Costs: Sprawl, naming clutter, novel pattern for the codebase
- Complexity: Medium-high

### Composed Protocols (chosen)
Split large interfaces into concern-based Protocols that compose via multiple inheritance. Small interfaces stay as single Protocols. Provider-side placement inside each component.
- Gains: Each sub-Protocol is 3-5 methods (PEP 544 sweet spot). Full composed Protocol available when needed. Acts as design constraint. Consumers can type-hint narrowly.
- Costs: More Protocol definitions for throttling. Someone decides which concern a method belongs to.
- Complexity: Medium

## Chosen Approach
**Composed Protocols** — concern-based decomposition for large interfaces, single Protocol for small ones. Provider-side `protocols.py` in each component.

- **cloud_api**: `SessionManagerProtocol` (2 methods) — small enough as-is
- **dynamodb**: Protocol for async utility function surface + existing ABCs for pluggable parts
- **throttling**: Composed — `ThrottleRegistration`, `ThrottleAdmission`, `ThrottleContext`, `ThrottleFunctions` sub-Protocols composing into `ThrottleServiceProtocol`

Validation: explicit Protocol subclassing on concrete classes (ty checks conformance at definition time) + `@runtime_checkable` + behavioral contract tests.

## Key Context Discovered During Shaping
- `typing.IO`/`TextIO` (15 methods) is a counter-example, not precedent — typeshed is deprecating them in favor of smaller Protocols (`typing` discussion #829)
- PEP 544 gives no method count threshold — "too complex" targets stdlib containers with dundermethods, not domain services
- Community consensus: "create compact protocols and combine them" (Stanza, Idego, Real Python)
- ThrottleService methods naturally cluster: registration (5), admission (3), context (3), penalty (1), plus nested `.functions` (7 consumed)
- Only 1 Protocol exists in the codebase today (`DiscoveryMetrics` — trivial). This is a pattern introduction.
- Provider-side placement aligns with Polylith's interface-inside-component convention
- `@runtime_checkable` gap: only checks method existence, not signatures. Mitigated by explicit Protocol subclassing + behavioral tests.
- Glyph (Twisted creator): dependency arrow argument validates consumer-defined Protocols conceptually, but in a monorepo provider-side is pragmatic
- Cosmic Python (canonical hex-arch text): acknowledges Protocol as alternative to ABC for ports

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-cross-brick-protocol-contracts.md`
