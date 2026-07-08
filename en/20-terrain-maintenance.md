# Module 20 — Maintenance (preventive, corrective, predictive)

> **Version**: 3.0 (rewrite verified line by line against the source code of July 7, 2026 — major corrections versus v2.0: there are now **11 tabs** and not 9, because **Meters** is now a genuine data-entry screen and **AI Assistant** is a full tab in its own right; the real count is **37 endpoints** (31 for management and statistics + 6 for the AI), not 41; the AI models are **Claude Sonnet 4-6** (free-form question, checklist) and **Claude Opus 4-8** (diagnosis, preventive plan, intervention analysis, cost estimate), not "opus-4-7"; the `MR-#####` number is **generated uniquely** by probing against the uniqueness constraint, not by an unprotected random draw; meter readings (hours / kilometers) **genuinely trigger** usage alerts; **none** of the 37 endpoints has a role guard — any authenticated employee can create, edit, and delete; the `check_ai_guard` AI access control is **neutral** — only the credit balance blocks (402 error); the module subtitle is **never** displayed.)
> **Menu label**: "Maintenance" ("FIELD" group in the sidebar, `Wrench` icon) — route `/maintenance`. Ref. `Sidebar.tsx:68,76`, `nav.json:20,26`.
> **Displayed page title**: "Maintenance" (`MaintenancePage.tsx:166`). The i18n subtitle "Preventive and corrective maintenance management" exists but is **never rendered**.
> **Reference code (server side)**: the entire module lives in **`ERP_REACT/backend/routers/secondary.py`** (the combined "Secondary Modules" router: Real Estate, Logistics, Rental, **Maintenance**, Weather, Compliance, Grants). The Maintenance section spans lines **6180 → 8646**: **37 endpoints** under **`/maintenance/*`** (31 for management and statistics + 6 for the AI assistant), the DDL of the 8 tables (1736-1898) and its helper `_ensure_maintenance_tables` (1899-1948), the AI system prompt (1664-1723). **There is no `routers/maintenance.py` or `routers/maintenance_ai.py` file.**
> **Do not confuse with `routers/terrain.py`**: that is a **separate** land-analysis (cadastre) module, with no connection whatsoever to equipment maintenance.
> **Actual API paths**: prefix `/api/erp/v1` (`erp_config.py:9`, mounted at `erp_api.py:1025`) — so `/api/erp/v1/maintenance/*`. The name "`/maintenance`" denotes both the React route (on the screen) and the prefix of the server calls: here the two coincide.
> **Reference code (client side)**: `ERP_REACT/frontend/src/pages/MaintenancePage.tsx` (**1,960 lines, a single file**, 11 tabs, all sub-components inline); `frontend/src/api/maintenance.ts` (445 lines, 36 functions); state store `store/useMaintenanceStore.ts`; text under `i18n/locales/fr/terrain.json` (the `terrain.maintenance.*` sub-section).
> **PostgreSQL tables** (one set **per tenant**, **created on demand**): `maintenance_types`, `maintenance_planification`, `maintenance_demandes`, `maintenance_interventions`, `maintenance_pieces`, `maintenance_historique`, `maintenance_compteurs`, `maintenance_alertes`.
> **Scope**: lightweight computerized maintenance management (CMMS) for the upkeep of construction equipment (site equipment fleet, vehicle fleet, tools). The module catalogs maintenance **types** (preventive / corrective / predictive), **schedules** recurring due dates, manages **requests** (work orders) that give rise to **interventions** and to the consumption of **parts**, produces **alerts**, keeps a **history** and **meter readings** (hours / kilometers / cycles) that trigger usage-based alerts, displays **statistics**, and offers an **AI assistant** (5 tools). It is **not** a document module: it produces **no** printable work order, **no** PDF, **no** CSV, and accepts **no** photo or attachment.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step workflows](#3-step-by-step-workflows)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

Give a Quebec construction company a **single register of the upkeep of its equipment**, from three complementary angles:

- **Preventive maintenance**: define standard procedures (the "Types" catalog), then schedule recurring due dates on a piece of equipment (every 30 days, every 250 operating hours, every 5,000 km…). When a due date approaches, the module raises an **alert**.
- **Corrective maintenance**: open a **request** (a breakdown or repair ticket) numbered `MR-#####`, move it forward through **statuses**, log one or more **interventions** in it, attach the **parts** consumed, then close it — which automatically feeds the equipment's **history**.
- **"Predictive" / usage-based maintenance**: enter **meter readings** (engine hours, mileage); when a piece of equipment crosses the interval set by a usage-based planning, the module creates an alert automatically.

The module concretely answers questions such as:

- "Which preventive maintenance is due or overdue this week?"
- "This excavator has broken down: let's open a request, note the symptoms, the intervention, the parts, and close it."
- "This generator has run 250 hours since its last service: should an alert be triggered?"
- "What is the total maintenance cost this month? How many requests are in progress, pending, completed?"
- "Given these symptoms, what is the probable diagnosis, which parts, and what cost should I estimate?" (AI assistant)

> **A note on the word "predictive."** In this module, "predictive" is **only a category label** in the types catalog (`PREDICTIVE`). There is **no predictive-analysis engine** (no learning, no sensor trending). The only real "condition-based" automation is the **usage alerts** triggered by meter readings (see 3.9 and 4.6).

### 1.2 What the module does (verified against the code)

- **Types catalog**: create, list, search, filter by category, edit, and deactivate (soft delete) standard procedures (name, category, frequency in days, estimated duration, estimated cost, required skills).
- **Preventive plannings**: create, list, search, filter by priority, edit, and delete recurring due dates per piece of equipment, with an **automatic next-due-date calculation** for day, week, and month frequencies. (These recurring schedules are called "plannings" in the UI.)
- **Maintenance requests**: create (`MR-#####` number generated by the server, initial status `DEMANDE`), list, filter by status, open a full **detail**, change the status, enter an actual cost and a solution, and delete (only outside the `EN_COURS` and `TERMINE` states).
- **Interventions**: create an intervention from a request's detail (which moves the request to `EN_COURS`), then, from the Interventions tab, edit its status, time spent, and report. Moving an intervention to `TERMINE` **automatically closes** the parent request and writes a history line.
- **Parts**: from a request's detail, add the parts consumed (name, reference, quantity, unit cost); the total cost is computed and, if the part is linked to an inventory item, **stock is decremented**. Read-only global view in the Parts tab.
- **Alerts**: generate, in one click, alerts for the plannings whose due date is approaching or is past, filter by priority, mark them "read" then "processed."
- **History**: manually log events per piece of equipment (commissioning, breakdown, inspection, replacement…) and read the chronological history. A line is also **added automatically** when a request is closed.
- **Meters**: enter hour / kilometer / cycle readings; a reading can **trigger an alert** if a usage-based planning interval is reached.
- **Statistics**: ten indicators (requests, costs, plannings, alerts, interventions) plus breakdowns by status and by priority.
- **AI assistant**: five tools (chat, diagnosis, preventive plan, checklist, cost estimate) that produce advice written by Claude, charged to the company's AI credits.

### 1.3 What the module does NOT do (important limits)

> **Read this before relying on the module.** Several natural expectations are **not** covered, and some labels visible elsewhere in the interface correspond to no screen.

- **No linked equipment.** There is **no equipment record**, **no dropdown** to inventory, rental, or vehicles, and **no validation** of the identifier. Everywhere (Planning, Requests, History, Meters), a piece of equipment is designated by a **pair**: an **Equipment type** (Inventory / Rental / Vehicle) and an **ID** typed by hand as a raw number. The screen shows "Inventory #42" without ever looking up the real name of equipment 42.
- **No export, no printing, no upload, no photo.** No PDF, no CSV, no printable work order, no attachment. The module produces **no** document handed to a client or a technician.
- **No "Maintenance orders," "Equipment," or "Schedule" (calendar) tab.** Translation labels exist for these names (`tabs.ordres` = "Orders", `tabs.equipements` = "Equipment", `tabs.planning` = "Schedule"), but **the code has only 11 tabs** and none of these three appears. Planning is a plain **list**, not a calendar view or a Gantt chart.
- **Interventions cannot be created from their tab.** The Interventions tab only **edits** and **deletes**. **Creation** goes mandatorily through a request's detail (see 3.6).
- **History and Meters are add-only.** The interface allows **neither editing nor deleting** a history entry or a meter reading. To correct one, add a new entry.
- **No automatic notification** (email, text message, browser notification) on due dates, delays, or alerts. Alerts live only in the `maintenance_alertes` table and the Alerts tab.
- **No alert cron.** Due-date alerts do **not** generate on their own: you must click **"Generate alerts"** (see 3.8). Usage alerts, for their part, are born at the moment of a meter reading.
- **No accounting entry.** A request's actual cost stays in the maintenance table; nothing is posted to Accounting (Module 14).
- **No structured technician field.** The technician is plain free text (in the history); the server columns `technicien_interne_id` and `fournisseur_externe_id` exist but are **not exposed** in the forms.
- **Some filters remain in French even in English.** The request-status, intervention-status, and event-type filter menus are **hard-coded** (untranslated): they display in French regardless of the interface language.

### 1.4 Access from the sidebar

- Left sidebar → **FIELD** group (collapsible) → **Maintenance** (`Wrench` icon). Ref. `Sidebar.tsx:68,76`.
- Direct URL: `/maintenance`.
- Page title: **"Maintenance."**
- **Default tab**: Dashboard.

> **A tenant that has never opened the page has empty tables.** The 8 `maintenance_*` tables are created **on first use** (on the fly, at the head of each endpoint). A new tenant sees counters at **0** and empty lists: this is normal, not an error. These tables are **not** created when the tenant is opened, nor by the startup auto-repair (same pattern as the Metering module).

### 1.5 Permissions and roles

> **Warning — writing open to the whole tenant.** Unlike the other modules in the same file (Logistics, Rental…), **none** of the 37 Maintenance endpoints has a role guard. The only protection is authentication and the global **consultation mode**.

| Action | Who can do it |
|--------|---------------|
| **View** (all tabs, all GETs) | Any authenticated tenant user (`get_current_user`). |
| **Create / edit / delete** a type, a planning, a request, an intervention, a part, an alert, a reading, a history entry | **Any authenticated tenant user** — there is **no** `require_role` or `require_tenant_admin_or_role` on these endpoints (`secondary.py:6208-7993`). |
| **Use the AI assistant** | Any authenticated user **with AI credits** (see 1.7). |

> **Context reminder.** The Logistics module, in the **same file**, protects its own equipment maintenance with `require_tenant_admin_or_role(*LOGISTICS_WRITE_ROLES)`. Here, that is not the case: a plain employee can delete a request or a planning. Take this into account in how you organize the work.

> **The platform super administrator cannot use the CRUD.** Each endpoint requires a **tenant context** (`if not user.schema → 400 "Missing tenant context"`). A platform super administrator (with no tenant schema) therefore receives **400** on the management operations; they remain exempt on the credit side of the AI assistant, however.

> **Consultation mode (read-only) at the tenant level.** If the company has no active Stripe subscription (canceled or absent), the whole tenant switches to **consultation mode**: reads remain allowed, but **every** write request (POST / PUT / DELETE) returns **403** (`erp_auth.py:526`). A **deactivated** company returns **401**. Since the five AI-assistant endpoints are POSTs, they also fall under this rule: **the AI assistant is not usable in consultation mode.** This guard is **global** (in `get_current_user`), which offsets the absence of a local role guard on writes.

### 1.6 The 11 tabs

Source: `MaintenancePage.tsx:150-162` (the `tabs` array) and `:188-198` (rendering). The values in parentheses are dynamic counters.

| # | Internal key | Displayed label | Icon | Counter | Actual content |
|---|--------------|-----------------|------|---------|----------------|
| 1 | `dashboard` | Dashboard | `BarChart3` | — | Indicators + urgent requests + due plannings + latest alerts |
| 2 | `types` | Types | `Settings` | — | Catalog of standard maintenance procedures |
| 3 | `planification` | Planning (N) | `Calendar` | overdue plannings | Recurring due dates per piece of equipment (list) |
| 4 | `demandes` | Requests (N) | `ClipboardList` | pending requests | Requests (tickets) + detail (update, parts, interventions) |
| 5 | `interventions` | Interventions (N) | `Wrench` | in-progress interventions | Interventions performed (edit and delete) |
| 6 | `pieces` | Parts | `Package` | — | Global view of parts consumed (read-only) |
| 7 | `alertes` | Alerts (N) | `Bell` | unread alerts | Preventive alerts + "Generate alerts" button |
| 8 | `historique` | History | `History` | — | Chronological history per piece of equipment |
| 9 | `compteurs` | Meters | `Gauge` | — | Hour / kilometer / cycle readings |
| 10 | `stats` | Statistics | `BarChart3` | — | Ten indicators + status / priority breakdowns |
| 11 | `ia` | AI Assistant | `Sparkles` | — | 5 assistance tools (paid via AI credits) |

> **Dynamic badges.** The `(N)` badge on Planning shows the number of overdue plannings; on Requests, the number of pending requests; on Interventions, the number of in-progress interventions; on Alerts, the number of unread alerts. A badge appears only if the number is greater than zero.

### 1.7 Costs and billing (AI assistant only)

- **The entire register part is free**: managing types, plannings, requests, interventions, parts, alerts, history, and meters consumes no credit.
- **Only the AI assistant is paid.** Each call to one of the tools consumes the company's **prepaid AI credits** (a wallet shared with the ERP's other assistants). The cost is the used model's rate **plus a 30% markup**:
  - **Claude Sonnet 4-6** tools (chat, checklist): about US$3 per million input tokens, US$15 per million output tokens, × 1.30.
  - **Claude Opus 4-8** tools (diagnosis, preventive plan, cost estimate — and the dormant intervention-analysis endpoint): about US$5 per million input tokens, US$25 per million output tokens, × 1.30 (plus any caching).
