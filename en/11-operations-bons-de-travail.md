# Module 11 — Work Orders

> **Version**: 3.0 (complete overhaul verified against the actual source code)
> **Reference code**:
> - Frontend: `frontend/src/pages/BonsTravailPage.tsx` (≈ 3,519 lines, single page with 4 tabs + detail view), `frontend/src/components/bt/BtAssistantTab.tsx` (AI Assistant tab), API `frontend/src/api/production.ts` (≈ 521 lines), state `frontend/src/store/useProductionStore.ts` (≈ 464 lines)
> - Backend: `backend/routers/production.py` (≈ 7,209 lines — file **shared** with the Tracking module: Kanban, Gantt, Calendar), `backend/routers/bt_ai.py` (≈ 471 lines, AI Assistant), `backend/routers/operation_templates.py` (≈ 497 lines, schedule generator)
> - API prefix: `/api/erp/v1/production` for the core of the module; the AI Assistant lives under `/api/erp/v1/bons-travail/ai`
> **PostgreSQL tables (per tenant)**: `formulaires` (a work order = `type_formulaire = 'BON_TRAVAIL'`), `formulaire_lignes` (materials), `operations` (tasks), `bt_assignations` (assigned employees), `bt_comments` (comments), `operation_types` (task catalog); indirectly affected tables: `produits` and `mouvements_stock` (stock movements), `time_entries` (time tracking, read-only), `dossier_formulaires` (attachment to the opportunity case file); table shared between tenants: `public.operation_templates` (schedule models).
> **Scope**: a **work order** (WO) is the operational unit of work on the job site. This module is used to **create** a WO, to **advance it through statuses** (draft → in progress → paused → completed, or cancelled), to break it down into **operations** (tasks grouped by phase, with planned and actual hours, a responsible employee and a subcontractor), to list the **materials** consumed (each line tied to inventory **triggers a stock movement**), to **assign** employees, to exchange **comments**, to generate a printable **HTML/PDF document**, and to monitor the **weekly labor capacity**. A **catalog** of operation types and an **AI Assistant** round out the set. The module **does not generate** an operation schedule from the creation form (that goes through the Projects module or the conversion of a quote into a project), **produces no invoice or tax** (the total amount is informational), and **does not enter time-tracking hours** (those are entered in Module 12 Time Tracking). The same work orders also appear in the **Kanban, the Gantt and the Calendar** of the Tracking module, but those views do not belong to this page.

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

The Work Orders module is the company's **intervention control desk**. It lets you:

- create a numbered **work order** (`BT-00001`), optionally attached to a project;
- track its **progress** through a status cycle governed by a state machine (start, pause, resume, complete, cancel);
- break the work down into **operations** (tasks) grouped by **phase** (foundation, framing, plumbing, etc.), each with its planned hours, actual hours, employee, subcontractor, dates and status;
- list the **materials** consumed: each line tied to a catalog product **automatically debits the stock** and leaves a trace in the inventory movements;
- **assign** employees to the work order (with a free-text role);
- keep a **timestamped comment thread**;
- generate a professional **HTML document**, themed to the company's colors, ready to print or save as a PDF;
- consult a **global view of all ongoing operations** and a **weekly capacity** table (planned load versus actual load);
- manage a **catalog** of reusable operation types;
- query an **AI Assistant** that reads the work orders and can **propose a new one** (created only after confirmation).

### 1.2 What the module does NOT do

- **No schedule generation from the creation form.** The "New work order" button creates an **empty** WO (no operation pre-filled). Schedules by project type (residential, renovation, public) apply **elsewhere**: through the "Generate schedule" button of the Projects module, or automatically when a quote is converted into a project (see §3.16).
- **No invoice, no tax, no accounting export.** The "Total amount" field is the sum of the material lines; it is **informational** and issues no invoice. No GST/QST (Goods and Services Tax / Québec Sales Tax) calculation, no margin.
- **No time-tracking entry here.** An operation's actual hours are entered **by hand**. Employee time tracking (the `time_entries` table) is done in **Module 12 Time Tracking**; no mechanism automatically carries time entries into the operations' actual hours.
- **The total amount is not editable.** It always recalculates from the lines; there is no editable amount field.
- **The AI Assistant creates only a work order**, and only on confirmation. It creates neither operation, nor material line, nor assignment. When reading, it has no access to payroll, human-resources or security data.
- **No direct permanent deletion.** An active work order cannot be erased in one step: you must first **cancel** it (soft-delete), and then, only if it is cancelled, **delete it permanently**.
- **The Kanban, the Gantt and the Calendar are not in this page.** They do display the work orders, but they belong to **Module 02 Tracking**.

### 1.3 Access

- **Side menu** → **Operations** section → **Work Orders** (clipboard icon).
- **Address**: `/bons-travail`.
- Protected page: you must be authenticated in a tenant.
- **Displayed title**: "Work Orders".
- **Direct opening of a work order**: a link of the form `/bons-travail?open=<id>` (for example from the Calendar) automatically opens the detail view of the relevant WO.

### 1.4 Permissions and roles

**Viewing** the page is open to any authenticated user of the tenant. **Writing** depends on the role. Two distinct guards coexist in the module:

| Guard | What it protects | Authorized roles |
|-------|------------------|------------------|
| **Work-order write** (`require_tenant_admin_or_role`) | Create / edit / delete / restore a WO, change its status, manage its operations, its material lines, its assignments and its comments; generate a schedule | administrator (`is_admin`), the **admin** role, **super_admin**, **manager** (`gestionnaire`) or **foreman** (`contremaitre`) |
| **Catalog write** (`_require_admin`) | Create / edit / delete an **operation type** in the catalog | administrator (`is_admin`), the **admin** or **super_admin** role **only** |

> **Nuance to remember.** A **manager** or a **foreman** can fully drive the work orders (operations, materials, assignments, statuses), but **cannot** modify the operation-types catalog: that last action is reserved for **administrators and super-administrators**. The `require_tenant_admin_or_role` guard always takes into account the owner's administrator status (`is_admin`, re-read server-side) so as never to exclude a boss whose role happens to be "user".
>
> **Read-only mode.** A tenant in **read-only mode** (suspended subscription) is blocked upstream: all work-order writes (creation, edit, deletion, status change) are refused; reading remains possible.

### 1.5 Statistics cards (KPI)

Four figure cards permanently cap the page, above the tabs (source: `GET /production/statistics`):

| Card | Content | Color |
|------|---------|-------|
| **Total** | Total number of work orders | neutral |
| **In progress** | Work orders in status **EN_COURS** | blue |
| **Completed** | Work orders in status **TERMINE** | green |
| **Total amount** | Sum of the (materials) amounts of all work orders, in dollars | primary |

> The endpoint also returns a breakdown by status and the number of assignments, but those values are not shown on the cards.

### 1.6 Key concepts

- **Work order (WO)**: the unit of work. Technically, a row of the `formulaires` table with `type_formulaire = 'BON_TRAVAIL'`. Numbered `BT-00001` (5 digits). Carries a status, a priority, a project, dates (start, end, due date), an amount and notes.
- **Operation (task)**: a step of the work order, attached to a **phase**. It carries planned hours, actual hours, an employee, a supplier (subcontractor), a status and dates. Operations are **independent** of the WO status (no automatic synchronization).
- **Phase** (`poste_travail`): the grouping of operations (for example "Foundation", "Framing"). It is simple text; the operations table shows a section header per phase.
- **Material line**: an item consumed by the work order. If the line is tied to an inventory product, it **debits the stock** (issue movement); deleting it **credits back** the stock (return movement).
- **Assignment**: an employee attached to the work order with a free-text role ("Team lead", "Helper", etc.). An employee can only be assigned once per WO.
- **Operation type (catalog)**: a reusable task template (name, category, code, standard hours) that feeds the "Station/Operation" drop-down of the forms. The catalog is independent: renaming or deleting a type does **not** change operations already created (they keep their name as a snapshot).
- **Automatic schedule**: a "Schedule - {project}" work order generated from a model according to the project type. It is triggered **outside this page** (see §3.16). Only one automatic-schedule WO per project.
- **State machine**: the set of allowed status transitions. It prevents, for example, jumping directly from "Draft" to "Completed", or reopening a completed WO (except super-administrator).
- **Stock invariant**: "cancelled work order ⇔ stock restored". Cancelling a WO credits back all the consumed stock; restoring it re-consumes it (see §4.8).

---

## 2. Interface

### 2.1 General layout

```
+------------------------------------------------------------------+
|  Clipboard  "Work Orders"                                        |
+------------------------------------------------------------------+
|  [ Total ]   [ In progress ]   [ Completed ]   [ Total amount ]  |  <- 4 KPI cards
+------------------------------------------------------------------+
|  Work Orders | Operations | Catalog | AI Assistant               |  <- tab bar
+------------------------------------------------------------------+
|  Content of the active tab                                       |
+------------------------------------------------------------------+
```

The page has **four tabs** and a contextual **detail view**:

| Tab | Key | Role |
|-----|-----|------|
| **Work Orders** | `liste` | Paginated, filterable and sortable list of work orders |
| **Operations** | `operations` | Global view of all operations + weekly capacity |
| **Catalog** | `catalogue` | Management of reusable operation types |
| **AI Assistant** | `assistant` | Chat that queries the work orders and proposes new ones |

> **The detail view is not a tab.** It opens when you click a work order or the "New work order" button. It then replaces the tab bar with a breadcrumb: "← Work Orders / {number} — {name}". The back button returns to the list.

### 2.2 "Work Orders" tab (list)

#### 2.2.1 Command bar

- **New work order** (primary button, "+" icon): opens the detail view in creation mode (full page).
- **Search** ("Search..."): real-time filter on the **name** and the **number** (300 ms typing debounce).
- **Status filter** (drop-down): "All statuses", or one of the five statuses (Draft, In progress, Paused, Completed, Cancelled). Single selection.
- **Priority filter** (drop-down): "All priorities", or one of the four (Low, Normal, High, Urgent). Single selection.

#### 2.2.2 Work-orders table

Columns (sortable and resizable):

| Column | Content |
|--------|---------|
| **Number** | `BT-00001` (monospace font) |
| **Name** | Work-order name (or project name, or number if left empty) |
| **Status** | Colored badge |
| **Priority** | Colored badge |
| **Project** | Associated project (if set) |
| **Start** | Planned start date — **editable directly on click** (inline date field) |
| **End** | Planned end date — **editable directly on click** |
| **Due date** | Due date |
| **Amount** | Materials total (right-aligned) |
| **Actions** | Edit; Delete (conditional) |

**Per-row actions**:

- **Edit** (pencil): opens the work order directly in edit mode.
- **Delete** (red trash): appears **only if the status is "Cancelled"**. It triggers a **permanent deletion** (with a browser confirmation and an anti-double-click lock). A non-cancelled work order never shows this button in the list.

Clicking a row (away from the buttons) opens the **detail view** in read mode. Below the table: a "{n} work order(s)" counter and pagination (25 per page). If the list is empty: "No work order found". On phone, the table collapses into equivalent cards.

> **Only the Start and End columns are inline-editable** in the list. The other fields (name, status, priority, project, due date, notes) are edited from the detail view.

