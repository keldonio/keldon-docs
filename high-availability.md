---
title: "High Availability"
description: "Configure replication, failover, and self-healing behavior."
category: "Guides"
weight: 4
---

Keldon uses PostgreSQL streaming replication with synchronous or asynchronous commit modes. The operator continuously monitors cluster health and performs automatic failover when the primary becomes unreachable.

## Replication topology

A standard production cluster consists of one primary and two replicas:

```yaml
spec:
  instances: 3
  minSyncReplicas: 1
  maxSyncReplicas: 1
```

With `minSyncReplicas: 1`, the primary waits for at least one replica to acknowledge each write before returning success.

## Failover behavior

When a primary failure is detected, the operator:

1. Promotes the replica with the most advanced WAL position
2. Reconfigures remaining replicas to stream from the new primary
3. Updates the `-rw` service endpoint
4. Reports the failover event in cluster status

Typical failover completes in under 30 seconds.

## Pod disruption budgets

The operator creates PDBs automatically to prevent Kubernetes from evicting all replicas at once during node maintenance.

## Anti-affinity

By default, the operator schedules instances across different nodes. For stricter isolation across availability zones:

```yaml
spec:
  affinity:
    topologyKey: topology.kubernetes.io/zone
```
