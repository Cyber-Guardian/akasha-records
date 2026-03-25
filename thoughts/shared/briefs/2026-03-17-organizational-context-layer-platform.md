---
title: "FileScience as Organizational Context Layer"
type: state
status: active
created: 2026-03-17
touched: 2026-03-17
tags: [strategy, platform, vision, MCP, API, AI-agents]
related:
  - thoughts/shared/research/2026-03-14-deep-filescience-brand-product-web.md
---

# FileScience as Organizational Context Layer

## Core thesis

FileScience's backup product accidentally built the foundation for an AI agent substrate. Backup gives us authenticated, persistent connections to every cloud service a company uses, a continuously updated data mirror, historical depth, and trust — all prerequisites that every AI agent platform needs and nobody else has pre-solved.

**Reframe:** Backup sells insurance (recovery). The platform sells memory (organizational intelligence). Same data, different value proposition.

## The insight

Every AI agent startup is building integrations from scratch, begging for OAuth tokens, starting with zero context. FileScience already has the connections, the data, and the trust — earned through the most boring, trustworthy ask possible ("can I back up your data?").

The company that can undo anything is the company you trust to do anything.

## Four layers

### Layer 1 — Sync Engine (exists today)
Continuous authenticated read from Clio, M365, Box. Normalized data model. Full history. This IS the backup product, reframed as an organizational data replication layer.

### Layer 2 — Entity Graph (the intelligence leap)
- **Cross-service entity resolution** — one person/matter/document unified across all systems
- **Relationship mapping** — matters -> documents -> emails -> calendar events -> files, regardless of which system they live in
- **Temporal dimension** — not just current state but how entities and relationships change over time (backup heritage = superpower here)
- **Semantic indexing** — understand what documents are about, not just their metadata

### Layer 3 — API / MCP Surface
```
Resources (read)
  entities/{id}         - unified view across all services
  graph/connections      - relationship traversal
  timeline/{entity}      - activity over time
  search                 - semantic search across everything
  snapshots/{date}       - point-in-time organizational state
  diff/{from}/{to}       - what changed between two points

Tools (write)
  create, update, delete - scoped operations on source systems
  simulate               - dry-run against backup snapshot
  automate               - define recurring flows

Prompts (context packages)
  org-context            - organization overview
  entity-brief           - everything about {entity}
  change-digest          - what happened recently
```

### Layer 4 — Agent Runtime (full platform)
- Scoped permissions per agent per system
- Simulation/dry-run against backup snapshots (zero-risk testing for free)
- Audit trail for every agent action
- Rollback capability (backup DNA)
- Approval gates for high-impact operations

## Defensibility

1. **Data gravity** — once you're the organizational context layer, switching means rebuilding everything + losing history
2. **Cold start moat** — competitors need to convince companies to connect all services. We already have the connections.
3. **Network effects** — each new service makes the graph exponentially more valuable
4. **Temporal irreplaceability** — you can replicate current state, you cannot replicate history. Every day deepens the advantage.
5. **Trust compounding** — backup trust -> read API trust -> write API trust -> agent execution trust. Each step earned.

## Go-to-market sequencing

| Phase | What | Value prop shift |
|-------|------|-----------------|
| 0 (now) | Backup product | Insurance — recover when things go wrong |
| 1 | Read API over backup data | Programmatic access — search/query your data |
| 2 | Entity graph + intelligence | Understanding — we know your organization |
| 3 | MCP server | Platform — any AI tool plugs into your org context |
| 4 | Write-back + agent runtime | Substrate — AI agents operate through FileScience |

Each phase funds the next. Backup never goes away — it's the trust foundation and data engine.

## Revenue model evolution

- **Backup:** per-seat/per-GB flat fee (commodity pressure)
- **API access:** usage-based (scales with value)
- **MCP/Platform:** seat-based for connected AI tools + usage
- **Agent runtime:** execution-based (pure upside)

## Vertical wedge

Start with legal/professional services (Clio anchor):
- Underserved by AI tooling
- High-value data relationships (matters -> documents -> communications)
- Compliance requirements favor audit/rollback capabilities
- Billable hours drive clear ROI math

Architect horizontally from day one — adding a new cloud service should be a connector problem, not an architecture problem.

## Adjacent product angles (from earlier brainstorming)

### Auto-research on customer data
- Operational intelligence: usage patterns, growth, staleness, risk signals
- Compliance/risk: data residency analysis, retention gap detection, audit trail synthesis
- Cross-cloud correlation: unified view that no single SaaS tool has
- Cost optimization: dedup across services, license utilization, tiering recommendations

### AI-native automation
- Agent builds flows from natural language intent
- Visual canvas UI for editing the artifact
- Test against backup snapshots before deploying
- Low-code but the AI does the building, human refines

## Key risk

Source services (Clio, Microsoft, Box) restricting API access. Mitigation: backup data portability has legal protection, and the platform drives usage of their services rather than competing. Worth monitoring.

## Open questions

- What's the minimum viable entity graph that proves cross-service value?
- How fresh does data need to be for platform use cases vs backup cadence?
- Permission model for API/MCP — how granular, who configures?
- Pricing for platform tiers — what's the anchor?

## Origin

Emerged from a brainstorming session exploring: auto-research agent swarms -> system-level optimizations -> research-as-a-service on customer data -> automation platform -> organizational context layer / AI agent substrate. Each idea built on the last.
