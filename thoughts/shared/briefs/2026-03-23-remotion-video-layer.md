---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Remotion as Video Production Layer

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
The /video skill produces narrated slideshows — static screenshots with FFmpeg crossfade transitions. Real product demo videos need animation: messages appearing one by one with typing indicators, dynamic text reveals, smooth scene transitions. The Caroline investor video proved this: good script + good narration + static mockups = bad output.

## Constraints
- Remotion renders React components to video via Chrome Headless Shell — anything in HTML/CSS/React becomes a video frame
- Official Claude Code agent skills (37 skills including messaging/chat with iMessage styling)
- Already uses FFmpeg internally for final encoding
- Node.js already installed (for Marp CLI)
- Free for teams of 3 or fewer
- 2-5 min render time for 90s animation on M2
- ElevenLabs narration pipeline works well — keep it
- Self-review pipeline (review_video.py) works — keep it
- /deck stays on Marp for PDF — only /video changes

## Options Considered

### Option A: Replace FFmpeg assembly with Remotion (CHOSEN)
Keep ElevenLabs, review pipeline, Marp for decks. Replace only the video assembly layer. FFmpeg slideshow stays as "quick mode" fallback.
- Gains: Dynamic animation, animated iMessage mockups, real transitions, frame-accurate audio sync
- Costs: New dependency (Remotion + Chrome Headless), React compositions needed
- Complexity: Medium

### Option B: Remotion as full video layer
Remotion replaces both Marp PNG export AND FFmpeg assembly. Slides are React components.
- Gains: Maximum flexibility, no intermediate PNG step
- Costs: Breaks "sibling output" model — /deck and /video can't share storyboards
- Complexity: High

### Option C: Hybrid — Remotion for hero scenes, FFmpeg for simple slides
Use Remotion only for animated scenes, FFmpeg for static slides, stitch together.
- Gains: Animation where it matters, fast render for simple content
- Costs: Two rendering backends to coordinate
- Complexity: Medium-High

## Chosen Approach
**Option A** — Replace the assembly layer, keep everything else. The "sibling output" model still works: /deck produces PDF via Marp, /video produces MP4 via Remotion. They share the script/narrative, not the rendering format.

## Key Context Discovered During Shaping
- Remotion has official `@remotion/skills` package with 37 agent skills for Claude Code — includes messaging/chat skill with iMessage/WhatsApp bubble layout and staggered entrances
- `npx remotion render --props=scenes.json` accepts JSON scene descriptions — Claude writes JSON, Remotion renders
- The `template-prompt-to-motion-graphics-saas` official template has a Messaging skill covering "Chat UI — bubble layouts, WhatsApp/iMessage styling, staggered entrances"
- Spring physics (`spring()`) and interpolation (`interpolate()`) are the core animation primitives
- `<Sequence>` component provides frame-accurate audio/visual sync
- Static PNG/JPG import via `staticFile()` — existing screenshots can be mixed with animated scenes
- Known issue: AI-generated compositions sometimes have timing mismatches with audio. Fix: generate audio first, measure duration, then build scenes — matches our existing ElevenLabs pipeline
- Render pipeline: React component tree → Chrome Headless Shell (parallel frame capture) → FFmpeg stitches → MP4

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-remotion-video-layer.md`
