---
type: event
created: 2026-03-24
status: active
date: 2026-03-24T15:07:07-04:00
author: claude
git_commit: e2e460b7
branch: main
repository: filescience
topic: "Audio Engine + Dual-Voice Caroline Video Handoff"
tags: [handoff, audio-engine, video, remotion, caroline, elevenlabs, dual-voice]
last_updated: 2026-03-24
last_updated_by: claude
---

# Handoff: Audio Engine Built, Dual-Voice Caroline Needs Multi-Speaker Architecture

## Task(s)

### Audio Engine — **completed**
Built intelligent audio production engine at `tools/audio-engine/` across 5 phases. Python package with FastMCP MCP server facade. 11 modules, 85 tests, 9 MCP tools. Merged to main via PRs #188 and #194.

### Dual-Voice Caroline Video — **in_progress**
Rendered Caroline V3 with Eric (narrator) + Lily (as Caroline). Voice pairing sounds good but **transitions between speakers are jarring** — needs multi-speaker architecture work and transition smoothing. Also need to shape how multi-speaker presentations work generally.

### Audio Feedback Systems — **planned**
Brief written at `memory-bank/thoughts/shared/briefs/2026-03-24-audio-feedback-systems.md`. Three tools needed: clip-level analysis (pre-render), rendered audio analysis (post-render), voice audition pipeline. Not yet implemented.

## Critical References

- **Audio engine plan:** `memory-bank/thoughts/shared/plans/2026-03-24-audio-engine.md`
- **Audio engine brief:** `memory-bank/thoughts/shared/briefs/2026-03-24-audio-engine.md`
- **Audio feedback brief:** `memory-bank/thoughts/shared/briefs/2026-03-24-audio-feedback-systems.md`
- **Caroline investor video brief:** `memory-bank/thoughts/shared/briefs/2026-03-23-caroline-investor-video.md`
- **Video SKILL.md:** `.claude/skills/video/SKILL.md`

## Recent changes

