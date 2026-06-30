---
title: "Operations"
description: "Day-two operational topics: cluster lifecycle, scaling, high availability, failover, backups, and upgrades."
category: "Documentation"
weight: 3
---

{{< doc-intro title="Before you start" >}}
This page covers what you do once a cluster is running. It's organized around the operational lifecycle: understanding how the operator drives a cluster, scaling it, keeping it highly available, recovering from failures, taking backups, and upgrading the operator and database engine.

For initial install and first-cluster setup, see [Getting Started](../getting-started). For full spec references of the resources mentioned here, see [Custom Resources Reference](../custom-resources).
{{< /doc-intro >}}

## Cluster Lifecycle

The operator runs a continuous reconciliation loop. Every time the cluster's spec or state changes — a YAML applied, a pod restarted, a node disappeared — the operator wakes up, compares declared spec to actual state, and takes whatever action is needed to converge them.

### Bootstrap phases

When you first apply a `DatabaseCluster`, the operator drives it through a sequence of phases until it reaches `Running`. Each phase corresponds to a discrete step in the bootstrap process.

| Phase | What happens | Typical duration |
|-------|--------------|------------------|
| `PodsStarting` | StatefulSets created. Pods being scheduled, images pulled, containers started. | 30s – 3 min |
| `SSHReady` | All pods are responsive over SSH. Trust established between them. | 10 – 30s |
| `Initializing` | `gpinitsystem` runs on the coordinator. The catalog is created, segments registered. | 60 – 120s |
| `Running` | Cluster fully operational. Accepting connections. | — |

Watch progression with:

```bash
kubectl get dbc -w
```

> The phase reflects the current activity. If you see a phase persisting for much longer than the typical duration, the operator is likely retrying due to an external problem (image pull failure, storage provisioning issue, etc.) — see [Troubleshooting](../troubleshooting).
{.note}


### Pod restart behavior

When a pod restarts (image upgrade, node drain, OOM kill), the operator's behavior depends on the pod's role:

- **Coordinator restart with active standby** — The operator promotes the standby and updates the Service selector. The original coordinator returns as a new standby once it's healthy.
- **Coordinator restart with no standby** — The cluster becomes unavailable until the coordinator pod returns. Brief downtime, typically 15 – 60 seconds.
- **Standby restart** — No client impact. Standby re-syncs from the primary via WAL streaming.
- **Segment restart with mirror present** — The mirror is promoted to active. The original primary becomes the new mirror after recovery.
- **Segment restart with no mirror** — Queries that need that segment fail until the pod returns. Then queries resume.
- **Mirror restart** — No client impact. Mirror catches up via WAL replication.

The operator handles all of this automatically when `spec.recovery.autoRecover: true`.

---

## Scaling

Keldon supports horizontal scaling of segments without downtime. Add segments, the operator provisions them and redistributes data automatically.

### Scaling up

Increase `spec.segments.count`:

```bash
kubectl patch dbc cluster-demo --type merge \
  -p '{"spec":{"segments":{"count":4}}}'
```

Or edit the resource directly:

```bash
kubectl edit dbc cluster-demo
```

The operator transitions the cluster through:

```
Running  →  Scaling  →  Running
            ↑
            New segment pods provisioned
            New segments added to gp_segment_configuration
            Data redistributed across the expanded cluster
```

A scaling operation on a 100GB cluster going from 2 to 4 segments typically takes 5 – 20 minutes depending on storage performance.

> Watch progress with `kubectl get dbc -w` and `kubectl get pods -l keldon.io/cluster=<name>`. New segment pods appear with incrementing ordinals: `cluster-demo-segment-2`, `cluster-demo-segment-3`.
{.tip}

When scaling up with mirroring enabled, new mirror pods are also provisioned, paired with the new primaries.

### Scaling down is not supported


> Decreasing `spec.segments.count` is not supported in the current version and it's rejected by the admission webhook. keldon will support scaling down in futur version, for now to shrink a cluster, the correct path is to take a backup, create a new smaller cluster, and restore into it.
{.warning}

The migration path:

```bash
# 1. Back up the existing cluster
kubectl apply -f backup-before-shrink.yaml

# 2. Wait for backup completion
kubectl wait --for=condition=Complete --timeout=1h dbb/backup-before-shrink

# 3. Create the smaller target cluster (different name)
kubectl apply -f cluster-shrunken.yaml

# 4. Restore into the new cluster
kubectl apply -f restore-into-shrunken.yaml

# 5. Update applications to point at the new cluster
# 6. Delete the original cluster
```


---

## High Availability

Keldon provides high availability at two layers: coordinator (via standby) and segments (via mirrors). These are independent — you can enable either or both.

### Standby coordinator

The standby coordinator is a hot replica of the primary, kept in sync via PostgreSQL WAL streaming. It runs in its own pod, on a separate StatefulSet, with its own PVC.

```yaml
spec:
  standby:
    enabled: true
    storage: 10Gi
    storageClassName: local-path
```

Once configured, the standby:

- Streams WAL from the primary coordinator continuously
- Maintains an identical catalog (databases, users, schema definitions)
- Is **not** writable while in standby mode
- Can be promoted to active coordinator in case of primary failure

Verify the standby is in sync:

```bash
kubectl describe dbc <cluster-name>
```

You should see entries for the primary coordinator (`role='p'`) and standby (`role='m'`), both with `status='u'`.

### Mirror segments

Mirror segments are hot replicas of primary segments. Each primary segment has exactly one mirror, on a different pod and ideally a different node.

