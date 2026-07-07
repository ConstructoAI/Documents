# Module 18 — Grants and Government Assistance

> **Version**: 3.0 (rewrite verified line by line against the source code of July 7, 2026 — corrected the number of tabs (6, not 5), the AI model, the actual write permissions, the calculation of the financial indicators, the exact number of programs and the AI pricing)
> **Menu label**: "Grants" (`nav.subventions` key), "FIELD" group in the sidebar, `Landmark` icon — route `/subventions`
> **Reference code (backend)**: `ERP_REACT/backend/routers/subventions.py` (1687 lines, 23 endpoints including 5 AI tools); `ERP_REACT/backend/routers/subventions_data.py` (732 lines, **a pure static-data module — this is NOT a mounted router**, no endpoints)
> **Reference code (frontend)**: `ERP_REACT/frontend/src/pages/SubventionsPage.tsx` (1921 lines, 6 tabs, all components inline — the `components/subventions` folder does not exist); `ERP_REACT/frontend/src/api/subventions.ts`; `ERP_REACT/frontend/src/store/useSubventionsStore.ts` (Zustand store)
> **Actual API path**: `/api/erp/v1/subventions` (the `/subventions` prefix mounted with `API_PREFIX = /api/erp/v1`)
> **PostgreSQL tables (one per tenant, created on demand)**: `subventions_categories`, `subventions_programmes`, `subventions_demandes`, `subventions_documents` (the `fichier_data` BYTEA column for attachments)
> **AI model**: Claude Opus 4.8 (`claude-opus-4-8`), 32,000 tokens maximum per call, billed to the tenant's prepaid AI credits
> **Scope**: this module is a **grant program catalog and an application tracking register** (discovery → eligibility check → application → submission → decision → disbursement). It is **not** an accounting module: disbursements are **not** posted automatically (see Module 15), and it is **not** an official connector: the module files no application with any agency and connects to no government registry. It is specialized in programs available to Quebec construction, renovation and related-services companies (federal, provincial, municipal).

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

Centralize, on a single screen, the **monitoring and tracking of grant programs** available to Quebec construction companies, so that nothing is left on the table: spot the relevant programs, check your eligibility, prepare a solid file and track each application through to the disbursement of funds.

The module answers five concrete needs:

- "Which grant, loan or tax-credit programs are available to my company?"
- "Am I eligible, and which are most worthwhile given my sector and budget?"
- "Where do my in-progress applications stand (draft, submitted, approved, disbursed)?"
- "What documents must I provide, and how do I maximize my chances of approval?"
- "How much have I requested, how much was granted to me, and which programs are coming up on their deadline?"

### 1.2 What the module does (verified against the code)

- Offer a **catalog of 47 preloaded programs** (federal, provincial, municipal), filterable by category, aid type, government level, difficulty and free text.
- Provide an **algorithmic eligibility checker** (sector- and budget-based scoring), instant and **free** (no AI credits consumed).
- Maintain an **application register** with a full **9-status life cycle** (`BROUILLON` → `EN_PREPARATION` → `SOUMISE` → `EN_EVALUATION` → `INFO_SUPPLEMENTAIRE` → `APPROUVEE` / `REFUSEE` → `VERSEE`, plus `ANNULEE`).
- Allow **uploading supporting documents** (PDF, Word, Excel, images, text, CSV) up to 10 MB per file, stored in the database, with a per-document status and a **download** when needed.
- Display a **dashboard**: four key indicators, three charts (by category, by level, by status) and an alert for programs expiring within the next 30 days.
- Gather **resources**: 8 partner organizations, the Plan PME 2025-2028 ($219M) and two blocks of practical advice.
- Offer a **five-tool AI assistant** (Claude Opus 4.8): suggest programs, analyze eligibility in depth, chat with an expert, generate a checklist for a program and analyze an application's readiness.

### 1.3 What the module does NOT do (important limitations)

> **Read this before relying on the module.** Several natural expectations are **not** covered. This module is an internal watch-and-tracking tool, not an official submission portal and not accounting software.

- **No application filing with the agencies.** The "Submit" button only moves the **internal** status to `SOUMISE` for your own tracking. The actual submission of the file is done **manually** on the official portals (Investissement Québec, SCHL, BDC, your MRC, etc.).
- **No connection to government registries.** The catalog is a dataset preloaded into the ERP; it does not query any official site live. Amounts and conditions are **indicative** and change frequently: always validate on the official sites.
- **No automatic accounting entry.** A disbursement (`statut = VERSEE`) is **not** posted to the general ledger. You must create the entry manually in Module 15 (Accounting).
- **No data export or printing**: no PDF, no CSV, no print button, no report generation. The **only** possible download is an **attachment** you uploaded yourself.
- **Read-only catalog.** You cannot create, modify or delete a program or a category from the interface. The catalog is populated solely by an internal dataset (no write endpoint exists for programs).
- **No email, notification or calendar reminders** (iCal, Google Calendar). The expiring-programs alert lives only in the dashboard.
- **No state machine.** Nothing technically prevents moving an application from `BROUILLON` directly to `VERSEE`: the life-cycle logic is left to your discipline.
- **No internal approval workflow** and no electronic signature.
- **No partial disbursement.** An application carries a single granted amount and a single disbursement date (create several applications to track staggered disbursements).
- **Unused legacy display keys.** Some translation strings (subtitle, "View", "Apply", an old lowercase status block) remain in the language files but are **rendered nowhere** in the current React interface. This manual describes only what is actually displayed.

### 1.4 Access from the sidebar

