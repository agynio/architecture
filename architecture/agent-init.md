# Agent Init Container

## Overview

The agent container runs the user's **workspace image** — their tools, language runtimes, project dependencies. Platform binaries are not baked into it. They are injected at Pod startup by **init containers** that write into a volume the main container shares.

Three images are injected, from two different places, because they change for different reasons:

| Injected | Source | Contains |
|---|---|---|
| **`agynd-cli-init`** | [Agents Orchestrator](agents-orchestrator.md) configuration — chart-pinned, not a catalog entry | `agynd`, the workload's runtime |
| **`agyn-cli-init`** | Agents Orchestrator configuration — chart-pinned, not a catalog entry | The [`agyn`](agyn-cli.md) CLI |
| **Agent runtime image** | The [environment](../product/environments/environments.md) | An agent CLI binary and its `config.json` |

The two platform images are not choices anyone makes. They ship with the platform's Helm chart and are injected into every workload, in the same class as the [Ziti sidecar](openziti.md#ziti-sidecar) — plumbing the platform provides. The agent runtime image is selected from the [image catalog](../product/images/images.md) through the environment, and is what decides which agent CLI an agent runs.

## Problem

The platform supports multiple agent CLI types (Codex, Claude Code, [`agn`](agn-cli.md), and others as they are added). Each requires a specific binary and SDK integration inside `agynd`. The main container image is the user's workspace — requiring users to rebuild it with `agynd` and an agent CLI baked in is not viable.

**Constraints:**

- The user's workspace image must remain untouched.
- The mechanism must work with any Linux base image regardless of distro, libc, or installed packages.
- `agynd` is always the main process in the agent container — there is no case without it.
- `agynd` speaks the platform's APIs on one side and each agent CLI's protocol on the other. Its platform side must never lag the platform, or a workload silently fails to implement a flow the platform expects — a failure that produces no error, just an agent that does nothing.
- An agent CLI releases on its own cadence and must be upgradable without a platform release.

## Design

```mermaid
graph TB
    subgraph Pod
        subgraph "Init containers (run first, in order)"
            AD[agynd-cli-init<br/>from orchestrator config<br/>agynd → /agyn/bin]
            AC[agyn-cli-init<br/>from orchestrator config<br/>agyn → /agyn/bin]
            AR[agent-runtime<br/>from environment<br/>agent CLI → /agyn/bin<br/>config.json → /agyn]
        end
        subgraph "Main container (user's workspace image)"
            AGYND[/agyn/bin/agynd<br/>reads /agyn/config.json<br/>spawns agent CLI]
        end
        subgraph Sidecars
            MCP[MCP servers]
        end
        VOL[emptyDir: agyn]
    end

    AD -->|writes to| VOL
    AC -->|writes to| VOL
    AR -->|writes to| VOL
    VOL -->|mounted in| AGYND
```

### Why the split

`agynd` shipping with the platform is what makes the platform side of its contract unbreakable. When `agynd` travelled inside a user-selected image, an old image on a new platform produced exactly the failure that is hardest to diagnose: wire-compatible, semantically stale, and silent. Deprecated fields still deserialize; a flow `agynd` does not know to perform simply does not happen. No amount of version metadata on the image reliably prevents that, because the metadata has to be maintained by hand and the failure appears only when it is wrong.

Shipping `agynd` with the platform removes the possibility rather than detecting it. The agent runtime image then holds nothing platform-coupled, so its versions can be pinned freely — its tags correspond to agent CLI versions, which is a choice a user can actually make.

An agent CLI upgrade is a new agent runtime image, published without a platform release. An `agynd` change — including one adapting to a CLI's protocol change — ships with the platform. The residual coupling is `agynd`'s adapter against a CLI version a user pinned, which fails loudly at spawn, with `agynd` present to report it.

### Shared Volume Contract

All three init containers write into an `emptyDir` mounted at `/agyn`, which the main container also mounts:

