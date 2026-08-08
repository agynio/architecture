# Setup Wizard

## Purpose

A new organization is empty, and every path out of it runs through resources whose names mean nothing yet. An owner who has just created an organization has to know that an environment must exist before an agent, that an agent references a model rather than a provider, and that a workload with no volume forgets everything it wrote. None of that is discoverable from the sidebar, and the [empty states](console.md#empty-states) it produces each ask for a resource without saying why.

The setup wizard walks a new organization to its first working thing — an agent to talk to, or a sandbox to work in — and teaches the platform's shape on the way. It creates ordinary resources through ordinary APIs; nothing it produces is special, and everything it produces is editable afterward in the sections it came from.

**It is finishable in one sitting, in the browser.** No step asks the user to install software, reach a host inside a corporate network, or wait on a colleague. Capabilities that need any of those are named at the end as things the platform can do, not asked for as steps — an onboarding flow that stalls on a tunneler install strands the user before they ever see the product work.

## User Stories

- As a new organization owner, I want the platform to build my first working setup with me, so I learn what its pieces are by watching them appear rather than by reading the sidebar.
- As someone evaluating the platform, I want to reach something that visibly works within a couple of minutes, so I can judge it before deciding how much of it to configure.
- As an engineer who wants a shell more than a chatbot, I want a path that ends in a terminal and does not make me create an agent I did not ask for.
- As someone paying for an LLM through a plan rather than an API key, I want to use that plan where it is allowed, and to be told plainly where it is not.
- As a new owner, I want a bad credential caught while I am still on the step that took it, not at my first message.
- As a new owner, I want to know what else the platform does before I leave the wizard, without being asked to set any of it up now.

## When It Runs

The wizard starts automatically when a user creates their **first** organization, immediately after the [organization creation flow](console.md#context-switcher) succeeds and the Console switches to the new context.

It does not start automatically on subsequent organizations. A cluster admin creating ten organizations is not onboarding ten times, and the second one teaches nothing the first did not. On those, and on any organization where the wizard was abandoned, it is offered from the [Overview](console.md#overview) instead: while the organization has neither an agent nor a sandbox, the Overview replaces its counter grid with a prompt to run setup. Once either exists, the counters return and the prompt is gone.

The wizard is dismissible at every step. Dismissing leaves whatever it has already created — see [Commit Per Step](#commit-per-step) — and returns the user to the Console.

## Shape

One question, then three or four steps, then a finish overlay.

```
What to try first? ─┬─ agent    → Environment → LLM → Files tool → Agent   → Finish
                    └─ sandbox  → Environment → LLM →              Sandbox → Finish
```

Steps are presented as a short numbered flow, not a uniform progress bar. Every remaining step costs about the same, which is what makes a stepper honest here — and what made the earlier design's tunnel and device-enrollment steps impossible to place in one.

## Step 0 — What to Try First

```
What do you want to try first?

  Chat with an agent
  An AI teammate that works on its own and reports back in a conversation.

  Work in a sandbox
  The same setup an agent runs in, with you at the keyboard: Claude Code or
  Codex, and a terminal in your browser.
```

The two options are written as outcomes rather than as the platform's nouns. A person four minutes into an account cannot sort themselves into a taxonomy they have not been taught, so the question asks what they want to happen and lets the nouns follow as labels.

*First*, not *instead* — the screen says the other is a few clicks away afterward. The answer selects what the wizard finishes with; it does not characterize the organization, and both paths leave an organization that can do either.

The agent option is listed first because agents are the product's own story; a flow that leads with sandboxes teaches that this is a hosted-development product. The sandbox option carries equal visual weight regardless.

**Why the fork exists at all.** Its payload is narrow: whether a **subscription** may be offered for Claude. Everything else that differs between the paths follows from that one answer, plus the fact that the sandbox path has no agent to attach anything to. See [Vendor terms](../environments/environments.md#vendor-terms).

The question must precede the LLM step, since that is what it changes. It is deliberately **not** folded into the environment step: an environment serves both workload kinds, and asking "what will run here" on it would teach something false about the model.

## Step 1 — Environment

Two facts and no more than one decision.

| Field | Presentation |
|---|---|
| **Runtime** | A choice: **Claude Code** or **Codex**. Resolves to an `agent_runtime` [image](../images/images.md) and its newest tag |
| **Workspace** | Stated, not chosen — the name and description of the single `workspace` image available, with one line saying what a workspace is: *the container work happens in — tools, runtimes, your code* |

A picker holding one option is not a choice; it is a step that claims to matter and then gives the user nothing to do. Where the catalog offers one workspace image, the wizard shows it as a fact. The teaching lives in the label and the description, not in the act of selecting. If the catalog later offers several, this becomes a real picker with the newest tag preselected — the wizard follows the [version picker](console.md#images) rules like every other surface.

The description is written for both paths: *work* happens in the workspace, not *your agent's work*, because on the sandbox path there is no agent.

**Creates:** an [environment](../environments/environments.md) named `default`, with the chosen runtime image, the workspace image, and one **persistent `/workspace` volume**.

The volume is the wizard's own choice, not a platform default — [environments declare no volumes by default](../environments/environments.md#volumes), deliberately. But a starter environment is curated, and both things it can produce are worse without one: an agent whose state does not survive a restart reads as broken, and a sandbox comes back empty after its idle stop. The finish overlay names the volume among what was created.

## Step 2 — LLM

The step asks how the organization pays, never which `llm_mode` to use. People know their billing relationship with a vendor; nobody arrives knowing what native mode is. The mode is derived from the answer and never shown.

### What is offered

| Path | Claude | Codex |
|---|---|---|
| **Agent** | API key only | Subscription or API key |
| **Sandbox** | Subscription or API key | Subscription or API key |

Where both are offered on the sandbox path, subscription is listed first: it is the cheaper thing to try, and it removes the flow's largest barrier — pasting a vendor API key on day one is a materially bigger ask than pasting a token for a plan already being paid for.

Where a subscription is not offered, the reason is given in one line rather than omitted:

> Claude subscriptions can't be used for autonomous agents under Anthropic's terms. Agents need an API key.

The path is not offered with a warning attached. The wizard is the platform's own voice recommending a route, and the thing this route builds is an autonomous agent. The platform does not block the configuration — see [Vendor terms](../environments/environments.md#vendor-terms) for why — so attaching a subscription for that purpose stays available in the [Console](console.md#llm-providers-and-models), where an operator chooses it deliberately.

### API key

The vendor list is filtered by the protocol the chosen runtime speaks — `anthropic_messages` for Claude Code, `responses` for Codex — so every entry shown can actually serve the CLI the previous step selected.

| Vendor | Endpoint | Protocol | Auth |
|---|---|---|---|
| **Anthropic** | Fixed | `anthropic_messages` | `x_api_key` |
| **OpenAI** | Fixed | `responses` | `bearer` |
| **Microsoft** | Template with one blank — the customer's resource name | `responses` | `bearer` |
| **Custom** | Free text | Selected | Selected |

**A vendor appears here only if choosing it produces a working model on the first try.** That is the admission rule, and it is stricter than "the platform could be configured to reach it": a preset that needs a protocol the [LLM Proxy](../../architecture/llm-proxy.md#protocols) does not speak, or a request shape it does not produce, turns a thirty-second step into a support conversation. Anything that fails the rule belongs behind **Custom**, which is also where the OpenAI-compatible gateways (litellm, OpenRouter, vLLM) live.

**Creates:** an [LLM Provider](../../architecture/providers.md#llm-provider), and one [Model](../../architecture/providers.md#model) naming the vendor's current recommended model for that CLI. The environment stays in `platform` mode.

**The step does not advance until the credential is verified.** The wizard runs the same `"Hello, world"` call as the [model test dialog](console.md#llm-providers-and-models) and reports the result inline. A wizard that accepts a bad key and fails at the first message is worse than no wizard: the failure surfaces two screens away from its cause, in an app the user has never seen before.

### Subscription

A single token field, plus the account identifier where the vendor requires one.

**Creates:** a [Secret](../../architecture/providers.md#secret) holding the token, a [Subscription](../../architecture/providers.md#subscription) naming the vendor and that secret, and an [attachment](../../architecture/providers.md#subscription-attachment) binding it to the environment. The environment is set to `native` mode.

Changing the mode here is legal precisely because of where this step sits: `llm_mode` is frozen only on an environment [some agent references](../environments/environments.md#constraints), and no agent exists yet on either path.

**A subscription-backed environment is created `private`, not `internal`.** An `internal` environment can be run in by any organization member, and a shell there reaches everything it carries — so one person's plan would silently become an organization-wide credential the moment a second member joined. `private` keeps it with its creator, who can grant `user` to whoever should share it. With one member the difference is invisible, which is the right time to get it right.

**There is no verification gate on this branch.** The test call resolves a Model, and native mode has none, so a bad subscription token is not caught here the way a bad API key is. The finish overlay sets that expectation for this branch rather than implying the same guarantee.

## Step 3 — Files Tool

**Agent path only.** Skipped on the sandbox path, where it has nothing to do: [files-mcp](../../architecture/files-mcp.md) resolves `agyn://file/<id>` references that only exist in a thread, and a sandbox has no thread.

One toggle, on by default, framed by what it produces rather than by what it is:

> **Let your agent read files you send it.** Attach a file in a conversation and your agent can open it.

The step exists to teach that tools are added rather than assumed, and files-mcp earns the slot because its payoff lands exactly where the wizard ends. **Add your own** links to the Console's MCP form for anyone who wants more now.

One line prevents the obvious misreading: the agent CLIs already carry file and shell tools for the workspace, so this step is not what makes the agent functional.

**Creates:** an MCP server on the **environment**, not on the agent — which is both what the model prefers for tooling common to a runtime, and what is possible, since the agent does not exist yet. The consequence is a sidecar in sandboxes that run this environment, where it does nothing; it is cheap, and the alternative reorders the flow so that creating the agent is no longer its climax.

## Step 4 — The Thing Itself

### Agent path

| Field | Presentation |
|---|---|
| **Name** | Free text, prefilled with something concrete |
| **What it does** | Two or three starter options that write a real behavioral configuration |
| **Model** | Preselected — the one step 2 created. Absent in `native` mode |
| **Environment** | Preselected — the one step 1 created. Shown, not chosen |
| **Availability** | `internal`, set without asking |

The starter options are the load-bearing field. An agent with an empty behavioral configuration answers its first question with nothing in particular, and the first conversation is the whole payoff of this path — a blank system prompt spends it.

In `native` mode there is no platform Model, so the picker is absent and the agent takes the CLI's own default rather than pinning a vendor model name. One less decision, and the right default.

Skills, ENVs, egress rules, and roles are not first-run material. They are named on the finish overlay and edited in the [agent detail](console.md#agents) page.

### Sandbox path

Creates a [sandbox](../sandboxes/sandboxes.md) on the environment and **starts it**. The finish overlay's value is that one click puts the user in a terminal; landing them on a card with a Start button and a wait is a weak ending to a two-minute flow.

The overlay names the idle timeout, since a sandbox that stops on its own is a surprise exactly once.

## Finish

The Console dims. Two things sit on it.

**Centre — what happened, and what else is here.**

A congratulation, then one line naming what was built in the platform's own nouns:

> You have an environment, a model, and an agent.

Three nouns give the reader a mental model of the platform's shape that no amount of prose during the steps would have produced. The sandbox path's line reads *an environment and a running sandbox*.

Then four capabilities, written as things the agent or sandbox does rather than as section names — a bold lead phrase for scanning, one short line under it, and the section's name as a quiet label so it is recognizable in the sidebar later.

Agent path:

> **Give it credentials it can't leak.** Environment variables can come from a secret — the agent uses the value, nobody can read it from a shell. *Secrets & ENVs*
>
> **Let it reach your own systems.** An internal database, a private Git server, a staging API — without exposing any of it to the internet. *Private Resources*
>
> **Let it call APIs without holding the token.** The platform injects the credential on the way out, and blocks destinations you don't want reached. *Egress Rules*
>
> **Add capabilities to the whole organization.** Message your agents from Telegram, let them set reminders and follow up with you later. *Apps*

Sandbox path replaces the last with the conversion move, and keeps the other three:

> **Let an agent work here on its own.** The same setup can run an agent that picks up work from a conversation. *Agents*

Ordered by how soon a real team reaches them. **None of the four is a link.** The moment they are clickable they become competing calls to action on a screen that has exactly one, and the reader does not need to go anywhere — they need to recognize these names in the sidebar next week.

Where the sandbox path used a **Claude** subscription, the conversion line says what it costs: that environment is subscription-backed, so an agent there is not available, and creating one means an API key and a second environment. Better stated here than discovered when an agent will not start. On Codex there is no such wrinkle and the line stays plain.

**Top left — the one call to action.**

The [product switcher](console.md#top-bar) in its real position, **already open**, with the destination highlighted: **Chat** on the agent path, **Sandboxes** on the sandbox path. It is the only lit, interactive thing on a dimmed screen, so the eye reads the centre and then goes to the only place it can go.

Putting the call to action there rather than on a centre button is deliberate. A centre button gets the user into Chat and teaches them nothing about getting back; the switcher is where every app on this platform is reached from, every day after this one, and the wizard's last act is the best chance to teach it. Opening it rather than merely pointing at it keeps the cost at one click — the coach-mark version asks the user to find the right item inside a control they have never used.

Clicking through lands on **a conversation with the new agent**, or **the running sandbox** — not the app's home. Arriving in an empty app and having to work out how to start a conversation with an agent created ninety seconds ago is a poor last impression.

**There is a way out that is not the call to action** — clicking the dimmed area, or a quiet *Explore the Console*. Some people want to look at what they just built, and if the only exit is the destination, the celebration is a trap.

The overlay is shown once. Dismissed is gone.

## Commit Per Step

Each step commits complete, valid resources before the next one opens. A user who closes the tab mid-flow has an environment and a provider — not a half-written draft, and not nothing.

Failures are handled on the step that caused them: the error is shown inline, the step stays open, and the action can be retried. A step never advances on a failed write, and never leaves a partial resource behind.

Because every step commits, abandoning is safe and re-entry is cheap: the Overview's prompt starts a fresh run, and the steps whose resources already exist are the user's to reuse or replace in the Console.

## Constraints

- The wizard creates resources through the same APIs every other surface uses. It has no privileged path, no resource type of its own, and nothing it creates is marked as wizard-owned.
- It runs automatically only on a user's first organization. Everywhere else it is entered from the Overview prompt.
- It never asks the user to install software, enroll a device, or run anything outside the browser.
- Every step is in the browser and completes in seconds. A step that could block on an external party does not belong in the flow.
- `llm_mode` is set during step 2 and is frozen afterward by the [environment constraint](../environments/environments.md#constraints) once an agent references it.
- The API-key branch does not advance on an unverified credential. The subscription branch has no equivalent check and does not claim one.
- The finish overlay is informational. Its capability list carries no links and no actions.
- Copy never says where a workload runs. The platform ships both hosted and as a [local bundle](../../architecture/operations/local-bundle.md), so *cloud* and *remote* are false in one of the two deployments — the flow says what a thing is and where it is reached from instead.

## Related

- [Console](console.md) — layout, entry flows, and every section the wizard's resources land in
- [Flavors and Environments](../environments/environments.md) — what an environment contains, LLM modes, subscriptions
- [Images](../images/images.md) — the catalog the runtime and workspace images come from
- [Sandboxes](../sandboxes/sandboxes.md) and [Sandboxes App](../sandboxes/sandboxes-app.md)
- [Providers, Models, and Secrets](../../architecture/providers.md)
- [LLM Proxy](../../architecture/llm-proxy.md) — protocols and native-mode interception
- [files-mcp](../../architecture/files-mcp.md)
