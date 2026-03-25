---
type: event
created: 2026-03-21
status: active
tags: [akasha, ux, interaction-design, perceived-performance, agent-personality]
---
# Idea Brief: Akasha Interaction Design Manifesto

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
Akasha's architecture (MCP + generative UI + channels) and product thesis (trust ladder, text/call-first, "the agent becomes the only app") are well-defined. What's missing is the felt experience layer -- the principles governing how every interaction communicates intelligence, care, and trustworthiness. For a product whose moat is temporal irreplaceability, this layer isn't polish -- it's core product.

## Constraints
- Must work across channels: iMessage, WhatsApp, voice, web dashboard -- each with different timing/rendering capabilities
- Must not feel manipulative -- research shows users punish perceived deception when fake delays are discovered
- Akasha Capability Platform brief already defines generative UI protocol -- this complements, not competes
- Dashboard is Next.js + shadcn/ui + Tailwind -- animation/transition primitives available
- Trust in AI declining (42%, down from 58% in 2023) -- anthropomorphism alone doesn't fix this
- Novice/expert split: human-like pacing helps novices, frustrates power users

## Options Considered

### Inline Additions to Existing Plans
Add interaction timing guidance as sections within each downstream plan (dashboard v1.1, consumer product, generative UI protocol).
- Gains: No new artifact, guidance lives where implementers need it
- Costs: Duplicated principles across plans, no single source of truth, inconsistency across surfaces
- Complexity: Low

### Standalone Interaction Design Manifesto
A reference document defining timing principles, presence patterns, adaptive rhythm, and anti-patterns. All downstream plans cite it.
- Gains: Single source of truth, ensures consistency across dashboard + agent + generative UI. Prevents each plan from reinventing interaction philosophy.
- Costs: One more document in the chain. Risk of becoming aspirational shelfware if not referenced.
- Complexity: Medium

### Full Design System Spec (Components + Timing + Motion)
A comprehensive design system covering component library, motion language, timing, and channel-specific behavior.
- Gains: Most complete, directly implementable
- Costs: Premature -- no dashboard code exists yet, consumer product is vision-stage. Would over-specify for the current moment.
- Complexity: High

## Chosen Approach
**Standalone Interaction Design Manifesto** -- a principles document that downstream plans reference. Scoped to interaction philosophy (timing, rhythm, presence, effort signaling), not component specs or motion language.

## Key Context Discovered During Shaping

### Academic foundations (from web research)
- **Effort heuristic** (Buell & Norton 2011): Visible effort increases perceived value by ~8%. But amplifies existing opinions -- bad output + visible effort = worse perception than no effort shown.
- **Maister's Laws** (1985): Unoccupied time feels longer. Pre-process waits are worst. Houston airport solved complaints by making passengers walk, not by speeding luggage.
- **Anthropomorphism** (IJHCI 2025, ACM CUI 2024): Typing indicators increase social presence but don't reliably build trust. Textual justification of delays ("reasoning through your request") outperforms pacing alone (N=194). Expert users interpret human-like delays as slowness.
- **Deliberate friction** (Wells Fargo, TurboTax): Intentional slowdowns increase trust in high-stakes contexts. Collapses if users discover the delay is fake.
- **Perceived performance** (ECCE 2018): Skeleton screens reduce perceived wait 20-30%. Optimistic UI eliminates perceived latency for low-risk actions.

### Akasha-specific connections
- Trust ladder (Section 7 moat): effort signaling is the mechanism for climbing backup trust -> coaching trust -> action trust
- Text/call-first interface (Section 11): channel architecture defined but interaction rhythm not specified
- Generative UI protocol (Capability Platform brief): default components need timing/animation spec
- Dashboard facelift (topic note): greenfield -- interaction philosophy can be baked in from the start

### Key insight
The user's brainstorm note "if it takes longer it feels more thoughtful and by extension more intelligent" maps directly to the effort heuristic. For Akasha's consumer agent, this is the core mechanism: perceived effort -> perceived intelligence -> trust. Each rung of the trust ladder needs different timing/rhythm.

## Surfaces This Manifesto Governs

| Surface | Key Principles |
|---------|---------------|
| FileScience Dashboard (B2B) | Perceived performance: skeleton screens, optimistic UI, action queuing, consistent response times |
| Akasha Agent (B2C text/call) | Conversational rhythm: typing indicators, read receipts, variable pacing, effort signaling, textual justification |
| Generative UI components (both) | Transition choreography: how components appear, load, and respond inline across channels |
| High-stakes actions (both) | Deliberate friction: validation phases for write-back actions, visible security/integrity checks |

## Anti-Patterns to Codify
- Fake delays on routine actions (only high-stakes contexts warrant deliberate friction)
- Anthropomorphism in expert/power-user contexts (adaptive, not universal)
- Theater without substance (effort signaling only works when underlying quality is high)
- Silent waits (any delay > 300ms needs visible acknowledgment)

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-akasha-interaction-design-manifesto.md`
