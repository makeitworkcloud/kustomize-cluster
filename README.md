# kustomize-cluster

hello world

GitOps manifests for the k3s cluster behind [makeitwork.cloud](https://makeitwork.cloud/). ArgoCD reconciles this repo using KSOPS for inline secret decryption.

## Contributor and operator guides

- [Add a workload](docs/adding-a-workload.md)
- [Roll out or roll back a chart workload](docs/rollout-and-rollback.md)
- [Configure kubectl access](docs/kubeconfig.example.yaml)
- [Author and publish a chart](https://github.com/makeitworkcloud/charts/blob/main/docs/adding-a-chart.md)
- [Configure post-publish GitOps PR automation](https://github.com/makeitworkcloud/charts/blob/main/docs/gitops-update-automation.md)

## Layout

```
bootstrap/   ArgoCD configuration, OIDC RBAC, CI service account, App-of-Apps roots
operators/   Cluster operators that install CRDs (cert-manager, cloudflare, tor, ARC, …)
workloads/   Workload Applications that depend on operator CRDs
```

The root `kustomization.yaml` is for local `kustomize build` testing only. ArgoCD drives production sync from the per-Application sources defined in `bootstrap/`.

## Sync Wave Flow

```
Bootstrap apply: ArgoCD configuration, RBAC, CI, and independent root Applications
gitops-operators: installs operator controllers and CRDs
gitops-workloads PreSync: wait-for-crds blocks until required CRDs exist
gitops-workloads Sync: creates child Applications and direct workload CRs
```

The `bootstrap-secrets`, `gitops-operators`, and `gitops-workloads` root
Applications reconcile independently. Sync waves are local to each Application;
the `gitops-workloads` `PreSync` hook gates its child Application and direct CR
creation until every workload-required operator CRD is available.

## External Traffic

| Domain | Path | TLS |
|---|---|---|
| `<app>.makeitwork.cloud` | HTTP via cloudflare-operator `TunnelBinding` | Cloudflare edge |
| `api.makeitwork.cloud` | HTTPS via `ClusterTunnel` to kube-apiserver; Kubernetes OIDC required | Cloudflare edge |
| `k3s.makeitwork.cloud` | TCP via `ClusterTunnel` to kube-apiserver, gated by Cloudflare Access (migration fallback) | Cloudflare edge |

There is no in-cluster ingress controller and no public IP. Every external entry point is a Cloudflare Tunnel.

### TunnelBinding DNS

`tfroot-cloudflare` owns the bootstrap `api` and `k3s` CNAMEs required to
reach the Kubernetes API before cluster workloads reconcile. Their
`TunnelBinding` manages routes with `tunnelRef.disableDNSUpdates: true`.

Every workload `TunnelBinding` owns its own CNAME and
`_managed.<fqdn>` ownership TXT record through cloudflare-operator. Add and
retire workload DNS through the binding; do not declare it in OpenTofu.
`subjects[].name` must match the real Kubernetes `Service` name in the same
namespace.

## Authentication

GitHub OAuth provides SSO for ArgoCD, Grafana, Forgejo, and kubectl. Dex issues
kubectl tokens for the public `kubectl` client defined in
`bootstrap/argocd-config.yaml`; the API server validates that audience and maps
the `makeitworkcloud:admins` GitHub team to cluster-admin through
`bootstrap/oidc-rbac.yaml`. CI uses a separate `ci-deployer` ServiceAccount.

### OpenCode access

`opencode.makeitwork.cloud` is public at the Cloudflare edge so that
`opencode attach` can use its native HTTP authentication. OpenCode enforces HTTP
Basic authentication with the SOPS-encrypted `opencode-server-auth` Secret; it
does not support Dex/OIDC or SAML authentication. Obtain the password through
the approved secret-access process, store it in a local secure credential store,
and connect without Cloudflare Access:

```bash
opencode attach https://opencode.makeitwork.cloud --username opencode --password "$OPENCODE_SERVER_PASSWORD"
```

Do not put the password in shell history, kubeconfig, or a committed OpenCode
configuration file. After rotating the encrypted Secret, restart the Deployment
so its environment is refreshed.

### kubectl access

Normal access connects directly to `https://api.makeitwork.cloud`. Cloudflare
proxies HTTPS to the in-cluster API Service, while Kubernetes remains the
authentication and authorization boundary. `kubelogin` obtains a Dex token;
Dex includes the requested `email` and `groups` claims, which the API server
and Kubernetes RBAC validate.

Install `kubectl` and the `kubectl oidc-login` plugin. Confirm both commands are
available before continuing.

Create a dedicated kubeconfig such as `~/.kube/makeitworkcloud-k3s.yaml`, mode
`0600`. Its cluster entry points to `https://api.makeitwork.cloud`; the public
Cloudflare certificate uses normal system CA trust. The kubeconfig must not
contain a token, client certificate, client key, or private cluster CA. Do not
copy `/etc/rancher/k3s/k3s.yaml` off the node: it contains cluster-admin client
credentials.

Copy `docs/kubeconfig.example.yaml` to that dedicated path and set mode `0600`.
Its user exec credential is:

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
reuse an unrelated production or staging context. First confirm the dedicated
file's current context, server, and user name without displaying credentials:

```bash
export KUBECONFIG="$HOME/.kube/makeitworkcloud-k3s.yaml"
kubectl config current-context
kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}{"\n"}{.users[0].name}{"\n"}'
```

The expected context is `makeitworkcloud-k3s`, the server is
`https://api.makeitwork.cloud`, and the user is the dedicated OIDC exec user. Then
verify the authenticated identity before performing any change:

```bash
kubectl --context makeitworkcloud-k3s auth whoami
kubectl --context makeitworkcloud-k3s auth can-i '*' '*' --all-namespaces
```

`auth whoami` should show your email and the `makeitworkcloud:admins` group;
`auth can-i` should return `yes`. The OIDC plugin caches tokens in the operating
system keyring; do not run a global credential-cache cleanup unless you have
reviewed its scope.

During migration, the previous Access-gated TCP path remains available at
`k3s.makeitwork.cloud` using `cloudflared access tcp`. Do not remove it until
direct API discovery, watches, logs, exec, copy, and port-forward have been
validated through the HTTPS endpoint.

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

For a full KSOPS deprecation with no secret bootstrap loop, the operator should use ambient AWS auth, not static AWS access keys in a Kubernetes Secret. The interactive public kubectl endpoint is not the stable static issuer AWS STS should depend on, so the candidate design was a small public static Kubernetes ServiceAccount OIDC issuer endpoint hosted outside the cluster by the `makeitworkcloud/www` static site.

That design was evaluated and dropped: the endpoint was provisioned at `makeitwork.cloud/oidc` but never consumed (no AWS IAM OIDC provider, no operator wiring), and a static JWKS goes stale on any k3s signing-key rotation, so the path was removed. If ambient AWS auth for the operator is revisited, it needs a publisher that refreshes discovery metadata and JWKS on key rotation. Until then, static access keys are acceptable only for short-lived validation, not as the target endstate.

```bash
sops -e -i secret.yaml      # encrypt in place
sops -d secret.yaml         # decrypt to stdout
sops secret.yaml            # decrypt → editor → re-encrypt on save
```

The age public key is committed in `.sops.yaml`. The matching private key is loaded into the cluster as the `sops-age-keys` Secret in the `argocd` namespace and consumed by the KSOPS plugin during ArgoCD manifest generation.

## CI/CD

Pull requests and `main` run `.github/workflows/test.yml`, which executes the
hooks in `.pre-commit-config.yaml`: YAML and repository hygiene checks, secret
scanning, and kube-linter. After `main` passes, `.github/workflows/sync.yml`
uses the in-cluster runner to request a sync of `bootstrap-secrets`,
`gitops-operators`, and `gitops-workloads` at the tested SHA. It initiates
reconciliation but does not wait for final health. Follow
[Rollout and rollback](docs/rollout-and-rollback.md) for the publication, review,
verification, failure, and rollback procedure.

## Resource Sizing

This is a single-node cluster. Default to **no `resources` block** on app containers — explicit requests trigger `Insufficient cpu/memory` and limits cause throttling or OOM kills with spare capacity. See `AGENTS.md` for guidance on operators installed via remote refs.

## License

GPLv3
