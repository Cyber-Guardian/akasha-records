---
type: event
created: 2026-03-21
status: active
tags: [akasha, caroline, imessage, poc, agent, interaction-design]
---
# Idea Brief: Caroline — iMessage Agent POC

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
The Akasha interaction manifesto defines how every agent interaction should feel, but no one has experienced it. A browser demo can't prove the thesis — the whole point is "no new apps." Caroline is a conversational AI agent that lives in iMessage, demonstrating all 4 manifesto principles (Never Leave in Dark, Show the Work, Earn Trust Through Friction, Adapt to the Human) in the app 1B+ people already use.

## Constraints
- Apple Messages for Business requires MSP approval (months) — not viable for POC
- Linq (gray-market iMessage API) provides typing indicators, read receipts, rich media with free sandbox and no Apple approval
- Linq does NOT support interactive Forms, Apple Pay, or List/Time Pickers (MSP-only features)
- Apple can revoke gray-market access at any time — acceptable for POC, not for production
- The POC is about feel, not features — Caroline doesn't need real data connectors
- Company is Akasha, agent is Caroline — "scary company name with friendly product name"
- Sound symbolism: Caroline hits nasals (warmth), open syllables (friendly), CVCV-adjacent pattern

## Options Considered

### Web Chat POC (assistant-ui + Next.js)
Browser-based chat widget with full timing control. Shareable URL.
- Gains: Maximum control, matches dashboard stack, no platform risk
- Costs: Doesn't prove the thesis — "no new apps" means no new apps, including a browser tab
- Complexity: Medium

### WhatsApp Sandbox + Web Hybrid
Twilio WhatsApp sandbox for real phone experience, web dashboard for rich display.
- Gains: Real phone experience, native typing indicator
- Costs: Only shows ellipsis (no custom reasoning text), sandbox limitations, dual-channel complexity
- Complexity: High

### iMessage via Linq (Chosen)
Blue bubble iMessage agent using Linq's API. Typing indicators, read receipts, rich media. User texts a number, Caroline responds.
- Gains: Proves the thesis — no new apps, real iMessage, blue bubbles. Typing indicators feel native. Shareable as "text this number."
- Costs: Gray-market risk (Apple can revoke). No interactive forms. Production would need MSP.
- Complexity: Medium

## Chosen Approach
**iMessage via Linq** — the only option that proves "the agent becomes the only app" in the app people already use. Gray-market risk is acceptable for POC. The interaction choreography (timing, pacing, effort signaling) works entirely through plain messages + typing indicators.

## Key Context Discovered During Shaping
- Linq: free sandbox, no Apple approval, SOC 2, $20M Series A, 100M+ msg/month, sub-120ms latency
- Linq supports typing indicators, read receipts, rich media, emoji reactions, webhooks
- Lindy.ai rebuilt their iMessage integration on Linq in 4 hours
- Existing triage bot stack (pydantic-ai + durable harness + thread state + Lambda) is reusable
- Claude API streaming is model-paced — all timing choreography is application-side (buffer + re-emit)
- [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] agent patterns section is the spec for Caroline's behavior

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-caroline-imessage-poc.md`
