# Agent Instructions

## Ownership and safety

`kustomize-cluster` is live GitOps desired state for the Make IT Work Cloud k3s cluster. Argo CD reconciles `bootstrap/secrets`, `operators`, and `workloads/apps` using KSOPS. Treat every manifest change as potentially production-affecting.

Use GitHub MCP for repository work and Argo CD, Kubernetes, and Grafana MCPs for read-only diagnosis. Do not use local `kubectl`, `sops`, `argocd`, or shell assumptions in this headless session. Do not sync, patch, restart, scale, delete, or run an Argo resource action without explicit confirmation.

## Structure and ordering

- `bootstrap/` owns App-of-Apps wiring, Argo CD configuration, OIDC/RBAC, CI bootstrap, and bootstrap secrets.
- `operators/` owns cluster operators and their configuration.
- `workloads/` owns workload applications; `workloads/apps` is the reconciled application entry point.
- Sync waves order resources only within an Argo CD Application. Cross-Application ordering is implemented by App-of-Apps wiring and wait jobs.

Read the closest nested `AGENTS.md` before changing a subtree. Preserve Kustomize layout, KSOPS generators, Argo sync behavior, and existing ownership boundaries.

## Cross-repository ownership

- `charts/opencode-server` owns the packaged OpenCode server chart. Its post-publish workflow opens or updates a `kustomize-cluster` pull request that pins the `opencode` Application to the immutable published version. The generated PR must pass cluster CI and be reviewed and merged before GitOps reconciles it; the automation never syncs Argo CD or deploys directly.
- `tfroot-cloudflare` owns bootstrap/non-tunnel Cloudflare infrastructure. Workload TunnelBinding resources own workload routes and DNS.
- `images` owns shared runner images; `shared-workflows` owns reusable Actions.

## Secrets and validation

Keep Secret data SOPS-encrypted and never expose plaintext, age keys, tokens, kubeconfigs, certificates, or sensitive plan output. Use repository CI as validation evidence. Report affected Application, paths, sync/health evidence, and rollout state.
