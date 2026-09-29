# Staged read-only Discord MCP backend

**Status: staged and inactive.** This directory is NOT referenced by any
active Kustomization (`workloads/mcp-gateway/kustomization.yaml` does not
include it) and no Argo CD Application points at it. Merging this directory
changes no cluster state. Activation happens only through the sequence below;
the `Validate staged discord MCP contract and inactivity` step in
`.github/workflows/test.yml` fails if an active reference from any
kustomization, `workloads/apps` registration, or tracked Application manifest
appears before then.

## Trust boundary

Owner decision, 2026-09-29 (trusted-cluster exception): the owner explicitly
accepted the risk of a cluster-reachable, unauthenticated, read-only MCP
endpoint. There is no public endpoint: no `groupRef` (kept out of the
externally exposed VirtualMCPServer aggregation), no TunnelBinding subject,
and no remote-proxy entry. The backend is reachable only inside the cluster
at the ToolHive-generated proxy Service. Brief rationale: the cluster is
owner-operated single-tenant, the tool surface is read-only (three tools),
and the accepted blast radius is read access to channels the bot can already
see. Adding authentication (for example OIDC) later is a legitimate separate
change, not a violation of this design.

## Read-only scope

`discord-toolconfig.yaml` pins the tool surface to exactly:

- `get_server_info`
- `find_channel`
- `read_messages`

There are no posting, administration, emoji, webhook, or DM tools. A message
the bot cannot read fails or returns empty content; a healthy backend is not
proof of functional reads. Backend read verification, negative tool scope,
denied-channel behavior, and state health are separate verification gates run
at activation, not by this staging change.

## Credentials

`discord-mcpserver.yaml` references `Secret/mcp-discord` key `token` as
`DISCORD_TOKEN`. The Secret is **not supplied by this staged overlay** —
there is no Secret manifest and no KSOPS generator registration here — and
must be supplied and verified before activation. Plaintext tokens are
prohibited in the repository; the approved delivery is a SOPS-encrypted
manifest in GitOps using the repository's existing public age recipient,
added by the owner through the normal reviewed process. This bot token is NOT
shared with DisCode's token; each integration uses its own bot.

## Activation sequence

1. Create a dedicated Discord bot for this integration, restricted by Discord
   role/channel permissions to the intended read scope (proposed: `#general`
   `1540492160988610643`). Do not reuse or share the DisCode bot.
2. The owner adds the SOPS-encrypted `mcp-discord` Secret (`token` key) and
   its KSOPS wiring in a reviewed PR; plaintext never enters the repository.
3. A separately approved PR registers the overlay in the active
   Kustomization/Application and updates this CI inactivity gate in the same
   change.
4. Verify the backend before any client rollout: the pod renders and starts,
   the tool surface is exactly the three read-only tools, denied-channel
   behavior is correct, and the upstream jar under `/app` is readable by
   uid 1000 with the read-only root filesystem.
5. Only after backend verification, wire clients (for example an OpenCode
   chart PR with a release pin).

This staging change preserves the original setup: no custom image, no Secret
material, no client wiring.

## Discord-side configuration

- `DISCORD_GUILD_ID=1540492160103620668` is only the upstream default guild;
  it is not a guild or channel allowlist, and the application default does
  not enforce channel restrictions.
- The proposed read scope is the `#general` channel (`1540492160988610643`),
  enforced only through Discord permission configuration on the bot role, not
  by this workload.
- The bot uses the portal-requested `GUILD_MEMBERS` intent and the Message
  Content entitlement for general REST history reads; reads are REST-based,
  so no gateway missing-intent claim is made.

## Image provenance

Image: `docker.io/saseq/discord-mcp@sha256:92a5849c282750cf92f96865ca3f0ccad9560f505e6c71e764bae206a9b47a96`
([Docker Hub](https://hub.docker.com/r/saseq/discord-mcp)). Upstream source:
[SaseQ/discord-mcp](https://github.com/SaseQ/discord-mcp) at commit
[`0b35793bf9f86cc65849ae42ebae940718fbf812`](https://github.com/SaseQ/discord-mcp/tree/0b35793bf9f86cc65849ae42ebae940718fbf812).
The published digest corroborates the correspondence between this image and
that source commit, but the image carries no build attestation: the
relationship is corroborated, not proven.

## Runtime shape

- stdio transport behind the ToolHive `streamable-http` proxy; no
  `SPRING_PROFILES_ACTIVE=http` and no `mcpPort`, so the upstream raw HTTP
  listener stays disabled.
- A single backend/proxy pair is required. The MCPServer `replicas` field was
  not verified against the pinned ToolHive CRD (0.46.0), so it is omitted
  here; verify the deployed configuration is single-replica before
  activation.
- Stateless: no PVC and no resources block (single-node policy). Logs go to
  stderr only via `/config/logback.xml`, a generated content-hashed ConfigMap
  mounted read-only; `name-reference.yaml` teaches Kustomize the RawExtension
  reference so a log policy change rolls the pod. `/tmp` is a disk emptyDir
  bounded at 256Mi (provisional, not runtime-measured).
- Non-root uid/gid 1000 with fsGroup 1000, RuntimeDefault seccomp, all
  capabilities dropped, no privilege escalation, read-only root filesystem;
  the backend pod opts out of the service-account token.
- Multi-client: all MCP clients share one bot identity; there is no per-user
  identity mapping.
