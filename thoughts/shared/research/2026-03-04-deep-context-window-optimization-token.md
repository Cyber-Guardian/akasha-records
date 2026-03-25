---
date: 2026-03-04T12:00:00-05:00
researcher: Claude
git_commit: bf8dd50f2a584ab7f07d4bd73446e9a7fef16be4
branch: main
repository: filescience
topic: "Context window optimization and token reduction strategies for Claude Code"
tags: [deep-research, token-optimization, hooks, context-management, subagents]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: Context Window Optimization & Token Reduction

## Research Question
How can we make the most of our context window and reduce token outputs from tool calls and other sources?

## Summary
The largest controllable token costs are subagent spawns (~50K baseline each), large file reads (up to 25K tokens with 1.7x line-number overhead), and unbounded Bash stdout. The most impactful intervention is a **PreToolUse read-limiter hook** that auto-injects `limit` on large files and blocks wasteful paths before tokens enter context — this is strictly more effective than PostToolUse compression because `additionalContext` is additive-only (never replaces original output). Beyond hooks, **subagent model discipline** (haiku for search, sonnet for implementation, maxTurns limits) and **session rotation at 60-65% capacity** with handoff documents are the next highest-leverage practices. Our always-loaded instruction context (~3,290 tokens) is already lean and not worth further compression.

## Perspectives Explored
1. **Hook Engineering** — Mapped our complete hook system (6 event types, zero output compression), identified PreToolUse `updatedInput` as the highest-impact lever, and documented the limitations of PostToolUse additionalContext.
2. **Context Architecture** — Established subagent isolation model (only final result returns to parent), session lifecycle budgeting (rotate at 60-65%), and PreCompact pipeline status (bugged).
3. **Tool Output Economics** — Built empirical cost hierarchy from subagent spawns (~50K) down to always-loaded instructions (~3.3K), with per-tool definition costs.
4. **Frontier Techniques** — Catalogued MCP proxy ecosystem (mcproxy, token-optimizer-mcp, mcp-compressor, TOON), confirmed PreCompact hook limitations, and documented environment variables.
5. **Prompt & Instruction Optimization** — Measured always-loaded context at ~3,290 tokens, identified CLAUDE.md routes as most compressible but diminishing returns at current size.

## Detailed Findings

### 1. Hook Engineering — Prevention Over Compression

**The key insight:** Preventing tokens from entering context (PreToolUse) is strictly more effective than compressing them after (PostToolUse), because PostToolUse `additionalContext` is additive — it appends alongside the original output, never replacing it. Only MCP tools support true output replacement via `updatedMCPToolOutput`.

**Our current hook gaps vs. claude-warden:**

| Capability | claude-warden | Our hooks | Gap |
|---|---|---|---|
| Read path blocking (node_modules, dist) | read-guard | scope_guard (write-only) | **Missing** |
| Read size limiting (>300 lines → skeleton) | read-compress | None | **Missing** |
| System-reminder stripping | post-tool-use | None | **Missing** |
| Bash output truncation (>20KB) | post-tool-use | RTK (command rewrite only) | **Missing** |
| Verbose command blocking (npm install) | pre-tool-use | None | **Missing** |
| Write scope enforcement | N/A | scope_guard | We have this |
| Command rewriting for token savings | N/A | RTK (rtk-rewrite.sh) | We have this |

**Recommended first hook — PreToolUse Read Limiter (Python, ~100 LOC):**
- Match on `Read` tool
- `os.path.getsize()` check → deny files >2MB
- `wc -l` or heuristic → auto-inject `limit: 200` for files >300 lines
- Pattern-match path → deny `__pycache__/`, `.venv/`, `node_modules/`, `dist/`, `.terraform/`, `.terragrunt-cache/`
- Output: `updatedInput` with `limit` field, or `permissionDecision: deny` with reason

