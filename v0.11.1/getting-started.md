---
title: "Getting Started"
description: "Install Keldon and deploy your first Cloudberry cluster."
category: "Documentation"
weight: 2
---

{{< doc-intro title="Before you start" >}}
This guide walks you through installing the Keldon operator and bringing up your first MPP database cluster on Kubernetes.
{{< /doc-intro >}}

## Prerequisites

Keldon runs on any CNCF-conformant Kubernetes distribution. Before you install, make sure your environment meets the following requirements.

### Kubernetes version

Keldon requires **Kubernetes 1.29 or later**. 

Check your cluster version:

```bash
kubectl version --short
```

> Keldon is regularly tested against the three most recent Kubernetes minor versions. If you're on a version older than 1.29, upgrade your cluster before continuing.
{.note}

### cert-manager

[cert-manager](https://cert-manager.io) must be installed in your cluster **before** you install Keldon. The operator uses it to provision and rotate the TLS certificate for its admission webhook.

Install cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```



> If you already run cert-manager for other workloads (Ingress TLS, internal CAs, etc.), Keldon shares the same installation. You don't need a dedicated cert-manager instance.
{.tip}



## Install the Operator


### Install via Helm

Install the operator into a dedicated namespace:

```bash
helm repo add keldon https://charts.keldon.io
helm install keldon-operator keldon/keldon-operator \
  --namespace keldon-system \
  --create-namespace
```

The install creates the operator Deployment, ServiceAccount, RBAC rules, CRDs, and the admission webhook configuration. cert-manager will provision the webhook TLS certificate automatically.

> The first install may take 30-60 seconds because cert-manager needs to issue the webhook certificate before the operator pod becomes Ready.
{.note}

### Verify the operator is running

Wait for the operator pod to reach the `Running` state:

```bash
kubectl get pods -n keldon-system -w
```

You should see something like:

```
NAME                               READY   STATUS    RESTARTS   AGE
keldon-operator-7764946649-r76dn   1/1     Running   0          45s
```

Press Ctrl+C once the pod is `1/1 Running`.

### Verify the CRDs are installed

```bash
kubectl get crds | grep keldon.io
```

You should see all six CRDs:

```
databasebackups.keldon.io
databaseclasses.keldon.io
databaseclusters.keldon.io
databaseconfigs.keldon.io
databaseimages.keldon.io
databaserestores.keldon.io
```

### Verify the webhook certificate

```bash
kubectl get certificate -n keldon-system
```

The webhook certificate should show `READY: True`:

```
NAME                              READY   SECRET                            AGE
keldon-operator-webhook-cert      True    keldon-operator-webhook-cert      1m
```

> If the certificate is stuck on `READY: False`, check cert-manager pods are running: `kubectl get pods -n cert-manager`. The most common cause is cert-manager itself not being fully ready when Keldon was installed. A quick fix: `kubectl rollout restart deployment/keldon-operator -n keldon-system`.
{.tip}

### Uninstall

To uninstall the operator:

```bash
helm uninstall keldon-operator -n keldon-system
kubectl delete namespace keldon-system
```

> Uninstalling the operator does **not** delete existing `DatabaseCluster` resources or their PersistentVolumeClaims. Database pods will continue running. To fully clean up, delete all `DatabaseCluster` resources first, wait for their pods to terminate, then manually delete the leftover PVCs.
{.warning}

## Your First Cluster

This walkthrough deploys a 2-segment Cloudberry cluster with full high availability — a standby coordinator and mirror segments. Total time to running: roughly 3-5 minutes after the YAML is applied.

### Create a DatabaseImage

The `DatabaseImage` resource registers a container image and database version that clusters can reference. It's cluster-scoped, so one image can be used by many clusters across namespaces.

Save the following as `database-image.yaml`:

```yaml
apiVersion: keldon.io/v1alpha1
kind: DatabaseImage
metadata:
  name: cloudberry-2.1.0
spec:
  version: "2.1.0"
  image: ghcr.io/keldonio/cloudberry:2.1.0
```

Apply it:

```bash
kubectl apply -f database-image.yaml
```

Verify:

```bash
kubectl get dbi
```

```
NAME               VERSION   IMAGE                                AGE
cloudberry-2.1.0   2.1.0     ghcr.io/keldonio/cloudberry:2.1.0   5s
```

> Always pin to a specific image tag in production (`:2.1.0`, not `:latest`). Floating tags will pull a different image on every pod restart, breaking reproducibility.
{.tip}

### Create a DatabaseCluster

Save the following as `database-cluster.yaml`:

```yaml
apiVersion: keldon.io/v1alpha1
kind: DatabaseCluster
metadata:
  name: cluster-demo
spec:
  databaseImage: cloudberry-2.1.0
  mirroring: true
  standby:
    enabled: true
    storage: 10Gi
    storageClassName: local-path
  coordinator:
    storage: 10Gi
    storageClassName: local-path
  segments:
    count: 2
    storage: 10Gi
    storageClassName: local-path
```

> The cluster name (`metadata.name`) must be **18 characters or fewer**. This limit comes from Cloudberry's internal `MAXHOSTNAMELEN=64` constraint and the way pod DNS names are formed. The admission webhook will reject names longer than 18 characters with a clear error.
{.warning}

Apply it:

```bash
kubectl apply -f database-cluster.yaml
```

> for creating a production grade cluster — see the [Production best practicies](../production) .
{.note}

### Watch the bootstrap process

The operator now provisions all six pods, sets up storage, distributes SSH trust, and runs `gpinitsystem`. Watch the cluster phase transitions:

```bash
kubectl get dbc -w
```

A successful bootstrap progresses through the following phases:

| Phase                    | What's happening                                                    | Typical duration |
|--------------------------|---------------------------------------------------------------------|------------------|
| `Pending`           | Creating kubernetes ressources   | 1s — 5s       |
| `PodsStarting`           | StatefulSets created; pods pulling images and starting containers   | 30s — 2 min      |
| `SSHReady`               | All pods responsive to SSH; trust established between them          | 10 — 30s         |
| `Initializing`           | `gpinitsystem` running, creating the cluster catalog and tablespaces| 60 — 90s         |
| `Running`                | Cluster fully operational                                           | —                |

Total: roughly 1-3 minutes for a small HA cluster.

> If a phase seems stuck for more than a few minutes, check the operator logs with `kubectl logs -n keldon-system deployment/keldon-operator`. The operator emits structured log lines for each step of the reconcile loop — see the [Troubleshooting](../troubleshooting) guide for what to look for.
{.tip}

### See all the pods

Once the cluster reaches `Running`, six pods are up:

```bash
kubectl get pods -l keldon.io/cluster=cluster-demo
```

```
NAME                          READY   STATUS    RESTARTS   AGE
cluster-demo-coordinator-0    1/1     Running   0          3m
cluster-demo-standby-0        1/1     Running   0          3m
cluster-demo-segment-0        1/1     Running   0          3m
cluster-demo-segment-1        1/1     Running   0          3m
cluster-demo-mirror-0         1/1     Running   0          3m
cluster-demo-mirror-1         1/1     Running   0          3m
```

### Connect via psql

The operator exposes the coordinator through a ClusterIP Service named after the cluster. For local development, use kubectl port-forward to reach the coordinator from your laptop:

```bash
kubectl port-forward svc/cluster-demo 5432:5432
```

In a separate terminal, connect with **psql**:
```bash
psql -h localhost -p 5432 -U gpadmin -d postgres -c "SELECT version();"
```


You should see Cloudberry's version output:

```
                       version
-----------------------------------------------------------
 PostgreSQL 14.4 (Cloudberry Database 2.1.0-incubating) ...
(1 row)
```

You can also check the internal segment configuration to verify all segments are healthy:


```bash
  psql -U gpadmin -d postgres -c "SELECT * FROM gp_segment_configuration ORDER BY content, role;"
```

You should see entries for each segment, each mirror, the coordinator, and the standby — all with `status = 'u'` (up) and `mode = 's'` (synchronized).

>You can also run `kubectl describe dbc cluster-demo` to inspect the same health state from Kubernetes. The resource status includes each coordinator, standby, segment, and mirror, along with its `status = 'u'` (up) and  `mode = 's'` (synchronized), without requiring a psql session.
{.tip}


## Connecting to a Cluster

External clients reach your database through the coordinator Service. For production or shared environments, you can expose the coordinator through a LoadBalancer or an ingress controller with TCP support.

### The coordinator Service

Keldon creates a ClusterIP Service named after the cluster:

```bash
kubectl get svc cluster-demo
```

```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
cluster-demo   ClusterIP   10.43.123.45    <none>        5432/TCP   5m
```

Workloads running inside the cluster can connect using the Service's DNS name:

```
cluster-demo.default.svc.cluster.local:5432
```

>The Service's selector points at whichever pod is currently the active coordinator. When the operator triggers a failover from primary to standby, it updates the selector — clients reconnect transparently to the new coordinator.
{.note}

### Port-forwarding

 As shown in the previous section, the fastest way to connect from your laptop is by using kubectl port-forward

```bash
kubectl port-forward svc/cluster-demo 5432:5432
```

In a separate terminal:

```bash
psql -h localhost -p 5432 -U gpadmin -d postgres
```

> Port-forwarding is only intended for development and debugging. The connection breaks if the pod restarts, and it doesn't survive disconnections. For production traffic, use a LoadBalancer or Ingress.
{.tip}

### LoadBalancer

In cloud environments, the simplest way to expose the database externally is a LoadBalancer Service. Patch the coordinator Service:

```bash
kubectl patch svc cluster-demo -p '{"spec":{"type":"LoadBalancer"}}'
```

Or, if you'd rather keep the coordinator Service ClusterIP-only and add a separate LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cluster-demo-external
spec:
  type: LoadBalancer
  selector:
    keldon.io/cluster: cluster-demo
    keldon.io/role: coordinator
  ports:
    - port: 5432
      targetPort: 5432
```

> Exposing a database directly to the internet is rarely the right choice for production. Either restrict the LoadBalancer to internal traffic (`metadata.annotations` for cloud-specific internal LB), or front the cluster with a bastion host, VPN, or service mesh.
{.warning}




---

You now have a running HA Cloudberry cluster on Kubernetes. Next up:

- **[Custom Resources Reference](../custom-resources)** — full spec reference for all six CRDs
- **[Operations](../operations)** — day-two topics: scaling, backups, failover, upgrades
- **[Advanced Features](../advanced-features)** — SSH trust modes, multi-fork support, more
