---
type: event
created: 2026-03-26
status: active
touched: 2026-03-26
tags: [strategy, product, akasha, filescience, carrie, caroline, agent-safety, version-control, gtm]
---
# Product Thesis: Akasha — The Undo Button for the Agent Era

**Date:** 2026-03-26
**Status:** Active
**Origin:** Shaped from cognitive architecture exploration → competitive gap analysis

---

## 1. The Insight

FileScience already has the hardest piece: a sync engine, discovery engine, recovery engine, and entity graph across cloud services. Reframed as agent infrastructure instead of backup, this becomes **version control for everything agents touch** — the "git for the agent era."

Combined with Carrie/Caroline (a cognitive agent platform with pluggable capabilities), Akasha occupies a position no one else can reach:

1. **Version control across every system an agent touches** (not just code — SaaS configs, documents, financial data, cloud state)
2. **Agent-native API primitive** ("before you act, snapshot; if wrong, revert") — not a backup restoration workflow
3. **An intelligent agent platform** with pluggable capabilities that uses this safety layer natively

No existing product combines all three. The backup vendors (Rubrik, Veeam, Cohesity) are racing toward #1-2 but will never become agent platforms. The agent platforms (LangChain, CrewAI) have zero interest in #1-2. The gap is real and validated.

---

## 2. Competitive Landscape

### Who's Doing Pieces

| Player | What They Have | What They're Missing |
|---|---|---|
| **Rubrik Agent Rewind** (Aug 2025) | Selective rollback of agent changes. Integrates with Agentforce, Copilot Studio, Bedrock. | Not an agent platform. Restore from backup, not version-as-workflow. |
| **Veeam Agent Commander** (Feb 2026) | "Detect AI. Protect AI. Undo AI." Data Command Graph. Per-file undo. | Not an agent platform. M365 only (currently). |
| **Cohesity + ServiceNow** (Mar 2026) | Immutable snapshots + ServiceNow orchestration + Datadog anomaly triggers. | Not an agent platform. Backup-first DNA. |
| **Druva DruAI** | Most ambitious: AI agents INTO backup platform. Open MCP/A2A framework. | Agents operate on backup metadata only. Not general-purpose capabilities. |
| **LangChain/CrewAI/AutoGen** | Agent orchestration frameworks. | Zero state versioning for agent effects. |
| **Dolt** | Git for SQL databases. Branch, merge, diff. | SQL only. Not agent-aware. Not a platform. |
| **lakeFS** | Git for data lakes (S3/GCS). | Data lakes only. Not SaaS state. Not agent-aware. |
| **Replit Snapshot Engine** | Architecturally closest: pre-action snapshots + DB forks + agent-aware. | Scoped to Replit development environments only. |

### The Whitespace

Nobody combines: agent-native versioning across all digital assets + an intelligent agent platform with pluggable capabilities + cognitive architecture that makes the agent good at managing.

---

## 3. The Product: One Agent, Pluggable Capabilities, Built-in Undo

### Carrie / Caroline

One agent interface. Pluggable capability modules. Each capability provides connectors, tools, personality profile, verification rules, knowledge context, and triggers.

### Capabilities (ordered by ship priority)

| # | Capability | Connectors | Primary User | Revenue |
|---|---|---|---|---|
| 1 | **Dev** | Linear, GitHub, Slack, CI/CD, LSP | Internal (dogfood) | None — validates architecture |
| 2 | **Ops** | Slack, Linear, Metrics, Status | Internal (dogfood) | None — validates management loop |
| 3 | **IT** | FileScience MCP, Clio, M365, Intercom | CGCG clients | B2B — first external capability |
| 4 | **Marketing** | Analytics, CMS, Social, Copywriting | Internal + external | B2B |
| 5 | **Finance** | Plaid, Stripe, Budget engine | Consumer beta | B2C — forces connector generalization |
| 6 | **Health** | HealthKit, Oura, FHIR | Consumer beta | B2C — forces derivative context |
| 7 | **Time** | Calendar, Todoist, Screen Time | Consumer beta | B2C |

### The Undo Layer (FileScience)

Every capability gets undo for free:
- Before agent action → FileScience snapshots pre-action state
- During action → cognitive architecture verifies (PRM + Z3 = prevention)
- After action → entity graph records what changed, when, which agent, why
- On failure → "Hey Carrie, undo that" → FileScience restores

```yaml
# capability contract — undo section
undo:
  strategy: snapshot
  scope: connector
  retention: 90d
  irreversible:
    - send_email
    - process_payment
    - post_to_social
  safeguard: require_confirmation  # for irreversible actions
```

### The Git Analogy

