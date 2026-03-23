# Apps Concept

## Target

- [Apps](../architecture/apps.md)
- [Apps Service](../architecture/apps-service.md)

## Delta

The Apps concept does not exist in the current implementation. No Apps Service, no app identity type, no app registration or enrollment flow.

## Acceptance Signal

- Apps Service is deployed and operational.
- An app can be registered via cluster admin API (or Terraform).
- Registered apps receive enrollment tokens and can enroll via OpenZiti.
- App profiles are resolvable by Chat for message display.

## Notes

- Depends on: cluster admin authorization (see `2025-06-25-cluster-admin.md`), OpenZiti app policies, Ziti Management `CreateAppIdentity` RPC.
- The Channels concept will eventually be subsumed by Apps. Channels continue to work as-is until migration.
