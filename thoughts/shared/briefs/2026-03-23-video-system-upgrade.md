---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Video Production System Upgrade

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
The /video system produces visually monotonous output. The SKILL.md has no storytelling structure guidance, and the component library has only 3 scene types (iMessage, textReveal, staticImage). The Caroline investor video dogfood proved this: 4 back-to-back iMessage screens with no setup, no framing, no emotional build. Your mom wouldn't know what she's looking at.

## Constraints
- Remotion pipeline is live and working (Phase 1-3 of Remotion plan complete)
- 37 Remotion agent skills installed
- ElevenLabs API available for TTS, music, and SFX
- `react-device-mockup` available for proper iPhone frames
- `@remotion/noise`, `@remotion/light-leaks`, `@remotion/captions` available as packages
- No real photography/B-roll — all visuals must be synthesized in React/CSS

## Research Findings (5 parallel agents + /ground validation)

### Narrative Patterns (most critical)
- Apple rotates 3 shot types (hero/detail/lifestyle), never >2 same type consecutive
- YC: "demos seldom work" — investors buy vision, not features. Product appears as proof, not subject.
- "Her" never showed Samantha's UI. AI presence proven by human's reaction (Kuleshov effect).
- PAS in video: agitation = specific consequence, not louder problem statement
- Duarte Sparkline: oscillate "what is" / "what could be" 3x before resolution

### Grounded corrections (from /ground)
- **No magic number for visual register duration.** The "8-10s" claim was practitioner extrapolation, not research. Principle: "vary visual registers frequently" — don't enshrine a specific second count.
- **No fixed UI-to-context ratio.** The "40/60" number was a synthesis heuristic. Principle: "context before content, product as proof not subject."
- **3 video types, not 5.** Only Product Pitch and Internal Update have evidence of real need. "Explainer," "Culture/Brand," and "Demo Walkthrough" are speculative. Add types when actually needed.
- **Don't adopt remotion-ui.** 13 commits, dormant since Sep 2025, AI-generated. Writing 20-line equivalents is less work AND less risk. Cherry-pick patterns from their source for inspiration.
- **Film grain is selective, not universal.** Works on mood scenes and device mockups. Looks like noise on stat cards and dense text. Apply to ambient/cinematic scenes only.
- **Storyboard gate = narrative check, not visual preview.** Marp PDF from scenes.json is useful for "does the story flow?" but useless for "does it look right?" `renderStill()` per scene (~40s) is the visual preview tier.
- **Per-scene audio is correct, with a gotcha.** Remotion Studio shows a "waterfall" timeline with per-scene Sequences. Workaround: `showInTimeline={false}`. Document this.
- **Skill should use examples, not rules.** Per-type register rotation rules create a complex decision matrix. One set of universal principles + per-type examples showing what good looks like. Examples are easy to pattern-match; rules are hard to follow.

### Visual Variety (Tier 1 components to build from scratch)
- Animated Stats Card — rolling counters, ~30 lines, highest impact/effort ratio
- Word-Cascade Title Card — Apple WWDC style, spring-in words, ~40 lines
- End Card — logo + tagline + CTA, every video needs one
- Film Grain Overlay — wrapper component for mood/cinematic scenes only (not universal)
- Gradient Mood Scene — animated gradient breathing room between product scenes

### Audio Production
- Per-scene narration clips with `<Sequence>` is correct architecture (not concatenated)
- `previous_request_ids` chains maintain prosody across ElevenLabs clips
- Sidechain-style ducking: asymmetric ramps (fast attack 0.25s, slow release 0.8s)
- Remotion `volume` callback frame index starts from audio start, not composition start — gotcha
- Remotion Studio waterfall timeline with per-scene Sequences — use `showInTimeline={false}`

### Storyboard Preview Gate (two tiers)
- Tier 1: Marp PDF from scenes.json (~3s) — narrative flow review only
- Tier 2: `renderStill()` per scene (~40s for 10 scenes) — pixel-accurate visual review

### Remotion Ecosystem (use directly, don't wrap)
- `react-device-mockup` — iPhone 15 Pro Dynamic Island frame, pure React divs
- `@remotion/noise` — Perlin noise for organic camera drift
- `@remotion/light-leaks` — WebGL overlays at scene cuts via `<TransitionSeries.Overlay>`
- `@remotion/captions` — word-level synced captions from Whisper/ElevenLabs Scribe

## Video Types (grounded to actual need)

### Product Pitch (proven need — Caroline investor video)
- Audience: External (investors, prospects)
- Structure: Duarte Sparkline — oscillate problem/possibility, product as proof
- Principles: Context before content. Earn the right to show the product. Vary visual registers frequently.
- Example: Open with the world's problem → context card → one product moment → reaction → agitation → second product moment → stat → resolution → end card

### Internal Update (proven need — existing /video usage for artifact rendering)
- Audience: Internal (team, board, stakeholders)
- Structure: What changed → what we learned → what's next
- Principles: Data-forward. Stats cards and progress visualization. Faster pacing, less emotional arc.

### Artifact Walkthrough (proven need — /video's original purpose: render briefs/plans as video)
- Audience: Mixed (whoever the artifact targets)
- Structure: Follow the artifact's own structure, adapted for video pacing
- Principles: 3 bullets max per scene, promote visuals, suggest images for abstract concepts

## Chosen Approach
**C — Full Pipeline Upgrade (corrected).** Drop remotion-ui adoption. Write primitives from scratch (they're 20-30 lines each). Build 3 video types not 5. Use principles + examples in the skill, not per-type rule matrices. Two-tier storyboard preview. Per-scene audio with documented gotchas.

## Key Context Discovered During Shaping
- Remotion's `renderStill()` can render single frames in seconds — enables fast visual storyboard
- ElevenLabs `previous_request_ids` parameter is critical for natural prosody across per-scene clips
- Remotion's `volume` callback frame index starts from audio start, not composition start
- `@remotion/captions` + `createTikTokStyleCaptions()` enables word-level highlighting synced to narration
- Custom transition presentations are fully supported — any CSS/SVG effect can be a transition
- Remotion Studio `showInTimeline={false}` fixes the waterfall display with per-scene audio

## Next Step
Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-video-system-upgrade.md`
