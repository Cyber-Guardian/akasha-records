---
date: 2026-02-13T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-13-excalidraw-roundtrip-design-workflow.md
last_generated: 2026-02-13T20:38:59.992891+00:00
---

# Research Brief: 2026-02-13-excalidraw-roundtrip-design-workflow

## TL;DR

**Verdict: Highly feasible with Excalidraw as the target format.**

The Excalidraw JSON format is simple, well-documented, and has a Python library (`Excalidraw-Interface`) for programmatic generation. tldraw can import Excalidraw files, so generating `.excalidraw` covers both tools. The codebase already has a comprehensive diagram generation pipeline (`scripts/generate_diagrams.py`) with tool-agnostic data extraction that can be extended to produce Excalidraw output alongside existing Structurizr/Mermaid output.

The key insight: our existing IR layer (dataclasses for `InfraStack`, `PolylithBrick`, `PolylithProject`, `ArchitectureModel`) already extracts all the structural data needed. A new renderer targeting Excalidraw JSON would reuse the same parsers.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Layout algorithm**: Grid vs force-directed vs hierarchical? Force-directed via NetworkX would look most natural but adds complexity.
2. **Granularity levels**: Should we generate separate Excalidraw files per C4 level (L1, L2, L3) or one combined diagram?
3. **customData convention**: What metadata to embed in elements for round-trip identification? Minimum: `{ "source": "generated", "component": "cloud_api", "file": "components/filescience/cloud_api/" }`.
4. **Diff strategy**: When Claude reads back a modified diagram, should it compare against the last generated version, or parse semantically?
5. **tldraw preference**: User currently designs in tldraw. Should we generate `.excalidraw` (simpler) and let tldraw import, or invest in direct `.tldr` generation?
6. **Excalidraw-Interface viability**: Need to verify the library works on Python 3.13 and generates valid output for our use case. May be simpler to generate raw JSON.

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-13-excalidraw-roundtrip-design-workflow.md`
