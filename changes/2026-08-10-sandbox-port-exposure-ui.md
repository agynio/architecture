# Port Exposure in the Sandboxes App

## Target

- [Sandboxes App — Ports](../product/sandboxes/sandboxes-app.md#ports)
- [Sandboxes App — Ports](../architecture/sandboxes-app.md#ports)
- [Sandboxes App — Gateway API Surface](../architecture/sandboxes-app.md#gateway-api-surface)
- [Expose Service — Authorization](../architecture/expose-service.md#authorization)
- [Authorization — Expose Service](../architecture/authz.md#expose-service)
- [Sandboxes — What's Inside](../product/sandboxes/sandboxes.md#whats-inside)
- [Port Exposure — Flow](../product/port-exposure/port-exposure.md#flow)

Builds on [2026-08-10 — Entity-Named Port Exposure](2026-08-10-entity-named-port-exposure.md): the address a port is served at, and the `owner_kind` this change authorizes against, come from there.

## Delta

**A sandbox's ports can only be managed from inside the sandbox.** The [Sandboxes app](../product/sandboxes/sandboxes-app.md) states it as a constraint — "port exposure is not managed here, `agyn expose` runs inside the sandbox, from its shell" — so an engineer who started a dev server has to drop into the terminal, run a command, and read the URL out of its output. There is nowhere in the app that answers "what is this sandbox serving, and at what address?"

The sandbox page gains a **Ports** strip: one row per exposed port with its address as a link and a copy action, a control to expose a port, and a remove action per row. It sits between the header and the terminal.

**The blocker is authorization, not UI.** Every existing path through `AddExposure` assumes the caller *is* the workload — the Gateway injects `x-workload-id` from a verified OpenZiti connection, and the Expose service checks identity equality against it. A browser is not the workload. The only other route is the explicit-`workload_id` path, which is cluster-admin-only. As written, a sandbox owner cannot expose a port from the app, and cannot un-expose one either: `RemoveExposure` falls back to organization *owner*.

So the explicit path gains a sandbox clause:

| Operation | Before | After |
|---|---|---|
| `AddExposure` (explicit `workload_id`) | `admin` on `cluster:global` | `can_connect` on `sandbox:<workload.owner_id>` for a sandbox-owned workload; cluster admin otherwise |
| `RemoveExposure` | workload, or `owner` on the organization | workload; `can_connect` on the sandbox when sandbox-owned; or `owner` on the organization |
| `ListExposures` | workload, or `member` on the organization | unchanged |

**`can_connect` is the right relation because it grants nothing.** A shell in the sandbox can already run `agyn expose add`; doing it from the app is the identical capability with fewer keystrokes. A stricter gate would refuse a button to someone the terminal in the next tab obeys. It introduces no new relation and no new tuple — [`sandbox`](../architecture/authz.md#sandbox) already defines `can_connect`, and it is what the [Terminal Proxy](../architecture/terminal-proxy.md#authorization) checks at ticket issuance.

`ListExposures` needs nothing: a collaborator is necessarily an organization `member`, which the existing check already admits.

The clause is scoped to `owner_kind = sandbox`. An agent-instance workload has no comparable relation and no surface asking for one, so it stays cluster-admin-only.

### What the strip states rather than assumes

Three facts are invisible from the page and each has bitten someone:

- **A link only resolves on an enrolled device.** The address is on the private overlay; a browser with no running Ziti tunnel gets nothing. The strip says so once, beside the first address, and links to Devices — the moment someone has a link to click is the moment enrolling makes sense.
- **Anyone on the platform network can reach an exposed port.** Addresses are readable and therefore guessable, and exposure carries no per-user access control. Stated where ports are exposed, not in a help page.
- **Ports do not survive a stop; their addresses do.** An idle-out clears the list, and re-exposing after a start returns the identical address because it is derived from the sandbox rather than the exposure. A bookmark keeps working — it just needs the port exposed again on the other end.

## Acceptance Signal

- An engineer runs a server on port 3000 in a sandbox shell, opens the sandbox page, exposes 3000 from the strip, clicks the address, and reaches the server from an enrolled device.
- The same port exposed with `agyn expose add 3000` in the terminal appears in the strip on the next refetch, with the same address.
- Exposing a port that is already exposed leaves one row and returns the same address.
- Removing a row un-exposes the port and the address stops resolving.
- A collaborator on a shared sandbox can expose and remove ports; a member of the organization who is neither owner nor collaborator is refused both.
- A cluster admin can still pass `workload_id` for an agent-instance workload; a sandbox owner passing one for someone else's sandbox is refused.
- A stopped sandbox shows the empty state and offers no expose action; starting it and exposing 3000 returns the address the sandbox had before it stopped.
- The strip names the enrolled-device requirement and links to Devices, and states that any identity on the platform network can reach an exposed port.

## Notes

- **No live updates.** The Expose service publishes no [Notifications](../architecture/notifications.md) events, so there is no room to subscribe to and a port exposed from inside the shell cannot surface as it happens. The app refetches on mount, after its own actions, and on the transition into `running`. An exposure notification room is the fix and is deliberately not specified here — the list is short, the staleness is visible, and a refetch is cheap.
- **No Console surface.** An organization owner's fleet view has no ports column and gains none. It is a different question (what is exposed across the organization) with a different authorization story, and nobody has asked it yet.
- **Nothing about the Expose service's data model changes.** No new RPC, no new field, no new OpenZiti object — only who may call two methods that already exist.
- **Re-exposing on restart is not automatic.** Restoring the ports a sandbox had before it idled out is a reasonable convenience and is left for later; doing it silently would start listeners the engineer did not ask for at addresses they may have shared.
