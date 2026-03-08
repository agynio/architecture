# Tracing

## Overview

The Tracing service ingests and queries tracing data. It implements an **extended version of the OpenTelemetry protocol** to support real-time in-progress events — standard OpenTelemetry only captures completed spans, while Agyn needs visibility into ongoing agent operations.

## Responsibilities

- **Ingest** tracing data from agents and platform services.
- **Query** traces and spans for observability UI.
- **Real-time events** for in-progress operations (e.g., an agent currently processing a tool call).

## Integration

Tracing is an **optional** dependency for agents. Agents can run without a tracing server configured.

## Status

The previous tracing implementation (tracing SDK/server/UI stack) was removed from the monolith. A new implementation is planned as a standalone service.
