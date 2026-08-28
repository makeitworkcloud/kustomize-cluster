# Adding a workload

Use this procedure to add a chart-backed workload without bypassing the
App-of-Apps boundary. The chart may be new and unpublished while the cluster
change is prepared, but the Application must pin a published, immutable chart
version before it can merge.

## 1. Define ownership

Decide which repository owns each resource before writing manifests:

- The chart owns the reusable workload release: Deployments or StatefulSets,
  non-secret configuration, probes, and chart values.
- This repository owns cluster-specific integration under `workloads/<name>`:
  encrypted Secrets, namespace metadata, storage, stable Services, and
  operator-specific resources such as `TunnelBinding`.
- Give every Kubernetes object one owner. A second Argo CD source is rendered
  alongside the chart; it does not patch chart output automatically.

Keep chart authoring and publication details in the charts repository. Follow
its [adding a chart
guide](https://github.com/makeitworkcloud/charts/blob/main/docs/adding-a-chart.md)
to publish an immutable OCI version. GitOps updater automation is a separate,
explicit opt-in; follow the charts repository's [automation
guide](https://github.com/makeitworkcloud/charts/blob/main/docs/gitops-update-automation.md)
when the workload needs it. Do not reproduce updater credential or GitHub App
rotation procedures here.

## 2. Create the cluster overlay

Create `workloads/<name>/kustomization.yaml` only when the chart needs
cluster-owned resources. Register each resource in that Kustomization. Typical
files include:

```text
workloads/<name>/
  kustomization.yaml
  namespace.yaml
  persistent-volume-claim.yaml
  service.yaml
  tunnel-binding.yaml
  ksops-<name>-secrets.yaml
  <name>-secret.yaml
```

Include only the files the workload needs:

- Use an explicit `Namespace` when labels, annotations, or other namespace
  metadata must be Git-owned. Otherwise `CreateNamespace=true` can create the
  destination namespace.
- Keep a Service here only when the chart does not own the required stable
  Service. Its selector and ports must match the chart workload.
- Keep a PVC here when storage is cluster-owned. Configure the chart to use the
  existing claim, and review storage-class, reclaim, upgrade, rollback, and
  prune behavior before relying on it for persistent data.
- Add a `TunnelBinding` only for externally reachable workloads. It must be in
  the Service namespace, and `subjects[].name` must exactly match that Service.
  The binding owns workload tunnel DNS; do not duplicate its CNAME in
  `tfroot-cloudflare`.

For secrets, create SOPS-encrypted Secret manifests using the approved
secret-editing process and list them in a KSOPS generator:

```yaml
generators:
  - ksops-<name>-secrets.yaml
```

Follow `.sops.yaml` selective-encryption rules. Never commit plaintext, print
decrypted values into review output, or decrypt secrets merely to run routine
validation.

## 3. Create the child Application

Create `workloads/apps/<name>-app.yaml`. Use placeholders while preparing a
chart that has not been published, then replace `<published-chart-version>`
with the immutable OCI chart version before merge.

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <name>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  sources:
    - chart: <chart-name>
      repoURL: <oci-registry>/<organization>/charts
      targetRevision: <published-chart-version>
    - repoURL: https://github.com/<organization>/<cluster-repository>.git
      path: workloads/<name>
      targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - SkipDryRunOnMissingResource=true
    retry:
      limit: 5
      backoff:
        duration: 30s
        maxDuration: 5m
        factor: 2
```

Remove the second source when no cluster overlay is needed. Keep `project`,
destination server, sync policy, and retry behavior aligned with neighboring
Applications unless an approved workload requirement says otherwise. Set the
destination namespace explicitly for namespaced charts. Use
`SkipDryRunOnMissingResource=true` only when custom resources require it; it is
not a substitute for making required CRDs available.

The annotation orders the child Application resource within the
`gitops-workloads` sync. It does not order resources inside the child or across
independent root Applications. Choose a different wave only for a demonstrated
dependency among resources managed by `gitops-workloads`.

## 4. Handle CRDs and ordering

Operator-owned CRDs and controllers belong under `operators/`, not in the
workload overlay. If the workload creates a custom resource whose CRD might not
exist when child Applications are created, add the CRD name to
`workloads/apps/wait-for-crds.yaml`. The `PreSync` job must observe the CRD as
`Established` before the root creates workload resources.

Do not use sync waves to imply ordering across independent Argo CD
Applications. Add a wait condition when a real cross-Application dependency
exists. A chart that owns its own CRDs should keep their installation and
resource ordering within that chart unless the CRDs are promoted to an
operator-owned boundary.

## 5. Register the Application

Add only the child Application manifest to
`workloads/apps/kustomization.yaml`:

```yaml
resources:
  - <name>-app.yaml
```

Do not add `workloads/<name>` to that file. The child Application's repository
source owns the overlay.

No bootstrap change is needed for a normal workload addition because
`bootstrap/workloads-app.yaml` already reconciles `workloads/apps`. Change
bootstrap only when the App-of-Apps source, ownership boundary, destination, or
bootstrap-applied resources themselves must change. A new child Application,
namespace, chart version, or workload overlay alone is not a bootstrap change.

## 6. Validate and review

Run the repository checks before opening the pull request:

```bash
pre-commit run --all-files
git diff --check
```

The `test` workflow runs the hooks configured in `.pre-commit-config.yaml`,
including YAML syntax and repository hygiene checks, secret scanning, and
kube-linter. It does not prove that Argo CD can pull an unpublished chart or
that the workload will become Healthy. Confirm in review that:

- Every `kustomization.yaml` reference exists and only the child Application
  was added to `workloads/apps/kustomization.yaml`.
- The chart version is published and pinned, not a mutable tag or range.
- Chart and cluster resources do not claim the same Kubernetes object.
- CRD gates, namespace, Service, PVC, `TunnelBinding`, and encrypted Secret
  ownership are explicit.
- The generated GitOps pull request changes desired state only. It does not
  deploy or sync Argo CD directly.

After merge, follow [Rollout and rollback](rollout-and-rollback.md).
