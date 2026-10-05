# Rollout and rollback

This guide covers chart-backed workloads whose immutable OCI version is pinned
in `workloads/apps/<name>-app.yaml`.

## Delivery flow

1. Merge the chart change only after chart CI passes. Publication creates the
   immutable OCI artifact; publication alone does not change cluster desired
   state.
2. If the chart is explicitly configured for post-publish automation, the
   charts updater opens or updates a generated GitOps pull request that pins
   the child Application to the published `targetRevision`. Otherwise, open a
   focused GitOps pull request manually after publication. Neither kind of pull
   request deploys or syncs Argo CD.
3. Review the version, diff scope, ownership boundaries, and repository checks.
   Merge only after the target version exists in the registry and cluster CI
   passes.
4. The merged commit runs the `test` workflow on `main`. On success,
   `.github/workflows/sync.yml` requests a sync of the App-of-Apps roots at the
   tested commit SHA.
5. `gitops-workloads` reconciles the child Application definition. The child's
   automated policy then reconciles the pinned chart and any repository overlay.

The sync workflow returns after submitting the root sync requests. It does not
wait for root or child Applications to reach `Synced` or `Healthy`; verification
is a separate operator step.

## Verify a rollout

Use approved Argo CD and Kubernetes access. Replace every placeholder rather
than treating an existing workload's names as defaults.

1. Confirm the merged child Application specifies the intended immutable chart
   version. For a multi-source Application, inspect the chart source directly:

   ```bash
   kubectl -n argocd get application <application> \
     -o jsonpath='{range .spec.sources[*]}{.chart}{"\t"}{.targetRevision}{"\n"}{end}'
   ```

   A pinned `targetRevision` selects only the chart version; it does not by
   itself prove which application image the workload runs. When the child
   Application sets Helm values that override the image, such as `image.tag`
   or `image.digest`, note the expected override here and verify the rendered
   image on the live workload in step 4.

2. Confirm both the root and child have reconciled, and inspect any reported
   conditions or failed operation:

   ```bash
   kubectl -n argocd get application gitops-workloads <application>
   kubectl -n argocd describe application <application>
   kubectl -n argocd get application gitops-workloads \
     -o jsonpath='{.status.sync.revision}{"\n"}'
   kubectl -n argocd get application <application> \
     -o jsonpath='{.status.sync.revision}{"\n"}{.status.sync.revisions}{"\n"}'
   ```

   The affected root and child must report the expected revision, `Synced`, and
   `Healthy`. For multi-source children, Argo CD may report multiple source
   revisions; verify the chart version and repository revision separately.

3. Inspect the child Application's resource tree in Argo CD. Confirm all
   expected resources are present, no unexpected resources are being pruned,
   and no resource remains Progressing, Degraded, Missing, or OutOfSync.

4. Verify the controller rollout, the rendered image, and the resulting pods:

   ```bash
   kubectl -n <namespace> rollout status deployment/<deployment> --timeout=5m
   kubectl -n <namespace> get deployment <deployment> \
     -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
   kubectl -n <namespace> get pods
   ```

   Confirm every container image matches the expected tag or digest, including
   any `image.tag` or `image.digest` override set on the child Application; the
   verified chart revision alone does not prove the running application image.
   Use the applicable rollout and image commands for a StatefulSet, DaemonSet,
   or other controller instead of assuming every chart creates a Deployment.

5. If reconciliation or startup is not clean, inspect events and bounded logs
   without printing credentials:

   ```bash
   kubectl -n <namespace> get events --sort-by=.lastTimestamp
   kubectl -n <namespace> logs deployment/<deployment> --all-containers --tail=100
   ```

6. Run a protocol-appropriate functional check against `<endpoint>` from an
   approved network location. Validate authentication and a representative
   request without placing credentials in shell history or logs.

## Handle a failed rollout

Locate the failing boundary before changing desired state:

- Publication failure: confirm the immutable version exists in the registry.
- Updater failure: inspect the charts workflow and whether the generated pull
  request has the expected single-version change.
- Cluster CI failure: fix the pull request; do not bypass required checks.
- Root failure: inspect `gitops-workloads`, including the `wait-for-crds`
  `PreSync` job and Application conditions.
- Child failure: inspect its source revisions, operation state, resource tree,
  Kubernetes events, rollout state, and logs.
