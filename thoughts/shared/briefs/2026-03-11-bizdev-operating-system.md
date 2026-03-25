# Idea Brief: BizDev Operating System

**Date:** 2026-03-11
**Status:** Shaped → Planning

## Problem
FileScience has a capable sales director who converts inbound demos but has zero pipeline generation, no strategic targeting, and no market intelligence feeding him. He's reactive by nature — great at conversations, demos, and building trust, but doesn't self-direct strategy or outbound. Jordan sets strategy but has limited time to execute GTM. There's no system connecting market intelligence → targeting → pipeline → conversations → learning. Whether the campaign is HIPAA, NetDocuments, or something new, the same infrastructure gap exists.

The team has done extensive market intelligence (20+ cloud scorecards, HIPAA pivot thesis, competitive analysis) but none of it flows systematically to the person who actually talks to prospects. Apollo is in use but as a rolodex, not as part of a system.

## Constraints
- Jordan has limited weekly time for GTM — system must reduce his per-cycle overhead, not add to it
- Sales director needs "call X because Y" briefs, not "figure out strategy" — he's a closer, not a strategist
- HIPAA pivot is a thesis, not validated buyer behavior — must test before automating outreach
- No dedicated GTM engineer — intelligence layer must be buildable by the engineering team or AI agents
- Apollo already in use — build on it, don't replace it
- HIPAA marketing window peaks Q3-Q4 2026 — time-sensitive but uncertain (rule finalization "more aspirational than a deadline")

## Options Considered

### Cadence-First (Manual Process Before Tools)
Define weekly operating rhythm: Monday Jordan writes briefs, week sales director executes, Friday feedback form, monthly review. No new tools.
- Gains: Cheapest, forces feedback loop to exist, exposes what needs automating
- Costs: Heavy on Jordan's weekly time. Doesn't scale. Manual intelligence is inconsistent.
- Complexity: Low

### Automated Intelligence + Human Execution
Build automated intelligence layer (competitive monitoring, signal detection, enriched target lists, content drafts). Jordan reviews weekly instead of researching. Sales director executes manually from prepared briefs.
- Gains: Reduces Jordan's time per cycle from hours to minutes. Campaign-agnostic. Foundation for any future GTM motion.
- Costs: Build time for intelligence layer. Risk of building infra that doesn't get used if no process wrapper.
- Complexity: Medium

### Full GTM Stack (Composable Tools End to End)
Apollo + Clay + Instantly/Smartlead + n8n + AI content pipeline. System generates intelligence, builds lists, enriches, drafts outreach, sends sequences, routes replies.
- Gains: Highest throughput. ~200-400 touches/week. Equivalent to half-time SDR for $300-500/month.
- Costs: Significant setup. Requires domain hygiene discipline. "GTM engineering fails when companies lack repeatable sales processes" — and we don't have one yet.
- Complexity: High

## Chosen Approach
**Automated Intelligence + Human Execution** with a thin process wrapper and validation sprint. Grounding research validated:
- Intelligence layer is universally Stage 1 in every GTM maturity model — no source argues against building it early
- Automating research/intelligence ≠ automating outreach — the "don't automate before PMF" warning applies to outreach, not intelligence
- A lightweight handoff protocol is still needed (not 4-6 weeks of manual grinding, but a thin wrapper)
- Validation with ≤50 contacts should happen before any outreach automation

## Scope (Plannable Now)

### 1. Automated Intelligence Layer (campaign-agnostic)
- **Competitive monitoring:** Weekly tracking of Veeam, AvePoint, Druva, Datto, Keepit for healthcare/HIPAA positioning changes, pricing, feature announcements, G2 review sentiment
- **Regulatory tracking:** HIPAA Security Rule finalization monitoring (CMS.gov, HHS.gov, healthcare trade press). Alert when rule status changes.
- **Signal detection:** Healthcare MSP hiring patterns, community discussions (r/msp, MSPGeek), backup vendor churn signals
- **Enriched target lists:** Healthcare MSP identification + enrichment from Apollo, ChannelE2E Top 100, CloudTango directory, named targets (Anatomy IT, Medicus IT, CompassMSP, MedicalITG, Meriplex, Dataprise)

### 2. Lightweight Handoff Protocol
- Weekly intelligence digest (1-page, AI-generated, Jordan reviews)
- 5-10 target cards per week ("reach out to X because Y, here's the pitch angle")
- Sales director feedback loop (what resonated, what objections, what signals)
- Monthly strategy review (Jordan adjusts ICP/messaging based on accumulated feedback)

### 3. Validation Sprint (HIPAA Healthcare MSP)
- First 50 targets from intelligence layer output
- Jordan writes 2-3 messaging angles based on grounded findings:
  - "Less than 43% of MSPs meet HIPAA backup standards" hook
  - "Built for when the rule finalizes, not if" positioning
  - "Compliance dashboard, not backup dashboard" differentiation
- Sales director executes personalized outreach with scripts
- Goal: validate which message + ICP segment converts before any outreach automation

### Deferred (Follow-on after validation)
- Content pipeline (AI-drafted HIPAA content informed by validation learnings)
- Outreach automation (Clay enrichment + Instantly/Smartlead sequencing)
- Pax8 marketplace listing strategy
- Full GTM stack buildout

## Key Context Discovered During Shaping

### From research (3 agents, 100+ sources):
- GTM Engineering is a formal discipline: Clay + Apollo + n8n is the canonical stack. Companies hire first GTM engineer at $5-10M ARR.
- Hybrid AI+human outperforms either alone: 2x meetings (117 vs 56/month), 35% higher close rates
- Small cohorts (≤50) yield 2.76x better results than mass campaigns
- 84% of B2B buyers pick vendor before contacting sales (dark funnel dominates)
- AI SDR underperforms humans 2.6x on revenue ($56K vs $147K) but costs 54x less
- Cold email spam problem is real: inbox placement dropped from 50% to 28% for high-volume senders
- At $2,400-$24,000 ACV: warm outbound + SEO + LinkedIn are the right motions

### From healthcare MSP research:
- No M365 backup vendor has healthcare-specific product — everyone just says "HIPAA-compatible"
- Keepit is the only competitor creating healthcare-specific content (one blog post)
- Pax8 (~30K MSP partners) is the highest-leverage distribution move
- MSP purchasing dominated by peer recommendation, not product specs
- Less than 43% of MSPs meet HIPAA backup compliance (Axcient/Infrascale survey)
- 52% of organizations plan to switch backup providers within 12 months
- HIPAA rule finalization is uncertain under current administration — "more aspirational than a deadline"
- Named healthcare MSPs: Anatomy IT, Medicus IT, CompassMSP, MedicalITG, Meriplex, Dataprise

### From grounding:
- Intelligence layer is Stage 1 in every GTM maturity model — universally supported as pre-automation foundation
- Automating intelligence ≠ automating outreach — the premature automation warning applies to messaging, not research
- Market research validates potential; only buyer conversations validate behavior. FileScience has the former, needs the latter.

## Related Artifacts
- [[2026-03-11-healthcare-hipaa-pivot|Healthcare HIPAA Pivot Brief]] — the first campaign to run through the system
- [[2026-03-10-cloud-ranking-summary|Cloud Integration Ranking]] — market intelligence foundation
- [[2026-03-10-integration-scorecard|Integration Scorecard Brief]] — the evaluation framework
- [[2026-03-05-deep-emergent-markets-for-filescience|Emergent Markets Research]] — version intelligence vision
- [[2026-01-28-filescience-product-context-expanded-notes|Product Context Notes]] — current positioning

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-11-bizdev-operating-system.md`
