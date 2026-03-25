---
date: 2026-03-05T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-deep-generative-ui-use-cases.md
last_generated: 2026-03-05T15:28:16.643931+00:00
---

# Research Brief: 2026-03-05-deep-generative-ui-use-cases

## TL;DR

Generative UI (GenUI) — where LLMs generate actual UI components at runtime rather than just text — is a real and maturing pattern, with Vercel, SAP, CopilotKit, and others shipping production implementations. The industry is converging on **structured JSON to component catalog mapping** (json-render, Google A2UI, OpenAI Open-JSON-UI) as the safe, testable approach, replacing the now-paused AI SDK RSC streaming. Developer reception is deeply mixed: non-determinism, accessibility gaps, and debugging difficulty are persistent concerns, with significant "solution looking for a problem" skepticism. For FileScience's Next.js + FastAPI dashboard, GenUI has **strong fit for natural language queries** ("show failed backups last week") and **compliance report generation**, but **restore wizards should remain deterministic**. The recommended implementation path is **CopilotKit** (runtime with native FastAPI SDK) paired with **assistant-ui** (chat/tool UI primitives), using a constrained shadcn/ui component catalog for accessibility compliance.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **CopilotKit vs assistant-ui hands-on evaluation** — Both have FastAPI support, but which integrates more cleanly with the existing shadcn/ui component set? Needs a prototype.
2. **Claude as the GenUI model** — If FileScience uses Claude (via Bedrock) as the LLM backend, how well does it perform at structured JSON generation for component catalogs vs. OpenAI?
3. **Latency budget** — NL query → rendered component has an inherent latency (LLM inference + streaming). Is this acceptable for IT/MSP operators who expect sub-second dashboard interactions?
4. **Cost model** — Every GenUI interaction requires an LLM call. What's the per-query cost at scale (hundreds of MSP operators, dozens of tenants each)?
5. **When to start** — GenUI is Phase 2/3 work. The dashboard doesn't exist yet. Should GenUI considerations influence Phase 1 component architecture (e.g., ensuring all shadcn/ui components are CopilotKit-compatible from the start)?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-deep-generative-ui-use-cases.md`
