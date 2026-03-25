---
date: 2026-02-04T18:30:00-05:00
researcher: Claude
git_commit: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
branch: main
repository: filescience
topic: "Valkey Queue Processor AllConnectionsUnavailable Error Analysis"
tags: [research, valkey, glide, lambda, connection-error, elasticache]
status: complete
last_updated: 2026-02-04
last_updated_by: Claude
---

# Research: Valkey Queue Processor AllConnectionsUnavailable Error Analysis

**Date**: 2026-02-04T18:30:00-05:00
**Researcher**: Claude
**Git Commit**: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
**Branch**: main
**Repository**: filescience

## Research Question

Why is the valkey-queue-processor Lambda failing to connect to the Valkey cluster with `AllConnectionsUnavailable` errors?

## Summary

The valkey-queue-processor Lambda successfully creates a GlideClusterClient connection during cold start, but then fails with `AllConnectionsUnavailable` when attempting to execute the first command (~500ms later). Analysis of AWS CloudWatch logs and infrastructure configuration reveals this is likely due to **GLIDE's lazy connection behavior** combined with a **very short default connection timeout (250ms)**.

**Key Finding**: The connection "succeeds" during `GlideClusterClient.create()` but fails when the first actual command (`EXISTS`) is attempted. This is consistent with GLIDE's lazy connect behavior where network establishment is deferred until the first command.

**Verified**: Transit encryption is **disabled** on the ElastiCache cluster (`TransitEncryptionEnabled: false`), so TLS mismatch is ruled out.

**Root Cause**: The `connection_timeout` defaults to 250ms and must be set via `AdvancedGlideClusterClientConfiguration`, not the main config. The current code only sets `request_timeout`.

## Detailed Findings

### 1. Error Timeline Analysis

From CloudWatch logs (`/aws/lambda/filescience-dev-valkey-queue-processor`):

| Timestamp | Event |
|-----------|-------|
| 17:18:43.091 | "Creating new Valkey client connection" (cold_start=true) |
| 17:18:43.240 | "Valkey connection established successfully" |
| 17:18:43.698 | "Starting throttle context registration" |
| 17:18:43.777 | **ERROR**: "No random connection found- AllConnectionsUnavailable" |

**Gap**: 537ms between "connection established" and error
**Pattern**: Connection appears successful but fails on first actual command execution

### 2. Current Infrastructure Configuration

**Lambda Environment Variables** (from AWS Lambda get-function-configuration):
```
VALKEY_HOST: filescience-dev-valkey.ubl0dl.clustercfg.use1.cache.amazonaws.com
VALKEY_HOSTS: filescience-dev-valkey-0001-001.ubl0dl.0001.use1.cache.amazonaws.com,filescience-dev-valkey-0001-002.ubl0dl.0001.use1.cache.amazonaws.com
VALKEY_PORT: 6379
VALKEY_USE_TLS: false
```

