# Module 11 — Employees (human resources)

> **Version**: 3.0 (complete overhaul verified against the actual source code)
> **Reference code**:
> - Frontend: `frontend/src/pages/EmployeesPage.tsx` (≈ 1,088 lines, single master-detail page), `frontend/src/components/employes/EmployesAssistantTab.tsx` (workforce assistant), API `frontend/src/api/employees.ts`
> - Backend: `backend/routers/employees.py` (≈ 2,162 lines; employee record + SIN reveal + statistics; this same file also hosts the time-tracking endpoints `/time-entries*`, but those are driven by **Module 13 Time Tracking**, not by this screen), `backend/routers/employes_ai.py` (≈ 257 lines, read-only workforce assistant)
> - API prefix: `/api/erp/v1` — the CRUD responds under `/employees` (English), the assistant under `/employes/ai` (French)
> **PostgreSQL tables (per tenant)**: `employees` (record), `employee_competences` (skills, **read-only** here), `time_entries` (time entries, **read-only** here — 5 most recent), `nas_decrypt_audit` (SIN access log, Law 25 — Québec's privacy law); table read for reference: `payroll_periods`
> **Scope**: this module is the **employee directory** — the human-resources record. It is used to **create, search, filter, edit and deactivate** employees, to **export the list to CSV** and to query an **AI workforce assistant**. The record also captures **mobile security** data (PIN, mobile role, stock-management permission) and **tax data** (encrypted Social Insurance Number, address) "for the T4 / RL-1 slips". It **does not enter** hours (that is done in **Module 13 Time Tracking**), it **does not calculate** payroll or tax slips (that is done in **Module 15 Accounting / Payroll**), and it **never deletes** an employee (only deactivation exists). Skills are **displayed** but **cannot be edited** here.

---

## Table of contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step processes](#3-step-by-step-processes)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The Employees module is the company's **personnel roster**. It lets you:

- keep a **record** per employee (identity, contact details, position, department, contract type, status, hire date, hourly rate, salary, weekly capacity, photo, notes);
- manage each employee's **mobile time-tracking security** data: their **4-digit PIN** (to clock in on the `/mobile` app), their **mobile role** (rights in the app) and their **stock-management permission** (barcode reading and movements on mobile);
- capture the **tax information** required for the year-end slips: the **Social Insurance Number (SIN)**, encrypted at rest, and the full mailing **address**;
- **search** and **filter** employees (by name or email, by department, by status);
- **deactivate** an employee who has left (their status becomes Inactive — they are never erased);
- **export** the filtered list to a CSV file;
- consult, **read-only**, the **skills / certifications** and the **5 most recent time entries** of each employee;
- ask questions of an **AI workforce assistant** that answers only on **non-identifying totals** (how many employees per department, per position, per status).

> **Important boundary.** This module is the **management** of employees. The **Time Tracking** of hours (entry, approval, timesheet, export) is a separate module (`/pointage`, Module 13). **Payroll** and the **T4 / RL-1 / PD7A slips** live in **Accounting** (Module 15). The SIN and the address are **entered here**, but they serve only the payroll processes located elsewhere.

### 1.2 What the module does NOT do

- **No deletion** of an employee. There is no "Delete" button. Only **deactivation** (status → Inactive) is possible. This preserves the payroll, time-tracking and billing history.
- **No "Reactivate" button.** To put an employee back in service, you go through **Edit → Status → Active**.
- **No skills editor.** Skills and certifications are **displayed** (badges) but **cannot be created, edited or deleted** from this screen. The backend module does not, in fact, provide any skill write operation.
- **No hours entry or approval.** Time entries appear only **read-only** (the 5 most recent). All hours management (creation, approval, weekly timesheet, export, cost per project) is done in **Module 13 Time Tracking**.
- **No generation of tax slips.** The SIN and the address are **captured** here; the T4 / RL-1 / PD7A slips are **produced** in Accounting / Payroll.
- **No bulk action.** No multi-selection, no batch processing in the list.
- **Strictly read-only AI assistant.** It writes nothing, proposes no action to confirm, and accesses **no individual data** (no SIN, no salary, no hourly rate, no named list).
- **No field time tracking here.** The employee clocks their hours from the mobile app (`/mobile`) with their PIN; entry on this desktop screen is for consultation only.

### 1.3 Access

- **Sidebar** → collapsible **Operations** group → **Employees** (checked-silhouette icon). The entry sits between **Store** and **Work Orders / Time Tracking**.
- **Address**: `/employes`.
- Protected page: you must be authenticated in a tenant.
- **Title displayed** at the top of the page: "Employees".

### 1.4 Permissions and roles

Basic **consultation** is open to any authenticated user of the tenant. Sensitive data and write actions are reserved for specific roles. Three tiers coexist:

| Tier | Who | What they can additionally do |
|------|-----|-------------------------------|
| **Consultation** | Any authenticated user of the tenant | View the list and the detail (name, position, department, email, phone, status), the statistics cards, the skills and the 5 most recent time entries, and use the workforce assistant. **Sees neither salaries, nor hourly rates, nor addresses, nor the masked SIN.** |
| **Payroll manager / administrator** | Administrator (`is_admin`), or the **admin** / **super-admin** role | Everything above, **plus**: see the **salary**, the **hourly rate**, the **address** and the **masked SIN**; **create**, **edit** and **deactivate** an employee; grant the **mobile role** and the **stock-management permission**. |
| **Tenant administrator** | Tenant administrator only (`is_admin`, the true signal, re-read on the server on every request) | Everything above, **plus**: **reveal the SIN in the clear** (with a justification and Law 25 logging). |

> **Payroll privacy filter (rule enforced on the server, 2026-07-06).** For a user who is **not** a payroll manager (neither administrator, nor admin / super-admin role), the server **strips** from the list **and** the detail: the salary, the hourly rate, the address, the city, the postal code and the masked SIN. Concrete consequence: the "Rate/h" column and the Salary / Rate / Address / SIN lines **are empty** for a standard employee. This is not a bug — it is the safeguard that prevents an employee from reading colleagues' pay and address through the directory.
>
> **Granting mobile rights is reserved to the administrator.** An ordinary user cannot grant themselves the stock-management permission or change a mobile role: both fields are **disabled** in the modals for a non-administrator, and the server silently **ignores** these fields if the request comes from a non-administrator.

### 1.5 Statistics cards (KPI)

Three numeric cards sit at the top of the page as soon as the statistics are loaded (source: `GET /employees/statistics`):

| Card | Content | Color |
|------|---------|-------|
| **Total** | Total number of employees (all statuses) | neutral |
| **Active** | Number of employees with **Active** status | green |
| **Departments** | Number of distinct departments **among active employees** | purple |

> The **Departments** card only counts the departments of **active** employees. Empty or null values are grouped under "Undefined" to avoid overcounting.

### 1.6 Key concepts

- **Employee record**: the central record. Two compensation values may appear on it: the **hourly rate** (dollars per hour) and the annual **salary**. They are optional and visible only to payroll managers.
- **Status**: an employee's state (Active, On leave, Training, Work stoppage, Inactive). Only **Active** employees enter the payroll runs of Module 15.
- **Mobile PIN**: a **4-digit** code the employee enters to clock in on the mobile app. It is **hashed** (bcrypt) on the server — never stored in the clear. On the record, only a "Configured" or "Not configured" badge is shown.
- **Mobile role**: the employee's permission level in the `/mobile` app (Employee, Apprentice, Manager, Admin).
- **Stock-management permission**: allows the employee to read barcodes and perform stock movements from mobile.
- **Weekly capacity**: the target number of hours per week (default 40). Module 13 uses it to flag an under-loaded or over-loaded week.
- **Encrypted SIN (Law 25)**: the Social Insurance Number is **encrypted at rest**. Only a **mask** is displayed (for example `XXX-XXX-1234`). Only a tenant administrator can reveal it in the clear, and every reveal is **logged**.
- **Tax information**: the mailing address (address, city, postal code, province) required to produce the T4 / RL-1 slips.
- **Skills**: the employee's certifications (CCQ (Commission de la construction du Québec — Québec construction commission) card, RBQ (Régie du bâtiment du Québec — Québec building authority), welding, etc.), displayed **read-only** on this screen.
- **Workforce assistant**: an AI chat that answers only from **aggregate, non-identifying totals** (counts by department, position, status).

---

## 2. Interface

### 2.1 General layout

```
+------------------------------------------------------------------+
|  Title "Employees"                                               |
+------------------------------------------------------------------+
|  [New employee] [AI Assistant] [Export CSV]   Search  ▾Dept ▾Status |  <- command bar
+------------------------------------------------------------------+
|  [ Total ]        [ Active ]        [ Departments ]              |  <- 3 KPI cards
+----------------------------+-------------------------------------+
|  Employee list             |  Detail panel (on the right)        |
|  (paginated, sortable)     |  (record + skills + time entries)   |
+----------------------------+-------------------------------------+
```

The page is a **master-detail** view: the list on the left, the detail panel on the right as soon as you select an employee. On a phone, the list collapses into cards, and the detail takes up the whole screen (with a back button). The three modals (New employee, Edit employee, AI Assistant) open on top of the page.

### 2.2 Command bar

**Actions (on the left)**

| Button | Icon | Effect |
|--------|------|--------|
| **New employee** | plus | Opens the creation modal. |
| **AI Assistant** | stars | Opens the workforce assistant modal. |
| **Export CSV** | download | Generates and downloads a CSV file of **all** employees matching the current filters. The button is disabled during export. |

**Filters (on the right)**

| Filter | Behavior |
|--------|----------|
| **Search** | Text field ("Search…"). Filters by **full name** or **email**. Resets pagination to the first page. |
| **Department** | Dropdown: "All" + the 11 departments. |
| **Status** | Dropdown: "All statuses" + the 5 statuses. |

### 2.3 Employee list

**Table (desktop)** — sortable and resizable columns:

| Column | Content |
|--------|---------|
| **Name** | Avatar (photo or initials) + "first name last name", with the email underneath. |
| **Position** | The position, or `--` if empty. |
| **Dept.** | The translated department label. |
| **Status** | A colored badge (green Active, amber On leave, blue Training, red Work stoppage, grey Inactive). |
| **Rate/h** | "{rate} $/h", or `--`. **Empty for a user who is not a payroll manager** (privacy filter). |

- A **click** on a row loads the detail on the right; the active row is highlighted.
- **Empty list**: "No employees".
- **Pagination**: 20 employees per page; the controls only appear beyond one page.
- **Sorting**: clicking a header sorts the column; widths are adjusted with the mouse (resize handle) or by auto-fit.

**Cards (phone)**: avatar, name, "position — department", status badge, and the hourly rate (or, failing that, the salary).

### 2.4 Detail panel

**Header**: "first name last name", "position — department", an **Edit** button (pencil), a **close** button (X, or a "back to list" arrow on mobile), and the **status badge**.

**Information displayed** (each appears only if it is filled in):

- **Email** (envelope icon) and **Phone** (handset icon).
- **Contract type** (translated label).
- **Hourly rate** "{x} $/h" — *payroll managers only*.
- **Salary** (formatted in dollars) — *payroll managers only*.
- **Hired**: the hire date.

**Functional badges block**:

- **PIN**: "Configured" or "Not configured".
- **Approver**: present if the employee has the right to approve hours.
- **Stock management**: present if the employee can manage stock on mobile.
- **Mobile role**: present if the mobile role is not "Employee", with the role label.

**SIN and its reveal**:

- The **SIN** is displayed **masked** (for example `XXX-XXX-1234`) — *payroll managers only*.
- The **Reveal SIN** button (eye icon) appears **only for the tenant administrator**. Clicking it opens a **Reason** field ("…Law 25 audit, minimum 10 characters") accompanied by two buttons **Reveal** / **Cancel** and a warning: "Audited access (Law 25). Every consultation is traced." The **Reveal** button stays disabled as long as the reason has fewer than 10 characters. Once revealed, the SIN is displayed in the clear with a "SIN in the clear" badge.

**Address** (pin icon): the concatenation of address, city, postal code, province — *payroll managers only*.

**Deactivate**: the **Deactivate** button (crossed-out silhouette icon) appears only if the status is not already Inactive. Clicking it shows an **inline confirmation** ("Deactivate this employee (Inactive status)?") with a **Deactivate** (red) button and **Cancel**. Confirming changes the status to **Inactive**.

**Additional sections (read-only)**:

- **Skills** (with the count): badges — **green** if the skill is certified, grey otherwise. No adding or editing here.
- **Recent time entries**: the **5 most recent** time entries (date + hours). No creation or approval here — that happens in Module 13.

### 2.5 "New employee" modal

A dedicated error banner appears at the top if there is a problem. The fields, in order:

1. **Photo**: round preview, buttons "Add a photo" / "Change photo" and "Remove". The image (`image/*`) is **compressed in the browser** before upload ("Processing…" state; "Image could not be processed…" message on failure).
2. **First name \*** / **Last name \*** (required).
3. **Email** (email type) / **Phone**.
4. **Position** / **Department** (dropdown, 11 options).
5. **Contract type** (dropdown, 7 options) / **Status** (dropdown, 5 options).
6. **Hire date**.
7. **Hourly rate ($/h)** / **Annual salary ($)** (numbers, step 0.01).
8. **Weekly capacity (h/wk)** (number from 0 to 168, example: 40).
9. **"Mobile time-tracking security" section** (key icon):
   - **PIN code (4 digits)**: masked field, digits only, exactly 4 long. A "The PIN must contain 4 digits" warning appears if you enter 1 to 3 digits.
   - **"Can approve hours"** checkbox.
   - **"Can manage stock (mobile scan)"** checkbox — **disabled for a non-administrator**.
10. **Mobile role (/mobile app)** (dropdown): "Employee (limited)", "Apprentice (limited)", "Manager (full rights)", "Admin (full rights)" — **disabled for a non-administrator**.
11. **"Contact details and tax information (T4 / RL-1)" section** (pin icon):
    - **SIN** ("Social Insurance Number (SIN)", 9 digits). Displayed note: "Encrypted at rest (Law 25). Required for the T4 / RL-1 slips." The field accepts only digits, spaces and hyphens.
    - **Address**; then **City** / **Postal code** / **Province**.
12. **Notes** (text area, 3 lines).

**Buttons**: **Cancel** / **Create**. The **Create** button stays disabled as long as the first name or last name is empty, or as long as the PIN is incomplete (1 to 3 digits). Closing the modal clears the form.

### 2.6 "Edit employee" modal

**Identical** structure to creation (same fields 1 to 12), with these differences:

- The **SIN** field is **left empty**; if a SIN already exists, the input hint shows "SIN on file (masked)". **Leaving it empty = do not change** the existing SIN.
- The **PIN** field shows "••••" if a PIN is already configured; leaving it as is keeps the PIN.
- The action button is called **Save** (with a progress indicator).

### 2.7 "AI Assistant — HR Workforce" modal

- **Header**: "AI Assistant — HR Workforce", subtitle "Workforce view from non-identifying aggregates (read-only)."
- **Empty state**: three example questions ("How many employees do I have in total?", "What is the breakdown by department?", "How many active employees per position?") and a **privacy note**: the assistant "does not access any sensitive individual data (SIN, salaries, hourly rates)".
- **Chat**: user / assistant bubbles, an "Analyzing…" indicator, an input area ("Ask your question about the workforce…") and a **Send** button (the Enter key without Shift sends). A lock prevents double submission.
- **Metadata** shown under each answer: "HR" profile, token count, cost, duration.
- **Bilingual** (French / English depending on the interface language).

---

## 3. Step-by-step processes

### 3.1 Create an employee

1. Click **New employee** in the command bar.
2. At minimum, fill in the **First name** and **Last name** (the only required fields).
3. Complete as needed: contact details, position, department, contract type, status (default **Active**), hire date, hourly rate or salary, weekly capacity.
4. Click **Create**. The employee appears in the list and an "Employee created" banner confirms the operation.

> Reserved for **payroll managers / administrators**. An ordinary user will not see this action succeed (the server refuses the creation).

### 3.2 Add or change the photo

1. In the creation or edit modal, **Photo** section, click **Add a photo** (or **Change photo**).
2. Choose an image file. The application **compresses** it automatically ("Processing…" state).
3. The round preview updates. To remove it, click **Remove**.
4. Save the record. The avatar (photo or, failing that, initials) then appears in the list and the detail.

### 3.3 Configure mobile time tracking (PIN, role, stock permission)

1. Open the record in **Edit** mode, go to the **Mobile time-tracking security** section.
2. Enter a **4-digit PIN**: this will be the employee's code to clock in on the `/mobile` app.
3. Check **"Can approve hours"** if the employee should be able to approve time entries.
4. If you are an **administrator**: check **"Can manage stock"** and choose the appropriate **Mobile role**. (Both controls are disabled for a non-administrator.)
5. Save. The PIN is **hashed** on the server; the record then only shows a "Configured" badge.

> **After a change to the mobile app's sub-path, employees may have to re-subscribe to notifications** (see Module 13). The PIN itself remains valid.

### 3.4 Enter tax information (SIN and address)

1. Open the record in **Edit** mode, go to the **Contact details and tax information (T4 / RL-1)** section.
2. Enter the **SIN** (9 digits). The system **validates** the number (Luhn check) and **encrypts** it before saving; if the number is invalid, the save is refused.
3. Fill in the **address**, the **city**, the **postal code** and the **province**.
4. Save. The record will then only show a **masked SIN**; the address is visible only to payroll managers.

> This data is used only for **payroll** and the **slips** produced in Accounting (Module 15). Entering it here is a prerequisite for generating the T4 / RL-1 slips.

### 3.5 Edit an employee

1. Select the employee, then click the **Edit** pencil in the detail header.
2. Adjust the desired fields. For the **SIN**, leaving the field **empty** keeps the existing number; for the **PIN**, leaving "••••" keeps the existing code.
3. Click **Save**. A "Changes saved" banner confirms.

### 3.6 Deactivate an employee (departure)

1. Select the employee (their status must be other than Inactive).
2. In the detail panel, click **Deactivate**.
3. Confirm in the inline prompt ("Deactivate this employee (Inactive status)?").
4. The status changes to **Inactive** and an "Employee deactivated" banner confirms. The employee **stays** in the system (their history is preserved), but they drop out of lists filtered on active employees and out of payroll runs.

> There is **no deletion**. Deactivation is the intended way to take an employee out of service.

### 3.7 Reactivate an employee

1. Find the employee (filter on **Inactive** status if needed).
2. Click **Edit**.
3. Set the **Status** back to **Active** (or another active status such as On leave / Training).
4. **Save**. There is no dedicated "Reactivate" button — this is the normal path.

### 3.8 Reveal the SIN in the clear (administrator, Law 25)

1. Be a **tenant administrator**. Select the employee.
2. In the detail panel, on the SIN line, click **Reveal SIN**.
3. Enter a **reason** of at least 10 characters (for example: "Producing the 2026 RL-1 slip").
4. Click **Reveal**. The system **logs** the access (who, when, the reason, the IP address) **before** decrypting, then displays the SIN in the clear with the "SIN in the clear" badge.

> **Compliance.** If the audit log cannot be written, decryption is **blocked** (503 error) — no trace, no access. The response is never cached by the browser. Every consultation is traceable.

### 3.9 Search and filter

1. Type a name or an email in the **Search**: the list narrows immediately (and returns to page 1).
2. Choose a **Department** and/or a **Status** in the dropdowns to refine.
3. Combine the three filters as needed. The filters also apply to the CSV export.

### 3.10 Export the list to CSV

1. Set the filters (search, department, status) to target the desired employees.
2. Click **Export CSV**. The application fetches **all** matching employees (in batches), not just the displayed page.
3. The **`employes_export.csv`** file downloads. Columns: ID, First name, Last name, Email, Phone, Position, Department, Status, Contract type, Hire date, Hourly rate.

> The CSV **contains neither the salary nor the SIN**. For a user who is not a payroll manager, the **Hourly rate** column comes out **empty** (privacy filter). The file is encoded in UTF-8 (with BOM) so it opens cleanly in Excel in Québec.

### 3.11 View skills and recent time entries

1. Select the employee.
2. In the detail, the **Skills** section lists the certifications (green badge = certified).
3. The **Recent time entries** section shows the **5 most recent** time entries (date + hours).
4. Both sections are **read-only**. To enter hours, go to Module 13 Time Tracking; skills cannot be edited in the ERP.

### 3.12 Query the workforce assistant

1. Click **AI Assistant** in the command bar.
2. Ask a question about the **workforce** ("How many active employees per position?").
3. The assistant answers from the **aggregate totals** (by department, position, status). The cost and duration are shown under the answer.
4. If you ask for a SIN, a salary, an hourly rate or a named list, the assistant **politely refuses**: this information is inaccessible to it by design.

---

## 4. Reference

### 4.1 Reference data

**11 departments** (mirror of the backend `DEPARTEMENTS`):

`CHANTIER`, `STRUCTURE_BETON`, `CHARPENTE_BOIS`, `FINITION`, `MECANIQUE_BATIMENT`, `ELECTRICITE`, `INGENIERIE`, `QUALITE_CONFORMITE`, `ADMINISTRATION`, `COMMERCIAL`, `DIRECTION`.

> The four trades `STRUCTURE_BETON`, `CHARPENTE_BOIS`, `FINITION`, `MECANIQUE_BATIMENT` (together with `CHANTIER` and `ELECTRICITE`) determine, at payroll time (Module 15), whether the employee falls under the CCQ. Classifying the department correctly here is therefore useful downstream.

**5 statuses** (`STATUTS`, default **ACTIF**):

| Status | Badge color | Enters payroll (Module 15) |
|--------|-------------|-----------------------------|
| **Active** (`ACTIF`) | green | Yes |
| **On leave** (`CONGE`) | amber | No |
| **Training** (`FORMATION`) | blue | No |
| **Work stoppage** (`ARRET_TRAVAIL`) | red | No |
| **Inactive** (`INACTIF`) | grey | No |

**7 contract types** (`TYPES_CONTRAT`, default **CDI**):

`CDI`, `CDD`, `TEMPORAIRE`, `SAISONNIER`, `CONSULTANT`, `STAGE`, `APPRENTISSAGE`.

**4 mobile roles** (rights in the `/mobile` app, default **Employee**):

| Role | Label in the interface |
|------|------------------------|
| `EMPLOYE` | Employee (limited) |
| `APPRENTI` | Apprentice (limited) |
| `MANAGER` | Manager (full rights) |
| `ADMIN` | Admin (full rights) |

### 4.2 Record fields and input limits

| Field | Type / bounds |
|-------|----------------|
| First name, Last name | Text (required). |
| Email, Phone | Text. |
| Position | Free text. |
| Department | One of the 11 values (accepted freely on the backend). |
| Status | One of the 5 values (validated; default Active). |
| Contract type | One of the 7 values (validated; default CDI). |
| Hire date | Date. |
| Hourly rate | Number ≥ 0, at most 100,000. |
| Salary | Number ≥ 0, at most 100,000,000. |
| Weekly capacity | Number from 0 to 168 (hours/week). |
| PIN | Exactly 4 digits; **hashed** (bcrypt) on the server. |
| Mobile role | One of the 4 roles; granting **reserved to the administrator**. |
| Stock-management permission | Yes / No; granting **reserved to the administrator**. |
| Photo | Compressed image (embedded data); high technical limit (anti-abuse). |
| SIN | At most 20 characters; **validated** (Luhn, 9 digits); **encrypted** at rest. |
| Address / City / Postal code / Province | Text (respective limits 300 / 120 / 12 / 60 characters). |
| Notes | Free text. |

### 4.3 Endpoints (API)

**HR management** endpoints (base `/api/erp/v1/employees`):

| Method + path | Required role | Purpose |
|---------------|---------------|---------|
| `GET /employees` | Any user of the tenant | Paginated list + search + filters (payroll masked if not a payroll manager). |
| `GET /employees/{id}` | Any user of the tenant | Detail + skills + recent time entries (payroll masked if not a payroll manager). |
| `GET /employees/statistics` | Any user of the tenant | Totals for the KPI cards. |
| `POST /employees` | Payroll manager / administrator | Create an employee. |
| `PUT /employees/{id}` | Payroll manager / administrator | Edit (returns 404 if the employee does not exist). |
| `POST /employees/{id}/reveal-nas` | **Tenant administrator** | Reveal the SIN in the clear (audited, fail-closed). |

Workforce assistant (base `/api/erp/v1/employes/ai`):

| Method + path | Required role | Purpose |
|---------------|---------------|---------|
| `POST /employes/ai/chat` | Any user of the tenant (+ AI guard + credits) | Answers on workforce aggregates. No individual access. |

> The **time-tracking** endpoints (`/employees/time-entries*`, `/employees/payroll-summary`) live in the same backend file but are **documented in Module 13 Time Tracking**. This screen uses them read-only for the "5 most recent time entries" only.

### 4.4 SIN security (Law 25)

| Mechanism | Behavior |
|-----------|----------|
| **Validation** | Luhn check on 9 digits; invalid number → refusal (422 error). |
| **Encryption** | Encrypted at rest (Fernet). Only the **encrypted** number and the **last 4 digits** are stored; **never** the cleartext SIN. If encryption is unavailable, the save fails rather than storing in the clear. |
| **Display** | Always masked (`XXX-XXX-1234`) in the list and the detail; removed for non–payroll-managers. |
| **Reveal** | Reserved to the tenant administrator, on **justification** (≥ 10 characters). |
| **Audit** | Every reveal is logged (who, when, reason, IP) **before** decryption. If the log fails, decryption is **blocked** (503). |
| **No caching** | The response containing the cleartext SIN carries a "do not cache" header. |
| **Logs** | Any 9-digit pattern is scrubbed from the technical logs. |

### 4.5 Workforce assistant — behavior and cost

| Aspect | Detail |
|--------|--------|
| **Scope** | Workforce view only: totals by department, position and status. |
| **Data access** | **No SQL tool.** The model receives only an aggregated context prepared by fixed queries that **never** read a sensitive column (SIN, salary, hourly rate, date of birth). |
| **Writing** | None. Strictly read-only. |
| **Refusal** | Refuses any request for a SIN, salary, hourly rate or named list. |
| **Model** | `claude-sonnet-4-6`, response capped at 4,000 tokens, a single call (no tool loop). |
| **Billing** | Cost = (input tokens × $0.003 + output tokens × $0.015) ÷ 1000, then × 1.30 (30% markup). Charged to the tenant's **prepaid AI credits**. A depleted balance returns a 402 error (top-up required); the super-admin is exempt. |
| **Rate** | Limit of 20 requests per minute per IP address. |

### 4.6 Rules and validations enforced

| Rule | Effect |
|------|--------|
| Empty first or last name | Creation / edit refused. |
| Status not in the list | Refused (validation). |
| Contract type not in the list | Refused (validation). |
| Negative hourly rate / salary | Refused (input bounds). |
| Weekly capacity outside 0–168 | Refused. |
| PIN other than 4 digits | Blocks the **Create** button; refused on the server. |
| Invalid SIN (Luhn) | Refused (422). |
| Reveal reason < 10 characters | **Reveal** button disabled; refused on the server. |
| Editing a nonexistent employee | 404 error (no false success). |
| Write by a non–payroll-manager | Refused (creation / edit / SIN reveal). |
| Mobile role / stock permission sent by a non-administrator | Silently ignored. |
| Consultation mode (read-only subscription) | All writes are blocked (403). |

---

## 5. Integrations and FAQ

### 5.1 Module 13 — Time Tracking

- **Time tracking** (entry, approval, weekly timesheet, cost per project, CSV export of hours) is done in Module 13, at `/pointage`.
- The **weekly capacity** entered here feeds the under-load / over-load indicators of the Module 13 timesheet.
- The **PIN** and the **mobile role** set here govern field time tracking on the `/mobile` app.
- On this screen, only the **5 most recent time entries** appear, read-only.

### 5.2 Module 15 — Accounting / Payroll

- The **SIN** and the **address** captured here are prerequisites for the **T4, RL-1 and PD7A slips**, produced in Accounting.
- Generating a slip **decrypts** the SIN in an **audited** way (the same logging mechanism as the manual reveal).
- **Access nuance**: the manual SIN reveal (this screen) is reserved to the tenant **administrator**, whereas the production of RL-1 slips is open to the **administrator or the accountant**. Two different doors to the same number, each logged.
- The **Active** status is what brings an employee into the payroll runs; the **department** determines CCQ applicability.

### 5.3 Mobile app (`/mobile`)

- Employees clock in from the mobile PWA with their **4-digit PIN**.
- The **mobile role** (Employee / Apprentice / Manager / Admin) and the **stock-management permission** control what the employee can do in the app (clock in/out, approve, read stock barcodes).

### 5.4 Module 10 — Store

- The **stock-management permission** granted here allows the employee to perform stock movements and barcode scanning from mobile. Catalog and movement management on the desktop side stays in the Store.

### 5.5 AI Assistant (scope)

- The **workforce** assistant in this module is **separate** from the ERP's general AI Assistant (Module 25). It is deliberately **siloed**: no free-form database queries, no individual data. For any named or sensitive question, it directs the user to the Employees / Payroll module and its Law 25 controls.

### 5.6 FAQ

**Q: How do I delete an employee?**
A: You do not delete an employee. You **deactivate** them (the **Deactivate** button, status → Inactive), which preserves all their history. This is intentional.

**Q: I deactivated someone by mistake, how do I put them back in service?**
A: **Edit** their record and set the **Status** back to **Active**. There is no separate "Reactivate" button.

**Q: Why don't I see hourly rates, salaries or addresses?**
A: Because you are not a **payroll manager**. For privacy protection (Law 25), the server removes the salary, hourly rate, address and masked SIN from the list **and** the detail for any user who is neither an administrator nor of the admin / super-admin role. The "Rate/h" column and these detail lines then stay **empty**. This is not a bug.

**Q: Why is the "Hourly rate" column in my CSV export empty?**
A: Same reason: the export applies the same privacy filter. A payroll manager who exports will see the column filled in.

**Q: How do I add a skill or a certification to an employee?**
A: This is **not possible** from this screen. Skills are only **displayed** here (badges, green if certified). No skill entry is implemented in the ERP for now.

**Q: Where do I enter work hours?**
A: In **Module 13 Time Tracking** (`/pointage`). This screen only shows the **5 most recent** time entries, read-only.

**Q: How do I produce the T4 / RL-1 slips?**
A: In **Accounting / Payroll** (Module 15). Here, you only **enter** the required SIN and address. Make sure these fields are filled in for each relevant employee before the production period.

**Q: Who can see the SIN in the clear?**
A: Only the **tenant administrator**, and only by providing a **reason** (≥ 10 characters). Every reveal is **logged**. An accountant, for their part, accesses it indirectly when generating an RL-1 slip (also audited access).

**Q: Is the SIN stored in the clear anywhere?**
A: No. It is **encrypted at rest**; only the encrypted number and the last 4 digits are kept. If encryption is not available, the save **fails** rather than recording a cleartext number.

**Q: Why is my SIN entry refused?**
A: The number must be a **valid** SIN (9 digits, Luhn check). Check your entry.

**Q: I'm not an administrator; why are the "Can manage stock" and "Mobile role" fields greyed out?**
A: Granting these rights is reserved to the tenant administrator. Even if you forced it, the server would ignore these fields.

**Q: Can the AI assistant give me someone's salary or edit a record?**
A: No. It is **strictly read-only** and **refuses** any individual data (SIN, salary, hourly rate, named list). It answers only on **totals** (how many employees per department / position / status) and never writes anything.

**Q: Can I process several employees at once (bulk deactivation, etc.)?**
A: No. There are no bulk actions or multi-selection. Each operation is done employee by employee.

**Q: Is the PIN used to log into the ERP?**
A: No. The 4-digit PIN is used for **time tracking on the mobile app**. Logging into the desktop ERP uses the usual user account.

**Q: What happens if I change an employee's hourly rate?**
A: On this record, the new rate applies to **future** time entries. Time entries **already** recorded keep the rate captured at the moment of the entry (Module 13 freezes that rate to avoid retroactively recalculating payroll).

---

## 6. Summary

- The **Employees** module (`/employes`) is the **personnel directory**: create, search, filter, edit, **deactivate** (never delete), export to CSV.
- **3 KPI cards**: Total, Active, Departments (departments counted among the **active** employees).
- **11 departments**, **5 statuses** (only **Active** enters payroll), **7 contract types**, **4 mobile roles**.
- **Three permission tiers**: consultation (everyone), payroll manager / administrator (sees payroll, creates / edits / deactivates), tenant administrator (reveals the SIN).
- **Law 25 privacy filter**: salary, hourly rate, address and masked SIN **hidden** from non–payroll-managers — hence empty columns / lines for a standard employee.
- **SIN**: validated (Luhn), **encrypted** at rest, **masked** on display; reveal **audited** and **fail-closed** (blocked if the log fails), reserved to the administrator.
- **Mobile PIN** (4 digits, hashed), **mobile role** and **stock-management permission**: granting reserved to the administrator.
- **Skills** and **5 most recent time entries**: **read-only** here (hours entry in Module 13, skills not editable).
- **Workforce assistant**: **strictly read-only**, non-identifying aggregates, no individual access, no writing; charged to AI credits (30% markup).
- **No deletion, no dedicated reactivation, no skill editing, no bulk actions, no tax slips here.**

---

**Documentation generated from the source code**: `backend/routers/employees.py`, `backend/routers/employes_ai.py`, `frontend/src/pages/EmployeesPage.tsx`, `frontend/src/components/employes/EmployesAssistantTab.tsx`, `frontend/src/api/employees.ts`.

**Related manuals**:
- Module 10 (Store — stock-management permission) — `10-operations-magasin.md`
- Module 13 (Time Tracking — hours entry and approval) — `13-operations-pointage.md`
- Module 15 (Accounting / Payroll — T4 / RL-1 / PD7A slips) — `15-operations-comptabilite.md`
- Module 25 (General ERP AI Assistant) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — users and roles) — `28-configuration.md`
