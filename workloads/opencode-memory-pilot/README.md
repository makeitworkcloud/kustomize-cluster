# opencode-memory-pilot (staged, inactive)

Internal memory-pilot workload evaluating the OpenCode memory plugin. The
owner-selected design is now **OpenAI HTTP embeddings**, replacing the
original local-embeddings proposal. This directory is **staged preparation
only**: nothing here is referenced by `workloads/apps/kustomization.yaml` or
any other active kustomization, so merging this branch creates no Application,
no workload, no storage, and no Service in the cluster. The staged files
deploy nothing by themselves; activation is the separate owner-confirmed
procedure in [Activation gates](#activation-gates-owner-confirmed-separate-pr).

This change prepares the secure owner handoff and CI guards, **not a fully
wired remote-embeddings deployment**. The staged Application still pins
`opencode-server` **0.4.1** and keeps its existing two-secret selector
contract. That pin cannot run the remote design. Proposed chart **0.4.6 is
not yet published**; final generator wiring and the pilot pin/configuration
change remain blocked on owner-provisioned encrypted files and a published
chart. No credentials have been retrieved or their existence verified.

## What this is

- Opt-in evaluation of the chart's `memoryPilot` mode. In pilot mode the
  chart renders only an isolated ConfigMap and Deployment; it does not
  touch the production OpenCode server, its agents, MCP credentials, or
  artifacts.
- The proposed chart 0.4.6 uses `opencode-mem` **2.26.0** in the stock
  OpenCode image with OpenAI HTTP embeddings: model
  `text-embedding-3-small`, **1536 dimensions**, `taskPrefixes: false`,
  and `autoCapture: false`. No new image, embeddings Service, or sidecar
  is required. These are the selected target settings, not settings wired
  into the current staged 0.4.1 Application.
- The only Service remains ClusterIP `opencode-memory-pilot` on port 4096,
  with no TunnelBinding, ingress, or NodePort.
- The provider model remains `zai-coding-plan/glm-5.3`. Its dedicated
  `ZHIPU_API_KEY` and the pilot server-auth password remain prerequisites;
  their existence is unknown. The pilot does not reuse production OpenAI
  OAuth or any production provider credential.
- **Synthetic data only.** Disabling `autoCapture` does not disable raw
  prompt persistence: synthetic prompts are still stored as `chatMessage`
  records. Embedding input is sent to OpenAI, and paid synthetic testing
  requires separate approval. Isolation is structural: the pilot mounts
  none of the existing production `opencode-home` or artifacts claims and
  has no MCP wiring to production agents. This is not an RBAC or
  network-isolation proof, and no NetworkPolicy enforcement is claimed.

## Files

| File | Role |
|---|---|
| `kustomization.yaml` | Overlay entry: PVC + Service only, no generators until owner-provisioned encrypted files exist. Deliberately does not include `application.yaml`. |
| `persistent-volume-claim.yaml` | `opencode-memory-pilot-home`, 10Gi, ReadWriteOnce, cluster-default storage class (k3s `local-path`), matching the existing claim convention (`opencode-home`). |
| `service.yaml` | ClusterIP `opencode-memory-pilot`, port 4096 only, selector `app: opencode-memory-pilot`; verify against the selected published chart at activation. |
| `application.yaml` | Staged child Application, unchanged at 0.4.1, intentionally excluded from `workloads/apps` and this overlay's kustomization. |
| `README.md` | This document. |

The Application sources this directory from `main`, targets namespace
`opencode`, and deliberately declares **no `automated` sync policy**.
Registration and manual sync are separate owner-confirmed activation steps.

## Owner credential handoff

Use the owner's trusted SOPS provisioning process, outside agent access.
Never paste a key or password into chat, shell arguments, logs, or review
comments; do not print environment values or decrypt to stdout. Keep
Kubernetes metadata clear and commit only encrypted credential values in
SOPS-encrypted manifests, never plaintext stubs or placeholders. Do not
retrieve existing production secrets to populate the pilot.

All paths below are relative to `workloads/opencode-memory-pilot/`. Each
manifest must be a `v1` `Secret`, `type: Opaque`, namespace `opencode`, with
exactly the listed `stringData` key:

| Owner-provisioned file | Secret name | `stringData` key | Purpose |
|---|---|---|---|
| `opencode-memory-pilot-embeddings-secret.yaml` | `opencode-memory-pilot-embeddings` | `apiToken` | Dedicated OpenAI embeddings API key. |
| `opencode-memory-pilot-provider-secret.yaml` | `opencode-memory-pilot-provider` | `ZHIPU_API_KEY` | Isolated provider key for `zai-coding-plan/glm-5.3`. |
| `opencode-memory-pilot-server-auth-secret.yaml` | `opencode-memory-pilot-server-auth` | `password` | Isolated HTTP Basic auth password for the pilot server. |

The existing `.sops.yaml` fallback encryption regex covers `apiToken`; no
policy change is needed. The encrypted files must follow the **first
matching creation rule**, including its existing public age recipient and
`encrypted_regex`. The embeddings key is intended to reach only the pilot
runtime as **`OPENCODE_EMBEDDING_API_KEY`** through a Secret key reference in
the later chart configuration. Never set global `OPENAI_API_KEY`, embed the
key in chart values, or reuse production OpenAI OAuth. This runtime wiring
is not present in the staged 0.4.1 Application.

After encrypted files are actually committed, a reviewed follow-up may add
`ksops-opencode-memory-pilot-secrets.yaml` with `apiVersion: viaduct.ai/v1`,
`kind: ksops`, and metadata name `ksops-opencode-memory-pilot-secrets`.
Its `files` list must contain exactly the owner-provisioned encrypted files
actually committed from the table, with no missing references. Only then
may `kustomization.yaml` declare that one generator. Until then, leave the
current five-file overlay and its no-generators state unchanged. Do not
modify production KSOPS wiring.

CI allows this validated optional set while continuing to require
inactivity. It checks encrypted shape, first-match policy, and exact
references without decryption or credential logging. Partial provisioning
is permitted for staged handoff only; **all three credentials are required
before activation**. Missing credentials are explicitly pending, not proof
of readiness. Passing shape checks proves neither credential validity nor
cluster-side decryptability.

## Storage and state model

- The PVC holds the plugin's persistent state, seeded at
  `/home/opencode/.opencode-mem/data`, alongside OpenCode session data on
  the dedicated pilot home PVC. Sessions are outside the backup scope
  below, not outside the claim. Production `opencode-home` is never mounted.
- The chart Deployment uses `Recreate` with a single replica. This merely
  minimizes overlap during Deployment-managed updates; it provides no
  crash-consistency or lock-safety guarantee for the ReadWriteOnce volume.
- Application values remain name selectors only (`providerSecretName`,
  `serverSecretName`), with fixed keys (`ZHIPU_API_KEY`, `password`) under
  the staged 0.4.1 contract. CI deliberately keeps that contract until a
  separate reviewed published-chart pin/configuration change.
- If old local-engine vectors exist, switching embedding models requires
  an explicit migration/re-embedding decision. Do not assume compatibility
  or delete vectors, databases, or the PVC without owner approval. No
  migration or deletion is authorized by this preparation.

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

1. The target `opencode-server` chart (proposed 0.4.6) is published to
   `ghcr.io/makeitworkcloud/charts`; its schema and rendered pilot settings
   are verified. A separate reviewed change replaces the staged 0.4.1 pin
   and configures the remote design, including the runtime-only embeddings
   Secret reference and matching CI contract. Do not activate 0.4.1 as a
   substitute for the remote design.
2. All three isolated pilot credentials are owner-provisioned through
   separately approved SOPS-encrypted manifests and the repository's KSOPS
   process, with complete generator wiring and no missing references.
   Their existence, validity, and decryptability are not established here.
3. Controlled synthetic-client access is approved: only the operator-run
   client connects using the dedicated pilot Basic auth credentials. Obtain
   explicit approval for the **exact paid synthetic test**, its target and
   scope, before making provider or OpenAI embedding calls. No paid test is
   authorized or performed by this preparation.
4. Repository CI and the review checks from `docs/adding-a-workload.md`
   pass on the later PR. Verify the selected published chart's pilot pod
   labels match `app: opencode-memory-pilot` and expose the named `http`
   container port 4096 targeted by the staged Service. Runtime HTTP
   embeddings and persistence behavior remain untested here.
5. Any existing-vector migration is separately approved; no automatic
   deletion or reset is allowed. Backup hardening remains deferred as above.
6. An owner-confirmed activation PR registers the Application in
   `workloads/apps` per repository convention and updates the inactivity
   guard accordingly. This preparation does not register it.
7. The owner explicitly approves the exact manual sync and confirms health.
   Keep no `automated` sync policy; registration is not sync authorization.

Production chart pinning is unaffected: chart publication continues its
independent production pin PR and auto-merge pipeline. That pipeline pins
only the production `opencode` Application; it does not select the pilot
version, register it, or activate it. Pilot selection remains a separate
reviewed operation.

## Isolation summary

- Names: all pilot resources are `opencode-memory-pilot*`; no production
  object is claimed or patched.
- Credentials: distinct embeddings key, provider key, and server password;
  no production OpenAI OAuth or MCP gateway credential reuse.
- Storage: pilot state lives only on `opencode-memory-pilot-home`; existing
  production `opencode-home` and artifacts claims are never mounted.
- Network: ClusterIP port 4096 only, with no TunnelBinding, ingress, or
  NodePort. The boundary is dedicated HTTP Basic auth, not a claimed
  NetworkPolicy; the remote design sends embedding input to OpenAI.
- Data: synthetic prompts only, still persisted as raw `chatMessage`
  records despite `autoCapture: false`; no mounts or MCP wiring expose
  pilot data to production agents.
