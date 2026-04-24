# Activity Lists Move to Server-Side Sort and Filter

## Target

- [Console — Resource Lists](../product/console/console.md#resource-lists)
- [Console — Activity](../product/console/console.md#activity) (Workloads, Storage, Threads)
- [Runners — List Query Shape](../architecture/runners.md#list-query-shape)
- [Threads — ListOrganizationThreads request shape](../architecture/threads.md#listorganizationthreads-request-shape)
- [Authorization — Organization Permissions](../architecture/authz.md#organization-permissions)

## Delta

The Activity lists (Workloads, Storage, Threads) can grow past what fits in a single page. Today the Console loads them one page at a time but sorts, filters, and searches over only the loaded subset on the client — so results silently omit items that live on later pages, and column-header sorting reorders only what is visible.

- `ListWorkloads`, `ListVolumes`, and `ListOrganizationThreads` take only coarse filters today (`organization_id`, `runner_id`, `status_in`, `pending_sample`). Spec now defines full sort + filter + pagination envelopes on each endpoint. Filter sets: Workloads — agent, runner, status, started range; Storage — status, runner, attached-to kind; Threads — status, participant, created range. Sort fields per spec.
- List responses denormalize display names so the UI never renders raw IDs. `ListWorkloads` response items include `agent_name` and `runner_name`. `ListVolumes` response items include `volume_name` and `attached_to_name`. `ListOrganizationThreads` resolves participant `@nickname` per participant in-band.
- Console Activity pages sort, filter, and search server-side only. Any change to sort/filter/search resets pagination and refetches from the first page. No client-side cross-page logic.
- Workloads list rows are links to the Workload Detail view.
- Workloads and Storage pages gain filter bars (Agent / Runner / Status / Started for Workloads; Status / Runner / Attached-to for Storage). Threads page gains a filter bar (Status / Participant / Created range).
- Authorization model adds two new organization relations, `can_view_workloads` and `can_view_volumes`, each defined as `owner or admin from cluster`. `can_view_threads` is updated to the same shape so cluster admins can list threads in every organization. `ListWorkloads`, `GetWorkload`, `StreamWorkloadLogs`, and the `workload:{id}` notification room now require `can_view_workloads` (previously `member`). `ListVolumes` and `GetVolume` require `can_view_volumes`. `ListWorkloadsByThread` and `ListVolumesByThread` move to `can_read` on the thread.

## Acceptance Signal

- `ListWorkloads`, `ListVolumes`, and `ListOrganizationThreads` proto requests accept the sort, filter, and pagination fields defined in the respective request-shape sections.
- Response items include the denormalized name fields listed above; the Console renders names, not IDs, on the Workloads, Storage, and Threads pages.
- Console Workloads, Storage, and Threads pages show the filter bars defined in the spec. Changing any filter clears the cursor and triggers a fresh fetch. No client-side filtering or sorting is performed across pages.
- Each Workloads list row navigates to the corresponding Workload Detail view.
- OpenFGA model deploys with `can_view_workloads` and `can_view_volumes` on the organization type, and with the updated `can_view_threads` definition. An identity with `admin` on `cluster:global` passes `Check(can_view_workloads | can_view_volumes | can_view_threads, organization:<any>)` without any per-org tuple write.
- A regular org member (non-owner, non-cluster-admin) receives permission denied on `ListWorkloads`, `GetWorkload`, `StreamWorkloadLogs`, `ListVolumes`, `GetVolume`, and `ListOrganizationThreads`.
