---
title: "Advanced Features"
description: "Deep dives on SSH trust modes, kernel tuning, reusable topologies, custom configuration, multi-fork support, the admission webhook, and pod probes."
category: "Documentation"
weight: 4
---
{{< doc-intro title="Before you start" >}}
This page goes beyond the basics — it covers the operator's more sophisticated features, the reasoning behind them, and operational considerations for production use.

For field references, see [Custom Resources Reference](../custom-resources). For day-two operational topics, see [Operations](../operations).
{{< /doc-intro >}}

## SSH Trust Models

Greenplum-family databases rely heavily on SSH between nodes — for `gpinitsystem`, `gpstart`/`gpstop`, segment recovery, and ad-hoc administration. Traditional deployments use the `authorized_keys` model: every node has every other node's public key, distributed manually.

Keldon supports two SSH trust models. You choose per cluster via `spec.sshTrust`.

### Static mode

```yaml
spec:
  sshTrust: static
```

In static mode, the operator generates one SSH key pair per cluster and distributes the public key to every pod's `authorized_keys`. This is the simpler model and works without certificate infrastructure.

**How it works:**

- A `Secret` named `<cluster>-ssh-key` holds the cluster's RSA key pair
- The operator copies the public key into each pod's `~/.ssh/authorized_keys` at startup
- All inter-pod SSH uses this one key pair

**Trade-offs:**

- Simple to understand and debug
- One key for the entire cluster lifetime — rotation requires re-keying every pod
- Adding a new segment requires updating SSH config on existing pods

> Static mode is the right choice for development clusters, CI environments, and any setup where SSH keys aren't part of your security perimeter. It's faster to bootstrap and easier to debug.
{.tip}

### Certificate mode

```yaml
spec:
  sshTrust: certificate
```

In certificate mode, the operator runs an SSH certificate authority (CA) and signs short-lived certificates for every pod. Pods are configured to trust the CA's public key via `TrustedUserCAKeys` in `sshd_config`.

**How it works:**

- The operator generates an SSH CA on first reconcile (stored in `<cluster>-ssh-ca` Secret)
- Each pod gets host and user certificates signed by the CA, valid for a configurable TTL
- Pods trust any certificate signed by the CA — no `authorized_keys` distribution needed
- Certificates are rotated before expiry automatically

**Trade-offs:**

- No `N²` key distribution problem — adding a segment is trivial
- CA rotation rotates all pod certs in a single operation
- Short-lived certificates limit the blast radius of a key compromise
- Slightly more complex to debug (`ssh -v` shows certificate-based auth)

> Certificate mode is the right choice for production. The operational savings on rotation alone justify the slight added complexity. If you're going to grow your cluster fleet over time, start with certificate mode from day one.
{.tip}

### Choosing a mode

| Concern | Static | Certificate |
|---------|--------|-------------|
| Bootstrap speed | Fastest | ~5s slower for CA setup |
| Adding segments | Requires `authorized_keys` update | Trivial — just sign a new cert |
| Key rotation | Manual, re-keys everything | Built-in, annotation-driven |
| Blast radius if compromised | Cluster lifetime | Certificate TTL only |
| Debugging | Easier (familiar SSH model) | Harder (`ssh -v` for cert validation) |
| Best for | Dev, CI, ephemeral clusters | Production, long-lived clusters |

You can change modes by updating `spec.sshTrust` and reapplying — but it's a disruptive operation that re-keys all pods. Don't switch modes lightly on a running production cluster.

### Triggering CA rotation

Rotate the CA at any time (in certificate mode):

```bash
kubectl annotate dbc cluster-demo keldon.io/rotate-ca=true
```

The operator:

1. Generates a new CA key pair
2. Updates the `<cluster>-ssh-ca` Secret
3. Updates `sshd_config` on every pod to trust the new CA
4. Reissues all pod certificates signed by the new CA
5. Removes the annotation when complete

Total rotation time is typically 30 – 60 seconds for a small cluster, longer for clusters with many segments.

> Rotation does not require pod restarts. SSHd is reloaded on each pod after the new CA is in place. Active SSH sessions are not interrupted; new connections use the new CA.
{.note}

### TTL configuration

Configure certificate TTL via `DatabaseClusterClass`:

```yaml
apiVersion: keldon.io/v1alpha1
kind: DatabaseClusterClass
metadata:
  name: prod-medium
spec:
  sshTrust: certificate
  sshTrustConfig:
    certTTL: 24h
```

Valid range: `1h` to `720h` (30 days). The webhook validates these bounds.

Shorter TTLs increase rotation frequency but reduce blast radius. Longer TTLs reduce churn but extend the period during which a compromised cert could be misused. `24h` is the recommended default for production.

> Certificates are automatically renewed before expiry (typically at 80% of TTL). If you set `certTTL: 1h`, you'll get rotation churn every 50 minutes. Reasonable production values are `24h` to `168h`.
{.warning}

---

## Kernel Tuning

