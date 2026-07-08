# Module 14 — Accounting (general ledger, invoices, payroll, contingency fund)

> **Version**: 3.0 (complete overhaul verified against the actual source code)
> **Reference code**:
> - Frontend: `frontend/src/pages/ComptabilitePage.tsx` (≈ 6,437 lines — a single page with **18 fixed tabs + 1 conditional tab**), sub-components `components/comptabilite/ComptesPayablesTab.tsx` (≈ 399 lines), `DecomptesTab.tsx` (≈ 441 lines), `WipTab.tsx` (≈ 293 lines), `RetenuesTab.tsx` (≈ 302 lines), `ComptaAssistantTab.tsx` (≈ 372 lines); API client `frontend/src/api/accounting.ts` (≈ 1,527 lines)
> - Backend: `backend/routers/accounting.py` (≈ 15,455 lines — **93 endpoints**, of which **16 are reserved for the "accountant" role**), `backend/routers/accounting_ai.py` (≈ 895 lines — accounting assistant), `backend/routers/payroll.py` (≈ 1,789 lines — **Payroll module, separate**), `feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (tax slips — **Payroll module, separate**), `feuillets_common.py` (SIN/employer foundation), `fonds_prevoyance.py` (≈ 3,213 lines — **Real Estate / Land area, separate**)
> - API prefix: `/api/erp/v1` — accounting responds under `/accounting`, the assistant under `/accounting/ai`
> - Translation namespace (i18n): `compta` (`frontend/src/i18n/locales/{fr,en}/compta.json`)
> **PostgreSQL tables (per tenant)**: `factures` (header of customer and supplier invoices, progress claims and credit notes), `facture_lignes` (invoice lines), `journal_entries` / `journal_lines` (double-entry general ledger), `plan_comptable` (accounts), `cost_centers` (cost centers), `periodes_comptables` (open/closed periods), `retenues_chantier` (holdbacks), `cedule_postes` / `decompte_lignes` (CCDC progress claims), `immobilisations` / `amortissement_ecritures`, `factures_recurrentes`, `accounting_audit_log` (seven-year audit log). Tables of related modules: `payroll_*` and `feuillets_fiscaux` (Payroll module), `fp_*` (contingency fund, Real Estate area).
> **Scope**: this module is the **general accounting and invoicing** of a construction company. A single tabbed page brings together the **invoicing** of both customers and suppliers, the **general ledger** (chart of accounts and double-entry journal entries), the **financial statements**, **construction billing** (CCDC progress claims, work in progress, holdbacks 1150 / 2150), the **GST/QST** sales taxes, **fixed assets**, **accounting periods**, and an **AI accounting assistant**. The module runs **multi-currency and multi-jurisdiction** (Quebec–Canada by default; United States conditionally). **Important vocabulary point**: despite the title, **payroll**, the **tax slips** (T4, RL-1, PD7A) and the **contingency fund** (Law 16) are **NOT in this module**. They live on other pages of the system (see §1.2). This manual covers them in the integrations chapter (§5) because they **feed** the accounting, but it tells you clearly **where** to use them.

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

The Accounting module is the company's **single general ledger**. It lets an authorized person:

- **invoice customers**: create a numbered sales invoice (`FACT-2026-00031`), enter lines, apply GST and QST, send it by email (with a 90-day **public link**), record partial or full **payments**, produce a **credit note** compliant with the Quebec Sales Tax Act, set up **recurring invoices**, and chase late payers with four-level **reminders**;
- **enter supplier expenses**: enter a purchase invoice (by hand or by **AI scan**), split it across several expense accounts, track **accounts payable** (aging of balances) and record disbursements;
- **keep the general ledger in double entry**: manage the **chart of accounts** (35 accounts preloaded for Quebec), post balanced **journal entries**, **validate** them, review the **general ledger** account by account, and the **transactions**;
- **produce the financial statements**: balance sheet, income statement, cash flow, and the **sales-tax return** (GST/QST or US sales tax), all exportable;
- **bill construction** the industry way: **progress claims** (CCDC method with a schedule of values), tracking of **work in progress** (WIP, cost-to-cost method) and **holdbacks** (account 1150 for customers, 2150 for subcontractors);
- **manage fixed assets** and their depreciation (straight-line or declining-balance);
- **close accounting periods** and block any backdated entry in a closed period;
- **export** to QuickBooks (IIF format), Sage 50 (CSV), and to CSV/Excel/PDF;
- query an **AI accounting assistant** that **proposes** entries or invoices that you **confirm** before they exist.

### 1.2 Where "payroll" and "contingency fund" actually live

The title of this module mentions payroll and the contingency fund because they are part of the company's accounting universe. **But their interface is not in `/comptabilite`.** Here is the exact map, to avoid any pointless search:

| Title theme | Where to use it in the system | Dedicated manual |
|----------------|-------------------------------|--------------|
| **General ledger** and **invoices** | **Accounting** menu (`/comptabilite`) — **this is this module** | This manual (§2 to §4) |
| **Payroll** (RRQ, RRQ2, RQAP, EI, FSS, CNESST, CCQ, pay periods, PDF pay stubs) | **Time Tracking** menu (`/pointage`), the **"Time Tracking & Payroll"** page, **Payroll** and **CCQ Payroll** tabs | Module 12 — Time Tracking and Hours |
| **Tax slips** (T4, RL-1, PD7A remittance) | **Time Tracking** menu (`/pointage`), **Slips** tab | Module 12 — Time Tracking and Hours |
| **Contingency fund** (Law 16, studies, 25-year projections) | **Real Estate / Land** area (`/immo` storefront, developer back-office) | Module 34 — Real Estate |

Accounting and these modules **communicate**: a payroll automatically generates a draft journal entry in accounting (§5.1), and the slips are fed by the payroll's annual totals (§5.2). The contingency fund, for its part, is a condominium-management tool **entirely independent** of the contractor's accounting; there is **no contingency-fund screen in `/comptabilite`** (§5.3).

### 1.3 What the module does NOT do

- **No payroll or pay stubs here.** The complete Quebec payroll is in the **Time Tracking** module (`/pointage`). No Accounting tab computes a salary.
- **No T4 / RL-1 / PD7A slips here.** They are also in **Time Tracking**.
- **No contingency fund here.** It is a tool of the Real Estate area.
- **You do not edit an invoice's lines in the detail panel.** The detail shows the lines **read-only**; any change goes through the **Edit invoice** window.
- **You do not set an invoice to "Paid" by hand.** An invoice's status is changed inline, **except** to "Paid" or "Partially paid": these two are obtained only by **recording a payment**.
- **You do not delete just any invoice.** Deletion is restricted to invoices in **Draft** or **Cancelled**. A posted invoice is neutralized by **cancellation** (automatic reversal), never by deleting the entry.
- **You never erase a validated journal entry.** A **Validated** entry is frozen forever (accounting standard); you correct it with a **reversal** (a reversing entry), never by an erasure.
- **Recurrence has no "New" button.** A recurring-invoice template is created **from an existing invoice** (the "Recurring" button in the detail panel), not from the Recurring tab.
- **The estimated cost of work in progress is not always reliable.** The WIP tab shows "to verify" or "unavailable" when the source of the estimated cost (the accepted quote) has not been propagated to the project.
- **Some slips are not transmittable as-is.** The RL-1 is **preliminary and unofficial** (Revenu Québec certification not obtained); the T4 and PD7A are **aids** whose compliance with the official templates remains to be finalized (see §5.2).

### 1.4 Access from the side menu

- **Side menu** → **Accounting** (calculator icon `Calculator`).
- **Address**: `/comptabilite`.
- **Default open tab**: **Global View** (the financial dashboard).
- **Opening an invoice directly**: a link like `/comptabilite?open=<id>` (for example from a case file) automatically switches to the **Invoices** tab and opens the requested record.
- Protected page: you must be authenticated in a tenant.

### 1.5 Permissions and roles

Two levels of protection intersect in this module.

**a) Display guards (in the screen).** Two tabs appear only under certain conditions:

| Tab | Display condition |
|--------|-----------------------|
| **Terms** | Reserved for **administrators** (`role == 'admin'` or `is_admin`) |
| **Tax USA** | Shown only for companies whose country is the **United States** (`tenantCountry == 'US'`) |

**b) Write guards (server side).** **Consulting** the accounting is open to any authenticated user of the tenant. **Sensitive operations** require the **accountant** role (the `require_tenant_admin_or_role('comptable')` guard, which also automatically admits any administrator or super-administrator, even if their nominal role is "user"). Sixteen endpoints are protected this way:

- creating, editing and deactivating an **account** in the chart of accounts;
- creating, adding lines to and **validating** a **journal entry**;
- **deleting**, **paying** and issuing a **credit note** for an invoice;
- creating and **releasing** a **holdback**;
- generating **depreciation**;
- reading **work in progress** (WIP);
- **closing** and **reopening** an accounting period.

> **Good to know.** **Creating** an invoice, adding lines, and the posting that occurs when an invoice **leaves draft** are open to any authenticated user — this is not reserved for the accountant. In other words, an ordinary user can invoice and trigger the sales entry in the general ledger; the more sensitive actions (delete, collect, credit, close) remain reserved for the accountant role or an administrator.

**c) View-only mode (read-only).** If the tenant's subscription is missing or cancelled, the system switches to **view-only mode**: reading remains possible everywhere, but **every write is refused** (403 error). This is a global protection, independent of the module.

### 1.6 Summary cards and the "Synchronize" button

As soon as a summary is loaded, **four figure cards** top the page (they concern invoicing as a whole):

| Card | What it shows |
|-------|-------------------|
| **Total revenue** | Revenue billed to customers (net of credit notes) |
| **Collected** | Total customer payments received |
| **Balance due** | Accounts receivable still to collect |
| **Overdue invoices** | Number and amount of invoices past their due date |

At the top right, the **"Synchronize"** button launches `POST /accounting/sync-all`: it **retroactively generates** the missing general-ledger entries from invoices, payments, received purchase orders and employee hours, then reloads the current tab and the summary. The operation is **idempotent** (it never creates a duplicate).

### 1.7 The 18 tabs (plus one for the United States)

The tabs are grouped by theme, with a visual separator between each group.

| Tab | Short label | Group | Purpose |
|--------|---------------|--------|----------------|
| **Global View** | Global | Overview | Dashboard: revenue, expenses, profit, monthly breakdown |
| **Invoices** | Invoices | Invoicing | Customer and supplier invoices (the richest tab) |
| **Accounts payable** | Payable | Invoicing | What the company owes its suppliers, by due date |
| **Terms** *(admin)* | Terms | Invoicing | Default invoicing terms and banking details |
| **Recurring** | Recur. | Invoicing | Automatically repeated invoice templates |
| **Progress claims** | Claims | Construction | CCDC progress claims (schedule of values) |
| **Work in progress** | WIP | Construction | Progress and billing variances per project |
| **Holdbacks** | Hold. | Construction | Customer and subcontractor holdbacks |
| **Chart of Accounts** | Chart | General ledger | List of accounts |
| **Journal** | Journal | General ledger | Double-entry entries |
| **Transactions** | Trans. | General ledger | Aggregated view of revenue / expenses / credit notes |
| **General Ledger** | Ledger | General ledger | Movements of one account with a running balance |
| **Financial Statements** | Statements | Reports | Balance Sheet, Income Statement, Cash Flow, Taxes |
| **Cost Centers** | Costs | Reports | Budgets and expenses per center |
| **Periods** | Periods | Management | Opening and closing of accounting periods |
| **Fixed Assets** | Assets | Management | Assets and depreciation |
| **AI Assistant** | AI | Assistant | Accounting assistant (propose → confirm) |
| **Tax USA** *(U.S.)* | Tax US | Tax | 1099-NEC slips and W-9 forms |

### 1.8 Key concepts

- **Double entry.** Every operation produces a **journal entry** whose total **debits** equal its total **credits** (a one-cent tolerance). An unbalanced entry is refused.
- **Pre-tax and tax-inclusive.** Amounts labelled "pre-tax" (HT) exclude tax; "tax-inclusive" (TTC) includes GST and QST. Taxes are computed **at the invoice level**, not line by line.
- **Tax snapshot.** Each invoice **freezes** its tax labels and rates at creation (for example GST 5%, QST 9.975% in Quebec). Changing the company configuration later does not change invoices already issued.
- **"Draft" status then posting.** An invoice in **Draft** does not yet have a general-ledger entry. As soon as it **leaves draft** (Sent, etc.), the system automatically generates the sales (or purchase) entry.
- **Reversal.** A validated entry is never erased: to cancel its effect, the system creates a **reversing** entry (debits and credits swapped). This is what happens when you cancel an invoice, credit an invoice, or return a posted invoice to draft.
- **Accounting period.** A date range that can be **closed**. Once closed, **no entry** can be posted with a date within the period.
- **Document numbers.** They are assigned in a **concurrency-safe** way (never by counting): invoices `FACT-2026-00031`, credit notes `AV-2026-00007`, journal entries `JE-VTE-00042`, holdbacks `JE-RET`, etc.

---

## 2. Interface

This section describes each tab and each window of the `/comptabilite` module. The Payroll, Slips and Contingency-fund integrations (which are not in this screen) are covered in §5.

### 2.1 "Global View" tab (financial dashboard)

This is the tab open by default. It presents **three cards** — **Revenue**, **Total expenses**, **Net profit** — then a **"Monthly breakdown"** card: a Month / Revenue / Expenses / Profit table with a **total row**. If no data exists yet, a prompt message is shown. Source: `GET /accounting/dashboard`.

### 2.2 "Invoices" tab

The most complete tab. **Master-detail** view: the list of invoices on the left, the detail on the right.

#### 2.2.1 Action bar

- **New invoice**: opens the window to create a **customer** (revenue) invoice.
- **New expense**: opens the same window, preset as a **supplier** (expense) invoice.
- **Scan invoice with AI** (purple button, camera icon): opens a file picker (image or PDF); the AI reads the supplier invoice and pre-fills the fields (see §2.19 and §3.11).
- **Status filter** (drop-down menu): All, Draft, Sent, Partially paid, Paid, Overdue, Cancelled.
- **Search**: by number or by customer/supplier name.

#### 2.2.2 Invoice list

On a computer, a four-column table: **Number**, **Customer**, **Status**, **Total (incl. tax)**. The **status** is a colored badge **editable inline** through a drop-down — **except** the "Paid" and "Partially paid" statuses, which are obtained only by a payment. On a phone, each invoice becomes a card (number, badge, customer, date, total). An empty list shows a prompt message.

#### 2.2.3 Detail panel

When you select an invoice, the right-hand panel (full screen on a phone) shows:

- **Header**: number, customer name, **pencil** button (Edit — hidden if the invoice is Paid or Cancelled), close button.
- **Status badge**, dates (invoice / due), **linked project**, **notes** and **internal notes**.
- **Amounts block**, with **dynamic tax labels** (taken from the invoice; default GST 5% and QST 9.975%): Subtotal (pre-tax), tax 1, tax 2, **Total (incl. tax)**, Paid, **Balance due**.
- **Action bar**, whose buttons appear depending on the status:

| Button | When it appears | Effect |
|--------|-------------------|-------|
| **Send** | Draft | Confirmation window, then posting and change to Sent |
| **Email** | Except Paid / Cancelled | Send the invoice by email (PDF attached, 90-day public link) |
| **Payment** | Except Paid / Cancelled / Credit note | Record a payment |
| **Credit note** | Except Draft / Cancelled / Credit note | Issue a credit note |
| **Recurring** | Except Cancelled / Credit note | Create a recurring-invoice template |
| **Reminder** | Sent / Partially paid / Overdue | Send a reminder |
| **Reminder history** | If at least one reminder has been sent | List of reminders |
| **HTML preview** | Always | Preview of the laid-out invoice |
| **PDF** | Always | Download the PDF (WeasyPrint engine) |
| **Print** | Always | Open the HTML for printing |
| **Excel** | Always | Download the `.xlsx` file |
| **QuickBooks CSV** | Always | Download a QuickBooks-compatible CSV (and "Copy CSV" to the clipboard) |
| **Delete** | Draft or Cancelled | Delete the invoice (automatic reversal if an entry is linked) |

- **Lines section** (**read-only**): description, quantity × unit price, amount. To change a line, use the **Edit invoice** window.

### 2.3 "Accounts payable" tab

A standalone view of what the company **owes** its suppliers. Source: `GET /accounting/payables/summary`.

- **Three indicators**: **Total payable**, **Overdue**, **Suppliers** (count).
- **"Aging of balances (by due date)"** card: five bands — **Current**, **1-30 days**, **31-60 days**, **61-90 days**, **90+ days**.
- **Per-supplier table**: Supplier, Invoices, Total due, Overdue, Oldest due date. Each row is **clickable**: it opens the **detail** of the supplier's unpaid invoices (number, date, due date, balance due, status) with a **Pay** button.
- **"Record a disbursement" window**: amount, date, method (Transfer, Cheque, Card, Cash, Other). It records the payment to the supplier (debit to account 2100 Accounts payable, credit to account 1010 Cash).

### 2.4 "Terms" tab (administrators only)

Two settings cards that feed the **bottom of invoices** and their PDF. Source: `GET /accounting/factures/defaults`.

- **"Invoicing terms"**: a text area (one term per line), with the **Insert template** (the country's default template), **Reset** and **Save** buttons.
- **"Banking details (direct deposit)"**: Financial institution, Institution number (3 digits), Transit (5 digits), Account number (5 to 20 digits), Beneficiary, and a **"Show on invoices" checkbox** (disabled until a field is filled). A **live preview** shows the Payment section as it will appear. The same validations are applied server side.

### 2.5 "Recurring" tab

List of **recurring-invoice templates**. A **status filter** (All, Active, Paused, Finished, Cancelled). The table shows: Name, Customer, **Frequency** (with its multiplier), Next generation, Generated (of maximum), Status, Automatic email sending, Actions. Per-row actions: **Pause**, **Reactivate**, **Generate now**, **Cancel** the template.

> **How do you create a template?** There is **no** "New" button here. You create a template from an **existing invoice**, via the **Recurring** button in its detail panel (see §2.2.3 and §3.7).

### 2.6 "Progress claims" tab (CCDC progress billing)

Billing by progress, in the style of the construction industry's progress claims (CCDC usage). You first choose a **project**, then:

- **"Schedule of values" card**: the breakdown of the contract into **value items**. If the schedule is empty, a **Quote number** field and an **Import from quote** button let you build it automatically from an accepted quote. The table shows, per item: the label (and the percentage already billed), the **Contract value**, the **Already billed**, an **% complete to date** field (editable) and **This claim** (the computed amount). A **total** row and a reminder of the holdback rate complete the card. The **Create progress claim** button generates the claim invoice (disabled while the total is zero).
- **"Issued progress claims" card**: the claims already produced (Number `#n`, Date, Total incl. tax, Holdback, **Net payable**, Status).
- **"Value item" window** (add or edit): **Description** and **Contract value**.

