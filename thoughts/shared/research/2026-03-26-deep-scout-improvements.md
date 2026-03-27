---
type: event
created: 2026-03-26
status: active
date: 2026-03-26T16:30:00-04:00
researcher: Claude
git_commit: 220c1ba6
branch: main
repository: filescience
topic: "How can we further improve Scout as an agentic search substrate?"
tags: [deep-research, scout, agentic-search, retrieval, embedding, exploration-engine]
research_depth: standard
iterations_completed: 2
last_updated: 2026-03-26
last_updated_by: Claude
---

# Deep Research: Scout Agentic Search Substrate Improvements

## Research Question
How can we further improve Scout as an agentic search substrate? Covering: exploration engine improvements, smarter corpus adapters, SOTA agentic RAG patterns, subagent effectiveness, embedding model upgrades, and index freshness.

## Summary
Scout's highest-leverage improvements are (1) fixing 5 concrete bugs in the exploration engine that prevent it from working as designed, (2) upgrading from Model2Vec potion-base-8M to bge-small-en-v1.5 for a nearly-free 27% retrieval quality improvement, and (3) adding CRAG-style retrieval grading to filter irrelevant results before they reach agents. The knowledge-explorer subagent would benefit most from Multi-HyDE query generation, step-back prompting, and a mandatory contradiction-search step. These are all implementable without GPU, training, or heavy infrastructure.

## Perspectives Explored
1. **Retrieval Architecture** — Self-RAG, CRAG, and Adaptive-RAG represent the SOTA for self-reflective retrieval. CRAG's grading and Adaptive-RAG's query routing are directly applicable.
2. **Embedding & Ranking Quality** — The 27% quality gap between potion-base-8M and bge-small is real and closable at near-zero latency cost at our scale.
3. **Corpus Adapter Intelligence** — Centroid-similarity routing and CRAG confidence filtering are the practical patterns for our scale.
4. **Subagent Effectiveness** — The exploration engine has 5 fixable bugs; knowledge-explorer needs Multi-HyDE and contradiction search.
5. **Index Freshness & Coherence** — Our TTL pattern is within production norms; git hooks would be the next step.

## Detailed Findings

### Retrieval Architecture

**Self-RAG** (ICLR 2024 Oral, Asai et al.) trains reflection tokens that decide on-demand whether to retrieve, evaluate relevance, and critique output. Only 2% of correct predictions happen without retrieved passages vs 18-20% for vanilla LLMs. Production-viable via framework re-implementations but requires fine-tuned model.

**CRAG** (Corrective RAG) wraps a lightweight evaluator around any RAG pipeline. Classifies retrieved documents as correct/ambiguous/incorrect and triggers web search fallback when confidence is low. Self-CRAG combo achieves highest FactScore gains (+0.456 on Biography). The most production-ready pattern — implementable as a composable middleware.

**Adaptive-RAG** (Jeong et al. 2024) classifies queries into no-retrieval, single-step, or multi-step BEFORE retrieval begins. Can be approximated without training using query length + entity chain heuristics.

**For Scout**: CRAG-style grading is the highest-leverage addition. Two implementation paths:
1. **Cross-encoder reranker** (bge-reranker-v2-m3 via sentence-transformers): ~130ms for batches under 30 candidates, no API call. Score threshold filters irrelevant chunks.
2. **LLM-as-judge** (single Haiku call, binary "is this relevant?"): ~100-150ms per batch, more accurate but adds API dependency.

Grading should insert after fan-out retrieval but BEFORE cross-corpus merge/dedup — prune irrelevant chunks per-document, not per-corpus.

### Embedding & Ranking Quality

potion-base-8M scores **31.11 NDCG@10** on MTEB retrieval vs:
- all-MiniLM-L6-v2: 42.92 (+38%)
- bge-small-en-v1.5: mid-40s NDCG (+45%)
- snowflake-arctic-embed-s: mid-40s NDCG (+45%)

The 27% quality gap is significant even at ~2000 chunks. However, at our scale **BM25 is competitive with neural embeddings** — our hybrid BM25+neural approach (already implemented) is the strongest practical choice regardless of model.

**Recommendation**: Upgrade to bge-small-en-v1.5.
- 33M params, 384-dim, sub-20ms on Apple Silicon CPU
- LanceDB has a built-in `sentence-transformers` registry entry — no custom wrapper needed
- Migration requires table recreation (dimension change from 256 to 384), but LanceDB supports dual-vector columns during migration
- Speed cost is negligible at our scale (latency is index-lookup dominated, not inference dominated)

**nomic-embed-text-v1.5** is the alternative if we want to stay at 256-dim (Matryoshka truncation) — outperforms text-embedding-3-small at 512-dim.

### Corpus Adapter Intelligence

**Centroid-similarity routing**: Embed the query, compute cosine against pre-computed per-corpus centroid embeddings, route to top-k corpora. No training needed, negligible compute. This is the most practical query router for our scale.

**Adaptive-RAG's 3-tier classification** can be approximated with heuristics:
- Short, specific query (< 5 words, named entity) -> single corpus, keyword search
- Medium query (question form, 5-15 words) -> hybrid search, single corpus
- Complex query (multi-part, reasoning required) -> fan-out all corpora, multi-step

