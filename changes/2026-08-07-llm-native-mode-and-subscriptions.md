# LLM Native Mode and Subscriptions

## Target

- [Flavors and Environments — LLM Access](../product/environments/environments.md#llm-access)
- [Sandboxes — What's Inside](../product/sandboxes/sandboxes.md#whats-inside)
- [Providers, Models, and Secrets — Subscription](../architecture/providers.md#subscription)
- [Providers, Models, and Secrets — Subscription Attachment](../architecture/providers.md#subscription-attachment)
- [Resource Definitions — Environment](../architecture/resource-definitions.md#environment)
- [Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [LLM Service — Subscription Resolution](../architecture/llm.md#subscription-resolution)
- [LLM Proxy — Native Mode](../architecture/llm-proxy.md#native-mode)
- [OpenZiti — Identity Resolution](../architecture/openziti.md#identity-resolution)
- [OpenZiti — Static Policies](../architecture/openziti.md#static-policies)
- [agynd — Native Mode Configuration](../architecture/agynd-cli.md#native-mode-configuration)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Agent Init Container — Environment Variable Contract](../architecture/agent-init.md#environment-variable-contract)
- [Metering](../architecture/metering.md)
- [Secrets — Referential Integrity](../architecture/secrets.md#referential-integrity)
- [agyn CLI — Subscription Commands](../architecture/agyn-cli.md#subscription-commands)

## Delta

A workload can reach an LLM exactly one way today: [`agynd`](../architecture/agynd-cli.md) writes the [LLM Proxy](../architecture/llm-proxy.md) endpoint into the agent CLI's configuration, and the model is a platform [Model](../architecture/providers.md#model) UUID resolved to an [LLM Provider](../architecture/providers.md#llm-provider) holding an API key. An organization that pays for a vendor's own subscription plan has no API key to put in a provider and no platform model to reference, so it cannot use the platform at all. A [sandbox](../product/sandboxes/sandboxes.md) is worse off still: `agynd` runs as a holder and writes no LLM configuration whatsoever, so the agent CLI an engineer finds on `PATH` has no credentials in either case.

### `llm_mode` on the environment

The environment gains `llm_mode` (`platform` | `native`, default `platform`) and `llm_allowed_models`. Neither exists.

`platform` is today's behavior, unchanged. `native` means the agent CLI is left in its stock configuration and its vendor-bound traffic is intercepted onto the LLM Proxy, which injects the credential.

The mode lives on the environment and nowhere else — a sandbox has no agent to read it from, it is a mode rather than an additive attachment, and it determines what the Orchestrator stamps on the workload identity. Changing it on an environment any agent references is rejected.

### Subscription and Subscription Attachment

Two new resources in the [LLM service](../architecture/llm.md), which today owns providers and models only:

- `Subscription` — org-scoped, holding a `vendor` from a closed set (`claude`, `codex`), a `secret_id`, and an optional `account_id`. The token is held **by reference** to a [Secret](../architecture/providers.md#secret), not inline as `LLMProvider.token` is.
- `SubscriptionAttachment` — binds one to an agent or an environment, **unique on `(vendor, target)`**. Agent scope shadows environment scope for the same vendor.

The uniqueness constraint is load-bearing, not a simplification: an intercepted request carries nothing that identifies a credential, so `(caller, vendor)` must resolve to exactly one.

The LLM service gains subscription and attachment CRUD, `ResolveSubscription(agent_id, environment_id, vendor)`, `CountSubscriptionsReferencingSecret` for the Secrets service, an existence check against Secrets on create/update, and `subscription.updated` / `subscription_attachment.updated` events on the organization's Notifications room.

`ResolveSubscription` must return everything the proxy needs to serve the connection, so the proxy performs no configuration lookups of its own: the `subscription_id` (metering's `resource_id`, with no other source) and the environment's `allowed_models` alongside the credential and upstream. That gives the LLM service a new read against the [Agents](../architecture/agents-service.md) service — acceptable because it sits on a path that runs once per connection, and because the alternative puts that call on the proxy's per-request path and contradicts the no-Agents-dependency property that `environment_id`-on-the-identity exists to provide.

The vendor's `placeholder_env` is returned on the **attachment listing** only, not on `ResolveSubscription`. The proxy strips `Authorization` and `x-api-key` unconditionally and never needs the name; the orchestrator is its only consumer.

### Native-mode interception

One OpenZiti service per vendor (`llm-intercept-claude`, `llm-intercept-codex`), bound by the LLM Proxy, with static Bind and Dial policies. None exist.

Nothing here is provisioned per organization, per environment, or per attachment — unlike [egress rules](../architecture/egress-rules-service.md), whose destinations are user-authored. The per-workload decision is carried by new `llm-native-<vendor>` identity role attributes instead, so this feature introduces no dynamic OpenZiti resources and no reconciliation loop.

### LLM Proxy: a second listener

The proxy serves one plain-HTTP surface today. It must additionally bind the vendor intercept services, terminate TLS for the vendor hostname with a leaf minted from the [Egress CA](../architecture/egress-gateway.md#egress-ca), and serve the vendor's own API surface as a passthrough:

- The mode is never inferred from a request — the two arrive on different binds, and the vendor is known from SNI before the body is read.
- The `model` field is **not** rewritten and the caller endpoint is **not** validated against a provider protocol; there is no Model resource involved.
- The caller's `Authorization` is stripped and the subscription token injected. Every other header and the entire body pass through untouched.
- Bindings resolve once per connection, not per request, and are dropped on Notifications events so a detached or rotated credential does not survive a long-lived connection.
- Vendor errors — expired credential, rate limit, quota — pass through unchanged, so the CLI's own handling works.
- `native` is not exposed on the public ingress. It authenticates by workload identity alone.

### Identity resolution carries the environment

`ResolveIdentity` returns `(identity_id, identity_type, workload_id)`. It must also return `agent_id` and `environment_id` for workload identities, recorded at identity creation. Without this the proxy cannot resolve a subscription without calling the Agents service on the request path, or must trust a value the workload asserts about itself. Neither is acceptable.

### agynd writes less

In `native` mode `agynd` must write **no** endpoint configuration — no base URL, no custom model provider, no platform model UUID.

It must still write the configuration **files**. The mode gates keys, not files: `writeClaudeSettings` emits `permissions`, `env`, and `mcpServers` into one `~/.claude/settings.json`, and Codex's `config.toml` carries `mcp_servers` next to `model_provider`. Omitting the file to satisfy "no endpoint configuration" would drop the environment's MCP sidecar wiring and restore interactive tool approval, stalling a turn on a human who is not there. Only the endpoint and credential keys go.

It learns the mode from a new `LLM_MODE` environment variable rather than by fetching anything. `agynd` has no `GetEnvironment` call today — the endpoint already arrives statically as `LLM_BASE_URL`, injected by the assembler for agent workloads and sandboxes alike — and holder mode has nothing prepared to hang a fetch off. `LLM_MODE` and `LLM_MODEL_NAME` follow the variable that is already there.

**`agynd` must not write the placeholder credential.** An earlier reading of this put it in the subprocess environment `agynd` builds; that cannot work for a sandbox. The runner's `Exec` builds a `PodExecOptions` carrying no environment of its own, so an interactive session inherits the *container spec's* environment — and in holder mode `agynd` spawns no subprocess at all. A placeholder written by `agynd` is invisible to the only session that needs it.

### Orchestrator

Workload spec assembly must resolve the environment's `llm_mode`, ask the LLM service which vendors have a subscription attached, and for each one stamp an `llm-native-<vendor>` role attribute on the workload identity **and inject that vendor's placeholder credential into the main container**, alongside `LLM_MODE` and `LLM_MODEL_NAME`. The placeholder is keyed on the vendor rather than the agent CLI, which is what makes it injectable here — the orchestrator treats the agent runtime image as opaque and does not know which CLI it carries.

A `native` environment with no subscription attached for any vendor must fail assembly with a descriptive error rather than start a workload whose first model call would fail.

### Agent model reference

`Agent.model` becomes conditional on the environment's mode, and a new `Agent.model_name` holds an opaque vendor model name for `native` mode. Validation belongs at `CreateAgent`/`UpdateAgent` against the environment's mode — required-and-exclusive in each direction — so a mismatch fails when someone configures it, not when the agent runs.

### Metering labels

Native-mode records have no `model_id` to attribute to. `resource`/`resource_id` become `subscription` and the subscription UUID, with new `vendor` and `model_name` labels carrying what the Model UUID carried. Native-mode token counts must not be aggregated as spend — a subscription is a flat fee, and summing its tokens alongside API tokens produces a bill that does not exist.

Keeping them apart depends on two things that are **already broken** and are part of this delta, not assumptions it rests on:

- The proxy writes the provider's remote model name into the `resource` label, where [Metering](../architecture/metering.md) specifies a resource *type*. Until that is a type, `resource=subscription` cannot distinguish anything.
- The Console's usage query carries no `resource` filter at all, so there is currently no aggregation boundary to put native-mode tokens outside of.

Both are pre-existing defects. This change is the first thing that depends on them being correct.

## Acceptance Signal

- An environment created with `--llm-mode native` and a Claude subscription attached runs a sandbox in which `claude` starts, answers a prompt, and shows its own model picker — with nothing written to `~/.claude/settings.json` about an endpoint. The session that does this is an ordinary `agyn sandbox connect`, so the placeholder must be visible in `env` from that shell: it comes from the container spec, not from `agynd`.
- `cat` of every credential path in that sandbox, and `env`, yield the placeholder — never the subscription token.
- The call appears in tracing and metering, labelled `resource=subscription` with the vendor model name, and does not appear in any spend aggregation.
- Detaching the subscription stops the next call on an already-open connection, without restarting the sandbox.
- An environment in `native` mode with no subscription attached refuses to start a sandbox, and `agyn environments show` flags it before anyone tries.
- A second Claude subscription attached to the same environment is refused, naming the existing one.
- `DeleteSecret` on a secret a subscription references is refused, naming the subscription.
- An agent in a `platform` environment is unchanged in every respect — same configuration written, same resolution path, same metering labels.
- `CreateAgent` naming a `native` environment is rejected when it carries a `model` UUID, and accepted with `model_name` unset.
- A workload in a `platform` environment reaching `api.anthropic.com` is not intercepted — its identity carries no `llm-native-*` attribute.

## Notes

- **Obtaining the token is out of scope.** A subscription references a [Secret](../architecture/providers.md#secret) an operator created by whatever means the vendor offers. Capturing it through the `agyn` CLI is a separate change and does not block this one.
- **Claude first; `codex` is declared, not shipped.** `CreateSubscription` returns `UNIMPLEMENTED` for it. The row in the vendor table records the intended host, upstream, and protocol so the shape is fixed, but nothing can create a record that would produce a working workload. Two independent gaps stand in the way, and closing either alone is not enough:
  - **No coherent placeholder.** The variable a Codex CLI reads (`OPENAI_API_KEY`) selects its *API-key* mode, and a Codex CLI in API-key mode addresses `api.openai.com` — not the `chatgpt.com` the intercept service captures. The subscription-mode credential lives in `~/.codex/auth.json`, and the placeholder mechanism specified here injects environment variables only. A file-delivered placeholder is not specified anywhere in this change.
  - **No refresh.** The ChatGPT-plan credential is a short-lived, refreshing token pair, and the platform refreshes nothing — the same limitation [egress rule injection](../product/egress-gateway/egress-gateway.md#constraints) already documents. A usable Codex subscription needs an external refresher or a vendor-side long-lived credential.
- **Sequencing.** `agyn environments subscriptions` extends the `agyn environments` command group, which is itself still an open change — this cannot land before it. The `agynio/llm` service repository is not checked out in the current workspace, so the resource and resolution work there has not been sized against real code the way the orchestrator, runner, and proxy sides have.
- **Prototype before building the proxy side.** Four unknowns are cheaper to settle with a throwaway intercepting proxy than with a design argument: whether Claude Code accepts a placeholder credential without a local or preflight validation call; whether it addresses hosts beyond `api.anthropic.com` that would also need intercepting for quota and status to work; **what ALPN the vendor APIs and the CLIs negotiate, and what the CLI does when the intercept offers h2 versus http/1.1**; and whether vendor abuse heuristics tolerate sandbox-concurrency traffic from cluster egress IPs.
- **Guardrails are a stub here.** `llm_allowed_models` is the only one specified, because it is the one that falls out of the proxy already reading the model name. Its enforcement needs the proxy to subscribe to `environment.updated` — which [Agents already publishes](../architecture/agents-service.md#notifications) — alongside the subscription events, or a tightened allowlist would not reach a connection a sandbox is holding open. Next-connection semantics were considered and rejected: a restriction that waits hours to apply is not one. Content inspection and spend caps need their own design, and output-side inspection on streaming responses in particular has a real fork — buffer and lose the streaming UX, or scan incrementally and only ever cut off mid-stream.
- **Per-initiator credentials are deliberately absent.** A subscription attaches to an agent or an environment, never to a user, so an environment shared by several engineers uses one credential for all of them. Attaching per initiator extends the resolution key with a third term rather than changing its shape — the `(vendor, target)` uniqueness constraint is what keeps that door open.
- **The LLM service now holds credentials two ways** — inline for providers, by reference for subscriptions. Converging `LLMProvider.token` onto `secret_id` is a data migration and is not part of this change.
- Native mode is a default, not a wall: an engineer in a sandbox can authenticate the CLI themselves and bypass the platform path. Composing a `deny` [egress rule](../product/egress-gateway/egress-gateway.md) on the vendor hostnames closes that, and needs no new mechanism.
