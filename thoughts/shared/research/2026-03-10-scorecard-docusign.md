# Cloud Evaluation: DocuSign

**Date:** 2026-03-10
**Evaluator:** Jordan
**Method:** Manual (deep research)
**Confidence:** Moderate

## Gate Criteria

| Gate | Result | Evidence | Source Tier |
|------|--------|----------|-------------|
| G1: Technical Buyer | CONDITIONAL PASS (scope: enterprise deployments with IT/procurement involvement) | Legal/ops buyer at SMB, not IT. MSP involvement near-zero for e-signature. However, enterprise deployments (CLM, IAM expansion) involve IT procurement. DocuSign's IAM push (Agreement Cloud, Intelligent Agreement Management) is shifting buyer profile toward IT. | T1: DocuSign investor relations, T2: Gartner CLM analysis |
| G2: Data Criticality | PASS | Signed contracts are legally irreplaceable originals. Audit trails have evidentiary value. DPA shifts backup responsibility to customer. 24-hour deletion window after account closure. HIPAA (6yr), FINRA, SOX retention requirements apply to signed documents. | T1: DocuSign DPA, HIPAA/FINRA guidance, T2: DocuSign security whitepaper |

**Gate result:** PROCEED WITH SCOPE RESTRICTION: Evaluate for enterprise segment with IT involvement only.

## Scored Dimensions

| Dimension | Weight | Score (1-10 or IE) | Weighted | Evidence | Source Tier |
|-----------|--------|-------------------|----------|----------|-------------|
| MSP/Channel Interest | 20% | 3 | 0.60 | E-signature is not in MSP toolkits. DocuSign is purchased by legal/ops/procurement, not IT departments at most organizations. No r/msp discussions. Channel is SI/consulting (Accenture, Deloitte for CLM), not MSP. | T2: DocuSign partner directory, T3: MSP community absence |
| Marketplace/Ecosystem | 15% | 7 | 1.05 | App Center with 1,000+ apps across multiple categories. Build Partner track available. No backup product listed -- first-mover slot is open in a large marketplace. High visibility distribution channel. | T1: DocuSign App Center, Build Partner program docs |
| Untapped Market Evidence | 15% | 6 | 0.90 | HYCU, ProBackup, Keepit, CloudAlly all offer some DocuSign coverage but no dominant player. No App Center listing for any backup vendor. API restore limitation (no upload of signed docs) means market is "compliance archive" not "full backup/restore." Gap is real but product positioning is constrained. | T1: App Center inspection, T2: HYCU/ProBackup/Keepit product pages |
| Partnership Durability | 15% | 7 | 1.05 | Formal Build/Sell/Service partner program launched April 2025. App Center distribution with install tracking. IAM expansion creating new integration surface. Multiple paths: App Center, direct enterprise, SI channel, compliance/GRC vendors. | T1: DocuSign partner program (April 2025), App Center |
| API Quality | 15% | 7 | 1.05 | REST API with JWT OAuth. Full read coverage for envelopes, documents, audit trails, templates. Webhooks via DocuSign Connect for real-time notifications. 3,000 calls/hr rate limit. Critical limitation: no restore via API (cannot re-upload signed documents with original signatures). Product must be archive/export, not full restore. | T1: DocuSign developer docs, Connect documentation |
| Organic Demand | 10% | 5 | 0.50 | Compliance-driven demand. HIPAA, FINRA, SOX retention requirements create structural need. Some community discussion about export/archival. But "DocuSign backup" is not a searched term. Demand is regulatory, not user-initiated. | T2: compliance forums, T3: community posts |
| Market Size | 10% | 9 | 0.90 | 1.7M organizations. 67% of global e-signature market. 95% of Fortune 500. Massive addressable base even with enterprise-only scope restriction. $2.8B annual revenue. | T1: DocuSign investor relations, IDC e-signature market data |
| **Total** | **100%** | | **6.05** | | |

## Score Band: Weak Go

## Recommendation

Weak Go -- massive market with critical data and a formal partnership program, but the non-IT buyer profile and API restore limitation constrain the opportunity. Product must be positioned as "compliance archive" or "agreement vault" rather than traditional backup/restore. First-mover advantage in the App Center is real and valuable. The IAM expansion (Intelligent Agreement Management) is gradually shifting the buyer toward IT, which could strengthen G1 over time. Best pursued opportunistically alongside M365 (many DocuSign customers are M365 shops).

## Key Risks

- Non-IT buyer complicates channel strategy (MSP score of 3)
- API cannot restore signed documents -- product is archive-only
- DocuSign could build native archive/vault (IAM roadmap includes lifecycle management)
- 24-hour deletion window is extremely aggressive

## Sources

- DocuSign developer docs (Tier 1)
- DocuSign investor relations (Tier 1)
- DocuSign partner program (April 2025) (Tier 1)
- DocuSign DPA (Tier 1)
- IDC e-signature market data (Tier 1)
- HYCU/ProBackup/Keepit product pages (Tier 2)
- Gartner CLM analysis (Tier 2)
