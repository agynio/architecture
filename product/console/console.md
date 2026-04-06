# Console

## Purpose

The Console is the platform's management interface for organizations, users, agents, LLM providers, models, secrets, runners, apps, and operational monitoring.

## User Stories

### Cluster Admin

- As a cluster admin, I want to create and manage platform users so I can control who has access to the platform.
- As a cluster admin, I want to grant or revoke the cluster admin role so I can delegate platform administration.
- As a cluster admin, I want to register and manage cluster-scoped runners so agents in any organization can use shared compute.
- As a cluster admin, I want to view and manage all organizations so I can oversee platform usage.

### Organization Owner

- As an organization owner, I want to see a summary of my organization so I can understand its current state at a glance.
- As an organization owner, I want to create and configure agents with models, tools, hooks, and environment variables so agents can perform work.
- As an organization owner, I want to manage LLM providers and models so agents have access to language models.
- As an organization owner, I want to manage secret providers and secrets so agents can access sensitive credentials.
- As an organization owner, I want to register org-scoped runners so I can control where my organization's agents execute.
- As an organization owner, I want to install apps into my organization and configure them so agents can use app capabilities.
- As an organization owner, I want to publish apps from my organization so other organizations can install them.
- As an organization owner, I want to invite users to my organization and assign roles so teammates can collaborate.
- As an organization owner, I want to monitor active agent workloads so I can see what is running and troubleshoot issues.

### Any Authenticated User

- As a user, I want to create an organization so I can start using the platform.
- As a user, I want to view and accept pending organization invites so I can join teams.
- As a user, I want to manage my API tokens so I can set up programmatic access.
- As a user, I want to view and edit my profile so my identity is accurate across the platform.

## Roles

| Role | Scope | What they see |
|------|-------|---------------|
| **Cluster admin** | Platform-wide | Cluster administration (users, organizations, cluster-scoped runners) + all organization-level sections |
| **Organization owner** | Per-organization | Organization-level sections: agents, volumes, LLM providers, models, secret providers, secrets, runners, apps, members, monitoring |

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

- **Top bar** — context switcher (left), user menu (right). Always visible.
- **Sidebar** — navigation within the current context. Sections change based on the selected context.
- **Main area** — list-detail pattern: resource list → resource detail/edit.

### Top Bar

#### Context Switcher

The context switcher is the primary navigation control. It determines what the sidebar displays.

The switcher dropdown lists:

- **Organizations** — all organizations where the user is an owner, ordered alphabetically. Selecting an organization loads its sections in the sidebar.
- **Cluster Administration** — visible only to cluster admins. Selecting it loads cluster-level sections in the sidebar.
- **Create Organization** — action at the bottom of the dropdown. Opens the organization creation flow. On success, the new organization appears in the list and the Console switches to it.

The currently selected context is displayed in the top bar. On load, the Console selects the last-used context (persisted in local storage). If no previous context exists, the Console selects the first organization alphabetically, or Cluster Administration if the user has no organizations but is a cluster admin.

#### User Menu

The user menu is accessible from the top bar (right side). It shows the user's name and avatar.

The dropdown contains:

| Item | Description |
|------|-------------|
| **Profile** | View and edit the user's own profile (name, nickname, photo URL). Read-only fields: OIDC subject |
| **API Tokens** | Create, list, and revoke API tokens for programmatic access. Token value is shown once at creation and cannot be retrieved again |
| **Pending Invites** | List of pending organization invites with accept/decline actions. Badge on the user menu shows the count of pending invites. Accepting an invite adds the organization to the context switcher and switches to it |
| **Logout** | Ends the session |

### Sidebar

The sidebar lists navigation sections for the currently selected context. Selecting a section loads its content in the main area.

#### Organization Sections

Visible when an organization is selected in the context switcher. Available to organization owners and cluster admins.

