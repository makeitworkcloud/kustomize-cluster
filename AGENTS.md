# Agent Context for kustomize-cluster

## Repository Overview

GitOps repository for OpenShift CRC cluster. Uses ArgoCD with KSOPS for secret decryption.

## Sync Wave Architecture

```
Wave 0: Bootstrap and cluster baseline configuration
Wave 1: Operator and CRD provider layer
Wave 2: Workload layer depending on installed operators
PostSync: Follow-up operational automation
```

- Sync waves are per-Application, not global across all Applications

## Domain Architecture

| Domain | Access Method | TLS Handling |
|--------|---------------|--------------|
| `*.makeitwork.cloud` | Cloudflare Tunnel | TLS terminated at Cloudflare edge |
| `*.apps.makeitwork.cloud` | WARP only | Let's Encrypt cert in cluster |
| `api.makeitwork.cloud` | WARP only | Let's Encrypt cert in cluster |

## Key Namespaces

- `openshift-config` - Cluster-level secrets (certs, OAuth configs)
- `openshift-ingress` - Router/IngressController resources
- `openshift-ingress-operator` - IngressController CR
- `cert-manager` - cert-manager controller pods
- `openshift-gitops` - ArgoCD and KSOPS
- `cloudflare-operator-system` - Cloudflare operator, tunnel deployment, DNS API secret

## Certificate Management

Certificates are managed by cert-manager with Let's Encrypt via DNS-01 (Cloudflare).

**Critical:** cert-manager needs external DNS servers for DNS-01 validation because cluster DNS cannot resolve external domains. This is configured via `CertManager` CR:

```yaml
spec:
  controllerConfig:
    overrideArgs:
      - "--dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53"
      - "--dns01-recursive-nameservers-only"
```

**Certificate locations:**
- `openshift-config/wildcard-apps-makeitwork-cloud-tls` - for componentRoutes (console, oauth)
- `openshift-config/api-makeitwork-cloud-tls` - for API server
- Cloudflare API token in `cert-manager/cloudflare-api-token`

**OpenShift config resources:**
- `ingress.config.openshift.io/cluster` - componentRoutes for console/oauth certs
- `apiserver.config.openshift.io/cluster` - API server cert

## Cloudflare Tunnel DNS Management

Public `*.makeitwork.cloud` app DNS records are operator-managed from `TunnelBinding` resources.

- Keep `TunnelBinding.tunnelRef.disableDNSUpdates: false` for operator-managed DNS
- `subjects[].name` must match the real Kubernetes `Service` name in the same namespace
- cloudflare-operator stores ownership metadata in `_managed.<fqdn>` TXT records
- Do not delete CNAME records without deleting matching `_managed.<fqdn>` TXT records; stale TXT `DnsId` values cause reconcile failures (`81044`)
- The old `dns-adoption-job` hook is intentionally not used

## SOPS/KSOPS Encryption

Secrets are encrypted with age using **selective field encryption**. Only actual secret values are encrypted; metadata, comments, and non-sensitive configuration remain readable.

### Configuration

The `.sops.yaml` file defines `encrypted_regex` to target only sensitive fields:

```yaml
encrypted_regex: '^(token|api-token|clientID|clientSecret|password|secret|github_token|CLOUDFLARE_API_TOKEN|credentials\.json|.*_SERVICE_KEY|GF_AUTH_GITHUB_CLIENT_SECRET|GF_SECURITY_ADMIN_PASSWORD|dex\.github\.clientID|dex\.github\.clientSecret)$'
```

### File Structure Best Practices

**DO:**
- Create separate Secret files for sensitive values
- Reference secrets from Applications/CRDs by name
- Keep non-secret manifests completely unencrypted
- Use comments in secret files to document purpose

**DON'T:**
- Encrypt entire Kubernetes manifests (configs, Namespaces, RBAC)
- Mix secrets with configuration in the same file
- Encrypt metadata fields (names, namespaces, labels, annotations)

### Example: Proper Secret Structure

```yaml
# GitHub OAuth for ArgoCD - encrypted with sops
apiVersion: v1
kind: Secret
metadata:
  name: argocd-github-oauth
  namespace: openshift-gitops
  labels:
    app.kubernetes.io/part-of: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
type: Opaque
stringData:
  # Only these values are encrypted
  dex.github.clientID: Ov23liV3VghvjBnQjsWQ
  dex.github.clientSecret: ae75f6c64ba9833bf7323c205f7b5ea368390788
```

### Commands

```bash
# Encrypt a file (applies encrypted_regex from .sops.yaml)
sops -e -i secret.yaml

# Decrypt for viewing (stdout only, doesn't modify file)
sops -d secret.yaml

# Edit an encrypted file (decrypts in editor, re-encrypts on save)
sops secret.yaml

# Check if encryption worked correctly
sops -d secret.yaml | grep -E "(apiVersion|kind|metadata|name|namespace)"
```

### Adding New Secrets

1. Create a plain YAML Secret file with the sensitive values
2. Run `sops -e -i your-secret.yaml`
3. Verify only the secret values are encrypted (metadata should be readable)
4. Add the file to the appropriate `ksops-*.yaml` generator
5. Never commit unencrypted secret files

### KSOPS Integration

Each directory with secrets has a KSOPS generator file that lists encrypted files:

```yaml
# ksops-example-secrets.yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: ksops-example-secrets
  annotations:
    config.kubernetes.io/function: |
      exec:
        path: ksops
files:
  - github-oauth-secret.yaml
  - api-token-secret.yaml
```

The kustomization.yaml separates resources (unencrypted) from generators (encrypted):

```yaml
resources:
  - deployment.yaml        # Unencrypted manifest
  - configmap.yaml         # Unencrypted config
generators:
  - ksops-example-secrets.yaml  # Decrypts secrets during kustomize build
```

