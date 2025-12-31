# kustomize-cluster

Kustomize configurations for OpenShift cluster workloads. Uses ArgoCD sync waves and KSOPS for secret decryption.

## Structure

```
bootstrap/              # ArgoCD bootstrap and cluster configuration
├── console-branding/   # OpenShift console branding and banner removal
├── openshift-oauth/    # GitHub OAuth identity provider for OpenShift
├── ci-service-account  # CI/CD service account for GitHub Actions
└── ci-token-sync-job   # PostSync job to sync SA token to GitHub secrets
operators/              # Operator installations and CRDs
├── ansible/            # AWX Operator (OLM Subscription)
├── arc/                # GitHub Actions Runner Controller (Helm)
├── cert-manager/       # Let's Encrypt certs via DNS-01 (Cloudflare)
├── cloudflare/         # Cloudflare Tunnel Operator + ClusterTunnel
├── generator/          # Shared KSOPS generator config
├── grafana/            # Grafana Operator (OLM Subscription)
└── tor-controller/     # Tor hidden service operator
workloads/              # CRs and resources that depend on operator CRDs
├── apps/               # App-of-Apps orchestrator (ArgoCD Applications)
├── ansible/            # AWX instance + GitHub SSO + Tor + TunnelBinding
├── arc/                # DinD runners + image registry + pull-through cache
├── argocd-proxy/       # Tor hidden service + TunnelBinding for ArgoCD
├── grafana/            # Grafana instance + GitHub SSO + Tor + TunnelBinding
├── makeitwork-proxy/   # Tor hidden service for makeitwork.cloud
├── uptime-kuma/        # Uptime monitoring + Tor + TunnelBinding
└── warp/               # Cloudflare WARP connector for private network access
```

## Sync Wave Flow

```
Wave 0: ArgoCD config (KSOPS patch, wait for repo-server)
    │   ├── Console branding (custom logo, favicon, remove security banner)
    │   ├── OpenShift OAuth (GitHub identity provider, cluster-admin for org members)
    │   └── CI service account (ci-deployer with cluster-admin)
    ▼
Wave 1: gitops-operators Application → operators/ (CRDs installed)
    │   └── wait-for-crds Job ensures CRDs are ready
    ▼
Wave 2: gitops-workloads Application → workloads/apps/ (App-of-Apps)
    │   ├── Wave 0: argocd-proxy, makeitwork-proxy, uptime-kuma (no CRD deps)
    │   └── Wave 1: ansible, arc, grafana (depend on operator CRDs)
    ▼
PostSync: ci-token-sync Job syncs ci-deployer token to GitHub Actions secrets
```

Operators must be installed before workloads to ensure CRDs exist.

## Features

- **GitHub SSO**: OpenShift, ArgoCD, AWX, and Grafana all authenticate via GitHub OAuth
- **Cloudflare Tunnels**: External apps via cloudflare-operator with TunnelBindings per app
- **Tor Hidden Services**: Centralized tor-controller with OnionService CRDs per workload
- **Let's Encrypt Certs**: Wildcard `*.apps.makeitwork.cloud` via cert-manager DNS-01 (Cloudflare)
- **Pull-Through Cache**: Docker registry mirror for ARC runners to reduce rate limits
- **App-of-Apps**: Each workload is a separate ArgoCD Application for independent sync

## Requirements

- OpenShift GitOps operator
- OpenShift cert-manager operator
- `sops-age-keys` secret in `openshift-gitops` namespace (for SOPS decryption)

## CI/CD

On push to `main`, GitHub Actions:
1. Runs pre-commit tests (YAML lint, etc.)
2. Connects to cluster via Cloudflare WARP
3. Triggers ArgoCD sync via OpenShift API

The `ci-deployer` service account provides cluster-admin access for CI/CD workflows. Its token is automatically synced to GitHub Actions secrets (`OPENSHIFT_TOKEN`) via a PostSync job after each ArgoCD sync.

## SOPS Encryption

Secrets are encrypted with age. Each directory with secrets has a KSOPS generator:

```bash
# Encrypt a secret
sops -e --age age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l secret.yaml

# Decrypt for viewing
sops -d secret.yaml
```
