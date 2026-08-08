# Platform Installation

## Overview

The platform installs as **one Helm chart**: one release, one version, one set of values, and no operator step in the middle. The chart deploys the platform's workloads and provisions the resources the release ships — see [Platform Resource Provisioning](platform-provisioning.md) for what gets created and as whom.

This document describes the packaging: what is in the chart, what an operator is expected to set, and what the release still expects to find already present.

## One Chart

The platform shipped as two charts — a control plane chart and a workload chart — because of a single ordering problem: the runner and the bundled apps mount a service token that only the running platform can mint, so installing them required a human to register those components between the two installs and copy a credential out of the Console.

Provisioning removes that step, and with it the reason for the split. What remains is ordering inside one release, which Helm expresses directly.

The chart contains:

| Layer | Contents |
|---|---|
| **Control plane** | The services behind the API — see [Services](../services.md) |
| **Workload layer** | The [runner](../runners.md) that executes agent workloads, and the [apps](../apps.md) bundled with the release |
| **Provisioning** | The controller that reconciles the release's declared resources, and the declarations themselves |
| **Dependencies** | Datastores and infrastructure the platform can bring with it, each individually declinable |

**The workload layer is version-locked to the release.** That is the intended relationship for components the platform ships: they are part of the release, not integrations with it. A third-party runner or app is registered through the API and deployed on its own schedule, and is unaffected — see [Runners — Enrollment](../runners.md#enrollment).

## Parameter Interface

Three principles decide whether a value belongs in the operator's surface:

**An operator declares intent, not wiring.** Service addresses, secret key references, and per-service environment are consequences of the values an operator sets, and are computed from them. They are not themselves an interface.

**Nothing is kept in sync by hand.** If two values must agree for the release to work, at most one of them is an input and the other is derived. A chart that validates operator-maintained consistency has already failed — the constraint it is checking should not have been expressible.

**A bundled dependency is a choice, not a default.** Every dependency the chart can deploy is either explicitly bundled or explicitly external. Nothing both ships and switches itself on, because a cluster that already runs it then gets a second copy.

The surface is organized in four tiers:

| Tier | Decides | Examples |
|---|---|---|
| **Platform** | How the install is addressed and who may sign in | Base domain, TLS, OIDC issuer and client |
| **Dependencies** | Bundled or external, per dependency, with connection details when external | Database, object store, authorization store, event bus, cache, container registry, OpenZiti controller |
| **Provisioning** | What the release declares the platform should contain | System organization slug, cluster administrators, shipped images, the runner, which bundled apps are enabled |
| **Operations** | Sizing and placement | Replica counts, resource requests, scheduling constraints, workload isolation |

The provisioning tier renders into the objects the [provisioning controller](platform-provisioning.md) reconciles: what a release provisions is a property of the values, not of the binary that applies them. It is also the one tier an operator must fill in, because a release that declares no [cluster administrator](platform-provisioning.md#cluster-administrators) installs a platform nobody can administer.

## Declarations and Their Schemas

The release carries both the custom resource definitions the [provisioning controller](platform-provisioning.md) reads and the declarations themselves. Shipping them together is what makes the install one step, and it imposes two requirements on how they are packaged.

**A schema is established before anything declared against it.** A declaration whose kind is not yet registered is rejected by the API server rather than queued, so within a release the definitions are applied first — an ordering the release must guarantee rather than assume.

**A schema travels with the release that needs it.** Kinds gain fields as the platform gains capabilities, so a definition that installs once and never updates would pin every declaration to the shape it had on the day the cluster was first built, and put an out-of-band step back into the upgrade path.

**A definition is never deleted in order to be replaced.** Definitions are cluster-scoped and own every object of their kind: removing one destroys the declarations it describes. An upgrade updates them in place.

Because definitions are cluster-scoped and may be governed separately from the workloads that use them, installing them is a choice like any other bundled dependency — an operator managing them out of band turns it off and supplies them, exactly as with the values the chart otherwise generates.

Uninstalling removes the declarations, and the [removal semantics](platform-provisioning.md#principles) then apply: provisioned resources are orphaned and left in place, while cluster administrator grants are revoked. Reinstalling re-declares both.

## Secrets

Two kinds, distinguished by who owns the value.

**Operator-provided.** Credentials for things the platform did not create — external databases, object storage, the OIDC client. The chart consumes these by reference to an existing Secret and never requires the value in plaintext in a values file.

**Chart-generated with a stable identity.** Values the platform mints for itself and must keep across upgrades: the secrets encryption key, the egress CA, and the [platform admin bootstrap token](platform-provisioning.md#the-platform-admin-identity). Each is generated on first install and reused afterwards.

Reuse-on-upgrade requires reading what already exists in the cluster, which a renderer that runs without cluster access cannot do — it finds nothing and generates a new value, rotating the old one out from under everything that depends on it. Every generated value therefore also has a mode where the operator owns it and the chart only consumes it. That mode is what a GitOps install uses.

## Ordering

There is none, and that is deliberate.

Installing applies a set of objects: workloads, and the [declarations](platform-provisioning.md) of what the release provisions. Nothing in the chart sequences them — no dependency graph, no ordered waves, no install-time wait — because every precondition provisioning has becomes true at a moment the chart cannot predict:

| Precondition | Becomes true |
|---|---|
| A service's schema is current | Inside that service's own startup — [migrations](database-migrations.md) run in-process, so a service that is serving is a service that is migrated |
| The platform's overlay services exist | When each service [self-enrolls](../openziti.md#service-identity-self-enrollment) at startup, which is also what the overlay's named-service policies wait on |
| An external dependency is reachable | On its own schedule, outside the release entirely |

Waiting for workloads to report Ready would not answer any of these. A Ready Gateway pod still fails calls that depend on a service which has not yet self-enrolled, and no readiness probe in this release says anything about a controller or database that is not in it.

So the release does not decide when the platform is ready. It applies what it declares, and reconciliation converges behind it — see [Provisioning — Reconciliation](platform-provisioning.md#reconciliation). Installing returns once the objects are applied; whether the platform is provisioned is a question its objects answer, continuously, rather than a property of whether a command finished.

The workload layer converges the same way. The runner and the bundled apps mount Secrets the controller creates, so on a first install they may start before their credential exists and retry until it appears. Nothing is lost and no operator action is required. For the first minutes of a fresh install some workloads are waiting on something the same release is still producing, and that is the expected shape rather than a failure.

## Outside the Chart

The chart installs the platform, not the cluster it runs on. A release expects to find:

- Kubernetes, with the service mesh and ingress the platform's traffic model assumes
- DNS for the platform's domain, and a certificate authority or issuer for its TLS
- An OIDC provider for [user authentication](../authn.md#user-authentication-oidc)
- Optionally, node-level workload isolation — a node-level change that must precede the install, since applying it restarts the container runtime

## Upgrade

An upgrade moves one release. The workloads are rewritten from the new charts, and provisioning re-runs — create-if-absent, so it adds what a release introduced and leaves everything else alone, including anything an operator has edited.

The [Local Bundle](local-bundle.md#upgrade-model) applies this to a running VM, and distinguishes it from replacing the image underneath.

## Related Architecture

- [Platform Resource Provisioning](platform-provisioning.md)
- [Services](../services.md)
- [OpenZiti Integration — Static Policies](../openziti.md#static-policies)
- [Local Bundle](local-bundle.md)
- [Local Development](local-development.md)
- [CI/CD](ci-cd.md)
