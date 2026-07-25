# Service Identity Loss: Fail Fast Instead of Silent Degradation

## Target

- [OpenZiti Integration — Identity Loss](../architecture/openziti.md#identity-loss)
- [OpenZiti Integration — Identity Lease GC](../architecture/openziti.md#identity-lease-gc)
- [OpenZiti Integration — Service Identity Self-Enrollment](../architecture/openziti.md#service-identity-self-enrollment)

## Delta

Services that self-enroll OpenZiti identities (Gateway, Tracing, LLM Proxy, Egress Gateway, Runners Service, Orchestrator) do not react to identity loss. When lease renewals are interrupted long enough for GC to delete the identity of a still-running pod (observed on the local bundle VM after host suspend), the pod keeps running with a dead overlay bind:

- Lease extension failures are not surfaced — no error logs, no health signal.
- The pod never terminates, so it never re-enrolls; its hosted services have no terminators until someone manually restarts the deployment.
- Consumers see the failure first (`dial failed: service has no terminators`), with no corresponding signal on the hosting side.

Desired state: a definitive identity-loss signal (`NOT_FOUND` on `ExtendIdentityLease`, or SDK authentication rejected for a previously valid session) causes the pod to log the loss at error level and terminate; the restart path re-enrolls a fresh identity. Transient extension failures are retried with backoff and logged.

## Acceptance Signal

- Deleting a service identity out from under a running pod causes that pod to terminate with an error log naming the lost identity ID, restart, re-enroll, and re-establish its terminators without manual intervention.
- Lease extension failures (transient or definitive) are visible in the service's logs.
- E2E test: enroll → delete identity via Ziti Management → pod restarts → dialing the hosted service succeeds again.

## Notes

Observed 2026-07-22 on the local bundle: router↔controller heartbeat flaps and missed lease renewals (host suspend) led to GC deleting identities of live pods; `gateway`, `tracing`, and `llm-proxy` lost all terminators for days with no error logs on the hosting side. The Runners Service was the only service that re-enrolled (its own recovery loop), cycling `svc-runners-*` identities hourly.
