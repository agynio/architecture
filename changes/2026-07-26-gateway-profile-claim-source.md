# Configurable Profile Claim Source at the Gateway

## Target

[Authentication — User Authentication (OIDC)](../architecture/authn.md#user-authentication-oidc)

## Delta

The Gateway obtains provisioning-time profile claims only from the IdP's UserInfo endpoint. The endpoint and the claim names are both fixed.

The desired state adds configuration:

- `profile_source` selects where claims are read from — `userinfo` (default) or `token`.
- `claims.name`, `claims.email`, `claims.picture`, and `claims.preferred_username` name the claim holding each profile field, for both sources.

Without the `token` source, a deployment cannot use an IdP that issues audience-restricted access tokens. Such a token is required for the Gateway to validate a JWT, but the OIDC UserInfo endpoint rejects any token carrying an `aud` claim, so first login fails while returning users continue to work.

## Acceptance Signal

With `profile_source = token` and an IdP that issues an audience-restricted access token carrying profile claims, a first-time user is provisioned with name and email populated, and the Gateway makes no request to the UserInfo endpoint.

With `profile_source` unset, behaviour is unchanged: claims come from the UserInfo endpoint.

A profile field whose configured claim is absent is provisioned empty rather than failing the request.

## Notes

Applies to any IdP implementing [RFC 8707 Resource Indicators](https://datatracker.ietf.org/doc/html/rfc8707); the restriction is in the OIDC UserInfo specification, not in a particular product. IdPs that leave the access token unrestricted are unaffected and continue to work with the default source.

Configuring the IdP to include the profile claims in the access token is a deployment-side concern.
