# FileScience Product Context — Expanded Notes (2026-01-28)

These are **archival expanded notes** capturing fuller context, phrasing, and considerations from public site content (plus some explicitly labeled internal positioning).

Only read this file if you need more context than what’s in the durable summary:

- `memory-bank/durable/00-core/product_context.md`

## Reality check (internal; keep this accurate)
- **Supported clouds today**: **Clio**, **Microsoft 365**, **Box**
- Other clouds shown on the marketing site are **prospecting / interest tests** (we list them to find potential customers; we’ll build/support them when there’s demand).

## Product positioning (internal)
FileScience is a cloud-to-cloud backup and recovery platform that helps organizations stay operational when the unexpected happens. We connect to cloud applications and run automated daily backups. If something goes wrong, our intuitive recovery viewer lets anyone quickly find and restore specific items, users or an entire environment, without IT support.

## Canonical sources (public)
- Landing: `https://filescience.io/`
- Security: `https://filescience.io/security`
- Pricing: `https://filescience.io/pricing`
- About: `https://filescience.io/about`
- Get started: `https://filescience.io/get-started/`
- Contact: `https://filescience.io/contact/`
- Clouds index: `https://filescience.io/clouds/`
- Industries index: `https://filescience.io/industries/`
- Blog index: `https://filescience.io/blog/`
- Privacy policy: `https://filescience.io/privacy/`
- Terms: `https://filescience.io/terms/`

## High-level positioning (from landing)
- Headline positioning: “Secure your cloud” / “Effortless, enterprise-grade backup and recovery.”
- Value prop: “Keep your business flowing” by backing up data from “over 20 essential cloud apps.”
- Emphasis on **operational resilience** and **business continuity**, not just storage.

## Cloud integrations (supported vs prospecting)

### Supported today (internal)
- Clio: `https://filescience.io/clouds/clio/`
- Microsoft 365: `https://filescience.io/clouds/microsoft-365/`
- Box: `https://filescience.io/clouds/box/`

Common themes across these pages:
- “The cloud sync trap”: provider redundancy protects their servers, not user-level mistakes; FileScience is framed as a “rewind button.”
- Backup + restore vocabulary: “recurring daily backups”, “immutable snapshots”, “compare file versions”, “granular restoration”, “recovery viewer”.
- “Zero-touch deployment” / “automated handshake” setup language.

### Prospecting clouds shown on the marketing site (NOT supported today)
Listed on `https://filescience.io/clouds/` (shown to find potential suitors; we’d build these when there’s demand):
- NetDocuments, Smartsheet, Zoom, Zoho Docs, Slack, QuickBooks, Paychex, Dropbox, iManage, MyCase
- The clouds index also includes a “Don’t see your cloud listed?” lead-gen form.

## Product narrative / problems called out
Themes that repeatedly show up across pages/FAQ:
- **Mistakes propagate in SaaS**: “one wrong click deletes it everywhere” (error propagation).
- **Independent access**: backup is valuable precisely because it’s not the SaaS provider.
- **Platform limitations**: native SaaS export/restore flows can be slow, limited, or operationally painful.
- **Ransomware & account compromise**: independent, isolated backups reduce blast radius.

## Target users / buyer framing
Explicitly mentioned / strongly implied:
- **MSPs / IT teams**: multi-tenant dashboard, centralized billing/reporting, “grow services” by adding/removing orgs.
- **Regulated industries**: compliance language + vertical examples (site navigation implies legal/healthcare/finance/etc.).
- “Enterprise-grade” signals a bias toward organizations with compliance + uptime sensitivity.

## Key marketed capabilities (expanded notes)
- **Backup**
  - One-click connection to cloud apps/services.
  - Automated backups: daily delta backups by default; FAQ suggests optional increased frequency (as often as ~10 minutes “upon request”).
- **Recovery**
  - Interactive recovery viewer: choose what to restore (entire cloud, specific users, individual objects/versions).
  - “Reliable egress / 24/7 access” framing: quick retrieval when needed.
- **Operations / MSP features**
  - Multi-tenant dashboard and workflows.
  - Centralized billing and reporting.
  - 24/7 expert support.
- **Analytics**
  - Usage analytics: storage patterns, backup performance, “predict and control expenses.”

