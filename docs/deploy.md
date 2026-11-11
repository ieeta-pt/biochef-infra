# Deploy

This doc covers where each piece runs, what is public, and every connection between them. The pieces are described in [architecture.md](architecture.md), the local setup in [local-build.md](local-build.md), and logins in [identity.md](identity.md).

## Three targets

None of them take variables. The host's `.env` picks the environment and the variant.

| Target | Where | Installs |
|---|---|---|
| `make up` (= `make local`) | A developer laptop | Everything, with simulated silos ([local-build.md](local-build.md)) |
| `make deploy` | The central server | The central unit; `.env` says staging or production. **Never the agent** |
| `make agent-deploy` | A TRE host | The node bundle; `.env` picks the step runner and the registry avenue |

## Central unit

`make deploy` installs the coordinator, the global registry and Keycloak on one server with Docker Compose, behind Caddy ([draw.io](diagrams/03-central-deploy.drawio)). The catalogue, Keycloak and the coordinator each have their own Postgres database. The coordinator keeps all its state in its database, so it can run as several instances. The unit holds no agent and no build tooling: agents run in nodes, and tools are built in CI.

| Name | Serves | Public? |
|---|---|---|
| `home.biochef.app` | Landing page and tutorials | Yes |
| `coordinator.biochef.app` | Coordinator API for people | Yes, with their Keycloak token |
| `sites.coordinator.biochef.app` | Coordinator API for agents | Yes, with mTLS. Its server certificate comes from the federation PKI |
| `registry.biochef.app` | Global registry | Read: yes. Write: CI only |
| `auth.biochef.app` | Keycloak | Login: yes. `/admin`: no |
| none | Databases, Keycloak admin, monitoring, Docker socket | No route at all |

## Private by default

| # | Layer | How |
|---|---|---|
| 1 | One door | Only Caddy publishes 80/443; every other service sits on internal Docker networks. Docker-published ports bypass `ufw`, so publish nothing else. |
| 2 | Host firewall | Allow 80/443. Allow 22 only from IEETA or the VPN. |
| 3 | Admin surfaces | No public route. Use an SSH tunnel; or a VPN plus a Caddy `remote_ip` allowlist; or `forward_auth` to the IdP. |
| 4 | TLS | Caddy obtains and renews certificates, except for `sites.coordinator.biochef.app`, whose certificate comes from the federation PKI. Use the DNS-01 challenge if the server isn't reachable from the internet. |
| 5 | Headers | HSTS, CSP (must allow `blob:` and `'wasm-unsafe-eval'` for WebAssembly), `nosniff`, Referrer-Policy, frame-ancestors. |
| 6 | Limits | Request size, timeouts, login rate limits. |
| 7 | Secrets | A `.env` file outside git with mode 600, or Docker secrets. Signing keys stay in CI, never on the server. |
| 8 | Hygiene | Images pinned by digest, nightly backups, health checks. |

## Connections

Who opens each connection, and how it is authenticated ([draw.io](diagrams/05-security-connections.drawio)). Nothing opens a connection into a TRE.

| # | From → to | Protocol | Authentication | Carries |
|---|---|---|---|---|
| 1 | Browser → the editor | HTTPS | None | The editor bundle |
| 2 | Editor → Keycloak | OIDC code + PKCE | Public client | Login |
| 3 | Keycloak → LS Login | OIDC | Confidential client | Brokered login |
| 4 | Editor → `coordinator.` | HTTPS | Keycloak access token | Member keys, studies, jobs (signed, encrypted per node), partials |
| 5 | Coordinator → Keycloak | HTTPS, internal | None | JWKS, discovery |
| 6 | Editor → registry | OCI over HTTPS | Anonymous | Catalogue, site list, descriptors, tools |
| 7 | Agent → `sites.coordinator.` | HTTPS + mTLS | Client certificate from the federation PKI; the agent checks the server certificate against the federation CA, not a pinned certificate | Polls, agent states, tasks, encrypted partials |
| 8 | Agent → registry (avenue A) | OCI over HTTPS | Anonymous; signatures checked offline | Tools by digest |
| 9 | Agent → CEGA/FEGA (remote mode) | htsget | EGA token carried in the job | Data slices |
| 10 | CI → registry | OCI push | Robot account (CI secret) | Signed tools, catalogue |
| 11 | CI → Sigstore | HTTPS | GitHub OIDC (keyless) | Signatures, transparency log |
| 12 | Operator → server | SSH | Key, from IEETA or the VPN only | Admin, tunnels to the databases |
| 13 | Editor → LS Login | OIDC | Public client, `openid` scope only | The ID token that ties the member's keys to them ([identity.md](identity.md#tying-a-members-keys-to-them)) |

Never routed: the databases, Keycloak `/admin`, the Docker socket, monitoring, and the agent API and agent page in any TRE.

## Registry

One global registry, read anonymously, ships with `make deploy`. It holds tools, the signed catalogue, the site list, node descriptors, dataset metadata and synthetic datasets. The editor's catalogue shows the silos and their datasets, and offers only the tools the selected TREs can run ([draw.io](diagrams/07-registry-catalogue.drawio)).

Each TRE picks how tools reach its agent:

| Avenue | Tools reach the agent through | TRE egress | Fits |
|---|---|---|---|
| A: global registry | OCI pull by digest, through the egress allowlist | Coordinator + registry | TREs with controlled egress |
| B: silo registry | A registry inside the TRE, synced from the global one or loaded from an encrypted tar | Coordinator only (plus the sync, if any) | TREs without network, or that want fewer outside calls |

Either way the agent resolves each tool through the signed catalogue and verifies every signature offline against the trusted root.

## Nodes

`make agent-deploy` installs the node bundle ([draw.io](diagrams/04-agent-deploy.drawio)):
- the agent, with cosign, running as a user that is not root;
- the step runner the TRE picks, below;
- local policy, which maps each dataset ID to a path;
- the node's approval and encryption keys, made inside the node;
- the Sigstore trusted root, the catalogue key and a local base image;
- strict signing;
- optionally, a silo registry (avenue B).

Diagrams 02 and 04 build the agent from its Dockerfile, which comes with [biochef-agent#73](https://github.com/ieeta-pt/biochef-agent/pull/73), still open.

The agent launches every step in a container through the runner the TRE picks in its `.env`:

| Runner | Fits | Each step |
|---|---|---|
| Podman, rootless | Servers and cloud tenants | No network, its own workspace, a clean environment, a read-only root and limits on processes and memory |
| Podman, rootful | Hosts whose Podman only runs as root | The same; the agent calls one helper through a single sudo rule, and each step runs as a dedicated user that is neither root nor the agent |
| Apptainer, or SingularityCE | SLURM clusters, and hosts without Podman | No network, its own workspace, a clean environment |

The agent refuses to start with a runner whose steps could reach the network, unless the operator turns that rule off explicitly, which is audited at every start. At start it runs one step and checks that it has no network. What each runner guarantees is in section 14.1 of the [agent spec](https://github.com/ieeta-pt/biochef-agent/pull/106); the Podman and SingularityCE runners and these checks come with [biochef-agent#120 to #123](https://github.com/ieeta-pt/biochef-agent/pull/120), still open.

Outputs leave only after egress: the study's output rules first, then the TRE's airlock or the data manager's release in the agent page, whichever the node is set up for. The agent page is a separate process on its own port, reachable only inside the TRE.

Egress is limited to the coordinator (`sites.coordinator.`), the registry (avenue A) and, in remote mode, the data service. There is no Sigstore, Docker Hub or refget access at run time.
