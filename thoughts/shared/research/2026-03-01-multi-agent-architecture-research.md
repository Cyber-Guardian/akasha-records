# Multi-Agent Architecture Research — Informing Helm Design

**Date:** 2026-03-01
**Purpose:** Survey of production and academic multi-agent coding systems to inform the FileScience agent helm orchestration design.

## TL;DR

Conflict prevention beats conflict resolution. Design/decomposition quality is the #1 bottleneck at scale. Two agents coordinating achieve half the success of one (CooperBench). The only successful tournament systems use automated evaluators (test execution, fitness functions), not generic LLM judges. Stacked diffs + serial merge queue is more battle-tested than parallel merge. Advisory file locks work in practice with capable models.

---

## Production Systems

### Gas Town (Steve Yegge) — The Reference Implementation

**Scale:** 20-30 concurrent Claude Code instances. ~$100/hr at full scale.
**Source:** [GitHub](https://github.com/steveyegge/gastown) | [HN discussion](https://news.ycombinator.com/item?id=46734302)

**Architecture:** Kubernetes-inspired hierarchical fleet orchestration.

| Role | Function |
|------|----------|
| Mayor | Human-facing coordinator; never writes code |
| Polecats | Ephemeral worker agents; produce merge requests, then disappear |
| Refinery | Manages merge queue serially; resolves conflicts; can re-imagine implementations |
| Witness | Monitors workers, unblocks stuck agents |
| Deacon | Oversight patrol loops |
| Dogs | Maintenance/cleanup |

**Work units:**
- Molecules = chained task sequences stored as Beads in git
- Beads = git-backed issue-tracking units with explicit acceptance criteria
- Convoys = bundles of Beads assigned to a single agent

**Key patterns:**
- **Stacked diffs** — atomic changes merge independently, minimizing conflict surface
- **Serial merge queue** — Refinery integrates branches one at a time
- **Nondeterministic idempotence** — if agent crashes, next session reads Bead state and resumes. Path may differ, outcome converges because acceptance criteria are fixed.
- All state lives in git-backed Beads, not agent memory

**Critical quote from Yegge:** "Gas Town churns through plans so quickly you need to do a LOT of design to keep the engine fed."

**Lesson for us:** Design/decomposition is the true bottleneck. The helm's decomposition engine quality determines everything. Also: serial merge queue (Refinery pattern) is more reliable than trying to merge in parallel.

---

### Cursor 2.0 Multi-Agent Suite

**Scale:** Up to 8 agents in parallel.
**Source:** [Changelog](https://cursor.com/changelog/2-0) | [SkyWork analysis](https://skywork.ai/blog/vibecoding/cursor-2-0-multi-agent-suite/)

**Architecture:** Hierarchical Planner/Worker/Judge triad with git-worktree isolation.

**What Cursor tried and abandoned:**
- **Equal-status agents with file locking** → agents held locks too long; 20 agents degraded to throughput of 2-3
- **Optimistic concurrency control** → agents became risk-averse, avoided hard tasks

**What works:** Hierarchical planning + strict file ownership + worktrees. Merging at human review time, not during execution.

**Lesson for us:** File locking degrades throughput. Optimistic concurrency makes agents timid. Pre-assigned file ownership via decomposition is the winning pattern.

---

### Claude Code Agent Teams (Anthropic)

**Source:** [Official docs](https://code.claude.com/docs/en/agent-teams)

**Architecture:** Lead + 5-6 teammates. Shared task list with dependency edges.

**Key details:**
- No native worktree isolation — must manually ensure non-overlapping file ownership
- Mailbox system for P2P direct messaging (auto-delivery, no polling)
- `TeammateIdle` and `TaskCompleted` hooks for quality gates
- Teammates do NOT inherit lead's conversation history — each loads context fresh from CLAUDE.md
- Recommended team size 3-5; beyond that coordination overhead dominates

**Lesson for us:** The task decomposition with dependency edges maps directly to our helm manifest. The "teammates don't inherit lead context" constraint is important — each sub-issue agent starts cold.

---

### OpenAI Codex

**Source:** [OpenAI announcement](https://openai.com/index/introducing-codex/)

Each task runs in its own cloud sandbox container with git worktree isolation. No agent-to-agent communication — strictly human-mediated. Strong isolation model that sidesteps coordination entirely.

**Lesson for us:** The simplest pattern that works: give each agent its own sandbox, let the human merge. Our Cyrus worktree model is essentially this.

---

### Google Jules

**Source:** [Google blog](https://blog.google/technology/google-labs/jules/)

Single async agent with internal **Critique Agent** that reviews code before completion. 2M token context window. Not multi-agent coordination — one agent, one task, long-running.

**Lesson for us:** The internal Critique Agent is a clean pattern for quality gates. Could inform our judge agent design — an internal reviewer before PR submission.

---

### Entire.io (Thomas Dohmke)

**Source:** [GitHub CLI](https://github.com/entireio/cli) | [TechCrunch](https://techcrunch.com/2026/02/10/former-github-ceo-raises-record-60m-dev-tool-seed-round-at-300m-valuation/)

Not an orchestrator — a **provenance layer**. Captures full prompt/response transcript, files touched, tool calls, token usage on a shadow git branch (`entire/checkpoints/v1`). Keeps working branch clean.

Three-layer vision: git-compatible database → semantic reasoning layer → AI-native fleet management UI.

**Lesson for us:** Agent provenance is orthogonal and valuable. If we adopt checkpoints, we get debugging and review for free. Shadow branches don't pollute working branches.

---

### Hivemind MCP

**Source:** [hivemindai.dev](https://hivemindai.dev/) | [HN Show](https://news.ycombinator.com/item?id=47088912)

Append-only event log as MCP server. Advisory file locks with TTL. Semantic search via embeddings. ~10 MCP tools, <50ms query latency.

Creator reports advisory locking "eliminated most conflicts" in practice with capable models.

**Lesson for us:** Advisory file locks with TTL are lightweight and effective. Append-only event log preserves full coordination history (event sourcing pattern). Worth evaluating as the cross-session awareness layer.

---

### Agent-Recall (SQLite Knowledge Graph)

**Source:** [GitHub](https://github.com/mnardit/agent-recall) | [HN Show](https://news.ycombinator.com/item?id=47165499)

Extracted from a 30+ concurrent agent production system. Scope-chain isolation (global → org → project). Bitemporal SQLite. SessionStart hook injects AI briefings. Cache invalidation when one agent's writes affect another's scope.

**Lesson for us:** Structured facts > fuzzy recall. Scope chains prevent data leakage between projects. The briefing-at-session-start pattern mirrors our `cyrus-setup.sh`.

---

### Claude Code Agent Farm (Dicklesworthstone)

**Source:** [GitHub](https://github.com/Dicklesworthstone/claude_code_agent_farm)

JSON file-based coordination: `active_work_registry.json`, `completed_work_log.json`, `planned_work_queue.json`. Stale locks auto-cleaned after 2 hours. tmux orchestration.

Key finding: Opus-class models "simply smart enough to understand and reliably implement" prompt-level coordination protocols without code enforcement.

**Lesson for us:** File-based coordination works with capable models. Prompt-level protocols can be sufficient without hard enforcement — but our scope_guard.py deterministic backstop is still valuable.

---

## Academic Research

### CooperBench — Why Agents Fail as Teammates

**Source:** [arXiv:2601.13295](https://arxiv.org/html/2601.13295v2)

**The sobering finding:** Two coordinating agents achieve **HALF** the success rate of one agent doing both tasks. ~50% single-agent → ~25% with two agents.

**Three failure categories:**
1. **Expectation failures (42%)** — Agent ignores partner's communicated information
2. **Commitment failures (32%)** — Agent breaks promises about code changes
3. **Communication failures (26%)** — Ineffective messages, ignored questions, hallucinated facts

**The communication paradox:** Agents can coordinate on WHERE to edit (spatial) but fail at WHAT to build (semantic). Current LLMs lack theory-of-mind for partner state modeling.

**Lesson for us:** This is the strongest argument for the helm's approach — decompose into non-overlapping scopes and DON'T require agents to coordinate. Each agent works independently within its scope. The helm handles coordination, not the agents.

---

### AgentSpawn — Adaptive Dynamic Spawning

**Source:** [arXiv:2602.07072](https://arxiv.org/html/2602.07072)

Spawns specialized children reactively based on 5 complexity metrics (file interdependency, cyclomatic complexity, test failure cascade, context overflow, agent uncertainty). Lock-free optimistic concurrency with semantic merge.

**Results:** +97% vs single-agent on SWE-bench. Despite 54% more tokens and 92% more API calls, cost per successful completion is 9% LOWER due to higher completion rates.

**Semantic merge success rates:**
- Auto-merge (non-overlapping lines, same file): 100% success (15% of cases)
- LLM semantic merge (overlapping changes): 73% success (73% of cases)
- Escalation to parent: 0% automatic success (12% of cases)

**Lesson for us:** Reactive spawning based on complexity is interesting but complex. The key finding is that even with overlap, LLM-mediated semantic merge works 73% of the time — but our approach (prevent overlap via decomposition) is simpler and more reliable.

---

### Codified Context — Three-Tier Knowledge Architecture

**Source:** [arXiv:2602.20478](https://arxiv.org/html/2602.20478v1)

Based on 283 real development sessions. Three tiers:
- **Tier 1 (Hot, ~660 lines):** Always loaded. Standards, routing rules.
- **Tier 2 (Domain specialists, ~9,300 lines):** Embedded project knowledge, loaded per agent type.
- **Tier 3 (Cold, ~16,250 lines):** On-demand retrieval.

Knowledge-to-code ratio: 24.2%.

**Critical failure mode:** Specification staleness → silent failures. Stale specs cause syntactically correct code that conflicts with recent refactors.

**Lesson for us:** Our CLAUDE.md + .claude/reference/ + .claude/rules/ already implements this three-tier pattern. The staleness warning is important — plan files on main must stay current.

---

## Tournament / Evaluation Patterns

### Google AI Co-Scientist — The Gold Standard Tournament

**Source:** [arXiv:2502.18864](https://arxiv.org/abs/2502.18864)

Six specialized agents: Generation, Reflection, Ranking, Evolution, Proximity, Meta-review.

**Tournament mechanics:**
- Pairwise debates between candidates (not absolute scoring)
- **Elo rating system** produces ranked population
- Evolution agent generates mutations of top-Elo candidates → re-enters population
- Asynchronous compute allocation based on candidate promise
- Loop continues until compute budget exhausted

Validated with real biomedical discoveries (drug repurposing candidates, epigenetic targets).

**Lesson for us:** Pairwise comparison + Elo is more reliable than absolute scoring. The evolution operator (mutate top candidates) is interesting for iterating on plans. But this requires domain-specific automated evaluation — for code, that means test execution.

---

### Best-of-N Research Summary

| Technique | Improvement | Cost | Source |
|-----------|-------------|------|--------|
| AlphaCode 2 (1M samples + filter + cluster + score) | 2x solve rate | Extreme | DeepMind |
| Pairwise RM knockout tournament | 40-60% on hard problems | O(N log N) | arXiv:2501.13007 |
| Self-Certainty (reward-free BoN) | Matches RM at ~0 cost | Free | arXiv:2502.18581 |
| Majority-of-Bests | 2-11% over standard BoN | Negligible | arXiv:2511.18630 |

**Key finding:** Log-linear scaling — accuracy increases linearly as compute increases exponentially. The question is whether the quality gain justifies the cost for your specific use case.

---

### LLM-as-Judge for Code — What Works

**Source:** [AXIOM, arXiv:2512.20159](https://arxiv.org/html/2512.20159v1)

- **Pairwise comparison works:** "which of these two is better?" is more reliable than "rate this code 1-10"
- **Scalar scoring fails:** Up to 14% variance from position-order bias alone
- **Pessimism bias:** LLM judges systematically label correct code as defective
- **Stronger models, simpler prompts:** Capable judges work better with single-pass than multi-step workflows

**Lesson for us:** If we build judge agents, use pairwise comparison (tournament bracket), not absolute scoring. And pair with execution-based verification (tests pass? linter clean? contracts hold?).

---

### Multi-Agent Debate — Mostly Fails

**Source:** [ICLR 2025](https://d2jud02ci9yv69.cloudfront.net/2025-04-28-mad-159/blog/mad/) | [arXiv:2511.07784](https://arxiv.org/abs/2511.07784)

Most MAD frameworks fail to outperform simple Chain-of-Thought and Self-Consistency. The strongest predictor of debate success is **individual model accuracy** (coefficient 0.600, p<0.001). Structural parameters (debate order, rounds, confidence visibility) have negligible effect.

**Exception:** Multi-agent debate for improving judge reliability works (+4-8%). The pattern: multiple LLM judges evaluate, debate, update until stable.

**Lesson for us:** Don't build multi-agent debate for code generation. It doesn't help. If we use debate at all, use it for evaluation/judging, not generation.

---

### AlphaEvolve — Evolutionary Code Generation

**Source:** [arXiv:2506.13131](https://arxiv.org/abs/2506.13131) | [DeepMind blog](https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)

Population database + Gemini Flash (breadth) + Gemini Pro (depth) + automated evaluators. Production results: 0.7% Google data center resource recovery, 23% kernel speedup, 32.5% FlashAttention improvement.

**Lesson for us:** Evolutionary approaches work brilliantly when you have a measurable fitness function. For general software engineering, the fitness function is "tests pass + linter clean + contracts hold" — which we already have. Worth exploring for algorithm-heavy tasks, not for general feature development.

---

## Cross-Cutting Patterns

### What Wins for Conflict Prevention

| Mechanism | Used By | Verdict |
|-----------|---------|---------|
| Git worktrees (physical isolation) | Codex, Cursor, Cyrus | Most reliable. Our approach. |
| Pre-assigned file ownership (decomposition) | Claude Code Teams, Cursor | Works when decomposition is clean. Our helm's core job. |
| Advisory file locks (TTL) | Hivemind, Agent Farm | Effective with capable models. Lightweight. |
| Serial merge queue | Gas Town (Refinery) | More reliable than parallel merge. |
| Optimistic concurrency + semantic merge | AgentSpawn | 73% success, but adds complexity. |

**Our position:** We already have worktree isolation (Cyrus creates one per issue) + scope_guard.py (deterministic enforcement). The helm adds pre-assigned file ownership via decomposition. This is the strongest combination.

### What Wins for Communication

| Mechanism | Used By | Verdict |
|-----------|---------|---------|
| No communication (human-mediated) | Codex, our current system | Simplest. Works at 1-3 agents. |
| Shared task list + mailbox | Claude Code Teams | Good for single-machine. |
| Append-only event log | Hivemind | Best for cross-session awareness. |
| Git-backed state | Gas Town | Survives crashes. |
| JSON file registry | Agent Farm | Simple and effective. |

**Our position:** Start with no inter-agent communication (each agent works independently within its scope). The helm monitors via Linear API, not agent-to-agent messaging. Add event log later only if the "what changed since I started?" question becomes a real problem at scale.

### What Wins for Quality Gates

| Gate | Used By | Verdict |
|------|---------|---------|
| Execution-based (tests + linter) | AlphaCode 2, AlphaEvolve, our system | Most reliable. |
| Internal critique agent | Jules | Clean pattern for pre-PR review. |
| Pairwise tournament with RM | AI Co-Scientist | Best for evaluation, not generation. |
| Human plan approval | Cursor, Q Developer | Slowest but most reliable. |

**Our position:** We already have test + linter + import-linter as automated gates. Add pairwise comparison as the judge pattern if we build tournament ideation (Phase 4 of macro plan).

---

## Design Implications for Our Helm

### What the research validates in our existing plan:
1. **Decomposition-first approach** — Gas Town, Cursor, Claude Code Teams all converge on this
2. **Worktree isolation** — Universal pattern for physical conflict prevention
3. **Scope enforcement (scope_guard.py)** — Deterministic backstop that Agent Farm's prompt-level approach lacks
4. **Serial merge ordering** — Gas Town's Refinery pattern validates our `/helm merge` dependency DAG approach
5. **Human-in-the-loop at merge time** — No production system auto-merges; we shouldn't either

### What the research suggests we should change or add:
1. **Skip inter-agent communication initially** — CooperBench shows coordination makes things worse. Let the helm coordinate; agents work independently.
2. **Consider the Gas Town "Witness" role** — A lightweight monitor that detects stuck agents and nudges them. Our GH Action monitor partially does this but could be more proactive.
3. **Provenance/checkpoints (Entire.io pattern)** — Worth adopting for agent session debugging. Shadow branch captures are non-invasive.
4. **Don't build multi-agent debate** — Research shows it doesn't help for code generation. If we build tournament ideation, use execution-based evaluation (tests pass?), not LLM debate.
5. **Expect ~$100/hr at full scale (20-30 agents)** — Gas Town's cost data is a useful reference point. Need token cost tracking.

### What the research says about tournament ideation (Phase 4):
1. **Only works with automated evaluators** — Test execution, not LLM judges, for code quality
2. **Pairwise comparison > absolute scoring** — AXIOM research is clear on this
3. **Log-linear scaling** — Need to determine where on the curve we are (is 3x approaches worth 3x cost for our tasks?)
4. **Best-of-N with execution filter** — The simplest effective pattern: generate 3 implementations, run tests, pick the one that passes with best test coverage
5. **Don't invest heavily until Phase 4 controlled comparison** — This is a research bet, not a certainty

---

## References

### Production Systems
- [Gas Town](https://github.com/steveyegge/gastown) — 20-30 parallel agents, $100/hr
- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Entire.io CLI](https://github.com/entireio/cli) — Agent provenance/checkpoints
- [OpenAI Codex](https://openai.com/index/introducing-codex/)
- [Google Jules](https://blog.google/technology/google-labs/jules/)
- [Cursor 2.0](https://cursor.com/changelog/2-0)
- [Hivemind MCP](https://hivemindai.dev/)
- [Agent-Recall](https://github.com/mnardit/agent-recall)
- [Claude Code Agent Farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)

### Academic Papers
- [CooperBench — arXiv:2601.13295](https://arxiv.org/html/2601.13295v2) — Two agents = half the success
- [AgentSpawn — arXiv:2602.07072](https://arxiv.org/html/2602.07072) — +97% on SWE-bench
- [Codified Context — arXiv:2602.20478](https://arxiv.org/html/2602.20478v1) — Three-tier knowledge
- [AI Co-Scientist — arXiv:2502.18864](https://arxiv.org/abs/2502.18864) — Tournament with Elo
- [AXIOM — arXiv:2512.20159](https://arxiv.org/html/2512.20159v1) — LLM-as-Judge for code
- [Pairwise RM — arXiv:2501.13007](https://arxiv.org/html/2501.13007v1) — Knockout tournament
- [MAD ICLR 2025](https://d2jud02ci9yv69.cloudfront.net/2025-04-28-mad-159/blog/mad/) — Debate mostly fails
- [AlphaEvolve — arXiv:2506.13131](https://arxiv.org/abs/2506.13131) — Evolutionary code generation
- [AlphaCode 2](https://storage.googleapis.com/deepmind-media/AlphaCode2/AlphaCode2_Tech_Report.pdf) — 1M samples + filter + score

### Analysis / Synthesis
- [Anthropic Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Anthropic 2026 Agentic Coding Trends](https://resources.anthropic.com/2026-agentic-coding-trends-report)
- [AI Coding Agents in 2026: Coherence Through Orchestration](https://mikemason.ca/writing/ai-coding-agents-jan-2026/)
- [What 371 Git Worktrees Taught Me](https://levelup.gitconnected.com/what-371-git-worktrees-taught-me-about-multi-agent-ai-36d4d61acfb5)
- [Maggie Appleton's Gas Town Synthesis](https://maggieappleton.com/gastown)