- Usage is tracked under a `maintenance_chat`, `maintenance_diagnose`, `maintenance_preventive`, `maintenance_checklist`, or `maintenance_estimate_cost` feature in the super administrator's AI usage tracking.
- An account with **no credits** gets a **402** "Insufficient AI credits" error and cannot launch the tool; the maintenance register itself remains free.

> **Warning — no idempotency on the AI charge.** The credit charge is performed **with no idempotency key**: a **double-click** or a **network retry** on the same call can **charge twice** (the rate already being marked up 30%). Moreover, a failed charge is **silently ignored** (the answer is returned anyway). Wait for the answer (or the error message) before re-running a tool.

### 1.8 Technical architecture

```
Frontend  MaintenancePage.tsx (1,960 lines, 11 tabs, a single file)
    │
    ├── Dashboard / Types / Planning / Requests / Interventions /
    │   Parts / Alerts / History / Meters / Statistics
    │        └─ api/maintenance.ts ──> secondary.py  /api/erp/v1/maintenance/*   (31 management + statistics endpoints)
    │                                   maintenance_* tables (8, created on demand, per tenant)
    │
    └── AI Assistant tab
             └─ api/maintenance.ts ──> secondary.py  /api/erp/v1/maintenance/ia/*  (6 POST endpoints, 5 wired)
                                        Claude sonnet-4-6 (chat / checklist)
                                        Claude opus-4-8  (diagnose / preventive / analyze-intervention / estimate-cost)
                                        prepaid AI credit charge + usage tracking
```

> **Point of attention for a new tenant.** The 8 `maintenance_*` tables are created **at the head of each endpoint** (`_ensure_maintenance_tables`, `secondary.py:1899`). They are **not** created when the tenant is opened, nor by the startup auto-repair. Concretely: as soon as the first action happens (for example, creating a type), everything falls into place; before that, the counters stay at zero.

