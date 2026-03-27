---
type: event
created: 2026-03-26
status: active
touched: 2026-03-26
tags: [roadmap, cognitive-architecture, jig, akasha, strategy]
---
# Roadmap: Akasha Cognitive Architecture

**Date:** 2026-03-26
**Status:** Active
**Research basis:** [[2026-03-26-cognitive-architecture-research|Cognitive Architecture Research Brief]]
**Parent vision:** [[2026-03-20-personal-life-ontology-agent-platform|Akasha — The Interface Substrate]]

---

## Principles

1. **Incremental value.** Every tier is independently useful. Don't wait for the full stack.
2. **Platform-first.** The cognitive architecture is HoldCo IP. Both products consume it.
3. **Existing models first.** Tier 1-2 work with frontier APIs. Custom models come later.
4. **Measure before optimizing.** Telemetry and observation before training loops.
5. **Verify before acting.** Trust ladder requires verification before the agent earns action trust.

---

## Dependencies: What Must Ship First

These are already planned/in-flight and are prerequisites for the cognitive architecture:

| Prerequisite | Status | Why Needed |
|---|---|---|
| Platform Connector Abstraction | Planned | Multi-product core — cognitive arch serves both B2B and B2C |
| Backup MCP Server | Planned | First MCP surface — cognitive arch needs tools to reason about |
| Entity Graph (v1) | Not started | Cross-domain reasoning requires connected data |

The cognitive architecture can begin in parallel with these — Tier 1 doesn't require the entity graph.

---

## Tier 1: Foundation — The Modulated Agent Loop

**Timeline:** Concurrent with Jig Phase 1-2
**Requires:** Custom harness (Jig core agent loop)
**Works with:** Frontier APIs only (Claude, GPT, Ollama local)

### What Gets Built

**1.1 — The ASSESS Stage**
Add a metacognitive controller to the agent loop that runs before generation. On every task:
- Classify complexity (heuristic v1: message length, files referenced, tool history)
- Set 4 neuromodulator axes: effort (low/med/high), novelty (routine/novel), caution (bold/cautious), focus (broad/narrow)
- Select model tier based on modulation state
- Gate downstream stages (simulate? verify? which personality?)

**1.2 — 4-Tier Context Manager**
Programmatic context window management:
- Invariant tier (~2-4K): system prompt, identity, modulation state. Pinned.
- Working tier (~16-32K): current plan, recent actions, constraints. Staleness-based eviction.
- Evidence tier (variable): tool outputs. Observation masking by default. Promote to working only if referenced.
- Reference tier (external): Scout vault, codebase via LSP. Load on demand, evict after use.
- Reorder before each model call: critical info at context boundaries (Lost in the Middle mitigation).
- Pre-rot threshold: begin compression at 25% of context window.

