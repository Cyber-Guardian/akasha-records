---
type: event
created: 2026-03-16
status: active
date: 2026-03-16T14:00:00-04:00
author: jordan
git_commit: e99f7b2
branch: main
repository: filescience
topic: "Agent Factory Testing Strategy"
tags: [decisions, testing, agent-factory, pass-1]
last_updated: 2026-03-16
last_updated_by: jordan
---

# Decision: Agent Factory Testing Strategy

## Context

Pass 1 exit test: "Can an agent implement a bounded V2 task and pass mutation testing?" Without machine-verifiable correctness gates, every agent PR requires human review — defeating the autonomy goal. Current gates are all deterministic/static (ruff, import-linter, ty). The gates that catch semantic regressions (mutation, contracts, RBSM) are either missing or advisory-only.

Research grounding (2026-03-09 and 2026-03-16 shaping session):
- Google mutation testing: advisory-only across 6000+ engineers. Incremental (changed lines, cap 7/file) is the only PR-viable approach.
- Slack: moved 99% E2E post-merge — p95 turnaround -40%, flakiness -90%, no production defect increase.
- Simon Willison red/green TDD: tests must exist AND fail before agent implementation. Non-optional.
- `@runtime_checkable` + `isinstance()` only checks method existence, not signatures — behavioral assertions required.
- OOPSLA 2025: stateless property-based tests kill 50x more mutations than unit tests per test line. Hypothesis `@given` for pure function invariants is highest-leverage test type.
- beartype/typeguard: not recommended for Protocol checking. beartype uses O(1) probabilistic sampling (50% miss rate on single-element violations). Static analysis (ty) + behavioral tests is better ROI.

## Decision

Adopt a **layered testing pyramid** with incremental mutation testing as a PR gate, Protocol-based contracts at cross-brick boundaries, and a strict red/green TDD workflow that progressively trusts agents as gates prove themselves.

---

## 1. Testing Pyramid

| Layer | Gate Type | Status | What It Proves | Tools | Threshold/Config |
|-------|-----------|--------|---------------|-------|-----------------|
| Static analysis | PR hard gate | **Active** | Well-formed, typed, architecturally sound, secure | ruff ALL, ty, import-linter (9 contracts), semgrep, detect-secrets, pip-audit | Zero violations |
| Unit tests | PR hard gate | **Active** | Individual functions correct | pytest, moto, respx, pytest-asyncio | 80% diff-cover on changed lines |
| Stateless property tests | PR hard gate | **Planned** | Pure functions obey invariants (round-trip, idempotence, commutativity) | Hypothesis `@given` (stateless, not RBSM) | Run with unit tests; 50x mutation kill rate vs unit tests per OOPSLA 2025 |
| Contract conformance | PR hard gate | **Planned** | Cross-brick interfaces honored | Protocol + `test_contracts.py` per brick | All contract tests pass |
| Incremental mutation | PR hard gate | **Planned** | Tests actually verify behavior | `scripts/incremental_mutmut.py` + mutmut | 70% kill rate on changed lines, 7 mutants/file cap |
| RBSM (Hypothesis) | PR gate (always) | **Active** | State machines handle arbitrary events | Hypothesis RBSM, `tools-hypothesis` CI job | ci profile: 100 examples / 50 steps (`tools/test/conftest.py:60`) |
| Full mutation | Nightly advisory | **Active** (job fails below threshold) | Codebase-wide test quality | mutmut + `mutation_filter.py` | 70% meaningful score (`sys.exit(1)` on failure) |
| Nightly mutation ratchet | Nightly advisory | **Planned** | Score doesn't regress | Artifact-based score comparison in `mutation.yml` | Alert if score drops >2% week-over-week |
| RBSM deep exploration | Nightly advisory | **Active** | Deep state space coverage | Hypothesis nightly profile | 1000 examples / 100 steps with `target()` guidance (`tools/test/conftest.py:73`) |
| E2E smoke | Post-deploy | **Planned** | Full workflows function | pytest `@pytest.mark.e2e` suite | One file per service workflow, deselected by default |

---

## 2. Contract Test Pattern

### Which components need Protocols

**Need Protocols** (consumed by bases with behavioral interfaces):
- `cloud_api` — `SessionManager`, `ApiClient`, `ResponseIterator` consumed by discover and entity_discovery_trigger
- `dynamodb` — `Table`, `aioTable`, query/scan/write functions consumed by discover and entity_discovery_trigger
- `throttling` — largest Protocol surface. `ThrottleService` (14+ methods) + nested `.functions.*` chain (7 methods) + data types consumed across bases: `M365ThrottleContext`, `AdmissionRequest`, `AdmissionToken`, `RequestUsage`, `RequestPolicyType`, `build_request_policy`, `RateGateSpec`, `OnedriveTraversalPolicy`, `OutlookTraversalPolicy`. This sub-issue is the biggest of the three — scope accordingly.

**Do NOT need Protocols** (interfaces are pure data or trivial):
- `models` — pure dataclasses, consumed via direct imports
- `domain_backup` — pure dataclasses (`@dataclass(kw_only=True)`), same logic as models
- `lambda_utils` — pure functions (`capture_failed_events`), no behavioral interface
- `telemetry` — pure functions (`capture_method`, `increment_counter`), no behavioral interface
- `tool_config` — pure data read (`get_secret()` returning strings), no behavioral interface

