# Deploy

This doc covers where each piece runs, what is public, and every connection between them. The pieces are described in [architecture.md](architecture.md), the local setup in [local.md](local.md), and logins in [identity.md](identity.md).

## Three targets

None of them take variables. The host's `.env` picks the environment and the variant.

| Target | Where | Installs |
|---|---|---|
| `make up` (= `make local`) | A developer laptop | Everything, with simulated silos ([local.md](local.md)) |
| `make deploy` | The central server | The central unit; `.env` says staging or production. **Never the agent** |
| `make agent-deploy` | A TRE host | The node bundle; `.env` picks the runtime and the registry avenue |

## Central unit

`make deploy` installs the coordinator, its Postgres, the global registry and Keycloak on one server with Docker Compose, behind Caddy ([draw.io](diagrams/03-central-deploy.drawio)). It holds no agent and no build tooling: agents run in nodes, and tools are built in CI.

| Name | Serves | Public? |
|---|---|---|
| `biochef.app` | Editor | Yes |
| `home.biochef.app` | Landing page and tutorials | Yes |
| `coordinator.biochef.app` | Coordinator API | Yes. People use their Keycloak token; nodes use mTLS |
| `registry.biochef.app` | Global registry | Read: yes. Write: CI only |
| `auth.biochef.app` | Keycloak | Login: yes. `/admin`: no |
| none | Databases, Keycloak admin, monitoring, Docker socket | No route at all |

## Private by default

| # | Layer | How |
|---|---|---|
| 1 | One door | Only Caddy publishes 80/443; every other service sits on internal Docker networks. Docker-published ports bypass `ufw`, so publish nothing else. |
| 2 | Host firewall | Allow 80/443. Allow 22 only from IEETA or the VPN. |
| 3 | Admin surfaces | No public route. Use an SSH tunnel; or a VPN plus a Caddy `remote_ip` allowlist; or `forward_auth` to the IdP. |
| 4 | TLS | Caddy obtains and renews certificates. Use the DNS-01 challenge if the server isn't reachable from the internet. |
| 5 | Headers | HSTS, CSP (must allow `blob:` and `'wasm-unsafe-eval'` for WebAssembly), `nosniff`, Referrer-Policy, frame-ancestors. |
| 6 | Limits | Request size, timeouts, login rate limits. |
| 7 | Secrets | A `.env` file outside git with mode 600, or Docker secrets. Signing keys stay in CI, never on the server. |
| 8 | Hygiene | Images pinned by digest, nightly backups, health checks. |

## Connections

Who opens each connection, and how it is authenticated ([draw.io](diagrams/05-security-connections.drawio)). Nothing opens a connection into a TRE.

| # | From → to | Protocol | Authentication | Carries |
|---|---|---|---|---|
| 1 | Browser → biochef.app | HTTPS | None | The editor bundle |
| 2 | Editor → Keycloak | OIDC code + PKCE | Public client | Login |
| 3 | Keycloak → LS Login | OIDC | Confidential client; tokens stored | Brokered login, Passport |
| 4 | Editor → coordinator | HTTPS | Keycloak access token | Studies, jobs (signed, encrypted per node), partials |
| 5 | Coordinator → Keycloak | HTTPS, internal | None | JWKS, discovery |
| 6 | Editor → registry | OCI over HTTPS | Anonymous | Catalogue, site list, descriptors, tools |
| 7 | Agent → coordinator | HTTPS + mTLS | Client certificate from the federation PKI; coordinator certificate pinned | Polls, agent states, tasks, encrypted partials |
| 8 | Agent → registry (avenue A) | OCI over HTTPS | Anonymous; signatures checked offline | Tools by digest |
| 9 | Agent → CEGA/FEGA (remote mode) | htsget | EGA token carried in the job | Data slices |
| 10 | CI → registry | OCI push | Robot account (CI secret) | Signed tools, catalogue |
| 11 | CI → Sigstore | HTTPS | GitHub OIDC (keyless) | Signatures, transparency log |
| 12 | Operator → server | SSH | Key, from IEETA or the VPN only | Admin, tunnels to the databases |

Never routed: the databases, Keycloak `/admin`, the Docker socket, monitoring, and the agent API in any TRE.

## Registry

One global registry, read anonymously, ships with `make deploy`. It holds tools, the signed catalogue, the site list, node descriptors, dataset metadata and synthetic datasets. The editor's catalogue shows the silos and their datasets, and offers only the tools the selected TREs can run ([draw.io](diagrams/07-registry-catalogue.drawio)).

Each TRE picks how tools reach its agent:

| Avenue | Tools reach the agent through | TRE egress | Fits |
|---|---|---|---|
| A: global registry | OCI pull by digest, through the egress allowlist | Coordinator + registry | TREs with controlled egress |
| B: silo registry | A registry inside the TRE, synced from the global one or loaded from an encrypted tar | Coordinator only (plus the sync, if any) | TREs without network, or that want fewer outside calls |

Either way the agent verifies every signature offline against the trusted root.

## Nodes

`make agent-deploy` installs the node bundle ([draw.io](diagrams/04-agent-deploy.drawio)):
- the agent image, with Apptainer and cosign;
- local policy, which maps each dataset ID to a path;
- the Sigstore trusted root and a local base image;
- strict signing;
- optionally, a silo registry (avenue B).

| Variant | Agent runs as | Steps run on |
|---|---|---|
| Server or cloud tenant | Docker or Podman compose | Apptainer on the same host |
| Cluster, or a TRE without Docker | Apptainer instance plus systemd | SLURM |

Egress is limited to the coordinator, the registry (avenue A) and, in remote mode, the data service. There is no Sigstore, Docker Hub or refget access at run time.
