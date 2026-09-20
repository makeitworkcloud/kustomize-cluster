# OpenCode SMS bridge (retired)

The Twilio-backed OpenCode SMS bridge was retired by owner decision on
2026-09-20. There are no active integrations, no archive is retained, and the
owner has authorized the resulting SMS outage and data loss.

## Phase 1 (this state)

The `opencode-sms-bridge` Application stays registered so its existing
automated sync performs the destructive work safely:

- the Application's Helm chart source
  (`ghcr.io/makeitworkcloud/charts/opencode-sms-bridge` 0.1.2) is removed, so
  the chart no longer renders;
- this overlay intentionally builds zero manifests (`resources: []`) so the
  Application prunes every previously managed resource: the Deployment, its
  ConfigMap, the Service, the PVC `opencode-sms-bridge-state`, the three
  bridge Secrets, and the `opencode-sms-bridge` TunnelBinding.

Two empty-state safeguards must both be released for the automated prune to
run. Kustomize has no `allowEmpty` field, so the explicit empty `resources`
list is what builds an empty target on the deployed kustomize v5.8.1 (a fully
empty kustomization file would fail kustomize's empty check). Argo CD by
default refuses to auto-sync an application down to zero live resources, so
`spec.syncPolicy.automated.allowEmpty: true` is set temporarily on this
Application only; it is a targeted decommission switch, not a pattern to
copy, and it is removed together with the Application in phase 2. The
Application carries no `resources-finalizer`, so removing its registration
before the prune is observed would orphan the live resources instead of
deleting them.

## Phase 2 (after the prune is observed)

Remove `opencode-sms-bridge-app.yaml` from `workloads/apps/kustomization.yaml`,
delete the Application manifest (which also drops the temporary
`allowEmpty: true`) and this directory, then finish the external cleanup.

## Deferred follow-ups

- `tfroot-twilio` provider teardown (owner-gated); afterwards remove the
  already-cached `/repos/tfroot-twilio` root from the `mcp-repo-cache` PVC.
- Depublication of the `opencode-sms-bridge` OCI chart (`0.1.2`) in the
  `makeitworkcloud/charts` registry.
- The `twilio-docs` Cloudflare Access application in `tfroot-cloudflare`, in
  the documented reverse delivery order (see `workloads/mcp-gateway/README.md`).

The shared `opencode` namespace, its Secret, and its PVC are not managed by
this Application and are unaffected.
