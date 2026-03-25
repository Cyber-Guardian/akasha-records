---
topic: Helm Orchestration
status: active
touched: 2026-03-04
related:
  - thoughts/shared/plans/2026-03-04-helm-full-pipeline.md
  - thoughts/shared/briefs/2026-03-04-helm-extensions-post-execute.md
  - thoughts/shared/research/2026-03-03-helm-mvp-weaknesses.md
---

# Helm Orchestration

## Current State
Helm V1 execution layer is complete. 9 commands: `decompose`, `status`, `merge`, `retry`, `recover`, `validate`, `list`, `cancel`, `review`. 17/19 MVP weaknesses resolved. W18 (`helm promote`) and W11 (GH Action JavaScript tests) remain open.

Helm is a swarm orchestration harness — it sits above the agent runtime, coordinating concurrent agents at the macro level. Deterministic Python state machine; Claude is a callable subprocess only for repairs.

Linear project: "Helm Orchestrator"

## Key Decisions
- Parent issue creation by default in `helm execute`, `--issue ENG-XXXX` override for existing
- 60s poll interval, max 2 retries per failure type per sub-issue
- `HelmState.EXECUTING` prevents premature `sync_manifest` auto-completion
- `DELEGATED` local status NOT used — sync overwrites from Linear immediately
- CWD gotcha (W6): `uv run --project tools/helm` changes CWD — always use absolute paths for `--plan`

## Open Questions
- W1: Dual source of truth (local manifest vs Linear) needs shaping
- W11: GH Action JavaScript is untested
- `helm promote` and `helm execute <plan-file>` are next (full pipeline plan)

## Artifacts
- `tools/helm/src/helm/` — all CLI modules
- `.github/workflows/helm-monitor.yml` — monitoring workflow
