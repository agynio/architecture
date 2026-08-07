# Console

## Purpose

The Console is the platform's management interface for organizations, users, agents, apps, runtime and networking configuration, LLM providers and models, credentials, and operations.

## User Stories

### Cluster Admin

- As a cluster admin, I want to create and manage platform users so I can control who has access to the platform.
- As a cluster admin, I want to grant or revoke the cluster admin role so I can delegate platform administration.
- As a cluster admin, I want to register and manage cluster-scoped runners so agents in any organization can use shared compute.
- As a cluster admin, I want to view and manage all organizations so I can oversee platform usage.

### Organization Owner

- As an organization owner, I want to see a summary of my organization so I can understand its current state at a glance.
- As an organization owner, I want to create and configure agents with models, tools, and environment variables so agents can perform work.
- As an organization owner, I want to set each agent's availability and assign per-agent roles to specific identities so I can control who configures and chats with which agents.
- As an organization owner, I want to manage LLM providers and models so agents have access to language models.
- As an organization owner, I want to manage secret providers and secrets so agents can access sensitive credentials.
- As an organization owner, I want to define egress rules and attach them to agents so I can control which destinations agents reach and inject credentials without exposing secrets to the agent container.
- As an organization owner, I want to register org-scoped runners so I can control where my organization's agents execute.
- As an organization owner, I want to install apps into my organization and configure them so agents can use app capabilities.
- As an organization owner, I want to see the status and audit log reported by an installed app so I can diagnose configuration problems and understand what the app is doing.
- As an organization owner, I want to publish apps from my organization so other organizations can install them.
- As an organization owner, I want to invite users to my organization and assign roles so teammates can collaborate.
- As an organization owner, I want to monitor active agent workloads so I can see what is running and troubleshoot issues.
- As an organization owner, I want to list and read all threads in my organization so I can inspect agent conversations and troubleshoot issues.
- As an organization owner, I want to see my agent instances with their state and pause reason so I can tell why an agent stopped picking up work and resume it.
- As an organization owner, I want to see every private resource in my organization in one list, regardless of which network it sits behind, so I can audit what is reachable and who can reach it.

### Any Authenticated User

- As a user, I want to create an organization so I can start using the platform.
- As a user, I want to view and accept pending organization invites so I can join teams.
- As a user, I want to manage my API tokens so I can set up programmatic access.
- As a user, I want to approve a CLI sign-in from the browser so my terminal gets a credential without me copying a token by hand.
- As a user, I want to view and edit my profile so my identity is accurate across the platform.

## Roles

| Role | Scope | What they see |
|------|-------|---------------|
| **Cluster admin** | Platform-wide | Cluster administration (users, organizations, cluster-scoped runners, app catalog) + all organization-level sections |
| **Organization owner** | Per-organization | Organization-level sections: the organization and its groups; agents and apps; runtime (images, environments, runners); networking (private networks, private resources, egress rules); LLM providers and models; credentials; operations |

Organization members do not have Console access. A user can be an organization owner in one organization and a regular member (no Console access) in another. The Console displays only the organizations where the user is an owner.

## Entry Flows

### Self-Hosted Bootstrap

1. Administrator deploys the cluster. Bootstrap Terraform provisions the OIDC configuration, a synthetic admin identity with an API token, and uses that token to create the real admin user (with their OIDC subject) via the platform API.
2. Administrator opens `console.agyn.dev` and authenticates via OIDC. The platform resolves the user record — the administrator has `cluster:global admin`.
3. The Console auto-selects Cluster Administration context. The administrator registers runners, creates organizations, configures LLM providers, and adds members.

### Cloud Onboarding

1. User signs up and authenticates via OIDC. The platform provisions the user record on first login.
2. User opens the Console. No organizations exist — the Console displays the empty state with a prompt to create an organization.
3. User creates an organization via the context switcher. The Console switches to the new organization context.
4. User configures LLM providers and models, creates agents, and invites teammates.

## Layout

The Console uses a three-region layout:

- **Top bar** — current page title (left), context switcher and user menu (right). Always visible.
- **Sidebar** — navigation within the current context. Sections change based on the selected context.
- **Main area** — page content only. No page-level headers — the page title is in the top bar. List-detail pattern: resource list → resource detail/edit.

### Top Bar

#### Page Title

The top bar displays the title of the currently active section (e.g., "Agents", "Volumes", "Users") on the left. This is the only place the page title appears — the main area does not repeat it.

#### Context Switcher

The context switcher is the primary navigation control. It determines what the sidebar displays.

The switcher dropdown lists:

