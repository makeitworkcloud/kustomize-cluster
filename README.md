# kustomize-cluster

Kustomize configurations for OpenShift cluster workloads. Uses ArgoCD sync waves and KSOPS for secret decryption.

## Structure

```
bootstrap/            # ArgoCD bootstrap applications
├── console-branding/ # OpenShift console branding and banner removal
operators/            # OLM Subscriptions for operator CRDs
├── ansible/          # AWX Operator
├── arc/              # GitHub Actions Runner Controller
├── cert-manager/     # Cert Manager Operator
└── grafana/          # Grafana Operator
workloads/            # CRs and resources that depend on operator CRDs
├── ansible/          # AWX instance + GitHub SSO
├── arc/              # Runner scale sets
├── grafana/          # Grafana instance + GitHub SSO
├── makeitwork-proxy/ # Tor hidden service proxy
└── uptime-kuma/      # Uptime monitoring dashboard
```

## Sync Wave Flow

```
Wave 0: ArgoCD config (KSOPS patch, ClusterRoleBinding, wait for repo-server)
    │   └── Console branding (custom logo, favicon, remove security banner)
    │
    ▼
Wave 1: gitops-operators Application → operators/ (CRDs installed)
    │   └── wait-for-crds Job ensures CRDs are ready
    ▼
Wave 2: gitops-workloads Application → workloads/ (CRs deployed)
```

Operators must be installed before workloads to ensure CRDs exist.

## Requirements

- OpenShift GitOps operator
- `sops-age-keys` secret in `openshift-gitops` namespace (for SOPS decryption)

## SOPS Encryption

Secrets are encrypted with age. Each directory with secrets has a KSOPS generator:

```bash
# Encrypt a secret
sops -e --age age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l secret.yaml

# Decrypt for viewing
sops -d secret.yaml
```
