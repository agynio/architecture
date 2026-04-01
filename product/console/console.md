# Console

## Purpose

The Console is the platform's management interface. It is the single place where administrators configure the platform — organizations, users, agents, LLM providers, models, secrets, runners, and apps — and monitor active workloads and resource usage.

Two entry points lead to the Console:

- **Self-hosted setup.** An administrator who deployed the cluster opens the Console to complete cluster configuration — register runners, create organizations, invite users, set up LLM providers.
- **Cloud onboarding.** A new user signs up, lands in the Console, creates an organization, invites teammates, and configures agents before starting conversations.

## Roles

The Console is accessible to two roles. Organization members do not have Console access.

| Role | Scope | What they see |
|------|-------|---------------|
| **Cluster admin** | Platform-wide | Everything: cluster-level configuration (users, cluster-scoped runners, cluster-scoped apps) + all organization-level sections |
| **Organization owner** | Per-organization | Organization-level sections only: agents, LLM providers, models, secret providers, secrets, org-scoped runners, members, monitoring |

A user can be an organization owner in one organization and have no Console access in another (if they are a regular member there). The Console resolves the user's role per organization and displays only the organizations where they are an owner.

## Entry Flows

### Self-Hosted Bootstrap

1. Administrator deploys the cluster. Bootstrap Terraform provisions the OIDC configuration, a synthetic admin identity with an API token, and uses that token to create the real admin user (with their OIDC subject) via the platform API.
2. Administrator opens `console.agyn.dev` and authenticates via OIDC. The platform resolves the existing user record — the administrator has `cluster:global admin`.
3. The Console displays the cluster administration view. The administrator registers runners, creates organizations, configures LLM providers, and invites users.

### Cloud Onboarding

1. New user signs up and authenticates via OIDC. The platform auto-provisions the user record on first login.
2. User opens the Console. They have no organizations yet — the Console displays the organization creation flow.
3. User creates an organization, configures LLM providers and models, creates agents, and invites teammates.

## Navigation

### Layout

- **Top bar** — organization switcher (for users with access to multiple organizations), user profile, links to other platform applications (Chat, Tracing).
- **Sidebar** — navigation within the current scope. Sections change based on role.
- **Main area** — content for the selected section. List-detail pattern: resource list → resource detail/edit.

### Cluster Admin Sections

Visible only to users with `cluster:global admin`. These sections are hidden entirely for non-cluster-admins.

| Section | Description |
|---------|-------------|
| **Users** | List, create, and manage platform users. Assign cluster admin role. Assign users to organizations with a role (owner or member) |
| **Cluster Runners** | List, register, and manage cluster-scoped runners. View enrollment status, labels, and connected workloads |
| **Cluster Apps** | List, register, and manage cluster-scoped apps. View enrollment status |

### Organization Sections

Visible to organization owners within their organization. Also visible to cluster admins for any organization.

