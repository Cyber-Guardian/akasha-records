---
title: "Platform V2: MSP-First Breadth Strategy"
created: 2026-03-12
type: brief
status: shaped
tags: [strategy, product, msp, integrations, platform-v2, differentiation]
---

# Platform V2: MSP-First Breadth Strategy

## The Idea

Pivot FileScience from legal-vertical backup to **general MSP-focused backup**, competing on integration breadth + AI intelligence. Build a modular connector architecture that makes adding new cloud integrations fast and cheap. Use CGCG (our parent MSP) as the alpha customer to drive priorities. Differentiate on **connector depth, AI-native capabilities, and RMM/PSA integration** — not price or raw connector count.

---

## Why Now

**The gap is real but closing.** Datto/Kaseya — the dominant MSP backup stack — only cover M365 and Google Workspace. They don't back up vertical SaaS apps. MSPs managing diverse client stacks are forced to either ignore those apps or cobble together multiple vendors. 46% of MSPs are actively consolidating vendors (Datto's own 2025 State of the MSP report).

**Competitors have noticed, but they're weak:**
- **Asigra SaaSAssure** (launched June 2024) — MSP-first, ~10 connectors (M365, Salesforce, HubSpot, QuickBooks, Box, Jira, Confluence). But: QuickBooks has NO restore. Teams private channels missing. Zero AI features. No RMM/PSA integration. Added zero new connectors in 12 months. 85 employees, bootstrapped, slow.
- **Keepit** — expanding via AI-assisted connector building, $50M funding, targeting "hundreds" of connectors by 2028. No legal vertical.
- **HYCU R-Cloud** — 200+ apps, but enterprise-priced and not MSP-first.

---

## CGCG Client Cloud Stack (Real Data)

CGCG's clients run these SaaS apps — this IS our integration priority list:

| Cloud | SaaSAssure covers? | FileScience covers? | Category |
|-------|--------------------|---------------------|----------|
| Google Workspace | No | No | Productivity |
| Salesforce | Yes | No | CRM |
| Box | Yes | Yes (Silver) | Storage |
| Slack | No | No | Communication |
| Dropbox | No | No | Storage |
| Monday.com | No | No | Project management |
| Zoom | No | No | Communication |
| NetSuite | No | No | ERP/Finance |
| Workday | No | No | HR |
| AirTable | No | No | Database/PM |
| AirBase | No | No | Finance |
| Sage Cloud | No | No | Accounting |
| UNIT4 | No | No | ERP |
| Financial Edge | No | No | Nonprofit finance |
| Raisers Edge | No | No | Nonprofit CRM |
| Gatekeeper | No | No | Contract management |
| eClinicalWorks | No | No | Healthcare EHR |
| Meraki config | No | No | Network config |
| Unifi config | No | No | Network config |
| Verkada config | No | No | Security/camera config |

**Key finding:** SaaSAssure covers only **2 of ~20** CGCG client clouds. Nobody covers most of this list. The market is even more open than expected.

**Priority candidates for first Bronze integrations** (highest cross-MSP demand + API feasibility):
1. **Google Workspace** — massive MSP staple, SaaSAssure doesn't have it
2. **Salesforce** — SaaSAssure has it but with broken restores
3. **Slack** — compliance-critical for regulated industries
4. **Dropbox Business** — common MSP client app
5. **Monday.com** — growing fast, no backup exists
6. **NetSuite** — finance data = high criticality, high willingness to pay

*Note: Meraki, Unifi, Verkada are network/device configs, not traditional SaaS — may need a different backup model (config export + versioning rather than API traversal).*

---

## The Strategy

### 1. Breadth-first, depth-second

Build many decent integrations. See which perform. Deepen winners.

| Tier | What it means | Recovery experience |
|------|---------------|---------------------|
| **Gold** | Full backup + point-in-time recovery + delta sync + in-place restore | One-click restore to original location |
| **Silver** | Full backup + granular export recovery + basic change detection | Browse and download specific items |
| **Bronze** | Full backup + bulk export recovery | Download everything as a zip/archive |

Every tier must include recovery. MSPs promise their clients they can restore.

### 2. Integration factory

Extract the connector abstraction from existing M365 integration incrementally. Each new cloud stress-tests and improves the factory. Don't design in a vacuum.

Current architecture has 7 integration points that need modularizing: OAuth dispatch, session manager, cloud model registry, service handlers, throttle policies, dispatcher mapping, VQP policy registry. The domain model (Entity → Resource → Artifact) is already cloud-agnostic.

### 3. CGCG as alpha customer

CGCG's client stack (above) drives integration priorities. CGCG dogfoods. This is the fastest path to building what MSPs actually need.

### 4. ICP: Non-Kaseya general MSPs

Kaseya 365 bundles backup for free ($2.79/user includes SaaS backup + 4 other tools). Kaseya shops are NOT addressable. Target: ConnectWise shops, N-able shops, independent MSPs who need standalone backup.

---

## How We Stand Out

### Differentiation stack (ordered by defensibility)

**1. AI intelligence layer — no MSP backup vendor has this**

Enterprise vendors (Druva, Cohesity, Rubrik) have AI, but it's enterprise-priced and not multi-tenant MSP. SaaSAssure has zero AI. This is the moat.

