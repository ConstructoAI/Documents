# Module 02 — Tracking (Kanban, Gantt, Calendar)

> **Version**: 3.0 (overhaul verified against the current source code — replaces v2.0, which described 3 tabs, 6 Kanban boards and 3 zoom levels)
> **Reference code**: `frontend/src/pages/SuiviPage.tsx` (8,267 lines, 4 tabs), `frontend/src/pages/suivi/SuiviAssistantTab.tsx`, `frontend/src/components/suivi/QuickCreateModal.tsx` (1,301 lines), `frontend/src/components/suivi/SuiviShareButton.tsx`
> **Backends**: `backend/routers/production.py` (core: Kanban, Gantt, Calendar, operations, assignments, sharing), `backend/routers/projects.py` ("Projects" Gantt with phases + project assignments), `backend/routers/suivi_ai.py` (AI Assistant, read-only). Cross-cutting dependencies: `crm.py` (Sales/opportunities), `devis.py`, `suppliers.py`, `employees.py`, `ai.py` (Calendar widget).
> **Route**: `/suivi` — sidebar, "Tracking" section, "Tracking" label
> **Interface labels**: i18n namespace `crm.suivi.*` (file `crm.json`, lines 367 to 1038). A few Gantt accessibility labels live under `production.gantt.a11y.*`.
> **PostgreSQL tables**: the module has **no business tables of its own**; it composes entities from other modules. Technical tables it creates and maintains itself (self-repair): `gantt_dependencies`, `gantt_task_baselines`, `calendar_notes`, `bt_assignations`, `devis_assignations`, `achat_assignations`, `project_assignments`, plus the shared table `public.calendar_share_tokens` (public sharing).
> **Scope**: a **cross-module** control center for viewing and light editing. It aggregates Sales (CRM), Quotes, Projects, Work Orders (WO), Operations, Purchases (purchase orders) and Invoices from four angles: **Kanban**, **Gantt**, **Calendar** and **AI Assistant**. It does not replace the source modules: actions with an accounting effect (invoices) or an inventory effect (goods receipt, work-order cancellation) remain **blocked** here and point back to the dedicated module.

---

## Contents

