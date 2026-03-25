---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Audio Perception Layer

**Date:** 2026-03-24
**Status:** Shaped -> Planning

## Problem
Claude produces audio/video but can't perceive either modality. All feedback is numeric proxies (LUFS, frame counts) that miss perceptual issues like "fade starts too soon" or "transition feels jarring." Need sensory translation layers that convert audio/video into representations Claude can reason about — waveform images, volume timelines, word timestamps, spectrograms.

## Constraints
- Must work within existing audio engine MCP server pattern
- No multimodal video model reviewer
- WhisperX adds wav2vec2 dependency (heavier than other tools)
- Analysis must add <15s total to pipeline
- Must produce artifacts Claude can read: images, numbers, text

## Options Considered

### Checkpoint Pipeline (sequential gates)
Analysis at fixed checkpoints with automatic gating. Linear, stops on failure.
- Gains: Simple, clear failure points
- Costs: Sequential, can't skip ahead
- Complexity: Low

### Analysis Sidecar (parallel observer)
Standalone analyze tool, not coupled to pipeline. Claude calls it when wanted.
- Gains: Flexible, composable, works on any audio/video
- Costs: Claude has to remember to call it
- Complexity: Medium

### Integrated Perception Layer (always-on)
Full analysis wired into pipeline at both checkpoints AND into /video skill. Claude never presents without having "seen" and "heard" first.
- Gains: Always-on perception, catches issues before user does
- Costs: +10-15s per pipeline run, heavier dependencies
- Complexity: High

## Chosen Approach
**Integrated Perception Layer** — full kitchen sink, always running. Every render goes through analysis at both pre-render (per-clip) and post-render (rendered MP4) checkpoints.

## Key Context Discovered During Shaping

### Grounded findings
- ffmpeg `astats=metadata=1:reset=1` confirmed correct for per-frame volume (Peak_level, RMS_level, Crest_factor)
- Whisper native word timestamps are unreliable — need WhisperX (wav2vec2 forced alignment) for production accuracy
- VLMs reading spectrograms: GPT-4o hit 73.75% on 10-class sound classification (beat human experts). Works for coarse checks.
- Professional voiceover fade standard: 0.5-1.5s. Current code has 2.5s linear — confirmed too aggressive.
- Mel spectrogram preferred over linear for voice/music discrimination
- No off-the-shelf "word cutoff" detector — must compose ASR timestamps + signal amplitude check

### Current state (from codebase analysis)
- `review_video.py:155-183` already extracts key frames at scene midpoints
- `processing.py:66-67` has `fade_out_s: float = 2.5` — needs fixing to 0.8s + logarithmic curve
- `takes.py:27-52` has `measure_lufs` via ffmpeg ebur128 — only tool that analyzes audio today
- No waveform, spectrogram, or per-frame volume analysis exists anywhere

### Five analysis channels
1. **astats per-frame volume** — numeric, ~2s, catches pumping/fading/ducking
2. **Waveform images** — visual, ~1s/clip, catches fade shape + clipping
3. **WhisperX transcription** — text + timestamps, ~7-10s, catches word cutoffs
4. **Mel spectrogram images** — visual, ~2s/clip, catches voice/music balance
5. **Extracted video frames** — visual, already built, catches visual composition

## Next Step
- [Plan] -> create plan via `.work/plans/`
