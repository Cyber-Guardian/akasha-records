---
topic: Dashboard Facelift
status: active
touched: 2026-03-21
related:
  - thoughts/shared/reference/2026-02-22-backend-frontend-monorepo-structure.md
  - thoughts/shared/research/2026-02-22-saas-dashboard-framework-for-agent-development.md
  - thoughts/shared/decisions/2026-02-19-linear-workspace-restructure.md
---

# Dashboard Facelift

## Current State
"Dashboard v1.1 Facelift" is a named Linear project under the Dashboard Evolution initiative. No implementation started — no code, no plan file, no brief.

Stack and structure decisions made but not yet acted on. **Interaction design philosophy now defined** -- the [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] governs timing, rhythm, presence, and effort signaling for the dashboard surface. Dashboard-specific patterns (skeleton screens, optimistic UI, staged progress, compact/power-user mode) are specified there.

Deep research on **generative UI** completed (2026-03-05). Key findings: GenUI has strong fit for NL queries and compliance reporting in the dashboard, but restore wizards should stay deterministic. Recommended implementation: CopilotKit (runtime + FastAPI SDK) + assistant-ui (chat primitives) using constrained shadcn/ui catalog. Phase 2/3 work — doesn't block Phase 1 static dashboard build. See `thoughts/shared/research/2026-03-05-deep-generative-ui-use-cases-brief.md`.

## Key Decisions
- 2026-03-21: Interaction design philosophy defined in [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] -- 4 principles governing perceived performance, effort signaling, deliberate friction, and adaptive rhythm
- 2026-02: Frontend lives in `frontend/apps/dashboard` within this repo (isolated sub-workspace)
- 2026-02: Next.js App Router + shadcn/ui + Tailwind CSS (highest agent training data, v0.dev native)
- 2026-02: pnpm workspaces + optional Turborepo when frontend grows
- 2026-02: `contracts/openapi/` for API contract synchronization
- Marketing/landing content stays outside this repo

## Open Questions
- No implementation timeline set
- Dashboard Evolution roadmap: v1.1 facelift -> v1.2 features -> V2 reskin (target 2026-12-31)

## Artifacts
- [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] -- interaction design philosophy (timing, rhythm, presence, effort signaling)
- No code artifacts yet
