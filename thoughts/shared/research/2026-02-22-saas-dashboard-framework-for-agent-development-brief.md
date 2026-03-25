---
date: 2026-02-22T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-22-saas-dashboard-framework-for-agent-development.md
last_generated: 2026-02-23T03:37:46.100151+00:00
---

# Research Brief: 2026-02-22-saas-dashboard-framework-for-agent-development

## TL;DR

**Recommendation: Next.js App Router + shadcn/ui + Tailwind CSS** — with explicit agent guardrails in CLAUDE.md.

Next.js + shadcn/ui wins on every axis that matters for agent-first development: largest training data corpus, best tooling ecosystem (official MCP server, llms.txt, v0.dev), most dashboard templates, and convergence signal from all major AI builder tools. The main risk — agents generating stale App Router patterns — is well-documented and mitigable with system prompt rules.

React Router v7 is the strongest alternative if App Router complexity proves too costly. SvelteKit is technically excellent but carries ongoing agent overhead. htmx + FastAPI is viable for internal tools but caps interactivity.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Dashboard interactivity requirements** — How much client-side state does the FileScience dashboard need? (file tree browsing, drag-and-drop, real-time updates?) This determines whether htmx is viable or React is required.
2. **Starter template selection** — Which of the 4+ Next.js + shadcn dashboard starters best matches the FileScience auth/billing/multi-tenancy needs?
3. **API contract generation** — Should the TypeScript client be generated from FastAPI's OpenAPI spec (recommended) or hand-written?
4. **Deployment target** — Vercel, AWS Amplify, S3+CloudFront, or self-hosted? This influences Next.js config (standalone output vs. edge runtime).

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-22-saas-dashboard-framework-for-agent-development.md`
