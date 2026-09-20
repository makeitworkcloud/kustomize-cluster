# opencode-memory-pilot (staged, inactive)

Internal memory-pilot workload evaluating the OpenCode memory plugin with
its default local embeddings. This directory is **staged preparation
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
- Embeddings run in-process through the plugin's default local engine
  (`Xenova/nomic-embed-text-v1`). There is no remote embeddings service or
  API and no sidecar; the only Service is the ClusterIP
  `opencode-memory-pilot` on port 4096, with no TunnelBinding, ingress, or
  NodePort.
- The provider model is `zai-coding-plan/glm-5.3`, seeded by chart
  configuration. The pilot does not reuse production OpenAI OAuth or any
  production provider credential.
- **Synthetic data only.** The pilot is driven with synthetic prompts; raw
  prompt content is persisted to pilot storage as part of normal plugin
  operation. Isolation is structural: the pilot mounts none of the existing
  production `opencode-home` or artifacts claims and has no MCP wiring to
  production agents. This is not an RBAC or network-isolation proof, and no
  NetworkPolicy enforcement is claimed.

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

- The PVC holds the plugin's persistent state — the chart seeds `storagePath:
  /home/opencode/.opencode-mem/data` — alongside OpenCode session data: both
  live on this same dedicated pilot home PVC. Sessions are outside the backup
  scope below, not outside the claim. The production `opencode-home` claim is
  never mounted.
- The chart Deployment uses `Recreate` with a single replica. This merely
  minimizes overlap during Deployment-managed updates; it provides no
  crash-consistency or lock-safety guarantee for the ReadWriteOnce volume.
- All plugin configuration is chart-seeded; this repository adds no
  ConfigMap, no secret stubs, and no plaintext or encrypted secret material.
  The pilot secrets are provisioned before activation (see gates) and are
  distinct from every production secret:

  | Secret | Key | Purpose |
  |---|---|---|
  | `opencode-memory-pilot-provider` | `ZHIPU_API_KEY` | Isolated provider key for `zai-coding-plan/glm-5.3`. |
  | `opencode-memory-pilot-server-auth` | `password` | Isolated HTTP Basic auth password for the pilot server. |

- The `memoryPilot` options used by `application.yaml` are name selectors
  only (`providerSecretName`, `serverSecretName`); the data keys
  (`ZHIPU_API_KEY`, `password`) are fixed by the chart. Verifying the
  published 0.3.2 schema is an activation gate, and CI
  (`.github/workflows/test.yml`) pins the staged contract.

## Backup and hardening (deferred)

External backup, encryption, and restore hardening are deferred for this
pilot. No bespoke backup backend or destination must be chosen before the
synthetic pilot runs. Until hardening is done, pilot state exists only on
the dedicated PVC on the cluster node: node or volume loss loses pilot
state, and no node-loss disaster-recovery claim is made.

Future hardening, when the pilot earns it: an operator-only encrypted
off-node copy scoped to the plugin state and raw prompt history — never the
whole home, and never `.auth-token`, `auth.json`, or any other credentials
(those are excluded unconditionally, with no agent retrieval path). Raw
prompt content may be sensitive, so the operator classifies prompt history
before any copy and prefers memory-state inventory data over auth tokens.
Lock files with stale PIDs are inspected and removed only by deliberate
human decision, never auto-deleted; plugin export features are known
incomplete and are not a backup substitute; a pod restart is normal
operation and is distinct from restore. Runtime restore gaps are deferred
and accepted; nothing here is a complete-persistence guarantee.

## Activation gates (owner-confirmed, separate PR)

This staging merge deploys nothing. Activation happens only when all of the
following hold:

1. `opencode-server` chart 0.3.2 is published to
   `ghcr.io/makeitworkcloud/charts` and its `memoryPilot` schema matches the
   contract asserted by `.github/workflows/test.yml`.
2. Local embeddings runtime gate: the plugin's native local embedding path
   (`Xenova/nomic-embed-text-v1`) is confirmed compatible with the stock
   Alpine runtime during activation. No runtime has been tested yet; this is
   a runtime compatibility check, not a persistence property.
3. Credential provisioning: the two isolated pilot secrets are provisioned
   through the canonical GitOps path — separately approved SOPS-encrypted
   Secret manifests merged via the repository's KSOPS process — not by
   manual cluster console changes. No plaintext stub is committed here.
4. Controlled synthetic-client access: during the pilot only the
   operator-run synthetic client connects, using the dedicated pilot Basic
   auth credentials.
5. Validations pass: repository CI on the activation PR plus the review
   checks from `docs/adding-a-workload.md` (single ownership, Service
   selector against published templates, storage behavior).
6. An owner-confirmed activation PR registers the Application in
   `workloads/apps` per repository convention (a
   `workloads/apps/opencode-memory-pilot-app.yaml` entry or an explicit
   reference to this directory's staged `application.yaml`), updating the CI
   guard accordingly.
7. The owner manually syncs the `opencode-memory-pilot` Application and
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
- Network: a ClusterIP Service only, port 4096 — reachable from within the
  cluster, with no TunnelBinding, ingress, or NodePort, so not externally
  reachable. The boundary is the pilot's dedicated HTTP Basic auth; no
  NetworkPolicy enforcement is claimed.
- Data: synthetic prompts only, with raw prompt content persisted to pilot
  storage; no mounts or MCP wiring expose it to production agents.
