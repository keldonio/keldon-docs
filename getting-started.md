---
title: "Getting Started"
description: "Deploy your first PostgreSQL cluster on Kubernetes in under five minutes."
category: "Guides"
weight: 10
---

This guide walks you through the fastest path to a running PostgreSQL cluster on Kubernetes. By the end, you'll have a three-node highly available PostgreSQL setup with automated failover and backups configured.

## Prerequisites

Before you begin, make sure you have:

- A running Kubernetes cluster (v1.28 or later)
- `kubectl` configured to access the cluster
- Cluster-admin privileges to install CRDs

## Install the operator

Apply the operator manifest from the latest release:

```bash
kubectl apply --server-side -f \
  https://releases.keldon.io/operator/latest.yaml
```

Verify the operator is running:

```bash
kubectl get pods -n cnb-system
```

You should see the operator pod in the `Running` state.

## Deploy your first cluster

Create a file named `cluster.yaml`:

```yaml
apiVersion: postgresql.cnb.io/v1
kind: Cluster
metadata:
  name: my-first-cluster
spec:
  instances: 3
  storage:
    size: 10Gi
  bootstrap:
    initdb:
      database: app
      owner: app
```

Apply it:

```bash
kubectl apply -f cluster.yaml
```

## Verify the cluster

Watch the cluster come online:

```bash
kubectl get cluster my-first-cluster -w
```

Once ready, connect to it:

```bash
kubectl port-forward svc/my-first-cluster-rw 5432:5432
psql -h localhost -U app app
```

## Next steps

- [Configure backups](/docs/backups/) to S3-compatible storage
- [Tune PostgreSQL parameters](/docs/configuration/)
- [Set up monitoring](/docs/observability/)