**1.3 — Cognitive Hygiene Protocol**
Formalize the transition between tasks:
- Ready-to-Resume note generation (Leroy's attention residue research)
- Working memory eviction on task switch
- Evidence tier full clear between unrelated tasks
- Meta-learning retention (what worked/didn't carries forward)

**1.4 — Personality Profiles (Static)**
YAML-configurable cognitive profiles (from the Forge/cognitive personalities brief):
- Convergent/Divergent thinking styles
- Tool rail sets (scoped permissions per personality)
- System prompt templates per profile
- The ASSESS stage selects personality based on task type

### Value Delivered
- Cost reduction via intelligent model routing (57-80% per AlphaEvolve pattern)
- Better context management → fewer hallucinations, less context rot
- Clean task transitions → less inter-task interference
- Foundation for everything that follows

### Key Decisions Required
- [ ] Jig reprioritization timeline — Tier 1 is blocked on Jig Phase 1
- [ ] Personality profile schema — extend Forge YAML or redesign?
- [ ] Context manager implementation — Rust-native in Jig or inference-server-side?

---

## Tier 2: Intelligence — Reasoning Amplification

**Timeline:** After Tier 1 + Jig Phase 3 (Provider Mesh)
**Requires:** Jig with provider mesh, self-hosted inference capability (vLLM/Ollama)
**Works with:** Frontier APIs + one self-hosted model (Qwen 2.5 Coder or similar)

### What Gets Built

**2.1 — Provider Mesh with Modulation-Driven Routing**
Extend Jig Phase 3 provider mesh to use the full neuromodulator state:
- Low effort + routine → local model (Ollama, TurboQuant-compressed)
- Medium effort → Sonnet-class API
- High effort → Opus-class API
- High caution → route to model with best PRM scores for this task type
- Daily budget tracking with graceful degradation

**2.2 — PRM as Somatic Marker**
Add step-level reasoning scoring:
- v1: Use the model's own log-probabilities as a confidence proxy (no separate PRM needed)
- v2: Fine-tuned PRM on Jig telemetry (Tier 3 dependency)
- On each reasoning step: if PRM score < threshold AND effort ≥ medium → trigger MCTS exploration
- Low PRM on a simulation → revise plan before executing

**2.3 — MCTS Over Reasoning Paths**
Implement rStar-style inference-time tree search:
- Generate N candidate next-steps per reasoning point
- PRM scores each candidate
- Explore top-K, backtrack on dead ends
- For high-effort tasks only (gated by neuromodulator)
- Target: 3-5x accuracy improvement on complex tasks with no model changes

**2.4 — Preflight Simulation**
Before executing a plan or tool call sequence:
- Pull episodic memories of similar past tasks (via embedding similarity over telemetry)
- Generate a "mental image": what files will be touched, what tools needed, what could go wrong
- PRM scores the simulation
- If PRM < threshold: revise before executing
- Gated by: novelty ≥ medium OR caution ≥ high

**2.5 — Formal Verification as Tool Call**
Add Z3/SMT solver as an MCP tool:
- For high-caution tasks: translate constraints to SMT, verify plan satisfies them
- Constraint types: import boundaries (Polylith), API contracts, dependency ordering, scheduling conflicts
- UNSAT → solver returns conflicting constraints → agent revises plan
- For the personal product: financial constraints ("does this payment leave enough for rent?")

**2.6 — Episodic Memory (Approximation)**
Before Titans (requires custom model), approximate with:
- Structured JSONL log of (task, outcome, surprise_score, key_learnings) per session
- Embedding index over episodic entries (LanceDB, same as Scout)
- Retrieval via embedding similarity when ASSESS detects novel task
- Surprise score: heuristic based on outcome divergence from prediction

### Value Delivered
- Dramatic accuracy improvement on complex tasks (MCTS)
- Fewer execution failures via preflight simulation
- Formal correctness guarantees for high-stakes actions (verification)
- Learning from past sessions via episodic memory
- The trust ladder becomes viable — verification enables action trust

### Key Decisions Required
- [ ] PRM v1 approach — self-evaluation confidence vs. external lightweight model?
- [ ] MCTS compute budget — how many branches per step? Token cost cap?
- [ ] Z3 integration — MCP tool or embedded in Jig's Rust runtime?
- [ ] Episodic memory storage — LanceDB (same as Scout) or separate?

---

## Tier 3: Learning — The Self-Improving Agent

**Timeline:** After Tier 2 + Jig Phase 6 (Telemetry) + Phase 7 (Self-Hosted Inference)
**Requires:** Telemetry pipeline, GPU inference infrastructure, training infrastructure
**Works with:** Self-hosted open-weight models + frontier APIs

### What Gets Built

**3.1 — Telemetry Pipeline → Training Data**
Jig Phase 6 captures (prompt, tool_calls, result, outcome) per turn. Extend with:
- PRM scores per reasoning step
- Neuromodulator state per task
- Surprise scores per tool output
- Complexity classifier accuracy (was the routing decision correct?)
- Export: `jig export --format grpo` for training data

**3.2 — GRPO Training on Jig Telemetry**
Fine-tune open-weight models on successful Jig trajectories:
- Filter telemetry by outcome quality (only train on successful runs)
- GRPO (no critic model needed — group relative advantage)
- Reward: task completion + PRM score + efficiency (fewer tokens)
- Target: custom base model tuned to Akasha's specific task distribution
- Deploy via vLLM, compressed via TurboQuant for local inference

**3.3 — PRM Fine-Tuning**
Train a dedicated PRM on Jig's reasoning traces:
- Training data: (reasoning_step, outcome_quality) pairs from telemetry
- The PRM becomes the agent's calibrated "gut feeling" about reasoning quality
- Deploy as a lightweight model called per-step (latency-sensitive)

**3.4 — Self-Play Training Data Generation (SWE-RL Pattern)**
For code-domain tasks:
- Bug-injector role: modify code to introduce bugs + weaken tests
- Bug-solver role: diagnose and fix given weakened test as oracle
- Verifier: deterministic test execution (not LLM judge)
- Generates unlimited training data without human annotation

For personal product domain:
- Scenario generator: create life situations (budget crises, scheduling conflicts, health alerts)
- Coach role: generate appropriate coaching response
- Verifier: rule-based outcome check + ensemble LLM judge
- Builds training data for coaching quality

**3.5 — Complexity Classifier Upgrade**
Replace heuristic classifier with learned model:
- Train on (task_features, actual_complexity, routing_quality) from telemetry
- Features: message embedding, tool call history embedding, file scope, user context
- Output: continuous complexity score → smoother routing decisions

### Value Delivered
- Custom models tuned to Akasha's specific domains
- Better somatic markers (fine-tuned PRM)
- Unlimited training data via self-play
- More accurate routing → better cost/quality tradeoff
- The self-improvement flywheel begins turning

### Key Decisions Required
- [ ] GPU infrastructure — ECS on EC2 (g5.2xlarge ~$400/mo) vs RunPod/Modal pay-per-request?
- [ ] Base model for fine-tuning — Qwen 2.5? Llama? Mistral?
- [ ] Self-play scope — code-only first, or coach-play simultaneously?
- [ ] Training cadence — weekly GRPO runs? Monthly?

---

## Tier 4: Evolution — The Self-Improving Platform

**Timeline:** After Tier 3 is producing stable training loops
**Requires:** Stable Tier 3, archive infrastructure, benchmark suites
**Works with:** Custom fine-tuned models + frontier APIs + evolutionary search

### What Gets Built

**4.1 — Evolutionary Harness Self-Improvement (DGM Pattern)**
The agent evolves its own scaffold:
- Archive of agent variants (MAP-Elites over behavioral dimensions)
- Mutation operators: cheap model proposes config changes, expensive model evaluates
- Behavioral dimensions: tool usage patterns, reasoning depth, revision frequency, delegation patterns
- Fitness: task success + efficiency + user satisfaction
- Novelty rejection: embedding similarity check prevents archive fill with paraphrases
- Defenses: trip-wire tests, private benchmark split, reward capping

What gets evolved:
- System prompts and personality profiles
- Tool selection strategies
- Modulation parameter defaults
- Context management policies
- Preflight simulation triggers

**4.2 — Titans-Style Neural Episodic Memory**
Replace the Tier 2 approximation with genuine test-time learning memory:
- Neural MLP with surprise-driven weight updates during inference
- MAC variant (memory as context — best on long-range tasks)
- Persistent memory (frozen task knowledge) + test-time memory (continuously updating)
- For personal product: the agent's memory of YOU deepens with every interaction
- For B2B: organizational pattern memory compounds

Options:
- Train from scratch (expensive, most control)
- Adapt ATLAS (Google's Titans successor — +80% at 10M context)
- Approximate via MemAct (RL-trained memory curation as action policy)

**4.3 — Diffusion Specialist Models**
NOT replacing the AR agent loop. Specialist roles within it:
- **Plan synthesis:** Generate holistic multi-step plans via masked diffusion (constraint-satisfied simultaneously)
- **Tool selection:** Non-causal evaluation of tool library
- **Context summarization:** Holistic compression of evidence tier
- STAR-LDM pattern: AR generation pauses → diffusion planner runs → AR resumes with plan

**4.4 — Mamba-Transformer Hybrid for Efficiency**
For the local inference path:
- Jamba-style interleaved Mamba + attention layers
- O(N) inference for long contexts (vs O(N²) for pure attention)
- TurboQuant compression for memory efficiency
- Target: run the full cognitive architecture locally on a Mac with GPU

**4.5 — Cross-Domain Derivative Context Engine**
The moat-building system specific to Akasha:
- Sleep vs. spending correlations
- Exercise vs. productivity patterns
- Social isolation vs. health decline
- Temporal trend detection across all connected domains
- Requires: entity graph + episodic memory + high-effort modulated reasoning
- The architecture from Tiers 1-3 makes this possible; Tier 4 makes it good

### Value Delivered
- The platform improves itself without human engineering
- Neural episodic memory creates temporal irreplaceability moat
- Diffusion planning improves complex multi-step task quality
- Local inference makes the personal product viable at consumer scale
- Cross-domain intelligence that no single-domain app can produce

### Key Decisions Required
- [ ] Archive infrastructure — where do agent variants live? How are they evaluated?
- [ ] Titans vs MemAct vs approximation — training cost vs quality tradeoff
- [ ] Diffusion model selection — Dream 7B base? Train custom?
- [ ] Local inference target hardware — Mac M-series? What memory floor?

---

## Timeline Overlay with Akasha Sequencing

```
2026 Q2-Q3: Akasha prerequisites ship
  ├── Platform Connector Abstraction
  ├── Backup MCP Server
  └── Generative UI Protocol (v1)

2026 Q3-Q4: Tier 1 — Foundation
  ├── Jig Phase 1 (core agent loop) ← GATE: Jig reprioritization
  ├── ASSESS stage + neuromodulator model
  ├── 4-tier context manager
  ├── Cognitive hygiene protocol
  └── Static personality profiles

2026 Q4 - 2027 Q1: Tier 2 — Intelligence
  ├── Jig Phase 3 (provider mesh)
  ├── PRM v1 (self-evaluation confidence)
  ├── MCTS over reasoning paths
  ├── Preflight simulation
  ├── Z3 verification tool
  └── Episodic memory approximation

2027 Q1-Q2: Tier 3 — Learning
  ├── Jig Phase 6 (telemetry pipeline)
  ├── Jig Phase 7 (self-hosted inference)
  ├── GRPO training → custom models
  ├── PRM fine-tuning
  ├── Self-play training data
  └── Learned complexity classifier

2027 Q2+: Tier 4 — Evolution
  ├── Evolutionary harness self-improvement
  ├── Titans-style episodic memory
  ├── Diffusion specialist models
  ├── Local inference (Mamba hybrid + TurboQuant)
  └── Cross-domain derivative context engine
```

## Critical Path

The single biggest gate is **Jig reprioritization**. The cognitive architecture requires owning the agent loop. Without Jig, only prompt-level modulation is possible (useful but limited to ~3 of 13 elements).

**If Jig stays deprioritized:** Extract the context manager and modulation model into a Python middleware that wraps Claude Code / Agent SDK calls. Loses Rust performance and deep loop control but preserves most of Tier 1's value. Tier 2+ blocked.

**If Jig gets built:** Full architecture is available. The 4-tier timeline above applies.

---

## The Moat Story

Every tier deepens the moat:
- **Tier 1:** Better agent UX than competitors (modulated responses, clean context, fewer hallucinations)
- **Tier 2:** Dramatically more accurate on complex tasks (MCTS + verification). Trust ladder becomes viable.
- **Tier 3:** Models tuned to Akasha's specific domains. Self-play generates unlimited training data. Competitors can't replicate without the telemetry.
- **Tier 4:** The platform improves itself. Episodic memory creates temporal irreplaceability. Cross-domain intelligence that exists nowhere else.

Each tier compounds on the previous. The architecture is the substrate's nervous system.

---

## Related Artifacts

- [[2026-03-26-cognitive-architecture-research|Research Brief]] — full research landscape (13 elements, brain mapping, key results)
- [[2026-03-20-jig-rust-agent-core|Jig Plan]] — the harness layer
- [[2026-03-12-agent-harness-composition|Cognitive Personalities Brief]] — personality config system
- [[2026-03-20-personal-life-ontology-agent-platform|Akasha Vision]] — founding thesis
- [[2026-03-20-akasha-capability-platform|Capability Platform]] — MCP + generative UI
