---
type: event
created: 2026-03-13
status: active
---

# Idea Brief: Agent-Native Work Orchestration Platform

**Date:** 2026-03-13
**Status:** Shaped -> Parked

## Problem
FileScience is positioned as a cloud backup/recovery tool, but the name, infrastructure (state machines, queues, agent orchestration, DynamoDB), and Jordan's working philosophy (in-loop/out-loop agents) all suggest a bigger play: a data-first work orchestration platform that automatically tracks projects, finds efficiencies, and has AI agents execute tasks autonomously from cross-app data.

## Constraints
- Current integration breadth is narrow (3 supported clouds: Clio, M365, Box) — moat requires more
- No precedent for backup -> work orchestration (Druva/Veeam expanded to security intelligence, not work understanding)
- "Work orchestration" is severely crowded — Monday, Asana, Wrike, ClickUp all adding AI agents; 354 companies in AI agent space with $228B+ funding
- Buyer is undefined — market evidence shows buyers purchasing AI from existing PM vendors, not backup companies
- "Automatic project tracking" requires understanding work semantics from file/API data — fundamentally harder than indexing for recovery

## Options Considered

### Backup-as-Trojan-Horse
Build integrations under backup banner (proven buyer, clear value prop), then layer intelligence and agent capabilities on the same data pipes. Backup is Phase 1 of a multi-phase expansion.
- Gains: Proven market entry, integration moat builds naturally, revenue from day one
- Costs: Slow path to the vision, backup brand may limit perception
- Complexity: Low (incremental)

### Agent-Native Platform from Day One
Rebrand/reposition as work orchestration immediately. Build integrations for intelligence, not backup. Agents are the product.
- Gains: Aligned brand, attracts the right early adopters, differentiated positioning
- Costs: No proven buyer, competing against $228B+ funded space, requires breadth of integrations fast
- Complexity: High (market risk)

### Vertical-Specific Work Intelligence
Pick one vertical (e.g., legal/Clio) and build deep work intelligence for that vertical. "FileScience for law firms" — automatic matter tracking, efficiency finding, agent-assisted workflows from Clio data.
- Gains: Narrow focus, existing Clio integration, legal vertical already in product positioning
- Costs: Small TAM, vertical lock-in, may not generalize
- Complexity: Medium

## Chosen Approach
**Parked** — the vision has legs (infrastructure fits, market is real) but critical unknowns remain: buyer identity, differentiation vs. incumbents adding AI, and the data-to-understanding capability gap. Worth revisiting after integration breadth grows and the triage-agent pattern matures.

## Key Context Discovered During Grounding
- Druva has done backup -> intelligence but stayed in security/compliance, not work orchestration
- Gartner predicts 40% of enterprise apps will have task-specific AI agents by EOY 2026
- Integration moat literature says the *workflow embedding* matters more than *data access* alone
- Wrike launched autonomous AI agents in early 2026 — incumbents are moving fast
- 55% of PM software buyers cite AI as the main purchase driver (Capterra 2025)
- "filescience" name genuinely fits better for "science of work data" than "backup your stuff"

## Next Step
- [Parked] -> Linear backlog issue: BUS-13
- Revisit triggers: (1) integration count reaches 5+ supported clouds, (2) triage-agent pattern proves out agent-from-data capability, (3) a clear buyer signal emerges