### Audio Engine (all on main)
- `75e84715` — Phase 1: package scaffolding, ElevenLabs TTS core, MCP server (#188)
- `00866e6f` — Phases 2-5: auto-timing, enhancement, music/SFX, orchestration (#194)

### Key files
- `tools/audio-engine/src/audio_engine/server.py` — FastMCP server with 9 tools
- `tools/audio-engine/src/audio_engine/elevenlabs.py` — Async ElevenLabs client (v3 + timestamps + audio tags)
- `tools/audio-engine/src/audio_engine/pipeline.py` — `produce_audio()` and `produce_scene_audio()` orchestrators
- `tools/audio-engine/src/audio_engine/processing.py` — Pedalboard DSP chain + pyloudnorm LUFS normalization
- `tools/audio-engine/src/audio_engine/timing.py` — Timestamps → word-level timings → durationInFrames
- `tools/audio-engine/src/audio_engine/takes.py` — Multi-take generation + LUFS-based selection
- `tools/audio-engine/src/audio_engine/music.py` — ElevenLabs Music Compose 3-tier API
- `tools/audio-engine/src/audio_engine/sfx.py` — SFX from text prompts
- `tools/audio-engine/src/audio_engine/pronunciation.py` — Pronunciation dictionary management
- `.mcp.json` — `audio_engine` server registered

### Local uncommitted changes
- `tools/audio-engine/src/audio_engine/processing.py` — Added true peak limiter (clamp to -1 dBTP after LUFS normalization) to fix clipping
- `tools/audio-engine/src/audio_engine/pipeline.py` — Added per-scene `voice_id` support for dual-voice narration + type annotation fixes

### Temp files (not committed)
- `/tmp/video/caroline-investor/scenes-v3-dual.json` — Dual-voice scenes (Eric narrator + Lily as Caroline)
- `/tmp/video/caroline-investor/caroline-v3-dual.mp4` — Latest render (~91s, 2741 frames, dual voice)
- `/tmp/video/caroline-investor/v3-dual/` — Generated audio clips
- `/tmp/video/caroline-investor/voice-audition/` — Voice audition clips

## Learnings

### Audio Engine Architecture
- **Form factor:** Python package + thin FastMCP facade. Same pattern as scout and workspace-mcp. Library importable from `render_video.py`, MCP tools for Claude sessions.
- **Remotion owns ducking, not the engine:** Engine outputs normalized individual tracks (narration at -16 LUFS). Remotion's `computeMusicVolume()` handles sidechain ducking at render time. No pre-mixing in Python — avoids double-ducking.
- **Per-scene voice_id:** Added `narration.voice_id` field for dual-voice support. Pipeline reads it and overrides the default voice per scene (`pipeline.py:_process_scene`).

### ElevenLabs v3 Details
- **v3 does NOT support `previous_request_ids`** — architectural incompatibility (holistic context planning vs autoregressive). Use consistent seed + Natural stability + crossfade smoothing instead.
- **Audio tags:** `[pause]`, `[excited]`, `[as if sharing a secret]`, `[warm]`, `[calm]` — in-text markup, case-insensitive, v3-only. Best responsiveness at stability 0.35-0.45.
- **Music compose:** 3-tier API — `/v1/music/create-composition-plan` (free) → adjust sections → `/v1/music/compose-detailed`. Music generation failed in our tests (likely subscription tier issue).
- **SFX:** `/v1/sound-generation`, duration range 0.5-30.0s (NOT 0.1).

### Voice Selection
- **Lily** (`pFZP5JQG7iQjIQuC4Bku`) — "Velvety Actress", chosen as Caroline's voice. Warm, expressive, responds well to v3 audio tags.
- **Eric** (`cjVigY5qzO86Huf0OWal`) — "Smooth, Trustworthy", chosen as narrator. Agentic tenor.
- The v3-curated "Gen3" voices from ElevenLabs Studio (Archer, Christopher, Mark) are NOT accessible via the API — Studio-only. We used the premade voices instead.

### Clipping Issue (partially fixed)
- pyloudnorm `normalize.loudness()` pushes past 0dBFS → clipping on nearly every clip
- **Fix applied locally (uncommitted):** True peak limiter after normalization — clamps to -1 dBTP (0.891 linear). pyloudnorm still warns internally but output is clean.
- Need to commit this fix.

### What's Broken / Needs Work
1. **Multi-speaker transitions are jarring** — abrupt voice switch between Eric and Lily. Need crossfade, room tone bridge, or transition design.
2. **No audio analysis tools yet** — Can't autonomously detect clipping, LUFS inconsistency, or volume pumping. Brief written but not implemented.
3. **Music generation failed** — ElevenLabs Music subscription may be required. Fallback to existing `music.mp3` from V1 or Meta MusicGen.
4. **Scene durations for iMessage scenes** — Auto-timing sets duration from narration length, but iMessage scenes need enough frames for all messages to appear. May need `max(narration_duration, message_animation_duration)`.

## Artifacts

### Code (on main)
- `tools/audio-engine/` — Complete audio engine (11 modules, 85 tests)
- `.mcp.json` — audio_engine server registered
- `.claude/skills/video/SKILL.md` — Updated with audio engine workflow (Step 2.75)

### Plans & briefs
- `memory-bank/thoughts/shared/plans/2026-03-24-audio-engine.md`
- `memory-bank/thoughts/shared/briefs/2026-03-24-audio-engine.md`
- `memory-bank/thoughts/shared/briefs/2026-03-24-audio-feedback-systems.md`

### Temp files
- `/tmp/video/caroline-investor/scenes-v3-dual.json` — Dual-voice scenes JSON
- `/tmp/video/caroline-investor/caroline-v3-dual.mp4` — Latest render
- `/tmp/video/caroline-investor/v3-dual/scenes-produced.json` — Audio engine output
- `/tmp/video/caroline-investor/voice-audition/*.mp3` — Voice comparison clips

## Action Items & Next Steps

### 1. Shape multi-speaker presentation architecture (highest priority)
The user wants to `/shape` how multi-speaker presentations work generally — not just Caroline. Questions:
- How should voice transitions feel? Crossfade? Silence gap? Room tone bridge?
- Should the scenes JSON encode speaker roles (narrator vs character)?
- How does the audio engine handle speaker-aware enhancement (different EQ for different voices)?
- Should the Remotion composition show speaker indicators?

### 2. Fix speaker transitions in Caroline
Apply whatever the shaping decides. The jarring transitions need:
- Possibly: 200-300ms crossfade at voice switch points
- Possibly: brief silence (150ms) + room tone between speakers
- Possibly: music bed that smooths transitions (currently no music in V3)

### 3. Commit local changes
The clipping fix (`processing.py` true peak limiter) and per-scene voice_id (`pipeline.py`) are uncommitted. Commit to main.

### 4. Build audio feedback systems
Implement the three analysis tools from the brief:
- `analyze_clips` — LUFS consistency, true peak detection, silence ratio
- `analyze_rendered_audio` — extract audio from MP4, check overall LUFS, detect volume jumps
- `audition_voices` — automated voice comparison with acoustic metrics

### 5. Add background music
Either fix ElevenLabs Music subscription or use existing `music.mp3` from V1. The sidechain ducking in Remotion handles the rest.

### 6. Fix iMessage scene duration mismatch
When narration is shorter than message animation time, the scene cuts off before all messages appear. Need `max(narration_frames, last_message_frame + buffer)` logic.

## Other Notes

### Render commands
```bash
# Full pipeline: generate audio + render
PYTHONPATH=. ELEVENLABS_API_KEY=$(grep ELEVENLABS_API_KEY .env | cut -d= -f2) uv run --project tools/audio-engine python -c "
import asyncio, json, sys
sys.path.insert(0, 'tools/audio-engine/src')
from audio_engine.pipeline import produce_audio
async def main():
    result = await produce_audio(
        scenes_json_path='/tmp/video/caroline-investor/scenes-v3-dual.json',
        output_dir='/tmp/video/caroline-investor/v3-dual/',
        voice_id='cjVigY5qzO86Huf0OWal',
        stability=0.35,
        enhance=True,
        generate_music_flag=False,
    )
    print(json.dumps(result, indent=2))
asyncio.run(main())
"

# Render from produced scenes
PYTHONPATH=. python scripts/render_video.py --scenes /tmp/video/caroline-investor/v3-dual/scenes-produced.json -o /tmp/video/caroline-investor/caroline-v3-dual.mp4

# Storyboard preview
PYTHONPATH=. python scripts/storyboard.py /tmp/video/caroline-investor/scenes-v3-dual.json
```

### Voice IDs
| Voice | ID | Role |
|-------|-----|------|
| Eric (narrator) | `cjVigY5qzO86Huf0OWal` | Story framing scenes |
| Lily (Caroline) | `pFZP5JQG7iQjIQuC4Bku` | Product demo scenes, closing |
| Chris (old default) | `iP95p4xoKVk53GoZ742B` | Deprecated for Caroline |

### Dual-voice scene assignment
| # | Type | Voice | Narration |
|---|------|-------|-----------|
| 0 | gradientMood | Eric | "You wake up. A text is already waiting." |
| 1 | wordCascade | Eric | "What if your phone actually knew you?" |
| 2 | imessage | Lily (Caroline) | "I noticed your electric bill bounced..." |
| 3 | gradientMood | (silent) | — |
| 4 | textReveal | Eric | "Forty-five minutes gone..." |
| 5 | imessage | Lily (Caroline) | "Osteria Mozza. March third..." |
| 6 | statsCard | Eric | "Four seconds. Zero clarification." |
| 7 | imessage | Lily (Caroline) | "I'll help you with that..." |
| 8 | statsCard | Eric | "Four books. Eighty-seven new words..." |
| 9 | wordCascade | Eric | "Calendar. Email. Messages..." |
| 10 | textReveal | Eric | "One interface. No new app." |
| 11 | endCard | Lily (Caroline) | "I'm Caroline. Not an assistant..." |

### ElevenLabs API key
In `.env` as `ELEVENLABS_API_KEY`. The audio engine reads it via `python-dotenv` in `server.py` or from the environment when called directly.