- Functional failure: preserve evidence and determine whether configuration,
  compatibility, data migration, networking, or the chart caused the failure.

A manual sync may retry the desired Git revision, but it does not create a new
desired state or replace review. Live patches are diagnostic or emergency-only:
automated self-heal can revert drift, prune can remove resources absent from
Git, and root reconciliation can restore the child Application specification.
Any emergency live change must be captured in a reviewed Git change or removed
after diagnosis so Git again matches the cluster.

## Roll back

Rollback is a reviewed desired-state change, not a registry retag or live
Application patch.

1. Identify the previous known-good immutable chart version from Git history
   and confirm that artifact still exists. Review chart compatibility with
   current Secrets, configuration, CRDs, and persisted data. A chart rollback
   cannot undo an incompatible data migration.
2. Open a focused pull request changing the chart source's `targetRevision` in
   `workloads/apps/<name>-app.yaml` to that previous version. Include the failed
   version, reason for rollback, and verification plan.
3. Run and review the normal repository checks. Merge the rollback pull request
   through the protected branch; do not bypass review because the change is
   urgent.
4. After the `test` workflow succeeds, confirm the sync workflow submitted the
   tested SHA. Then wait for `gitops-workloads` to update the child Application
   and for the child to reconcile the previous chart.

Repeat the complete rollout verification after rollback: pinned target version,
root and child revisions, `Synced`, `Healthy`, resource tree, controller rollout,
pods, events, bounded logs, persistent-data behavior, and the functional
endpoint. Record any emergency action and follow-up fix in the canonical Git
history.

## Verify an OpenCode usage-exporter change

This is a cluster-overlay change, not an OpenCode chart publication. After a
separately approved merge, use authorized Argo CD, Kubernetes, and Grafana access:

1. Verify the tested Git revision on `gitops-workloads` and the `opencode` child,
   plus `Synced`/`Healthy`. Verify the `grafana` child separately; its dashboard
   overlay reconciles independently, not through a cross-Application sync wave.
2. Confirm the exporter rollout and expected `checksum/opencode-session-metrics`
   pod-template annotation. Check Ready pods, unchanged image/Secret references,
   unchanged OpenCode chart/server pins, and no unexpected server-pod replacement.
3. Check `up{job="opencode-session-metrics"}` and
   `opencode_session_usage_scan_up` are 1, truncation is 0, and the last-success
   timestamp is positive and fresh. Check the baseline timestamp, scan pages,
   session coverage, duration, and stable errors. Coverage need not exceed 1,000
   if the dataset does not; an exactly-full final page without a continuation
   header is valid. The 15-second scan budget is soft, not a hard HTTP deadline.
4. Expect token panels to show **No data** during failed/stale collection and
   until five-minute rate or one-hour stat warmup completes. Stat queries are
   instant so they cannot retain an older healthy value as the current result.
   After passive normal usage, verify growth for previously observed sessions;
   do not create paid inference traffic merely to force a signal. First-seen
   sessions are baselined without historical credit. Zero in a valid window means
   no observed growth, not proof that all sessions were idle or consumed no tokens.
5. Do not dump session API responses, identifiers, titles, prompts, directories,
   or credentials. Inspect aggregate metrics instead. On failure, compare health,
   truncation/errors and budgets with the release-specific contract; do not delete
   sessions, patch collector state, or raise bounds blindly.

