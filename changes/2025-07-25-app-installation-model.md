# App Installation Model

## Target

- [Apps — App Definition](../architecture/apps.md#app-definition)
- [Apps — App Installation](../architecture/apps.md#app-installation)
- [Apps Service](../architecture/apps-service.md)

## Delta

The platform does not implement app definitions (org-owned, with visibility) or app installations (per-org, with slug, configuration, and permissions bridge). The current implementation uses cluster-scoped app registrations with a single global slug.

Specifically:

- Apps Service does not have `app_definitions` or `app_installations` tables.
- Apps Service API does not include `CreateApp`, `InstallApp`, `GetInstallationBySlug`, `UpdateInstallation`, `UninstallApp`, `GetInstallationConfiguration`.
- Gateway app proxy resolves slugs globally, not within organization context.
- Gateway does not propagate `x-app-installation-id` header to apps.
- Authorization model does not include `can_create_thread` or `can_add_participant` relations on organizations for app identities.
- App definitions do not have `visibility` (`public` / `internal`).

## Acceptance Signal

- App definitions can be created within an organization with a slug and visibility level.
- App definitions are addressable as `{org-slug}/{app-slug}`.
- App installations can be created, linking an app definition to a target organization with a slug and configuration.
- Multiple installations of the same app within one organization (different slugs) work correctly.
- Installation creates authorization tuples granting the app identity permissions in the target org.
- Uninstall removes authorization tuples.
- Gateway resolves installation slugs within the caller's organization context.
- Gateway propagates `x-app-installation-id` header to apps on forwarded requests.
- Apps can retrieve installation configuration via `GetInstallationConfiguration`.
- Visibility enforcement: `internal` apps can only be installed by the owning org; `public` apps by any org.

## Notes

- Depends on: existing Apps Service, Gateway app proxy, Authorization model.
- The Reminders app and any other existing cluster-scoped apps need migration to the new model (create definition + installation).
- Installation configuration secrets are deferred — see [Open Questions — Installation Configuration Secrets](../open-questions.md#installation-configuration-secrets).
