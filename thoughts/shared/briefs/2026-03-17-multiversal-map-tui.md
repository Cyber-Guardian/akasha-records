---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Multiversal Map TUI

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
The autoresearch harness runs campaigns that explore combinatorial technique spaces — but there's no way to see the shape of the search. Experiments appear as a flat sequential log. You can't see which regions of the space have been explored, where the champion lead threaded through, or what each researcher is trying at a glance. The system also lacks a structured coordinate system — LLM-driven experiments describe technique choices in free text, making it impossible to map them spatially.

## Constraints
- The harness is domain-general (any `evaluator_cmd` + `strategy_file`), so the coordinate system can't be hardcoded to compression techniques
- Sweep mode already has structured coordinates (`ParameterFork`); LLM mode does not
- `ExperimentRecord` is the universal data format — any coordinate system must flow through it
- Textual is already a dependency (live dashboard exists); the map extends it
- The coordinate system is a data model change, not just a visualization — it should also feed back into the leader's prompt (coverage awareness)

## Options Considered

### Campaign-Defined Dimensions
YAML config declares axes and options upfront. Leader picks from them. TUI reads config to build the map.
- Gains: Legible map from experiment 1, structured data, easy to validate
- Costs: Requires campaign author to know the space upfront; rigid if the space evolves mid-campaign
- Complexity: Low

### Emergent Dimensions
Leader tags experiments with freeform key-value pairs. Visualization clusters dynamically.
- Gains: Maximum flexibility, no upfront config needed
- Costs: No spatial structure until enough data accumulates; hard to render as a grid/map
- Complexity: High (clustering, layout algorithms)

### Hybrid: Declared Axes + Freeform Tags
Config declares expected dimensions (structured grid). Leader picks one value per dimension AND can add ad-hoc tags for sub-variations. Map renders declared dimensions as axes; tags show on drill-in.
- Gains: Structured map from the start, flexible for mid-campaign discovery, works for any domain
- Costs: Slightly more complex config schema; leader prompt needs dimension-awareness
- Complexity: Medium

## Chosen Approach
**Hybrid: Declared Axes + Freeform Tags** — gives a legible map from experiment 1 (because axes are declared in config), but doesn't box in campaigns that discover unexpected dimensions mid-run. Different campaigns declare different dimensions — compression uses `chunker × dedup × delta × compress`, an ML campaign uses `architecture × optimizer × augmentation`. The leader's output contract extends to: "pick one value per declared dimension, optionally add freeform tags."

## Key Context Discovered During Shaping
- `ExperimentRecord` at `models.py:124` carries `hypothesis` (free text) and `eval_metadata` (dict) — the coordinate data needs a new structured field
- `LabLeaderDecision` at `models.py:105` returns structured JSON already — adding `technique_selections: dict[str, str]` is a natural extension
- `sweep.py` `ParameterFork` at `sweep.py:28` already has structured parameter data that can map to the same coordinate system
- The existing `dashboard.py` uses Textual's `DataTable`, `Sparkline`, `Static` — the map view would be a new `Screen` pushed on top
- `CampaignWatcher` polls `state.json` + `experiments.jsonl` — the map just needs a richer snapshot
- Coverage data fed back into the leader prompt ([[2026-03-16-autoresearch-research-quality|Research Quality Brief]]) would be a meaningful intelligence upgrade — the leader could see explored vs. unexplored regions

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-multiversal-map-tui.md`
