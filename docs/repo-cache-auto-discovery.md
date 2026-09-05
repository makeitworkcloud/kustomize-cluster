---
title: Repository-cache public-source auto-discovery design
status: draft
owners:
  - makeitworkcloud
created: 2026-09-04
last_reviewed: 2026-09-05
source_repositories:
  - makeitworkcloud/kustomize-cluster
  - makeitworkcloud/tfroot-github
  - makeitworkcloud/images
tags:
  - repository-discovery
  - mcp
  - git-sync
  - design
---

# Repository-cache public-source auto-discovery design

## Scope

This design replaces the manually enumerated Make IT Work Cloud public source set with automatic discovery of eligible public repositories, while preserving the cache's read-only and eventually consistent contract. It does not authorize implementation or change the existing OpenCode endpoint. Discovery stays public-only and never grants cache access to private repositories; the only private cache sources are the owner-approved explicit credentialed allowlist in `workloads/mcp-gateway/repo-cache-sync-private.yaml` (`agent-knowledge` and `channel-project`).

## Observed baseline

**Verified fact (2026-09-04):** `workloads/mcp-gateway/repo-cache-sync.yaml` starts one `git-sync` container per repository because `git-sync` synchronizes one repository per process. The `repo-search` MCP server mounts the shared PVC read-only at `/repos` and reads arbitrary repository roots under that path. Its stable consumer contract is `/repos/<repository>/current`, with the synced commit visible in the adjacent git-sync worktree.

**Verified fact (2026-09-04):** `tfroot-github/main.tf` is the canonical organization repository-policy inventory. At revision `aca473b319f52fd5a562effe71b4516cdbd89009`, `tfroot-twilio` is active and public; `agent-knowledge` and `channel-project` are private; and the three historical Ansible repositories are archived. The current `kustomize-cluster` cache revision `f1085c64a75719c6d4f316e0f0c1862f18568466` lacked a `tfroot-twilio` source, so this change adds it as a static bridge.

## Intended design

### Eligibility and policy

A dedicated in-cluster controller discovers all non-archived public repositories in the `makeitworkcloud` organization. It must use GitHub's public organization-repositories API with complete pagination, conditional requests (`ETag` / `If-None-Match`), and a 15-minute discovery interval. A 15-minute interval needs at most a few unauthenticated API requests per hour at the current organization size; it must not require a GitHub token, Kubernetes API write permission, webhook listener, or external route.

The controller applies this fixed policy before cloning:

1. include only repositories reported public and not archived;
2. reject names that do not safely map to one cache path segment;
3. exclude a reviewed deny list containing at least `agent-knowledge` and `channel-project`, even if their visibility were changed accidentally, because both are owned by the explicit private allowlist (`repo-cache-sync-private.yaml`) and must never be duplicated by uncredentialed public discovery; and
4. synchronize each repository's reported default branch only.

`tfroot-github` remains the canonical owner of repository creation, visibility, and archival. The controller consumes public GitHub metadata; it does not parse, run, or modify OpenTofu state or `tfroot-github` source.

### Cache writer

The proposed `repo-cache-controller` is a small, pinned image owned by `makeitworkcloud/images` and selected by `kustomize-cluster`. It replaces only the static public `repo-cache-sync` writers; the credentialed private-allowlist containers in `repo-cache-sync-private.yaml` remain static desired state, and the filesystem MCP backend, its ToolHive policy, and the OpenCode `repo-search` endpoint remain unchanged.

For every eligible repository, the controller fetches the default branch at the existing 120-second cadence. It stages a complete immutable checkout below that repository's cache root, then atomically publishes the new checkout and flips `current` only after the checkout is complete. The published layout must retain the existing agent-facing form:

```
/repos/<repository>/current -> <immutable checkout at synced commit SHA>
```

A failed fetch preserves the last known-good `current` checkout and records the repository as stale. A repository may be pruned only after a fully paginated, successful discovery response omits it; discovery errors and partial responses must never trigger pruning. No private-repository credential or token may be added to this controller to make a missing repository visible; private repositories reach the cache only through the owner-approved explicit allowlist in `repo-cache-sync-private.yaml`, never through discovery.

The controller exposes aggregate readiness only after its first complete discovery and successful initial synchronization of every eligible source. After that point, transient upstream failures should retain readiness and last-good data, while metrics and logs identify stale repositories and the last successful discovery/sync timestamps. The pod keeps `automountServiceAccountToken: false`, a read-only root filesystem, dropped capabilities, non-root execution, and a writable temporary directory only.

### Staleness and observability

Repository-content staleness remains bounded by one 120-second fetch cycle after discovery. A newly public repository can appear up to 17 minutes after creation (15-minute inventory interval plus one sync cycle). Controllers must expose at least:

- discovery success time and last error;
- eligible, synchronized, stale, and denied repository counts; and
- a per-repository current commit SHA and last-success time.

The MCP read path remains intentionally non-authoritative for remote `HEAD`, branch protection, visibility, or freshly pushed source. Agents continue to record the visible cache SHA and use GitHub MCP for writes and freshness-critical reads; the two allowlisted private repositories are additionally readable through the credentialed cache path.

## Delivery plan and acceptance criteria

1. **Author and validate a controller image:** add source and unit tests in `images`; publish an immutable digest only after its pull-request checks pass. Tests must cover pagination, ETag reuse, archived/private/deny-list filtering, path validation, failed-fetch retention, successful-source-only pruning, and atomic publication.
2. **Shadow the writer:** deploy the controller against a separate PVC and non-advertised filesystem MCP server. Do not allow static and dynamic writers to share one PVC. Verify every existing eligible root, `current` symlink, and source SHA against the static cache within its documented staleness bound.
3. **Cut over desired state:** switch the existing read-only backend to the validated PVC/controller after cluster CI passes. Verify the `mcp-gateway` Application, cache writer, filesystem backend/proxy, and an MCP listing of `/repos/tfroot-twilio/current` separately.
4. **Exercise lifecycle behavior:** create or use an approved temporary public test repository, observe automatic inclusion without a manifest edit, then archive it and observe pruning only after a successful inventory. This is a confirmation-gated organization mutation and is not part of the current change.
5. **Retire static writers:** remove the per-repository public `git-sync` containers only after the cutover and lifecycle checks succeed. Keep the existing static `tfroot-twilio` bridge until then; the credentialed private-allowlist containers in `repo-cache-sync-private.yaml` are outside this design and are not retired by it.

## Open decisions and invalidation conditions

Implementation must select a Git checkout library or bundled Git implementation that can preserve the immutable-worktree and atomic-`current` contract above. It must not rely on undocumented `git-sync` internals or dynamically mutate a Kubernetes Deployment to add containers.

Revisit this design if the organization exceeds one GitHub API page, the cache must support another organization, repository visibility policy changes, a repository needs explicit opt-in instead of all-public inclusion, or the filesystem MCP narrows its `/repos` access model.
