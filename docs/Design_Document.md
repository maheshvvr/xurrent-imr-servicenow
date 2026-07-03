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

## 4. Security

- **Inbound auth:** OAuth 2.0 (client credentials) bound to a dedicated least-privilege integration user (`rest_service` + `x_xurre_imr.import_incident_user` + `x_xurre_imr.import_work_note_user`). No System Administrator binding ships with the app.
- **Authorization:** custom application-scoped ACLs — one per REST operation and one per Scripted REST service — all granting the `rest_service` role. The platform "Scripted REST External Default" ACL is not used or modified.
- **Outbound integrity:** HMAC-SHA256 request signing.

## 5. Custom fields on OOB tables

Two fields are added to `incident`, presented in a dedicated **Xurrent IMR** form section (not the default body):

| Field | Type | Purpose |
|---|---|---|
| `x_xurre_imr_originated_from` (Originated From) | Choice | Marks the record's origin; used for loop-prevention so incidents created by Xurrent IMR do not echo back. |
| `x_xurre_imr_x_xurre_imr_incident` (Xurrent IMR Incident) | URL | Deep link to the corresponding incident in the Xurrent IMR portal. |

## 6. Data handled & residency

See the App Privacy Policy UI page (`x_xurre_imr_privacy_policy`). Incident and work note content selected for sync is transmitted to and stored by the Xurrent IMR / Zenduty service outside ServiceNow.

## 7. Configuration surface

System properties: `x_xurre_imr.integration_enabled`, `x_xurre_imr.webhook_secret`, `x_xurre_imr.service_now_instance_identifier` (+ event API host per deployment).
