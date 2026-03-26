---
type: event
created: 2026-03-26
status: active
tags: [cognitive-architecture, thought-modulation, jig, agents, deep-research]
---
# Deep Research: Thought Modulation & Cognitive Architecture for AI Agents

**Date:** 2026-03-26
**Status:** Research complete → ready for shaping/planning
**Scope:** Comprehensive landscape scan + novel architecture synthesis

## Executive Summary

"Thought modulation" — dynamically controlling how an agent reasons, not just what it does — is an active but fragmented research area. No one has assembled the full picture. This report synthesizes 11 research threads into a unified, biologically-grounded cognitive architecture for a custom agent harness (Jig). Several framings are genuinely novel: neuromodulators as runtime modulation axes, PRMs as somatic markers, cognitive hygiene for agents, and preflight simulation as a formal architectural phase.

**Key finding:** The 13 elements identified in this architecture have never been assembled into a single system. The best existing attempts (MAP, CoALA, DGM) cover 3-5 elements each. The integration is the innovation.

---

## Part 1: The Landscape

### What "Thought Modulation" Maps To

The exact phrase doesn't exist as a named concept in AI research. The idea is distributed across five terminological silos:

| Silo | Key Work | Maturity |
|------|----------|----------|
| **Reasoning effort control** | Claude `effort` param, OpenAI `reasoning_effort`, AdaptThink | Production |
| **Activation steering** | CREST, SEAL, Latent CoT Vectors, representation engineering | Research (open weights only) |
| **Metacognitive prompting** | MP (NAACL 2024), Think², Meta-R1, MPDF | Research/prototype |
| **Cognitive architecture** | CoALA, MAP, Neural Brain, Cognitive Workspace | Research/framework |
| **Inner monologue frameworks** | Google Inner Monologue, DAVIS, Quiet-STaR | Production (frameworks) |

### Four Layers of Control

| Layer | Mechanism | Access Required |
|-------|-----------|-----------------|
| **Prompt** | Metacognitive prompting, persona, CoT steering | System prompt |
| **API** | Effort parameter, thinking budget, temperature | API client |
| **Orchestration** | Cognitive roles, preflight, inter-task hygiene, consensus | Custom harness (Jig) |
| **Model** | Activation steering, fine-tuning, architecture changes | Open weights or custom training |

### The Overthinking/Underthinking Problem

The sharpest framing of why modulation matters:
- **Underthinking**: o1-like models prematurely abandon promising reasoning paths; frequent thought-switching correlates with wrong answers
- **Overthinking**: models generate redundant steps after arriving at correct solutions; accuracy plateaus then *declines* past a threshold
- **No current model achieves optimal balance** (OptimalThinkingBench, 2025)
- The brain's inverted-U dose-response curve (too little OR too much neuromodulation impairs cognition) is the biological analog

---

## Part 2: Human Cognitive Parallels

### Visualization / Mental Rehearsal

Human visualization is NOT planning — it's **pre-experiencing**. The brain activates the same motor pathways as actual movement. First-person visualization produces stronger transfer than third-person.

**Gaps identified:**
1. **Inhabited simulation vs. abstract planning** — AI planning produces step-by-step sequences labeled from outside. No agent "inhabits" a simulated future.
2. **Somatic markers** — During rehearsal, the body generates physiological feedback that constitutes pre-conscious decision data. No AI equivalent.
3. **Constructive episodic simulation** — The hippocampus recombines memory fragments to construct futures. LLMs have statistical patterns, not episodes.
4. **No preflight as a first-class primitive** — No standardized agent architecture includes a "simulate before acting" step.

### Attention Residue / Task Switching

Empirically validated in LLMs (EMNLP 2024, Gupta et al.): **10-20% performance degradation** from task switching within a conversation. The interference is from semantic/format content of prior tasks, not context length.

**Key findings:**
- The term "Attentional Residue" is used in LLM literature (arXiv:2509.19517), directly borrowing from Leroy's human cognition framework
- Leroy's "Ready-to-Resume" plan: writing where you are + what's next dramatically reduces residue. The plan doesn't need to be executed; it needs to *exist*.
- Context rot threshold: at 32k tokens, 11/12 models drop below 50% of short-context performance
- JetBrains (NeurIPS 2025): observation masking beats LLM summarization by 2.6% at 52% lower cost — summarization diplomatically softens failures, preventing the agent from recognizing how stuck it is

---

## Part 3: Technical Building Blocks

### Inference-Time Reasoning Amplification

