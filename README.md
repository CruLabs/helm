# CruLabs Helm Charts

Helm charts for the CruLabs cluster.

## Charts

- **tenant** - Manages Flux CD tenants
- **cnpg-cluster** - CloudNativePG cluster wrapper

## Releasing

Tag the chart with the version to trigger a release:

```bash
git tag tenant-0.2.0
git push origin tenant-0.2.0
```