Cloudberry benefits from kernel parameters tuned for high-concurrency MPP workloads. However, Kubernetes pods share the host kernel, so not every Linux sysctl can safely or reliably be applied from inside a pod.

Keldon separates kernel tuning into two categories:

- **Pod-level sysctls** — namespaced settings that can be applied to the database pods.
- **Node-level sysctls** — host-wide settings that must be configured directly on every Kubernetes node.

### Enabling pod-level defaults

```yaml
spec:
  kernelTuning: true
```

When enabled, the operator applies a conservative set of pod-compatible sysctls to the cluster pods.

**Pod-level sysctls that Keldon can tune:**

- `kernel.shmall` — shared memory pages
- `kernel.shmmax` — maximum shared memory segment size
- `kernel.shmmni` — maximum number of shared memory segments
- `kernel.sem` — semaphore limits
- `kernel.msgmnb`, `kernel.msgmax`, `kernel.msgmni` — System V message queue limits
- `net.ipv4.ip_local_port_range` — local ephemeral port range
- `net.ipv4.tcp_syncookies` — TCP SYN cookie behavior, when supported by the kernel

The `kernel.shm*`, `kernel.msg*`, and `kernel.sem` families are namespaced but considered **unsafe sysctls** by Kubernetes. They must be explicitly allowed on the kubelet before pods can use them.

Safe sysctls such as `net.ipv4.ip_local_port_range` are available by default, but `net.*` sysctls cannot be used with `hostNetwork`.

For K3s, allow the unsafe pod-compatible sysctls with:

```yaml
kubelet-arg:
  - "allowed-unsafe-sysctls=kernel.shm*,kernel.msg*,kernel.sem"
```

Then restart K3s:

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
sudo systemctl restart k3s-agent
```

### Overriding individual parameters

For fine-grained control, override specific parameters via `spec.kernelParameters`:

```yaml
spec:
  kernelTuning: true
  kernelParameters:
    kernel.shmmax: "810810728448"
    kernel.shmall: "197951838"
    kernel.sem: "250 2048000 200 8192"
```

Overrides merge with defaults: anything you set replaces the operator's value; anything you do not set keeps the default.

### Node-level sysctls

Some useful Linux tuning parameters are not pod-safe. They are either host-wide, not namespaced, or not reliably writable inside a container runtime.

These should be configured directly on every node:

- `vm.overcommit_memory`
- `vm.overcommit_ratio`
- `vm.swappiness`
- `vm.dirty_ratio`
- `vm.dirty_background_ratio`
- `fs.file-max`
- `net.core.rmem_max`
- `net.core.wmem_max`
- `net.core.netdev_max_backlog`
- `kernel.pid_max`
- `kernel.core_pattern`
- `kernel.core_uses_pid`

In particular, `net.core.rmem_max` and `net.core.wmem_max` should not be treated as reliable pod-level tuning parameters. They may look like network namespace sysctls, but they are not safe candidates for predictable pod-level tuning and should be configured at the node level instead.

Example node-level configuration:

```bash
sudo nano /etc/sysctl.d/keldon-cloudberry.conf
```

```conf
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
vm.swappiness = 10
fs.file-max = 1000000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
```

Apply the configuration:

```bash
sudo sysctl --system
```



> Keldon can tune only pod-compatible sysctls from the cluster spec. Host-wide settings must be configured outside the operator, using node provisioning, `/etc/sysctl.d`, cloud-init, systemd-sysctl, or a privileged node-level automation mechanism.
{.warning}

>Cloud-managed Kubernetes services such as EKS, GKE, and AKS may restrict privileged workloads or unsafe sysctls. If a pod fails during startup, check the kubelet allowlist, Pod Security policy, and container runtime errors.
{.note}

---

## Reusable Cluster Topologies

`DatabaseClusterClass` lets you define a cluster shape once and reference it from many clusters. This is the way to enforce consistency across environments — every staging cluster has the same topology as production, just smaller.

### When to use a class

```
Use DatabaseClusterClass when:
  - You manage 3+ clusters with similar shapes
  - Platform team owns the "shape", app team owns the "instance"
  - You want to enforce production HA standards (mirroring + standby)
  - Clusters are templated/promoted across environments

Skip and use inline spec when:
  - You have one or two unique clusters
  - Every cluster has genuinely different requirements
  - Quick prototyping
```

For a full spec reference, see [DatabaseClusterClass in the CRD reference](../custom-resources#databaseclusterclass).

### Override semantics

The merging rule between class and cluster is simple: **cluster fields beat class fields, at the leaf level.**

Example:

```yaml
# Class definition
apiVersion: keldon.io/v1alpha1
kind: DatabaseClusterClass
metadata:
  name: prod-medium
spec:
  topology:
    segments:
      count: 4
    mirroring: true
  resources:
    segments:
      requests:
        cpu: "4"
        memory: 16Gi
---
# Cluster referencing the class
apiVersion: keldon.io/v1alpha1
kind: DatabaseCluster
metadata:
  name: cluster-demo
