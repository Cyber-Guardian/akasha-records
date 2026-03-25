<!-- This file is the AI-facing project identity, loaded into every Claude session.
     For human onboarding, see README.md -->

# FileScience — Project Identity

## What we're building
**Polylith monorepo** for FileScience backend services — multiple services/apps sharing reusable components. Automated cloud backups + granular recovery (item, user, or environment).

## Product north star
Fast, slick, accessible business continuity tool for cloud apps. Few-click setup, fast search, fast restore. Simple > complex.

**Personas:** IT/MSP operators (multi-tenant leverage) and non-IT operators (self-serve recovery).

## Supported clouds (hard constraint)
Clio, Microsoft 365, Box. Other `/clouds/*` pages are prospecting/interest tests.

## Tools / environment
- **Python** `>=3.13`, **uv** for packaging, **Polylith** architecture
- **OpenTofu/Terragrunt** for IaC, **AWS** (Lambda, DynamoDB, Valkey, S3)
- **Claude Code** with hooks, skills, agents — see `.claude/`
- **Helm** — swarm orchestration harness for concurrent agents (`tools/helm/`)
- Config: `pyproject.toml`, `workspace.toml`, `uv.lock`

## Repo layout
- `components/filescience/` — reusable bricks (cloud_api, config, domain_backup, dynamodb, lambda_utils, models, telemetry, throttling)
- `bases/filescience/` — entry points (discover, entity_discovery_trigger, valkey_queue_processor)
- `projects/` — runnable services (discover, entity-discovery-trigger, valkey-queue-processor)
- `infrastructure/` — Terraform modules + Terragrunt live configs
- `tools/helm/` — Helm CLI (Typer + httpx + Pydantic)
- `memory-bank/` — durable project memory + archival artifacts

## What's deployed (dev, us-east-1)
- VPC `vpc-0a52880852981019f` with bastion `i-0931761e83ccd3e0a`
- DynamoDB: `work-queue`, `resources`, `dev-idempotency`
- Valkey: `filescience-dev-valkey.ubl0dl.clustercfg.use1.cache.amazonaws.com:6379` (standalone mode — GLIDE cluster workaround)
- Content-hashed Lambda artifacts via `make build && make upload-sha256`
- 10 components, 3 bases, 3 projects. 7 import-linter contracts enforcing boundaries.

> CI health, Polylith inventory, and artifact versions are in `state.md` (CI-generated).

## Quick commands
```bash
polylith info                    # workspace overview
make test                        # full test suite
make lint-imports                # architectural boundary check
make build && make upload-sha256 # build + deploy artifacts
aws ssm start-session --target i-0931761e83ccd3e0a  # bastion
```
