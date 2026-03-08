# Gateway

## Overview

The Gateway exposes platform methods for external usage. It is the single entry point for external clients (web app, mobile app, integrators) to interact with the platform.

## Responsibilities

- Route external API requests to internal services.
- Translate between external (OpenAPI/REST) and internal protocols.
- Validate requests and responses against OpenAPI specs.
- Handle authentication at the boundary.

## Classification

The Gateway is a **data plane** service — it carries live API traffic.

## Current Implementation

The gateway (`agynio/gateway`, Go) currently:
- Serves the Team Management API (`/team/v1/`) with OpenAPI request validation (and optional response validation).
- Uses `oapi-codegen` for server stub generation from the OpenAPI spec.
- Proxies remaining routes (`/api/*`, `/health`) to the upstream platform-server.
- Applies CORS, request ID, and recovery middleware.
