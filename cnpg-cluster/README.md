# cloudnative-pg

![Version: 1.1.2](https://img.shields.io/badge/Version-1.1.2-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

This cloudnative-pg Helm Chart is a simple wrapper chart to deploy a [CloudNativePG](https://cloudnative-pg.io) cluster in Kubernetes.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Examples](./examples/)
- [Secrets](#secrets)
- [Backups](#backups)
- [Parameters](#parameters)
- [Roadmap](#roadmap)
- [References](#references)
- [Changelog](#changelog)

## Features

- Deploys a **CloudNativePG cluster** via Helm
- Easy configuration through values
- Built-in options for:
  - PostgreSQL image & replicas
  - InitDB (database, owner, secrets)
  - Superuser access
  - Storage & StorageClass
  - Backups (S3-compatible via Rook/Ceph Object Storage, including retention & schedule)
  - Affinity & PodAntiAffinity
  - Monitoring via PodMonitor

## Installation

Typically, the chart is deployed in Flux with a HelmRelease inside a namespace:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: cnpg-cluster
  namespace: <namespace>
spec:
  interval: 24h
  url: oci://ghcr.io/crulabs/helm/cnpg-cluster
  layerSelector:
    mediaType: "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
    operation: copy
  ref:
    tag: "1.0.0"
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <database-cluster-name>
  namespace: <namespace>
spec:
  interval: 1h
  chartRef:
    kind: OCIRepository
    name: cnpg-cluster
  values:
    # configuration values go here
```

## Secrets

When deploying the chart, the creation of Kubernetes secrets for the **database owner** and optionally for **backups** is needed.

The **owner secret** (for `initdb.owner`) is a standard Kubernetes secret containing the database user credentials (username/password). The name of this secret can be provided via `initdb.secret.name` in `values.yaml`.

When using default values for initdb owner:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cnpg-app-user
  namespace: <namespace>
type: kubernetes.io/basic-auth
stringData:
  username: app
  password: <app-user-password>
```

## Backups

Backups are configured via S3-compatible object storage. The chart natively integrates with **Rook/Ceph Object Storage** via an `ObjectBucketClaim` (OBC).

### Rook/Ceph Integration

When `backup.enabled: true`, the chart automatically creates an `ObjectBucketClaim` named `<release-name>-backups` in the same namespace. Rook/Ceph provisions the bucket and creates a matching ConfigMap and Secret with the same name containing the connection details:

```
ConfigMap: <release-name>-backups
  BUCKET_HOST  → RGW service hostname
  BUCKET_NAME  → auto-generated bucket name
  BUCKET_PORT  → RGW service port

Secret: <release-name>-backups
  AWS_ACCESS_KEY_ID     → bucket access key
  AWS_SECRET_ACCESS_KEY → bucket secret key
```

The chart reads these automatically, no manual configuration of `destinationPath` or `endpointURL` is required when using Rook/Ceph.

```yaml
backup:
  enabled: true
  retentionPolicy: "30d"
  schedule: "0 0 0 * * *"
```

### Custom S3 Endpoint

If you want to use a different S3-compatible backend, you can override `destinationPath` and `endpointURL` manually, and provide your own credentials secret:

```yaml
backup:
  enabled: true
  destinationPath: "s3://my-bucket/"
  endpointURL: "https://my-s3-endpoint"
  secret:
    name: my-backup-credentials
  retentionPolicy: "30d"
  schedule: "0 0 0 * * *"
```

The credentials secret must contain the keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-backup-credentials
  namespace: <namespace>
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: <access-key-id>
  AWS_SECRET_ACCESS_KEY: <access-secret-key>
```

## Parameters

| Parameter | Description | Default |
|---|---|---|
| `imageName` | PostgreSQL container image | `ghcr.io/cloudnative-pg/postgresql:17.5` |
| `type` | 'postgres' or 'timescaledb' | `postgres` |
| `instances` | Number of cluster instances | `2` |
| `version` |  major version, only needed for timescaledb, ignored for postgres (number, not string) | `16` |
| `initdb.database` | Default database name | `""` defaults to release name |
| `initdb.owner` | Database owner | `app` |
| `initdb.secret.name` | Secret containing user credentials | `cnpg-app-user` |
| `initdb.postInitSQL` | List of SQL queries executed as superuser in the postgres database after cluster creation | `[]` |
| `initdb.postInitApplicationSQL` | List of SQL queries executed as superuser in the app database after cluster creation | `[]` |
| `enableSuperuserAccess` | Enable/disable superuser access | `false` |
| `superuserSecret.name` | Secret with superuser credentials | `cnpg-superuser` |
| `storageClass` | StorageClass for PVCs | `local-path` |
| `storage.size` | Volume size | `5Gi` |
| `backup.enabled` | Enable backups | `false` |
| `backup.destinationPath` | Backup destination (S3-compatible) | `""` auto-resolved from OBC ConfigMap |
| `backup.endpointURL` | S3 endpoint URL | `""` auto-resolved from OBC ConfigMap |
| `backup.secret.name` | Secret with backup credentials | `""` defaults to `<release-name>-backups` |
| `backup.schedule` | Cron schedule for backups | `0 0 0 * * *` |
| `backup.retentionPolicy` | Retention policy for backups | `30d` |
| `affinity.enablePodAntiAffinity` | Enable PodAntiAffinity | `false` |
| `affinity.topologyKey` | Topology key for PodAntiAffinity | `kubernetes.io/hostname` |
| `monitoring.enablePodMonitor` | Enable PodMonitor CR for Prometheus | `true` |

## Roadmap

- Restore from backup

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [Rook/Ceph Object Storage](https://rook.io/docs/rook/latest/Storage-Configuration/Object-Storage-RGW/object-storage/)

## Changelog
### 1.1.2
- **feat:** make resource quota configurable

### 1.1.1

- **fix**: `postgresUID` and `postgresGID` are now dynamically set based on `.Values.type` to prevent immutability conflicts on existing PostgreSQL clusters. TimescaleDB uses UID/GID 1000, all other types fall back to 26.

### 1.1.0
- introduce `type` switch between postgres and timescaledb
- add ImageCatalogRef support for TimescaleDB deployments
- refactor postInitApplicationSQL handling to always render a valid array
- ensure CNPG compatibility by preventing null values in SQL configuration
- add automatic TimescaleDB extension injection for timescaledb type
- add shared_preload_libraries configuration for TimescaleDB
- add examples directory reference in README
- remove outdated minimal-with-backup example
- improve test values and introduce version field for major DB selection
- ensure Helm templates are safe for server-side apply (typed validation fixes)

### 1.0.2
- Fix malformed `printf` call in cluster template causing render error

### 1.0.1
- Fix OBC and secret name missing `-backups` suffix, causing credential lookup to fail

### 1.0.0
**BREAKING CHANGE**: secret keys renamed from `ACCESS_KEY_ID`/`ACCESS_SECRET_KEY` to `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`

- Add native Rook/Ceph OBC integration for automatic bucket provisioning
- Auto-resolve `destinationPath` and `endpointURL` from ObjectBucketClaim ConfigMap
- Create OBC only when no custom `destinationPath` is provided
- Default secret name to `<release-name>-backups` when not specified
- Set `backup.enabled` to `false` by default
- Remove hardcoded MinIO endpoint from default values

### 0.4.1
- Fix for wrong value path

### 0.4.0
- Add support for `initdb.postInitApplicationSQL` in CNPG cluster chart
- Allow execution of SQL statements for application database after cluster initialization
- README documentation for usage

### 0.3.0
- Add support for `initdb.postInitSQL` in CNPG cluster chart
- Allow execution of SQL statements after cluster initialization
- README documentation for post-init SQL usage

### 0.2.0
- **fix**: Set `backupOwnerReference: cluster` in ScheduledBackup to ensure old backup objects are automatically cleaned up according to the retention policy

### 0.1.0
- Initial release