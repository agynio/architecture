# Tracing App

## Purpose

The Tracing app is the platform's observability interface. It lets operators and developers inspect agent runs — seeing what an agent did, why it made specific decisions, and where failures occurred.

## Navigation Structure

Three levels, no sidebar:

```
Home → Organization → Run
```

- **Home** — lists the organizations the user has access to.
- **Organization** — lists runs for the selected organization.
- **Run** — shows the [Run Timeline](run-timeline.md) for a single run.

## Breadcrumb

A breadcrumb sits in the top-left corner on every page. Each segment is a dropdown.

```
[User ▾] / [Organization ▾] / [Run name]
```

- The **User** segment is always present. Opening it shows the user menu.
- The **Organization** segment appears on the Organization and Run pages. Opening it lists all organizations the user has access to — selecting one navigates to that organization's runs.
- The **Run** segment appears only on the Run page. It is a static label (run ID or name), not a dropdown.

### User Dropdown

| Item | Description |
|------|-------------|
| **Profile** | User profile |
| **Sign out** | Ends the session |

### Organization Dropdown

Lists all organizations the user has access to. The current organization is highlighted. Selecting a different organization navigates to that organization's runs page.

## Home Page

Shown when the user has no selected organization or navigates to the app root.

Displays a list of organizations the user has access to. Each entry shows the organization name. Clicking an entry navigates to the Organization page for that organization.

If the user has access to only one organization, the app navigates directly to that organization's page on load.

## Organization Page

Lists all runs for the selected organization. Ordered by start time, newest first.

Each run entry shows:

| Field | Description |
|-------|-------------|
| Agent | Agent name |
| Thread | Thread ID |
| Status | `starting`, `running`, `stopping`, `stopped`, `failed` |
| Started | Start timestamp |
| Duration | Elapsed time. Live counter for running workloads |

Clicking a run navigates to the Run page.

**Real-time updates** — new runs appear at the top; status and duration update in place while runs are active.

**Pagination** — cursor-based. Scrolling to the bottom loads older runs.

## Run Page

See [Run Timeline](run-timeline.md).

## Related architecture

- [Product to architecture map (Tracing)](../../maps/product-to-architecture.md#tracing)
- [Tracing](../../architecture/tracing.md)
- [Runners](../../architecture/runners.md)
