---
type: event
created: 2026-03-24
status: active
date: 2026-03-24T11:39:44-04:00
author: claude
git_commit: bea7d05a9c89869f42a53913c3bb3f50cc6f88bc
branch: main
repository: filescience
topic: "Caroline Investor Video — Audio Sync + Story Cohesion"
tags: [handoff, video, remotion, caroline, audio, elevenlabs, storytelling]
last_updated: 2026-03-24
last_updated_by: claude
---

# Handoff: Caroline V2 — Audio Timing & Story Cohesion Polish

## Task(s)

### Caroline V2 video iteration — **in_progress**
The video system upgrade is complete (6 phases shipped). Caroline V2 renders at 84.5s with 12 scenes, 7 visual registers, and narration. But the audio is a single concatenated track — it needs per-scene sync. And the storytelling needs iteration passes watching the actual video.

### Specific work remaining:
1. **Per-scene audio sync** — wire the 11 existing narration clips into the scenes JSON as per-scene `narration` fields instead of one global track. The infrastructure is built (Phase 4) but not used yet for Caroline.
2. **Story cohesion** — watch the video, identify timing mismatches between narration and visuals, adjust scene durations and message frame timings.
3. **Narration re-generation** — some clips may need re-recording after timing adjustments. Use `previous_request_ids` chaining for prosody continuity.
4. **Background music** — not yet added. ElevenLabs Music Compose API available (`/v1/music/compose`) or manual music selection.

## Critical References

- **Video system upgrade plan (archived):** `memory-bank/thoughts/shared/plans/2026-03-23-video-system-upgrade.md`
- **Caroline V2 scenes JSON:** `/tmp/video/caroline-investor/scenes-v2.json`
- **Caroline brief:** `memory-bank/thoughts/shared/briefs/2026-03-23-caroline-investor-video.md`
- **Video system brief (with grounding corrections):** `memory-bank/thoughts/shared/briefs/2026-03-23-video-system-upgrade.md`
- **SKILL.md:** `.claude/skills/video/SKILL.md` — storytelling-driven, 3 video types, full scene reference

## Recent changes

### Video system (all on main, all pushed)
- `dde6a326` — SKILL.md storytelling rewrite (Phase 1)
- `5e818cc2` — Tier 1 components: StatsCard, WordCascade, EndCard, GradientMood (Phase 2)
- `c27c29fc` — Ecosystem: DeviceMockup, FilmGrain (noise3D), CameraDrift, LightLeaks, unified TransitionSeries (Phase 3)
- `b849c35b` — Per-scene audio: Sequence-based clips, asymmetric sidechain ducking (Phase 4)
- `e00760e7` — Storyboard preview: `scripts/storyboard.py` + `render.mjs --stills` (Phase 5)
- `bea7d05a` — Plan archived, handoffs written

### Caroline V2 video (in /tmp, not committed)
- `/tmp/video/caroline-investor/caroline-v2.mp4` — current render (84.5s, narrated, 12 scenes)
- `/tmp/video/caroline-investor/scenes-v2.json` — scene definitions
- `/tmp/video/caroline-investor/audio/v2_01.mp3` through `v2_12.mp3` — per-scene narration clips (no v2_04 — scene 4 is a silent mood scene)
- `/tmp/video/caroline-investor/audio/v2_narration_full.mp3` — concatenated narration (48.9s)
- `/tmp/video/caroline-investor/storyboard-v2.pdf` — narrative storyboard PDF
- `/tmp/video/caroline-investor/review-v2/` — extracted key frames

## Learnings

### Architecture (what's built and working)
- **8 scene types:** imessage, textReveal, staticImage, statsCard, wordCascade, endCard, gradientMood, deviceMockup
- **Composition effects:** filmGrain (noise3D overlay), cameraDrift (noise3D translate wrapper)
- **Transitions:** Unified TransitionSeries with `none()` default. Light leaks via `<TransitionSeries.Overlay>`. Fade/slide/wipe available.
- **Audio:** Per-scene `narration` field on every scene type. `<Audio>` inside `<Sequence showInTimeline={false}>`. Asymmetric sidechain ducking (8-frame attack, 24-frame release). Global narration/music still supported as fallback.
- **Preview:** `scripts/storyboard.py` generates Marp PDF from scenes.json (~3s). `render.mjs --stills` renders one PNG per scene (~40s for 10 scenes).
- **Render pipeline:** `render_video.py --scenes scenes.json` → Remotion renderMedia() → ffmpeg post-process (yuv420p + faststart) → QuickTime-compatible MP4.