**Lambda VPC Configuration**:
- VPC: `vpc-0a52880852981019f`
- Subnets: `subnet-0e951044a04181fb2` (10.1.101.0/24, us-east-1b), `subnet-016ecda8b9476941f` (10.1.100.0/24, us-east-1a)
- Security Group: `sg-07e0dffd498855b1e` (filescience-dev-valkey-queue-processor-sg)
  - Egress: All outbound allowed (0.0.0.0/0)
  - Ingress: None (Lambda doesn't need inbound)

**Valkey Cluster Configuration** (from ElastiCache describe-replication-groups):
- Cluster: `filescience-dev-valkey`
- Status: `available`
- Node Groups: 1 shard, 2 nodes (0001-001, 0001-002)
- Parameter Group: `default.valkey8.cluster.on` (cluster mode enabled)
- Engine: Valkey 8.2, Node Type: cache.t4g.micro

**Valkey Security Group** (`sg-0268d4e05ee557a6a` - filescience-dev-valkey-sg):
- Ingress: Port 6379 TCP from VPC CIDR (10.1.0.0/16)
- Ingress: Port 6379 TCP from bastion security group

### 3. Client Configuration in Code

Location: `bases/filescience/valkey_queue_processor/handler.py:117-124`

```python
if use_cluster:
    config = GlideClusterClientConfiguration(
        addresses=addresses,
        request_timeout=5000,  # 5 second timeout
        use_tls=VALKEY_USE_TLS,  # Currently "false" in dev
    )
    _valkey_client = GlideClusterClient.create(config)
```

**Note**: No `connection_timeout` is set (defaults to 250ms per GLIDE documentation).

### 4. GLIDE Client Behavior Research

From web research on valkey-glide issues:

1. **Lazy Connect**: GLIDE defers actual network connection until the first command is executed. The `GlideClusterClient.create()` call may succeed without actually establishing TCP connections.

2. **Connection Timeout Bug** (GitHub Issue #4959): The default `connectionTimeout` is actually 250ms, not 2000ms as documented. This is too short for many ElastiCache configurations.

3. **TLS Requirement**: When ElastiCache has encryption in transit enabled, TLS must be enabled in the client. Without TLS, connections silently fail.

4. **AllConnectionsUnavailable**: This error indicates that the client attempted to send a command but could not find any available connection to any node. This typically occurs when:
   - All connection attempts timed out
   - TLS handshake failed
   - Network connectivity issues

### 5. Security Group Rules Analysis

Lambda SG (`sg-07e0dffd498855b1e`) -> Valkey SG (`sg-0268d4e05ee557a6a`):
- Lambda is in subnets 10.1.100.0/24 and 10.1.101.0/24
- Valkey SG allows port 6379 from 10.1.0.0/16
- **Network path should be allowed** (Lambda subnets are within VPC CIDR)

### 6. Verified Infrastructure State

**ElastiCache Encryption Settings** (verified via `aws elasticache describe-replication-groups`):
```json
{
    "TransitEncryptionEnabled": false,
    "AtRestEncryptionEnabled": true,
    "AuthTokenEnabled": false
}
```

**Valkey Subnet Group**: Same subnets as Lambda (`subnet-016ecda8b9476941f`, `subnet-0e951044a04181fb2`)

### 7. Root Cause Analysis

1. **TLS Mismatch**: ❌ **RULED OUT** - Transit encryption is disabled on cluster, `VALKEY_USE_TLS=false` is correct

2. **Connection Timeout Too Short** (HIGH PROBABILITY):
   - No explicit `connection_timeout` set, defaults to **250ms**
   - `connection_timeout` must be set via `AdvancedGlideClusterClientConfiguration`, not the main config
   - VPC Lambda cold start and ENI attachment add latency
   - **Fix**: Add `advanced_configuration=AdvancedGlideClusterClientConfiguration(connection_timeout=5000)`

3. **Lazy Connect Behavior**:
   - `GlideClusterClient.create()` returns immediately without establishing connections
   - Actual TCP connections attempted on first command execution
   - With 250ms timeout, connections fail before completing

### 8. Required Code Change

The current code at `handler.py:117-124`:
```python
config = GlideClusterClientConfiguration(
    addresses=addresses,
    request_timeout=5000,
    use_tls=VALKEY_USE_TLS,
)
```

Should be changed to include `connection_timeout` via advanced configuration:
```python
from glide_sync import AdvancedGlideClusterClientConfiguration

config = GlideClusterClientConfiguration(
    addresses=addresses,
    request_timeout=5000,
    use_tls=VALKEY_USE_TLS,
    advanced_configuration=AdvancedGlideClusterClientConfiguration(
        connection_timeout=5000,  # 5 seconds for initial connection
    ),
)
```

## Code References

- `bases/filescience/valkey_queue_processor/handler.py:89-142` - `get_valkey_client()` function
- `bases/filescience/valkey_queue_processor/config.py:12-28` - Environment variable configuration
- `components/filescience/throttling/valkey/functions.py:765` - `throttle_context_exists()` where error occurs
- `infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor/terragrunt.hcl:81-90` - Environment variables
- `infrastructure/live/non-prod/us-east-1/dev/valkey/terragrunt.hcl:36` - `transit_encryption_enabled = false`

## Architecture Documentation

### Current Connection Flow

1. DynamoDB stream triggers Lambda
2. Lambda handler calls `get_valkey_client()` (lazy singleton)
3. On cold start, creates `GlideClusterClient` with:
   - Addresses: 2 hardcoded node endpoints
   - request_timeout: 5000ms
   - use_tls: false
4. Client creation logs success (but connection is lazy)
5. First command (`EXISTS` key check) triggers actual connection
6. Connection fails with `AllConnectionsUnavailable`

### Network Topology

```
Lambda (10.1.100.0/24 or 10.1.101.0/24)
    |
    | (SG egress: 0.0.0.0/0)
    v
Valkey Cluster (SG allows 10.1.0.0/16:6379)
    - filescience-dev-valkey-0001-001 (us-east-1a)
    - filescience-dev-valkey-0001-002 (us-east-1b)
```

## Historical Context (from memory-bank/thoughts/)

- `memory-bank/durable/01-active/current_work.md` - Documents this as active blocker: "Valkey connection failing in Lambda: AllConnectionsUnavailable even with cluster client + node endpoints"
- Previous work deployed `VALKEY_USE_TLS="false"` to match dev environment (transit encryption disabled)

## Related Research

- [GitHub valkey-glide #4959](https://github.com/valkey-io/valkey-glide/issues/4959) - Connection timeout bug (250ms default)
- [GitHub valkey-glide #2533](https://github.com/valkey-io/valkey-glide/issues/2533) - GlideClusterClient creation failures with ElastiCache

## Open Questions

1. ✅ ~~**Verify actual ElastiCache transit encryption status**~~: Confirmed `TransitEncryptionEnabled: false`
2. **Test with explicit connection_timeout**: Try setting connection_timeout to 5000ms via `AdvancedGlideClusterClientConfiguration`
3. **Test connectivity from bastion**: SSH to bastion and test `valkey-cli` connection to rule out network issues
4. **Verify AdvancedGlideClusterClientConfiguration import**: Confirm the class is available in `glide_sync` package

## Next Steps

1. Update `handler.py` to import `AdvancedGlideClusterClientConfiguration`
2. Add `connection_timeout=5000` to the client configuration
3. Rebuild and deploy Lambda
4. Test with DynamoDB stream trigger
