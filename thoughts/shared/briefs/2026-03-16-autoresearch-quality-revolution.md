---
type: event
created: 2026-03-16
status: active
related:
  - thoughts/shared/plans/2026-03-16-autoresearch-production-hardening.md
  - thoughts/shared/plans/2026-03-16-autonomous-experiment-harness-v3.md
---
# Autoresearch Quality Revolution — Shaping Brief

> The infrastructure works. The research quality doesn't. This brief shapes improvements across every layer of the autonomous experiment harness — from hypothesis generation to evaluation to learning.

## Problem Statement

The harness ran 3 experiments in its first real campaign: 1 crash ($1.01 wasted), 1 keep (5% improvement), 1 discard (regression). The crash burned half the budget producing nothing. The "keep" was modest. The system doesn't learn from failure, doesn't pre-validate implementations, and throws away most of the signal it generates.

**Root causes, by layer:**

### Layer 1: The lab leader is flying blind on failures

The history summary shows crashes as `#N [crash] score=N/A hypothesis=<80 chars>`. No error message. No stack trace. No "what went wrong." The lab leader's `reasoning` and `confidence` fields are captured in the record but never fed back. The researcher's `implementation_summary` is stored in JSONL but invisible to the lab leader. The diff is stored but never shown. The lab leader makes the next decision with minimal signal about *why* the last attempt failed.

**Evidence:** Experiment 1 crashed. Experiment 2's brief said "likely caused by an import error or edge case" — a guess, not grounded in the actual crash. The lab leader had no error information to work from.

### Layer 2: The researcher gets no historical context

The researcher receives only: the lab leader's brief, hypothesis, personality preamble, research directions, and target file list. It does NOT receive: campaign history, prior failure details, champion code, what approaches have already been tried, or why previous experiments failed. Each researcher starts from zero context about the campaign's trajectory.

**Evidence:** Experiment 3 tried smaller chunks + dictionary compression, which is a reasonable idea — but it had no way to know that the champion already used CDC chunking. It could have focused on refining the champion's approach instead of partially reinventing it.

### Layer 3: No pre-validation = expensive crashes

There is zero validation between "researcher finishes writing code" and "full evaluator runs." No syntax check. No import check. No quick smoke test. The commit is created before evaluation. A researcher that writes `import zstd` (wrong package name — should be `zstandard`) burns the full eval timeout before the crash is detected.

**Evidence:** Experiment 1 crashed during evaluation, not during implementation. A 2-second `python -c "import target"` check would have caught it before the expensive eval.

### Layer 4: Dead signals and wasted data

| Signal | Captured? | Fed back? | Used? |
|--------|-----------|-----------|-------|
| `LabLeaderDecision.reasoning` | Yes (JSONL) | No | Dead |
| `LabLeaderDecision.confidence` | Yes (JSONL) | No | Dead |
| `ResearcherResult.implementation_summary` | Yes (JSONL) | No | Dead |
| `ExperimentRecord.diff` | Yes (5KB cap) | No | Dead |
| `progress_log` (MCP) | In-memory only | No | Discarded |
| `ResearcherUsage.num_turns` | Yes (dispatch) | No | Not in JSONL |
| `ResearcherUsage.duration_ms` | Yes (dispatch) | No | Not in JSONL |
| `metric_variance` | Field exists | Never populated | Dead |
| `researcher_timeout` config | Field exists | Never wired | Dead |
| `leader_timeout` config | Field exists | Never wired | Dead |

### Layer 5: No novelty filtering

The system can re-propose the same approach. There's no embedding-based similarity check, no deduplication of hypotheses, no mechanism to say "we already tried CDC chunking, propose something different." The lab leader's only signal is the 80-char truncated hypothesis in the history summary.

### Layer 6: Evaluation is all-or-nothing

Single-stage evaluation: run the full evaluator or nothing. No fast proxy metric. No staged gate. No early stopping for obviously bad implementations. Every experiment pays the full eval cost even if a 2-second check would reveal it's broken.

