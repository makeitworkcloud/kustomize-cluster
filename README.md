# kustomize-cluster

GitOps manifests for the k3s cluster behind [makeitwork.cloud](https://makeitwork.cloud/). ArgoCD reconciles this repo using KSOPS for inline secret decryption.

## Layout

```
bootstrap/   ArgoCD configuration, OIDC RBAC, CI service account, App-of-Apps roots
operators/   Cluster operators that install CRDs (cert-manager, cloudflare, tor, ARC, …)
workloads/   Workload Applications that depend on operator CRDs
```

The root `kustomization.yaml` is for local `kustomize build` testing only. ArgoCD drives production sync from the per-Application sources defined in `bootstrap/`.

## Sync Wave Flow

```
Wave 0: ArgoCD configuration, RBAC, CI service account
Wave 1: bootstrap-secrets and gitops-operators Applications
Wave 2: gitops-workloads Application
PostSync: ci-token-sync, wait-for-* jobs
```

Sync waves order resources within a single Application — they are not global across Applications. Cross-Application ordering is enforced by the App-of-Apps structure and `wait-for-*` post-sync jobs.

## External Traffic

| Domain | Path | TLS |
|---|---|---|
| `<app>.makeitwork.cloud` | HTTP via cloudflare-operator `TunnelBinding` | Cloudflare edge |
| `k3s.makeitwork.cloud` | TCP via `ClusterTunnel` to kube-apiserver, gated by Cloudflare Access | Cloudflare edge |

There is no in-cluster ingress controller and no public IP. Every external entry point is a Cloudflare Tunnel.

### TunnelBinding DNS

Public DNS under `*.makeitwork.cloud` is owned by cloudflare-operator from `TunnelBinding` resources in this repo.

- Keep `tunnelRef.disableDNSUpdates: false` so the operator manages CNAMEs
- `subjects[].name` must match the real `Service` name in the same namespace
- The operator stores ownership in `_managed.<fqdn>` TXT records; deleting a CNAME without removing its matching TXT record yields Cloudflare error `81044`

## Authentication

GitHub OAuth provides SSO for ArgoCD, Grafana, AWX, and kubectl/Headlamp (via OIDC). Cluster-admin RBAC for the maintainer GitHub team is defined in `bootstrap/oidc-rbac.yaml`. CI uses a dedicated `ci-deployer` ServiceAccount whose token is synced to GitHub Actions secrets by a PostSync job.

## SOPS / KSOPS

Secrets are age-encrypted with field-level selective encryption. The `.sops.yaml` `encrypted_regex` targets only sensitive values (tokens, passwords, OAuth client secrets) so metadata stays diffable.

```bash
sops -e -i secret.yaml      # encrypt in place
sops -d secret.yaml         # decrypt to stdout
sops secret.yaml            # decrypt → editor → re-encrypt on save
```

The age public key is committed in `.sops.yaml`. The matching private key is loaded into the cluster as the `sops-age-keys` Secret in the `argocd` namespace and consumed by the KSOPS plugin during ArgoCD manifest generation.

## CI/CD

`.github/workflows/ci.yml`:

1. **test** (`ubuntu-latest`) — runs pre-commit (yamllint, kube-linter, conventional-commit, etc.)
2. **sync** (`arc` runner, `main` only) — `kubectl patch` each App-of-Apps root (`bootstrap-secrets`, `gitops-operators`, `gitops-workloads`) to trigger an ArgoCD sync at the new SHA

The in-cluster ARC runner uses its ServiceAccount token to talk to the API directly.

## Resource Sizing

This is a single-node cluster. Default to **no `resources` block** on app containers — explicit requests trigger `Insufficient cpu/memory` and limits cause throttling or OOM kills with spare capacity. See `AGENTS.md` for guidance on operators installed via remote refs.

## License

GPLv3
