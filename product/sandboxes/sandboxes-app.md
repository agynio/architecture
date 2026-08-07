# Sandboxes App

## Purpose

The Sandboxes app is where a person works with their own [sandboxes](sandboxes.md): starts one, gets a shell in the browser, stops it when done, and shares it with a colleague. It is the member-facing half of the sandbox surface — the [Console](../console/console.md) holds the other half, an organization owner's view over every sandbox in the organization.

Two facts make it a separate app rather than a Console section. Any organization member may create a sandbox, and members have no Console access at all. And attaching to a sandbox is a right the Console's sandbox section deliberately does not carry — an organization owner can terminate any sandbox and open none of them.

| | Console → Operations → Sandboxes | Sandboxes app |
|---|---|---|
| Audience | Organization owners, cluster admins | Any organization member |
| Backed by | `can_list_sandboxes` on the organization | `owner` / `collaborator` on the sandbox |
| Purpose | See and terminate what is running | Start, enter, share, stop your own |
| Terminal | No | Yes |

## User Stories

- As an engineer, I want to start a sandbox and get a shell in my browser, so I can work in my agents' runtime without setting up a terminal.
- As an engineer, I want to see how long my sandbox has before it stops and before it disappears, so neither is a surprise.
- As an engineer, I want to know before I start working whether my files will survive the sandbox going idle.
- As an engineer, I want to hand a colleague a shell in the sandbox I am debugging, without giving them anything else.
- As an engineer, I want a sandbox someone shared with me to appear in my list, so I do not depend on them sending me a link.
- As a sandbox owner, I want to see exactly what sharing gives away before I share.
- As an engineer, I want to come back to a stopped sandbox and start it again from the same page.

## Navigation

Three levels, no sidebar:

```
Sandboxes → Sandbox
```

A breadcrumb sits top-left on every page: the product switcher, then a **Sandboxes** segment linking back to the list, then the **Sandbox** name as a static label on a sandbox page. The way out of a page is therefore in the same place on every page rather than inside the page's own actions.

The **user menu** sits top-right and carries the organization switcher, as in the [Console](../console/console.md) — it shows who is signed in and which organization, and selecting another loads its sandboxes. Every route here is organization-scoped, so there is no organization picker screen: a caller lands on their last-used organization, or the first one they belong to, and switches from the menu.

The [product switcher](../console/console.md) reaches Chat, Tracing, and the Console.

## Sandboxes

The landing page. Two groups: **Your sandboxes**, then **Shared with you**. Each is a grid of cards that collapses to a single column on a narrow viewport. Terminated sandboxes are not shown.

Cards rather than a table. A member has a handful of sandboxes, and this page exists to get them into one — a launcher, not a list to sort and sweep. The Console's org-wide view is a table for the opposite reason.

Each card shows:

