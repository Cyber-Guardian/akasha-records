# Idea Brief: Deep Research System

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
Current research infrastructure (`/research_codebase`) does a single fan-out of parallel subagents, synthesizes once, and writes a research doc. This works for 5-10 minute codebase exploration but falls short for truly deep, holistic research — there's no iterative deepening, no gap identification, no "go deeper on finding X," and no way to sustain a 30-60 minute research session across a complex topic.

## Constraints
- Must work within Claude Code's tool system (Agent tool for subagents)
- Context window is the binding constraint — deep research generates massive text
- Should integrate with existing memory-bank for persistence
- Should be composable — usable for codebase research, external research, or both
- Must not replace `/research_codebase` — opt-in enhancement for when depth is needed

## Options Considered

### Research Harness CLI (helm-style Python tool)
Standalone Python CLI (`tools/research/`) orchestrating via `claude -p` subprocesses, like helm does for implementation.
- Gains: Full process control, resumable, can add non-LLM tools (vector search, embeddings)
- Costs: Heavy build (~2-3 weeks), separate from conversation flow, overkill for many tasks
- Complexity: High

### Deep Research Skill (Claude Code extension with externalized state)
A `/deep_research` skill running the iterative loop inside the conversation, externalizing state to a research manifest file (gaps, findings, plan) that survives context compression.
- Gains: Interactive (user steers mid-research), uses existing Agent tool, integrates with memory-bank, moderate build (~1 week), composable with existing skills
- Costs: Bound by session lifetime, main context is supervisor so very long sessions hit compression
- Complexity: Medium

### Enhanced Research Codebase (iterative upgrade to existing skill)
Add a reflection/gap loop to existing `/research_codebase` — after first fan-out, check for gaps, fan out again, repeat N times.
- Gains: Smallest change, no new infrastructure, immediately usable
- Costs: Doesn't solve context management, no externalized state, limited depth (2-3 iterations)
- Complexity: Low

## Chosen Approach
**Deep Research Skill** — The supervisor isolation pattern works within Claude Code. Externalized state (research manifest file) is the key unlock — tracks plan, gaps, findings, sources. Distillation-first context management (subagents return summaries, not raw content) keeps the main context lean. Interactive steering is the killer feature over a CLI tool.

## Key Context Discovered During Shaping

### Industry patterns worth stealing:
- **Distillation-first** (Tavily) — compress tool outputs to reflections immediately, achieving linear not quadratic token growth
- **Explicit gap tracking** (Enterprise DR) — maintain a gaps list, terminate when empty or budget exhausted
- **Perspective enumeration** (STORM) — identify distinct angles of inquiry before decomposing into searches
- **Plan exposure** (Gemini) — show research plan to user before committing compute
- **Source dedup** (Tavily/Enterprise DR) — track seen sources, detect scope narrowing as termination signal

### Existing infrastructure to build on:
- `/research_codebase` skill — single-round fan-out pattern, subagent types (locator, analyzer, pattern-finder, thoughts-locator, thoughts-analyzer, web-search-researcher)
- `research_brief.py` hook — auto-generates `-brief.md` companions for research docs
- Memory-bank persistence — `memory-bank/thoughts/shared/research/` with YAML frontmatter
- Helm manifest pattern — externalized JSON state that survives across invocations

### Key commercial/open-source systems researched:
- OpenAI Deep Research: 4-agent pipeline (triage → clarify → instruct → research), clarification-before-commitment
- Google Gemini Deep Research: single-agent iterative loop with gap identification, user-approved plan, million-token context
- Perplexity Deep Research: plan-and-execute with multi-model routing, pre-processed ranked context for synthesis
- GPT-Researcher: chief editor + parallel researchers + reviewer + writer, evaluated on DeepResearchGym
- LangChain Open Deep Research: supervisor-subagent with compress_research node, `Send` primitive for parallel dispatch
- Stanford STORM: perspective-driven multi-expert Q&A simulation, highest source diversity
- Tavily Deep Research: distillation-first (66% token reduction), global source registry, scope narrowing termination
- Enterprise DR (arxiv:2510.17797): 5 specialized search agents, stateful task objects, running summaries, user steering

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-04-deep-research-system.md`
