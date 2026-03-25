---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Autoresearch Live Dashboard

**Date:** 2026-03-17
**Status:** Shaped -> Planning

## Problem
When autoresearch runs overnight, there's no way to observe experiment progression, score trends, cost burn, or hypothesis quality in real time. The only feedback is Slack pings and after-the-fact CLI commands.

## Constraints
- State is file-based: `state.json` (atomic overwrite per experiment) + `experiments.jsonl` (append-only JSONL)
- No push/event mechanism — UI must poll files
- No mid-experiment progress written to disk (can't see "researcher is running")
- `EvaluatorResult.metadata` (dedup_ratio, throughput, chunks) is not persisted to JSONL — only `metric_value`
- No existing UI libraries in the dependency tree
- Harness runs as a CLI daemon from a plain terminal

## Options Considered

### Textual Live Dashboard
Full Textual TUI app with file-polling reactor. Watches state.json + experiments.jsonl on a timer, renders live panels. Runs as `autoresearch watch`. Can also `textual serve` for browser access.
- Gains: Rich live display, dual TUI+web, no API layer, no frontend build tooling
- Costs: New dependency (textual). Limited charting vs browser-native.
- Complexity: Medium

### Textual + Eval Metadata Fix
Same as above, plus fix the build_record gap so eval_metadata is persisted to JSONL. Dashboard shows per-experiment diagnostic breakdown.
- Gains: Everything above plus diagnostic visibility — see why a score changed, not just that it changed
- Costs: Small coordinator change + backward compat for old JSONL records
- Complexity: Medium

### Minimal Tail Watcher
Rich-based watch command that clears and reprints a status table every 2s. No interactivity.
- Gains: Simplest possible thing, ~50 lines
- Costs: No interactivity, no browser mode, no charts. Outgrown quickly.
- Complexity: Low

## Chosen Approach
**Textual + Eval Metadata Fix** — the dashboard is only as good as the data it shows. Fixing the metadata gap is small but makes the dashboard significantly more useful. Textual gives TUI-first + browser-optional with no frontend tooling.

## Key Context Discovered During Shaping
- `state.json` is atomically overwritten once per experiment — safe to poll
- `experiments.jsonl` is append-only — can tail from byte offset for incremental reads
- `EvaluatorResult.metadata` contains dedup_ratio, throughput_mbs, peak_memory_mb, unique_chunks, total_chunks but `build_record` only extracts `metric_value` (coordinator.py)
- `ExperimentRecord.metric_variance` field exists but is never populated
- Textual supports `textual serve` for browser access from a TUI app
- Slack notifications fire at: champion events, crashes, budget exhaustion, every 10th experiment
- Existing CLI has `status`, `history`, `champion` commands — dashboard would be `watch`

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-autoresearch-live-dashboard.md`
