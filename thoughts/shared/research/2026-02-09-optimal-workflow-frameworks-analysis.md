# Optimal Claude Code Workflow Frameworks — Comprehensive Analysis

**Date:** 2026-02-09
**Purpose:** Research 7 frameworks to cherry-pick best patterns for FileScience workflow improvement
**Frameworks:** GSD, Claude Context OS, Superpowers, RIPER-5, CEK, Continuous Claude v3, ACE-FCA

---

## Executive Summary

After deep-diving all 7 frameworks, a clear consensus emerges around **5 core principles** that every serious framework independently discovered:

1. **Context is everything** — LLMs are stateless functions; context window quality is the only lever
2. **Fresh contexts beat accumulated contexts** — Subagents with clean 200K windows outperform degraded long sessions
3. **Phase separation prevents error compounding** — Research, plan, implement must be distinct with quality gates between
4. **Structured artifacts beat prose** — Templates with explicit fields mechanically prevent compression failures
5. **Progressive disclosure saves tokens** — Load only what's needed, when it's needed

---

## Framework-by-Framework Summary

### 1. GSD (Get Shit Done)
**Source:** https://github.com/gsd-build/get-shit-done (12.7K stars)

**Core insight:** Context rot degrades quality at 50%+ utilization. Solution: spawn fresh subagents per task.

**Key patterns:**
- **Lean orchestrator**: Main session stays at 30-40% capacity; agents do heavy lifting
- **Aggressive atomicity**: 2-3 tasks per plan (fits in ~50% of fresh context)
- **Goal-backward verification**: "What must be TRUE?" not "What tasks did we do?"
- **Plans ARE prompts**: XML-structured PLAN.md files read directly by executors
- **Wave-based parallelism**: Independent tasks run simultaneously; dependent tasks wait
- **Codebase mapping**: 4-dimensional analysis (tech, arch, quality, concerns) before planning
- **Traceable requirements**: IDs in REQUIREMENTS.md → 100% coverage validation in ROADMAP.md
- 11 specialized agents (planner, executor, verifier, researcher, etc.)

### 2. Claude Context OS
**Source:** https://github.com/Arkya-AI/claude-context-os

**Core insight:** Prose summaries trigger the same compression that caused context loss. Fix: structured templates.

**Key patterns:**
- **~3.5K token core file** — templates marked "do NOT read at session start"
- **Structured fields prevent failure**: Exact metrics, IF/THEN constraints, decision rationale + rejected alternatives
- **One concern per session**: Research OR writing OR review (not all three)
- **Manual compact at 60-70%** — don't wait for system-forced compaction
- **Handoff efficiency**: Fresh session with handoff = ~5K tokens vs ~50K at message 30
- **docs/summaries/ as persistent brain**: 00-project-brief.md, source-*.md, analysis-*.md, decision-*.md, handoff-*.md
- 5 templates, 3 subagent contracts, 3 workflow phases

### 3. Superpowers
**Source:** https://github.com/obra/superpowers

**Core insight:** Skills must be MANDATORY workflow gates, not suggestions. Leverages commitment/consistency psychology.

**Key patterns:**
- **"Iron Law"**: Skills invoked BEFORE any response, even clarifying questions
- **No rationalization allowed**: Explicit list of rejected excuses ("just a simple question", "I need context first")
- **TDD for everything**: No production code without failing test first (code written before test must be deleted)
- **Two-stage subagent review**: (1) Spec compliance, then (2) Code quality — order matters
- **Claude Search Optimization (CSO)**: Skill descriptions start "Use when..." (triggers, not summaries)
- **Progressive disclosure**: ~2K token startup, full skills loaded on-demand
- **TDD for process documentation**: Observe agent failing without skill, then write minimal guidance
- **Implementer prompt pattern**: Task text pasted directly + context field + "Before You Begin" questions + self-review
- **Git worktrees** for isolated feature work

### 4. RIPER-5
**Source:** https://github.com/tony/claude-code-riper-5

**Core insight:** 5 strict phases with enforced separation: Research → Innovate → Plan → Execute → Review.

**Key patterns:**
- **3 consolidated agents** instead of 5+ (reduces redundant context loading)
- **Phase separation**: Research phase is read-only (no implementation)
- **Memory bank persistence**: Plans stored in `.claude/memory-bank/*/plans/`
- **Strict protocol enforcement**: Prevents premature implementation, context waste, undocumented decisions
- **Minimum necessary capabilities**: Each phase uses only the tools it needs

### 5. CEK (Context Engineering Kit)
**Source:** https://cek.neolab.finance / https://github.com/NeoLabHQ/context-engineering-kit

