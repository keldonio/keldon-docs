---
title: "Configuration"
description: "Tune PostgreSQL, storage, networking, and more."
category: "Reference"
weight: 3
---

## Cluster resource

A minimal `CloudberryCluster` resource looks like this:

```yaml
apiVersion: keldon.io/v1
kind: CloudberryCluster
metadata:
  name: my-cluster
spec:
  instances: 3
  storage:
    size: 10Gi
```

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `instances` | `1` | Number of cluster instances |
| `storage.size` | `1Gi` | PVC size per instance |
