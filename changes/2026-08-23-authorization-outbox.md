# Authorization Outbox

## Target

- [Authorization — Relationship Writes](../architecture/authz.md#relationship-writes)
- [Authorization — Tuple Lifecycle](../architecture/authz.md#tuple-lifecycle)

## Delta

- No service has an `authorization_outbox` table or processor. All nine tuple-writing services (Agents, Apps, Groups, Identity, LLM, Organizations, Runners, Threads, Users) call the Authorization service's `Write` directly from the request path. A process death between the record write and the tuple write leaves the two stores permanently inconsistent; nothing detects it.
- Agents compensates on `CreateEnvironment` by deleting the record when the tuple write returns an error. The crash between the two writes and the failing compensation are uncovered — either leaves a record every check refuses, including for its creator.
- Agents runs `BackfillEnvironmentAuthorization` from `cmd/agents-service/main.go` on every process start: unversioned, unrecorded, one tuple per `Write` call, cost proportional to the environment count on every restart (175 environments ≈ 4 minutes in a local bundle).
- Groups runs a periodic in-process reconciler (`internal/server/reconciler.go`) over unpaged full scans of all groups and memberships, writing and deleting tuples to converge them. Both mechanisms are obsolete under the outbox.

## Acceptance Signal

- Every tuple-writing service enqueues tuple intent in the transaction that mutates the record, delivers in-request after commit, and sweeps stranded entries within seconds.
- No service calls the Authorization service's `Write` from a request path except as outbox delivery.
- Killing a service process between a record commit and tuple delivery leaves no permanently unreachable record: the sweeper delivers the stranded entry on its next pass.
- The Agents boot backfill is deleted. Environments predating the environment authorization type are repaired by a one-shot outbox enqueue recorded in the data-migration ledger — see [Data Migrations](2026-08-23-data-migrations.md).
- The Groups reconciler is deleted.

## Notes

- Rollout order: Agents first (most tuple call sites, and it carries the boot backfill), Groups second (deleting the reconciler proves the replacement), the remaining seven after.
- Purge entries cover resource deletion: role grants exist only as tuples, so delete-all cannot be enumerated at enqueue time.
- The revocation-delay bound (sweep interval, seconds) applies only to entries stranded by a crash; the happy path delivers in-request.
