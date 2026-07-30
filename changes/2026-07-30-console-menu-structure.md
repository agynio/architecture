# Console — Menu Structure

**Date:** 2026-07-30

## Target

- [Product: Console](../product/console/console.md)
- [Product: Private Networks](../product/private-networks/private-networks.md)
- [Architecture: Console](../architecture/console.md)

## Rationale

The Console sidebar had grown to 20 organization-context items in 5 always-expanded groups, grouped by owning service rather than by what the group governs. Three problems followed:

- The **Agents** group held 8 of the 20 items, only one of which was an agent. It had absorbed storage, networking, runtime images, interactive compute, capacity, and the app catalog — everything that fit no other group. A group that can absorb an arbitrary new section is not a category.
- Three of five group headers repeated one of their own section names (`Agents › Agents`, `Models › Models`, `Secrets › Secrets`), which makes the header useless for scanning.
- **Private Resources** had no navigation entry, no route, and no detail page. Resources — including their access grants, the security-critical surface — lived as cards inside a tab of a network detail page, contradicting the product spec, which already said "resource detail page" twice.

The revised structure groups by what the group governs, follows the platform's own model, and adds the grouping rules that keep it from re-accumulating a leftovers bucket.

## Delta

### Sidebar structure

Organization context becomes 7 groups, 22 sections, no ungrouped items:

| Group | Sections |
|---|---|
| Organization | Overview, Members, Groups |
| Agents & Apps | Agents, Apps |
| Runtime | Environments, Volumes, Runners |
| Networking | Private Networks, Private Resources, Egress Rules |
| LLM | Providers, Models |
| Credentials | Secrets, Secret Providers, Image Pull Secrets |
| Operations | Threads, Instances, Workloads, Sandboxes, Provisioned Storage, Usage |

Not implemented in `agynio/console-app` (verified at `157d7f0`, all navigation defined in `src/layout/AppLayout.tsx`):

- Group membership and ordering per the table above. Today: `Organization` (Overview, Members, Groups), `Agents` (Agents, Volumes, Egress Rules, Private Networks, Environments, Sandboxes, Runners, Apps), `Models`, `Secrets`, `Activity` (Workloads, Storage, Threads, Usage).
- **Collapsible groups** with per-group collapsed state persisted in local storage. Today all groups are permanently expanded; 22 sections plus 7 headers does not fit a laptop viewport.
- **Unique icon per section.** Five collisions today: `BoxesIcon` (Models, org Apps, cluster Apps), `KeyIcon` (Secret Providers, Image Pull Secrets, API Tokens), `ShieldIcon` (Groups, Secrets), `HardDriveIcon` (Volumes, Storage), `ServerIcon` (both Runners).
- Renames: `Dashboard` → `Overview` (cluster context), `Cluster Runners` → `Runners`, cluster `Apps` → `App Catalog`, `Storage` → `Provisioned Storage`, `LLM Providers` → `Providers` (the `LLM` group header now carries the qualifier).

### Private Resources extraction

- New organization-scoped resource list at `organizations/:id/private-resources`, listing resources across all of the organization's networks, with network as a column and filter. Org scope matches the `(organization_id, intercept_host, port)` uniqueness constraint the model already enforces.
- New resource detail route at `organizations/:id/private-resources/:resourceId`, owning the resource's fields, the **Copy connection string** affordance, and the access-grant list. Resources have no route at all today.
- Network detail drops its `Tunnels` / `Resources` tabs — tunnels become the only thing a network contains directly.
- `src/pages/OrganizationPrivateNetworksPage.tsx` (1254 lines) splits; the grant principal picker (`usePrincipalOptions`, `GrantDialog`) moves to the resource detail surface.

### Instances

Agent instances have **no Console surface today**. Outside generated code, the only references in `console-app` are read-only `Instance ID` fields on `WorkloadDetailPage.tsx` and `VolumeDetailPage.tsx`. This is a new page, not a navigation move:

- Instance list with handle, class, state, pause reason, inbox, last activity; filters on agent, state, and unacked inbox.
- Instance detail with linked threads, workloads (`ListWorkloadsByAgentInstance`), and storage (`ListVolumesByAgentInstance`).
- Pause and Resume actions.

Instance state (`active` / `paused` / `terminated`) and `pause_reason` (`idle_ttl_exceeded`, `start_failures_exhausted`, `volume_lost`, `runner_deprovisioned`, `manual`) are currently unobservable in the Console — a paused instance and its cause cannot be seen at all.

### Gateway

- [Agents service](../architecture/agents-service.md) already defines `ListInstances` (server-side sort/filter/pagination on `agent_id`, `state_in`, `has_unacked`), `GetInstance`, `PauseInstance`, `ResumeInstance`. [Gateway](../architecture/gateway.md) describes `AgentsGateway` only as "All CRUD methods for agents and sub-resources" — the instance methods need to be confirmed as exposed, and listed explicitly if they are not.
- `RunnersGateway` already exposes `ListWorkloadsByAgentInstance` and `ListVolumesByAgentInstance`, so the instance-scoped queries Instance Detail needs exist.

### Route canonicalization

`src/App.tsx` registers two working URLs for each Operations page — `activity/threads` and `threads`, `activity/usage` and `usage` — while the sidebar links to `activity/workloads`, `activity/storage`, `threads`, and `usage`. Groups no longer map to path prefixes. Collapse to one flat canonical path per section and redirect the superseded paths.

### Known gaps, not introduced here

[Gateway — Exposed Services](../architecture/gateway.md#exposed-services) and the [Console Gateway API Surface](../architecture/console.md#gateway-api-surface) table have no rows for the Networks, Groups, Egress Rules, Apps, Environments, or Sandboxes gateway services, though all six back Console sections. Left unchanged rather than invented; needs its own pass.

## Acceptance Signal

- Organization sidebar shows the 7 groups above, in that order, with every section in a group and no ungrouped items.
- No group header repeats one of its own section names; no icon appears twice within a context.
- Groups collapse and expand, and each group's state survives a reload.
- Private Resources is reachable from the sidebar, lists resources across every network in the organization, and each resource has its own URL that can be shared and bookmarked.
- A resource's access grants are managed on the resource's own detail page; adding and revoking a grant works from there.
- Network detail shows tunnels with no tab bar.
- Instances is reachable from the sidebar; a paused instance shows its pause reason in readable form; Pause and Resume work; instance detail links to the instance's threads, workloads, and storage.
- Each Operations section resolves at exactly one canonical URL, with the superseded paths redirecting.
