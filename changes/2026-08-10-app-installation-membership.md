# App Installations Grant Organization Membership

## Target

- [Apps — Organization Membership](../architecture/apps.md#organization-membership) (new)
- [Apps — Permissions Bridge](../architecture/apps.md#permissions-bridge)
- [Authorization — App Installation Permissions](../architecture/authz.md#app-installation-permissions)
- [Apps Service — Interface](../architecture/apps-service.md#api)
- [Threads — Agent Availability Check](../architecture/threads.md#agent-availability-check)
- [Agents Service — Roles](../architecture/agents-service.md#roles)

## Delta

**An installed app cannot add any agent to any thread.** It holds the permissions it declared and still fails the [Agent Availability Check](../architecture/threads.md#agent-availability-check) on every agent in the organization, at both availability levels.

`can_initiate` resolves as `owner or maintainer or participant or member from internal_access`. An installation writes `thread_create`, `thread_write`, `participant_add` and `inbox_write` — never `member`. So:

- **`internal` agents** are not reachable. Availability `internal` writes `organization:<org_id>, internal_access, agent:<id>`, and the derived clause is `member from internal_access`. An app is not a member, so nothing resolves. `internal` means "any org member", and an app is not one.
- **`private` agents** are not reachable either, and cannot be made reachable. The documented remedy is for an agent `owner` to grant the app a role via `SetAgentRole` — but that call "rejects identities that are not members of the agent's organization", so the grant is refused. The Console's share dialog offers organization members only, so there is no path through the UI either.

Both bundled connectors are affected. The failure surfaces the first time a connector tries to serve a mention:

```
add participant: internal: create agent instance:
rpc error: code = PermissionDenied desc = identity cannot initiate this agent
```

This change makes an installation write `identity:<app_identity_id>, member, organization:<org_id>` alongside the permission tuples, removed on uninstall.

### Why membership rather than a narrower grant

Two narrower designs were considered and rejected.

A fifth installation permission (`agent:initiate`) feeding a new `initiate_access` relation on the agent type would be org-wide: one grant would let a connector initiate every agent in the organization, which is broader per-agent than membership is, while adding a relation to the model.

Relaxing `SetAgentRole` to accept "an org member **or** an app installed in this org" keeps per-agent granularity but pushes a second branch into a membership check — and that check is not the only one. Every relation computed from `member` would eventually need the same branch, in every service that evaluates one. The branching is a larger and more permanent cost than the reach it avoids.

### What membership widens

An installed app gains what any org member has: `can_create_thread`, `can_create_sandbox`, `can_create_environment`, agent and environment metadata reads, and `can_initiate` on `internal` agents. Sandboxes and environments are wider than an app needs. They are accepted rather than carved out, for the reason above.

### Not written to the memberships table

Membership is recorded only as an authorization tuple. The [Organizations](../architecture/organizations.md) service's `memberships` table models people joining organizations — invitations, acceptance, and nickname seeding that is explicitly conditioned on the target identity being a user. None of it applies to a service installed by an org owner. An app therefore does not appear in `ListMembers`, and member listings stay human.

## Acceptance Signal

- `InstallApp` writes `identity:<app_identity_id>, member, organization:<org_id>` for every installation, whatever the app declared, including apps declaring no permissions.
- `UninstallApp` removes that tuple with the permission tuples.
- No `memberships` row is created for an app, and `ListMembers` does not return app identities.
- An app installed in an organization satisfies `can_initiate` on an agent whose availability is `internal`, with no per-agent grant.
- `SetAgentRole` accepts an installed app as the target identity, so an agent `owner` can grant a `private` agent to a connector.
- A connector configured with an `internal` agent forwards a mention end to end without a `PermissionDenied` on `AddParticipant`.
- Uninstalling an app revokes `can_initiate` on `internal` agents in that organization; agent roles granted to the app are left in place.
- Cross-organization grants remain rejected: an app installed in org A is not a member of org B and cannot hold a role on an agent there.

## Notes

Found while bringing up the [Slack Connector](../architecture/apps/slack-connector.md) against a real workspace: the connector reached `AddParticipant` and was refused on an `internal` agent. The [Telegram Connector](../architecture/apps/telegram-connector.md) has the same gap and would fail identically on a clean install.

The Console's agent share dialog reads *"Select an organization member who can use this agent when Availability is Private."* Both halves mislead once an app can hold a role — the grantee need not be a person, and an app needs the grant only for `private` agents, having `internal` ones through membership. Whether apps should be offered in that picker at all is a product decision this change does not make: under tuple-only membership they will not appear there, since the picker lists `ListMembers`.

`agyn` has no `agents roles` command — only `environments roles`. Granting an app a role on a `private` agent is Console-only until one exists.
