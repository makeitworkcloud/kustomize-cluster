# DisCode retirement

The owner verified phone access through the existing OpenCode web interface
and approved deletion of DisCode-only routing state. This directory is an
empty retirement overlay, not an active bridge or proof of completed removal.

## Phase 1: prune through the existing Application

Keep `workloads/apps/discode-app.yaml` registered and its source path intact.
The empty Kustomization produces no resources; `namespace: opencode` is a
transform setting, not a Namespace resource. The child retains automated
pruning and self-heal and enables `allowEmpty: true` to prune its old resources.
See [Argo CD 2.14 automated pruning](https://argo-cd.readthedocs.io/en/release-2.14/user-guide/auto_sync/#automatic-pruning-with-allow-empty-v18).

**Merge can trigger destructive automatic reconciliation immediately.** The
main test/sync workflow is not a pause or approval gate for Argo CD watching
main. Merge and live cleanup require explicit owner confirmation.

The intended removals, all in namespace `opencode`, are:

- Deployment `discode` and its dependent ReplicaSets/pods.
- Generated ConfigMap `discode-config-<hash>`.
- Secret `discode-bot-auth` (the encrypted source and generator are removed).
- PVC `discode-state`, including its Discord-to-session routing mappings.

The DisCode-specific SOPS creation rule and CI static/build artifact chain
are removed with their consumer. Other CI checks and secret rules remain.

Do not remove the Application first: its existing manifest has no resources
finalizer, so deleting it does not guarantee deletion of its workloads.
Do not use `/oc close` for cleanup; that command deletes OpenCode sessions.

## Preserve OpenCode

The DisCode Deployment referenced, but did not own, `opencode-server-auth`.
Keep that Secret, the namespace, all OpenCode sessions/storage, the server,
chart pin, native authentication, public route, and HSTS unchanged.

## Verification and Phase 2

After approved merge, verify the tested revision on `gitops-workloads` and
`discode`, successful child pruning, and an empty child resource tree.
Confirm separately that the listed objects and DisCode pods/PVC are absent;
never retrieve Secret contents to check existence. Verify OpenCode remains
healthy, native unauthenticated API rejection still works, and phone access
and existing-session continuity remain functional.

Only after those checks pass, prepare a separate reviewed change removing
the empty child Application, its registration, and this source directory.
If pruning fails, retain the Application and diagnose the failure first.

## Data loss, rollback, and Discord

PVC deletion can irreversibly delete its backing data. The PV reclaim policy
could not be read with the available permissions; neither forensic erasure
nor recovery is guaranteed. Git history restores manifests, not deleted
routing data. Closing an unmerged PR leaves live resources unchanged; do not
automatically restore the incompatible bridge after removal.

Removing the cluster Secret does not revoke the Discord token. The owner must
remove the bot from Discord and revoke its token through the approved Discord
administration surface. Do not send the token into chat or change OpenCode
authentication as part of this cleanup.
