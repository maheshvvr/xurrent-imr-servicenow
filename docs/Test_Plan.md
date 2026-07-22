# Xurrent IMR for ServiceNow — Test Plan

> Template note: This document follows the ServiceNow-approved Test Plan template. Add expected screenshots / evidence columns before submission.

## 1. Test environment

- Instance: (dev/test instance name)
- App: Xurrent IMR (`x_xurre_imr`), package `99b1a1ff83762210b474b755eeaad3d0`
- Preconditions: OAuth configured, dedicated integration user bound, system properties set, `x_xurre_imr.integration_enabled = true`.

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
- **Expected:** 200; `x_xurre_imr_x_xurre_imr_incident` link set on the target incident.

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

### TC-11 Form section
- **Expected:** The two custom incident fields appear only under the **Xurrent IMR** form section, not the default incident body.

### TC-12 Privacy & Support modules
- **Expected:** App Privacy Policy and Contact Support modules open the in-scope UI pages; external links carry `rel="noopener noreferrer nofollow"`.

## 3. Regression
Re-run TC-02 through TC-06 after any change to the operations, transform scripts, or staging tables; all must return success.
