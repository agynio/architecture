# Caller-Set Sandbox Idle Timeout

## Target

- [Resource Definitions — Sandbox](../architecture/resource-definitions.md#sandbox)
- [Organizations — Sandbox Settings](../architecture/organizations.md#sandbox-settings)
- [Sandboxes — Choosing an idle timeout](../product/sandboxes/sandboxes.md#choosing-an-idle-timeout)
- [agyn-cli — Sandbox Idle Timeout](../architecture/agyn-cli.md#sandbox-idle-timeout)

## Delta

`Sandbox.idle_timeout` exists and is enforced, but no caller can set it: the Agents service snapshots `organization.sandbox_default_idle_timeout` at creation and there is no request field, no CLI flag, and no local preference. Every sandbox in an organization therefore gets one number regardless of what it runs or why it was started.

The semantics need no change. Idle already means *nothing attached* — each attached TTY session drives `TouchWorkload` every 10s, so the clock starts at the last detach, and `SYNC` sessions deliberately do not count ([Terminal Proxy — Sandbox Activity Reporting](../architecture/terminal-proxy.md#sandbox-activity-reporting)). What is missing is control over the value:

- **`CreateSandbox` accepts `idle_timeout`.** Validated against the organization's ceiling; omitted, the organization's default applies. Stored on the sandbox at creation like TTL, and not re-read afterwards.
- **A new organization setting `sandbox_max_idle_timeout`**, defaulting to the platform maximum. The default and the maximum must be separate settings: if one field served as both, the default would also be the most expensive option on offer.
- **A request above the ceiling is rejected, naming it** — not clamped. A silently reduced timeout is a number the engineer never sees and plans around wrongly.
- **`agyn sandbox start --idle-timeout DURATION`**, and a per-profile `sandboxIdleTimeout` set through `agyn profile set --sandbox-idle-timeout`. Resolution is flag → profile → organization default. The profile default is a local preference, not policy: the server validates it like any other request value.
- **`agyn sandbox list` shows the idle timeout beside the remaining TTL.** Two bounds govern a sandbox's life; showing one and hiding the other is what makes the second a surprise.

TTL is unchanged and remains the backstop: a long idle timeout keeps a sandbox alive while nobody is attached, but nothing outlives its TTL.

## Acceptance Signal

- `agyn sandbox start --idle-timeout 4h` produces a sandbox that survives four hours with nothing attached and stops after it.
- `agyn profile set cloud --sandbox-idle-timeout 2h` makes subsequent `agyn sandbox start` invocations on that profile create 2h sandboxes; `--idle-timeout` still wins per invocation; switching profiles switches the default.
- A request above `sandbox_max_idle_timeout` fails with an error naming the ceiling, and no sandbox is created.
- Lowering `sandbox_default_idle_timeout` or `sandbox_max_idle_timeout` afterwards leaves existing sandboxes running on their stored values.
- A sandbox with a 4h idle timeout and 1h of TTL remaining is terminated by TTL.
- `agyn sandbox list` shows both bounds per row.

## Notes

- The idle timeout stays **immutable after creation**, matching TTL. Letting an owner extend a live sandbox ("I am about to be idled out mid-call") is a reasonable follow-up and deliberately not specified here.
- "Idle means no open connection" holds for terminal sessions only. A `SYNC` session is an open connection and still does not count, for the reason already recorded: a laptop left syncing would keep a sandbox alive indefinitely, defeating the timeout and the metering it bounds. Changing that is a separate decision.
- Environments carry no idle timeout. Per-environment defaults — a `gpu` environment wanting a shorter one than `cpu-small` — remain unaddressed; this change puts the control on the sandbox, where the person who knows the answer is.