| Section | Description |
|---------|-------------|
| **Overview** | Organization summary — member count, agent count, active workloads, resource usage |
| **Members** | List organization members and their roles. Invite users (assign role). Remove members. Change roles |
| **Agents** | List, create, and manage agents and all sub-resources (MCPs, skills, hooks, ENVs, init scripts, volume attachments). View agent configuration, referenced model, runner labels |
| **Volumes** | List, create, and manage volumes. View attachments |
| **LLM Providers** | List, create, and manage LLM provider connections. Configure endpoint and authentication |
| **Models** | List, create, and manage models. Configure provider reference and remote model name |
| **Secret Providers** | List, create, and manage secret provider connections |
| **Secrets** | List, create, and manage secret references |
| **Runners** | List and manage org-scoped runners. View enrollment status, labels, and connected workloads |
| **Apps** | Manage app installations within the organization. App installation model is an [open question](../../open-questions.md) |
| **Monitoring** | Active workloads, storage, and usage metrics (see [Monitoring](#monitoring)) |

## Resource Management

### Agents

The agent management section is the most complex resource view. An agent has multiple sub-resource types, each managed as a nested list within the agent detail view.

**Agent list** — table of agents in the organization. Columns: name, role, model (resolved name), status (has active workloads or not), created date.

**Agent detail** — full agent configuration with inline sub-resource management:

- **Configuration** — name, role, model (selector from organization's models), image, init image, compute resources, runner labels, agent behavioral configuration (JSON editor).
- **MCPs** — list of MCP server definitions. Each MCP shows its image, command, compute resources, and its own ENVs and init scripts.
- **Skills** — list of prompt fragments. Each skill has a name and body (text editor).
- **Hooks** — list of event-driven functions. Each hook shows its event trigger, entrypoint, image, compute resources, and its own ENVs and init scripts.
- **ENVs** — environment variables attached directly to the agent (as opposed to those on MCPs or hooks). Each ENV has a name and either a plain value or a secret reference (selector from organization's secrets).
- **Init Scripts** — shell scripts attached directly to the agent.
- **Volume Attachments** — volumes mounted on the agent container. Select from organization's volumes.

### LLM Providers and Models

**LLM Provider detail** — endpoint URL, auth method, token (masked, reveal on click). List of models using this provider.

**Model detail** — internal name, provider (selector), remote model name. Shows which agents reference this model.

### Secret Providers and Secrets

**Secret Provider detail** — type (Vault), connection configuration (address, token masked).

**Secret detail** — provider reference, remote name. Shows which ENVs reference this secret.

### Runners

**Runner detail** — name, scope (cluster or org), enrollment status (`pending`, `enrolled`, `offline`), labels (key-value editor), service token (shown once at registration, then masked). List of active workloads on this runner.

### Users (Cluster Admin)

**User list** — all platform users. Columns: name, email, organizations (with roles), cluster admin status.

**User detail** — profile (name, nickname, photo URL, OIDC subject), role assignments. Cluster admin can:
- Grant or revoke `cluster:global admin`.
- Add the user to an organization with a specific role (owner or member).
- Remove the user from an organization.
- Change the user's role within an organization.

**Create user** — OIDC subject (required), profile fields (name, nickname, photo URL — optional), role assignments (cluster admin flag, organization memberships with roles).

### Members (Organization Owner)

Organization owners manage membership within their organization. They cannot create users — they select from existing platform users (or invite by OIDC subject, which triggers user creation if the user doesn't exist yet).

**Member list** — users in the organization. Columns: name, role (owner or member).

**Add member** — search existing users or enter OIDC subject for a new user. Assign role.

**Change role / Remove** — inline actions on the member list.

## Monitoring

The monitoring section provides operational visibility into running workloads and resource consumption.

### Active Workloads

Real-time view of running agent workloads.

| Column | Description |
|--------|-------------|
| Agent | Agent name (link to agent detail) |
| Runner | Runner name (link to runner detail) |
| Thread | Thread ID (link to conversation in Chat) |
| Status | Workload status |
| Containers | Container count and states |
| Started | Workload start time |
| Duration | Time since start |

Workload detail shows container-level information: container name, image, state, resource usage.

### Storage

View of persistent volumes in the organization.

| Column | Description |
|--------|-------------|
| Volume | Volume name (link to volume detail) |
| Size | Provisioned capacity |
| Attached to | Current attachment target (agent, MCP, or hook) |
| Status | Bound, pending, available |

### Usage Metrics

Aggregated usage data for the organization.

| Metric | Description |
|--------|-------------|
| Token consumption | Total LLM tokens consumed, broken down by model and agent |
| Compute hours | Total container runtime, broken down by agent |
| Active workloads | Current count of running workloads |
| Storage used | Total persistent volume capacity in use |

Time range selector for historical views (24h, 7d, 30d).

Cluster admins see platform-wide aggregates in addition to per-organization views.

## Constraints

- The Console does not manage conversations or messages — that is the Chat application's domain.
- The Console does not display tracing data — that is the Tracing application's domain.
- The Console does not replace Terraform or CLI for infrastructure-as-code workflows. It provides a UI for the same operations.
