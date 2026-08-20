# App Configuration Schema

## Target

- [Apps — Configuration Schema](../architecture/apps.md#configuration-schema) (new)
- [Apps — Reporting](../architecture/apps.md#reporting) (new)
- [Apps — Compatibility](../architecture/apps.md#compatibility) (new)
- [Apps — Configuration](../architecture/apps.md#configuration)
- [Apps — Installation Model](../architecture/apps.md#installation-model)
- [Apps Service — API](../architecture/apps-service.md#api)
- [Apps Service — App Resource](../architecture/apps-service.md#app-resource)
- [Apps Service — Installation Resource](../architecture/apps-service.md#installation-resource)
- [Apps Service — Schema Compatibility](../architecture/apps-service.md#schema-compatibility) (new)
- [Apps Service — Reference Validation](../architecture/apps-service.md#reference-validation) (new)
- [Authorization — Apps Service](../architecture/authz.md#apps-service)
- [Gateway — Exposed Services Table Update](../architecture/gateway.md#exposed-services-table-update)
- [Console — Configuration Forms](../product/console/console.md#configuration-forms) (new)
- [Console — Installed Apps](../product/console/console.md#installed-apps)
- [Console — Published Apps](../product/console/console.md#published-apps)
- [Telegram Connector — Configuration](../architecture/apps/telegram-connector.md#configuration)
- [Slack Connector — Configuration](../architecture/apps/slack-connector.md#configuration)

## Delta

### An app cannot say what configuration it needs

There is no `configuration_schema` — not on the App resource, not in `apps.proto`, not in the `agyn_app` Terraform resource. An app's configuration keys exist in its own documentation and nowhere the platform can read. The [Telegram Connector](../architecture/apps/telegram-connector.md#configuration) needs a `bot_token` and an `agent_id`; the platform learns this when the connector reports `configuration_invalid` in its audit log, after the installation was created.

There is likewise no RPC by which an app could say it. The three methods an app calls about itself — `GetInstallationConfiguration`, `ReportInstallationStatus`, `AppendInstallationAuditLogEntry` — are all installation-scoped. Nothing in the Apps Service is scoped to the app itself, so `ReportConfigurationSchema` is the first method with that shape and the first entry it needs in [authz](../architecture/authz.md#apps-service).

The user-facing docs describe the field as though it exists, and describe it in the wrong place — `agyn/platform/docs/build-extend/apps.md` shows `configuration_schema` on the `agyn_app` Terraform resource and `administer/apps.md` lists **Configuration schema** as a field on the Console's New app form. Neither surface has it, and neither should get it: the schema is reported by the running app, so both documents need correcting rather than implementing.

### Nothing constrains how an app changes its mind

There is no schema, so there is nothing to evolve and no rule about evolving it — but the shape the delta closes is the one that will exist the moment a schema does. An app whose next version reads a different set of keys is today a `configuration_invalid` in one org's audit log; with a form generated from a schema, it is a form that stopped matching the values behind it, in every organization that installed the app, at the moment its author deployed.

Nothing in the Apps Service counts installations by configuration key, which is what makes a deprecated property removable, and no service in the platform validates one reported document against the one it replaces. `ReportRunnerCatalog` is the closest thing and deliberately has no such rule — a runner catalog entry that disappears stops scheduling, which is recoverable by putting it back; a configuration property that disappears takes stored values with it.

### Installing an app means writing JSON by hand

`InstallAppDialog` and `UpdateInstallationDialog` both render a textarea, `JSON.parse` it, and send the result. The only validation is that the text parses. An org admin installing the Telegram Connector types a JSON object from memory or from a doc in another tab, including an `agent_id` UUID copied out of the agents page — there is no picker, because nothing tells the Console that the key names an agent.

A typo in a key name is accepted, stored, and returns as an app status days later. A UUID belonging to another organization's agent is accepted the same way.

### Bot tokens are readable in the browser

`configuration` comes back in full from `GetInstallation`, and the installation detail page prints it into a textarea. Every credential any app was ever installed with is plain text on that page for anyone who can open it. Nothing marks a value as sensitive, so nothing could mask it.

### The Apps Service validates nothing about configuration

`InstallApp` and `UpdateInstallation` store the `google.protobuf.Struct` as received. There is no per-property error path, and no service-to-service check of any value — the Apps Service calls neither Agents nor LLM today, and reference validation is a new dependency in a service that has none of this kind.

## Acceptance Signal

- `ReportConfigurationSchema` exists, is authorized to the calling app's own identity, and replaces the app's stored schema wholesale. A caller that is not the app is refused.
- A reported document outside the [honored subset](../architecture/apps.md#honored-subset) is rejected naming the keyword at fault, and the previously stored schema is still the one the install form is built from. A document using `oneOf`, `$ref`, a nested object, or an unsupported type is rejected.
- The Telegram and Slack connectors report their schemas at startup, after enrollment.
- Deploying a new version of an app moves the stored schema to the new version without any human action, and rolling back moves it back.
- A report that removes a property, changes a property's type or `items.type`, adds a property to `required`, narrows a constraint, changes or adds `x-agyn-ref`, or clears `x-agyn-secret` is rejected, naming the property and what about it was narrowed. The stored schema is unchanged, including the compatible changes in the same document.
- A report that adds an optional property, marks one `deprecated`, relaxes a constraint, drops a property from `required`, or sets `x-agyn-secret` on one that lacked it is accepted.
- A `deprecated` property is removable on a report once no installation of the app carries it, and rejected while any does — with the count in the error and no organizations named.
- A deprecated property with a stored value appears in the installation form marked deprecated and clearable; one with no value does not appear at all.
- An installation whose configuration was written before the app's first schema still serves that configuration to the app, and shows the mismatch notice with a JSON editor.
- `CreateApp` and `UpdateApp` do not accept a schema, and the Console's New app form has no schema field.
- App detail shows the reported schema read-only, when it was reported, and a preview of the install form it produces — or says the app has never reported one.
- Installing an app with a schema shows a generated form: labels and help text from the schema, required markers, defaults prefilled, a select for an `enum`, a toggle for a boolean, a repeatable list for a list of strings.
- A property marked `x-agyn-secret` is a password input on install. On installation detail it reads **Set** with a Replace action and no value. Every method returning an installation omits the property from `configuration` and names it in `secret_keys_set` — `GetInstallation`, `GetInstallationBySlug`, `ListInstallations`, `GetInstallationByIdentityId` — and `GetInstallationConfiguration` alone returns the value.
- Reading an installation and writing it back unchanged, through any client, leaves the stored secret intact rather than overwriting it with a placeholder.
- Submitting an update without resubmitting a secret property leaves the stored value unchanged.
- A property marked `x-agyn-ref: agent` is an agent picker. Submitting the ID of an agent in another organization is rejected, with the same error as a nonexistent one.
- `InstallApp` and `UpdateInstallation` reject an invalid configuration as a whole and return errors per property; the Console attaches each to the input it names.
- An app with an empty schema shows "no configuration required" instead of an editor. An app with no schema shows the JSON editor.
- Changing an app's schema does not touch existing installations. An installation whose stored configuration no longer validates shows the mismatch and a JSON editor to repair it, and still serves its configuration to the app in the meantime.
- Installing the Telegram or Slack connector requires no hand-written JSON.
- `agyn/platform/docs` describes the schema as reported by the app: the `configuration_schema` attribute is gone from the `agyn_app` examples in `build-extend/apps.md` and `administer/apps.md`, and the Console's New app steps no longer list it.

## Notes

The schema is reported rather than declared because it is a property of the deployed binary, and the same reasoning that made runner catalogs [report-only](../architecture/runners.md#catalog-reporting) applies here: two writers to one field is drift by construction, and the drift is silent until an admin fills in a correct form and the app rejects it anyway. Ordering across a deploy needs no version counter — only starting replicas report, and a deploy starts replicas of one version — which is why nothing in this change stores a schema version.

The cost is that `CreateApp` no longer produces an installable app definition on its own. An app that has never started has no form, and its first installs use the JSON editor. That is the state the fallback exists for.

Compatibility is enforced by the platform rather than left to app authors because the authors are the ones who cannot see the breakage. An app is installed into organizations its publisher has no access to, holding configuration it cannot read or migrate, and the only signal that a deploy invalidated it is a support conversation with someone else's admin. The rule costs an app author a two-step migration for a type change; the alternative costs every installing organization a broken form they did not cause.

The one rule that reads installation data — a deprecated property is removable only when no installation carries it — makes a report's outcome depend on state its caller cannot see. That is accepted deliberately: the failure is safe (rejected, previous schema stands, error names the count), and the alternative is either a schema that can never shed a property or a removal that silently orphans values in organizations that lagged.

The honored subset is deliberately narrower than JSON Schema. Every construct left out is one the Console cannot render, and a schema that validates configuration no form can produce is worse than no schema — it passes review and fails at the only moment it matters. The cost is that an app with genuinely nested configuration flattens it into dotted keys or keeps a JSON editor by declaring nothing.

`x-agyn-ref` gives the Apps Service its first outbound calls to Agents and LLM. It is what turns "paste a UUID" into "pick an agent", which is most of the difference between the two experiences, and it is also the part with a real failure mode: a reference check that cannot reach its service fails the write. That is the intended behavior — an unvalidated reference is discovered by the app at runtime, where there is no form to fix it.

This closes two of the four questions under [Installation Configuration Secrets](../open-questions.md#installation-configuration-secrets): which keys are sensitive, and who may read them. How secret values are held at rest is untouched — they are still plain text in the Apps Service database, and are still submitted in plain text. Masking them in the Console is not encryption, and the change should not be read as making them safe.

Nothing here changes what the app receives. `GetInstallationConfiguration` returns the same JSON object it always did, so an app gains a schema without changing a line of its configuration handling.

Omitting secret properties on read rather than masking them is what keeps the Terraform provider working. `agyn_app_installation` carries `configuration` as a `jsonencode` blob and refreshes it; a masked value would come back as a placeholder and either produce a permanent diff or be written back over the credential on the next apply. With the key absent, the provider keeps the configured value in its own state and refreshes the rest, which is the ordinary write-only-attribute pattern.

`agyn` has no app commands, so the Console is the only human surface this reaches. Terraform is touched only by *not* gaining the attribute its documentation already advertises.