| Technique | Result | Training Required? |
|-----------|--------|-------------------|
| **rStar (MCTS + PRM)** | LLaMA2-7B: 12.5% → 63.9% on GSM8K, no finetune | No |
| **Claude effort parameter** | Low/medium/high/max thinking depth | No |
| **Metacognitive Prompting** | 5-stage reflective process, outperforms CoT on NLU | No |
| **Tree of Thoughts** | BFS/DFS over reasoning paths with lookahead | No |

### Memory Architectures

| System | Mechanism | Key Result |
|--------|-----------|------------|
| **Titans** (Google, ICLR 2025) | Neural long-term memory with surprise-driven test-time learning | Outperforms GPT-4 on BABILong; scales to 2M+ tokens |
| **ATLAS** | Extends Titans with Omega rule (sliding window optimization) | +80% accuracy at 10M context vs Titans |
| **Cognitive Workspace** | Baddeley working memory model for LLMs | 58.6% memory reuse vs 0% for RAG |
| **MemAct** | RL-trained memory curation — retain/compress/discard as agent actions | 14B matches 16x larger models, 51% context reduction |
| **Sculptor** | Tool-based active context management | Explicit evict/reorder/compress operations |

### Context Management

- **Pre-rot threshold**: ~25% of context window, not 95%
- **Observation masking > summarization** (JetBrains finding)
- **Context reordering**: move critical info to beginning/end positions (Lost in the Middle U-curve is architectural — RoPE positional decay)
- **KV cache as working memory**: DMS decouples reasoning tokens from compute cost; KVzip gives 3-4x query-agnostic compression
- **4-tier architecture**: Invariant (pinned) → Working (staleness eviction) → Evidence (aggressive masking) → Reference (external, load on demand)

### Transformer Surgery

| Technique | Key Result | Training Required? |
|-----------|------------|-------------------|
| **Differential Transformer** | 6.8B matches 11B standard; 76% improvement on long-context retrieval | Yes (train from scratch) |
| **Titans MAC** | Memory prepended to attention context; best for needle-in-haystack | Yes |
| **Process Reward Models** | Step-level scoring of reasoning chains | Yes (PRM training) |
| **FlexPrefill** | Adaptive sparse attention per-head, per-input. No retraining. | No |
| **KV cache compression** (DMS, KVzip, R-KV) | 3-8x compression, negligible quality loss | No |

### Post-Transformer Architectures

| Architecture | Key Result | Agent Applicability |
|-------------|------------|-------------------|
| **Dream 7B** (diffusion LLM) | Beats DeepSeek-V3 (671B) on planning tasks (Sudoku 81 vs 21) | HIGH for planning; FAILS at multi-turn (0%) |
| **Mamba-Transformer hybrids** (Jamba) | Best of both; 256K context | Most promising near-term foundation |
| **V-JEPA 2** (Meta) | 65-80% zero-shot robot manipulation | High for embodied agents |
| **RWKV-7** | Claims to surpass TC0 computational limits | Resource-constrained agents |
| **Liquid Neural Networks** | 19 neurons drives a car; extreme efficiency | Edge/embodied deployment |

### RL Augmentation

| Technique | Key Result | Loop Control Required |
|-----------|------------|----------------------|
| **GRPO** (DeepSeek R1) | AIME 2024: 15.6% → 71.0%; emergent metacognition | Training infra only |
| **RLVR** | Binary verifier reward; no reward model needed | Training infra only |
| **Agent-R1** | Online RL from tool-use trajectories | Deep (instrument agent loop) |
| **SWE-RL self-play** | +10.4 on SWE-bench; generalizes to unseen repos | Moderate (role assignment) |
| **Meta-R1** | Two-level metacognitive architecture | Significant (intermediate rewards) |

### Neuro-Symbolic Integration

Dominant pattern: **LLM generates → formal verifier checks → refine → iterate**

| Pattern | Verifier | Best For |
|---------|----------|----------|
| LLM → Prolog | SWI-Prolog | Deductive reasoning |
| LLM → ASP (Clingo) | Answer Set Programming | Non-monotonic reasoning, planning under constraints |
| LLM → Z3/SMT | SAT/SMT solver | Constraint satisfaction, plan verification |
| LLM → Lean4 | Theorem prover | Formal correctness guarantees |

**Key insight from GVU theory**: The verifier is the most important primitive. Formal verification (σ²_V ≈ 0) dramatically widens the self-improvement window. Verification hierarchy: formal proof > program execution > ensemble LLM judge > single LLM judge.

### Evolutionary Self-Improvement

