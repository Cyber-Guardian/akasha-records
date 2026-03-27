---
type: event
created: 2026-03-26
status: active
touched: 2026-03-26
tags: [research, cognitive-architecture, thought-modulation, jig, akasha, vision]
---
# Research Brief: Biologically-Grounded Cognitive Architecture for Akasha

**Date:** 2026-03-26
**Status:** Shaped → Roadmap
**Scope:** Foundational technical vision for Akasha's intelligence layer

## 1. Origin

Exploration of "thought modulation" — dynamically controlling how agents reason — expanded into a comprehensive cognitive architecture that integrates 13 research threads no one has previously assembled. 10 parallel research agents surveyed the full landscape across neuroscience, ML, agent frameworks, and post-transformer architectures.

## 2. The Finding

**No one has assembled all the pieces.** The field has ~5-7 serious attempts at multi-component cognitive architectures (MAP, CoALA, Neural Brain, DGM, Sculptor), each covering 3-5 of 13 identified elements. None reaches 8+. Several framings are genuinely novel:
- PRM as somatic markers (gut-feeling quality signals)
- Neuromodulator model (4-axis effort/novelty/caution/focus)
- Cognitive hygiene / attention residue management for AI agents
- Preflight simulation as a formal architectural phase

## 3. The 13 Elements

1. Dynamic reasoning effort modulation
2. Biologically-inspired neuromodulator model (effort/novelty/caution/focus)
3. Process Reward Models as somatic markers
4. MCTS over reasoning paths
5. Surprise-driven episodic memory (Titans-style)
6. Active context management (4-tier working memory)
7. Preflight simulation (mental rehearsal before execution)
8. Cognitive hygiene / attention residue management
9. Formal verification integration (Z3/Lean/Prolog)
10. Self-play training loops (SWE-RL pattern)
11. Evolutionary self-improvement of the harness (DGM/AlphaEvolve)
12. Multi-model routing based on task complexity
13. Diffusion models as specialist planners (NOT primary — 0% multi-turn)

## 4. Brain Region → Agent Component Mapping

| Brain Region | Function | Agent Component |
|---|---|---|
| Prefrontal Cortex | Cognitive governor | Metacognitive Controller (ASSESS stage) |
| Hippocampus | Simulation engine + episodic memory | Titans-style surprise-driven memory |
| vmPFC / Somatic Markers | Pre-conscious quality signal | Process Reward Model |
| Basal Ganglia | Action gating | Provider Mesh Router |
| Dorsolateral PFC | Working memory | Context Manager (4-tier) |
| Anterior Cingulate Cortex | Conflict/error monitor | Verification Layer (Z3, AgentGuard) |
| Default Mode Network | Background processing | Learning + Evolution loops |
| Cerebellum | Forward model correction | Diffusion denoiser (specialist) |

## 5. The Neuromodulator Model

Four axes modulate the entire agent loop:

- **Effort (≈ Dopamine):** thinking depth, model selection, verification intensity. Inverted-U: too little = underthinking, too much = overthinking.
- **Novelty (≈ Norepinephrine):** exploration vs exploitation, memory encoding intensity. High = more MCTS branches, stronger memory writes.
- **Caution (≈ Serotonin):** action inhibition, verification depth, risk assessment. High = gate execution behind verification.
- **Focus (≈ Acetylcholine):** context composition, attention allocation. Broad = wide context, narrow = specific files only.

## 6. The Agent Loop

```
1. PERCEIVE → parse input, retrieve episodic + semantic memory
2. ASSESS → classify complexity, set 4 neuromodulator axes, select model + personality
3. SIMULATE → if novel/cautious: preflight mental rehearsal, PRM scores simulation
4. GENERATE → per-step: generate → PRM scores → MCTS if low score → verify if cautious
5. EXECUTE → tool calls, compare outcome to prediction, surprise → write to memory
6. REFLECT → update calibration (classifier, routing, PRM)
7. TRANSITION → cognitive hygiene: Ready-to-Resume note, evict task context, retain meta-learnings
```

## 7. Memory Architecture

Three composing systems:
- **Working Memory (Context Window):** 4-tier (invariant/working/evidence/reference), pre-rot at 25%, observation masking > summarization, reorder critical info to boundaries
- **Episodic Memory (Titans-style):** surprise-driven encoding, test-time learning, used for preflight simulation
- **Semantic Memory (Scout vault):** structured knowledge, long-term persistence, search-based retrieval

## 8. The 6-Layer Stack

```
Evolution  → operates on harness config (hours/days timescale)
Learning   → improves models on trajectories (sessions/weeks)
Verification → scores + verifies reasoning (per-step)
Modulation → 4-axis dynamic control (per-task)
Memory     → context scheduling (continuous)
Harness    → agent loop (per-turn)
```

## 9. Akasha Integration

The cognitive architecture IS Akasha's nervous system — the intelligence layer between the data layer (sync engine, entity graph, connectors) and the interface layer (MCP, generative UI, iMessage).

- **Trust ladder maps to modulation:** backup = low effort/caution, coaching = high effort/broad focus, action = maximum caution + verification
- **Temporal irreplaceability = episodic memory:** every day deepens the moat
- **Derivative context = cross-domain reasoning:** requires deep modulated thought
- **Both products share the cognitive core:** B2B (professional, scoped) and B2C (personal, holistic) are consumers of the same architecture
- **Platform IP sits at HoldCo level**

## 10. Key Research Results

| Result | Implication |
|---|---|
| rStar: 7B → 64% via MCTS+PRM, no finetune | Inference-time scaffolding alone can 5x a model |
| Dream 7B beats 671B on planning (but 0% multi-turn) | Diffusion = specialist planner, not primary backbone |
| DGM: 20% → 50% SWE-bench via self-evolution | Open-ended self-improvement is real |
| Titans: 2M+ context via surprise-driven memory | Active memory outperforms passive RAG by orders of magnitude |
| TurboQuant: 6x memory, 8x speed, zero accuracy loss | Local custom models become practical |
| GRPO: R1-Zero spontaneously developed metacognition | RL + verifiable rewards causes emergent self-regulation |
| GVU theory: verifier quality > generator quality | Formal verification dramatically widens self-improvement window |
| Attention residue: 10-20% LLM performance drop from task-switching | Cognitive hygiene is architecturally necessary, not optional |
| JetBrains: observation masking > summarization | Simple structural compression beats intelligent compression |
| ASP inter-agent communication as prompt injection firewall | Formal logic between agents eliminates social engineering |

## 11. Related Prior Work

- [[2026-03-20-jig-rust-agent-core|Jig — Rust Agent Core]] — the harness layer
- [[2026-03-12-agent-harness-composition|Agent Harness Composition via Cognitive Personalities]] — personality configs
- [[2026-03-19-autoresearch-wave3-thinker-architecture|Autoresearch Thinker Architecture]] — campaign-level modulation
- [[2026-03-20-akasha-capability-platform|Akasha Capability Platform]] — MCP + generative UI
- [[2026-03-20-personal-life-ontology-agent-platform|Akasha — The Interface Substrate]] — founding thesis
- [[akasha-interaction-manifesto|Akasha Interaction Manifesto]] — interaction design philosophy

## 12. Next Step

→ Roadmap: `thoughts/shared/briefs/2026-03-26-cognitive-architecture-roadmap.md`