Re-verify the native pagination contract on server upgrades: this repair uses
OpenCode 1.18.29's array response and timestamp continuation header, not the newer
v2 Page envelope. See the pinned [upstream handler](https://github.com/anomalyco/opencode/blob/16747470f976aca3d362ad730bcd3fe82ecc2c9a/packages/opencode/src/server/routes/instance/httpapi/handlers/experimental.ts).
A separately reviewed Git revert restores the prior collector/dashboard behavior;
exporter replacement resets in-memory baselines and counters. Source, static CI,
reconciliation, pod health, and observed token growth remain separate proof stages.

The exporter now speaks both protocol generations behind an explicit
`OPENCODE_API_VERSION` selector (`1` default, `2` opt-in) and never
auto-detects or downgrades between them. Version 1 keeps OpenCode 1.18.29's
`/session/status` and `/experimental/session` array responses with the integer
timestamp continuation header described above. Version 2 targets OpenCode
2.0.22 per the pinned [session
group](https://github.com/anomalyco/opencode/blob/527f0b931d1f9b3ebd34e106c51b31ce5db5b075/packages/protocol/src/groups/session.ts)
and
[handler](https://github.com/anomalyco/opencode/blob/527f0b931d1f9b3ebd34e106c51b31ce5db5b075/packages/server/src/handlers/session.ts):
`/api/session` returns the `{data, cursor:{previous, next}}` envelope with an
opaque base64url cursor, emits `cursor.next` exactly on non-empty pages, and a
complete scan terminates on the first empty page with cursor-cycle detection.
`time.updated` remains an integer epoch-millisecond number
(`DateTimeUtcFromMillis` decodes from a finite number, never an ISO string),
token field names are unchanged, and the global list has no archived or
project filter, so session history is never filtered away. With version 2,
`/api/session/active` maps its `running` entries onto
`opencode_active_sessions{state="busy"}`; V2 exposes no retry aggregate, so
the `state="retry"` series is absent rather than zero while version 2 is
selected.

V2 rollout gate (owner-approved 2026-10-02): the rollout branch couples the
selector flip (`OPENCODE_API_VERSION: "2"`) with the `opencode` Application
pin to chart `opencode-server 0.5.0` in one reviewed PR/merge selecting both
desired versions; Application/resource reconciliation is not atomic or
ordered by Git coupling, a temporary version mismatch may fail the polls
closed, and the server and exporter must be verified separately after
rollout. The exporter source still supports protocol 1 and its in-code
default remains 1; the deployed selector is now 2, and the earlier
manual-flip sequencing note is retired as historical. The generated chart
updater will find this branch already carrying the 0.5.0 pin and reuse the
pre-existing draft pull request coupled here before charts pull request #123
merges and publishes producer 2.0.22 as chart 0.5.0, so the updater race is
prevented rather than trusted. Chart publication of 0.5.0 is not yet done —
the live cluster still runs 0.4.8 — and no other future gate is claimed
beyond the normal repository CI. The owner explicitly waived backup/restore
for this rollout on 2026-10-02: no verified recovery guarantee exists, and
the untested OAuth seed-rotation compatibility risk is accepted. After
rollout, re-run the exporter verification above and expect the absent
`retry` series.

## Verify the physical Hero Node Exporter target

`operators/kube-prometheus-stack/application.yaml` selects Hero through the
pinned chart's `prometheus.prometheusSpec.additionalScrapeConfigs` setting. The
upstream chart owns the generated scrape-configuration Secret; do not hand-edit
or dump it. No credentials, new image, chart upgrade, retention change, or
replacement of the existing in-cluster Node Exporter job is involved. Host
installation and firewall ownership remain in `hero-host-config`.

PR CI runs the existing YAML, repository-hygiene and manifest checks. It does
not validate the embedded scrape configuration, render Helm, run Prometheus,
or verify network isolation. After a separately approved merge, main CI requests
root reconciliation automatically. Verify `gitops-operators` at the tested Git
SHA and the `kube-prometheus-stack` child at the unchanged chart version with
the intended Helm values. The chart revision alone does not prove job selection.

Check root and child sync/health, Prometheus and StatefulSet health, then verify
`up{job="hero-node-exporter",instance="hero"}` is 1 and
`node_exporter_build_info{job="hero-node-exporter",instance="hero"}` is present.
Confirm fresh, advancing sample timestamps across at least two scrape intervals
rather than relying on old samples or a one-off HTTP probe.

The listener is Hero's isolated bridge endpoint, not its LAN or WARP SSH endpoint.
Revalidate the translated source after node/network changes; its DHCP address
has no verified reservation. Do not broaden the host rule to a subnet merely
to recover a failed target. Denied-access acceptance is separate: an existing
authorized external test client must reach the actual listener with a confirmed
nonallowed source, correlated with arrival at Hero and a successful allowed
control. Timeout without arrival is not proof of firewall rejection. Same-node
pods may share the allowed source, and Hero-local delivery or requests to a
non-listening address do not prove external rejection. New probes, privileged
captures, or network changes require exact operation/target approval.

Rollback removes this job through a reviewed GitOps change and the same
reconciliation chain. It does not uninstall the host exporter or undo its
firewall rules; those are separately approved Hero IaC operations.
