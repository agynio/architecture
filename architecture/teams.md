# Teams

## Overview

The Teams service manages team resources — the configuration entities that define what agents, tools, workspaces, and MCP servers are available.

This is a **control plane** service. It stores desired state; other services reconcile toward it.

All resources are scoped to a [tenant](tenancy.md). Every API call requires an authenticated identity with a resolved tenant. See [Authentication](authn.md).

## External API

Defined in `agynio/api` at `openapi/team/v1/openapi.yaml`. Released to GHCR as OpenAPI specs.

## Resources

| Resource | Description | CRUD |
|----------|-------------|------|
| **Agents** | Agent definitions: LLM config, behavior, system prompts | ✓ |
| **Tools** | Tool definitions with type classification | ✓ |
| **MCP Servers** | MCP server configs: command, env vars, connection settings | ✓ |
| **Workspace Configurations** | Workspace templates: image, env, volumes, platform | ✓ |
| **Memory Buckets** | Memory bucket definitions with scope and config | ✓ |
| **Attachments** | File attachments associated with team entities | Create, Read, Delete |

All list endpoints use cursor-based pagination.

## Entity Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |
