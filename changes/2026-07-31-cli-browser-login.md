# Browser Sign-In for the CLI

## Target

- [CLI Login](../architecture/cli-login.md)
- [Product — CLI Login](../product/cli-login/cli-login.md)
- [agyn-cli — Authentication](../architecture/agyn-cli.md#authentication)
- [agyn-cli — Profiles](../architecture/agyn-cli.md#profiles)
- [Gateway — Unauthenticated Methods](../architecture/gateway.md#unauthenticated-methods)
- [Users — CLI Login Requests](../architecture/users.md#cli-login-requests)
- [API Tokens — Token Model](../architecture/api-tokens.md#token-model)
- [Console — CLI Login Approval](../architecture/console.md#cli-login-approval)
- [Product — Console — Standalone Routes](../product/console/console.md#standalone-routes)

## Delta

`agyn auth login` exists as a stub that prints "Not yet implemented. Place token in `~/.agyn/credentials`". Signing in means opening the Console, creating an [API token](../architecture/api-tokens.md), copying it out of a dialog that shows it once, and feeding it to `agyn auth set-token`. This is the first thing a new user does, and on the cloud it is the first thing a signup does.

Everything the flow needs on the CLI side already exists: [profiles](../architecture/agyn-cli.md#profiles) with a gateway, organization and CA; per-profile tokens in `~/.agyn/credentials`; `agyn profile set|use|select`; `agyn auth set-token` and `whoami`. What is missing is the platform half — nothing can issue a credential to a client that does not yet have one:

- The platform issues no credential to a CLI. `agyn auth create-token` requires a credential to call, so it cannot bootstrap one.
- No login request resource exists. The [Users](../architecture/users.md) service owns API tokens but has nothing that lets a browser session mint one on a terminal's behalf.
- Every Gateway method is authenticated, so there is no endpoint a credential-less client can call.
- The Console has no approval surface — no route where a user confirms a sign-in a CLI asked for.
- API tokens have no `origin`, so a token a machine is using is indistinguishable in the list from one made by hand for CI.
- A profile must be configured by hand before it can be used (`agyn profile set NAME --gateway-url URL`); there is no `--host`, and nothing creates a profile as a side effect of signing in.

Desired state: `agyn auth login [--host HOST]` opens a browser, the user approves once, and the CLI is signed in — a platform-brokered [device authorization grant](https://datatracker.ietf.org/doc/html/rfc8628) that ends in an ordinary API token stored in the active profile. The identity provider still performs the human authentication, inside the browser session, so a deployment needs no new IdP client and the CLI needs no refresh-token handling.

Required by the above:

- Two unauthenticated Gateway methods (`StartCLILogin`, `PollCLILogin`) with their own rate limits — the platform's first, and confined to obtaining a credential.
- A `cli_login_requests` resource in Users, with the issued token sealed to the device code so the row never holds a usable credential at rest.
- A standalone `/cli-login` Console route rendering the approval screen: the confirmation code to compare against the terminal, the requesting machine, its address, and Deny as a peer of Approve.
- An `origin` field on API tokens, and CLI-issued tokens auto-named for the requesting machine.
- `--host` on `agyn auth login`, resolving to the existing `gatewayUrl` field, and a profile created and made current by a successful sign-in.

## Acceptance Signal

A user who has never touched the platform runs `agyn auth login --host <cloud>`, signs in and approves in the browser that opens, and the terminal reports the identity it signed in as — with no token displayed anywhere, and no profile configured beforehand.

The code printed in the terminal is the code shown in the browser, and the approval screen names the machine that asked and where the request came from.

Denying the request leaves the CLI signed out and issues nothing. Letting it sit for the TTL ends it with an instruction to run the command again.

The issued token appears in the Console's API tokens list named for the machine (`CLI on <hostname>`); revoking it there stops that machine's next request.

On a host with no browser, the same command prints a URL and code that complete the flow from a browser on another machine.

A machine signed in to a cloud platform and running a local VM switches between them with `agyn profile use` and does not re-authenticate for either.

`StartCLILogin` and `PollCLILogin` are the only Gateway methods reachable without a credential, and both are rate-limited.

## Notes

- Alternatives considered and rejected in [CLI Login — Why the Platform Brokers It](../architecture/cli-login.md#why-the-platform-brokers-it): loopback redirect + PKCE and an IdP-side device grant both require an OIDC client registered per deployment at an identity provider the platform does not own, and both yield a short-lived token the platform can neither show nor revoke.
- The residual risk of any device grant is a user approving a request they did not start. The mitigation is the code comparison plus the requester metadata on the approval screen — which is why the code is shown even though the common path carries it in the URL.
- CI keeps using `agyn auth set-token` and `AGYN_TOKEN`. The browser flow is for humans, and a deployment can disable it entirely. The browser flow deliberately does not add a `--token` flag: `set-token` reads from stdin so a token never enters shell history, and a flag would undo that.
- The [Profiles](../architecture/agyn-cli.md#profiles) section documents what already ships (added by [multiple local VMs](2026-07-31-multiple-local-vms.md)); the only delta in it is `--host` and sign-in-creates-a-profile.
- The user-facing docs in `agyn/platform/docs/build-extend/agyn-cli.md` describe a different CLI — `agyn login --gateway URL` storing to `~/.config/agyn/credentials`, and a browser flow that already works. None of those exist. That is a stale-doc delta of its own, not covered here.
