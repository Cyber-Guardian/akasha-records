---
type: event
created: 2026-03-13
status: active
---

# Idea Brief: FileScience Knowledge Engine

**Date:** 2026-03-13
**Status:** Shaped → Parked

## Problem
FileScience's north star says "fast search, fast restore" but today there is zero search capability. The `resources` DynamoDB table stores metadata only (names, IDs, parent refs, delta tokens). Actual content lives in source clouds behind download URLs. No full-text index, no vector DB, no search API, no search UI. Users can't find what they need to restore. Meanwhile, competitors are just beginning to add AI search (Veeam: Cosmos DB + vector; Cohesity Gaia: conversational; Druva: metadata-only graph) — the window for differentiation is 3-4 years before GenAI in backup becomes commoditized (Gartner: 90% by 2029, vs <25% today).

## Constraints
- Multi-tenant isolation non-negotiable (keys already tenant-prefixed)
- Three clouds today: M365 (OneDrive + Outlook), Clio (docs + contacts), Box
- Heterogeneous content: files, emails, contacts, calendar events
- AWS-native stack (Lambda, DynamoDB, Valkey, S3, Terragrunt)
- Polylith architecture — search would be new component(s) + base(s)
- Scale target: 10M+ artifacts per tenant
- Agent-first (LLM/agent consumption is primary interface)
- Existing discovery pipeline already has delta tracking, throttle system, entity hierarchy

## Options Considered

### Option A: Metadata-Only Search with Semantic Layer
Index existing metadata (names, paths, types, dates, senders) with semantic embeddings on metadata fields. No content download.
- Gains: Fast to build, no new infrastructure beyond vector index, covers 80% of "find and restore" use cases
- Costs: Can't search inside documents/emails, no temporal content search, no eDiscovery/DLP expansion
- Complexity: Low

### Option B: Full Content Hybrid Search (OpenSearch)
Download and store all content in S3, process through extraction pipeline, index in OpenSearch Serverless with BM25 + kNN.
- Gains: Standard industry approach, AWS-managed, well-understood, enables content-level search
- Costs: OpenSearch OCU costs at scale (~$450/month base), no graph layer, no temporal search, not differentiated
- Complexity: Medium

### Option C: FileScience Knowledge Engine (SOTA Platform)
Three-layer architecture: hybrid content index (Turbopuffer, S3-native) + identity-aware knowledge graph (LazyGraphRAG over existing DynamoDB hierarchy) + temporal version index. Agent-native MCP interface with composable tools. Serves six product layers: search/restore, eDiscovery, DSPM/DLP, insider threat, compliance-as-a-service, organizational memory.
- Gains: Genuinely unprecedented capabilities (temporal search, cross-cloud identity, proactive intelligence). Platform play with six revenue streams. Massive competitive moat. Research shows economics are favorable (~$2,300 one-time per tenant for 10M backfill, ~$100-300/month steady-state).
- Costs: Largest engineering investment. Content download pipeline is a prerequisite. Multiple new infrastructure components. Phased delivery over multiple quarters.
- Complexity: High (but decomposable into independent phases)

## Chosen Approach
**Option C: FileScience Knowledge Engine** — the full platform play. The economics are surprisingly favorable (BGE-M3 self-hosted at $0.001/1M tokens, ColPali cheaper than OCR+embedding, content-addressed S3 storage with SHA-256 dedup). The competitive window is real and timed. The existing codebase provides strong foundations: delta tracking, throttle system, entity hierarchy, DynamoDB composite keys.

## Key Context Discovered During Shaping

### Architecture
- **BGE-M3** is the clear embedding choice: dense + sparse + multi-vector from one model, self-hosted on A10G at $0.001-0.003/1M tokens (20-60x cheaper than API)
- **Turbopuffer** for multi-tenant vector search: S3-native storage, millions of namespaces, cold tenants cost essentially nothing, BM25 + vector in one system
- **LazyGraphRAG** (not full GraphRAG): 0.1% of GraphRAG indexing cost, matches quality. NLP noun-phrase extraction builds concept graph; LLM relevance assessment only at query time
- **Content-addressed storage** with SHA-256 dedup: same file across 10K users = 1 S3 object. ~30% dedup rate expected
- **Late chunking** (free quality): run full document through BGE-M3, mean-pool over chunk boundaries. 10-12% retrieval improvement at zero cost
- **Contextual retrieval** (Anthropic pattern): prepend context blurb per chunk at index time. 35-67% fewer retrieval failures for ~$1/1M doc tokens one-time

### Agent Interface (A-RAG pattern, Feb 2026)
- Expose separate tools: `keyword_search`, `semantic_search`, `artifact_read`, `graph_expand`, `temporal_search`, `identity_search`, `classify`, `restore`
- Agents compose these adaptively — outperforms fixed pipelines (HotpotQA: 94.5% vs 81.2%)
- Three-part responses (Azure AI Search pattern): grounding data + source references + retrieval trace
- `output_schema` parameter: agent specifies JSON shape it wants back
- Adaptive complexity routing: simple (BM25 only, <50ms) → moderate (hybrid, ~200ms) → complex (multi-step, ~2-5s)
- CRAG (Corrective RAG) built into the service: auto-refine low-confidence retrievals before surfacing

