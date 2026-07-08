# Module 12 — Time Tracking and Hours

> **Version** : 3.0 (complete overhaul verified against the actual source code)
> **Reference code** :
> - Frontend: `frontend/src/pages/PointagePage.tsx` (≈ 1,907 lines, a single page with **7 tabs**), `frontend/src/components/pointage/PointageAssistantTab.tsx` (AI Assistant tab), API `frontend/src/api/employees.ts` (time sheets), `frontend/src/api/payroll.ts` (CCQ payroll), `frontend/src/api/feuillets.ts` (T4 / RL-1 / PD7A slips), `frontend/src/api/pointageAi.ts` (assistant), `frontend/src/api/projects.ts` and `frontend/src/api/production.ts` (Project / Work order / Operation drop-down menus)
> - Backend: `backend/routers/employees.py` (the heart of time tracking — endpoints `/time-entries*` and `/payroll-summary`, ≈ lines 579 to 1615; **not** `production.py`), `backend/routers/pointage_ai.py` (≈ 300 lines, read-only assistant), `backend/routers/payroll.py` (CCQ payroll, Payroll module), `backend/routers/feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (tax slips), `backend/routers/payroll_rates.py` (contribution rates)
> - API prefix: `/api/erp/v1` — time sheets respond under `/employees` (English), the assistant under `/pointage/ai`, payroll under `/payroll`, tax slips under `/feuillets`
> **PostgreSQL tables (per tenant)** : `time_entries` (the master table of time entries), `employees` (source of the frozen hourly rate), `projects` (the linked project), `formulaires` (the work order) and `companies` (the client, via the work order), `operations` (the linked operation), `payroll_periods` (closed-period lock), `payroll_entries` (generated CCQ pay records)
> **Scope** : this module is the **workstation for entering and managing hours worked** on a desktop computer. It is used to **enter** a time sheet on behalf of an employee (clock-in time, clock-out time, project, work order, operation), to **approve** it, to **edit** it, to **delete** it, to review hours **by week** and **by project**, to produce a **quick payroll summary** (approximate), to trigger the **complete CCQ payroll** (periods, detailed records, PDF pay stubs), to generate the **year-end tax slips** (T4, RL-1, PD7A), and to query an **AI assistant** about hours volume. This is **not** a real-time time clock: the employee's geolocated **clock in / clock out** in the field is done exclusively in the **mobile application** (`/mobile`), not here. Desktop entry is **manual**. The module **does not compute** an invoice (the link with invoicing is passive — see §5.4) and **does not access any GPS geolocation** (GPS tracking concerns the vehicle fleet, an entirely different module — see §5.6).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step procedures](#3-step-by-step-procedures)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The Time Tracking module (page `/pointage`, displayed title **"Time Tracking & Payroll"**) is the company's **hours register** and the **starting point of payroll**. It lets an authorized person (administrator, foreman, payroll manager):

- **enter** a time entry on behalf of an employee: clock-in time, clock-out time, project, work order, operation, notes, and the "billable" indicator;
- **approve** each time entry before it flows into invoicing or payroll;
- **edit** or **delete** a time entry as long as it is neither invoiced nor frozen by a closed payroll period;
- review hours **by week** (a Sunday-to-Saturday time sheet, aligned with the CCQ calendar) with a **per-employee capacity** table;
- review hours **by project** (an aggregate with drill-down by employee);
- produce a **quick payroll summary** over 7 to 90 days (**approximate** figures);
- trigger the **complete CCQ payroll**: create a period, compute the records (regular and overtime hours, deductions, employer contributions, CCQ), review each pay stub and **download it as a PDF**, then close the period (irreversible action);
- generate the **year-end tax slips**: **T4** (federal, CRA), **RL-1** (Quebec, Revenu Québec) and the **PD7A source-deductions remittance** statement;
- **export** time entries to a CSV file;
- query an **AI assistant** that answers only about **aggregated hours volumes** (never about payroll amounts).

> **Time tracking is the source of truth for hours.** Labour cost, client invoicing and CCQ payroll all rely on the `time_entries` table. An erroneous entry propagates downstream. This is why **entry, approval, editing and deletion** are restricted to **approvers** (see §1.4), and why a time entry that is **invoiced** or caught in a **closed payroll period** becomes **locked**.

### 1.2 What the module does NOT do

- **No real-time time clock on the web.** There is **no** "Clock in" / "Clock out" button or break timer on the `/pointage` page. Desktop entry is **manual**: you type the clock-in time and the clock-out time into two date-time fields. The real time clock (live "punch", with geolocation) is the **mobile application** (`/mobile`).
- **No employee self-service entry on the web.** On the desktop, only an approver can create a time entry, choosing the employee from a list. For an employee to clock themselves, they use the mobile application with their **4-digit PIN**.
- **No GPS geolocation in this module.** The `/pointage` page reads **no** GPS position. Company GPS tracking concerns the **vehicle fleet and logistics** (geographic zones, routes) — a separate module, unrelated to time sheets. Geolocation of a time entry exists **only** in the mobile application.
- **The "Payroll summary" is not real payroll.** It is an **estimate** at flat rates, with no ceilings, no exemptions, no income tax and no employer contributions. Publishable figures are produced in the **CCQ Payroll** tab (see §2.6).
- **No automatic meal-break deduction.** The system does not remove "30 minutes after 6 hours." For an unpaid break, enter two separate time entries or adjust the duration.
- **No lateness or absence detection.** The module records what is entered, without comparing it to a planned schedule.
- **No bulk approval.** You approve one time entry at a time; there is no "Approve all" button.
- **No file upload.** You attach no document (photo, signed note) to a time entry in this module.
- **No printing of the time-entry list.** The list is exported to **CSV**. The only PDFs produced are the **CCQ pay stubs** and the **T4 / RL-1 / PD7A slips**.
- **No reopening of a closed payroll period.** Closing is **irreversible**; to correct, use an adjustment period (Payroll module).
- **No automatic carry-over** of tracked hours to the actual hours of a work order's operations (that entry stays manual in Module 11).

### 1.3 Access

- **Side menu** → **Operations** section → **Time Tracking** (clock icon). The entry sits next to Store, Employees, Work Orders and Accounting.
- **Address** : `/pointage`.
- Protected page: you must be authenticated in a tenant.
- **Default tab** : "Time Entries".
- The page has **7 tabs** (see §1.5).

### 1.4 Permissions and roles

**Viewing** hours is open to any authenticated user of the tenant. Any **write** action (and the payroll summary) is restricted to **approvers**. An "approver" is, in the server's terms (`_require_timecard_approver`, `employees.py:310`):

- a tenant **administrator** (`is_admin` — the real signal, re-read on the server; many owners log in with the "user" role but are administrators), **or**
- a user with **role** `admin` or `super_admin`, **or**
- a platform **super-administrator** (`user_type = super_admin`).

A **plain employee** is excluded (403 error).

| Tier | Who | What they can do |
|------|-----|------------------|
| **Viewing** | Any authenticated user of the tenant | View the time-entry list, the weekly view, the by-project view, the CSV export and the AI assistant. **Cannot** create, approve, edit or delete a time entry. |
| **Approver** | Administrator (`is_admin`), role `admin` / `super_admin`, or super-administrator | Everything above, **plus**: **create**, **approve**, **edit** and **delete** a time entry; open the **Payroll summary**; run **CCQ Payroll**; generate the **tax slips**. |

> **The interface reflects the role, but the server remains the authority.** For a user who is not an approver, the interface **hides** the "New entry" button, the "Approve" buttons, the edit and delete icons, and the **"Tax slips"** tab. Even by bypassing the interface, the server refuses (403) any write by a non-approver.
>
> **A known minor inconsistency.** The **"Payroll summary"** and **"CCQ Payroll"** tabs stay **visible** to a non-approver user (only the Tax slips tab is hidden). But the payroll-summary endpoint returns 403 to non-approvers: a standard employee who opens the "Payroll summary" tab will therefore see a **"Error loading payroll"** message. This is not a calculation bug; it is the protection that prevents an employee from reading their colleagues' payroll.
>
> **View-only mode (read-only subscription).** If the tenant is in **view-only mode** (suspended subscription), **all** writes are blocked (403), including creating / approving / editing / deleting time entries, generating payroll, and sending a message to the AI assistant. Reading remains possible.

### 1.5 The 7 tabs

The page is a tab bar. Here is the exact order, each tab's role and the backend module it calls:

| # | Tab | Icon | Role | Backend | Visible to |
|---|-----|------|------|---------|-----------|
| 1 | **Time Entries** | clock | List, create, approve, edit, delete, search, filters | `employees.py` | everyone (actions restricted to approvers) |
| 2 | **Week View** | calendar | Sunday-to-Saturday time sheet + per-employee capacity | `employees.py` | everyone |
| 3 | **By Project** | briefcase | Hours aggregated by project + drill-down by employee | `employees.py` | everyone |
| 4 | **Payroll Summary** | dollar | **Approximate** payroll over 7 to 90 days | `employees.py` | everyone (data restricted to approvers) |
| 5 | **CCQ Payroll** | calculator | **Complete** payroll: periods, detailed records, PDF pay stubs | `payroll.py` | everyone (actions restricted to approvers) |
| 6 | **Tax Slips** | spreadsheet | Year-end T4, RL-1 and PD7A | `feuillets_t4/rl1/pd7a.py` | **approvers only** |
| 7 | **AI Assistant** | sparkles | Read-only chat about hours volume | `pointage_ai.py` | everyone |

> **One screen, four backend modules.** This page crosses four distinct routers: `employees.py` (the core: time sheets + payroll summary), `payroll.py` (CCQ payroll), the three slip routers, and `pointage_ai.py` (assistant). That is why some topics point to **Module 14 Accounting / Payroll** (CCQ payroll, slips) and **Module 10 Employees** (hourly rate, SIN).

### 1.6 Key concepts

- **Time entry (time sheet)** : a record in the `time_entries` table = one employee, one time range (in / out), and optionally a project, a work order and an operation.
- **Computed hours** : the duration `(out − in)` in hours, rounded to two decimals. It is computed automatically as soon as both timestamps are entered (on screen **and** on the server).
- **Rate frozen at entry (immutable)** : at creation time, the system **captures** the employee's hourly rate (or, failing that, their annual salary ÷ 2080) and **freezes** it in the time entry. A future rate change **never** retroactively recomputes an already-entered or invoiced time entry. It is this frozen value that drives cost.
- **Billable** : indicates whether the hour can be re-billed to the client. Checked by default.
- **Approved** : a time entry marked "approved" by an approver, with the trace of who and when. A prerequisite for invoicing and payroll.
- **Invoiced (lock)** : once a time entry has been included in an invoice (set by the Accounting / Files module), it becomes **immutable**: it cannot be edited, deleted or un-approved.
- **Closed payroll period (lock)** : a closed CCQ payroll period **freezes** every time entry whose date falls inside it. You can no longer insert, move, approve or delete a time entry there — which protects the year-end totals (T4 / RL-1 / PD7A).
- **CCQ week (Sunday → Saturday)** : the weekly view is **aligned with the construction payroll calendar**: it starts on **Sunday** and ends on **Saturday** (not Monday to Sunday).
- **Weekly capacity** : the target number of hours per week for an employee (default **40 h**, set on their record in Module 10). Used to colour occupancy (under-load, normal, over-load).

### 1.7 Coordination with the mobile application

Constructo AI has a **mobile application** (`/mobile`, PWA) for field time tracking. The two tools:

- **share the same `time_entries` table** : a time entry created on mobile appears in the desktop Time Entries tab, and vice versa;
- differ in their use: the **mobile** is the **real time clock** (the employee clocks their own arrival and departure, with their PIN and geolocation), while the **desktop** is for **administrative entry** and **management** (approval, correction, payroll).

> The desktop page **does not display** a "source" column: nothing visually indicates that a time entry comes from mobile rather than desktop entry. For documentation of the mobile application, see the separate MOBILE_REACT manual.

---

## 2. Interface

### 2.1 General layout

```
+---------------------------------------------------------------------+
|  Time Tracking & Payroll                                [ Export ]  |  <- title bar
+---------------------------------------------------------------------+
| [Time Entries][Week View][By Project][Payroll Summary][CCQ Payroll] |  <- 7 tabs
| [Tax Slips][AI Assistant]                                           |
+---------------------------------------------------------------------+
|                                                                     |
|   (content of the active tab)                                       |
|                                                                     |
+---------------------------------------------------------------------+
```

- **Title bar** : the title "Time Tracking & Payroll" and, on the right, an **"Export"** button (download icon) that produces the CSV file of all the tenant's time entries.
- **Alerts** : an error banner (red) or success banner (green) appears at the top when needed. **Success** messages disappear after 3 seconds; **error** messages stay displayed until the next action.

### 2.2 "Time Entries" tab (default)

This is the main view: the list of all time sheets, with creation, approval, editing and deletion.

**Command bar**

| Element | Role |
|---------|------|
| **New entry** (blue button, + icon) | Opens the creation modal. **Approvers only.** |
| **Search** (text field) | Searches **on the server** (350 ms debounce) across the employee, project, work order, client, operation and notes — over the **entire** data set, not just the displayed page. |
| **Status filter** (drop-down) | "All", "Approved", "Not approved", "Invoiced". |

**Table** (sortable, resizable columns)

| Column | Content |
|--------|---------|
| **Employee** | First and last name. |
| **Client** | Client name, linked via the work order. |
| **Project** | Project name, or blank. |
| **WO** | Work order number, or blank. |
| **Operation** | Operation name (or description), or blank. |
| **Start** | Clock-in time. |
| **End** | Clock-out time. |
| **Hours** | Computed duration. |
| **Status** | See below. |
| **Actions** | Edit and delete icons (approvers). |

**"Status" column**

- Blue **"Invoiced"** badge if the time entry is invoiced (read-only).
- Otherwise, a green **"Approved"** badge (with a check) if it is approved.
- Otherwise, an **"Approve"** button (for an approver) or a grey **"To approve"** badge (for a non-approver).

**"Actions" column** (approvers)

- **Pencil** = edit; **trash** = delete.
- Both icons are **disabled** if the time entry is invoiced (tooltip "Time entry invoiced — locked").
- Deletion asks for confirmation ("Delete this time entry?").

**Empty state** : "Start by recording a time entry." with a "New entry" button.

**Pagination** : 20 time entries per page.

**"New time entry" modal**

| Field | Detail |
|-------|--------|
| **Employee \*** | Required drop-down. |
| **Project** | Drop-down (option "-- No project --"). |
| **Work order** | Drop-down (option "-- No WO --"), format "number — project". |
| **In (Punch In)** | Date-time field. |
| **Out (Punch Out)** | Date-time field. |
| *Computed hours* | Automatic box "Computed hours: {N} h" as soon as the duration is positive. |
| **Notes** | Text area. |
| **Billable** | Check box (checked by default). |

**Cancel** / **Create** buttons ("Create" disabled until an employee is chosen). Input controls: employee required; you must supply **both** timestamps or **neither**; the out time must follow the in time; the duration cannot exceed **24 h**.

> **The creation modal does not let you choose the operation.** To link a time entry to a specific operation, create it first with its work order, then use **Edit** (see §3.2).

**"Edit time entry" modal** (richer than creation)

- Grid: **Employee**, **Project**, **Work order**, **Operation** (drop-down **dependent** on the work order: it fills once the WO is chosen; label "Operation (loading...)" while loading, "-- Select a WO --" if there is no WO).
- **In** / **Out** : date-time fields to the **second**.
- **"Computed hours"** box + **"Work type"** field (free text, e.g. "Installation", "Repair").
- **Notes** (text area).
- **"Billable"** and **"Approved"** check boxes (so you can approve or un-approve directly here).
- If the time entry is invoiced: the note **"Already invoiced — edits refused server-side"** (lock icon).
- Saving sends **only the changed fields**; clearing Project / WO / Operation detaches them.

### 2.3 "Week View" tab

Weekly time sheet, **Sunday to Saturday** (CCQ-aligned).

- **Navigation** : **"Previous week"** and **"Next week"** buttons (± 7 days), the range "{start} to {end}", and a **"Total: {N} h"** badge.
- **Days table** (7 rows) : **Day**, **Date**, **# Entries**, **Total Hours**; footer row **"Week total"**. Days at 0 h are greyed out.
- **"Weekly capacity per employee" card** : legend "Green < 80% · Yellow 80-100% · Red > 100% (overloaded)". Columns: **Employee**, **Hours worked**, **Capacity**, **Occupancy** (coloured percentage) and **Load** (progress bar). Default capacity is **40 h** (adjustable per employee in Module 10); an employee at 100% or more is flagged "overloaded".

### 2.4 "By Project" tab

Aggregate table: **Project**, **Hours** (sum), **# Employees** (distinct employees). The rows are **clickable** (keyboard-accessible): a click **expands** the detail by employee (name + that employee's hours on that project). The aggregate covers **all** the tenant's time entries (no date filter); it shows the top 20 projects by descending hours.

> This view **excludes** time entries with no project (only project-linked time entries appear here).

### 2.5 "Payroll Summary" tab (approximate)

A **quick, estimated** payroll view.

- **Period selector** : **7 days**, **14 days**, **30 days** (default) or **90 days**.
- **Two cards** : **"Gross payroll"** ($) and **"Employees"** (count).
- **Table** : **Employee**, **Dept.**, **Hours**, **Rate** ($/h), **Gross**, **Deductions** (in red), **Net** (in green). Card layout on phone.

> **Approximate figures — do not publish.** The calculation is deliberately simplified (`employees.py:897` : "flat employee rates, no caps/exemption"). The only deductions applied are **RRQ (Quebec Pension Plan) 6.30%**, **RQAP (Quebec Parental Insurance Plan) 0.43%** and **EI (Employment Insurance) 1.30%** on the gross (≈ 8.03% total). **No ceiling**, **no exemption** (RRQ $3,500), **no** federal or provincial **income tax**, **no employer contributions** (CNESST, FSS, CCQ). Only **active** employees are counted. For real payroll, use the **CCQ Payroll** tab.
>
> **Restricted to approvers.** A non-approver user sees this tab but gets an error on load (see §1.4).

### 2.6 "CCQ Payroll" tab (complete payroll)

Actual payroll, with detailed deductions, employer contributions and CCQ. The detail of the rates and tax brackets is documented in **Module 14 Accounting / Payroll**; here is the surface of this tab.

- **Period selector** : the list of payroll periods, in the format "{start} to {end} — {type} [CLOSED]". Three **types** exist: **Weekly** (52 periods/yr), **Bi-weekly** (26/yr) and **Monthly** (12/yr).

**Actions** (on a **non-closed** period)

| Button | Effect |
|--------|--------|
| **New period** | Opens a modal: **Start date \***, **End date \***, **Period type**. |
| **Compute payroll** (calculator icon) | Generates the period's records. If a draft already exists, a confirmation warns that it will be **replaced**. |
| **Close period** (lock icon, red) | Confirmation "Close this payroll period? This action is irreversible." Once closed, a **"Period closed"** badge appears and the action buttons disappear. |

**Four total cards** (if records exist) : **Employees**, **Gross payroll**, **Net payroll**, **Employer cost**.

**Table** : **Employee**, **Dept.**, **Reg. H**, **OT H** (in orange), **Gross**, **Deductions**, **Net**, **Employer Cost**, **CCQ** (Yes/No badge) and **Record** (icon that opens the pay stub). Card layout on phone.

**"Pay record" modal** (detailed pay stub)

- **Header** : name, position — department, period, type.
- **Hours worked** : regular, overtime (in orange), hourly rate.
- **Three cards** : Gross pay, Net pay (green), Employer cost (purple).
- **Earnings** : "Vacation pay paid" (if applicable).
- **Employee deductions** : Federal tax, Quebec provincial tax, **RRQ (6.40%)**, **RRQ2 (4.00%)** (if applicable), **RQAP (0.494%)**, **EI (1.32%)**, Total deductions.
- **Employer contributions** : Employer RRQ (6.40%), Employer RRQ2 (4.00%) (if applicable), Employer RQAP (0.692%), Employer EI (1.848%), **CNESST (1.80%)**, **FSS (Health Services Fund) (1.65%)**, **CCQ (12.5%)** (Applicable / N/A badge), Total contributions.
- "Accrued vacation" (informational, if applicable).
- **Close** and **"Download PDF pay stub"** buttons.

> The percentages above are the **labels displayed on the pay stub**. Rate governance (tax brackets, ceilings, exemptions, CCQ by trade) belongs to the Payroll module — see **Module 14**.

### 2.7 "Tax Slips" tab (approvers only)

Production of year-end slips. A **"Tax year"** field (number, from 2000 to 2100) drives the three cards, loaded in parallel.

**"T4 (federal — CRA)" card**

- **"Generate"** button.
- **Warnings** (amber banner) in case of missing data (for example a missing SIN).
- **T4SUM summary** : "Slips produced", **Box 14 — Income**, **Box 22 — Federal tax**, **Box 17 — RRQ**, **Box 18 — EI**, **Box 55 — RQAP**.
- Per-employee list with an individual **PDF download** button.

**"RL-1 (Quebec — Revenu Québec)" card**

- **"Preliminary"** badge.
- **"Generate"** button.
- **Summary** : **Box A — Income**, **Box E — Quebec tax**, **Box B — RRQ**, **Box H — RQAP**.
- Per-employee list with individual **PDF**.

**"PD7A — Source-deductions remittance (DAS)" card**

- **"Month"** drop-down (1 to 12) and **"Download PDF"** button.
- **Three cards** : "To remit — CRA (federal)", "To remit — Revenu Québec", "Total to remit".
- Empty if no payroll has been generated for the chosen month.

### 2.8 "AI Assistant" tab

**Read-only** and **non-monetary** chat about hours volume.

- **Header** : "AI Assistant — Hours tracking", subtitle "Volume of tracked hours from non-monetary aggregates (read-only)."
- **Empty state** : three sample questions ("How many hours were tracked in total?", "What is the breakdown of hours by work type?", "How many billable hours this week?") and a **privacy note** : the assistant accesses no payroll amount.
- **Chat** : user / assistant bubbles with metadata (profile "Time Tracking", tokens, cost, duration), an "Analyzing..." indicator, an input area (Enter to send, Shift+Enter for a line break) and a **"Send"** button. A lock prevents double-sending.
- **Bilingual** (French / English depending on the interface language).

> **Strictly limited scope.** The assistant has **no SQL query tool**. The server provides it with a **fixed aggregated context** (total hours, hours by work type, hours by billability, last 7 days) that **never** reads the hourly rate, cost, salary or SIN. See §4.13 for cost.

---

## 3. Step-by-step procedures

### 3.1 Create a manual time entry

1. **Time Entries** tab → **New entry**.
2. Choose the **Employee** (required).
3. If needed, choose the **Project** and the **Work order**.
4. Enter the **In** and the **Out** (both, or neither). The "Computed hours" box updates automatically.
5. Add **Notes**; leave **Billable** checked if the hour is re-billable.
6. Click **Create**. The time entry appears in the list, **to approve** by default.

> **At creation time, the server freezes the employee's hourly rate** into the time entry. If the employee has no hourly rate, the system takes their annual salary ÷ 2080. This frozen rate will no longer change, even if you edit the employee's record later.
>
> **Possible refusals** : the in time must precede the out time (otherwise an error), the duration cannot exceed 24 h, and the date must not fall in a **closed payroll period** (otherwise creation is refused). The system also prevents a **second open time entry** for the same employee (a time entry with no out time while another is already open).

### 3.2 Link an operation to a time entry

The **creation** modal does not offer the operation. To link it:

1. Create (or open) a time entry that already has a **work order**.
2. Click the **pencil** (Edit).
3. In the **Operation** menu (which filled from the work order), choose the operation.
4. **Save**.

> The server checks that the operation **actually belongs to** the chosen work order; an operation foreign to the WO, or an operation with no WO, is refused.

### 3.3 Edit a time entry

1. **Time Entries** tab → **pencil** on the desired row.
2. Adjust the fields: employee, project, work order, operation, in / out, work type, notes, billable, approved.
3. **Save**. Only the fields that actually changed are sent; if nothing changed, the modal closes without a request.

> If you change the in or the out without imposing a duration, the server **recomputes** the hours. The **cost** is recomputed from the **frozen rate** stored in the time entry, never from the employee's current rate. A time entry that is **invoiced** or caught in a **closed period** cannot be edited (400 error).

### 3.4 Approve or un-approve a time entry

**Two ways to approve** :

- **From the list** : click the row's **"Approve"** button. The badge turns green and the server records **who** approved and **when**.
- **From the edit modal** : check **"Approved"** then save.

**Un-approve** : open the edit modal and uncheck **"Approved"**. The server clears the approval trace.

> There is **no** bulk approval: one approval per time entry. Approval is **restricted to approvers** (on the server, not just on screen). It is refused on an **invoiced** time entry or one in a **closed period**. Re-approving an already-approved time entry has no effect (a no-op).

### 3.5 Delete a time entry

1. Click the row's **trash** icon.
2. Confirm ("Delete this time entry?").

> Deletion is **permanent** (no recovery bin). It is **refused** if the time entry is **invoiced** or in a **closed period** (400 error). Restricted to approvers.

### 3.6 Review the week and capacity

1. **Week View** tab → by default, the current week (Sunday → Saturday).
2. Navigate with **Previous week** / **Next week**.
3. Read the days table (entries and total per day) and the **Week total**.
4. In the **Capacity** card, spot employees in **over-load** (occupancy ≥ 100%, in red) or in **under-load** (< 80%, in green).

### 3.7 View hours by project

1. **By Project** tab → the top 20 projects by hours.
2. Click a row to **expand** the detail by employee.

> No date filter: the view aggregates the tenant's entire history. Time entries with no project do not appear.

### 3.8 Produce the quick payroll summary

1. **Payroll Summary** tab (approver).
2. Choose the period (7 / 14 / 30 / 90 days).
3. Read the gross payroll, the employee count and the table (hours, rate, gross, deductions, net).

> Reminder: **approximate** figures (RRQ 6.30% + RQAP 0.43% + EI 1.30%, no tax or employer contributions). To publish, move to CCQ Payroll.

### 3.9 Full CCQ payroll cycle

1. **CCQ Payroll** tab → **New period** : enter the **start date**, the **end date** and the **type** (weekly, bi-weekly or monthly), then create.
2. Select the period → **Compute payroll**. The system reads active employees, sums their hours over the period, separates regular from overtime, computes the deductions and contributions, and produces the records. (Repeatable as long as the period is open; a new computation **replaces** the existing draft after confirmation.)
3. Review a **Pay record** (icon in the "Record" column): hours, gross / net / cost, employee deductions, employer contributions, CCQ.
4. **Download the PDF pay stub** from the record.
5. When everything is verified, **Close the period** (confirmation; **irreversible**). Closing **freezes** the period's time entries.

> For the detail of the rates, tax brackets and CCQ, see **Module 14**.

### 3.10 Generate and download the tax slips

1. **Tax Slips** tab (approver) → enter the **Tax year**.
2. **T4** : click **Generate**. Read the T4SUM summary (boxes 14, 22, 17, 18, 55); fix the employees flagged with a **warning** (for example a missing SIN); download each individual **PDF**.
3. **RL-1** : click **Generate** (slip marked **"Preliminary"**). Read the summary (boxes A, E, B, H); download each **PDF**.
4. **PD7A** : choose the **Month** → **Download PDF**. The statement shows "To remit — CRA", "To remit — Revenu Québec" and the total.

> The slips rely on the **generated** CCQ payroll: without pay records for the period, the summaries stay empty. Each employee's SIN and address (entered in Module 10) are required for a complete slip.

### 3.11 Export time entries to CSV

1. Click **Export** (title bar).
2. The file **`pointages_export.csv`** downloads. Columns: **ID, Employee, Project, WO Number, In, Out, Hours, Type, Notes, Approved** (Yes / No).

> The CSV **contains no amount** (no rate, no cost, no salary): it therefore cannot disclose payroll data. Text fields are **neutralized** against formula injection (protection when opened in a spreadsheet). Filters (`employee_id`, `date_debut`, `date_fin`) exist in the API but are not exposed in the desktop interface.

### 3.12 Query the AI assistant

1. **AI Assistant** tab.
2. Ask a question about **hours volume** ("How many billable hours this week?").
3. Read the answer; the cost and duration appear below it.

> If you ask for a payroll amount, a rate or a salary, the assistant **cannot** answer: that data is never provided to it. Each message consumes **AI credits** (see §4.13).

---

## 4. Reference

### 4.1 The 7 tabs and their backend

| Tab | Main endpoint | Router |
|-----|---------------|--------|
| Time Entries | `/employees/time-entries` (+ create / update / delete / validate / export-csv) | `employees.py` |
| Week View | `/employees/time-entries/weekly` | `employees.py` |
| By Project | `/employees/time-entries/by-project` | `employees.py` |
| Payroll Summary | `/employees/payroll-summary` | `employees.py` |
| CCQ Payroll | `/payroll/*` | `payroll.py` |
| Tax Slips | `/feuillets/t4|rl1|pd7a` | `feuillets_t4/rl1/pd7a.py` |
| AI Assistant | `/pointage/ai/chat` | `pointage_ai.py` |

### 4.2 "Time Entries" table columns

| Column | Source |
|--------|--------|
| Employee | Employee's first + last name. |
| Client | Client name (via the work order). |
| Project | Project name. |
| WO | Work order number. |
| Operation | Operation name or description. |
| Start / End | Clock-in / clock-out time. |
| Hours | Computed duration. |
| Status | Invoiced / Approved / To approve (or Approve button). |
| Actions | Edit / Delete (approvers). |

### 4.3 Time-entry statuses

| Status | Condition | Display |
|--------|-----------|---------|
| **Invoiced** | The time entry is included in an invoice | Blue "Invoiced" badge; edit / delete / approval blocked |
| **Approved** | Approved, not invoiced | Green "Approved" badge (check) |
| **To approve** | Not approved, not invoiced | "Approve" button (approver) or grey "To approve" badge |

### 4.4 Filters and search (Time Entries tab)

| Filter | Behaviour |
|--------|-----------|
| **Search** | **On the server** (350 ms debounce) across employee, project, work order, client, operation and notes; covers the entire data set. |
| **Approved** | Only approved time entries. |
| **Not approved** | Only non-approved ones. |
| **Invoiced** | Only invoiced ones. |
| **All** | No filter (default). |

### 4.5 Modal fields

| Field | Creation | Edit |
|-------|----------|------|
| Employee | yes (required) | yes |
| Project | yes | yes |
| Work order | yes | yes |
| Operation | — | yes (depends on the WO) |
| In / Out | yes | yes (second precision) |
| Computed hours | auto display | auto display |
| Work type | — | yes (free text) |
| Notes | yes | yes |
| Billable | yes (checked) | yes |
| Approved | — | yes |

### 4.6 Validations and locks (on the server)

| Rule | Effect |
|------|--------|
| Missing employee (creation) | "Create" button disabled; refused on the server. |
| Only one of the two timestamps (creation) | Refused (supply both or neither). |
| Out before in | 400 error. |
| Duration > 24 h (input control) | Refused on screen. |
| Entered duration out of bounds | Clamped to ≥ 0 and ≤ 100,000 (anti-overflow guard). |
| Date in a **closed period** (creation or move) | Refused (400 error). |
| Second **open time entry** for the same employee | Refused (409 error). |
| Operation foreign to the work order | 400 error. |
| Operation with no work order | 400 error. |
| Edit / delete / approve an **invoiced** time entry | 400 error. |
| Edit / delete / approve in a **closed period** | 400 error. |
| Write by a **non-approver** | 403 error. |
| **View-only mode** (suspended subscription) | All writes blocked (403). |

### 4.7 Hours calculation

| Where | Formula |
|-------|---------|
| On screen | `(out − in)` in hours, rounded to 2 decimals, displayed live. |
| Server (creation) | `round((out − in) / 3600, 2)` if the duration is not supplied. |
| Server (edit) | Identical recalculation if the in or the out changes without an imposed duration. |
| Cost | `hours × frozen rate` (the rate captured at entry, never the current rate). |

### 4.8 Payroll summary — rates (approximate)

| Deduction | Rate |
|-----------|------|
| RRQ | 6.30% |
| RQAP (employee) | 0.43% |
| EI (employee) | 1.30% |
| **Total deductions** | **≈ 8.03%** of gross |

**Excluded** : ceilings, RRQ exemption, federal tax, provincial tax, CNESST, FSS, CCQ, employer contributions. **Active** employees only.

### 4.9 CCQ Payroll — percentages displayed on the pay stub

| Employee deduction | Label |
|--------------------|-------|
| RRQ | 6.40% |
| RRQ2 | 4.00% |
| RQAP | 0.494% |
| EI | 1.32% |
| Federal / provincial tax | per the brackets (Module 14) |

| Employer contribution | Label |
|-----------------------|-------|
| RRQ | 6.40% |
| RRQ2 | 4.00% |
| RQAP | 0.692% |
| EI | 1.848% |
| CNESST | 1.80% |
| FSS | 1.65% |
| CCQ | 12.5% (Applicable / N/A badge) |

> Labels as displayed on the pay stub. Governance of rates, ceilings and brackets: **Module 14**.

### 4.10 Tax slips — displayed boxes

| Slip | Summary boxes |
|------|---------------|
| **T4 (T4SUM)** | Box 14 (Income), Box 22 (Federal tax), Box 17 (RRQ), Box 18 (EI), Box 55 (RQAP) |
| **RL-1** | Box A (Income), Box E (Quebec tax), Box B (RRQ), Box H (RQAP) |
| **PD7A** | To remit — CRA, To remit — Revenu Québec, Total to remit (by month) |

### 4.11 CSV export — columns

`ID`, `Employee`, `Project`, `WO Number`, `In`, `Out`, `Hours`, `Type`, `Notes`, `Approved` (Yes / No). File `pointages_export.csv`. **No amount** exported.

### 4.12 Endpoints (API)

**Time sheets** (base `/api/erp/v1/employees`) :

| Method + path | Required role | Role |
|---------------|---------------|------|
| `GET /employees/time-entries` | Any user of the tenant | Paginated list + search + filters. |
| `POST /employees/time-entries` | **Approver** | Create (frozen rate, period / double-open-entry locks). |
| `GET /employees/payroll-summary` | **Approver** | **Approximate** payroll summary. |
| `PUT /employees/time-entries/{id}/validate` | **Approver** | Approve (idempotent, audited). |
| `PUT /employees/time-entries/{id}` | **Approver** | Edit (diff only, recalculation on frozen rate). |
| `DELETE /employees/time-entries/{id}` | **Approver** | Delete (refused if invoiced / closed). |
| `GET /employees/time-entries/weekly` | Any user of the tenant | Sunday → Saturday sheet + capacity. |
| `GET /employees/time-entries/by-project` | Any user of the tenant | Hours by project (top 20). |
| `GET /employees/time-entries/export-csv` | Any user of the tenant | CSV export (no amount). |

**CCQ Payroll** (base `/api/erp/v1/payroll`) : `GET/POST /payroll/periods`, `POST /payroll/generate`, `GET /payroll/entries`, `GET /payroll/entries/{id}`, `GET /payroll/entries/{id}/pdf`, `PUT /payroll/periods/{id}/close`.

**Slips** (base `/api/erp/v1/feuillets`) : `.../t4/generate|summary|{id}/pdf`, `.../rl1/generate|summary|{id}/pdf`, `.../pd7a` (+ `/pdf`).

**Assistant** (base `/api/erp/v1/pointage/ai`) : `POST /pointage/ai/chat`.

### 4.13 AI Assistant — behaviour and cost

| Aspect | Detail |
|--------|--------|
| **Scope** | Hours volume only: total, by work type, by billability, last 7 days. |
| **Data access** | **No SQL tool.** Fixed aggregated context, prepared by queries that **never** read rate, cost, salary or SIN. |
| **Write** | None. Strictly read-only. |
| **Model** | `claude-sonnet-4-6`, response capped at 4,000 tokens, a single call. |
| **Bilingual** | French / English depending on the interface. |
| **Cost** | (input tokens × $0.003 + output tokens × $0.015) ÷ 1000, then **× 1.30** (30% markup). Debited from the tenant's **prepaid AI credits**. |
| **Errors** | 403 if the AI guard refuses; 402 if credits are exhausted; 503 if the AI service is unavailable; blocked in view-only mode. |

---

## 5. Integrations and FAQ

### 5.1 Module 10 — Employees

- The employee record supplies the **hourly rate** (or salary) **frozen** into each time entry at creation, as well as the **weekly capacity** used in the capacity card.
- Only **active** employees enter the Payroll summary and CCQ Payroll.
- The **SIN** and **address** entered in Module 10 are required for the T4 / RL-1 slips.
- This screen does not show the HR record; it only uses that data.

### 5.2 Module 11 — Work Orders

- A time entry can be linked to a **work order** and to an **operation** of that WO (the operation is chosen in the edit modal).
- The work order number and the client (via the WO) appear in the table.
- **No automatic carry-over** : tracked hours do not fill the "actual hours" of the work order's operations — that entry stays manual in Module 11.

### 5.3 Module 08 — Projects

- A time entry can be linked to a **project**; the **By Project** tab aggregates hours by project.
- Labour cost (hours × frozen rate) feeds the project's financial view.
- The By Project view has **no** date filter and excludes time entries with no project.

### 5.4 Module 14 — Accounting / Payroll / Slips

- **CCQ Payroll** (tab 5) and **Tax Slips** (tab 6) are surfaces of this desktop module, but their engine lives in the Accounting / Payroll module (rates, brackets, ceilings, CCQ by trade).
- **Passive link with invoicing** : when a time entry is invoiced elsewhere (Accounting / Files), it receives an "invoiced" flag that **locks** it here (edit / delete / approval refused). The Time Tracking module never sets that lock itself.
- Closing a payroll period **freezes** the period's time entries, which protects the slip totals.

### 5.5 Mobile application (`/mobile`)

- The mobile application is the **real time clock** : the employee clocks their arrival and departure there, with their **4-digit PIN** and **geolocation**.
- Mobile and desktop time entries **share the same table**; they appear together in the Time Entries tab, with no source marker.
- Documentation: separate MOBILE_REACT manual.

### 5.6 GPS and logistics (no link with time tracking)

- The `/pointage` page **uses no** GPS function. There is no position column in the time sheets, and no automatic tracking by geographic zone.
- Company GPS tracking concerns the **vehicle fleet** (vehicles, positions, geographic zones, routes) — a separate module. Geolocation of a **time entry** exists only in the mobile application.

### 5.7 AI Assistant (scope)

- This module's assistant is **distinct** from the ERP's general AI Assistant (Module 24) and from the accounting assistants. It is deliberately **compartmentalized** : no free query against the database, no payroll amount, no sensitive personal data. For any financial analysis, turn to the accounting modules and their controls.

### 5.8 FAQ

**Q: Why is there no "Clock in" button on the web?**
A: The desktop page is built for **administrative entry** (an approver records an employee's hours). The real live, geolocated time clock is the **mobile application** (`/mobile`), where the employee clocks with their PIN.

**Q: Can an employee enter their own hours on the web?**
A: No. On the web, only an **approver** (administrator or admin / super-admin role) creates time entries. For self-service, use mobile.

**Q: Does the module do GPS geolocation?**
A: No, not in `/pointage`. The ERP's GPS tracks the **vehicle fleet**, not time entries. A time entry's position is captured only on mobile.

**Q: Why do I see the "Payroll summary" tab but get an error?**
A: Because you are not an **approver**. The tab stays displayed, but loading the figures is restricted to administrators (protection of payroll). This is not a bug.

**Q: Does the "Payroll summary" give official figures?**
A: No. It is an **estimate** (RRQ 6.30% + RQAP 0.43% + EI 1.30%, no tax, no ceiling, no employer contributions). Publishable figures are in **CCQ Payroll**.

**Q: Why does the week start on Sunday?**
A: The time sheet is aligned with the **construction payroll calendar (CCQ)** : Sunday to Saturday.

**Q: What happens if I change the in time without touching the hours?**
A: The server **recomputes** the duration from the new timestamps. The cost, however, is recomputed on the time entry's **frozen rate**, never on the employee's current rate.

**Q: Why can't I edit or delete a time entry?**
A: It is probably **invoiced** or caught in a **closed payroll period**. In both cases, it is locked. To correct it, you must first cancel the invoice (Accounting module) or use an adjustment period.

**Q: Can I approve all time entries at once?**
A: No. There is no bulk approval; you approve one time entry at a time.

**Q: How do I handle an unpaid meal break?**
A: No automatic deduction. Enter two time entries (for example 8 a.m.–12 p.m. and 1 p.m.–5 p.m.) or adjust the duration.

**Q: Can I clock several employees in a single entry?**
A: No. One time entry = one employee. For ten employees, ten entries.

**Q: Does the CSV export contain salaries?**
A: No. The CSV exports **no amount** (no rate, no cost, no salary) — only the hours and metadata. It therefore cannot disclose payroll.

**Q: How can I tell whether a time entry comes from mobile?**
A: There is **no** visible source marker. Mobile and desktop time entries are mixed in the same list.

**Q: Can the AI assistant give me the payroll total or an hourly rate?**
A: No. It answers only about **hours volumes**; it never has access to amounts, rates or salaries. Each question consumes AI credits.

**Q: Can a closed payroll period be reopened?**
A: No, closing is **irreversible**. You must use an adjustment period (Payroll module).

**Q: Does overtime appear in the Time Entries tab?**
A: No. A 45-hour time entry simply shows "45 h". The regular / overtime split is computed when the **CCQ payroll is generated**.

---

## 6. Summary

- **Page** `/pointage`, title **"Time Tracking & Payroll"**, **7 tabs** : Time Entries, Week View, By Project, Payroll Summary, CCQ Payroll, Tax Slips, AI Assistant.
- **Manual entry on the web** (two date-time fields) — **no** real-time time clock or geolocation. The real geolocated time tracking is on the **mobile application**.
- **Writes restricted to approvers** (administrator, admin / super-admin role) : create, approve, edit, delete + Payroll Summary + CCQ Payroll + Slips. Viewing is open to everyone.
- **Rate frozen at entry** : cost uses the rate captured at entry time, never recomputed retroactively.
- **Locks** : a time entry that is **invoiced** or in a **closed payroll period** is immutable (edit / delete / approval refused). Anti double-open-entry (409), out after in, duration ≤ 24 h.
- **Week View** aligned with CCQ (**Sunday → Saturday**) with a **capacity** card (green < 80%, yellow 80-100%, red > 100%).
- **By Project** : top 20 projects, drill-down by employee, no date filter, excludes time entries with no project.
- **Payroll Summary = approximate** (RRQ 6.30% + RQAP 0.43% + EI 1.30%, no tax or contributions). Real payroll is **CCQ Payroll** (deductions, employer contributions, CCQ 12.5%, PDF pay stubs).
- **Tax Slips** (approvers) : **T4** (boxes 14/22/17/18/55), **RL-1** (boxes A/E/B/H, "Preliminary"), **PD7A** by month. Individual PDFs.
- **CSV export** (10 columns, **no amount**); the only PDFs are the pay stubs and the slips.
- **AI Assistant** : read-only, hours volumes only, no access to amounts; debited from AI credits (30% markup).
- **No GPS link**, no automatic break deduction, no lateness detection, no bulk approval, no reopening of a closed period, no file upload.

---

**Documentation generated from the code** : `backend/routers/employees.py` (time sheets + payroll summary), `backend/routers/pointage_ai.py` (assistant), `backend/routers/payroll.py` (CCQ payroll), `backend/routers/feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (slips), `backend/routers/payroll_rates.py` (rates), `frontend/src/pages/PointagePage.tsx`, `frontend/src/components/pointage/PointageAssistantTab.tsx`, `frontend/src/api/employees.ts` / `payroll.ts` / `feuillets.ts` / `pointageAi.ts` / `projects.ts` / `production.ts`.

**Related manuals** :
- Module 10 (Employees — hourly rate, capacity, SIN, mobile PIN) — `10-operations-employes.md`
- Module 11 (Work Orders — operations, actual hours) — `11-operations-bons-de-travail.md`
- Module 08 (Projects — labour cost, hours by project) — `08-ventes-projets.md`
- Module 14 (Accounting / Payroll — rates, brackets, T4 / RL-1 / PD7A slips) — `14-operations-comptabilite.md`
- Module 24 (ERP general AI Assistant) — `24-communication-assistant-ia.md`
- Module 30 (Configuration — users and roles) — `30-configuration.md`
- Separate manual: MOBILE_REACT (mobile application for geolocated field time tracking)
