# Waking Promptly

## Target

- [k8s-microvm-runner — Wake](../architecture/k8s-microvm-runner.md#wake)
- [k8s-microvm-runner — Namespace and RBAC](../architecture/k8s-microvm-runner.md#namespace-and-rbac)
- [agynd — Wake](../architecture/agynd-cli.md#8-wake)

## Delta

A woken agent could take over a minute to answer a message, or not answer until another one came.

Under nested virtualization a guest restored on demand crawls for as long as it keeps touching new pages. The slot supervisor can restore by copy instead, but the runner gave an install no way to ask for it.

`agynd` took any sign of a wake within 20 seconds of the last for a repeat of it. A turn can start, end and hibernate again inside a few seconds, so the wake for the next message can land in that window. It was dropped: nothing read the inbox, and the message sat unread until another one arrived.

The same gap exists without a wake. A running agent whose Notifications stream dropped reopened it without reading the inbox, so a message published in between waited for the next one too.

Desired state:

- How a wake loads guest memory is an install setting, `slot.guestRestoreMode`: `OnDemand` by default, `Copy` under nested virtualization.
- `agynd` tells a wake from the repeats of one by whether a sleep lies between them, not by how close together they come.
- `agynd`'s clock check sees a sleep of a second or more, so a sleep shorter than a keepalive tick is noticed even if both of the runner's signals go missing.
- `agynd` pulls the inbox whenever the Notifications stream reopens.

## Acceptance Signal

- On a node under nested virtualization with `Copy`, a hibernated workload answers a message within seconds of it.
- A message sent as soon as the workload has hibernated again is answered, every time, with no other message needed.
- A message published while a running agent's Notifications stream is down is picked up when the stream is back.

## Notes

Measured on the local bundle VM on 2026-11-05, a Lima VM with nested virtualization, with the test agent's stub model:

| Restore | `agynd` | Reply after the message |
|---|---|---|
| `OnDemand` | 0.25.0 | 33–79 s |
| `Copy` | 0.25.0 | 5–10 s, and the wake after that one lost: no reply until the next message |
| `Copy` | with this change | 1–3 s, on all 7 wakes in a row |

Why on-demand is this slow under nesting is not pinned down. Seen from the VM, the vCPUs were running rather than blocked on faults, the fault handler used about a second of CPU in a minute, and the guest's memory grew by about 50 MB in that minute. The earlier prototype replied 8.7 s after a message restoring on demand, with 4 vCPUs against the 2 of this flavor, which has not been re-measured. `Copy` removes the delay either way.

`OnDemand` stays the default because it is the design for hardware KVM, where it has not been measured yet.
