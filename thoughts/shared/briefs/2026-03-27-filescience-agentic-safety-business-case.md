---
type: event
created: 2026-03-27
status: active
touched: 2026-03-27
tags: [strategy, product, filescience, business-case, market, fundraising, series-a]
---
# Business Case: FileScience as Agentic Safety Infrastructure

**Date:** 2026-03-27
**Status:** Active
**Origin:** Shaped from product thesis + competitive analysis + market research
**Related:** [[2026-03-26-akasha-agent-era-version-control|Product Thesis: Undo Button for the Agent Era]]

---

## The One-Liner

FileScience is git for SaaS state — version control, branching, and review for everything AI agents touch. The missing infrastructure layer between "agents can act" and "agents can act safely."

---

## 1. Market Size

### The Agentic AI Market Is Exploding

| Source | 2025 | 2030 | 2032-33 | CAGR |
|--------|------|------|---------|------|
| MarketsandMarkets (AI Agents) | $7.8B | $52.6B | — | 46% |
| MarketsandMarkets (Agentic AI) | $7.1B | — | $93.2B | 45% |
| Grand View Research | $7.6B | — | $183B | 50% |
| Mordor Intelligence | $7.0B | — | $57.4B (2031) | 42% |

**Consensus:** ~$7-8B today, **$50-90B by 2030-2032**, CAGR 42-50%.

This is one of the fastest-growing enterprise technology segments ever measured.

### The Safety Layer Is a Required Subset

Every agent that acts on production systems needs safety infrastructure. The question is whether you build it or buy it.

- Gartner: **40% of enterprise apps will include AI agents by end of 2026** (up from <5% in 2025)
- IDC: Agent usage among Global 2000 to **increase 10x by 2027**
- 79% of enterprises have adopted AI agents, but only 11% run them in production — the **68-point deployment gap** is largely a trust/safety problem
- 88% of AI agents fail to reach production — the survivors return 171% ROI

**The deployment gap is a safety gap.** Enterprises want agents but can't deploy them without rollback, audit, and review infrastructure. That's our market.

### TAM Calculation

**Top-down:** Agent safety/governance is typically 5-15% of infrastructure spend.
- Conservative (5% of $52B): **$2.6B by 2030**
- Mid (10% of $93B): **$9.3B by 2032**
- Aggressive (15% of $183B): **$27B by 2033**

**Bottom-up:** Global 2000 companies deploying agents × average annual safety spend.
- 2026: 8,000 companies × 40% adoption × $50K avg = **$160M near-term**
- 2028: 8,000 × 70% × $150K avg = **$840M**
- 2030: 8,000 × 90% × $300K avg + SMB tail = **$3-5B**

**Adjacent market (data protection & recovery):**
- Data protection & recovery solutions: $9.8B (2025) → $23.6B (2030), CAGR 19.5%
- Broader data protection: $177B (2025) → $499B (2032), CAGR 15.9%
- FileScience can capture customers from BOTH the agent infra market AND the data protection market simultaneously

### Our TAM: $5-10B by 2030

Agent safety infrastructure + SaaS state versioning + data protection for agent-managed systems. New category at the intersection of three large, growing markets.

---

## 2. Competitive Landscape

### Nobody Has the Full Stack

| Competitor | What They Have | What's Missing | Revenue/Valuation |
|-----------|---------------|----------------|-------------------|
| **Rubrik** | Agent Rewind — rollback agent actions. $1.35B ARR, 80% gross margin | Not a platform. Enterprise seats, not developer API. No branching/preview. | $9.7B market cap, ~7x revenue |
| **Veeam** | Agent Commander — "Detect. Protect. Undo." | M365 only. Backup-first DNA. Not composable. | Private, est. $5B+ |
| **Cohesity** | + ServiceNow + Datadog — anomaly detection + restoration | Partnership model, not a platform. Backup-first. | Eyeing 2026 IPO at ~$7B+ |
| **StateCLI** | MCP server — checkpoint/rollback/replay for agents | File/code scope only. No SaaS state. No branching. Open source, no managed offering. | Seed stage |
| **Galileo** | Agent Control — policy enforcement plane | Prevention only. Can't help when things go wrong. Different category. | Series B, ~$150M+ val |
| **Dolt** | True git-style branching for SQL databases | SQL databases only. Not SaaS state. Not agent-aware. | Seed/Series A |

