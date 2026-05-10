# Notifications Self-Subscription Sentinel

## Target

- [Notifications — Self-Subscription Sentinel](../architecture/notifications.md#self-subscription-sentinel)
- [Notifications — Authorization](../architecture/notifications.md#authorization)
- [Authorization — Notifications Service](../architecture/authz.md#notifications-service)
- [agynd — Platform Connection](../architecture/agynd-cli.md#1-platform-connection)
- [Agent — Communication Protocol](../architecture/agent/overview.md#communication-protocol)
- [agyn-cli — Wait Behavior](../architecture/agyn-cli.md#wait-behavior)

## Delta

The Notifications service authorizes `thread_participant:{id}` subscriptions by identity equality (`id == caller.identity_id`). Subscribing therefore requires the caller to embed its own `identity_id` in the room name. Clients running inside agent containers do not have that value available: they have `AGENT_ID`, the agent resource UUID, which is a different UUID issued by the [Agents service](../architecture/agents-service.md). The platform `identity_id` — generated when the Agents service registers the agent with the [Identity service](../architecture/identity.md) and used as the OpenZiti identity — is not surfaced inside the pod.

The architecture documents the `:me` self-subscription sentinel (the Notifications service rewrites `thread_participant:me` to `thread_participant:{caller.identity_id}` before authorization), but the Notifications service does not yet implement it. The `agyn` and `agynd` clients also do not yet use it:

- The Notifications service treats `:me` as an opaque id segment. A subscribe to `thread_participant:me` fails the equality check and the stream is closed.
- `agynd` constructs the subscribe room from `AGENT_ID`. Because `AGENT_ID` is the agent resource UUID rather than the platform `identity_id`, every subscribe is rejected and the stream closes immediately. This was introduced when subscribe rooms were scoped (previously the subscribe sent no room and was tolerated by the server).
- The `agyn` CLI's `--wait` paths (`threads create --wait`, `threads send --wait`, `threads read --wait`) subscribe via the same `AGENT_ID`-derived room and fail with `notification stream closed`.
- Subscribe failures surface to the user as a bare `notification stream closed` message. The underlying authorization rejection is not propagated.

## Acceptance Signal

- The Notifications service rewrites `:me` to the caller's `identity_id` on Subscribe before authorization. `Subscribe(rooms=["thread_participant:me"])` succeeds for any authenticated identity; the existing `id == caller.identity_id` check passes against the rewritten value. Other identity-scoped rooms inherit the same rewrite if added later; non-identity-scoped rooms (`workload:`, `agent:`, `trace:`) ignore the sentinel.
- `agynd` subscribes to `thread_participant:me`. The `identity_id` argument derived from `AGENT_ID` is removed from the notification subscription call chain. `agynd` continues to use `AGENT_ID` for Agents-service calls (`GetAgent`, `ListSkills`, `ListMCPs`, `ListInitScripts`) where the agent resource UUID is the correct value.
- The `agyn` CLI's `--wait` paths subscribe to `thread_participant:me`. The CLI does not read `AGENT_ID` for the purpose of building the subscribe room and works identically inside an agent pod (Ziti identity) and on a developer machine (token identity).
- Subscribe rejections surface to the caller as the underlying error (`PermissionDenied`, `Unauthenticated`) rather than a bare stream-closed message. `agyn`'s `--wait` exit message names the cause.

## Notes

The bug that triggered this change was an `agyn threads create --wait` call inside an agent container failing with `Error: notification stream closed`. Root cause: a recent CLI commit started passing `Rooms: ["thread_participant:" + AGENT_ID]` on Subscribe, where `AGENT_ID` is the agent resource UUID. The Notifications service authorizes by `id == caller.identity_id`, and those two UUIDs are distinct (see [`ResolveAgentIdentity`](../architecture/agents-service.md#internal-api), which exists precisely to translate between them).

A `Whoami` RPC was considered as an alternative — the client would resolve its `identity_id` once and build the room name itself. Rejected for now because the only consumer of "what is my `identity_id`" today is the self-subscription path; the round-trip cost is not worth it for one call site, and the sentinel approach composes with the existing `rooms` array without adding a new RPC. If other callers need their `identity_id` later, `Whoami` can be added independently.

A boolean `include_caller_thread_participant_room` field on `SubscribeRequest` was also considered and rejected. It does not compose with other rooms in the same subscribe, requires a proto change, and would need a parallel field for every future identity-scoped room.

`AGENT_ID` semantics are unchanged — it remains the agent resource UUID, used for Agents-service calls. The change is to stop misusing it as a stand-in for the platform `identity_id` in notification rooms.
