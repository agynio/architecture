# Activity Lists Move to Server-Side Sort and Filter

## Target

- [Console — Resource Lists](../product/console/console.md#resource-lists)
- [Console — Activity](../product/console/console.md#activity) (Workloads, Storage, Threads)
- [Console — Real-Time Updates](../product/console/console.md#real-time-updates)
- [Console — Context Switcher](../product/console/console.md#context-switcher) and [Console — Role Resolution](../architecture/console.md#role-resolution)
- [Runners — List Query Shape](../architecture/runners.md#list-query-shape)
- [Threads — ListOrganizationThreads request shape](../architecture/threads.md#listorganizationthreads-request-shape)
- [Authorization — Organization Permissions](../architecture/authz.md#organization-permissions)

## Delta

The Activity lists (Workloads, Storage, Threads) can grow past what fits in a single page. Today the Console loads them one page at a time but sorts, filters, and searches over only the loaded subset on the client — so results silently omit items that live on later pages, and column-header sorting reorders only what is visible.

### API

- `ListWorkloads`, `ListVolumes`, and `ListOrganizationThreads` take only coarse filters today. Spec now defines full sort + filter + pagination envelopes on each endpoint, plus a stable `id` secondary sort so pagination is deterministic. Filter sets: Workloads — agent, runner, status, started range; Storage — status, runner, attached-to kind; Threads — status, participant, created range. Sort fields per spec.
- Volume search: `ListVolumes` additionally accepts a substring search on volume name (case-insensitive).
- List responses denormalize display names so the UI never renders raw IDs.
  - `ListWorkloads` response items include `agent_name` and `runner_name`.
  - `ListVolumes` response items include `volume_name` and an `attachments` list — each attachment has `kind` (`agent` / `mcp` / `hook`), `id`, and `name`. A single provisioned volume can have multiple attachments when the PVC is mounted by multiple containers in a pod.
  - `ListOrganizationThreads` resolves each participant's `@nickname` in-band.

### Console UI

- Workloads, Storage, and Threads sort, filter, and search server-side only. Any change to sort/filter/search resets pagination and refetches from the first page. No client-side cross-page logic.
- Usage remains an aggregation view and is out of scope for the server-side-list rule.
- Workloads list rows link to the Workload Detail view. Workload Detail renders agent and runner names (not IDs).
- Workloads, Storage, and Threads pages gain filter bars.
  - Workloads: Agent, Runner, Status, Started range.
  - Storage: Status, Runner, Attached-to kind. Storage also exposes substring search on volume name.
  - Threads: Status, Participant, Created range.
- Storage: the Volume status enum shown in the table is the canonical set from the Runners service (`provisioning`, `active`, `deprovisioning`, `deleted`, `failed`), replacing the earlier K8s PV phase values in the spec.
- Storage: when a volume has multiple attachments, the list column shows the primary attachment's name with `+N more`; the detail view lists every attachment.
- Real-time updates with active filters: when a `workload.updated` or `volume.updated` event arrives while a filter, sort, or search is active, the Console refetches the current page from the server rather than mutating the local list. With no active filter (default view), updates apply in place as before.
- Context switcher widens for cluster admins: the dropdown lists every organization on the platform for cluster admins (not only ones they own), since their `admin from cluster` relation grants organization read permissions everywhere. Role Resolution switches to `ListOrganizations` for cluster admins and `ListMyMemberships` for non-admin users.

### Authorization

- Two new organization relations: `can_view_workloads` and `can_view_volumes`, each defined as `owner or admin from cluster`.
- `can_view_threads` updated to the same shape so cluster admins can list threads in every organization.
- `ListWorkloads`, `GetWorkload`, `StreamWorkloadLogs`, and the `workload:{id}` notification room now require `can_view_workloads` (previously `member`).
- `ListVolumes` and `GetVolume` require `can_view_volumes` (previously `member`).
- `ListWorkloadsByThread` and `ListVolumesByThread` keep `member`-on-org (unchanged).
- `ListOrganizations` authorization note extended to say cluster admins see every organization.

## Acceptance Signal

- `ListWorkloads`, `ListVolumes`, and `ListOrganizationThreads` proto requests accept the sort, filter, pagination, and (for volumes) search fields defined in the respective request-shape sections. Responses are deterministically ordered for identical inputs.
- Response items include the denormalized name fields listed above; the Console renders names, not IDs, on the Workloads, Storage, Threads, and Workload Detail pages.
- Console Workloads, Storage, and Threads pages show the filter bars defined in the spec. Changing any filter clears the cursor and triggers a fresh fetch. No client-side filtering or sorting is performed across pages.
- Each Workloads list row navigates to the corresponding Workload Detail view.
- Real-time event with an active filter on a list page triggers a refetch of the current page rather than an in-place mutation.
- Context switcher shows every organization on the platform for cluster admins.
- OpenFGA model deploys with `can_view_workloads` and `can_view_volumes` on the organization type, and with the updated `can_view_threads` definition. An identity with `admin` on `cluster:global` passes `Check(can_view_workloads | can_view_volumes | can_view_threads, organization:<any>)` without any per-org tuple write.
- A regular org member (non-owner, non-cluster-admin) receives permission denied on `ListWorkloads`, `GetWorkload`, `StreamWorkloadLogs`, `ListVolumes`, `GetVolume`, and `ListOrganizationThreads`.