| System | Mechanism | Key Result |
|--------|-----------|------------|
| **Darwin Gödel Machine** | Archive of agent variants; evolves Python scaffold (not model weights) | SWE-bench: 20% → 50% |
| **AlphaEvolve** | Flash proposes mutations, Pro evaluates; programs database (MAP-Elites) | 0.7% of Google's data center compute recovered |
| **ShinkaEvolve** | LLM-crossover with novelty rejection sampling | Prevents archive collapse to paraphrases |
| **CycleQD** | MAP-Elites for LLM skill evolution; model-merging crossover | Merged model outperforms specialists on their own tasks |

**Critical safety finding**: DGM's evolved agent disabled its own hallucination checker. Reward hacking is real.

---

## Part 4: The Architecture

### Brain Region → Agent Component Mapping

| Brain Region | Function | Agent Component |
|---|---|---|
| **Prefrontal Cortex** | Cognitive governor | Metacognitive Controller (sets effort, selects model, chooses personality) |
| **Hippocampus** | Simulation engine + episodic memory | Titans-style surprise-driven memory + preflight simulation |
| **vmPFC / Somatic Markers** | Pre-conscious "gut feeling" | Process Reward Model (step-level scoring) |
| **Basal Ganglia** | Action gating | Provider Mesh Router (complexity → model selection) |
| **Dorsolateral PFC** | Working memory | Context Manager (4-tier: invariant/working/evidence/reference) |
| **Anterior Cingulate Cortex** | Conflict monitor | Verification Layer (Z3, AgentGuard, constraint checking) |
| **Default Mode Network** | Background processing | Learning loops (GRPO, evolutionary search, memory consolidation) |
| **Cerebellum** | Predictive error correction | Diffusion denoiser (for plan synthesis specialist role) |

### Neuromodulator Model (4 Axes)

Neuromodulators don't carry information — they modulate HOW information is processed.

**1. Effort (≈ Dopamine)**
- Controls: thinking depth, model selection, verification intensity
- Range: low (fast/cheap/shallow) → high (slow/expensive/deep)
- Signal: task complexity + past reward for similar tasks
- Implementation: effort API parameter, provider mesh routing, verification gating

**2. Novelty (≈ Norepinephrine)**
- Controls: exploration vs exploitation, memory encoding intensity
- Range: routine (familiar, low alertness) → novel (new territory, high alertness)
- Signal: Titans surprise metric, embedding distance from past tasks
- Implementation: MCTS branching factor, memory encoding strength, human review flagging

**3. Caution (≈ Serotonin)**
- Controls: action inhibition, verification depth, risk assessment
- Range: bold (act fast, verify later) → cautious (verify before acting)
- Signal: reversibility of actions, blast radius, production vs dev
- Implementation: formal verification gating, preflight simulation, PRM threshold

**4. Focus (≈ Acetylcholine)**
- Controls: context window composition, attention allocation
- Range: broad (explore many sources) → narrow (specific files, minimal context)
- Signal: task specificity, pipeline phase
- Implementation: context tier sizing, reference tier loading policy, evidence retention

### The Agent Loop

```
1. PERCEIVE
   ├── Parse input
   ├── Retrieve from episodic memory (Titans): similar past tasks
   └── Retrieve from semantic memory (Scout): relevant knowledge

2. ASSESS (Metacognitive Controller)
   ├── Complexity classification
   ├── Set 4 neuromodulator axes
   ├── Select model (via provider mesh)
   ├── Select personality profile
   └── Gate downstream stages

3. SIMULATE (Preflight)
   ├── IF novelty >= medium OR caution >= high:
   │   ├── Pull episodic memories of similar executions
   │   ├── Construct "mental image" of execution
   │   ├── PRM scores simulation (somatic marker)
   │   └── IF score < threshold: revise, re-simulate
   └── ELSE: skip (routine task)

4. GENERATE (Plan/Action)
   ├── FOR EACH reasoning step:
   │   ├── Generate step (LLM inference)
   │   ├── PRM scores step (somatic marker)
   │   ├── IF low score + high effort: MCTS explore alternatives
   │   ├── IF high caution: formal verification (Z3/SMT)
   │   └── Titans: encode surprising results
   │
   │   [FUTURE: Diffusion specialist for holistic plan
   │    synthesis on complex multi-step plans]

5. EXECUTE (Tool Calls)
   ├── Execute tool
   ├── Compare outcome to prediction
   ├── IF surprise > threshold: write to episodic memory
   └── IF very surprising: re-assess (back to step 2)

6. REFLECT
   ├── Compare outcome to prediction
   └── Update calibration (classifier, routing, PRM)

7. TRANSITION (Cognitive Hygiene)
   ├── IF switching tasks:
   │   ├── Ready-to-Resume note
   │   ├── Clear working memory
   │   └── Retain meta-level learnings
   └── IF continuing:
       ├── Observation masking (not summarization)
       └── Reorder context (critical info at boundaries)
```