### Caroline V2 storyboard (the "kind coach" angle)
The video was reshaped from "she sees your darkness" to "she helps you grow." Key narrative beats:

| # | Type | Beat | Narration |
|---|------|------|-----------|
| 1 | gradientMood | Opening atmosphere | "You wake up. A text is already waiting." |
| 2 | wordCascade | Hook question | "What if your phone actually knew you?" |
| 3 | imessage | Proof #1: Morning brief | "Not a notification. A brief..." |
| 4 | gradientMood | Breathing room | (silent) |
| 5 | textReveal | Agitation: cost of fragmentation | "45 minutes gone. 15 apps. Nobody connecting the dots." |
| 6 | imessage | Proof #2: Instant recall | "What was that restaurant? Four seconds." |
| 7 | statsCard | Data proof | "Four seconds. Zero clarification." |
| 8 | imessage | Proof #3: Kind coach | "New year's resolution: read more" → word of the day → "4 books this quarter" |
| 9 | statsCard | Growth data | 4 books / 87 words / 12-day streak |
| 10 | wordCascade | Vision cascade | "Calendar. Email. Messages..." |
| 11 | textReveal | Resolution | "One interface. No new app. Just text." |
| 12 | endCard | Close | "Caroline. The first product from Akasha." |

### ElevenLabs details
- Voice: Chris (`iP95p4xoKVk53GoZ742B`) — American, casual, conversational
- Model: `eleven_multilingual_v2`
- Stability: 0.38-0.42, similarity: 0.75, style: 0.3-0.4
- API key in `.env` as `ELEVENLABS_API_KEY`
- `previous_request_ids` available for prosody chaining but NOT used in current clips (each generated independently)
- Music Compose API: `/v1/music/compose` with `force_instrumental: True`, requires Music subscription
- SFX: `/v1/sound-generation` for UI sounds (notification dings, typing)

### Key gotchas
- Remotion outputs `yuvj420p` — post-process in `render.mjs` fixes to `yuv420p + faststart` for QuickTime
- `volume` callback `f` parameter counts from audio start, not composition start
- `showInTimeline={false}` on per-scene audio Sequences prevents waterfall in Remotion Studio
- Film grain uses `noise3D()` from `@remotion/noise` — NOT static PNG (that produces tiling artifacts)
- `react-device-mockup` is installed but not used — DeviceMockup is a custom CSS component instead

## Artifacts

### Code (all on main)
- `tools/remotion/src/types.ts` — 8 scene types + SceneNarration + FilmGrainConfig + CameraDriftConfig
- `tools/remotion/src/VideoComposition.tsx` — unified TransitionSeries, per-scene audio, sidechain ducking
- `tools/remotion/src/components/` — IMessageChat, TextReveal, StaticImageScene, StatsCard, WordCascade, EndCard, GradientMood, DeviceMockup, FilmGrain, CameraDrift, TypingIndicator
- `tools/remotion/render.mjs` — renderMedia + renderStill + ffmpeg post-process
- `tools/remotion/src/Root.tsx` — composition registration with calculateMetadata
- `scripts/render_video.py` — Python orchestrator (Remotion-only, per-scene audio resolution)
- `scripts/review_video.py` — quality review pipeline
- `scripts/storyboard.py` — narrative PDF from scenes.json
- `scripts/render_deck.py` — extracted `render_marp_to_pdf()` helper
- `.claude/skills/video/SKILL.md` — storytelling-driven skill (3 types, 5 principles, full reference)

### Plans & briefs
- `memory-bank/thoughts/shared/plans/2026-03-23-video-system-upgrade.md` (archived)
- `memory-bank/thoughts/shared/plans/2026-03-23-remotion-video-layer.md` (archived)
- `memory-bank/thoughts/shared/briefs/2026-03-23-video-system-upgrade.md`
- `memory-bank/thoughts/shared/briefs/2026-03-23-caroline-investor-video.md`

### Temp files (in /tmp, not committed)
- `/tmp/video/caroline-investor/scenes-v2.json`
- `/tmp/video/caroline-investor/caroline-v2.mp4`
- `/tmp/video/caroline-investor/audio/v2_*.mp3` (11 clips)
- `/tmp/video/caroline-investor/storyboard-v2.pdf`
- `/tmp/video/caroline-investor/review-v2/frame_*.jpg`

## Action Items & Next Steps

