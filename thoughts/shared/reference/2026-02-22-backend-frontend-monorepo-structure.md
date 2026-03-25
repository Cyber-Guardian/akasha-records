# Backend + Frontend Monorepo Structure Research (FileScience)

**Date:** 2026-02-22  
**Scope:** Determine the best way to include a product dashboard frontend in the existing FileScience backend monorepo without degrading backend velocity.

---

## Current Repo Reality (constraints we must preserve)

- Repo is explicitly a **backend Polylith monorepo** (`components/`, `bases/`, `projects/`) with Python-first tooling (`uv`, `ruff`, `ty`, import-linter).
- Architectural boundaries are already enforced for Python modules via import-linter contracts.
- There is precedent for isolated non-core subtrees (`tools/triage-bot`) with local tooling and deployment flow.
- Product guardrails: this repo is not for marketing/landing content; it is for product/backend systems.

**Implication:** Frontend can be included, but should be isolated from Polylith internals and from backend build/lint pipelines.

---

## External Research Findings

### Turborepo (Context7: `/vercel/turborepo`)

Strong guidance for:
- `apps/*` for deployable apps and `packages/*` for shared libraries/config packages.
- Task graph orchestration via `turbo.json` (`dependsOn`, `outputs`, cache control, persistent `dev` tasks).
- CI acceleration through deterministic task caching and consistent root scripts.

Best fit: JS/TS app orchestration and cacheable pipelines when frontend footprint grows.

### Nx (Context7: `/websites/nx_dev`)

Strong guidance for:
- Explicit architectural boundaries using project tags and enforced dependency constraints.
- "Affected" CI workflows to run lint/test/build only for impacted projects.
- Monorepo governance at scale (especially with many frontend/backend projects).

Best fit: large multi-team repo where governance and dependency policy become a primary concern.

### pnpm Workspaces (Context7: `/websites/pnpm_io`)

Strong guidance for:
- Workspace package discovery via `pnpm-workspace.yaml` include/exclude globs.
- Targeted commands with `--filter` selectors for package-level operations.
- Efficient dependency installation and reliable lockfile behavior.

Best fit: baseline workspace foundation for frontend packages/apps.

---

## Options for FileScience

### Option A: Separate frontend repo

**Pros**
- Zero coupling with backend workflows/tooling.
- Independent release and CI policy.

**Cons**
- Harder API contract synchronization.
- More cross-repo coordination overhead for product UI + API changes.

### Option B: Keep frontend in this repo, but isolate it as a sub-workspace (**recommended now**)

**Pros**
- Enables atomic PRs across API + dashboard when needed.
- Preserves current Python/Polylith flow by isolating Node/TS concerns.
- Incremental adoption with low disruption.

**Cons**
- Still introduces a second toolchain.
- Needs clear CI path filters to avoid unnecessary full-repo jobs.

### Option C: Full unified orchestrator at root (Nx/Turbo across most tasks)

**Pros**
- One task graph and stronger cross-project governance.
- Better long-term scaling if frontend surface area grows rapidly.

**Cons**
- High migration and operational overhead now.
- Risk of destabilizing current mature backend workflows.

---

## Recommended Structure for This Repo

Use a **hybrid polyglot monorepo**:

```text
filescience/
├── components/                 # existing Python Polylith bricks
├── bases/                      # existing Python Polylith bases
├── projects/                   # existing Python deployable units
├── tools/                      # existing operational tools
├── frontend/                   # new isolated JS/TS workspace root
│   ├── apps/
│   │   └── dashboard/          # product UI (in scope)
│   ├── packages/
│   │   ├── ui/                 # shared UI components (optional, later)
│   │   ├── config-eslint/      # shared frontend lint config
│   │   └── config-ts/          # shared frontend tsconfig package
│   ├── package.json
│   ├── pnpm-workspace.yaml
│   ├── pnpm-lock.yaml
│   └── turbo.json              # optional initially; recommended as frontend grows
├── contracts/
│   └── openapi/                # backend API contract used by dashboard/client generation
└── Makefile                    # keeps backend commands; add namespaced frontend targets only
```

Why `frontend/` instead of root-level `apps/` right now:
- Reduces collision with existing Python conventions and mental model.
- Keeps root clean while still staying in one repository.
- Allows future promotion to root-level apps/packages if frontend expands significantly.

---

## Boundary Rules (critical for maintainability)

1. **No direct imports from frontend into Python source trees** (`components/`, `bases/`, `projects`) and vice versa.
2. Cross-stack integration occurs through **versioned contracts** (`contracts/openapi`) and generated clients.
3. Frontend toolchain/config stays in `frontend/`; backend toolchain/config stays at repo root.
4. CI uses **path-based triggers** so backend-only changes do not run frontend jobs and frontend-only changes do not run full backend jobs.
5. Keep marketing/landing site outside this repo unless a future decision explicitly changes the repo charter.

---

## CI/CD Model

Minimum viable split:
- **Backend job** (existing): run current Python checks/tests for changes in backend/infrastructure paths.
- **Frontend job** (new): run `pnpm install` + `pnpm --filter dashboard lint test build` for `frontend/**` changes.
- **Contract job** (new): when `contracts/openapi/**` changes, validate schema and regenerate/check frontend API client.

As frontend grows:
- Add Turborepo task caching in `frontend/turbo.json`.
- Optionally adopt "affected" semantics (Nx-style) if project count and CI time justify it.

---

## Implementation Sequence (low-risk)

1. Create `frontend/` workspace with one app: `apps/dashboard`.
2. Add path-scoped CI job for frontend.
3. Define API contract flow (`contracts/openapi`) and client-generation rule.
4. Add frontend lint/type/test/build gates.
5. Introduce Turborepo only when there are 2+ apps/packages or CI pain appears.
6. Reassess root-level orchestrator decision once frontend team/project count increases.

---

## Decision Framework

Choose **Option B** now if:
- Dashboard is a product surface tightly coupled to backend behavior.
- Team benefits from atomic cross-stack PRs.
- You want to preserve backend velocity while adding frontend safely.

Escalate toward **Option C** only when:
- Frontend has multiple apps/packages and CI scale pain becomes visible.
- You need stronger dependency-governance primitives across many teams.

