---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Audio Feedback Systems for Autonomous Review

**Date:** 2026-03-24
**Status:** Shaped -> Implementing

## Problem
Claude is blind to audio quality. The current pipeline generates clips, renders video, and presents it — with zero ability to detect clipping, volume pumping, LUFS inconsistency, or voice quality issues before the user watches. The V3 Caroline render had audible volume pumping caused by LUFS normalization pushing past 0dBFS (pyloudnorm warned "Possible clipped samples" on nearly every clip). The voice (Chris) also sounds flat/dated compared to newer v3-optimized voices.

## Constraints
- Analysis must be automated — Claude runs it without user intervention
- Must work with the existing audio engine pipeline (tools/audio-engine/)
- Should not add more than ~10s to the production pipeline
- Voice audition is a per-project one-time cost, not per-render
- Need both pre-render gates (catch before wasting render time) and post-render verification

## Options Considered

### Clip-Level Analysis (pre-render gate)
Automated audio analysis on each generated clip before rendering. Metrics: integrated LUFS, true peak (clipping), LUFS consistency across clips, silence ratio, duration validation. Implemented as `analyze_clips` tool.
- Gains: Catches the exact V3 clipping problem. Prevents wasted renders.
- Costs: ~5s analysis per clip
- Complexity: Low

### Rendered Video Audio Extraction + Analysis (post-render gate)
Extract audio from rendered MP4, analyze: overall LUFS, dynamic range, sudden volume changes (>6dB in 100ms = ducking artifact), spectral balance. Implemented as `analyze_rendered_audio` tool.
- Gains: Catches Remotion mixing artifacts, ducking problems, overall loudness
- Costs: Requires render to complete first
- Complexity: Medium

### Voice Audition Pipeline
Generate same text with N voices, analyze for warmth metrics (pitch variance, speaking rate, dynamic range), present comparison. Implemented as `audition_voices` tool.
- Gains: Data-driven voice selection. Catches flat/robotic voices.
- Costs: N API calls per audition
- Complexity: Medium

## Chosen Approach
**All three as a layered feedback system** — they're checkpoints at different stages, not alternatives:

1. Voice audition (one-time per project setup)
2. Clip-level analysis (after generation + after enhancement)
3. Rendered audio analysis (after final render)

Pipeline: generate → clip analysis → fix clipping → enhance → clip analysis → render → rendered audio analysis → present

## Key Context Discovered During Shaping
- V3 clipping caused by `pyloudnorm.normalize.loudness()` pushing past 0dBFS — true peak detection would catch this instantly
- Chris voice (`iP95p4xoKVk53GoZ742B`) is v2-era, not optimized for v3 audio tags
- ElevenLabs v3 has newer voices designed for its holistic context architecture
- Acoustic metrics (pitch variance via librosa, dynamic range, speaking rate) can proxy for "warmth" and "expressiveness" without human listening
- The existing `review_video.py` checks audio presence but not audio quality

## Next Step
- [Implementing] -> extend the audio engine with three new tools, then re-render Caroline with better voice + quality gates
