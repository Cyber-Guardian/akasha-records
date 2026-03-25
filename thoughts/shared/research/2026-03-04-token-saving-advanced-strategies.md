# Research: Advanced Token Saving Strategies for Claude Code

**Date:** 2026-03-04
**Context:** Follow-up research for `briefs/2026-03-04-token-saving-strategies.md` and `plans/2026-03-04-token-saving-strategies.md`

## Background

RTK covers CLI output filtering (60-90% savings). The current plan covers MCP pruning, .claudeignore, effortLevel, and agent effort signals. This research explores what else exists beyond those basics.

## Tier 1 — High-Impact, Directly Applicable

### claude-warden Hook Suite
**Source:** [github.com/johnzfitch/claude-warden](https://github.com/johnzfitch/claude-warden)

A comprehensive hook suite that addresses token waste RTK doesn't cover:

| Hook | Type | What It Does |
|------|------|--------------|
| `read-compress` | PostToolUse (Read) | Extracts structural signatures (imports, functions, classes) for files >300 lines in subagents, >500 lines in main. Replaces full reads with skeleton. |
| `post-tool-use` | PostToolUse | Strips `<system-reminder>` blocks. Compresses Task output >6KB. Truncates Bash >20KB to 10KB. Suppresses >500KB. Detects binary via NUL-byte scanning. |
| `read-guard` | PreToolUse (Read) | Blocks reads on `node_modules/`, `dist/`, `.min.js`. Enforces 2MB file size limit. |
| `pre-tool-use` | PreToolUse | Blocks verbose commands (`npm install`, `cargo build`, `pip`) without quiet flags. Blocks recursive grep/find without limits. |
| `permission-request` | PermissionRequest | Auto-denies `rm -rf /`, `mkfs`, `curl | bash`. Auto-allows safe read-only ops. |
| `statusline.sh` | Composite | Real-time display: model, context %, tokens, cache stats, subagent count. |

Uses Anthropic token counting API asynchronously for exact savings measurement. Logs events to `~/.claude/.statusline/events.jsonl`.

### System-Reminder Injection Problem
**Source:** [github.com/anthropics/claude-code/issues/4464](https://github.com/anthropics/claude-code/issues/4464)

Hidden cost most developers don't know about. Claude Code injects full file diffs as `<system-reminder>` tags when files are modified externally (linters, build tools, hooks). A 1,534-line JSON file injected twice silently consumes enormous context.

**Workarounds:**
- PostToolUse hook to strip `<system-reminder>` blocks (claude-warden does this)
- Add generated/large files to `.claudeignore`
- As of Claude Code v2.1.63, ~40 system reminders exist

### Python Traceback Compactor
**Source:** [github.com/tarekziade/claude-tools](https://github.com/tarekziade/claude-tools)

- UserPromptSubmit + PostToolUse (Bash) hooks
- Intelligent frame scoring: project code prioritized over stdlib
- Fingerprinting for error deduplication — won't reprocess identical tracebacks
- ~80% reduction: 250+ tokens → ~40 tokens
- Configurable `--max-frames N`, `--project-root` for relevance scoring
- Zero dependencies, pure Python

## Tier 2 — Medium Effort, High Potential

### PreToolUse Read `limit` Auto-Injection
No published implementation exists. The hooks API supports it via `updatedInput`:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "file_path": "/some/file.py",
      "limit": 200
    }
  }
}
```

Could auto-add `limit` for files known to be large, or enforce a default max unless explicitly overridden. Novel opportunity.

### Prompt Cache Optimization
**Source:** [dev.to/kitaekatt/mastering-cache-hits-in-claude-code-5648](https://dev.to/kitaekatt/mastering-cache-hits-in-claude-code-5648)

Cache reads cost 10% of normal input price (10x reduction). Key architectural insights:

- **Cache invalidation hierarchy:** Tool definitions → System prompt → CLAUDE.md → Conversation history. Changing anything invalidates everything below it.
- **Keep CLAUDE.md and tool definitions stable within a session** — don't modify mid-session.
- **Batch related work into single sessions** rather than fragmenting. Loading 100K once + 5 questions = 1.65x cost vs 6.25x across separate sessions.
- **Fork sessions for parallel investigation** — base session pays cache write once, forks get cache hits.
- **Anti-patterns:** `/rewind` breaks prefix match. Switching models mid-session invalidates cache. Speculative preloading wastes cache-read tokens on irrelevant content.
- Pro/Max subscribers: 1-hour TTL vs 5-minute TTL — dramatically different economics.

### SWE-Pruner: Task-Aware File Read Compression
**Source:** [arxiv.org/abs/2601.16746](https://arxiv.org/abs/2601.16746) | [github.com/Ayanami1314/swe-pruner](https://github.com/Ayanami1314/swe-pruner)

- Middleware between coding agent and environment — intercepts file reads
- Agent emits a "Goal Hint" before reading; a 0.6B encoder scores each line for relevance
- Prunes low-relevance lines before agent sees them
- First-token latency <100ms even at 8K tokens
- 23–54% token reduction on SWE-Bench with <1% task success degradation
- 18–26% fewer interaction rounds
- Would need to run as a local sidecar process

### Undocumented Environment Variables
**Source:** [code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

| Variable | Effect | Default |
|---|---|---|
| `CLAUDE_CODE_AUTOCOMPACT_PCT_OVERRIDE` | When auto-compaction fires (1-100) | ~95% |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max tokens per single API response | 32,000 |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Override file read token limit | — |
| `BASH_MAX_OUTPUT_LENGTH` | Max chars in bash outputs (middle-truncated) | — |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Disable 1M context variants | off |

Key insight: compaction buffer = 33K tokens (16.5% of 200K). Auto-compaction fires at ~83.5% usage → ~167K usable tokens.

## Tier 3 — Interesting, Heavier Lift

### Aider-Style Repomap
**Source:** [aider.chat/docs/repomap.html](https://aider.chat/docs/repomap.html)

- Tree-sitter parses all files → extract classes, functions, types, signatures
- NetworkX graph: files as nodes, dependencies as edges
- PageRank with personalization toward active files
- Top-ranked identifiers fit within ~1K token budget
- Entire repo skeleton for ~1K tokens instead of reading 50 files

Could be approximated in Claude Code via:
- PreToolUse hook substituting structural summaries for large reads
- UserPromptSubmit hook injecting pre-generated repo skeleton
- Slash command that generates and caches a repo map file

### Context Rot Research
**Source:** [research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)

All 18 tested frontier models (2025) degrade with longer inputs, even when context window isn't full. "Lost in the Middle" effect: models attend strongest to start and end of context; mid-context information gets discounted (>30% drop).

**Implication:** Aggressive context pruning improves both cost AND quality. Smaller, well-curated context outperforms larger noisy context.

### TOON (Token-Oriented Object Notation)
**Source:** [tensorlake.ai/blog-posts/toon-vs-json](https://www.tensorlake.ai/blog-posts/toon-vs-json)

30-60% token reduction on structured data by stripping JSON syntax overhead. 500-row JSON dataset: 11,842 → 4,617 tokens (61% reduction). Relevant for tool results returning large JSON payloads.

### CLAUDE.md Lazy Loading
**Source:** [gist.github.com/johnlindquist/849b813e76039a908d962b2f0923dc9a](https://gist.github.com/johnlindquist/849b813e76039a908d962b2f0923dc9a)

54% initial token reduction via:
- Trigger table compression (verbose docs → minimal tables)
- Content consolidation (merge redundant docs)
- Skill compression scripting (12KB → ~800 bytes, full docs load on-demand)
- Audit hooks for redundant context injection

Not applicable to our setup (CLAUDE.md is already lean at ~3.5K tokens), but relevant if it grows.

### Observation Masking vs LLM Summarization
**Source:** [blog.jetbrains.com/research/2025/12/efficient-context-management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

JetBrains research comparing two approaches:
- **Observation masking:** Replace older tool outputs with placeholders. Simple, fast, no model call overhead. 52% cheaper.
- **LLM summarization:** Compress older turns into prose summaries. Caused agents to run ~15% more steps (summaries obscure stop signals).
- **Verdict:** Masking as first line of defense; summarization only where necessary.

### Embedding-Based Code Retrieval
**Sources:** Continue.dev uses `@codebase` with vector DB + LLM reranking. Qodo-Embed-1 (1.5B params) outperforms OpenAI text-embedding-3-large on code retrieval. Voyage-code-3 supports Matryoshka dimension truncation.

Relevant if we ever build a RAG layer for code context, but overkill for current workflows.

## Monitoring & Analytics Tools

| Tool | What It Does |
|------|-------------|
| [ccusage](https://claudelog.com/claude-code-mcps/cc-usage/) | CLI analyzing local Claude Code logs. Daily/monthly usage, burn rate. |
| [Claude-Code-Usage-Monitor](https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor) | Real-time terminal dashboard with ML-based rate limit predictions. |
| [claude-warden statusline](https://github.com/johnzfitch/claude-warden) | Real-time filtering decisions logged to events.jsonl with savings estimates. |

## Key Technical Details

### PostToolUse Cannot Replace Built-In Tool Output
**Source:** [github.com/anthropics/claude-code/issues/18653](https://github.com/anthropics/claude-code/issues/18653)

`updatedMCPToolOutput` only works for MCP tools. For built-in tools (Read, Bash), PostToolUse can only append `additionalContext`. True output truncation must happen at PreToolUse (blocking/limiting) or via injecting compressed summaries as additional context.

### 17 Hook Events Available
SessionStart, UserPromptSubmit, PreToolUse, PermissionRequest, PostToolUse, PostToolUseFailure, Notification, SubagentStart, SubagentStop, Stop, TeammateIdle, TaskCompleted, ConfigChange, WorktreeCreate, WorktreeRemove, PreCompact, SessionEnd.

### PostToolUse `additionalContext` Field
Inject text into Claude's context after a tool runs — use for summaries, warnings, or compressed results:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "[Truncated: file was 5000 lines, showing structural summary only]"
  }
}
```