### 2.3 Detail view — Creation mode (full page)

Opened by "New work order". Header "New work order" with the **Cancel** and **Create** buttons (the create button shows the count of pre-entered items, for example "Create + 3 op. 2 prod.").

**Header fields**:

| Field | Notes |
|-------|-------|
| **Name** | Optional — "the project name will be used if empty" (and failing that, the number) |
| **Priority** | Low / Normal / High / Urgent (default: Normal) |
| **Project** | "No project" or a project from the list |
| **Planned start date** | Optional |
| **Planned end date** | Optional (must be after the start) |
| **Due date** | Optional |
| **Total amount** | **Read-only** — hint "-- (calculated from the lines)" |
| **Notes** | Free text |

**"Operations" card**: the **Add a task** button opens an operation subform (same fields as in the detail view, see §2.6). The operations entered here are **stacked locally** (pending) and a summary table shows Operation / Quantity / Supplier / Planned hours / Status, with the total hours.

**"Products / Materials" card**: the **Add a product** button adds editable lines (inventory product with its stock, quantity, read-only unit, unit price, amount), with a "Materials total".

> **How creation actually works.** The server first creates an **empty work order** (Draft status, number `BT-00001`) with no operation at all. It **then** creates, one by one, the operations and lines you pre-entered, via the child endpoints. If one of those creations fails, the work order still exists and a message reports the number of items not created. **Creation does not roll out any schedule model**: the templates by project type go through the Projects module (see §3.16).

### 2.4 Detail view — Read mode

Displayed by clicking an existing work order.

**Header**: number (monospace font), status badge, priority badge (with an alert-triangle icon if "Urgent"), then the name.

**Action buttons**:

- **Edit** (pencil): switches to edit mode.
- **HTML** and **Preview**: generate the work order's HTML document and display it in an **embedded window** (see §2.10).
- **PDF** (printer): opens the document in a **new browser tab**, ready to print or save as a PDF.

**Status transition buttons** (depending on the current status):

| Current status | Transition buttons |
|----------------|--------------------|
| **Draft** | **Start** (→ In progress) |
| **In progress** | **Pause** (→ Paused) · **Complete** (→ Completed) |
| **Paused** | **Resume** (→ In progress) |
| **Completed** | (no transition) |
| **Cancelled** | **Restore** (→ Draft) · **Delete permanently** |

**Lifecycle buttons** (in addition):

- If the work order is **cancelled**: **Restore** (back arrow, → Draft, re-consumes the stock) and **Delete permanently** (trash, irreversible action).
- Otherwise, if the work order is **not completed**: **Cancel** (no-entry icon, → Cancelled, **soft-delete that restores the stock**).

All these actions require a confirmation and are protected by an anti-double-click lock.

**Information grid**: Project, Planned start, Planned end, Due date, Total amount, Created on, plus a Notes block.

### 2.5 Detail view — Edit mode

Title "Edit the work order", with **Cancel** and **Save**. Editable fields: Name, **Status** (full menu, including Cancelled), Priority, Project, Planned start date, Planned end date, Due date, Notes. The **Total amount** stays read-only ("calculated from the lines"). Saving sends only the **fields that were actually changed** and respects the state machine (see §4.2).

### 2.6 Operations section (in the detail view)

Header "Operations ({n})" and an **Add a task** button.

**Operation add form**:

| Field | Notes |
|-------|-------|
| **Station/Operation** | Drop-down populated by the active catalog; an "off-catalog" option appears if the entered name is not found in it |
| **Quantity** | Numeric (0 to 1,000,000) |
| **Assigned to** | Employee (drop-down) |
| **Supplier/Subcontractor** | "-- Internal --" (default) or a supplier from the Store; an inherited free-text value is preserved |
| **Planned hours** | Numeric (0 to 100,000) |
| **Status** | Pending / In progress / Completed / Cancelled |
| **Start date** | Optional |
| **End date** | Optional |
| **Phase** | Text (`poste_travail`), for example "Foundation", "Framing" |
| **Description** | Free text |

**Operations table** (grouped by **Phase**, with a section header):

Columns: **Operation**, **Qty**, **Assigned to**, **Supplier**, **Start**, **End**, **Plan. hrs**, **Actual hrs**, **Status** (drop-down editable directly inline), and the actions (pencil for inline editing, "X" to delete). Inline editing adds the **Actual hours** field. The footer shows the **Totals** (sum of planned hours and actual hours). Equivalent cards on phone.

### 2.7 Lines section (materials)

Header "Lines ({n})" and an **Add** button.

Columns: **Description** (with an "Inventory" badge if the line is tied to a product), **Qty**, **Unit**, **U.P.** (unit price), **Amount**, actions (pencil / "X"). Footer: **Total**. If empty: "No lines".

> **Effect on stock.** Adding a line tied to a product **debits the stock** (issue movement). Editing it adjusts the stock by the **delta**. Deleting it **credits back** the stock (return movement). These movements are **ignored** if the work order is cancelled (so as not to double-count with the restoration). See §4.8.

### 2.8 Assignments section

Header "Assignments ({n})" and an **Assign** button. Each line shows the employee's initials, their name, their role, the assignment date and an "X" button to remove them. If empty: "No employee assigned".

### 2.9 Comments section

Header "Comments ({n})" (speech-bubble icon). Chronological thread: avatar, author name, relative time, text. At the bottom, an "Add a comment..." input area and a **Send** button.

> A posted comment **can neither be edited nor deleted**.

### 2.10 Modals

- **Work-order preview**: an embedded window (sandboxed `iframe`) displays the generated HTML document. Title "Work-order preview {number}", with "Open in a new tab" and "Close".
- **Add a line**: "Inventory product (optional)" menu (with the "Free entry (no product)" option and the products with their stock), "Description *", "Quantity", "Unit" ("m, kg, unit...") and "Unit price". Cancel / Add buttons.
- **Assign an employee**: "Employee *" menu and a "Role" field ("e.g. Welder, Team lead..."). Cancel / Assign buttons.

### 2.11 "Operations" tab (global view)

This tab displays **all ongoing operations** across all work orders, under the text "Overview of all ongoing operations across the work orders."

**Table** (columns): **WO** (number + name), **Operation**, **Qty**, **Assigned to**, **Supplier**, **Plan. hrs**, **Actual hrs**, **Status** (badge), **Actions** (full inline editing / deletion). Inline validation requires an operation name and zero-or-positive values for the quantity and hours. The footer shows "Totals ({n} operations)" and the sum of hours. Cards with an editing form on phone.

> **Global-view cap.** This table loads up to **200 operations** (beyond that, a banner signals that the list is truncated). This cap concerns **only** this global view: inside a work order, all of its operations are always displayed, with no limit. This is also the endpoint that, historically, appeared "empty" on older tenants — a fix added the missing technical column and raised the cap; the tab is now fully functional.

**"Weekly capacity per operation" card**: below the table, a workload tool. Week navigation (‹ › with the Monday-to-Sunday range) and a legend: **green** below 80%, **yellow** from 80 to 100%, **red** above 100% (hour-budget overrun). The table lists: Operation, WO, Planned hours, Actual hours, Progress (%) and a colored progress bar (utilization = actual hours / planned hours).

> The weekly capacity covers only operations whose **start date** falls within the selected week.

### 2.12 "Catalog" tab

Management of reusable **operation types** (table `operation_types`). These types feed the "Station/Operation" menu of the operation forms.

**Bar**: **New operation** button; search (accent- and case-insensitive); **Active only** checkbox.

**Table**: **Name**, **Category**, **Code** (monospace font), **Std hours** (standard hours), **Active** (Active / Inactive badge), **Actions** (Edit / Delete). Footer: "{filtered} / {total} operations".

**Create / edit window**: "Name *", "Category" ("e.g. Carpentry"), "Code" ("e.g. POSE-FEN"), "Standard hours" ("e.g. 4.0"), and the "Active (visible in the work-order drop-down)" checkbox. Renaming or deleting a type **warns** that existing work orders keep the old name (historical snapshot — there is no strong link between the operation and the catalog).

> **Eighteen types are provided by default** (see §4.5) if the catalog is empty. **Permission reminder**: only **administrators** and **super-administrators** can modify the catalog; a manager or a foreman cannot.

### 2.13 "AI Assistant" tab

An "AI Assistant — Work Orders" chat under the subtitle "Query your work orders and create them on confirmation." Example suggested questions: "Which work orders are in progress?", "Create a work order for project 15, HIGH priority, due Friday.", "Which operations are pending, by status?".

**Behavior**:

- The assistant **reads** the data using a strict table allowlist (`formulaires`, `formulaire_lignes`, `operations`, `projects`, `companies`). The **payroll, human-resources, employee and security** tables (SIN — Social Insurance Number, AI credits, Stripe, etc.) are **refused**.
- To create a work order, the assistant shows a **proposal card** (Name, Priority, Project, Start, End, Due date) with the **Cancel** and **Confirm** buttons. **Nothing is written until you confirm.** On confirmation, the server **re-checks the write right** and then creates the work order.
- The assistant **creates only work orders**: no operations, no lines, no assignments.

> Each exchange consumes prepaid **AI credits** (see §4.11). If the balance is exhausted, the call is refused.

---

## 3. Step-by-step processes

### 3.1 Create a work order (empty)

1. **Work Orders** tab → **New work order**.
2. Fill in as needed the **Name** (optional), the **Priority**, the **Project**, the **dates** and the **Notes**.
3. **Create**. The work order appears in status **Draft**, numbered `BT-00001`.

> If you leave the name empty while choosing a project, the work order takes the **project name**. With no name and no project, it takes its **number**. If the project is linked to an opportunity that has a case file, the work order is **attached to the case file** automatically.

### 3.2 Create a work order with pre-entered operations and materials

1. Open **New work order**.
2. In the **Operations** card, click **Add a task** and fill the subform; repeat for each task.
3. In the **Products / Materials** card, click **Add a product** and fill the lines.
4. **Create + {n} op. {m} prod.**. The server creates the work order, then each operation and each line.

> If an operation or a line fails to be created, the work order still exists; a message indicates the number of items not created (to be added afterward from the detail view).

### 3.3 Start, pause, resume, complete

1. Open the work order (click the row).
2. Depending on the status, click the desired transition:
   - **Draft** → **Start** (moves to In progress);
   - **In progress** → **Pause** (moves to Paused) or **Complete** (moves to Completed);
   - **Paused** → **Resume** (moves to In progress).
3. The status badge changes color.

> **The state machine protects the transitions.** You cannot jump from "Draft" to "Completed" directly, nor move from "Paused" to "Completed" without going back through "In progress". A **completed or cancelled work order is terminal**: only a super-administrator can pull it out.
>
> **Work-completion guard.** The server **refuses to move a work order to "Completed"** if there remain non-terminal operations (neither completed nor cancelled). Complete or cancel those operations first. (A super-administrator can override.)

### 3.4 Cancel a work order (soft-delete, stock restored)

1. Open a **non-completed** work order → **Cancel** (confirmation).
2. The status becomes **Cancelled**. **The stock consumed by the lines is fully restored** (return movements). The work order's history (operations, lines, comments) is **kept**.

