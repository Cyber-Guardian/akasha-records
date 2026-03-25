# Cloud Integration Ranking Summary

**Date:** 2026-03-10
**Scope:** All clouds evaluated across the prospecting exercise (validation, legal tech deep research, original batch, quick-scored, pre-triaged)
**Framework:** Integration Scorecard v1 (see `memory-bank/durable/scorecard-template.md`)

---

## Full Ranked Table (Gates-Passing Clouds Only)

Sorted by weighted score descending. Only clouds that passed both gates (including conditional passes) are ranked.

| Rank | Cloud | Score | Band | G1 | G2 | Phase | Scorecard |
|------|-------|-------|------|----|----|-------|-----------|
| 1 | Microsoft 365 | 8.85 | Strong Go | PASS | PASS | Validation | [[2026-03-10-scorecard-m365\|M365 Scorecard]] |
| 2 | NetDocuments | 7.10 | Strong Go | PASS | PASS | Validation | [[2026-03-10-scorecard-netdocs\|NetDocs Scorecard]] |
| 3 | Box | 6.63 | Weak Go | PASS | PASS | Validation | [[2026-03-10-scorecard-box\|Box Scorecard]] |
| 4 | Filevine | 6.15 | Weak Go | COND | PASS | Legal Deep Research | [[2026-03-10-scorecard-filevine\|Filevine Scorecard]] |
| 5 | DocuSign | 6.05 | Weak Go | COND | PASS | Legal Deep Research | [[2026-03-10-scorecard-docusign\|DocuSign Scorecard]] |
| 6 | iManage | 5.50 | Weak Go | PASS | PASS | Legal Deep Research | [[2026-03-10-scorecard-imanage\|iManage Scorecard]] |
| 7 | Smartsheet | 5.45 | Borderline | COND | PASS | Original Batch | -- |
| 8 | Zoom | 5.45 | Borderline | COND | COND | Original Batch | -- |
| 9 | Zoho Docs | 5.05 | Borderline | COND | PASS | Original Batch | -- |
| 10 | Slack | 4.80 | Borderline | COND | COND | Original Batch | -- |

## Gate Failures and No-Go Clouds

These clouds either failed a gate criterion or scored below 4.0, making them unsuitable for investment.

| Cloud | Score | Verdict | Reason |
|-------|-------|---------|--------|
| Clio | 4.55 | Borderline, G1 FAIL | 80% solo/small firms without IT. No technical buyer. Structural ICP mismatch. |
| Actionstep | ~3.95 | No-Go | Small-firm legal practice management. Same G1 problem as Clio with smaller market. |
| Assembly Neos | ~3.5 | No-Go | Niche legal PM. Insufficient market size and no technical buyer path. |
| 8am/MyCase/LawPay | 3.30 | No-Go, G1 FAIL | Confirmed Clio repeat. 64% at 2-10 employee firms. 1/10th Clio's market. Zero demand signals. |
| PracticePanther | Gated | No-Go, G1 FAIL | Solo/small firm legal PM. Same structural failure as Clio and MyCase. |
| Lawmatics | Gated | No-Go, G1+G2 FAIL | Legal CRM/intake. Fails both gates -- no IT buyer and data is reconstructible (leads, intake forms). |

## Skipped (Pre-Triaged)

These clouds were excluded before scoring due to clear disqualifiers.

| Cloud | Reason |
|-------|--------|
| Litify | Already served -- OwnBackup, Spanning, and Druva cover Salesforce platform (Litify is built on Salesforce). |
| Smokeball | Native backup exists -- Smokeball provides built-in backup, eliminating the market need. |
| Caret Legal | Too small -- $2.1M estimated revenue. Market cannot justify integration investment. |
| Supio | Too small -- $1M estimated revenue. Pre-scale AI legal tool. |
| Docketwise | Too small -- $2.4M estimated revenue. Immigration-only niche. |

---

## Top 3 Recommendations

