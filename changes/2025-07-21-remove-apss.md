# Remove Agent Persistent State Service (APSS)

## Target

- [Agent State](../architecture/agent/state.md)
- [Agent Implementation](../architecture/agent/implementation.md)
- [System Overview](../architecture/system-overview.md)

## Delta

The architecture now describes agent state as disk-based (persistent volume), managed by each agent implementation. The current implementation includes the APSS service (`agynio/agent-state` repository) with a gRPC API backed by PostgreSQL. This service, its proto definitions (`agynio/api/agent_state/v1/`), and the Gateway proxy (`AgentStateGateway`) need to be removed.

## Acceptance Signal

- `agynio/agent-state` repository is archived or deleted.
- Agent state proto definitions (`agent_state/v1/`) and gateway proto (`gateway/v1/agent_state.proto`) are removed from `agynio/api`.
- `agn` uses only filesystem-based state persistence (remote/APSS backend removed).
- Agent containers use a persistent volume for state; no gRPC calls to Agent State service.

## Notes

- The `agynio/agent-state` service is currently deployed and in use.
- Migration path: agents switch to filesystem persistence backed by PVCs. Existing APSS data does not need migration — agent state is ephemeral by nature and will be rebuilt from threads on next activation.
