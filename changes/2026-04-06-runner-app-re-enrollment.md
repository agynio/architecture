# Runner and App Re-Enrollment: Idempotent Identity Cleanup

## Target

- [OpenZiti Integration — Runner Provisioning](../architecture/openziti.md#runner-provisioning)
- [OpenZiti Integration — App Identity Lifecycle](../architecture/openziti.md#app-identity-lifecycle)
- [OpenZiti Integration — Enrollment Invariants](../architecture/openziti.md#enrollment-invariants)
- [Runners — Enrollment](../architecture/runners.md#enrollment)

## Delta

Re-enrollment succeeds on the OpenZiti Controller (previous identity is deleted by `externalId` before creating a new one) but fails at the database layer. The previous managed identity record is not cleaned up before inserting the new one, violating the unique constraint on `identity_id`. Both runner and app enrollment are affected.

## Acceptance Signal

- A runner can be restarted and successfully re-enroll without manual intervention.
- App re-enrollment behaves identically.
- E2E test: register runner → enroll → restart → re-enroll succeeds.
