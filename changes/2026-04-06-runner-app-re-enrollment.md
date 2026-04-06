# Runner and App Re-Enrollment: Idempotent Identity Cleanup

## Target

- [OpenZiti Integration — Runner Provisioning](../architecture/openziti.md#runner-provisioning)
- [OpenZiti Integration — App Identity Lifecycle](../architecture/openziti.md#app-identity-lifecycle)
- [Runners — Enrollment](../architecture/runners.md#enrollment)
- [Runners — Registration Flow](../architecture/runners.md#registration-flow)
- [Runners — Deletion](../architecture/runners.md#deletion)

## Delta

**Runner enrollment fails on restart.** `CreateRunnerIdentity` in Ziti Management creates a new OpenZiti identity with `externalId: runnerId` without deleting the previous one. The OpenZiti Controller enforces a unique index on `externalId`, causing a `400 Bad Request` (`duplicate value in unique index on identities store`).

Specific gaps between desired state and current implementation:

1. **Ziti Management `CreateRunnerIdentity`** does not look up or delete a previous managed identity for the same runner before creating a new one.
2. **Ziti Management `CreateRunnerIdentity`** creates the per-runner OpenZiti service (`runner-{runnerId}`) on every enrollment call. The architecture specifies service creation during registration (`RegisterRunner`), not enrollment. This also fails on re-enrollment (duplicate service name).
3. **Runners Service `RegisterRunner`** does not create the per-runner OpenZiti service. The architecture specifies it should call Ziti Management `CreateService`.
4. **Runners Service `DeleteRunner`** does not clean up the runner's OpenZiti identity or service. The architecture specifies it should call Ziti Management `DeleteRunnerIdentity`.
5. **Ziti Management** is missing the `CreateService`, `DeleteRunnerIdentity`, and `DeleteAppIdentity` RPCs described in the architecture.
6. **`CreateAppIdentity`** has the same re-enrollment gap as runners — no cleanup of previous identity.

## Acceptance Signal

- A runner can be restarted (or upgraded) and successfully re-enroll without manual intervention.
- `CreateRunnerIdentity` deletes the previous identity before creating a new one.
- `CreateRunnerIdentity` does not create the OpenZiti service (service creation happens at registration).
- `RegisterRunner` creates the per-runner OpenZiti service via `CreateService`.
- `DeleteRunner` cleans up the OpenZiti identity and service via `DeleteRunnerIdentity`.
- App enrollment (`CreateAppIdentity`) handles re-enrollment identically.
- E2E test: register runner → enroll → restart → re-enroll succeeds.

## Notes

The architecture docs incorrectly claimed "the previous identity is cleaned up by Ziti Management lease GC." Lease GC only applies to ephemeral service identities (Orchestrator, Gateway pods), not to runner/app managed identities. The architecture has been updated to describe explicit cleanup during re-enrollment.
