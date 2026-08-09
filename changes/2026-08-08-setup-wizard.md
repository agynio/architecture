# Console Setup Wizard

## Target

- [Setup Wizard](../product/console/setup-wizard.md) (new)
- [Console — Entry Flows](../product/console/console.md#entry-flows)
- [Console — Context Switcher](../product/console/console.md#context-switcher)
- [Console — Overview](../product/console/console.md#overview)
- [Console — Empty States](../product/console/console.md#empty-states)
- [Flavors and Environments — Vendor terms](../product/environments/environments.md#vendor-terms)

## Delta

**A new organization has no guided path to a working state.** [Cloud Onboarding](../product/console/console.md#entry-flows) ends at "user configures LLM providers and models, creates agents, and invites teammates" — four resource types in dependency order that the Console never states. Each section offers its own empty state ("No agents yet. Create your first agent."), and following them in the order they appear fails: an agent needs an environment with an agent runtime image, and a model before that. Nothing surfaces that ordering, that a workload with no volume keeps nothing across a restart, or that an agent references a model rather than a provider.

This change adds a **setup wizard** that runs on a user's first organization and builds the first working setup end to end. It creates ordinary resources through the same APIs every other surface uses — no privileged path, no resource type of its own, nothing marked as wizard-owned — and everything it creates is editable afterward in the section it came from.

### The flow

One question, then three or four steps, then a finish overlay.

```
What to try first? ─┬─ agent    → Environment → LLM → Files tool → Agent   → Finish
                    └─ sandbox  → Environment → LLM →              Sandbox → Finish
```

- **Step 0 — what to try first.** Chat with an agent, or work in a sandbox. Written as outcomes rather than as the platform's nouns, and as *first* rather than *instead*: the answer selects what the wizard finishes with, not what the organization is. Its payload is narrow — whether a **subscription** may be offered for Claude — and everything else that differs between the paths follows from that plus the absence of an agent on the sandbox path.
- **Step 1 — environment.** Runtime is a choice (Claude Code or Codex); workspace is a stated fact where the catalog offers one image, not a one-item picker. Creates an environment with a **persistent `/workspace` volume** — the wizard's own curation, not a platform default, because both things it can produce are worse without one.
- **Step 2 — LLM.** Asks how the organization pays, never which `llm_mode` to use; the mode is derived and never shown. The API-key branch does not advance until the credential passes the same `"Hello, world"` call as the [model test dialog](../product/console/console.md#llm-providers-and-models). The subscription branch creates a Secret, a Subscription, and an attachment, sets `native` mode, and makes the environment `private`.
- **Step 3 — files tool.** Agent path only. [files-mcp](../architecture/files-mcp.md) as one toggle, on by default, attached to the environment.
- **Step 4 — agent or sandbox.** The agent form is name, a starter behavioral configuration, and preselected model and environment. The sandbox is created **and started**.
- **Finish.** The Console dims. Centre: what was built, in the platform's nouns, plus four capabilities as plain non-interactive text. Top left: the [product switcher](../product/console/console.md#top-bar) already open with the destination highlighted, landing on a conversation with the new agent or on the running sandbox — not on the app's home.

### Supporting changes

- **[Overview](../product/console/console.md#overview) gains a pre-setup state.** While the organization has neither an agent nor a sandbox, the counter grid is replaced by a prompt to run the wizard. This is the re-entry point for an abandoned run and for every organization after a user's first, where the wizard does not start on its own.
- **[Vendor terms](../product/environments/environments.md#vendor-terms) is new on the environments spec.** Subscriptions are consumer-plan credentials, and vendor terms differ along the line the platform already draws between agents and sandboxes: an `anthropic` subscription does not cover autonomous agents, an `openai` one does. The platform does not enforce it — it cannot distinguish the two uses, and a refusal would assert a vendor's policy with no way to track it changing — so the constraint is recorded where subscriptions are defined and repeated by the surfaces that offer the choice.

### What the wizard deliberately does not do

- **It never asks the user to install anything.** Tunnelers for [private networks](../product/private-networks/private-networks.md) and the tunnel client for [port exposure](../product/port-exposure/port-exposure.md) were the two candidate steps that would have; both need a host, and one may need a colleague. Every step is in the browser and completes in seconds. Private resources are named on the finish overlay as a capability; device enrollment is not named at all — see Notes.
- **It has no persistent checklist.** Removing the steps that could block removed the deferred work a checklist would have tracked. Abandonment is handled by the Overview prompt instead.
- **The finish overlay's capability list carries no links.** They would be competing calls to action on a screen that has exactly one.

## Acceptance Signal

- A user who has never created an organization creates one and is put into the wizard without navigating to it.
- The agent path ends with the user in a conversation with the agent they just created, reached in one click from the switcher — not on Chat's home, and not via a centre button.
- The sandbox path ends with the user at a terminal in a running sandbox, having created no agent.
- Choosing Claude on the agent path offers only an API key, and states why in one line. Choosing Claude on the sandbox path offers a subscription. Codex offers both on either path.
- A wrong API key is rejected on the LLM step, with the provider's error shown, and the step does not advance.
- A subscription-backed environment is created `private`; an API-key-backed one is created `internal`.
- The environment the wizard creates declares a persistent `/workspace`, and the finish overlay names it among what was created.
- Closing the tab after the environment step leaves a complete, valid environment in the organization — not a draft — and the Overview then offers the wizard again.
- The wizard does not start automatically on the same user's second organization; the Overview offers it there instead.
- The Overview prompt disappears once the organization has an agent or a sandbox, and the counter grid returns.
- The finish overlay can be left without going to the destination, and does not appear again once dismissed.
- No copy in the flow says *cloud* or *remote*; the same wording is correct on a [local bundle](../architecture/operations/local-bundle.md) install and on a hosted one.

## Notes

- **The product switcher is still undocumented in this corpus.** The finish overlay makes it load-bearing — it is the flow's only call to action — but the switcher itself remains pre-existing drift, first flagged in [2026-08-07-sandboxes-app](2026-08-07-sandboxes-app.md). Specifying it is not in scope here.
- **Device enrollment has no home after this change.** It was the second candidate step and is not on the finish overlay either, because the overlay's list is Console configuration and enrollment is a per-user action in the user menu. The right moment for it is the first time a user clicks an `exposed-*.agyn` link that does not resolve — a link handler in [Chat](../product/chat/chat.md), at the point of actual motivation, rather than an onboarding surface. Not specified here.
- **The subscription branch has no verification gate.** The test call resolves a Model and `native` mode has none, so a bad subscription token is caught at first use rather than on the step that took it — an asymmetry with the API-key branch that the overlay's copy accounts for and the flow does not otherwise close. A lightweight native-mode probe through the [LLM Proxy](../architecture/llm-proxy.md) would close it and is not specified.
- **Amazon is absent from the vendor presets, and the reason is a request shape rather than an endpoint.** Bedrock API keys are bearer tokens, so auth is not the obstacle; the proxy posts to `/v1/messages` and `/v1/responses` on the provider's base URL, and Bedrock's runtime API exposes neither — the model identifier lives in the path and the streaming envelope differs. That makes it a third protocol, not a third endpoint, and it fails the admission rule that a preset must produce a working model on the first try. Worth confirming against the proxy implementation; if it holds, Bedrock support is a proxy change and adding the preset afterward is one row.
- **Microsoft's preset is a template, not a constant.** The endpoint carries the customer's resource name, and it must be the `/openai/v1` surface — the classic `?api-version=` path does not match what the proxy appends.
- **Starter behavioral configurations are not enumerated.** The spec requires two or three and says why they matter; which personas ship is a content decision.
- **No cluster-admin equivalent.** [Self-Hosted Bootstrap](../product/console/console.md#entry-flows) lands an administrator in Cluster Administration with runners and organizations to register, which this flow does not address. An admin who then creates their own first organization gets the wizard like anyone else.