```yaml
spec:
  mirroring: true
```

When enabled, the operator:

- Provisions one mirror pod per primary segment
- Sets up WAL streaming from primary to mirror
- Updates `gp_segment_configuration` to track primary/mirror relationships
- Configures Cloudberry's `gp_fts_probe_interval` for failure detection

Check segment health:

```bash
kubectl describe dbc <cluster-name>
```

You should see, for each `content` (segment number), two rows: one with `role='p'` (primary) and one with `role='m'` (mirror), both with `status='u'` (up).




> Replicas are NOT backups. A `DROP TABLE` replicates to mirrors and standby instantly. If you need to recover deleted data, you need an out-of-band backup.
{.warning}

---

## Failover and Recovery

### Coordinator failover

When the active coordinator pod becomes unavailable, the operator:

1. Detects the failure (pod NotReady for > 30 seconds, or unresponsive to health probes)
2. Verifies the standby is healthy and caught up
3. Runs `gpactivatestandby` on the standby to promote it
4. Updates the Service selector to point at the promoted pod
5. Sets `status.phase = CoordinatorFailover`, then `Running` once promotion completes

Total client-visible downtime is typically 30 – 90 seconds. Existing connections are dropped; clients must reconnect.

```
Before failover:
  cluster-demo-coordinator-0    Active   (primary)
  cluster-demo-standby-0        Ready    (standby)

After failover:
  cluster-demo-coordinator-0    Failed   (will not auto-recover)
  cluster-demo-standby-0        Active   (now serving as primary)
```

> Failover is one-way. The operator does not automatically fail back to the original primary even after it returns to health. Auto-failback is a planned feature but currently requires manual intervention. To re-establish HA, the original coordinator becomes the new standby on next reconciliation, after which you can manually swap roles via a planned downtime window.
{.note}



### Automatic segment recovery

When `spec.recovery.autoRecover: true`, the operator detects failed segments and bring them back online as mirrors:

```yaml
spec:
  recovery:
    autoRecover: true
```



### Automatic rebalancing



When `spec.recovery.autoRebalance: true`, the operator does rebalance the segment data automatically after a configurable grace period:

```yaml
spec:
  recovery:
    autoRecover: true
    autoRebalance: true
    rebalanceDelaySeconds: 30
```

`rebalanceDelaySeconds` lets you batch recovery and rebalancing if multiple segments fail in quick succession.

> Set `rebalanceDelaySeconds` to a few minutes (e.g. 300) on busy production clusters. Rebalance moves data and creates load — better to do it once after the cluster stabilizes than after every individual recovery.
{.tip}



---

## Upgrading the Operator

Operator upgrades use the standard Helm upgrade flow.

### Helm upgrade workflow

```bash
helm repo update
helm upgrade keldon-operator keldon/keldon-operator -n keldon-system
```

The upgrade:

1. Pulls the new chart from `charts.keldon.io`
2. Updates the operator Deployment to the new image
3. Applies any CRD updates included in the new chart
4. Rolls the operator pod (brief downtime of the controller, not the database)

> During an operator upgrade, running database clusters are unaffected. The operator going offline briefly means no reconciliation happens — but pods stay running, queries continue. The operator catches up on missed work when it comes back.
{.note}


### Rolling back

If an upgrade goes wrong, Helm makes rollback easy:

```bash
# See revision history
helm history keldon-operator -n keldon-system

# Roll back to the previous revision
helm rollback keldon-operator -n keldon-system
```

> Helm rollback restores the operator binary and chart configuration, but does **not** automatically roll back CRD schema changes. If the new version added or modified CRDs, rolling back the chart may leave incompatible CRDs in place. Read the upgrade notes carefully — they'll tell you whether rollback is safe.
{.warning}

### CRD migration between versions

CRDs evolve with the operator. Backward-incompatible changes are flagged in release notes. The chart's pre-upgrade hook applies any necessary CRD updates before the new operator starts.

To inspect what changed:

```bash
# View the current CRD spec
kubectl get crd databaseclusters.keldon.io -o yaml > current-crd.yaml

# After upgrade, compare
kubectl get crd databaseclusters.keldon.io -o yaml > new-crd.yaml
diff current-crd.yaml new-crd.yaml
```

---

## Upgrading a Database

Upgrading the Cloudberry version inside a running cluster is a separate operation from upgrading the operator.

### Creating a new DatabaseImage version

First, register the new image:

```yaml
apiVersion: keldon.io/v1alpha1
kind: DatabaseImage
metadata:
  name: cloudberry-2.2.0
spec:
  version: "2.2.0"
  image: ghcr.io/keldonio/cloudberry:2.2.0
```

Apply it. This doesn't change anything yet — it just registers the new image.

### Pointing the cluster at the new image

Update the `DatabaseCluster` to reference the new image:

```bash
kubectl patch dbc cluster-demo --type merge \
  -p '{"spec":{"databaseImage":"cloudberry-2.2.0"}}'
```

The operator detects the image change and starts a rolling update of cluster pods. Pods restart one role at a time:

```
1. Mirror segments restarted     (no client impact, mirrors only)
2. Primary segments restarted    (brief query disruption per segment)
3. Standby restarted              (no client impact)
4. Coordinator restarted          (brief connection disruption)
```

Total upgrade time depends on data volume and the size of WAL backlogs to replay.



---

That covers day-to-day cluster operations. Next: [Advanced Features](../advanced-features) for SSH trust modes, multi-fork support, kernel tuning, and webhook configuration.
