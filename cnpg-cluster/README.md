# cloudnative-pg

![Version: 0.4.1](https://img.shields.io/badge/Version-0.4.1-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

This cloudnative-pg Helm Chart is a simple wrapper chart to deploy a [CloudNativePG](https://cloudnative-pg.io) cluster in Kubernetes.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
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
  - Backups (S3-compatible, including retention & schedule)
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
    tag: "0.2.0"
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

### Minimal Values Configuration

```yaml
  values:
    storage:
      size: 5Gi
    backup:
      enabled: false
```

## Secrets

When deploying the chart, the creation of Kubernetes secrets for the **database owner** and optionally for **backups** is needed.

- The **owner secret** (for 'initdb.owner') is a standard Kubernetes secret containing the database user credentials (username/password). The name of this secret can be provided via `initdb.secret.name` in `values.yaml`.

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
  password: <app-user-password> # todo: add super safe password
```

- The **backup secret** is required if backups are enabled. The name of the secret can be configured in `backup.secret.name`. The keys inside the secret **must** be named `ACCESS_KEY_ID` and `ACCESS_SECRET_KEY`.

Example backup credentials secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backup-credentials       # see 'backup.secret.name'
  namespace: <namespace>
type: Opaque
stringData:
  ACCESS_KEY_ID: <access-key-id>
  ACCESS_SECRET_KEY: <access-secret-key>
```

## Parameters

| Parameter                | Description                                      | Default |
|---------------------------|--------------------------------------------------|---------|
| `imageName`               | PostgreSQL container image                       | `ghcr.io/cloudnative-pg/postgresql:17.5` |
| `instances`               | Number of cluster instances                      | `2` |
| `initdb.database`         | Default database name                            | `""` defaults to release name |
| `initdb.owner`            | Database owner                                   | `app` |
| `initdb.secret.name`      | Secret containing user credentials               | `cnpg-app-user` |
| `initdb.postInitSQL`      | List of SQL queries to be executed as a superuser in the postgres
database right after the cluster has been created | `[]` |
| `initdb.postInitApplicationSQL' | List of SQL queries to be executed as a superuser in app database | `[]` |
| `enableSuperuserAccess`   | Enable/disable superuser access                  | `false` |
| `superuserSecret.name`    | Secret with superuser credentials                | `cnpg-superuser` |
| `storageClass`            | StorageClass for PVCs                            | `local-path` |
| `storage.size`            | Volume size                                      | `5Gi` |
| `backup.enabled`          | Enable backups                                   | `true` |
| `backup.destinationPath`  | Backup destination (S3-compatible)               | `""` defaults to s3://release-name/ |
| `backup.endpointURL`      | S3 endpoint URL                                  | `http://minio.minio.svc.cluster.local:9000` |
| `backup.secret.name`      | Secret with backup credentials                   | `backup-credentials` |
| `backup.schedule`         | Cron schedule for backups (daily at midnight)    | `0 0 0 * * *` |
| `backup.retentionPolicy`  | Retention policy for backups                     | `30d` |
| `affinity.enablePodAntiAffinity` | Enable PodAntiAffinity                    | `false` |
| `affinity.topologyKey`    | Topology key for PodAntiAffinity                 | `kubernetes.io/hostname` |
| `monitoring.enablePodMonitor` | Enable PodMonitor CR for Prometheus          | `true` |

## Roadmap

- restore from backup
- make helm- or OCI-Repository

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/) 

## Changelog

### 0.4.1
- fix for wrong value path

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