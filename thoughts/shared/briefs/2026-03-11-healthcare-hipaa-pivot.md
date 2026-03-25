# Idea Brief: Healthcare HIPAA Pivot

**Date:** 2026-03-11
**Status:** Shaped → Planning

## Problem
FileScience bet on legal (Clio) as its launch vertical. After months live, the result is 1 customer at $18/mo. The integration scorecard confirmed Clio is a structural G1 failure — 80% of users are at 1-10 employee firms with no IT buyer. NetDocuments (the #1 greenfield legal target) rejected partnership. The product works; the market doesn't.

Meanwhile, the 2026 HIPAA Security Rule (finalization expected May 2026, 180-240 day compliance window) creates a once-in-a-decade regulatory forcing function requiring immutable backups, 72-hour recovery capability, mandatory AES-256 encryption, and universal MFA — for ALL covered entities.

## Constraints
- Team: Jordan (v2 monorepo), Niklas + Harsh (v1 legacy)
- Parent company: Cyber Guardian Consulting Group (CGCG) — an MSP/MSSP based in Kingston, NY. Funded. CGCG does cybersecurity for SMBs.
- Existing cloud connectors: Clio (shipped, low traction), M365, Box
- M365 backup market has 20-30 incumbents (Veeam, Druva, AvePoint, Datto) but they're enterprise-ugly, IT-admin-focused, and none have HIPAA-specific compliance features
- HIPAA wave is time-sensitive — marketing window peaks Q3-Q4 2026
- Clio can run but should not receive further investment

## Options Considered

### Stay on Legal, Try Different Clouds
Pivot to iManage or other legal DMS platforms.
- Gains: Stays in legal vertical where some domain knowledge exists
- Costs: iManage has HYCU as incumbent. NetDocuments said no. Remaining legal clouds are Weak Go or No-Go.
- Complexity: Medium

### Healthcare HIPAA Pivot (M365 + MSP Channel)
Reframe from "legal backup" to "HIPAA-compliant backup for healthcare." M365 as anchor cloud. CGCG as dogfood customer and channel. Compliance dashboard as product differentiator. Ride the HIPAA 2026 wave with targeted ads.
- Gains: Regulatory forcing function (must-buy, not nice-to-have). MSP channel validated by parent company. M365 is universal in healthcare. Mid-market healthcare orgs (50-500 employees) have IT buyers. Incumbents don't have HIPAA-specific features.
- Costs: Competing in M365 backup (crowded). Healthcare compliance domain knowledge needed. New marketing assets required.
- Complexity: Medium — existing infrastructure applies, pivot is market positioning + compliance features + UX.

### Broad Mid-Market Backup Platform
Go horizontal — M365 + Box for any industry via MSP channel.
- Gains: Larger addressable market. No vertical expertise needed.
- Costs: No differentiation against incumbents. Commodity race.
- Complexity: Low

## Chosen Approach
**Healthcare HIPAA Pivot** — because the HIPAA 2026 rule change creates time-bound demand, CGCG validates the MSP channel, and competing on compliance outcomes (not backup features) avoids the head-on fight with Veeam/Druva.

## How to Win on M365 (Five Angles)

### 1. Sell Compliance, Not Backup (Positioning)
Dashboard shows HIPAA compliance status, not backup job status. Immutable backup running? 72-hour recovery SLA met? Encryption verified? One-click audit report PDF. The compliance officer doesn't care about backup jobs — they care about audit proof.

### 2. UX as Moat (Experience)
"Linear of backup." 5-minute setup (vs Veeam's complex config, AvePoint's 45-day seed). Visual timeline. Plain English restore ("restore Sarah's inbox from last Tuesday"). Zero-training onboarding.

### 3. Version Intelligence (Novel Differentiator)
Temporal semantic search ("find the email where patient records were sent to the wrong address"). NL change summaries. Anomaly detection (bulk deletion → ransomware alert). Compliance autopilot (AI verifies HIPAA requirements continuously). No incumbent has this.

### 4. Price the Mid-Market Gap (Cost)
$4/user/month, HIPAA-ready out of the box. Incumbents are enterprise-priced. AvePoint is "challenging for SMBs" per their own reviews.

### 5. MSP-Native by DNA (Channel)
CGCG builds what MSPs actually need: multi-tenant, per-client compliance dashboards, bulk onboarding, margin-friendly pricing. Built by an MSP, for MSPs.

## Product Stack
```
HIPAA Compliance Dashboard (audit reports, SLA proof, gap alerts)
Version Intelligence Layer (semantic search, NL summaries, anomaly detection)
Beautiful UX / Visual Timeline (5-min setup, plain English restore)
MSP Management Plane (multi-tenant, bulk ops, margin reporting)
Backup Engine (M365 + Box, immutable, encrypted, fast)
Custom Storage Engine (Phase 2: CDC + global dedup + delta compression)
```

## GTM Strategy
- **Ads:** Target HIPAA 2026 keywords ("HIPAA backup requirements 2026", "immutable backup healthcare", "72-hour recovery HIPAA")
- **Content:** Blog posts, compliance checklists, "is your practice ready?" assessments
- **MSP channel:** CGCG as first customer → healthcare MSP outreach
- **Target:** Healthcare orgs 50-500 employees (group practices, multi-location clinics, small hospital systems)

## Target Healthcare SaaS Stack
- **M365** (email, SharePoint, OneDrive — universal, 80% of PHI outside EHR)
- **Box** (document storage, HIPAA-compliant tier)
- **Phase 2:** athenahealth, Kareo/Tebra, DrChrono (EHR/practice management connectors)

## Industries Evaluated
| Industry | Rank | Key Factor |
|----------|------|-----------|
| Healthcare | 1 | HIPAA 2026 forcing function, MSP channel, PHI irreplaceable |
| Financial Services / RIA | 2 | FINRA/SEC mandates, clear buyer, zero backup for RIA apps |
| Accounting / CPA | 3 | FTC Safeguards, QBO anchor, but weaker compliance pressure |
| Construction / AEC | 4 | Underserved but wrong buyer path for CGCG |

## Key Context Discovered During Shaping
- Clio: 1 customer at $18/mo after months live. G1 structural failure confirmed by scorecard.
- NetDocuments rejected partnership — greenfield legal opportunity closed.
- CGCG (parent) is an MSP/MSSP — validates the MSP channel thesis.
- Veeam: complex setup, vague logging, poor integration, pricing backlash.
- AvePoint: can't answer basic questions, no on-demand backup, 45-day seed time, expensive for SMBs.
- HIPAA 2026: finalization ~May 2026, 180-240 day window, biggest changes in a decade.
- M365 Backup Storage API (GA Aug 2024) creates new differentiation surface.

## Related Artifacts
- [[2026-03-10-cloud-ranking-summary|Cloud Integration Ranking]] — scorecard evaluations
- [[2026-01-28-filescience-product-context-expanded-notes|Product Context Notes]] — current positioning
- [[2026-03-05-deep-emergent-markets-for-filescience|Emergent Markets Research]] — version intelligence vision
- [[2026-03-11-agentic-storage-rd-lab|Agentic Storage R&D Lab]] — Phase 2 storage engine (parked)
- [[2026-03-09-platform-v2-roadmap-strategy|Platform V2 Roadmap]] — spiral passes model
- [[2026-02-28-complexity-as-moat|Complexity as Moat]] — strategic rationale

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-11-healthcare-hipaa-pivot.md`
