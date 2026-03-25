# Cloud Evaluation: Filevine

**Date:** 2026-03-10
**Evaluator:** Jordan
**Method:** Manual (deep research)
**Confidence:** Moderate

## Gate Criteria

| Gate | Result | Evidence | Source Tier |
|------|--------|----------|-------------|
| G1: Technical Buyer | CONDITIONAL PASS (scope: firms with 50+ attorneys or IT-managed deployments) | Attorney/admin-led at small firms. IT involvement increases significantly at 50+ attorney firms where dedicated legal ops or IT staff manage the platform. Filevine's own enterprise tier targets these larger deployments. | T2: Filevine enterprise docs, legal ops job postings |
| G2: Data Criticality | PASS | Extremely high criticality. Case files, medical records (PI firms process 20M pages/day), client communications, trust accounting, litigation timelines. DPA explicitly shifts backup responsibility to subscriber. 30-day deletion post-termination. | T1: Filevine DPA, Filevine Legal Encyclopedia, T2: PI firm case studies |

**Gate result:** PROCEED WITH SCOPE RESTRICTION: Evaluate for 50+ attorney firms and IT-managed deployments only.

## Scored Dimensions

| Dimension | Weight | Score (1-10 or IE) | Weighted | Evidence | Source Tier |
|-----------|--------|-------------------|----------|----------|-------------|
| MSP/Channel Interest | 20% | 5 | 1.00 | Filevine has a structured 4-tier partner program (Referral, Reseller, Technology, Strategic). Some legal IT consultants resell. But not in standard MSP toolkits. Legal-vertical MSPs are thin. No r/msp presence. | T1: Filevine partner program page, T2: legal IT consultant listings |
| Marketplace/Ecosystem | 15% | 6 | 0.90 | Filevine Marketplace exists with integrations (LawPay, Smith.ai, Zapier). Technology partner tier allows ISV listings. No backup or data-protection category. No backup products listed. First-mover slot open. | T1: Filevine Marketplace, partner program docs |
| Untapped Market Evidence | 15% | 8 | 1.20 | Strongest untapped signal of any non-DMS legal cloud. Neostella (only known backup vendor) covers documents only, no structured data backup. DataBridge (Filevine's own migration tool) explicitly disclaims backup role. Filevine DPA admits gap: "Subscriber is responsible for maintaining backup copies." Legal Encyclopedia article on data protection reinforces the need. Clear gap + documented demand + vendor's own docs advertise the problem. | T1: Filevine DPA, DataBridge docs, Legal Encyclopedia, T2: Neostella product page |
| Partnership Durability | 15% | 6 | 0.90 | Structured 4-tier program with Neostella as precedent technology partner. Marketplace distribution. Direct to legal ops/IT at enterprise firms. Risk: Filevine raised $400M (2023) and could build native backup. But current DPA language suggests this is not imminent. | T1: Filevine partner program, funding announcements, T2: Neostella partnership |
| API Quality | 15% | 7 | 1.05 | REST API v2 with PAT authentication. Batch document download via S3 presigned URLs. Webhooks for real-time change notification. 320 req/min standard rate limit. Document-specific rate limit is lower by default (requires negotiation for bulk). Covers projects, documents, contacts, tasks, notes, custom fields. | T1: Filevine API v2 docs, T2: developer community posts |
| Organic Demand | 10% | 5 | 0.50 | Filevine's own Legal Encyclopedia publishes articles on data backup importance. DPA language creates awareness. Some forum discussions about export/backup. But no dedicated search demand or analyst coverage for "Filevine backup." Latent, compliance-driven. | T2: Legal Encyclopedia, T3: forum posts |
| Market Size | 10% | 6 | 0.60 | ~6,000 customers. $200M+ ARR. $3B valuation (2023). 50%+ YoY growth trajectory. Expanding rapidly in PI, family law, immigration, and government legal. Market is growing fast but total org count is mid-range. | T1: Filevine funding announcements, T2: Sacra estimates |
| **Total** | **100%** | | **6.15** | | |

## Score Band: Weak Go

## Recommendation

Weak Go -- strongest untapped market signal of any non-DMS legal cloud. Filevine's own DPA, DataBridge disclaimer, and Legal Encyclopedia all advertise the backup gap. The API is feasible with good document coverage via S3 URLs. Growth trajectory (50%+ YoY) means the addressable market is expanding. The conditional G1 pass limits the addressable segment to larger firms with IT involvement, but Filevine's enterprise push is growing that segment. Best pursued as a secondary target after NetDocuments, leveraging shared legal-vertical positioning.

## Key Risks

- Filevine could build native backup post-$400M raise
- G1 conditional -- small-firm segment (majority of customers) has no technical buyer
- Document rate limit requires negotiation for bulk operations
- Neostella could expand from doc-only to full backup

## Sources

- Filevine DPA (Tier 1)
- Filevine API v2 documentation (Tier 1)
- Filevine partner program (Tier 1)
- Filevine funding/valuation announcements (Tier 1)
- DataBridge documentation (Tier 1)
- Filevine Legal Encyclopedia (Tier 1)
- Neostella product page (Tier 2)
- Sacra research estimates (Tier 2)
