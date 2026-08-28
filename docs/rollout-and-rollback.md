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

4. Verify the controller rollout and resulting pods:

   ```bash
   kubectl -n <namespace> rollout status deployment/<deployment> --timeout=5m
   kubectl -n <namespace> get pods
   ```

   Use the applicable rollout command for a StatefulSet, DaemonSet, or other
   controller instead of assuming every chart creates a Deployment.

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
