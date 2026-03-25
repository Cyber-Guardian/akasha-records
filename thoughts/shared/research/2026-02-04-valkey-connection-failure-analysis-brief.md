---
date: 2026-02-04T18:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-04-valkey-connection-failure-analysis.md
last_generated: 2026-02-04T18:22:14.159917+00:00
---

# Research Brief: 2026-02-04-valkey-connection-failure-analysis

## TL;DR

The valkey-queue-processor Lambda successfully creates a GlideClusterClient connection during cold start, but then fails with `AllConnectionsUnavailable` when attempting to execute the first command (~500ms later). Analysis of AWS CloudWatch logs and infrastructure configuration reveals this is likely due to **GLIDE's lazy connection behavior** combined with a **very short default connection timeout (250ms)**.

**Key Finding**: The connection "succeeds" during `GlideClusterClient.create()` but fails when the first actual command (`EXISTS`) is attempted. This is consistent with GLIDE's lazy connect behavior where network establishment is deferred until the first command.

**Verified**: Transit encryption is **disabled** on the ElastiCache cluster (`TransitEncryptionEnabled: false`), so TLS mismatch is ruled out.

**Root Cause**: The `connection_timeout` defaults to 250ms and must be set via `AdvancedGlideClusterClientConfiguration`, not the main config. The current code only sets `request_timeout`.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. ✅ ~~**Verify actual ElastiCache transit encryption status**~~: Confirmed `TransitEncryptionEnabled: false`
2. **Test with explicit connection_timeout**: Try setting connection_timeout to 5000ms via `AdvancedGlideClusterClientConfiguration`
3. **Test connectivity from bastion**: SSH to bastion and test `valkey-cli` connection to rule out network issues
4. **Verify AdvancedGlideClusterClientConfiguration import**: Confirm the class is available in `glide_sync` package

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-04-valkey-connection-failure-analysis.md`
