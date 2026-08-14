# Agent Context for kustomize-cluster

GitOps manifests for the k3s cluster behind makeitwork.cloud. ArgoCD reconciles this repo using KSOPS for inline secret decryption.

## Sync Wave Architecture

```
Wave 0: ArgoCD config, OIDC RBAC, CI service account
Wave 1: bootstrap-secrets, gitops-operators
Wave 2: gitops-workloads
PostSync: ci-token-sync, wait-for-* jobs
```

Sync waves order resources within a single ArgoCD Application — they are **not** global across Applications. The App-of-Apps structure plus `wait-for-*` post-sync jobs enforces cross-Application ordering.

The active Applications reconcile `bootstrap/secrets`, `operators`, and
`workloads/apps`; they do **not** reconcile the rest of `bootstrap/`. Changes
to the ArgoCD CR, OIDC RBAC, or CI ServiceAccount require a separately reviewed
bootstrap apply or node provisioning.

## Domain Architecture

| Domain | Path | TLS |
|---|---|---|
| `<app>.makeitwork.cloud` | HTTP via cloudflare-operator `TunnelBinding` | Cloudflare edge |
| `k3s.makeitwork.cloud` | TCP via `ClusterTunnel` to kube-apiserver, gated by Cloudflare Access | Cloudflare edge |

There is no in-cluster ingress controller and no public IP. Every external entry point — public web, kubectl, everything — uses a Cloudflare Tunnel managed by cloudflare-operator. App CNAMEs are declared in `tfroot-cloudflare` and must stay aligned with the routes here. Legacy hostnames `api.makeitwork.cloud` and `*.apps.makeitwork.cloud` are not in use.

## Key Namespaces

- `argocd` — ArgoCD, KSOPS plugin, `sops-age-keys` Secret
- `cert-manager` — cert-manager controllers + Cloudflare API token
- `cloudflare-operator-system` — cloudflare-operator, tunnel deployment, Cloudflare API secret
- `arc-systems` — ARC controller (Actions Runner Controller)
- `arc-runners` — ARC scale sets, listeners, and ephemeral runner pods

## Certificate Management

cert-manager issues Let's Encrypt certs via the Cloudflare DNS-01 solver. The cluster DNS cannot resolve external domains, so the controller is configured to use external recursive nameservers:

```yaml
extraArgs:
  - --dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53
  - --dns01-recursive-nameservers-only
```

The Cloudflare API token lives in `cert-manager/cloudflare-api-token` and is referenced from `ClusterIssuer` resources.

## Cloudflare Tunnel DNS

App DNS and routes have coordinated owners:

- `tfroot-cloudflare/cf-tunnels.tf` declares CNAMEs
- `TunnelBinding` resources declare routes and the operator-managed ownership TXT records
- `subjects[].name` must match the real Kubernetes `Service` name in the same namespace; if it doesn't exist, status reports `http_status:404`
- Remove and reconcile the `TunnelBinding` before removing its Terraform hostname; deleting only the CNAME leaves a stale `_managed.<fqdn>` TXT record and causes update-by-stale-id failures (`Record does not exist. (81044)`)

## kubectl Access

Use the dedicated `makeitworkcloud-k3s` kubeconfig and the Cloudflare/Dex OIDC
procedure in `README.md#kubectl-access`. Never use or modify an unrelated
production or staging context. If no Make IT Work Cloud context is configured,
ask for the approved public server CA and create the non-secret exec kubeconfig
from `docs/kubeconfig.example.yaml`; do not copy the k3s admin kubeconfig off
the node.

## SOPS / KSOPS

**Selective field encryption only.** `.sops.yaml` defines `encrypted_regex` so only sensitive values are encrypted; manifests stay diffable.

```yaml
encrypted_regex: '^(token|api-token|clientID|clientSecret|password|secret|github_token|CLOUDFLARE_API_TOKEN|credentials\.json|.*_SERVICE_KEY|GF_AUTH_GITHUB_CLIENT_SECRET|GF_SECURITY_ADMIN_PASSWORD|dex\.github\.clientID|dex\.github\.clientSecret)$'
```

### Conventions

- One Secret per file; reference by name from CRDs/Applications
- Never encrypt non-Secret manifests (Namespaces, RBAC, ConfigMaps)
- Never encrypt metadata (names, namespaces, labels, annotations)

### Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-github-oauth
  namespace: argocd
type: Opaque
stringData:
  dex.github.clientID: Ov23liV3VghvjBnQjsWQ
  dex.github.clientSecret: <only this and clientID get encrypted>
```

### Commands

```bash
sops -e -i secret.yaml      # encrypt in place
sops -d secret.yaml         # decrypt to stdout
sops secret.yaml            # edit decrypted, re-encrypt on save
```

### KSOPS generators

Each directory with secrets has a generator listing its encrypted files; the kustomization separates plain `resources` from KSOPS `generators`:

```yaml
resources:
  - deployment.yaml
  - configmap.yaml
generators:
  - ksops-example-secrets.yaml
```

The age private key is mounted into the ArgoCD repo-server as the `sops-age-keys` Secret in the `argocd` namespace.

## Tor Hidden Services

Managed by tor-controller with `OnionService` CRDs per workload.

- Tor keys must use `data` (not `stringData`) with base64-encoded raw binary; the file starts with `== ed25519v1-secret: type0 ==`
- Public `.onion` addresses are documented in `../www/onion.makeitwork.cloud/index.html`

## Resource Sizing

Single-node cluster — default to **no container `resources` block**.

- Explicit requests trigger `Insufficient cpu/memory` scheduling failures
- CPU limits cause throttling even with spare capacity
- Memory limits can cause avoidable OOM kills

```yaml
containers:
  - name: app
    image: example/image:tag
    resources: {}
```

For operators installed via Helm/Subscription, prefer values that disable resources (`resources: {}` or operator-specific fields). For operators installed via kustomize remote refs, strip `resources` with a JSON patch:

```yaml
patches:
  - patch: |
      - op: remove
        path: /spec/template/spec/containers/0/resources
    target:
      kind: Deployment
      name: controller-manager
```

If kube-linter complains, annotate the Deployment:

```yaml
annotations:
  ignore-check.kube-linter.io/unset-cpu-requirements: "single-node policy"
  ignore-check.kube-linter.io/unset-memory-requirements: "single-node policy"
```

## Pre-commit

```bash
pre-commit install --hook-type commit-msg --hook-type pre-push
pre-commit run --all-files
```

| Hook | Purpose |
|---|---|
| `conventional-pre-commit` | Conventional commit message format |
| `check-yaml` | YAML syntax |
| `detect-private-key` | Block private key commits |
| `kube-linter` | Kubernetes manifest sanity |
| `trailing-whitespace`, `end-of-file-fixer` | Formatting |

## Common Gotchas

1. **ArgoCD waves are per-Application** — Cross-Application ordering needs hooks or separate sync operations.
2. **TunnelBinding `subjects[].name` is a Service lookup key** — A typo here surfaces as `http_status:404` in operator status, not a missing-Service error.
3. **Cloudflare stale TXT records break reconciliation** — Remove orphan `_managed.<fqdn>` TXT records before recreating CNAMEs.
4. **Tor secret format** — Use `data` with raw binary base64; `stringData` double-encodes.
5. **KSOPS needs the age key in the repo-server pod** — Without `sops-age-keys` mounted, manifest generation fails before any sync.
6. **DNS-01 requires external resolvers** — cluster DNS cannot validate Let's Encrypt challenges; the cert-manager controller args above are required.
7. **ARC upgrades can leave a stale listener** — because pruning is disabled for controller-generated listener resources, a chart upgrade can leave a listener referencing a deleted `EphemeralRunnerSet`. The listener then restarts with `ephemeralrunnersets.actions.github.com "<name>" not found`, and `arc-tf` jobs remain queued. Inspect the listener pod's owner and current ARC custom resources before deleting the stale `AutoscalingListener`; deleting only its pod recreates the same broken listener.

## Useful Commands

```bash
kubectl get certificate -A          # cert status
kubectl get challenges -A           # DNS-01 validation
sops -d path/to/secret.yaml         # inspect a secret
argocd app sync <app-name>          # force sync
argocd app get <app-name>           # app status
```

## Related Repositories

- `makeitworkcloud/ansible-site-cluster` — k3s cluster provisioning
- `makeitworkcloud/www` — static site, source of `.onion` documentation
- `makeitworkcloud/shared-workflows` — reusable GitHub Actions workflows
