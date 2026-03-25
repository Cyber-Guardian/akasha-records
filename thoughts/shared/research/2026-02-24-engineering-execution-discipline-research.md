# Research: Engineering Execution Discipline for Human + Agent Teams

**Date:** 2026-02-24
**Linear Issue:** ENG-2128 — Define execution discipline for autonomous agents
**Status:** Research complete, ready for shaping into sub-issues

## Sources

### External Research
- [Branching Patterns — Martin Fowler](https://martinfowler.com/articles/branching-patterns.html)
- [Continuous Integration — Martin Fowler](https://martinfowler.com/articles/continuousIntegration.html)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [Build Pipelines, Deployment, and Immutable Artifacts — Brunton-Spall](https://medium.com/@bruntonspall/build-pipelines-deployment-and-immutable-artifacts-48ae926178a5)
- [Semantic Pull Request — GitHub Action](https://github.com/marketplace/actions/semantic-pull-request)
- [python-semantic-release — Monorepo Guide](https://python-semantic-release.readthedocs.io/en/latest/configuration/configuration-guides/monorepos.html)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)
- [microservice-api-patterns.org — SemVer for APIs](https://microservice-api-patterns.org/patterns/evolution/SemanticVersioning)

### Internal Context
- User diagrams: Path to Deployment, Environment Tiers, Strategy Comparison, Dev vs Ephemeral, Rebase Strategies
- Existing codebase CI/CD: `.github/workflows/`, `Makefile`, `infrastructure/`
- Brief: `memory-bank/thoughts/shared/briefs/2026-02-24-linear-agent-orchestration-system.md`
- Decision: `memory-bank/thoughts/shared/decisions/2026-02-19-linear-workspace-restructure.md`

---

## Part 1: Deployment Strategy — Strategy 1 (Approval Per Release)

### Why Strategy 1

Strategy 1 (approval required per release) is the right choice for FileScience because:

1. **High PR volume**: Multiple humans + many agents generating PRs simultaneously. Strategy 1 allows faster merging into mainline because mainline doesn't need to always be releasable — the Testing environment acts as a buffer.
2. **Decouples merge from deploy**: Engineers/agents can merge frequently without triggering production deployments. Releases are explicit, intentional acts.
3. **Batch efficiency**: When many PRs land in a short window, they're tested together in Testing env rather than each one triggering a full deployment pipeline independently.
4. **Aligns with Fowler's CI principle**: "Integrate at least daily" — Strategy 1 has the lowest friction for frequent integration.

### Strategy 1 Pipeline Flow

```
Feature Branch → PR → Merge to Main
                         │
                         ▼
              Deploy to Testing ENV
              Run short e2e tests
                         │
                         ▼ (on pre-release creation)
              Deploy to Staging ENV
              Run e2e tests
              Run quality gates / regression tests
                         │
                         ▼ (if tests pass + manual approval → create release)
              Deploy to Production
              Post-deployment smoke tests / synthetic canaries
                         │
                         ▼ (if tests fail)
              Branch from release tag, fix, PR back to main
              Create new pre-release revision
```

### Pre-Release Versioning

When it's deployment time, a pre-release is created:
- Form: `{project}-v{MAJOR}.{MINOR}.{PATCH}-rc.{N}` (e.g., `discover-v1.2.0-rc.1`)
- Pre-release creation triggers: deploy to Staging + full test suite
- If staging tests pass + manual approval → create full release (`discover-v1.2.0`)
- Release creation triggers: deploy to Production
- If staging tests fail → fix on branch, merge to main, create new pre-release revision (`discover-v1.2.0-rc.2`)

### Comparison to Strategy 2

| Aspect | Strategy 1 (chosen) | Strategy 2 |
|--------|---------------------|------------|
| Merge friction | Low — merge freely | High — merge = deploy trigger |
| Mainline state | Not always releasable | Always releasable |
| PR volume handling | Efficient — batch in Testing | Bottleneck — each merge triggers pipeline |
| Environments needed | Testing + Staging + Prod | Staging + Prod |
| Release control | Explicit release creation | Every merge is a release |
| Team fit | Better for large/agent teams | Better for solo/small teams |
| Mainline head = deployed? | No (latest release is) | Yes |

---

## Part 2: Environment Strategy

### Five Environment Tiers

| Tier | Purpose | In deploy pipeline? | Infrastructure | Lifetime |
|------|---------|---------------------|---------------|----------|
| **Production** | Live customer traffic | Yes — on release creation | Dedicated AWS account | Permanent |
| **Staging** | Pre-prod quality gates, regression testing | Yes — on pre-release creation | Shared AWS account (us-west-2) | Permanent |
| **Testing** | Integration testing, stability buffer | Yes — on merge to main | Shared AWS account (us-east-1) | Permanent |
| **Development** | Shared sandbox for manual testing | No — manual deploy | Shared AWS account (us-east-1) | Permanent |
| **Ephemeral** | Isolated environments for IaC/breaking changes | No — on-demand | Shared AWS account (dynamic) | Temporary |

### Service-Specific Routing

Not all services need all tiers. The Testing env exists primarily for services with inter-service dependencies where a bad deploy can cascade.

| Service | Skip Testing? | Rationale |
|---------|--------------|-----------|
| Dashboard / Landing Site | Yes → direct to Staging | Frontend, no backend dependencies |
| Discover (Lambda) | No → deploy to Testing first | Has DynamoDB, Valkey, Graph API dependencies |
| Queue Processor (Lambda) | No → deploy to Testing first | Stream processing, failure modes matter |
| Entity Discovery Trigger | No → deploy to Testing first | Upstream of Queue Processor |
| Triage Bot (Lambda) | Yes → direct to Staging | Independent, external integrations only |

### Dev vs. Ephemeral Decision

```
Dev environment if:
  - NOT modifying IaC significantly
  - Smaller, service-level changes
  - Want pre-built infrastructure (fast feedback)

Ephemeral environment if:
  - Making infrastructure-breaking changes
  - Larger service changes that might interfere with other dev work
  - Testing IaC changes in isolation
  - Need guaranteed clean state
```

### Current State vs. Target

| Environment | Today | Target |
|-------------|-------|--------|
| Production | Does not exist | Dedicated account, us-east-1 |
| Staging | Does not exist | Shared account, us-west-2 |
| Testing | Does not exist | Shared account, us-east-1 (new) |
| Development | Exists (us-east-1, `dev`) | Keep as-is |
| Ephemeral | Does not exist | On-demand via Terragrunt |

Testing environment is a **later** item within this issue family — not Day 1.

---

## Part 3: Branching & Commit Conventions

### Branching Strategy

Fowler's key insight: **integration frequency is the single most important variable**. Strategy 1 already optimizes for fast merging. The branching conventions should minimize friction.

**Branch naming** (already partially established):
```
{type}/ENG-{ID}-{short-description}
```
Types: `feat`, `fix`, `refactor`, `chore`, `docs`

**Branch lifecycle**:
1. Create from `main`
2. Work (human or agent commits)
3. Frequently rebase/merge from `main` (Fowler: "no more than a day's work unintegrated")
4. PR with semantic title
5. Squash merge → main
6. Delete branch

### Conventional Commits

Format: `type(scope): description`

```
feat(discover): add Teams connector support
fix(valkey-queue-processor): handle empty batch records
refactor(cloud-api): extract session retry logic
chore(ci): add semantic PR validation
docs(memory-bank): update deployment strategy research
test(throttling): add concurrency gate edge cases
```

**Breaking changes** use `!`:
```
feat(discover)!: remove legacy traversal mode
```

**Allowed types**: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`

**Scopes** (map to Polylith bricks + infrastructure):
- Project scopes: `discover`, `valkey-queue-processor`, `entity-discovery-trigger`
- Component scopes: `models`, `cloud-api`, `throttling`, `dynamodb`, `config`, `domain-backup`, `lambda-utils`, `suppression`, `bedrock`, `glide-sync`
- Base scopes: `discover-base`, `valkey-processor-base`
- Infra scopes: `ci`, `terraform`, `deps`

### PR Title Validation

Enforce via `amannn/action-semantic-pull-request` GitHub Action:
- PR title must follow conventional commit format
- Repo configured for squash merge with "Default to PR title"
- PR title becomes the commit message on mainline
- This means every mainline commit is a conventional commit

### Rebase Strategy: Selective Squash

Before merge, the developer/agent should clean up the branch:

**Preferred approach** — Squash non-passing commits into passing ones:
```
Before rebase:
  Commit A (red - failing)    ─┐
  Commit B (green - passing)   │ squash A into B
  Commit C (red - failing)    ─┘─ Atomic commit 1
  Commit D (green - passing)  ─── Atomic commit 2

After rebase:
  Atomic commit 1: feat(discover): add delta sync support
  Atomic commit 2: test(discover): add delta sync test coverage
```

**Why not full squash** (squash everything into one commit):
- Loses meaningful commit granularity
- A single commit doing 5 different things is harder to review and revert

**Why not no rebase** (merge all commits as-is):
- Noisy history with WIP/broken commits
- Harder to bisect

**Practical note**: For squash merge PRs, the individual commits don't land on mainline anyway — only the PR title does. The selective rebase is primarily valuable for the PR review experience (reviewable commit-by-commit) and for reverting specific changes within a feature if needed pre-merge.

---

## Part 4: Versioning Strategy

### SemVer Applies to Applications Too

SemVer was designed for libraries with "public APIs," but the concept maps cleanly to deployed services. For a service, the "public API" is its observable contract:

| What constitutes the API | Breaking change (MAJOR) | Additive (MINOR) | Fix (PATCH) |
|--------------------------|------------------------|-------------------|-------------|
| HTTP endpoints | Remove endpoint | Add new endpoint | Fix endpoint bug |
| Request/response schemas | Remove/rename field | Add optional field | Fix field validation |
| Event schemas (DynamoDB stream, SQS) | Change field type | Add optional field | Fix handler bug |
| Environment variables | Remove required var | Add optional var | Fix var parsing |
| IAM permissions | Change required perms | — | — |
| Lambda behavior (side effects) | Change what gets written | New side effect (additive) | Fix existing behavior |

### Hybrid Approach: SemVer Tags + Content Hash Deployment

The current SHA256 content-addressed deployment is already correct and production-grade. SemVer adds a **communication layer** on top:

```
Deployment artifact: s3://bucket/discover/sha256-abc123/discover.zip  (immutable, content-addressed)
SemVer tag:          discover-v1.2.0                                   (human-readable, changelog)
GitHub Release:      discover-v1.2.0 with auto-generated release notes
```

The Terraform/Terragrunt `artifact-versions.json` continues to reference SHA256 keys. SemVer tags are metadata, not deployment identifiers. This gives you:
- Immutable, reproducible deployments (SHA256)
- Human-readable release history (SemVer + changelog)
- Automated version bumps from conventional commits (python-semantic-release)

### Independent Versioning Per Project

Each Polylith project gets its own version lifecycle:

```
projects/discover/pyproject.toml           → version = "1.2.0"    tag: discover-v1.2.0
projects/valkey-queue-processor/pyproject.toml → version = "0.8.1" tag: vqp-v0.8.1
projects/entity-discovery-trigger/pyproject.toml → version = "0.3.0" tag: edt-v0.3.0
```

**Shared component changes** (e.g., `components/filescience/models/`) trigger version bumps on all dependent projects via `path_filters` in python-semantic-release config.

### Tooling: python-semantic-release

- Reads conventional commits since last tag
- Determines bump type (MAJOR/MINOR/PATCH) per project
- Updates `pyproject.toml` version
- Creates git tag (`discover-v1.2.0`)
- Generates CHANGELOG.md
- Creates GitHub Release with auto-generated notes
- Monorepo support via `directory` input + `path_filters`

### Pre-Release Flow (Maps to Strategy 1)

```
1. Commits land on main with conventional commit messages
2. python-semantic-release runs on main, determines next version
3. Pre-release created: discover-v1.2.0-rc.1
   → Triggers: deploy to Staging, run full test suite
4. If tests pass + manual approval:
   → Release created: discover-v1.2.0
   → Triggers: deploy to Production
5. If tests fail:
   → Fix merged to main
   → New pre-release: discover-v1.2.0-rc.2
   → Repeat from step 3
```

---

## Part 5: CI Quality Gates

### PR Approval Steps (on every PR)

```
PR Created/Updated
  ├── Semantic PR title validation (amannn/action-semantic-pull-request)
  ├── Ruff lint + format check
  ├── import-linter architectural contracts (7 contracts)
  ├── ty type checking
  ├── pytest (unit tests, fast)
  ├── Security scans (TBD — pip-audit, detect-secrets)
  └── Static analysis (TBD — complexity gates)
```

### On Merge to Main

```
Merge to main
  ├── Build artifacts (make build)
  ├── Upload SHA256 artifacts (make upload-sha256)
  ├── Deploy to Testing ENV
  ├── Run short e2e tests
  └── Update artifact-versions.json
```

### On Pre-Release Creation

```
Pre-release created (discover-v1.2.0-rc.1)
  ├── Deploy to Staging ENV
  ├── Run e2e tests (full suite)
  ├── Run quality gates / regression tests
  └── If pass → eligible for release approval
```

### On Release Creation

```
Release created (discover-v1.2.0)
  ├── Deploy to Production
  ├── Post-deployment smoke tests
  ├── Synthetic canary monitoring
  └── If fail → alert, rollback to previous release artifact
```

---

## Part 6: Mapping to Agentic Context

This is where the engineering discipline connects to the agent orchestration system (ENG-2128 original scope + ENG-2129/ENG-2130).

### What's Different When Agents Make PRs

Traditional CI/CD assumes human developers who:
- Understand the broader project context
- Can assess whether their change conflicts with others' work
- Know when to stop and ask questions
- Review their own changes before submitting

Agents don't have these properties by default. The execution discipline must **encode these expectations into structure**.

### Agent-Specific Constraints on Top of General Discipline

#### 1. Conventional Commits Enable Cross-Agent Communication

The conventional commit format isn't just for changelogs — it's a **communication protocol between agents**. When Agent A inspects Agent B's recent commits on a sibling branch:

```
feat(discover): add Teams connector support         → Agent A knows: new cloud service added
fix(throttling): handle race in gate release         → Agent A knows: throttling behavior changed
refactor(cloud-api)!: extract session retry logic    → Agent A knows: breaking change to shared component
```

The `!` for breaking changes is critical — an agent working on code that depends on `cloud-api` can detect that a sibling has made a breaking change and flag a potential conflict.

**Scopes are the key primitive**: They tell agents which Polylith brick was modified without reading the diff. Combined with import-linter contracts (which define dependency boundaries), an agent can reason:
- "My work depends on `models`"
- "Agent B just committed `feat(models): add FileType enum`"
- "I should pull main and check if my code still compiles"

#### 2. Atomic Commits Per Plan Phase

Autonomous agents should commit after each plan phase, not just at the end:

```
feat(ENG-2128): phase 1 — add semantic PR validation workflow
feat(ENG-2128): phase 2 — configure squash merge and PR title defaults
feat(ENG-2128): phase 3 — add python-semantic-release config per project
```

This serves three purposes:
- **Checkpointing**: If an agent fails mid-execution, the completed phases are preserved
- **Cross-agent visibility**: The orchestrator (ENG-2130) can see exactly where each agent is in its plan
- **Reviewability**: Each phase is independently reviewable and revertable

#### 3. Per-Branch Memory State

Current `memory-bank/durable/01-active/` is per-repo, not per-branch. When multiple agents work on different feature branches:

```
Agent A (feat/ENG-2128-execution-discipline):
  memory-bank/durable/01-active/current_work.md → "Working on commit conventions"

Agent B (feat/ENG-2095-discover-concurrency):
  memory-bank/durable/01-active/current_work.md → "Working on throttle gates"
```

These conflict on merge. Design options:

**Option A — Branch-scoped active files**:
- `memory-bank/durable/01-active/` → branch-scoped (each branch gets its own copy)
- `memory-bank/durable/00-core/` → shared (project_brief, product_context don't change per-branch)
- On merge: discard branch-scoped files, human updates main's active files
- Pro: Clean isolation. Con: Active files may drift from reality on main.

**Option B — Agent-scoped files within active**:
- `memory-bank/durable/01-active/agents/ENG-2128.md` → agent's working context
- `memory-bank/durable/01-active/current_work.md` → human-maintained summary of all active work
- On merge: agent file gets archived to `memory-bank/thoughts/shared/`
- Pro: No merge conflicts. Con: current_work.md needs manual maintenance.

**Option C — Don't put agent context in durable memory at all**:
- Agents use Linear issue comments for their working context
- `memory-bank/durable/01-active/` stays human-maintained
- Cross-agent visibility comes from Linear + git, not memory-bank
- Pro: Simplest, no merge conflicts, Linear is already the source of truth. Con: Agents lose memory-bank context on session start.

**Recommendation: Option C** — it aligns with the Linear-as-orchestrator approach. Agents read durable memory for stable context (project_brief, product_context) but track their own working state in Linear comments. The alignment checker (ENG-2129) monitors Linear + git, not memory-bank.

#### 4. Autonomous Qualification Criteria

Not every issue should be autonomous. During planning, tag issues as `autonomous` or `co-dev`:

| Criterion | Must be TRUE for `autonomous` |
|-----------|------------------------------|
| Well-scoped plan exists | Plan in `memory-bank/thoughts/shared/plans/` with numbered phases |
| Acceptance criteria are concrete and testable | Each AC maps to a test assertion or verifiable state |
| No open design questions | All "how" decisions are made; only "do it" remains |
| No cross-cutting concerns with active work | import-linter contracts + Linear dependency graph show no conflicts |
| Estimated at ≤3 plan phases | Longer plans need human checkpoints |
| Changes don't affect shared components | Or: shared component changes are additive-only (MINOR/PATCH) |

If any criterion is FALSE → `co-dev` label → human stays in the loop.

#### 5. Agent PR Conventions

Agent-created PRs should be identifiable and include extra context:

```markdown
## Summary
[What this PR does and why]

Resolves ENG-XXXX

## Plan Phase
Phase N of [plan-link] — [phase description]

## Agent Context
- Execution mode: autonomous
- Commits: N atomic commits (1 per plan phase)
- Shared components modified: [list or "none"]

## Changes
- [Key change 1]
- [Key change 2]

## Test plan
- [ ] [Verification step 1]
- [ ] [Verification step 2]
```

The "Shared components modified" field is critical for the alignment checker — it's the trigger for cross-agent conflict detection.

#### 6. Semantic PR Validation for Agents

The `amannn/action-semantic-pull-request` action works identically for human and agent PRs. But agents need **additional** validation:

- PR title references a Linear issue ID (enforceable via `subjectPattern`)
- PR body includes plan phase reference (enforceable via GH Actions body check)
- All commits reference the same Linear issue
- No commits to `memory-bank/durable/` (agents shouldn't modify shared state — per Option C above)

### How the Alignment Checker (ENG-2129) Uses This Discipline

The execution discipline defined here is the **input contract** for the alignment checker:

```
Alignment Checker reads:
  ├── Conventional commits on feature branches → what each agent is doing
  ├── Scopes in commits → which Polylith bricks are affected
  ├── Linear issue status → where each agent is in its plan
  ├── import-linter contracts → which bricks depend on which
  ├── PR metadata → shared components modified, plan phase
  └── Git branch names → which issue each branch belongs to

Alignment Checker flags:
  ├── Two agents modifying the same Polylith brick
  ├── Agent modifying a shared component (models, cloud-api) without marking it
  ├── Agent's commits drifting from its plan's acceptance criteria
  ├── Agent stuck (no commits for N hours while issue is In Progress)
  └── Conflicting changes detected via git merge --no-commit --no-ff dry run
```

Without the commit conventions, branch naming, and PR metadata defined here, the alignment checker has no structured input to reason about.

### How the Orchestrator Agent (ENG-2130) Extends This

The orchestrator graduates from flag-only to taking action:

```
Orchestrator can:
  ├── Reorder issue priorities based on dependency DAG
  ├── Block a PR if alignment check fails (required status check)
  ├── Create follow-up issues when a breaking change is detected
  ├── Post daily "state of the swarm" summaries to Slack
  └── Recommend plan adjustments based on what other agents discovered
```

This is only possible because the execution discipline creates a **machine-readable protocol** for agent activity. Conventional commits + Linear issues + structured PRs = a complete activity log that another agent can consume.

---

## Part 7: Key Learnings & Principles

### From Fowler (Branching Patterns + CI)

1. **Integration frequency is the #1 variable** — Strategy 1 optimizes for this
2. **Healthy branch = every commit passes tests** — enforced by PR checks + selective rebase
3. **Never use environment branches** — externalize config, same artifact everywhere
4. **Fix broken builds immediately** — CI failure → Linear pipeline already handles this
5. **10-minute commit build target** — current lint + test suite should stay under this
6. **Feature flags for incomplete work** — agents should merge partial features behind flags

### From Brunton-Spall (Immutable Artifacts)

7. **Build once, deploy everywhere** — already done with SHA256 content-addressed artifacts
8. **Same deployment scripts for all environments** — Terragrunt handles this
9. **Never re-fetch dependencies at deploy time** — Lambda zips bundle everything
10. **Track what has been tested** — pre-release tags serve as "tested" markers

### From SemVer Research

11. **SemVer applies to services** — redefine "public API" as observable contract (endpoints, schemas, env vars, IAM)
12. **Content hash for deployment, SemVer for communication** — hybrid approach
13. **Independent versioning per project in monorepo** — `python-semantic-release` with `path_filters`
14. **Shared component changes cascade** — a change to `models/` bumps all dependent projects

### For the Agentic Context Specifically

15. **Conventional commits are an inter-agent communication protocol** — not just for changelogs
16. **Scopes + import-linter contracts = machine-readable dependency graph** — agents can reason about conflicts
17. **Atomic commits per plan phase = progress telemetry** — the orchestrator sees where each agent is
18. **Agent state lives in Linear, not memory-bank** — avoids merge conflicts, aligns with Linear-as-orchestrator
19. **Autonomous qualification criteria are a planning-phase gate** — prevents premature autonomy
20. **Structured PR metadata is the alignment checker's input contract** — without it, the checker is blind

---

## Proposed Sub-Issue Decomposition

ENG-2128 should be **re-titled** to "Define engineering execution discipline" and broken into sub-issues:

### Sub-issue 1: Branching & Commit Conventions (foundational, implement first)
- Document Strategy 1 as official deployment strategy
- Add `amannn/action-semantic-pull-request` GH Action
- Configure repo: squash merge, PR title → commit message
- Define allowed types + scopes
- Document selective rebase strategy
- **Deliverable**: Working PR validation + conventions doc

### Sub-issue 2: Versioning & Release Automation
- Add `python-semantic-release` per project
- Configure `path_filters` for shared component cascading
- Set up pre-release flow (rc tags → staging, release → prod)
- Integrate with existing `artifact-versions.json` pipeline
- **Deliverable**: Automated version bumps + GitHub Releases

### Sub-issue 3: Environment Strategy & Deployment Pipeline (later, design-heavy)
- Document 5-tier environment hierarchy
- Define service-specific routing (skip Testing or not)
- Design auto-deploy flow on merge/pre-release/release
- Implement GH Actions deployment workflows
- **Deliverable**: Architecture doc + initial Testing/Staging/Prod pipeline

### Sub-issue 4: Agent Execution Discipline (the original ENG-2128 core)
- Define autonomous qualification criteria
- Document agent commit conventions (extending sub-issue 1)
- Define agent PR template with shared-component metadata
- Implement memory strategy (Option C: Linear-based, not memory-bank)
- Add Linear-issue-reference validation to PR checks
- **Deliverable**: Updated CLAUDE.md + agent execution conventions

### Sub-issue 5: CI Quality Gates (extends existing CI)
- Add security scanning (pip-audit, detect-secrets)
- Define e2e test strategy per environment
- Add staging promotion gates
- Add production smoke test / canary framework
- **Deliverable**: Updated GH Actions workflows

### Dependency Order
```
Sub-issue 1 (Branching/Commits)
  └── Sub-issue 2 (Versioning) — needs conventional commits
  └── Sub-issue 4 (Agent Discipline) — extends commit conventions
       └── feeds into ENG-2129 (Alignment Checker)
Sub-issue 3 (Environments) — independent, later
Sub-issue 5 (Quality Gates) — independent, later
```
