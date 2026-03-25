# Detrix Analysis — Architecture, Patterns, and Integration Opportunities

**Date:** 2026-03-04
**Source:** https://github.com/flashus/detrix
**Status:** Research complete

---

## What Detrix Is

Detrix is a Rust-based runtime observability daemon that lets AI agents observe running code in real-time — no code changes, no restarts, no pauses. It sits between an AI agent (Claude Code, Cursor) and a language-native debugger, using DAP logpoints to capture variable values at full execution speed.

**Key insight:** It's not a debugger. It's an **agent-to-debugger bridge** that makes runtime observation a first-class MCP tool surface.

---

## Architecture Deep Dive

### 13-Crate Clean Architecture

Five strict dependency tiers — application services never import infrastructure crates:

```
Interface → Application → Ports → Infrastructure → Domain
```

| Crate | Layer | Role |
|-------|-------|------|
| `detrix-core` | Domain | Pure entities: Metric, MetricEvent, Connection |
| `detrix-config` | Domain | TOML-loaded config structs |
| `detrix-ports` | Ports | All trait definitions (MetricRepository, DapAdapter, etc.) |
| `detrix-application` | Application | 14+ services: metric, connection, streaming, event capture, safety validation, DLQ recovery |
| `detrix-dap` | Infra | DAP protocol adapter. BaseAdapter<P: OutputParser> achieves ~80% code reuse across languages |
| `detrix-storage` | Infra | SQLite persistence with event batching |
| `detrix-lsp` | Infra | Optional deep purity analysis via language servers |
| `detrix-output` | Infra | GELF/Graylog structured log routing |
| `detrix-api` | Interface | gRPC (50061), REST (8090), WebSocket, MCP (stdio) |
| `detrix-cli` | Interface | CLI: daemon, metric, connection, event, group, validate, inspect, diff, mcp |
| `detrix-logging` | Support | GELF-compatible structured logging |
| `detrix-tui` | Support | Experimental terminal UI |
| `detrix-testing` | Test | Mock implementations of all port traits + E2E clients |

### Request Flow (MCP Bridge Mode)

```
Claude Code
  → JSON-RPC over stdio
detrix mcp (stdio server)
  → HTTP POST to daemon /mcp endpoint
Detrix Daemon
  → routes to Application Services
DAP Broker
  → DAP protocol
Language Debugger (debugpy / Delve / lldb-dap)
  → logpoint fires on target process
Event captured → SQLite → returned to Claude Code
```

### Observation Points vs Breakpoints

Traditional breakpoints pause execution. Detrix uses **DAP logpoints** — a breakpoint variant where `logMessage` is set to an expression. The debugger evaluates the expression and emits the result as an `OutputEvent`; execution continues immediately.

Wire format: `DETRICS:name={expr1}\x1F{expr2}\x1F...`

Go exception: Delve doesn't support function calls in logpoints, so Detrix falls back to breakpoint + REPL evaluate (~1ms pause).

### Client Init Sequence

`detrix.init()` embeds a lightweight control plane HTTP server in the app process:

1. Starts HTTP server on auto-assigned port
2. Registers metadata with daemon (name, PID, language, control URL)
3. Does NOT start debugger — remains SLEEPING (zero overhead)

`wake()` activates on demand: health check → start debugger → register connection.
`sleep()` deactivates: unregister → stop debugger.

### 29 MCP Tools (9 categories)

| Category | Tools | Notable |
|----------|-------|---------|
| Workflow | 3 | `observe()` (quick), `enable_from_diff()` (git diff → metrics), `find_variable()` (no line numbers needed) |
| Metrics | 5 | Full CRUD + enable/disable |
| Groups | 3 | Organize metrics into logical sets |
| Events | 3 | `query_metrics()`, `latest_event()`, stream |
| Connections | 4 | DAP connection lifecycle |
| Diagnostics | 2 | `validate_expression()`, `inspect_file()` (scope discovery) |
| Config | 4 | Runtime config management |
| System | 3 | `status()`, `health()`, `list_processes()` |
| Remote | 2 | `wake()` / `sleep()` remote debuggers |

### Three-Layer Safety

1. **Tree-sitter AST analysis** — scope-aware mutation detection (assignments, deletes, list/dict mutations)
2. **Expression validator** — function purity classification + sensitive variable blocking (password, api_key, token, secret)
3. **LSP purity analyzer** — deep call hierarchy analysis via pyright/gopls/rust-analyzer, cached

### Capture Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `stream` | Every hit (default) | Low-frequency paths |
| `sample` | Every Nth hit | High-frequency loops |
| `sample_interval` | Time-based periodic | Time-series observation |
| `first` | Single capture, self-disables | One-shot investigation |
| `throttle` | Rate-limited per second | Bursty paths |

All metrics support **TTL** — auto-disable after N seconds. Primary production safety mechanism.

### Skills/Agent Integration

Ships with `skills/detrix/SKILL.md` + `references/workflow.md` + `references/errors.md`. Co-located with the tool — designed from day one for AI agents.

Cardinal rule: "NEVER add print() or log.debug() statements. Use Detrix instead."

---

## Engineering Patterns Worth Learning From

### 1. BaseAdapter<P: OutputParser> — Generic + Trait Object

80% code reuse across Python/Go/Rust with ~100 lines per language. Only the variable parts (output parsing, message formatting, continuation behavior) are language-specific. The DAP connection management, logpoint CRUD, and event routing are fully shared.

**Our analog:** We have a similar opportunity in `cloud_api` — different cloud providers share backup/restore semantics but differ in API details. The BaseAdapter pattern with a trait for cloud-specific behavior would reduce duplication.

