---
date: 2026-02-13T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-13-l4-diagram-improvement-opportunities.md
last_generated: 2026-02-13T19:51:19.381465+00:00
---

# Research Brief: 2026-02-13-l4-diagram-improvement-opportunities

## TL;DR

The L4 system is well-built — **27 of 29 IR fields are used in rendering** — but five concrete areas have clear gaps: (1) two parsed MethodInfo fields (`is_property`, `is_classmethod`) are never rendered, (2) method parameter/return type relationships are not visualized, (3) theme toggle loses zoom/pan and search state, (4) same-name classes in different sub-modules silently collide, and (5) namespace depth is capped at 1 level. The biggest structural gap vs C4 is the lack of any machine-readable output (JSON/SVG export).

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Should method parameter dependency arrows be dashed (`..>`) to distinguish from field composition (`--*`)? Mermaid classDiagram supports `-->` (association) and `..>` (dependency).
- Would deeper namespace nesting make cloud_api more readable or more cluttered? (64 classes across ~5 sub-packages)
- Is CI-friendly SVG export needed, or is browser-only viewing acceptable long-term?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-13-l4-diagram-improvement-opportunities.md`
