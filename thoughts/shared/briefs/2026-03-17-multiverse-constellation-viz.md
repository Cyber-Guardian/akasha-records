---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Multiverse Constellation Visualization

**Date:** 2026-03-17
**Status:** Shaped → Implementing

## Problem
The autoresearch web dashboard's Multiverse tab uses a force-directed graph that produces random-looking layouts with no spatial meaning. Position encodes nothing, links represent temporal sequence (meaningless), and 450 possible dimension combinations are invisible. The visualization fails to answer "where have we explored, what worked, and where should we look next?"

## Constraints
- 4 dimensions × variable options = 450 combinations — too many for one-node-per-combo
- `dimension_coordinates` currently empty on all experiments — must degrade gracefully
- 2.5s polling — layout must be deterministic/stable across refreshes
- Next.js + shadcn + dark theme aesthetic
- Local dev tool — performance edge cases acceptable

## Options Considered

### Dimensional Constellation (Radial Sectors)
Radial layout: center = campaign, sectors = primary dimension values, sub-sectors = secondary dimension. Each experiment = a glowing node in its sector. Empty sectors = visible voids. Dimension selector changes primary/secondary axes. Unpositioned experiments in a "limbo" zone.
- Gains: Spatial metaphor, position has meaning, voids show unexplored space, deterministic layout, unique identity
- Costs: Flattens 4D to 2D, custom layout math, doesn't show correlations as clearly as parallel coordinates
- Complexity: Medium

### Parallel Coordinates (Domain Standard)
Replace with Optuna/W&B-style parallel coordinates. Each dimension + score = vertical axis, each experiment = polyline.
- Gains: Information-dense, shows all 4 dimensions simultaneously, domain-standard, shows correlations
- Costs: No spatial exploration feel, looks like every other ML tool, lines not nodes
- Complexity: Low

### Constellation + Parallel Coordinates (Hybrid)
Both views in the tab with cross-highlighting.
- Gains: Both cognitive modes (exploration + analysis), cross-highlighting creates insight bridges
- Costs: Two visualizations to maintain, complex interaction design
- Complexity: High

### Semantic Force Graph (Fix Current)
Keep force graph but use dimensional similarity for forces instead of temporal links.
- Gains: Cheapest change, clusters emerge from shared dimensions
- Costs: Stochastic layout (jittery with polling), no clusters without coordinates
- Complexity: Low

## Chosen Approach
**Dimensional Constellation** — delivers the "multiverse exploration" spatial metaphor where position encodes dimension coordinates and empty voids show unexplored regions. Parallel coordinates deferred as analytical complement (fast follow).

## Key Context Discovered During Shaping
- /ground research confirmed parallel coordinates is the domain standard for search space viz (Optuna, W&B) — but that's the analytical tool, not the exploration tool
- Mind maps/trees impose false hierarchy on combinatorial search spaces — radial sectors with dimension-based positioning is the right compromise
- react-force-graph-2d can be used with physics disabled (fixed positions) OR replaced with plain Canvas/SVG
- The campaign config declares dimensions with named options — this gives us deterministic sector assignments from day 1
- Experiments without coordinates degrade to a "limbo" zone, not a broken layout

## Next Step
- [Implementing] → no plan needed, proceeding directly — replace multiverse-map.tsx and graph-utils.ts
