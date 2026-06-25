---
title: "Backups"
description: "Continuous WAL archiving and point-in-time recovery to object storage."
category: "Guides"
weight: 5
---

Keldon performs physical base backups and continuously archives WAL segments to object storage. This combination enables point-in-time recovery to any moment within the retention window.

## Configuring a backup destination

Backups are stored in any S3-compatible bucket. Configure credentials as a `Secret` and reference them in the cluster spec:

```yaml
spec:
  backup:
    barmanObjectStore:
      destinationPath: "s3://my-bucket/cnb/"
      s3Credentials:
        accessKeyId:
          name: s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: s3-creds
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
    retentionPolicy: "30d"
```

## Scheduled backups

Create `ScheduledBackup` resources to trigger full base backups on a cron schedule:

```yaml
apiVersion: postgresql.cnb.io/v1
kind: ScheduledBackup
metadata:
  name: nightly
spec:
  schedule: "0 2 * * *"
  cluster:
    name: prod
```

## Point-in-time recovery

Restore a cluster to any moment in the retention window by bootstrapping from a backup:

```yaml
spec:
  bootstrap:
    recovery:
      source: prod-backup
      recoveryTarget:
        targetTime: "2026-04-22 10:15:00"
```

## Verifying backups

List completed backups:

```bash
kubectl get backups
```

Each `Backup` resource reports its size, duration, and storage location.
