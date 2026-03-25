# Idea Brief: Triage Agent Memory Bank Access

**Date:** 2026-03-05
**Status:** Shaped -> Planning

## Problem
The triage agent (PydanticAI, durable Lambda harness) already has multi-turn tool_use with `search_linear_issues`, `create_linear_issue`, `ask_human`, and `post_slack_message`. But it operates in a context vacuum — zero project knowledge beyond a static YAML routing table with 12 regex patterns. It can't reason about component architecture, active priorities, prior decisions, or existing plans when shaping issues. Issue routing accuracy, description richness, and duplicate detection all suffer.

## Constraints
- Agent runs on Haiku 4.5 in Lambda — cost-sensitive ($1/$5 per M input/output tokens)
- Current zip is 6.1 MB with 244 MB headroom
- Processor Lambda has no S3 permissions today — needs IAM policy + env vars
- Memory-bank is 3.9 MB total (~993K tokens) — only durable+topics are practical to inline (~5,900 tokens)
- S3 sync workflow from [[2026-03-03-memory-mcp-server|Memory MCP plan]] designed but not implemented
- PydanticAI `@agent.tool` with `RunContext[Deps]` is the canonical pattern for retrieval tools
- Anthropic guidance: progressive disclosure — agents should retrieve context on demand, not load everything upfront
- `sqlite3` FTS5 is in Python stdlib — zero extra dependencies for search
- Knowledge base deployments must be decoupled from Lambda code deployments

## Options Considered

### S3 Sync + Custom Index Generation
S3 sync on push to main. Generate a JSON index file with titles, keywords, summaries. Lambda loads index on cold start, searches in-memory, fetches matching docs from S3.
- Gains: Fast search, decoupled deploys, reusable index
- Costs: Custom index format, custom search code, no ranking quality guarantees
- Complexity: Medium

### S3 Sync + Full Memory MCP Server
Implement the complete [[2026-03-03-memory-mcp-server|Memory MCP plan]] — Lambda-backed MCP server with search tools. Triage agent consumes via HTTP.
- Gains: Centralized search for all agents, matches existing plan
- Costs: New Lambda to build/deploy/maintain, MCP wiring, extra hop latency
- Complexity: High

### QMD over HTTP (expose existing local server)
Run QMD (`qmd mcp --http`) on EC2/Fargate. Lambda calls it over HTTP.
- Gains: Reuse exact search quality, zero new search code
- Costs: Always-on infra ($5-15/month), ~2GB GGUF models, auth/networking
- Complexity: Medium

### S3 Sync + SQLite FTS5 Index
Pre-build an FTS5 index in CI, upload to S3 alongside raw content. Lambda downloads index to /tmp on cold start. BM25 search via stdlib sqlite3. Agent gets `search_memory` and `get_memory_doc` tools.
- Gains: Battle-tested BM25, zero extra dependencies (stdlib), single portable file, decoupled deploys, reusable by any Lambda
- Costs: BM25 only (no semantic search — fine for 218 docs), CI build script needed
- Complexity: Low

### LanceDB on S3 (hybrid search)
Embedded DB with S3 as first-class backend. Hybrid BM25 + vector in one query.
- Gains: Semantic + keyword search, S3-native, scales well
- Costs: New dependency, needs Bedrock for embeddings, overkill for 218 docs
- Complexity: Medium-High

## Chosen Approach
**S3 Sync + SQLite FTS5 Index** — For 218 markdown files, BM25 is sufficient. SQLite FTS5 is stdlib (zero deps), the index is a single portable file, and the CI build script is ~50 lines of Python. Decoupled from Lambda deploys — push new markdown, CI rebuilds index and syncs to S3. Graduation path to LanceDB if semantic search is needed later.

## Key Context Discovered During Shaping
- The triage bot was **already rewritten** as a PydanticAI agent at `tools/bases/tooling/triage_agent/` — the old `tools/triage-bot/src/triage_bot/` is legacy code. Handler, tools, prompts, dispatcher, callback handler all exist in the new location.
- Durable harness tool registry at `tools/components/tooling/durable_harness/tools.py` — `@register_tool()` decorator pattern. Adding new tools is straightforward.
- `identity.md` (~900 tokens) should go in the system prompt (always relevant). Topic notes and archive should be tool-accessible on demand.
- Prompt caching threshold for Haiku 4.5 is 4,096 tokens — system prompt should either stay well under or deliberately exceed to benefit from caching.
- S3 read latency is ~100-200ms same-region for small objects. Index download to /tmp is one-time per cold start.
- Memory MCP plan Phase 2 (S3 sync workflow) is designed but not yet implemented — can be reused here.

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-05-triage-agent-memory-access.md`
