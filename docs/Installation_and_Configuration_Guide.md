# Xurrent IMR for ServiceNow — Installation & Configuration Guide

> Template note: This document follows the ServiceNow-approved Installation & Configuration Guide template. Replace partner-specific placeholders (URLs, support contacts, screenshots) before submission.

## 1. Overview

The Xurrent IMR application integrates ServiceNow Incident Management with the Xurrent IMR service (powered by Zenduty). It synchronizes incidents and work notes bidirectionally:

- **Outbound:** Business Rules on `incident` and `sys_journal_field` send signed events to Xurrent IMR (HMAC-SHA256, `X-Xurrent-IMR-Signature` header).
- **Inbound:** Scripted REST APIs receive create/update requests and write to OOB tables **through Import Set staging tables and Transform Maps** (no direct OOB inserts).

- **Scope:** `x_xurre_imr`
- **Package sys_id:** `99b1a1ff83762210b474b755eeaad3d0`

## 2. Prerequisites

- ServiceNow release: (state minimum supported family).
- Roles to install/configure: `admin`.
- A Xurrent / Zenduty account with ServiceNow integration enabled.

## 3. Install the application

Install the Xurrent IMR application from the ServiceNow Store. All artifacts (Scripted REST APIs, Business Rules, Import Set tables, Transform Maps, ACLs, roles, system properties, application menu and modules) are provisioned automatically.

## 4. Configure OAuth (inbound authentication)

The OAuth Application Registry record is **not shipped with the app** — create it manually so no credentials are packaged:

1. Navigate to **System OAuth → Application Registry → New → Create an OAuth API endpoint for external clients**.
2. Set **Name** (e.g. `Xurrent IMR OAuth`); leave **Client ID**/**Client Secret** blank so ServiceNow auto-generates them on save.
3. Set the **Application** scope to **Xurrent IMR**, and set the **User** field to the dedicated integration user created in section 5.
4. Save, then copy the generated **Client ID** and click **Generate Client Secret** to copy the secret (shown once).
5. Enter both the Client ID and Client Secret in the Xurrent IMR portal under the ServiceNow integration settings.

## 5. Create a dedicated integration user (least privilege)

Do **not** run the integration as System Administrator. Create a dedicated local integration user:

1. Navigate to **User Administration → Users → New**. Create e.g. `xurrent.imr.integration`.
2. Assign these **application roles** (shipped with the app):
   - `x_xurre_imr.integration` — access to the Scripted REST API endpoints (replaces the deprecated `rest_service` role; scopes endpoint access to this user only)
   - `x_xurre_imr.import_incident_user`
   - `x_xurre_imr.import_work_note_user`
   - `x_xurre_imr.incident_meta_user`
3. Also assign the **platform (built-in) roles** required so `GlideRecordSecure` can read/write the OOB tables the integration touches:
   - `itil` — create/read/write `incident`, read `sys_choice`, `sys_user_group`, `service_offering`, `cmdb_ci_service`, and journal (`sys_journal_field`)
   - `cmdb_read` — read `cmdb_ci` (if not already covered by `itil`)
   - `personalize_dictionary` (and `personalize_choices`) — read `sys_db_object` / `sys_dictionary` for the Fetch Entity endpoints
4. Navigate to **System OAuth → Application Registry → Xurrent IMR OAuth** and set the **User** field to this dedicated user.

**GlideRecordSecure:** every app script (REST operations, transform scripts, Business Rules) uses `GlideRecordSecure`, so all reads/writes are ACL-checked for the acting user — the app does **not** ship any ACLs on out-of-scope (OOB/system) tables. Instead the integration user must hold the platform roles above. The app ships only in-scope ACLs (staging tables, `x_xurre_imr_incident_meta`, and the REST endpoints).

> **Note (`sys_scope`):** the *Validate Connections* endpoint reads `sys_scope`, which has no non-admin built-in read role. If validation returns empty for the integration user, grant a role that can read `sys_scope` (or run validation as a user who can).
>
> **Note (new entity tables):** when you add a table to `x_xurre_imr.allowed_entity_tables`, make sure the integration user also has a platform role granting read on that table, or GlideRecordSecure returns no rows for it.

## 6. Configure system properties

Navigate to **System Properties** (`sys_properties.list`) and set:

| Property | Purpose | Example |
|---|---|---|
| `x_xurre_imr.service_now_instance_identifier` | Instance identifier issued by Xurrent/Zenduty | `your-instance-id` |
| `x_xurre_imr.webhook_secret` | Shared HMAC-SHA256 secret for outbound signature; must match the value configured in the Xurrent portal | (32+ char secret) |
| `x_xurre_imr.integration_enabled` | Master on/off gate for outbound Business Rules | `true` (after setup) |
| `x_xurre_imr.allowed_entity_tables` | Comma-separated list of tables the integration may expose through the Fetch Entity endpoints. The admin controls this list. | `cmdb_ci,service_offering,cmdb_ci_service,sys_user_group` |

> Generate a webhook secret with, e.g., `openssl rand -hex 32`, and paste the same value into both ServiceNow and the Xurrent portal.

> **Entity table access:** Fetch Entity Types returns only tables in `x_xurre_imr.allowed_entity_tables`; Fetch Entity Records and Fetch Entity Type Reference Fields reject (HTTP 403) any table not on the list. Edit the property to add/remove the tables your integration needs.

## 7. Enable the integration

Set `x_xurre_imr.integration_enabled = true`. Until this is `true`, the outbound Business Rules are a no-op and no data leaves the instance.

## 8. Incident metadata (related list)

The application stores its per-incident metadata in its own table `x_xurre_imr_incident_meta` (fields: `incident`, `originated_from`, `imr_incident`) — no custom columns are added to the OOB `incident` table. To view it on the incident form, add the related list: open an incident → **Configure → Related Lists** → move **Incident Meta** (the `Incident Meta->Incident` entry) into Selected → Save.

## 9. Verification

See the Test Plan document. At minimum: create an incident with the integration enabled and confirm an outbound call in **System Log → Outbound HTTP Requests** with the `X-Xurrent-IMR-Signature` header; and confirm an inbound create/update from Xurrent IMR lands via the Import Set staging table and Transform Map.

## 10. Uninstall / disable

Set `x_xurre_imr.integration_enabled = false` to halt outbound sync without uninstalling.
