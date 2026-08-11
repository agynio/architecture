# One Ingress Port Value, Not Nineteen

## Target

- [agyn-cli — Upgrade](../architecture/agyn-cli.md#upgrade)
- [Local Bundle — Networking](../architecture/operations/local-bundle.md#networking)

## Delta

The port a deployment is reached on is spelled out nineteen times in the `agyn-platform` release's values — eighteen service URLs (`oidcAuthority`, `origin`, `corsAllowedOrigin`, `mediaProxyUrl`, `websocketUrl`, the four app origins, and the issuer repeated per service) plus `keycloak.wiring.ingressHostPort.port`. Nothing derives them from one another.

The chart says why, in a comment on `global`:

> No `global.ingress` here yet. `service-base` 0.1.5 templates env values so a subchart can derive its own hostnames from one domain — agents-orchestrator already builds `IMAGE_PROXY_HOST` that way — but 19 of the 36 charts this umbrella pulls still embed `service-base` 0.1.4, whose schema declares `global` `additionalProperties: false`. Setting it fails their render, so the domain stays per-service until those charts are released against 0.1.5.

Desired state: one value — a domain and a port — that every URL in the umbrella derives from. `global.ingress` is the natural home, and reaching it requires the 19 subcharts still on `service-base` 0.1.4 to be released against 0.1.5 first.

Until then each consumer sets nineteen values that must agree, and nothing detects them disagreeing. `agyn local upgrade` rewrites all nineteen from the release's own values on every run, which closes the local bundle's exposure without closing the gap: a deployment configured by hand, or by any other tool, can still be given a set that disagrees with itself.

Not in scope: the OpenZiti advertised ports and the `istio-ingressgateway` Service port, which belong to their own releases and move separately.

## Acceptance Signal

- A deployment's ingress hostname and port are set once and every URL the umbrella renders follows.
- No consumer of the chart enumerates per-service URLs to change a port.
- A values file that names two different ports is rejected at render time rather than installed.
- No subchart in the umbrella embeds `service-base` 0.1.4.

## Notes

- The 19-subchart release against `service-base` 0.1.5 is the whole cost, and it is mechanical. Nothing else blocks the value.
- This is what made `agyn local upgrade` slow rather than merely wrong: the values named the baked port, the upgrade re-applied it, and a second pass patched the workloads back — two rollouts of most of the platform with a crash-loop window between them. The CLI now corrects the values instead, and a values-only correction on a second VM took 20 seconds against eleven minutes.
- A production install never noticed, because it serves 443 and the port is invisible in every URL. The local bundle is the only deployment where it varies, which is why nineteen copies of it went unremarked.
