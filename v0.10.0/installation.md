---
title: "Installation"
description: "Set up the operator on your Kubernetes cluster."
category: "Guides"
weight: 2
---

## Helm

You can install Keldon using the official Helm chart:

```bash
helm repo add keldon https://charts.keldon.io
helm repo update
helm install keldon keldon/keldon --namespace keldon-system --create-namespace
```

## Kubectl

Alternatively, install directly with kubectl:

```bash
kubectl apply -f https://github.com/keldonio/keldon-operator/releases/download/v0.10.0/install.yaml
```
