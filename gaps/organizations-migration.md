# Gap: Tenants → Organizations Migration

## Context

The architecture has been updated to replace the multi-tenancy isolation model with organizations and ReBAC. See [Organizations](../architecture/organizations.md).

## Changes Required

### Tenants Service → Organizations Service

- Rename service: `agynio/tenants` → `agynio/organizations`.
- Rename database table: `tenants` → `organizations`.
- Rename all API methods: `CreateTenant` → `CreateOrganization`, etc.
- Update proto definitions in `agynio/api`.

### Agents Service

- Rename `tenant_id` → `organization_id` on `agents` and `volumes` tables.
- Sub-resources (MCPs, Skills, Hooks, ENVs, InitScripts, Volume Attachments) inherit org scope through parent — no `organization_id` column unless denormalized for query performance.
- Remove `tenant_id` filter from sub-resource queries; resolve org through parent chain.
- Update proto definitions.

### LLM Service

- Add `organization_id` to `llm_providers` and `models` tables (replacing `tenant_id`).
- Update proto definitions.

### Secrets Service

- Add `organization_id` to `secret_providers` and `secrets` tables (replacing `tenant_id`).
- Update proto definitions.

### Chat Service

- Add `organization_id` to chat records (replacing `tenant_id`).
- Update proto definitions.

### Channels

- Add `organization_id` to channel configuration (replacing `tenant_id`).
- Update proto definitions.

### Threads Service

- Remove `tenant_id` from threads, messages, and message_recipients tables.
- Access controlled via ReBAC on the thread resource.

### Files Service

- Remove `tenant_id` from file metadata.
- Remove `tenant_id` prefix from S3 object keys.
- Access controlled via ReBAC on the thread that references the file.

### Agent State Service

- Remove `tenant_id` from agent state records.
- Access controlled via ReBAC — owned by agent (`agent_id`).

### Runner

- Remove `tenant_id` label from workloads.
- Workload access controlled via ReBAC — owned by agent.

### Gateway

- Remove `tenant_id` header validation from request pipeline.
- Remove `x-tenant-id` from gRPC metadata propagation.
- Gateway authenticates identity only; organization membership is not validated at the gateway level.

### Agents Orchestrator

- Remove `tenantId` from `CreateAgentIdentity` calls.
- Update workload spec assembly to remove tenant context.

### OpenZiti / Ziti Management

- Remove `tenant_id` from identity mappings in PostgreSQL.
- Remove `tenant-<tenantId>` from role attributes on OpenZiti identities.
- Update `ResolveIdentity` response to return `(identity_id, identity_type)` without `tenant_id`.
- Remove `x-tenant-id` from internal identity propagation metadata.

### Authorization Model (OpenFGA)

- Rename `tenant` type → `organization` type in the authorization model DSL.
- Update all relationship tuples referencing `tenant:` to `organization:`.
- Update model tests.
- Deploy new model version via Terraform.

### Notifications

- No structural changes. Room names are based on resource IDs (no tenant prefix).
