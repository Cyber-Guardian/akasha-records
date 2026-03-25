# Idea Brief: Stripe Minions Brainstorm Compilation

**Date:** 2026-03-09
**Status:** Shaped → Parked (partially — Thread A+B shaped further in [[2026-03-09-plugin-architecture-and-helm-intelligence|Plugin Architecture + Helm Intelligence]])

## Problem
Raw brainstorm session surfaced 13+ ideas inspired by Stripe's Minions architecture, agentic coding talks, and product strategy thinking. Needed compilation, deduplication against prior work, and triage of what's genuinely new vs already covered.

## Idea Clusters

### Cluster 1: Blueprint Engine Pattern (Stripe-inspired harness evolution)
Interleave deterministic + agentic nodes in workflow graphs. Pre-push lint daemons. Bounded CI iteration (max 2 cycles). Team-customizable blueprints.
- **Prior work:** Helm V1 (deterministic state machine), PostToolUse hooks (point checks)
- **Delta:** Blueprint-as-graph formalism upgrades Helm's execution model
- **Status:** Feeds into [[2026-03-09-plugin-architecture-and-helm-intelligence|Helm Intelligence brief]]

### Cluster 2: Devbox Pool / Agent Sandboxes
Warm pool of pre-provisioned environments. 10-second startup. Mac minis as agent farm. Better than cold git worktrees.
- **Prior work:** [[2026-02-28-agent-swarm-harness|Swarm Harness Brief]] (Windows dev server)
- **Delta:** Mac mini pool + warm environments replaces Windows server assumption
- **Status:** Parked — hardware decision, not software architecture

### Cluster 3: In-loop vs Out-of-loop Execution
Codev (human-supervised) vs autonomous. Hand off more to out-of-loop. Zero-touch engineering. Meta-agentics.
- **Prior work:** [[2026-02-24-linear-agent-orchestration-system|Linear Orchestration Brief]] defined autonomous/co-dev modes
- **Delta:** Strategic framing shift, not a new system. Concrete next step: expand what qualifies as autonomous.
- **Status:** Feeds into [[2026-03-09-plugin-architecture-and-helm-intelligence|Helm Intelligence brief]]

### Cluster 4: MCP Toolshed
Curated tool subsets per agent type. ~500 tools at Stripe, agents get limited defaults.
- **Prior work:** None — all agents currently get all tools
- **Delta:** Tool curation per agent role
- **Status:** Parked — Helm V2 feature

### Cluster 5: Deep Research Visualization
Tree/graph growing as agents gain knowledge. Shows all search paths.
- **Prior work:** `/deep_research` skill with manifest tracking but no visualization
- **Delta:** Visual renderer for research process
- **Status:** Parked — small, could implement directly when ready

### Cluster 6: Product/Business Ideas
- AI knowledge base for sales (Intercom/fin.ai) — **new, parked**
- Morning brief AI — **new, parked**
- Plugin architecture for integrations — **shaped further** → [[2026-03-09-plugin-architecture-and-helm-intelligence|Plugin Architecture brief]]
- pfst + deterministic codegen — **new, parked**
- Automate integration development — **shaped further** (same brief)

## Key References
- [Stripe Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [pi-mono coding agent](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent)
- [Goose (Block)](https://github.com/block/goose)
- [pfst 0.3.0](https://www.reddit.com/r/Python/s/0rqerf1WHU) — Python source manipulation

## Next Step
- [Parked] — individual clusters parked; Thread A+B shaped into separate brief
