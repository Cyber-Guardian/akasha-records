# Cloud Evaluation: 8am / MyCase / LawPay

**Date:** 2026-03-10
**Evaluator:** Jordan
**Method:** Manual (deep research)
**Confidence:** High

## Gate Criteria

| Gate | Result | Evidence | Source Tier |
|------|--------|----------|-------------|
| G1: Technical Buyer | FAIL | Same ICP as Clio -- solo practitioners and small firms. 64% of MyCase users are at firms with 2-10 employees. No IT department. Attorneys self-administer. 8am (parent company AffiniPay) acquired MyCase and LawPay to serve this segment. No MSP ecosystem. | T1: Software Advice firm size data, AffiniPay press, T2: MyCase customer profiles |
| G2: Data Criticality | PASS | Case data, client documents, billing records, trust accounting (LawPay handles IOLTA compliance). Documents excluded from native backup. Bar rules require retention. Compliance stakes from trust accounting. | T1: MyCase docs, state bar IOLTA rules |

**Gate result:** STOP (G1 fails)

## Scored Dimensions

Scores below are for calibration only since the evaluation is gated at G1.

| Dimension | Weight | Score (1-10 or IE) | Weighted | Evidence | Source Tier |
|-----------|--------|-------------------|----------|----------|-------------|
| MSP/Channel Interest | 20% | 2 | 0.40 | No MSP presence. LawPay is sold direct to attorneys. MyCase has no channel partner program. 8am/AffiniPay focuses on direct sales and bar association endorsements. | T2: AffiniPay partner page, T3: MSP community absence |
| Marketplace/Ecosystem | 15% | 4 | 0.60 | MyCase integrations page lists ~30 partners (LawPay, QuickBooks, Dropbox). No formal marketplace. No ISV program. No backup category. Limited ecosystem. | T2: MyCase integrations page |
| Untapped Market Evidence | 15% | 3 | 0.45 | No known backup product for MyCase. But also no evidence of demand -- no community requests, no analyst mentions, no compliance-driven discussion. Empty market with no demand is a dead market, not an opportunity. | T3: absence of evidence across all sources |
| Partnership Durability | 15% | 5 | 0.75 | AffiniPay has bar association relationships (150+ endorsements for LawPay). Could theoretically distribute through bar associations. But no technology partner program, no marketplace, no MSP channel. Single path (bar associations) with uncertain backup relevance. | T1: AffiniPay/LawPay bar endorsements |
| API Quality | 15% | 4 | 0.60 | MyCase API launched 2023 (relatively young). Basic REST with API key auth. Advanced tier features gated behind higher subscription plans. Document access likely limited -- no bulk download, no S3 URLs. Sparse documentation. | T2: MyCase developer docs (limited), T3: developer community posts |
| Organic Demand | 10% | 2 | 0.20 | No search volume for "MyCase backup." No community discussion. No analyst coverage. No G2/Capterra category. Complete absence of demand signals. | T3: absence across all sources |
| Market Size | 10% | 3 | 0.30 | ~15,000 MyCase firms (estimated). Roughly 1/10th of Clio's base. AffiniPay total payments volume is large ($23B) but the practice management user base is small. Average firm size 2-10 employees means low per-account value. | T2: AffiniPay press, T3: market estimates |
| **Total** | **100%** | | **3.30** | | |

## Score Band: No-Go

## Recommendation

No-Go -- confirmed Clio repeat. Same ICP (solo/small law firms without IT), smaller market (1/10th of Clio), weaker API, zero demand signals. The G1 failure is structural and identical to the Clio finding: practice management tools for small law firms do not have a technical buyer in the purchase path. The AffiniPay consolidation (8am + MyCase + LawPay) does not change the underlying ICP. Do not invest.

## Key Risks

- Structural G1 failure identical to Clio
- API is immature (2023 launch) with gated features
- Market too small to justify integration cost

## Sources

- AffiniPay press releases (Tier 1)
- State bar IOLTA rules (Tier 1)
- Software Advice firm size data (Tier 1)
- MyCase integrations page (Tier 2)
- MyCase developer docs (Tier 2)
