---
title: "Getting Started"
description: "Deploy your first cluster in minutes."
category: "Guides"
weight: 1
---

## Prerequisites

Make sure you have a Kubernetes cluster available and the following tools installed:

```bash
kubectl version --client
helm version
```

## Install Keldon

```bash
helm repo add keldon https://charts.keldon.io
helm repo update

helm install keldon keldon/keldon \
  --namespace keldon-system \
  --create-namespace
```

Verify the operator is running:

```bash
kubectl get pods -n keldon-system
```
