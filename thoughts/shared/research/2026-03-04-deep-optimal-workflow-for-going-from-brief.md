---
date: 2026-03-04T14:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-optimal-workflow-for-going-from.md
last_generated: 2026-03-05T03:14:26.977305+00:00
---

# Research Brief: 2026-03-04-deep-optimal-workflow-for-going-from

## TL;DR

The optimal workflow follows a six-stage pipeline: (1) scaffold with v0.dev, (2) refine in Figma + pull via Figma MCP, (3) generate components with Claude Code using a types-first approach, (4) wire data layers with TanStack Query + Zustand, (5) build dashboard widgets (TanStack Table, Tremor charts, SSE feeds), (6) deploy via Vercel + Railway/Fargate. The critical enablers are: shadcn/ui as the component library (its "open code" model gives LLMs full source context), OpenAPI codegen from FastAPI for type-safe API contracts, and CLAUDE.md as an active constraint engine with 10-15 hard rules. The inner loop targets 10-15 minute cycles for frontend components using a plan-then-implement approach with screenshot+browser-context revision feedback. Testing requires a 60/30/10 pyramid (Vitest + Playwright + Schemathesis) with mandatory SAST gates since 45% of AI-generated code fails security tests.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How will Figma MCP's bidirectional "Code to Canvas" evolve — will it close the iterative update loop?
- Will v0.dev add accessibility and test generation to its output?
- Is there a better full-stack preview solution than manually coordinating Vercel + Railway branch deploys?
- How should dashboard-specific CLAUDE.md rules evolve as AI tools improve at React generation?
- What's the optimal approach when the dashboard needs to embed AI features (chat, copilot) via Vercel AI SDK alongside being built by AI?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-optimal-workflow-for-going-from.md`
