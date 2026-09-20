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

Kustomize has no `allowEmpty` field; an explicit empty `resources` list is the
supported way to build an empty target on the deployed kustomize v5.8.1 (a
fully empty kustomization file would fail kustomize's empty check). The
Application carries no `resources-finalizer`, so removing its registration
before the prune is observed would orphan the live resources instead of
deleting them.

## Phase 2 (after the prune is observed)

Remove `opencode-sms-bridge-app.yaml` from `workloads/apps/kustomization.yaml`,
delete the Application manifest and this directory, then finish the external
cleanup.

## Deferred follow-ups

- `tfroot-twilio` provider teardown (owner-gated); afterwards remove the
  already-cached `/repos/tfroot-twilio` root from the `mcp-repo-cache` PVC.
- Depublication of the `opencode-sms-bridge` OCI chart (`0.1.2`) in the
  `makeitworkcloud/charts` registry.
- The `twilio-docs` Cloudflare Access application in `tfroot-cloudflare`, in
  the documented reverse delivery order (see `workloads/mcp-gateway/README.md`).

The shared `opencode` namespace, its Secret, and its PVC are not managed by
this Application and are unaffected.
