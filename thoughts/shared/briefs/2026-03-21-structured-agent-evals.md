---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Structured End-to-End Agent/Agentic System Evaluations

**Date:** 2026-03-21
**Status:** Shaping

## Problem Statement

FileScience has four agentic systems (autoresearch, workspace MCP, triage agent, Helm) with no structured way to verify they work end-to-end. The current validation layers are:

| Layer | What it catches | What it misses |
|-------|----------------|----------------|
| **Unit tests** (mock-heavy) | Individual function behavior | System-level composition failures, tool interaction bugs, prompt regressions |
| **RBSM** (Hypothesis state machines) | State transition bugs in stateful components | Multi-agent coordination, prompt quality, outcome correctness |
| **Manual "run it and see"** | Everything, if you look | Non-obvious regressions, intermittent failures, things that happen at 2am |
| **Overnight campaigns** (autoresearch) | Experiment harness health, code quality | No structured pass/fail criteria; a campaign that runs = "success" even if every experiment crashes |

**Specific gaps we cannot validate today:**

1. **Triage agent e2e:** Does a Slack message arrive, get routed, produce a Linear issue, and respond correctly? The FunctionModel harness tests scripted tool sequences, not actual agent reasoning.
2. **Autoresearch e2e:** Does the thinker-researcher-evaluator loop produce experiments that improve on the baseline? Overnight campaigns are de facto integration tests but have no structured quality criteria beyond "did it crash?"
3. **Workspace MCP e2e:** Does create_workspace -> write_file -> run_command -> push_branch -> create_pr work as a complete workflow? Unit tests mock Docker; the real Docker path is tested only manually.
4. **Helm e2e:** Does decompose -> dispatch -> merge produce a valid result? The state machine is tested (RBSM) but the full orchestration with real agents is not.
5. **Cross-system interactions:** Autoresearch uses workspaces. Helm uses workspaces. Triage creates Linear issues that Helm decomposes. None of these integration points are tested.
6. **Prompt regression detection:** When a system prompt changes, there is no automated way to verify the agent still behaves correctly. Changes are validated by "run it once and eyeball the output."

## Landscape: What Approaches Exist?

### Anthropic's Guidance (March 2026)

