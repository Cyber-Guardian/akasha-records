---
date: 2026-02-22T12:00:00-05:00
researcher: Claude Opus 4.6
git_commit: 05cdb25fafeceecd1f369b5b5f6f7b245b2a8792
branch: feat/dashboard
repository: filescience
topic: "Best SaaS dashboard framework for AI agent-driven development"
tags: [research, frontend, dashboard, agent-development, framework-selection]
status: complete
last_updated: 2026-02-22
last_updated_by: Claude Opus 4.6
---

# Research: Best SaaS Dashboard Framework for AI Agent-Driven Development

**Date**: 2026-02-22
**Researcher**: Claude Opus 4.6
**Git Commit**: 05cdb25fafeceecd1f369b5b5f6f7b245b2a8792
**Branch**: feat/dashboard
**Repository**: filescience

## Research Question

What is the best framework for building a SaaS dashboard in the FileScience monorepo, optimized for AI agents (Claude Code, Cursor, v0, etc.) as primary developers? Consider tooling, community size, training data volume, and agent code generation quality.

## Summary

**Recommendation: Next.js App Router + shadcn/ui + Tailwind CSS** — with explicit agent guardrails in CLAUDE.md.

Next.js + shadcn/ui wins on every axis that matters for agent-first development: largest training data corpus, best tooling ecosystem (official MCP server, llms.txt, v0.dev), most dashboard templates, and convergence signal from all major AI builder tools. The main risk — agents generating stale App Router patterns — is well-documented and mitigable with system prompt rules.

React Router v7 is the strongest alternative if App Router complexity proves too costly. SvelteKit is technically excellent but carries ongoing agent overhead. htmx + FastAPI is viable for internal tools but caps interactivity.

## Decision Matrix

| Criterion | Next.js + shadcn | React Router v7 | SvelteKit | htmx + FastAPI |
|---|---|---|---|---|
| Agent training data volume | **Highest** | Medium | Low | High |
| Agent error rate (out-of-box) | Medium (RSC confusion) | **Low** | High (runes) | **Low** |
| Component library maturity | **107k stars** | 107k (same shadcn) | 8.2k (shadcn-svelte) | N/A |
| Dashboard templates | **Extensive** | Growing | Small | Sparse |
| Official MCP server | **Yes (shadcn)** | No | Yes (Svelte) | No |
| v0.dev integration | **Native** | No | No | No |
| Mental model simplicity | Low | High | High | **Highest** |
| Python team coherence | Low | Low | Low | **Highest** |
| Interactive dashboard fit | Good | Good | Good | Moderate |
| pnpm workspace fit | Native | Native | Native | N/A |

## Detailed Findings

### 1. Training Data & Benchmark Evidence

**Web-Bench (ByteDance, arXiv May 2025)** — the most relevant cross-framework LLM benchmark:

| Model | React | Vue | Angular | Svelte |
|---|---|---|---|---|
| Claude 3.7 Sonnet | **65%** | 30% | 40% | 25% |
| Claude 3.7 Sonnet (Thinking) | **60%** | 40% | 50% | 55% |
| GPT-4o | **35%** | 30% | 5% | 20% |

React led in 4 of 6 models tested. The paper attributes this to training data volume — React has ~235K GitHub stars vs Vue's 208K, Angular's 96K, Svelte's 79K.

**npm weekly downloads (Feb 2026):**
- React: 73.5M (~84% of total across major frameworks)
- Next.js: 25.4M
- Vue: 8.5M
- Svelte: 2.7M

**Stack Overflow Developer Survey 2025:**
- React: 44.7% of respondents
- Next.js: 20.8%
- Svelte: 7.2%

### 2. Next.js + shadcn/ui: The Agent Ecosystem

**Why it wins for agents:**

