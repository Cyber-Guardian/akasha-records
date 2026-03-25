---
type: event
created: 2026-03-09
status: active
---
# Idea Brief: Generative UI for FileScience

**Date:** 2026-03-09
**Status:** Shaped → Parked

## Problem
FileScience surfaces data from three structurally different cloud apps (Clio, M365, Box). Static dashboard views either flatten everything into generic tables or require bespoke views per entity type per cloud — neither scales. Users (IT/MSP operators) want to describe what they need and get contextual, cloud-native-looking results without navigating deep menu hierarchies.

## Constraints
- Dashboard stack decided: Next.js App Router + shadcn/ui + Tailwind (no code yet)
- Backend is Python Lambda + DynamoDB — generative UI is frontend, but data contracts matter
- Three supported clouds with very different data models and native UIs
- Product north star: fast, slick, accessible — generative UI must feel polished, not janky
- Existing Slack triage agent provides a natural first surface for generative UI

## Options Considered

### Build-time Agent Library + Runtime Search Bar
Autonomous agents expand a component registry at dev time (tested, PR'd components mimicking native cloud app UIs). At runtime, an LLM selects from the registry via a search-bar-as-primary-interface, parameterizes components with backup data, and streams results.
- Gains: Most powerful UX — natural language → contextual, cloud-native views. Flywheel: runtime gaps drive build-time generation.
- Costs: Highest complexity. Requires component registry, DSL layer, LLM integration, streaming infrastructure.
- Complexity: High

### Slack Generative UI (Starting Point)
Upgrade existing triage agent to output structured Slack Block Kit components instead of plain text. Agent selects from a registry of Block Kit templates (restore cards, diff views, item tables with checkboxes) based on user intent.
- Gains: Lowest friction to ship — triage agent already exists, Block Kit is a constrained registry. Validates the pattern in production before investing in full dashboard generative UI.
- Costs: Block Kit has limited expressiveness compared to React. Not a replacement for the dashboard vision.
- Complexity: Low–Medium

### Browser Extension with Contextual Sidebar
FileScience sidebar renders inside Clio/M365/Box, adapting its backup-health UI to the current cloud app context.
- Gains: "Be where the user already is" — zero context switching.
- Costs: Cross-browser extension development, per-cloud page context parsing, maintenance burden as native UIs change.
- Complexity: High

## Chosen Approach
**Slack Generative UI** as starting point — validates the generative UI pattern (registry → DSL → renderer) in the lowest-friction environment. Triage agent already exists. Block Kit is a constrained component registry. Success here de-risks the full dashboard search bar vision.

## Key Context Discovered During Shaping
- Thesys architecture: LLM outputs structured DSL (not raw HTML), rendering engine owns the DOM — same pattern works across render targets (React, Block Kit, email HTML)
- shadcn/ui "open code" model is uniquely suited to build-time agent generation — agents can read/write/test actual component source
- The meta-architecture is renderer-agnostic: Component Registry → DSL → Render Targets. Investment is in the registry + DSL, not any single interface.
- Prior research: [[2026-02-22-saas-dashboard-framework-for-agent-development|Dashboard Framework Research]], [[dashboard-facelift|Dashboard Facelift topic]]
- Interaction design: [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] — defines how generative UI components should feel (entrance animations, source attribution, approval gates, adaptive density)

## Adjacent Ideas (for future shaping)
- **Search bar as primary dashboard interface** — strongest long-term vision
- **Anomaly narration** — proactive generative UI (system generates incident views when anomalies detected)
- **Restore previews** — generate preview of what restore will change, rendered as cloud-native diff
- **Multi-tenant MSP comparison views** — risk-ranked dashboards generated on the fly
- **Natural language policy builder** — describe backup policy in words, get config UI

## Next Step
- [Parked] → ENG-2394 (Slack Block Kit generative UI for triage agent)