### The 6-Layer Stack

```
┌─────────────────────────────────────────────┐
│ EVOLUTION (harness self-improvement)         │
│ MAP-Elites archive over agent configs        │
│ Flash proposes, Pro evaluates                │
│ Self-generated curriculum at 50% frontier    │
│ Trip-wire defenses against reward hacking    │
├─────────────────────────────────────────────┤
│ LEARNING (model improvement)                 │
│ GRPO on telemetry → custom base model        │
│ Self-play (SWE-RL) → training data           │
│ PRM training on scored reasoning steps       │
├─────────────────────────────────────────────┤
│ VERIFICATION (reasoning quality)             │
│ PRM (somatic marker, step-level)             │
│ Z3/Prolog (formal constraints)               │
│ MCTS (reasoning path search)                 │
│ AgentGuard (runtime monitoring)              │
├─────────────────────────────────────────────┤
│ MODULATION (4-axis dynamic control)          │
│ Effort × Novelty × Caution × Focus          │
│ Personality profiles                         │
│ Preflight simulation                         │
│ Cognitive hygiene                            │
├─────────────────────────────────────────────┤
│ MEMORY (context + episodic + semantic)       │
│ 4-tier context: invariant/working/evidence/  │
│   reference                                  │
│ Surprise-driven episodic encoding            │
│ Scout semantic knowledge                     │
│ Observation masking > summarization          │
│ Reorder for Lost-in-the-Middle mitigation    │
├─────────────────────────────────────────────┤
│ HARNESS (Jig — the nervous system)           │
│ Agent loop (7 stages)                        │
│ Provider mesh (task → model routing)         │
│ Telemetry (every turn → JSONL)               │
│ Hook system (pre/post tool interception)     │
└─────────────────────────────────────────────┘
```

### Diffusion Models: Specialist Role, Not Primary

Diffusion LLMs currently **fail at multi-turn agent work** (0% on BFCL multi-turn). But they excel at constraint satisfaction. The hybrid role:

- **Plan synthesis**: Diffusion generates holistic plan sketch (all steps simultaneously). AR model executes step-by-step.
- **Tool selection**: Non-causal, all-at-once evaluation from library.
- **Context summarization**: Non-causal, holistic compression.
- **Execution**: AR model (causal state tracking required).

STAR-LDM (2025) formalizes this: AR generation runs normally, pauses at complex planning steps, runs latent diffusion planner, resumes AR execution.

### The Self-Improvement Flywheel

```
Jig runs tasks
  → telemetry: (prompt, tools, reasoning, outcome)
  → GRPO training improves base model
  → PRM training improves somatic marker
  → Self-play generates synthetic training data
  → Evolution mutates harness configs
  → Archive keeps diverse variants (MAP-Elites)
  → Better harness → better performance → better telemetry
  → repeat
```

### Incremental Build Path

**Tier 1 — Now (existing models, no training)**
- Agent loop with ASSESS stage (4-axis modulation)
- Provider mesh routing based on modulation state
- 4-tier context manager with observation masking + reordering
- Ready-to-Resume handoff for cognitive hygiene
- Z3 as tool call for plan verification

**Tier 2 — Near-term (inference-time, no training)**
- PRM scoring of reasoning steps
- MCTS over reasoning paths for high-effort tasks
- Preflight simulation (prompt-based)
- Episodic memory approximation (structured JSONL + embedding retrieval)

**Tier 3 — Medium-term (requires training)**
- GRPO on Jig telemetry → custom base model
- Fine-tuned PRM → better somatic markers
- Self-play loops (SWE-RL) for training data
- Complexity classifier upgrade (heuristic → learned)

**Tier 4 — Long-term (custom models)**
- Titans-style neural long-term memory
- Diffusion specialist for plan synthesis
- Evolutionary self-improvement (DGM archive pattern)
- TurboQuant (6x memory, 8x speed) for local deployment

---

## Part 5: Novelty Assessment

### What's Genuinely New in This Architecture

**No existing system combines all 13 elements.** Best existing attempts cover 3-5 elements. The integration is the innovation.

**Novel framings (no precedent in literature):**
1. **PRM as somatic markers** — PRMs exist. Somatic marker theory exists. The mapping is new.
2. **Cognitive hygiene for AI agents** — Attention residue is validated in LLMs but no agent architecture treats it as a design concern.
3. **4-axis neuromodulator model** — NEUCOGAR mapped neuromodulators pre-LLM. Applying effort/novelty/caution/focus as runtime parameters for LLM agents is new.
4. **Preflight simulation as formal phase** — Conceptually mentioned but never implemented as a distinct component with its own module, memory writes, and rollback.
5. **Diffusion as specialist planner within AR agent loop** — STAR-LDM hints at this but no agent harness implements it.