### Convention

- One `protocols.py` per component that has cross-brick consumers
- One `test_contracts.py` per brick in `test/components/filescience/<brick>/`
- Recording fakes (e.g., `RecordingSlackClient`) must implement the Protocol
- Conformance test: `isinstance(RecordingFake(), TheProtocol)` + behavioral assertions

### Nested interface note

`throttle_client` in `discover/queue.py:57` is typed as `Any` and exposes a `.functions.*` chain:
- `throttle_client.functions.try_prune_throttle_context(...)`
- `throttle_client.functions.penalize_throttle_context(...)`
- `throttle_client.functions.get_throttle_context_data(...)`

The Protocol for `throttling` must capture this nested interface — not just the top-level `ThrottleService` class. Define a `ThrottleFunctionsProtocol` for the `.functions` attribute.

### Code sketch: cloud_api Protocol + contract test

```python
# components/filescience/cloud_api/protocols.py
from __future__ import annotations

from typing import Protocol, runtime_checkable

import httpx


@runtime_checkable
class SessionManagerProtocol(Protocol):
    """Contract for session management consumed by bases."""

    async def get_session(self, tenant_id: str, cloud: str) -> httpx.AsyncClient: ...
```

```python
# test/components/filescience/cloud_api/test_contracts.py
from filescience.cloud_api import SessionManager
from filescience.cloud_api.protocols import SessionManagerProtocol


class TestSessionManagerConformance:
    def test_implements_protocol(self) -> None:
        assert isinstance(SessionManager(), SessionManagerProtocol)

    async def test_get_session_returns_client(self, mock_session_manager) -> None:
        """Behavioral assertion: method actually works, not just exists."""
        session = await mock_session_manager.get_session("tenant-1", "microsoft")
        assert isinstance(session, httpx.AsyncClient)
```

This pattern is load-bearing — all subsequent Protocol + contract test sub-issues should follow it.

---

## 3. Red/Green TDD Workflow

### "Agent-ready" checklist

**Pass 1 (human writes tests, agent implements):**
Before an agent starts implementing on a brick:
- [ ] Failing tests exist for every acceptance criterion in the issue
- [ ] Contract tests exist for any new cross-brick interface
- [ ] The brick's test directory exists with at least `conftest.py`
- [ ] Human has verified the tests fail for the right reason (not import errors)

**Pass 2+ (agent writes tests, gates validate):**
Before an agent starts implementing:
- [ ] Contract tests exist and pass for all cross-brick interfaces the brick touches
- [ ] Incremental mutation gate is active in CI
- [ ] Agent writes its own tests — incremental mutation validates they're real

### The workflow

```
Human writes red tests → Agent makes green → Incremental mutation validates
                                            → Human reviews architecture only
```

### Graduation criteria (Pass 1 → Pass 2+)

A brick graduates from Pass 1 (human-written tests) to Pass 2+ (agent-written tests) when ALL of:
1. Contract tests exist and pass for all interfaces the brick consumes
2. Incremental mutation kill rate >= 70% on 3 consecutive nightly runs for that brick
3. Jordan explicitly approves the promotion

Track graduation status in the brick's test directory `conftest.py` as a comment: `# Agent-ready: Pass 2+ (promoted YYYY-MM-DD)`

---

## 4. Mutation Testing Enforcement

### Current state (Active)

- **Full nightly run**: mutmut via Docker copy-tree, `mutation_filter.py --threshold 70`
- **Behavior**: `mutation_filter.py` exits with code 1 if meaningful score < 70% (`return 0 if summary["passed"] else 1` at `scripts/mutation_filter.py:249`)
- **The nightly job DOES fail** on threshold breach — this is a real gate, not just a report
- **No week-over-week ratchet** — threshold is static 70%, no historical comparison

### Target state (Planned)

- **Incremental PR gate** (pending sub-issue): mutate only changed lines via `scripts/incremental_mutmut.py`. Cap at 7 mutants per file. 70% kill rate threshold. Modeled after `scripts/affected_tests.py` diff-scoped selection pattern.
- **Nightly ratchet** (pending sub-issue): store current score as GitHub Actions artifact on each nightly run. Fetch prior score from most-recent artifact. Fail if delta exceeds -2%.

### Skip annotation convention

`# mutmut: skip — [justification]` with mandatory justification comment. Track skip rate — if >30%, heuristic refinement needed.

---

## 5. E2E / Smoke Test Structure

- **Directory**: `test/e2e/` for cross-service scenarios
- **Convention**: one file per service workflow (e.g., `test_discover_flow.py`, `test_entity_trigger_flow.py`)
- **Gate type**: post-deploy smoke (not PR-blocking)
- **Marker**: `@pytest.mark.e2e` (deselected by default, run by deploy pipeline)
- **Scope**: end-to-end workflows that exercise the full path through a service, using moto/respx for external deps

---

## 6. Implementation Roadmap

