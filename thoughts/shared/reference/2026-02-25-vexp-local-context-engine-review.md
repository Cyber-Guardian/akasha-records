# Reference: vexp Local-First Context Engine Review

**Date:** 2026-02-25  
**Scope:** Evaluate product claims and practical fit for FileScience workflows  
**Sources:** `https://vexp.dev/` and Reddit thread URL provided by user

## What vexp appears to be

vexp presents as a local-first context engine for coding agents with:
- local indexing via tree-sitter + SQLite
- graph-assisted retrieval (calls/imports/implementations)
- "pivot + skeleton" context packaging to reduce token usage
- MCP tooling integration for agent workflows
- multi-repo graph and session-memory features

## Strong claims that are plausible

- **Meaningful token reduction is plausible** when switching from full-file dumps to focused pivots + signatures.
- **Local-first architecture is credible**: AST parsing and graph retrieval do not require cloud by default.
- **Graph-based blast-radius analysis is valuable** for refactors and impact scoping.
- **LSP edge augmentation can improve precision** for typed call relationships compared to AST-only linking.

## Claims to treat as marketing until independently verified

- **"No hallucination"**: retrieval quality can improve hallucination rate but cannot eliminate model hallucinations.
- **"Zero missed dependencies" / complete context coverage**: static + LSP graphing helps, but dynamic behavior, runtime wiring, and external boundaries can still be missed.
- **Specific performance numbers** (`<15s` index, `<500ms` P95, `65-70%` savings): likely environment-dependent and should be benchmarked on our codebase.
- **Overhead claims** (e.g., near-zero session capture overhead): possible, but needs measurement under real developer load.

## Fit for FileScience

Potential upside:
- lower token spend and tighter context handoffs in large multi-component tasks
- better dependency/impact visibility before edits (aligns with import-linter discipline)
- stronger cross-repo traceability if/when more repos are coordinated in daily work

Potential risk:
- extra moving parts in the agent loop can add failure/debug surface
- retrieval quality can create false confidence if developers assume completeness
- tool lock-in if process artifacts rely on proprietary tool semantics

## Adoption checklist (recommended)

1. **Repro benchmark on this repo**
   - Baseline: current token usage + task latency over 10 representative tasks
   - Trial: same tasks with vexp context capsule flow
   - Metrics: token delta, task completion time, regression rate, number of follow-up fixes

2. **Correctness stress tests**
   - dynamic call paths, polymorphic dispatch, generated code, and cross-boundary changes
   - compare retrieved impact graph vs actual changed call sites

3. **Operational checks**
   - index freshness under rapid file changes
   - failure behavior when graph is stale or partial
   - developer ergonomics: false positives/negatives and override flow

4. **Security/privacy validation**
   - verify network egress behavior in local environment
   - inspect storage locations and retention for session memory and manifests

## Bottom line

vexp’s core architecture (local graph indexing + bounded context capsules) is technically sound and likely useful, especially for token pressure and navigation quality.  

Treat "no hallucination" and "complete dependency certainty" as non-literal claims. The right path is a short internal benchmark with explicit success criteria before workflow-wide adoption.

## Links

- Product site: https://vexp.dev/
- Reddit thread (requested for review): https://www.reddit.com/r/ClaudeCode/comments/1rdo5ul/i_cut_claude_codes_token_usage_by_65_with_a_local/
