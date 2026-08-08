# OpenAI Subscriptions and Vendor Naming

## Target

- [Providers, Models, and Secrets — Vendors](../architecture/providers.md#vendors)
- [Providers, Models, and Secrets — Placeholder Delivery](../architecture/providers.md#placeholder-delivery)
- [LLM Service — Subscription Management](../architecture/llm.md#subscription-management)
- [agynd — Native Mode Configuration](../architecture/agynd-cli.md#native-mode-configuration)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Agent Init Container — Environment Variable Contract](../architecture/agent-init.md#environment-variable-contract)
- [OpenZiti — Static Policies](../architecture/openziti.md#static-policies)
- [LLM Proxy — Vendor Intercept Services](../architecture/llm-proxy.md#vendor-intercept-services)
- [Metering — Labels](../architecture/metering.md)
- [agyn CLI — Subscription Commands](../architecture/agyn-cli.md#subscription-commands)

## Delta

[Native LLM mode](../architecture/llm-proxy.md#native-mode) shipped with one working vendor. `CreateSubscription` returns `UNIMPLEMENTED` for the other, so an organization paying for a ChatGPT plan cannot use the platform at all — the case native mode exists to serve, half-served.

Two changes, one of which is the cause of the other.

### Vendors are named after APIs, not agent CLIs

`vendor` is `claude` | `codex`. It becomes `anthropic` | `openai`.

This is not cosmetic. Naming the enum after CLIs is what produced the incoherent OpenAI binding that got it disabled: the row paired `chatgpt.com` — the *subscription* host a Codex CLI calls at `/backend-api/codex/responses` — with `OPENAI_API_KEY`, the variable that puts a Codex CLI in *API-key* mode, where it addresses `api.openai.com` instead. Two mutually exclusive configurations in one row, because the row was named after the client rather than the API it opens. A credential belongs to the vendor whose API it authenticates; which CLI happens to present it is incidental and free to change.

The rename carries through the OpenZiti service names (`llm-intercept-anthropic`, `llm-intercept-openai`), the `llm-native-<vendor>` identity role attributes, the static Dial policies, the `vendor` metering label, and `agyn subscriptions create --vendor`.

**Existing subscription rows carry the old values and must be migrated.** So must any OpenZiti service, policy, and identity role attribute already provisioned under the old names — a workload holding `llm-native-claude` dials a service that no longer exists, and stops reaching its vendor.

### OpenAI becomes a supported vendor

`CreateSubscription` stops returning `UNIMPLEMENTED`. What was actually missing was a placeholder mechanism, not anything about the vendor.

The placeholder — the dummy credential an agent CLI needs in order to start, which the proxy discards and replaces — is delivered as an environment variable on the container spec. Codex's subscription mode does not read one: its credential lives in `~/.codex/auth.json`. So the mechanism gains a second kind:

| Kind | Written by | Why it reaches a sandbox shell |
|---|---|---|
| `env` (`anthropic` → `CLAUDE_CODE_OAUTH_TOKEN`) | Orchestrator, onto the container spec | The runner's `Exec` carries no environment of its own, so an interactive session inherits the container spec's — and nothing else |
| `file` (`openai` → `~/.codex/auth.json`) | `agynd`, at container start, in agent and holder mode alike | The file lands on the container filesystem and stays there, so a session exec'd hours later finds it |

The file kind must be `agynd`'s and cannot be the orchestrator's: the path is CLI-specific and resolves against `HOME`, which the platform does not manage, while `agynd` reads the CLI from `config.json`. Holder mode spawns no subprocess but is not inert — it already runs the environment's init scripts, and this is the same kind of preparation.

`ListSubscriptionAttachments` currently returns `placeholder_env`. It must return `placeholder_kind` (`env` | `file`) plus the variable name or the file's contents template, so neither the orchestrator nor `agynd` carries a vendor table of its own.

The OpenAI row's host and upstream are `chatgpt.com` and `https://chatgpt.com/backend-api/codex`.

## Acceptance Signal

- An environment in `native` mode with an OpenAI subscription attached runs a sandbox in which `codex` starts and answers a prompt, with `~/.codex/auth.json` present and holding the placeholder.
- `env` and every credential path in that sandbox yield the placeholder, never the subscription token.
- `agyn subscriptions create --vendor openai` succeeds; `--vendor codex` is rejected as an unknown vendor.
- An Anthropic-subscription sandbox is unchanged in every respect — same placeholder variable, same interception, same metering.
- Subscription rows created before this change resolve under their new vendor value, and their workloads reach their vendor on the first start after migration.
- Native-mode metering records carry `vendor=anthropic` or `vendor=openai`.

## Notes

- **The migration is the risk, not the feature.** Renaming a shipped enum touches database rows, OpenZiti services, static policies, and role attributes stamped on live workload identities. A partial migration leaves workloads dialing services that no longer exist, and the symptom — vendor traffic silently not intercepted — looks like a network fault rather than a rename.
- **Neither credential is refreshed by the platform**, and OpenAI's expires sooner than Anthropic's. Whatever minted it keeps the [Secret](../architecture/providers.md#secret) current. Same limitation [egress rule injection](../product/egress-gateway/egress-gateway.md#constraints) documents; it is a property of the credential, not grounds for treating a vendor as unsupported.
- **How Codex frames its calls over the wire is not specified here** and does not need to be: interception and injection happen at the connection and request level, and the proxy forwards what it receives. If per-call observability turns out to differ between vendors once this runs, that is a metering question to answer with a measurement, not a design branch to take in advance.
- Obtaining either token stays out of scope. A subscription references a Secret an operator created by whatever means the vendor offers.
