# Installation Status and Audit Log

## Target

- [Apps — Installation Status and Audit Log](../architecture/apps.md#installation-status-and-audit-log)
- [Apps Service — API](../architecture/apps-service.md#api)
- [Apps Service — Installation Resource](../architecture/apps-service.md#installation-resource)
- [Apps Service — Installation Audit Log Entry Resource](../architecture/apps-service.md#installation-audit-log-entry-resource)
- [Console — Apps](../product/console/console.md#apps)

## Delta

Apps have no way to communicate operational state back to the platform. Installation issues such as invalid configuration, missing credentials, or failed external connections are invisible to org admins.

Specifically:

- `installation.status` field does not exist.
- `ReportInstallationStatus` RPC does not exist.
- `AppendInstallationAuditLogEntry` RPC does not exist.
- `ListInstallationAuditLogEntries` RPC does not exist.
- `installation_audit_log_entries` table does not exist.
- Console installation detail does not display status or audit log.

## Acceptance Signal

- App can call `ReportInstallationStatus(installation_id, status)` to set a markdown status string on its installation.
- Calling `ReportInstallationStatus` again replaces the previous status.
- App can call `AppendInstallationAuditLogEntry(installation_id, message, level)` to append an entry.
- `ListInstallationAuditLogEntries(installation_id)` returns entries newest first, paginated.
- Both write RPCs are authorized to the app's own identity only.
- `ListInstallationAuditLogEntries` is authorized to org members.
- `GetInstallation` response includes the `status` field.
- Console installation detail renders the status as markdown when present.
- Console installation detail shows the audit log table when entries exist.
- Console hides the status section and audit log section when there is nothing to show.