**Relevance feedback**: At ~1000 chunks, LLM-generated sub-queries (RAG-Fusion pattern) beat click-through learning (insufficient signal volume). Generate 3-5 query variants, search each, RRF-merge.

### Subagent Effectiveness

#### Exploration Engine Bugs (all difficulty 2-3, zero test breakage risk)

1. **Source overlap metric is monotonic** (engine.py:580): Cumulative scan over all findings. Fix: compute inter-iteration delta using prior reflection's `source_overlap_pct`. Difficulty: 2.

2. **Angle priority unused** (engine.py:644): `_suggest_next_action` ignores `Angle.priority`. Fix: sort angles by priority before building underexplored list. Difficulty: 2.

3. **`plan_still_aligned` hardcoded True** (engine.py:253): Never computed. Fix: compare current angle set against original plan. Difficulty: 2.

4. **Heuristic angle generation** (engine.py:440): Five hardcoded templates, no semantic inference. Fix: add optional LLM-based angle suggestion (requires async signature change). Difficulty: 3.

5. **Source-string-only dedup** (engine.py:765): Semantically identical content from different URLs survives. Fix: add snippet-similarity check (difflib ratio, O(n^2) per batch). Difficulty: 2.

#### Agent Synthesis Strategy Improvements

**Multi-HyDE** (Multi-query Hypothetical Document Embeddings): Generate multiple hypothetical answer documents, search for similar. Achieves recall 0.817 vs 0.689 for standard HyDE. Implementable as a prompt instruction for knowledge-explorer.

**Step-back prompting** (ICLR 2024): Abstract the query to high-level principles before searching. Improved MMLU/TimeQA by 7-27%. Zero-code change — pure prompt engineering.

**Contradiction search**: After initial synthesis, explicitly search for contradicting evidence. One of the highest-ROI prompt-implementable improvements. Deep research agents score below 68% on expert rubrics (ResearchRubrics) — technical nugget coverage is especially weak (~40%).

**Stopping heuristic**: Agents should halt when sub-questions reach coverage saturation (diminishing returns on new sources), not at a fixed iteration count.

### Index Freshness & Coherence

Our current TTL-based polling (5-min `_maybe_resync`) is within production norms for low-velocity corpora. Next level improvements:

- **Git post-receive hook** for commit index: trigger incremental reindex on push instead of polling
- **Chunk-level hash comparison**: Only re-embed changed chunks (compare content hashes, keep lineage for stale-row deletion) — 99%+ savings vs full re-embed
- **Staleness monitoring**: Track "time from source change to index availability" as a first-class metric

## Prioritized Improvement Roadmap

| Priority | Improvement | Effort | Impact | Dependencies |
|----------|-----------|--------|--------|-------------|
| 1 | Fix 5 exploration engine bugs | 1-2 days | High — makes existing system work as designed | None |
| 2 | Upgrade to bge-small-en-v1.5 | 1 day | High — 27% retrieval quality improvement | Table recreation + SCHEMA_VERSION bump |
| 3 | Add CRAG-style retrieval grading | 1-2 days | High — filters irrelevant results before agents | Cross-encoder or Haiku dependency |
| 4 | Improve knowledge-explorer search protocol | Half day | Medium — Multi-HyDE, step-back, contradiction search | Prompt changes only |
| 5 | Add centroid-similarity query routing | Half day | Medium — smarter corpus selection | Pre-computed centroids |
| 6 | Git hook-triggered commit reindexing | Half day | Low — reduces staleness from 5min to seconds | Git server hook access |

## Key Sources

### Codebase
- tools/scout/src/scout/engine.py:440-493, 580-598, 644-665, 765-778
- tools/scout/src/scout/state.py:95

### Research Papers
- Self-RAG (arXiv:2310.11511) — self-reflective retrieval with reflection tokens
- CRAG (arXiv:2401.15884) — corrective retrieval augmented generation
- Adaptive-RAG (arXiv:2403.14403) — query complexity routing
- MA-RAG (arXiv:2505.20096) — multi-agent CoT for multi-hop
- Step-Back Prompting (arXiv:2310.06117) — ICLR 2024
- Multi-HyDE (arXiv:2509.16369) — multi-query hypothetical document embeddings
- ResearchRubrics (arXiv:2511.07685) — benchmark for deep research agents
- Arctic-Embed (arXiv:2405.05374) — scalable text embedding models
- RAGRoute (arXiv:2502.19280) — efficient federated search

### Production Systems
- Perplexity on Vespa.ai (vespa.ai/perplexity/)
- LangGraph CRAG implementation (blog.langchain.com)
- LanceDB sentence-transformers registry (lancedb.com/documentation)
- BAAI/bge-small-en-v1.5 (huggingface.co)

## Open Questions
- Would a cross-encoder reranker (bge-reranker-v2-m3) or LLM-as-judge (Haiku) produce better relevance grading for our corpus?
- Should angle generation be LLM-powered or is the heuristic approach sufficient with the other fixes?
- What's the right snippet-similarity threshold for semantic dedup (Jaccard? difflib? embeddings)?
- Should we add Matryoshka support (nomic-embed) as a fallback for environments where bge-small is too slow?

## Research Metadata
- Depth mode: standard
- Iterations completed: 2 / 5
- Termination reason: all high-priority gaps exhausted, remaining gaps are medium-priority incremental
- Manifest: `.claude/deep-research/2026-03-26-how-can-we-further-improve.md`
