# Cluster logging

The owner-approved design is cluster-wide **eligibility**, 30-day retention and a
100Gi node-local allocation. This branch authors the logging foundation; it does
not opt in any production source. Native CI, approved merge, reconciliation and
live acceptance are separate stages. Upstream charts are already published; this
change publishes no chart or custom image.

Follow [Adding a workload](../../docs/adding-a-workload.md) and
[Rollout and rollback](../../docs/rollout-and-rollback.md).

## Ownership and delivery

| Producer or integration | Consumer | Delivery stage |
| --- | --- | --- |
| Published Loki chart 18.13.7, app 3.7.8 | [loki Application](../apps/loki-app.yaml) | Exact upstream pin authored; no new publication |
| Published Alloy chart 1.13.0, app v1.20.0 | [alloy Application](../apps/alloy-app.yaml) | Exact upstream pin authored; no new publication |
| [Application registry](../apps/kustomization.yaml) | `gitops-workloads`, then independent `loki` / `alloy` children | Changed; reconciliation automatic only after approved merge and successful main CI |
| [Logging PKI](../../operators/cert-manager/logging-pki.yaml) | Existing cert-manager, then workload Certificates | Changed; issuance asynchronous, not guaranteed by sync waves |
| [Loki overlay](loki/) / [Alloy overlay](alloy/) | Chart-backed workloads in namespace `logging` | Changed; certificate and monitoring integration |
| [Loki datasource](../grafana/loki-datasource.yaml) and [client Certificate](../grafana/loki-client-certificate.yaml) | Grafana Operator and the main authenticated Grafana | Changed; independently reconciled by the `grafana` child |
| Existing Reloader | Loki / Alloy certificate renewal | Chart 2.2.16 and global strategy unchanged; namespace watch list extended to `logging`; narrowly selected leaf Secret reloads added |
| Existing OpenCode chart, storage, credentials and exporter | OpenCode | Unchanged; no OpenCode log source enabled |
| Production source opt-ins | Alloy | None in this change; later reviewed source PRs required |

Every object has one owner. Repository overlays do not implicitly patch another
Argo source's chart output. Sync waves cannot order independent Applications.
Main `test` success leads to automatic root-sync submission; submission does not
prove child health, issued certificates, a provisioned datasource or ingestion.
Merge and live rollout remain separately confirmation-gated.

## Retention and capacity

- Loki is one monolithic process, replication factor one, using filesystem
  storage on a chart-managed 100Gi `local-path` PVC. There is no HA or backup;
  node/disk loss can lose all retained logs.
- Compactor retention is enabled for 720h with a 24h TSDB index period. Deletion
  is asynchronous. Thirty days is a target within the ingestion budget, not a
  guarantee at arbitrary traffic and not billing or complete-consumption history.
- Sustained accepted ingest is limited to 0.02MiB/s, with a 1MiB burst: about
  50.6Gi raw over 30 days, before index/WAL overhead. Limits also bound streams,
  line size and queries. Rejections, collector retry exhaustion and restart can
  lose logs; none of those outcomes has been observed or validated by this source
  document.
- The provisioner path is `/var/lib/rancher/k3s/storage` on the separate
  `/var/lib/rancher` data filesystem. Monitor that filesystem, not the small OS
  root filesystem or an unmounted subdirectory.
- The inspected local-path provisioning creates directories; requested PVC size
  is not a verified filesystem quota. The StorageClass uses `Delete` reclaim and
  has expansion disabled. Do not promise resizing or data recovery from a PVC
  capacity request.
- Desired StatefulSet retention is `Retain` on deletion and scaling, with
  `Prune=false,Delete=false` on generated PVCs. Verify the rendered policy and
  bound volume before relying on it. Manually deleting a PVC can still delete its
  storage; a reviewed Git rollback is not permission to delete the volume.

The monitoring overlay reports data-filesystem headroom below 20%, Loki
rejections/discards and missing/down Loki and Alloy scrape targets. Alerts do not
mutate the system. Before widening source opt-in, measure real ingestion and
storage growth. On sustained rejection or low headroom, review a source opt-out
or a separately approved capacity/retention change; do not blindly raise limits.

## Authentication and certificate lifecycle

Loki's native HTTP listener on 3100 requires a client certificate signed by the
logging CA. This protects direct service access, rather than trusting a gateway
that can be bypassed. There is no public Loki route and no assumption that
ClusterIP or an unverified NetworkPolicy authenticates clients. The single tenant
trusts its issued client certificates; separate certificates are not separate
read/write authorization roles.

The existing cert-manager controller generates dedicated keys **in cluster after
approved deployment**. No production key material is generated, decrypted,
retrieved or committed by this preparation workflow.

- `logging-selfsigned` bootstraps the `logging-root-ca` Certificate and Secret in
  namespace `cert-manager`; `logging-ca` uses that CA.
- The root Certificate lasts ten years and retains its key. Root CA replacement
  is a manual, reviewed trust-migration operation, not an automatic trust-rollover
  claim. Never delete the CA Secret as a troubleshooting shortcut.