---

## 2. Interface

Source: `MaintenancePage.tsx` (1,960 lines, inline sub-components).

### 2.1 General layout

- **Title** "Maintenance" always shown at the top (`:166`). The module subtitle is never rendered.
- **Tab bar** horizontally scrollable on a small screen; the active tab is underlined in blue.
- **Error banner** (red, dismissible) above the tabs after a failed action; it resets when you change tabs.
- Each tab generally has a **command bar** (main action button on the left) and, on the right, a **search field** and one or more **filters**. Search is **local** (it filters the already-loaded page), except the Requests status filter, which is applied **server-side**.
- **Responsive**: tables (desktop display) turn into stacked cards on a phone.

### 2.2 Dashboard tab

Source: `:207-305`. Fed by `GET /maintenance/statistics` and the already-loaded lists.

**Four indicator cards:**

| Card | Value | Color |
|------|-------|-------|
| Interventions | Interventions this month | blue |
| In progress | Requests with status `EN_COURS` | green |
| Requests | Pending requests (status `DEMANDE`) | yellow |
| Unread alerts | Alerts neither read nor processed | red |

**Three list cards (first 5 items):**

- **Requests** (`AlertTriangle` icon): up to 5 **urgent** requests (priority `CRITIQUE` or `HAUTE`), with the title, "{number} - {description}", and priority and status badges. Empty: "No data."
- **Planning due**: up to 5 active plannings whose next due date has already been reached, with the name, "Planned: {date}", and a priority badge. Empty: "No overdue planning."
- **Latest alerts**: up to 5 recent alerts, with the title, the message, and a priority badge. Empty: "No active alert."

### 2.3 Types tab

Source: `:311-488`. Catalog of the **standard maintenance procedures** (for example, "Engine oil change 250 h," "Annual electrical inspection").

**Command bar**: **"New type"** button. On the right: search ("Search...") and **Category** filter (All / Preventive / Corrective / Predictive, local filter).

**"Maintenance types" table**, columns:

| Column | Content |
|--------|---------|
| Name | Type name |
| Category | Badge: `CORRECTIVE` (yellow), `PREDICTIVE` (teal), otherwise blue (`PREVENTIVE`) |
| Frequency | "{n} days" or "-" |
| Est. duration | "{n} h" or "-" |
| Est. cost | Dollar amount or "-" |
| Actions | Edit · Delete |

Empty list: "No maintenance type. Create one to get started."

**"New type" / "Edit type" window:**

| Field | Type | Required |
|-------|------|----------|
| Name | text | **Yes** (only required field) |
| Description | text area | no |
| Category | menu (Preventive / Corrective / Predictive) | no |
| Frequency (days) · Estimated duration (h) | number · number | no |
| Estimated cost ($) | number | no |
| Required skills | text area | no |

The Create / Update button is **disabled as long as the name is empty** (with a guard against double submission).

> **Hidden fields.** The server columns `checklist_json` and `pieces_requises_json` exist (and the API accepts them, `api/maintenance.ts:38,42`) but do **not** appear in the form.

> **Delete = deactivate.** Deletion asks "Deactivate this maintenance type?" and performs a **soft delete** (`actif = FALSE`): the type disappears from the list (default filter on active items) but stays in the database and remains referenceable by existing plannings.

### 2.4 Planning tab

Source: `:494-731`. **Recurring** maintenance due dates per piece of equipment.

**Command bar**: **"New planning"** button. On the right: search and **Priority** filter (All / Low / Normal / High / Critical).

**"Plannings" table**, columns:

| Column | Content |
|--------|---------|
| Name | Planning name |
| Equipment | "{type} #{id}" |
| Frequency | "{value} {type}" (days / weeks / months / operating hours / kilometers) |
| Next due date | Date; **in red** with an **"Overdue"** badge if it is past |
| Priority | Badge |
| Actions | Edit · Delete |

Empty list: "No planning."

**"New planning" window:**

| Field | Type | Required |
|-------|------|----------|
| Name | text | **Yes** |
| Equipment type | menu (Inventory / Rental / Vehicle) | **Yes** |
| Equipment ID | number | **Yes** |
| Maintenance type | menu ("All" + the created types) | no |
| Description | text area | no |
| Frequency type | menu (Days / Weeks / Months / Operating hours / Kilometers) | no |
| Value | number (≥ 1) | no |
| Start date · Next due date | date · date | no |
| Alert threshold (days) · Priority | number · menu | no |

The Create button is **disabled as long as the name or the equipment ID is missing**.

> **Automatic next-due-date calculation** (at creation, if the field is left empty): for `JOURS`, `start date + value days`; for `SEMAINES`, `+ value × 7 days`; for `MOIS`, `+ value × 30 days` (**approximation** — not a true calendar month). For `HEURES_UTILISATION` and `KILOMETRES`, no date is computed (these frequencies are handled by the "usage" path: see 3.9 and 4.6).

> **Validation.** A frequency value `≤ 0` is refused (**400**). Deletion asks "Delete this planning?" and is **permanent** (soft delete server-side, removal from the list).

### 2.5 Requests tab

Source: `:737-923`. Maintenance requests (the "tickets": corrective, preventive, or urgent).

**Command bar**: **"New request"** button. On the right: search and **Status** filter — this filter is applied **server-side** (it re-runs `GET /maintenance/requests`). Its menu labels are **hard-coded in French** (shown in French even when the interface is in English).

**"Maintenance requests" table**, columns:

| Column | Content |
|--------|---------|
| Number | `MR-#####` (monospace font) |
| Title | Request title |
| Type | Maintenance type |
| Priority | Badge |
| Status | Badge (see 4.1) |
| Date | Request date |
| Actions | Eye (Detail) · Delete |

Empty list: "No requests."

**"New request" window:**

| Field | Type | Required |
|-------|------|----------|
| Title | text | no (auto-generated from the first 80 characters of the description if empty) |
| Description | text area | **Yes** |
| Symptoms | text area | no |
| Maintenance type | menu (Corrective / Preventive / Urgent) | no |
| Priority | menu | no |
| Equipment type · Equipment ID | menu · number | no |
| Estimated cost ($) | number | no |

The Create button is **disabled as long as the description is empty**. At creation, the server assigns the `MR-#####` number and sets the status `DEMANDE`.

> **Conditional deletion.** Deletion asks "Delete this request?". The server **refuses (400)** to delete a request that is **`EN_COURS`** or **`TERMINE`**. To delete, first bring the request back to another status (`DEMANDE`, `APPROUVE`, `PLANIFIE`, `EN_ATTENTE_PIECES`, or `ANNULE`). Deletion removes the request's parts and interventions in cascade.

#### "Request {number}" window (detail)

Source: `RequestDetailModal`, `:929-1114` (large size). This is the heart of the corrective work. It contains four cards.

1. **Info (read-only)**: Title, Priority (badge), Type, Equipment ("#id"), Description and, if present, the Symptoms.
2. **Update** (three-item grid): a **Status** menu, an **Actual cost ($)** field, and a **Save** button (which sends only the modified fields); plus a **Solution** area.
3. **Parts ({n}) — Total: {amount}**: an **Add** button opens a form **Part name** (required) + Reference + Quantity + Unit cost + Save. The table lists Name, Reference, Quantity, Cost and a **✕** button to remove a line. A line's total cost = quantity × unit cost.
4. **Interventions ({n})**: a **New** button opens a form **Type** ("Ex: Service, Repair...") + **Description** (required) + Save. The list shows the type (or "Intervention"), a status badge, the work description, and the date. Empty: "No intervention recorded."

> **Parts and interventions are entered here, not elsewhere.** The Parts and Interventions tabs are only for viewing (and, for interventions, editing / deleting). The entry point for creation is this detail window.

### 2.6 Interventions tab

Source: `:1120-1259`. Interventions performed, attached to a request.

