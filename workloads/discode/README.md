# DisCode — activation-gated workload preparation

[DisCode](https://github.com/upiscium/DisCode) (package name
`opencode-discord-bridge`) is a Discord control surface for OpenCode
sessions: bound Discord threads drive OpenCode sessions over the OpenCode
server API. This overlay prepares the workload **disconnected**: nothing
here reaches a live pod until the owner completes the activation gates.

## Status: inactive

- `deployment.yaml` ships with `spec.replicas: 0` — no pod is scheduled, so
  referenced Secrets are never resolved and the sentinel config below never
  runs.
- `../apps/discode-app.yaml` is staged but **not registered** in
  `workloads/apps/kustomization.yaml`.
- `discode-config.toml` carries `REPLACE_WITH_*` sentinel **identifiers**
  (not credentials) for the Discord IDs the owner has not supplied.
- No SOPS/KSOPS files are committed: the `discode-bot-auth` Secret does not
  exist yet and is only referenced.
- No Service, Ingress, or TunnelBinding: the workload is pod-internal.

## Source and image pins

- **Upstream source:** `upiscium/DisCode` at immutable commit
  `6b936f6c7c6cfbb5995a44cbff9361b9d72e4bc2`
  ("feat: add host-scoped tilde directory expansion (#69)", 2026-09-13).
  Verified via GitHub at that SHA: `package.json` (engines `node >=22`,
  build `tsc`, start `node dist/index.js`, committed `package-lock.json`),
  `src/config.ts` (strict TOML schema), `src/discord/commands.ts`
  (command surface), and the upstream README.
- **Archive checksum:** GitHub publishes no checksum for codeload tarballs,
  and computing one during preparation would have required upstream network
  contact outside this session's scope, so none is pinned. Integrity rests
  on the immutable commit SHA plus a fail-closed fetch (HTTP 200 only,
  redirects rejected, 60 s timeout, 64 MiB ceiling). The init script keeps
  an `EXPECTED_SHA256` enforcement hook the owner can pin at activation.
- **Base image:** `docker.io/library/node:22-slim@sha256:83f487e0a63425e5b4d146fb5e5be574bcbe1b7b843d3ebafdd95eaf7767a7e5`
  — the same immutable digest already vetted and deployed for
  `makeitwork-hero-ssh` and `makeitwork-codebase-memory` (owner
  third-party-images policy, 2026-09-09). Docker Hub (checked 2026-09-19)
  shows the `22-slim` tag has since moved to a newer index digest; the
  pinned digest remains immutable and in active use. Re-vetting a newer
  digest is owner policy, not done here.

## Bootstrap behavior (chosen option)

Owner approved upstream base image + upstream commands instead of a custom
image or chart. Every pod start:

1. Credentialless init container (uid 1000, no Secret mounts, read-only
   rootfs) fetches the commit archive and runs the unmodified upstream
   build: `npm ci --ignore-scripts --no-audit --no-fund`, `npm run build`,
   `npm prune --omit=dev --ignore-scripts`, then stages `dist/`, pruned
   `node_modules/`, and `package.json` into the `/app` emptyDir.
2. Runtime container (same image/digest) mounts `/app` read-only and runs
   `node dist/index.js` with `HOME=/tmp`.

