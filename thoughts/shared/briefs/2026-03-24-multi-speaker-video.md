---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Multi-Speaker Video Presentation Architecture

**Date:** 2026-03-24
**Status:** Shaped -> Planning

## Problem
The dual-voice Caroline video (Eric narrator + Lily as Caroline) has jarring speaker transitions — one voice stops, the next starts immediately with zero audio or visual bridge. It sounds like two separate recordings spliced together. Professional multi-speaker content uses music beds, silence gaps, visual cues, and natural turn-taking to make voice switches feel seamless.

## Constraints
- Must work with existing Remotion composition and audio engine pipeline
- Music bed is the audio spine — it never stops between speakers
- No crossfading between voices (sounds weird — professional standard is clean cuts with room tone)
- Visual cues should be subtle (5-10% hue shift, not jarring color changes)
- ElevenLabs Text to Dialogue API only works within a single request, not across scenes
- Speaker identity needs to be a first-class concept in the scenes JSON schema

## Options Considered

### Music Bed + Silence Gaps (audio spine)
Background music runs continuously. 300-500ms silence gap filled with room tone at speaker switches. Music stays constant during gap. Documentary L-cut technique.
- Gains: Fixes 80% of jarring feel, zero visual changes needed
- Costs: No visual speaker identity
- Complexity: Low

### Speaker-Aware Composition (visual identity)
Add `speaker` field to scenes JSON. Remotion renders subtle background hue shifts per speaker + lower-third name/role on first appearance (3-5s slide-in animation).
- Gains: Viewer knows who's speaking before voice changes, primes brain for switch
- Costs: New Remotion component, schema extension
- Complexity: Medium

### ElevenLabs Dialogue API for Tight Transitions
Use `/v1/text-to-dialogue/with-timestamps` for scene pairs where narrator hands off to character. Single audio file with natural turn-taking. Individual clips for standalone scenes.
- Gains: Most natural transitions at handoff points
- Costs: Breaks per-scene timing model for dialogue blocks, needs new pipeline concept
- Complexity: High

## Chosen Approach
**All three as stacked layers** — not mutually exclusive:
1. Music bed as audio spine (foundation)
2. Speaker-aware visual composition (identity layer)
3. Dialogue API for tight narrator-to-character transitions (polish layer)

## Key Context Discovered During Shaping
- Professional standard: music bed never stops between speakers, 300-500ms silence gap filled with room tone, no crossfade between voices
- Apple keynotes use verbal bridge + 2-4s music sting between speakers
- Documentary L-cut: music/ambient continues uninterrupted across voice switch, acting as shared acoustic space
- Lower thirds: show name+role on first appearance only, 3-5s duration, slide-in animation
- Background hue shift per speaker should be barely perceptible (5-10%), not a full color change
- EQ/reverb matching: apply same short room reverb to both voices so they sound like shared space
- SFX lead by 2-4 frames ahead of visual action (brain processes audio faster)

### Voice assignments (Caroline video)
- **Eric** (`cjVigY5qzO86Huf0OWal`) — narrator, story framing scenes
- **Lily** (`pFZP5JQG7iQjIQuC4Bku`) — Caroline's voice, product demo + closing

### Existing infrastructure
- Audio engine at `tools/audio-engine/` with per-scene voice_id support (pipeline.py)
- True peak limiter added to prevent clipping (processing.py)
- Remotion TransitionSeries with fade transitions (VideoComposition.tsx)
- Per-scene narration audio rendering (VideoComposition.tsx:313-321)
- Sidechain music ducking (VideoComposition.tsx:92-144)

## Next Step
- [Plan] -> create plan via `.work/plans/` system