## Priority Assessment for Follow-Up Work

| Candidate | Effort | Impact | Notes |
|---|---|---|---|
| Cherry-pick claude-warden hooks (read-compress, system-reminder strip, read-guard) | Low-Medium | High | Directly additive to RTK |
| Traceback compactor | Low | Medium | Drop-in, pure Python, zero deps |
| Read `limit` auto-injection hook | Medium | Medium | Novel, no one's built it yet |
| Prompt cache hygiene rules | Low | Medium | Just behavioral discipline, no code |
| SWE-Pruner sidecar | High | High | Requires running a 0.6B model locally |
| Repomap generation | High | High | Tree-sitter + PageRank pipeline |

## References

- [claude-warden](https://github.com/johnzfitch/claude-warden)
- [claude-tools (traceback compactor)](https://github.com/tarekziade/claude-tools)
- [SWE-Pruner paper](https://arxiv.org/abs/2601.16746)
- [Aider repomap](https://aider.chat/docs/repomap.html)
- [Context rot research](https://research.trychroma.com/context-rot)
- [Prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [System-reminder issue #4464](https://github.com/anthropics/claude-code/issues/4464)
- [Tool result transform request #18653](https://github.com/anthropics/claude-code/issues/18653)
- [ACON context compression](https://arxiv.org/html/2510.00615v1)
- [JetBrains context management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [Factory.ai compression evaluation](https://factory.ai/news/evaluating-compression)
- [LLMLingua](https://github.com/microsoft/LLMLingua)
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [awesome-claude-code-toolkit](https://github.com/rohitg00/awesome-claude-code-toolkit)
- [everything-claude-code hooks](https://github.com/affaan-m/everything-claude-code/blob/main/hooks/hooks.json)
- [Cache hits in Claude Code](https://dev.to/kitaekatt/mastering-cache-hits-in-claude-code-5648)
- [TOON vs JSON](https://www.tensorlake.ai/blog-posts/toon-vs-json)
- [CLAUDE.md 54% reduction gist](https://gist.github.com/johnlindquist/849b813e76039a908d962b2f0923dc9a)
