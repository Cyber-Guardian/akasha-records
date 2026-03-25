---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Akasha Capability Platform — MCP-First Backup with Generative UI

**Date:** 2026-03-20
**Status:** Shaped → Planning

## Problem
FileScience is a dashboard-first backup product in a world moving toward agent-native interfaces. Current onboarding requires humans to navigate a web app (signup, OAuth, billing, cloud connection). This limits the market to people who actively seek backup solutions and self-serve through a UI. Meanwhile:
- No B2B SaaS product — and no backup product — is messaging-first or agent-first today
- The "agent-as-customer" market is emerging: autonomous agents managing IT will need to procure backup services programmatically
- MSPs want white-label but FileScience doesn't offer it — building per-customer UIs doesn't scale
- The conversational commerce market is projected $8.8B → $52.8B by 2034

## Constraints
- OAuth for Clio/M365/Box requires a browser-based consent flow — cannot be fully eliminated, but can be minimized (magic links, ASWebAuthenticationSession on iOS)
- Apple Messages for Business requires MSP approval (3-6 months) or going through an existing MSP (Twilio, Infobip)
- MCP is the emerging standard for agent-to-tool communication but discovery/registry is still maturing
- Current Akasha plan is a repo split (knowledge base extraction) — this vision is architecturally upstream
- The generative UI protocol space is early: A2UI (Google, Apache 2) and AG-UI (CopilotKit) are v0.x but aligned with this direction

## Options Considered

### MCP Server Only (Headless)
Expose FileScience backup as MCP tools. No UI layer. Agents consume raw tool responses and build their own rendering.
- Gains: Simplest to build, maximum flexibility for consumers, fastest to ship
- Costs: Poor out-of-box experience, every consumer rebuilds the same UI patterns, no showcase
- Complexity: Low

### iMessage Agent Only (Vertical)
Build a conversational agent on Apple Messages for Business. Premium native experience with Apple Pay, OAuth, Forms.
- Gains: Best consumer UX, Apple Pay eliminates billing friction, verified business badge, first-mover in backup
- Costs: iOS only, MSP gatekeeping, platform lock-in, doesn't serve the agent-as-customer market
- Complexity: Medium

### MCP-First + Generative UI Protocol + iMessage Showcase (Platform)
Build the MCP server as the core capability layer. Ship a generative UI component protocol with default components (forms, pickers, auth cards, status views) that render tool responses into interactive UI. Defaults work out of the box. Override any component for custom branding/UX. Build an iMessage agent as the first-party client using defaults — proves the experience and serves as direct distribution.
- Gains: Maximum distribution (every agent is a channel), white-label for free via overrides, own one premium channel (iMessage), validates both "agent-as-customer" and "human via text" theses simultaneously, aligned with emerging A2UI/AG-UI standards
- Costs: Three layers to build (MCP server + UI protocol + iMessage client), generative UI protocol is novel engineering, no existing standard to adopt wholesale
- Complexity: High — but each layer is independently valuable and shippable

## Chosen Approach
**MCP-First + Generative UI Protocol + iMessage Showcase.** Akasha becomes the capability platform. FileScience backup is the first capability. Ships with a default generative UI component library (overrideable). iMessage agent is the owned distribution channel using defaults.

**The pitch:** "Use our backup. Build your own experience."

## Key Context Discovered During Shaping

### Market research
- No B2B SaaS product is messaging-first today — the space is genuinely open
- Apple Messages for Business supports Forms (iOS 18.4+), Apple Pay, native OAuth 2.0, list/time pickers, custom iMessage app extensions
- WhatsApp Flows offers comparable interactive forms cross-platform (3B users)
- A2UI (Google) and AG-UI (CopilotKit) are emerging open protocols for agent-to-UI communication — declarative JSON model matches messaging platform payloads
- Bain 2025: "systems of record + agent operating systems + outcome interfaces" is the winning architecture
- South American bank running payments entirely through WhatsApp is the closest real-world analog

### Architecture
- MCP server exposes tools: create_account, authorize_cloud, configure_backup, restore_item, search_items, list_anomalies, etc.
- Generative UI protocol: default components render MCP tool responses into interactive elements (platform-adaptive)
- Override system: consumers swap any component (branding, custom flows, industry-specific UX) or go fully headless
- iMessage agent: thin client — conversation design + AMB adapter + default UI components

### Origin threads (from team brainstorm 2026-03-20)
- "need IDE that lets you develop at a more macro level" — led to agent-native thinking
- "agent is the customer, not humans?" — led to MCP-first architecture
- "backup your account from text message" — led to iMessage channel
- "via MCP?" — the architectural breakthrough: expose capability, let anyone build the interface
- "comes with some protocol for generative UI component lib, default set, overrideable" — the platform play
- "becomes first capability in Akasha" — the scope: Akasha is the platform, not just a knowledge base

### Related prior work
- [[2026-03-20-akasha-records|Akasha Records Repo Split]] — Phase 0 infrastructure (knowledge base extraction)
- [[2026-03-16-custom-deterministic-agentic-backpressure|Backpressure Tooling]] — agent quality gates for autonomous execution
- [[2026-03-01-multi-agent-architecture-research-brief|Multi-Agent Architecture Research]] — conflict prevention > resolution
- [[2026-03-11-shapes-brainstorm-best-ideas|Shapes Brainstorm]] — "backup is distribution, data layer is moat"
- [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] — interaction design philosophy governing timing, rhythm, and effort signaling across all Akasha surfaces

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-akasha-capability-platform.md`