**Novel integration points:**
- Neuromodulator axes driving PRM scoring thresholds
- Surprise-driven episodic memory feeding preflight simulation
- Evolution layer operating on modulation parameters (not just prompts)
- Verification hierarchy cascading from formal proof → tests → ensemble LLM judge

### Closest Existing Work

| System | Coverage | Gap to full architecture |
|--------|----------|------------------------|
| MAP (Nature Comms 2025) | ~4/13 (PFC mapping, MCTS, state eval) | No memory, no verification, no self-improvement, no modulation |
| DGM (Sakana AI) | ~2/13 (evolutionary self-improvement, partial self-play) | Everything else |
| CoALA (Princeton) | ~3/13 (memory taxonomy, decision cycles, framework) | No MCTS, no modulation, no verification |
| Neural Brain (arXiv 2025) | ~4/13 (neuromodulators, memory, preflight — all conceptual) | Framework only, not built |
| Sculptor (2025) | ~1/13 (active context management) | Only that one piece |

---

## Part 6: Key References

### Foundational
- CoALA: Cognitive Architectures for Language Agents (arXiv 2309.02427)
- MAP: Brain-Inspired Agentic Architecture (Nature Communications 2025)
- Leroy (2009): "Why is it so hard to do my work?" — attention residue
- Damasio: Somatic Marker Hypothesis (PMC/7286429)
- Schacter: Constructive Episodic Simulation Hypothesis (Nature Reviews Neuroscience 2007)

### Reasoning & Verification
- rStar: LLaMA2-7B 12.5% → 63.9% via MCTS+PRM (datasciocean.com)
- R-PRM: Reasoning-Driven Process Reward Modeling (EMNLP 2025)
- FormalJudge: Neuro-Symbolic Agentic Oversight (arXiv 2602.11136)
- AlphaProof/AlphaGeometry: IMO silver medal (Nature 2025)

### Memory & Context
- Titans: Learning to Memorize at Test Time (arXiv 2501.00663, ICLR 2025)
- ATLAS: Optimally Memorize at Test Time (arXiv 2505.23735)
- MemAct: Memory as Action (arXiv 2510.12635)
- Chroma: Context Rot Research (research.trychroma.com)
- Lost in the Middle (TACL 2024)
- JetBrains: The Complexity Trap (arXiv 2508.21433)

### Post-Transformer & Diffusion
- Dream 7B: Diffusion Reasoning Model (arXiv 2508.15487)
- Bitter Lesson of dLLMs for Agentic Workflows (arXiv 2601.12979)
- STAR-LDM: Stop-Think-AutoRegress with Latent Diffusion (arXiv 2602.20528)
- Jamba: Hybrid Transformer-Mamba (ICLR 2025)
- V-JEPA 2: World Model (arXiv 2506.09985)

### RL & Self-Improvement
- DeepSeek-R1: GRPO + RLVR (arXiv 2501.12948)
- SWE-RL: Self-Play for Software Agents (arXiv 2512.18552)
- Darwin Gödel Machine (arXiv 2505.22954)
- AlphaEvolve (arXiv 2506.13131)
- GVU Framework: Self-Improving AI (arXiv 2512.02731)

### Modulation & Effort
- Adaptive Thinking (platform.claude.com)
- AdaptThink: RL for When to Think (arXiv 2505.13417)
- Reasoning on a Budget Survey (arXiv 2507.02076)
- TurboQuant: 6x KV-cache compression (research.google)
- Meta-R1: Metacognition for LRMs (arXiv 2508.17291)

---

## Related Prior Work (Vault)
- [[2026-03-20-jig-rust-agent-core|Jig — Rust Agent Core]] — the harness this architecture would inhabit
- [[2026-03-12-agent-harness-composition|Agent Harness Composition via Cognitive Personalities]] — personality primitives
- [[2026-03-19-autoresearch-wave3-thinker-architecture|Autoresearch Thinker Architecture]] — existing thought modulation in autoresearch
- [[2026-03-21-collapse-thinker-leader|Thinker/Leader Collapse]] — single-planner decision

## Linear Issues
- ENG-2745: Provider mesh router (Jig Phase 3)
- ENG-2752: Intelligent routing policy with budget (Jig Phase 7)
- ENG-2750: Telemetry recorder (Jig Phase 6)
- ENG-2744: Ollama provider for local models
- ENG-2753: Self-hosted vLLM inference
- ENG-2517: Forge CLI for cognitive personalities