### 1. NetDocuments -- Primary Legal Vertical Target

**Score:** 7.10 (Strong Go) | **Gates:** PASS / PASS

NetDocuments is the clear winner in the legal vertical. Zero existing backup products (untapped market score: 9), proven CIO-led purchase path, 7,000+ organizations, and 4-5 independent partnership paths including legal MSPs, implementation partners, and marketplace. The HYCU/iManage playbook applies directly -- iManage has HYCU as incumbent; NetDocuments has nobody. API is feasible (REST v2, OAuth 2.0, Postman-governed). ndMirror (native sync tool) has documented gaps that backup solves.

**Why first:** Highest-scoring legal cloud. Zero competition. Growing market (eDOCS acquisition expanding base). Legal MSP channel is established and asking for this.

### 2. Microsoft 365 -- Maximum TAM Play

**Score:** 8.85 (Strong Go) | **Gates:** PASS / PASS

Highest absolute score. 430M+ users, 94% penetration but only 6% protected by third-party backup. New Backup Storage API (GA Aug 2024) creates differentiation surface. Competition is intense (20-30 vendors) but the unprotected base is enormous. MSP channel is the strongest of any cloud evaluated (score: 10).

**Why second (despite highest score):** Competition is fierce and incumbents are well-funded. NetDocuments offers a greenfield opportunity with zero competition, making it a better first integration where early wins build credibility. M365 should be the scale play once the product and team are proven.

### 3. Box -- Enterprise Content Adjacency

**Score:** 6.63 (Weak Go) | **Gates:** PASS / PASS

Solid secondary market. Enterprise buyer with IT procurement involvement. 100K+ paying organizations. API is the strongest evaluated (full CRUD, webhooks, 1,000 req/min). Competition is limited (~8 vendors vs 23+ for M365). Natural adjacency to M365 and NetDocuments -- many enterprise legal departments use Box alongside these tools, enabling multi-cloud backup positioning.

**Why third:** Good standalone fundamentals, but the real value is as a bundle play. "We protect your NetDocuments, M365, and Box" is a stronger pitch than Box alone.

---

## Clio Recommendation: Cut Losses

Clio fails G1 structurally. The finding is not a close call -- 80% of users are at 1-10 employee firms with no IT department and no MSP involvement. The 4.55 Borderline score (even ignoring the gate failure) reflects this reality. The deep research phase confirmed this pattern repeats across all small-firm legal practice management tools:

- **Clio:** G1 FAIL, 4.55
- **8am/MyCase/LawPay:** G1 FAIL, 3.30
- **PracticePanther:** G1 FAIL, gated
- **Actionstep:** ~3.95, same ICP

This is a systematic pattern, not a one-off finding. Practice management tools for small law firms do not have a technical buyer in the purchase path. The backup sale requires an IT decision-maker or MSP, and this market segment has neither. Any existing Clio investment should be wound down or deprioritized in favor of NetDocuments, which serves the same legal vertical but targets CIO-led organizations.

---

## Key Insights

### 1. Legal DMS is the clear winner; practice management systematically fails G1

The legal vertical splits cleanly into two categories:
- **Document Management Systems** (NetDocuments, iManage): CIO-led purchase, MSP ecosystem, enterprise IT involvement. Both pass G1.
- **Practice Management** (Clio, MyCase, PracticePanther, Actionstep, Lawmatics): Attorney/admin-led, solo/small firms, no IT department. All fail G1.

This is the single most important finding of the evaluation. It means the legal vertical opportunity is real but narrower than it appears -- it lives in DMS, not PM.

### 2. The MSP/channel dimension is the strongest differentiator across all clouds

MSP/Channel Interest (20% weight) is the dimension that most cleanly separates Strong Go from Borderline:
- M365: 10, NetDocuments: 7, Box: 5.5 (the top 3)
- DocuSign: 3, 8am/MyCase: 2 (the bottom)

