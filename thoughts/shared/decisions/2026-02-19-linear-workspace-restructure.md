# Decision: Linear Workspace Restructure

**Date:** 2026-02-19
**Status:** IMPLEMENTED
**Participants:** jordan

---

## Context

Linear was not functioning as source of truth. The 2026 roadmap lived outside Linear. Platform v2.0 was a monolithic project with minimal issue coverage. No initiatives existed. Cycles were overloaded (78 issues/sprint). Issue quality was inconsistent.

## Decision

### 1. Initiative-based hierarchy

Created 5 initiatives mapping to 2026 strategic themes:

| Initiative | Status | Target | Scope |
|-----------|--------|--------|-------|
| **V2 Platform** | Active | 2026-12-31 | Discover, Transfer, Export, Bedrock/CloudBridge, Infrastructure, Inplace Recovery |
| **Clio Launch** | Active | 2026-06-30 | Integration expansion, recovery, marketing, partnerships |
| **Dashboard Evolution** | Planned | 2026-12-31 | v1.1 facelift → v1.2 features → V2 reskin |
| **Landing Site & Growth** | Active | 2026-12-31 | Site features, marketing, ads, content, visibility |
| **DevOps & Quality** | Active | 2026-12-31 | CI/CD, testing, security, IAM, AWS backups |

### 2. Platform v2.0 split into 7 projects

Old "Platform v2.0" project (monolithic) retired. Replaced with:

| Project | Initiative | Timeline | Lead |
|---------|-----------|----------|------|
| Discover V2 | V2 Platform | Q1-Q2 (active) | jordan |
| Transfer V2 | V2 Platform | Q2-Q3 | TBD |
| Export V1.5 | V2 Platform | Q1-Q2 | TBD |
| Bedrock / CloudBridge V2 | V2 Platform | Q2-Q3 | TBD |
| Infrastructure V2 | V2 Platform | Q3 | TBD |
| Inplace Recovery | V2 Platform | Q3-Q4 | TBD |
| Dashboard / Stargate V1.5 Compat | V2 Platform | Q4 | TBD |

Rationale: components have different timelines, different owners, and ship independently.

### 3. Existing projects linked to initiatives

| Project | Initiative |
|---------|-----------|
| Clio Integration v1.1 | Clio Launch |
| Clio Marketing | Clio Launch |
| Landing Site v2.2 | Landing Site & Growth |
| Dashboard v1.1 Facelift (new) | Dashboard Evolution |
| DevOps CI/CD (new) | DevOps & Quality |

### 4. Issues migrated

6 issues moved from Platform v2.0 → Discover V2:
- ENG-2030: Throttle Max Request Currency Rule
- ENG-1876: Outlook
- ENG-2081: Lambda Debug Harness
- ENG-2039: Migrate Bedrock/Cloudbridge Components
- ENG-2037: Migrate Discover V2 to Monorepo
- ENG-1812: OneDrive Service Integration

Platform v2.0 marked as Completed (archived).

## Process changes (agreed, not yet implemented)

- **Issue-first workflow**: create Linear issue before starting significant work
- **Minimum issue template**: title (action verb), 1-3 sentence description, acceptance criteria (max 3), scope (in/out)
- **Bidirectional linking**: Linear ↔ memory-bank plans, branch naming `feat/ENG-123-description`
- **Cycle discipline**: only pull in realistically completable work (target 15-25 issues/cycle)
- **Weekly triage**: 15 min — review inbox, stale items, pull next work

## Still TODO

- Backfill Discover V2 with real issues for active work streams
- Triage 10 stale "Idea" projects (archive or promote)
- Standardize issue template across team
- Add Route G to CLAUDE.md (execute Linear issue)
- Create `/create-linear-issue` + `/resolve-linear-issue` commands
