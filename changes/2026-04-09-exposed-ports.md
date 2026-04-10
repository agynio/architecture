# Exposed Ports (Dev Server Links)

## Target

- [Product — Exposed Ports](../product/chat/exposed-ports.md)
- [Architecture — Expose Service](../architecture/expose.md)
- [OpenZiti Integration — Dynamic Policies](../architecture/openziti.md#dynamic-policies)

## Delta

- There is no Users service + Console workflow for minting Ziti Desktop Edge / tunnel enrollment tokens for user machines.
- There is no Expose service for managing exposed ports.
- Agents cannot request that a local port be exposed over OpenZiti.
- The platform does not provision per-exposed-port OpenZiti services/configs/policies for port exposure.

## Acceptance Signal

- A user can generate an enrollment token in the Console and enroll Ziti Desktop Edge / tunnel.
- An agent can request an exposed port and receive a URL `http://exposed-<id>.ziti:<port>`.
- A user can open the URL and reach the dev server inside the agent pod.
- Exposed ports are cleaned up when revoked or when the workload stops (with GC as a backstop).

## Notes

- Access scope (which user devices can dial exposed ports) is TBD. Track the decision in [OpenZiti: Exposed Ports Access Scope](../open-questions.md#openziti-exposed-ports-access-scope).