Clouds where MSPs are active create a self-reinforcing sales channel. Clouds without MSP involvement require expensive direct sales to non-technical buyers. This weighting is correct and validated by the evaluation outcomes.

### 3. "Untapped market" requires both gap AND demand

The scoring rubric's insistence that high untapped market scores require both a gap and evidence of demand proved critical:
- NetDocuments: gap (zero backup products) + demand (ILTA discussions, partner guides, deletion policy) = 9
- 8am/MyCase: gap (no backup products) + no demand (zero community requests, no analyst coverage) = 3

An empty market with no demand is a dead market, not a greenfield opportunity.

### 4. Conditional G1 passes cluster at Weak Go

DocuSign (6.05), Filevine (6.15), and the Borderline clouds (Smartsheet, Zoom, Zoho, Slack) all have conditional G1 passes. The restricted scope narrows the addressable market, which mechanically depresses MSP/Channel and Market Size scores. Conditional G1 is a real signal that the opportunity exists but is constrained.

### 5. HYCU is the recurring incumbent in legal DMS

HYCU appears in three evaluations: iManage (official partner), Box (entered Nov 2024), and NetDocuments (absent -- the gap). Understanding HYCU's positioning is critical for competitive strategy. Their absence from NetDocuments is the clearest market signal in the entire exercise.

---

## Confidence Assessment

| Category | Confidence | Notes |
|----------|-----------|-------|
| Validation clouds (M365, NetDocs, Box, Clio) | High | Extensive research, multiple T1 sources, sensitivity analysis performed |
| Legal deep research (Filevine, DocuSign, iManage) | Moderate | Good T1/T2 coverage but fewer independent sources than validation clouds |
| Original batch (Smartsheet, Zoom, Zoho, Slack) | Moderate | Scored during framework development; may benefit from re-evaluation with refined weights |
| Quick-scored (Actionstep, 8am/MyCase, Assembly Neos) | Moderate-Low | Lighter research depth; sufficient for No-Go determination but not for Go decisions |
| Pre-triaged (Litify, Smokeball, Caret, Supio, Docketwise) | High (for exclusion) | Clear disqualifiers verified from T1 sources |

The framework's sensitivity analysis (documented in the scorecard template) confirms that current weights are robust. No +/-5% weight swap changes the band assignment for any cloud except NetDocuments at the Strong/Weak Go boundary (7.10), which is already noted as a boundary case.

---

## Evaluation Artifacts

All individual scorecards are stored at `memory-bank/thoughts/shared/research/2026-03-10-scorecard-[cloud].md` with corresponding briefs at the same path with `-brief` suffix.

| Cloud | Scorecard | Brief |
|-------|-----------|-------|
| Microsoft 365 | [[2026-03-10-scorecard-m365\|Full]] | [[2026-03-10-scorecard-m365-brief\|Brief]] |
| NetDocuments | [[2026-03-10-scorecard-netdocs\|Full]] | [[2026-03-10-scorecard-netdocs-brief\|Brief]] |
| Box | [[2026-03-10-scorecard-box\|Full]] | [[2026-03-10-scorecard-box-brief\|Brief]] |
| Clio | [[2026-03-10-scorecard-clio\|Full]] | [[2026-03-10-scorecard-clio-brief\|Brief]] |
| iManage | [[2026-03-10-scorecard-imanage\|Full]] | [[2026-03-10-scorecard-imanage-brief\|Brief]] |
| Filevine | [[2026-03-10-scorecard-filevine\|Full]] | [[2026-03-10-scorecard-filevine-brief\|Brief]] |
| DocuSign | [[2026-03-10-scorecard-docusign\|Full]] | [[2026-03-10-scorecard-docusign-brief\|Brief]] |
| 8am/MyCase/LawPay | [[2026-03-10-scorecard-8am-mycase\|Full]] | [[2026-03-10-scorecard-8am-mycase-brief\|Brief]] |
