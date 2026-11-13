# Agent

The agent (`biochef-agent`) runs workflows next to the data, inside each node ([draw.io](diagrams/04-agent-deploy.drawio)). How it is installed is in [deploy.md](deploy.md#nodes).

- It runs on the machine where its steps run, never as root, and never submits steps to a scheduler such as SLURM.
- It opens every connection. It polls the coordinator at `sites.coordinator.biochef.app` over mTLS, and nothing connects into the node.
- Only agent states (alive, running, waiting, error) and encrypted partials leave the node. Data, logs and step output never do.

## How a task goes

1. The agent polls the coordinator and receives an encrypted task, with a lease.
2. It checks the task (below). The first check that fails refuses it, and the signed refusal goes back to the coordinator so the member sees why.
3. It gets each tool by digest, through the signed catalogue, and verifies its signatures offline.
4. It runs each workflow step in a container of its own (see [Runners](#runners)).
5. The outputs pass the study's output rules, then the TRE's airlock or the data manager (see [Outputs](#outputs)).
6. It encrypts the released outputs for the study's result recipients, signs them with the node's approval key, and sends the partial to the coordinator.

## Checks on every task

A task runs only if all of these hold, checked in this order:

1. It is signed by a member of the study version it names, with that member's registered signing key.
2. It is for this node.
3. That study version is approved on this node.
4. The workflow is approved for that study version.
5. It has not expired.
6. It is not a replay. The same task delivered again joins the first run, or gets the same refusal.
7. It decrypts with the node's encryption key.
8. The workflow rebuilt from the decrypted tools and values has the digest the member signed.
9. The member's visas allow every dataset it names, and belong to the member ([identity.md](identity.md#visas)).
10. Every dataset it names is bound to data on this node.

## The agent page

The page where the data manager works. It is a separate process on its own port, reachable only inside the TRE. The TRE's own login decides who can open it; the agent has no user accounts.

| The data manager | Detail |
|---|---|
| Approves, refuses or withdraws study versions | Only after every member's ID token checks out ([identity.md](identity.md#tying-a-members-keys-to-them)), and once the DAC has decided. Only one version of a study is approved at a time |
| Approves, refuses or withdraws workflow versions | Per study version. Some are approved automatically ([architecture.md](architecture.md#a-federated-run)) |
| Follows runs | Steps, logs and outputs stay on the page, inside the TRE |
| Releases or withholds outputs | When the node reviews outputs itself rather than through an airlock |
| Binds datasets | Maps a study's dataset IDs to data on this node, when the node allows it from the page |

Every decision is signed with the node's approval key and written to an audit log.

## Node keys

| Key | Use |
|---|---|
| Approval key | Signs the node's decisions and its partials |
| Encryption key | Jobs are encrypted for it |

Both are made inside the node and never leave it. Their fingerprints are in the site list, so editors know which keys belong to which node.

## Datasets

Jobs carry dataset IDs, never data. Local policy maps each ID to a path on the node, and to the visa value the DAC issues for that dataset. A path outside the node's data root is refused.

## Tools

Every tool is resolved through the hub's signed catalogue, so the agent only runs a digest the catalogue lists. It pulls the tool by that digest from the global registry (avenue A) or from the node's own silo registry (avenue B), and verifies its signatures offline against the trusted root. The avenues are in [deploy.md](deploy.md#registry).

## Runners

The TRE picks the runner in the node's `.env`.

| Runner | Fits | Each step gets |
|---|---|---|
| Podman, rootless | Servers and cloud tenants | No network, its own workspace, a clean environment, a read-only root, limits on processes and memory |
| Podman, rootful | Hosts whose Podman only runs as root | The same. The agent calls one helper through a single sudo rule, and each step runs as a dedicated user that is neither root nor the agent |
| Apptainer, or SingularityCE | Hosts without Podman, including a cluster node | No network, its own workspace, a clean environment |
| TES | A TRE that already runs a GA4GH TES server, such as Funnel | Sent to that server as one task: the image by digest, nothing of the agent's in the task, and every returned file checked against a SHA-256 manifest. Network and filesystem isolation are up to the server |

Two rules hold for every runner:

- **Isolation is required.** The agent refuses to start with a runner whose steps could reach the network. An operator can turn the rule off, which the agent records at every start. The TES runner needs it off, since only the TES server can isolate its tasks.
- **The network is checked at start.** The agent runs one step and checks it cannot reach the network. If it can, the agent stops.

## Next to a TES server

The agent can also offer a TES API, a listener of its own, so the TRE's own workflow engines can send tasks to the node and have them run under the agent's runner. It shares no credentials, queue or port with the rest of the agent, needs a token on every request, and reads and writes only listed directories, never the node's datasets.

A TRE that already runs a TES server such as Funnel (ADELE does) can install the agent three ways:

| | Beside Funnel | Sending steps through Funnel | In place of Funnel |
|---|---|---|---|
| Federated steps run by | Podman, Apptainer or SingularityCE | Funnel, through the TES runner | The agent's runner |
| The TRE's own tasks run by | Funnel | Funnel | The agent's runner, through the TES API |
| Step isolation | Enforced by the agent | Up to Funnel; the isolation rule turned off | Enforced by the agent |
| The TRE provides | A container runtime for the agent's user | A directory both can read and write at the same path, an image with basic shell tools, a token | A container runtime; its engines pointed at the TES API |
| Cost | Two systems share the machine with no shared limits | The workspace is copied twice per step, and output arrives when the task ends | Only simple tasks: one command, no environment variables, volumes or resource requests |

## Outputs

Nothing leaves before egress:

1. The study's output rules, checked by the Policy Engine: for example aggregates only, a minimum count, allowed file types and a size limit. Files that fail never leave.
2. Then either the TRE's airlock, or the data manager's release on the agent page, depending on how the node is set up.
3. Only released files are encrypted for the result recipients and signed. A withheld result is sent as a signed notice with no files, so the member knows.

Outputs are kept on the node until they are sent or withheld.
