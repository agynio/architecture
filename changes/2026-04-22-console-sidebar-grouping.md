# Console Sidebar Grouping and Activity Split

## Target

- [Console — Sidebar](../product/console/console.md#sidebar)
- [Console — Activity](../product/console/console.md#activity)

## Delta

The Console sidebar currently renders organization sections as a flat list of 14 items. The Monitoring page combines Workloads, Storage, and Usage on a single screen; Storage and Usage always render empty and the combined layout does not scale to long lists.

Specifically:

- Sidebar organization sections are not grouped. Spec now defines 5 always-expanded groups: **Organization** (Overview, Members), **Agents** (Agents, Volumes, Runners, Apps), **Models** (LLM Providers, Models), **Secrets** (Secret Providers, Secrets, Image Pull Secrets), **Activity** (Workloads, Storage, Threads, Usage).
- Monitoring page combines multiple views. Spec now defines four separate pages under **Activity**: Workloads, Storage, Threads, Usage.
- Storage page (persistent volumes view with PV name, size, used, attached to, status) does not exist.

## Acceptance Signal

- Sidebar renders the 5 groups with section headers and spacing; no collapse/expand interaction.
- Workloads, Storage, Threads, and Usage are separate routes/pages, each reachable from the sidebar.
- Storage page lists persistent volumes across the organization with the columns defined in the spec.
- Old combined Monitoring page is removed.
