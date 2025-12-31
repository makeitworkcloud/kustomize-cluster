# Agent Context for kustomize-cluster

## Repository Overview

GitOps repository for OpenShift CRC cluster. Uses ArgoCD with KSOPS for secret decryption.

## Sync Wave Architecture

```
Wave 0: Bootstrap (ArgoCD config, console branding, OAuth)
Wave 1: Operators (CRDs installed, certs issued)
Wave 2: Workloads (CRs that depend on CRDs)
```

- `wait-for-crds.yaml` Job blocks wave 2 until operator CRDs are ready
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

## SOPS/KSOPS Encryption

Secrets encrypted with age. Each directory with secrets has a KSOPS generator file.

```bash
# Encrypt
sops -e -i secret.yaml

# Decrypt for viewing
sops -d secret.yaml
```

**Key:** `age152ek83tm4fj5u70r3fecytn4kg7c5xca24erjchxexx4pfqg6das7q763l`

## Tor Hidden Services

Managed by tor-controller operator with OnionService CRDs per workload.

**Critical:** Tor keys must use `data` field (not `stringData`) with base64-encoded raw binary. The key file starts with `== ed25519v1-secret: type0 ==`.

Expected .onion addresses are documented in `../www/onion.makeitwork.cloud/index.html`.

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