- The server Certificate produces `loki-server-tls`; the Alloy client produces
  `loki-alloy-client`, both in `logging`. Grafana's client produces
  `loki-grafana-client` in `grafana`. Leaf duration is 90 days with key rotation.
- Alloy mounts its client files. Grafana Operator `valuesFrom` references the
  certificate, key and CA fields for datasource UID `loki`, with verification
  enabled. The selector is `dashboards: grafana`, not the public status instance.
- Issuer/Certificate CRDs are added to the root availability gate. Certificates
  retry asynchronously; workloads can remain Pending until their Secrets exist.
- Reloader selects only the named mounted leaf Secrets. The two Applications
  ignore only the exact controller-generated renewal hash entries and the
  documented renewal annotation, with `RespectIgnoreDifferences=true`. They do
  not ignore general environment, configuration or Secret changes. The Reloader
  chart and global reload strategy are unchanged; its namespace watch list is
  extended from `grafana`/`opencode` to include `logging`. Operators and
  workloads are independent roots and `logging` is created by the workload
  Applications' `CreateNamespace`, so the initial Reloader Role apply for the
  newly watched namespace may retry until that namespace exists. After rollout,
  verify the Reloader watch Roles and the health of every affected root and
  child; this interaction is not guaranteed harmless.

[Native Loki authentication](https://grafana.com/docs/loki/latest/operations/authentication/)
and [Reloader annotations](https://docs.stakater.com/reloader/main/reference/annotations.html)
explain these boundaries. Live issuance, renewal, client trust and controller
rollout must be verified before claiming that rotation works.

## Listeners and monitoring

The main HTTP listener is mTLS-only. Internal gRPC binds to loopback, with the
single-process in-memory ring, frontend worker and the unused memberlist
listener all bound to loopback as well. An operational HTTP listener on 3101
serves readiness, metrics, build info and ring status, not log query or push
routes. CI must test this separation; do not broaden that operational listener's
routing.

Kubernetes probes use 3101. `loki-internal` and its ServiceMonitor scrape that
listener with explicit job `loki`; Alloy's chart ServiceMonitor scrapes its own
12345 metrics. Health/scrape success is not evidence that any production log
source is approved, that ingestion works, or that 30-day history exists.

## Source eligibility and privacy

A source requires owner review of its emitted operational payloads and a separate
GitOps opt-in. Add `logging.makeitwork.cloud/approved: "true"` to the **Pod template**
of the approved workload, not merely to Deployment metadata.

Alloy requires a stable `app.kubernetes.io/name` label and applies the opt-in
filter before creating the Kubernetes API log source. Namespaces `opencode`,
`mcp` and `arc-runners`, and containers named `runner`, are hard-denied even if
annotated. Removing those exclusions is a separate source/privacy decision.

Final indexed labels are only `cluster=k3s`, `namespace`, `application` and
`container`. Pod names/UIDs are internal API targeting metadata, not indexed
labels. Downstream regex redaction does not prove arbitrary source output safe:
prompts, outputs, tool arguments/results, credentials, session/user identifiers
and directories must remain excluded. Do not copy raw logs into PRs, chat or the
knowledge repository. No paid inference is needed for verification.

API collection covers Pod logs, not host services or node journald. No hostPath,
privileged collector or Secrets API grant is needed. Cluster-wide read-only Pod
log permissions are a collector trust boundary; opt-in filters constrain normal
pipeline behavior, not the permissions of a compromised collector.

## Validation and acceptance

`ci-logging-charts` renders exact pinned charts from the real Application values
and checks ports, PVC retention, TLS references, minimal RBAC, monitoring,
certificate contracts and fail-closed privacy fixtures. `ci-logging-native`
validates the actual Alloy/Loki configurations and runs isolated Loki with
synthetic ephemeral CI certificates to test authenticated push/query,
unauthenticated/wrong-CA denial, operational-route separation and remote gRPC
rejection. No CI test key material is uploaded or copied into desired state.

The privacy fixtures check the shipped fixed relabel rules; native Alloy parsing
is not a live Kubernetes-source ingestion test. CI also cannot prove actual
cert-manager issuance, renewal, bound-volume retention, production source safety,
real scrape discovery, disk growth or deletion after 30 days. No local native
validation is claimed.

After separately approved merge and rollout, use authorized read-only Argo,
Kubernetes and Grafana evidence plus protocol-appropriate tests from an approved
client. Check root and every affected child revision/health, issuer and
Certificate readiness, retained PVC binding, datasource health, collector
metrics, and authorized/denied synthetic access separately. Do not retrieve
production certificate keys, dump session/log payloads, exec or restart workloads
as an implicit validation step. A live renewal test requires an explicitly
approved operation and target.

Rollback is a reviewed desired-state change. Remove source opt-ins before
changing the collector; retain the PVC and dedicated trust material for recovery.
Do not cascade-delete, reset storage, rotate keys or manually sync as a shortcut.

Remaining gates are passing CI and completed reviews, owner merge/rollout
approval, then live PKI/authentication/monitoring acceptance. Every production
source opt-in remains a later review gate.