- Left sidebar → **FIELD** group (collapsible) → **Grants** (`Landmark` icon).
- Direct URL: `/subventions`.
- Title shown at the top of the page: "**Grants**" (next to the `Landmark` icon). The subtitle defined in the translations ("Government programs and financial assistance") is **not** displayed.
- **Default open tab**: **Catalog** (the component's initial state), even though the "Dashboard" tab appears first in the tab bar.

### 1.5 Permissions and roles

Rights are **not** the same for everyone — this is the most important correction versus earlier versions of this manual, which wrongly claimed that "all users can create and edit applications."

| Action | Who can do it | Technical guard |
|--------|---------------|-----------------|
| **View** the catalog, eligibility, applications, statistics, resources | Any user logged into the tenant | `get_current_user` |
| Download an attachment | Any user logged into the tenant | `get_current_user` |
| Run the **eligibility checker** (algorithmic, free) | Any user logged into the tenant | `get_current_user` |
| Use the **5 AI tools** | Any logged-in user, **if the tenant has AI credits** | prepaid credits (see below) |
| **Create / edit / submit / delete an application** | Administrator **or** "comptable" (accountant) role | `require_tenant_admin_or_role("comptable")` |
| **Upload / change the status / delete an attachment** | Administrator **or** "comptable" (accountant) role | `require_tenant_admin_or_role("comptable")` |

- The "administrator or comptable" role is **re-checked server-side** on every request (`is_admin`, which cannot be forged): it cannot be bypassed from the browser. It covers the tenant administrator, the `admin` role, the `comptable` role and the super-administrator.
- **A nuance to know**: the five AI tools are protected only by `get_current_user`. A user who is **neither administrator nor accountant** can therefore **launch the paid AI** (suggestions, analyses, chat) without being able to **create** an application. Govern this usage if cost control matters.
- The **AI tools consume the tenant's prepaid AI credits**. The server first checks service availability (otherwise a 503 error), then the credit balance: a depleted balance returns a **402** error shown in the red banner. The super-administrator and exempted companies are not blocked by this check.
- All data is **strictly partitioned by tenant** (the company's own PostgreSQL schema). No cross-tenant access is possible.

### 1.6 The module's six tabs

| # | Tab (displayed label) | Actual content | Icon |
|---|-----------------------|----------------|------|
| 1 | **Dashboard** | Indicators, charts, expiring-programs alert | `BarChart3` |
| 2 | **Catalog** | 47 filterable programs + AI checklist + application creation | `BookOpen` |
| 3 | **Eligibility** | Free algorithmic checker based on the company profile | `Target` |
| 4 | **My Applications** | Application register + life cycle + documents + AI analysis | `FileText` |
| 5 | **Resources** | Practical advice, 8 organizations, Plan PME 2025-2028 | `Layers` |
| 6 | **AI Assistant** | Three AI sub-tools (suggest, analyze eligibility, chat) | `Sparkles` |

> **Major correction vs the old manual**: the module has **6 tabs**, not 5. The "**AI Assistant**" tab was added and did not exist in the previous documentation. In fact, the internal comment in the page file still wrongly says "5 tabs."

---

## 2. Interface

### 2.0 Elements common to all tabs

- **Page title**: `Landmark` icon + "Grants."
- **Initial load**: on opening, the page fires **seven requests in parallel** (constants, categories, programs, applications, statistics, programs expiring within 30 days, resources). Until the constants arrive, a full-page skeleton is shown.
- **Banners**: a red banner shows errors; a green banner confirms successes (it hides itself after about 4 seconds).
- **Tab switching**: when you leave a tab, the displayed error and the **transient AI results** are cleared; the chat **conversation history**, however, is kept.
- **Search**: the catalog search field waits **400 ms** after the last keystroke before sending the request (special characters are escaped server-side).
- **Confirmations**: submitting or deleting an application, like deleting a document, requires confirmation through a small native browser dialog.

### 2.1 "Dashboard" tab

A summary screen fed by `GET /statistics`.

**Four key indicators (top cards)**

| Indicator | Color | Exact calculation |
|-----------|-------|-------------------|
| **Active programs** | blue | Number of active programs in the catalog |
| **Total applications** | purple | Total number of applications, all statuses combined |
| **Amount requested** | yellow | Sum of the requested amounts of **all** applications **except** those with status `ANNULEE` |
| **Amount granted** | green | Sum of the granted amounts of applications with status `APPROUVEE` or `VERSEE` |

> **Correction vs the old manual**: the "Amount requested" indicator adds up **all** non-cancelled applications — including an application that is merely `SOUMISE` and awaiting a decision. The old documentation wrongly claimed it counted only approved or disbursed applications. "Amount granted," however, does count only `APPROUVEE` and `VERSEE`.

**Three charts**

- **Programs by category** (bar chart) — count by catalog category.
- **Programs by level** (pie chart) — federal / provincial / municipal / mixed breakdown.
- **Applications by status** (bar chart) — shown only if applications exist.

**"Programs expiring within the next 30 days" alert card**

A list of programs whose end date falls within 30 days (name, organization, end date), with the `AlertTriangle` icon. If no program expires, a reassuring message is shown with the `CheckCircle2` icon.

> Most preloaded programs **have no** end date filled in: they will therefore never appear in this alert. Only a few (the ESSOR streams, for example) carry a deadline. This is normal.

### 2.2 "Catalog" tab

The heart of discovery. Two columns of program cards, preceded by a filter bar.

**Filter bar (5 fields, combined)**

| Filter | Values |
|--------|--------|
| **Category** | "All" + the 8 categories |
| **Aid type** | "All" + the 5 types (Grant, Loan, Tax credit, Mixed, Loan guarantee) |
| **Level** | "All" + Federal / Provincial / Municipal / Mixed |
| **Difficulty** | "All" + Easy / Medium / Complex |
| **Search** | Text field ("Program, organization…"), with a 400 ms delay |

Below the filters, a counter shows "**{n} program(s) found**".

> **Note**: the API can also filter by **business sector**, but that filter **is not exposed** in the catalog interface (only the five fields above are). For sorting by sector, use the **Eligibility** tab instead.

**Program card** — each card shows:

- the program **name** and the **organization**;
- a colored **aid-type badge**;
- the **description** (limited to three lines);
- **badges** for level, difficulty and category;
- the **amount range** "min amount – max amount" followed by the **assistance percentage** (%) where applicable;
- a clickable **phone link** (`tel:`) and an **external link** to the official site (new tab);
- the **deadline** ("Deadline: {date}") if the program has one;
- two buttons: **"Checklist (AI)"** (`ClipboardList` icon) and **"Create an application"** (`FilePlus2` icon, primary button).

**Empty state**: "No program matches the filters."

**"Checklist — {name}" modal (AI)**

Clicking "Checklist (AI)" opens a large modal that calls the AI (`POST /ai/checklist`). While it computes: "Generating the checklist…". The AI returns a **Markdown checklist** with checkboxes (`- [ ]`), organized into **five sections**: documents to gather, information to prepare, application items, chronological steps and tips to maximize your chances.

> This action **consumes AI credits**. A lock prevents launching two AI calls at once: if an AI request is already running, the click is ignored.
>
> Since the preloaded programs have **no** eligibility criteria or document list filled in the data (those fields are empty), the checklist relies mostly on the AI's general knowledge and may show "Not specified" for these items. Use it as a starting point, not as official truth.

### 2.3 "Eligibility" tab

An **algorithmic** checker, instant and **free** (no AI credits). Not to be confused with the AI Assistant's "Eligibility analysis" (section 2.6), which is paid.

An info box restates the tool's purpose and specifies: "The quick checker is based on your business sectors and your budget. For a full analysis (size, region, projects), use the AI Assistant tab."

**Profile form**

| Field | Type |
|-------|------|
| **Company size** | dropdown, 5 sizes (Self-employed, Micro, Small, Medium, Large) |
| **Region** | dropdown, 18 Quebec regions (17 administrative regions + "Other") |
| **Approximate budget** | number (default 50,000, step of 10,000) |
| **Urgency** | dropdown, 4 levels (Immediate < 3 months, Short term, Medium term, Long term) |
| **Business sectors** | 19 multi-select chips |
| **Project types** | 13 multi-select chips |

The **"Check my eligibility"** button is **disabled until at least one sector is checked**. A **"Clear"** button appears once a result is obtained.

**Results panel**

Title "**{n} potentially eligible program(s)**", then the **top 10** programs (highest-scored), each with:

- a **score badge** (green if the score is **≥ 50**, amber otherwise);
- the amount "**Up to {amount}**";
- an "**Official site**" link;
- a **"Create an application"** button.

If nothing comes up: "No program is a perfect match. Try adjusting your profile."

**How the score is calculated** (see also section 4.5)

- **+20 points** per business sector shared between your profile and the program;
- **+15 points** if the program's cap covers at least **10% of your budget** — or if the program is **uncapped** (a maximum amount of zero, i.e., a program expressed as a percentage or unlimited);
- **+25 bonus points** if you checked **Construction** or **Renovation** and the program targets the **Construction** sector.

Only programs with a strictly positive score are kept, sorted from highest to lowest; the **top 10** are displayed.

### 2.4 "My Applications" tab

A register of grant applications and their life cycle.

**Command bar**: **"New application"** button (primary), a search field and a **Status** filter ("All statuses" + the 9 statuses). The search is **local** (client-side) and covers the program name, the reference, the notes and the status.

**Application card** — each card shows:

- the **internal reference** (`SUB-...` format) and a **status badge**;
- the **program name** and the **organization**;
- the **requested amount** (on the right) and "Granted: {amount}" if an amount has been granted;
- the **notes** (two lines maximum);
- the **dates**: Created, Submitted, Decision.

**Conditional action buttons**

| Action | Available when… |
|--------|-----------------|
| **Details** | Always |
| **Edit** | The status is **not** `ANNULEE` |
| **Submit** | The status is `BROUILLON` or `EN_PREPARATION` (with confirmation) |
| **Delete** | The status is **neither** `APPROUVEE` **nor** `VERSEE` (with confirmation) |

**Empty state**: "No grant application."

**"New grant application" modal**

- **Program** (required, dropdown). When you arrive from a "Create an application" button in the catalog or eligibility, the program is **pre-filled** (see 3.3).
- **Requested amount ($)** (number).
- **Notes** (text area).

If the program is not chosen: "Program required" error. A safeguard prevents double submission. The application is created with status `BROUILLON` and an automatically generated internal reference.

**"Edit {ref}" modal**

- **Status** (dropdown of the 9 statuses);
- **Requested amount ($)**;
- **Granted amount ($)**;
- **External reference** (the number assigned by the agency);
- **Notes**;
- **Denial reason** (visible **only** if the chosen status is `REFUSEE`).

> **Financial safeguard**: the granted amount cannot exceed the requested amount (the server returns a 400 error otherwise). When you move the status to `APPROUVEE` or `REFUSEE`, the **decision date** is stamped automatically; when you move to `VERSEE`, the **disbursement date** is too. These dates are computed on the **tenant's local calendar** (not the server's UTC time) and never replace a date you entered by hand.

**"Application {ref}" detail modal**

- **Status** and **government-level** badges;
- Program / organization block;
- Grid: requested amount, granted amount, submission date, decision date;
- Notes, **denial reason** (red box if present), program criteria;
- **"AI Analysis" section**: **"Analyze (AI)"** button (`POST /ai/analyze-demande`). The AI returns a **"Readiness: {score}/100"** badge (green ≥ 70, amber ≥ 40, red below), an estimated processing time, then five lists — **Strengths**, **Areas to improve**, **Documents probably missing**, **Writing tips**, **Denial risks** — and an **overall tip**.
- **"Documents" section**: **"Upload"** button, then the list of attachments. Each row shows the name, a document-status badge, the type and size (in KB) and the upload date, with per row: a **status menu** (To provide / Provided / Validated / Rejected), a **Download** button and a **Delete** button (with confirmation). If no file: "No document uploaded."

**File types accepted for attachments**: PDF, Word (DOC, DOCX), Excel (XLS, XLSX), images (JPG, PNG, WebP), text (TXT) and CSV — that is **10 MIME types**. Maximum size: **10 MB** per file.

> Unlike Module 17 (Compliance), which accepts only PDF and images, this module also accepts Word, Excel, text and CSV documents — handy for attaching a business plan, financial projections or a spreadsheet.

### 2.5 "Resources" tab

Three information sections, served by `GET /resources`.

**Practical advice (2 blocks)**

- **Recommended steps** (5 tips): start with your MRC (regional county municipality — the official entry point); stack programs (maximum 80% of eligible expenses); prepare your file (financial statements, business plan, projections); meet the deadlines; consult an expert (MRC advisors are free).
- **Important points** (4 findings): maximum stacking of 80% of eligible expenses; in 2024-2025, 95% of Investissement Québec's direct assistance goes to SMEs (small and medium-sized enterprises); Quebec has roughly 230,000 SMEs (99.7% of the industrial fabric); programs change frequently — check the official sites.

**Partner organizations (8 cards)**: each card carries the name, the role (in italics), a phone contact if any and a "Site" link if any.

| Organization | Role | Contact / site |
|--------------|------|----------------|
| Réseau Accès PME | 500+ professionals for guidance | Via your MRC |
| Investissement Québec | Administers the ESSOR and other programs | 1 844 474-6367 |
| SADC | Community Futures Development Corporations | reseau-sadc.qc.ca |
| APCHQ | Association of construction professionals | apchq.com |
| MicroEntreprendre | Microcredit for entrepreneurs | microentreprendre.ca |
| Annuaire des subventions | 2,696 financial support programs | subventionsquebec.net |
| Gouvernement du Canada | Business benefits finder tool | canada.ca |
| Gouvernement du Québec | Financial assistance for businesses | quebec.ca |

**Plan PME 2025-2028 ($219M)**: a titled card with the total amount and a **three-column table** (Program / Envelope / Description):

| Program | Envelope | Description |
|---------|----------|-------------|
| ESSOR | $136M | Program renewal |
| Réseau accès PME | $22.6M | 450 economic development advisors |
| MicroEntreprendre | $12.7M | Microcredit services |
| Espaces PME innovation | $14.4M | Support for innovative projects |
| Groupes sous-représentés | $14.88M | Training and support |
| Repreneuriat | $17M | Business transfers |

### 2.6 "AI Assistant" tab

Pill-style sub-navigation, with an introductory box. Each call **consumes AI credits**; the "The AI is analyzing your request…" indicator is shown during processing. Three sub-tools:

**1. "Suggest programs"** (`POST /ai/suggest`)

- A **"Describe your project"** text area + an **"Estimated budget ($)"** field + a "Suggest programs" button.
- Result: a **total potential amount** (large figure), two columns **"Federal programs"** and **"Provincial programs"**, two lists **"Tax credits"** and **"Other assistance"**, a **"Financing strategy"** box and a **"Points to watch"** warning box.
- The AI relies on its general knowledge of Quebec and Canadian programs: it may therefore mention programs **absent** from the preloaded catalog.

**2. "Eligibility analysis"** (`POST /ai/analyze-eligibility`)

- **Sector**, **Size**, **Region** dropdowns, **"Revenue ($)"** and **"Number of employees"** fields, **"Planned projects"** chips, an "Analyze my eligibility (AI)" button (disabled until a sector is chosen).
- Result: a **total potential amount**, a **"Recommended programs"** list (each with a compatibility-score badge, the reason, the potential amount, the difficulty of obtaining it and the required actions as a checklist), a **"Programs to avoid"** list, a **"Recommended strategy"** box and an ordered **"Next steps"** list.

**3. "Chat"** (`POST /ai/chat`)

- A chat with a **"Grants expert"**: conversation bubbles, keyboard entry (Enter to send), a "Clear" button, an "Ask your question about grants…" field. Empty state: "Ask a question to start the conversation."
- The conversation history is kept even if you switch tabs (unlike the other AI results, which are cleared).

> **Two "eligibility analyses" not to be confused**: the one in the **Eligibility** tab (section 2.3) is **algorithmic, free and instant**; the **AI Assistant**'s above is **powered by Claude, paid** and richer (it accounts for size, region, revenue and projects). Start with the free version, then refine with the AI as needed.

---

## 3. Step-by-step workflows

### 3.1 Discover available programs

1. **Catalog** tab.
2. Apply the filters: **Category** (for example "Construction & Renovation" or "Energy & Environment"), **Level** (Provincial or Federal), **Difficulty** (start with "Easy", then "Medium").
3. If needed, type a word in **Search** (program or organization name).
4. On an interesting card: click the **phone link** to call the organization, or the **external link** to open the official site in a new tab.

### 3.2 Check your eligibility (free)

1. **Eligibility** tab.
2. Fill in the profile: **Size**, **Region**, **Budget**, **Urgency**, check at least one **Sector** (for example Construction and Renovation) and some **Project types**.
3. Click **"Check my eligibility"** (the button stays disabled until at least one sector is checked).
4. Read the **top 10** programs, from highest to lowest score. A green badge (score ≥ 50) flags the best matches.
5. Click "Create an application" on a selected program, or "Official site" to learn more.

> **Tip**: if you check Construction or Renovation, any program targeting the Construction sector gets a 25-point bonus — so it will rise to the top of the list.

### 3.3 Create an application

1. From the **Catalog** or **Eligibility**, click **"Create an application"** on a program: the app switches to the **My Applications** tab and opens the modal with the **program already filled in**. (You can also start from scratch with the **"New application"** button and choose the program from the menu.)
2. Enter the **Requested amount** and **Notes** as needed.
3. Click **"Create the application"**. The application is created with status `BROUILLON` and an **internal reference** like `SUB-20260707143052-00031`.

*Permission reminder: restricted to the administrator or the "comptable" (accountant) role.*

### 3.4 Upload supporting documents

1. **My Applications** tab → **Details** on the application → **Documents** section → **"Upload"**.
2. Choose a file: PDF, Word, Excel, image (JPG/PNG/WebP), text or CSV, up to **10 MB**.
3. The server validates the **type** (otherwise a 415 error) and the **size** (otherwise a 413 error), then stores the file in the database.
4. The document appears in the list, with status **Provided** by default.

> **Best practice**: attach the original document (the signed PDF received by email, the financial-statements spreadsheet, the business plan) rather than a photo, especially for demanding programs like SCHL (Canada Mortgage and Housing Corporation) or Investissement Québec.

### 3.5 Track a document's status

In an application's Documents section, each row's **status menu** offers four values: **To provide** (gray, a placeholder), **Provided** (blue, default on upload), **Validated** (green, compliant) or **Rejected** (red, to be redone). The **Download** button retrieves the file; the **Delete** button erases it permanently (with confirmation).

### 3.6 Mark an application as "submitted"

> **Important**: this action **files nothing** with the agency. It only moves the **internal** status to `SOUMISE` for your tracking.

1. On a card with status `BROUILLON` or `EN_PREPARATION`, click **"Submit"** and confirm.
2. The status moves to `SOUMISE` and the **submission date** is stamped (tenant's local calendar).
3. **Actual external action**: submit the file on the program's official portal.
4. Back in the ERP, open **Edit** to fill in the **External reference** (the number assigned by the agency).

### 3.7 Track an application's life cycle

Typical progression:

```
BROUILLON → EN_PREPARATION → SOUMISE → EN_EVALUATION
  → INFO_SUPPLEMENTAIRE → APPROUVEE / REFUSEE → VERSEE
```

At each step, open **Edit** and update the **status**, the **granted amount** (if approved), the **denial reason** (if denied) and the **external reference**. The decision and disbursement dates are set automatically when the status changes (see 2.4).

> **No state machine**: nothing technically stops you from skipping steps. Follow the natural cycle out of discipline, so your indicators stay consistent.

### 3.8 Analyze an application's readiness with AI

1. **My Applications** tab → **Details** on the application → **"AI Analysis"** section → **"Analyze (AI)"**.
2. Read the **readiness score out of 100** (green ≥ 70, amber ≥ 40, red otherwise), the estimated time, then the strengths, areas to improve, documents probably missing, writing tips and denial risks.
3. Fix the file accordingly **before** submitting it to the agency.

*This action consumes AI credits.*

### 3.9 Get suggestions and a financing strategy (AI)

1. **AI Assistant** tab → **"Suggest programs"**.
2. Describe the project and enter an estimated budget, then run the suggestion.
3. Read the federal and provincial programs, the tax credits, the other assistance, the total potential amount and the proposed **financing strategy** (the AI accounts for stacking programs).
4. For a cross-check with your full profile, use **"Eligibility analysis"**.

### 3.10 Record a disbursement (manual)

> **The module creates no accounting entry.**

When an application moves to `VERSEE`: go to **Module 15 (Accounting)** → Journal → **New entry**. Record the receipt of funds (debit cash, credit a grant-revenue account). Carry the application's **internal reference** into the entry's description to ease reconciliation.

### 3.11 Delete an application

1. **My Applications** tab → **Delete** on the card (with confirmation).
2. The server **refuses** deletion if the status is `APPROUVEE` or `VERSEE` (400 error). For those applications, use the `ANNULEE` status via **Edit** instead if you need to set them aside.
3. Otherwise, the application is permanently deleted and its **documents** are cascade-deleted.

> There is **no** trash bin and no restore: deletion is permanent.

---

## 4. Reference

### 4.1 The six tabs

| Internal key | Displayed label | Actual content |
|--------------|-----------------|----------------|
| `dashboard` | Dashboard | Indicators, charts, expiration alert |
| `catalogue` | Catalog | 47 programs + AI checklist + application creation |
| `eligibilite` | Eligibility | Free algorithmic checker |
| `demandes` | My Applications | Application register + documents + AI analysis |
| `ressources` | Resources | Advice, organizations, Plan PME |
| `assistant` | AI Assistant | Suggest / Analyze eligibility / Chat |

### 4.2 API endpoints (23 total, `/api/erp/v1/subventions` prefix)

**Metadata (2)** — read, any tenant user:

| Method + path | Role |
|---------------|------|
| `GET /constants` | Enumerations and reference lists for the interface |
| `GET /resources` | 8 organizations + Plan PME 2025-2028 + practical advice |

**Catalog — read-only (4)**:

| Method + path | Guard |
|---------------|-------|
| `GET /categories` | any user |
| `GET /programmes` (filters category, type, level, difficulty, sector, search) | any user |
| `GET /programmes/expiring?days=30` (bounds 1-365) | any user |
| `GET /programmes/{id}` | any user |

> No `POST` / `PUT` / `DELETE` exists on programs or categories: **the catalog cannot be modified through the interface**.

**Applications (6)**:

| Method + path | Guard |
|---------------|-------|
| `GET /demandes` (status filter) | any user |
| `GET /demandes/{id}` (joins the program and the documents) | any user |
| `POST /demandes` | administrator or accountant |
| `PUT /demandes/{id}` | administrator or accountant |
| `POST /demandes/{id}/soumettre` | administrator or accountant |
| `DELETE /demandes/{id}` | administrator or accountant |

**Documents (4)**:

| Method + path | Guard |
|---------------|-------|
| `POST /demandes/{id}/documents` (multipart, 10 MB) | administrator or accountant |
| `GET /documents/{id}/download` | any user |
| `PUT /documents/{id}/status` | administrator or accountant |
| `DELETE /documents/{id}` | administrator or accountant |

**Statistics and eligibility (2)**:

| Method + path | Role |
|---------------|------|
| `GET /statistics` | Indicators and 4 breakdowns (any user) |
| `POST /eligibility-check` | Algorithmic checker — **no AI, no credits** (any user) |

**AI Assistant (5)** — any tenant user, actual blocking via credits:

| Method + path | Role |
|---------------|------|
| `POST /ai/suggest` | Suggest programs based on a project description |
| `POST /ai/chat` | Chat with a grants expert |
| `POST /ai/checklist` | Checklist (Markdown) for a catalog program |
| `POST /ai/analyze-demande` | Analysis of an application's readiness |
| `POST /ai/analyze-eligibility` | In-depth eligibility analysis based on a full profile |

> **Architecture note**: the reference data (enums, catalog, organizations, Plan PME, advice, the AI system prompt) come from `subventions_data.py`, which is a plain **data module** — no endpoint is defined there. All endpoints live in `subventions.py`, mounted directly under `/api/erp/v1/subventions`.

### 4.3 The 9 application statuses

| Status | Displayed label | Color | Set by |
|--------|-----------------|-------|--------|
| `BROUILLON` | Draft | gray | Automatic on creation |
| `EN_PREPARATION` | In preparation | amber | Manual |
| `SOUMISE` | Submitted | blue | Automatic via "Submit" |
| `EN_EVALUATION` | Under review | purple | Manual |
| `INFO_SUPPLEMENTAIRE` | Info required | orange | Manual |
| `APPROUVEE` | Approved | green | Manual |
| `REFUSEE` | Denied | red | Manual (enter the denial reason) |
| `ANNULEE` | Cancelled | dark gray | Manual |
| `VERSEE` | Disbursed | dark green | Manual |

Life-cycle rules:

- **Submission** allowed **only** from `BROUILLON` or `EN_PREPARATION` (400 error otherwise).
- **Deletion** blocked if the status is `APPROUVEE` or `VERSEE` (400 error).
- **Status change** validated against the list above (400 error for an unknown value).
- **Automatic stamping** (tenant's local calendar, never UTC): decision date when the status moves to `APPROUVEE` or `REFUSEE`; disbursement date when it moves to `VERSEE`. A manually entered date is never overwritten.

### 4.4 The 4 document statuses

| Status | Label | Color | Meaning |
|--------|-------|-------|---------|
| `A_FOURNIR` | To provide | gray | Reserved placeholder, document expected |
| `FOURNI` | Provided | blue | Default on upload |
| `VALIDE` | Validated | green | Checked and compliant |
| `REJETE` | Rejected | red | To be redone |

### 4.5 Eligibility checker (algorithmic scoring)

For **each** active program, the score starts at 0 and accumulates:

| Rule | Points |
|------|--------|
| Per shared business sector (case-insensitive comparison) | +20 each |
| The program's cap covers ≥ 10% of the budget, **or** the program is uncapped (a maximum amount of zero or negative) | +15 |
| Construction or Renovation profile **and** program targeting the Construction sector | +25 |

- Only programs with a **strictly positive** score are kept, sorted from highest to lowest; the **top 10** are returned.
- **Free and instant**: no AI credits are consumed.
- The special handling of "uncapped" programs (a maximum amount of 0) fixes an old defect where those programs — often the most generous — were unfairly denied the 15-point bonus.

### 4.6 Dashboard indicators (exact calculations)

| Indicator | Calculation |
|-----------|-------------|
| Active programs | Number of active programs |
| Total applications | Total number of applications |
| Amount requested | Sum of requested amounts, **all** applications **except** `ANNULEE` |
| Amount granted | Sum of granted amounts, `APPROUVEE` or `VERSEE` applications |
| Programs by category | Count by category (catalog join) |
| Programs by level | Count federal / provincial / municipal / mixed |
| Applications by status | Count by status |
| Programs expiring (30 d) | Programs whose end date falls within the next 30 days |

### 4.7 The 8 categories

| Code | Label | Purpose |
|------|-------|---------|
| `PME_GENERAL` | SMEs & Businesses | General programs for SMEs |
| `CONSTRUCTION` | Construction & Renovation | Construction sector programs |
| `ENERGIE` | Energy & Environment | Energy efficiency and sustainable development |
| `FORMATION` | Training & Employment | Training and skills development |
| `INNOVATION` | Innovation & Technology | R&D and digital transformation |
| `REGIONAL` | Regional Development | Regional and municipal programs |
| `DEMARRAGE` | Startup & Business Succession | Business creation and takeover |
| `EXPORT` | Export | Export programs |

### 4.8 Reference enumerations

- **Aid types (5)**: Grant · Loan · Tax credit · Mixed · Loan guarantee.
- **Government levels (4)**: Federal · Provincial · Municipal · Mixed.
- **Difficulties (3)**: Easy · Medium · Complex.
- **Business sectors (19)**: SMEs, Construction, Renovation, Manufacturing, Energy, Housing, Commercial, Residential, Digital, Training, Employer, Exporter, Startup, Business creation, Business succession, Rural, Low income, Heritage, Wood.
- **Regions (18)**: Bas-Saint-Laurent, Saguenay–Lac-Saint-Jean, Capitale-Nationale, Mauricie, Estrie, Montréal, Outaouais, Abitibi-Témiscamingue, Côte-Nord, Nord-du-Québec, Gaspésie–Îles-de-la-Madeleine, Chaudière-Appalaches, Laval, Lanaudière, Laurentides, Montérégie, Centre-du-Québec, Other.
- **Company sizes (5)**: Self-employed · Micro (1-4 employees) · Small (5-49 employees) · Medium (50-199 employees) · Large (200+ employees).
- **Project types (13)**: Startup, Expansion, Modernization, Digital transformation, Energy efficiency, Training, Export, Business succession, Renovation, Equipment, R&D, Hiring, Green energy.
- **Urgency levels (4)**: Immediate (< 3 months) · Short term (3-6 months) · Medium term (6-12 months) · Long term (> 12 months).

### 4.9 Catalog of the 47 preloaded programs

Breakdown: **47 programs** total (SMEs & Businesses 6, Construction & Renovation 6, Energy & Environment 9, Training & Employment 6, Innovation & Technology 6, Regional Development 4, Startup & Business Succession 4, Export 6).

> **Correction vs the old manual**: the actual count is **47**, not "50+". The code's internal comments even say "40+" and **underestimate**; the exact program count is 47.

> Note: the program names and organization names below are stored as internal (French) seed data and display in French in the interface regardless of your UI language. Only the column headers, the aid type and the level are localized.

**SMEs & Businesses (6)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| ESSOR – Volet 1 Études | Investissement Québec | Grant · Provincial | $100,000 · 50% |
| ESSOR – Volet 2 Productivité | Investissement Québec | Mixed · Provincial | $5M · 50% |
| ESSOR – Volet 3 Environnement | Investissement Québec | Mixed · Provincial | $2M · 50% |
| ESSOR – Volet 4 International | Investissement Québec | Mixed · Provincial | $1M · 50% |
| Fonds locaux d'investissement (FLI) | MRC locales | Loan · Municipal | $5,000 – $500,000 |
| Financement PME BDC | Banque de développement du Canada | Loan · Federal | $10,000 – $5M |

**Construction & Renovation (6)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| Fonds logement abordable – Construction | SCHL | Loan · Federal | $50M · up to 95% |
| Fonds logement abordable – Rénovation | SCHL | Mixed · Federal | $10M |
| Rénovation écoénergétique (immeubles collectifs) | SCHL | Grant · Federal | $170,000 / dwelling |
| Certification Novoclimat | Transition Énergétique Québec | Grant · Provincial | No cap · 25% premium |
| Programme Maisons Canada | Gouvernement du Canada | Grant · Federal | $10M |
| Crédit d'impôt RenoVert | Revenu Québec | Tax credit · Provincial | $10,000 · 20% |

**Energy & Environment (9)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| LogisVert | Hydro-Québec | Grant · Provincial | $22,000 |
| Rénoclimat | Transition Énergétique Québec | Grant · Provincial | $20,000 |
| Prêt canadien maisons plus vertes | Gouvernement du Canada | Loan · Federal | $40,000 (interest-free) |
| Initiative maisons plus vertes | Gouvernement du Canada | Grant · Federal | $5,000 |
| Chauffez Vert | Transition Énergétique Québec | Grant · Provincial | $15,000 |
| ÉcoPerformance | Transition Énergétique Québec | Grant · Provincial | $100,000 · 50% |
| Technoclimat | Transition Énergétique Québec | Grant · Provincial | $5M · 50% |
| RénoRégion | SHQ | Grant · Provincial | $25,000 |
| Éconologis | Transition Énergétique Québec | Grant · Provincial | Free services · 100% |

**Training & Employment (6)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| PACME – Formation PME | Emploi-Québec | Grant · Provincial | $100,000 · 50% |
| Crédit d'impôt pour apprenti | Gouvernement du Canada | Tax credit · Federal | $2,000 / apprentice |
| Crédit d'impôt stage en milieu de travail | Revenu Québec | Tax credit · Provincial | No cap · 30% |
| Crédit d'impôt formation PME | Revenu Québec | Tax credit · Provincial | $5,460 / employee |
| Mesure de formation de la main-d'œuvre (MFOR) | Services Québec | Grant · Provincial | $100,000 · 75% |
| Subvention salariale | Emploi-Québec | Grant · Provincial | $50,000 · 50% |

**Innovation & Technology (6)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| Innovation PARI-CNRC | CNRC-PARI | Grant · Federal | $500,000 · 80% |
| RS&DE (recherche et développement) | Agence du revenu du Canada | Tax credit · Federal | $3M · 35% (SMEs) |
| PCAN – Croître en ligne | Gouvernement du Canada | Grant · Federal | $2,400 |
| PCAN – Adoption technologique | Gouvernement du Canada | Mixed · Federal | $15,000 + 0% loan |
| ESSOR – Volet numérique | Investissement Québec | Grant · Provincial | $50,000 · 50% |
| Offensive transformation numérique (OTN) | MEI | Grant · Provincial | $100,000 · 50% |

**Regional Development (4)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| SADC – Développement économique régional | DEC Canada | Mixed · Federal | $250,000 |
| Rénovation de façades commerciales | Municipalités | Grant · Municipal | $66,000 · 50% |
| Restauration de bâtiments patrimoniaux | Municipalités | Grant · Municipal | $100,000 · 50% |
| Dispositifs antirefoulement | Ville de Québec | Grant · Municipal | $5,000 · 50% |

**Startup & Business Succession (4)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| MicroEntreprendre | MicroEntreprendre | Loan · Provincial | $500 – $20,000 |
| Programme Relève entreprise | MEI | Grant · Provincial | $100,000 |
| Repreneuriat Québec | MEI | Grant · Provincial | $50,000 |
| Campus du repreneuriat | MEI | Grant · Provincial | $25,000 |

**Export (6)**

| Program | Agency | Type · Level | Max amount · Assistance |
|---------|--------|--------------|-------------------------|
| CanExport PME | Gouvernement du Canada | Grant · Federal | $75,000 · 50% |
| Export Québec | Investissement Québec | Mixed · Provincial | $100,000 · 50% |
| Programme Frontière (tarifs douaniers) | Investissement Québec | Loan · Provincial | $50M |
| Chantier productivité | Investissement Québec | Mixed · Provincial | $5M |
| Programme IRRT | MEI | Grant · Provincial | $500,000 · 50% |
| BDC – Bois d'œuvre | BDC | Loan · Federal | $10M |

> The exact names, amounts, rates, phone numbers and links live in the internal dataset. Several programs show a maximum amount of zero: these are **uncapped** programs (percentage-based assistance or services), treated as such by the eligibility checker. Amounts are **indicative** and change often — validate on the official sites.

### 4.10 Internal reference format

**`SUB-YYYYMMDDHHMMSS-NNNNN`** — for example `SUB-20260707143052-00031`.

- The timestamp part is built from the creation date (UTC timestamp); the `NNNNN` part is the application's identifier zero-padded to 5 digits.
- The reference is generated **atomically** (insert returning the identifier, then update), so no collision is possible.
- The **external reference** is a separate free-text field, reserved for the number assigned by the agency (for example `ESSOR-2026-12345`).

### 4.11 Validations and error codes

| Rule or limit | HTTP response |
|---------------|---------------|
| Program / project / company / application / document not found | 404 |
| Empty update body or no valid field | 400 |
| Status outside the list of 9 statuses | 400 |
| Submission from a status other than `BROUILLON` / `EN_PREPARATION` | 400 |
| Deletion of an `APPROUVEE` or `VERSEE` application | 400 |
| Granted amount greater than the requested amount | 400 |
| Amounts outside 0 to 1,000,000,000 | 422 |
| Notes over 5,000 characters; denial reason over 2,000; external reference over 255 | 422 |
| Malformed date in an update | 422 |
| Empty file | 400 |
| File over 10 MB | 413 |
| File type outside the allowed list | 415 |
| AI credits depleted | 402 |
| AI access denied (billing guard) | 403 |
| AI service unavailable (SDK missing) | 503 |
| AI overloaded ("overload") | 503 |
| AI request too large | 413 |
| Empty, non-JSON or unexpected-format AI response | 502 |

- AI input bounds: project description 1 to 5,000 characters; chat question 1 to 2,000; context up to 10,000; sector and type lists up to 50 entries; number of employees and revenue bounded.
- The `days` parameter of `GET /programmes/expiring` is bounded from 1 to 365.

### 4.12 AI costs

- **Model**: `claude-opus-4-8`, 32,000 tokens maximum per call. (An old code comment still mentions "Opus 4.6"; the model actually configured is indeed Opus 4.8.)
- **Base rates**: US$5 per million input tokens, US$25 per million output tokens, US$6.25 per million for cache writes, US$0.50 per million for cache reads.
- **Markup**: × 1.30 (30%).
- **Billing happens AFTER the response is validated**: a call that fails, returns empty or is malformed is **not billed** (the JSON response is parsed before any billing).
- **Dedicated rate limit**: 10 AI calls per minute per IP address on the `/subventions/ai/` paths — the most expensive endpoint class.
- **The real AI access control is credits.** The internal billing guard (`check_ai_guard`) in practice lets any authenticated user through; it is the **prepaid-credit balance check** that blocks (402 error "AI credits depleted"). The only indirect link with Stripe is an automatic credit top-up when the balance falls below a threshold.

### 4.13 PostgreSQL tables (tenant schema)

The four tables are created **on demand** (on the first request), not when the tenant is created. The default catalog (8 categories, 47 programs) is **seeded once**, idempotently, on first access.

| Table | Content and specifics |
|-------|-----------------------|
| `subventions_categories` | 8 seeded categories; unique `code`, display order, active |
| `subventions_programmes` | 47 seeded programs; aid type, level, amounts, percentage, `secteurs_admissibles` in JSONB, dates, difficulty; partial unique index on the code |
| `subventions_demandes` | Applications; `reference_interne` (`SUB-...` format), status, amounts, submission / decision / disbursement dates, notes, denial reason. Links to the program, the project and the company are validated **at the application level** (no foreign-key constraint in the database) |
| `subventions_documents` | Attachments; `fichier_data` as BYTEA, MIME type, size, status; **cascade** deletion with the application |

> **Partial seeding**: the programs' "eligibility criteria", "required documents", "email" and "notes" fields are **not** populated by the seed (they are empty). That is why the AI checklist and the application analysis may show "Not specified" for these items.

---

## 5. Integrations and FAQ

### 5.1 Integration with Accounting (Module 15)

- **No automatic accounting entry.** A disbursement (`statut = VERSEE`, granted amount, disbursement date) is not posted to the general ledger.
- To be recorded manually: receipt of funds (debit cash, credit a grant-revenue account). Carry the application's **internal reference** into the entry's description.
- Tax credits (RS&DE, RenoVert, training credit, apprentice credit) are tracked here for reference, but their tax treatment is handled with your accountant and in your tax return.

### 5.2 Integration with Compliance (Module 17)

- **No join** between applications and RBQ (Régie du bâtiment du Québec — Quebec building authority) licenses, CCQ (Commission de la construction du Québec) cards or attestations.
- Several programs (SCHL, Maisons Canada, Investissement Québec) **require** a valid RBQ license and up-to-date tax attestations. Check them in Module 17 **before** bidding, and attach a copy in the application's Documents section.

### 5.3 Integration with Projects and CRM

- An application can carry a **project** identifier and a **company** (client) identifier. These links are validated on creation and edit if the corresponding tables exist, but **there is no linking screen** in the current interface: they are not exposed as editable fields. There is also no project filter in the applications list.
- The links carry **no foreign-key constraint** in the database: integrity relies on application-level validation.

### 5.4 Integration with Documents (Module 7)

- Application attachments are stored **in the Grants module's tables** (as BYTEA), separately from the Documents module. There is no file sharing between the two.

### 5.5 AI and credits integration (Module 25)

- The 5 AI tools go through the **same prepaid-credit mechanism** as the ERP's other AI features (see Module 25).
- The cost is logged after each successful call; a failed call is not billed.
- No integration with QuickBooks, the Vapi voice agent or the SEAOP module in this module.

### 5.6 What the module does not do (reminder)

No official filing to agencies, no connection to registries, no automatic accounting entry, no export, printing or reporting, no user-created program, no email or calendar reminder, no partial disbursement, no state machine, no approval workflow and no electronic signature.

### 5.7 FAQ

**Q: Does the module file my application with the agency when I click "Submit"?**
A: **No.** "Submit" only moves the internal status to `SOUMISE` for your tracking. You must submit the file yourself on the program's official portal.

**Q: Who can create and edit applications?**
A: The tenant **administrator** or someone with the **comptable** (accountant) role (the super-administrator too). An ordinary user can **view** applications, run the free eligibility checker and **use the paid AI**, but **cannot** create, edit, submit or delete an application. (This is a correction: earlier versions wrongly claimed everyone could do everything.)

**Q: How many programs are preloaded?**
A: **47**, across 8 categories (see 4.9). The old documentation said "50+" and the code's internal comments say "40+" — both are inaccurate.

**Q: How many tabs are there?**
A: **Six**: Dashboard, Catalog, Eligibility, My Applications, Resources and AI Assistant. The old manual described only five (the AI Assistant tab was added).

**Q: What is the difference between the Eligibility tab and the AI Assistant's "Eligibility analysis"?**
A: The **Eligibility** tab is an **algorithmic, free and instant** checker (scoring by sector and budget, top 10). The **AI Assistant**'s "Eligibility analysis" is **powered by Claude, paid** and richer (it factors in size, region, revenue and projects). Start with the free one.

**Q: Can I export my applications to Excel, CSV or PDF, or print a report?**
A: **No.** There is no export, no printing and no report generation. The only possible download is an **attachment** you uploaded yourself.

**Q: Can I add or edit a program in the catalog?**
A: **No.** The catalog is read-only and populated by an internal dataset. To track a missing program, create an application on a similar program and document the difference in the Notes, or have the program added by the team that maintains the ERP.

**Q: Why does the AI checklist show "Not specified" for criteria and documents?**
A: Because those fields are not filled in the programs' preloaded data. The AI then relies on its general knowledge. Check the program's official site for the exact list.

**Q: What happens if I enter a granted amount higher than the requested amount?**
A: The server refuses (400 error). The granted amount cannot exceed the requested amount, to keep the financial indicators consistent.

**Q: Can I delete an approved or disbursed application?**
A: **No** (400 error). Use the `ANNULEE` status via Edit instead if you need to set it aside. Deletable applications are removed **permanently**, along with their documents (no trash bin).

**Q: Will the module send me a reminder before a program's deadline?**
A: **No.** The expiring-programs alert lives only in the dashboard (and many programs have no end date filled in). Get into the habit of checking it and note important deadlines in your calendar.

**Q: What files can I upload, and up to what size?**
A: PDF, Word (DOC/DOCX), Excel (XLS/XLSX), images (JPG, PNG, WebP), text (TXT) and CSV — 10 types in all, up to **10 MB** per file. Beyond that, compress the file: there is no automatic compression.

**Q: Can the AI suggest programs that are not in the catalog?**
A: **Yes.** The "Suggest programs" and "Eligibility analysis" tools rely on Claude's general knowledge and may mention programs absent from the preloaded catalog. For exact scoring across the whole catalog, use the Eligibility tab.

**Q: Am I billed if the AI's response is bad?**
A: **No.** Billing occurs after the response is validated: a call that fails, returns empty or has an unexpected format is not billed.

**Q: Can a non-administrator user run up the AI bill?**
A: **Yes, in theory**: the five AI tools are accessible to any logged-in user (as long as credits remain), even without write rights on applications. A limit of 10 calls per minute per IP address caps abuse, and the credit balance remains the ultimate safeguard.

**Q: Can I record several applications for the same program?**
A: **Yes.** Each application is independent; use several applications to track, for example, staggered disbursements or different fiscal years.

**Q: Does the module check program stacking?**
A: **No.** The general rule in Quebec caps stacking at roughly 80% of eligible expenses, but the module does not enforce it. Check stacking with the relevant agencies (see the tips in the Resources tab).

---

## 6. Summary

- **Role**: a grant catalog and application-tracking register for Quebec construction companies. **Not** an accounting module, **not** an official submission portal.
- **Access**: sidebar → FIELD group → **Grants** (`Landmark` icon), route `/subventions`. Default open tab: **Catalog**.
- **Six tabs**: Dashboard · Catalog · Eligibility · My Applications · Resources · AI Assistant. (Correction: the old manual described only five.)
- **47 preloaded programs** in **8 categories** (SMEs 6, Construction 6, Energy 9, Training 6, Innovation 6, Regional 4, Startup 4, Export 6), **read-only**.
- **9 application statuses**: `BROUILLON` → `EN_PREPARATION` → `SOUMISE` → `EN_EVALUATION` → `INFO_SUPPLEMENTAIRE` → `APPROUVEE` / `REFUSEE` → `VERSEE`, plus `ANNULEE`. Submission from draft/preparation only; deletion blocked if approved or disbursed; decision and disbursement dates stamped automatically in local time.
- **Permissions**: viewing, the free eligibility checker and the AI tools for everyone; **creating and editing applications and documents restricted to the administrator or the accountant**.
- **Algorithmic eligibility** (free): +20 per shared sector, +15 if the cap covers ≥ 10% of the budget or if the program is uncapped, +25 construction bonus; top 10.
- **Five AI tools** (Claude Opus 4.8, 32,000 tokens, 30% markup, not billed if the response fails, 10 calls/min per IP): suggest programs, analyze eligibility, chat, generate a checklist, analyze an application. The real blocker is **prepaid credits** (402 error).
- **Documents**: upload to the database up to 10 MB, 10 file types (PDF, Word, Excel, images, text, CSV), 4 statuses (To provide / Provided / Validated / Rejected).
- **Indicators**: Amount requested = sum of all applications except cancelled; Amount granted = sum of approved and disbursed.
- **Internal reference**: `SUB-YYYYMMDDHHMMSS-NNNNN`, generated collision-free. **External reference**: free-text field for the agency's number.
- **Key limits**: no official filing, no registry connection, no automatic accounting entry, no export or printing, non-editable catalog, no email or calendar reminder, no partial disbursement, no state machine.
- **23 endpoints** under `/api/erp/v1/subventions`; **4 tables** per tenant created on demand.

---

**Documentation generated from the code (verified files)**:
- `ERP_REACT/backend/routers/subventions.py` (1687 lines, 23 endpoints including 5 AI tools)
- `ERP_REACT/backend/routers/subventions_data.py` (732 lines, static-data module — 8 categories, 47 programs, 8 organizations, Plan PME 2025-2028, 2 advice blocks, the AI system prompt)
- `ERP_REACT/frontend/src/pages/SubventionsPage.tsx` (1921 lines, 6 tabs)
- `ERP_REACT/frontend/src/api/subventions.ts`
- `ERP_REACT/frontend/src/store/useSubventionsStore.ts` (Zustand store)
- `ERP_REACT/frontend/src/i18n/locales/fr/terrain.json` (interface labels, `subventions` section)

**Related manuals**:
- Module 7 (Documents — separate document management) — `07-ventes-dossiers.md`
- Module 9 (Projects — optional project ↔ application link) — `09-ventes-projets.md`
- Module 15 (Accounting — manual recording of disbursements) — `15-operations-comptabilite.md`
- Module 17 (RBQ/CCQ Compliance — licenses and attestations required by some programs) — `17-terrain-conformite.md`
- Module 19 (Real Estate — SCHL affordable-housing programs) — `19-terrain-immobilier.md`
- Module 25 (AI Assistant — AI credits and general AI operation) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — tenant, role and access management) — `28-configuration.md`
