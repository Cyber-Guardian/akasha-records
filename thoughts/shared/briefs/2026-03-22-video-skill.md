---
type: event
created: 2026-03-22
status: active
---
# Idea Brief: /video Skill — Video as an Output Modality

**Date:** 2026-03-22
**Status:** Shaped -> Planning

## Problem
Our brainstorming-to-artifact pipeline terminates at static PDF. Video is more persuasive across every audience we care about (B2B prospects, B2C consumers, internal team, C-suite), but we have no way to output video from the same human+AI brainstorming loop. The tools to generate video programmatically now exist and are mature. We need a `/video` skill that works exactly like `/deck`: brainstorm together, then render to MP4.

## Constraints
- Must fit the existing flow: brainstorm -> shape artifact -> render (not a separate production workflow)
- Python-native preferred (matches our stack), Node.js acceptable (already used for Marp)
- Local-first: core path should not depend on paid cloud APIs (AI voiceover/images are optional enhancements)
- Brand consistency: FileScience purple/rose theme must carry through like it does in `/deck`
- FFmpeg 8.0.1, ImageMagick 7.1.1, Node.js already installed locally
- Common intermediate format needed so future modalities (wiki video embeds) can reuse the same pipeline

## Options Considered

### Animated Deck (extend /deck to video)
Marp already renders slides as images. FFmpeg assembles them with transitions (crossfade, push, wipe), optional Ken Burns, and optional audio. Reuses all existing deck infrastructure.
- Gains: Fastest to ship, zero new deps, every deck becomes a potential video
- Costs: Limited to "slides that move" — no true motion graphics
- Complexity: Low

### Python Video Engine (MoviePy v2 + FFmpeg)
Build `render_video.py` alongside `render_deck.py`. Claude generates MoviePy code from the brief — images, text overlays, Ken Burns, transitions, audio mixing. Can handle both AI-generated and real content.
- Gains: Python-native, rich compositing, handles generated and real content
- Costs: New dependency (moviepy), more complex rendering logic
- Complexity: Medium

### Remotion Motion Graphics
React/TypeScript components rendered to MP4. Official Claude Code Agent Skills. Highest quality ceiling — spring animations, data viz, branded motion graphics. Superpowers ecosystem adds voiceover + stock footage.
- Gains: Highest quality, declarative, version-controllable, official Claude integration
- Costs: React/TS ecosystem (different from Python stack), heavier setup, commercial license considerations
- Complexity: High

### Hybrid Pipeline (incremental layers)
Build layers progressively. Layer 1: Animated Deck (quick, always available). Layer 2: MoviePy engine for richer compositing. Layer 3: Remotion, Manim, or AI video APIs as optional enhancers. Common scene description format decouples content from rendering backend.
- Gains: Covers full spectrum, progressive complexity, each layer independently useful
- Costs: More infrastructure over time, layer boundaries need definition
- Complexity: Medium (incremental) to High (all at once)

## Chosen Approach
**Hybrid Pipeline (incremental)** — Start with Layer 1 (Animated Deck, near-zero marginal cost), add MoviePy when the ceiling is hit, then specialized backends (Remotion/Manim/AI APIs) for specific use cases. The key architectural decision is defining a common intermediate scene format that any backend can consume.

## Key Context Discovered During Shaping
- Anthropic's own growth team uses Claude + FFmpeg for video ads (30 min -> 30 sec per variation)
- Paul Dervan documented a working Claude + Kling + FFmpeg pipeline at <1 euro per finished ad
- Manim MCP (572 stars) and Remotion (official Agent Skills) are the two most mature video MCPs
- MoviePy v2 has breaking API changes from v1 (method renames, import path changes)
- FFmpeg `xfade` filter handles crossfade transitions cleanly; `zoompan` does Ken Burns
- ImageMagick can generate slide frames from scratch via CLI (text, backgrounds, compositing)
- Video as a wiki modality is a natural future extension (parked as separate Linear issue)

## Ecosystem Reference
| Tool | Type | Best for |
|------|------|----------|
| FFmpeg (local) | CLI | Assembly, transitions, Ken Burns, audio mix |
| ImageMagick (local) | CLI | Frame generation, text rendering |
| MoviePy v2 | Python lib | Compositing, timing, Ken Burns, captions |
| Remotion | React/TS | Motion graphics, data viz, animations |
| Manim MCP | Python MCP | 3Blue1Brown-style explainers |
| Kling AI MCP | Cloud API | Image-to-motion clips |
| InVideo MCP | Cloud SaaS | Text-to-video with stock footage |
| FFmpeg MCPs | MCP wrappers | Editing/post-processing existing footage |

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-22-video-skill.md`
- Wiki video modality parked as separate Linear issue
