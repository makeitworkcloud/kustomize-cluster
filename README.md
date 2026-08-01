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

App hostnames have two coordinated declarative control points:

- `tfroot-cloudflare/cf-tunnels.tf` declares the CNAME at Cloudflare.
- Each `TunnelBinding` declares the in-cluster route and cloudflare-operator
  tracks it with a `_managed.<fqdn>` TXT record.

The hostname lists must stay aligned. To retire a route, remove and reconcile
the `TunnelBinding` first so the operator can clear its ownership record, then
remove the Terraform hostname and review a narrow OpenTofu plan. Do not delete
only the CNAME: a stale managed TXT record causes Cloudflare error `81044`.
`subjects[].name` must match the real Kubernetes `Service` name in the same
namespace.

## Authentication

GitHub OAuth provides SSO for ArgoCD, Grafana, Forgejo, and kubectl. Dex issues
kubectl tokens for the public `kubectl` client defined in
`bootstrap/argocd-config.yaml`; the API server validates that audience and maps
the `makeitworkcloud:admins` GitHub team to cluster-admin through
`bootstrap/oidc-rbac.yaml`. CI uses a separate `ci-deployer` ServiceAccount.

### kubectl access

Access has two independent gates:

1. Cloudflare Access authorizes the TCP connection to
   `k3s.makeitwork.cloud` using the policy in `tfroot-cloudflare`.
2. `kubelogin` obtains a Dex token. Dex includes the requested `email` and
   `groups` claims, which the API server and Kubernetes RBAC validate.

Install `cloudflared`, `kubectl`, and the `kubectl oidc-login` plugin. Confirm
all three commands are available before continuing. Start the local TCP proxy
in a dedicated terminal:

```bash
cloudflared access tcp --hostname k3s.makeitwork.cloud --url localhost:6443
```

If port 6443 is occupied, choose another local port and use the same port in
the kubeconfig server URL.

Create a dedicated kubeconfig such as `~/.kube/makeitworkcloud-k3s.yaml`, mode
`0600`. Its cluster entry must point to `https://127.0.0.1:6443` and trust the
public k3s server CA supplied by an administrator. That CA may be copied from
`/var/lib/rancher/k3s/server/tls/server-ca.crt` on the k3s VM through an
approved channel. The kubeconfig must not contain a token, client certificate,
or client key. Do not copy `/etc/rancher/k3s/k3s.yaml` off the node: it contains
cluster-admin client credentials.

Copy `docs/kubeconfig.example.yaml` to that dedicated path, replace its
`certificate-authority` placeholder with the absolute path to the approved CA
file, and set mode `0600`. Its user exec credential is:

```yaml
user:
  exec:
    apiVersion: client.authentication.k8s.io/v1
    command: kubectl
    interactiveMode: IfAvailable
    args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://argocd.makeitwork.cloud/api/dex
      - --oidc-client-id=kubectl
      - --oidc-extra-scope=email
      - --oidc-extra-scope=groups
      - --oidc-pkce-method=S256
      - --token-cache-storage=keyring
```

Give this cluster a distinct context name such as `makeitworkcloud-k3s`; never
reuse an unrelated production or staging context. With the proxy running,
first confirm the dedicated file's current context, server, and user name
without displaying credentials:

```bash
export KUBECONFIG="$HOME/.kube/makeitworkcloud-k3s.yaml"
kubectl config current-context
kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}{"\n"}{.users[0].name}{"\n"}'
```

The expected context is `makeitworkcloud-k3s`, the server is
`https://127.0.0.1:6443`, and the user is the dedicated OIDC exec user. Then
verify the authenticated identity before performing any change:

```bash
kubectl --context makeitworkcloud-k3s auth whoami
kubectl --context makeitworkcloud-k3s auth can-i '*' '*' --all-namespaces
```

`auth whoami` should show your email and the `makeitworkcloud:admins` group;
`auth can-i` should return `yes`. When finished, stop the local `cloudflared`
process. OIDC and Cloudflare Access sessions expire independently; do not run a
global credential-cache cleanup unless you have reviewed its scope.

For break-glass access, SSH to the k3s VM through `hero.makeitwork.cloud` and
run kubectl there with `/etc/rancher/k3s/k3s.yaml`; see `tfroot-libvirt`.

The App-of-Apps Applications reconcile `bootstrap/secrets`, `operators`, and
`workloads/apps`. They do not reconcile the rest of `bootstrap/`. Changes to
the ArgoCD CR, OIDC RBAC, or CI ServiceAccount therefore require a separately
reviewed bootstrap apply or node provisioning; an ordinary ArgoCD sync is not
enough.

## SOPS / KSOPS

Secrets are age-encrypted with field-level selective encryption. The `.sops.yaml` `encrypted_regex` targets only sensitive values (tokens, passwords, OAuth client secrets) so metadata stays diffable.

### sops-secrets-operator validation path

`operators/` installs `sops-secrets-operator` as the highest-priority child Application within the actively reconciled `gitops-operators` tree (`sync-wave: "-2"`). This is intentionally additive: KSOPS remains active until KMS-backed `SopsSecret` resources are proven and every existing age-encrypted Secret has been ported.

Future KMS-backed `SopsSecret` manifests should use the SOPS KMS recipient managed by the [`makeitworkcloud/tfroot-aws`](https://github.com/makeitworkcloud/tfroot-aws) repository with `encrypted_suffix: Templates`. Do not copy raw KMS key identifiers into docs or chat; use the applied OpenTofu output locally when encrypting migration manifests.

For a full KSOPS deprecation with no secret bootstrap loop, the operator should use ambient AWS auth, not static AWS access keys in a Kubernetes Secret. Because the cluster API is not publicly reachable, the target design is a small public static Kubernetes ServiceAccount OIDC issuer endpoint for AWS STS discovery/JWKS, not public cluster access. The endpoint is hosted outside the cluster by the `makeitworkcloud/www` static site at `https://makeitwork.cloud/oidc`, with discovery metadata at `/oidc/.well-known/openid-configuration` and JWKS at `/oidc/openid/v1/jwks`.

That endpoint serves only public metadata: `/.well-known/openid-configuration` and `/openid/v1/jwks`. The private ServiceAccount signing key remains host-local to k3s/provisioning, k3s issues projected ServiceAccount tokens with the public issuer URL, and AWS IAM reads only the public discovery document and JWKS. This removes the Kubernetes/GitOps AWS-credential bootstrap loop, but it does not eliminate the host-level trust root required for k3s to sign ServiceAccount tokens. Static access keys are acceptable only for short-lived validation, not as the target endstate.

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
