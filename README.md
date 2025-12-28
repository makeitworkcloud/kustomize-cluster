# kustomize-cluster

Kustomize configurations for OpenShift cluster workloads. Uses ArgoCD sync waves and KSOPS for secret decryption.

## Structure

```
bootstrap/              # ArgoCD bootstrap and cluster configuration
├── console-branding/   # OpenShift console branding and banner removal
└── openshift-oauth/    # GitHub OAuth identity provider for OpenShift
operators/              # OLM Subscriptions for operator CRDs
├── ansible/            # AWX Operator
├── arc/                # GitHub Actions Runner Controller
├── cert-manager/       # Cert Manager Operator
├── generator/          # Shared KSOPS generator config
└── grafana/            # Grafana Operator
workloads/              # CRs and resources that depend on operator CRDs
├── ansible/            # AWX instance + GitHub SSO + Tor hidden service
├── arc/                # DinD runners + image registry + pull-through cache
├── argocd-proxy/       # Tor hidden service for ArgoCD
├── grafana/            # Grafana instance + GitHub SSO + Tor hidden service
├── makeitwork-proxy/   # Tor hidden service for makeitwork.cloud
└── uptime-kuma/        # Uptime monitoring dashboard (Helm) + Tor hidden service
```

## Sync Wave Flow

```
Wave 0: ArgoCD config (KSOPS patch, wait for repo-server)
    │   ├── Console branding (custom logo, favicon, remove security banner)
    │   └── OpenShift OAuth (GitHub identity provider, cluster-admin for org members)
    ▼
Wave 1: gitops-operators Application → operators/ (CRDs installed)
    │   └── wait-for-crds Job ensures CRDs are ready
    ▼
Wave 2: gitops-workloads Application → workloads/apps/ (App-of-Apps)
        ├── Wave 0: argocd-proxy, makeitwork-proxy, uptime-kuma (no CRD deps)
        └── Wave 1: ansible, arc, grafana (depend on operator CRDs)
```

Operators must be installed before workloads to ensure CRDs exist.

## Features

- **GitHub SSO**: OpenShift, ArgoCD, AWX, and Grafana all authenticate via GitHub OAuth
- **Tor Hidden Services**: Each workload has an optional Tor v3 hidden service with persistent keys
- **Pull-Through Cache**: Docker registry mirror for ARC runners to reduce rate limits
- **App-of-Apps**: Each workload is a separate ArgoCD Application for independent sync

## Requirements

- OpenShift GitOps operator
- `sops-age-keys` secret in `openshift-gitops` namespace (for SOPS decryption)

## CI/CD

On push to `main`, GitHub Actions:
1. Runs pre-commit tests (YAML lint, etc.)
2. Connects to cluster via Cloudflare WARP
3. Triggers ArgoCD sync via OpenShift API

## SOPS Encryption

Secrets are encrypted with age. Each directory with secrets has a KSOPS generator:

```bash
# Encrypt a secret
sops -e --age age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l secret.yaml

# Decrypt for viewing
sops -d secret.yaml
```
