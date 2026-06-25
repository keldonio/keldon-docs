---
title: "Installation"
description: "Install the Keldon operator on your Kubernetes cluster."
category: "Guides"
weight: 2
---

Keldon can be installed in several ways depending on your environment and preferences. We recommend Helm for production deployments because it makes upgrades predictable.

## Helm (recommended)

Add the chart repository and install:

```bash
helm repo add cnb https://charts.keldon.io
helm repo update

helm install cnb cnb/keldon \
  --namespace cnb-system \
  --create-namespace
```

## Raw manifests

If you prefer a plain manifest, apply the released YAML directly:

```bash
kubectl apply --server-side -f \
  https://releases.keldon.io/operator/v1.24.0.yaml
```

## Verify the installation

After installation, confirm the operator is healthy:

```bash
kubectl get deploy -n cnb-system
kubectl get crds | grep cnb.io
```

You should see the `Cluster`, `Backup`, `ScheduledBackup`, and `Pooler` CRDs registered.

## Upgrading

Helm handles upgrades transparently:

```bash
helm upgrade cnb cnb/keldon --namespace cnb-system
```

The operator performs a rolling update and does not interrupt running clusters.

## Uninstalling

Removing the operator does not delete your data. Cluster CRs and PVCs remain until you explicitly delete them.

```bash
helm uninstall cnb --namespace cnb-system
```
