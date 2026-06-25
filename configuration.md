---
title: "Configuration"
description: "Tune PostgreSQL parameters, storage, and cluster behavior."
category: "Guides"
weight: 3
---

The `Cluster` resource is the heart of Keldon. Everything about a PostgreSQL cluster — replicas, storage, credentials, tuning, backups — is declared in a single manifest.

## Minimal cluster

```yaml
apiVersion: postgresql.cnb.io/v1
kind: Cluster
metadata:
  name: prod
spec:
  instances: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
```

## PostgreSQL parameters

Override any `postgresql.conf` parameter directly:

```yaml
spec:
  postgresql:
    parameters:
      max_connections: "400"
      shared_buffers: "2GB"
      work_mem: "32MB"
      effective_cache_size: "6GB"
```

The operator validates parameters before applying them and rolls out changes safely.

## Resource limits

Set CPU and memory per instance:

```yaml
spec:
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
```

## Bootstrap modes

Three ways to initialize a cluster:

| Mode | Use case |
|------|----------|
| `initdb` | Fresh, empty cluster |
| `recovery` | Restore from a backup |
| `pg_basebackup` | Clone an existing cluster |

## Credentials

Secrets for application users are generated automatically. Access them via:

```bash
kubectl get secret prod-app -o jsonpath='{.data.password}' | base64 -d
```
