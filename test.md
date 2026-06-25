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


## Cleanup

To remove the test cluster:

## Notes

This page is only used for testing.

You can delete it later once the sidebar sync and documentation layout are working correctly.