**Core insight:** Productionize context engineering theory into executable commands with quality gates.

**Key patterns:**
- **Plugin architecture**: Namespaced commands (`/sadd:*`, `/sdd:*`, `/tdd:*`, `/kaizen:*`, `/reflexion:*`)
- **LLM-as-Judge quality gates**: `/sadd:do-and-judge`, `/sadd:judge-with-debate`
- **ADI cycle for decisions**: Abduction (hypotheses) → Deduction (verify logic) → Induction (test evidence)
- **Kaizen commands**: Five Whys root cause, Gemba Walk, Value Stream mapping, A3 problem analysis
- **Intelligent model selection**: Haiku (simple), Sonnet (workflows), Opus (complex reasoning)
- **Tree of Thoughts**: Systematic exploration, pruning, synthesis
- **Competitive generation**: Multiple implementations + multi-judge evaluation
- **3-layer progressive disclosure**: Metadata → Full content → Source files
- **Quality gate math**: Each phase ~80% accurate → 5 ungated phases = 33% cumulative (0.8^5)

### 6. Continuous Claude v3
**Source:** https://github.com/parcadei/Continuous-Claude-v3

**Core insight:** "Compound, don't compact" — extract learnings from thinking blocks before context loss.

**Key patterns:**
- **TLDR 5-layer code analysis** (95% token savings, 155x faster):
  - L1: AST (~500 tokens), L2: Call Graph (+440), L3: CFG (+110), L4: DFG (+130), L5: PDG (+150)
  - Total: ~1,330 tokens vs 26,000+ for raw file reading
- **Thinking block mining**: Headless Claude daemon extracts "why" reasoning after session ends
- **Semantic index**: FAISS + BGE-large-en-v1.5 (1024-dim vectors) for code search by intent
- **32 agents, 109 skills**, two-tier (Sonnet for exploration, Opus for critical work)
- **Meta-skills as workflow chains**: `/fix bug` = sleuth → premortem → kraken → arbiter → commit
- **File claims locking**: Prevents simultaneous edits in multi-agent scenarios
- **Priority levels**: CRITICAL / RECOMMENDED / SUGGESTED / OPTIONAL
- **Blackboard pattern**: Shared state for agent coordination
- **Natural language skill activation**: Intent recognition vs explicit slash commands

### 7. ACE-FCA (Advanced Context Engineering for Coding Agents)
**Source:** https://github.com/humanlayer/advanced-context-engineering-for-coding-agents

**Core insight:** LLMs are stateless functions. Context window is the ONLY lever. Engineer it deliberately.

**Key patterns:**
- **Frequent Intentional Compaction (FIC)**: Target 40-60% utilization throughout
- **Structured artifacts as compaction output**: Markdown docs that distill raw operations
- **Research → Plan → Implement** workflow with sub-agents per phase
- **Context isolation**: Each phase has different pollution risks; subagents mitigate
- **Human leverage at key review points**: Spec review, plan approval, implementation verification
- **Spec-first development**: Specifications are primary artifacts; code is "compiled output"
- **12-factor agent principles**: Stateless design, own your prompts, manage context explicitly
- **CodeLayer vision**: "Post-IDE IDE" for scaling ACE methodology across teams

---

## Cross-Framework Pattern Analysis

### Universal Patterns (found in 5+ frameworks)

| Pattern | GSD | COS | SP | R5 | CEK | CC3 | ACE | Priority |
|---------|-----|-----|----|----|-----|-----|-----|----------|
| Fresh subagent contexts | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | **HIGH** |
| Phase separation (R→P→I) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **HIGH** |
| Quality gates between phases | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | **HIGH** |
| Progressive disclosure | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | **HIGH** |
| Structured artifacts (not prose) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **HIGH** |
| Context utilization targeting | ✓ | ✓ | | | ✓ | ✓ | ✓ | **MEDIUM** |
| Atomic commits per task | ✓ | | ✓ | | | ✓ | | **MEDIUM** |
| Verification before claims | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | **HIGH** |

### Unique/Novel Patterns (found in 1-2 frameworks)