> **Interface limit.** The value-item window only exposes the description and the value. The technical fields `code_poste`, `categorie` and `sequence_poste` exist in the API but are **not** offered on screen.

### 2.7 "Work in progress" tab (WIP)

Tracking of the financial progress of job sites using the **cost-to-cost** method, read-only. Source: `GET /accounting/wip`. You choose an "As of" date. Five indicators top the tab: **Contract value**, **Recognized revenue**, **Billed**, **Overbilling**, **Underbilling**. Then a per-project table:

- **Progress** (percentage and bar; red if over budget; a warning triangle flags an unreliable or unavailable estimated cost);
- **Costs incurred**, **Contract value**, **Recognized revenue**, **Billed**;
- **Variance**: blue = **overbilling** (billed beyond progress, a liability); amber = **underbilling** (billed below progress, an asset).

Each row **expands** to show the sources: labour cost (with the note "actual payroll" or "estimate"), materials cost, estimated cost (a "To verify" badge if reliability is doubtful), and the source of the contract value.

> **Why "to verify"?** The estimated cost comes from the **accepted quote**. When that cost has not been propagated to the project, the tab flags it honestly rather than showing a false progress.

### 2.8 "Holdbacks" tab

Management of **holdbacks**, for **customers** (account 1150, "Holdbacks receivable") as well as for **subcontractors** (account 2150, "Subcontractor holdbacks payable"). The type is **derived from** the linked invoice, not chosen by hand.