```
/agyn/
├── bin/
│   ├── agynd      # daemon binary            (agynd-cli-init)
│   ├── tmux       # shell multiplexer        (agynd-cli-init)
│   ├── agyn       # platform CLI binary      (agyn-cli-init)
│   └── codex      # agent CLI binary         (agent runtime image)
├── tmux.conf      # platform tmux config     (agynd-cli-init)
└── config.json    # runtime config for agynd (agent runtime image)
```

Init containers run in order and write disjoint paths, so the three images compose without coordination beyond the layout above. This is what allows each binary to ship in its own image. `agyn`, `agynd`, and `tmux` are **reserved names** in `bin/`: an agent runtime image must not ship a binary called any of them.

`/agyn/bin` is the single `PATH` entry, and everything on it is a binary — configuration lives beside it at `/agyn`, not on `PATH`. The volume is a delivery surface written by init containers and read by the main container; it is not writable agent state, so no ownership fixing is required for non-root images.

### tmux

`tmux` backs [persistent shells](terminal-proxy.md#persistent-shells) and ships with `agynd` because `agynd` is what runs it — one image, one version pair, no skew between the binary and the process that starts the server.

It is **statically linked against musl**, with the terminfo entries it needs compiled in. Both properties are requirements rather than preferences: a sandbox runs whatever image its environment names, and a distro build would fail on a musl base for its libc and on a slim base for `libtinfo`, while a glibc static build would fail `getpwuid` — which tmux calls to find the user's shell — because NSS cannot be linked statically. Compiled-in terminfo removes the last filesystem dependency, so the server starts in an image carrying no terminal database at all.

Two things it still needs from the image, and neither is new: a shell for it to spawn, and a writable directory for its socket. The [Sandboxes App](../product/sandboxes/sandboxes-app.md#constraints) already states the first; `agynd` handles the second by placing the socket where the platform controls it rather than in the image's `/tmp`.

Delivering it on `PATH` rather than beside the configuration is deliberate. It shadows any `tmux` the image ships, which is the point: a client and a server disagreeing on protocol version fail to connect, and an engineer running `tmux` inside a shell would otherwise meet that error with no way to interpret it. One binary means one version. The platform's own server is kept off the default socket regardless, so a personal `tmux` still gets its own server and its own configuration.

The agent CLI binary keeps its original name — someone who execs into the container to debug sees the real one.

`PATH` is established by whoever starts a process, because nothing inside the container extends it on its own:

| Process | How `/agyn/bin` gets on `PATH` |
|---|---|
| Agent subprocess | `agynd` prepends it when spawning |
| Interactive session | The [Terminal Proxy](terminal-proxy.md#session-kinds) prepends it in the command it binds, expanded inside the container so the image's own `PATH` survives |
| Persistent shell | The [tmux configuration](agynd-cli.md#shell-supervision) prepends it in the command the server spawns — the same construction, in the only place that reaches a shell the client did not start |
| Platform-invoked process | Not at all — invoked by absolute path |

### config.json

Written by the agent runtime image, read by `agynd` at startup:

```json
{
  "sdk": "codex",
  "bin": "codex"
}
```

| Field | Type | Values | Description |
|---|---|---|---|
| `sdk` | string | `codex`, `claude`, `agn`, … | Which SDK module `agynd` uses to communicate with the agent CLI |
| `bin` | string | relative path | The agent CLI binary's location under `/agyn/bin`, relative to it — normally just the binary's name |

`bin` is **relative to the volume**, not an absolute path. An image states what it carries; where the platform mounts it is the platform's business, and an image that hardcodes a mount point has to be rebuilt whenever that mount moves.

The image describes itself. The [Agents Orchestrator](agents-orchestrator.md) sets no binary paths, no SDK types, and no agent CLI arguments — it treats the agent runtime image as an opaque reference to something the catalog resolves, exactly as it treats the workspace image.

Keeping this file in the image rather than moving it onto the catalog record means an image cannot be mislabelled by whoever registered it: what runs is decided by what is actually inside. The cost is that the catalog does not know which agent CLI a given `agent_runtime` image provides, and conveys it only through the name and description its registrar chose.

## Platform Binary Images

One image per binary, each published by the repository that owns it:

| Image | Published by | Contents |
|---|---|---|
| `ghcr.io/agynio/agynd-cli-init:<platform-version>` | `agynio/agynd-cli` | `agynd`, statically linked (`CGO_ENABLED=0`) |
| `ghcr.io/agynio/agyn-cli-init:<platform-version>` | `agynio/agyn-cli` | `agyn`, statically linked |

Splitting them is a release decision. A single combined image cannot be built by either binary's repository, since neither owns both — it needs a third repository whose only content is a Dockerfile and a version-pin file, and every bump to either binary becomes a pull request there. One image per repository makes a release *build the binary, build the image, push*, with nothing to coordinate.

They are pinned together by the chart to the platform version, so splitting the packaging does not let them drift; what coupled them was never the versioning.

The `-init` suffix records how the image is delivered: its entire job is to copy content into a shared volume. An image from the same repository that later runs as something other than an init container takes a different suffix rather than overloading these.

The Orchestrator reads both references from configuration. There is no per-agent, per-environment, or per-organization override: an agent's behaviour is configured through the agent, and the platform's own binaries are not a configuration surface.

Both are injected into **every** workload, including [sandboxes](../product/sandboxes/sandboxes.md) whose environment names no agent runtime image — which is what makes `agyn` available inside a plain sandbox.

## Agent Runtime Images

One image per agent CLI, each holding a binary and a `config.json`:

| Image | Contents |
|---|---|
| `ghcr.io/agynio/agyn-runtime-codex:<codex-version>` | `codex` (static musl binary) + `config.json` with `sdk: codex` |
| `ghcr.io/agynio/agyn-runtime-claude:<claude-version>` | `claude` (native `linux-x64-musl` binary) + `libgcc`/`libstdc++` + `config.json` with `sdk: claude` |
| `ghcr.io/agynio/agyn-runtime-agn:<agn-version>` | `agn` (static Go binary) + `config.json` with `sdk: agn` |

Each lives in its own repository, so bumping one CLI rebuilds nothing else. A repository is a Dockerfile, a `config.json`, a version pin, and a release workflow; it pins only its own CLI version, since `agynd` and `agyn` are no longer among its contents.

Tags correspond to agent CLI versions, which is what makes them meaningful in a [version picker](../product/images/images.md#selecting-a-version). Sizes are modest for the same reason — the platform binaries that dominated these images are elsewhere.

All binaries are statically linked and run on any Linux base image. The Claude Code CLI's musl variant depends on `libgcc` and `libstdc++`, which its image bundles alongside the binary.

## Binary Delivery

The init containers copy their contents into the `emptyDir`. The copy happens on every Pod start, not once per node, so a large agent CLI is copied again on each cold start and the `emptyDir` occupies node storage for the life of the Pod.

A runner may satisfy the same contract by mounting the image contents read-only instead of copying them, where its cluster supports doing so. That is a [runner](runner.md)-side choice: the platform declares which images must be available at which paths, and the runner decides how. Mounting read-only additionally prevents anything in the Pod from replacing `agynd` or the agent CLI while the workload runs, which a writable `emptyDir` does not.

## Startup Sequence

```mermaid
sequenceDiagram
    participant K as Kubernetes
    participant AD as agynd-cli-init
    participant AC as agyn-cli-init
    participant AR as agent-runtime init
    participant ZS as Ziti Sidecar
    participant MC as Main Container
    participant GW as Gateway

    K->>AD: Start agynd-cli-init
    AD->>AD: cp agynd → /agyn/bin
    AD-->>K: Exit 0

    K->>AC: Start agyn-cli-init
    AC->>AC: cp agyn → /agyn/bin
    AC-->>K: Exit 0

    K->>AR: Start agent runtime init container
    AR->>AR: cp agent CLI → /agyn/bin, config.json → /agyn
    AR-->>K: Exit 0

    K->>ZS: Start Ziti sidecar container
    ZS->>ZS: Enroll OpenZiti identity (JWT)
    Note over ZS: Resolves gateway.agyn / llm-proxy.agyn, intercepts via DNS + TPROXY

    K->>MC: Start main container
    Note over MC: command: /agyn/bin/agynd

    MC->>MC: Read /agyn/config.json → sdk type, bin path
    MC->>GW: GetAgent + ListSkills + ListInitScripts + ListMCPs (agent and environment scopes)
    GW-->>MC: Agent config, skills, init scripts, MCP definitions
    MC->>MC: Prepare environment (skills to filesystem, LLM Proxy config, MCP endpoints)
    MC->>MC: Execute init scripts in order (environment's, then the agent's; /bin/sh -lc each)
    MC->>MC: Spawn agent CLI at bin path (via SDK module)
    Note over MC: Child PATH includes /agyn/bin
    MC->>GW: Subscribe to notifications
    MC->>MC: Begin message sync loop
```

Two workloads deviate from this sequence:

- **A workspace-only environment** (no agent runtime image) runs the two platform init containers alone. No agent CLI and no `config.json` are present.
- **A [sandbox](../product/sandboxes/sandboxes.md)** runs `agynd` as a long-lived holder, whether or not its environment names an agent runtime image. It fetches the environment-scoped configuration, prepares the agent CLI as far as the environment determines — its configuration file, its [first-run state](agynd-cli.md#agent-cli-first-run-state), any file placeholder — and runs the environment's [init scripts](resource-definitions.md#initscript), there being no agent and so no agent-scoped ones. Then it holds, spawning no agent CLI and running no inbox loop — preparing a CLI it will not spawn because a person will, by hand, and an unconfigured one stops to ask them a question. See [Preparation in Holder Mode](agynd-cli.md#preparation-in-holder-mode). What it does start is the [tmux server](agynd-cli.md#shell-supervision) that holds the sandbox's [persistent shells](terminal-proxy.md#persistent-shells), which is the one thing in a holder that must exist before anyone connects.

No agent CLI being spawned, nothing extends `PATH` for the container at large, and `/agyn/bin` is reachable only by absolute path unless a session establishes `PATH` itself. Both kinds of session do, in the two places listed above, which is what puts `agyn` and the environment's agent CLI on `PATH` for a person driving a sandbox by hand.

## Environment Variable Contract

What the Orchestrator passes to the main container:

| Env Var | Source | Description |
|---|---|---|
| `AGENT_ID` | Agent resource | Agent class UUID |
| `AGENT_NAME` | Agent resource | Agent name |
| `AGENT_ROLE` | Agent resource | Agent role label |
| `AGENT_MODEL` | Agent resource | Model UUID reference |
| `AGENT_CONFIG` | Agent resource | Opaque configuration JSON |
| `AGENT_INSTANCE_ID` | Reconciler | [Agent instance](agent-instances.md) UUID this workload serves |
| `WORKLOAD_ID` | Reconciler | Workload UUID for activity keepalives and span attribution |
| `GATEWAY_ADDRESS` | Orchestrator config | Single Gateway endpoint (e.g., `gateway.agyn`) |
| `AGENT_MCP_SERVERS` | MCP sub-resources | Comma-separated `name:port` pairs. See [MCP — Port Allocation](mcp.md#port-allocation) |
| `LLM_MODE` | Environment | `platform` or `native`. Whether `agynd` writes LLM endpoint configuration at all |
| `LLM_MODEL_NAME` | Agent resource | Vendor model name to pin in `native` mode. Absent otherwise, and absent for sandboxes |
| Vendor placeholder credential | Resolved [Subscription](providers.md#subscription) | A dummy credential per resolving vendor whose placeholder is an environment variable, so the agent CLI starts. Set on the container because a sandbox's interactive session inherits the container's environment, not `agynd`'s. File-kind placeholders are written by `agynd` instead — see [Placeholder Delivery](providers.md#placeholder-delivery) |

Nothing identifies the agent CLI. `agynd` reads that from `config.json` — which is also why a placeholder that has to land at a CLI-specific path is `agynd`'s to write and not the orchestrator's.

## Pod Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-<uuid>
  labels:
    agyn.dev/managed-by: agents-orchestrator
    agyn.dev/agent-id: <agent-uuid>
spec:
  restartPolicy: Never
  automountServiceAccountToken: false
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 127.0.0.1          # Ziti sidecar DNS
      - <cluster-dns-ip>   # CoreDNS fallback

  imagePullSecrets:
    - name: workload-<uuid>-pull          # image proxy credential for this workload

  initContainers:
    - name: agynd-cli-init
      image: ghcr.io/agynio/agynd-cli-init:<platform-version>   # chart-pinned, not proxied
      volumeMounts:
        - name: agyn
          mountPath: /agyn

    - name: agyn-cli-init
      image: ghcr.io/agynio/agyn-cli-init:<platform-version>    # chart-pinned, not proxied
      volumeMounts:
        - name: agyn
          mountPath: /agyn

    - name: agent-runtime
      image: <proxy-host>/<org>/<image-name>:<tag>   # from environment.agent_runtime_image
      volumeMounts:
        - name: agyn
          mountPath: /agyn

    - name: ziti-sidecar
      image: ghcr.io/agynio/ziti-sidecar:latest
      restartPolicy: Always
      securityContext:
        capabilities:
          add: ["NET_ADMIN"]
      env:
        - name: ZITI_ENROLLMENT_JWT
          value: "<jwt>"

  containers:
    - name: agent-<short-id>
      image: <proxy-host>/<org>/<image-name>:<tag>   # from environment.workspace_image
      command: ["/agyn/bin/agynd"]
      env:
        - name: AGENT_ID
          value: "<uuid>"
        - name: GATEWAY_ADDRESS
          value: "<gateway-addr>"
        - name: AGENT_INSTANCE_ID
          value: "<uuid>"
        - name: WORKLOAD_ID
          value: "<uuid>"
        - name: AGENT_MCP_SERVERS
          value: "filesystem:8100,github:8101"
        # ... user-defined agent ENVs (resolved secret values included)
      volumeMounts:
        - name: agyn
          mountPath: /agyn
        - name: env-workspace                        # from environment.volumes
          mountPath: /workspace

    - name: mcp-<short-id>
      image: <proxy-host>/<org>/<image-name>:<tag>   # from mcp.image
      volumeMounts:
        - name: env-workspace                        # named in mcp.shared_volumes
          mountPath: /workspace                      # same path the main container uses
        - name: mcp-<short-id>-index                 # the sidecar's own volume
          mountPath: /var/lib/index
      # ... (only receives its own MCP ENVs — no agent secrets)

  volumes:
    - name: agyn
      emptyDir: {}
    - name: env-workspace                            # PVC for this owner, or emptyDir when not persistent
      persistentVolumeClaim:
        claimName: <owner-uuid>-workspace
    - name: mcp-<short-id>-index
      persistentVolumeClaim:
        claimName: <owner-uuid>-<mcp-short-id>-index
```

Pod-level volume names are generated by the orchestrator from the declaring resource, which is why an environment volume and an MCP volume may share a `name`. A persistent volume's claim is per **owner** — the agent instance or sandbox — so the same environment declaration produces a different `claimName` in every workload that runs it. An environment declaring no volumes leaves `agyn` as the only entry here.

Catalog images — the workspace image, the agent runtime image, and each MCP's — are [proxy](image-proxy.md) references. Platform containers are not: `agynd-cli-init`, `agyn-cli-init` and the Ziti sidecar are chart-pinned and pulled directly from a public registry, because the proxy is itself a platform component and cannot serve the containers a workload needs before it is reachable. No **user-configured** image reference reaches a workload as an upstream registry address.

## Related Architecture

- [Images (product)](../product/images/images.md)
- [Image Proxy](image-proxy.md)
- [Flavors and Environments](../product/environments/environments.md)
- [Agents Orchestrator](agents-orchestrator.md)
- [agynd](agynd-cli.md)
- [agyn CLI](agyn-cli.md)
- [k8s-runner](k8s-runner.md)
