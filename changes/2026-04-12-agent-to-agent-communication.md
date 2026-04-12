# Agent-to-Agent Communication

## Target

- [Architecture: agyn CLI — Thread Commands](../architecture/agyn-cli.md#thread-commands)
- [Architecture: agyn CLI — Output Format](../architecture/agyn-cli.md#output-format)
- [Architecture: agynd CLI](../architecture/agynd-cli.md)
- [Architecture: Threads](../architecture/threads.md)
- [Architecture: Agents Orchestrator — Reconciliation](../architecture/agents-orchestrator.md#reconciliation)
- [Architecture: Identity](../architecture/identity.md)
- [Architecture: Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [Architecture: Agents Service](../architecture/agents-service.md)
- [Architecture: Users](../architecture/users.md)
- [Architecture: Apps Service](../architecture/apps-service.md)

## Delta

### agyn CLI — Thread Commands

- `agyn threads` command group does not exist. Agents have no CLI mechanism for agent-to-agent communication.
- `agyn threads create`, `send`, `read`, `add`, `list` commands do not exist.
- Local thread ref state (`~/.agyn/threads.json`) is not implemented.
- `--wait` (notification-based blocking read) is not implemented.
- `--unread`, `--tail`, `--limit` read options do not exist.
- `--passive` flag for participant creation does not exist.
- Multi-thread read (`--thread` repeated) is not supported.
- `--json` and `--yaml` global output format flags do not exist.

### agynd CLI

- `agynd` does not scope `GetUnackedMessages` to `AGYN_THREAD_ID`. It processes messages across all threads.
- `AGYN_THREAD_ID` environment variable is not injected by the Orchestrator.

### Threads Service

- `Participant` model does not have a `passive` field.
- `AddParticipant` does not accept a `passive` flag.
- `AddParticipant` does not accept `@nickname` — only `identity_id`.
- `GetUnackedMessages` does not support `thread_id` filter.

### Agents Orchestrator

- Reconciliation loop does not check participant `passive` flag — it would spin up a workload for passive agent participants.

### Identity Service

- Nickname index does not exist. `SetNickname`, `RemoveNickname`, `ResolveNickname` methods do not exist.
- `org_nicknames` storage does not exist.

### Agent Resource

- `nickname` field does not exist on the Agent resource.
- Agents service does not register or update nicknames with the Identity service on agent create/update.

### Users Service

- User profile update does not support `nickname` (per-org handle). Users have no way to set a `@mention` handle within an organization.

### Apps Service

- App installation does not register a default nickname on install.
- `UpdateInstallation` does not support `nickname`.

## Acceptance Signal

- An agent can call `agyn threads create --ref research --add @research_bot --message "..." --wait 120` and receive a response.
- `agyn threads read --thread research --thread planning --unread --wait 60` returns messages from both threads, formatted as markdown by default and as JSON with `--json`.
- `agynd` scopes message processing to `AGYN_THREAD_ID` — sub-thread messages are not fed into the main agent loop.
- Passive participants receive messages but the Orchestrator does not start a workload for them.
- `@nickname` resolution works across agents, users, and app installations within an org. Nicknames are unique per org.
- Agent, user, and app installation nicknames can be set and updated via their respective service update methods.

## Notes

- Agent-to-agent communication replaces the previous special-purpose tool approach with a general CLI interface compatible with all agent implementations (Claude Code, Codex, agn).
- Nickname uniqueness is enforced by the Identity service. Individual services call Identity internally — external callers interact only with the owner service.
