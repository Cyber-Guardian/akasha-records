# Idea Brief: Visualization in Shaping & Planning Workflow

**Date:** 2026-02-27
**Status:** Shaped → Planning

## Problem
During shaping and planning, we reason about architecture, flows, and domain models in prose only. The highest-leverage diagrams are "forward design" / "thinking tools" — drawn before code to help reason about design — but we only have infrastructure for "backward design" diagrams (generated from existing code via `scripts/diagrams/`). There's no way to go from "here's a design idea" → editable diagram during shaping/planning.

## Constraints
- Must stay in Python ecosystem (no Node.js runtime dependencies)
- Must produce editable output (not static images) — the diagram is a thinking tool, not documentation
- LLMs cannot reliably compute pixel-level spatial layout (the coordinate problem)
- Must integrate with existing skill system (`.claude/commands/` markdown files)
- Output should be viewable in VS Code (Excalidraw extension is mature)
- Claude must be able to review and iterate on the diagram (feedback loop)
- Layout engine should be a default, not a constraint — Claude can override positioning

## Options Considered

### 1. Mermaid-First Skill
LLM generates Mermaid syntax, validate, render as fenced blocks in briefs/plans.
- Gains: Simplest, zero deps, renders natively in GitHub/VS Code
- Costs: Not editable, looks "finished" (wrong signal for thinking tools), Dagre-only layout
- Complexity: Low

### 2. Excalidraw via Python Pipeline
LLM outputs structured graph description, Python layout engine computes coordinates, emits `.excalidraw`.
- Gains: Editable output, hand-drawn aesthetic, builds on existing primitives, stays in Python
- Costs: Need to implement layout algorithm, specialized layout for sequence diagrams
- Complexity: Medium

### 3. Excalidraw MCP
Use official `excalidraw/excalidraw-mcp` server for live diagram generation in Claude chat.
- Gains: Handles layout/rendering, agent can see output, low effort
- Costs: External dependency, requires MCP server running, less portable
- Complexity: Low-Medium

### 4. Hybrid Pipeline (Mermaid → Excalidraw)
LLM generates Mermaid, convert to Excalidraw for flowcharts, render SVG for rest.
- Gains: Reliable generation + editable output where possible
- Costs: Two rendering paths, Node.js dependency, conversion only works for flowcharts
- Complexity: Medium-High

## Chosen Approach
**"Mermaid Brain, Excalidraw Body" (synthesis of 2 + 4)** — LLM generates a simple YAML graph spec (simpler than Mermaid, zero parsing ambiguity), Python layout engine computes coordinates with auto/manual/assisted modes, emits `.excalidraw` (editable) + `.svg` (for Claude's iteration loop). Claude iterates via SVG review, final visual check via rendered screenshot.

Key design decisions:
- Graph spec supports optional position/size overrides so Claude can break out of cookie-cutter layouts
- Three layout modes: `auto` (engine decides), `manual` (Claude places everything), `assisted` (engine handles unpositioned nodes, respects overrides)
- SVG output enables fast iteration (Claude reads SVG as image), final check uses actual Excalidraw render
- Style presets map diagram concepts to colors (service=blue, database=green, queue=orange, external=purple)

## Key Context Discovered During Shaping

### Existing infrastructure
- `scripts/diagrams/excalidraw/primitives.py` — rect, text, arrow, frame, document builders (not connected to pipeline, no tests)
- `scripts/diagrams/excalidraw/layout.py` — arrow binding helpers (`distribute_focus`, `bind_arrow_to_elements`)
- `scripts/diagrams/ir.py` — intermediate representation dataclasses
- Full diagram pipeline exists for backward-design: parsers → renderers (structurizr, mermaid, html)

### Research on LLM diagram generation
- LLMs generate Mermaid most reliably (most training data), validate+repair loop is proven (Microsoft ships this)
- LLMs cannot do spatial layout — pixel coordinates with no overlap is the hard problem
- Official Excalidraw MCP server exists (from Excalidraw team + Anthropic)
- `Excalidraw-Interface` Python library exists on PyPI (v0.1.3, Oct 2025)
- `graphviz` Python package can be used as a layout engine (compute coordinates, extract positions)
- Proven pattern: separate semantic generation (LLM) from geometric layout (engine)

### Diagram types most valuable during shaping/planning
- Tier 1: C4 L1/L2 (architecture), entity/domain models, data flow diagrams
- Tier 2: Sequence diagrams, state machines, dependency graphs
- Tier 3: Event Storming boards, trade-off matrices, Wardley Maps

### Forward vs backward design insight
- "Diagrams for thinking" (messy, iterative, pre-code) vs "diagrams for documenting" (clean, canonical, post-code)
- We handle backward design already. The gap is entirely forward design.
- Excalidraw's hand-drawn aesthetic signals "still being decided" — the right visual language

### Full research artifacts
- Standard SWE diagram taxonomy with design-phase rankings
- Non-standard diagram types (Wardley Maps, Event Storming, Impact Maps, OSTs, CLDs, etc.)
- LLM diagram generation state of the art (MCP servers, skills, Mermaid validation pipelines)
- Excalidraw format deep dive (JSON schema, 11 element types, programmatic generation, layout algorithms)
- Existing codebase analysis (primitives, layout helpers, pipeline architecture)

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-27-visualization-in-shaping-planning.md`
