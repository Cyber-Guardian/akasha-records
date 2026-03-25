---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Multiversal Map Web Dashboard

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
The TUI heatmap (Phase 4-5 of the multiversal map plan) crashed and was visually confusing — terminal rendering can't do justice to a multidimensional search space visualization. The data model (dimensions, coordinates, leader intelligence) from Phase 1-3 is solid and stays. We need a proper web frontend to visualize the experiment search space.

## Constraints
- Data layer already exists: `state.json` + `experiments.jsonl` polled from disk, `ExperimentRecord.dimension_coordinates` populated by leader and sweep
- Autoresearch runs locally — this is a local dev tool, not deployed to Vercel
- The monorepo uses Polylith; new project goes in `tools/projects/`
- User wants idiomatic, boring choices — minimize decision overhead

## Options Considered

### shadcn Table + Tailwind heatmap
Use a styled HTML `<table>` with inline HSL backgrounds for cell coloring. shadcn has a `tables-heatmap` block for this pattern. Click handlers are just `<td onClick>`.
- Gains: Simplest approach, zero chart library overhead, perfect shadcn dark mode fit, custom text per cell is trivial
- Costs: No fancy animations/transitions, limited to small grids
- Complexity: Low

### Nivo @nivo/heatmap
Purpose-built React heatmap component with categorical axes, `onClick`, and `cellComponent` for custom rendering.
- Gains: Full-featured chart with axes, tooltips, color legends built in. Canvas fallback for large grids.
- Costs: 0.x version, theme object doesn't read CSS variables natively (needs workaround), heavier bundle
- Complexity: Medium

### visx @visx/heatmap (D3 primitives)
Airbnb's low-level D3+React library. Maximum control, minimum abstraction.
- Gains: Pixel-level control, tiny bundle, pairs naturally with Tailwind
- Costs: 150-250 lines of boilerplate for axes/tooltips/labels, docs site partially broken
- Complexity: Medium-High

## Chosen Approach
**shadcn Table + Tailwind heatmap** — the grid is small (categorical, ~5x5), needs custom text per cell and click-to-drill-down. A styled HTML table is simpler and more native to shadcn than any chart library. Upgrade to Nivo if the grid outgrows the table approach.

Supporting stack: Next.js App Router, TanStack Query with `refetchInterval: 2500` for polling, pnpm, API route reading disk via `fs`.

## Key Context Discovered During Shaping
- TUI `map_screen.py` and `detail_screen.py` crashed and should be deleted
- shadcn has a `tables-heatmap` block that's the exact pattern needed
- Tailwind can't interpolate dynamic class names but inline `style={{ backgroundColor: hsl(...) }}` works perfectly
- TanStack Query polling is simpler than SSE for a local tool and has zero connection management
- Next.js App Router SSE has known buffering bugs requiring specific headers — avoided by using polling instead
- The data model from [[2026-03-17-multiversal-map-tui|Multiversal Map TUI Plan]] Phase 1-3 is the foundation

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-multiversal-map-web-dashboard.md`
