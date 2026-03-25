---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Video Production Feedback Loop

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
The video production feedback loop requires a human to watch every iteration because Claude is blind to the output. Each cycle costs 2-5 minutes (render + watch + formulate feedback + describe verbally + re-render). Across 8 iterations in the first video session, this consumed 20-40 minutes of human time on feedback that Claude should have caught itself (wrong stock photos, centered text, uniform pacing, bad font, zoom on text slides).

## Constraints
- Claude CAN read images (used successfully for stock photo review in the same session)
- FFmpeg can extract key frames from video
- ffprobe provides quantitative metrics (duration, bitrate, WPM analysis)
- The render script (render_video.py) is ours to extend
- Each render takes 10-30s for FFmpeg assembly
- The /video skill and quality rule already exist but aren't wired into an automated loop

## Options Considered

### Self-Review Pipeline
Build review_video.py that extracts key frames, reads them as images, runs quantitative checks (WPM, sync, durations), produces structured report. Claude iterates 3-5x autonomously.
- Gains: Catches 60-70% of issues before human sees it
- Costs: 10-20s per iteration for extraction + analysis
- Complexity: Medium

### Shot List Manifest (Phase 2)
YAML scene manifest with per-slide timing, transitions, zoom, text overlays declared upfront. Decisions happen in text, not rendered video.
- Gains: First render much closer to final. Textually reviewable.
- Costs: More upfront authoring. Can't catch visual issues pre-render.
- Complexity: Medium

### Preview Renders
Render only first 3 slides as quick 10s preview. Lock visual style before full render.
- Gains: 3-5x faster iteration on style decisions
- Costs: Doesn't catch full-video pacing/flow issues
- Complexity: Low

## Chosen Approach
**Hybrid** — all three, shipped incrementally. Self-Review Pipeline first (highest immediate impact), then Preview Renders (low effort), then Shot List Manifest (Phase 2 of the video plan).

## Key Context Discovered During Shaping
- Claude successfully reviewed stock photos by reading image files — the capability exists, just not wired into the video pipeline
- Most feedback in the v1-v8 iteration cycle was structural (pacing, font, composition), not subjective (taste)
- The quality rule drafted mid-session (.claude/rules/video-production-quality.md) was never saved due to tool error — needs to be recreated
- Per-slide WPM analysis already exists in the review code — flagged slides 2, 7, 10 as issues

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-video-feedback-loop.md`