1. **shadcn/ui Official MCP Server** — gives agents structured access to component registry, demos, metadata, and blocks. Config: `pnpm dlx shadcn@latest mcp init --client claude`
2. **shadcn/ui llms.txt** — machine-readable index of all 60+ components across 5 logical groups at [ui.shadcn.com/llms.txt](https://ui.shadcn.com/llms.txt)
3. **v0.dev flywheel** — Vercel's AI generator outputs shadcn/ui exclusively, creating a closed-loop training data cycle
4. **"Open code" model** — components are copied into your codebase, so agents can read and modify actual source (vs. fighting abstracted APIs in MUI/Ant Design)
5. **CLI for agents** — `npx shadcn@latest add button card sidebar data-table` is non-interactive and directly invokable from any agent shell tool

**Concrete numbers:**
- shadcn/ui: 107,085 GitHub stars, +26,300 in 2025 alone
- Next.js: ~137,886 GitHub stars, 25.4M weekly npm downloads
- 4+ production SaaS dashboard starters with 6k-15k stars each

**Known agent failure modes (documented, mitigable):**

| Failure | Root Cause | Fix |
|---|---|---|
| Pages Router patterns in App Router | Stale training data | CLAUDE.md rules |
| `useEffect` for server-fetchable data | RSC unfamiliarity | CLAUDE.md rules |
| Sync `params` (Next.js 15+ broke this) | Training cutoff pre-2024 | CLAUDE.md rules |
| Over-broad `"use client"` | Defensive pattern | CLAUDE.md rules |
| `redirect()` inside `try/catch` | `redirect()` throws internally | CLAUDE.md rules |

All failures are addressable via explicit system prompt rules. Community `.cursorrules` files already codify these.

### 3. React Router v7: The Simpler Alternative

React Router v7 (née Remix) has a dramatically simpler mental model: every route exports a `loader` (reads) and `action` (writes), both server-only. No RSC/client component distinction, no `"use client"` / `"use server"` directives.

**Pros for agents:**
- Lower conceptual surface area → fewer subtle errors
- Deterministic loader/action boundary → easier to test
- No bundler-level magic to misunderstand

**Cons for agents:**
- Less training data than Next.js (framework is newer under this name)
- No official MCP server or llms.txt yet (community has filed requests)
- Fewer dashboard templates/starters
- v0.dev does not generate React Router v7 code

**Verdict:** Strongest runner-up. Consider if Next.js App Router complexity causes persistent agent failures in practice.

### 4. SvelteKit: High DX, High Agent Tax

Svelte 5 runes (`$state`, `$derived`, `$effect`) replaced Svelte 4's reactivity model in October 2024. Most LLM training data contains Svelte 4 patterns.

**The concrete problem:**

| Correct (Svelte 5) | What agents generate (Svelte 4) |
|---|---|
| `let x = $state(value)` | `let x = value` |
| `let y = $derived(expr)` | `$: y = expr` |
| `onclick={handler}` | `on:click={handler}` |
| `let { prop } = $props()` | `export let prop` |

**Mitigations exist:** Official MCP server, llms.txt, svelte-bench, `.cursorrules` — but all require per-project setup. SvelteBench shows Claude Opus 4.6 now scores 100% on Svelte 5, but that's a simple benchmark, not complex multi-step tasks.

**shadcn-svelte:** 8,250 stars vs React shadcn/ui's 107k. Component parity is tracked daily but not complete.

**Verdict:** Best developer experience (#1 in State of JS satisfaction), but the agent overhead tax is real and ongoing. Not recommended for agent-first development today.

### 5. htmx + FastAPI: Stay in Python

For a Python-first team, htmx eliminates the entire JS build toolchain. A production SaaS post-mortem reported ~70% dev time savings vs React + separate API.

**How it works:** htmx attributes (`hx-get`, `hx-post`, `hx-trigger`, `hx-swap`) on Jinja2 templates trigger server round-trips that return HTML fragments. No JS build step, no client-side state management.

**Agents generate htmx well** — the attribute surface is small and well-represented in training data. Jinja2 is extremely well-represented.

**Limitations:**
- Every interaction requires a server round-trip (perceptible at >200ms)
- Rich client-side interactivity (drag-and-drop, optimistic UI, complex filter state) requires Alpine.js patches
- Backend routes become "UI-aware" — `views.py` grows substantially
- Error handling across hypermedia responses is non-trivial

**Verdict:** Viable for internal ops dashboards with moderate interactivity. Not recommended if the dashboard needs rich client-side interactions (file tree navigation, multi-step wizards, real-time updates).

### 6. Component Library Ranking for Agent Generation

All major AI code generation tools (v0, Cursor, Claude Code, Lovable) default to shadcn/ui + Tailwind without being prompted. This is an industry convergence signal.

| Library | Training Data | Agent Customizability | Dashboard Fit |
|---|---|---|---|
| **shadcn/ui** | Rapidly growing (107k stars) | **Best** (open source in your codebase) | Excellent |
| **MUI** | Highest (oldest) | Poor (abstracted API) | Good |
| **Ant Design** | High (esp. Chinese ecosystem) | Moderate | **Best for data tables** |
| **Radix UI (raw)** | Good | Good but redundant vs shadcn | Good |

### 7. Dashboard Templates (Next.js + shadcn/ui)

| Template | Stars | Key Features |
|---|---|---|
| nextjs/saas-starter | 15,400 | Auth, RBAC, Stripe billing, dashboard |
| satnaing/shadcn-admin | 11,100 | 10+ pages, dark mode, RTL |
| Kiranism/next-shadcn-dashboard-starter | 6,000 | Multi-tenant, RBAC, Kanban, 6+ themes |
| ixartz/SaaS-Boilerplate | 6,800 | i18n, multi-tenancy, E2E tests |

### 8. Monorepo Integration

Per the existing [backend-frontend monorepo structure research](2026-02-22-backend-frontend-monorepo-structure.md), the recommended layout is:

```
filescience/
├── components/          # existing Python Polylith
├── bases/               # existing Python Polylith
├── projects/            # existing Python deployable units
├── frontend/
│   ├── apps/
│   │   └── dashboard/   # Next.js + shadcn/ui
│   ├── packages/
│   │   ├── ui/          # shared shadcn components
│   │   └── api-client/  # generated TS client from FastAPI OpenAPI
│   ├── package.json
│   ├── pnpm-workspace.yaml
│   └── pnpm-lock.yaml
├── contracts/
│   └── openapi/         # API contract (source of truth for client generation)
└── pyproject.toml       # existing Python workspace root
```

## Agent Guardrails (Required CLAUDE.md Rules)

If choosing Next.js App Router, add these rules to the frontend CLAUDE.md:

```
## Next.js App Router Rules (for AI agents)
- NEVER use Pages Router patterns (getServerSideProps, getStaticProps)
- NEVER use useEffect for data that can be fetched in a Server Component
- ALWAYS use async params: `const { slug } = await params` (Next.js 15+)
- PREFER Server Components; only add "use client" for Web API access (useState, useEffect, event handlers)
- USE Server Actions for mutations, NOT Route Handlers
- NEVER put redirect() inside try/catch
- ALWAYS use Suspense boundaries ABOVE async components, not inside them
- USE `npx shadcn@latest add <component>` to add new components
- CHECK shadcn MCP server for component APIs before generating custom implementations
```

## Recommendation

**Primary:** Next.js App Router + shadcn/ui + Tailwind CSS + pnpm workspace

**Why:**
1. Largest agent training data corpus (73.5M React + 25.4M Next.js weekly downloads)
2. Only stack with official MCP server for component registry
3. All major AI builder tools converge on this stack
4. Most dashboard templates and starters available
5. Known agent failure modes are documented and addressable via system prompt rules

**Fallback:** React Router v7 + shadcn/ui — if App Router complexity causes persistent agent failures in practice.

**Not recommended for this use case:** SvelteKit (ongoing agent overhead), htmx (interactivity ceiling), Astro (wrong paradigm for dashboards), Retool/Appsmith (no portable code output).

## Sources

### Benchmarks & Data
- [Web-Bench: A LLM Code Benchmark Based on Web Standards and Frameworks (ByteDance, arXiv May 2025)](https://arxiv.org/html/2505.07473v1)
- [DesignBench: MLLM-based Front-end Code Generation (arXiv June 2025)](https://arxiv.org/html/2506.06251v1)
- [WebApp1K: A Practical Code-Generation Benchmark (arXiv 2024)](https://arxiv.org/html/2408.00019v1)
- [SvelteBench Results](https://khromov.github.io/svelte-bench/benchmark-results-merged.html)
- [Stack Overflow Developer Survey 2025](https://survey.stackoverflow.co/2025/technology)
- [2025 JavaScript Rising Stars](https://risingstars.js.org/2025/en)
- [State of JavaScript 2024](https://2024.stateofjs.com/en-US/)

### Framework Research
- [The React + AI Stack for 2026 — Builder.io](https://www.builder.io/blog/react-ai-stack-2026)
- [React Frameworks in 2026: Next.js vs Remix vs React Router 7 — Medium](https://medium.com/@pallavilodhi08/react-frameworks-in-2026-next-js-vs-remix-vs-react-router-7-b18bcbae5b26)
- [State of React Community 2025 — Mark Erikson](https://blog.isquaredsoftware.com/2025/06/react-community-2025/)
- [Next.js vs Remix 2025 — Strapi](https://strapi.io/blog/next-js-vs-remix-2025-developer-framework-comparison-guide)
- [TanStack Start v1 — InfoQ](https://www.infoq.com/news/2025/11/tanstack-start-v1/)
- [React Survey Shows TanStack Gains — The Register](https://www.theregister.com/2026/02/17/react_survey_shows_tanstack_gains/)

### Next.js + shadcn/ui Agent Ecosystem
- [shadcn/ui GitHub (107k stars)](https://github.com/shadcn-ui/ui)
- [shadcn MCP Server docs](https://ui.shadcn.com/docs/mcp)
- [shadcn llms.txt](https://ui.shadcn.com/llms.txt)
- [Common Mistakes with the Next.js App Router — Vercel](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them)
- [LLM prompts and AI IDE setup — Next.js Discussion #81291](https://github.com/vercel/next.js/discussions/81291)
- [v0.dev FAQ](https://v0.app/docs/faqs)
- [AI Frontend Generator Comparison 2025 — Hans Reinl](https://www.hansreinl.de/blog/ai-code-generators-frontend-comparison)

### SvelteKit
- [Better AI LLM Assistance for Svelte 5 — Stanislav Khromov](https://khromov.se/getting-better-ai-llm-assistance-for-svelte-5-and-sveltekit/)
- [AI / LLM Prompt/Rules for Svelte 5 — GitHub Discussion #14125](https://github.com/sveltejs/svelte/discussions/14125)
- [shadcn-svelte GitHub](https://github.com/huntabyte/shadcn-svelte)
- [AI-Assisted SvelteKit Development — svelteconsulting.dev](https://svelteconsulting.dev/blog/ai-assisted-sveltekit-development-with-claude-code)

### htmx + Python
- [HTMX in Production SaaS Post-Mortem — Substack](https://enriquebruzual.substack.com/p/speeding-up-a-lead-qualifying-saas)
- [Building Real-Time Dashboards with FastAPI and HTMX — Medium](https://medium.com/codex/building-real-time-dashboards-with-fastapi-and-htmx-01ea458673cb)
- [fasthx: Declarative FastAPI HTMX SSR — GitHub](https://github.com/volfpeter/fasthx)

### Monorepo
- [Complete Monorepo Guide: pnpm Workspaces 2025 — jsdev.space](https://jsdev.space/complete-monorepo-guide/)
- [Nx 22 Release — Nx Blog](https://nx.dev/blog/nx-22-release)

## Related Research

- **[[2026-02-22-optimal-agentic-nextjs-development|Optimal Agentic Next.js Development]]** — follow-up deep-dive covering AGENTS.md, shadcn MCP workflow, project structure, DX toolchain, testing strategy, starter templates, auth/state/data patterns, and deployment

## Open Questions

1. **Dashboard interactivity requirements** — How much client-side state does the FileScience dashboard need? (file tree browsing, drag-and-drop, real-time updates?) This determines whether htmx is viable or React is required.
2. **Starter template selection** — Which of the 4+ Next.js + shadcn dashboard starters best matches the FileScience auth/billing/multi-tenancy needs?
3. **API contract generation** — Should the TypeScript client be generated from FastAPI's OpenAPI spec (recommended) or hand-written?
4. **Deployment target** — Vercel, AWS Amplify, S3+CloudFront, or self-hosted? This influences Next.js config (standalone output vs. edge runtime).
