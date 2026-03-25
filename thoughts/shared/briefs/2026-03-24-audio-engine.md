---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Intelligent Audio Engine

**Date:** 2026-03-24
**Status:** Shaped -> Planning

## Problem
Our video system's rendering layer is production-grade (per-scene audio placement, sidechain ducking, fade envelopes in Remotion) but the audio generation layer is entirely manual. Every video session requires hand-driven ElevenLabs API calls, manual duration math, and file wiring — 30-60 minutes of repetitive work that produces inconsistent results. The engine also serves as foundational audio infrastructure for Caroline's future vocal modality.

## Constraints
- ElevenLabs v3 does NOT support `previous_request_ids` stitching (architectural incompatibility — holistic context planning vs autoregressive). Use v3 + Natural stability + crossfade smoothing. Fall back to v2 with stitching for consistency-critical tracks.
- Timestamps TTS (`/with-timestamps`) returns character-level timing — can auto-compute exact clip durations and `durationInFrames`
- Music compose has 3 tiers: prompt -> plan (free) -> detailed section control
- SFX generation caps at 30s per call
- Remotion is a presentation layer — heavy mixing happens pre-render in Python
- Form factor: Python package + thin FastMCP server facade (same pattern as workspace-mcp and scout)
- Pronunciation alias rules work with all models including v3; phoneme rules only v2/flash/turbo

## Options Considered

### Script-Level Automation
Automate only the manual steps: ElevenLabs API calls, duration computation, file wiring. No mixing, enhancement, or intelligence.
- Gains: Fast to build, immediately useful
- Costs: Audio stays at raw TTS quality — no mastering, no professional signal chain
- Complexity: Low

### Production-Grade Audio Engine
Full 7-stage pipeline (generation, enhancement, music, SFX, mix, master, deliver) with professional signal chain. But reactive — you tell it what to generate.
- Gains: Professional output quality, consistent mastering, multi-take support
- Costs: Heavier build, more dependencies (Pedalboard, pyloudnorm, soundfile)
- Complexity: Medium

### Intelligent Audio Engine
Everything in Production-Grade, plus the engine understands the video: auto-timing from timestamps API, adaptive music scoring, video-to-SFX via MMAudio, take selection via audio analysis, frequency-aware 3-layer ducking.
- Gains: SOTA. Duration becomes a derived property. Claude shifts from "audio technician" to "creative director"
- Costs: Heaviest build. MMAudio needs GPU. 3-layer ducking requires custom DSP.
- Complexity: High

## Chosen Approach
**Intelligent Audio Engine** — SOTA quality regardless of cost. The engine receives scenes JSON with narration text and returns a fully-produced audio layer with correct timing. Immediate use is the /video production pipeline; also serves as foundational audio infrastructure for Caroline's vocal modality.

## Key Context Discovered During Shaping

### ElevenLabs v3 Architecture
- v3 is holistic context planning, not autoregressive — fundamentally incompatible with request stitching
- Audio tags (`[excited]`, `[whispers]`, `[pause]`) are v3-only, in-text markup, case-insensitive
- Stability 0.35-0.45 for maximum tag responsiveness, 0.5-0.6 for cross-clip consistency
- Text to Dialogue API is v3's answer to multi-speaker coherence (within a single request)
- Crossfade at clip boundaries (100-200ms at sentence breaks) + music sidechaining masks tonal drift

### Production Pipeline (SOTA reference)
- Professional signal chain: HPF 80-100Hz -> surgical EQ -> dual-stage de-esser -> multi-stage compression -> LUFS normalization
- 3-layer ducking: spectral carve at 2kHz (dynamic EQ sidechain), mid/side sub-bass duck, look-ahead pre-duck (VO shifted 2s early)
- LUFS targets: narration -16, music bed -26, SFX -18, final mix -14 (YouTube/social)
- Lead SFX placement by 2-4 frames ahead of visual action (brain processes audio faster)
- MMAudio (Sony, CVPR 2025): video-to-SFX synthesis, MIT license, 1.23s per 8s clip

### Technology Stack
- **Spotify Pedalboard**: JUCE-backed DSP, 300x faster than pySoX, reverb/compression/EQ
- **pyloudnorm**: ITU-R BS.1770-4 LUFS normalization, pure numpy/scipy
- **librosa**: Beat detection for music-cut sync
- **Resemble Enhance**: AI super-resolution, upscales 8kHz to 48kHz, MIT license
- **MMAudio**: Video-to-audio synthesis, dual visual encoder (CLIP + Synchformer)
- **Cartesia Sonic-3**: Alternative TTS worth evaluating (preferred over ElevenLabs in some blind tests)
- **Beatoven.ai / ElevenLabs Music**: Adaptive scoring APIs

### Existing Infrastructure (from codebase gap analysis)
- Per-scene audio rendering: automated (`VideoComposition.tsx:313-321`)
- Sidechain music ducking: automated (`VideoComposition.tsx:92-144`)
- Per-scene audio file resolution: automated (`render_video.py:199-232`)
- Narration fade-in/out: NOT automated (scalar volume, not callback — `VideoComposition.tsx:319`)
- ElevenLabs TTS/music/SFX generation: entirely manual
- Duration computation from clip lengths: entirely manual
- Multi-take generation and selection: entirely manual
- Pronunciation dictionary management: entirely manual

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-24-audio-engine.md`
