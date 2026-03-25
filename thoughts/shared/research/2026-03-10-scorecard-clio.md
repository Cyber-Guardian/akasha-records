# Cloud Evaluation: Clio

**Date:** 2026-03-10
**Evaluator:** Jordan
**Method:** Manual
**Confidence:** Moderate

## Gate Criteria

| Gate | Result | Evidence | Source Tier |
|------|--------|----------|-------------|
| G1: Technical Buyer | FAIL | 80% of users are 1-10 employee firms without IT dept. Only 9 channel partners. Solo lawyers self-administer. MSPs structurally marginal. | T1: Software Advice, Clio Legal Trends Report |
| G2: Data Criticality | PASS | Legal matters, billing, trust accounting, documents. Bar rules require 5-7 year retention. Fiduciary obligations. | T1: ABA Model Rules, Clio docs |

**Gate result:** STOP (G1 fails)

## Scored Dimensions

Scores below are for calibration only since the evaluation is gated at G1.

| Dimension | Weight | Score (1-10 or IE) | Weighted | Evidence | Source Tier |
|-----------|--------|-------------------|----------|----------|-------------|
| MSP/Channel Interest | 20% | 3 | 0.60 | Only 9 channel partners vs 170 ISV. No r/msp presence. Legal-vertical MSPs exist but thin. | T1: Clio partner program, T2: Partnerbase |
| Marketplace/Ecosystem | 15% | 6 | 0.90 | 302 apps in Clio App Directory. FasterLaw backup products listed. Security category exists. But thin for backup. | T1: Clio App Directory |
| Untapped Market Evidence | 15% | 4 | 0.60 | FasterLaw covers docs only. Native Data Escrow has weekly overwrite flaw. Gap exists for structured data but demand is diffuse. | T1: Clio help center, T2: FasterLaw |
| Partnership Durability | 15% | 4 | 0.60 | Direct, App Directory, channel partners, bar association endorsements. But Clio controls chokepoint ($5B company could close gap easily). | T1: Clio partnerships page |
| API Quality | 15% | 7 | 1.05 | REST API v4, OAuth 2.0, full CRUD, webhooks, all data types. 50 req/min rate limit during peak. Well-documented. | T1: Clio developer docs |
| Organic Demand | 10% | 3 | 0.30 | Help articles on backup exist but no r/msp threads, no analyst mentions. Demand diffuse, not urgent. | T2-T3: help center density, FasterLaw SEO |
| Market Size | 10% | 5 | 0.50 | 150K+ legal professionals, ~50-75K firms. TAM $6-18M ARR. Small market. Solo firms spend 1% on software. | T1: Sacra, Clio Legal Trends |
| **Total** | **100%** | | **4.55** | | |

## Score Band: Borderline

## Recommendation

No-Go -- fails G1 gate. Even ignoring the gate, the 4.55 Borderline score reflects structural ICP mismatch. No technical buyer in the dominant customer segment means no path to sale through MSP/channel.

## Key Risks

- Clio could improve native backup easily
- Solo firms have near-zero willingness to pay for add-ons

## Sources

- Clio developer docs (Tier 1)
- Sacra research (Tier 1)
- Software Advice (Tier 2)
- Clio partner program (Tier 1)
- FasterLaw guides (Tier 2)
