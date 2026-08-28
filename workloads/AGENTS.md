# Workload guidance

This directory owns workload Applications and their manifests. Use `workloads/apps` for reconciled application entry points; preserve application boundaries and Argo ordering.

External services use Cloudflare Tunnel resources rather than an in-cluster ingress controller. A workload TunnelBinding must reference an existing Service in the same namespace. Workload tunnel DNS is operator-owned; do not add its CNAME to `tfroot-cloudflare`.

For OpenCode, chart configuration is owned by `charts/opencode-server`; version selection is owned here and requires a separately approved GitOps rollout after chart publication.