- **Toolbar**: **All / Customers / Subcontractors** filter and a **New holdback** button.
- **Table**: Type (Customer or Subcontractor badge), Invoice, Customer, Rate, Amount held, Work completion, Release, Status (**Held** in yellow, **Released** in green), and a **Release** button (with a warning: the release entry is **validated and irreversible**).
- **"New holdback" window**: choice of an **invoice** (issued, not draft and not cancelled), **Rate (%)** (empty = the tenant's default rate), **Work completion date**, **Notes**.

### 2.9 "Chart of Accounts" tab

The list of **accounts**. Toolbar: **Export CSV** and **New account**. A **"Show inactive accounts" checkbox**. The table shows: **Code** (indented by level), **Name**, **Type**, **Normal balance**, **Active**, and actions (**Edit**, **Deactivate / Reactivate**). On first display, the chart is **automatically preloaded** according to the tenant's country (35 accounts for Quebec; see §4.4).

### 2.10 "Journal" tab

The double-entry **journal entries**. Toolbar: **Export CSV**, **QuickBooks IIF**, **Sage 50 CSV**, **New entry**. The table shows: Entry number, Date, Description, **Type** (Sales, Purchases, Receipt, Disbursement, Payroll, Depreciation, etc.), Amount, Status (**Draft**, **Validated**, **Cancelled**), and a **Validate** button on draft entries.

> A **Validated** entry is **frozen**: you can no longer add a line to it. To correct it, you post an adjusting entry (reversal).

### 2.11 "Transactions" tab

An aggregated, readable view of the movements. Filter by type: **All / Revenue / Expense**. The table shows: **Type** (green badge for revenue, red for an expense, gray for a credit note), Reference, Date, Description, Amount (signed), Status. Credit notes appear as negative revenue (gray).

### 2.12 "General Ledger" tab

The detail of the movements **of one account**. You choose an **account** in a drop-down, then click **Load**. The table shows: Date, Entry number, Label, **Debit**, **Credit** and **Running balance**. Only **validated** entries are included.

### 2.13 "Financial Statements" tab

Four **sub-tabs**. All count only **validated** entries.

- **Balance Sheet**: an "As-of date" filter, a **Current fiscal year** button and **Apply**. Sections **Assets** (current and long-term), **Liabilities** (current and long-term), **Equity**, totals and a **Balanced / variance** indicator. The **net income for the year** is injected into equity.
- **Income Statement**: Revenue, Cost of sales, **Gross margin**, Operating expenses, **Net income**.
- **Cash Flow**: a Month / Inflows / Outflows / Net table. *(This sub-tab has no date filter of its own, unlike the Balance Sheet and Taxes.)*
- **Taxes**: a **From / To** filter, **Calculate** and **Export CSV** buttons. Three net cards — **tax 1** (GST), **tax 2** (QST) and **Total net due** ("To remit" or "Refund") — then a monthly Collected / Paid / Net table per tax. The calculation is **credit-note-aware** (the taxes of credit notes are subtracted).

### 2.14 "Cost Centers" tab

Toolbar: **New cost center**. The centers table shows: Code, Name, Type (Project, Department, Activity, Other), Annual budget. A **"Cost summary per center"** card compares Budget, Expenses and **Variance** (green or red). A project cost center carries the code `PRJ-00007`.

### 2.15 "Periods" tab

Toolbar: **New period**. The table shows: Name, Start, End, Status (**Open** / **Closed**), Closed by, and a **Close** button (with an irreversibility warning) or **Reopen** (reserved for accountants and administrators).

> **Actual effect of closing.** Unlike older versions of the system, closing **effectively blocks** any entry dated within the period: the server refuses to post anything there. It is a lock, not just a marker.

### 2.16 "Fixed Assets" tab

Four indicators: **Number of assets**, **Acquisition cost**, **Accumulated depreciation**, **Net value**. Toolbar: a **month** field, a **Generate depreciation** button and a **New asset** button. The table shows: Name, Category, Acquisition date, Cost, Accumulated depreciation, Net value, Method (and life in months).

### 2.17 "AI Assistant" tab

A conversational **accounting assistant**. The header shows "AI accounting assistant" and three example questions. The assistant works on the **propose → confirm** principle: it can **search** the database (read-only and on a strict allowlist of accounting tables — never payroll or HR data), then **propose** either a **journal entry** or an **invoice**. Each proposal is shown in a **card** (with its debit/credit lines or its description/quantity/price lines and its totals) and **Confirm** / **Cancel** buttons. **No entry is created without your confirmation.** Locks prevent a double submission or a double confirmation, and usage metadata (tokens, cost, time) is displayed. Source: `POST /accounting/ai/chat` and `POST /accounting/ai/confirm-action`.

### 2.18 "Tax USA" tab (US companies only)

Two sections, visible only if the tenant's country is the United States.

- **1099-NEC Contractors**: a fiscal-year selector; a table with checkboxes, Contractor, TIN (masked), Total paid (USD), presence of a **W-9** (Yes/No badge), and the **24% backup withholding**. The **Generate 1099-NEC** button opens a window (PDF format, IRIS CSV, or both; list and download links).
- **W-9 Contractors**: a **Request W-9** button (choose a supplier; sends a public link). The table shows: Name, Status (Pending, Sent, Received, Verified, Expired), TIN, Signature date, and the **Verify** and **PDF** actions. The recipient fills out a complete **public W-9 form**.

### 2.19 The windows (modals)

The following windows open from the tabs above:

| Window | Opened from | Main fields |
|---------|----------------|-------------------|
| **New invoice (customer / supplier)** | Invoices | **Customer (revenue) / Supplier (expense)** toggle; customer OR (supplier + supplier invoice number + expense account); issue and due dates; project; terms; **lines** (description, quantity, price, amount; for a supplier, a **per-line expense account** allows a multi-account split); live totals; notes and internal notes |
| **Edit invoice** | Invoices | Same sections + a **Status** section (Mark overdue, Cancel invoice) |
| **Scan a supplier invoice (AI)** | Invoices | Preview of the extracted data (supplier, number, dates, pre-tax, GST, QST, incl. tax, number of lines, confidence level) then **Create supplier invoice** |
| **Confirm sending** | Invoice detail | Preview of the **accounting entry** that will be generated (Sales or Purchases) |
| **Send by email** | Invoice detail | Recipient, CC, subject, message; mention of the **shareable public link (90 days)** |
| **Record a payment** | Invoice detail | Total incl. tax, already paid, balance due, amount, date, **method**, reference, **cash account** (default Cash 1010) |
| **Credit note** | Invoice detail | Original invoice, **reason**, total credit amount (incl. tax), date, internal notes (QSTA section 350 compliance) |
| **Recurring invoice** | Invoice detail | Name, **frequency** (weekly, semi-monthly, monthly, bi-monthly, quarterly, semi-annual, annual), multiplier, first-generation date, end date, maximum number of occurrences, initial status, automatic email sending |
| **Payment reminder** | Invoice detail | Four levels (Courteous D+3, Firm D+15, Insistent D+30, Formal demand D+60), recipient, message |
| **New journal entry** | Journal | Description, **type**, lines (account, label, debit, credit), **Balanced / variance** indicator |
| **New / Edit account** | Chart of accounts | **Code** (4 to 10 digits, immutable), name, type, class, normal balance, description, active |
| **New accounting period** | Periods | Fiscal year, period, start and end dates, name |
| **New cost center** | Cost centers | Code, name, type, annual budget |
| **New fixed asset** | Fixed assets | Name, category, acquisition date, cost, useful life (months), method (straight-line / declining-balance), residual value |

---

## 3. Step-by-step procedures

### 3.1 Create and send a customer invoice

1. **Invoices** tab → **New invoice** button.
2. Leave the toggle on **Customer (revenue)**. Choose the **customer**, the **issue date** (the due date fills in automatically), a **project** if needed, and the **terms**.
3. Add the **lines**: description, quantity, unit price. The subtotal, the taxes (GST/QST) and the total compute live.
4. If needed, enter **notes** (visible to the customer) and **internal notes**.
5. **Save**. The invoice appears as **Draft** (no general-ledger entry for now).
6. In the detail panel, click **Send**. A window shows the **sales entry** that will be generated: **debit 1100** (accounts receivable, incl. tax), **credit 4100** (revenue, pre-tax), **credit 2200** (GST), **credit 2210** (QST). Confirm: the invoice moves to **Sent** and the entry is posted.
7. To send it out: the **Email** button (the PDF is attached and a **90-day public link** is created), **PDF** (download) or **Print**.

### 3.2 Enter a supplier invoice (expense)

1. **Invoices** tab → **New expense** button (or the **Supplier** toggle in the creation window).
2. Choose the **supplier**, enter its **invoice number** and an **expense account** (for example 5100 Cost of materials).
3. Add the **lines**. To split the expense across **several accounts**, set a **per-line expense account**: posting will create one general-ledger line per account.
4. **Save**, then **Send**. The purchase entry generated is: **debit 5100** (or your expense accounts, pre-tax), **debit 1200** (recoverable GST / ITC), **debit 1210** (recoverable QST / ITR), **credit 2100** (accounts payable, incl. tax).

### 3.3 Scan a supplier invoice with AI

1. **Invoices** tab → **Scan invoice with AI** button.
2. Choose an **image** or a **PDF** (up to 20 MB).
3. The AI reads the document and shows the extracted data: supplier, number, dates, pre-tax, GST, QST, incl. tax, number of lines and **confidence level**.
4. Review, correct if needed, then **Create supplier invoice**. The invoice is created as a draft as in §3.2.

> The scan consumes the tenant's **AI credits** (actual token cost, plus a 30% markup). Always check the result: the AI proposes, you validate.

### 3.4 Record a payment

1. Open the invoice → **Payment** button.
2. Enter the **amount** (partial or full), the **date**, the **method** (Transfer, Cheque, Card, Cash, Other), a **reference** and, if needed, the **cash account** (default Cash 1010).
3. **Save**. The system:
   - refuses an **overpayment** (amount greater than the balance);
   - accounts for active **credit notes** (net balance = incl. tax − credit notes);
   - sets the status to **Paid** if the balance reaches zero, otherwise to **Partially paid**;
   - posts the **receipt**: **debit 1010** (cash), **credit 1100** (accounts receivable).

For a supplier invoice, the operation is a **disbursement**: **debit 2100**, **credit 1010**. You can also launch it from the **Accounts payable** tab (the **Pay** button or **Record a disbursement**).

### 3.5 Issue a credit note

1. Open a **sent or paid** invoice → **Credit note** button.
2. Enter the **reason**, the **total credit amount** (incl. tax), the **date** and **internal notes**.
3. **Save**. The credit note is **created then applied immediately**: a balanced **reversing** entry is posted and the **original invoice's balance** is recalculated. The cumulative credit notes can never exceed the original tax-inclusive total. The number follows the format `AV-2026-00007`.

> Compliance: the credit note respects section 350 of the Quebec Sales Tax Act. The credit note's taxes are **inherited** from the original invoice to guarantee balance.

### 3.6 Chase a late payer

1. Open a **sent / partially paid / overdue** invoice → **Reminder** button.
2. Choose the **level**: Courteous (D+3), Firm (D+15), Insistent (D+30) or Formal demand (D+60). Adjust the recipient and the message.
3. **Send**. The history is available via the **Reminder history** button.

> Reminders can also go out **automatically** each day (see §5.7 on the daily job).

### 3.7 Set up recurring invoicing

1. Open an existing invoice that will serve as a **template** → **Recurring** button.
2. Set the **name**, the **frequency** (monthly, quarterly, etc.) and its **multiplier**, the **first-generation date** (it must be in the future), the **end date** or the **maximum number of occurrences**, the **status** of the produced invoices (Draft or Sent) and **automatic email sending**.
3. **Save**. The template appears in the **Recurring** tab, where you can **pause** it, **reactivate** it, **generate now** an occurrence or **cancel** it.

### 3.8 Post a manual journal entry

1. **Journal** tab → **New entry** button.
2. Enter the **description**, the **type** (Sale, Purchase, Salary, Adjustment, Other) and at least **two lines** (account, label, and a **debit** or a **credit** — never both on the same line).
3. The **Balanced / variance** indicator must be green (debits = credits, to within one cent). **Save**: the entry is born as **Draft**.
4. When it is ready, click **Validate**. The system rechecks the balance then freezes the entry (**Validated**).

### 3.9 Produce a progress claim (CCDC)

1. **Progress claims** tab → choose the **project**.
2. If the **schedule of values** is empty, enter a **Quote number** and click **Import from quote** (or add the **value items** one by one). The value of an imported item is the **selling price** (the quote amount marked up with administration, contingencies and profit).
3. For each item, enter the **% complete to date**. The **This claim** column shows the amount to bill (progress × contract value, less what is already billed).
4. Click **Create progress claim**. The system:
   - bills only the items whose progress has **increased**;
   - computes the **holdback** (pre-tax × rate, default 10%) and records it on the claim;
   - creates a **Progress claim** invoice as **Draft**, numbered in order within the project;
   - protects the operation with a **per-project lock** (no double billing).
5. The **Net payable** = Total incl. tax − Holdback. The claim appears under "Issued progress claims".

### 3.10 Create and release a holdback

1. **Holdbacks** tab → **New holdback** button.
2. Choose an **invoice** (issued), a **rate** (empty = the tenant's rate), the **work completion date** and **notes**. **Save**.
   - For a **customer**: **debit 1150** (holdbacks receivable), **credit 1100** (accounts receivable).
   - For a **subcontractor**: **debit 2100** (accounts payable), **credit 2150** (holdbacks payable).
   - The calculation base is the **pre-tax** amount.
3. At the end of the warranty, select the holdback and click **Release**.
   - For a **customer**: **debit 1010** (cash), **credit 1150**.
   - For a **subcontractor**: **debit 2150**, **credit 1010** (cash).
   - The release entry is **validated and irreversible**.

### 3.11 Close an accounting period

1. **Periods** tab → **New period** button (fiscal year, dates, name), if needed.
2. On an **Open** period, click **Close** and confirm.
3. Result: **no entry** can be posted with a date in the period anymore. Any attempt to post (invoice, payment, progress claim, reversal) is **refused**.
4. An administrator or an accountant can **Reopen** a period if necessary.

### 3.12 Record and depreciate a fixed asset

1. **Fixed Assets** tab → **New asset** button: name, category, acquisition date, **cost**, **useful life** (in months), **method** (straight-line or declining-balance), residual value.
2. Each month, set the **month** in the toolbar and click **Generate depreciation**: the system computes the charge (straight-line = (cost − residual) / life; declining-balance = net value × rate / 12) and records it in the general ledger. The final period settles exactly to the residual value.

### 3.13 Export the accounting data

- **Journal tab**: **Export CSV**, **QuickBooks IIF**, **Sage 50 CSV**.
- **Chart of Accounts tab**: **Export CSV**.
- **General Ledger tab**: export the ledger to CSV.
- **Financial Statements → Taxes**: **Export CSV** of the return.
- **Per invoice** (detail panel): **PDF**, **Excel (.xlsx)**, **QuickBooks CSV** (and "Copy CSV").

All CSVs are encoded in UTF-8 with a BOM (for correct display of accented characters in Excel) and protected against formula injection.

### 3.14 Use the AI accounting assistant

1. **AI Assistant** tab. Enter your request in natural language (for example "post the entry for the $1,200 insurance paid in cash").
2. The assistant can **consult** your accounting data, then **propose** an entry or an invoice as a detailed **card**.
3. Review the proposal, then click **Confirm** (the entry is then created) or **Cancel**. Nothing is written until you have confirmed.

---

## 4. Reference

### 4.1 Invoice statuses

| Status | Color | How it is obtained |
|--------|---------|----------------------|
| **Draft** | gray | On creation. No general-ledger entry |
| **Sent** | indigo | On leaving draft (triggers posting) |
| **Partially paid** | amber | **Automatic** after a partial payment |
| **Paid** | green | **Automatic** when the balance reaches zero |
| **Overdue** | red | Manual, or **automatic** via the daily job (past due date, remaining balance) |
| **Cancelled** | gray | By cancellation (automatic reversal of the linked entry) |

Document types: **Invoice**, **Credit note** (avoir), **Progress claim** (progress billing).

### 4.2 Automatically generated general-ledger entries

| Operation | Debit | Credit | Journal type |
|-----------|-------|--------|-----------------|
| **Customer** invoice (sale) | 1100 Accounts receivable (incl. tax) | 4100 Revenue (pre-tax) + 2200 GST + 2210 QST | Sales |
| **Customer** credit note | reverse of the sale | reverse | Sales (credit note) |
| **Supplier** invoice (expense) | 5100 (or expense accounts, pre-tax) + 1200 GST + 1210 QST | 2100 Accounts payable (incl. tax) | Purchases |
| **Supplier** credit note | reverse of the purchase | reverse | Purchases (credit note) |
| Customer **receipt** | 1010 Cash | 1100 Accounts receivable | Receipt |
| Supplier **disbursement** | 2100 Accounts payable | 1010 Cash | Disbursement |
| Customer **holdback** | 1150 Holdbacks receivable | 1100 Accounts receivable | JE-RET |
| Subcontractor **holdback** | 2100 Accounts payable | 2150 Holdbacks payable | JE-RET |
| Customer holdback **release** | 1010 Cash | 1150 | JE-LIB |
| Subcontractor holdback **release** | 2150 | 1010 Cash | JE-LIB |
| **Payroll** (from the Time Tracking module) | 6100 Wages (gross + contributions) | 2300 Net + 2310 Source deductions + 2320 Contributions | Payroll (draft) |
| **Depreciation** | 6900 Depreciation | 1510 Accumulated depreciation | Depreciation |

Before any recording, the system **checks the balance** (|debits − credits| ≤ $0.01) and refuses the entry otherwise. In multi-jurisdiction (United States), the second-tax line (QST) is **not** inserted when the tenant has no second tax.

### 4.3 Journal types and number prefixes

| Type | Number prefix |
|------|-------------------|
| Sales | `JE-VTE-…` |
| Purchases | `JE-ACH-…` |
| Bank | `JE-BNQ-…` |
| Receipt | `JE-ENC-…` |
| Payroll | `JE-PAI-…` |
| Inventory | `JE-STK-…` |
| Miscellaneous operations | `JE-OD-…` |
| Depreciation | `JE-AMO-…` |
| General | `JE-GEN-…` |
| Reversal | `JE-CP-…` |

> The types are stored in the **plural** ("Ventes", "Achats"). The display also recognizes the older singular forms, for backward compatibility.

### 4.4 Quebec chart of accounts (35 preloaded accounts — extract)

| Code | Name | Type |
|------|-----|------|
| 1010 | Cash | Asset |
| 1100 | Accounts receivable | Asset |
| 1150 | Holdbacks receivable | Asset |
| 1200 | GST receivable (ITC) | Asset |
| 1210 | QST receivable (ITR) | Asset |
| 1500 / 1510 | Fixed assets / Accumulated depreciation | Asset |
| 2100 | Accounts payable | Liability |
| 2150 | Subcontractor holdbacks payable | Liability |
| 2200 | GST payable | Liability |
| 2210 | QST payable | Liability |
| 2300 | Wages payable | Liability |
| 2310 | Source deductions | Liability |
| 2320 | Payroll taxes (CNESST, etc.) | Liability |
| 4100 | Construction revenue | Revenue |
| 5100 – 5500 | Direct costs (materials, labour, subcontracting, equipment, site) | Expense |
| 6100 – 6900 | Overhead (administration, rent, depreciation) | Expense |

The chart is preloaded according to the country: **Quebec 35 accounts**, **United States 36** (with a state-tax account), **Canada standard 35**. Preloading is **idempotent** (it never duplicates accounts). You can add your own accounts; the **code** (4 to 10 digits) is immutable after creation, and an account already used in the general ledger can no longer be **reclassified** (type, class, normal balance).

### 4.5 Main endpoints (API)

Common prefix: `/api/erp/v1/accounting`. "accountant" = the `require_tenant_admin_or_role('comptable')` guard.

| Method and path | Access | Business role |
|-------------------|------|-------------|
| `GET /invoices` · `GET /invoices/{id}` | authenticated | List / open invoices |
| `POST /invoices` | authenticated | Create an invoice (customer or supplier) |
| `PUT /invoices/{id}` | authenticated | Edit (posts on leaving draft; reverses if cancelled or returned to draft) |
| `DELETE /invoices/{id}` | **accountant** | Delete (draft or cancelled only) |
| `POST /invoices/{id}/payment` | **accountant** | Record a payment |
| `POST /invoices/{id}/credit-note` | **accountant** | Issue a credit note |
| `POST /invoices/{id}/send` · `/generate-html` · `/pdf` | authenticated | Send / preview / PDF |
| `POST /invoices/ai/scan` | authenticated | AI scan of a supplier invoice |
| `GET /invoices/public/{token}` | **public** | View without an account (90-day link) |
| `GET/POST /chart-of-accounts` · `PUT/DELETE /chart-of-accounts/{id}` | read: authenticated; write: **accountant** | Chart of accounts |
| `POST /journal` · `/with-lines` · `/{id}/lines` · `PUT /{id}/validate` | **accountant** | Journal entries |
| `GET /journal` · `/ledger` · `/trial-balance` | authenticated | Lookups |
| `GET /balance-sheet` · `/income-statement` · `/cash-flow` · `/tax-declaration` | authenticated | Financial statements |
| `POST /cedule/from-devis/{id}` · `GET /cedule` · `POST /decomptes` | authenticated | CCDC progress claims |
| `GET /wip` | **accountant** | Work in progress |
| `GET /holdbacks` · `POST /holdbacks` · `PUT /holdbacks/{id}/release` | read: authenticated; write: **accountant** | Holdbacks |
| `GET/POST /periods` · `PUT /{id}/close` · `/{id}/reopen` | read/create: authenticated; close/reopen: **accountant** | Periods |
| `GET/POST /fixed-assets` · `POST /generate-depreciation` | read: authenticated; depreciation: **accountant** | Fixed assets |
| `POST /sync-all` · `/sync-factures` · `/sync-depenses` | authenticated | Retroactive synchronization |
| `POST /ai/chat` · `/ai/confirm-action` | authenticated (+ AI credits) | Accounting assistant |
| `POST /cron/daily` | **task token** (no session) | Automatic daily maintenance |

### 4.6 Calculations to know

- **Invoice taxes**: Total incl. tax = pre-tax + GST (5%) + QST (9.975%) in Quebec. The actual rates are those **frozen** on the invoice.
- **Holdback**: Amount held = **pre-tax × rate** (default 10%).
- **Progress claim**: This claim = (% complete × contract value) − already billed, only if positive.
- **Work in progress (WIP)**: % complete = costs incurred / estimated cost; Recognized revenue = min(%, 100%) × contract value; Variance = Billed − Recognized revenue (positive = overbilling, negative = underbilling). Everything is **pre-tax**.
- **Depreciation**: straight-line = (cost − residual value) / life in months; declining-balance = net book value × rate / 12.
- **AI cost**: actual token cost **× 1.30** (a 30% markup), debited from the tenant's prepaid credits.

### 4.7 Known limits

- Invoice lines are **read-only** in the detail (edit via the Edit window).
- Moving to **Paid / Partially paid** is not possible by hand (only through a payment).
- Invoice **deletion** is limited to the Draft and Cancelled statuses.
- **Recurrence**: no direct-creation button (you start from an existing invoice).
- **Progress-claim schedule**: the value-item window only exposes the description and the value.
- **Cash Flow**: no date filter of its own.
- **WIP**: the estimated cost is flagged "to verify / unavailable" when the source is not reliable.
- The invoice **public link** is throttled by a rate-limit protection (effective per server instance).

---

## 5. Integrations and FAQ

### 5.1 Payroll and its link with accounting (Module 12 — Time Tracking)

The **complete Quebec payroll** is **not** in this module: it is in the **Time Tracking** menu (`/pointage`, the "Time Tracking & Payroll" page), **Payroll** and **CCQ Payroll** tabs. Its engine (`payroll.py`, guard `require_payroll_access`) computes, for each employee and each period:

- the **gross pay** (fixed salary or regular hours + overtime hours × 1.5) and **vacation pay** (4% by default in Quebec);
- the progressive federal and provincial **income tax**;
- the **RRQ** (Quebec Pension Plan, 6.30%) and **RRQ2** (4% on the upper band) contributions, **RQAP** (Quebec Parental Insurance Plan, 0.430% employee / 0.602% employer), **EI** (Employment Insurance, 1.30% / 1.82%);
- the **employer contributions**: **CNESST** (Quebec's workplace health and safety board, 1.54% by default, **to be configured per company**), **FSS** (Health Services Fund, 1.65%) and **CCQ** (Commission de la construction du Québec, funding rate + hourly rate per the collective agreement).

Periods are counted 52 times a year (weekly), 26 (every two weeks) or 12 (monthly). When a payroll is generated, the system produces it as a draft, locks the period against duplicates, and recomputes the annual (year-to-date) totals.

**The bridge with accounting**: a payroll (a "payroll run") automatically generates a **draft journal entry** (manual validation required, on the accountant's advice): **debit 6100** (wages + contributions, allocated by project), **credit 2300** (net pay), **credit 2310** (source deductions), **credit 2320** (employer contributions). You find it in the Accounting **Journal** tab, where you **validate** it.

> **Rates to validate.** The CNESST and CCQ rates are **settings** (default estimates), not fixed legal rates. Configure them for your company. The year's rate tables are marked "to be validated".

### 5.2 The T4 / RL-1 / PD7A tax slips (Module 12 — Time Tracking)

The year-end slips are also in the **Time Tracking** menu, **Slips** tab. They are fed by the payroll's **annual (year-to-date) totals**:

- **T4** (federal, CRA — Canada Revenue Agency): boxes 14 (income), 16/17 (RRQ), 18 (EI), 22 (tax), 55 (RQAP), etc.
- **RL-1** (Quebec, Revenu Québec — Quebec's revenue agency): boxes A (income), B (RRQ), C (EI), E (tax), G, H, I.
- **PD7A**: a calculation aid for the **monthly source-deduction remittance** (CRA block = federal tax + EI; Revenu Québec block = provincial tax + RRQ + RQAP + FSS + CNESST).

The **Social Insurance Number (SIN)** is decrypted in an **audited** manner (Law 25 compliance) each time a slip is produced.

> **Important warning.** The **RL-1 is preliminary and unofficial** (Revenu Québec certification not obtained; the PDF carries a watermark). The **T4** and **PD7A** are **aids**: their XML files and vouchers are **not** guaranteed to comply with the official transmission templates. Moreover, the second **RRQ2 contribution is not yet tracked per pay record**, so it is 0 in the PD7A remittance. In plain terms: **these slips do not replace a certified, transmittable payroll.** Have them validated by your accountant.

### 5.3 The contingency fund (Law 16) — Real Estate / Land area

The **contingency fund** (reserve fund) is a **condominium** management tool (contingency-fund study under Quebec's **Law 16**). It has **no screen in `/comptabilite`**: it lives in the **Real Estate / Land area** (`/immo` storefront, developer back-office). Its engine (`fonds_prevoyance.py`) lets you:

- describe a **condominium (co-ownership)** and its **building components** (envelope, structure, mechanical systems, site amenities);
- launch a **study** and generate **25-year projections** with **three** contribution **scenarios** (level, progressive +3%/yr, variable with a safety margin), accounting for cyclical replacements and inflation;
- keep a **maintenance log** and produce sale **certificates**;
- compute a **reconstruction value** (2025 Quebec rates: Economy $250, Standard $325, Mid-range $387, High-end $487 per square foot);
- rely on **AI** analyses (health score, Law 16 compliance, contribution suggestion).

All these functions are open to any authenticated user of the tenant (they do not require the accountant role). To use it, refer to **Module 34 — Real Estate**.

### 5.4 Quotes, case files and CRM

- When creating a customer invoice, a **project** can be linked; the invoice then inherits the **quote's taxes** if a quote is linked.
- **Progress claims** import their schedule of values **from an accepted quote** (the value item equals the quote's **selling price**).
- An invoice linked to an opportunity case file is **automatically attached** to it, and appears in the case file's 360° view.

### 5.5 Purchase orders (Module 13 — Store)

A supplier invoice can be **created from a received purchase order**. The **accounting synchronization** (`sync-depenses`) generates purchase entries for transmitted orders, and wage entries from employee hours. See **Module 13 — Purchase Orders**.

### 5.6 B2B customer portal and Stripe

- Invoices are **not** displayed in the B2B customer portal; to send an invoice, use **email** (with the 90-day public link) or the **PDF**.
- There is **no** invoice payment via Stripe in this module. Stripe is used for the tenant's subscription and AI credits, not for settling customer invoices.

### 5.7 Automatic daily maintenance

A scheduled job (`POST /accounting/cron/daily`, protected by a **task token** rather than a user session) runs each night across all active tenants to: (1) flip past-due, still-unpaid invoices to **Overdue**; (2) **generate** the recurring invoices that are due; (3) send the **automatic reminders**. This is why an invoice can turn "Overdue" on its own.

### 5.8 AI assistant: what it can and cannot do

- It **proposes**, you **confirm**. No entry or invoice is created without your **Confirm** click.
- It reads only **accounting tables** (a strict allowlist): it **does not see** payroll, wages or HR data.
- Creating an **entry** via the AI requires, at confirmation time, an **accountant** or **administrator** role. Creating an **invoice** via the AI follows the same rule as manual creation (open to any authenticated user).
- Each exchange consumes **AI credits** (actual cost plus a 30% markup).

### 5.9 Frequently asked questions

**Where is payroll? I can't find it in Accounting.**
It is in the **Time Tracking** menu (`/pointage`), **Payroll** and **CCQ Payroll** tabs. See Module 12. Accounting only **receives** the payroll entry (as a draft) and lets you validate it.

**And the T4 / RL-1?**
Also in **Time Tracking**, **Slips** tab. Note: the RL-1 is **unofficial** and the files are **not** guaranteed transmittable (see §5.2).

**Where is the contingency fund announced in the title?**
In the **Real Estate / Land** area (Module 34), not here. No tab of `/comptabilite` gives access to it.

**Why doesn't my invoice have a general-ledger entry?**
Because it is still in **Draft**. The entry is generated when the invoice **leaves draft** (Sent). Click **Send**.

**Can I return a posted invoice to Draft?**
Yes, and in that case the system automatically **reverses** the sales entry and unlinks it from the invoice. This is recent behaviour, safe from an accounting standpoint.

**How do I correct an already-paid invoice?**
You do not edit it directly. Issue a **credit note** to neutralize all or part of it, then re-invoice if needed.

**How do I cancel a validated entry?**
You cannot erase it. Post a reversing **adjustment entry** (reversal), or let the system do it via an invoice cancellation.

**Do Draft entries appear in the financial statements?**
No. The balance sheet, income statement, general ledger and tax return count only **Validated** entries.

**I am the owner but my role is "user". Am I blocked from accounting actions?**
No. The guard recognizes the **administrator** flag (`is_admin`), unforgeable and re-read server side. An administrator-owner has access to the accounting operations even if their nominal role is "user".

**My screen is read-only and I can't save anything. Why?**
The tenant is probably in **view-only mode** (subscription missing or cancelled). Regularize the subscription to regain write access.

**Can I invoice a customer outside Quebec (HST, US tax)?**
Yes: the taxes are **multi-jurisdiction**. An invoice freezes its own labels and rates; for US companies, the **Tax USA** tab and the single-tax logic apply (no second tax).

**Does the QuickBooks export work with QuickBooks Online?**
The **IIF** export targets QuickBooks Desktop. For QuickBooks Online, use the per-invoice **QuickBooks CSV**, or the OAuth connector of the Integrations module.

**The progress claim creates an invoice: do I need to do anything more?**
The progress-claim invoice is born as a **Draft**. Send it like an ordinary invoice to post it (the holdback is already computed and deducted from the net payable).

### 5.10 What does not exist (limits)

- No payroll, no pay stubs, no tax slips **in this module** (they are in Time Tracking).
- No contingency fund **in this module** (Real Estate area).
- No invoice payment via Stripe, no display of invoices in the B2B portal.
- No line editing in the detail (Edit window), no manual move to "Paid".
- No deletion of an invoice other than Draft or Cancelled, no erasure of a validated entry.
- No second tax for a single-tax tenant (United States).

---

## 6. Summary

- The **Accounting** module (`/comptabilite`, calculator icon) is the company's **general ledger and invoicing**. Default open tab: **Global View**.
- **Despite the title, payroll, the slips and the contingency fund are NOT here**: **payroll** and the **slips** (T4, RL-1, PD7A) are in **Time Tracking** (Module 12); the **contingency fund** (Law 16) is in **Real Estate** (Module 34). Accounting **receives** the payroll entry (as a draft) but computes no salary.
- **18 tabs** (plus **Tax USA** for US companies), in six groups: Overview, Invoicing, Construction, General ledger, Reports, Management, plus the AI Assistant. The **Terms** tab is visible only to administrators.
- **Invoices**: customer/supplier creation, **AI scan**, sending by email (90-day public link), **payments** before/after credit notes, **credit notes** (QSTA section 350), **recurrence** (from an existing invoice), four-level **reminders**, PDF / Excel / QuickBooks CSV.
- **Double-entry general ledger**: chart of accounts (35 accounts for Quebec), **Draft → Validated** entries (frozen), general ledger, transactions, and **financial statements** (Balance Sheet, Income Statement, Cash Flow, Taxes) that count only validated entries.
- **Construction billing**: **CCDC progress claims** (schedule of values imported from a quote), **work in progress** (WIP, cost-to-cost, read-only), **holdbacks** (1150 customers, 2150 subcontractors, pre-tax base).
- **Automatic posting**: an invoice that leaves draft generates its entry; cancellation or a return to draft **reverses** automatically (never an erasure of a validated entry).
- **Periods**: closing **effectively blocks** any entry dated within the period.
- **Permissions**: consultation open to everyone; sensitive operations (delete, collect, credit, validate an entry, manage accounts, release a holdback, depreciate, close) reserved for the **accountant** role or an **administrator**. The `is_admin` flag prevails over the nominal role. **View-only mode** (read-only) if the subscription is missing or cancelled.
- **Money impact**: no direct Stripe payment here; the only costs charged to the tenant are the **AI credits** (invoice scan and accounting assistant, actual cost plus a 30% markup).
- **Exports**: QuickBooks IIF, Sage 50 CSV, chart of accounts / journal / general ledger / trial balance / tax return in CSV, invoices in PDF / Excel / CSV.

---

**Documentation generated from the source code**: `frontend/src/pages/ComptabilitePage.tsx` (18 tabs), sub-components `ComptesPayablesTab.tsx`, `DecomptesTab.tsx`, `WipTab.tsx`, `RetenuesTab.tsx`, `ComptaAssistantTab.tsx`, `api/accounting.ts`; `backend/routers/accounting.py` (93 endpoints), `accounting_ai.py` (assistant), `payroll.py` / `feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` / `feuillets_common.py` (Payroll module), `fonds_prevoyance.py` (Real Estate area).

**Related manuals**:
- Module 12 — Time Tracking and Hours (CCQ payroll, T4 / RL-1 / PD7A slips) — `12-operations-pointage.md`
- Module 13 — Purchase Orders (supplier invoices, posting of purchases) — `13-operations-bons-de-commande.md`
- Module 07 — Quotes (the quotes that originate progress claims) — `07-ventes-soumissions.md`
- Module 08 — Projects (linking of invoices and work in progress) — `08-ventes-projets.md`
- Module 06 — Case Files / 360 View (direct opening of an invoice) — `06-ventes-dossiers.md`
- Module 34 — Real Estate (contingency fund, Law 16) — `34-terrain-immobilier.md`
- Module 24 — AI Assistant (AI credits for the scan and the accounting assistant) — `24-communication-assistant-ia.md`
- Module 30 — Configuration (taxes, document theme, banking details) — `30-configuration.md`
