# Staged read-only Discord MCP backend

**Status: staged and inactive.** This directory is NOT referenced by any
active Kustomization (`workloads/mcp-gateway/kustomization.yaml` does not
include it) and no Argo CD Application points at it. Merging this directory
changes no cluster state. Activation happens only through a separate,
owner-approved PR that wires the overlay in; the `Validate staged discord MCP
contract and inactivity` step in `.github/workflows/test.yml` fails if any
active reference appears before then.

## Trust boundary

Owner decision, 2026-09-29 (trusted-cluster exception): the owner explicitly
accepted the risk of a cluster-reachable, unauthenticated, read-only MCP
endpoint. There is no public endpoint: no `groupRef` (kept out of the
externally exposed VirtualMCPServer aggregation), no TunnelBinding subject,
no remote-proxy entry, no OIDC. The backend is reachable only inside the
cluster at the ToolHive-generated proxy Service. Brief rationale: the cluster
is owner-operated single-tenant, the tool surface is read-only (three
tools), and the accepted blast radius is read access to channels the bot can
already see.

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
`DISCORD_TOKEN`. That Secret does not exist yet and is intentionally not
created here: there is no Secret manifest and no KSOPS generator registration
in this overlay, so there is no dangling reference to a missing encrypted
file. The owner creates the Secret later using the repository's existing
public-recipient SOPS process; the token is never pasted into the repository,
chat, or CI. This bot token is NOT shared with DisCode's token; each
integration uses its own bot.

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

`docker.io/saseq/discord-mcp@sha256:92a5849c282750cf92f96865ca3f0ccad9560f505e6c71e764bae206a9b47a96`,
built from upstream source commit
`0b35793bf9f86cc65849ae42ebae940718fbf812`. Digest publication is
corroborated, not signature-verified.

## Runtime shape

- stdio transport behind the ToolHive `streamable-http` proxy; no
  `SPRING_PROFILES_ACTIVE=http` and no `mcpPort`, so the upstream raw HTTP
  listener stays disabled.
- One backend pod by operator default: the MCPServer `replicas` field was not
  verified against the pinned ToolHive CRD (0.46.0), so the field is omitted
  and the default is relied on; verifying single-replica behavior is a
  documented activation gate.
- Stateless: no PVC and no resources block (single-node policy). Logs go to
  stderr only via `/config/logback.xml`, a generated content-hashed ConfigMap
  mounted read-only; `name-reference.yaml` teaches Kustomize the RawExtension
  reference so a log policy change rolls the pod. `/tmp` is a disk emptyDir
  bounded at 256Mi (provisional, not runtime-measured).
- Non-root uid/gid 1000 with fsGroup 1000, RuntimeDefault seccomp, all
  capabilities dropped, no privilege escalation, read-only root filesystem;
  the backend pod opts out of the service-account token. Whether the upstream
  jar under `/app` is readable by uid 1000 is a CI/activation verification
  item, not something this staging change pretends to have run.
- Multi-client: all MCP clients share one bot identity; there is no per-user
  identity mapping.