| Pattern | Source | Description | Cherry-pick value |
|---------|--------|-------------|-------------------|
| Goal-backward verification | GSD | "What must be TRUE?" not "What did we do?" | **HIGH** |
| Skills as mandatory gates | Superpowers | No rationalization; invoked BEFORE response | **HIGH** |
| TDD for process docs | Superpowers | Observe failure before writing guidance | **HIGH** |
| Two-stage review (spec→quality) | Superpowers | Separate reviewers, order matters | **HIGH** |
| CSO for descriptions | Superpowers | "Use when..." triggers, not summaries | **HIGH** |
| TLDR 5-layer code analysis | CC3 | 95% token savings via AST+graphs | **MEDIUM** (complex) |
| Thinking block mining | CC3 | Headless daemon extracts "why" reasoning | **MEDIUM** (complex) |
| ADI cycle for decisions | CEK | Abduction→Deduction→Induction | **MEDIUM** |
| Kaizen/Five Whys | CEK | Lean methodology for root cause analysis | **MEDIUM** |
| Quality gate math | CEK | 0.8^5 = 0.33 without gates | **HIGH** (mental model) |
| One concern per session | COS | Research OR write OR review (not all) | **MEDIUM** |
| Manual compact at 60-70% | COS | Proactive, not reactive compaction | **HIGH** |
| File claims locking | CC3 | Prevent simultaneous edits | **LOW** (not needed yet) |
| Competitive generation | CEK | Multiple solutions + multi-judge | **LOW** (overkill for us) |

---

## What We Already Do Well

Comparing against our current setup:

| Capability | Our Current State | Assessment |
|-----------|-------------------|------------|
| Memory bank (durable/archival split) | ✅ Mature | Better than most frameworks |
| PostToolUse hooks (ruff/ty/import-linter) | ✅ Active | Ahead of most frameworks |
| Phase separation (routes A-E) | ✅ Defined | Good foundation, could be stricter |
| Structured plans | ✅ Via /create_plan | Good, could add verification criteria |
| Archival artifacts | ✅ Dated research/plans/decisions | Very strong pattern |
| Durable memory size targets | ✅ Documented | Need enforcement |
| Subagent types | ✅ 12 types defined | Good coverage |
| Architectural fitness | ✅ 7 import-linter contracts | Unique strength |
| Session closeout | ✅ Defined in CLAUDE.md | Could be more automated |

---

## Recommended Improvements (Prioritized)

### Tier 1: High Value, Low Effort (Do First)

#### 1. Add verification criteria to plans
**From:** GSD (goal-backward verification), Superpowers (verification-before-completion)
**What:** Every plan phase gets `## Verification` with "What must be TRUE?" criteria
**Why:** Prevents "looks done" without evidence. 0.8^5 = 0.33 quality without gates.
**Implementation:** Update `/create_plan` skill template

#### 2. Rewrite route descriptions as triggers
**From:** Superpowers (Claude Search Optimization)
**What:** Change CLAUDE.md route descriptions from narrative to "Use when..." patterns
**Example:** Route A becomes: "Use when user asks where/how something works, wants to understand existing code, or needs to trace a dependency."
**Why:** Claude's pattern matching works better with concrete triggers than narrative descriptions

#### 3. Add mandatory skill enforcement language
**From:** Superpowers (skills as gates)
**What:** Add to CLAUDE.md: "If a route matches, you MUST use the corresponding skill BEFORE responding. Common rationalization patterns to reject: [list]"
**Why:** Prevents route-skipping on "simple" tasks

#### 4. Proactive compact suggestion at 60-70%
**From:** Claude Context OS, ACE-FCA
**What:** Hook or CLAUDE.md instruction to suggest /compact or session handoff at 60-70% capacity
**Why:** Prevents surprise auto-compaction that loses context. We already have `strategic_compact_suggester.sh` — ensure threshold is right.

#### 5. Enforce durable memory size limits
**From:** CEK (progressive disclosure), ACE-FCA (FIC)
**What:** PostToolUse hook on Write/Edit of durable files that warns when exceeding targets
**Why:** Prevents context bloat; forces linking to archival artifacts

### Tier 2: High Value, Medium Effort

#### 6. Two-stage validation in /implement_plan
**From:** Superpowers (spec→quality review), CEK (LLM-as-Judge)
**What:** After each phase: (1) Spec compliance check ("Does this match the plan?"), (2) Quality check (tests pass, types check, architecture holds)
**Implementation:** Modify `/implement_plan` skill to spawn validator subagent between phases

#### 7. Structured implementer prompts for subagents
**From:** Superpowers (implementer prompt pattern), GSD (task context)
**What:** When `/implement_plan` spawns subagents, use structured prompt: task text (pasted, not file ref) + context (where task fits) + "Before You Begin" (ask questions first) + self-review checklist
**Why:** Subagents perform better with structured context than bare instructions

#### 8. Wave-based parallel execution
**From:** GSD (wave parallelism)
**What:** In `/implement_plan`, identify independent tasks and run them simultaneously in Wave 1, then dependent tasks in Wave 2
**Implementation:** Add dependency analysis to plan execution logic