### Migration from Full-File Encryption

If you encounter files where everything is encrypted (apiVersion, kind, metadata):

1. Decrypt the file: `sops -d old-file.yaml > decrypted.yaml`
2. Split into separate files:
   - One for Secret resources (re-encrypt with `sops -e -i`)
   - One for non-secret resources (keep unencrypted)
3. Update the kustomization.yaml to reference new file names
4. Delete the old over-encrypted files

**Key:** `age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l`

## Tor Hidden Services

Managed by tor-controller operator with OnionService CRDs per workload.

**Critical:** Tor keys must use `data` field (not `stringData`) with base64-encoded raw binary. The key file starts with `== ed25519v1-secret: type0 ==`.

Expected .onion addresses are documented in `../www/onion.makeitwork.cloud/index.html`.

## Resource Management

**Single-node CRC policy:** avoid container CPU/memory reservations by default.

- Prefer `resources: {}` or no `resources` block on app containers
- Avoid both `requests` and `limits` unless a workload has a proven stability need
- High requests on single-node CRC commonly trigger `Insufficient cpu/memory` scheduling failures
- CPU limits cause throttling; memory limits can cause avoidable OOM kills

When adding new workloads, default to no container requests/limits:
```yaml
containers:
  - name: app
    image: example/image:tag
    resources: {}
```

For operators installed via OLM (Subscription), tune through supported CR/Subscription fields where available (for example `spec.config.resources: {}` or operator-specific `*_resource_requirements: {}`). If the operator ignores these fields, accept operator defaults.

For operators installed via kustomize remote refs, use JSON patches to remove the entire `resources` block:

```yaml
patches:
  - patch: |
      - op: remove
        path: /spec/template/spec/containers/0/resources
    target:
      kind: Deployment
      name: controller-manager
```

If KubeLinter checks require explicit ignores for this cluster policy:
```yaml
annotations:
  ignore-check.kube-linter.io/unset-cpu-requirements: "No requests on single-node cluster"
  ignore-check.kube-linter.io/unset-memory-requirements: "No limits on single-node cluster"
```

## Pre-commit Hooks

This repository uses [pre-commit](https://pre-commit.com/) to enforce code quality and catch issues before they reach the repository.

### Setup

```bash
# Install pre-commit hooks (run once after cloning)
pre-commit install --hook-type commit-msg --hook-type pre-push

# Verify hooks are installed
ls -la .git/hooks/pre-commit .git/hooks/pre-push
```

### Pre-commit Checks

| Hook | Purpose |
|------|---------|
| `conventional-pre-commit` | Validates conventional commit message format |
| `check-yaml` | Validates YAML syntax |
| `detect-private-key` | Prevents accidental commit of private keys |
| `kube-linter` | Validates Kubernetes manifests |
| `trailing-whitespace` | Removes trailing whitespace |
| `end-of-file-fixer` | Ensures files end with newline |

### Usage

**Before committing:**
```bash
# Run all checks on changed files
pre-commit run

# Run all checks on all files
pre-commit run --all-files
```

**If pre-commit fails:**
1. Fix the reported issues
2. Stage your changes (`git add`)
3. Run `pre-commit run` again to verify
4. Then commit

**Bypass (emergencies only):**
```bash
git commit --no-verify  # Skips pre-commit hooks
```

### Pre-push Protection

The pre-push hook runs all checks before allowing `git push`. This prevents broken code from reaching the remote repository.

## Common Gotchas

1. **OpenShift operators reconcile routes** - Manual patches to routes get reverted. Use proper config resources (`ingress.config.openshift.io`, etc.)

2. **componentRoutes vs IngressController default cert** - Different consumers:
   - `IngressController.spec.defaultCertificate` - expects secret in `openshift-ingress`
   - `Ingress.spec.componentRoutes` - expects secret in `openshift-config`

3. **CertManager CR vs deployment patch** - The CertManager CR's `controllerConfig.overrideArgs` should apply to deployment, but verify with:
   ```bash
   kubectl get deploy cert-manager -n cert-manager -o jsonpath='{.spec.template.spec.containers[0].args}'
   ```

4. **Tor secret format** - Using `stringData` with base64 content causes double-encoding. Use `data` field directly.

5. **ArgoCD sync waves** - Waves only order resources within a single Application. Cross-Application ordering requires hooks or separate sync operations.

6. **OAuth Replace=true causes sync failures** - The `argocd.argoproj.io/sync-options: Replace=true` annotation causes ArgoCD to delete+create resources. OpenShift protects singleton resources like `oauths.config.openshift.io/cluster` from deletion. Use `ServerSideApply=true` instead for these resources.

7. **Cloudflare stale TXT records break DNS reconciliation** - If `_managed.<fqdn>` TXT records point to deleted CNAME IDs, cloudflare-operator attempts update-by-stale-ID and fails with `Record does not exist. (81044)`. Remove stale `_managed.*` TXT records, then reconcile TunnelBindings.

8. **TunnelBinding subject name is service lookup key** - `subjects[].name` is used to read the Kubernetes Service object. If this name does not exist, operator status falls back to `http_status:404`.

## Useful Commands

```bash
# Check cert status
kubectl get certificate -A

# Check challenges (DNS-01 validation)
kubectl get challenges -A

# Verify cert on endpoint
openssl s_client -connect host:port -servername host 2>/dev/null | openssl x509 -noout -subject -issuer

# Decrypt SOPS secret
sops -d path/to/secret.yaml

# Force ArgoCD sync
argocd app sync <app-name>

# Check ArgoCD app status
argocd app get <app-name>
```

## Related Repositories

- `makeitworkcloud/www` - Static site with .onion address documentation
- `makeitworkcloud/ansible-role-crc` - CRC cluster provisioning
