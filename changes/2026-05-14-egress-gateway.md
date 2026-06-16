# Egress Gateway

## Target

- [Product — Egress Gateway](../product/egress-gateway/egress-gateway.md)
- [Architecture — Egress Gateway](../architecture/egress-gateway.md)
- [Architecture — EgressRules Service](../architecture/egress-rules-service.md)
- [Resource Definitions — Egress Rule](../architecture/resource-definitions.md#egress-rule)
- [Agents Orchestrator — Egress CA Distribution](../architecture/agents-orchestrator.md#egress-ca-distribution)
- [Runner — Inline Files](../architecture/runner.md#inline-files)
- [k8s-runner — Workload Egress NetworkPolicy](../architecture/k8s-runner.md#workload-egress-networkpolicy)
- [k8s-runner — Workload-to-Kubernetes Mapping](../architecture/k8s-runner.md#workload-to-kubernetes-mapping)
- [OpenZiti — Static Policies](../architecture/openziti.md#static-policies)
- [Authorization — EgressRules Service](../architecture/authz.md#egressrules-service)
- [Tracing — Egress Spans](../architecture/tracing.md#egress-spans)
- [Metering — Egress Records](../architecture/metering.md#egress-records)
- [Secrets — Referential Integrity with EgressRules](../architecture/secrets.md#referential-integrity-with-egressrules)

## Delta

There is no mechanism today for the platform to control outbound HTTP/HTTPS traffic from agent workloads. Secrets used by agents to authenticate to third-party APIs (e.g., a GitHub token) are injected as environment variables into the agent container — they live in the agent's process environment for the lifetime of the workload, are visible to every tool the agent invokes, and travel with any process the agent forks. There is also no mechanism to block agents from reaching specific destinations or to observe which destinations an agent actually calls. Workloads can reach arbitrary cluster-internal services.

The new EgressRules service, the Egress Gateway, the workload-namespace NetworkPolicy installed with the runner, and the cert-manager-managed Egress CA collectively close this gap. The desired state is described in the documents linked above; this delta enumerates what's missing.

- **EgressRules service** — `agynio/egress-rules` repository does not exist. The service owns the `EgressRule` and `EgressRuleAttachment` resources, their PostgreSQL storage, the per-rule OpenZiti service lifecycle via [Ziti Management](../architecture/openziti.md), per-attachment Dial policy lifecycle, reconciliation loop, and change-event publishing. Provides `ListEgressRulesByAgent` as an internal RPC consumed by the Egress Gateway. Does not exist in any form today.

- **Egress Gateway service** — `agynio/egress-gateway` repository does not exist. The service binds the OpenZiti `egress-services` role via the SDK, terminates TLS using the platform-managed CA, evaluates attached rules per request via `ListEgressRulesByAgent`, resolves secret references via the [Secrets](../architecture/secrets.md) service, and forwards to the upstream destination. Refuses WebSocket upgrades with `426 Upgrade Required`. Does not exist today.

- **cert-manager dependency in the platform cluster** — cert-manager is not currently deployed. It is required to manage the Egress CA lifecycle. Bootstrap must install cert-manager and apply the `ClusterIssuer` (`agyn-selfsigned`) and the `Certificate` (`egress-ca`, `isCA: true`, 10y duration, ECDSA) that materialize the `egress-ca` Secret in the platform namespace.

- **`EgressRule` resource shape** — structured as a `matcher` sub-object (`domain_pattern` required, `ports` default `[80, 443]`, `methods?`, `path_pattern?`) plus an `effect` sub-object (`action` enum, `inject` headers list). Uniqueness on `(organization_id, matcher.domain_pattern)`. Reserved domain patterns rejected at create time: `*.ziti`, `*.svc`, `*.cluster.local`, anything overlapping `100.64.0.0/10`. Each `inject` header carries a `name`, an optional auth `scheme` (`bearer` / `basic`), and exactly one of `value` (literal credential) or `secret_id` (reference to a [Secret](../architecture/providers.md#secret) in the rule's org). When `scheme` is set, the emitted header value is `<Scheme> <credential>`.

- **Proto definitions in `agynio/api`** — new package `egress/v1` with `EgressRule`, `EgressRuleAttachment`, the matcher/effect/header messages, and `EgressRulesGateway` service definition (`CreateEgressRule`, `GetEgressRule`, `ListEgressRules`, `UpdateEgressRule`, `DeleteEgressRule`, `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment`, `ListEgressRuleAttachments`). Internal-only `ListEgressRulesByAgent` not exposed through the Gateway.

- **OpenZiti static policy and role attribute** — bootstrap must add the `egress-gateway-bind` Bind policy (`#egress-gateway-hosts` → `#egress-services`) and the role attribute `egress-gateway-hosts` for the gateway's self-enrolled identity. The role attribute `egress-services` is the per-rule service tag.

- **Per-rule OpenZiti service provisioning** — the EgressRules service must call [Ziti Management](../architecture/openziti.md) `CreateService` on rule create (with `intercept.v1` keyed on `matcher.domain_pattern` and `matcher.ports`, and `host.v1` with `forwardAddress: true`, `forwardPort: true`, `forwardProtocol: true`) and `DeleteService` on rule delete. Attaching a rule must call `CreateServicePolicy` (Dial, `identityRoles: ["#agent-<agent_id>"]`, `serviceRoles: ["@<openziti_service_id>"]`); detaching must call `DeleteServicePolicy`. The `egress-rule-<rule_id>` value is the service name; `@<openziti_service_id>` targets the concrete OpenZiti service ID returned by Ziti Management and stored with the rule. Following the [Expose Service](../architecture/expose-service.md) pattern. Reconciliation loop required to repair drift.

- **Inline-file support in the workload spec** — `StartWorkloadRequest` in `agynio/api` must gain a `map<string, bytes> inline_files` field with semantics: keys are absolute container paths, values are file bytes (mounted read-only via projected volume in the listed containers — per-container, not pod-wide). The [Agents Orchestrator](../architecture/agents-orchestrator.md) populates it with the Egress CA public cert (`/etc/agyn/egress-ca/ca.crt`). The [k8s-runner](../architecture/k8s-runner.md) materializes each entry as a per-pod Secret (or projected volume) and mounts it at the requested path in the workload containers that reference it. Used uniformly for in-cluster and external runners.

- **Workload-namespace egress NetworkPolicy** — the runner installation (Helm chart, kustomize, or operator-supplied manifest) must ship a parameterizable NetworkPolicy template in the workload namespace that allows egress to the OpenZiti synthetic range (`100.64.0.0/10`), cluster DNS, and public internet, while blocking the cluster pod CIDR, cluster service CIDR, and any operator-declared internal CIDRs. The policy targets all agent workload pods by the `agyn.dev/managed-by: agents-orchestrator` label. The k8s-runner runtime does not manage this NetworkPolicy and does not require `networkpolicies` RBAC. See [k8s-runner — Workload Egress NetworkPolicy](../architecture/k8s-runner.md#workload-egress-networkpolicy).

- **CA-trust env vars in every container** — the [Agents Orchestrator](../architecture/agents-orchestrator.md) must set `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`, `CURL_CA_BUNDLE`, and `SSL_CERT_DIR` in every container of the workload spec (agent, MCP sidecars, hooks), pointing at the mounted CA file path.

- **Authorization model** — no OpenFGA changes; rules are org-scoped and attachment uses the existing `can_edit_config` on the agent. The [Authorization](../architecture/authz.md) doc gains a new "EgressRules Service" section enumerating the per-operation checks.

- **Tracing and Metering integration** — the [Tracing](../architecture/tracing.md) and [Metering](../architecture/metering.md) docs gain new sections describing the egress span shape (one span per request, attributes listed in [Egress Gateway — Observability](../architecture/egress-gateway.md#observability)) and the egress `COUNT` record. The receiving services need no code change beyond accepting the new attribute and label sets.

- **Notifications event types** — `egress_rule.updated` and `egress_rule_attachment.updated` event types do not exist. Published by the EgressRules service to org-scoped rooms; the Egress Gateway subscribes per org for which it has cached rules.

- **Secrets referential integrity** — two new internal RPCs. `Secrets.ResolveSecretExists(secret_id)` lets the EgressRules service validate a `secret_id` reference at rule create/update. `EgressRules.CountRulesReferencingSecret(secret_id)` lets the Secrets service refuse to delete a Secret while any active rule references it. Neither call recurses into the other.

- **Console** — the UI is specified in [Console — Egress Rules](../product/console/console.md#egress-rules) (rule CRUD with the matcher/effect/injected-header editor), the sidebar Egress group, and the agent-detail [Egress Rules](../product/console/console.md#agents) attachment section. None of it is built yet.

- **Terraform provider** — `agyn_egress_rule` and `agyn_egress_rule_attachment` resources do not exist in `agynio/terraform-provider-agyn`. Must be added with plan-time validation of the rule shape (matcher required fields, header value xor secret_id) and attachment uniqueness.

- **`agyn` CLI** — no commands exist. Add `agyn egress rule {create,list,get,update,delete,attach,detach}`.

## Acceptance Signal

- An organization owner creates an `EgressRule` (`matcher: { domain_pattern: "*.github.com", methods: ["GET"], ports: [443] }`, `effect: { inject: [{ name: "Authorization", scheme: "bearer", secret_id: "<github-token-secret>" }] }`) via the Console, the `agyn` CLI, and the Terraform provider — all three paths succeed and produce equivalent records.
- The same owner attaches the rule to an agent via all three entry points.
- An agent workload of that agent, with no `Authorization` env var or secret in its container, runs `curl https://api.github.com/repos/foo/bar` and receives a `200` with the GitHub data. The agent's container has no knowledge of the GitHub token — the request body and headers from inside the agent contain no `Authorization`, but the response is the authenticated one.
- An organization owner creates an `EgressRule` (`matcher: { domain_pattern: "*.openai.com" }`, `effect: { action: "deny" }`) and attaches it. An agent in that org running `curl https://api.openai.com/v1/models` receives `403` from the platform — the request never reaches OpenAI.
- An agent calls a destination not covered by any rule (e.g., `https://example.com/`). The gateway does not see the request — `tcpdump` on the gateway shows no traffic, and the agent receives a direct response from `example.com`.
- An agent attempts to connect to a cluster-internal service (e.g., another pod's IP, a ClusterIP service). The connection is refused by the workload's NetworkPolicy — the request never reaches the destination, regardless of any egress rule.
- A rule create attempt with `matcher.domain_pattern: "*.ziti"` (or `*.svc`, `*.cluster.local`, or any pattern overlapping `100.64.0.0/10`) is rejected at create time with a clear validation error.
- The [Tracing](../architecture/tracing.md) span with `egress.*` attributes is emitted for each handled request, visible in the trace viewer.
- A [Metering](../architecture/metering.md) `COUNT` record per request appears with labels `resource=egress`, `agent_id`, `host`, `outcome`.
- Rotating the secret referenced by an egress rule via the [Secrets](../architecture/secrets.md) service causes the next request (after the gateway's `SECRET_CACHE_TTL`) to use the new value. No restart of the gateway or the agent is needed.
- Attempting to delete a Secret referenced by an active `EgressRule.effect.inject` is rejected by the Secrets service with a clear error pointing to the referencing rule.
- An external runner hosting an agent pod receives the inline CA bytes in the workload spec, materializes the CA mount locally, and the agent in that pod completes the GitHub test above end-to-end identically to an in-cluster agent. The external runner's cluster has the Workload Egress NetworkPolicy installed at runner-install time (from the same template as the in-cluster k8s-runner) but no cert-manager, no trust-manager, no platform-specific webhook or operator.
- Editing a rule's `matcher.domain_pattern` propagates to the gateway within the documented 15-second window; subsequent requests evaluate against the new pattern.
- A `curl https://upgradable.example.com/` with `Upgrade: websocket` returns `426 Upgrade Required` from the gateway when a matching rule exists.
- cert-manager renews the Egress CA at the configured `renewBefore`; the `egress-ca` Secret reflects the new cert and key; restarting the gateway picks them up without manual intervention.

## Notes

- Default-allow at the egress layer is intentional. Strict deny-all semantics is deferred to a later optional layer ([NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) at the cluster network level explicitly blocking unmatched egress, or an opt-in wildcard egress mode); both can be composed on top of the rule-based egress without architectural rework.
- One rule per `(organization, matcher.domain_pattern)`. Methods, path, and ports inside the matcher. If finer-grained per-method or per-path policies on the same domain become a requirement, extend the schema by allowing a list of `(matcher, effect)` clauses inside one rule rather than allowing multiple rules per domain — keeps the OpenZiti intercept space clean (one Ziti service per rule).
- Egress CA distribution is uniform: orchestrator reads `tls.crt` from the cert-manager-managed `egress-ca` Secret and inlines the bytes in every workload spec. trust-manager is intentionally not used — the runner contract stays "K8s + the runner agent, nothing else."
- The workload-namespace NetworkPolicy is static infrastructure installed alongside the runner deployment, not per-workload runtime config. Cluster CIDRs are parameterized into the manifest at install time. CIDR changes are an operator action (Helm upgrade, Terraform apply); running workloads pick up the new policy immediately via the CNI.
- TLS interception breaks tools that hardcode CA paths, pin certificates, or use a custom DNS resolver that bypasses the Ziti sidecar. The product doc enumerates the limitation; users whose tooling falls outside the standard set must install the CA into their image's system trust store, avoid bypassing the system DNS, or avoid attaching rules for destinations called by those tools.
- WebSocket support, HTTP/3 / QUIC, OAuth-style automatic token refresh for injected secrets, multi-segment wildcard hostnames, per-rule conditions lists, finer-grained per-org egress gateway isolation, OpenZiti control-plane scale validation (binding many per-rule services), and automated migration from ENV-injected secrets are out of scope for v1 and tracked in [Open Questions](../open-questions.md).
- Wildcard hostname intercept (`*.github.com`) was validated against `openziti/ziti-edge-tunnel:v1.15.1` on a kind cluster — fresh subdomains are intercepted on first DNS query without leakage to `eth0`.
