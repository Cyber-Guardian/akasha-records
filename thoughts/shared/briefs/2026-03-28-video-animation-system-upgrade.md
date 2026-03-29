---
type: event
created: 2026-03-28
status: active
---
# Idea Brief: Video Animation System Upgrade

**Date:** 2026-03-28
**Status:** Shaped → Planning

## Problem
The Remotion video system has sophisticated audio (sidechain ducking, multi-speaker, SFX) but visually is limited to 8 hardcoded scene types with basic spring/fade animations. Every component uses `{damping: 200}` (overdamped, no personality), inline styles, and a rigid JSON discriminated union that caps the visual ceiling at whatever scene types exist. The agent has no flexibility to create novel visuals, and no feedback loop to judge output quality.

## Constraints
- Remotion 4.0 frame-based rendering — all animation must use `useCurrentFrame()`, never wall-clock time
- Framer-motion and react-spring are incompatible (time-based)
- AI agent is the primary author, not a human editor
- Must maintain existing audio engine integration (sidechain ducking, per-scene narration, SFX)
- Official Remotion packages available: `@remotion/tailwind-v4`, `@remotion/motion-blur`, `@remotion/paths`, `@remotion/rive`, `@remotion/three`, `@remotion/lottie`
- Community libs available: `remotion-animated`, `remotion-bits` (AnimatedText), `remotion-glitch-effect`
- GSAP now free (SplitText, DrawSVG, MorphSVG) but time-based caveat
- JSON Render pattern (catalog + registry + schema) is the ecosystem's answer to reliable AI video
- Remotion's own AI strategy is skills-based (30+ rule files)

## Options Considered

### Pure Code Generation
Agent writes full TSX compositions importing from a component library.
- Gains: Maximum flexibility, any Remotion feature accessible
- Costs: Fragile, hallucination-prone, context rot as prompts grow. Remotion's own docs warn against this as primary mode.
- Complexity: High

### JSON Scene Expansion (more scene types, same architecture)
Add 5 new scene types to existing discriminated union, tune springs, add Tailwind.
- Gains: Reliable, backwards-compatible, well-tested pattern
- Costs: Ceiling stays at whatever scene types exist. No creative flexibility.
- Complexity: Low

### Four-Pillar System with Phase-Based Animation (chosen)
Component library with phase-based animation model + Remotion skill system + narrow code gen escape hatch + vision review MCP with budget pre-validation.
- Gains: Reliable fast path for 80% of scenes, creative escape hatch for 20%, quality feedback loop, scalable constraint system
- Costs: More architectural complexity than pure JSON expansion. Vision review MCP is novel territory.
- Complexity: Medium-High

### Constraint-Driven Budget Algebra (from ideation engine)
Formal animation budget system where each component has a complexity token cost. Constraint solver verifies plans before rendering.
- Gains: Eliminates wasted renders, scales to 1000s of videos/week
- Costs: Limits creative flexibility, requires hand-tuning budget tokens
- Complexity: Medium

## Chosen Approach
**Four-Pillar System with Phase-Based Animation** — absorbing insights from the ideation engine (animation state machines, budget pre-validation, composition-first over code-gen-first).

The phase-based animation model is the key architectural insight: each scene type decomposes into named phases (enter → highlight → exit) with normalized 0-1 progress. The AI describes "what phase" not "what frame." Components handle physics internally.

## Key Context Discovered During Shaping

### Technical
- Every component uses `{damping: 200}` — overdamped, no personality. Tuning springs alone is a massive visual uplift.
- `@remotion/motion-blur` `<CameraMotionBlur>` and `<Trail>` are drop-in cinematic wrappers
- `@remotion/paths` has `evolvePath()` for SVG draw-on, `getPointAtLength()` for path following
- CSS cinematic effects (vignette, color grading via `mix-blend-mode`, lens flare) are zero-dep
- `remotion-bits` `<AnimatedText>` has word/character/line splitting built in
- Remotion's `random()` function is deterministic — use for particles, never `Math.random()`

### Ecosystem
- Remotion's own AI SaaS template uses skill-detection + code generation + in-browser compilation
- JSON Render pattern (catalog + registry + schema) prevents hallucination and ensures type-safe output
- `remotion-superpowers` (DojoCodingLabs) is the most comparable project — full Claude Code + Remotion with 13 commands
- Nobody in the ecosystem has built multimodal video critique for motion graphics — this is novel
- Remotion Skills (30+ rule files) are the foundation for reliable AI code generation

### Ideation Engine Signals
- Constraint satisfaction > capability expansion (every mutation adding features scored lower)
- Phase-based animation model: decompose scenes into named states with 0-1 progress (from game engine analogy)
- Animation budget algebra: pre-validate complexity before rendering to reduce wasted cycles
- Semantic composition > code generation: AI is better at selecting/composing than writing arbitrary code

## Four Pillars (Detailed)

### Pillar 1: Component Library (`@filescience/video-lib`)
JSON-invocable scene types refactored with phase-based animation model:
- **Phase model**: Each component has `enter → highlight → exit` phases with normalized 0-1 progress
- **Existing 8 types**: Refactored with Tailwind, tuned springs (kill damping:200), motion blur wrapping
- **New types**: splitScreen, dataViz, codeBlock, statement, timeline
- **Cinematic layers**: Vignette, color grading, lens flare as composable wrappers
- **Dependencies**: `@remotion/tailwind-v4`, `@remotion/motion-blur`, `@remotion/paths`

### Pillar 2: Remotion Skill System (`.claude/skills/remotion/`)
Domain knowledge that teaches the agent composition rules + valid Remotion patterns:
- Modeled after Remotion's official 30+ rule files
- Animation patterns, spring configs, chart idioms, text animation patterns
- Composition rules: what scenes pair well, register variation, pacing
- Budget constraints: complexity tokens per component type
- What NOT to do: no framer-motion, no CSS transitions, no Math.random()

### Pillar 3: Code Gen Escape Hatch
Narrow, composition-first. The agent composes from the library by default; writes TSX only when the library can't express the need:
- Imports from `@filescience/video-lib` + raw Remotion primitives
- Type-checked before render via `tsc --noEmit`
- Skill system (Pillar 2) ensures quality
- Expected usage: ~20% of scenes

### Pillar 4: Vision Review MCP
Multimodal LLM critiques rendered output with budget pre-validation:
- Pre-render: Budget algebra validates animation plan is achievable
- Post-render: Multimodal model (Gemini for video analysis) scores pacing, animation quality, visual coherence, storytelling
- Returns structured critique with specific fix suggestions per scene
- Powers autonomous render→critique→fix loops
- Novel territory — expect 2-3 iterations on critique taxonomy

## Dogfood Plan
Each phase tested by producing an Akasha/FileScience "AI safety infrastructure" video, iterated through all four pillars.

## Next Step
- [Plan] → `/create_plan thoughts/shared/briefs/2026-03-28-video-animation-system-upgrade.md`