| Git (code) | FileScience (everything else) |
|---|---|
| `git status` | Discover — what changed across connected services? |
| `git log` | Entity graph — full timeline, audit trail |
| `git diff` | Delta engine — what's different between then and now? |
| `git revert` | Recovery — restore any entity to any prior state |
| `git branch` | Sandbox — let agents experiment before committing |
| `git blame` | Agent audit trail — which agent, why |
| `.gitignore` | Scope policy — what agents can touch, what's protected |

---

## 4. The Trust Ladder (Completed)

```
Level 1 — BACKUP TRUST: "FileScience protects my data"
Level 2 — READ TRUST: "Carrie understands my data"
Level 3 — COACHING TRUST: "Carrie advises me well"
Level 4 — ACTION TRUST: "Carrie acts on my behalf — and if anything
           goes wrong, FileScience reverts it instantly"
```

Action trust is unlocked by the undo button. Not by accuracy alone.

---

## 5. The Cognitive Architecture

Carrie/Caroline uses a biologically-grounded cognitive architecture (13 elements, no existing system combines all):

- **Modulation:** 4-axis neuromodulator model (effort/novelty/caution/focus) adapts reasoning per task
- **Memory:** 4-tier context manager + surprise-driven episodic memory + Scout semantic knowledge
- **Verification:** PRM somatic markers + Z3 formal constraints + MCTS reasoning search
- **Learning:** GRPO on telemetry → custom models, self-play training data generation
- **Evolution:** DGM/AlphaEvolve pattern — the harness evolves its own management strategies

Full research: [[2026-03-26-cognitive-architecture-research|Cognitive Architecture Research Brief]]
Roadmap: [[2026-03-26-cognitive-architecture-roadmap|Cognitive Architecture Roadmap]]

---

## 6. Three Parallel Tracks

```
Track A: Dogfood             Track B: B2C Prototype        Track C: FileScience
Dev + Ops capabilities       Finance + Health caps          Connector abstraction
Validates cognitive loop     Forces generalization          Product infrastructure
Internal use                 50 users hands-on              Revenue
    │                             │                              │
    └──────────────┬──────────────┘──────────────────────────────┘
                   ▼
          SHARED MODULAR CORE
          Cognitive components + Connectors + Entity graph +
          MCP surface + Agent runtime + Undo layer
```

Every feature built for any track benefits all tracks. The platform crystallizes from building products.

---

## 7. Business Model

| Tier | What | Price |
|---|---|---|
| Free | Carrie + 2 basic capabilities (calendar, reminders) | $0 |
| Pro | Carrie + all first-party capabilities | $20/mo |
| Enterprise | Caroline + custom capability configs + CGCG IT ops + undo | Custom |
| Platform | Capability SDK — "build your own capability" | Revenue share |

---

## 8. The Narrative

**For investors:**
> "We're the undo button for the agent era. Agents are going to manage everything — your business, your finances, your schedule. But they're going to make mistakes. FileScience version-controls everything agents touch, and Carrie is the intelligent agent that acts through it. We're the company you trust to protect your data AND act on it."

**For enterprise:**
> "Deploy AI agents with confidence. FileScience tracks every change your agents make across every connected service. If anything goes wrong — wrong config, deleted file, bad migration — one click to turn back the clock."

**For consumers:**
> "Meet Carrie. She manages your money, your schedule, your health data. And if she ever gets something wrong, just say 'undo.'"

---

## 9. Why We Win

1. **We already have the hard piece.** Sync engine, discovery, recovery, entity graph across cloud services. Rubrik/Veeam built this over a decade.
2. **Agent-first, not backup-first.** Backup vendors are bolting agent awareness onto infrastructure. We're building the agent platform with safety built in.
3. **The architecture compounds.** Every interaction improves the cognitive architecture. Every capability deepens the entity graph. Every undo builds trust data.
4. **Temporal irreplaceability.** Clone our features, not our memory of your digital life.
5. **The cognitive architecture is novel.** 13 elements no one has assembled. Biologically-grounded modulation, PRM somatic markers, self-improving harness.

---

## 10. Related

- [[2026-03-26-cognitive-architecture-research|Cognitive Architecture Research Brief]]
- [[2026-03-26-cognitive-architecture-roadmap|Cognitive Architecture Roadmap]]
- [[2026-03-20-personal-life-ontology-agent-platform|Akasha — The Interface Substrate]]
- [[2026-03-20-akasha-capability-platform|Akasha Capability Platform]]
- [[2026-03-20-jig-rust-agent-core|Jig — Rust Agent Core]]
- [[2026-03-12-agent-harness-composition|Cognitive Personalities]]
