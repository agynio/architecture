# Sandboxes App and Sandbox Sharing

## Target

- [Sandboxes App](../product/sandboxes/sandboxes-app.md) (new)
- [Sandboxes App](../architecture/sandboxes-app.md) (new)
- [Sandboxes — Permissions](../product/sandboxes/sandboxes.md#permissions)
- [Sandboxes — Sharing](../product/sandboxes/sandboxes.md#sharing)
- [Authorization — sandbox](../architecture/authz.md#sandbox)
- [Agents Service — Sandbox Sharing](../architecture/agents-service.md#sandbox-sharing)
- [Terminal Proxy — Authorization](../architecture/terminal-proxy.md#authorization)
- [Notifications — Room Naming Convention](../architecture/notifications.md#room-naming-convention)
- [Console — Sandboxes](../product/console/console.md#sandboxes)
- [Services](../architecture/services.md)

## Delta

**Sandboxes have no user-facing surface.** Every organization member can create one — `can_create_sandbox` is computed from `member` — but the only UI for them is the [Console](../product/console/console.md), which members cannot open. A member's sandboxes are reachable exclusively through [`agyn sandbox`](../architecture/agyn-cli.md#sandbox-commands). The Console's Operations section is the wrong home for them regardless: it is an org owner's list-all view over other people's workloads, and its authorization deliberately excludes attaching.

This change adds **`agynio/sandboxes-app`**, a fourth SPA served at `sandboxes.<domain>`, alongside Chat, Tracing, and the Console. Its job is the working surface: start a sandbox, get a shell in the browser, stop it, share it. The Console's Sandboxes section stays where it is and keeps its own job — the org-wide fleet view — and the two are distinguished by the permissions that already exist (`can_list_sandboxes` there, `owner`/`can_connect` here).

Nothing about a sandbox's composition changes. It remains the environment's contents without the agent loop, and the app creates one from an environment reference plus at most a name and an idle timeout.

### The app

Three routes, no sidebar — an org switcher in the breadcrumb, following [Tracing App](../product/tracing/tracing-app.md) rather than the Console's three-region layout.

- **List** — the caller's sandboxes as cards, then a second group for sandboxes shared with them. A card carries the name and status, the environment, both lifetime clocks (idle and TTL), the no-persistent-volume warning when it applies, and a primary action that fills it: Connect when running, Start otherwise.
- **New sandbox** — an environment picker, an optional name, an optional idle timeout. Everything else is shown read-only as it resolves from the environment, so what the sandbox will contain is visible before it exists.
- **Sandbox** — status, environment, both clocks, and a full-page terminal over the [Terminal Proxy](../architecture/terminal-proxy.md).

Cards rather than rows because a member has a handful of sandboxes and the list is a launcher, not a table to sweep. The Console's list-all view stays a sortable table for the opposite reason.

### Sharing

A sandbox owner may grant other identities in the organization access to one specific sandbox. This introduces a `collaborator` relation on the `sandbox` type:

```
type sandbox
  relations
    define org: [organization]
    define owner: [identity]
    define collaborator: [identity, group#member]
    define can_read: owner or collaborator or owner from org
    define can_connect: owner or collaborator
    define can_stop: owner or collaborator or owner from org
    define can_delete: owner or owner from org
    define can_share: owner
```

- **`can_connect` gains `collaborator`.** A collaborator attaches a shell exactly as the owner does, through the same ticket path, for every [session kind](../architecture/terminal-proxy.md#session-kinds) including `SYNC` — a session reaching the same filesystem a shell does is authorized identically, which is the rule already recorded for owners.
- **Start comes free.** `EnsureSandboxRunning` is already gated by `can_connect`, so a collaborator can restart an idled-out sandbox without a further grant. `can_stop` is extended to match: a collaborator who can start what they are working in can stop it too.
- **`can_delete` and `can_share` stay with the owner.** A collaborator cannot destroy the sandbox or its disks, and cannot pass the grant on. Organization owners keep force-delete and still cannot attach or share.
- **`can_read` is introduced** so `GetSandbox` has one relation to check instead of enumerating owner, collaborator, and org owner at each call site.
- **New RPCs on the Agents service** — `ShareSandbox`, `UnshareSandbox`, `ListSandboxShares` — all gated by `can_share`. Targets must satisfy `member` on the sandbox's organization, the same check `SetAgentRole` applies.
- **`ListSandboxes` returns collaborated sandboxes**, distinguished by scope, so a share is discoverable without the owner sending a link.
- **A new `sandbox:{id}` notification room**, gated by `can_read`, carries `sandbox.updated` to collaborators. The existing `sandbox_owner:{owner_id}` room is identity-keyed and cannot serve them; it is unchanged.

**A share is a grant over one sandbox, not over its environment.** `can_use` on an [environment](../product/environments/environments.md#who-can-use-an-environment) governs whether someone may *create* a sandbox there; a share governs who may enter *this* one. They are separate questions and a collaborator needs only the second.

The consequence is deliberate and must be stated where a person can act on it: a shell in a sandbox reaches the environment's secret-backed ENVs, its credential-injecting egress rules, and the contents of its volumes, so sharing hands all three to someone who may hold no role on that environment. [Sandboxes — Permissions](../product/sandboxes/sandboxes.md#permissions) previously described environments as the effective sharing boundary; with a per-sandbox grant that is no longer true as written and the claim is corrected there. The share dialog names what it hands over at the moment of sharing.

Metering is unaffected: `FLAVOR_SECONDS` records stay attributed to the organization and labeled with the sandbox and its owner. A collaborator can therefore start compute that bills to the owner, which the share dialog also states.

### Supporting change

**`can_use` is exposed on environment reads.** The `Environment` message carries no indication of whether the caller may run in it, and there is no per-caller role listing for environments. A picker can infer the answer for `availability=internal` and is blind for `private`, so the create form would offer environments that `CreateSandbox` then rejects. Environment metadata reads gain a computed `can_use` boolean for the calling identity — one field on an existing response, no new RPC.

## Acceptance Signal

- An organization member with no Console access opens `sandboxes.<domain>`, creates a sandbox in an environment they can use, and gets a working shell — colors, resize, `vim`, correct exit code — without touching the CLI.
- Creating in an environment that declares no persistent volume warns before the sandbox exists, and the resulting card carries the warning.
- The create form offers only environments the caller can use, including `private` ones where they hold a role, and omits `private` ones where they do not.
- An owner shares a sandbox with a member holding no role on its environment; that member sees it under shared sandboxes, opens a shell, stops it, and starts it again.
- The same collaborator cannot delete the sandbox and cannot add a third person.
- An organization owner sees every sandbox in the Console's list-all view, force-deletes one, and is refused a terminal on a sandbox they do not own or collaborate on.
- Unsharing takes effect on the next ticket issuance; an already-attached session is unaffected until it ends.
- Status transitions driven by another client — the CLI, the idle timeout — appear on the owner's list and on a collaborator's open detail page without a refresh.
- Compute started by a collaborator appears in the owner's usage.

## Notes

- **No CLI share commands here.** `agyn sandbox share` is the obvious counterpart and is deliberately not specified in this change, which is scoped to the app and the authorization model beneath it. The RPCs are CLI-ready.
- **`agyn sandbox list` picks up shared sandboxes for free**, since it calls the same `ListSandboxes` that now returns them. Distinguishing them in its output is a CLI presentation decision left to [agyn-cli](../architecture/agyn-cli.md#sandbox-commands).
- **Workspace sync and `cp` stay out of the app.** Both need a process on the engineer's machine; a browser tab is the wrong host. They remain [CLI features](../architecture/sandbox-sync.md).
- **Unsharing does not kill live sessions.** Terminal tickets are validated at issuance, and a session already attached runs until it ends. Revocation that reaches into open sessions is a Terminal Proxy capability that does not exist for any consumer today and is not introduced for this one.
- **Collaborators are not carried across a re-create.** Shares live on the sandbox record; deleting a sandbox and starting a new one starts with an empty share list.
- **`comingSoon` in the Console's product switcher.** The switcher already lists a `sandboxes` product marked unreleased; deploying the app is what releases it. The switcher itself remains undocumented in this corpus — pre-existing drift, not addressed here.
- Per-environment or per-organization policy on who may be shared with — an organization wanting sharing confined to holders of `can_use` on the environment — is a reasonable future setting and deliberately not specified. v1 puts the decision with the sandbox owner.
