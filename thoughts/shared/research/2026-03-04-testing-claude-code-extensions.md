---
date: 2026-03-04T13:15:00-05:00
researcher: claude
git_commit: eec3a0c
branch: worktree-memory-v2
repository: filescience
topic: "Testing and Observability for Claude Code Extensions"
tags: [research, claude-code, hooks, testing, extensions, observability, agent-testing]
status: complete
last_updated: 2026-03-04
last_updated_by: claude
last_updated_note: "Added follow-up: workaround strategies and agent observability framing"
---

# Research: Testing Claude Code Extensions

**Date**: 2026-03-04T13:15:00-05:00
**Researcher**: claude
**Git Commit**: eec3a0c
**Branch**: worktree-memory-v2

## Research Question

How can we effectively test Claude Code extensions (hooks, skills, rules, session startup, settings)? What testing infrastructure exists, what's testable programmatically vs manually, and what gaps exist?

## Summary

There is **no official testing framework** from Anthropic for Claude Code extensions. Testing falls into three tiers: (1) **unit tests** for hook logic via stdin JSON injection + `unittest.mock`, (2) **subprocess integration tests** piping JSON to hook scripts, and (3) **end-to-end tests** via `claude -p` subprocess — which has a critical blocker when run from within Claude Code due to the `CLAUDECODE=1` env var. Skills, rules, and agent definitions are markdown files with no programmatic test path — they can only be validated by running a real Claude session.

## Detailed Findings

### Extension Type Taxonomy

| Type | Format | Count | Testable Programmatically? |
|------|--------|-------|---------------------------|
| Hooks (command) | Python/Bash scripts | 9 scripts | Yes — stdin JSON piping |
| Hooks (agent/prompt) | Inline in settings.json | 1 (Stop hook) | No — requires Claude runtime |
| Skills | Markdown (SKILL.md) | 5 | No — prompt template only |
| Commands | Markdown (.md) | 15 | No — prompt template only |
| Rules | Markdown (.md) | 7 | No — injected into context |
| Agents | Markdown (.md) | 10 | No — agent definitions only |
| Settings | JSON | 2 | Schema-validatable only |
| Reference docs | Markdown (.md) | 4 | No — static reference |

### Tier 1: Unit Testing Hooks (What We Have)

The repo has one test file covering hooks: `test/scripts/test_scope_guard.py` (496 lines). It uses a proven pattern:

**Import pattern** — hooks live outside the Python path, so use `importlib`:
```python
import importlib.util
from pathlib import Path

HOOKS_DIR = Path(__file__).resolve().parent.parent.parent / ".claude" / "hooks"

def _import_hook(module_name, file_name):
    spec = importlib.util.spec_from_file_location(module_name, HOOKS_DIR / file_name)
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    return mod
```

**Stdin simulation** — patch `sys.stdin`, env vars, and `builtins.print`:
```python
from io import StringIO
from unittest.mock import patch

def _run_hook_main(stdin_data, project_dir):
    with (
        patch.object(sys, "stdin", StringIO(stdin_data)),
        patch.dict("os.environ", {"CLAUDE_PROJECT_DIR": str(project_dir)}),
        patch("builtins.print") as mock_print,
    ):
        hook_module.main()
    return json.loads(mock_print.call_args[0][0])
```

**Fixture factory** — `tmp_path` as isolated project root:
```python
def _make_scope(tmp_path, allowed):
    scope_dir = tmp_path / ".claude"
    scope_dir.mkdir(exist_ok=True)
    (scope_dir / "scope.json").write_text(json.dumps({"allowed_paths": allowed}))
```

**Coverage**: Only `scope_guard.py` and `scope_guard_bash.py` have tests. The remaining 7 hooks have zero test coverage.

### Tier 2: Subprocess Integration Testing

Hooks can be tested as subprocesses by piping JSON to stdin:

```bash
echo '{"tool_name":"Write","tool_input":{"file_path":".env"}}' | python3 .claude/hooks/scope_guard.py
echo $?  # 0 = allow, 2 = block
```

**pytest variant**:
```python
result = subprocess.run(
    ["python3", ".claude/hooks/scope_guard.py"],
    input=json.dumps({"tool_input": {"file_path": ".env"}}),
    capture_output=True, text=True,
    env={**os.environ, "CLAUDE_PROJECT_DIR": str(tmp_path)}
)
assert result.returncode == 0
assert json.loads(result.stdout) == {}
```