> This is the normal way to "delete" an active work order: it leaves the workflow but stays consultable (the "Cancelled" filter) and its stock is returned.

### 3.5 Restore a cancelled work order

1. Open a **cancelled** work order → **Restore** (confirmation).
2. The status returns to **Draft** and **the stock is consumed again** (the lines re-debit the inventory).

> Restoration is the exact inverse of cancellation. The stock stays consistent in both directions.

### 3.6 Permanently delete a work order

1. The work order must **already be cancelled** (otherwise cancel it first, §3.4).
2. From the list (trash button on a cancelled row) **or** from the detail view (**Delete permanently**) → confirmation.
3. The work order and all its children (operations, lines, assignments, comments) are **physically erased**. **Irreversible** action.

> The delete button **appears only on cancelled work orders**. An active work order never shows this button in the list.

### 3.7 Add an operation (task)

1. Detail view → **Operations** section → **Add a task**.
2. Choose the **Station/Operation** (catalog or free name), the **Quantity**, the **employee**, the **Supplier/Subcontractor**, the **Planned hours**, the **Status**, the **dates**, the **Phase** and a **Description**.
3. **Save**. The operation appears under its phase's header.

> The server refuses to add an operation to a **completed or cancelled** work order (except super-administrator). It also verifies that the employee exists and that the status is valid.

### 3.8 Change an operation's status or actual hours

- **Status**: in the operations table, change the row's **Status** drop-down (Pending / In progress / Completed / Cancelled). Saved immediately.
- **Actual hours**: click the row's pencil for inline editing, enter the **Actual hours**, then save.

> **Actual hours are entered by hand.** No automatic carry-over from time tracking: the `time_entries` table does not feed the operations' actual hours. An operation's status is **independent** of the work-order status.

### 3.9 Add a material line (stock movement)

1. Detail view → **Lines** section → **Add**.
2. Choose an **inventory product** (or "Free entry"), the **Description**, the **Quantity**, the **Unit** and the **Unit price**.
3. **Add**. The line amount (quantity × price) and the work-order total recalculate.

> If the line is tied to a product, the stock is **debited** (issue movement) and traced in the inventory movements. The server refuses to add a line to a **completed or cancelled** work order (except super-administrator). Stock-level tracking and threshold alerts live in **Module 09 Store**.

### 3.10 Edit or delete a line

- **Edit** (pencil, inline editing): adjusts the amount and the total; if a product is tied, the stock is adjusted by the quantity **delta** (extra issue if the quantity increases, return if it decreases).
- **Delete** ("X"): removes the line, recalculates the total and **credits back** the tied product's stock (return movement).

> These stock adjustments are **suspended** if the work order is cancelled (the stock has already been returned by the cancellation).

### 3.11 Assign or unassign an employee

1. Detail view → **Assignments** section → **Assign**.
2. Choose the **Employee** and enter a free-text **Role** → **Assign**.
3. To remove an employee: click the "X" on their line.

> A given employee can only be assigned **once** per work order (a second attempt is refused). Removing an assignment does not affect operations where the employee is already designated.

### 3.12 Add a comment

1. Detail view → **Comments** section → input area → **Send**.
2. The comment is added to the thread with the author and the timestamp.

### 3.13 Generate and print the HTML / PDF document

1. Detail view → **HTML** or **Preview** (embedded window), or **PDF** (new tab).
2. The document reproduces the company header (configured theme and colors), the number, the project, the dates, the status, the priority, the material lines, the operations with their hours, and the assignments.
3. From the new tab (PDF), use the browser's print function (Ctrl+P) to print or **Save as PDF**.

