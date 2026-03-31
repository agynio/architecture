# 2025-06-27: Channel Removal & Unified Runner Provisioning

## Channel Removal

**Decision:** The `channel` identity type and Channels service have been removed. Bidirectional integrations (e.g., Slack) are now participant apps under the [Apps](../architecture/apps.md) model.

**What changed:**
- Identity types reduced to four: `user`, `agent`, `runner`, `app`.
- `channels.md` deleted.
- All references to channels as a current entity removed across: `identity.md`, `authn.md`, `openziti.md`, `control-data-plane.md`, `gateway.md`, `threads.md`, `notifications.md`, `organizations.md`, `system-overview.md`, `authz.md`, `open-questions.md`, `README.md`, `apps.md`.
- Historical references in `apps.md` and `changes/2025-06-25-apps-concept.md` retained for context.

**Rationale:** The apps model fully subsumes channels. There is no architectural difference between a Slack bridge and a Reminders service — both are apps with different capability sets.

## Unified Runner Provisioning

**Decision:** All runners — regardless of deployment location — use the same provisioning model: register via Terraform/CLI → receive service token → deploy with token → enroll at startup → receive OpenZiti identity.

**What changed:**
- Removed the internal/external runner distinction. No more self-enrollment for runners; only apps concept services (Orchestrator, Gateway, LLM Proxy) use self-enrollment.
- `runner.md`, `k8s-runner.md`, `runners.md`, `authn.md`, `openziti.md` updated to reflect unified flow.

**Rationale:** Runners follow the same pattern as apps — service token provisioning. Self-enrollment is reserved for true infrastructure services that are part of the platform itself.

## Per-Runner OpenZiti Addressing

**Decision:** Each runner gets its own OpenZiti service (`runner-{runnerId}`) created at registration time. Callers (Orchestrator, Terminal Proxy) dial a specific runner by service name.

**What changed:**
- `RegisterRunner` now creates a per-runner OpenZiti service with `roleAttributes: ["runner-services"]` via Ziti Management.
- Static policies: `runners-bind` (Bind, `#runners` → `#runner-services`), `orchestrators-dial-runners` (Dial, `#orchestrators` → `#runner-services`), `terminal-proxy-dial-runners` (Dial, `#terminal-proxy-hosts` → `#runner-services`).
- Runner learns its service name from the enrollment response.
- `DeleteRunner` cleans up the OpenZiti service + identity.
- "Per-Runner OpenZiti Addressing" open question removed from `open-questions.md`.
- `runners.md`, `runner.md`, `k8s-runner.md`, `openziti.md`, `authn.md`, `agents-orchestrator.md` updated.

**Rationale:** With few runners (cluster-level HA + some enterprise-added), per-runner services are simple and follow the same pattern apps already use. No per-runner policies needed — static policies match role attributes on dynamic services.
