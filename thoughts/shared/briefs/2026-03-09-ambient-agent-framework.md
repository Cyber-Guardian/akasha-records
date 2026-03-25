# Idea Brief: Ambient Agent Framework

**Date:** 2026-03-09
**Status:** Shaped → Parked

## Problem
Short-running task agents are well-understood, but there's no good internal framework for persistent agents that run continuously, maintain state, and push insights — the "ambient agent" pattern. As a solo founder, strategic thinking, competitive monitoring, customer signal synthesis, and operational awareness all compete for attention with building. Always-on agents could handle the continuous perception layer.

## Constraints
- Must be cheap enough to run 24/7 (naive heartbeat = 170K tokens/tick — need "cheap check first, LLM on escalation" pattern)
- Signal-to-noise is everything — noisy agents get ignored
- Should build for one use case first, then extract the framework
- Existing substrate: memory-bank + QMD, helm orchestration, triage agent (seed ambient agent), hook system
- Input sources available: Linear, Intercom, Slack, GitHub, AWS CloudWatch, web scraping

## Options Considered

### Chief of Staff Briefing Agent
Single agent that aggregates Linear, GitHub, Intercom, calendar into a daily morning push. Tells you what matters today. Natural evolution point into specialized sub-agents.
- Gains: Bundles multiple signals, immediate daily value, low-risk starting point
- Costs: Breadth over depth — jack of all trades
- Complexity: Medium

### Customer Signal Synthesizer
Always-on agent monitoring Intercom + Slack for customer patterns, churn risk, feature request themes. Surfaces weekly digest + real-time alerts for high-signal moments.
- Gains: Directly revenue-connected, clear signal sources, measurable value
- Costs: Narrower scope, requires good Intercom/Slack integration
- Complexity: Medium

### Competitive Watchdog
Monitors competitor websites, pricing, job postings, app store listings. Delivers diffs when something changes.
- Gains: Nobody else is watching the market, event-driven (low noise)
- Costs: Web scraping is fragile, signal may be infrequent
- Complexity: Low-Medium

### SRE Sentinel
Interprets CloudWatch/Lambda signals — not just alerting but reasoning about patterns. "Cold starts increased 40% after last deploy."
- Gains: Infra runs unattended, extends existing triage agent pattern
- Costs: CloudWatch already alerts — value is in interpretation layer
- Complexity: Medium

## Chosen Approach
Not yet chosen — parked for later. **Chief of Staff** or **Customer Signal Synthesizer** are the strongest candidates for first use case.

## Key Context Discovered During Shaping

### Market landscape
- "Ambient agents" is the emerging term (Harrison Chase / LangChain + Sequoia) — push-based vs. pull-based agents
- No product exists that specifically does "living strategic roadmap" — the gap is real but broad
- OpenClaw (200K+ GitHub stars) is the dominant open-source 24/7 agent framework — heartbeat pattern, 5-tier memory, 13K+ community skills
- Proven production patterns: IT ops (PagerDuty 50% faster), customer support (Avi Medical 81% automation), ambient clinical docs (AtlantiCare 80% adoption)
- The pattern that works: cheap deterministic check first, LLM only on escalation
- Snowplow's 7 principles are the best architectural framework for ambient agents
- SDR/outbound agents DON'T work well (11x.ai 70%+ churn) — monitoring + drafting works, autonomous execution of relationship tasks doesn't

### Prior internal work
- [[2026-02-24-linear-agent-orchestration-system|Linear Agent Orchestration Brief]] — "Swarm with Autonomous Orchestrator" option is adjacent
- [[2026-02-28-macro-agent-strategy|Macro Agent Strategy]] — the broader agent development roadmap this would fit into
- Triage agent is already a seed ambient agent (event-driven, responds to signals)

### Technical primitives available
- Memory infrastructure: Letta (MemGPT), Mem0, Zep temporal knowledge graphs — all production-ready
- LangGraph provides persistence, human-interrupt, long-term memory, cron scheduling
- OpenClaw's architecture: Gateway daemon + Agent Runtime + Storage (markdown + SQLite)

## Next Step
- [Parked] → Linear backlog issue: ENG-2392
