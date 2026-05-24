# Services

Central inventory of deployable long-running services in the platform. This list excludes CLIs, SDKs, init images, Terraform providers, documentation repositories, and in-process packages.

| Service | Repository | Architecture doc | Plane | Role |
|---------|------------|------------------|-------|------|
| Agents | [agynio/agents](https://github.com/agynio/agents) | [Agents Service](agents-service.md) | Control plane | Manages agent resource definitions, volumes, MCP servers, skills, hooks, and related configuration. |
| Agents Orchestrator | [agynio/agents-orchestrator](https://github.com/agynio/agents-orchestrator) | [Agents Orchestrator](agents-orchestrator.md) | Control plane | Reconciles thread demand and runtime state into agent workloads. |
| Apps Service | [agynio/apps](https://github.com/agynio/apps) | [Apps Service](apps-service.md) | Control plane / Data plane | Manages app definitions, installations, app identities, profiles, and installation configuration. |
| Authorization | [agynio/authorization](https://github.com/agynio/authorization) | [Authorization](authz.md#authorization-service) | Data plane | Central gRPC proxy for OpenFGA permission checks and relationship tuple writes. |
| Chat | [agynio/chat](https://github.com/agynio/chat) | [Chat](chat.md) | Data plane | Implements the built-in chat experience on top of Threads, including unread counts and activity status. |
| Chat App | [agynio/chat-app](https://github.com/agynio/chat-app) | [Chat](chat.md) | App | Hosts the user-facing chat SPA. |
| Console App | [agynio/console-app](https://github.com/agynio/console-app) | [Console](console.md) | App | Hosts the platform administration SPA. |
| Expose | [agynio/expose](https://github.com/agynio/expose) | [Expose Service](expose-service.md) | Data plane | Manages OpenZiti port exposures for running agent workloads. |
| Files | [agynio/files](https://github.com/agynio/files) | [Media Support](media.md) | Data plane | Stores uploaded file metadata and content, and serves file access APIs backed by object storage. |
| Gateway | [agynio/gateway](https://github.com/agynio/gateway) | [Gateway](gateway.md) | Data plane | Authenticates external requests and routes Gateway APIs to internal platform services. |
| Identity | [agynio/identity](https://github.com/agynio/identity) | [Identity](identity.md) | Data plane | Maintains the central identity registry and nickname resolution. |
| k8s-runner | [agynio/k8s-runner](https://github.com/agynio/k8s-runner) | [k8s-runner](k8s-runner.md) | Runtime | Executes workloads as Kubernetes pods and reports runtime state. |
| LLM | [agynio/llm](https://github.com/agynio/llm) | [LLM](llm.md) | Data plane | Manages LLM providers and models and resolves model IDs for the LLM Proxy. |
| LLM Proxy | [agynio/llm-proxy](https://github.com/agynio/llm-proxy) | [LLM Proxy](llm-proxy.md) | Data plane | Serves OpenAI-compatible and Anthropic-compatible LLM APIs for agents and forwards requests to providers. |
| Media Proxy | [agynio/media-proxy](https://github.com/agynio/media-proxy) | [Media Proxy](media-proxy.md) | Data plane | Authenticated media proxy for external media URLs and platform file URLs. |
| Metering | [agynio/metering](https://github.com/agynio/metering) | [Metering Service](metering.md) | Data plane | Ingests, stores, and queries platform usage records. |
| Notifications | [agynio/notifications](https://github.com/agynio/notifications) | [Notifications](notifications.md) | Data plane | Fans out real-time events to subscribed clients and services. |
| Organizations | [agynio/organizations](https://github.com/agynio/organizations) | [Organizations](organizations.md) | Control plane | Manages organization lifecycle, membership, invitations, and organization-scoped access lists. |
| Reminders | [agynio/reminders](https://github.com/agynio/reminders) | [Reminders](apps/reminders.md) | App | Provides delayed reminder messages as a platform app. |
| Runners | [agynio/runners](https://github.com/agynio/runners) | [Runners](runners.md) | Control plane / Data plane | Registers runners and stores workload and volume runtime state. |
| Secrets | [agynio/secrets](https://github.com/agynio/secrets) | [Secrets](secrets.md) | Data plane | Manages secret providers and secrets, and resolves secret values at runtime. |
| Telegram Connector | [agynio/telegram-connector](https://github.com/agynio/telegram-connector) | [Telegram Connector](apps/telegram-connector.md) | App | Bridges Telegram bot conversations into platform threads and back. |
| Threads | [agynio/threads](https://github.com/agynio/threads) | [Threads](threads.md) | Data plane | Stores messages, thread participants, and acknowledgments for participant-agnostic conversations. |
| Tracing | [agynio/tracing](https://github.com/agynio/tracing) | [Tracing](tracing.md) | Data plane | Ingests, stores, and queries OpenTelemetry spans with upsert semantics. |
| Tracing App | [agynio/tracing-app](https://github.com/agynio/tracing-app) | [Tracing](tracing.md) | App | Hosts the tracing and run timeline UI. |
| Users | [agynio/users](https://github.com/agynio/users) | [Users](users.md) | Data plane | Provisions user identities, serves user profiles, manages devices, and resolves API tokens. |
| Ziti Management | [agynio/ziti-management](https://github.com/agynio/ziti-management) | [OpenZiti](openziti.md) | Infrastructure-adjacent | Manages OpenZiti identities, services, policies, and enrollment flows. |