Shell hooks (`durable_memory_size_checker.sh`, `task_completed_gate.sh`) can only be tested this way — no importable functions.

### Tier 3: End-to-End Testing (The Gap)

**`claude -p` flags for automated testing**:

| Flag | Purpose |
|------|---------|
| `-p "query"` / `--print` | Non-interactive: run, print, exit |
| `--max-turns N` | Bound agentic turns |
| `--output-format json` | Structured output |
| `--dangerously-skip-permissions` | Skip permission prompts |
| `--settings ./test-settings.json` | Test-specific settings |
| `--no-session-persistence` | Don't persist session |
| `--allowedTools "Read,Bash"` | Pre-approve tools |
| `--init-only` | Run initialization hooks and exit |
| `--debug "hooks"` | Verbose hook execution |
| `--max-budget-usd 0.50` | Cost cap |

**The CLAUDECODE=1 blocker**: Claude Code sets `CLAUDECODE=1` in the environment. Any subprocess `claude` invocation from within a Claude session fails:

```
Error: Claude Code cannot be launched inside another Claude Code session.
```

**Workaround**: `CLAUDECODE= claude -p "..."` bypasses the check, but in practice produces empty output (observed in this session). The Agent SDK has the same issue (GitHub issue #573). A potential fix (PR #594) filters `CLAUDECODE` from the child environment.

**Additional limitation**: `PermissionRequest` hooks don't fire in `-p` mode. The `--max-turns` flag exits non-zero at the limit, complicating assertions.

### What Can't Be Tested Programmatically

1. **Skills and commands** — Markdown prompt templates. The only "test" is running a real Claude session and invoking `/skill-name`. No way to validate that the prompt produces correct behavior without LLM execution.

2. **Rules** — Markdown files injected into Claude's system prompt. No programmatic validation that Claude follows them. Could lint for structural correctness (YAML frontmatter, expected sections) but not behavioral correctness.

3. **Agent definitions** — Markdown agent prompts. Same as skills — require real execution.

4. **`agent`/`prompt` type hooks** (e.g., the Stop hook) — These invoke an LLM. Can't be unit tested without mocking the entire Claude runtime.

5. **Settings schema** — No official JSON Schema for `.claude/settings.json`. Validation is "does Claude Code start without errors."

6. **SessionStart context injection** — The `additionalContext` from `session_start.py` is injected into Claude's conversation. Verifying it appears correctly requires a real session.

7. **Inter-extension interactions** — Rules referencing skills, hooks blocking tool calls that skills depend on, agent definitions that invoke skills. The composition layer is untestable in isolation.

### Hook Input/Output Protocol Reference

**Stdin JSON** (common fields for all events):
```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {"command": "npm test"}
}
```

**Output protocol**:
- Exit `0` + `{}` → allow
- Exit `0` + `{"decision":"block","reason":"..."}` → block (PostToolUse/Stop)
- Exit `2` + stderr message → block (TaskCompleted, PreToolUse)
- Exit `0` + `{"hookSpecificOutput":{"hookEventName":"...","additionalContext":"..."}}` → inject context

### Agent SDK MockTransport Pattern

The Agent SDK's own test suite (not a public API) uses `MockTransport` for testing hook callbacks:

```python
class MockTransport(Transport):
    def __init__(self):
        self.written_messages = []
        self.messages_to_read = []
    async def write(self, data): self.written_messages.append(data)
    def read_messages(self):
        async def _read():
            for msg in self.messages_to_read: yield msg
        return _read()
```

Hook callbacks in the SDK are pure async functions — testable by constructing input dicts directly. But `MockTransport` is not exported as a public API.

### Existing Test Coverage Map

| Hook | Pure Functions | Unit Tests | Integration Tests |
|------|---------------|------------|-------------------|
| `scope_guard.py` | 3 (`load_scope`, `resolve_relative_path`, `is_path_allowed`) | Full | Full |
| `scope_guard_bash.py` | 1 (`extract_write_targets`) + 3 shared | Full | Full |
| `session_start.py` | 0 (all in `main`) | None | None |
| `research_brief.py` | 4 (`_parse_frontmatter`, `_extract_section`, etc.) | None | None |
| `durable_memory_size_checker.sh` | 0 (linear bash) | N/A | None |
| `task_completed_gate.sh` | 0 (linear bash) | N/A | None |
| `validators/ruff.py` | 0 | None | None |
| `validators/ty.py` | 0 | None | None |
| `validators/import_linter.py` | 1 (`_extract_broken_section`) | None | None |

## Code References

- `test/scripts/test_scope_guard.py` — Only hook test file (496 lines, 22 test cases)
- `.claude/hooks/session_start.py` — SessionStart hook, untested
- `.claude/hooks/scope_guard.py` — PreToolUse Edit/Write guard, fully tested
- `.claude/hooks/scope_guard_bash.py` — PreToolUse Bash guard, fully tested
- `.claude/hooks/research_brief.py` — PostToolUse brief generator, untested
- `.claude/hooks/validators/ruff.py` — PostToolUse ruff validator, untested
- `.claude/hooks/validators/ty.py` — PostToolUse ty validator, untested
- `.claude/hooks/validators/import_linter.py` — PostToolUse import-linter validator, untested
- `.claude/hooks/durable_memory_size_checker.sh` — PostToolUse size checker, untested
- `.claude/hooks/task_completed_gate.sh` — TaskCompleted gate, untested
- `.claude/settings.json:33-125` — Hook registration configuration
- `.claude/skills/extension-authoring/SKILL.md` — Extension authoring reference

## Architecture Documentation

### Testing Tiers (as they exist today)

```
Tier 1: Unit tests (pytest + mock)
  scope_guard.py ............ COVERED
  scope_guard_bash.py ....... COVERED
  (7 other hooks) ........... NOT COVERED

Tier 2: Subprocess integration (pipe JSON to stdin)
  All command hooks ......... POSSIBLE but not implemented
  Shell hooks ............... POSSIBLE but not implemented

Tier 3: End-to-end (claude -p subprocess)
  Session startup flow ...... BLOCKED (CLAUDECODE=1 / empty output)
  Skill invocation .......... BLOCKED (same)
  Rule enforcement .......... BLOCKED (same)
  Hook composition .......... BLOCKED (same)

Tier 4: Manual verification
  Fresh session behavior .... MANUAL ONLY
  Skill routing ............. MANUAL ONLY
  Rule compliance ........... MANUAL ONLY
  Agent delegation .......... MANUAL ONLY
```

### The Testability Spectrum

```
Fully testable          Partially testable       Manual only
(stdin/stdout)          (subprocess)             (requires Claude runtime)

scope_guard.py    ──►   session_start.py    ──►  Skills (.md)
scope_guard_bash  ──►   research_brief.py   ──►  Rules (.md)
ruff.py           ──►   size_checker.sh     ──►  Agents (.md)
ty.py             ──►   task_gate.sh        ──►  Stop hook (agent type)
import_linter.py  ──►                       ──►  Settings composition
                                             ──►  SessionStart context
                                             ──►  Inter-extension interactions
```

## Historical Context

- `memory-bank/thoughts/shared/research/2026-02-05-claude-hooks-validators.md` — Earlier research documenting the hook stdin shape and decision format
- The scope guard tests were written as part of the scope guard implementation and represent the only tested extension pattern in the repo

## Open Questions

1. **`claude -p` empty output from nested sessions**: Even with `CLAUDECODE=` override, subprocess invocations produce empty output. Is this a hook consuming turns, stdout buffering, or a deeper runtime issue? The `--init-only` flag (runs init hooks and exits) could help isolate.

2. **Agent SDK MockTransport**: Should we copy the SDK's internal `MockTransport` pattern for testing agent-type hooks, or wait for it to become a public API?

3. **Snapshot testing for context injection**: Could we snapshot the `additionalContext` output from `session_start.py` and regression-test it without needing a real Claude session?

4. **Schema validation for settings.json**: No official JSON Schema exists. Could we derive one from the docs and validate our settings file in CI?

5. **LLM-as-judge for skills/rules**: Could we use a cheaper model (Haiku) to evaluate whether skill prompts and rules produce expected behavior, using recorded transcripts as fixtures?

---

## Follow-up Research: Workaround Strategies and Agent Observability

### The Broader Framing

Testing Claude Code extensions and agent observability are two sides of the same coin. The "manual only" tier exists because we lack visibility into what Claude does with our extensions at runtime. If we can observe agent behavior — what context was loaded, which skills were invoked, what routing decisions were made — we can validate extensions without needing a dedicated test framework.

The question shifts from "how do we unit test a markdown prompt" to "how do we observe and assert on agent behavior?"

### Strategy 1: Cross-Reference Linting (Zero LLM Cost)

A static linter that validates the structural integrity of the extension graph. This catches the class of bugs where files reference things that don't exist — the most common failure mode after refactoring.

**What it validates:**
- Every `/command` in CLAUDE.md routing table maps to a `.claude/commands/<name>.md` file
- Every subagent type in commands (e.g., `"plan-implementer"`) maps to `.claude/agents/<name>.md`
- Every hook script path in `settings.json` resolves to an existing, executable file
- Every file path in `memory-rules.md` and reference docs exists on disk
- Startup read paths in CLAUDE.md exist
- `plansDirectory` in settings.json exists

**What was found by the cross-reference analysis:**
- CLAUDE.md routing table references `/debug` but only `investigate.md` exists in commands/ (potential mismatch)
- All 10 hook script paths in settings.json resolve correctly
- All startup read paths exist
- All `.claude/reference/*.md` files exist

**Implementation**: Python script, ~100 lines, runs in CI. Parses markdown for backtick-quoted paths and `/slash-commands`, resolves against disk. Could be a PreToolUse hook that runs on Edit/Write of any `.claude/` file.

### Strategy 2: SessionStart Output Snapshot Testing (Zero LLM Cost)

The `session_start.py` hook builds an `additionalContext` string that gets injected into every Claude session. This is the primary mechanism for loading dynamic state. We can snapshot-test its output.

**What's testable:**
- Given `source="startup"` and a mocked git log, the output contains exactly: session ID, session directory, git log, and journal conventions
- Given `source="compact"`, the output additionally contains todo.md and log.md contents
- The output never contains references to deleted files (queue.md, blockers.md after v2 migration)
- Property: `session_id` from input always appears verbatim in output lines 1 and 2

**Implementation**: pytest tests using the existing `_import_hook` + stdin mock pattern from `test_scope_guard.py`. Mock `subprocess.run` for git log, use `tmp_path` for filesystem isolation.

### Strategy 3: CI-Based E2E Smoke Tests (LLM Cost, Runs Outside Claude)

Run `claude -p` in GitHub Actions — outside any Claude session, so no `CLAUDECODE=1` blocker.

**Pattern:**
```yaml
- name: Smoke test - session startup
  run: |
    claude -p "What files are listed in your startup reads?" \
      --max-turns 3 \
      --max-budget-usd 0.50 \
      --output-format json \
      --no-session-persistence \
      --dangerously-skip-permissions \
    | jq -r '.result' > /tmp/startup-test.txt

    # Assert identity.md was loaded
    grep -q "identity.md" /tmp/startup-test.txt
    # Assert queue.md is NOT referenced
    ! grep -q "queue.md" /tmp/startup-test.txt

- name: Smoke test - routing table coherence
  run: |
    claude -p "List all slash commands mentioned in CLAUDE.md routing table" \
      --max-turns 2 \
      --max-budget-usd 0.30 \
      --output-format json \
      --json-schema '{"type":"object","properties":{"commands":{"type":"array","items":{"type":"string"}}},"required":["commands"]}' \
      --no-session-persistence \
      --dangerously-skip-permissions \
    | jq '.structured_output.commands[]' | sort > /tmp/routed-commands.txt

    # Assert all routed commands have corresponding files
    for cmd in $(cat /tmp/routed-commands.txt); do
      test -f ".claude/commands/${cmd}.md" || echo "MISSING: $cmd"
    done
```

**Cost**: ~$0.50-2.00 per CI run. Run on `push to main` only, not every PR.

**Key flags:**
- `--no-session-persistence` — clean state, no stale sessions
- `--dangerously-skip-permissions` — no permission prompts in CI
- `--json-schema` — structured output for assertions
- `--max-budget-usd` — hard cost cap

### Strategy 4: Transcript-Based Post-Hoc Validation (Zero Additional LLM Cost)

After any real Claude session, parse the JSONL transcript to validate extension behavior.

**Transcript location**: `~/.claude/projects/<path>/<session-id>.jsonl`
**Subagent transcripts**: `~/.claude/projects/<path>/agent-<shortId>.jsonl`

**JSONL fields useful for validation:**
- `type: "assistant"` messages with `content[].type: "tool_use"` — what tools were called
- `tool_use.name` — which tool (Read, Write, Bash, Skill, Agent)
- `tool_use.input` — arguments passed
- `isSidechain: true` — marks subagent messages
- `agentId` — identifies which subagent

**Validation script pattern:**
```python
import json
from pathlib import Path

def validate_session(transcript_path: Path, assertions: list[callable]):
    messages = [json.loads(line) for line in transcript_path.read_text().splitlines() if line.strip()]

    tool_calls = []
    for msg in messages:
        if msg.get("type") == "assistant":
            for block in msg.get("message", {}).get("content", []):
                if block.get("type") == "tool_use":
                    tool_calls.append(block)

    for assertion in assertions:
        assertion(messages, tool_calls)

# Example assertions
def assert_read_identity_first(messages, tool_calls):
    reads = [t for t in tool_calls if t["name"] == "Read"]
    assert any("identity.md" in str(t.get("input", {})) for t in reads[:3]), \
        "identity.md should be read in first 3 tool calls"

def assert_no_queue_references(messages, tool_calls):
    for t in tool_calls:
        assert "queue.md" not in str(t.get("input", {})), \
            f"Tool call references deleted queue.md: {t}"
```

**When to run**: As a `Stop` hook or as a post-session CI step. The `transcript_path` is available in every hook's stdin JSON.

### Strategy 5: Hook-Based Runtime Observability

Use hooks themselves as instrumentation points to observe agent behavior in real time.

**Architecture** (based on [disler's reference implementation](https://github.com/disler/claude-code-hooks-multi-agent-observability)):
```
PostToolUse hook → HTTP POST → local SQLite → WebSocket → dashboard
SubagentStop hook → same pipeline
Stop hook → session summary
```

**What hooks can observe today:**

| Hook Event | Observable Data | Limitation |
|-----------|----------------|------------|
| SessionStart | Session ID, source, git state | No tool calls yet |
| PreToolUse | Tool name, input args | Can block but can't attribute to subagent |
| PostToolUse | Tool name, input, response | Same — no `agent_id` field |
| SubagentStop | Stop reason | No `SubagentStart` event exists |
| Stop | Full session end | Can read `transcript_path` for post-hoc |

**The critical gap**: No `agent_id` in hook events (GitHub issue #14859, open, no Anthropic response). You can see every tool call but can't tell which agent made it. Post-hoc, you can reconstruct from JSONL `isSidechain` + `agentId` fields.

### Strategy 6: LLM-as-Judge for Behavioral Validation

For the truly "manual only" tier (skills, rules, agent definitions), use a cheaper model to evaluate transcripts.

**Framework**: promptfoo is the best fit — YAML-declarative, CI-friendly, supports `llm-rubric` assertions.

**Pattern:**
```yaml
# .claude/tests/skill-routing.yaml
prompts:
  - file://CLAUDE.md  # the routing table IS the prompt

tests:
  - vars:
      user_message: "debug this error in the auth module"
    assert:
      - type: llm-rubric
        value: "The response should invoke the /debug or /investigate skill"
      - type: not-contains
        value: "/create_plan"

  - vars:
      user_message: "ENG-2095"
    assert:
      - type: llm-rubric
        value: "The response should invoke /resolve-linear-issue"
      - type: contains
        value: "ENG-2095"
```

**Cost**: ~$0.01-0.05 per test case with Haiku. A suite of 20 routing tests costs ~$0.50.

**What this validates:**
- CLAUDE.md routing table directs to correct skills
- Skills don't reference deleted files/commands
- Rules produce expected behavioral constraints
- Agent definitions invoke expected subagent types

### Strategy 7: OpenTelemetry Bridge (Future)

The OTel GenAI semantic conventions are experimental but provide a standard for agent tracing. Building a hook-to-OTel bridge would connect Claude Code to Langfuse/Jaeger/Grafana.

**OTel GenAI agent span attributes (experimental):**
- `gen_ai.agent.id`, `gen_ai.agent.name` — agent identification
- `gen_ai.tool.name`, `gen_ai.tool.call.id` — tool call tracing
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` — token tracking

**Not viable today** because:
1. No `agent_id` in hook events (can't attribute spans to agents)
2. No `costUSD` per message in JSONL transcripts
3. OTel conventions are still experimental

**Viable when**: GitHub issue #14859 is resolved (agent_id in hooks) and OTel GenAI conventions stabilize.

### Strategy Comparison

| Strategy | Cost | Catches | Runs In | Effort |
|----------|------|---------|---------|--------|
| 1. Cross-ref linting | $0 | Broken references, missing files | CI (every PR) | Low (~100 LOC) |
| 2. SessionStart snapshot | $0 | Context injection regressions | CI (every PR) | Low (~50 LOC) |
| 3. CI smoke tests | ~$1/run | Startup flow, routing, skill invocation | CI (push to main) | Medium |
| 4. Transcript validation | $0 | Post-hoc behavioral regression | Stop hook or CI | Medium |
| 5. Hook observability | $0 | Real-time tool call monitoring | Always-on | Medium-High |
| 6. LLM-as-judge | ~$0.50/suite | Behavioral correctness of prompts | CI (weekly) | Medium |
| 7. OTel bridge | $0 | Full distributed tracing | Future | High |

### Recommended Adoption Order

1. **Cross-reference linter** (Strategy 1) — highest ROI, catches the most common failure class (broken references after refactoring), zero cost, runs every PR
2. **SessionStart snapshot tests** (Strategy 2) — validates the most critical hook, uses existing test patterns, zero cost
3. **Transcript validation** (Strategy 4) — can be added as a Stop hook, validates real session behavior post-hoc
4. **CI smoke tests** (Strategy 3) — validates end-to-end on push to main, ~$1/run
5. **LLM-as-judge** (Strategy 6) — weekly CI job for behavioral regression, ~$0.50/suite
6. **Hook observability** (Strategy 5) — always-on monitoring, requires local server setup
7. **OTel bridge** (Strategy 7) — blocked on upstream issues, revisit when #14859 resolves

### Relation to Existing Observability Research

This research builds on `memory-bank/thoughts/shared/research/2026-03-03-agent-observability.md` which documented agent failure modes and observability strategies at a conceptual level. The key insight from that research — agents fail "silently, probabilistically, and at transition points" — directly informs why extension testing matters: the extensions ARE the transition points (SessionStart, routing decisions, hook gates).

The FS-1 agent harness plan (`memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`) with progressive journaling is a complementary approach — it addresses observability at the harness level (Helm), while this research addresses observability at the extension level (hooks, skills, rules).

### Sources

- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Run Claude Code Programmatically](https://code.claude.com/docs/en/headless)
- [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Agent SDK Cost Tracking](https://platform.claude.com/docs/en/agent-sdk/cost-tracking)
- [Agent SDK Hooks Reference](https://platform.claude.com/docs/en/agent-sdk/hooks)
- [GitHub Issue #14859 — Agent Hierarchy in Hooks](https://github.com/anthropics/claude-code/issues/14859)
- [GitHub Issue #573 — CLAUDECODE=1 Env Var](https://github.com/anthropics/claude-agent-sdk-python/issues/573)
- [OTel GenAI Agent Spans (experimental)](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/)
- [claude-code-hooks-multi-agent-observability (disler)](https://github.com/disler/claude-code-hooks-multi-agent-observability)
- [Claude Code Data Structures (samkeen gist)](https://gist.github.com/samkeen/dc6a9771a78d1ecee7eb9ec1307f1b52)
- [claude-code-transcripts (simonw)](https://github.com/simonw/claude-code-transcripts)
- [Promptfoo — Evaluate Coding Agents](https://www.promptfoo.dev/docs/guides/evaluate-coding-agents/)
- [DeepEval — Agent Evaluation](https://deepeval.com/guides/guides-ai-agent-evaluation)
- [Anthropic — Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Helicone — Claude Code Integration](https://docs.helicone.ai/integrations/anthropic/claude-code)
- [Portkey — Claude Code Gateway](https://portkey.ai/for/claude-code)
- Prior research: `memory-bank/thoughts/shared/research/2026-03-03-agent-observability.md`
- Prior plan: `memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`
