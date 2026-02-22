# kustomize-cluster

Kustomize configurations for OpenShift cluster workloads. Uses ArgoCD sync waves and KSOPS for secret decryption.

## Sync Wave Flow

```
Wave 0: Bootstrap and cluster baseline configuration
Wave 1: Operator layer and CRD providers
Wave 2: Workload layer that depends on installed operators
PostSync: Operational follow-up automation
```

Waves are evaluated per ArgoCD Application. They provide ordering intent but do not create global ordering across all Applications.

## Features

- **GitHub SSO**: OpenShift, ArgoCD, AWX, and Grafana all authenticate via GitHub OAuth
- **Cloudflare Tunnels**: External apps via cloudflare-operator with TunnelBindings per app
- **Tor Hidden Services**: Centralized tor-controller with OnionService CRDs per workload
- **Let's Encrypt Certs**: Wildcard `*.apps.makeitwork.cloud` via cert-manager DNS-01 (Cloudflare)
- **Public Status Page**: `status.makeitwork.cloud` served by dedicated anonymous Grafana instance with blackbox probe metrics
- **Pull-Through Cache**: Docker registry mirror for ARC runners to reduce rate limits
- **App-of-Apps**: Each workload is a separate ArgoCD Application for independent sync

## Requirements

- OpenShift GitOps operator
- OpenShift cert-manager operator
- CRC with monitoring enabled (`crc config set enable-cluster-monitoring true`)
- `sops-age-keys` secret in `openshift-gitops` namespace (for SOPS decryption)

## Cloudflare DNS Ownership

Public app DNS under `*.makeitwork.cloud` is managed by cloudflare-operator from `TunnelBinding` resources in this repo.

- Keep `TunnelBinding.tunnelRef.disableDNSUpdates: false` for operator-managed DNS
- Set `subjects[].name` to the real Service name in the same namespace
- The operator writes `_managed.<fqdn>` TXT records alongside CNAMEs for ownership tracking
- Do not delete CNAME records without deleting matching `_managed.<fqdn>` TXT records (stale TXT `DnsId` values cause reconcile error `81044`)
- `operators/cloudflare/dns-adoption-job.yaml` is legacy and is intentionally not referenced from `operators/cloudflare/kustomization.yaml`

## CI/CD

On push to `main`, GitHub Actions:
1. Runs pre-commit tests (YAML lint, etc.)
2. Connects to cluster via Cloudflare WARP
3. Triggers ArgoCD sync via OpenShift API

The `ci-deployer` service account provides cluster-admin access for CI/CD workflows. Its token is automatically synced to GitHub Actions secrets (`OPENSHIFT_TOKEN`) via a PostSync job after each ArgoCD sync.

## Resource Management

This is a single-node CRC cluster. Prefer **no container requests/limits** unless there is a proven stability need:

- High requests commonly trigger `Insufficient cpu/memory` and block scheduling
- CPU limits cause throttling even with spare capacity
- Memory limits can cause avoidable OOM kills

See `AGENTS.md` for detailed guidance on resource configuration.

## SOPS Encryption

Secrets are encrypted with age. Each directory with secrets has a KSOPS generator:

```bash
# Encrypt a secret
sops -e --age age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l secret.yaml

# Decrypt for viewing
sops -d secret.yaml
```
