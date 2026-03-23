# OpenZiti App Policies

## Target

- [OpenZiti — App Identity Lifecycle](../architecture/openziti.md#app-identity-lifecycle)

## Delta

No OpenZiti static policies exist for apps. No `CreateAppIdentity` RPC in Ziti Management. No OpenZiti service creation per app.

## Acceptance Signal

- Static policies `apps-dial-gateway`, `apps-bind`, `gateway-dial-apps` are provisioned at bootstrap.
- Ziti Management exposes `CreateAppIdentity` RPC.
- App registration creates an OpenZiti service for the app (e.g., `app-reminders`).
- Gateway can dial app services via OpenZiti.

## Notes

- Depends on: Ziti Management service changes, bootstrap Terraform updates.
