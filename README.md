# Agyn

Product and architecture documentation for the Agyn AI agent orchestrator platform.

## How to Use This Documentation

### `/product` — Product Source of Truth

The desired state of the product. Declarative descriptions of capabilities, behaviors, and user experiences. No references to current implementation state, no transitional statements, no legacy caveats. When the product changes, this directory is updated to reflect the new target — not annotated with "currently X, will become Y."

### `/architecture` — Architecture Source of Truth

The desired state of the system. Declarative descriptions of all services, patterns, protocols, and contracts. No references to current implementation state, no transitional statements, no legacy caveats. When the architecture changes, this directory is updated to reflect the new target — not annotated with "currently X, will become Y."

### `/changes` — Pending Deltas

Gaps between the desired state described in `/product` or `/architecture` and current reality. Each file represents a single delta. When reality matches the desired state, the file is deleted. Git history preserves the full record.

### `/open-questions.md` — Unresolved Decisions

Topics that require further investigation and a decision. Specifications for areas with unresolved questions should be avoided. Raise an open question or wait for a decision instead of specifying behavior that may change.

## Structure

### [Product](product/) — Desired Product State

Target product definition. Describes how the product should work from the user's perspective. See [Product README](product/README.md) for the full list of product docs.

### [Architecture](architecture/) — Desired Architecture State

Target architecture of the platform. Describes how the system should work. See [Architecture README](architecture/README.md) for the full list of architecture docs.

### [Maps](maps/) — Cross-links

Navigation between product and architecture docs.

| Document | Description |
|----------|-------------|
| [Product to Architecture Map](maps/product-to-architecture.md) | Cross-links between product specs and architecture docs |

### [Operations](architecture/operations/) — CI/CD, Local Development, Configuration

How services are built, deployed, run locally, and configured. See [Architecture README](architecture/README.md#operations) for the full list.

### [Changes](changes/) — Pending Deltas

Gaps between desired state and current reality. See [Changes README](changes/README.md) for lifecycle rules.

### [Open Questions](open-questions.md)

Unresolved product and architectural decisions requiring discussion.
