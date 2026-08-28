# Workload guidance

This directory owns workload Applications and their manifests. Use `workloads/apps` for reconciled application entry points; preserve application boundaries and Argo ordering.

External services use Cloudflare Tunnel resources rather than an in-cluster ingress controller. A workload TunnelBinding must reference an existing Service in the same namespace. Workload tunnel DNS is operator-owned; do not add its CNAME to `tfroot-cloudflare`.

For OpenCode, chart configuration is owned by `charts/opencode-server`. When a new version is published, its post-publish workflow opens or updates a GitOps PR that changes only the consuming Application's pinned chart version. The PR remains subject to this repository's CI and normal merge review; it does not sync Argo CD or deploy directly.