spec:
  databaseImage: cloudberry-2.1.0
  databaseClusterClass: prod-medium
  segments:
    count: 8                              # overrides class count
    storage: 100Gi
    storageClassName: gp3                 # cluster-only setting
  # resources not specified → inherited from class
```

The resulting cluster has:

- 8 segments (cluster override)
- Mirroring enabled (inherited from class)
- 100Gi storage on `gp3` (cluster-only)
- CPU/memory requests inherited from class

> Override is at the leaf field level, not whole-object replacement. Setting `spec.segments.count` doesn't blank out the rest of the `segments` object from the class — `storage`, `resources`, etc. are still inherited.
{.note}


>Updating a `DatabaseClusterClass` does **not** automatically reconcile existing clusters that reference it. This is intentional — silent behavior changes from a class update would be dangerous in production.
{.warning}



---

## Custom Database Configuration

`DatabaseConfig` provides declarative management of `postgresql.conf` and `pg_hba.conf`. The operator applies the config to the cluster pods and reloads PostgreSQL.

For a spec reference, see [DatabaseConfig in the CRD reference](../custom-resources#databaseconfig).



### Authentication rules (pg_hba)

`pg_hba.conf` controls who can connect from where and how they authenticate. The operator-managed rules cover internal traffic (coordinator ↔ segments, standby replication, mirror replication). Your `DatabaseConfig.spec.pgHba` rules are appended to those.

Common patterns:

**Allow internal cluster network only:**

```yaml
spec:
  pgHba:
    - type: host
      database: all
      user: all
      address: "10.0.0.0/8"
      method: scram-sha-256
```

**Mixed internal trust + external auth:**

```yaml
spec:
  pgHba:
    - type: host
      database: all
      user: all
      address: "10.0.0.0/16"      # internal pod network
      method: trust
    - type: host
      database: all
      user: all
      address: "0.0.0.0/0"        # external clients
      method: scram-sha-256
```

**Per-database access:**

```yaml
spec:
  pgHba:
    - type: host
      database: analytics
      user: app_user
      address: "10.0.0.0/8"
      method: scram-sha-256
    - type: host
      database: reporting
      user: reader
      address: "10.0.0.0/8"
      method: scram-sha-256
```

### Validation

The admission webhook validates `DatabaseConfig` for:

- Unknown `postgresql.conf` parameters (typos)
- Invalid values for numeric or unit-bearing parameters
- `pg_hba` entries that would break operator-managed connectivity
- Syntactic errors in pg_hba `address` fields

Invalid configurations are rejected at admission time with a clear error message, preventing broken state from reaching the cluster.

---




## Admission Webhook Validation

Keldon's admission webhook validates every CRD operation before it reaches the cluster's etcd. Webhook validation catches errors at the API boundary, where they're easy to fix, instead of producing failed pods deep in a reconcile loop.

### What the webhook checks

| Check | Resource | What it enforces |
|-------|----------|------------------|
| Cluster name length | `DatabaseCluster` | Name ≤ 18 characters (Cloudberry `MAXHOSTNAMELEN` constraint) |
| Required fields | All CRDs | Mandatory fields like `databaseImage`, `coordinator.storage` |
| CRD references | `DatabaseCluster` | Referenced `DatabaseImage`, `DatabaseClusterClass`, `DatabaseConfig` exist and are valid |
| CertTTL bounds | `DatabaseClusterClass` | `sshTrustConfig.certTTL` between `1h` and `720h` |
| HA node availability | `DatabaseCluster` | If `standby.enabled` or `mirroring`, at least 2 schedulable nodes |
| Standby storage default | `DatabaseCluster` | Auto-fills `standby.storage` to coordinator's storage if unset |
| Segment count delta | `DatabaseCluster` | Reject scale-down (decrementing `spec.segments.count`) |

### Required field validation

Beyond simple presence checks, the webhook validates relationships:

- A `DatabaseCluster` with `mirroring: true` and `spec.segments.count = 1` is rejected — mirroring requires at least 2 segments
- A `DatabaseRestore` referencing a non-existent `DatabaseBackup` is rejected
- A `DatabaseCluster` referencing a non-existent `DatabaseImage` is rejected (CRD-level reference validation)

### HA node availability check

If you enable HA (`standby.enabled` or `mirroring`) on a single-node cluster, the webhook rejects the spec — there's no value in HA if both replicas land on the same node.

The check counts schedulable nodes at admission time. If you have 2+ nodes but anti-affinity rules prevent scheduling, that failure happens at pod scheduling time, not admission time.

### Default value mutators

Some webhook handlers run in "mutating" mode — they fill in missing fields rather than rejecting them. Current mutators:

- `standby.storage` defaults to `coordinator.storage` if unset
- `recovery.rebalanceDelaySeconds` defaults to `30` if `autoRebalance: true` but the delay is unset
- `sshTrust` defaults to `static` if unset



---

That covers the advanced features. Next: [Reference](../reference) for the spec-level details — Helm values, all annotations, phase transitions, validation rules, and known limitations.
