---
date: 2026-01-29T00:00:00-05:00
author: cursor-agent
git_commit: TBD
branch: main
repository: filescience
topic: "Valkey CLI + cluster-mode testing conventions"
tags: [decisions, infrastructure, aws, elasticache, valkey, testing]
status: complete
last_updated: 2026-01-29
last_updated_by: cursor-agent
---

# Decision: Valkey CLI + cluster-mode testing conventions

## Context
- We are using **Amazon ElastiCache Valkey** (cluster mode enabled) for cache/work-queue adjacency.
- Testing and ops access go through an **SSM bastion** (no NAT in dev), so “how to connect” needs to be consistent and repeatable.

## Decision
- **Use `valkey-cli`** for connectivity testing from the bastion (not `redis-cli`).
- For **cluster mode enabled**, connect using the **configuration endpoint** and use **cluster-aware mode**:
  - Use the module output `configuration_endpoint` and run `valkey-cli -c ...`.
- For staging/prod when **in-transit encryption** is enabled:
  - Use `--tls` with the OS CA bundle.
  - If `auth_token` is set, provide it to the client (prefer env var usage).

## Options considered
- **A: Use `redis-cli`**
  - Pros: common muscle memory
  - Cons: mismatched naming vs Valkey; encourages drift in docs/examples
- **B: Use `valkey-cli`** (chosen)
  - Pros: matches engine choice; clearer mental model; aligns with AWS Valkey docs
  - Cons: requires ensuring the package is installed on the bastion AMI

## Tradeoffs
- We accept a small amount of “install the right CLI” setup in exchange for consistent terminology and fewer operational footguns.

## Consequences
- Documentation and runbooks should reference:
  - `valkey-cli -c -h <valkey-configuration-endpoint> -p 6379 ...`
  - TLS/auth variant for staging/prod (when enabled)
- Any future examples should avoid ambiguous `<valkey-endpoint>` placeholders; prefer `<valkey-configuration-endpoint>` for cluster mode.

## References
- Plan updated with these conventions: `memory-bank/thoughts/shared/plans/2026-01-29-opentofu-infrastructure-setup.md`
