# Gateway

## Overview

The Gateway exposes platform methods for external usage. It is the single entry point for external clients (web app, mobile app, integrators) to interact with the platform.

## Responsibilities

- Route external API requests to internal services.
- Translate between external (OpenAPI/REST) and internal (gRPC) protocols.
- Handle authentication and authorization at the boundary.

## Classification

The Gateway is a **data plane** service — it carries live API traffic and does not manage desired state or perform reconciliation.
