---
title: "Test Page"
description: "A dummy documentation page used to test sidebar ordering and section anchors and automatic commit."
category: "Guides"
weight: 6
---

This is a dummy documentation page for testing the Keldon docs layout.

The goal of this page is to verify that:

- The page appears in the docs sidebar.
- The page is ordered using `weight: 6`.
- The subtitles appear automatically under the active page.
- Anchor links scroll correctly to each section.

## Prerequisites

Before using Keldon, make sure you have a Kubernetes cluster available.

You should also have the following tools installed locally:

```bash
kubectl version --client
helm version
```

For testing purposes, any local Kubernetes cluster is enough.

Examples:

- kind
- k3d
- minikube
- Docker Desktop Kubernetes

## Install Keldon

You can install Keldon using Helm.

```bash
helm repo add keldon https://charts.keldon.io
helm repo update

helm install keldon keldon/keldon \
  --namespace keldon-system \
  --create-namespace
```

After the installation, check that the operator is running:

```bash
kubectl get pods -n keldon-system
```

## Create a Test Cluster

Once the operator is installed, you can create a dummy database cluster.

```yaml
apiVersion: keldon.io/v1alpha1
kind: DatabaseCluster
metadata:
  name: test-cluster
spec:
  version: "latest"
  replicas: 1
```

Apply it with:

```bash
kubectl apply -f test-cluster.yaml
```

## Verify the Result

Check that the custom resource was created:

```bash
kubectl get databaseclusters
```

You can also describe the resource:

```bash
kubectl describe databasecluster test-cluster
```

## Cleanup

To remove the test cluster:

```bash
kubectl delete databasecluster test-cluster
```

To uninstall Keldon:

```bash
helm uninstall keldon --namespace keldon-system
```

## Notes

This page is only used for testing.

You can delete it later once the sidebar sync and documentation layout are working correctly.
