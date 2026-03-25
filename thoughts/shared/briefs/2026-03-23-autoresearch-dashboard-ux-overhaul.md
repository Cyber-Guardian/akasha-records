---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Autoresearch Dashboard UX Overhaul

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
The autoresearch coordinator makes dozens of decisions per iteration (tree parent selection, plan generation, novelty filtering, eval tier progression, champion promotion, stale detection) but none are visible. The dashboard only shows output (experiment records) not reasoning. A lab lead should never wonder "what's it doing right now" — every coordinator state should be surfaced. Additionally, the researcher cards are too dense and don't scale past 3 researchers, and there's no information hierarchy guiding the lab lead's attention.

## Constraints
- `CampaignJournal` already exists at `journal.py:20-38` — append-only JSONL, fail-safe, timestamped, wired into coordinator at 6 call sites
- Journal currently only logs 4 event types: campaign_started, experiment_complete/crash, budget_exhausted, campaign_complete
- Dashboard does NOT currently read `campaign-journal.jsonl`
- Next.js dashboard reads JSONL via `readFileSync` + split — naive but sufficient for <50KB files
- Python `CampaignWatcher` has incremental JSONL reading via seek+offset — available if needed
- Max 5 parallel researchers, typically 3
- 2.5s polling interval

## Options Considered

### Create New live_events.jsonl
New event log file alongside existing journal. Dashboard reads it separately.
- Gains: Clean separation from existing journal
- Costs: Duplicates existing CampaignJournal infrastructure. Two JSONL files for similar purpose.
- Complexity: Medium

### Enrich Existing CampaignJournal
Add ~15 more `journal.log()` calls for thinker/eval/novelty/tree decisions. Read `campaign-journal.jsonl` in the dashboard. Render as timeline.
- Gains: Zero new infrastructure. Reuses proven append-only pattern. Journal doubles as debug log. One source of truth for all coordinator events.
- Costs: Journal file grows with campaign (but <50KB for typical campaign). Mixing lifecycle events with decision events in one file.
- Complexity: Low-Medium

### Event Bus via Meta-Server (SSE)
Push events through meta-server with SSE endpoint.
- Gains: True real-time
- Costs: New protocol, reconnect complexity, meta-server needs coordinator awareness
- Complexity: High

## Chosen Approach
**Enrich Existing CampaignJournal** — the infrastructure already exists. The work is: (1) add journal.log() calls at every coordinator decision point, (2) read campaign-journal.jsonl in the dashboard, (3) build a Lab Lead Timeline component, (4) simplify researcher cards to minimal status indicators.

## Key Context Discovered During Shaping
- `CampaignJournal` at `journal.py:20-38` is append-only JSONL with `{timestamp, event_type, **details}` — exactly the event stream we want
- 6 existing call sites in coordinator.py: lines 2296, 2329, 2353, 2394, 2413, 2437
- Missing events: baseline_started/complete, thinker_started/complete (with plan summary), researcher_dispatched, eval_tier_started/passed/failed, champion_promoted, novelty_rejected, stale_detected, sweep_triggered, tree_parent_selected
- The Python TUI watcher (`watcher.py`) has incremental JSONL reading via seek+offset — could be ported to the Next.js dashboard if file size becomes an issue, but <50KB is fine with full reads
- Researcher cards show 6 pieces of info per card; the only question during execution is "progressing or stuck?" — cards should be simplified to status + hypothesis + elapsed

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-autoresearch-dashboard-ux-overhaul.md`