## Desired End State

An overnight 50-experiment campaign where:
1. Crashes cost < $0.10 each (caught by pre-validation, not burned on full eval)
2. The lab leader names *specific* prior failures and avoids them (Reflexion-style verbal memory)
3. The researcher knows what the champion code looks like and what's been tried before
4. Near-duplicate hypotheses are rejected before spending tokens (embedding-based novelty gate)
5. Evaluation is staged: syntax → smoke → full eval (failures caught early = cheap)
6. Every data signal feeds back into the loop (reasoning, implementation_summary, failure notes)
7. Cost per useful experiment (keep) drops by 50%+

## Approach Options

### Option A: Feedback Loop Fix (highest leverage, lowest effort)

Fix the information flow without changing the architecture. Focus on getting more signal into prompts and feeding data back.

**Changes:**
1. **Richer history summary** — include failure notes, implementation summaries, and researcher costs in the history shown to lab leader. Sort by score (OPRO-style trajectory prompting).
2. **Reflexion-style failure memory** — after each crash/discard, generate a 1-sentence "lesson learned" and store it in the experiment record. Feed all lessons into the lab leader prompt as "## Mistakes to Avoid."
3. **Feed champion code to researcher** — embed current champion code in the researcher's system prompt so it knows what it's building on.
4. **Wire dead signals** — populate `metric_variance`, persist `progress_log`, write `num_turns`/`duration_ms` to JSONL, wire `researcher_timeout`/`leader_timeout`.
5. **Richer lab leader system prompt** — replace the generic "You are a research director" with campaign-specific context including the strategy doc (already present in the user prompt but the PydanticAI `instructions` field is wasted).

**Effort:** ~1 plan, 2-3 phases. Mostly prompt changes + model field updates.
**Impact:** Moderate. Better decisions, less blind repetition. Doesn't prevent crashes.

### Option B: Pre-Validation Gate (biggest cost saver)

Add a validation stage between researcher completion and full evaluation.

**Changes:**
1. **Syntax gate** — `python -c "import ast; ast.parse(open('target.py').read())"` — 0.1s, catches syntax errors
2. **Import gate** — `python -c "import target"` — 1s, catches import errors (the Experiment 1 killer)
3. **Quick smoke test** — run evaluator with a tiny dataset or short timeout (5s) to catch runtime crashes
4. **Self-debug loop** — if pre-validation fails, give the researcher one chance to fix (AIDE debug operator pattern). Include the error message. Cap at 1 retry.
5. **Skip commit for pre-validation failures** — don't create a git commit for code that doesn't parse. Save the git ops cost.

**Effort:** ~1 plan, 1-2 phases. New `validate_experiment()` function in coordinator.
**Impact:** High. Crashes drop from $1+ to $0.10. The debug retry may recover some crashes into keeps.

### Option C: Staged Evaluation Pipeline (evaluation efficiency)

Replace the all-or-nothing evaluator with a multi-stage funnel.

**Changes:**
1. **Stage 1: Syntax + import** (0.1s) — catch broken code immediately
2. **Stage 2: Smoke eval** (5s) — run evaluator on a 10% data sample or with reduced iterations
3. **Stage 3: Full eval** (30-120s) — only for experiments that pass stages 1-2
4. **Early stopping** — if running average after 30% of eval iterations is below champion by >2σ, stop
5. **Multiple runs with significance testing** — `evaluator_runs > 1` already supported but `metric_variance` is never populated. Wire it and use it for statistical comparisons.

**Effort:** ~1 plan, 2 phases. Evaluator refactor + coordinator integration.
**Impact:** High for long evals. Moderate for short evals. Prevents wasting 120s on obviously bad code.

### Option D: Novelty Gate + Atomic Hypotheses (exploration quality)

Prevent redundant work and improve attribution.

**Changes:**
1. **Embedding-based novelty filter** (ShinkaEvolve) — embed each hypothesis, reject if cosine similarity > 0.95 to any prior experiment. Use `text-embedding-3-small` ($0.00002/1K tokens — negligible).
2. **Atomic hypothesis constraint** (AIDE) — instruct the lab leader to propose exactly ONE change per experiment. "Switch from zlib to zstd" not "switch compression + change chunk size + add dictionary training." Cleaner attribution of what caused improvement.
3. **OPRO-style sorted trajectory** — present history sorted by score ascending so the lab leader sees the progression toward better solutions.
4. **Failure taxonomy** — classify crashes into categories (import error, syntax error, runtime error, eval timeout, budget exceeded) and include the category in history.

**Effort:** ~1 plan, 2 phases. Embedding API integration + prompt restructuring.
**Impact:** Moderate-high. Reduces wasted experiments on duplicate approaches. Better attribution means the lab leader learns faster.

### Option E: Full Architecture Evolution (long-term)

Adopt the best patterns from AlphaEvolve, AIDE, and ShinkaEvolve.

**Changes:**
1. **Tree search over hypotheses** (AI Scientist v2) — instead of linear sequence, maintain a tree of experiment branches. Prune bad branches, deepen good ones.
2. **Draft/Debug/Improve operators** (AIDE) — separate modes for the researcher: initial implementation, fix broken code, or refine working code. Different prompts and tool configs per mode.
3. **UCB1 bandit over models** (ShinkaEvolve) — track which model produces the best improvement-per-dollar. Dynamically allocate budget.
4. **Island model** (AlphaEvolve/OpenEvolve) — run multiple independent populations with periodic migration of best solutions.
5. **Campaign-level learning** — after a campaign completes, summarize what worked into a "campaign insight doc" that seeds the next campaign's strategy.

**Effort:** Multiple plans, significant refactor. Weeks of work.
**Impact:** Transformative, but overkill for current scale (single-campaign, single-target-file experiments).

## Recommendation

**Do B first, then A, then D.** In that order.

**Why B first:** Pre-validation is the single highest-leverage change. Experiment 1's $1.01 crash would have cost $0.05 with a syntax/import gate. The self-debug retry might have recovered it into a working experiment. This is pure cost savings with zero prompt engineering risk.

**Why A second:** The feedback loop fix requires no architectural changes — it's prompt engineering and data wiring. The Reflexion-style failure memory and richer history will compound over campaigns. Every experiment generates more learning signal for the next.

**Why D third:** Once the system doesn't crash and learns from failures, novelty filtering prevents the remaining failure mode: re-trying the same approach. Atomic hypotheses make the learning signal cleaner.

**Park C** until evaluations regularly exceed 60s. Current evals are ~5-25s — staged gating adds complexity for marginal gain at this scale.

**Park E** until we've run 5+ successful campaigns and have empirical data on where the current architecture hits its ceiling.

## Key References

- **AIDE** (Weco AI) — Draft/Debug/Improve operators, atomic changes, tree search: [arxiv 2502.13138](https://arxiv.org/html/2502.13138v1)
- **Reflexion** — Verbal failure memory: [arxiv 2303.11366](https://arxiv.org/abs/2303.11366)
- **OPRO** — Trajectory-based prompt optimization: [arxiv 2309.03409](https://arxiv.org/abs/2309.03409)
- **ShinkaEvolve** — Novelty filtering, UCB1 model bandit: [arxiv 2509.19349](https://arxiv.org/abs/2509.19349)
- **AlphaEvolve** — Dual-model strategy, island model: [DeepMind blog](https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)
- **AI Scientist v2** — Tree search over hypotheses: [arxiv 2504.08066](https://arxiv.org/abs/2504.08066)
- **Darwin Gödel Machine** — Staged evaluation, full lineage archive: [arxiv 2505.22954](https://arxiv.org/abs/2505.22954)
