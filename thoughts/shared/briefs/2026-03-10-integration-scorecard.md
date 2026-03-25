# Idea Brief: Integration Scorecard

**Date:** 2026-03-10
**Status:** Shaped -> Implementing (Phase 1 complete)

## Problem

FileScience has lost significant revenue ($50-100k MRR from NetDocs) and invested engineering time with near-zero return (Clio: one client at $18/mo). Both failures are business failures, not engineering ones. The team is engineers with no bizdev function — there's no systematic way to validate whether a cloud integration is worth building before committing months of engineering time.

The two failures have different root causes:
- **NetDocs**: Product worked, partnership fell through. Failure mode = fragile business dependency.
- **Clio**: Product exists but hasn't sold. Failure mode = ICP mismatch (no technical buyer, low perceived data criticality, push-based channel into a market that doesn't self-identify as needing the solution).

## Constraints

- No dedicated bizdev/market research person — solution must work for engineers
- Small team — can't afford speculative integration bets
- Current platform supports Clio, Microsoft 365, Box; 10 other clouds are prospecting pages
- V2 platform (this monorepo) is where new integrations would be built

## Options Considered

### Integration Scorecard (manual, evidence-calibrated)
Weighted scorecard with binary gates and scored dimensions. Engineers fill it out using web research before committing to a cloud. Retroactively validated against Clio/NetDocs.
- Gains: Cheap, immediate, prevents gut-feel decisions. Grounded in evidence.
- Costs: Manual research per cloud. Depends on discipline to use it.
- Complexity: Low

### Automated Market Signal Monitor
Always-on pipeline monitoring MSP communities, marketplace traction, analyst coverage, G2 gaps, search volume. Surfaces ranked opportunity list.
- Gains: Continuous intelligence. Catches emerging demand early.
- Costs: Build time. Signal quality varies. Needs tuning.
- Complexity: Medium-High

### Full Automated BizDev Associate
Agent handling full pre-sales research cycle: market sizing, ICP analysis, competitive mapping, demand scoring, recommendation briefs.
- Gains: Closest to having a real bizdev person.
- Costs: Significant build investment. Over-engineered before proving the scorecard model.
- Complexity: High

## Chosen Approach

**Integration Scorecard** with a Claude skill (/evaluate-cloud) to semi-automate the research per cloud. Start manual, automate research collection via skill, defer full pipeline (BUS-12) until scorecard criteria are validated.

## Key Context Discovered During Shaping

### From grounding research (6 claims validated):
1. Clio's customer base is 80% firms with 1-10 employees, no IT dept (Software Advice data). Product explicitly built for "no server, no IT maintenance." — **HOLDS**
2. NetDocuments installed base is 41% <50 employees (Enlyft), but purchase channel almost always involves IT director or MSP. Implementation is "not plug-and-play." — **PARTLY HOLDS** (firm size overstated, buyer type accurate)
3. DMS has stronger perceived backup need than practice management. HYCU built purpose-built backup for iManage; no equivalent for Clio. Documents are irreplaceable work product vs reconstructible structured records. — **HOLDS**
4. Cloud backup sells to technical buyers. Datto is MSP-exclusive. Veeam is 100% channel. Gartner addresses "I&O leaders." 86% of MSPs offer backup. — **HOLDS**
5. Organic search is useful but NOT reliable standalone signal. 70-80% of B2B buying in dark funnel. Search tools undercount by median 46%. OwnBackup validated via AppExchange, not SEO. — **PARTLY HOLDS**
6. Integration scorecards are established practice. Pandium published a 5-dimension weighted scorecard. Yotpo uses four-area framework. GE-McKinsey is the ancestor. — **HOLDS**

### Key insight from cross-validation:
The MSPs reaching out for NetDocs were doing so on behalf of their small-firm clients. The technical buyer was present even when the end-user firm was small. Clio fails because there's no technical intermediary in the purchase path at all.

## Next Step

Plan created: [[2026-03-10-integration-scorecard|Integration Scorecard Plan]]
Linear: Initiative "Market Intelligence", Project "Integration Scorecard", deferred BUS-12 for automated signal monitor.