#### 9. Add /debug skill with systematic methodology
**From:** Superpowers (systematic-debugging), CEK (Five Whys)
**What:** New skill that enforces: root-cause-tracing → hypothesis generation → experiment design → one-at-a-time testing
**Why:** Prevents guess-and-check debugging loops

#### 10. Plan header with execution instructions
**From:** Superpowers (meta-instructions in plans)
**What:** Every plan includes header: `> **Execution:** Use /implement_plan <this-path> to execute phase by phase.`
**Why:** Makes handoff explicit; future sessions know how to proceed

### Tier 3: Medium Value, Higher Effort (Future)

#### 11. TDD for process documentation
**From:** Superpowers (writing-skills)
**What:** Before adding new CLAUDE.md guidance, observe Claude failing without it, document failures, write minimal guidance
**Why:** Prevents over-engineering of process docs

#### 12. Codebase mapping before major work
**From:** GSD (/gsd:map-codebase)
**What:** Add `/map_codebase` skill that analyzes existing code in 4 dimensions before planning
**Implementation:** Spawn 4 parallel agents: tech stack, architecture, conventions, concerns

#### 13. ADI cycle for architectural decisions
**From:** CEK (First Principles Framework)
**What:** Route D decision-making enforces: generate 3+ hypotheses → verify logic → gather evidence → audit bias → document
**Why:** Prevents anchoring on first solution

#### 14. Quick mode for lightweight tasks
**From:** GSD (/gsd:quick)
**What:** Skip full planning for bug fixes, config changes, one-off tasks while maintaining quality guarantees
**Implementation:** New route/skill for tasks under ~30 minutes scope

---

## Hooks Opportunities

### Current hooks:
- PreToolUse: `strategic_compact_suggester.sh` (on all tools)
- PostToolUse: `ruff`, `ty`, `import-linter` (on Write/Edit of .py files)

### Recommended additions:

| Hook | Type | Trigger | Purpose | Source |
|------|------|---------|---------|--------|
| Durable memory size checker | PostToolUse | Write/Edit of `memory-bank/durable/**` | Warn if exceeding size targets | CEK, ACE-FCA |
| Verification reminder | PostToolUse | After test/check commands | Remind to document evidence | Superpowers |
| Session state saver | Stop | On session end | Auto-create minimal handoff | CC3, COS |

### NOT recommended (too complex for current needs):
- TLDR 5-layer code analysis (CC3) — requires significant infrastructure
- Semantic index with FAISS (CC3) — overkill for current codebase size
- File claims locking (CC3) — not doing parallel multi-agent edits yet
- Thinking block mining (CC3) — requires headless daemon infrastructure

---

## Key Mental Models to Internalize

### 1. The Quality Gate Tax
From CEK: Each ungated phase has ~80% accuracy. 5 ungated phases = 0.8^5 = **33% cumulative quality**. Adding gates between phases (even simple ones) dramatically improves outcomes.

### 2. Context Window as Budget
From ACE-FCA: Every token spent on context history is a token NOT available for reasoning. Target 40-60% utilization. At 70%+, quality measurably degrades.

### 3. Plans ARE Prompts
From GSD: A well-structured plan is directly executable by an agent. The plan file should be written FOR the agent, not for a human reviewing it later.

### 4. Compound, Don't Compact
From CC3: Don't let context accumulate until compaction destroys it. Extract learnings continuously and start fresh sessions with full context.

### 5. Observe Before Prescribing
From Superpowers: Before writing any process documentation, watch the agent fail without it. Document the exact failure modes. Write minimal guidance that addresses those specific failures.

---

## Sources

### Primary Repositories
- [GSD (Get Shit Done)](https://github.com/gsd-build/get-shit-done)
- [Claude Context OS](https://github.com/Arkya-AI/claude-context-os)
- [Superpowers](https://github.com/obra/superpowers)
- [RIPER-5](https://github.com/tony/claude-code-riper-5)
- [CEK (Context Engineering Kit)](https://cek.neolab.finance)
- [Continuous Claude v3](https://github.com/parcadei/Continuous-Claude-v3)
- [ACE-FCA](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents/blob/main/ace-fca.md)

### Secondary Sources
- [llm-tldr (5-layer code analysis)](https://github.com/parcadei/llm-tldr)
- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents)
- [DeepWiki: Continuous-Claude-v3](https://deepwiki.com/parcadei/Continuous-Claude-v3)
- [DeepWiki: ACE-FCA](https://deepwiki.com/humanlayer/advanced-context-engineering-for-coding-agents)