**PreToolUse `updatedInput` format:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "file_path": "/path/to/file.py",
      "limit": 200
    }
  }
}
```

Available since v2.0.10. Single hook with matcher `Read|Bash|mcp__.*` can cover multiple tool types.

Sources:
- [Hooks reference](https://code.claude.com/docs/en/hooks) — PreToolUse schema, updatedInput, decision control
- [claude-warden](https://github.com/johnzfitch/claude-warden) — read-guard, read-compress implementations
- [PreToolUse updatedInput feature](https://github.com/anthropics/claude-code/issues/4368) — implemented in v2.0.10
- `.claude/settings.json:33-135` — current hook registrations

### 2. Context Architecture — Isolation and Lifecycle

**Subagent isolation model:**
- Only the final result/summary returns to parent context — full transcripts stored in `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`
- Each subagent spawns its own 200K context window with ~50K baseline overhead (system prompt + CLAUDE.md + MCP schemas + tool definitions)
- The "distillation instruction" pattern (telling subagents to return concise summaries) is the key control lever
- Background agents (`run_in_background: true`) have identical token cost but don't block the parent

**Subagent model economics:**

| Model | Input $/M | Output $/M | Best for |
|---|---|---|---|
| Haiku | $1 | $5 | Search, exploration, file reading, pattern matching |
| Sonnet | $3 | $15 | Implementation, code review, moderate reasoning |
| Opus | $15 | $75 | Complex reasoning, architectural decisions |

- `model` parameter works reliably in agent definitions with aliases `sonnet`, `opus`, `haiku`, `inherit`
- `maxTurns` parameter limits agentic iterations — use to prevent runaway costs
- **Break-even rule:** Only spawn subagents when (a) output would be verbose and you want it isolated, (b) work is parallelizable, or (c) you need tool restrictions. For simple, targeted tasks, inline work is 5-10x cheaper.

**Session lifecycle budgeting:**
- Context rot is real: only **14-28% of loaded context is relevant** per interaction
- "Lost in the middle" effect compounds with session length — mid-context information gets >30% attention discount
- **Rotate proactively at 60-65% capacity**, not reactively at 75%+
- Use `/compact` at logical breakpoints with preservation hints
- Use `/clear` between unrelated tasks
- Handoff documents reduce context re-establishment from 10,000+ to <2,000 tokens
- Session forking (`--fork-session`) inherits parent history and KV cache prefix, but TTL may expire

**PreCompact → SessionStart pipeline (broken):**
- PreCompact hook is side-effects-only — cannot inject context directly
- Canonical pattern: PreCompact writes recovery file → SessionStart `compact` matcher reads and re-injects
- **Bug #15174:** SessionStart compact matcher stdout is silently dropped
- **Reliable alternative:** Write critical state to files (todo.md, log.md) and reference from CLAUDE.md, which always survives compaction

Sources:
- [Subagent docs](https://code.claude.com/docs/en/sub-agents)
- [Context rot research](https://vincentvandeth.nl/blog/context-rot-claude-code-automatic-rotation)
- [Handoff protocol](https://blackdoglabs.io/blog/claude-code-decoded-handoff-protocol)
- [SessionStart compact matcher bug](https://github.com/anthropics/claude-code/issues/15174)
- [Subagent cost analysis](https://dev.to/jungjaehoon/why-claude-code-subagents-waste-50k-tokens-per-turn-and-how-to-fix-it-41ma)

### 3. Tool Output Economics — The Cost Hierarchy

**Token cost by source (per occurrence, highest to lowest):**

| Source | Tokens | Controllable? | Mitigation |
|---|---|---|---|
| Subagent spawn (baseline) | ~50,000 | Partially | Haiku model, maxTurns, distillation |
| Read tool (large file) | up to 25,000 | Yes | PreToolUse limit injection |
| Read tool (line number overhead) | 1.7x raw | No | Anthropic would need to change format |
| Bash stdout (unbounded) | Variable | Yes | RTK + quiet flags + PreToolUse blocking |
| System reminders | ~500-2,000 each | Partially | PostToolUse stripping (additive) |
| Tool definitions (per session) | ~6,000-10,000 total | Yes | MCP Tool Search (already active, 85% reduction) |
| Always-loaded instructions | ~3,290 | Diminishing | Already lean |

**Per-tool definition costs:** ReadFile ~469, Grep ~300, Edit ~246, Task ~1,331, TodoWrite ~2,161 tokens.

**Key insight:** The Read tool's `cat -n` formatting adds 70% overhead (27,300 tokens for a file that would be 16,002 raw). This is baked into Claude Code and not controllable via hooks. The controllable lever is limiting HOW MUCH of the file is read via PreToolUse `limit` injection.

Sources:
- [Read tool line number overhead](https://github.com/anthropics/claude-code/issues/20223)
- [Tool definition token sizes](https://github.com/Piebald-AI/claude-code-system-prompts)
- [Read tool 25K cap](https://github.com/anthropics/claude-code/issues/15687)
- [Cost management docs](https://code.claude.com/docs/en/costs)

### 4. Frontier Techniques — MCP Ecosystem and New Tools

**MCP proxy/compression ecosystem:**

| Tool | What it does | Token savings |
|---|---|---|
| [mcproxy](https://github.com/team-attention/mcproxy) | Tool schema filtering — hide unused MCP tools | Variable |
| [token-optimizer-mcp](https://github.com/ooples/token-optimizer-mcp) | Caching + compression for MCP responses | Variable |
| [mcp-compressor](https://github.com/atlassian-labs/mcp-compressor) | Collapse tool counts / merge related tools | Variable |
| [toon-context-mcp](https://github.com/aj-geddes/toon-context-mcp) | TOON format wrapper for JSON responses | 40-60% |

**TOON (Token-Oriented Object Notation):** Strips JSON syntax overhead. 500-row dataset: 11,842 → 4,617 tokens (61% reduction). Purpose-built MCP server exists. Feature request for native Claude Code support: [issue #14374](https://github.com/anthropics/claude-code/issues/14374).

**`updatedMCPToolOutput`:** The only mechanism that truly replaces tool output (for MCP tools only). Officially documented in PostToolUse hooks. This is the correct field for intercepting Linear/Slack/Figma responses and compressing them before they enter context.

**Environment variables:**

| Variable | Effect | Default |
|---|---|---|
| `CLAUDE_CODE_AUTOCOMPACT_PCT_OVERRIDE` | When auto-compaction fires (1-100) | ~95% |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max tokens per API response | 32,000 |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Override file read token limit | — |
| `BASH_MAX_OUTPUT_LENGTH` | Max chars in bash outputs | — |

### 5. Prompt & Instruction Optimization

**Always-loaded context breakdown:**

| File | Chars | Tokens | Compressible? |
|---|---|---|---|
| CLAUDE.md | 5,890 | ~1,473 | Routes A-K structure is repeated; table is load-bearing |
| .claude/rules/ (7 files) | 4,200 | ~1,050 | Minimal — each rule is already concise |
| identity.md | 1,650 | ~413 | No — factual state |
| Global CLAUDE.md + RTK.md | 750 | ~188 | No — minimal |
| MEMORY.md | 1,450 | ~363 | No — operational patterns |
| **Total** | **13,940** | **~3,487** | |

At ~3.5K tokens, this is already in the efficient zone. The CLAUDE.md routes A-K expanded descriptions (~800 tokens) have repeated "Use when / Skill / Write / Journal / Update" structure that could be compressed into a denser format, saving ~300-400 tokens. But this is diminishing returns — the routing accuracy depends on the expanded descriptions.

**Effort-level tuning:** Already covered in the existing plan (effortLevel: medium, effort signals in agent files). Adaptive thinking on Opus 4.6 is the correct lever, not MAX_THINKING_TOKENS.

## Cross-cutting Patterns

### The Actionable Priority Stack

**Tier 1 — High impact, low effort (do now):**
1. **Build PreToolUse read-limiter hook** — Python, ~100 LOC, prevents largest controllable waste
2. **Enforce subagent model discipline** — Set `model: haiku` on search/explore agents, add `maxTurns` limits
3. **Session rotation discipline** — Rotate at 60-65% capacity, handoff documents for state transfer

**Tier 2 — Medium impact, medium effort (do next):**
4. **PostToolUse system-reminder stripping** — Strip `<system-reminder>` tags (additive but reduces noise)
5. **MCP output interception** — `updatedMCPToolOutput` for verbose Linear/Slack responses
6. **Verbose command blocking** — PreToolUse hook to block `npm install`, `pip install` without quiet flags

**Tier 3 — Interesting, higher effort (evaluate later):**
7. **TOON MCP server** — For JSON-heavy responses (40-60% savings)
8. **Read-compress structural skeleton** — Extract imports/classes/functions for files >300 lines
9. **MCP proxy layer** — mcproxy or token-optimizer-mcp for response caching/filtering
10. **Prompt cache optimization** — Batch related work, avoid mid-session model switches, stable CLAUDE.md

### What NOT to Optimize (diminishing returns)
- Always-loaded instructions (~3.5K tokens) — already lean
- Tool definitions — MCP Tool Search already handles this (85% reduction)
- CLAUDE.md routing table — load-bearing, compression risks routing accuracy

### Key Constraints Discovered
- PostToolUse `additionalContext` is additive, never replaces output → prevention (PreToolUse) beats compression (PostToolUse)
- PreCompact → SessionStart compact pipeline is bugged (#15174) → use file-based state + CLAUDE.md survival
- Read tool line-number formatting adds 70% overhead → not controllable, but limit injection reduces total
- Subagent 50K baseline is structural → mitigate with model choice and spawn discipline, not elimination

## Key Sources
### Codebase files
- `.claude/settings.json:33-135` — hook registrations
- `~/.claude/hooks/rtk-rewrite.sh:1-217` — RTK command rewriter
- `.claude/hooks/scope_guard.py` — write scope enforcement
- `CLAUDE.md` — routing and constraints
- `memory-bank/durable/identity.md` — project identity

### Community tools
- [claude-warden](https://github.com/johnzfitch/claude-warden) — comprehensive hook suite (read-compress, read-guard, system-reminder strip)
- [toon-context-mcp](https://github.com/aj-geddes/toon-context-mcp) — TOON format MCP server
- [mcproxy](https://github.com/team-attention/mcproxy) — MCP tool schema filtering
- [token-optimizer-mcp](https://github.com/ooples/token-optimizer-mcp) — MCP caching + compression
- [mcp-compressor](https://github.com/atlassian-labs/mcp-compressor) — tool count collapse
- [mvara-ai/precompact-hook](https://github.com/mvara-ai/precompact-hook) — PreCompact recovery pattern

### Claude Code documentation
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Subagent docs](https://code.claude.com/docs/en/sub-agents)
- [Cost management](https://code.claude.com/docs/en/costs)

### Research & analysis
- [Context rot research](https://vincentvandeth.nl/blog/context-rot-claude-code-automatic-rotation)
- [MCP Tool Search 85% reduction](https://medium.com/@joe.njenga/claude-code-just-cut-mcp-context-bloat-by-46-9-51k-tokens-down-to-8-5k-with-new-tool-search-ddf9e905f734)
- [Read tool 70% overhead from line numbers](https://github.com/anthropics/claude-code/issues/20223)
- [Subagent 50K baseline cost](https://dev.to/jungjaehoon/why-claude-code-subagents-waste-50k-tokens-per-turn-and-how-to-fix-it-41ma)
- [Prompt caching economics](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code)

### Bug references
- [#15174](https://github.com/anthropics/claude-code/issues/15174) — SessionStart compact matcher stdout dropped
- [#24788](https://github.com/anthropics/claude-code/issues/24788) — PostToolUse additionalContext not surfacing for MCP
- [#20223](https://github.com/anthropics/claude-code/issues/20223) — Read tool 70% token overhead from line numbers
- [#14374](https://github.com/anthropics/claude-code/issues/14374) — TOON support feature request

## Open Questions
- How much does the Read tool's `cat -n` overhead actually cost us per session? (Need session-level telemetry)
- When will bug #15174 (SessionStart compact matcher) be fixed? This would unlock the PreCompact → re-injection pipeline.
- Can `updatedMCPToolOutput` be used reliably for Linear/Slack output compression, or does bug #24788 block it?
- What's the actual quality impact of session rotation at 60% vs 80% capacity? (Need A/B comparison)
- Would a TOON MCP proxy for our Linear/Slack integrations save meaningful tokens, given we already have MCP Tool Search?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (all gaps closed by iteration 3)
- Manifest: `.claude/deep-research/2026-03-04-context-window-optimization-token.md`