| # | Sub-Issue | Scope | Execution Mode | Dependencies | Priority |
|---|-----------|-------|---------------|-------------|----------|
| 1 | Protocol + contracts: cloud_api | `protocols.py` + `test_contracts.py` | autonomous | None | High |
| 2 | Protocol + contracts: dynamodb | `protocols.py` + `test_contracts.py` | autonomous | None | High |
| 3 | Protocol + contracts: throttling | `protocols.py` (incl. nested `.functions`) + `test_contracts.py` | autonomous | None | High |
| 4 | Incremental mutation gate | `scripts/incremental_mutmut.py` + CI workflow | co-dev (spike) | None | High |
| 5 | Nightly mutation ratchet | Artifact storage + comparison in `mutation.yml` | autonomous | None | Medium |
| 6 | Recording fake conformance | Upgrade `RecordingSlackClient` + others to implement Protocols | autonomous | #1-3 (Protocols defined first) | Medium |
| 7 | Zero-test bootstrap: Tier 1 | `models`, `domain_backup`, `lambda_utils` — pure unit tests | autonomous | None | Medium |
| 8 | Zero-test bootstrap: Tier 2 | `cloud_api`, `tool_config`, `tool_slack_client`, `tool_linear_client` — mocked external deps | autonomous (Pass 2+) | #1-4 (Protocols + mutation gate active) | Medium |
| 9 | Zero-test bootstrap: Tier 3 | `tool_durable_harness` — stateful, needs RBSM | co-dev | #1-4 | Low |
| 10 | E2E smoke scaffold | `test/e2e/` directory + one reference test | autonomous | #7-8 (base coverage exists) | Low |
| 11 | Stateless property tests | `@given` properties for pure functions in `models`, `domain_backup`, `cloud_api/suppression.py`, `dynamodb` serializers. Round-trip, idempotence, commutativity invariants. | autonomous | None | Medium |
| 12 | Fix Hypothesis conftest docstring | Update stale 500/2000 to match actual 100/1000 | autonomous | None | Trivial |

---

## Options Considered

### A: Trust the Pyramid (advisory mutation)
Hard PR gates at bottom (static + unit + contract), mutation nightly advisory only with 2% ratchet.
- Pros: Matches Google/Slack/Block practice. Fast PR cycle.
- Cons: Agents can ship poorly-tested code that passes unit tests. No PR-time semantic verification.

### B: Incremental Mutation Gate (chosen)
Everything from A plus diff-scoped mutation on PR. 70% kill rate on changed lines.
- Pros: Catches the exact failure mode of agent-generated code. Per-Google data: 82% productive.
- Cons: Requires custom tooling. ~30-60s additional CI time.

### C: Strict Red/Green (human-only tests)
Human writes all tests, agent only implements. Mutation stays advisory.
- Pros: Eliminates "who watches the watchmen" completely.
- Cons: Bottlenecks on human. Doesn't scale past Pass 1.

## Tradeoffs

**What we gain**: Machine-verifiable correctness for agent-generated code. Progressive trust model. Concrete "agent-ready" definition. Every layer either blocks merge or alerts within 24 hours.

**What we give up**: Custom tooling investment for incremental mutation (~1 sub-issue spike). Human bottleneck during Pass 1 (acceptable for foundation work). ~30-60s additional PR CI time when incremental mutation ships.

## Consequences

1. All subsequent testing sub-issues point to this document as source of truth
2. Protocol definitions must be authored before agents can work autonomously on cross-brick features
3. The incremental mutation spike (sub-issue #4) is the critical path — if it proves infeasible, fall back to Approach A (advisory-only mutation)
4. Bricks without contract tests remain in Pass 1 (human-written tests only) until graduated
5. The Hypothesis conftest docstring is stale (says 500/2000, code says 100/1000) — fix trivially

## References

- [[2026-03-16-agent-factory-testing-strategy|Shaping Brief]]
- [[2026-03-09-platform-v2-roadmap-strategy|Platform V2 Roadmap Strategy]]
- [[2026-03-11-testing-infrastructure-portfolio|Testing Infrastructure Portfolio]]
- State machine testing guide: `.claude/reference/state-machine-testing-guide.md`
- Mutation filter: `scripts/mutation_filter.py:249` (exit code behavior)
- Hypothesis profiles: `tools/test/conftest.py:60-83` (actual values vs stale docstring)
- Cross-brick `Any` typing: `bases/filescience/discover/queue.py:57`, `dispatcher.py:40`
- Existing contract test: `test/bases/filescience/valkey_queue_processor/policies/test_registry.py`
- Recording fake: `test/shared/recording_fakes.py`
- OOPSLA 2025 PBT study: https://dl.acm.org/doi/10.1145/3764068
- Agentic PBT (arXiv): https://arxiv.org/abs/2510.09907
- Block.xyz AI agent testing pyramid: https://engineering.block.xyz/blog/testing-pyramid-for-ai-agents
- Meta JiT Testing: https://engineering.fb.com/2026/02/11/developer-tools/the-death-of-traditional-testing-agentic-development-jit-testing-revival/
- CPython runtime_checkable perf: https://github.com/python/cpython/issues/102936