[Anthropic's agent eval guide](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) is the most directly relevant reference. Key recommendations:

- **Start with 20-50 tasks drawn from real failures.** Early changes have large effect sizes, so small sample sizes suffice.
- **Grade outcomes, not trajectories.** Agents find valid approaches designers didn't anticipate. Check what the agent produced, not the exact path.
- **Build partial credit.** A triage agent that identifies the problem correctly but fails to create the issue is meaningfully better than one that fails immediately.
- **Separate capability evals from regression evals.** Capability evals start with low pass rates and measure improvement. Regression evals maintain near-100% pass rates and catch backsliding. Graduate high-pass capability evals into the regression suite.
- **Use pass@k and pass^k.** pass@k = probability of at least one success in k runs (useful when one working solution matters). pass^k = probability all k succeed (essential for reliability-critical paths).
- **Model-based graders handle nuance.** Code-based graders are fast but brittle. LLM-as-judge handles open-ended assessment. Human graders are gold standard but don't scale. Use all three.
- **Read transcripts.** "You won't know if graders work without reading transcripts from many trials."

### Open-Source Frameworks

| Framework | Maintained By | Approach | Relevance to FileScience |
|-----------|--------------|----------|--------------------------|
| [Inspect AI](https://inspect.aisi.org.uk/) | UK AISI | Declarative eval tasks, sandboxed execution, model-graded scoring, Docker isolation | **High** -- composable tasks, built-in agent evals (OSWorld, AgentBench, BFCL), Python-native, MIT license |
| [METR](https://metr.org/) | METR (non-profit) | 180+ tasks measuring autonomous capability, time-horizon analysis | Medium -- academic benchmark focus, useful conceptual model but not directly deployable for custom systems |
| [AgentBench](https://github.com/THUDM/AgentBench) | Tsinghua THUDM | 8-environment multi-dimensional benchmark (OS, DB, KG, web) | Low -- general LLM-as-agent benchmark, not for testing custom agentic systems |
| [Braintrust](https://www.braintrust.dev/) | Braintrust Inc. | Eval-driven CI/CD, golden sets, regression thresholds | Medium -- good eval CI patterns but proprietary, could use concepts without the product |
| [Bloom](https://alignment.anthropic.com/2025/bloom-auto-evals/) | Anthropic Alignment | Auto-generates behavioral evals from seed descriptions, 4-stage pipeline | Medium -- behavioral trait measurement, not directly for system-level evals |

### SWE-CI: The Emerging Pattern (March 2026)

[SWE-CI](https://arxiv.org/abs/2603.03823) represents a paradigm shift from static one-shot evaluation (SWE-bench) to **dynamic CI-integrated evaluation**. Key insight: agents that hard-code brittle fixes and agents that write clean extensible code both pass SWE-bench -- their difference in maintainability is invisible. SWE-CI introduces EvoScore (Evolution Score) measuring long-term code health across 233-day evolution histories.

This matters for FileScience because autoresearch experiments are exactly this pattern: a researcher agent modifies code, an evaluator scores it, and the system evolves over 50+ iterations per campaign. EvoScore-style metrics would catch researchers that game the evaluator with brittle hacks.

### Trajectory vs. Outcome Evaluation

The field has converged on the principle that **both are necessary**:

- **Trajectory evaluation** examines the sequence of reasoning steps: Did the agent use tools appropriately? Was it efficient? Did it recover from errors? LangChain's AgentEvals, T-Eval, and AgentBoard all implement trajectory scoring.
- **Outcome evaluation** checks the final environmental state: Did the Linear issue actually get created? Does the code compile? Is the PR mergeable? Anthropic explicitly recommends prioritizing outcomes over trajectories for most use cases.

For FileScience's systems, outcome evaluation is the higher-leverage starting point. We care whether the triage agent produced a correct Linear issue, not whether it called search_memory before or after get_memory_doc.

### Practical Testing Layers (Industry Consensus)

The industry has converged on a testing pyramid for agentic systems:

1. **Unit tests** -- validate JSON outputs, tool wrappers, retry logic
2. **Integration tests** -- multi-step scenarios against recorded tool responses ("tool tapes")
3. **E2E tests** -- agents against live tools or staging environments (small, focused set)
4. **Adversarial tests** -- prompt injection, malicious tool outputs, privilege escalation
5. **Human review** -- ongoing transcript sampling, annotation queues

## What We Already Have (Codebase Assets)

The codebase has more eval infrastructure than it appears at first glance. It just isn't unified under an "eval" umbrella:

### 1. Autoresearch Evaluator Pattern
`bases/filescience/tool_autoresearch/evaluator.py` is a production-grade eval runner:
- Subprocess-based evaluation with timeout handling, crash recovery, hash verification
- Multi-run averaging with adaptive confidence (MAD-based early stopping)
- Structured `EvaluatorResult` output: `metric_value`, `metadata`, `duration_seconds`, `eval_confidence`
- Immutability enforcement via directory hashing

This is already a **generic eval harness** -- it runs a command, parses JSON output with `metric_value`, and handles errors. It could evaluate any agentic system, not just autoresearch experiments.

### 2. Triage Agent Harness
`tools/test/harness/harness_utils.py` provides:
- `FunctionModel`-based deterministic testing (scripted tool call sequences)
- Structured `InvocationResult`: output, turns, tool_calls, state, duration_ms
- Real infrastructure invocation against ephemeral stack (moto + Valkey + DynamoDB)
- Tool call extraction and linking (ToolCallPart <-> ToolReturnPart)

This is a **deterministic replay harness** -- useful for regression but not for testing actual LLM behavior.

### 3. Dev-Harness MCP Server
`tools/dev-harness/src/dev_harness/server.py` is a universal service invocation framework:
- Adapter pattern for any Polylith brick (triage, edt, vqp, discover)
- Ephemeral stack integration (moto, Valkey, DynamoDB bootstrap)
- OTel instrumentation via component "dyno" testing
- Observability query tools (traces, logs, metrics via VictoriaMetrics)

This is the **execution infrastructure** -- it can invoke any service handler with instrumented tracing.

### 4. Forge (Planned) Goal Evaluator
`memory-bank/thoughts/shared/plans/2026-03-12-agent-harness-composition.md` defines:
- Machine-checkable goal sets (YAML) with pluggable evaluators
- Check types: command, lighthouse, file_exists, contains, test_suite
- Structured `GoalEvaluation` with per-check pass/fail and confidence
- Consensus protocols for multi-agent agreement

This is a **goal evaluation framework** -- not yet built but well-designed.

### 5. Testing Infrastructure Portfolio (Planned)
`memory-bank/thoughts/shared/plans/2026-03-11-testing-infrastructure-portfolio.md` maps:
- 6 milestones across 3 passes: Mutation Gate -> Security -> Model-Based Testing -> Interface Verification -> Verification Platform -> Production Confidence
- Pass 3 includes "Smoke / E2E" and "Progressive delivery" but no agent eval framework
- The portfolio is missing a "structured agent evals" milestone

### 6. Agent Observability Research
`memory-bank/thoughts/shared/research/2026-03-03-agent-observability.md` catalogs:
- 14 agent failure modes (MAST taxonomy)
- Compounding probability problem (97% per-step = 72% over 10 steps)
- Observability tooling comparison (Langfuse, Phoenix, Braintrust)
- OTel GenAI semantic conventions for traces/spans

## Candidate Approaches

### Option A: Outcome-Based Eval Suite (Lightweight, Custom)

Build a thin eval runner on top of the existing autoresearch evaluator pattern. Each agentic system gets a set of eval tasks defined as YAML + Python scripts.

**How it works:**
- Define eval tasks as `{input, expected_outcome, grader}` triples in YAML
- Graders are Python functions returning `{score: float, passed: bool, metadata: dict}`
- Runner executes tasks, aggregates results, outputs structured JSON
- CI integration: run regression eval suite on PRs touching agent code; run capability eval suite nightly
- Reuse `evaluator.py`'s subprocess execution, timeout handling, multi-run averaging

**Eval tasks per system:**
- **Triage:** 20 scenarios (Slack messages -> expected Linear issue properties, expected routing decisions, expected "ignore" for noise)
- **Autoresearch:** 10 scenarios (experiment configs -> expected evaluator behavior, thinker strategy quality, researcher code quality)
- **Workspace MCP:** 10 scenarios (lifecycle operations -> expected container state, file content, git state)
- **Helm:** 10 scenarios (plan files -> expected decomposition, dispatch, merge outcomes)

**Grading:** Code-based for deterministic outcomes (issue created? PR merged? container running?). LLM-as-judge for quality assessment (is the triage summary accurate? is the experiment plan sound?).

- **Gains:** Smallest possible investment; reuses existing infrastructure; directly tests what we care about (outcomes); fits existing pytest + CI workflow
- **Costs:** Custom framework maintenance; no trajectory analysis; no community benchmarks
- **Complexity:** Low-Medium
- **Time to first value:** 1-2 days for triage eval suite, 1 week for all four systems

### Option B: Inspect AI Integration (Framework-Based)

Adopt UK AISI's [Inspect AI](https://inspect.aisi.org.uk/) framework. Define eval tasks using Inspect's declarative API. Leverage its sandboxing, model-graded scoring, and existing agent eval patterns.

**How it works:**
- Define `@task` functions that set up scenarios, invoke the agentic system, and score results
- Use Inspect's `sandbox()` for Docker-isolated execution (mirrors workspace container model)
- Use Inspect's model-graded scoring for quality assessment
- CI integration via `inspect eval` command
- Community benchmarks (BFCL for tool calling, AgentBench patterns) as supplementary signals

**Example task (triage eval):**
```python
@task
def triage_routing():
    return Task(
        dataset=[
            Sample(input="New customer needs Clio backup", target="create_issue"),
            Sample(input="Hello!", target="ignore"),
        ],
        solver=triage_agent_solver(),
        scorer=model_graded_qa(),
    )
```

- **Gains:** Mature framework (MIT, active development, AISI-backed); built-in sandboxing; model-graded scoring; community; reproducible eval logs; dashboard for results
- **Costs:** New dependency; learning curve; may over-abstract for our custom systems; Inspect's solver/scorer model designed for LLM evals, not necessarily for system-level orchestration testing
- **Complexity:** Medium
- **Time to first value:** 3-5 days (learning curve + adaptation to custom systems)

### Option C: Hybrid -- Custom Outcome Suite + Trajectory Recording

Build Option A for immediate value, but add trajectory recording that captures every tool call, LLM response, and state transition. Use recorded trajectories for regression testing, debugging, and eventual LLM-as-judge scoring.

**How it works:**
- Phase 1: Outcome eval suite (Option A) -- 1 week
- Phase 2: Trajectory recording middleware -- instrument each agentic system to emit structured trajectory logs (tool calls, state transitions, LLM outputs) to JSON. This builds on the existing OTel work and the triage harness's `extract_tool_calls` pattern.
- Phase 3: Trajectory-based regression -- record golden trajectories from successful runs. On future runs, compare trajectory similarity (tool call sequence, state transitions) to detect regressions without re-running LLM-as-judge.
- Phase 4 (optional): LLM-as-judge trajectory scoring for quality dimensions (efficiency, tool appropriateness, reasoning quality)

- **Gains:** Immediate value from outcomes; trajectory data enables debugging and future LLM-as-judge scoring; golden trajectory regression is cheap to run; builds on existing OTel + harness infrastructure
- **Costs:** More total investment; trajectory recording needs per-system instrumentation; golden trajectory maintenance burden
- **Complexity:** Medium-High (phased)
- **Time to first value:** 1 week for Phase 1, then incremental

## Recommendation: Option C (Hybrid), Start with Option A

**Start with Option A** because it delivers the highest immediate value at the lowest cost. The autoresearch evaluator pattern is already proven and generalizable. The triage harness already captures structured results. The dev-harness already invokes services with instrumentation.

**Evolve toward Option C** by adding trajectory recording once outcomes are covered. Trajectory data is the prerequisite for LLM-as-judge scoring, debugging, and the kind of deep analysis Anthropic recommends ("read transcripts from many trials").

**Skip Option B** for now. Inspect AI is excellent for evaluating LLM capabilities against standard benchmarks, but our problem is testing custom agentic systems against domain-specific criteria. The overhead of fitting our systems into Inspect's solver/scorer model isn't justified when we already have a working eval runner pattern. Revisit if the eval suite grows beyond 100+ tasks and we need a proper eval platform.

### Why not Inspect AI?

Inspect AI's sweet spot is "evaluate Model X on Benchmark Y." FileScience's need is "verify System X produces Outcome Y when given Input Z." The systems under test aren't LLMs -- they're orchestration layers, state machines, and tool-calling agents with specific business logic. The autoresearch evaluator pattern (subprocess + JSON output + structured scoring) is a better fit for this level of testing.

### Alignment with existing plans

This slots into the Testing Infrastructure Portfolio as a new capability under **Production Confidence (Pass 3)**. The existing portfolio has "Define post-deploy smoke suite pattern" -- agent evals are the agentic equivalent of smoke tests. However, agent evals should likely start in Pass 2 (alongside the Verification Platform) since they're needed before production deployment.

## First Step: Triage Agent Eval Suite (Smallest Useful Thing)

**Why triage:** It's the simplest system (single agent, defined tool set, clear success criteria), has an existing harness (`harness_utils.py`), and has the most obvious eval tasks (Slack message -> Linear issue).

**What to build:**

1. **Eval task format:** YAML file defining `{input_text, expected_behavior, grader_type}`
   ```yaml
   tasks:
     - name: route-backup-request
       input: "Customer ACME needs Clio backup configured"
       expected:
         behavior: create_issue
         issue_contains: ["ACME", "Clio", "backup"]
       grader: outcome_check

     - name: ignore-greeting
       input: "Hey there!"
       expected:
         behavior: no_action
       grader: outcome_check

     - name: quality-triage-summary
       input: "We're seeing intermittent 429 errors from the Clio API during large org discoveries"
       expected:
         behavior: create_issue
         summary_quality: "accurately identifies rate limiting as the issue"
       grader: llm_judge
   ```

2. **Eval runner:** Extends the existing `invoke_triage_handler` to run against a real model (not FunctionModel), capture the full result, and pass it to graders.

3. **Graders:**
   - `outcome_check`: Did the agent take the expected action? Does the output contain expected content? (code-based, deterministic)
   - `llm_judge`: Is the output quality acceptable? (model-based, uses a rubric)

4. **CI integration:** `make eval-triage` runs the regression suite (near-100% expected pass rate). Nightly: run the capability suite with pass@3 scoring.

5. **Reporting:** Structured JSON output compatible with the existing autoresearch evaluator format (`metric_value`, `metadata`).

**Effort estimate:** 1-2 days for the triage eval suite. One more day each for autoresearch, workspace, and Helm eval suites.

**Non-goals for first step:**
- No trajectory recording yet
- No LLM-as-judge (start with code-based graders only)
- No new framework dependencies
- No dashboard or visualization

## Key Insight

The autoresearch harness IS an agent eval framework -- it just doesn't know it yet. It already runs a subprocess, parses `{metric_value, metadata}`, handles timeouts, does multi-run averaging, and verifies immutability. Generalizing this pattern to evaluate other agentic systems is a small delta from the current codebase.

## Related Work

- [[2026-03-11-testing-infrastructure-portfolio|Testing Infrastructure Portfolio]] -- the larger verification strategy this slots into
- [[2026-03-04-extension-validation-pipeline|Extension Validation Pipeline]] -- Claude Code extension testing (complementary scope)
- [[2026-03-03-agent-observability|Agent Observability Research]] -- failure taxonomies and OTel patterns (prerequisite knowledge for trajectory recording)
- [[2026-03-12-agent-harness-composition|Forge Harness Composition]] -- goal evaluator design (reusable patterns)
- [[2026-03-18-autoresearch-v2-platform|Autoresearch V2 Platform]] -- the system most ready for eval framework extraction

## Sources

- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Anthropic: Bloom Auto-Evals](https://alignment.anthropic.com/2025/bloom-auto-evals/)
- [UK AISI Inspect AI](https://inspect.aisi.org.uk/evals/)
- [METR: Measuring Autonomous AI Capabilities](https://metr.org/measuring-autonomous-ai-capabilities/)
- [AgentBench (ICLR 2024)](https://github.com/THUDM/AgentBench)
- [SWE-CI: CI-Integrated Agent Evaluation](https://arxiv.org/abs/2603.03823)
- [AI21: Scaling Agentic Evaluation -- Lessons from 200K SWE-bench Runs](https://www.ai21.com/blog/scaling-agentic-evaluation-swe-bench/)
- [LangChain: Trajectory Evaluations](https://docs.langchain.com/langsmith/trajectory-evals)
- [Beyond Task Completion: Assessment Framework for Agentic AI](https://arxiv.org/html/2512.12791v2)
- [VirtusLab: Testing Evaluating Agentic Systems](https://virtuslab.com/blog/ai/testing-evaluating-agentic-systems)
- [Maxim: How to Evaluate AI Agents](https://www.getmaxim.ai/articles/how-to-evaluate-ai-agents-comprehensive-strategies-for-reliable-high-quality-agentic-systems/)
- [AgentRewardBench: Evaluating Automatic Evaluations of Web Agent Trajectories](https://arxiv.org/pdf/2504.08942)

## Next Step

- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-structured-agent-evals.md`