### The Gap

Nobody combines:
1. **Cross-service SaaS state versioning** (Rubrik's domain but not their API model)
2. **Git-like branching/merge/review** (Dolt's model but only for databases)
3. **MCP-native agent integration** (StateCLI's approach but file-scoped)
4. **Composable platform** (every level of abstraction accessible)
5. **Progressive scope** (file → folder → user → service → tenant)

FileScience has #1 half-built (discovery + entity graph + LINK-based versioning). We add #2-5 to create a new category.

---

## 3. The Product

### "Git for SaaS State"

Every git concept maps to a FileScience primitive:

| Git | FileScience | What It Enables |
|-----|-------------|----------------|
| `commit` | `snapshot(scope)` | Checkpoint state before agent acts |
| `branch` | `branch(name, scope)` | Isolated sandbox for agent to experiment |
| `diff` | `diff(entity, v1, v2)` | See exactly what changed |
| `merge` | `merge(branch)` | Apply approved changes to production |
| `revert` | `revert(entity, version)` | Undo to any prior state |
| PR review | `review(branch)` | Human approves agent's changes before merge |
| `blame` | `audit(entity)` | Which agent, when, why |
| `bisect` | `bisect(entity, good, bad)` | Find which action broke something |
| drift detection | `drift(scope)` | Detect out-of-band changes |

### Architecture: Every Level Accessible

```
Level 3 — Interface:      Carrie/Caroline (swappable — bring your own agent)
Level 2 — Orchestration:  Akasha platform (multi-service coordination, entity graph)
Level 1 — Building Blocks: Connectors, scope policies, safety hooks
Level 0 — Primitives:     snapshot() revert() diff() branch() merge() audit()
```

Customers choose their level. Don't like Carrie? Swap her out. Want raw primitives? Call them directly via MCP, REST, or SDK.

### Integration Surface: MCP + REST + SDK

- **MCP:** For AI agents (primary). `mcp__filescience__snapshot()` before acting.
- **REST API:** For dashboards, monitoring, CI/CD integration.
- **Python/TypeScript SDKs:** For developers building custom agents.

MCP is the consensus standard — OpenAI deprecated its Assistants API in favor of MCP. 97M monthly SDK downloads.

---

## 4. Business Model

### Entities-Protected Pricing (Auth0 Model)

| Tier | Entities | Services | Capabilities | Price |
|------|----------|----------|-------------|-------|
| Free | 100 | 1 | Read + Snapshot | $0 |
| Pro | 10K | 5 | + Revert + Diff + Branch/Merge | $49/mo |
| Business | 100K | Unlimited | + Agent-delegated + Policies + Audit | $499/mo |
| Enterprise | Unlimited | Unlimited | + Custom connectors + SSO/SCIM + Dedicated | Custom ($2K-50K/mo) |

Usage overages for operations beyond plan limits. Land with free, expand with usage.

### Revenue Expansion Model

Like Stripe/Twilio/Auth0, revenue grows as customers:
1. **Connect more services** (M365 → then Salesforce → then Slack)
2. **Protect more entities** (start with one team → expand to org)
3. **Enable more agent actions** (read-only → snapshot → branch/merge → agent-delegated)
4. **Add more agents** (one agent → fleet)

**Target NRR: 130%+** (Rubrik's NRR is 130%+, Auth0 achieved similar expansion)

---

## 5. Comparable Company Trajectories

### Auth0 — The Closest Analog

Auth0 is the pattern. They took a universal developer pain (authentication) and made it a 3-line integration. FileScience takes a universal agent pain (safety) and makes it a 1-call integration.

| Milestone | Auth0 | FileScience (projected) |
|-----------|-------|------------------------|
| Seed | $855K at $8M cap, $10K MRR, 75 customers | Current stage |
| Series A | $9.3M, ~$50M val | Target: 12-18 months, $1-2M ARR |
| Series B | $15M, Bessemer led | Target: 24-30 months, $5-10M ARR |
| Scale | $120M Series F (2020) | — |
| Exit | Acquired by Okta for **$6.5B** at ~$200M ARR (**32x ARR**) | — |

Auth0 went from $10K MRR to $6.5B exit in 8 years.

### Stripe — The Aspiration

| Milestone | Stripe |
|-----------|--------|
| Seed (2011) | $2M at $20M val |
| Series A (2012) | $18M at $100M val |
| Series B (2012) | $20M at ~$500M val |
| Series C (2014) | $80M at $1.75B val |
| Today (2026) | $19.4B revenue, **$159B valuation**, 1M+ customers |

Stripe proved that making infrastructure disappear creates massive value. Payment processing existed before Stripe — it was just painful. Agent safety infrastructure exists (Rubrik) — it's just painful and not composable.

### Rubrik — The Incumbent to Displace

| Metric | Rubrik (2026) |
|--------|---------------|
| Revenue (TTM) | $1.32B |
| Subscription ARR | $1.35B |
| Market Cap | $9.7B (~7x revenue) |
| Gross Margin | 80% |
| NRR | 130%+ |
| FCF | $253M |
| IPO | April 2024 at $5.6B |

Rubrik built a $10B company on data protection. They're adding "Agent Rewind" as a feature. We're building "Agent Rewind" as the entire product, with a developer platform model they can't replicate.

---

## 6. Series A Projection

### 2026 Series A Benchmarks

| Metric | Benchmark | FileScience Target |
|--------|-----------|-------------------|
| ARR at raise | $1-2M | $1.5M |
| Growth rate | 100%+ | 150%+ (new category) |
| NRR | 110-120% | 120%+ |
| Gross margin | 70%+ | 85%+ (software, minimal COGS) |
| Multiple | 15-30x ARR (AI infra premium) | 20-25x |

### Valuation Range

| Scenario | ARR | Multiple | Valuation |
|----------|-----|----------|-----------|
| Conservative | $1M | 15x | **$15M** |
| Base | $1.5M | 20x | **$30M** |
| Strong | $2M | 25x | **$50M** |
| Breakout | $3M+ | 30x | **$90M+** |

AI-native infrastructure startups in 2026 command **15-30x ARR at Series A**, with outliers exceeding 50x. FileScience's positioning at the intersection of agent safety (hot) + developer platform (proven model) + data protection (established market) justifies the upper end.

### Path to $1.5M ARR (18-month target)

| Quarter | Milestone | ARR |
|---------|-----------|-----|
| Q3 2026 | MCP server live (Level 0 primitives). Dogfood with Carrie. | $0 |
| Q4 2026 | Beta: 20 design partners (free). 3 connectors (M365, Clio, Box). | $0 |
| Q1 2027 | GA: Pro tier live. First 50 paying customers. | $150K |
| Q2 2027 | Enterprise pilot: 5 enterprise customers ($2K+/mo). Expand connectors. | $500K |
| Q3 2027 | Expansion: NRR kicks in. Branch/merge feature live. | $1M |
| Q4 2027 | Series A close. 200+ customers. | $1.5M+ |

### Funding Ask

| Round | Amount | Use |
|-------|--------|-----|
| Pre-seed (now) | Bootstrap / angels $500K-1M | Build MCP primitives, 3 connectors, beta program |
| Seed (Q1 2027) | $3-5M | GA launch, first sales hire, 5 more connectors |
| Series A (Q4 2027) | $15-25M at $30-50M val | Scale sales, enterprise, 20+ connectors |

---

## 7. Why Now

Five converging forces make this the moment:

1. **The deployment gap is NOW.** 79% adopted agents, 11% in production. The 68-point gap is a safety problem. First mover in safety infrastructure captures the market as agents go production.

2. **MCP just became the standard.** OpenAI deprecated its Assistants API. Google, Microsoft, Anthropic all adopted MCP. A safety MCP server instantly integrates with every agent framework. This window didn't exist 12 months ago.

3. **Incumbents are slow.** Rubrik, Veeam, Cohesity are bolting agent features onto backup products. They can't become developer platforms — their sales motion, pricing, and architecture prevent it. This is the Stripe-vs-banks window.

4. **We have the hard piece.** Discovery engine, entity graph, multi-cloud connectors, DynamoDB versioning model. Building this from scratch takes 2+ years. We're 18 months ahead.

5. **Enterprise spend is committed.** $500K-$1B annually per enterprise on agentic workloads. Safety is a line item, not a nice-to-have. Regulated industries (finance, healthcare, legal) won't deploy agents without auditable rollback.

---

## 8. The 5-Year Vision

| Year | Product | Revenue | Milestone |
|------|---------|---------|-----------|
| 2026 | MCP primitives + 3 connectors | $0 (beta) | Dogfood + design partners |
| 2027 | GA + branch/merge + 8 connectors | $1.5M ARR | Series A |
| 2028 | Enterprise + 20 connectors + Carrie | $10-15M ARR | Series B |
| 2029 | Platform SDK + community connectors | $40-60M ARR | Category leader |
| 2030 | Full composable stack | $100-200M ARR | Series C / growth |

At $200M ARR with 130% NRR and 85% gross margin, comparable exits are:
- **Auth0 path (32x):** $6.4B acquisition
- **Rubrik path (7-10x):** $1.4-2B public company
- **Stripe path (8x revenue):** $1.6B+ (and growing)

The Auth0 path is most likely — Rubrik, Cohesity, CrowdStrike, Palo Alto, or a cloud hyperscaler acquires the category leader in agent safety.

---

## 9. The Pitch (30 seconds)

> "AI agents are about to manage everything — your files, your CRM, your finances, your infrastructure. They're going to make mistakes. Right now there's no git for the stuff agents touch. No branching, no review, no undo.
>
> FileScience is that missing layer. One API call to snapshot before an agent acts. One call to branch so an agent can experiment safely. One call to show a human what the agent wants to change. One call to merge or reject.
>
> We already have the discovery engine, entity graph, and cloud connectors. The data protection market is $180B. The agent market is $50B and growing 45% a year. We sit at the intersection.
>
> We're raising to go from beta to GA, add branch/merge, and capture the market before Rubrik figures out how to be a developer platform."

---

## Sources

- [MarketsandMarkets — AI Agents Market $52.6B by 2030](https://www.marketsandmarkets.com/PressReleases/ai-agents.asp)
- [Grand View Research — AI Agents $183B by 2033](https://www.grandviewresearch.com/industry-analysis/ai-agents-market-report)
- [Mordor Intelligence — Agentic AI Market](https://www.mordorintelligence.com/industry-reports/agentic-ai-market)
- [Digital Applied — 150+ Agentic AI Statistics 2026](https://www.digitalapplied.com/blog/agentic-ai-statistics-2026-definitive-collection-150-data-points)
- [Rubrik IPO S-1 Breakdown — Meritech Capital](https://www.meritechcapital.com/blog/rubrik-ipo-or-s-1-breakdown)
- [Cohesity Eyes 2026 IPO — CNBC](https://www.cnbc.com/2025/09/04/nvidia-backed-cohesity-eyes-2026-ipo-with-valuation-rivaling-17-billion-rubrik.html)
- [SaaS Valuation Multiples 2026 — ScaleWithCFO](https://www.scalewithcfo.com/post/saas-valuation-multiples-2026)
- [AI Startup Valuation Multiples 10-50x — Qubit Capital](https://qubit.capital/blog/ai-startup-valuation-multiples)
- [DoltHub — Cursor Says Agents Need Database Branches](https://www.dolthub.com/blog/2025-06-05-cursor-database-branches/)
- [Rubrik Agent Rewind](https://www.rubrik.com/products/agent-rewind)
- [Veeam Agent Commander — TechTarget](https://www.techtarget.com/searchdatabackup/news/366639456/Veeam-releases-Agent-Commander-to-combat-agentic-AI-risk)
- [StateCLI MCP Server — GitHub](https://github.com/statecli/mcp-server)
- [Agent Sandboxing in 2026 — Northflank](https://northflank.com/blog/how-to-sandbox-ai-agents)
- [GitHub Copilot Coding Agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [MCP vs API Enterprise Guide — BuzzClan](https://buzzclan.com/ai/mcp-vs-api/)
- [Stripe Revenue $19.4B — Backlinko](https://backlinko.com/stripe-users)
- [Auth0 Acquired by Okta for $6.5B](https://auth0.com/blog/auth0-raises-series-a-led-by-bessemer/)
