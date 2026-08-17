# Xurrent IMR for ServiceNow — Test Plan

> Template note: This document follows the ServiceNow-approved Test Plan template. Add expected screenshots / evidence columns before submission.

## 1. Test environment

- Instance: (dev/test instance name)
- App: Xurrent IMR (`x_xurre_imr`), package `99b1a1ff83762210b474b755eeaad3d0`
- Preconditions: OAuth configured; dedicated integration user bound with app roles (`x_xurre_imr.integration` + `import_incident_user` + `import_work_note_user` + `incident_meta_user`) **and** platform roles (`itil`, `cmdb_read`, `personalize_dictionary`) so GlideRecordSecure reads/writes succeed; system properties set (`x_xurre_imr.integration_enabled = true`, `x_xurre_imr.allowed_entity_tables` populated).

## 2. Test cases

### TC-01 OAuth inbound authentication
- **Steps:** Call any Scripted REST endpoint with a token issued to the dedicated integration user.
- **Expected:** 200/201; calls without a valid token or required role are rejected (401/403).

### TC-02 Create Incident (Import Set path)
- **Steps:** POST to `/api/x_xurre_imr/xurrent_imr_alerts/create_incident` with a valid payload.
- **Expected:** 201; a row is created in `x_xurre_imr_u_import_incident`, the Transform Map creates the incident, response returns the incident number; **no direct insert** to `incident`.

### TC-03 Update Incident Status
- **Expected:** 200; incident state/urgency/priority updated via the staging table + transform.

### TC-04 Update Xurrent IMR Incident
- **Expected:** 200; the `x_xurre_imr_incident_meta` row for the target incident is upserted with `imr_incident` set (and `originated_from=service_now` when newly created).

### TC-05 Create Work Note
- **Expected:** 200; work note added to the incident via the incident journal; returns the journal entry sys_id.

### TC-06 Update Work Note
- **Expected:** 200; updated content posted as a **new** work note (journal entries are immutable).

### TC-07 Outbound incident event + signature
- **Steps:** Create/update an incident (not IMR-originated) with integration enabled.
- **Expected:** Outbound call in **System Log → Outbound HTTP Requests** carrying `X-Xurrent-IMR-Signature: sha256=<hex>`; receiver verifies the HMAC with the shared secret.

### TC-08 Outbound work note event
- **Expected:** Work note (not prefixed `[IMR]`) triggers an outbound signed event.

### TC-09 Loop prevention
- **Expected:** Incidents/work notes originated by Xurrent IMR do not echo back out.

### TC-10 Integration disabled
- **Steps:** Set `x_xurre_imr.integration_enabled = false`; create an incident.
- **Expected:** No outbound call fires (Business Rules are a no-op).

### TC-11 Incident metadata related list
- **Expected:** No custom columns on the OOB `incident` table. The **Incident Meta** related list (from `x_xurre_imr_incident_meta`) shows the origin + IMR link for IMR-managed incidents; the incident body form is unchanged.

### TC-12 Privacy & Support modules
- **Expected:** App Privacy Policy and Contact Support modules open the in-scope UI pages; external links carry `rel="noopener noreferrer nofollow"`.

### TC-13 Entity types allowlist (filter)
- **Steps:** Call Fetch Entity Types with `x_xurre_imr.allowed_entity_tables` set to a subset.
- **Expected:** Only allowlisted tables returned; tables outside the list are absent.

### TC-14 Entity records/reference fields allowlist (deny)
- **Steps:** Call Fetch Entity Records and Fetch Entity Type Reference Fields for a table NOT in `x_xurre_imr.allowed_entity_tables`.
- **Expected:** HTTP 403 "Table not permitted".

### TC-15 Least-privilege role
- **Steps:** Inspect the integration user and endpoint ACLs.
- **Expected:** Endpoints authorize via `x_xurre_imr.integration` (not `rest_service`). All scripts run under GlideRecordSecure; the integration user holds the app roles + platform roles (`itil`, `cmdb_read`, `personalize_dictionary`) so reads/writes succeed. No ACLs on out-of-scope tables ship with the app. Confirm `sys_scope` read works for Validate Connections (else grant a role that reads it).

### TC-16 Idempotency (retry/replay)
- **Steps:** Repeat Create Incident with the same `x_xurre_imr_incident`; repeat Create Work Note with identical content.
- **Expected:** No duplicate incident (existing one returned) and no duplicate work note; single `x_xurre_imr_incident_meta` row per incident.

## 3. Regression
Re-run TC-02 through TC-06 after any change to the operations, transform scripts, or staging tables; all must return success.
