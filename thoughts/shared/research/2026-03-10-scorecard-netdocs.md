# Cloud Evaluation: NetDocuments

**Date:** 2026-03-10
**Evaluator:** Jordan
**Method:** Manual
**Confidence:** High

## Gate Criteria

| Gate | Result | Evidence | Source Tier |
|------|--------|----------|-------------|
| G1: Technical Buyer | PASS | CIO/IT Director leads purchases (Wilson Sonsini CIO, Sidley Austin CIO documented). Implementation partners (Optiable, Verlata) act as MSP layer for smaller firms. | T1: Legal IT Insider, NetDocuments press |
| G2: Data Criticality | PASS | Legal DMS storing briefs, contracts, matter documents, email filing, version histories. Permanently deleted items irrecoverable. ABA ethics obligations. | T1: NetDocuments docs, ABA standards |

**Gate result:** PROCEED

## Scored Dimensions

| Dimension | Weight | Score (1-10 or IE) | Weighted | Evidence | Source Tier |
|-----------|--------|-------------------|----------|----------|-------------|
| MSP/Channel Interest | 20% | 7 | 1.40 | Legal MSPs universally support NetDocuments. CRN 5-star partner program (2021-2024). esudo.com: "If an MSP says they don't support NetDocuments, run." But no MSP-specific backup product yet. | T1: CRN recognition, T2: MSP blogs |
| Marketplace/Ecosystem | 15% | 6 | 0.90 | App Directory exists on Salesforce-hosted portal. 30+ tech partners. But NO backup category and zero backup apps listed. ndConnect program (2025) is AI-focused only. | T1: App Directory inspection |
| Untapped Market Evidence | 15% | 9 | 1.35 | Zero competitor backup products for NetDocuments. HYCU built for iManage but not NetDocs. ndMirror (native) is sync-not-backup with key gaps (deletes propagate). Clear gap + compliance-driven demand. | T1: App Directory confirms zero, HYCU/iManage analogy |
| Partnership Durability | 15% | 8 | 1.20 | Marketplace listing, legal MSP channel, implementation partner channel (Optiable Platinum, Verlata Diamond), direct to legal IT, government channel via Carahsoft. 4-5 independent paths. | T1: partner program, CRN |
| API Quality | 15% | 7 | 1.05 | REST API v2, OAuth 2.0, backup-relevant scopes (read, full). Postman-governed (3x testing speed). ndMirror proves bulk extraction feasible. Rate limits not publicly documented. | T1: Postman case study, API docs, T2: ndMirror docs |
| Organic Demand | 10% | 5 | 0.50 | Dedicated "Backup" topic in support portal. Affinity Consulting and Optiable publish export guides. Deletion permanence is documented gap. But no r/msp threads, no G2 mentions of backup. Latent, not loud. | T2: support portal, partner blogs, T3: implied from deletion policy |
| Market Size | 10% | 7 | 0.70 | 7,000+ organizations, 300,000+ users (post-Worldox), expanding with eDOCS acquisition. 75% US. $1.4B valuation (2021). Inc. 5000 in 2024 and 2025. | T1: NetDocuments press releases |
| **Total** | **100%** | | **7.10** | | |

## Score Band: Strong Go

## Recommendation

Strong Go -- both gates pass, strong untapped market with zero existing backup products, proven MSP channel, multiple partnership paths. The HYCU/iManage playbook applies directly. API is feasible.

## Key Risks

- API rate limits unknown publicly
- ndMirror is vendor's own product (positioning needed)
- Demand is latent not loud
- ndConnect is AI-focused

## Sources

- Legal IT Insider (Tier 1)
- CRN Partner Guide (Tier 1)
- Postman case study (Tier 1)
- NetDocuments press releases (Tier 1)
- Optiable/Affinity guides (Tier 2)
