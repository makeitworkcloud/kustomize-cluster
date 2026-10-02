# Cluster logging

The approved design is cluster-wide reviewed-source eligibility, 30-day retention
and an initial 100Gi node-local allocation. This branch stages the complete Loki,
Alloy and Grafana configuration. It enables no production log source and does not
publish a chart or image. Upstream chart pins are already published.

Follow [Adding a workload](../../docs/adding-a-workload.md) and
[Rollout and rollback](../../docs/rollout-and-rollback.md).

## Two-phase delivery gate

**Phase 1:** register only [native admission policies](../../operators/cert-manager/logging-admission.yaml)
through the existing cert-manager operator overlay. The logging CA/bootstrap
issuers, root Certificate, Loki/Alloy Applications, Grafana integration and
Reloader watch patch are authored but **not registered**. No logging issuer or
key becomes available merely by applying this phase.

**Activation is a separate reviewed PR.** Before making the logging CA available,
verify that the target API accepts the policies and their `Deny` bindings, denies
an unauthorized issuance attempt specifically because of the policy, and admits
an authorized control. Use explicitly approved, non-persisting server-side dry-run
requests from an approved client; do not retrieve keys or use malformed requests
whose schema/webhook rejection could masquerade as policy enforcement. Inventory
unexpected existing logging PKI resources by metadata only: admission is not
retroactive. That inventory must explicitly name every ClusterIssuer of any name
whose CA secret reference is `logging-root-ca` and every namespaced Issuer in
`cert-manager` referencing that root Secret — configuration references only,
never Secret data. Any such alias must be absent or resolved before the CA
exists, because the policies are not retroactive against issuers that already
reference the root Secret at activation. API version, resource acceptance, sync
waves and CI success are not proof of effective denial in the target cluster.

The observed target is k3s v1.31.13, compatible with the GA policy API introduced
in Kubernetes 1.30. The MCP identity cannot list ValidatingAdmissionPolicies;
no bypass was attempted and live enforcement remains unverified.

After that gate, activation can register [logging PKI](../../operators/cert-manager/logging-pki.yaml),
[Loki](../apps/loki-app.yaml), [Alloy](../apps/alloy-app.yaml), the
[Grafana client Certificate](../grafana/loki-client-certificate.yaml) and
[Loki datasource](../grafana/loki-datasource.yaml), and apply the staged
[Reloader watch patch](../../operators/reloader/logging-watch-patch.yaml).
Source opt-ins remain separate owner-reviewed changes after backend acceptance.

## Ownership and reconciliation

| Producer/integration | Consumer | Current stage |
| --- | --- | --- |
| Loki chart 18.13.7, app 3.7.8 | Loki Application | Published upstream; pin authored, Application unregistered |
| Alloy chart 1.13.0, app v1.20.0 | Alloy Application | Published upstream; pin authored, Application unregistered |
| Native admission policies (5) and bindings (5) | Target API server | Registered in operator desired state; not deployed or verified |
| Dedicated cert-manager PKI | Workload Certificates | Staged, unregistered; no logging keys generated |
| Logging overlays | Chart-backed workloads in namespace `logging` | Staged; certificate and monitoring integration |
| Grafana datasource/client Certificate | Main authenticated Grafana | Staged, unregistered; public status Grafana excluded |
| Reloader chart 2.2.16 | Certificate renewal | Current watch scope unchanged; future `logging` watch patch staged |
| OpenCode chart, storage, credentials and exporter | OpenCode | Unchanged; no OpenCode log source enabled |

One object has one owner. A repository overlay does not implicitly patch another
Argo source's chart output. Independent operator/workload/Grafana Applications
are not ordered by sync waves. After approved merge, main `test` success causes
root-sync submission, not final health or functional acceptance.

At activation, the workload Applications create `logging`. Reloader Roles for
that namespace and Certificates may initially retry while dependencies become
available. Verify every affected root/child and watch Role; do not promise these
interactions are harmless or guaranteed to complete. Merge and rollout are
separate approval gates in both phases.

## Issuance and authentication

Loki's native HTTP listener on 3100 requires a client certificate signed by the
logging CA, including direct service access. There is no public route, gateway
bypass or assumption that ClusterIP authenticates clients. Both issued client
certificates are trusted for the single tenant; they are not separate read/write
authorization roles.

Admission guardrails restrict Certificate creation/spec changes to the Argo
application-controller identity and fixed profiles. CertificateRequests require
the cert-manager controller and the matching Certificate owner/profile. Direct
built-in CSR paths for the logging signers are denied. Protected issuer/root
changes cannot escape via a changed issuer reference. Metadata-only updates use
deep spec equality, not a mutable generation counter. The policies also close the
alternate-issuer path to the same root Secret: any ClusterIssuer, under any name,
whose `ca.secretName` is `logging-root-ca` is guarded on both new and pre-existing
objects, with only the approved `logging-ca` ClusterIssuer permitted, and
namespaced Issuers in `cert-manager` referencing that root Secret, new or
pre-existing, are denied. The staged set is five ValidatingAdmissionPolicies with
five bindings and adds no controller. Unrelated PKI is out of
scope. Existing RBAC remains necessary; these controls do not defend against a
compromised CA controller or administrator able to alter policy/read the CA key.

