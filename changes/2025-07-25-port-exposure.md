# Port Exposure — User Access to Agent Ports

**Date:** 2025-07-25

## Target

- [Product: Port Exposure](../product/port-exposure/port-exposure.md)
- [Architecture: Expose Service](../architecture/expose-service.md)
- [Architecture: OpenZiti Integration](../architecture/openziti.md)
- [Architecture: Console](../architecture/console.md)
- [Product: Console](../product/console/console.md)

## Delta

Port exposure does not exist yet. The following are not implemented:

- Expose service (`agynio/expose`)
- Expose service proto definitions in `agynio/api`
- Ziti Management: `CreateExposureResources`, `DeleteExposureResources`, `CreateDeviceIdentity`, `DeleteDeviceIdentity` RPCs
- Gateway: `ExposeGateway` proto service
- `agyn` CLI: `agyn expose add <port>`, `agyn expose remove <port>`, `agyn expose list`
- Console: Devices section in user menu
- Agents Orchestrator: call `RemoveExposuresByWorkload` during workload stop flow
- Ziti sidecar: configuration to host exposed services (forward to localhost)
- OpenZiti static policy: `agents-host-exposed`

## Acceptance Signal

- Agent can expose a port via `agyn expose add <port>` and receive an access URL.
- Agent can un-expose a port via `agyn expose remove <port>`.
- User can enroll a device via Console and access the exposed URL from their browser.
- All OpenZiti resources are cleaned up when the agent workload stops.
- Failed or partial provisioning is eventually cleaned up by the reconciliation loop.

## Notes

- Device enrollment uses the OpenZiti one-time-token (OTT) flow. The user's Ziti tunnel client handles enrollment directly with the OpenZiti Controller.
- Access control for exposed ports is broad (all enrolled devices can access all exposed services). Scoped access control is deferred.
