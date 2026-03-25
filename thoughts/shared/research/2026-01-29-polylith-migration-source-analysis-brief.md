---
date: 2026-01-29T14:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-01-29-polylith-migration-source-analysis.md
last_generated: 2026-01-29T19:34:10.015655+00:00
---

# Research Brief: 2026-01-29-polylith-migration-source-analysis

## TL;DR

The migration involves 5 interconnected codebases that form a distributed backup discovery system:

| Component | Type | Purpose | Key Dependencies |
|-----------|------|---------|------------------|
| **Discover** | CLI App | Cloud resource traversal orchestrator | bedrock, cloudbridge, httpx, anyio, uvloop, glide |
| **Bedrock** | Library | Shared infrastructure (config, auth, persistence, throttling) | boto3, aioboto3, SQLAlchemy, pydantic, valkey-glide |
| **Cloudbridge** | Library | Cloud provider OAuth & API integrations | requests, jwt, boto3 (SSM) |
| **Valkey Lua** | Script | Distributed rate limiting with tree scheduling | Pure Lua for Redis/Valkey |
| **Lambda** | Function | DynamoDB stream processor for throttle context registration | bedrock.throttling, Lambda Powertools, glide |

### Dependency Graph
```
Discover (CLI)
    ├── bedrock (fsbedrock)
    │   ├── cloudbridge (fscloudbridge)
    │   └── valkey-glide
    └── cloudbridge (fscloudbridge)

Lambda (valkey_queue_processor)
    ├── bedrock.throttling
    ├── Lambda Layers
    │   ├── core (bedrock + cloudbridge)
    │   ├── valkey (glide)
    │   └── utils (powertools)
    └── Valkey Lua Script (deployed separately)
```

---

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Python Version Alignment**: Discover requires `>=3.13`, bedrock supports `>=3.12,<=3.15`. Monorepo should use 3.13.
2. **Git Dependency Migration**: How to convert git URL dependencies to Polylith workspace references?
3. **Lua Script Deployment**: Should the Lua script be a component resource or stay in infrastructure?
4. **Lambda Layer Strategy**: Replace CDK layer bundling with Polylith workspace packaging?
5. **Infrastructure Boundaries**: How much of the CDK/OpenTofu infrastructure should be in the monorepo vs separate?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-01-29-polylith-migration-source-analysis.md`
