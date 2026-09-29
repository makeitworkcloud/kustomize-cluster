# Staged read-only Discord MCP backend

**Status: staged and inactive.** No active Kustomization or Argo CD
Application references this directory, so merging does not register or deploy
Discord; the unchanged main sync workflow may still reconcile the existing
roots as usual. There is no bespoke inactivity test for this overlay: the
existing reference-validation, YAML hygiene, gitleaks, and kube-linter checks
apply, and the reviewer must verify that active root/Application references
are unchanged by this change. Activation is a separate, owner-approved PR.

## Trust boundary

Owner decision 2026-09-29 (trusted-cluster exception): the owner accepted a
cluster-reachable, unauthenticated, read-only MCP endpoint. No public
endpoint: no `groupRef`, TunnelBinding subject, or remote-proxy entry; the
backend is reachable only in-cluster at the ToolHive proxy Service.
Rationale: single-tenant owner-operated cluster, three read-only tools, blast
radius limited to channels the bot can already see. Adding authentication
(e.g. OIDC) later is a legitimate separate change.

## Scope and credentials

Tool surface is exactly `get_server_info`, `find_channel`, `read_messages` —
no posting, admin, emoji, webhook, or DM tools. `DISCORD_GUILD_ID` is only
the upstream default guild, not an allowlist; the proposed read scope
(`#general` `1540492160988610643`) is enforced by Discord-side bot
permissions. The bot uses the portal-requested GUILD_MEMBERS intent and
Message Content entitlement; reads are REST-based.

The MCPServer references `Secret/mcp-discord` key `token` as `DISCORD_TOKEN`.
The Secret is not supplied by this overlay and must be supplied and verified
before activation. Plaintext tokens are prohibited; the approved delivery is
a SOPS-encrypted GitOps manifest (existing public age recipient). This is a
dedicated bot — no sharing with the DisCode bot or token.

## Image provenance

`docker.io/saseq/discord-mcp@sha256:92a5849c282750cf92f96865ca3f0ccad9560f505e6c71e764bae206a9b47a96`
([Docker Hub](https://hub.docker.com/r/saseq/discord-mcp)); upstream
[SaseQ/discord-mcp](https://github.com/SaseQ/discord-mcp) at commit
[`0b35793bf9f86cc65849ae42ebae940718fbf812`](https://github.com/SaseQ/discord-mcp/tree/0b35793bf9f86cc65849ae42ebae940718fbf812).
The published digest corroborates the image-source correspondence; no build
attestation was verified (unknown, not absent).

## Runtime

stdio via the ToolHive `streamable-http` proxy; no `SPRING_PROFILES_ACTIVE=http`
or `mcpPort`, so the upstream raw HTTP listener stays disabled. One
backend/proxy pair is required; `replicas`/`backendReplicas` are optional
upstream CRD fields (toolhive `c6c425a9`) and are omitted here — the replica
defaults must be verified in the generated resources before functional test.
Stateless: no PVC, no resources block; logs stderr only via the generated
content-hashed ConfigMap at `/config` (RawExtension nameReference rolls the
pod on change); `/tmp` is a disk emptyDir at 256Mi (provisional). Non-root
uid/gid/fsGroup 1000, RuntimeDefault seccomp, drop ALL caps, no privilege
escalation, read-only rootfs, no service-account token. One shared bot
identity for all clients. Unreadable messages fail or return empty; backend
health is not proof of functional reads.

## Validation

Current CI (kustomization reference validation, YAML hygiene, gitleaks,
kube-linter, pre-commit) is pending on the PR; no kustomize render, Discord
connection, or runtime check has run for this overlay. Activation gates:
render the overlay and verify the content-hashed ConfigMap name rewrite,
uid 1000 start with the read-only root filesystem and `/app` jar
readability, the exact 3-tool surface (negative tool scope), and
denied-channel behavior and state health.

## Activation sequence

1. Create a dedicated restricted Discord bot (proposed `#general` scope); do
   not reuse the DisCode bot.
2. Owner PR: SOPS-encrypted `mcp-discord` Secret (`token` key) + KSOPS
   wiring, reviewed; plaintext never enters the repository.
3. Separately approved PR: register the overlay in the active
   Kustomization/Application, with explicit review of the active reference
   registration.
4. Verify the backend per the validation gates above.
5. Only then wire clients (e.g. an OpenCode chart PR with a release pin).

No custom image, no Secret material, no client wiring in this staging change.
