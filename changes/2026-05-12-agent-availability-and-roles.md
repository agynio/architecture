# Agent Availability and Roles

## Target

- [Architecture: Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [Architecture: Agents Service — Availability](../architecture/agents-service.md#availability)
- [Architecture: Agents Service — Roles](../architecture/agents-service.md#roles)
- [Architecture: Agents Service — Agent Role API](../architecture/agents-service.md#agent-role-api)
- [Architecture: Agents Service — Authorization](../architecture/agents-service.md#authorization)
- [Architecture: Agents Service — Tuple Lifecycle](../architecture/agents-service.md#tuple-lifecycle)
- [Architecture: Threads — Agent Availability Check](../architecture/threads.md#agent-availability-check)
- [Architecture: Threads — Authorization](../architecture/threads.md#authorization)
- [Architecture: Authorization — `agent` type](../architecture/authz.md#agent)
- [Architecture: Authorization — Tuple Lifecycle](../architecture/authz.md#tuple-lifecycle)
- [Architecture: Authorization — Agents Service](../architecture/authz.md#agents-service)
- [Architecture: Authorization — Threads Service](../architecture/authz.md#threads-service)
- [Architecture: Authorization — Chat Service](../architecture/authz.md#chat-service)
- [Architecture: Chat — Authorization](../architecture/chat.md#authorization)
- [Product: Concepts](../product/concepts.md)
- [Product: Console — Agents](../product/console/console.md#agents)

## Delta

### Agent resource

- `availability` field does not exist on the Agent resource. There is no way to mark an agent as `private`.
- `CreateAgent` accepts no availability value. With the new field required, callers must be updated to pass `internal` or `private`.

### Agents service — Roles API

- `SetAgentRole`, `RemoveAgentRole`, `ListAgentRoles`, `ListMyAgentRoles` do not exist.
- On `CreateAgent`, the calling identity is not granted any per-agent role. Currently only the org-level `owner` check gates access.
- Role assignment validation against organization membership is not implemented.

### Agents service — Authorization split

- `GetAgent` returns the full agent record gated only by `member` on org. Configuration fields and sub-resources are not separately gated.
- All sub-resource list and mutation endpoints (MCPs, Skills, Hooks, ENVs, InitScripts, Volume Attachments, Image Pull Secret Attachments) use the legacy `owner`-on-org / `member`-on-org checks rather than `can_read_config` / `can_edit_config` on `agent:<id>`.
- `DeleteAgent` and availability changes are gated by `owner` on org rather than `can_delete` on the agent.

### Threads service — Agent Availability Check

- `CreateThread` and `AddParticipant` do not perform the per-agent `can_initiate` check.
- Identity-type resolution at participant-add time exists only for `@nickname` resolution. `BatchGetIdentityTypes` is not called to discover agent participants.

### Authorization model

- The `agent` OpenFGA type does not exist. The current model has no place to hold per-agent roles, `internal_access`, or the derived `can_initiate` / `can_read_config` / `can_edit_config` / `can_manage_roles` / `can_delete` relations.
- Tuple lifecycle for `agent` creation, deletion, availability toggle, and role mutations is not implemented by the Agents service.

### Console UI

- Agent list does not show an availability column.
- Agent detail does not expose an availability selector.
- Agent detail does not have a Roles section. There is no UI to assign, change, or remove per-agent roles.

### Concepts

- `Agent Availability` and `Agent Role` concepts are not in the product glossary.

## Acceptance Signal

- Creating an agent without `availability` is rejected by the API.
- Creating an agent with `availability=private` and a single owner (the creator) makes the agent un-addable as a thread participant by any other identity until a role is granted.
- `SetAgentRole` rejects identities that are not members of the agent's organization.
- Flipping availability `internal → private` removes the `internal_access` tuple and prevents non-role-holders from adding the agent to new threads, without affecting existing thread participation.
- An app holding `participant:add` on the organization cannot add a `private` agent to a thread unless the app's identity holds an agent role.
- `DeleteAgent` removes every tuple on `agent:<id>` (org, internal_access, role tuples).
- The Console agent list and detail surface the availability value; the agent detail Roles section supports add / change / remove against the organization's members.

## Notes

- "Agent participant" (the per-agent role) and "thread participant" (the relation on the `thread` OpenFGA type) share a name but live on different types — no model collision. Prose disambiguates by context.
- Cross-organization role assignment is rejected for now. A future `public` availability value may permit it; tracked separately, no change file yet.