| Feature | MSP value | Build cost | Defensibility |
|---------|----------|------------|---------------|
| **Compliance evidence packs** — auto-generate audit evidence for cyber insurance | High — replaces manual labor, new billable service | Low — backup logs already have the data | Medium |
| **Cross-tenant anomaly detection** — aggregate signals across MSP customers | High — catches ransomware, unlocks security budget | Medium — ML + data pipeline | Very high (data network effect) |
| **Cross-SaaS employee forensics** — "everything about person X across all apps" | High — 86% of orgs lack offboarding data handling | Medium — metadata indexing + entity resolution | High (historical depth = switching cost) |
| **MCP-native multi-tenant interface** — natural language queries across fleet | Medium — sticky, low-cost | Low — MCP server | Medium (first mover) |

**Key technical insight:** Use metadata-only indexing (not full content RAG). 100-1000x cheaper, more privacy-safe, answers most high-value queries. Amazon S3 Vectors (GA July 2025) cuts embedding storage by 90%.

**The security budget unlock:** Cybersecurity growing 18%/yr in MSP services. Cyber insurers mandating backup controls. Anomaly detection + compliance evidence shifts product from IT-ops budget (flat) to security budget (growing). Different buyer, bigger wallet.

**2. Connector DEPTH — beat SaaSAssure on restore fidelity, not count**

| SaaSAssure gap | FileScience opportunity |
|----------------|----------------------|
| QuickBooks: **no restore at all** | Ship with restore = instant win |
| Teams: private channels, attachments, apps missing | Deep Teams fidelity |
| Salesforce: custom field restores fail | Reliable Salesforce restore |
| HubSpot: ticket attachments not restored | Complete HubSpot |
| SharePoint: classic not supported | Full SharePoint |
| Exchange: 150K+ item restores fail | Enterprise-grade Exchange |

**3. RMM/PSA integration — SaaSAssure's biggest MSP gap**

SaaSAssure has zero ConnectWise, Autotask, NinjaRMM, or Halo integration. MSPs live in their PSA. Backup alerts that don't flow into ticketing = operational friction. Build this.

**4. Legal vertical — beachhead nobody can touch**

Nobody covers Clio. For MSPs with law firm clients, FileScience is the only full-stack option (Clio + M365 + Box).

**5. Price — match market, don't lead with it**

| Metric | Number |
|--------|--------|
| Market vendor-to-MSP range | $1.30–$3.40/user/month |
| FileScience infra cost | $0.25–$0.75/user/month |
| Target wholesale price | ~$2/user/month |
| Gross margin at $2 | **75%+** |
| SaaSAssure floor | $1.30/user (M365) |

Per-user with unlimited/pooled storage is the expected model. MSPs hate per-GB surprises.

### What NOT to compete on

- **Raw connector count** — HYCU has 200+, Keepit targeting hundreds with $50M funding. Can't win this race.
- **Price** — Kaseya bundles backup for free. SaaSAssure undercuts at $1.30. Price wars destroy margins.
- **Enterprise features** — Druva/Cohesity/Rubrik own this. Stay MSP-focused.

---

## Sequence

1. **Now:** CGCG discovery ~~conversation~~ ✅ Got the cloud list. Next: get backup pain points, switching criteria.
2. **Week 2-4:** Run integration scorecards on CGCG's top clouds (Google Workspace, Salesforce, Slack, Dropbox, Monday.com, NetSuite)
3. **Month 2-3:** Extract connector abstraction from existing M365 integration
4. **Month 3-5:** Build 2-3 new Bronze integrations on the framework (Google Workspace + Salesforce first)
5. **Month 4-6:** Ship compliance evidence pack (AI feature #1 — lowest build cost, highest MSP value)
6. **Month 5-7:** CGCG dogfoods the multi-cloud product. Iterate.
7. **Month 6+:** Cross-tenant anomaly detection (AI feature #2)

---

## Open Questions

- ~~What SaaS apps does CGCG's client base run?~~ ✅ Answered — see table above
- What's CGCG currently using for backup and what's frustrating?
- Would CGCG switch? For what?
- Which CGCG clients have the highest data criticality / compliance needs?
- Network config backup (Meraki/Unifi/Verkada) — different model needed?
- How much MSP margin do we need to offer for active channel selling?
- Should legal remain a primary vertical or become one of many?

---

## Related Artifacts
- [[2026-03-12-brain-dump-validated-ideas|Brain Dump — Validated Ideas]]
- [[2026-03-11-shapes-brainstorm-best-ideas|Shapes Brainstorm Synthesis]]
- [[2026-03-11-bizdev-operating-system|BizDev Operating System Brief]]
- [[scorecard-template|Integration Scorecard Framework]]

## Scorecard Results (CGCG Stack)

Evaluated 2026-03-12. Full scorecards in session output files.

| Rank | Cloud | Score | Band | Build Phase |
|------|-------|-------|------|-------------|
| 1 | **Google Workspace** | 8.10 | Strong Go | Month 2-3 (Bronze → Silver) |
| 2 | **Salesforce** | 7.05 | Strong Go | Month 3-5 (Bronze) |
| 3 | **Dropbox Business** | 5.90 | Weak Go | Month 5-7 (Bronze) |
| 4 | **Monday.com** | 5.80 | Weak Go | Defer (API can't export automations) |
| 5 | **Slack** | 5.35 | Borderline | Defer (API gated to Enterprise Grid) |
| 6 | **NetSuite** | 4.65 | Borderline | Defer (channel mismatch) |

**Existing integrations:** M365 (8.85 Strong Go, Gold), Box (6.63 Weak Go, Silver), Clio (4.55 Borderline, legal wedge)

**Build sequence:** M365 depth → Google Workspace → Salesforce → Dropbox → (Monday.com when API matures)

## Next Step
- `/create_plan` the connector factory architecture + Google Workspace Bronze integration
