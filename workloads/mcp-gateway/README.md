# ToolHive MCP endpoints

This directory deploys the ToolHive backends used by OpenCode and external MCP
clients. OpenCode connects directly to the ClusterIP proxy Services. External
clients use one Cloudflare Access-protected endpoint per selected backend; the
`vmcp-gateway` aggregate remains available only during external-client
migration.

## External endpoints

Each direct endpoint is `https://mcp-<integration>.makeitwork.cloud/mcp` for:

`apify`, `argocd`, `aws`, `aws-docs`, `cloudflare`, `context7`, `gcp`,
`grafana`, `kubernetes`, `parallel-search`, `playwright`, `slidespeak`,
`terraform-docs`, and `twilio-docs`.

The `TunnelBinding` owns workload DNS and routes each hostname to its
ToolHive-generated ClusterIP proxy Service. `tfroot-cloudflare` owns the
matching Cloudflare Access applications. The proxy Services remain internal;
Cloudflare Access is the only external authentication boundary.

## Authentication and security boundary

Owner decision, 2026-09-12: for the solo-developer external-MCP use case, every
direct endpoint may use the existing shared MCP Gateway Cloudflare Access
service token. Clients provide its `CF-Access-Client-*` headers at the edge;
no Cloudflare Access credential is stored in these manifests, and backend API
credentials remain in their existing cluster-owned Secrets.

`github`, `hero-ssh`, and `codebase-memory` remain ClusterIP-only and have no
external TunnelBinding subject. The Cloudflare and provider credentials attached
to other ToolHive proxies remain separate backend authorization boundaries.

## Workload

- **GCP image:** `ghcr.io/makeitworkcloud/gcloud-mcp@sha256:1a2e38cf1f1855f445b17a1fe6669045b5a3fa794ca54ccf9a22c0aee68dd1d2`.
- **GCP identity:** dedicated Kubernetes ServiceAccount `mcp/gcloud-mcp` with a 3600-second projected token for the Google Workload Identity Federation (WIF) provider.
- **GCP configuration:** non-secret, content-addressed ConfigMaps carry the external-account configuration and restricted `gcloud` command allowlist.
- **Kustomize:** `gcloud-mcp-namereference.yaml` is required because ToolHive stores `podTemplateSpec` as a `RawExtension`, outside Kustomize's default name-reference rules.

`tfroot-gcp` owns and applies the Google Cloud WIF provider. The provider
accepts only `system:serviceaccount:mcp:gcloud-mcp` and impersonates
`gcloud-mcp@makeitworkcloud.iam.gserviceaccount.com`.

## Delivery and verification

The Cloudflare Access applications must be applied successfully from
`tfroot-cloudflare` **before** this GitOps change is merged: adding a
`TunnelBinding` subject first would expose a route without its required edge
Access application. This is a cross-repository ordering requirement.

After the Access apply, merge the GitOps route change and separately verify:

1. the `mcp-gateway` Argo CD Application and every generated proxy Service are healthy;
2. each expected CNAME, ownership TXT record, and tunnel route exists;
3. each direct endpoint rejects a request without Cloudflare Access headers;
4. each direct endpoint accepts an authenticated MCP `tools/list` request; and
5. no mutating MCP tool is used as a rollout probe.

Retain `mcp.makeitwork.cloud` and `vmcp-gateway` until every external client
uses direct endpoints. Roll back in reverse order: first remove the direct
TunnelBinding subjects and verify their operator-owned DNS cleanup, then revert
the Cloudflare Access application change through its environment-gated apply.