| Element | Description |
|---------|-------------|
| Name | The sandbox name, with a status indicator — `starting`, `running`, `stopped`, `failed` |
| Environment | The environment it runs, linked. The only configuration a sandbox has, and the one identifying fact besides its name |
| Stops in | Time remaining on the [idle timeout](sandboxes.md#lifecycle), counting down while nothing is attached. Absent while a session is attached — an attached sandbox is never idle — and absent when stopped |
| Expires in | Time remaining on the TTL. Always shown: it applies in every state, including stopped |
| Storage warning | Present when the environment declares no persistent volume: nothing in this sandbox survives a stop. See [Storage](sandboxes.md#storage) |
| Owner | On shared cards only — who owns the sandbox |
| Primary action | **Connect** when running, **Start** when stopped or failed. Fills the card |
| Manage menu | Stop, Share, Delete. Behind a menu so the destructive actions are not under the cursor |

A collaborator's card offers Connect, Start, and Stop, and no Share or Delete.

Both clocks appear because two independent bounds govern a sandbox's life; showing one and hiding the other is what makes the second a surprise.

**Real-time** — status changes appear without a refresh, whoever caused them: the idle timeout stopping a workload, the CLI starting one, a collaborator stopping the sandbox you are looking at.

**Empty state** — a single **New sandbox** action, and an explanation of what a sandbox is for.

## New Sandbox

A sandbox carries almost no configuration of its own — its contents come from its [environment](../environments/environments.md) — so the form is an environment picker and two optional fields.

| Field | Description |
|-------|-------------|
| Environment | Required. Lists the environments the caller may run workloads in. An organization with exactly one preselects it |
| Name | Optional. Auto-generated as `adjective-noun` when omitted. Unique within the organization |
| Idle timeout | Optional. Defaults to the organization's default. A value above the organization's ceiling is rejected, naming the ceiling — never quietly reduced, because a timeout the engineer never sees is one they plan around wrongly |

Selecting an environment shows what the sandbox will contain, read-only: its images, its flavor, its runner, its volumes and where they mount. The point is that what a sandbox is made of is visible before it exists, not discovered inside it.

**The storage warning is part of the form, not a footnote.** When the chosen environment declares no persistent volume, the form says so before the sandbox is created: everything written in this sandbox is lost when it stops, and it stops on its own. This is the easiest consequence in the product to walk into.

The picker offers only environments the caller may [use](../environments/environments.md#who-can-use-an-environment) — every `internal` environment in the organization, plus `private` ones where they hold a role. Offering an environment that creation would then reject teaches the wrong thing about who may do what.

## Sandbox

Status, environment, both clocks, and a terminal filling the rest of the page.

**Terminal** — a real PTY with no platform-imposed limits: colors through truecolor, alternate screen, mouse reporting, full-screen TUIs, resize, working signals, and the shell's exit code reported on exit. It fills the page below the header, because it is what the page is for. See [Sandboxes — Shell Access](sandboxes.md#shell-access).

**Tabs** — several shells in one sandbox, as any terminal emulator gives you: a tab strip above the terminal with a new-shell action, and a close action once more than one is open. Each tab is an independent session, so a background tab keeps running and keeps its scrollback — the [wire protocol](../../architecture/terminal-proxy.md#wire-protocol) has no resume, and a tab that was torn down on leaving it would come back a different shell. Closing a tab ends that session, exactly as closing the page ends them all.

A tab is named by what its shell announces — the title it sets, or failing that the directory it reports — and falls back to the number it opened with. That number is fixed at open and never reused: renumbering by position would rename the shells that were kept when one is closed.

Tabs are reordered by dragging, and with `Alt`+`←`/`→` for anyone not using a mouse.

While a session is attached the sandbox cannot idle out. Closing the tab ends that session — like dropping an SSH connection, the foreground process group is signalled and the container keeps running. Work that must outlive a tab belongs under `tmux` or `nohup` from the image.

**Stopped, starting, or failed** — the terminal area is replaced by the state and a **Start** action. Starting is a no-op when already running, a restart when stopped, and a fresh attempt when failed; the terminal appears once the workload runs. A `failed` sandbox stays failed until someone acts — nothing retries it in the background, because nothing needs a sandbox to run while nobody is connecting.

**Actions** — Stop, Share, Delete. Stopping a sandbox whose environment declares no persistent volume warns that everything in it is discarded. Deleting warns that the sandbox's disks go with it.

## Sharing

A sandbox owner can give other members of the organization access to that one sandbox.

**What a collaborator gets:** a shell, and the ability to start and stop the sandbox. Starting matters — a shared sandbox that idled out is useless to a collaborator who cannot bring it back.

**What a collaborator does not get:** deleting the sandbox, and sharing it with anyone else. The share list belongs to the owner.

**Managing shares** — on the sandbox page. Add people or [groups](../console/console.md#groups) from the organization; remove them one at a time. Removal takes effect on the next connection; a session already open runs until it ends.

**The dialog states the consequences before the share is made**, because none of them are visible from the outside:

- A shell in this sandbox reaches the environment's secrets, its credential-injecting egress rules, and the contents of its volumes. The person you share with gets all three, whether or not they may use that environment themselves.
- They can start it, and the compute bills to you.
- They cannot delete it and cannot share it further.

**Sharing a sandbox is not sharing its environment.** Being allowed to *create* a sandbox in an environment is a separate permission, held by whoever the environment's owner granted it to. A share opens one specific sandbox and grants nothing about the environment behind it — a collaborator cannot start a second sandbox there.

Organization owners are not collaborators by default. They can see and terminate every sandbox from the Console; entering one requires the owner to share it, exactly as for anyone else.

## Constraints

- Backend communication is ConnectRPC through the [Gateway](../../architecture/gateway.md); the terminal is a WebSocket to the [Terminal Proxy](../../architecture/terminal-proxy.md).
- The app is a static SPA with no backend of its own.
- [Workspace sync](sandboxes.md#workspace-sync) and `agyn sandbox cp` are not in the app. Both reconcile against a directory on the engineer's machine, which a browser tab cannot reach; they stay CLI features.
- Port exposure is not managed here — `agyn expose` runs inside the sandbox, from its shell.
- The app shows the caller's own and shared sandboxes only. The organization-wide view is the Console's.
- The environment's workspace image must provide a shell for the terminal to be useful; the platform injects none.

## Future

- A compact list mode for users who accumulate many sandboxes; cards trade scanning density for a bigger primary action, which is the right trade at a handful and the wrong one at thirty.
- Organization policy on who may be shared with — confining shares to identities that already hold `can_use` on the environment.
- Sharing that reaches into live sessions on revocation.

## Related architecture

- [Sandboxes App](../../architecture/sandboxes-app.md)
- [Sandboxes](sandboxes.md)
- [Terminal Proxy](../../architecture/terminal-proxy.md)
- [Agents Service](../../architecture/agents-service.md)
- [Authorization — sandbox](../../architecture/authz.md#sandbox)
- [Flavors and Environments](../environments/environments.md)
