# MCP Gateway — `gcp` ToolHive Backend

This directory deploys the local `gcp` backend for the MCP gateway. MCPGroup
members carry clean integration keys (`gcp`, `kubernetes`, `argocd`, `apify`,
`aws`, `aws-docs`, `cloudflare-read`, `context7`, `grafana`, `parallel-search`,
`playwright`, `slidespeak`, `terraform-docs`, `twilio-docs`); the `vmcp-gateway`
aggregate exposes their tools name-prefixed with those keys, in-cluster at
`http://vmcp-gateway.mcp.svc:4483/mcp` and externally at `mcp.makeitwork.cloud`
behind Cloudflare Access. `github`, `hero-ssh`, and `codebase-memory` remain
intentionally non-aggregated per-backend Services.

## Workload

- **Image:** `ghcr.io/makeitworkcloud/gcloud-mcp@sha256:1a2e38cf1f1855f445b17a1fe6669045b5a3fa794ca54ccf9a22c0aee68dd1d2`.
- **Identity:** dedicated Kubernetes ServiceAccount `mcp/gcloud-mcp` with a 3600-second projected token for the Google Workload Identity Federation (WIF) provider.
- **Configuration:** non-secret, content-addressed ConfigMaps carry the external-account configuration and restricted `gcloud` command allowlist.
- **Kustomize:** `gcloud-mcp-namereference.yaml` is required because ToolHive stores `podTemplateSpec` as a `RawExtension`, outside Kustomize's default name-reference rules.

## Ownership and Security Boundary

- `tfroot-gcp` owns and applies the Google Cloud WIF provider; this repository consumes it.
- The provider accepts only `system:serviceaccount:mcp:gcloud-mcp` and impersonates `gcloud-mcp@makeitworkcloud.iam.gserviceaccount.com`.
- The command allowlist and GCP IAM roles are independent read boundaries. No credentials or token values are committed.
- `groupRef: gateway` exposes the backend through the existing `mcp.makeitwork.cloud` Cloudflare Access path under the `gcp` integration key; this workload creates no dedicated TunnelBinding or DNS record. The per-integration TunnelBinding subjects were retired when OpenCode moved to the single internal aggregate endpoint; only the aggregate root route remains.
- Owner waiver, 2026-09-09: the aggregate is anonymous to in-cluster callers. The single developer consumer holds the external Cloudflare Access pre-shared key.

## Delivery and Verification

The WIF provider must apply before this workload can authenticate. After a GitOps merge, verify the rendered ConfigMap references, GCP WIF startup, Gateway/Argo reconciliation, and an allowed read-only MCP request. Do not use a write probe without separate approval.
