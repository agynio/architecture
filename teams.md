# Teams

## Overview

The Teams service manages team resources — the configuration entities that define what agents, tools, workspaces, and MCP servers are available.

This is a **control plane** service. It stores desired state; other services reconcile toward it.

## External API

Defined in `agynio/api` at `openapi/team/v1/openapi.yaml`. Released to GHCR as OpenAPI specs.

## Resources

### Agents

Agent definitions including LLM configuration, behavior settings, and system prompts.

Key configuration fields: `model`, `systemPrompt`, `debounceMs`, `whenBusy`, `processBuffer`, `sendFinalResponseToThread`, `summarizationKeepTokens`, `summarizationMaxTokens`, `restrictOutput`, `name`, `role`.

CRUD operations: Create, Read (list with pagination, get by ID), Update, Delete.

### Tools

Tool definitions with type classification.

CRUD operations: Create, Read (list with pagination, get by ID), Update, Delete.

### MCP Servers

MCP server configurations including command, environment variables, and connection settings.

CRUD operations: Create, Read (list with pagination, get by ID), Update, Delete.

### Workspace Configurations

Workspace templates defining image, environment, volumes, and platform settings.

Includes: environment variables, volume configurations, platform selection (`linux/amd64`, `linux/arm64`).

CRUD operations: Create, Read (list with pagination, get by ID), Update, Delete.

### Memory Buckets

Memory bucket definitions with scope and configuration.

CRUD operations: Create, Read (list with pagination, get by ID), Update, Delete.

### Attachments

File attachments associated with team entities.

CRUD operations: Create, Read (list with pagination, get by ID), Delete.

## Data Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |
