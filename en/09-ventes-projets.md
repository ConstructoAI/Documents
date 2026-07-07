# Module 09 — Construction Projects

> **Version**: 3.0 (line-by-line rewrite verified against the source code, 2026-07-07)
> **Reference code**: `backend/routers/projects.py` (2037 lines, 20 endpoints), `backend/routers/projets_ai.py` (460 lines, 2 endpoints), `frontend/src/pages/ProjectsPage.tsx` (1249 lines), `frontend/src/components/projets/ProjetsAssistantTab.tsx` (231 lines), `frontend/src/api/projects.ts`, `frontend/src/api/projetsAi.ts`
> **Related endpoints**: `backend/routers/production.py:2810` (schedule generation), `backend/routers/devis.py:11884` (automatic conversion of an accepted quote into a project)
> **PostgreSQL tables**: `projects`, `project_phases`, `project_notes`, `project_assignments`, `dossier_projets` (association); read-only aggregates from `devis`, `factures`, `bons_commande`, `time_entries`, `companies`, `contacts`
> **Route and menu**: `/projets` — label "Projects" (icon `Briefcase`), "Management" section of the sidebar
> **Scope**: this module manages the **project record** (the master record of a jobsite): list, creation, editing, duplication, deletion, statistics, AI-categorized notes, a financial summary of actual costs, and gateways to the 360° Record, the schedule (work order) and the linked quote. It **does not** do interactive Gantt scheduling (that lives in the **Tracking** module), phase management in write mode from the screen, or real-estate development (Module 11).

---

## Table of contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step workflows](#3-step-by-step-workflows)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The **Construction Projects** module is the master record of every jobsite. It lets a contractor or an employee:

- Keep a **paginated list** of all projects, with search and filters by status and priority.
- **Create**, **edit**, **duplicate**, and **delete** a project.
- Track four **key performance indicators (KPIs)**: total number of projects, projects in progress, completed projects, and total cumulative budget.
- Consult a rich **detail panel**: jobsite metadata, phases (read-only), linked quote, financial summary of actual costs, notes.
- **Categorize a note with AI** (Claude) using a construction-specific grid.
- **Generate a schedule** (a work order pre-filled with standard operations) in one click.
- **Navigate** to the linked 360° Record.
- **Export** the full list to CSV.
- Apply a **batch update** (status or priority) to several selected projects.
- Chat with a **dedicated AI Assistant** that reads real data and proposes creating a project upon confirmation.

A project can originate in two ways: **manually** ("New project" button or AI Assistant), or **automatically** when a quote moves to "Accepted" status (see §5.2). In both cases, the Projects screen is then used to steer the record.

### 1.2 Access and prerequisites

| Prerequisite | Detail |
|---|---|
| **Menu** | Sidebar → **Management** section → **Projects** (icon `Briefcase`). Route `/projets`. |
| **Authentication** | Open session. Every call goes through `Depends(get_current_user)`. |
| **Tenant context** | The user must be attached to a company (`user.schema`). Otherwise: `400 Missing tenant context`. |
| **AI categorization and AI Assistant** | AI service available, the AI guardrail cleared, and **sufficient prepaid AI credits**. Otherwise: `503` (service unavailable), `403` (AI access denied), or `402` (credits exhausted). |

### 1.3 Roles and permissions

An important point to know: **the Projects module enforces no role check**. There is no `require_role` or `require_tenant_admin` guard on the `projects.py` endpoints. Any authenticated user of the tenant — including an account with the "user" role — can create, edit, duplicate, and delete projects and notes.

Two nuances nonetheless frame all writes:

1. **Consultation mode (read-only)** — A tenant whose subscription is suspended or cancelled switches to consultation mode. This filter is applied upstream, in `get_current_user`, and blocks all writes with a `403` before they even reach the module.
2. **Schedule generation** — The "Generate schedule" button calls an endpoint that **is** protected by `require_tenant_admin_or_role(*BT_WRITE_ROLES)` (it lives in the Production / Work Orders module). A user without write access to Work Orders will be refused on this specific action.

### 1.4 Sub-modules and screens

| Screen / component | Content |
|---|---|
| **Project list** (3 modes: List, Table, Cards) | Left column: search, filters, sort, inline editing, batch selection. |
| **Detail panel** | Right column (~45%): metadata, phases, linked quote, financials, notes, 360° Record and Schedule buttons. |
| **Modals** | New project, Edit project, Add note, AI Assistant. |
| **AI Assistant — Projects** | Natural-language chat (reads data + proposes creation upon confirmation). |

---

## 2. Interface

### 2.1 General layout

```
+----------------------------------------------------------------------+
|  Projects                                                            |
+----------------------------------------------------------------------+
|  [ Total ]  [ In progress ]  [ Completed ]  [ Total budget ] 4 KPIs  |
+----------------------------------------------------------------------+
|  [ List ]  [ Table ]  [ Cards ]              Display-mode selector   |
+----------------------------------------------------------------------+
|  (if at least one project is checked)                                |
|  N project(s) selected   [ Change status... ]  [ Deselect ]         |
+----------------------------------------------------------------------+
|  [ + New project ]  [ AI Assistant ]  [ Export CSV ]                |
|                                     [ Search... ]   [ Status: All ]  |
+----------------------------------------------------------------------+
|  LIST AREA (left, 100% or ~55% if a detail is open)                 |
|  Number | Name | Client | Budget | Status | Priority | Start | End  |
+----------------------------------------------------------------------+
|  DETAIL AREA (right, ~45%, if a project is selected)                |
|  Name + badges + [ Duplicate ]  [ Edit ]  [ Close ]                 |
|  [ View 360° Record ]  [ Generate schedule ]                        |
|  Client | Budget | Description | Address | Dates                    |
|  Phases (N)   ·   Quote NUMBER (if a quote is linked)               |
|  Financials [ Show / Hide ]   ·   Notes (N) [ + Add ]               |
+----------------------------------------------------------------------+
```

### 2.2 Page header (always visible)

**Title**: "Projects".

**Four KPI cards** (fed by `GET /projects/statistics`):

| Card | Content | Color |
|---|---|---|
| **Total** | Total number of projects for the tenant. | Neutral |
| **In progress** | Projects with the "In progress" status. | Blue |
| **Completed** | Projects with the "Completed" status. | Green |
| **Total budget** | Sum of budgets (`budget_total`), formatted in dollars. | Neutral |

**Display-mode selector**: three buttons — **List** / **Table** / **Cards**.

**Command bar**:

| Button | Icon | Effect |
|---|---|---|
| **New project** | `Plus` | Opens the creation modal. |
| **AI Assistant** | `Sparkles` | Opens the AI Assistant modal. |
| **Export CSV** | `Download` | Downloads the file `projets_export.csv`. |
| **Search…** (field) | — | Filters on the name, description, and project number. |
| **Status** (dropdown) | — | Filter: All / Pending / In progress / Completed / Cancelled / Suspended. |

**Feedback banners**: a red alert on error; a green success alert, cleared automatically after 4 seconds.

### 2.3 Batch action bar (conditional)

It appears as soon as at least one project is checked:

- "N project(s) selected" counter.
- **"Change status…"** dropdown (Pending / In progress / Completed / Cancelled / Suspended) → applies the new status to the whole selection after a **confirmation**.
- **"Deselect"** button.

### 2.4 The three list display modes

#### 2.4.1 List mode (default)

A table whose columns are **sortable** and **resizable**:

| Column | Note |
|---|---|
| Checkbox | Individual or all/none selection. |
| **Number** | `numero_projet` (format `PROJ-YYYY-NNNNN`), in blue. |
| **Name** | Project name. |
| **Client** | Resolved client name (company, contact, or manual entry). |
| **Budget** | Right-aligned. |
| **Status** | Colored badge — **inline editing** (one click opens a dropdown). |
| **Priority** | Colored badge — **inline editing**. |
| **Planned start** | Date — **inline editing**. |
| **End date** | Date — **inline editing**. |
| Action | **Delete** button (icon `Trash2`, with confirmation). |

Clicking a row (outside an editable cell) opens the detail panel. The selected row is highlighted. On a phone, the list turns into compact cards (name, status badge, client, budget, creation date, delete button). Empty message: "No project".

#### 2.4.2 Table mode (compact)

A denser table, with additional columns: checkbox, **ID**, Name, Client, **Type**, Budget, Status, Priority, Start, End, **City**. Dates stay inline-editable there.

#### 2.4.3 Cards mode

A grid of cards: checkbox, status and priority badges, name, client, budget, end date, city (icon `MapPin`), delete button.

**Pagination**: 20 projects per page, at the bottom of the list, in all three modes.

### 2.5 Detail panel (right column)

It opens when you click a project. On opening, the application loads the project, its notes, the linked dossier, and the linked quote in parallel. On a phone, a "Back to list" button replaces the detail view.

**Panel header**: the project name, followed by three buttons:

| Button | Icon | Effect |
|---|---|---|
| **Duplicate** | `Copy` | Creates a copy of the project (double-click protection). |
| **Edit** | `Pencil` | Opens the edit modal. |
| **Close** | `X` | Closes the panel. |

**Badges**: status and priority.

**Contextual buttons**:

- **"View 360° Record"** (blue, icon `FolderOpen`) — visible **only** if a dossier is linked. Navigates to `/dossier/{id}`.
- **"Generate schedule"** (green, icon `CalendarClock`) — creates a work order pre-filled with standard operations based on the project type. The operation is **idempotent**: if an automatic schedule already exists, the button returns it without creating a second one. A "Generating…" state is shown during processing.

**Metadata**: Client, Budget, Description, jobsite Address and City (icon `MapPin`), Start, End.

**Phases section** — **read-only**. For each phase: its name and its progress as a percentage. Title "Phases (N)". There is **no** button to add, edit, or delete a phase on this screen (see §5.3).

**Linked quote section** — **read-only**, shown if a quote is attached to the project. It lists the quote's lines with a **markup recalculated client-side**: administration 3%, contingencies 12%, profit 15% by default (each line may override these percentages). It shows the subtotal, the taxes (GST / QST), and the total with taxes. Title "Quote {number} (N lines)". Editing the quote is done in the **Quotes** module, not here.

**Financials section** — collapsible (Show / Hide button). It calls `GET /projects/{id}/financials` and presents the **calculated actual costs** (see §4.4):

- Four cards: **Revenue** (green), **Expenses** (red), **Margin** (blue or orange, with the percentage), **Budget** (gray, if greater than 0).
- Revenue detail: accepted quotes (informational list), client invoices (list, amount collected, status badge).
- Expenses detail: materials (purchase orders, icon `Package`), labor (hours × cost, icon `Users`), supplier invoices.
- Empty state: "No financial data for this project".

**Notes section** — title "Notes (N)" and an **"Add"** button. For each note: title, category badge (if present), importance percentage (if present), content (two lines), date, and a **"Categorize with AI"** button (icon `Bot`). The analysis shows an "Analyzing…" state. Empty state: "No note".

**Bottom line**: "Created on {date}".

### 2.6 Modals

#### 2.6.1 New project (large, two columns)

| Field | Required | Note |
|---|---|---|
| **Project name** | Yes | Rejected if empty or made up only of spaces. |
| **Client PO No.** | No | Client's purchase order number. |
| **Client (Company)** | No | Dropdown fed by the CRM (100 companies maximum). |
| **Client (Person)** | No | Dropdown of contacts. |
| **Manual entry** | No | Free-form client name if the client is not in the CRM. |
| **Project type (schedule template)** | No | Dropdown of 5 types (see §4.3). Drives the schedule's operations template. |
| **Status** | No | Defaults to "Pending". |
| **Priority** | No | Defaults to "Medium". |
| **Planned work start** | No | Date. |
| **Planned work end** | No | Date. |
| **Budget ($)** | No | Number (no negative entry). |
| **Jobsite address** | No | |
| **Jobsite city** | No | |
| **Description** | No | Text area. |

A note reminds you that the field marked with an asterisk is required. The **Create** button stays disabled as long as the name is empty. **All these fields are saved** (including the client PO, the contact, and the manual entry — see §4.6).

#### 2.6.2 Add note

Three fields: **Title** (required), **Content** (required, text area), and **Category** (optional, e.g., "Technical, Safety, Budget…"). Cancel / Add buttons.

#### 2.6.3 Edit project (medium size)

Fields: Name (required), Description, Project type, Status, Priority, Start date, End date, Budget, Jobsite address, Jobsite city. Cancel / **Save** buttons. Clearing the Budget field erases the value (sends `null`). An alert is shown on an editing error.

#### 2.6.4 AI Assistant

Opens the `ProjetsAssistantTab` chat component (see §2.7).

### 2.7 AI Assistant — Projects

The assistant follows a **propose → confirm** model: it can read data and propose creating a project, but it **never writes** directly to the database. The user is the one who confirms.

**Header**: "AI Assistant — Projects" and a subtitle. The welcome screen offers three sample questions:

- "Which projects are in progress and for what total budget?"
- "Create a project …"
- "Which projects end this month?"

**Version 1 capabilities**:

1. **Read** — The `recherche_bd` tool queries a strict whitelist of three tables: `projects`, `companies`, `contacts`. A guard blocks any sensitive table (employees, payroll, salaries, SIN, users, AI credits, etc.). Maximum 50 rows per query.
2. **Action** — A single action type: **project creation**. The AI calls the `proposer_projet` tool, which displays a **proposal card** (icon `Briefcase`) showing the proposed fields. The user clicks **Confirm** (blue, icon `CheckCircle`) or **Cancel** (icon `X`). Only the Confirm button actually triggers the creation. An "Awaiting confirmation" badge accompanies the card.

> **Editing and deletion by the AI are not implemented** (a planned feature that has not shipped). The assistant can only read and propose a creation.

**Input area**: a text area (Enter = send, Shift+Enter = new line) and a **Send** button. Each message bubble shows metadata (profile, tokens, cost in dollars, duration). Synchronous locks prevent double submission and double confirmation.

---

## 3. Step-by-step workflows

### 3.1 Create a project manually

1. Click **"New project"**.
2. Enter the **Name** (required).
3. Choose the client: either **Client (Company)**, **Client (Person)**, or **Manual entry** if the client is not in the CRM. All three fields are saved.
4. Fill in the desired fields: Client PO No., Project type, Status, Priority, dates, Budget, Address, City, Description.
5. Click **Create**.

The project receives a number in the format `PROJ-YYYY-NNNNN` and, by default, the "Pending" status and an empty type.

### 3.2 Search and filter

- **Search**: type a term in the "Search…" field. The filter applies to the **name**, the **description**, and the **project number** (case-insensitive).
- **Status filter**: All, Pending, In progress, Completed, Cancelled, Suspended.

### 3.3 Switch display mode

Click **List**, **Table**, or **Cards** in the header selector.

### 3.4 Open a project

Click a row (or a card) to open the detail panel. The project, its notes, its linked dossier, and its linked quote load in parallel.

### 3.5 Inline editing (status, priority, dates)

- **Status or priority**: in the List view, click the badge. A dropdown appears. The change is applied immediately (optimistic update, with automatic rollback on failure and a guard against double submissions).
- **Dates**: click the date cell (Start or End), choose a date. Saving is immediate. Clearing the date erases it (writes `null`).

### 3.6 Edit a project

Click **Edit** (pencil) in the detail panel, adjust the fields, then **Save**. Unlike the previous version, all fields of this modal are properly persisted (the project type is editable).

### 3.7 Duplicate a project

Click **Duplicate** (icon `Copy`). The new project is named "Copy of …", receives a new number, and the forced "Pending" status. Duplication **does not copy** the phases, notes, or assignments. A message confirms the creation with the new identifier.

### 3.8 Delete a project

Click **Delete** (icon `Trash2`) then confirm. Deletion destroys, in cascade, the data specific to the project (phases, notes, operations, materials, jobsite logs, etc.) and **detaches** (sets to `NULL`, without destroying them) the accounting and sales documents: journal entries, expenses, quotes, invoices, purchase orders, timesheets, dossiers, emails.

> **Guard**: a project whose status starts with `termin` (Completed and its variants) **cannot be deleted** (`400`). You must first move it to another status.

### 3.9 Batch-update several projects

1. Check the desired projects. The batch action bar appears.
2. Choose a status in **"Change status…"**.
3. Confirm. The new status is applied to the whole selection.

### 3.10 Export the list to CSV

Click **"Export CSV"**. The file `projets_export.csv` (up to 20,000 rows) is downloaded. Its columns: ID, Numéro Projet, Nom Projet, Statut, Priorité, Type, Client, Date Début, Date Fin, Budget Total, Description, Adresse Chantier, Ville Chantier, Créé le, Modifié le. (The column headers are emitted in French by the export routine.) Text cells are protected against formula injection. There is **no** PDF export or printing in this module.

### 3.11 Generate a schedule (work order)

In the detail panel, click **"Generate schedule"**. The system creates a work order pre-filled with standard operations, chosen based on the **project type** (residential, renovation, commercial, institutional, public). Especially useful for projects created directly, without going through a quote. The action is idempotent (a single automatic schedule per project). This action requires write access to Work Orders.

### 3.12 Navigate to the 360° Record

If a dossier is linked, click **"View 360° Record"** to open `/dossier/{id}`: documents, communications, extras, and a 360-degree view of the jobsite.

### 3.13 Consult the financial summary

Open the **Financials section** of the panel (Show button). The four cards (Revenue, Expenses, Margin, Budget) and their details are computed on the fly from the invoices, purchase orders, and timesheets attached to the project.

### 3.14 Add a note

Notes section → **"Add"** → enter Title, Content and, if needed, a Category → **Add**.

### 3.15 Categorize a note with AI

Click **"Categorize with AI"** (icon `Bot`) under a note. Claude (Opus 4.8) classifies the note into one of ten construction categories: Technical, Safety, Budget, Planning, Quality, Communication, Environment, HR, Procurement, Other. It also assigns an importance level. The cost is deducted from the tenant's prepaid AI credits.

### 3.16 Use the AI Assistant

1. Click **"AI Assistant"** in the command bar.
2. Ask a question about the projects (progress, budgets, deadlines) or request the creation of a project.
3. For a read query, the assistant answers directly from the real data.
4. For a creation, the assistant shows a **proposal card**. Check the fields, then click **Confirm** (the project is created) or **Cancel**.

### 3.17 Open a project from an external link

A link of the form `/projets?open={id}` (for example from the calendar) automatically opens the detail panel of the relevant project.

---

## 4. Reference

### 4.1 Endpoints (22 in total)

Real path prefix: `/api/erp/v1`. The CRUD is under `/projects` (English), the assistant under `/projets/ai` (French) — a naming inconsistency accepted in the code.

**CRUD and read — `routers/projects.py`**

| Method | Path | Function (line) | Role |
|---|---|---|---|
| GET | `/projects` | list_projects (431) | Paginated list + filters (search name/description/number, status, priority, company). |
| GET | `/projects/statistics` | get_project_statistics (515) | KPIs: total, in progress, completed, total budget. |
| POST | `/projects/duplicate/{id}` | duplicate_project (564) | Duplicates a project (status forced to "Pending"). |
| GET | `/projects/export-csv` | export_projects_csv (674) | CSV export (20,000 rows maximum). |
| POST | `/projects/batch-update` | batch_update_projects (757) | Batch update of status or priority (1 to 1000 projects). |
| GET | `/projects/gantt` | get_gantt_data (800) | Gantt data (projects + phases). **Consumed by the Tracking module.** |
| GET | `/projects/{id}` | get_project (882) | Detail: project + client + phases + assignments. |
| GET | `/projects/{id}/financials` | get_project_financials (983) | Financial summary of actual costs. |
| POST | `/projects` | create_project (1266) | Create a project. |
| PUT | `/projects/{id}` | update_project (1384) | Edit a project (partial). |
| GET | `/projects/{id}/dossier` | get_project_dossier (1445) | Linked 360° Record. |
| POST | `/projects/{id}/phases` | create_phase (1489) | Create a phase (via API only). |
| PUT | `/projects/{id}/phases/{phaseId}` | update_phase (1538) | Edit a phase (via API only). |
| GET | `/projects/{id}/notes` | list_project_notes (1582) | List the notes. |
| POST | `/projects/{id}/notes` | create_project_note (1623) | Create a note. |
| POST | `/projects/{id}/notes/{noteId}/categorize` | categorize_project_note (1668) | Categorize a note with AI (billed). |
| GET | `/projects/{id}/assignments` | list_project_assignments (1794) | List employee assignments (via API only). |
| POST | `/projects/{id}/assignments` | add_project_assignment (1834) | Assign an employee (via API only). |
| DELETE | `/projects/{id}/assignments/{assignmentId}` | remove_project_assignment (1883) | Remove an assignment (via API only). |
| DELETE | `/projects/{id}` | delete_project (1921) | Delete a project ("completed" guard). |

**AI Assistant — `routers/projets_ai.py`**

| Method | Path | Function (line) | Role |
|---|---|---|---|
| POST | `/projets/ai/chat` | projets_ai_chat (289) | AI chat: read + project proposal (writes nothing). Billed. |
| POST | `/projets/ai/confirm-action` | confirm_projets_action (428) | Executes a confirmed proposal → delegates to `create_project`. |

**Related endpoint (outside the module, called by the "Generate schedule" button)**

| Method | Path | File | Role |
|---|---|---|---|
| POST | `/production/projects/{id}/generate-cedule` | production.py:2810 | Creates an idempotent work order from an operations template. Protected by the Work Order write role. |

> **What does not exist**: no DELETE endpoint for a phase (only POST and PUT). No PUT or DELETE endpoint for a note (only GET, POST, and categorization). A created phase or an added note therefore cannot be deleted through the interface or the API (they disappear only when the project is deleted, which erases them in cascade).

### 4.2 Statuses and priorities

**Statuses** (canonical value in the database: `En attente`, `En cours`, `Termine`, `Annule`, `Suspendu`):

| Display | Badge color |
|---|---|
| Pending | Yellow |
| In progress | Blue |
| Completed | Green |
| Suspended | Amber |
| Cancelled | Red |

**Priorities**: Low, Medium, High, Urgent. The system also tolerates old values inherited from a quote (NORMAL, URGENT, CRITIQUE) and maps them back to the canonical form.

> **Tolerant normalization**: on write (creation, editing, batch), variants of case, accents, or old format (for example `EN_ATTENTE`) are converted to the canonical form. There is **no state machine**: you can move freely from one status to another. The only status-related rule is the ban on deleting a completed project.

### 4.3 Project types and schedule template

The project type drives the operations template used by "Generate schedule":

| Type (menu) | Effect |
|---|---|
| *(empty)* | Residential by default. |
| Residential — New construction | Residential new-build template. |
| Residential — Renovation | Renovation template. |
| Commercial — New construction | Commercial new-build template. |
| Commercial — Renovation | Commercial renovation template. |
| Institutional | Institutional template. |
| Public | Public template. |

### 4.4 Calculations

**Statistics (KPIs)** — `GET /projects/statistics` groups by status:
- `total` = sum of all projects;
- `en_cours` = count with the exact status `En cours`;
- `termines` = count with the exact status `Termine`;
- `budget_total` = sum of the `budget_total` values.

> Nuance: this grouping is done on the **raw** status (not retroactively normalized). A very old project left in a non-canonical format in the database would form its own bucket and would not be counted under "in progress" or "completed".

**Financial summary (actual costs)** — `GET /projects/{id}/financials`. The budget is a **stored** field; the actual costs are **not** stored, they are recalculated on every call:

| Block | Source | Rule |
|---|---|---|
| Budget | `projects.budget_total` | Returned as-is. No budget-vs-actual variance calculation. |
| Revenue — quotes | Linked accepted quotes | Informational only (not counted in total revenue). |
| Revenue — invoices | **Client** invoices (excluding Cancelled and Draft) | **Pre-tax** basis. A credit note is subtracted. |
| Expenses — materials | Purchase orders (excluding cancelled and draft) | Excluded if already invoiced by the supplier (anti double-counting). |
| Expenses — labor | Logged time entries | Hours × (hourly rate, or salary ÷ 2,080). |
| Expenses — supplier | Supplier invoices (includes drafts) | Pre-tax basis, credit note subtracted. |
| **Total revenue** | = total of client invoices. | Quotes are not counted. |
| **Total expenses** | = materials + labor + supplier invoices. | |
| **Margin** | = revenue − expenses. | |
| **Margin %** | = margin ÷ revenue × 100 (0 if revenue ≤ 0). | |

**Gantt progress** (Tracking module) — a project's progress is **derived**: it is the average of its phases' progress values (rounded to one decimal), or 0 if it has no phase. It is never stored on the project.

**Markup on the linked quote** (detail panel, display) — multiplier = 1 + (administration + contingencies + profit) ÷ 100, with 3%, 12%, and 15% by default, overridable line by line. This calculation is display-only: it does not touch the quote.

### 4.5 Limits and bounds

| Element | Limit |
|---|---|
| Pagination | 1 to 100 per page (default 20). |
| Search | Case-insensitive substring on name + description + number. |
| Gantt | 500 projects, excludes cancelled projects. |
| CSV export | 20,000 rows. |
| Batch update | 1 to 1000 projects per call. |
| Project name | 300 characters, non-empty. |
| Description | 20,000 characters. |
| Budget | 0 to 999,999,999,999.99 (anti-overflow bound). |
| Note — title / content | 300 / 20,000 characters. |
| AI Assistant message | 1 to 8,000 characters. |
| AI Assistant rate | 20 requests/minute (chat), 30/minute (confirmation), per IP address. |

### 4.6 AI models and costs

| Surface | Model | Max tokens | Rate (before markup) | Markup |
|---|---|---|---|---|
| Note categorization | `claude-opus-4-8` | 32,000 | $5 / $25 per million (input / output) | +30% |
| AI Assistant (chat) | `claude-sonnet-4-6` | 8,000 | $3 / $15 per million | +30% |

Both surfaces are billed to the tenant's **prepaid credits**, protected by the AI guardrail and the balance check (`402` if exhausted). Every use is logged.

### 4.7 Field persistence on creation

| Modal field | Saved? |
|---|---|
| Project name | Yes |
| Client PO No. | **Yes** |
| Client (Company) | Yes |
| Client (Person) | **Yes** |
| Manual entry | **Yes** (freezes the client name) |
| Project type | Yes |
| Status, Priority | Yes |
| Start, End, Budget | Yes |
| Address, City, Description | Yes |

> **Correction of stale information**: earlier versions of this manual claimed that the Client PO No., the Client (Person), and the Manual entry were "silently ignored". That is **false** in the current code: `create_project` does insert `po_client`, `client_contact_id`, and `client_nom_direct`, and resolves the client name to freeze. Likewise, the Edit modal no longer has a "Manager" or "Notes" field: all of its fields are persisted.

### 4.8 Error codes

| Code | Cause |
|---|---|
| 400 | Missing tenant context / empty name / deletion of a completed project. |
| 402 | AI credits exhausted. |
| 403 | AI access denied, or write blocked in consultation mode. |
| 404 | Project, phase, note, or assignment not found. |
| 409 | Employee already assigned (duplicate assignment). |
| 500 | Internal error. |
| 503 | AI service unavailable or temporarily overloaded. |

---

## 5. Integrations and FAQ

### 5.1 Integration map

| Linked module | Nature of the link |
|---|---|
| **CRM (Companies / Contacts)** | Feeds the Client (Company) and Client (Person) lists at creation. The client name is frozen (`client_nom_cache`) then resolved at display time. |
| **Quotes** | Two links. **Inbound**: accepting a quote automatically creates a project (see §5.2). **Outbound**: the detail panel shows the linked quote in read-only mode. |
| **360° Record** | `dossier_projets` association table → "View 360° Record" button. |
| **Work Orders / Production** | "Generate schedule" button → work order pre-filled with operations based on the project type. |
| **Accounting** | The Financials section aggregates client invoices, purchase orders, timesheets, and supplier invoices to compute the actual costs and the margin. |
| **Tracking** | Consumes `GET /projects/gantt` to display the Gantt and the progress. The Gantt is **not** rendered on the Projects screen. |
| **Calendar** | An `?open={id}` link opens a project directly. |

### 5.2 Automatic conversion of a quote into a project

Creating a project from a quote is triggered **on the Quotes side**, not from this screen (there is no "Convert" button in the Projects module). Three paths trigger the `_create_project_from_devis` helper (devis.py:11884):

1. A quote moves to "Accepted" status (automatic creation).
2. The explicit endpoint `POST /devis/{id}/convert-to-project` (idempotent).
3. The public acceptance of a quote by the client, via their signed link (without authentication).

Particularities of a project derived from a quote, which distinguish it from a manual creation:

- The **budget** is the quote's total investment amount (with taxes), recalculated if necessary.
- The **status** is forced to "In progress" (not "Pending"); the default **type** is "Construction".
- The project is linked to the won CRM opportunity, the quote's attachments are copied over, and a work order is generated along the way (best-effort, without ever blocking the conversion).

### 5.3 What is not available on this screen

- **No Kanban / Gantt / Calendar / Statistics tabs**: the screen offers three **display modes** (List / Table / Cards), not planning tabs. The interactive Gantt lives in the **Tracking** module.
- **Phase management in write mode**: phases are shown read-only. The create and edit capabilities exist in the API, but no button exposes them here, and creating a project generates no phase automatically.
- **Employee assignment management**: the API can list, add, and remove assignments, but no element of the Projects screen uses them.
- **No PDF export, no printing, no file upload**: documents go through the linked 360° Record.
- **The AI Assistant does not edit or delete**: it only reads and proposes a creation.
- **The linked quote is read-only**: it is edited in the Quotes module.

### 5.4 FAQ

**Q: Where is the module in the menu?**
A: Sidebar, **Management** section, **Projects** entry (icon `Briefcase`). It is not in the "Tools" section.

**Q: How is a project created automatically?**
A: By accepting a quote. The project is then born with the "In progress" status and the quote's budget. See §5.2.

**Q: Where does the project number come from?**
A: It is generated in the format `PROJ-YYYY-NNNNN` from the internal identifier and the year. Old projects without a number are completed automatically on first viewing.

**Q: Can I delete a phase or a note?**
A: No. There is no deletion endpoint for phases or notes. They are erased only when the project is fully deleted (cascade). A phase can, however, be edited via the API (`PUT`).

**Q: Why can't I delete a "Completed" project?**
A: A business guard forbids it (returns `400`). First move the project to another status.

**Q: Do accepted quotes count in the Financials section's revenue?**
A: No. Total revenue comes only from client invoices (pre-tax basis). Accepted quotes are shown for information.

**Q: Why is the margin calculated on pre-tax amounts?**
A: To compare pre-tax with pre-tax. The taxes collected (GST, QST) are not revenue; including them would inflate the margin by roughly 13%.

**Q: Does the Financials section show me the gap between my budget and my actual costs?**
A: No. It displays the budget and the actual costs side by side, but computes no budget-completion percentage or automatic variance.

**Q: Who can create, edit, or delete a project?**
A: Any authenticated user of the tenant, regardless of their role. The only exception: consultation mode (suspended subscription) blocks writes, and schedule generation requires write access to Work Orders.

**Q: What is the format of the exported CSV file?**
A: `projets_export.csv`, UTF-8 encoding, comma separator, 15 columns. The "Notes" column that duplicated the description in the old version has been removed.

**Q: How much does categorizing a note with AI cost?**
A: It uses Claude Opus 4.8 and is billed to prepaid credits according to the tokens consumed (rate $5 / $25 per million, plus a 30% markup). In practice, a classification produces very little text, so the cost per note is minimal.

**Q: Can the AI Assistant edit an existing project?**
A: No. Version 1 is limited to reading and proposing a creation. Editing by the AI is planned but not implemented.

---

## 6. Summary

- **Mission**: manage the project record (master record of the jobsite) — list, creation, editing, duplication, deletion, KPIs, AI-categorized notes, financial summary of actual costs, gateways to 360° Record / Schedule / Quote.
- **Access**: sidebar → **Management** → **Projects** (route `/projets`).
- **Source code**: `projects.py` (2037 lines, 20 endpoints), `projets_ai.py` (460 lines, 2 endpoints), `ProjectsPage.tsx` (1249 lines), `ProjetsAssistantTab.tsx` (231 lines).
- **Three display modes**: List (inline editing of status, priority, and dates), Table (compact), Cards. No Kanban / Gantt / Calendar tabs.
- **Statuses**: Pending / In progress / Completed / Suspended / Cancelled. **Priorities**: Low / Medium / High / Urgent. Tolerant normalization of old values.
- **Two origins for a project**: manual creation (status "Pending") or automatic conversion of an accepted quote (status forced to "In progress", quote's budget).
- **Financials**: actual costs computed on the fly (pre-tax client invoices, purchase orders, labor, supplier invoices); stored budget; no budget-vs-actual variance calculation.
- **AI**: note categorization (Opus 4.8) and chat assistant (Sonnet 4.6), billed to prepaid credits, propose → confirm model, project creation only.
- **Permissions**: no role check on the module (any tenant user can do everything), except consultation mode and the Work Order write access for the schedule.
- **What does not exist**: phase or note deletion; editing of phases and assignments from the screen; PDF export or printing; editing by the AI; planning tabs (the Gantt lives in the Tracking module).

---

*Sources verified on 2026-07-07: `ERP_REACT/backend/routers/projects.py`, `ERP_REACT/backend/routers/projets_ai.py`, `ERP_REACT/backend/routers/production.py` (schedule generation), `ERP_REACT/backend/routers/devis.py` (quote → project conversion), `ERP_REACT/frontend/src/pages/ProjectsPage.tsx`, `ERP_REACT/frontend/src/components/projets/ProjetsAssistantTab.tsx`, `ERP_REACT/frontend/src/api/projects.ts`, `ERP_REACT/frontend/src/api/projetsAi.ts`.*
*Related manuals: Module 07 — Quotes · Module 08 — Tracking (Gantt / Kanban) · Module 10 — Work Orders · Module 11 — Real Estate · Accounting module · Dossiers module (360° Record).*
*Constructo AI ERP Manual — Module 09 — Construction Projects — v3.0 — 2026-07-07*
