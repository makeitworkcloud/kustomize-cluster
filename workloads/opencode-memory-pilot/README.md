# opencode-memory-pilot (staged, inactive)

Internal memory-pilot workload evaluating the OpenCode memory plugin with an
in-cluster text-embeddings server. This directory is **staged preparation
only**: nothing here is referenced by `workloads/apps/kustomization.yaml` or
any other active kustomization, so merging this branch creates no Application,
no workload, no storage, and no Service in the cluster. The staged files
deploy nothing by themselves; activation is the separate owner-confirmed
procedure in [Activation gates](#activation-gates-owner-confirmed-separate-pr).

## What this is

- Opt-in evaluation of the `memoryPilot` mode introduced by the
  `opencode-server` chart 0.3.2. In pilot mode the chart renders only an
  isolated ConfigMap and Deployment; it does not touch the production
  OpenCode server, its agents, MCP credentials, or artifacts.
- The chart runs the text-embeddings inference (TEI) container in the same
  pod on loopback only (CPU image `cpu1.9.4`, `nomic-embed-text` model at a
  pinned revision). There is no TEI Service, no TunnelBinding, no ingress,
  and no NodePort; the only Service is the ClusterIP
  `opencode-memory-pilot` on port 4096.
- The provider model is `zai-coding-plan/glm-5.3`, seeded by chart
  configuration. The pilot does not reuse production OpenAI OAuth or any
  production provider credential.
- **Synthetic data only.** The pilot is driven with synthetic prompts; raw
  prompt content is persisted to pilot storage as part of normal plugin
  operation. Production agents have no access to the pilot's resources,
  MCP credentials, secrets, or artifacts, and the pilot mounts none of the
  existing production `opencode-home` or artifacts claims.

## Files

| File | Role |
|---|---|
| `kustomization.yaml` | Overlay entry: PVC + Service only. Deliberately does not include `application.yaml`. |
| `persistent-volume-claim.yaml` | `opencode-memory-pilot-home`, 10Gi, ReadWriteOnce, cluster-default storage class (k3s `local-path`), matching the existing claim convention (`opencode-home`). |
| `service.yaml` | ClusterIP `opencode-memory-pilot`, port 4096 only, selector `app: opencode-memory-pilot` following the chart's `app: <fullname>` label convention; re-verify against the published 0.3.2 templates at activation. |
| `application.yaml` | Staged child Application, intentionally excluded from `workloads/apps` and from this overlay's kustomization. |
| `README.md` | This document. |

The Application pins chart `opencode-server` **0.3.2** (unpublished at staging
time; publication is an activation gate), sources this directory from `main`,
targets namespace `opencode`, and deliberately declares **no `automated`
sync policy** — activation remains a manual, owner-confirmed sync.

## Storage and state model

- The PVC holds the plugin's entire `.opencode-mem` directory. That directory
  is the **primary and complete** pilot state.
- The TEI container's model cache is disposable derived data; it may be
  deleted or rebuilt at any time without pilot-state loss.
- The chart Deployment uses `Recreate` with a single replica, which is what
  makes the ReadWriteOnce claim safe.
- All plugin configuration is chart-seeded; this repository adds no
  ConfigMap, no secret stubs, and no plaintext or encrypted secret material.
  The pilot secrets are provisioned by the owner before activation and are
  distinct from every production secret:

  | Secret | Key | Purpose |
  |---|---|---|
  | `opencode-memory-pilot-provider` | `ZHIPU_API_KEY` | Isolated provider key for `zai-coding-plan/glm-5.3`. |
  | `opencode-memory-pilot-server-auth` | `password` | Isolated HTTP Basic auth password for the pilot server. |

- The `memoryPilot` values keys used by `application.yaml`
  (`providerSecretName`, `providerSecretKey`, `serverSecretName`,
  `serverSecretKey`) must match the published 0.3.2 chart schema; verifying
  them is an activation gate, and CI (`.github/workflows/test.yml`) pins the
  staged contract.

## Backup and recovery (unresolved — acceptance procedure only)

The backup destination and encryption scheme for pilot state are
**unresolved**; no destination is configured and this repository deliberately
makes no blind S3 (or other) guess. Until resolved, the following conceptual
acceptance procedure defines what a backup/restore solution must demonstrate.
This document executes no live commands and implies no whole-home or
credential backup.

1. Quiesce: stop pilot writes by scaling the pilot Deployment to zero
   replicas (or otherwise pausing the plugin) so `.opencode-mem` is at rest.
2. Copy the **full plugin state** (the entire `.opencode-mem` directory) off
   node in encrypted form only; never copy the whole pod home, tokens,
   credentials, or any secret material.
3. Restore into a fresh claim and confirm the plugin resumes with complete
   history: session index, memories, and embeddings intact.
4. Record the decided encryption mechanism and destination here once
   resolved.

Lock-file handling: if the plugin lock references a stale PID after an
unclean stop, inspect the lock and the process table and remove it only as a
deliberate human decision; never auto-delete locks. Plugin export features
are known incomplete and are not a backup substitute. A pod restart (same
claim, TEI cache rebuilds empty) is normal operation and is distinct from
disaster recovery (claim lost; restore from the encrypted off-node backup).

## Activation gates (owner-confirmed, separate PR)

This staging merge deploys nothing. Activation happens only when all of the
following hold:

1. `opencode-server` chart 0.3.2 is published to
   `ghcr.io/makeitworkcloud/charts` and its `memoryPilot` values schema
   matches the contract asserted by `.github/workflows/test.yml`.
2. The owner provisions the two isolated pilot secrets in the `opencode`
   namespace (`opencode-memory-pilot-provider`/`ZHIPU_API_KEY` and
   `opencode-memory-pilot-server-auth`/`password`). They are not created,
   stubbed, or encrypted in this repository.
3. Validations pass: repository CI on the activation PR plus the review
   checks from `docs/adding-a-workload.md` (single ownership, Service
   selector against published templates, storage behavior).
4. An owner-confirmed activation PR registers the Application in
   `workloads/apps` per repository convention (a
   `workloads/apps/opencode-memory-pilot-app.yaml` entry or an explicit
   reference to this directory's staged `application.yaml`), updating the CI
   guard accordingly.
5. The owner manually syncs the `opencode-memory-pilot` Application and
   confirms health; the Application carries no `automated` sync policy, so
   even once registered nothing syncs itself.

Production chart pinning is unaffected: the charts post-publish workflow
continues to pin only the production `opencode` Application, and its
automation-generated PRs never register or activate this pilot Application.

## Isolation summary

- Names: all pilot resources are `opencode-memory-pilot*`; no production
  object is claimed or patched.
- Credentials: distinct provider key and server password; no production
  OpenAI OAuth reuse; no production MCP gateway credentials involved.
- Storage: pilot state lives only on `opencode-memory-pilot-home`; the
  existing production `opencode-home` and artifacts claims are never
  mounted.
- Network: ClusterIP Service only, port 4096, namespace-internal; TEI on
  pod loopback.
- Data: synthetic prompts only, with raw prompt content persisted to pilot
  storage; production agents have no access to any of it.