### 1. Per-scene audio sync (highest priority)
Wire the 11 existing narration clips into `scenes-v2.json` as per-scene `narration` fields:
```json
{
  "type": "gradientMood",
  "durationInFrames": 150,
  "narration": {"src": "/tmp/video/caroline-investor/audio/v2_01.mp3", "volume": 1.0},
  ...
}
```
Remove the global `--audio` flag from the render command. This gives per-scene sync control.

### 2. Watch and iterate timing
Play the video and note where narration doesn't match visuals:
- Do messages appear before/after the narrator mentions them?
- Are mood scenes long enough to breathe, or too long?
- Does the stats card hold long enough to read the numbers?
- Does the coach scene (8) feel rushed or spacious?

Adjust `durationInFrames` and message `frame` values in scenes-v2.json accordingly.

### 3. Narration re-generation (if needed)
If scene durations change significantly, regenerate specific clips:
- Use `previous_request_ids` chaining: generate scene 1 first, pass its request ID to scene 2, etc.
- This maintains prosody continuity across the full narration arc
- Cost: minimal (~$0.01 per clip at standard rates)

### 4. Background music
Options:
- ElevenLabs Music Compose: `POST /v1/music/compose` with `force_instrumental: True`, `positive_global_styles: ["ambient electronic", "optimistic", "instrumental"]`
- Manual: find a royalty-free ambient track and add as `"music": {"src": "bg.mp3", "volume": 0.12, "loop": true}`
- The sidechain ducking (Phase 4) will automatically duck music under narration

### 5. Final polish
- Add SFX (notification dings for iMessage scenes) via ElevenLabs `/v1/sound-generation`
- Consider light leak transitions between sections (set `transition.overlay: "lightLeak"`)
- Run storyboard preview (`scripts/storyboard.py`) after any structural changes
- Run `render.mjs --stills` for visual spot-check before full render

### 6. Compare V1 vs V2
V1 is at `/tmp/video/caroline-investor/caroline-final.mp4` (67.5s, 5 scenes, flat). V2 is at `caroline-v2.mp4` (84.5s, 12 scenes, story arc). Side-by-side comparison validates the upgrade.

## Other Notes

### Render commands
```bash
# Full render
PYTHONPATH=. python scripts/render_video.py --scenes /tmp/video/caroline-investor/scenes-v2.json -o /tmp/video/caroline-investor/caroline-v2.mp4

# With global audio (current approach)
PYTHONPATH=. python scripts/render_video.py --scenes /tmp/video/caroline-investor/scenes-v2.json --audio /tmp/video/caroline-investor/audio/v2_narration_full.mp3 -o /tmp/video/caroline-investor/caroline-v2.mp4

# Storyboard preview
PYTHONPATH=. python scripts/storyboard.py /tmp/video/caroline-investor/scenes-v2.json

# Visual stills
node tools/remotion/render.mjs --stills --props /tmp/video/caroline-investor/scenes-v2.json --output /tmp/video/caroline-investor/stills/
```

### Narration clip durations
| Clip | Scene | Duration |
|------|-------|----------|
| v2_01 | Mood opening | 4.1s |
| v2_02 | Hook question | 1.8s |
| v2_03 | Morning brief | 7.3s |
| (none) | Breathing room | — |
| v2_05 | Agitation | 8.2s |
| v2_06 | Restaurant | 5.9s |
| v2_07 | Stats | 1.8s |
| v2_08 | Kind coach | 4.1s |
| v2_09 | Growth stats | 2.6s |
| v2_10 | Vision list | 3.5s |
| v2_11 | Resolution | 2.0s |
| v2_12 | End card | 7.6s |
| **Total** | | **48.9s** |

### Video scene durations (frames at 30fps)
| # | Type | Frames | Seconds |
|---|------|--------|---------|
| 1 | gradientMood | 150 | 5.0 |
| 2 | wordCascade | 120 | 4.0 |
| 3 | imessage | 540 | 18.0 |
| 4 | gradientMood | 90 | 3.0 |
| 5 | textReveal | 180 | 6.0 |
| 6 | imessage | 390 | 13.0 |
| 7 | statsCard | 120 | 4.0 |
| 8 | imessage | 480 | 16.0 |
| 9 | statsCard | 120 | 4.0 |
| 10 | wordCascade | 180 | 6.0 |
| 11 | textReveal | 150 | 5.0 |
| 12 | endCard | 180 | 6.0 |
| | **Total** | **2700** | **~84.5s** (with transition overlaps) |