Only after approved activation does cert-manager generate dedicated keys:
`logging-root-ca` in `cert-manager`, `loki-server-tls`/`loki-alloy-client` in
`logging`, and `loki-grafana-client` in `grafana`. No production keys are generated,
retrieved, printed, decrypted or committed by this preparation workflow.

The root lasts ten years and retains its key; root trust migration is manual and
reviewed, not automatic. Leaf duration is 90 days with key rotation. Alloy mounts
its client files; Grafana Operator consumes TLS fields through Secret references,
with verification enabled and selector `dashboards: grafana` only.

The staged Reloader patch adds only `logging` to the existing namespace scope,
without changing chart version or global strategy. Named leaf Secret annotations
select renewal consumers. Applications ignore only the exact controller-generated
renewal hash entries and documented annotation, not general environment, Secret
or configuration changes. Verify actual issuance, renewal, trust propagation and
reload before claiming those mechanisms work; never delete the CA as a shortcut.

## Retention and capacity

The staged Loki is one monolithic process using a chart-managed 100Gi `local-path`
PVC. This is not a quota, backup or HA. Node/disk loss can lose all retained logs.
Compactor retention is 720h with a 24h TSDB index; deletion is asynchronous.
Thirty days is a bounded target, not complete history at arbitrary traffic.

Accepted sustained ingest is limited to 0.02MiB/s with a 1MiB burst, roughly
50.6Gi raw over 30 days before index/WAL. Stream, line, query and retry bounds can
cause rejection or loss. No rejection, retention, capacity or throughput result
is claimed from desired state alone.

The provisioner path `/var/lib/rancher/k3s/storage` is on the separate
`/var/lib/rancher` filesystem. Monitor that filesystem, not the small OS root or
an unmounted subdirectory. Local-path directory provisioning does not establish
a hard quota; the StorageClass uses `Delete` reclaim and has expansion disabled.
Desired PVC retention is `Retain` on StatefulSet deletion/scaling, with
`Prune=false,Delete=false`. Verify rendered/bound behavior. Manually deleting the
PVC can still delete storage; rollback does not authorize volume deletion.

Staged monitoring reports data-filesystem headroom below 20%, ingest discards
and absent/down scrape targets. Alerts perform no automatic mutation. Before
widening sources, measure actual ingestion/storage growth; review source opt-out
or a separately approved budget/retention change rather than blindly raising
limits.

## Listeners and privacy

The main HTTP listener is mTLS-only. Internal gRPC, memberlist and the
single-process ring/frontend worker are loopback-bound. The operational 3101
listener serves readiness, metrics, build info and ring status, not log query or
push. Kubernetes probes and the Loki ServiceMonitor use it; Alloy monitoring
uses 12345. CI and live acceptance must verify this separation.

Every source requires review and a later Pod-template annotation
`logging.makeitwork.cloud/approved: "true"`, plus a stable
`app.kubernetes.io/name`. Filters apply before API log tailing. `opencode`, `mcp`,
`arc-runners` and containers named `runner` remain hard-denied even if annotated.
Removing those exclusions is a separate decision.

Indexed labels are only cluster, namespace, application and container. Pod names
and UIDs are internal API targeting metadata. Downstream redaction does not prove
source content safe: prompts, outputs, tool arguments/results, credentials,
session/user IDs and directories remain excluded. No raw logs belong in PRs,
chat or knowledge. No paid inference is needed for tests.

API collection covers Pod logs, not node journald or host services. It needs no
hostPath, root or Secrets API grant. Cluster-wide Pod-log read permission remains
a collector trust boundary; opt-in filters do not constrain a compromised
collector's permissions.

## Validation and remaining gates

The reusable validation workflow is now published: `.github/workflows/test.yml`
calls `.github/workflows/logging-checks.yml` at the same committed revision; the
existing test jobs are retained. The reusable workflow is `workflow_call`-only
with `contents: read` and no Secret exchange. The child workflow is authored and
committed with the intended controls — strong-TLS oracle plus positive,
negative, escape and unrelated-resource cases against the real Kind API,
including ownerUID/client identity checks — but CI has not yet executed on this
branch and no PR is open; final reviews and passing CI remain required before
calling this branch CI-ready.

The intended checks render pinned Loki, Alloy and prospective Reloader values;
validate native Alloy/Loki configuration; test synthetic mTLS authorized
push/query and TLS-level rejection; verify operational-route separation and
remote RPC refusal; and exercise the real staged source admission guard policies
(Phase 1) on a native Kind isolated Kubernetes API at version 1.31 with positive,
negative, escape and unrelated-resource controls — this is not proof of actual
target policy enforcement. Synthetic CI keys/kubeconfig material must never be
printed or uploaded.

CI does not prove target policy enforcement, cert-manager issuance/renewal,
real scrape discovery, bound-volume retention, production source safety, disk
growth or 30-day deletion. Those are separate live gates using authorized
Argo/Kubernetes/Grafana evidence and approved protocol tests, with no implicit
exec, restart, key dump or paid inference. The target policy denial proof must
precede logging issuer activation, not merely precede source opt-in.

Rollback is a reviewed desired-state change. Remove source opt-ins before
collector changes and preserve volume/trust material for recovery. Do not
cascade-delete, reset storage, rotate keys or manually sync without exact approval.
