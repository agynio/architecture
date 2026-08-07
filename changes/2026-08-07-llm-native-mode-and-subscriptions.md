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

In `native` mode `agynd` must write **no** endpoint configuration — no base URL, no custom model provider, no platform model UUID. It writes a correctly-shaped placeholder credential into the subprocess environment so the agent CLI will start, and the agent's `model_name` through the CLI's own model setting when one is set.

### Orchestrator

Workload spec assembly must resolve the environment's `llm_mode`, ask the LLM service which vendors have a subscription attached, and stamp an `llm-native-<vendor>` role attribute per vendor on the workload identity. A `native` environment with no subscription attached for any vendor must fail assembly with a descriptive error rather than start a workload whose first model call would fail.

### Agent model reference

`Agent.model` becomes conditional on the environment's mode, and a new `Agent.model_name` holds an opaque vendor model name for `native` mode. Validation belongs at `CreateAgent`/`UpdateAgent` against the environment's mode — required-and-exclusive in each direction — so a mismatch fails when someone configures it, not when the agent runs.

### Metering labels

Native-mode records have no `model_id` to attribute to. `resource`/`resource_id` become `subscription` and the subscription UUID, with new `vendor` and `model_name` labels carrying what the Model UUID carried. Native-mode token counts must not be aggregated as spend — a subscription is a flat fee, and summing its tokens alongside API tokens produces a bill that does not exist.

## Acceptance Signal

- An environment created with `--llm-mode native` and a Claude subscription attached runs a sandbox in which `claude` starts, answers a prompt, and shows its own model picker — with nothing written to `~/.claude/settings.json` about an endpoint.
- `cat` of every credential path in that sandbox yields the placeholder, never the subscription token.
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
- **Claude first.** The `claude` vendor is a static bearer and closes end to end. The `codex` vendor's ChatGPT-plan credential is a short-lived, refreshing token pair, and the platform refreshes nothing — the same limitation [egress rule injection](../product/egress-gateway/egress-gateway.md#constraints) already documents. `codex` is specified here so the shape is fixed, but a usable Codex subscription needs either an external refresher or a vendor-side long-lived credential, and that question is not answered by this change.
- **Prototype before building the proxy side.** Three unknowns are cheaper to settle with a throwaway intercepting proxy than with a design argument: whether Claude Code accepts a placeholder credential without a local or preflight validation call; whether it addresses hosts beyond `api.anthropic.com` that would also need intercepting for quota and status to work; and whether vendor abuse heuristics tolerate sandbox-concurrency traffic from cluster egress IPs.
- **Guardrails are a stub here.** `llm_allowed_models` is the only one specified, because it is the one that falls out of the proxy already reading the model name. Content inspection and spend caps need their own design, and output-side inspection on streaming responses in particular has a real fork — buffer and lose the streaming UX, or scan incrementally and only ever cut off mid-stream.
- **Per-initiator credentials are deliberately absent.** A subscription attaches to an agent or an environment, never to a user, so an environment shared by several engineers uses one credential for all of them. Attaching per initiator extends the resolution key with a third term rather than changing its shape — the `(vendor, target)` uniqueness constraint is what keeps that door open.
- **The LLM service now holds credentials two ways** — inline for providers, by reference for subscriptions. Converging `LLMProvider.token` onto `secret_id` is a data migration and is not part of this change.
- Native mode is a default, not a wall: an engineer in a sandbox can authenticate the CLI themselves and bypass the platform path. Composing a `deny` [egress rule](../product/egress-gateway/egress-gateway.md) on the vendor hostnames closes that, and needs no new mechanism.