**Command bar**: **no action button** (creation happens from a request's detail). On the right: search and **Status** filter (Tous / En cours / Terminé / Reporté — hard-coded French labels; English gloss: All / In progress / Completed / Postponed).

**"Interventions" table**, columns:

| Column | Content |
|--------|---------|
| Request | Parent request number (or "#id") |
| Type | Intervention type (free text) |
| Description | Work description |
| Duration | "{n} h" or "-" |
| Status | Badge (In progress / Completed / Postponed) |
| Date | Intervention date |
| Actions | Edit · Delete |

Empty list: "No interventions."

**"Edit intervention" window** (edit only): a **Status** menu (In progress / Completed / Postponed), a **Time spent (h)** field, and a **Report** area.

> **Major automatic effect.** Moving an intervention to **`TERMINE`** automatically closes the **parent request** (status `TERMINE`, end date set) and **writes a history line** (see 4.7). Deletion asks "Delete this intervention?" and removes the parts attached to the intervention in cascade.

### 2.7 Parts tab

Source: `:1265-1350`. **Read-only global view** of the parts consumed.

**Command bar**: **no button**, only the search.

**"Parts used" card** with, at the top right, "Total cost: {sum}" (sum of the total costs of the filtered lines). Table columns: **Part, Reference, Request (#id), Quantity, Unit cost, Total cost**, plus a Delete action. Empty list: "No parts recorded."

> **No creation here.** To add a part, open the detail of the relevant request (2.5). Deletion asks "Delete this part?".

### 2.8 Alerts tab

Source: `:1356-1452`. Preventive alerts from the plannings whose due date is approaching or is past.

**Command bar**: **"Generate alerts"** button (`Zap` icon, calls `POST /maintenance/alertes/generate`). On the right: an **"Unread"** checkbox and a **Priority** filter. After generation, a green banner announces "{n} alert(s) generated."

**List (cards, not a table)**: for each alert, a priority badge, the title (with a **"Processed"** badge if it has been processed), the message, "{alert type} - {date}" and, where applicable, "Next due date: {date}." The background is **amber** as long as the alert is unread. Buttons per alert: **"Mark as read"** (if unread) and **"Mark as processed"** (if not processed). Empty list: "No alerts."

> **What "Generate alerts" does.** The server scans the active plannings whose next due date falls within the window `≤ today + alert threshold` (default 7 days) and that do not already have an unprocessed alert. For each, it creates a **`MAINTENANCE_RETARD`** alert if the due date is past, otherwise a **`MAINTENANCE_DUE`** alert. The operation is **idempotent** (a partial unique index prevents duplicates) and capped at 500 alerts per call.

### 2.9 History tab

Source: `:1458-1587`. Chronological history of events per piece of equipment.

**Command bar**: **"History"** button (`Plus` icon, opens the add form). On the right: search and **Event type** filter (Tous + Maintenance / Panne / Inspection / Remplacement / Mise en service / Mise hors service — hard-coded French labels; English gloss: All + Maintenance / Breakdown / Inspection / Replacement / Commissioning / Decommissioning).

**List (cards)**: for each event, a type badge (`PANNE` in red, `INSPECTION` in blue, otherwise green), "{type} #{id}", the description, the date, "Technician: {name}", the cost, and "{n} h." Empty list: "No history."

**Window (add an entry)**: Equipment type + Equipment ID; **Event type** (required); Description; Cost ($) + Technician (free text). The button is **disabled as long as the equipment ID is missing**.

> **Read-only afterward.** History has **neither editing nor deletion** in the interface. A `MAINTENANCE` entry is added **automatically** when a request is closed (see 4.7). To correct a mistake, add a complementary entry.

### 2.10 Meters tab

Source: `:1645-1769`. Usage readings (engine hours, mileage, cycles) — the engine of "usage-based" maintenance.

**Command bar**: **"New reading"** button. On the right: search. After creation, a banner announces "Reading saved." or, if a usage-based planning threshold is reached, "Reading saved. {n} maintenance alert(s) generated."

**"Meter readings" card** with the note "Record hour/kilometer readings. Usage-based schedules raise an alert when the interval is reached." Table columns: **Equipment, Meter type, Value** ("{n} {unit}", km / h / blank), **Reading date, Notes**. Empty list: "No readings recorded."

**"Meter reading" window**: Equipment type; Equipment ID; **Meter type** (Hours / Kilometers / Cycles); Value; Reading date; Notes. The button is **disabled as long as the equipment ID is missing**.

> **Add-only.** Like history, readings can be neither edited nor deleted from the interface.

> **Watch out for "Cycles" meters.** The usage-alert mechanism recognizes only **Hours** (→ "operating hours" frequency) and **Kilometers** (→ "kilometers" frequency). A **Cycles** meter is associated with no frequency and will therefore **never** trigger an automatic alert (see 4.6). You can still record it for tracking.

### 2.11 Statistics tab

Source: `:1593-1639`. Fed by `GET /maintenance/statistics` (`secondary.py:7909`).

**Ten indicator cards**: Total requests, In progress, Pending, Completed (completed this month), Total cost (actual cost), Estimated cost, Planning (active plannings), Overdue (overdue plannings), Unread alerts, Interventions (this month).

**Two breakdown cards**: **By status** (a status badge and its count) and **By priority** (a priority badge and its count). When there are no statistics, a skeleton is shown.

> The indicators are computed **on the fly** on each call (SQL aggregations), with no cache.

### 2.12 AI Assistant tab

Source: `IaAssistantTab`, `:1801-1959`. **Five tools** exposed, each with a **"Run"** button disabled as long as the required fields are not filled in, an "Analyzing..." indicator, and a result shown by a recursive viewer (`IaJsonView`) that formats the returned structured JSON. A **permanent warning** is displayed: "AI-generated answers — to be validated by a qualified technician."

| Tool | Model | What you enter | Endpoint |
|------|-------|----------------|----------|
| **Chat** | Claude Sonnet 4-6 | A free-form question (chat, `Enter` = send) | `POST /maintenance/ia/chat` |
| **Diagnosis** | Claude Opus 4-8 | Equipment + Observed symptoms (+ History, optional) | `POST /maintenance/ia/diagnose` |
| **Preventive plan** | Claude Opus 4-8 | Equipment + Usage (+ Last maintenance, optional) | `POST /maintenance/ia/preventive` |
| **Checklist** | Claude Sonnet 4-6 | Maintenance type + Equipment | `POST /maintenance/ia/checklist` |
| **Cost estimate** | Claude Opus 4-8 | Equipment + Problem (+ Urgency, optional) | `POST /maintenance/ia/estimate-cost` |

For the **Chat** tool, the area keeps the session's exchange history (user and assistant messages); when empty, it shows "Ask a question about equipment maintenance."

> **A sixth endpoint exists but is not wired.** The server also has `POST /maintenance/ia/analyze-intervention` (`secondary.py:8294`, client function `iaAnalyzeIntervention`, `api/maintenance.ts:415`), which analyzes the quality of an intervention. **No interface button calls it**: the AI Assistant tab therefore exposes only **5 of the 6** tools.

> **What the assistant is and is not.** It is an **advisor**: it drafts probable diagnoses, preventive plans, checklists (PPE — personal protective equipment, lockout, inspections…), and cost estimates from what you enter. It **writes nothing** into your data: it creates no request, no intervention, no part. The results are **thinking drafts**, to be validated by a qualified technician — especially the costs and safety.

> **Possible errors.** 402 "Insufficient AI credits" (top up the balance); 503 if the AI service is not configured; 413 if the request is too large; 403 in consultation mode (see 1.5).

### 2.13 Cross-cutting elements

- **Search**: always **local** (filters the already-loaded page), except the Requests status filter, which queries the server.
- **Hard-coded filters**: the request-status, intervention-status, and event-type menus remain in **French** even when the interface is in English.
- **No export, printing, CSV, PDF, upload, photo, or bulk action** anywhere.

---

## 3. Step-by-step workflows

### 3.1 Define a maintenance type

1. **Types** tab → **"New type."**
2. Enter the **Name** (required); as needed, the description, the **category** (Preventive / Corrective / Predictive), the frequency in days, the estimated duration, the estimated cost, and the required skills.
3. **Create** (`POST /maintenance/types`). The type becomes referenceable from the plannings.

### 3.2 Schedule recurring preventive maintenance

1. **Planning** tab → **"New planning."**
2. Enter the **Name** (required), the **Equipment type** (Inventory / Rental / Vehicle), and the **Equipment ID** (required); choose a **Maintenance type** (optional), a **Frequency type**, and a **Value**; fill in the **Start date** and, if you wish, the **Next due date** (otherwise it will be computed), the **Alert threshold** (default 7 days), and the **Priority**.
3. **Create** (`POST /maintenance/planification`). For a day / week / month frequency, the next due date is computed automatically if you left it empty; for operating hours / kilometers, the due date will be driven by the meters (see 3.9).

### 3.3 Open a corrective request (breakdown)

1. **Requests** tab → **"New request."**
2. Enter the **Description** (required; the Title fills in on its own from it if you leave it empty); as needed, the Symptoms, the **Maintenance type**, the **Priority**, the equipment (Type + ID), and an estimated cost.
3. **Create** (`POST /maintenance/requests`). The server assigns the `MR-#####` number and sets the status `DEMANDE`.

### 3.4 Advance a request (status, cost, solution)

1. **Requests** tab → click the **Eye** on the row to open the detail.
2. In the **Update** card, choose the new **Status** (`DEMANDE` → `APPROUVE` → `PLANIFIE`… or `EN_ATTENTE_PIECES`, `ANNULE`), enter the **Actual cost** and the **Solution** as needed, then **Save** (`PUT /maintenance/requests/{id}` — only the modified fields are sent).

> **No transition rule.** The server accepts any status from the list; it is the interface that suggests the logical order. Closing a request by moving it directly to `TERMINE` writes a history line (see 4.7).

### 3.5 Add a consumed part

1. Open the request's **detail** → **Parts** card → **Add.**
2. Enter the **Part name** (required); as needed, the reference, the quantity, and the unit cost.
3. **Save** (`POST /maintenance/pieces`). The total cost is computed (quantity × unit cost). If the part is linked to an inventory item, **stock is decremented** (see 5.1).

### 3.6 Log an intervention

1. Open the request's **detail** → **Interventions** card → **New.**
2. Enter the **Type** (for example "Service," "Repair") and the **Description** of the work (required).
3. **Save** (`POST /maintenance/interventions`). The intervention is born with status `EN_COURS` and, if the request was `DEMANDE` / `APPROUVE` / `PLANIFIE`, **it automatically moves to `EN_COURS`** with a start date.

### 3.7 Close an intervention (and the request)

1. **Interventions** tab → **Edit** the row.
2. Move the **Status** to `TERMINE` (or `REPORTE`), fill in the **Time spent (h)** and the **Report**.
3. **Save** (`PUT /maintenance/interventions/{id}`). On the move to `TERMINE`, the **parent request is closed automatically** (`TERMINE`, end date) and a **history line** is written (type `MAINTENANCE`, with the equipment, the description, the actual cost, and the actual time).

### 3.8 Generate and handle preventive alerts

1. **Alerts** tab → **"Generate alerts."**
2. The server creates alerts for the due or overdue plannings (see 2.8) and shows "{n} alert(s) generated."
3. On each card: **"Mark as read"** then, once the action is done, **"Mark as processed."** A processed alert drops out of the "Unread alerts" counter and shows the "Processed" badge.

> **Trigger regularly.** There is **no cron**: nothing generates due-date alerts automatically. Get into the habit of clicking "Generate alerts" (for example, once per business day). As long as an unprocessed alert exists for a planning, a new run **will not create a duplicate**.

### 3.9 Track usage and trigger a usage alert

1. **Prerequisite**: a planning whose **Frequency type** is **Operating hours** or **Kilometers**, on the target equipment (see 3.2).
2. **Meters** tab → **"New reading"** → enter the Equipment type, the ID, the **Meter type** (Hours or Kilometers), the **Value** read, and the date.
3. **Save** (`POST /maintenance/compteurs`). The **first** reading sets the reference value (no alert). After that, as soon as `value − reference ≥ interval` of the planning, a **`MAINTENANCE_DUE`** alert is created and the reference is advanced — the banner then shows "{n} maintenance alert(s) generated."

> **Reminder: "Cycles" meters trigger nothing.** Only Hours and Kilometers are tied to a frequency (see 4.6). A Cycles reading is kept but never produces an alert.

### 3.10 Manually enter a history event

1. **History** tab → **"History"** (the `+` button).
2. Enter the Equipment type and the **ID** (required), the **Event type** (Maintenance / Breakdown / Inspection / Replacement / Commissioning / Decommissioning), the description, the cost, and the technician.
3. **Save** (`POST /maintenance/historique`). Typical use cases: commissioning or decommissioning a piece of equipment, a breakdown noted outside the formal process, an external inspection.

### 3.11 Use the AI assistant

1. **AI Assistant** tab → choose the tool (Chat / Diagnosis / Preventive plan / Checklist / Cost estimate).
2. Fill in the required fields (for example, for Diagnosis: the equipment and the observed symptoms).
3. **Run.** The answer is shown (text or structured card); the **cost** is charged to the company's AI credits.

> Press **"Run"** only once and wait for the answer: the charge has no protection against double-clicks (see 1.7). Always validate the result with a qualified technician.

### 3.12 Understand a refusal (403 / consultation mode / 402)

- **All** writes fail for **everyone**, and the AI assistant returns 403: the tenant is in **consultation mode** (inactive Stripe subscription). Regularize the subscription to restore writing (Configuration / Subscription module).
- **401** everywhere: the company is **deactivated**.
- **400 "Missing tenant context"**: you are logged in as a platform super administrator (with no tenant); the maintenance CRUD is not accessible in this context.
- **402 "Insufficient AI credits"** in the assistant: top up the company's AI-credit balance. The maintenance register itself remains free.

---

## 4. Reference

### 4.1 Statuses and enumerations

All these values are validated server-side (Pydantic, in **UPPERCASE**, case-sensitive): a value outside the list returns **422** before ever reaching the database.

| Set | Values | Default |
|-----|--------|---------|
| **Request status** (7) | `DEMANDE`, `APPROUVE`, `PLANIFIE`, `EN_COURS`, `EN_ATTENTE_PIECES`, `TERMINE`, `ANNULE` | `DEMANDE` |
| **Intervention status** (3) | `EN_COURS`, `TERMINE`, `REPORTE` | `EN_COURS` |
| **Priority** (4) | `BASSE`, `NORMALE`, `HAUTE`, `CRITIQUE` | `NORMALE` |
| **Maintenance type (request)** (3) | `PREVENTIVE`, `CORRECTIVE`, `URGENTE` | `CORRECTIVE` |
| **Category (type / catalog)** (3) | `PREVENTIVE`, `CORRECTIVE`, **`PREDICTIVE`** | `PREVENTIVE` |
| **Equipment type** (3) | `INVENTORY`, `LOCATION`, `VEHICULE` | `INVENTORY` |
| **Frequency type** (5) | `JOURS`, `SEMAINES`, `MOIS`, `HEURES_UTILISATION`, `KILOMETRES` | `JOURS` |
| **Event type (history)** (6) | `MAINTENANCE`, `PANNE`, `INSPECTION`, `REMPLACEMENT`, `MISE_EN_SERVICE`, `MISE_HORS_SERVICE` | `MAINTENANCE` |
| **Meter type** (3) | `HEURES`, `KILOMETRES`, `CYCLES` | `HEURES` |
| **Alert type** (5) | `MAINTENANCE_DUE`, `MAINTENANCE_RETARD`, `PANNE`, `INSPECTION_REQUISE`, `GARANTIE_EXPIRATION` | — |

> **"PREDICTIVE" is the only place where "predictive" exists**: it is a catalog category. No predictive logic attaches to it (see 1.1).

**Request status badge colors** (`STATUT_COLORS`, `:40-43`): `DEMANDE` yellow, `APPROUVE` blue, `PLANIFIE` teal, `EN_COURS` green, `EN_ATTENTE_PIECES` amber, `TERMINE` green, `ANNULE` gray. **Priority colors** (`:113-118`): `CRITIQUE` red, `HAUTE` yellow, `NORMALE` blue, `BASSE` gray.

> **No transition rule.** Apart from the deletion guards (a request is not deletable in `EN_COURS`/`TERMINE`) and the auto-closures, any status from the list can be set. The recommended sequence remains `DEMANDE → APPROUVE → PLANIFIE → EN_COURS → TERMINE`.

### 4.2 Request life cycle (automations)

| Trigger | Automatic effect |
|---------|------------------|
| Creation of a request | Status `DEMANDE`, number `MR-#####` |
| **Creation of an intervention** on the request | If the request is `DEMANDE` / `APPROUVE` / `PLANIFIE` → moves to **`EN_COURS`** + start date |
| **Intervention moved to `TERMINE`** | Request → **`TERMINE`** + end date + **history line** (best-effort) |
| Request moved manually to `TERMINE` | **History line** (copy of equipment / description / actual cost / actual time) |
| Deletion of a request | **Refused (400)** if `EN_COURS` / `TERMINE`; otherwise cascade parts → interventions → request |

> **Non-regressive statuses.** Auto-closures never downgrade a terminal state: a late or replayed intervention does not "wake up" a request that is already `TERMINE` or `ANNULE`. The history insert is protected against duplicates (it happens only on the **real** move to `TERMINE`).

### 4.3 Next-due-date calculation

`_compute_next_maintenance` (`secondary.py:6180`):

| Frequency | Next-due-date calculation |
|-----------|---------------------------|
| `JOURS` | start date + `value` days |
| `SEMAINES` | start date + `value × 7` days |
| `MOIS` | start date + `value × 30` days (**approximation**, not a calendar month) |
| `HEURES_UTILISATION` | **no date** (driven by the meters) |
| `KILOMETRES` | **no date** (driven by the meters) |

### 4.4 Alert generation — the two paths

| Path | Trigger | Logic | Alert type |
|------|---------|-------|------------|
| **By due date** (`generate_maintenance_alertes`, `:7787`) | **"Generate alerts"** button | Active plannings whose `prochaine_maintenance ≤ today + threshold (default 7 d)` **and** with no unprocessed alert. Batch insert, `ON CONFLICT DO NOTHING`, cap 500. | `MAINTENANCE_RETARD` if overdue, otherwise `MAINTENANCE_DUE` |
| **By usage** (`_maybe_generate_usage_alerts`, `:7517`) | Entry of a **meter reading** | 1st reading = reference (nothing); then if `value − reference ≥ interval` and no unprocessed `MAINTENANCE_DUE` alert → alert + reference advance. Serialized (`FOR UPDATE`) against concurrent duplicates. | `MAINTENANCE_DUE` |

**Meter → frequency mapping** (`_COMPTEUR_TO_FREQUENCE`, `:7514`): `HEURES → HEURES_UTILISATION`, `KILOMETRES → KILOMETRES`. **`CYCLES` is not mapped**: no usage alert for this meter type.

> **De-duplication safe against concurrent access.** A partial unique index `idx_maint_alertes_dedup` on `(planification_id, type_alerte) WHERE traitee = FALSE AND planification_id IS NOT NULL` (`:1943`) prevents the same planning from having two unprocessed alerts of the same type, regardless of the number of concurrent runs.

### 4.5 Statistics (`GET /maintenance/statistics`, `:7909`)

The aggregate returns: `total`, `par_statut`, `par_priorite`, `cout_reel` and `cout_estime` (sums), `en_cours` (`EN_COURS`), `en_attente` (`DEMANDE` / `APPROUVE` / `PLANIFIE` / `EN_ATTENTE_PIECES`), `terminees_mois` (end date ≥ start of the month), `alertes_non_lues` (neither read nor processed), `planifications_actives`, `planifications_retard` (next due date < today), and `interventions_mois`.

### 4.6 Request number

`MR-#####` generated by `_gen_unique_numero` (`:962`, column `numero_demande` **UNIQUE**). The server **probes several candidates** against the uniqueness constraint and widens the entropy as a fallback: there is **never** a `COUNT(*)+1` or a `MAX(id)+1`, and the obtained number is **guaranteed unique** (a correction to v2.0, which wrongly described a random draw prone to collisions).

### 4.7 Endpoints — management and statistics (31)

Actual prefix: `/api/erp/v1`. All under `Depends(get_current_user)`, **with no role guard**. The "UI" column = triggerable from the screen.

**Types (4)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/types` | Yes | Filters `actif_only`, `categorie`. |
| POST | `/maintenance/types` | Yes | Creates a type; only `nom` is required. |
| PUT | `/maintenance/types/{id}` | Yes | Update. |
| DELETE | `/maintenance/types/{id}` | Yes | **Soft delete** (`actif = FALSE`); 404 if absent. |

**Planning (4 + 1 alias)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/planification` | Yes | Join on the types, sort by next due date. |
| POST | `/maintenance/planification` | Yes | Refuses `frequence_valeur ≤ 0` (**400**); computes the due date if absent. |
| PUT | `/maintenance/planification/{id}` | Yes | Update. |
| DELETE | `/maintenance/planification/{id}` | Yes | Soft delete. |
| GET | `/maintenance/preventive` | No | **Legacy alias** that delegates to the list of active plannings. |

**Requests (5)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/requests` | Yes | Filters `statut` / `equipement_type` / `equipement_id`; `limit` bounded 1-500. |
| GET | `/maintenance/requests/{id}` | Yes | Returns `{demande, pieces, interventions}`. |
| POST | `/maintenance/requests` | Yes | Number `MR-#####`, status `DEMANDE`. |
| PUT | `/maintenance/requests/{id}` | Yes | Atomic transaction; auto-history on the move to `TERMINE`. |
| DELETE | `/maintenance/requests/{id}` | Yes | **400** if `EN_COURS` / `TERMINE`; otherwise cascade parts + interventions. |

**Interventions (5)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/interventions` | Yes | Join on the request, `LIMIT 100`. |
| GET | `/maintenance/interventions/{id}` | No | Returns the intervention and its parts. |
| POST | `/maintenance/interventions` | Yes | 404 if the parent request is absent; moves the request to `EN_COURS`. |
| PUT | `/maintenance/interventions/{id}` | Yes | `TERMINE` → closes the request + history (SAVEPOINT). |
| DELETE | `/maintenance/interventions/{id}` | Yes | Cascade parts. |

**Parts (3)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/pieces` | Yes | Filters `demande_id` / `intervention_id`, `LIMIT 200`. |
| POST | `/maintenance/pieces` | Yes | Computes the total cost; **decrements inventory** if `inventory_item_id`. |
| DELETE | `/maintenance/pieces/{id}` | Yes | **No stock re-credit.** |

**History (2)** · **Meters (2)** · **Alerts (4)** · **Statistics (1)**

| Method | Path | UI | Notes |
|--------|------|----|-------|
| GET | `/maintenance/historique` | Yes | Equipment filters, `limit` 1-500. |
| POST | `/maintenance/historique` | Yes | Manual entry. |
| GET | `/maintenance/compteurs` | Yes | `LIMIT 100`. |
| POST | `/maintenance/compteurs` | Yes | **Triggers usage alerts**; returns `alertes_generees`. |
| GET | `/maintenance/alertes` | Yes | Filters `non_lues_only` / `priorite`, sort by priority then date, `LIMIT 100`. |
| POST | `/maintenance/alertes` | No | Manual creation of an alert. |
| PUT | `/maintenance/alertes/{id}` | Yes | Mark read / processed (`traitee` → processing date). |
| POST | `/maintenance/alertes/generate` | Yes | Generation by due date (see 4.4). |
| GET | `/maintenance/statistics` | Yes | Aggregate (see 4.5). |

### 4.8 Endpoints — AI assistant (6, of which 5 wired)

All under **POST**, protected by `get_current_user` + AI credits, capped at **32,000 tokens** in output. In consultation mode, they are blocked (403).

| Path | Model | Temperature | Output | UI |
|------|-------|-------------|--------|----|
| `/maintenance/ia/chat` | `claude-sonnet-4-6` | 0.4 | text | Yes |
| `/maintenance/ia/diagnose` | `claude-opus-4-8` | 0.3 | JSON | Yes |
| `/maintenance/ia/preventive` | `claude-opus-4-8` | 0.3 | JSON | Yes |
| `/maintenance/ia/analyze-intervention` | `claude-opus-4-8` | 0.3 | JSON | **No (dormant)** |
| `/maintenance/ia/checklist` | `claude-sonnet-4-6` | 0.3 | text | Yes |
| `/maintenance/ia/estimate-cost` | `claude-opus-4-8` | 0.3 | JSON | Yes |

**Billing chain (identical for all six):**

1. `check_ai_guard(user)` → **neutral in practice**: always returns "allowed" for an authenticated user (`ai.py:824-825`). Never blocks.
2. `_check_credits(user)` → **the real gatekeeper**: super administrator unlimited; internal instance (`BILLING_ENABLED=false`) unlimited; otherwise reads the prepaid credits (`FOR UPDATE`), **auto-tops up** as needed (Stripe), and returns **402 "Insufficient AI credits"** if the balance is exhausted.
3. Usage tracking (`track_ai_usage`, best-effort).
4. Call to Claude, **offloaded from the event loop** (`asyncio.to_thread`).
5. **Credit charge** — **with no idempotency key** and in a silent `try/except` (see the warning in 1.7).

**Cost**: the model's rate **plus a 30% markup** — Sonnet `(input × 0.003 + output × 0.015) ÷ 1000 × 1.30`; Opus `(input × 0.005 + output × 0.025) ÷ 1000 × 1.30` (plus any caching).

> **Anti-injection.** The free-form fields sent to the AI (symptoms, equipment description…) are **sanitized** before the call (`_prompt_safe`: whitespace collapsed, length bounded), and the system prompt reiterates the priority on safety and the recourse to certified technicians. The JSON answers are stripped of their ``` ``` ``` fences; if parsing fails, the `{raw, error}` object is returned (HTTP 200).

### 4.9 PostgreSQL tables (per tenant, created on demand)

DDL and helper: `secondary.py:1736-1948`.

| Table | Role |
|-------|------|
| `maintenance_types` | Catalog of standard procedures (category, frequency in days, duration, cost, skills, `actif`). |
| `maintenance_planification` | Recurring due dates per piece of equipment (frequency, next due date, threshold, priority, plus `derniere_maintenance_valeur` = usage reference). |
| `maintenance_demandes` | Requests (`numero_demande` unique `MR-#####`, status, priority, costs, solution…). |
| `maintenance_interventions` | Interventions attached to a request (type, description, duration, status, observations). |
| `maintenance_pieces` | Parts consumed (name, reference, `inventory_item_id`, quantity, costs). |
| `maintenance_historique` | Events per piece of equipment (type, date, description, cost, technician, meters). |
| `maintenance_compteurs` | Usage readings (meter type, value, date). |
| `maintenance_alertes` | Generated alerts (type, priority, due date, `lue`, `traitee`). |

> **Created at the head of each endpoint** (`_ensure_maintenance_tables`), with defensive column back-fill (`ALTER TABLE ... ADD COLUMN IF NOT EXISTS`) for older installations. A tenant with no Maintenance traffic simply does not have these tables.

### 4.10 Validations, bounds, and errors

| Rule | Effect |
|------|--------|
| Enumeration value outside the list (status, priority, type…) | **422** (before the database) |
| Cost > 99,999,999.99 · hours > 999.99 · meter value > 9,999,999,999.99 · negative number | **422** |
| Planning frequency `≤ 0` | **400** |
| Deletion of an `EN_COURS` / `TERMINE` request | **400** |
| Nonexistent request / type / planning / intervention / part / alert (PUT / DELETE) | **404** |
| Intervention with no parent request | **404** |
| Missing tenant context (platform super administrator) | **400 "Missing tenant context"** |
| Write in consultation mode / deactivated company | **403** / **401** |
| AI assistant with no credits | **402 "Insufficient AI credits"** |
| AI service not configured | **503** |
| AI request too large / service overloaded | **413** / **503** |
| Alert generation per call | cap **500** |
| Lists (requests / history 1-500; interventions 100; parts 200; meters 100; alerts 100) | fixed caps |

> **Defense against SQL injection.** Dynamic updates go through **column allowlists**; no field value builds SQL. The multi-step writes (request, intervention, part + stock, meter + alerts, alert generation) are **atomic**, and their side effects (history, stock, alerts) are isolated best-effort under **SAVEPOINT** — a failure of the side effect does not roll back the main write.

### 4.11 Rate limits and shortcuts

| Item | Detail |
|------|--------|
| `POST /maintenance/ia/*` (the 6 AI tools) | **10 requests per minute per IP address** (`erp_api.py:362,470,678-681`, key `{ip}:maintenance_ia`). |
| Other `/maintenance/*` endpoints | High general bound (≈1,500 per minute per IP). |
| Keyboard shortcut | In the AI assistant's Chat tool, `Enter` sends the message. No other module-specific shortcut. |

---

## 5. Integrations and FAQ

### 5.1 Store / Inventory (Module 09)

- `maintenance_pieces.inventory_item_id` may point to an inventory item. When a linked part is created, the server **decrements stock**: `inventory_items.quantite_metric = GREATEST(0, quantite_metric − quantity)` (best-effort, after checking that the column exists).
- **No re-credit on deletion** of a part: deleting a part line **does not return** the units to stock (fix with a manual inbound movement in the Store).
- **No** equipment link: the Type + ID pair is never matched against Store products.

### 5.2 Logistics (Module 18) — two distinct maintenance systems

> **Do not confuse.** There are **two** equipment-maintenance systems in the ERP, in the same server file but **entirely separate**.

- **This module (Maintenance, `/maintenance/*`)**: a rich CMMS — types, plannings, requests, interventions, parts, alerts, history, meters, AI — over **8** `maintenance_*` tables. **Writing open to every employee.**
- **Logistics maintenance (`/logistics/equipment/{id}/maintenance`)**: a mini upkeep log for the **internal fleet** (preventive / inspection / repair / certification interventions recorded under a logistics equipment, with a "next date" that updates the equipment's due date). Table **`logistics_equipment_maintenance`**, **role-protected** (`require_tenant_admin_or_role`).
- **No synchronization** between the two: the same machine can have data on both sides. In practice: Logistics for a quick view of a fleet vehicle, Maintenance for structured preventive / corrective tracking.

### 5.3 Rental (Module 19)

- The `LOCATION` equipment type is only a **label**: Maintenance neither reads nor writes the `location_*` tables. The `REPARATION` condition of a rental equipment stays a manual indicator, **with no automatic link** to a maintenance request.

### 5.4 Work Orders (Module 11) and Time Tracking (Module 12)

- **No integration.** Work orders plan **human work** on site; maintenance requests plan **interventions on equipment**. Nothing is created automatically from one to the other.
- Hours (`duree_heures`, actual time) are **entered by hand** in Maintenance and **do not flow** to Time Tracking or Payroll. The technician is plain free text.

### 5.5 Accounting (Module 14)

- **No accounting entry.** A request's actual cost and estimated cost stay in the maintenance table; nothing is posted to Accounting. Reconciliation (for example, to capitalize a repair) is manual.

### 5.6 AI credits (Module 24)

- The maintenance assistant shares the **same AI-credit wallet** as the ERP's other assistants.
- Each call is tracked under `maintenance_chat`, `maintenance_diagnose`, `maintenance_preventive`, `maintenance_checklist`, or `maintenance_estimate_cost`, visible in the super administrator's usage tracking.

### 5.7 Documents and photos (Module 06 Dossiers)

- **No** attachment or photo in Maintenance (the `photos_avant` / `photos_apres` columns exist in the database but are not exposed). To keep photos or reports, use the Dossiers module.

### 5.8 Frequently asked questions

**How do I know which piece of equipment is "Inventory #42"?**
There is **no** automatic name lookup. The module stores only a hand-entered Type + ID pair. Refer to the corresponding module (Store for `Inventory`, Rental for a `Rental` item, Logistics for `Vehicle`) with that identifier.

**Can I print a work order or export a report?**
No. The module has **no** export or printing: no PDF, no CSV, no printable order, no attachment.

**Where are the "Orders," "Equipment," and "Schedule" tabs?**
They don't exist. These names appear in the translation files, but **the code has only 11 tabs** and none of those. Planning is a plain list, not a calendar.

**Why can't I create an intervention from the Interventions tab?**
By design: creation happens in a **request's detail** (Interventions card). The tab is only for editing the status, the time spent, and the report, or for deleting.

**Can a history entry or a meter reading be edited?**
No. History and Meters are **add-only**. To correct one, add a new entry.

**Why does my "Cycles" reading never generate an alert?**
Because only **Hours** and **Kilometers** meters are tied to a planning frequency. The **Cycles** type is associated with no frequency: it is kept for tracking but triggers nothing.

**Do due-date alerts generate on their own?**
No. There is **no cron**: click **"Generate alerts"** regularly. **Usage** alerts, for their part, are born at the moment of a meter reading.

**Is the next due date computed for every frequency?**
No — only for **days**, **weeks**, and **months** (a month equals 30 days, an approximation). For **operating hours** and **kilometers**, the due date is driven by the meters, not by a date.

**Is the `MR-#####` number guaranteed unique?**
Yes. The server generates it by probing against a uniqueness constraint (never a `COUNT(*)+1`). This is a correction to the old manual, which wrongly described a random draw prone to collisions.

**Does a request's actual cost automatically include the parts?**
No. The actual cost is **entered by hand**. The parts total shows in the Parts card of the detail, without pre-filling the actual cost.

**Is stock returned if I delete a part?**
No. The decrement on add is **not** offset on deletion. Fix it with a manual inbound movement in the Store.

**Can any employee delete a request or a planning?**
Yes, as long as the tenant is not in consultation mode: **no** Maintenance endpoint has a role guard. Only the global read-only state (inactive subscription) blocks writing. Organize yourselves accordingly.

**Are the AI's diagnosis or estimate reliable?**
To be validated systematically by a qualified technician — that is precisely the permanently displayed warning. The costs are indicative, and the safety stakes must be verified.

**Why does the AI assistant return 403 even though I have credits?**
Probably consultation mode: since its calls are writes (POST), they are blocked when the Stripe subscription is inactive. Regularize the subscription.

**Did I pay twice for an AI call?**
It is possible in case of a double-click or a network retry: the charge has **no** idempotency key. Press "Run" only once and wait for the answer.

---

## 6. Summary

- **Purpose**: lightweight CMMS for the upkeep of construction equipment — **preventive** (types + plannings + due-date alerts), **corrective** (requests → interventions → parts → history), and **usage-based** (meters → alerts). "Predictive" is only a category label, with no predictive engine.
- **Access**: sidebar → **FIELD** group → **Maintenance** (`Wrench` icon), route `/maintenance`, title "Maintenance." Default tab: Dashboard. The subtitle is never displayed.
- **11 tabs**: Dashboard, Types, Planning, Requests, Interventions, Parts, Alerts, History, **Meters**, Statistics, **AI Assistant**.
- **A single server-side router**: everything lives in `secondary.py`, prefix **`/maintenance/*`** (**37 endpoints**: 31 for management + statistics, 6 for the AI, of which 5 are wired to the screen). **No** `maintenance.py` or `maintenance_ai.py` file; `terrain.py` is an unrelated cadastre module.
- **8** `maintenance_*` tables per tenant, **created on demand** (a new tenant sees counters at zero).
- **Permissions**: reading **and writing open to any authenticated employee** (no role guard, unlike Logistics maintenance); the only write protection = the global **consultation mode** (403) if the Stripe subscription is inactive; platform super administrator = 400 on the CRUD.
- **Number**: `MR-#####`, generated **atomically and uniquely**.
- **Request life cycle**: `DEMANDE` → (1st intervention) `EN_COURS` → (intervention `TERMINE`) `TERMINE` + automatic history; deletion forbidden in `EN_COURS` / `TERMINE`.
- **Frequencies**: `JOURS` / `SEMAINES` / `MOIS` compute a due date (month = 30 days, an approximation); `HEURES_UTILISATION` / `KILOMETRES` are driven by the meters.
- **Alerts**: two paths — by due date ("Generate alerts" button, **no cron**) and by usage (Hours or Kilometers meter reading; **Cycles triggers nothing**); de-duplication safe against concurrent access.
- **AI assistant**: 5 wired tools (Chat and Checklist on Sonnet 4-6; Diagnosis, Preventive plan, and Cost estimate on Opus 4-8), the model's rate × 1.30, real gatekeeper = the credit balance (402), neutral `check_ai_guard`. **Charge with no idempotency**: click only once. A 6th endpoint (intervention analysis) exists but stays **dormant** on the screen.
- **What the module does not do**: no linked equipment (Type + ID pair by hand, with no validation); no printing / PDF / CSV / upload / photo; no Orders / Equipment / Schedule tab; no notifications; no accounting entry; hours and technician unstructured; history and meters add-only.

---

**Documentation generated from the code**: `ERP_REACT/backend/routers/secondary.py` (Maintenance section lines 6180-8646: 37 `/maintenance/*` endpoints, DDL of the 8 tables 1736-1898, helper `_ensure_maintenance_tables` 1899-1948, AI system prompt 1664-1723, helpers `_compute_next_maintenance` 6180 / `_maybe_generate_usage_alerts` 7517 / `_gen_unique_numero` 962); `ERP_REACT/backend/routers/ai.py` (AI credits, neutral `check_ai_guard` 824-825, `_check_credits` 1192, `_deduct_credits` 1314); `ERP_REACT/backend/erp_auth.py` (consultation mode 526); `ERP_REACT/backend/erp_api.py` (mount 1025, AI rate limit 362/470/678-681); `ERP_REACT/frontend/src/pages/MaintenancePage.tsx` (1,960 lines, 11 tabs, a single file); `ERP_REACT/frontend/src/api/maintenance.ts` (445 lines, 36 functions); `store/useMaintenanceStore.ts`; text `i18n/locales/fr/terrain.json` (`terrain.maintenance.*`).

**Related manuals**:
- Module 09 (Store / Inventory — stock decrement on part add, no re-credit) — `09-operations-magasin.md`
- Module 11 (Work Orders — human work, no integration) — `11-operations-bons-de-travail.md`
- Module 12 (Time Tracking — maintenance hours not propagated) — `12-operations-pointage.md`
- Module 14 (Accounting — no automatic cost posting) — `14-operations-comptabilite.md`
- Module 18 (Logistics — separate fleet maintenance, role-protected) — `18-terrain-logistique.md`
- Module 19 (Rental — `REPARATION` condition with no automatic maintenance link) — `19-terrain-location.md`
- Module 24 (AI Assistant — shared credit wallet) — `24-communication-assistant-ia.md`