| Section | Description |
|---------|-------------|
| **Overview** | Organization summary (see [Overview](#overview)) |
| **Agents** | Agent CRUD and sub-resource management |
| **Volumes** | Volume CRUD |
| **LLM Providers** | LLM provider CRUD |
| **Models** | Model CRUD |
| **Secret Providers** | Secret provider CRUD |
| **Secrets** | Secret CRUD |
| **Runners** | Org-scoped runner management |
| **Apps** | App installations and published apps (see [Apps](#apps)) |
| **Members** | Member and invite management |
| **Monitoring** | Active workloads (see [Monitoring](#monitoring)) |

#### Cluster Administration Sections

Visible when Cluster Administration is selected in the context switcher. Only cluster admins see this option.

| Section | Description |
|---------|-------------|
| **Users** | Platform user CRUD |
| **Organizations** | View, update, and delete organizations across the platform |
| **Runners** | Cluster-scoped runner management |

### Empty States

**No organizations and not a cluster admin** — the main area displays an onboarding message: "You have no organizations. Create one to get started or wait for an invite." The placeholder includes a "Create Organization" button that opens the organization creation flow (same as the context switcher action). The pending invites section in the user menu is accessible so the user can accept invitations.

**No organizations but is cluster admin** — the Console auto-selects Cluster Administration. The context switcher shows only "Cluster Administration" and "Create Organization".

**Organization with no resources** — each section shows an empty state with a prompt to create the first resource (e.g., "No agents yet. Create your first agent.").

## Resource Lists

All resource lists in the Console share common behaviors.

**Sorting** — each list has a default sort order (typically by creation time, newest first). Column headers are clickable to change sort field and direction.

**Search** — a search input filters the list by name or other identifying fields (e.g., email for users, slug for apps). Search is client-side for small collections; server-side with debounced input for large collections.

**Pagination** — cursor-based. The list loads a page at a time with "load more" at the bottom.

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

### Agents

**Agent list** — table of agents in the organization. Columns: name, model (resolved name), status (has active workloads or not), created date. Default sort: creation time, newest first.

**Agent detail** — full agent configuration with inline sub-resource management:

- **Configuration** — name, model (selector from organization's models), image, init image, compute resources, runner labels, agent behavioral configuration (JSON editor).
- **MCPs** — list of MCP server definitions. Each MCP shows its image, command, compute resources, and its own ENVs and init scripts.
- **Skills** — list of prompt fragments. Each skill has a name and body (text editor).
- **Hooks** — list of event-driven functions. Each hook shows its event trigger, entrypoint, image, compute resources, and its own ENVs and init scripts.
- **ENVs** — environment variables attached directly to the agent. Each ENV has a name and either a plain value or a secret reference (selector from organization's secrets).
- **Init Scripts** — shell scripts attached directly to the agent.
- **Volume Attachments** — volumes mounted on the agent container. Select from organization's volumes.

### Volumes

**Volume list** — table of volumes in the organization. Columns: name, size, attachment target (agent, MCP, or hook — or "unattached"), created date. Default sort: creation time, newest first.

**Volume detail** — name, size, mount path. Shows current attachment (if any) with a link to the parent resource.

**Create volume** — name (required), size (required), mount path (required).

### LLM Providers and Models

**LLM Provider list** — table of providers. Columns: name, endpoint URL, model count, created date.

**LLM Provider detail** — endpoint URL, auth method, token (masked, reveal on click). List of models using this provider.

**Model list** — table of models. Columns: internal name, provider name, remote model name, agent count (how many agents reference this model).

**Model detail** — internal name, provider (selector), remote model name. Shows which agents reference this model.

### Secret Providers and Secrets

**Secret Provider list** — table of providers. Columns: name, type, secret count, created date.

**Secret Provider detail** — type (Vault), connection configuration (address, token masked). List of secrets using this provider.

**Secret list** — table of secrets. Columns: name, provider name, created date.

**Secret detail** — provider reference, remote name. Shows which ENVs reference this secret.

### Runners

In the organization context, the runners section shows org-scoped runners. In the Cluster Administration context, it shows cluster-scoped runners.

**Runner list** — table of runners. Columns: name, enrollment status (`pending`, `enrolled`, `offline`), labels (comma-separated summary), workload count, created date.

**Runner detail** — name, scope (cluster or org), enrollment status, labels (key-value editor), service token (shown once at registration, then masked). List of active workloads on this runner.

**Register runner** — name (required), labels (optional key-value pairs). Returns the service token once. The user must copy it before navigating away.

### Apps

The Apps section has two sub-sections, selectable via tabs:

#### Installed Apps

Apps installed into this organization. Each installation grants the app permissions to interact with the organization's resources.

**Installation list** — table of installations. Columns: app name, installation slug, app address (`{org-slug}/{app-slug}`), created date. Default sort: creation time, newest first.

**Installation detail** — installation slug, app name and address, app description, permissions granted, configuration (JSON editor). Actions: update configuration, update slug, uninstall.

**Install app** — search for available apps by name or address. The search returns public apps from any organization and internal apps from the current organization. Selecting an app shows its description and required permissions. The user sets the installation slug (defaults to the app's slug) and provides configuration (JSON). Submitting creates the installation and writes authorization tuples for the app's declared permissions.

#### Published Apps

Apps created and owned by this organization. These apps can be installed by other organizations (if public) or only by this organization (if internal).

**App list** — table of apps owned by this organization. Columns: name, slug, visibility (`public` or `internal`), installation count (across all organizations), created date.

**App detail** — name, slug, description, icon, visibility, declared permissions. List of installations of this app (across all organizations, showing org name and installation slug). Service token (shown once at creation, then masked). Actions: update name/description/icon/visibility, delete (only if no active installations exist).

**Create app** — name (required), slug (required, unique within the organization), description (optional), icon (optional), visibility (`public` or `internal`), permissions (multi-select from the permission vocabulary: `thread:create`, `thread:write`, `participant:add`). Returns the service token once.

### Users (Cluster Admin)

**User list** — all platform users. Columns: name, email, organizations (with roles), cluster admin status. Default sort: creation time, newest first.

**User detail** — profile (name, nickname, photo URL, OIDC subject), cluster role. Cluster admin can grant or revoke `cluster:global admin`.

**Create user** — OIDC subject (required), profile fields (name, nickname, photo URL — optional), cluster role (admin or none).

### Organizations (Cluster Admin)

**Organization list** — all organizations on the platform. Columns: name, member count, agent count, created date. Default sort: creation time, newest first.

**Organization detail** — name, member count, agent count, runner count. Cluster admin can update the organization name or delete the organization.

### Members

Organization owners manage membership within their organization.

**Member list** — members in the organization (active and pending). Columns: name, role (owner or member), status (active or pending), joined date. Default sort: active members first, then pending; within each group by join date.

**Invite member** — search existing platform users by name or email. Assign role (owner or member). Creates a pending membership (invite). The invited user must accept the invite via the Console's user menu before gaining access. Cluster admins can add members directly (active immediately, no invite step).

**Change role** — inline action on the member list. Switches between owner and member. Available to organization owners.

**Remove member** — inline action on the member list. Removes the member (any status). Available to organization owners.

## Monitoring

### Active Workloads

Real-time view of running agent workloads in the organization.

| Column | Description |
|--------|-------------|
| Agent | Agent name (link to agent detail) |
| Runner | Runner name (link to runner detail) |
| Thread | Thread ID |
| Status | Workload status (`starting`, `running`, `stopping`, `stopped`, `failed`) |
| Containers | Container count and states |
| Started | Workload start time |
| Duration | Time since start (live counter for running workloads) |

Default sort: start time, newest first.

Workload detail shows container-level information: container name, image, state, resource usage.

## Real-Time Updates

The Console receives real-time updates via WebSocket for data that changes during a session:

- **Runner enrollment status** — updates when a runner enrolls or goes offline.
- **Active workloads** — new workloads appear, status transitions update in place, completed workloads update without requiring a refresh.
- **Monitoring counters** — the overview page and monitoring section refresh workload counts on each update.

On WebSocket disconnection, the Console reconnects automatically and re-fetches the current view's data.

Resource management views (agents, providers, models, secrets, members) do not require real-time updates — they are loaded on navigation and refreshed on user action.

## Constraints

- Backend communication: ConnectRPC through the Gateway (HTTP/JSON for browser requests, gRPC-Web for streaming).
- The Console is a static SPA with no backend — all data comes from the Gateway API.
- File uploads are not supported in the Console (agent images and configurations are specified as references, not uploaded).
- Session persistence: the selected context (organization or cluster admin) is stored in local storage and restored on reload.