- **Organizations** — all organizations where the user is an owner, ordered alphabetically. Cluster admins see every organization on the platform in this list (not only ones they own), since their `admin from cluster` relation grants organization read permissions everywhere. Selecting an organization loads its sections in the sidebar.
- **Cluster Administration** — visible only to cluster admins. Selecting it loads cluster-level sections in the sidebar.
- **Create Organization** — action at the bottom of the dropdown. Opens the organization creation flow: display name and [slug](../../architecture/organizations.md#slug) (cluster-wide unique, suggested from the name). On success, the new organization appears in the list and the Console switches to it.

The currently selected context is displayed in the top bar. On load, the Console selects the last-used context (persisted in local storage). If no previous context exists, the Console selects the first organization alphabetically, or Cluster Administration if the user has no organizations but is a cluster admin.

#### User Menu

The user menu is accessible from the top bar (right side). It shows the user's name and avatar.

The dropdown contains:

| Item | Description |
|------|-------------|
| **Profile** | View and edit the user's own profile (name, `username`, nickname, photo URL). `username` is cluster-wide unique and used for invite discovery — see [Users — Username](../../architecture/users.md#username). Read-only fields: OIDC subject |
| **Devices** | Register and manage devices enrolled in the platform network for accessing [exposed agent ports](../port-exposure/port-exposure.md). Each device has a name, enrollment status, and a one-time enrollment JWT. See [Port Exposure — Devices](../port-exposure/port-exposure.md#devices) |
| **API Tokens** | Create, list, and revoke API tokens for programmatic access. Token value is shown once at creation and cannot be retrieved again. Tokens issued by a [CLI sign-in](../cli-login/cli-login.md) appear in the same list, named for the machine that requested them (`CLI on vitalii-mbp`), so a user can tell which entry is their laptop and revoke it |
| **Pending Invites** | List of pending organization invites with accept/decline actions. Badge on the user menu shows the count of pending invites. Accepting an invite adds the organization to the context switcher and switches to it |
| **Logout** | Ends the session |

### Sidebar

The sidebar lists navigation sections for the currently selected context. Selecting a section loads its content in the main area.

#### Grouping Rules

The sidebar groups sections rather than listing them flat. Three rules govern the grouping, and any new section must satisfy all three:

1. **Every section belongs to a group.** There are no ungrouped items — an item that fits no group means the grouping is wrong, not that the item is special. Importance and frequency of use are not grouping criteria; they determine order *within* the sidebar, never membership.
2. **No group is named after one of its own sections.** A `Secrets` group containing a `Secrets` section makes the header useless for scanning, because the reader cannot tell header from link. Where the natural name collides, the group takes the name of the *category* (`Credentials`, `LLM`) or names its members explicitly (`Agents & Apps`).
3. **Each group states what it governs**, and every member is governed by it. A group that can absorb an arbitrary new section is not a category — it is a leftovers bucket, and the sections in it belong somewhere else.

Groups are collapsible, expanded by default, and each group's collapsed state persists in local storage. Every section carries its own icon; no icon is reused across sections within a context.

#### Organization Sections

Visible when an organization is selected in the context switcher. Available to organization owners and cluster admins.

The groups follow the platform's own model: the organization itself, the actors that do work, where work runs, what work may use, and what actually ran.

**Organization** — the organization itself.

| Section | Description |
|---------|-------------|
| Overview | Organization summary (see [Overview](#overview)) |
| Members | Member and invite management (see [Members](#members)) |
| Groups | Membership groups used as grant principals (see [Groups](#groups)) |

**Agents & Apps** — the non-human thread participants. Both are definitions carrying their own platform [identity](../../architecture/identity.md), and both are grantable principals on [private resources](../private-networks/private-networks.md#granting-access).

| Section | Description |
|---------|-------------|
| Agents | Agent class CRUD and sub-resource management (see [Agents](#agents)) |
| Apps | App installations and published apps (see [Apps](#apps)) |

**Runtime** — where work runs.

| Section | Description |
|---------|-------------|
| Images | The organization's image catalog (see [Images](#images)) |
| Environments | Runtime definitions — runner, flavor, images, and the volumes, MCPs, init scripts, and ENVs a workload carries (see [Environments](../environments/environments.md)) |
| Runners | Org-scoped runner management (see [Runners](#runners)) |

Images come first because environments are assembled from them: the group reads as what runs, how it is composed, and where it lands. Storage has no section of its own here — a volume is part of an environment, edited on it.

**Networking** — what work can reach, inbound and outbound. Private Resources and Egress Rules are adjacent deliberately: they are the two sides of one per-destination decision (see [EgressRule interaction](../private-networks/private-networks.md#egressrule-interaction)).

| Section | Description |
|---------|-------------|
| Private Networks | Networks and their tunnels (see [Private Networks](../private-networks/private-networks.md)) |
| Private Resources | Org-wide resource list and access grants (see [Private Resources](#private-resources)) |
| Egress Rules | Egress rule CRUD and agent attachment (see [Egress Rules](#egress-rules)) |

**LLM** — what work reasons with.

| Section | Description |
|---------|-------------|
| Providers | LLM provider CRUD (see [LLM Providers and Models](#llm-providers-and-models)) |
| Models | Model CRUD (see [LLM Providers and Models](#llm-providers-and-models)) |

**Credentials** — what work authenticates with.

| Section | Description |
|---------|-------------|
| Secrets | Secret CRUD (see [Secret Providers and Secrets](#secret-providers-and-secrets)) |
| Secret Providers | Secret provider CRUD (see [Secret Providers and Secrets](#secret-providers-and-secrets)) |

Registry credentials are not here. They are a property of an [image](#images) and are edited on it — a credential that exists only to read one repository is an ingredient of that image, not a standalone thing to attach.

**Operations** — what actually ran. Ordered down the runtime hierarchy — thread → instance → workload — then storage and consumption.

| Section | Description |
|---------|-------------|
| Threads | List and read all threads in the organization (see [Threads](#threads)) |
| Instances | Agent instances with lifecycle state and pause reason (see [Instances](#instances)) |
| Workloads | Agent workloads (see [Workloads](#workloads)) |
| Sandboxes | User-started workloads with shell access (see [Sandboxes](../sandboxes/sandboxes.md)) |
| Provisioned Storage | Persistent volumes provisioned on runners (see [Provisioned Storage](#provisioned-storage)) |
| Usage | Resource consumption metrics — LLM tokens, compute, storage, platform activity (see [Usage](../usage/usage.md)) |

**Volume declarations and Provisioned Storage are different resources**, not two views of one. A declaration is part of an [environment](#environments) (or an MCP server) and is owned by the [Agents service](../../architecture/agents-service.md); Provisioned Storage (Operations) are the disks actually *provisioned on runners*, one per agent instance or sandbox, owned by [Runners](../../architecture/runners.md). They have separate lifecycles and separate APIs, and a single declaration can sit behind many rows here.

#### Cluster Administration Sections

Visible when Cluster Administration is selected in the context switcher. Only cluster admins see this option.

Five sections, ungrouped — the list is short enough that grouping would add headers without adding meaning.

| Section | Description |
|---------|-------------|
| **Overview** | Cluster summary. Named to match the organization context's landing page, which fills the same role |
| **Users** | Platform user CRUD |
| **Organizations** | View, update, and delete organizations across the platform |
| **Runners** | Cluster-scoped runner management. Not "Cluster Runners" — the selected context already establishes the scope |
| **App Catalog** | Platform-wide app catalog. Named distinctly from the organization context's **Apps** section, which manages *installations* — the two are different surfaces and must not share a label or an icon |

### Empty States

**No organizations and not a cluster admin** — the main area displays an onboarding message: "You have no organizations. Create one to get started or wait for an invite." The context switcher's "Create Organization" action is available. The pending invites section in the user menu is accessible so the user can accept invitations.

**No organizations but is cluster admin** — the Console auto-selects Cluster Administration. The context switcher shows only "Cluster Administration" and "Create Organization".

**Organization with no resources** — each section shows an empty state with a prompt to create the first resource (e.g., "No agents yet. Create your first agent.").

## Standalone Routes

Pages reached by link rather than by navigation, and rendered without the three-region layout — no sidebar and no context switcher, because nothing on them is scoped to an organization.

### CLI Login Approval

`/cli-login` is where a user approves a sign-in that the [`agyn` CLI](../cli-login/cli-login.md) requested. The CLI opens it; the user confirms that the code shown matches the one in their terminal, sees which machine asked, and approves or denies. Approving issues the credential directly to the waiting CLI — it is never displayed in the browser.

The page requires a signed-in session like every other Console route, so a user who is not signed in authenticates first and returns to it. For a user who has never signed in, that is also account creation, which makes `agyn auth` a valid first contact with the platform.

See [CLI Login — Approval Screen](../cli-login/cli-login.md#approval-screen) for what the page shows and why.

## Resource Lists

All resource lists in the Console share common behaviors.

**Sorting** — each list has a default sort order (typically by creation time, newest first). Column headers are clickable to change sort field and direction.

**Search** — a search input filters the list by name or other identifying fields (e.g., email for users, slug for apps).

**Pagination** — cursor-based. The list loads a page at a time with "load more" at the bottom.

**Where sort, filter, and search run.** Lists that can grow past a single page (any list backed by an endpoint that returns a `page_token`) sort, filter, and search **server-side only**. The Console sends the active sort, filter, and search parameters on every request, and any change to them resets the cursor and refetches from the first page. The client must not sort or filter across pages — doing so produces a view of only the loaded subset, which silently omits matches on later pages.

Small, bounded lists that fetch completely on the first request (e.g., an organization's Agents list scoped to an owner who has tens of agents) may sort and filter client-side for snappiness. The threshold is the endpoint's contract: if `page_token` can be non-empty, behave as a large collection.

Threads, Instances, Workloads, and Provisioned Storage in the [Operations](#operations) section are large collections and follow the server-side rule. Usage is an aggregation view, not a row list, and is out of scope for this rule — see [Usage](../usage/usage.md).

## Destructive Actions

Actions that permanently remove data (delete agent, remove member, uninstall app, delete organization, etc.) require confirmation. The Console shows a confirmation dialog stating what will be deleted and any consequences (e.g., "Removing this member will revoke their access to the organization immediately."). No undo — deletions are permanent.

Non-destructive mutations (update name, change role, toggle settings) apply optimistically — the UI updates immediately and rolls back on server error with an error message.

## Resource Management

### Overview

The overview is the landing page when an organization is selected. It displays summary counters:

| Counter | Description |
|---------|-------------|
| Members | Total active members |
| Agents | Total agents in the organization |
| Active workloads | Currently running agent workloads |
| Runners | Registered runners (org-scoped + cluster-scoped available to this org) |

Each counter links to the corresponding section.

### Members

Organization owners manage membership within their organization.

**Member list** — members in the organization (active and pending). Columns: name, role (owner or member), status (active or pending), joined date. Default sort: active members first, then pending; within each group by join date.

**Invite member** — search existing platform users by `username` (prefix match via the [Users — SearchUsers](../../architecture/users.md#user-directory) endpoint). Pick a result, assign a role (owner or member). Creates a pending membership (invite). The invited user must accept the invite via the Console's user menu before gaining access. Cluster admins can add members directly (active immediately, no invite step). Inviting users who do not yet exist on the platform is not supported — the invitee must sign in once before they can be discovered.

**Change role** — inline action on the member list. Switches between owner and member. Available to organization owners.

**Remove member** — inline action on the member list. Removes the member (any status). Available to organization owners.

### Groups

Named sets of identities — users, agents, and apps — used as a single grant principal. Membership changes propagate automatically to everything the group is granted, so a group is the way to avoid re-granting per identity. See [Groups service](../../architecture/groups-service.md).

**Group list** — table of groups in the organization. Columns: name, member count, created date. Default sort: creation time, newest first.

**Group detail** — name, description, and the member list (identity name, identity type — `user`, `agent`, or `app`). Actions: add member (search the organization's identities), remove member, rename, delete.

Groups are also reachable as an inline "Create group" affordance in the [private resource access picker](../private-networks/private-networks.md#granting-access) — same data, two entry points.

### Agents

**Agent list** — table of agents in the organization. Columns: name, model (resolved name), availability (`internal` or `private`), status (has active workloads or not), created date. Default sort: creation time, newest first.

**Agent detail** — full agent configuration with inline sub-resource management:

- **Configuration** — name, model (selector from organization's models), environment (selector; required — only environments with an agent runtime image are offered), idle timeout, availability (selector: `internal` or `private` — the create form prefills `internal`), agent behavioral configuration (JSON editor). The API has no default for availability — the create form always submits an explicit value. Images and compute come from the environment and are shown read-only with a link to it.
- **Roles** — list of identities with a role on this agent. Columns: identity (resolved name), role (`owner` / `maintainer` / `participant`). Actions: Add role (search the organization's members by name or `@nickname`, pick a role), Change role (inline), Remove role. The agent's creator is automatically granted `owner` at creation. Role assignment is restricted to identities that are members of the agent's organization — the search returns only org members. Organization owners hold owner-level capabilities on every agent and do not need an explicit role.
- **MCPs** — MCP server definitions belonging to this agent, shown alongside the ones its [environment](#environments) contributes (read-only here, with a link). Each MCP shows its image (name and tag, from the [image catalog](#images)), command, compute resources, shared volume names, and its own ENVs, init scripts, and volumes. Each has a Manage menu with: Edit, Environment Variables, Init Scripts, Volumes, Delete. An agent-level MCP whose name matches an environment-level one is marked as overriding it.
- **Skills** — list of prompt fragments. Each skill has a name and body (text editor).
- **ENVs** — environment variables attached directly to the agent. Each ENV has a name and either a plain value or a secret reference (selector from organization's secrets).
- **Init Scripts** — shell scripts attached directly to the agent, running after the environment's.
- **Storage** — read-only. Names the environment's volumes and links to it; agents declare no volumes of their own.
- **Egress Rules** — egress rules attached to the agent, controlling its outbound HTTP/HTTPS (deny destinations, inject credentials). A dropdown lists the organization's egress rules (excluding already-attached ones) with an inline Attach; attached rules are listed with their domain pattern and effect summary, each with a Detach button. Same attachment, viewable and editable from either side — see [Egress Rules](#egress-rules).

### Images

The organization's [image catalog](../images/images.md) — the images environments and MCPs are built from. See [Images](../images/images.md) for the product behavior.

**Image list** — table of images available to the organization: its own, plus every `public` image on the platform. Columns: name, type (`workspace`, `agent runtime`, or `mcp`), owning organization, visibility, latest version, created date. Images owned by another organization are marked and are read-only. Default sort: creation time, newest first.

**Image detail** — name, type, repository, visibility, description, credential (username; the password is write-only), and the discovered version list. Versions show tag, pushed date, and description; a stale image shows when discovery last succeeded. Actions: update name/description/credential/visibility/tag filter, refresh versions, delete. Repository and type are shown but cannot be changed.

**Register image** — name (required), type (required), repository (required), visibility (required), description, tag filter, and optionally a username and password. The repository is read before the record is stored, so a wrong address or credential fails here rather than at workload start.

**Version picker** — used wherever an image is selected (environment create/edit, MCP create/edit). Filters by the type the slot requires, lists semver tags newest-first with the newest preselected, hides other tags behind **show all tags**, and accepts a typed tag for anyone who knows exactly what they want. Each row shows its pushed date and description.

**Delete image** — requires confirmation, and warns that environments and MCPs referencing it will become unschedulable. It is not blocked by references; see [Images Service — Deletion](../../architecture/images-service.md#deletion).

### Environments

Runtime definitions owned by the [Agents service](../../architecture/agents-service.md) — see [Flavors and Environments](../environments/environments.md) for the product behavior. Any organization member can create one; what they can see and change on someone else's depends on their role.

**Environment list** — table of environments in the organization. Columns: name, runner, flavor, workspace image (name and tag), agent runtime image, availability (`internal` or `private`), created date. An environment whose flavor, storage class, or image no longer resolves is flagged **unschedulable** with the unresolved name. Default sort: creation time, newest first.

**Environment detail** — configuration plus inline sub-resource management. Members without `can_read_config` see the header only; the sections below are hidden rather than empty.

- **Configuration** — name, runner (selector), flavor (free text with a warning when the name is not currently reported), workspace image and tag, agent runtime image and tag (both via the [version picker](#images)), availability. Changing the runner warns that existing provisioned storage pins running owners to the old one.
- **Volumes** — the mounts every workload in this environment carries. Columns: name, mount path, persistent, size, storage class, TTL. Add/edit/delete inline. Deleting one warns that the disks provisioned from it will be deprovisioned for every agent instance and sandbox. **When the list is empty the section states it plainly** — workloads here keep nothing across a restart — because that consequence is invisible otherwise.
- **MCPs** — MCP servers that run in every workload of this environment. Each shows image and tag, command, compute resources, its shared volume names, and its own ENVs, init scripts, and volumes. Manage menu: Edit, Environment Variables, Init Scripts, Volumes, Delete. The shared-volume picker offers this environment's volume names.
- **Init Scripts** — scripts run in the main container before the agent CLI (or the sandbox shell). Ordered; the environment's run before any agent's.
- **ENVs** — environment variables injected into the main container of every workload. Plain value or secret reference.
- **Egress Rules** — rules attached to the environment, applying to every workload running it. Same attach/detach affordance as on an agent.
- **Roles** — identities with a role on this environment (`owner` / `maintainer` / `user`). Same add/change/remove affordances as [agent roles](#agents). The creator is `owner`. The section explains what `user` grants: starting a sandbox here, and pointing an agent here — both of which reach the secrets and volumes above.

**Create environment** — name, runner, flavor, workspace image, optional agent runtime image, availability. Volumes, MCPs, init scripts, and ENVs are added after creation from the detail page.

**Delete environment** — blocked while any agent or sandbox references it; the dialog lists what does.

### Runners

In the organization context, the runners section is split into two lists:

- **Organization runners** — runners scoped to the selected organization. Includes an "Enroll runner" action.
- **Cluster runners** — shared runners available to the organization. Organization owners can see the list but cannot open runner details. Cluster admins can view cluster runner details from the organization context, but edit and delete actions are only available in Cluster Administration.

In the Cluster Administration context, the runners section lists cluster-scoped runners and supports full management actions.

**Runner list** — each list shows runner name (with runner ID), enrollment status (`pending`, `enrolled`, `offline`), labels (comma-separated summary), a scope badge, and a View action.

**Runner detail** — name, runner ID, enrollment status, scope, identity ID, labels, and active workloads on the runner.

**Edit runner** — update runner name and labels. Available for org-scoped runners in organization context and cluster-scoped runners in Cluster Administration.

**Delete runner** — confirmation required. Available for org-scoped runners in organization context and cluster-scoped runners in Cluster Administration.

**Enroll runner** — name (required), labels (optional key-value pairs). Returns the service token once; the user must copy it before leaving the dialog.

### Apps

The Apps section has two sub-sections, selectable via tabs:

#### Installed Apps

Apps installed into this organization. Each installation grants the app permissions to interact with the organization's resources.

**Installation list** — table of installations. Columns: app name, installation slug, app address (`{org-slug}/{app-slug}`), created date. Default sort: creation time, newest first.

**Installation detail** — installation slug, app name and address, app description, permissions granted, configuration (JSON editor). Actions: update configuration, update slug, uninstall.

- **Status** — if the app has reported a status, it is displayed as a markdown-rendered block at the top of the detail view. If no status has been reported, this section is hidden.
- **Audit Log** — a table of events reported by the app. Columns: time, level (badge: info / warning / error), message. Newest first, paginated with "load more". If no entries exist, the section is hidden.

**Install app** — search for available apps by name or address. The search returns public apps from any organization and internal apps from the current organization. Selecting an app shows its description and required permissions. The user sets the installation slug (defaults to the app's slug) and provides configuration (JSON). Submitting creates the installation and writes authorization tuples for the app's declared permissions.

#### Published Apps

Apps created and owned by this organization. These apps can be installed by other organizations (if public) or only by this organization (if internal).

**App list** — table of apps owned by this organization. Columns: name, slug, visibility (`public` or `internal`), installation count (across all organizations), created date.

**App detail** — name, slug, description, icon, visibility, declared permissions. List of installations of this app (across all organizations, showing org name and installation slug). Service token (shown once at creation, then masked). Actions: update name/description/icon/visibility, delete (only if no active installations exist).

**Create app** — name (required), slug (required, unique within the organization), description (optional), icon (optional), visibility (`public` or `internal`), permissions (multi-select from the permission vocabulary: `thread:create`, `thread:write`, `participant:add`). Returns the service token once.

### LLM Providers and Models

**LLM Provider list** — table of providers. Columns: name, endpoint URL, auth method, model count, created date.

**LLM Provider detail** — endpoint URL, auth method, credentials (masked, reveal on click), and the list of models using this provider. The credentials section is driven by the selected auth method:

- **Bearer** — single `Token` input. Sent as `Authorization: Bearer <token>`.
- **x-api-key** — single `Token` input. Sent as `x-api-key: <token>`.
- **Custom headers** — editable list of header rows (key + value). Each row's value is masked with reveal-on-click. Each header is sent verbatim on every forwarded request. Use this for providers that need non-`Bearer` auth schemes or require additional routing/tenancy headers alongside the credential.

**Create / Edit LLM Provider** — name (required), endpoint URL (required), protocol (selector: `OpenAI Responses` or `Anthropic Messages`), auth method (selector: `Bearer`, `x-api-key`, `Custom headers`). Switching auth method swaps the credential editor between the single-token input and the headers list — values from the previous method are discarded on save. Saving with `Custom headers` requires at least one header row; the form rejects reserved header names (`Host`, `Content-Length`, `Connection`, `Transfer-Encoding`).

**Model list** — table of models. Columns: internal name, provider name, remote model name, agent count (how many agents reference this model). Each row has a **Test** action that opens the model test dialog.

**Model detail** — internal name, provider (selector), remote model name. Shows which agents reference this model.

**Model test dialog** — sends a predefined message (`"Hello, world"`) to the model via the LLM Proxy and displays the result inline:

- **Pending** — a loading indicator while the request is in flight.
- **Success** — the model's response text is displayed.
- **Failure** — a clear error message is shown (e.g., invalid credentials, provider unreachable, model not found).

### Secret Providers and Secrets

**Secret Provider list** — table of providers. Columns: name, type, secret count, created date.

**Secret Provider detail** — type (Vault), connection configuration (address, token masked). List of secrets using this provider.

**Secret list** — table of secrets. Columns: name, provider name, created date.

**Secret detail** — provider reference, remote name. Shows which ENVs reference this secret.

### Private Networks

Networks and the tunnels that reach them. A [Network](../private-networks/private-networks.md) has no settings beyond a name and description — it exists as the HA boundary and the OpenZiti binding unit for the resources behind it. Tunnels belong to exactly one network, and a network may have several for HA.

**Network list** — table of networks in the organization. Columns: name, description, tunnel count, resource count, reachability (derived — **reachable** if at least one tunnel is online, **degraded** otherwise). Default sort: creation time, newest first.

**Network detail** — name and description (editable inline), followed by the network's **tunnels**. No tabs: tunnels are the only thing a network contains directly. Each tunnel row shows enrollment state (`pending` / `enrolled`), connectivity (`online` / `offline` / `never enrolled`), last seen, provisioning state, and a Revoke action.

**Issue tunnel credential** — creates a credential and reveals the one-time enrollment JWT once, with install snippets per supported tunneler distribution. The JWT cannot be retrieved again.

Resources are **not** managed here. They are org-wide and have their own section — see [Private Resources](#private-resources).

### Private Resources

Addressable endpoints behind a private network, listed at organization scope rather than nested under a network. Org scope is what the data model already enforces: `intercept_host` + port uniqueness is checked across the whole organization, not per network (see [Private Networks — PrivateResource](../../architecture/private-networks.md#privateresource)), so a per-network list would present a namespace that does not exist and could not explain a collision.

**Resource list** — table of resources across all of the organization's networks. Columns: name, network (link to network detail), protocol (`tcp` / `http` / `https`), intercept address (`intercept_host:ports`), target address (`target_host:ports`), grant count, reachability (derived from the owning network's tunnels), provisioning state. Default sort: creation time, newest first.

**Filters** — filter bar with Network (multi-select), Protocol (multi-select), and Provisioning state (multi-select). Search matches on name and `intercept_host`.

**Resource detail** — its own page, not a card in a list. Shows the network it belongs to, protocol, target host and ports, intercept host and ports, provisioning state, and a **Copy connection string** affordance (`prod-postgres.corp:5432`) for pasting into agent configuration or tooling.

**Access grants** — the resource detail page owns the access list. Each grant binds a principal (`agent`, `user`, `app`, or `group`) to the resource; every grant materializes exactly one OpenZiti dial policy. Grants are create-and-delete only — there is no edit. The principal picker offers the organization's agents, users, apps, and groups, with an inline "Create group" affordance. Revocation takes effect immediately, with a propagation window of ≤15 seconds to live workloads; the confirmation dialog states this.

Creating a resource requires selecting a network. The form rejects the reserved intercept zones and warns — without blocking — when the chosen `intercept_host` is a real public hostname, since all agent traffic for that hostname will then route through the tunnel.

See [Private Networks](../private-networks/private-networks.md) for the full model.

### Egress Rules

Rules that control and shape agent outbound HTTP/HTTPS traffic — denying destinations or injecting credentials on the fly. See [Egress Gateway](../egress-gateway/egress-gateway.md) for the model. Org-scoped resources, attached to agents.

**Egress rule list** — table of egress rules in the organization. Columns: name, domain pattern, effect (summary — `allow`, `deny`, and/or injected header names), attached agent count, created date. Default sort: creation time, newest first.

**Egress rule detail** — the matcher (domain pattern, ports, methods, path pattern), the effect (action plus the list of injected headers with credentials masked, reveal-on-click), and the list of agents the rule is attached to (each linking to the agent), with Attach/Detach controls.

**Create / Edit egress rule:**

- **Name** (required).
- **Matcher** — domain pattern (required; single-segment wildcards like `*.github.com`; reserved zones `*.ziti`, `*.svc`, `*.cluster.local`, and the `100.64.0.0/10` synthetic range are rejected inline); ports (defaults to `80, 443`); methods (multi-select — `GET`, `POST`, …; empty means any); path pattern (glob, e.g. `/repos/**`; empty means any). Domain pattern is unique per organization — the form rejects a duplicate with a clear message.
- **Action** — selector: `None` (injection only), `Allow`, or `Deny`.
- **Injected headers** — editable list of header rows. Each row has: header name (e.g., `Authorization`, `X-Api-Key`), a scheme selector (`None`, `Bearer`, `Basic`), and one credential source — either a literal value (masked, reveal-on-click) or a reference to an organization Secret (secret selector). Exactly one credential source per row. When a scheme is set, the row shows a preview of the emitted header (e.g., `Authorization: Bearer ••••`); for `Basic`, helper text notes the credential must be the base64 of `user:pass`. This editor follows the same masked key/value pattern as the LLM provider [Custom headers](#llm-providers-and-models) editor.
- Saving requires at least Action or one injected header — a rule with neither is rejected.

**Delete egress rule** — requires confirmation. Rejected if the rule is attached to any agent; the dialog lists the attached agents and prompts the user to detach first.

**Attaching to agents** — the same attachment is editable from both sides: the rule detail page (select from the organization's agents) and the agent detail page's [Egress Rules](#agents) section (select from the organization's rules). Attaching one rule to many agents — the common case for a shared credential-injection rule — is done from the rule detail page.

### Users (Cluster Admin)

**User list** — all platform users. Columns: name, `username`, email, organizations (with roles), cluster admin status. Default sort: creation time, newest first.

**User detail** — profile (name, `username`, photo URL, OIDC subject), cluster role. Cluster admin can grant or revoke `cluster:global admin`.

**Create user** — OIDC subject (required), profile fields (name, `username`, photo URL — optional; `username` is derived from OIDC claims if omitted), cluster role (admin or none).

### Organizations (Cluster Admin)

**Organization list** — all organizations on the platform. Columns: name, member count, agent count, created date. Default sort: creation time, newest first.

**Organization detail** — name, member count, agent count, runner count. Cluster admin can update the organization name or delete the organization.

### Overview (Cluster Admin)

The landing page when Cluster Administration is selected. Displays platform-wide summary counters — users, organizations, cluster-scoped runners, active workloads — each linking to the corresponding section. Named **Overview**, matching the organization context's landing page, because it fills the same role.

### App Catalog (Cluster Admin)

The platform-wide app catalog: every app defined on the platform, across all organizations. This is a different surface from the organization context's [Apps](#apps) section, which manages *installations* into one organization — the two must not share a label or an icon.

**App list** — all apps on the platform. Columns: name, address (`{org-slug}/{app-slug}`), owning organization, visibility (`public` or `internal`), installation count across all organizations, created date. Default sort: creation time, newest first.

**App detail** — the same detail surface as [Published Apps](#published-apps), read-write for cluster admins regardless of owning organization.

## Operations

The Operations sections are the record of what actually ran. They follow the platform's runtime hierarchy: a [thread](../../architecture/threads.md) references agent [instances](../../architecture/agent-instances.md) as participants; workloads are scheduled on instances; volumes are provisioned for instances. Each section links to the next level down.

### Threads

Read-only view of all threads in the organization. Available to organization owners and cluster admins (`can_view_threads`).

**Thread list** — table of threads in the organization. Columns: ID (truncated), participants (@nicknames, comma-separated — resolved by the server, not the client), message count, status (`active`, `archived`, or `degraded`), created date. Default sort: creation time, newest first. Sortable columns: Created, Updated, Message count, Status.

**Filters** — filter bar with Status (multi-select), Participant (multi-select of identities), and Created range (from / to). All sort, filter, and search are server-side — see [Threads — ListOrganizationThreads request shape](../../architecture/threads.md#listorganizationthreads-request-shape).

**Thread detail** — participant list, paginated message history (newest first), and associated workloads. Each message shows the sender's @nickname, timestamp, and body. File attachments are listed as named download links. The detail view is read-only — owners cannot send messages or modify threads from this view. Associated workloads are loaded through `RunnersGateway.ListWorkloadsByThread(thread_id)` and are associated to the thread by each workload record's `thread_id` field.

A thread is **not** a view of instances. The relationship is many-to-many: a thread has several instance participants, and one instance's inbox spans many threads. To see instances, use [Instances](#instances).

Backing RunnersGateway API:

- `ListWorkloadsByThread(thread_id)` for associated workloads in Thread Detail.

### Instances

Agent [instances](../../architecture/agent-instances.md) in the organization. An instance is a persistent instantiation of an agent class with its own inbox, state, and identity — the level at which workloads are scheduled and state is keyed. This is the only Console surface for instance lifecycle; without it, a paused instance and its reason are invisible.

**Instance list** — table of instances in the organization.

| Column | Description |
|--------|-------------|
| Handle | The instance's `@nick#suffix` handle (see [Agent Instances — Handles](../../architecture/agent-instances.md#handles)). Resolved server-side |
| Agent | Class name (link to agent detail). The Console never displays the raw `agent_id` |
| State | `active`, `paused`, or `terminated` |
| Pause reason | Present only when `paused`. Rendered as readable text (e.g., `idle_ttl_exceeded` → "Idle timeout exceeded"), never the raw enum |
| Inbox | Whether the instance has unacked inbox items, from the `has_unacked` filter field |
| Last activity | `last_activity_at` — set by `agynd` when the workload last did work. Drives idle GC |
| Created | Creation time |

Default sort: last activity, most recent first. Sortable columns: Handle, Agent, State, Last activity, Created.

**Filters** — filter bar with Agent (multi-select of the organization's agent classes), State (multi-select), and Inbox (has unacked items). Filters combine with AND and map to the `agent_id`, `state_in`, and `has_unacked` filters on `ListInstances`. Sort, filter, and search are server-side — see [Agents Service — ListInstances](../../architecture/agents-service.md).

**Instance detail** — handle, class (link to agent detail), state, pause reason, label, `last_activity_at`, timestamps. Three linked sections resolved per instance:

- **Threads** — threads this instance participates in.
- **Workloads** — loaded via `RunnersGateway.ListWorkloadsByAgentInstance(instance_id)`.
- **Storage** — loaded via `RunnersGateway.ListVolumesByAgentInstance(instance_id)`.

**Actions** — Pause (requires confirmation; sets `pause_reason` to `manual`) and Resume (clears `pause_reason`). Both are available to organization owners and cluster admins. Pausing stops workloads from spawning but leaves the inbox accepting writes, so no messages are lost — the confirmation dialog states this.

Terminated instances are excluded from the default view and shown only when `terminated` is selected in the State filter.

### Workloads

Real-time view of running agent workloads in the organization.

| Column | Description |
|--------|-------------|
| Agent | Agent name (link to agent detail). Name comes from the list response — the Console never displays the raw `agent_id` |
| Runner | Runner name (link to runner detail). Name comes from the list response — the Console never displays the raw `runner_id` |
| Thread | Thread ID |
| Status | Workload status (`starting`, `running`, `stopping`, `stopped`, `failed`) |
| Containers | Counts grouped by container state (e.g., `3 running`, `1 waiting (ImagePullBackOff)`). Non-`running` counts include the runtime reason so problems are visible at a glance |
| Started | Workload start time |
| Duration | Time since start (live counter for running workloads) |

Default sort: start time, newest first. Sortable columns: Agent, Runner, Status, Started, Duration.

**Row action** — the row is a link to the [Workload Detail](#workload-detail) view.

**Filters** — a filter bar above the table with the following controls. Filters combine with AND. Any filter change resets pagination and refetches from the server — see [Resource Lists](#resource-lists).

| Filter | Control | Description |
|--------|---------|-------------|
| Agent | Multi-select of the organization's agents (by name) | Show only workloads for the selected agents |
| Runner | Multi-select of runners visible to the organization (org-scoped + cluster-scoped) | Show only workloads on the selected runners |
| Status | Multi-select of workload statuses | Show only workloads in the selected statuses |
| Started | Date range picker (from / to) | Show only workloads created within the range |

Sort, filter, and search are all server-side. See [Runners — ListWorkloads request shape](../../architecture/runners.md#listworkloads-request-shape) for the backing API.

#### Workload Detail

The workload detail view shows:

- Workload metadata — agent name (link to agent detail), runner name (link to runner detail), thread ID, status, started, duration. Names are resolved server-side like in the list; raw `agent_id` / `runner_id` are not displayed.
- A list of containers, ordered init containers first (in declaration order), then the main container, then sidecars. Each row shows name, role, image, a state badge (`running`, `waiting`, `terminated`) with the runtime reason when present (e.g., `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, `Completed`, `Error`), the runtime message as secondary text, exit code (when terminated), restart count, started at, and finished at.
- Storage associated with the workload's owner, loaded via `RunnersGateway.ListVolumesByAgentInstance(workload.owner_id)` for agent workloads and the sandbox-owner equivalent for sandboxes. This section is labeled as owner storage, not mounted volumes: it lists the disks provisioned for that owner and is not evidence of the exact volumes mounted by the workload's current pod.
- A log viewer for the selected container. The viewer loads the last **1000 lines** from the runtime and then follows new output in real time. A container selector switches between containers in the workload (init containers included). There are no tail-length or since-time controls — the fixed window keeps the UI simple.
  - If the container cannot be reached (unknown workload or container, or the Pod has already been deleted), the viewer shows an empty state: "Container no longer exists on the runner. Logs are only available while the container exists."
  - When the runtime closes the stream cleanly (logs exhausted, or the container terminated with the Pod still present), the viewer stays visible with a "Stream ended" marker so the user can scroll through what was captured; it does not clear or auto-close.

Container state refreshes within one reconciliation interval — a crash, image pull failure, or successful completion appears in the detail view and in the list's container summary without a manual refresh, even when the workload-level status is unchanged.

Backing RunnersGateway APIs:

- `GetWorkload(workload_id)` for workload metadata and containers.
- `ListVolumesByAgentInstance(workload.owner_id)` for owner storage shown from the workload context.

### Sandboxes

User-started workloads with shell access — an engineer's manual working copy of an agent's runtime, running an [environment](../environments/environments.md) and carrying its secrets and egress rules. Filed under Operations because a sandbox *is* a workload; it differs from an agent workload only in being started by a user rather than by message traffic.

**This is the organization-wide view, not a working surface.** It lists every sandbox in the organization — backed by `can_list_sandboxes`, which only owners and cluster admins hold — so an owner can see what is running on the organization's compute and terminate a forgotten one. The detail page's terminal is subject to the same check as everywhere else: it attaches only for a sandbox the viewer owns or has been [shared](../sandboxes/sandboxes.md#sharing). Listing every sandbox in the organization does not grant entry to any of them.

Members work with their own sandboxes in the [Sandboxes app](../sandboxes/sandboxes-app.md), which is where creating, connecting, and sharing live — and which members can reach at all, unlike the Console. The two surfaces are deliberately separate: this one is a fleet list to sort and sweep, that one is a launcher for a handful of sandboxes belonging to one person.

See [Sandboxes](../sandboxes/sandboxes.md) for the full specification.

### Provisioned Storage

Real-time view of persistent disks provisioned on runners across the organization. These are the volumes that actually exist on a runner, owned by [Runners](../../architecture/runners.md) — distinct from the declarations on an [environment](#environments) or an MCP server. One declaration appears here once per owner: an environment's `workspace` volume used by four agent instances and two sandboxes is six rows.

| Column | Description |
|--------|-------------|
| Name | Volume name, with the declaring environment or MCP beneath it (link to that resource) |
| Owner | The agent instance or sandbox the disk belongs to, by name |
| Size | Provisioned size |
| Used | Current usage |
| Mounted by | Containers holding the disk (agent or MCP). More than one when an MCP shares an environment volume; the column shows the first name followed by `+N more`, and the detail lists every mount |
| Status | Volume status (`provisioning`, `active`, `deprovisioning`, `deleted`, `failed`) |

Default sort: name. Sortable columns: Name, Size, Status, Created.

**Filters** — filter bar with Status (multi-select), Runner (multi-select), and Owner kind (multi-select of `agent instance`, `sandbox`).

**Search** — substring match on volume name, case-insensitive.

All sort, filter, and search are server-side — see [Runners — ListVolumes request shape](../../architecture/runners.md#listvolumes-request-shape).

### Usage

Resource consumption metrics — LLM tokens, compute, storage, and platform activity. See [Usage](../usage/usage.md) for the full specification.

## Real-Time Updates

The Console receives real-time updates via WebSocket for data that changes during a session:

- **Runner enrollment status** — updates when a runner enrolls or goes offline.
- **Active workloads** — new workloads appear, status transitions update in place, completed workloads update without requiring a refresh.
- **Container state** — changes to per-container state (e.g., `waiting → running`, `running → waiting (CrashLoopBackOff)`, restart count increments) update the Workloads list summary and the open Workload detail, even when the workload-level status is unchanged.
- **Workload counters** — the overview page and Workloads view refresh workload counts on each update.

On WebSocket disconnection, the Console reconnects automatically and re-fetches the current view's data.

**Real-time updates with active filters.** On the Workloads and Provisioned Storage pages, when a `workload.updated` or `volume.updated` event arrives and any filter, sort, or search is active, the Console refetches the current page from the server rather than mutating the local list. This keeps the view consistent with the server's filter predicate — a workload that moves out of the filtered set disappears, a new match appears in its sorted position — without requiring the client to mirror filter logic. With no active filter (default view), the Console applies the update in place as before.

Resource management views (agents, providers, models, secrets, members) do not require real-time updates — they are loaded on navigation and refreshed on user action.

## Constraints

- Backend communication: ConnectRPC through the Gateway (HTTP/JSON for browser requests, gRPC-Web for streaming).
- The Console is a static SPA with no backend — all data comes from the Gateway API.
- File uploads are not supported in the Console (agent images and configurations are specified as references, not uploaded).
- Session persistence: the selected context (organization or cluster admin) is stored in local storage and restored on reload.

## Related architecture

- [Product to architecture map (Console)](../../maps/product-to-architecture.md#console)
- [Console](../../architecture/console.md)
- [Gateway](../../architecture/gateway.md)
- [Organizations](../../architecture/organizations.md)
- [Users](../../architecture/users.md)
- [Agents service](../../architecture/agents-service.md)
- [Agent instances](../../architecture/agent-instances.md)
- [Threads](../../architecture/threads.md)
- [Runners](../../architecture/runners.md)
- [LLM](../../architecture/llm.md)
- [Secrets](../../architecture/secrets.md)
- [Apps](../../architecture/apps.md)
- [Apps service](../../architecture/apps-service.md)
- [EgressRules service](../../architecture/egress-rules-service.md)
- [Egress Gateway](../../architecture/egress-gateway.md)
- [Private Networks](../../architecture/private-networks.md)
- [Networks service](../../architecture/networks-service.md)
- [Groups service](../../architecture/groups-service.md)