1. [Overview and access](#1-overview-and-access)
2. [Interface](#2-interface)
3. [Step-by-step workflows](#3-step-by-step-workflows)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview and access

### 1.1 What the Tracking module is for

The **Tracking** module is the dashboard of a project manager or supervisor: it brings together on a single page, and in several forms, all of the company's work in progress. Instead of opening the Sales, Quotes, Projects, Work Orders, Purchases and Accounting modules one after another, the user sees everything in one place and can act directly:

- **Kanban** — drag cards from one column to another to move a file forward, reorder priorities within a column, assign employees.
- **Gantt** — visualize the construction schedule as bars, move and resize dates, link tasks with dependencies, draw a critical path and a baseline, export the schedule.
- **Calendar** — see deadlines by day, week, month or in agenda form, reschedule an event by dragging, add notes, reassign operations, get a summary from the Claude assistant.
- **AI Assistant** — ask natural-language questions about progress ("which projects are late?") and get an answer grounded in the tenant's real data, **read-only**.

### 1.2 Access and prerequisites

- **Sidebar**: "Tracking" section → **Tracking**.
- **Address (URL)**: `/suivi`.
- **Authentication** is required (`Depends(get_current_user)`) along with a **tenant context** (your company). Without a tenant context, the API returns `400`.
- **Read-only mode**: if your subscription is in read-only mode, all writes (drag-and-drop, date editing, assignments, notes, sharing) are blocked upstream; viewing remains available.
- **AI Assistant**: requires an active Anthropic key, the AI guardrail set to green and sufficient **AI credits**. Otherwise: `503` (service unavailable), `403` (AI access denied) or `402` (credits depleted).

### 1.3 The four tabs

The tab selector at the top of the page (`SuiviPage.tsx:418-443`, key `MainTab`) offers, in order:

| Tab | Internal key | Icon | Role |
|---|---|---|---|
| **Kanban** | `kanban` | Kanban | Card boards by status (7 boards) |
| **Gantt** | `gantt` | BarChart3 | Schedule chart (6 sources) |
| **Calendar** | `calendrier` | Calendar | Monthly/weekly/daily agenda |
| **AI Assistant** | `assistant` | Sparkles | Read-only tracking chat |

Kanban is the tab shown by default.

### 1.4 Roles and permissions (overview)

The module deliberately applies differentiated guards: **viewing** is open to any tenant member, **editing** is reserved for operational roles.

| Action | Who can do it |
|---|---|
| View Kanban, Gantt, Calendar, Assistant | Any authenticated tenant member |
| **Reorder** Kanban cards (vertical) | Any member (purely cosmetic gesture) |
| **Change status** via drag-and-drop (Kanban), edit Gantt dates/status/assignee, create/delete dependencies, set a baseline, assign an employee to a work order or a purchase, reassign an operation | `admin`, `super_admin`, `gestionnaire`, `contremaitre` (the `BT_WRITE_ROLES` roles) |
| Create/edit/delete a calendar **note** | Any member; a **shared** note can also be edited by an admin; only the author can change the personal/shared scope |
| Create/edit project **phases** or **assignments** | Any member (via the "Projects" Gantt) |
| **Generate / revoke** the public sharing link | `admin` or `super_admin` only (others can see and copy the existing link, but can neither create nor revoke it) |
| View the sharing **public page** | Anyone who has the link, **without logging in** |

> The `super_admin` role bypasses the work-order state machine and the "a work order can be completed only if its operations are complete" guard.

---

## 2. Interface

### 2.1 General layout

```
+----------------------------------------------------------------------+
|  Tracking                                                            |
+----------------------------------------------------------------------+
|  [ Kanban ] [ Gantt ] [ Calendar ] [ AI Assistant ]                 |
+----------------------------------------------------------------------+
|                                                                      |
|   Content of the active tab                                         |
|                                                                      |
+----------------------------------------------------------------------+
```

The Gantt and Calendar tabs switch to full height (dense layout); Kanban and Assistant keep standard scrolling. Any load error is shown in a dismissible red banner at the top of the content area.

### 2.2 Kanban tab

The Kanban tab (`KanbanTab`, `SuiviPage.tsx:631-1913`) presents **7 boards** selectable via pills, each with its own status columns.

#### The 7 boards and their columns

| Board | Label | Columns (left to right) | Source |
|---|---|---|---|
| Sales | **Sales** | `Prospecting`, `Qualification`, `Proposal`, `Negotiation` (won/lost are summarized in a callout) | CRM opportunities |
| Quotes | **Quotes** | `Draft`, `Validated`, `Sent`, `Pending`, `Accepted`, `Refused`, `Completed` | Quotes |
| Projects | **Projects** | `Pending`, `In progress`, `Suspended`, `Completed` | Projects |
| Purchases | **Purchases** | `Draft`, `Sent`, `Confirmed`, `In progress`, `Received`, `Invoiced`, `Cancelled` | Purchase orders |
| Work Orders | **Work Orders** | `Draft`, `In progress`, `Paused`, `Completed` | Work orders |
| Operations | **Operations** | `Pending`, `In progress`, `Completed`, `Cancelled` | Work-order operations |
| Invoices | **Invoices** | `Draft`, `Sent`, `Partially paid`, `Paid`, `Overdue` | Invoices (**read-only**) |

> The columns deliberately cover **all** possible statuses of each entity: this way, no card "disappears" from the board because its status would match no column.

#### Top bar

Above the columns: **Refresh** button, **Share** button (see 2.7), **Create** button (opens the quick-create modal — see 2.6). The **Create** button is hidden on the Invoices board (invoices are not created here).

#### Left side panel (workspace, on desktop)

A 288 px column presents summary cards for the active board:

- **Active view** — the name of the current board.
- **Information** — an overall progress bar (%) and three counters: "To do", "In progress", "Completed".
- **Total budget** — the sum of the amounts of the displayed cards, formatted as dollars (for example `$125,000.00`).
- **By status** — for each column, the number of cards and a proportional mini-bar.
- **By employee** — the workload split by assigned employee. **Clicking an employee filters the board** to show only their cards; an "Unassigned" row groups cards with no assignee. This card is **absent** for the Invoices board.

On mobile, the side panel is replaced by a compact stats bar; the columns scroll horizontally with ‹ › arrows and position dots.

#### Anatomy of a card

```
+------------------------------------------+
|  Title (name / number)             [⋮]   |
|  Company                       Amount    |   <- Sales
|  ███████░░░  Probability 65 %            |   <- Sales
|  Supplier: ...                           |   <- Purchases
|  Start: Jan 12, 2026   End: Jan 30       |
|  (assignee avatars)       [+]  [Due]     |
+------------------------------------------+
```

Depending on the board, a card displays: the title (name, number or title), the company and the amount (Sales), a probability bar (Sales), the supplier (Purchases), the start and end dates, a **due badge** ("Overdue", "Today" or "{n}d left"), the **avatars** of the assigned employees (photo or initials, with a "+N" beyond a certain number), a **+** button to add an assignee, and a **priority** badge (URGENT / HIGH) where applicable.

- **A single click** opens the detail modal.
- **A double-click** opens the source record in its module: Sales → `/ventes`, Quotes → `/devis`, Projects → `/projets`, Work Orders and Operations → `/bons-travail`, Purchases → `/magasin`, Invoices → `/comptabilite`.
- The **+** button (hidden on Invoices) opens the employee assignment window.

#### Modals and states

- **Card detail** — status, priority, amount, start date, due date (with badge), supplier, "Assigned team" with an **Add** button (hidden on Invoices) and a **Close** button.
- **Assign an employee** — a search field and a list of employees (up to 100 loaded).
- **Mobile "Move to" sheet** — on mobile, replaces drag-and-drop: it lists the target columns.
- **Notifications** — a green banner ("Status updated successfully", "Employee assigned successfully") or a floating red banner on failure.

### 2.3 Gantt tab

The Gantt tab (`GanttTab`, `SuiviPage.tsx:1926-5426`) shows an MS Project–style schedule chart: a table on the left, a timeline on the right.

#### The 6 sources

Selectable via pills: **Sales**, **Quotes**, **Projects**, **Purchases** (purchase orders), **Work Orders**, **Operations**. Projects and Work Orders display collapsible **subtasks** (project phases; a work order's operations) via a chevron.

#### Legend

Four status colors: **Pending**, **In progress**, **Completed**, **Cancelled**.

#### Toolbar (on the right)

| Control | Effect |
|---|---|
| **Zoom** (5 levels) | `24h`, `3 days`, `Week`, `2 Wks`, `Month`. On mobile, a drop-down replaces the pills. |
| **Date** selector + ‹ › + **Today** | Visible at `24h` / `3 days` zoom (the hourly "site view") to move hour by hour. |
| **Search** | Filters rows by text. |
| **Dependencies** | Shows or hides the link arrows between bars. |
| **Critical path** | Draws the critical path (zero-float tasks, in working days); a red ring on the critical bars. |
| **Baseline** | Shows the frozen plan (grey bar) and the schedule variance; a Set / Update / Clear panel. |
| **Export CSV** | Downloads the schedule as CSV. **Disabled for the Operations source.** |
| **Print** | Launches the browser print dialog (`window.print()`). |
| **Refresh**, **Share**, **Create** | Reload, public sharing, quick create. |

#### Left panel (MS Project–style columns)

**Sortable** columns (click the header) and **resizable** columns (handles): **Number**, **Name**, **Project**, **Amount**, **Priority**, **Status**, **Assignee**, **Supplier**, **Start**, **Duration**, **End**, **%**. Not all columns are shown for all sources (for example, Project and Supplier appear only for Work Orders, Purchases and Operations; Priority is absent from Quotes and Purchases).

Cells that can be **edited directly** (double-click):

- **Status** — a drop-down. For work orders, only the valid transitions from the current status are offered (see the state machine in 4.4).
- **Assignee** — a drop-down of employees.
- **Supplier** — editable only for Operations and work-order sub-rows (options: "Internal" + Store suppliers).
- **Start** and **End** — date fields.
- **Duration** and **%** are **read-only** (the % is computed automatically).

#### Timeline (the bars)

- Bars colored by status; **drag** to move, **resize** from the left or right edge (which changes the dates).
- A **drag handle** on the left of each row to reorder vertically.
- **Milestones** shown as a diamond when the duration is one day or less.
- A red **"now" line** (with the exact time in hourly view).
- A grey **baseline bar** and a red **critical-path ring** when enabled.
- Parent rows (work order → operations, project → phases) are collapsible.

#### Dependencies

Panel table: **Source**, **Target**, **Type** (FS / SS / FF / SF drop-down), **Lag (d)** and delete. Links also appear as **arrows** between bars. You create a dependency by **dragging from the right edge** of a bar onto another (link mode, "Link mode" banner, `Esc` to cancel). Clicking an arrow opens a delete confirmation. The arrows are **hidden** in hourly view (24h / 3 days).

#### Tooltip

On hover/click of a bar: title, status, Start, End, Progress %, Budget, Manager. A double-click on the bar or the number opens the source record.

#### Remembered preferences

Zoom, source, dependency display, critical path, baseline, displayed date, panel width, column widths and collapsed rows are kept in the browser (key `erp.suivi.gantt.prefs.v1`) and restored the next time you open the page.

### 2.4 Calendar tab

The Calendar tab (`CalendarTab`, `SuiviPage.tsx:5537-8267`) is a full agenda.

#### Four view modes

**Month**, **Week**, **Day**, **Agenda**. Keyboard shortcuts: `Shift+M`, `Shift+S`, `Shift+J`, `Shift+A`; `Shift+T` returns to today.

#### Header

‹ › navigation + the period title + a **Today** button; view toggle; a **Search** field (shortcut `/`); a **Share** button (read-only public link); a **Claude** / "Ask Claude" button (see below).

#### Filters

- **Employee** — an "All employees" drop-down + list; filters the calendar to one employee.
- **10 event types** as pills to enable/disable: Opportunity, Quote, Project, Purchase order, Work order, Invoice, Interaction, CRM Activity, Note, Operation.
- A search-result counter and a **Reset** button.

Preferences (view, filters) are remembered in the browser (`erp.suivi.calendar.prefs.v1`).

#### Month view

A 7-column grid with **ISO week numbers** and **Quebec public holidays** (2024–2030, red flag). Each cell shows the day's events and notes. On hover (on desktop), two buttons appear: **add a note** and **create an item**. Clicking a cell selects the day and opens a side panel (desktop) or a bottom sheet (mobile). **Dragging an event** onto another cell **reschedules** it (shifts its dates).

#### Week view

7 day-columns, headers with holidays, events per column, a create button per day, dragging to reschedule.

#### Day and Agenda views

A list/timeline of the day's or period's events, with navigation to the source record.

#### Free notes (sticky-note style)

You create and edit notes directly in a cell (mostly in **Month view**; the Week and Day views show the hint "open in Month view to edit"). Each note has a **scope**:

- **Personal** — visible to you only.
- **Shared** — visible to the whole company.

The note shows its author ("by …") and a Personal/Shared badge. Deletion asks for confirmation. **Only the author** can toggle a note from personal to shared (and vice versa); a shared note can also be edited or deleted by an administrator.

#### Operations in the calendar

For Operation-type events, you can directly reassign the **employee** and the **subcontractor/supplier**. This right is reserved for the `admin`, `super_admin`, `gestionnaire`, `contremaitre` roles; others see the assignment as read-only.

#### Calendar assistant — "Ask Claude"

The **Claude** button opens the "Claude Assistant — Calendar" modal: clickable suggestions, a character counter (1000 max), a privacy note ("No data leaves your tenant"), a **New** button (clears the conversation) and a **Send** button.

> **Important** — this widget is **distinct** from the AI Assistant tab (see 2.5). It uses the ERP's **general** assistant with a **compact calendar context** (up to 50 visible events); it is a summary/suggestion tool. It **also debits** AI credits.

### 2.5 AI Assistant tab

The **AI Assistant** tab (`SuiviAssistantTab.tsx`) is a **read-only chat** dedicated to Tracking. Header "AI Assistant — Tracking", subtitle "Track progress from your real data (read-only)." The empty state offers three example questions:

- "Which projects are in progress and which are late?"
- "What is my sales pipeline status by stage?"
- "Which work orders are not yet completed?"

The assistant queries the real data (sales, projects, quotes, work orders, purchases, operations) and answers about progress and status. It **writes nothing**: no "propose/confirm" card, no creation or modification. Each answer displays metadata (tokens, cost, duration). A lock prevents sending two questions at the same time.

### 2.6 Shared element — the "Create" modal

The **Create** button on the Kanban, Gantt and Calendar tabs opens the same quick-create modal (`QuickCreateModal.tsx`). The default type follows the active board/tab.

**6 types**: **Project**, **Opportunity**, **Quote**, **Work order**, **Purchase order**, **Operation**.

Fields depending on the type: Type, Name (required except for a purchase order), Parent work order (required for an operation), Employee and Supplier (operation), Supplier (required for a purchase order), Status, Priority (Low / Normal / High / Urgent — absent from purchase orders and operations), Customer (project/opportunity/quote), Associated project (quote/work order/purchase order), start and end dates, due/close/delivery/expected date (the label varies), Amount/Budget/Estimated price, Probability % (opportunity), Source (opportunity), Submission type Detailed/Budgetary (quote), Customer PO, Site address (project), Description and Notes.

**Batch operations** (work orders only): an "Operations" section lets you add several operations at once (Name required, Description, Quantity, Planned hours, Status, Internal/External supplier, Employee, dates, Workstation). They are created right after the work order; on partial failure, a message reports the number of operations in error.

Two submit buttons: **Create** (stays on the view and refreshes) and **More details →** (creates, then opens the full record page; hidden for work orders).

### 2.7 Shared element — public sharing

The **Share** button (present in all three views) gives access to **a single read-only public link per company**. The `?view=` parameter opens the right view (calendar, Gantt or Kanban) from the same link.

- **Generate** and **Revoke** the link are reserved for administrators; any other user can view and copy an already-active link, but cannot create one.
- **Scope** (restated in the interface): the link shows **only the schedule** — projects, work orders and **shared notes** (their titles and dates). It **never** exposes any quote, invoice, amount, opportunity or personal note. The link stays valid **until revoked**.

> Tip: since the titles are publicly visible, avoid entering sensitive data in them.

---

## 3. Step-by-step workflows

### 3.1 Move a Kanban card forward (status change)

1. Open the **Kanban** tab and choose the board (Sales, Quotes, Projects, Purchases, Work Orders, Operations).
2. On desktop, **drag** a card to another column; a dotted placeholder appears in the target column.
3. On drop, the card changes column immediately (optimistic update), then the API records the new status: `PUT /production/kanban/update-status` (or `crmApi.updateOpportunity` for Sales, `updateOperation` for Operations).
4. If the server refuses (forbidden transition, action reserved for another module), the card **snaps back** and a red message appears.
5. On mobile: drag-and-drop is replaced by the card's **Move** button, which opens the list of target columns.

> **Deliberate blocks** (drag-and-drop is not a universal shortcut): you cannot change an **invoice's** status here (driven by Accounting); you cannot move a **purchase** to "Received" or "Invoiced" (receiving = Store module) nor un-validate a purchase order already received/invoiced; you cannot move a **quote** to "Accepted" (goes through Quotes, which creates the linked project); you cannot **cancel** a work order from the Kanban (cancellation = WO module, to restore stock). See 4.3.

### 3.2 Reorder cards within a column

1. Drag a card **vertically** inside its column.
2. The order is saved via `PUT /production/kanban/reorder` (or `crmApi.reorderOpportunities` for Sales).
3. This gesture is **purely cosmetic**: it has no business effect and is therefore allowed to any tenant member, including for Invoices and Operations.

### 3.3 Assign an employee to a card

1. Click the **+** button next to a card's avatars (on mobile, go through the detail modal).
2. In the **Assign an employee** window, search by name and click the desired employee.
3. A green message confirms ("Employee assigned successfully") and the board reloads.

> The **Invoices** board does not allow assignment (no **+** button).

### 3.4 Filter a Kanban board by employee

1. In the side panel, **By employee** card, click an employee.
2. The board now shows only their cards; the "Unassigned" row isolates cards with no assignee.
3. Click again (or "Show all") to remove the filter.

### 3.5 Move or resize a Gantt bar

1. Open the **Gantt** tab and choose the source.
2. **Drag the center** of a bar to move it, or **grab an edge** (left/right) to resize it.
3. On release, the new dates are saved to the matching entity (project, work order, operation, quote, purchase order, opportunity).
4. To reorder a row, use the **drag handle** on the left.

> In **24h** or **3 days** view, bars are clickable (tooltip) but **not draggable**, and the dependency arrows are hidden.

### 3.6 Edit a Gantt cell (status, assignee, supplier, dates)

1. Double-click the cell to edit in the left panel.
2. Choose/enter the value: **Status** (drop-down; valid transitions for work orders), **Assignee** (employees), **Supplier** (Operations and work-order sub-rows only), **Start** / **End** (dates).
3. The value is saved immediately. Duration and % stay read-only.

### 3.7 Create a dependency between two bars

1. Enable the **Dependencies** display.
2. **Drag from the right edge** of a bar (link mode, "Link mode" banner) onto the target bar; `Esc` cancels.
3. Choose the **Type** (FS / SS / FF / SF) and the **Lag (d)** in the dependencies panel as needed.

> The server **refuses** a dependency that would create a **cycle** ("This dependency would create a cycle") and forbids linking an entity to itself. The lag is bounded to ±3650 days.

### 3.8 Delete a dependency

1. Click the dependency's **arrow** in the timeline (or its row in the panel).
2. Confirm in the "Delete this dependency?" window.

### 3.9 Set, update or clear a baseline

1. Enable **Baseline**.
2. **Set baseline** freezes the current dates (reference plan); a grey bar appears under each bar.
3. **Update** re-freezes to the current state; **Clear** removes the baseline — note that clearing applies to **all views** and asks for confirmation ("cannot be undone").

### 3.10 Show the critical path

1. Enable **Critical path**. Tasks with no float are marked with a red ring and a "critical" tag; others show their float ("F:{n} d").
2. If there is **no dependency** in the view, the critical path is limited to the longest task; link tasks for real scheduling.

### 3.11 Export the Gantt to CSV / print

1. **Export CSV** aggregates Projects + Work Orders + Quotes + Purchase orders + Dependencies into a single file (with a UTF-8 header for Quebec Excel).
2. **Print** launches the browser's print dialog (choose "Save as PDF" if needed).

> CSV export is **disabled** when the active source is **Operations**.

### 3.12 Navigate and change view in the calendar

1. Use ‹ › to change the period, **Today** to return to today's date.
2. Switch between **Month / Week / Day / Agenda** (or `Shift+M/S/J/A`).
3. Use **Search** (`/`) and the **type filters** to narrow down; **Reset** clears everything.

### 3.13 Create an item from a calendar day

1. In Month or Week view, hover a cell and click **create an item** (or the **Create** button).
2. The quick-create modal opens with the date pre-filled; complete it and click **Create** (or **More details →**).

### 3.14 Reschedule an event by dragging

1. In Month or Week view, **drag** an event onto another day.
2. The entity's dates are shifted automatically.

> Some types cannot be moved from the calendar; a message says so ("Items of type … cannot be moved from the calendar").

### 3.15 Add a note to the calendar

1. In **Month view**, click **Add a note** on the day's cell (or double-click the cell).
2. Enter the text (Enter to save, Esc to cancel; 500 characters max).
3. Choose the scope: **Make shared** (visible to the whole company) or **Make personal** (you only).
4. To edit/delete later: reopen in Month view. An administrator can also manage **shared** notes.

### 3.16 Reassign an operation from the calendar

1. On an **Operation**-type event, open the assignment selectors.
2. Choose an **employee** and/or a **subcontractor/supplier**.
3. Reserved for the `admin`, `super_admin`, `gestionnaire`, `contremaitre` roles.

### 3.17 Query the Tracking AI Assistant

1. Open the **AI Assistant** tab.
2. Ask a question about progress (the suggested examples are clickable).
3. Read the answer (grounded in your real data). The assistant changes nothing; each answer shows the cost in AI credits.

### 3.18 Ask the calendar for a summary (Claude widget)

1. In the **Calendar** tab, click **Claude**.
2. Ask a question or pick a suggestion ("Which events are overdue?", "Summarize my workload for this period").
3. **New** clears the conversation. This widget summarizes the **visible** events; it debits AI credits.

### 3.19 Share the tracking view (public link)

1. Click **Share**.
2. If you are an administrator: **Generate a public link** (or revoke an existing link).
3. **Copy** the link and share it. The recipient sees the schedule read-only, without logging in. The `?view=` parameter (calendar / gantt / kanban) chooses the displayed view.

---

## 4. Reference

### 4.1 Tabs and shortcuts

| Tab | Key | View shortcut (Calendar) |
|---|---|---|
| Kanban | `kanban` | — |
| Gantt | `gantt` | — |
| Calendar | `calendrier` | `Shift+M` Month · `Shift+S` Week · `Shift+J` Day · `Shift+A` Agenda · `Shift+T` Today · `/` Search |
| AI Assistant | `assistant` | — |

### 4.2 Kanban boards and columns

| Board | Columns | Status change by dragging |
|---|---|---|
| Sales | Prospecting · Qualification · Proposal · Negotiation (+ Won/Lost in a callout) | Yes |
| Quotes | Draft · Validated · Sent · Pending · Accepted · Refused · Completed | Yes, **except** to Accepted (blocked → Quotes) |
| Projects | Pending · In progress · Suspended · Completed | Yes |
| Purchases | Draft · Sent · Confirmed · In progress · Received · Invoiced · Cancelled | Yes, **except** to Received/Invoiced (blocked → Store) |
| Work Orders | Draft · In progress · Paused · Completed | Yes (state machine), **except** to Cancelled (blocked → WO module) |
| Operations | Pending · In progress · Completed · Cancelled | Yes |
| Invoices | Draft · Sent · Partially paid · Paid · Overdue | **No** (read-only) |

### 4.3 Kanban drag-and-drop blocking rules

The server (`PUT /production/kanban/update-status`) protects accounting and inventory effects:

| Case | Result | Reason |
|---|---|---|
| Any **Invoice** card | `400` | Status driven by Accounting (payments, journal entries) |
| **Purchase** to `Received` or `Invoiced` | `400` | Receiving/invoicing = Store module (stock movement) |
| Un-validate a **purchase** already `Received`/`Invoiced` | `400` | Stock consistency |
| **Quote** to `Accepted` | `400` | Goes through Quotes (creates the linked project + notifies) |
| **Work order** to `Cancelled` | `400` | Cancellation = WO module (stock restoration) |
| **Work order** to `Completed` with unfinished operations | `400` | All operations must be completed or cancelled (except `super_admin`) |

> **Reordering** (`reorder`), on the other hand, allows **all** entities, including Invoices and Operations, because it has no business effect.

### 4.4 Work-order (WO) state machine

Allowed transitions (the `super_admin` bypasses them). The values below are the stored status codes:

| From | To |
|---|---|
| BROUILLON | BROUILLON, EN_COURS, ANNULE |
| EN_COURS | EN_COURS, EN_PAUSE, TERMINE, ANNULE |
| EN_PAUSE | EN_PAUSE, EN_COURS, ANNULE |
| TERMINE | TERMINE (terminal) |
| ANNULE | ANNULE (terminal) |

The Gantt status drop-down offers only the valid transitions from the current status (inherited accented/spaced variants are normalized automatically).

### 4.5 Gantt sources and endpoints

| Source | Data endpoint | Notes |
|---|---|---|
| Sales | `crmApi.listOpportunities` | CRM opportunities |
| Quotes | `GET /production/gantt/devis` | Excludes Cancelled and Refused |
| Projects | `GET /projects/gantt` | Non-cancelled projects (**LIMIT 500**) + phases as subtasks; progress = average of the phases |
| Purchases | `GET /production/gantt/bons-commande` | Start = order date, end = expected delivery date |
| Work Orders | `GET /production/gantt/bons-travail` | Work order + operations as subtasks; supplier aggregated from subcontractors |
| Operations | `GET /production/gantt/operations` | One row per operation (orphan operations, with no work order, are hidden) |

> A `GET /production/gantt/projects` endpoint exists server-side but is **not** used by the interface; the "Projects" view goes through `GET /projects/gantt` (the one that carries the phases).

### 4.6 Dependency types

| Code | Label |
|---|---|
| `finish_to_start` | Finish → Start (FS) |
| `start_to_start` | Start → Start (SS) |
| `finish_to_finish` | Finish → Finish (FF) |
| `start_to_finish` | Start → Finish (SF) |

Lag (`lag_days`): an integer bounded **-3650 to +3650**. Entity types that can be linked: project, work order, quote, purchase order, operation, opportunity. Cycle detection is **fail-closed** (refuses when in doubt).

### 4.7 Calendar — event types and views

**View modes**: Month, Week, Day, Agenda.

**10 filterable types**: Opportunity, Quote, Project (+ "Project start" sub-type), Purchase order, Work order, Invoice, Interaction, CRM Activity, Note, Operation. Events come from the aggregator `GET /production/calendar-events` (projects, work orders, operations, quotes, purchase orders, invoices, interactions, activities), from the **notes** (`GET /production/calendar-notes`) and from the CRM **opportunities**.

### 4.8 Endpoints (full reference)

All prefixed by `/api/erp/v1`.

| Domain | Method + path | Access |
|---|---|---|
| Kanban | `GET /production/kanban` | Read |
| Kanban | `GET /production/kanban/achats` | Read |
| Kanban | `PUT /production/kanban/update-status` | Write (roles) |
| Kanban | `PUT /production/kanban/reorder` | Read (cosmetic) |
| Gantt data | `GET /production/gantt/{bons-travail,devis,bons-commande,operations}` | Read |
| Gantt data | `GET /projects/gantt` | Read |
| Gantt dependencies | `GET/POST/PUT/DELETE /production/gantt/dependencies` | Read (GET) / Write (roles) |
| Gantt baseline | `GET/POST/DELETE /production/gantt/baselines` | Read (GET) / Write (roles) |
| Gantt export | `GET /production/gantt/export-csv` | Read |
| Calendar | `GET /production/calendar-events?year&month` | Read |
| Calendar notes | `GET/POST/PUT/DELETE /production/calendar-notes` | Read; write by the author (or admin if shared) |
| Sharing | `POST/DELETE /production/calendar/share` | Admin only |
| Sharing | `GET /production/calendar/share` | Read (token masked from non-admins) |
| Public sharing | `GET /production/calendar/public/{token}?view=` | **No authentication** |
| Operation edit | `PUT /production/work-orders/{bt_id}/operations/{op_id}` | Write (roles) |
| WO assignment | `POST /production/work-orders/{bt_id}/assignations` | Write (roles) |
| Purchase assignment | `POST /production/achats/{achat_id}/assignations` | Write (roles) |
| WO dates | `PUT /production/work-orders/{bt_id}` | Write (roles) |
| Project phases | `POST/PUT /projects/{project_id}/phases` | Any member |
| Project assignments | `GET/POST/DELETE /projects/{project_id}/assignments` | Any member |
| Project dates | `PUT /projects/{project_id}` | Any member |
| AI Assistant | `POST /suivi/ai/chat` | Read (debits AI credits) |

### 4.9 Permissions and guards

- **Read** (`get_current_user`, any member): all data views, CSV export and Kanban **reordering**.
- **Write** (`require_tenant_admin_or_role` with `BT_WRITE_ROLES = admin, super_admin, gestionnaire, contremaitre`): Kanban status change, dependencies, baseline, operation editing, WO/purchase assignments.
- **Admin only**: generate/revoke public sharing (and the GET masks the token from non-admins).
- **No role guard but ownership-based**: calendar notes (author, or admin if shared; only the author changes the scope), project phases and assignments.
- **Public**: the sharing page, without authentication.

### 4.10 Limits, caps and rates

| Item | Limit |
|---|---|
| Kanban — cards per entity | 50 |
| Kanban — reorder (`ordered_ids`) | 10,000 max |
| Gantt — Projects | 500 rows |
| Baseline — items | 5,000 max |
| Dependencies — lag | -3650 to +3650 days |
| Dependencies — cycle detection | 1000 iterations (fail-closed) |
| Calendar note — length | 500 characters |
| Sharing — link | permanent (until revoked), 1 per tenant |
| AI Assistant — message | 1 to 8000 characters |
| AI Assistant — history | 40 turns sent (re-truncated to 12) |
| AI Assistant — response tokens | 8000 max; tool loop: 5 iterations |
| AI Assistant — rate | 20 requests/min per IP address |
| Public page — rate | 60 requests/min per IP address |
| Exchanged date format | `YYYY-MM-DD` |

### 4.11 Calculations

| Item | Rule |
|---|---|
| **Automatic progress** (Gantt, %) | 0 before the start date, 100 from the end date onward, otherwise `round(elapsed time / total duration × 100)` |
| **Operation progress** (WO) | `min(actual hours / planned hours × 100, 100)` |
| **Project progress** (Gantt) | average of the progress of its phases |
| **Due badge** | past due → "Overdue" (red); same day → "Today" (blue); within 3 days or less → "{n}d left" (yellow); beyond that → no badge |
| **Total budget** (Kanban panel) | sum of the amounts of the displayed cards |

### 4.12 Quebec public holidays (calendar)

The calendar highlights Quebec's public holidays from **2024 to 2030** (New Year's Day, Good Friday, Easter Monday, National Patriots' Day, Quebec National Holiday (Saint-Jean-Baptiste), Canada Day, Labour Day, Thanksgiving, Christmas, Boxing Day) with a red flag.

### 4.13 Effect on AI credits

The Tracking AI Assistant (`POST /suivi/ai/chat`) and the calendar's "Ask Claude" widget **debit your company's prepaid AI credits**. The billed cost corresponds to the real Anthropic token cost **marked up by 30%**. When the balance is depleted: `402` ("AI credits depleted"). No other paid integration (Stripe, QuickBooks, etc.) is involved in this module.

---

## 5. Integrations and FAQ

### 5.1 Integrations with the other modules

Tracking is a **cross-cutting view**: it reads and lightly edits entities from other modules, with no business tables of its own.

| Module | Role in Tracking | Manual |
|---|---|---|
| **CRM / Opportunities** | "Sales" board and Gantt source; reordering and status change of opportunities | [05-gestion-crm-opportunites.md](./05-gestion-crm-opportunites.md) |
| **Quotes** | "Quotes" board and Gantt source; Gantt dates | [07-ventes-soumissions.md](./07-ventes-soumissions.md) |
| **Projects** | "Projects" board and Gantt source (with phases); assignments and dates | [08-ventes-projets.md](./08-ventes-projets.md) |
| **Work Orders** | "Work Orders" board and source; operations as subtasks; assignments; state machine | [11-operations-bons-de-travail.md](./11-operations-bons-de-travail.md) |
| **Purchase orders (Purchases)** | "Purchases" board and source; delivery dates; assignments | [13-operations-bons-de-commande.md](./13-operations-bons-de-commande.md) |
| **Accounting (Invoices)** | "Invoices" board, read-only | [14-operations-comptabilite.md](./14-operations-comptabilite.md) |
| **Store (Inventory)** | Suppliers offered for operation assignment; purchase receiving | [09-operations-magasin.md](./09-operations-magasin.md) |
| **Employees** | Employee list for assignments | [10-operations-employes.md](./10-operations-employes.md) |

Double-click from any view: Sales → `/ventes`, Quotes → `/devis`, Projects → `/projets`, Work Orders and Operations → `/bons-travail`, Purchases → `/magasin`, Invoices → `/comptabilite`.

### 5.2 The two artificial-intelligence systems

Do not confuse them:

| | "AI Assistant" tab | "Ask Claude" widget (Calendar) |
|---|---|---|
| Location | Dedicated Tracking tab | "Claude" button on the Calendar tab |
| Nature | **Specialized** tracking chat | **General** assistant with calendar context |
| Data | Queries the Tracking tables (strict allowlist, denylist of sensitive data) | Summarizes the **visible** events (up to 50) |
| Writing | None (read-only) | None (summary/suggestion) |
| AI credits | Debited (30% markup) | Debited |

### 5.3 What does not exist / limits to be aware of

- The **Invoices board is read-only**: no status change by dragging, no assignment, no creation.
- **Gantt CSV export is not available** for the "Operations" source (button disabled).
- In **24h / 3 days** views, Gantt bars are **neither draggable nor resizable** and the dependency arrows are hidden.
- **Operations have no dedicated page**: a double-click always opens the **parent work order**.
- The **AI Assistant tab is read-only** — it neither creates nor modifies anything, with no action confirmation.
- **Notes** are edited mostly in **Month view** (the Week/Day views point back to Month view).
- **Reassigning operations in the calendar** is reserved for the `admin`, `super_admin`, `gestionnaire`, `contremaitre` roles.
- **Public sharing** exposes only the schedule (projects, work orders, shared notes): never any amounts, quotes, invoices, opportunities or personal notes.
- **Priority** is absent from Quotes and Purchase orders.

### 5.4 FAQ

**Q: Why can't I set an invoice to "Paid" by dragging its card?**
A: By design. Invoice status is driven by Accounting (payments, journal entries). The Tracking Invoices board is read-only.

**Q: I tried to move a purchase to "Received" in the Kanban, and it failed.**
A: That's normal. Receiving a purchase triggers a stock movement; it is done in the Store module, not here.

**Q: Why won't my quote move to "Accepted" from the Kanban?**
A: Accepting a quote creates the linked project and triggers notifications; it goes through the Quotes module.

**Q: I can't drag a work order to "Completed".**
A: A work order can be completed only if **all** its operations are completed or cancelled. Finish the remaining operations first (a `super_admin` can bypass this rule).

**Q: Is the vertical reordering of cards saved?**
A: Yes, the order is saved (`reorder`). It is a cosmetic gesture allowed to any member, including on the Invoices and Operations boards.

**Q: What is the difference between the "AI Assistant" tab and the calendar's "Claude" button?**
A: The AI Assistant tab is a specialized tracking chat (real data, read-only). The calendar's "Claude" button is the ERP's general assistant, fed by the calendar's visible events, for a quick summary. Both debit AI credits.

**Q: Why doesn't the "Projects" Gantt show all my projects?**
A: The query limits to 500 projects and excludes cancelled projects.

**Q: Why don't some Gantt bars move?**
A: In 24h or 3 days view (the hourly "site view"), the bars are frozen: click for the tooltip, but use a Week/2 Wks/Month zoom to move the dates.

**Q: How do I link two tasks?**
A: Enable Dependencies, then drag from the right edge of a bar onto another. The system refuses any link that would create a loop.

**Q: What is the baseline for?**
A: To freeze a reference plan. The grey bar then shows the gap between the initial plan and the current dates. Clearing applies to all views.

**Q: Who can create the public sharing link?**
A: An administrator (or super-admin). Other members can copy an already-generated link, but cannot create or revoke it. The link shows only the schedule, with no amounts.

**Q: Can one of my colleagues see a calendar note?**
A: Only if you make it **shared**. A personal note is visible to you only. Only the author can change this scope; an administrator can manage shared notes.

**Q: Double-clicking an operation doesn't open an operation page.**
A: Operations have no dedicated page; the double-click opens their parent work order.

**Q: Does the Tracking module have its own data?**
A: No. It composes entities from the other modules. It only creates technical tables (dependencies, baseline, notes, assignments, sharing tokens).

---

## 6. Summary

| Item | Detail |
|---|---|
| **Mission** | Cross-cutting control center: Kanban, Gantt, Calendar and AI Assistant over Sales, Quotes, Projects, Work Orders, Operations, Purchases and Invoices. |
| **Route / menu** | `/suivi` — "Tracking" sidebar entry. |
| **Source code** | `SuiviPage.tsx` (8,267 lines), `SuiviAssistantTab.tsx`, `QuickCreateModal.tsx` (1,301 lines), `SuiviShareButton.tsx`; backends `production.py` (core), `projects.py` (Projects Gantt), `suivi_ai.py` (Assistant). |
| **4 tabs** | Kanban · Gantt · Calendar · AI Assistant. |
| **Kanban** | 7 boards (Sales, Quotes, Projects, Purchases, Work Orders, Operations, Invoices); dragging = change status (with business blocks); vertical dragging = reorder; employee assignment. Invoices = read-only. |
| **Gantt** | 6 sources; 5 zoom levels (24h, 3 days, Week, 2 Wks, Month); draggable/resizable bars (except in hourly view); FS/SS/FF/SF dependencies; critical path; baseline; CSV export (except Operations); print. |
| **Calendar** | 4 views (Month, Week, Day, Agenda); QC public holidays; personal/shared notes; drag-to-reschedule; operation reassignment (roles); "Claude" widget. |
| **AI Assistant** | Read-only tracking chat; table allowlist, no writes; debits AI credits (30% markup). |
| **Create** | Shared modal, 6 types (Project, Opportunity, Quote, Work order, Purchase order, Operation) + batch operations for work orders. |
| **Sharing** | 1 read-only public link per tenant (`?view=` calendar/gantt/kanban); admin generation/revocation; exposes only the schedule. |
| **Permissions** | Read = any member; write = admin/super_admin/gestionnaire/contremaitre; sharing = admin; notes = author/admin; public page = no authentication. |
| **Deliberate blocks** | Invoice not editable; purchase not "Received/Invoiced"; quote not "Accepted"; work order not "Cancelled"; work order "Completed" only if operations are complete. |
| **Not in this module** | No dedicated operation page; no CSV export for Operations; no bar editing in hourly view; no writing by the AI Assistant; no business tables of its own. |

---

*Constructo ERP Manual — Module 02 Tracking (Kanban, Gantt, Calendar) — v3.0 verified against the source code — 2026-07-07*

**Verified sources**: `frontend/src/pages/SuiviPage.tsx`, `frontend/src/pages/suivi/SuiviAssistantTab.tsx`, `frontend/src/components/suivi/QuickCreateModal.tsx`, `frontend/src/components/suivi/SuiviShareButton.tsx`, `frontend/src/api/production.ts`, `frontend/src/api/suiviAi.ts`, `frontend/src/i18n/locales/fr/crm.json` (block `crm.suivi.*`, lines 367-1038); backends `backend/routers/production.py`, `backend/routers/projects.py`, `backend/routers/suivi_ai.py`.

**Related manuals**:
- Module 05 — CRM / Opportunities: [05-gestion-crm-opportunites.md](./05-gestion-crm-opportunites.md)
- Module 07 — Quotes: [07-ventes-soumissions.md](./07-ventes-soumissions.md)
- Module 08 — Projects: [08-ventes-projets.md](./08-ventes-projets.md)
- Module 11 — Work Orders: [11-operations-bons-de-travail.md](./11-operations-bons-de-travail.md)
- Module 13 — Purchase Orders: [13-operations-bons-de-commande.md](./13-operations-bons-de-commande.md)
- Module 14 — Accounting (Invoices): [14-operations-comptabilite.md](./14-operations-comptabilite.md)
- Module 24 — AI Assistant: [24-communication-assistant-ia.md](./24-communication-assistant-ia.md)
