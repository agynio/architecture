# CLI Login

## Purpose

Signing the [`agyn` CLI](../../architecture/agyn-cli.md) in should cost one command and one click. Before this, it cost a detour: open the Console, find the API tokens page, create a token, copy it before the dialog closes, then get it into the CLI without leaving it in shell history. That is four chances to give up, and it is the first thing a new user does.

`agyn auth` replaces the detour. The command opens a browser, the user confirms, and the CLI is signed in. Nobody sees a token.

```
$ agyn auth login --host agyn.cloud

  Confirmation code: WDJB-MJHT

  Opening https://console.agyn.cloud/cli-login?code=WDJB-MJHT
  Waiting for confirmation in the browser...

✓ Signed in as @vitalii (Vitalii Valkov) on agyn.cloud
  Organization: acme (the only one you belong to)
  This sign-in expires on 2026-10-29. Run `agyn auth` again to renew.
```

## User Stories

- As a new user, I want to sign the CLI in with one command so I can start using the platform without hunting for a token in the UI.
- As a user with a cloud account and a local VM, I want each to be a named context I can switch between so I do not have to re-authenticate every time I change target.
- As a user approving a sign-in, I want to see which machine asked and a code I can compare against my terminal so I never approve a request that is not mine.
- As a user, I want to see which sign-ins exist and end any of them so a laptop I no longer have loses access.
- As a user on a remote server with no browser, I want to complete the same flow from a browser on another machine.

## Flow

1. The user runs `agyn auth login`, optionally naming a platform with `--host`.
2. The CLI displays a confirmation code and opens the browser.
3. The browser lands on the platform's approval page. If the user is not signed in, the ordinary sign-in happens first — including first-ever sign-in, which creates the account.
4. The approval page shows the request. The user compares the code with the one in the terminal and approves.
5. The terminal confirms who was signed in, which organization is active, and when the sign-in expires.

Steps 2–4 take a few seconds. The CLI waits for up to 10 minutes and then gives up with an instruction to run the command again.

## Command Surface

| Command | Description |
|---------|-------------|
| `agyn auth login [--host HOST]` | Sign in through the browser. With no `--host`, the current context's platform. `agyn auth` on its own does the same |
| `agyn auth login --no-browser` | Print the URL and code instead of opening a browser |
| `agyn auth set-token` | Store an [API token](../../architecture/api-tokens.md) instead — for CI and scripts, where no browser exists. The token is typed or piped in, never passed as an argument, so it stays out of shell history |
| `agyn auth whoami` | Who is signed in, where, in which organization, and when the sign-in expires |
| `agyn auth logout [--revoke]` | Forget the stored credential. `--revoke` also ends it platform-side, so a copied credential stops working |

See [agyn CLI — Authentication](../../architecture/agyn-cli.md#authentication) for the full command group, including the token management commands.

## Approval Screen

The approval page is the one moment a human decides, so it shows what a person needs in order to decide — and nothing that reads as ceremony.

| Element | Purpose |
|---------|---------|
| **Confirmation code** | Displayed prominently, to be compared with the terminal. The page asks for that comparison in words: "Only continue if this code matches the one in your terminal" |
| **Machine** | The name of the machine that asked, and the CLI version |
| **Where from** | The network address the request came from, with its approximate location |
| **When** | When the request was made, and when it expires |
| **Approve / Deny** | Deny is a peer of Approve, not a quiet link. The page states plainly: if you did not start this, deny it |

A code that does not match the terminal means the request came from somewhere else, and that is the only case this screen exists to catch. After Approve, the page confirms and tells the user to return to their terminal. After Deny, it says the request was refused and nothing was issued.

Reaching the page with no code — by opening it directly — shows a field to type the code from the terminal, which is the same flow the [headless case](#no-browser-on-the-machine) uses.

## Contexts

A user launching on the cloud usually also has a local platform running. Both are addressable, and a sign-in belongs to one of them:

```sh
agyn auth login --host agyn.cloud   # sign in to the cloud, make it current
agyn local start                    # a local platform, provisioned as its own profile
agyn profile use local              # point the CLI at it
agyn profile use agyn.cloud         # and back — no re-authentication
```

Profiles already exist and already hold a gateway, an organization and a CA; what a browser sign-in adds is a profile that can be *created by signing in* rather than configured by hand first. `agyn auth whoami` names the current one. Starting a local VM provisions its profile but does not steal the current one unless nothing is selected yet — it prints how to switch instead. See [agyn CLI — Profiles](../../architecture/agyn-cli.md#profiles).

## No Browser on the Machine

On a server reached over SSH, or anywhere a browser cannot be opened, the CLI detects it and prints the URL and code instead of trying:

```
  Confirmation code: WDJB-MJHT

  Open https://console.agyn.cloud/cli-login and enter the code above.
  Waiting for confirmation...
```

The user opens it from a browser on any machine — a laptop, a phone — signs in there, types the code, and approves. The terminal completes on its own. `--no-browser` forces this mode when detection is not enough.

## Managing Sign-Ins

A browser sign-in appears in the Console's API tokens list (user menu → [API Tokens](../console/console.md#user-menu)), named for the machine that asked (`CLI on vitalii-mbp`), so the answer to "which of these is my laptop?" is readable at a glance. Revoking it there ends that machine's access immediately.

Sign-ins expire on their own — 90 days by default. The CLI warns during the last week and `agyn auth` renews without further ceremony.

## Constraints

- One sign-in per context per machine. Running `agyn auth` again on the same machine replaces the stored credential; the previous one stays valid until it expires or is revoked, so it should be revoked when a machine is decommissioned.
- The credential a browser sign-in produces is an ordinary API token with the user's full permissions. There is no narrower CLI-only scope.
- Approval requires a signed-in browser session on the platform being signed in to. Signing the CLI in to a platform the user has no account on is not possible without first being able to sign in to it.
- CI and other unattended environments use `--token` or the `AGYN_TOKEN` environment variable. The browser flow is for humans.

## Related Architecture

- [CLI Login](../../architecture/cli-login.md) — the flow, the grant model, and its security properties
- [agyn CLI](../../architecture/agyn-cli.md) — command group, profiles, credential storage
- [API Tokens](../../architecture/api-tokens.md) — the credential the flow issues
- [Authentication](../../architecture/authn.md) — how CLIs resolve credentials
- [Console](../console/console.md) — the approval page and the API tokens list