Cold starts therefore need egress to `codeload.github.com` and
`registry.npmjs.org`; this cluster enforces no NetworkPolicy. This is the
same trust class as the owner-approved `codebase-memory` npm-exec startup
(PR #224 waiver discussion). Rebuilds are deterministic per commit.

## Configuration

- TOML mode via `OCB_CONFIG_FILE=/config/config.toml` (ConfigMap
  `discode-config`, generated with a content hash so edits roll the pod).
  Schema verified against `src/config.ts` at the pinned commit: unknown
  keys fail startup. Note `allowed_user_ids` is an **array** of ID strings.
- `[host.cluster]` targets `http://opencode.opencode.svc.cluster.local:4096`
  (the `opencode` Service owned by `workloads/opencode`), username
  `opencode` (DisCode's own default and upstream documentation form; the
  chart consumes only the password), `home_directory=/home/opencode` and
  `allowed_roots=["/home/opencode"]` — the OpenCode server's own filesystem
  view. No cluster filesystem path is mounted into this pod.
- `STATE_FILE=/state/state.json` on the `discode-state` PVC (128Mi, RWO).
  Single writer; the Deployment uses `Recreate` so two pods can never share
  the claim. State holds routing/presentation metadata only; upstream never
  persists credentials.

## Secrets (referenced only, never read)

| Env | Secret | Key | Status |
|-----|--------|-----|--------|
| `OPENCODE_HOST_CLUSTER_PASSWORD` | `opencode-server-auth` | `password` | exists; owned by `workloads/opencode`; same-namespace reference, key name verified from the `charts/opencode-server` Deployment consumer |
| `DISCORD_TOKEN` | `discode-bot-auth` | `DISCORD_TOKEN` | activation gate 2: to be SOPS-created by the owner |

## Security record (source-verified at the pinned commit)

This workload is for a trusted household, **not a restricted sandbox**:

- `/oc start`, `/oc sessions`, and `/oc bind` all **require** a `directory`
  option (absolute path or `~/...`); `~` expands only because
  `home_directory` is configured, and authorization is containment within
  `allowed_roots` after host-side canonicalization.
- `/oc bind` surfaces and binds the owner's **existing** OpenCode sessions
  in authorized directories.
- `/oc close` **deletes** the bound OpenCode session and archives the
  thread; `/oc unbind` detaches without deleting.
- `/oc agent` (and `/oc model`) can select **any** entry from the host's
  current OpenCode catalog, revalidated at execution time. Nothing in this
  overlay restricts the catalog.
- `/oc subagents`/`subagent` expose read-only child-session views.
- Discord-side authorization is exactly `allowed_user_ids` within the one
  configured guild; `allow_permission_always=false` keeps the persistent
  "Allow always" permission button hidden.
- These manifests constrain directories, roots, and identity — they do
  **not** constrain what the OpenCode sessions themselves (or agents
  selected via `/oc agent`) may do on the server host. Do not claim
  otherwise.

## Metrics caveat

`[metrics] enabled=true` binds `0.0.0.0:9464` so kubelet tcpSocket probes
have something to touch; there is no Service, so nothing in-cluster scrapes
it routinely. `opencode_discord_bridge_ready` reflects OpenCode host
HTTP+SSE health only — a passing probe does **not** prove the Discord
gateway connection or end-to-end readiness. Use `/oc health` for that.

## Activation checklist (owner)

1. Replace every `REPLACE_WITH_*` sentinel in `discode-config.toml`.
2. Create the SOPS-encrypted `discode-bot-auth` Secret (key `DISCORD_TOKEN`)
   per the approved secret-editing process and add a KSOPS generator here.
3. Optionally pin `EXPECTED_SHA256` in `deployment.yaml`.
4. Register `../apps/discode-app.yaml` in `workloads/apps/kustomization.yaml`.
5. Scale `spec.replicas` to 1.
6. Verify Argo health, then run `/oc health` from Discord.

Rollback: scale to 0 / unregister the Application; the PVC retains state.

## CI coverage and gaps

`test.yml` (on PR) covers YAML/hygiene hooks, gitleaks, kube-linter, and
kustomization reference resolution for this overlay. **Not covered** until
activation: an actual `npm ci`/build against the real upstream tree (first
exercised in-pod), DisCode-side TOML validation (schema was verified by
source reading, not execution), Secret existence, and Argo rendering of the
child Application (CI cannot run KSOPS decryption, and the Application is
unregistered). No new workflow was added — no clean established pattern
exists for in-cluster-style build validation, and workflow dispatch was out
of scope for this preparation.