> The document is **bilingual** (per the tenant's configured language) and **escaped** against HTML injection. It is generated server-side, with no external dependency.

### 3.14 Manage the operation-types catalog

1. **Catalog** tab → **New operation**.
2. Fill in **Name**, **Category**, **Code**, **Standard hours**, and the **Active** checkbox.
3. **Save**. The type appears in the "Station/Operation" menu of the forms.
4. To remove a type from the menu without erasing it: edit it and **uncheck "Active"**.

> Reserved for **administrators / super-administrators**. Deleting or renaming a type **does not change** operations already created.

### 3.15 Use the AI Assistant

1. **AI Assistant** tab.
2. Ask a question ("Which operations are pending, by status?") or request the creation of a work order ("Create a work order for project 15, HIGH priority").
3. For a creation, check the **proposal card** and then **Confirm**. The work order is created (after re-checking the write right).

> The assistant creates only a **work order**; add its operations and lines afterward from the detail view.

### 3.16 Generate an automatic schedule (from the Projects module)

The per-project-type operation schedule **is not triggered from this page**:

1. Go to the **Projects module**, open the project, click **Generate schedule**.
2. The server creates a "Schedule - {project}" work order (Draft status) and rolls out the operations of a **model** chosen according to the project category: **residential** (28 operations), **renovation** (18 operations) or **public** (24 operations). The dates are anchored to the project start.
3. The operation is **idempotent**: only one automatic-schedule work order per project (re-running it returns the existing schedule).

> The schedule is also generated **automatically when a quote is converted into a project**. Once created, it is driven like any other work order in this page.

### 3.17 Track weekly capacity

1. **Operations** tab → **Weekly capacity per operation** card.
2. Navigate from one week to another with ‹ ›.
3. Read the utilization: **green** (headroom), **yellow** (close to the budget), **red** (hour overrun).

---

## 4. Reference

### 4.1 Endpoints (API)

All prefixed with `/api/erp/v1/production` (except the AI Assistant, under `/api/erp/v1/bons-travail/ai`). "Write" = guard `require_tenant_admin_or_role(admin, super_admin, gestionnaire, contremaitre)`.

**Work order (core of the module)**

| Method + path | Role | Right |
|---|---|---|
| GET `/statistics` | KPI cards | read |
| GET `/work-orders` | Paginated list (`per_page` 1 to 100), status / priority / search filters | read |
| POST `/work-orders` | Create a (empty) work order | write |
| GET `/work-orders/{id}` | Header only (+ project name) | read |
| GET `/work-orders/{id}/detail` | Work order + lines + assignments + comments + operations | read |
| PUT `/work-orders/{id}` | Edit (state machine, stock transition on cancellation) | write |
| DELETE `/work-orders/{id}` | Delete (dual: soft if active, hard if cancelled) | write |
| POST `/work-orders/{id}/restore` | Restore a cancelled work order (→ Draft) | write |
| POST `/work-orders/{id}/generate-html` | Themed HTML document | read |
| GET `/work-orders/{id}/time-entries` | Linked time entries (read-only, **not shown in the page**) | read |

**Material lines**

| Method + path | Role | Right |
|---|---|---|
| GET `/work-orders/{id}/lines` | List | read |
| POST `/work-orders/{id}/lines` | Add (stock issue; refused if completed/cancelled) | write |
| PUT `/work-orders/{id}/lines/{lid}` | Edit (delta stock adjustment) | write |
| DELETE `/work-orders/{id}/lines/{lid}` | Delete (stock return) | write |

**Operations**

| Method + path | Role | Right |
|---|---|---|
| GET `/operations` | **Global view**, paginated (`per_page` 1 to 200, default 50) | read |
| GET `/work-orders/{id}/operations` | All operations of the work order (no limit) | read |
| POST `/work-orders/{id}/operations` | Add (refused if completed/cancelled) | write |
| PUT `/work-orders/{id}/operations/{oid}` | Edit | write |
| DELETE `/work-orders/{id}/operations/{oid}` | Delete | write |

**Assignments and comments**

| Method + path | Role | Right |
|---|---|---|
| GET / POST `/work-orders/{id}/assignations` | List / assign (uniqueness per employee) | read / write |
| DELETE `/work-orders/{id}/assignations/{aid}` | Remove | write |
| GET / POST `/work-orders/{id}/comments` | List / add | read / write |

**Catalog and schedule**

| Method + path | Role | Right |
|---|---|---|
| GET `/operation-types` | List the types ("active only" option) | read |
| POST / PUT / DELETE `/operation-types[/{id}]` | Manage the catalog | **administrator** (`_require_admin`) |
| POST `/projects/{id}/generate-cedule` | Generate a project's schedule (idempotent) | write |

**AI Assistant** (`/api/erp/v1/bons-travail/ai`)

| Method + path | Role | Limit |
|---|---|---|
| POST `/chat` | Query + propose a work order (no write) | 20 requests / min |
| POST `/confirm-action` | Create the work order after confirmation (re-checks the right) | 30 requests / min |

**Endpoints shared with the Tracking module** (same router, but attached to Tracking): `GET /gantt/bons-travail` (the work orders and their operations as subtasks), `PUT /kanban/update-status` (changes a work order's status from the Kanban — **blocks the "Cancelled" transition**, which must be done from this page to restore the stock), `GET /calendar-events` (work-order due dates in the Calendar).

### 4.2 Work-order statuses and lifecycle

Statuses: **BROUILLON**, **EN_COURS**, **EN_PAUSE**, **TERMINE**, **ANNULE**.

| Status | Color | Allowed transitions |
|--------|-------|---------------------|
| **BROUILLON** (Draft) | gray | → EN_COURS, → ANNULE |
| **EN_COURS** (In progress) | blue | → EN_PAUSE, → TERMINE, → ANNULE |
| **EN_PAUSE** (Paused) | amber | → EN_COURS, → ANNULE |
| **TERMINE** (Completed) | green | (terminal) |
| **ANNULE** (Cancelled) | red | (terminal) |

> **TERMINE and ANNULE are terminal**: only a super-administrator can reopen them. The transition to **TERMINE** is additionally blocked if there remain non-terminal operations. Legacy statuses (accents, spaces, variants) are normalized server-side before any check.

### 4.3 Priorities

**BASSE** (Low), **NORMALE** (Normal, default), **HAUTE** (High), **URGENTE** (Urgent). The "Urgent" priority adds an alert icon in the detail view.

### 4.4 Operation statuses

**Pending** (`En attente`, default), **In progress** (`En cours`), **Completed** (`Terminé`), **Cancelled** (`Annulé`). These are the exact stored strings — **case-sensitive** (first letter capitalized, with spaces). No automatic link with the work-order status.

### 4.5 Default operation catalog (18 types)

If the catalog is empty, these 18 types are proposed:

```
Demolition · Decontamination · Excavation · Foundation/Formwork ·
Structure/Framing · Plumbing · Electrical · HVAC · Insulation ·
Drywall/Plaster · Painting · Roofing · Exterior cladding ·
Carpentry/Finishing · Flooring · Tiling · Landscaping ·
Final cleaning
```

A custom name (off-catalog) is still possible in the operation form.

### 4.6 Schedule models (per project type)

Applied **outside this page** (Projects module or quote → project conversion):

| Category | Number of operations | Indicative span |
|----------|----------------------|-----------------|
| **Residential** | 28 | ≈ day 0 to day 181 |
| **Renovation** | 18 | ≈ day 0 to day 98 |
| **Public** | 24 | ≈ day 0 to day 245 |

The category is inferred by the priority order **renovation > public > residential**. Each model places the operations in "Pending", with no employee, with dates anchored to the project start. Only one automatic-schedule work order per project.

### 4.7 Calculations

| Item | Formula | Triggered by |
|------|---------|--------------|
| **Line amount** | `quantity × unit price` | adding / editing a line |
| **Work-order total** | `Σ of line amounts` | adding / editing / deleting a line |
| **Line / operation sequence** | `MAX(sequence) + 1` (lock on the parent work order) | on creation |
| **Operation progress** (Gantt) | `actual hours / planned hours × 100` | display |
| **Weekly utilization** | `actual hours / planned hours` (80% / 100% thresholds) | display |

> **No GST/QST or margin calculation.** The total amount is an operational indicator, not a tax document.

### 4.8 Stock invariant

"Non-cancelled work order ⇔ stock consumed; cancelled work order ⇔ stock restored." The switch happens only when the "Cancelled" boundary is **actually crossed**:

| Event | Effect on stock |
|-------|-----------------|
| Adding a line tied to a product | **Issue** (debit) |
| Quantity change (delta > 0) | **Issue** of the delta |
| Quantity change (delta < 0) | **Return** of the delta |
| Deleting a line tied to a product | **Return** (credit) |
| Work-order cancellation (soft-delete) | **Return** of all lines (stock restored) |
| Work-order restoration (→ Draft) | **Issue** of all lines (stock re-consumed) |

> Line mutations on an **already-cancelled** work order do **not** adjust the stock (it has already been returned). The Kanban **forbids** cancelling a work order (you must go through this page for the stock to be restored). Every movement is traced in `mouvements_stock`.

### 4.9 Validations and limits

| Rule | Effect |
|------|--------|
| Empty name and a project chosen | Name = project name |
| Empty name and no project | Name = work-order number |
| Name over 255 characters / notes over 5,000 | Rejected |
| End date earlier than start date | Rejected |
| Disallowed status transition | Rejected (except super-administrator) |
| Moving to "Completed" with non-terminal operations | Rejected (except super-administrator) |
| Adding a line / an operation to a completed or cancelled work order | Rejected (except super-administrator) |
| Line quantity out of range (0 to 1,000,000) / price (0 to 10,000,000) | Rejected |
| Product quantity × price above 10^12 | Rejected (numeric-overflow protection) |
| Operation hours out of range (0 to 100,000) | Rejected |
| Non-existent employee (operation or assignment) | Rejected |
| Employee already assigned to the work order | Rejected (uniqueness) |
| Deleting a catalog type by a non-administrator | Rejected |
| Permanently deleting a non-cancelled work order | Not possible (cancel it first) |

### 4.10 Numbering

Format `BT-00001` (5 digits, leading zeros). The number is assigned in a **concurrency-safe** way: the work order is inserted with a temporary number, then renumbered from its identifier. Never `MAX + 1`.

### 4.11 Money effect and AI credits

The module **bills nothing** through Stripe or QuickBooks. The "Total amount" field issues no invoice. The **only monetary effect** is the **debit of prepaid AI credits** when the AI Assistant is used: the actual token cost ($0.003/thousand input, $0.015/thousand output) is **marked up by 30%**. If the credits are exhausted, the assistant is refused. The **only material effect** is the inventory stock movement triggered by the material lines (see §4.8).

### 4.12 PostgreSQL tables (per tenant)

| Table | Role |
|-------|------|
| `formulaires` | Work-order header (`type_formulaire = 'BON_TRAVAIL'`): number, name, status, priority, project, dates, amount, notes |
| `formulaire_lignes` | Material lines (description, quantity, unit, price, amount, tied product) |
| `operations` | Operations / tasks (name, phase, employee, supplier, planned and actual hours, status, dates) |
| `bt_assignations` | Employees assigned to the work order (role, date) — no strong foreign key |
| `bt_comments` | Comments (author, text, date) — no strong foreign key |
| `operation_types` | Catalog of operation types (name, category, code, standard hours, active) |
| `produits` / `mouvements_stock` | Inventory affected by the material lines |
| `time_entries` | Time entries linked to the work order (read-only) |
| `dossier_formulaires` | Attachment of the work order to the opportunity case file |
| `public.operation_templates` | Schedule models shared between tenants |

> The `formulaires` table (shared with quotes, purchase orders and invoices according to their `type_formulaire`) is **not** defined by the source code: it is provisioned by copy from a reference tenant, and the code defensively guarantees its columns at run time.

---

## 5. Integrations and FAQ

### 5.1 Links with the other modules

| Module | Link |
|--------|------|
| **Projects** (Module 08) | A work order can be attached to a project; the "Generate schedule" button (Projects page) creates a schedule work order; converting a quote into a project also generates the schedule automatically. |
| **Store / Inventory** (Module 09) | Material lines tied to a product debit the stock (issue) and credit it back on deletion or cancellation (return). Movements and threshold alerts are consulted in the Store. |
| **Tracking (Kanban / Gantt / Calendar)** (Module 02) | The same work orders display in these views. The Kanban changes their status (but blocks cancellation, to be done here); the Gantt shows operations as subtasks; the Calendar shows the due dates. |
| **Time Tracking** (Module 12) | Employees clock their hours; these time entries are **linked** to the work order but **do not carry over** into the operations' actual hours (manual entry). |
| **Employees** (Module 10) | Assignments and an operation's employee are chosen from the tenant's employees. |
| **Case Files / 360 View** (Module 06) | A work order attached to a project linked to an opportunity is **auto-attached** to the corresponding case file. |
| **Configuration** (Module 30) | The logo, the document colors and the language come from the company configuration. |
| **AI Assistant** (Module 24) | The module's assistant consumes the tenant's prepaid AI credits. |

### 5.2 FAQ

**How do I create a work order with a ready-made operation schedule?**
Not from this page. The "New work order" form creates an **empty** work order. Use the "Generate schedule" button of the Projects module (or convert a quote into a project). The schedule picks a model according to the project type (residential 28, renovation 18, public 24 operations).

**Are there really four tabs?**
Yes: **Work Orders**, **Operations**, **Catalog**, **AI Assistant**. A work order's detail view is not a tab (it opens on top). The Kanban, the Gantt and the Calendar, for their part, are in the Tracking module.

**Does cancelling a work order restore the stock?**
Yes. Cancelling a work order (soft-delete) **automatically** credits back all the stock consumed by its lines. Restoring it re-consumes it. This is an important change from older versions, where cancellation did not touch the stock.

**How do I permanently delete a work order?**
You must first **cancel** it (soft-delete). Once cancelled, the "Delete permanently" button appears (in the list and in the detail view) and physically erases the work order and its children. An active work order cannot be deleted in one step.

**Do the operations' actual hours fill in from time tracking?**
No. They are entered **by hand** (operation editing). No automatic carry-over from the time-entries table.

**Why can't I move my work order to "Completed"?**
Two possible reasons: the transition is not allowed from the current status (for example directly from "Draft" or "Paused"), or there remain **non-terminal operations**. Complete (or cancel) those operations first, going through "In progress" if needed.

**Who can edit the work orders?**
Administrators, super-administrators, managers and foremen. The **catalog** of operation types is more restricted: administrators and super-administrators only.

**Is the total amount editable?**
No. It always calculates from the material lines. There is no editable amount field, and the work order issues neither invoice nor taxes.

**Can the AI Assistant do everything?**
No. It **reads** the work orders (never payroll, employees or security data) and can **propose** a work order, created only after confirmation. It creates neither operation, nor line, nor assignment.

**Why did the "Operations" tab appear empty before?**
That was a corrected defect: a technical column was missing on older tenants and the display cap was low. The global view now loads up to 200 operations (with a warning beyond that); inside a work order, all of its operations are always displayed.

**Can I edit several fields directly in the list?**
Only the **Start** and **End** dates (inline editing). The other fields are edited from the detail view.

**Can the stock go below zero?**
Level tracking and threshold alerts are managed in the Store module. The work order simply records the movements there; consult the Store for the real inventory status.

### 5.3 What does not exist (known limits)

- No schedule generation from the creation form (it goes through the Projects module).
- No invoice, taxes or accounting export; the total amount is informational.
- No automatic carry-over of time tracking into the operations' actual hours.
- No "Time Tracking" section displayed in the page (the endpoint exists, but is not rendered).
- No editable amount field.
- No direct permanent deletion of an active work order (cancel it first).
- Global operations view capped at 200 rows; the weekly capacity covers only operations whose start falls within the chosen week.
- AI Assistant limited to the creation of a work order, on confirmation.
- The Kanban, the Gantt and the Calendar belong to the Tracking module, not to this page.

---

## 6. Summary

- The **Work Orders** module (`/bons-travail`, Operations section) brings together **4 tabs**: **Work Orders**, **Operations**, **Catalog**, **AI Assistant**, plus a contextual **detail view**.
- **4 KPI cards** at all times: Total, In progress, Completed, Total amount.
- **Governed lifecycle**: Draft → In progress → (Paused) → Completed, or Cancelled. Strict state machine (TERMINE and ANNULE terminal, except super-administrator), with a "no completion while non-terminal operations remain" guard.
- **Creation = empty work order**: no operation model is rolled out here; the schedules (residential 28 / renovation 18 / public 24) go through the Projects module or the conversion of a quote.
- **Dual deletion**: an active work order is **cancelled** (soft-delete, **stock restored**); a cancelled work order is **permanently deleted** (hard-delete). Restoring a cancelled work order re-consumes it.
- **Stock invariant**: material lines tied to a product debit the stock (issue) and credit it back on deletion or cancellation (return); mutations on a cancelled work order do not adjust the stock.
- **Operations** grouped by **phase**, with planned and actual hours (manual entry, no time-tracking carry-over), status editable inline, employee and subcontractor. Global view (max 200) + **weekly capacity** (green / yellow / red).
- **Catalog** of operation types (18 by default), reserved for administrators; renaming a type does not change existing operations.
- **AI Assistant**: reads the work orders (never payroll or employees), **proposes** a work order created on confirmation (right re-checked), and nothing else.
- **HTML/PDF document** themed and bilingual, generated server-side.
- **Permissions**: work-order write for admin / super-admin / manager / foreman; catalog write for admin / super-admin only; reading open to any user of the tenant.
- **Money effect** limited to **AI credits** (assistant); the **total amount issues no invoice** and there are **no taxes**.
- **Do not belong to this page**: the Kanban, the Gantt and the Calendar (Tracking module), and time-tracking entry (Time Tracking module).

---

**Documentation generated from the source code**: `BonsTravailPage.tsx`, `components/bt/BtAssistantTab.tsx`, `api/production.ts`, `store/useProductionStore.ts`; `backend/routers/production.py`, `bt_ai.py`, `operation_templates.py`.

**Related manuals**:
- Module 02 — Tracking & Gantt (Kanban, Gantt, work-order Calendar) — `02-suivi-gantt.md`
- Module 08 — Projects (schedule generation) — `08-ventes-projets.md`
- Module 09 — Store (material stock movements) — `09-operations-magasin.md`
- Module 10 — Employees (assignments, operation employee) — `10-operations-employes.md`
- Module 12 — Time Tracking (employee hours) — `12-operations-pointage.md`
- Module 24 — AI Assistant (AI credits) — `24-communication-assistant-ia.md`
- Module 30 — Configuration (document theme, language) — `30-configuration.md`
