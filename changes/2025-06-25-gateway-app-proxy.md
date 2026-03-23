# Gateway App Proxy

## Target

- [Gateway — App Proxy](../architecture/gateway.md#app-proxy)

## Delta

The Gateway has no generic app proxy mechanism. No `/apps/{slug}/{method}` routing. No app slug resolution via Apps Service. No OpenZiti dialing to app services.

## Acceptance Signal

- Gateway routes requests matching `/apps/{slug}/{method}` to the appropriate app via OpenZiti.
- Gateway resolves app slug via Apps Service `GetAppBySlug`.
- Caller identity is propagated to the app in forwarded request metadata.
- Adding a new app does not require Gateway code changes.

## Notes

- Depends on: Apps Service (`2025-06-25-apps-concept.md`), OpenZiti app policies (`2025-06-25-openziti-app-policies.md`).
