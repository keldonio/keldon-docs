---
name: Bug report
about: Report something that's not working as expected
title: ''
labels: bug
assignees: ''
---

<!--
Before filing a bug:
- Search existing issues to avoid duplicates
- Make sure you're on a supported version
- For security issues, do NOT file publicly — email security@keldon.io
-->

## Describe the bug

<!-- A clear and concise description of what the bug is. -->

## Steps to reproduce

<!-- Exact steps that reproduce the issue. -->

1.
2.
3.

## Expected behavior

<!-- What did you expect to happen? -->

## Actual behavior

<!-- What actually happened? Include logs or error messages if available. -->

```
<paste logs / error output here>
```

## Environment

<!-- Fill in as much as you can. -->

- Keldon Operator version (if applicable): <!-- e.g. v0.11.0 -->
- Helm chart version (if applicable):       <!-- e.g. 0.1.0 -->
- Cloudberry image (if applicable):         <!-- e.g. ghcr.io/keldonio/cloudberry:2.1.0 -->
- Kubernetes version:                       <!-- output of `kubectl version --short` -->
- Cluster type:                             <!-- k3s, kind, EKS, GKE, AKS, on-prem, etc. -->
- cert-manager version:                     <!-- e.g. v1.16.2 -->
- OS / arch:                                <!-- e.g. Linux x86_64, macOS arm64 -->

## Cluster spec

<!-- If relevant, paste the DatabaseCluster / DatabaseImage / DatabaseClusterClass CRs you applied. -->

```yaml
<paste CR YAML here>
```

## Additional context

<!-- Anything else that might help us understand the issue. -->
