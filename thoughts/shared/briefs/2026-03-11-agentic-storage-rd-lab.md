# Idea Brief: Agentic Storage R&D Lab

**Date:** 2026-03-11
**Status:** Shaped → Parked

## Problem
Cutting-edge storage research (post-dedup delta compression, semantic deduplication, trained dictionaries, SIMD-accelerated chunking, similarity reduction) is published across siloed academic venues (ASPLOS, USENIX ATC, ACM ToS, IET, ICPP) but no SaaS backup product integrates these techniques — let alone combines them. The combinatorial space of existing techniques is enormous and almost entirely unexplored. Manually reading papers, implementing, and benchmarking doesn't scale.

## Constraints
- Can't use real PHI/customer data (HIPAA) — need representative synthetic datasets
- Agents are convergent thinkers — good at implementing and testing known techniques, novel invention is harder
- Every experiment must be reproducible
- Results must translate to production (not just academic benchmarks)
- Helm orchestrator exists for multi-agent coordination
- This is a Phase 2 investment — after the healthcare product ships

## Options Considered

### Manual R&D
Read papers manually, implement promising techniques one at a time.
- Gains: Low infrastructure investment
- Costs: Slow, doesn't scale, can't explore the combinatorial space
- Complexity: Low

### Agentic R&D Lab (Scientific Method)
Build a persistent lab environment where agents autonomously survey research, form hypotheses about technique combinations, implement, experiment, measure, and iterate — following the scientific method.
- Gains: Systematic exploration of combinatorial space. Agents connect dots across subfields that humans miss. Continuous improvement while you sleep. Negative results are recorded, preventing re-exploration. The technique catalog accumulates empirical data beyond paper claims.
- Costs: Significant upfront investment in pipeline framework, benchmark harness, experiment tracking. Agent quality must be validated (Reviewer role). Synthetic datasets must be representative.
- Complexity: High

## Chosen Approach
**Agentic R&D Lab** — because the value is in combinations nobody has tried, not individual techniques. The lab systematically searches that space.

## Key Architecture

### The Scientific Loop
1. **Survey** — agent reads papers/repos, maintains technique catalog with properties and composability
2. **Hypothesize** — agent scans catalog + prior results, proposes untried combination with falsifiable prediction
3. **Design** — experiment spec with pipeline config, control baseline, metrics, success criteria
4. **Implement** — technique as composable pipeline module with standard interface
5. **Execute** — run against benchmark dataset, collect metrics (ratio, throughput, memory, restore speed)
6. **Analyze** — compare against baseline and current best. Confirm or falsify hypothesis.
7. **Record** — update experiment ledger, technique catalog (empirical data), and leaderboard
8. Loop back to step 2, informed by everything learned.

### Agent Roles
- **Scout** — surveys papers, updates technique catalog, generates hypotheses (weekly)
- **Experimenter** — implements modules, runs pipelines, records results (per experiment)
- **Reviewer** — validates reproducibility, checks for measurement errors, updates leaderboard (per batch)

### Composable Pipeline Framework
Modules snap together: chunker → dedup → delta → compress → storage. Each implements `process(chunks) → chunks + metrics`. Agents swap any layer without touching others.

### Technique Catalog (Knowledge Graph)
Organized by layer (chunking, dedup, delta, compression, storage). Each technique has: paper source, claims, implementation status, empirical results, known interactions, open questions. Combination matrix tracks what's been tried.

### Promotion Path
Lab experiment → leaderboard threshold → Reviewer validates → human reviews → production candidate → integration tests → shadow mode → production.

## Frontier Techniques to Explore (Starting Catalog)

### Chunking Layer
- FastCDC (~3 GB/s, 10x Rabin), UltraCDC (~2.7 GB/s), QuickCDC (jump to known-duplicate boundaries)
- VectorCDC (2025, 26x faster via SIMD 128/256/512-bit), SeqCDC (2025, 15x faster vectorized)
- Keyed FastCDC (HIPAA privacy — per-tenant boundaries, zero performance cost)

### Dedup Layer
- Global cross-tenant dedup (Duplicacy-style, 30-60% savings on similar fleets)
- XOR filters (immutable index, most space-efficient), Cuckoo filters (mutable, supports delete)
- Learned Bloom filters (2025, 40% smaller via ML classifier)
- MinHashLSH + Bloom (2025, 3x faster, 0.6% disk space for approximate matching)

### Post-Dedup Delta Compression (highest-value layer)
- Palantir (ASPLOS 2024) — hierarchical super-features, +26.5% over Finesse
- LoopDelta (ACM ToS 2025) — inline delta, inversed delta for faster restore, cache-aware
- Chunk2vec (IET 2024) — Sentence-BERT embeddings for chunk similarity, ANN search
- SpeedSketch (ICPP 2025) — ultra-fast sketch generation for delta encoding

### Compression Layer
- Zstd with trained per-data-type dictionaries (healthcare email dict, SharePoint dict)
- VAST-style similarity batch compression (group similar chunks, compress together)
- Shared dictionary RFC (finalized Sept 2025, Google/Pinterest/Notion in production)

### Intelligence Layer
- SemDeDup — embedding-based semantic dedup, 50% data removal with minimal quality loss
- GPTrace (ICSE 2026) — LLM embeddings for dedup
- Deep transfer learning for healthcare cloud dedup (Expert Systems, 2023)

## Projected Impact
```
Traditional backup (Veeam):    ~2:1 reduction
Good backup (Druva global):    ~3:1 reduction
FileScience frontier engine:   ~8-15:1 reduction (projected)
```
At 10,000 tenants × 500GB avg = 5PB raw. 3:1 → 1.67PB ($38K/mo S3). 10:1 → 500TB ($11.5K/mo). Delta = $26K/mo pure margin, scaling linearly.

## Connection to Product
The embedding infrastructure for dedup (Chunk2vec, SemDeDup) is the SAME infrastructure that powers product features: temporal semantic search, NL change summaries, anomaly detection. The storage engine funds the product differentiation.

## Key Context Discovered During Shaping
- No SaaS backup vendor does post-dedup delta compression (all stop at basic dedup)
- VAST Data's similarity reduction achieves 3:1 in production but is hardware-only — nobody has brought this to cloud-native SaaS backup
- Keyed FastCDC solves HIPAA chunk-fingerprint leakage with zero performance cost
- VectorCDC (2025) achieves 26x speedup via SIMD — chunking is no longer the bottleneck
- The combinatorial space (5+ chunkers × 4+ dedup strategies × 4+ delta approaches × 3+ compression methods) = 240+ untested combinations

## Related Artifacts
- [[2026-03-05-deep-emergent-markets-for-filescience|Emergent Markets Research]] — version intelligence product vision
- [[2026-02-28-complexity-as-moat|Complexity as Moat]] — strategic rationale
- [[2026-03-09-platform-v2-roadmap-strategy|Platform V2 Roadmap]] — spiral passes execution model

## Next Step
- [Parked] → Linear backlog issue, revisit after healthcare HIPAA product ships (target: Q4 2026)
