# Xurrent IMR for ServiceNow — Design Document

> Template note: This document follows the ServiceNow-approved Design Document template. Fill in architecture diagrams and partner specifics before submission.

## 1. Purpose

Bidirectional synchronization of ServiceNow Incidents and Work Notes with the Xurrent IMR service (powered by Zenduty).

## 2. Scope & package

- Application scope: `x_xurre_imr`
- Package sys_id: `99b1a1ff83762210b474b755eeaad3d0`

## 3. Architecture

### 3.1 Outbound (ServiceNow → Xurrent IMR)
- **ServiceNow Incident Alert Business Rule** (`incident`) and **ServiceNow Work Note Alert Business Rule** (`sys_journal_field`).
- Both are gated by the condition `gs.getProperty('x_xurre_imr.integration_enabled') == 'true'` — they are a complete no-op on instances where the integration is not configured, so they do not alter OOB behavior.
- Payloads are signed with **HMAC-SHA256** (embedded pure-JavaScript implementation, no platform crypto dependency) using `x_xurre_imr.webhook_secret`; the signature is sent in the `X-Xurrent-IMR-Signature: sha256=<hex>` header.
- Delivery is via the **ServiceNow Event** REST Message; the target host and instance identifier are parameterized.

### 3.2 Inbound (Xurrent IMR → ServiceNow)
- Three Scripted REST services: **Xurrent IMR Alerts**, **Xurrent IMR Integrations**, **Xurrent IMR Connections**.
- All write operations (Create Incident, Update Incident Status, Update Xurrent IMR Incident, Create Work Note, Update Work Note) insert into **Import Set staging tables** (`x_xurre_imr_u_import_incident`, `x_xurre_imr_u_import_work_note`) which are processed by **Transform Maps** with `onBefore` Transform Scripts. No operation writes directly to `incident` / `sys_journal_field`.
  - The Transform Scripts run with `ignore = true` and perform all target writes manually in the `onBefore` script; there are **no field-map (sys_transform_entry) rows**, so there are no reference/choice field mappings requiring an Ignore/Reject choice action.
  - The transform runs **synchronously on insert** (platform "Transform synchronously" business rule); `GlideImportSetTransformerWorker` is not used (it is blocked in scoped apps).
- Work note "update" posts a new work note (ServiceNow journal entries are immutable and scoped apps cannot write `sys_journal_field` directly).
- **Idempotency:** Create Incident returns the existing incident if one already maps to the same Xurrent IMR incident; Create/Update Work Note skip if the identical note already exists — so retries/replays don't duplicate.
- **Echo-loop prevention:** on incident create the transform writes the `x_xurre_imr_incident_meta` row (with a pre-generated incident sys_id) *before* `incident.insert()`, so the after-insert Business Rule sees the Xurrent-IMR origin and does not echo the incident back.

### 3.3 Entity endpoints (Connections)
- **Fetch Entity Types / Records / Reference Fields** are restricted to an admin-controlled allowlist property `x_xurre_imr.allowed_entity_tables`. Fetch Entity Types returns only allowlisted tables; Fetch Entity Records and Reference Fields reject a non-allowlisted table with HTTP 403.

## 4. Security

- **Inbound auth:** OAuth 2.0 (client credentials) bound to a dedicated least-privilege integration user. The OAuth Application Registry record is **created manually** by the admin (not shipped) so no credentials are packaged.
- **Authorization:** custom application-scoped ACLs — one per REST operation and one per Scripted REST service — all granting the dedicated **`x_xurre_imr.integration`** role. The deprecated `rest_service` role and the platform "Scripted REST External Default" ACL are not used.
- **Integration user roles:** app roles `x_xurre_imr.integration` + `x_xurre_imr.import_incident_user` + `x_xurre_imr.import_work_note_user` + `x_xurre_imr.incident_meta_user`, plus platform roles `itil`, `cmdb_read`, `personalize_dictionary`/`personalize_choices` needed for GlideRecordSecure reads/writes on OOB tables.
- **GlideRecordSecure everywhere:** all app scripts (REST operations, transform scripts, Business Rules) use `GlideRecordSecure` — every read/write is ACL-checked for the acting user. The app ships **no ACLs on out-of-scope tables**; access to OOB tables comes from the platform roles assigned to the integration user. Only in-scope ACLs ship (staging tables, `x_xurre_imr_incident_meta` incl. a public read for the work-note BR, and the REST endpoints granting `x_xurre_imr.integration`).
- **Outbound integrity:** HMAC-SHA256 request signing.

## 5. Application metadata table (no OOB columns)

The app adds **no custom columns to the OOB `incident` table**. Per-incident metadata lives in the app's own table `x_xurre_imr_incident_meta`, surfaced on the incident form as a related list.

| Field | Type | Purpose |
|---|---|---|
| `incident` | Reference (incident) | Links the metadata row to its incident. |
| `originated_from` | String | Record origin (`x_xurre_imr` or `service_now`); drives echo-loop prevention. |
| `imr_incident` | URL | Deep link to the corresponding incident in the Xurrent IMR portal; also the idempotency key for Create Incident. |

## 6. Data handled & residency

See the App Privacy Policy UI page (`x_xurre_imr_privacy_policy`). Incident and work note content selected for sync is transmitted to and stored by the Xurrent IMR / Zenduty service outside ServiceNow.

## 7. Configuration surface

System properties (admin-configurable only): `x_xurre_imr.integration_enabled`, `x_xurre_imr.webhook_secret`, `x_xurre_imr.service_now_instance_identifier`, `x_xurre_imr.allowed_entity_tables` (+ event API host per deployment).
