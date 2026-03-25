---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Autoresearch Progressive Scope Expansion

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
Autoresearch treats every experiment identically — same tools, same context, same file access, same prompt weight. A simple "write target.py with xxhash + zstd" gets the same 4000-token prompt, WebSearch tools, and architectural freedom as a complex multi-file refactor. This makes early experiments slow (researcher reads irrelevant context, deliberates over unused tools) and late experiments under-equipped (no ability to restructure, add packages, or improve the research environment itself).

The progressive effort system (shipped) scales turns, timeout, and model by campaign maturity. But it doesn't scale **what the researcher can see and do** — the scope is flat.

## Constraints
- Must be backwards-compatible with existing campaigns (progressive_effort=false reverts to flat scope)
- Each scope phase must produce experiments that are evaluable by the same evaluator
- The evaluator contract (target.py exports compress_and_dedup) is immutable across all phases
- Researchers in later phases must be able to build on earlier phases' work (champion code carries forward)
- The existing PlannedAction model carries through the router — scope metadata must fit in it or EffortProfile

## Progressive Scope Phases

### Phase 1: Single File (experiments 1-5)
**What the researcher gets:**
- Prompt: interface spec + hypothesis + available packages. No history, no champion code, no evaluator source.
- Tools: Write, Edit, run_in_container. No Read, Grep, Glob, WebSearch, WebFetch.
- Files: Can only create/modify target.py
- Context: ~500 tokens (interface + hypothesis + packages)

**Why:** A draft is "write one function that does X." The researcher doesn't need to read anything — there's nothing to read yet. Stripping context and tools means faster first response and fewer wasted turns.

**Expected speed:** ~60-90s per experiment (write file + test in container)

### Phase 2: Multi-File + Champion Context (experiments 6-15)
**What the researcher gets:**
- Prompt: interface spec + hypothesis + champion code + eval metadata. Still no full history.
- Tools: Read, Write, Edit, Grep, Glob, run_in_container, report_progress. No web search.
- Files: Can create target.py + helper modules (utils.py, etc.)
- Context: ~2000 tokens (interface + champion code + eval metadata)

**Why:** The researcher is now improving on proven code. It needs to read the champion to understand what to change. Multi-file lets it factor out utilities. But it still doesn't need web search or full campaign history — the champion code IS the context.

**Expected speed:** ~2-3 min per experiment

### Phase 3: Full Workspace + Architecture (experiments 16-30)
**What the researcher gets:**
- Prompt: full campaign context (history, tree summary, personality, knowledge)
- Tools: All current tools including WebSearch, WebFetch, read_strategy
- Files: Full workspace access (target.py, helpers, requirements.txt, config)
- Context: ~4000+ tokens (full current prompt)

**Why:** The campaign is mature. The researcher needs the full picture to make architectural changes — restructuring the pipeline, adding new dependencies, trying fundamentally different approaches informed by what's been tried.

**Expected speed:** ~3-5 min per experiment (current speed)

### Phase 4: Meta-Research (experiments 30+)
**What the researcher gets:**
- Everything in Phase 3 plus:
- Can modify requirements.txt (add new packages)
- Can create .claude rules for future researchers
- Can modify evaluate.py preprocessing (NOT the metric calculation)
- Can create dataset subsets or preprocessing scripts

**Why:** The campaign has explored the algorithmic space. Further gains come from improving the research environment itself — better evaluation, better tools, better constraints for future researchers.

**Expected speed:** ~5-10 min per experiment (complex, high-value)

## Options Considered

### Scope as EffortProfile Extension
Add scope fields to the existing EffortProfile: allowed_tools, allowed_files, context_level, prompt_template. The _compute_effort function returns the full scope alongside turns/model/timeout.
- Gains: Single source of truth for effort + scope. Existing wiring works.
- Costs: EffortProfile grows complex. Mixing effort knobs with scope decisions.
- Complexity: Medium

### Scope as Separate ScopeProfile
New ScopeProfile class parallel to EffortProfile. _compute_scope(config, state) returns it. Wired into _dispatch_researcher alongside effort.
- Gains: Clean separation. Scope logic is independently testable.
- Costs: Two profile objects to wire through. More params on _dispatch_researcher.
- Complexity: Medium

### Scope as PlannedAction Metadata (chosen)
The thinker (or fast-draft generator) specifies scope per action via new fields on PlannedAction: scope_phase, allowed_tools_override, context_level. The coordinator respects these when building the researcher prompt.
- Gains: Per-experiment scope control. Thinker can strategically choose scope. Compatible with the meta-thinker (ENG-2758) tuning knobs.
- Costs: PlannedAction model grows. Need to validate scope against campaign phase.
- Complexity: Medium

## Chosen Approach
**Scope as PlannedAction Metadata** — the thinker/fast-draft generator specifies scope per action, validated against campaign phase by the coordinator. This gives the meta-thinker (ENG-2758) another dimension to tune and keeps scope decisions close to the experiment planning.

For the "ship now" version: hard-wire scope phases in _compute_effort (like we did for turns/model) and strip the prompt/tools accordingly in _dispatch_researcher. Add PlannedAction metadata in a follow-up.

## Key Context Discovered During Shaping
- The researcher prompt is ~4000 tokens including evaluator source, campaign history, champion code, personality instructions, knowledge summary, research directions
- For a draft that's "write fixed-size chunking + xxhash + zstd," ~3500 of those tokens are irrelevant
- Fast drafts (shipped tonight) already skip the thinker — this extends that pattern to the researcher
- The EffortProfile already computes campaign phase — scope phases align 1:1
- Shipped tonight: progressive effort (turns/model/timeout), fast drafts (no thinker for first 5), dispatch instrumentation, baseline on signal tier

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-autoresearch-progressive-scope.md`
