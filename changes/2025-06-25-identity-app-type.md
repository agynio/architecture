# Identity App Type

## Target

- [Identity](../architecture/identity.md)
- [Authentication](../architecture/authn.md)

## Delta

The Identity service does not include `app` as an identity type. Authentication docs do not describe app identity.

## Acceptance Signal

- Identity service `identity_type` enum includes `app`.
- Apps Service can register identities with `identity_type: app`.
- Gateway and Ziti Management can resolve and propagate app identities.

## Notes

- Additive change — no existing identity types are affected.
