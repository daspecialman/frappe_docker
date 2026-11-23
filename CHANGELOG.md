# Changelog

All notable changes to this repository are documented in this file.

## [2026-11-23]

### Added

- `ansible/playbooks/fix_cash_request_save.yml` — repairs Cash Request saves that fail with HTTP 403 when the user's own Employee record is not readable and when "Follow Created Documents" is enabled.

### Fixed

- Cash Request save failure for a production user:
  - The "Requesting Employee" link search returned no results because the user could not read their own Employee record. The playbook grants read-only access to that one Employee record via a `DocShare`.
  - Saving a workflow-enabled Cash Request created a Workflow Action whose follow-up permission check raised `PermissionError`, rolling back the save. The playbook disables `follow_created_documents` and `document_follow_notify` for the user, matching the rest of the users.
  - The user's existing `Employee` user permission is preserved, so they remain limited to their own Employee record.

### Operations

- Applied to the production site with a site backup taken first; the site cache was cleared after the change.
