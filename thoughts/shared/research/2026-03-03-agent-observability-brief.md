---
date: 2026-03-03T20:49:42.732646+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-03-agent-observability.md
last_generated: 2026-03-03T20:49:42.732646+00:00
---

# Research Brief: 2026-03-03-agent-observability

## TL;DR

**Date:** 2026-03-03
**Type:** Research synthesis
**Status:** Complete
---
## TL;DR
Agents fail **silently, probabilistically, and at transition points** — not in the core reasoning loop. The industry now has formal failure taxonomies (MAST, Microsoft, OWASP), production-grade observability tools (Langfuse, Phoenix, Braintrust), and emerging standards (OTel GenAI semconv). The critical insight: traditional observability ("is it up?") is useless for agents — you need **semantic-quality monitoring** ("did it reason correctly?"). The compounding probability problem means a 10-step workflow at 97% per-step accuracy yields only ~72% end-to-end success.
---
## 1. What Fails — Failure Taxonomies
### MAST: Multi-Agent System Failure Taxonomy (NeurIPS 2025)
Source: [arXiv 2503.13657](https://arxiv.org/abs/2503.13657) — 1,600+ annotated traces, 7 frameworks, 4 models, kappa=0.88.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-03-agent-observability.md`
