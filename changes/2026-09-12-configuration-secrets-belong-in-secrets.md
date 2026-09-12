# Configuration Secrets Belong in the Secrets Service

## Target

- [Apps — Secret Values](../architecture/apps.md#secret-values)
- [Apps — Agyn Keywords](../architecture/apps.md#agyn-keywords)
- [Apps — Compatibility](../architecture/apps.md#compatibility)
- [Apps Service — API](../architecture/apps-service.md#api)
- [Apps Service — Installation Resource](../architecture/apps-service.md#installation-resource)
- [Console — Configuration Forms](../product/console/console.md#configuration-forms)
- [Open Questions — Installation Configuration Secrets](../open-questions.md#installation-configuration-secrets)

## Delta

### `x-agyn-secret` promised protection the platform does not provide

[App Configuration Schema](2026-08-19-app-configuration-schema.md) had the Apps Service omit `x-agyn-secret` values from every read path but `GetInstallationConfiguration`, and report their names in `secret_keys_set` so the Console could draw **Set / Replace**.

The value was never actually protected. It is submitted in plain text on install and update, stored in plain text in the Apps Service database, and returned in full to the app. Hiding it from the admin who typed it — while it sits unprotected in three other places — is an appearance, and an appearance in a credential path is worse than none: it invites everyone downstream to assume a guarantee that is not there.

The appearance was not free. It bought a redaction rule on every method returning an installation, a merge-on-write rule so a read-modify-write would not erase the value, an app lookup on every installation read to learn which keys to hide, and a compatibility rule forbidding an app from ever clearing the marker. All of it existed to maintain the illusion.

### The keyword is reserved rather than redefined

Keeping `x-agyn-secret` as a display hint would burn the name. When a property can genuinely name a [Secret](../architecture/secrets.md), that property is what `x-agyn-secret` should mean — and any app that had adopted it for masking would need migrating, with a window where the same keyword means two things.

So the keyword is **rejected at report time**, with an error naming the alternative. Masking moves to `format: password`, which is standard JSON Schema, says only how to draw the field, and promises nothing about protection. It is the one honored format that constrains no value, so gaining it is a compatible change where gaining `uuid` or `email` is not.

### What is still missing

A credential should not be a value in the installation object at all. It should name a [Secret](../architecture/secrets.md), the way an agent ENV already does, so the value lives in the service built to hold it and the installation carries a reference. That is not designed yet — the open questions above are the ones to answer first.

## Acceptance Signal

- `Installation.secret_keys_set` does not exist, in the proto or on any response.
- A reported schema using `x-agyn-secret` is rejected, and the error names `format: password` as the way to mask a field.
- `format: password` is honored, validates nothing, and gaining it on an existing property is a compatible change.
- `GetInstallation`, `GetInstallationBySlug`, `ListInstallations`, and `GetInstallationByIdentityId` return `configuration` whole, secret-marked properties included.
- `UpdateInstallation` replaces the stored configuration with what was submitted; there is no merge of omitted properties.
- Installation reads do not resolve the app's schema.
- A report that sets or clears `x-agyn-secret` on an existing property is accepted.
- In the Console, a `format: password` property is an obscured input holding its value, with a control to reveal it — no **Set / Replace**.
- The Telegram and Slack connectors declare their tokens as `format: password`.

## Notes

This reverses one decision from [App Configuration Schema](2026-08-19-app-configuration-schema.md) and leaves the rest of it standing: the reported schema, the honored subset, backward compatibility, per-property validation, and `x-agyn-ref` are unaffected.

Reserving a keyword by rejecting it is deliberate. Ignoring an unknown `x-agyn-secret` would let apps ship schemas that silently do nothing; rejecting it means the one app that tries learns immediately, while the name stays free for the design that earns it.

The reversal removes a cost that was easy to miss. Redaction is a property of the installation message, so every read path had to resolve the app's schema to build a response — an extra lookup per read, and per distinct app in a list. Reads are now the plain query they were.

Nothing about writes changes, in this change or the one it amends. Configuration has always been submitted whole in plain text, which is exactly why the read-side hiding could not mean what it appeared to.
