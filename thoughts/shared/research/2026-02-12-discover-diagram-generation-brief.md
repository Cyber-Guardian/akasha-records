---
date: 2026-02-12T18:53:33Z
source_research: memory-bank/thoughts/shared/research/2026-02-12-discover-diagram-generation.md
last_generated: 2026-02-12T19:07:52.745161+00:00
---

# Research Brief: 2026-02-12-discover-diagram-generation

## TL;DR

The best-fit path in this repository is to use **Mermaid diagrams in Markdown** as the source of truth for `discover` flow documentation, then optionally export static images via `@mermaid-js/mermaid-cli` when PNG/SVG artifacts are needed.

Why this fits current repo state:
- `discover` already has a current, detailed textual code map in `memory-bank/thoughts/shared/research/2026-02-12-discover-code-map.md`.
- There are currently no existing Mermaid/PlantUML/Graphviz diagram assets or diagram build workflows in this repo.
- Markdown-first documentation is already the dominant format in `memory-bank/thoughts/`.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Should the first diagram include only the `discover` base runtime, or also `entity_discovery_trigger` + stream processor Lambda as upstream/downstream context?
- Is a single high-level diagram sufficient, or are both context + sequence diagrams desired for handoff/debug workflows?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-12-discover-diagram-generation.md`