## Industries framing (from industries pages)
Industries index: `https://filescience.io/industries/`
- Legal services: `https://filescience.io/industries/legal-services`
- Healthcare & medical: `https://filescience.io/industries/healthcare-medical`
- Accounting & finance: `https://filescience.io/industries/accounting-finance`
- IT & MSP: `https://filescience.io/industries/it-msp`
- Architecture & engineering: `https://filescience.io/industries/architecture-engineering`
- Professional services: `https://filescience.io/industries/professional-services`

Recurring narrative pattern across industry pages:
- Risks: compliance gaps, ransomware risk, delete accidents, platform limitations / missing metadata relationships
- Remedy: FileScience as a secondary immutable layer + searchable/restorable archive + “zero-impact continuity” (runs silently in the background)

Notable details from legal vertical page:
- Positions compliance duties and “reasonable efforts” framing (ABA model rule 1.1 comment 8 is cited on page).
- Uses legal-specific concepts like “matter-centric recovery”, “evidentiary integrity”, and “deep app integration” (preserving relationships between matters/clients/billable events).

## Security posture (expanded notes)
From `https://filescience.io/security`:
- **Immutable, air-gapped backups** (ransomware resilience).
- **End-to-end encryption**: “encrypted at all times—in transit and at rest.”
- **Granular access control** / least privilege via user roles.
- “Product & org security embedded in development lifecycle company-wide.”
- “Secure data centers” with 24/7 monitoring.

Compliance/certifications listed on the page:
- **SOC 2 Compliant**
- **ISO 27001 Certified**
- **HIPAA Compliant**
- **PCI DSS Compliant**
- **GDPR Ready**
- **CCPA Ready**

Note for internal use: treat this as **claims/positioning** unless corroborated by internal compliance artifacts.

## Principles & promises (useful as product constraints)
“You Own Your Data. Period.”
- Explicit stance that FileScience is a **guardian** not an owner.
- “Freedom to leave”: export anytime; “permanently and verifiably erased” when you go.
- “API-level data access” language suggests the product values full-fidelity exports (files + metadata), but the site copy is still high-level.

## Pricing notes (from pricing page)
- Pricing is **customized**, positioned as straightforward/no surprises.
- Scales with business: users + data actively protected.
- Enterprise packaging highlights: dedicated account manager/onboarding, advanced security/compliance, custom pricing, unlimited egress.

## Terms & privacy: public statements vs internal reality
Why this matters: various public pages imply broad integration coverage, but for product truth inside this repo we need precision.

- Terms: `https://filescience.io/terms/`
  - “Cloud-to-cloud only” and explicitly: FileScience does **not** guarantee support for every cloud platform; compatibility is communicated.
- Privacy policy: `https://filescience.io/privacy/`
  - Contains language implying broad platform coverage (e.g. “supporting 50+ third-party cloud platforms…”).

Repo-internal truth (authoritative for engineering work):
- Supported clouds today: **Clio**, **Microsoft 365**, **Box**
- Other `/clouds/*` pages are prospecting / interest tests.

## Blog framing (context; marketing content)
Blog index: `https://filescience.io/blog/`
- Cloud-to-cloud backup framing as a compliance + continuity requirement:
  - `https://filescience.io/blog/the-ultimate-guide-to-cloud-to-cloud-backup-in-2025/`
- Business continuity / disaster recovery framing (BCDR/DRaaS language appears here):
  - `https://filescience.io/blog/business-continuity-and-disaster-recovery-solutions/`
- Legal vertical insight and compliance language (ClioCon recap):
  - `https://filescience.io/blog/filescience-at-cliocon-bridging-the-gap-between-data-and-law/`
## Language/phrasing we should preserve (for future docs)
- “Effortless, enterprise-grade backup and recovery”
- “Keep your business flowing”
- “Interactive recovery viewer”
- “Immutable, air-gapped backups”
- “End-to-end encryption”
- “You own your data. Period.”

## Open items / what these expanded notes do NOT answer
- Internal roadmap/timeline for prospecting clouds (NetDocuments/Smartsheet/etc.) to become real supported integrations.
- Exact technical details of backup frequency, retention, RPO/RTO, encryption key management, etc. (marketing pages only).