### 2. SLEEPING/AWAKE State Machine

Zero overhead at rest is a first-class design goal. The debugger doesn't run until an agent needs it. This is how production safety works — not "be careful" but "impossible to have overhead when idle."

**Our analog:** Our telemetry component already does this (`_ENABLED = False` by default, no-ops when unconfigured). Good pattern validation. Could extend to other expensive subsystems.

### 3. TTL as Production Safety Net

Every observation point auto-expires. Forgetting to clean up has bounded impact. This is better than process discipline.

**Our analog:** We don't have TTL on any of our debug/diagnostic state. The `capture_failed_events` decorator writes to S3 indefinitely. The harness tests don't self-clean. Worth adopting.

### 4. enable_from_diff() — Meet Developers Where They Are

If someone already wrote print statements while debugging, Detrix converts that work into non-invasive observation points. Zero friction adoption.

**Our analog:** We could build a similar pattern for our agent debugging — if an agent adds logging to investigate a bug, automatically convert those log points into telemetry observations.

### 5. Skill-First Agent Design

The tool ships with its own skill definition, workflow reference, and error reference. The agent knows exactly how to use the tool because the tool tells it.

**Our analog:** We already do this with our skills/ directory. Detrix validates our approach. Their `references/` subdirectory pattern (workflow.md, errors.md) is worth adopting — we could add reference docs to our existing skills.

### 6. DLQ Recovery for Events

Events that fail to persist go to a dead-letter queue with retry logic. Critical for Docker/remote environments.

**Our analog:** We already have DLQ patterns in our Valkey queue processor (batchItemFailures). But our `capture_failed_events` S3 writes don't retry — they swallow errors. Worth hardening.

### 7. Three-Tier Safety Validation

Progressive depth: fast structural check → semantic classification → deep call hierarchy. Each tier is optional for users who want speed over safety depth.

**Our analog:** Our import-linter contracts are a single tier. Could consider a progressive model: fast lint (ruff) → boundary check (import-linter) → deep semantic analysis (ty + custom rules).

---

## Integration Opportunities

### Tier 1 — Direct Adoption (Low effort, high value)

**A. Install Detrix as an MCP server for Claude Code debugging**

Detrix supports Python. Our Lambda handlers are Python. We could:
1. `pip install detrix` in our dev environment
2. Add `detrix.init(name="discover")` to handler entry points (behind env flag)
3. Configure Detrix MCP in `.claude/settings.json`
4. Claude Code can now observe running Lambda handlers in local dev

This gives us live variable inspection during debugging without print statements. Particularly valuable for the Valkey queue processor poison records blocker — we could observe the exact DynamoDB stream record shapes hitting the parser.

**B. Add references/ to our existing skills**

Adopt Detrix's pattern of co-locating workflow docs and error docs with skills:
```
.claude/skills/helm/
  SKILL.md
  references/
    workflow.md      # Step-by-step helm workflows
    errors.md        # Common failures + resolutions
```

Same for `/investigate`, `/resolve-linear-issue`, etc.

### Tier 2 — Pattern Adoption (Medium effort, medium value)

**C. Build an observability MCP server**

We have VictoriaMetrics + VictoriaLogs + VictoriaTraces + Grafana running locally. But Claude Code can't query them. An MCP server that exposes:
- `query_metrics(promql)` — query VictoriaMetrics
- `query_logs(logsql)` — query VictoriaLogs
- `query_traces(traceql)` — query VictoriaTraces
- `get_dashboard(name)` — fetch Grafana dashboard state

Would let agents self-diagnose telemetry issues. This is the "agent-to-telemetry feedback loop" gap we identified in the observability research.

**D. TTL patterns for debug state**

Add TTL to:
- S3 failed event captures (lifecycle rule)
- Harness test artifacts
- Any future diagnostic/debug state

**E. Adopt the OutputParser trait pattern for cloud providers**

Our `cloud_api` component talks to multiple clouds. The BaseAdapter<P: OutputParser> pattern — shared protocol handling with per-provider parsing — could reduce our cloud integration code.

### Tier 3 — Aspirational (High effort, high value)

**F. Agent semantic observability**

Detrix solves "observe running code." We need "observe running agents." The semantic-quality monitoring gap:
- Did the agent reason correctly at each step?
- Where did it silently fail?
- What was the actual token cost vs estimate?

This would combine Detrix-style non-invasive observation with our OTel infrastructure:
1. OTel spans for agent lifecycle (create, invoke, tool_call)
2. Semantic quality metrics (plan adherence, success rate per phase)
3. MCP server for agents to self-query their own execution history

This is the vision from our agent observability research, and Detrix's architecture shows how to build the tooling layer.

---

## What We Should NOT Do

- **Don't build our own Detrix.** It's MIT-licensed, actively maintained, and does one thing well. Use it.
- **Don't try to integrate Detrix with Lambda in production.** Debuggers + Lambda cold starts = bad idea. Keep it for local dev and Docker-based testing.
- **Don't over-invest in Rust tooling.** Our stack is Python. Detrix's Rust architecture is instructive but our tools should stay in Python/uv ecosystem.
- **Don't add Detrix as a hard dependency.** Keep it as an optional dev tool, exactly like our observability stack.

---

## Recommended Next Steps

1. **Try Detrix locally** — Install, add `detrix.init()` to discover handler, observe a test run
2. **Add references/ to helm skill** — Quick win, validates the pattern
3. **Scope an observability MCP server** — Brief/shape, add to backlog
4. **Use Detrix on the VQP poison records blocker** — Real debugging value immediately