### Competitive Landscape (as of March 2026)
| Vendor | Content Search | Semantic/Vector | Graph | Temporal | Agent API |
|--------|---------------|-----------------|-------|----------|-----------|
| Veeam | M365 only | Yes (new, Cosmos DB) | No | No | MCP (announced) |
| Cohesity Gaia | Yes (on-prem) | Yes | No | No | Copilot/Glean |
| Druva | **Metadata only** | No | Yes (MetaGraph) | No | Agents (metadata) |
| Rubrik | Classification only | No | No | No | No |
| Commvault | Optional add-on | No | No | No | No |

Genuine gaps nobody fills: temporal content search across versions, cross-cloud identity-aware search, content-level agent API with composable tools, proactive intelligence from backup deltas.

### Unprecedented Differentiators
1. **Temporal search**: "What did this document say 6 months ago?" — search across all versions simultaneously. Nobody does this.
2. **Cross-cloud identity**: "Show me everything related to Jordan across M365, Clio, Box" — Entra ID as canonical source, email as universal join key, union-find clustering.
3. **Proactive intelligence from free signals**: compression ratio anomaly (ransomware), deletion spikes (destruction), restore-before-departure (insider threat) — $0 cost, already computed by backup pipeline.
4. **Agent-native composable search → restore workflow**: search + graph expand + temporal diff + classify + restore in one agentic chain.

### Platform Revenue Streams (six layers on one index)
| Layer | Market | Buyer |
|-------|--------|-------|
| Search + Restore | Core product | IT/MSP ops |
| eDiscovery + Legal Hold | $20B (2026) | Legal/GRC |
| DSPM/DLP | $10B (2033) | CISO/Security |
| Insider Threat Detection | $17.4M/incident avg | Security ops |
| Compliance-as-a-Service | $44B GRC (2029) | MSPs → clients |
| Organizational Memory | $7.7B KM (2025) | CTO/CHRO |

### Content Pipeline Economics (per tenant, 10M artifacts)
| Component | Backfill (one-time) | Monthly |
|-----------|-------------------|---------|
| Content download + S3 | ~$100 | ~$23 → $4 (tiered) |
| BGE-M3 embeddings (self-hosted) | ~$150-250 | ~$15-25 |
| Contextual retrieval enrichment | ~$30K (Haiku) | negligible |
| Turbopuffer / vector index | — | ~$50-200 |
| **Total** | **~$2,300** | **~$100-300** |

### Cross-Cloud Identity Resolution Pipeline
- Tier 1: Structured ID match (Entra objectId, Box SCIM externalId) — 85-90%
- Tier 2: Email exact match against all proxyAddresses aliases — 7-10%
- Tier 3: Display name fuzzy (Levenshtein/Jaro-Winkler) — 2-3%
- Tier 4: Embedding similarity (bi-encoder + FAISS ANN) — 1%
- Tier 5: NER extraction from content (GLiNER, CPU-only) — <1%
- Union-Find clustering → canonical IdentityNode with bitemporal edges

### Proactive Intelligence (free + enriched signals)
| Signal | Cost | Detects |
|--------|------|---------|
| Compression ratio change | $0 (already computed) | Ransomware |
| Files deleted/modified ratio | $0 (already computed) | Mass destruction |
| Restore-before-departure | $0 (novel) | Insider exfiltration |
| Version count spikes | $0 (already tracked) | Unusual activity |
| PII drift (new locations) | Medium (needs classifier) | Compliance violations |
| Key-person risk | High (needs org graph) | Knowledge concentration |

### Suggested Phasing
1. **Content pipeline**: S3 storage + download + MIME parse + text extraction + SHA-256 dedup
2. **Hybrid search index**: BGE-M3 embeddings + Turbopuffer (BM25 + dense) + basic MCP tools
3. **Identity resolution**: Entra-rooted cross-cloud identity graph + identity-aware search
4. **Temporal search**: Version-aware embeddings + diff search + snapshot timeline queries
5. **Proactive intelligence**: Free signals first (compression, deletion, restore patterns), then PII classification
6. **Advanced layers**: eDiscovery workflow, DSPM/DLP scanning, org knowledge graph

### Key Research Sources
- A-RAG (Feb 2026): hierarchical retrieval tools for agents — arXiv 2602.03442
- LazyGraphRAG: 0.1% of GraphRAG cost — Microsoft Research blog
- HippoRAG: hippocampal-inspired graph retrieval, 10-20x faster — arXiv 2405.14831
- Path-Constrained Retrieval: 100% structural consistency — arXiv 2511.18313
- Turbopuffer: S3-native multi-tenant vector search — turbopuffer.com/docs/architecture
- BGE-M3: dense + sparse + multi-vector — huggingface.co/BAAI/bge-m3
- Anthropic Contextual Retrieval: 35-67% fewer failures — anthropic.com/news/contextual-retrieval
- Microsoft Graph scan guidance: delta-based content download patterns
- Splink: production entity resolution — moj-analytical-services.github.io/splink
- TGFR: hybrid ER framework, 0.978 F1 — arXiv 2509.17470

## Next Step
- [Parked] → Linear backlog issue with brief linked as context. Revisit when V2 Platform roadmap schedules search/retrieval work.
