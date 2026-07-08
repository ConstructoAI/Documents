# Module 16 — RBQ / CCQ Compliance

> **Version**: 3.0 (rewrite verified line by line against the source code of July 7, 2026 — corrected the number of RBQ categories, the actual tab labels, the AI model, permissions and the scoring scale)
> **Menu label**: "RBQ/CCQ" (`nav.rbqCcq` key), "FIELD" group in the sidebar, `Shield` icon — route `/conformite`
> **Reference code (backend)**: `ERP_REACT/backend/routers/conformite.py` (2531 lines, 31 endpoints including 7 AI tools); `ERP_REACT/backend/routers/conformite_data.py` (398 lines, **a pure static-data module — this is NOT a mounted router**, no endpoints)
> **Reference code (frontend)**: `ERP_REACT/frontend/src/pages/ConformitePage.tsx` (3164 lines, 6 tabs); `ERP_REACT/frontend/src/api/conformite.ts` (587 lines); `ERP_REACT/frontend/src/store/useConformiteStore.ts` (740 lines, Zustand store)
> **Actual API path**: `/api/erp/v1/conformite` (the `/conformite` prefix mounted with `API_PREFIX = /api/erp/v1`)
> **PostgreSQL tables (one per tenant, created on demand)**: `conformite_licences_rbq`, `conformite_cartes_ccq`, `conformite_attestations` (the `fichier_data` BYTEA column for attachments)
> **AI model**: Claude Opus 4.8 (`claude-opus-4-8`), 32,000 tokens maximum per call, billed to the tenant's prepaid AI credits
> **Scope**: a **manual** documentary register of Quebec construction regulatory compliance — the company's RBQ licenses, employees' CCQ skill cards, tax and sector attestations (with an attachment), a dashboard (score, indicators, expiration alerts) and a specialized AI assistant. The module **does not connect to any official registry**: all data entry is done by hand.

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

Centralize, on a single screen, the **regulatory document management** of a Quebec construction company, in order to avoid compliance gaps (an expired license, a lapsed skill card, an out-of-date tax attestation) that prevent you from invoicing, bidding, or starting a job site.

The module answers four concrete needs:

- "Which RBQ licenses does my company hold, and which ones are coming up for renewal?"
- "Which employees have a valid CCQ skill card, and for which trade?"
- "Are my tax attestations (Revenu Québec, CRA, CNESST) up to date, and where is the PDF?"
- "Am I compliant overall?" (score, alerts and AI diagnostics)

### 1.2 What the module manages (Quebec legal context)

- **RBQ licenses (Régie du bâtiment du Québec, Quebec's building authority)**: the RBQ issues to contractor **companies** the mandatory licenses provided for under the Building Act (chapter B-1.1). The module records the license number, the subcategories covered (**27 official codes from 1.1 to 16**), the issue and expiration dates, the bond, the civil liability insurance and the status.
- **CCQ skill cards (Commission de la construction du Québec, the Quebec construction commission)**: the CCQ manages the cards of **individual workers** under the R-20 Act. The module links a card to an employee, with the main trade (**28 trades**), a qualification (Journeyman, Apprentice by period, or Class), the accumulated hours and ASP Construction training.
- **Attestations (5 types)**: time-limited documents required to bid or to start a job — Revenu Québec, Canada Revenue Agency (CRA), CNESST (Commission des normes, de l'équité, de la santé et de la sécurité du travail — Quebec's labour standards and workplace health & safety board), CCQ (statement of situation) and the RBQ solvency attestation. Each attestation can receive **one** PDF or image attachment stored in the database.
- **Dashboard**: a compliance score from 0 to 100, key indicators, expiration alerts and breakdowns by category, trade and type.
- **AI assistant**: seven tools powered by Claude Opus 4.8, specialized in Quebec RBQ/CCQ regulation.

### 1.3 What the module does (verified against the code)

- Maintain a **CRUD register** (create, read, update, delete) of RBQ licenses, CCQ cards and attestations, each with its own filters (status, category/trade/type, text search).
- Attach **one** file (PDF, JPG, PNG or WebP, up to 10 MB) to each attestation, then **download** it when needed.
- Compute a **compliance score** and display a **colored badge** at the top of the page (green ≥ 80%, yellow ≥ 50%, red < 50%).
- Produce **expiration alerts** that can be reviewed in the dashboard (licenses and cards at 60 days, attestations at 30 days via their dedicated endpoint).
- Offer an **AI assistant** with seven tools: compliance analysis, regulatory chat, verification of a project's requirements, regulation search, renewal prediction, report generation and training recommendation.
- Provide a tab to **verify a project's regulatory requirements** (required licenses, CCQ trades, permits, attestations, minimum bond and insurance, journeyman/apprentice ratio).

### 1.4 What the module does NOT do (important limits)

> **Read this before relying on the module.** Several natural expectations are **not** covered. This module is a data-entry register, not an official connector.

- **No real-time verification** against the official RBQ or CCQ registries. Everything is entered by hand. The company name shown on a license is a **free-text field**; it is not pulled from the tenant's profile.
- **No monthly CCQ hours declaration.** The "hours accumulated" field is a manual running total, not a monthly log. Use the CCQ employer portal for official declarations.
- **No export or printing** of compliance data: no PDF, no CSV, no print button. The **only** possible download is an attestation's attachment. Even the **AI report is displayed on screen only** (no export, no save).
- **No bulk import**: no CSV upload of licenses or cards; each record is entered one at a time.
- **Ephemeral AI results.** Project verification, analysis, reports, searches and predictions are **not saved**: they disappear when the page is reloaded or when you switch tabs.
- **In-app alerts only.** No reminders by email, notification or calendar (iCal, Google Calendar). Alerts live in the dashboard.
- **A single attachment per attestation**: no multi-file, no version management.
- **No payroll or CCQ contributions**: hours serve as a renewal reference, not as a basis for wage calculation (see Module 12 Time Tracking and Module 10 Employees).
- **No electronic signature** and no internal approval workflow.
- **"Ghost" tabs from the old Streamlit legacy**: certain display keys (CSST, data-entry training, audit "responsible" columns) exist in the translation files but are **rendered nowhere** in the React interface. "Training" exists here only as an **AI recommendation**; "Audits and inspections" is in fact the **AI project verification** tab (see the caution about labels in 2.1).

### 1.5 Access from the sidebar

- Left sidebar → **FIELD** group (collapsible) → **RBQ/CCQ** (`Shield` icon). Ref. `Sidebar.tsx:72`.
- Direct URL: `/conformite`.
- Breadcrumb and top-bar title: "RBQ/CCQ".
- Title shown at the top of the page: "**RBQ / CCQ Compliance**", subtitle "**Tracking licenses, training and legal obligations**".
- **Default open tab**: **RBQ Licenses** (the component's initial state, `ConformitePage.tsx:158`), even though the "Dashboard" tab appears first in the tab bar.

### 1.6 Permissions and roles

The rights are **not** the same for everyone — this is an important correction compared with earlier versions of this manual.

| Action | Who can do it | Technical guard |
|--------|---------------|-----------------|
| **View** licenses, cards, attestations, statistics, alerts | Any user logged into the tenant | `get_current_user` |
| Download an attestation's attachment | Any user logged into the tenant | `get_current_user` |
| **Create / edit / delete an RBQ license** | Tenant admin | `require_tenant_admin_or_role()` |
| **Create / edit / delete a CCQ card** | Tenant admin | `require_tenant_admin_or_role()` |
| **Create / edit / delete / upload an attestation** | Admin **or** "accountant" role | `require_tenant_admin_or_role("comptable")` |
| **Use the 7 AI tools** | Any logged-in user, **if the tenant has AI credits** | prepaid credits (see below) |

- The "admin" status is **re-read server-side** (`is_admin`) on every request: it cannot be falsified from the browser.
- The **AI tools consume the tenant's prepaid AI credits**. The server first checks that the service is available (otherwise a 503 error), then a billing guard, then the credit balance: a depleted balance returns a **402** error shown in the red banner. The super-admin and exempt companies are not blocked by this check.
- All data is **strictly partitioned by tenant** (a PostgreSQL schema of its own per company). No cross-tenant access is possible.
- There is **no** dedicated "compliance officer" role. Internal best practice: designate a single person responsible for data entry to avoid concurrent edits.

### 1.7 The module's six tabs

| # | Tab (displayed label) | Actual content | Icon |
|---|-----------------------|----------------|------|
| 1 | **Dashboard** | Score, indicators, alerts, breakdowns, resources | `BarChart3` |
| 2 | **RBQ Licenses** (N) | CRUD register of RBQ licenses | `Shield` |
| 3 | **CCQ Cards** (N) | CRUD register of skill cards by employee | `UserCheck` |
| 4 | **Legal documents** (N) | **In reality: the tax and sector attestations** | `FileText` |
| 5 | **Audits and inspections** | **In reality: the AI verification of a project's requirements** | `CheckCircle2` |
| 6 | **AI Assistant** | Six AI tools (analysis, chat, search, prediction, report, training) | `Sparkles` |

The number in parentheses is a **live counter** (number of licenses, cards, attestations). On mobile, the labels are abbreviated: Dash / RBQ / CCQ / Attest. / Verif. / AI.

> **Caution about two misleading labels.** The **"Legal documents"** tab contains exactly the **attestations** (Revenu Québec, CRA, CNESST, CCQ, RBQ). The **"Audits and inspections"** tab manages no audits: it is the **AI project verification** tool. These two labels are leftovers from the old Streamlit version and do not faithfully describe the content. This manual uses the displayed labels, but each time it specifies the actual content.

---

## 2. Interface

### 2.0 Elements common to all tabs

- **Global score badge**: to the right of the page title, a colored badge shows the compliance percentage (green ≥ 80%, yellow ≥ 50%, red < 50%). It is computed by the `GET /conformite/statistics` endpoint.
- **Banners**: a red banner shows errors; a green banner confirms successes (it hides itself after about 4 seconds).
- **Startup error screen**: if the reference data (categories, trades, etc.) fails to load, the page shows "Unable to load the Compliance module" with a **"Retry"** button.
- **Command bar**: each register tab (Licenses, Cards, Legal documents) has a bar with a primary create button, a search field (with a 400 ms debounce before sending) and filter dropdowns.
- **Deletion**: each deletion asks for confirmation via a small native browser dialog ("Delete this license?", etc.).

### 2.1 "Dashboard" tab

Summary screen. It reloads its statistics and alerts each time it is shown.

**Four key indicators (top cards)**

- **Active RBQ licenses** (with the total in the subtitle: "N total")
- **Active CCQ cards**
- **Valid attestations**
- **Compliance score** (percentage)

**Three alert indicators**

- **To renew (60 days)**: the sum of licenses, cards and attestations that expire within the next 60 days.
- **Expired**: the sum of items already expired.
- **Total bond**: the sum of the bonds recorded on licenses, in dollars.

**"Compliance alerts" list**

Up to 30 alerts, each with a priority badge, a plain-text message and a date. If everything is up to date: "No alerts - All documents are up to date". The alerts come from the `GET /conformite/alertes` endpoint (see section 4.9 for the exact windows).

**Three breakdowns**

- Breakdown **by category** (RBQ licenses)
- Breakdown **by trade** (CCQ cards)
- Breakdown **by type** (attestations)

Each breakdown shows a count per value (up to 10 rows for licenses and cards).

**Resources** (if they are loaded)

- **Reference organizations**: 8 official organizations with their role and a contact (RBQ, CCQ, CNESST, Revenu Québec, CRA, ASP Construction, Construction Ombudsman, CMEQ).
- **Practical tips**: 6 tip sections (monitoring deadlines, maintaining financial compliance, managing cards, preparing job-site start-ups, preventing sanctions, developing skills).

### 2.2 "RBQ Licenses" tab

**Command bar**: **"New license"** button, search field, **Status** filter ("All statuses" + the statuses) and **Category** filter ("All categories").

**Table (desktop)** — columns:

| Column | Content |
|--------|---------|
| **Number** | License number (monospace font) |
| **Company** | Name of the holding company (free text) |
| **Categories** | Blue badges (up to 3, then "+N") |
| **Bond** | Formatted amount ("x xxx $") or "--" |
| **Expiration** | Expiration date |
| **Status** | Colored badge (see the rule below) |
| **Actions** | Edit and Delete icons |

**Cards (mobile)**: number, status badge, company name, category badges (up to 4), "Exp: {date}", Edit and Delete links.

**Empty state**: shield icon + "No RBQ license recorded" + "Add a license" button. If filters are active and return nothing: "No results for these filters" + "Reset filters".

**Status badge color rule**: red if the license is expired (or its status is EXPIREE or REVOQUEE); yellow if it expires within 60 days, or if it is SUSPENDUE or EN_RENOUVELLEMENT; green if it is ACTIVE. (These status codes are the literal values stored in the system; see section 4.3 for their meanings.)

**"New RBQ license" / "Edit RBQ license" modal** — fields:

- **License number** (required, e.g. "5734-1234-01", 100 characters max, **unique** within the tenant)
- **Company name** (required, 255 characters max, entered manually)
- **RBQ categories** ("{n} selected"): a scrollable checkbox list showing, for each category, "code — label (subgroup)"
- **Issue date** / **Expiration date** (date pickers)
- **Status** (dropdown: ACTIVE, SUSPENDUE, EXPIREE, REVOQUEE)
- **Bond ($)** / **Liability insurance ($)** (numbers, step of 1000)
- **Notes** (text area, 5000 characters max)

**Validations**: the number and name are required; the issue date must be earlier than or equal to the expiration date; a number already in use is rejected (409 error). "Cancel" and "Create" buttons (or "Save" when editing).

### 2.3 "CCQ Cards" tab

**Command bar**: **"New card"** button, search, **Status** filter and **Trade** filter ("All trades").

**Table (desktop)** — columns:

| Column | Content |
|--------|---------|
| **Employee** | Employee name (or "#id" if the record is missing) |
| **Number** | Card number (monospace font) |
| **Trade** | Main trade |
| **Qualification** | Journeyman / Apprentice by period / Class |
| **Hours** | Accumulated hours ("x xxx h") |
| **ASP** | Green "ASP" badge or "--" |
| **Renewal** | Renewal date |
| **Status** | Colored badge |
| **Actions** | Edit and Delete |

**Empty state**: "No CCQ card recorded" + "Add a card".

**"New CCQ card" / "Edit CCQ card" modal** — fields:

- **Employee**:
  - When **creating**: a dropdown lists **active** employees (up to 100). If the list is empty or unavailable, the interface switches to **manual entry of the employee ID**.
  - When **editing**: the employee is **locked** (read-only) — a card cannot be reassigned to another employee.
- **Card number** (required, e.g. "CCQ-12345", 100 characters max, **unique**)
- **Trade** (dropdown, 28 trades; default "Apprentice")
- **Skill** (dropdown whose options **change depending on the trade**; when the trade changes, the first qualification is selected automatically)
- **Skill category** ("{n}"): checkboxes for **additional trades** (two-column grid; the main trade is excluded)
- **Accumulated hours** (number, step of 100)
- **Issue date** / **Expiration date** (the latter serves as the renewal date)
- **ASP Construction** (checkbox)
- **Status** (dropdown: ACTIVE, SUSPENDUE, EXPIREE)
- **Notes** (text area)

**Validations**: the number and trade are required; the employee ID must be greater than 0; if the employees table exists, the ID must match a real employee (otherwise a 404 error "Employee not found").

**Dynamic qualifications by trade**

| Trade | Qualifications offered |
|-------|------------------------|
| Apprentice | 1st period, 2nd period, 3rd period, 4th period |
| Crane operator | Class 1, Class 2, Class 3, Class 4 |
| Heavy equipment operator | Class 1, Class 2, Class 3, Class 4 |
| Welder | Class A, Class B, Class C |
| Pipe welder | Class A, Class B |
| **All other trades (23)** | Journeyman |

### 2.4 "Legal documents" tab (the attestations)

> Reminder: despite its label, this tab manages only the tax and sector **attestations**.

**Command bar**: **"New attestation"** button, search (on the type, organization, number and notes), **Status** filter and **Type** filter.

**Table** — columns:

| Column | Content |
|--------|---------|
| **Type** | Attestation type |
| **Number** | Number (monospace font) |
| **Organization** | Issuing organization |
| **Expiration** | Expiration date |
| **Status** | Badge (VALIDE, EN_RENOUVELLEMENT, EXPIREE) |
| **File** | If an attachment exists: a **Download** button (with the size in KB); otherwise: an **Upload** button |
| **Actions** | Edit and Delete |

**Empty state**: "No attestation recorded" + "Add an attestation".

**Five official types**

| Code | Label | Issuing organization |
|------|-------|----------------------|
| `REVENU_QUEBEC` | Revenu Québec attestation | Revenu Québec |
| `ARC` | Canada Revenue Agency attestation | Canada Revenue Agency |
| `CNESST` | CNESST compliance attestation | CNESST |
| `CCQ` | CCQ attestation - Statement of situation | Commission de la construction du Québec |
| `RBQ` | RBQ solvency attestation | Régie du bâtiment du Québec |

**"New attestation" / "Edit attestation" modal** — fields: **Type** (required, dropdown of the 5 types) · **Attestation number** (required, 100 characters max) · **Issue date** / **Expiration date** · **Status** (dropdown) · **Notes**. The (type, number) pair is **unique**: a duplicate is rejected (409 error).

**"Upload a document" modal**

- Note shown: "**Accepted types: PDF, JPG, PNG, WebP. Max size: 10 MB.**"
- Only one file accepted. The "Upload" button stays disabled until a file is chosen.
- The file is validated server-side: MIME type in the allowed list (otherwise 415 error), size at most 10 MB (otherwise 413 error), non-empty file, and **header-byte verification** (the real format must match the extension). The filename is sanitized.
- After the upload, the row now shows the **Download** button (with the size).

### 2.5 "Audits and inspections" tab (AI project verification)

> Reminder: this tab records no audit. It is an **AI tool** that, from a project's parameters, enumerates the likely regulatory requirements. The result is **ephemeral** (it is not saved).

**"Regulatory requirements verification (AI)" form**

- **Project type** (dropdown, 7 values: Single-family residential, Multi-family residential, Commercial, Industrial, Institutional, Major renovation, Extension)
- **Estimated value ($)** (number, step of 10,000, default 100,000)
- **Region** (dropdown, 18 values: 17 administrative regions + "Other region")
- **Work types** ("{n}", required): checkboxes among 12 options (Foundation, Framing, Electrical, Plumbing, Heating/Ventilation, Roofing, Exterior cladding, Interior finishing, Masonry, Steel structure, Excavation, Swimming pool)

**"Check requirements"** button (disabled if no work type is checked; a spinner turns during the call).

**Result panel** — the AI returns:

- **Required RBQ licenses** ("Mandatory" / "Recommended" badges)
- **Required CCQ trades** (estimated count + qualification)
- **Required permits**
- **Required attestations**
- **Minimum bond** ($)
- **Minimum liability insurance** ($)
- **Journeyman/apprentice ratio**
- **Estimated** compliance **lead time**
- **Alerts** (potential non-conformities to watch)

> This diagnostic is **indicative**. The system prompt forbids the AI from inventing law numbers ("Prefer to indicate 'to be verified' rather than fabricate a reference"). Always validate complex cases with the RBQ, the CCQ or a specialized advisor.

### 2.6 "AI Assistant" tab

Pill sub-navigation; each tool reminds you: "**This action consumes AI credits.**" An internal lock prevents launching two AI calls at the same time.

Six sub-tools:

1. **"Analyze my compliance"** — "Analyze" button → score, risk level, summary, compliant points, non-conformities (with severity badge), risks, urgent renewals, recommendations and estimated compliance costs. Endpoint `POST /conformite/ai/analyze`.
2. **"Regulatory chat"** — conversation thread, input area (Enter to send, Shift+Enter for a line break, 5000 characters max), **"Include the context of my file"** checkbox, "Send" and "Clear" buttons. Endpoint `POST /conformite/ai/chat`.
3. **"Search a regulation"** — search field (500 characters max) → interpretation, direct answer, results (title, source, reference, summary, official link), key points, warnings and resources. Endpoint `POST /conformite/ai/search-regulations`. The returned links are sanitized server-side (only `http://` and `https://` are kept).
4. **"Predict my renewals"** — button → estimated annual cost, monthly budget, urgent renewals, 12-month calendar (cost, items, actions), expiration risks and recommendations. Endpoint `POST /conformite/ai/predict-renewals`.
5. **"Generate a report"** — button → a professional report (title, dates, overall score, assessment, executive summary, RBQ and CCQ Compliance blocks, attestations, risks, action plan, conclusion, next review). Endpoint `POST /conformite/ai/generate-rapport`. **The report is displayed on screen only: no export or download.**
6. **"Recommend training"** — optional entry of "Planned projects" as addable bullets → skills analysis (strengths, gaps, opportunities), recommended training (organization, duration, cost, audience, priority, benefits), suggested certifications, development plan, annual budget and estimated return. Endpoint `POST /conformite/ai/recommend-formations`.

> The module's seventh AI tool, **project verification** (`POST /conformite/ai/verify-project`), is not in this tab: it is used by the "Audits and inspections" tab (section 2.5).

---

## 3. Step-by-step workflows

### 3.1 Record an existing RBQ license

1. **RBQ Licenses** tab → **"New license"** button.
2. Enter the official **license number** (RBQ format, for example "5734-1234-01").
3. Enter the **name of the holding company** (it may differ from the tenant's trade name if it is a subsidiary).
4. Check all the **subcategories** covered by the license (for example 1.1 + 15.5 + 15.6 for residential + ventilation + air conditioning).
5. Fill in the **issue date**, the **expiration date**, the **status** (ACTIVE), the **bond** and the **liability insurance**.
6. Click **"Create"**. A success message confirms, and both the list and the statistics refresh.

*Permission reminder: reserved for the tenant admin.*

### 3.2 Renew a license before it expires

1. In the **Dashboard**, spot the "expires soon" alerts (60-day window).
2. When the renewal certificate arrives, open the license concerned (Edit icon).
3. Update the **expiration date** and set the **status** back to ACTIVE if the old one was EN_RENOUVELLEMENT or EXPIREE.
4. Adjust the **bond** if the RBQ changed the required threshold.
5. **Save**. The compliance score goes back up automatically.

### 3.3 Suspend or revoke a license

1. Open the license concerned.
2. Change the **status**: SUSPENDUE (temporary, yellow badge) or REVOQUEE (permanent, dark-red badge).
3. Document the reason and the RBQ file number in the **Notes**.
4. **Save**. Note: a suspension removes 6 score points, a revocation removes 15 (see 4.8).

### 3.4 Create the CCQ card of a new employee

1. Beforehand, create the employee's record in **Module 10 (Employees)**: the card attaches to an existing employee.
2. **CCQ Cards** tab → **"New card"** button.
3. Choose the **employee** in the dropdown (active employees). If the list is empty, enter the ID manually.
4. Enter the official **card number**.
5. Choose the **main trade**: the skill list updates automatically.
6. Choose the **skill** (Journeyman by default, otherwise a Class or an apprentice period).
7. Check **additional trades** as needed (the main trade is already excluded).
8. Fill in the **accumulated hours**, the **issue date**, the **expiration date** and check **ASP Construction** if the training is valid.
9. Click **"Create"**.

> **Reminder**: the employee **cannot be changed** after the card is created. To reassign a card, you must delete it and create a new one.

### 3.5 Update a worker's CCQ hours

1. Retrieve the running hours total from the CCQ employer portal or from payroll.
2. **CCQ Cards** tab → open the card (Edit icon).
3. Update the **accumulated hours**.
4. If the progression threshold is reached, change the **skill** (for example "4th period" → "Journeyman" by changing the trade from "Apprentice" to the real trade).
5. **Save**.

### 3.6 Renew a CCQ card

1. Spot the "to renew" alert in the dashboard.
2. Open the card concerned.
3. Update the **expiration date (renewal)** and set the **status** back to ACTIVE if needed.
4. **Save**.

### 3.7 Create an attestation (without an immediate attachment)

1. **Legal documents** tab → **"New attestation"** button.
2. Choose the **type** (Revenu Québec, CRA, CNESST, CCQ or RBQ).
3. Enter the attestation **number**.
4. Fill in the **issue date** and the **expiration date**.
5. Leave the status at VALIDE.
6. Click **"Create"**.

*Permission reminder: reserved for the admin or the "accountant" role.*

### 3.8 Upload an attestation's file

1. **Legal documents** tab → on the row without a file, click **"Upload"**.
2. Choose a **PDF, JPG, PNG or WebP** file of at most 10 MB.
3. Click **"Upload"**. The server validates the type, size and real format, then stores the file.
4. The row now shows **"Download"** with the size.

> **Best practice**: upload the original PDF received by email (not a photo) — it contains the digital signature required for public tenders.

### 3.9 Download an attachment

1. **Legal documents** tab → **"Download"** button (download icon).
2. The file is served as a forced download, with a sanitized name and a revalidated MIME type (served as a generic binary stream if the type falls outside the allowed list).

### 3.10 Verify a project's regulatory requirements (AI)

1. **Audits and inspections** tab.
2. Fill in the **project type**, the **estimated value**, the **region** and check at least one **work type**.
3. Click **"Check requirements"** (the call consumes AI credits).
4. Read the result panel: required licenses (mandatory or recommended), CCQ trades, permits, attestations, minimum bond and insurance, journeyman/apprentice ratio, lead time and alerts.
5. Compare with your existing licenses and cards to spot gaps **before** bidding.

> The result is not saved: copy the important items elsewhere if you need to keep them.

### 3.11 Diagnose your compliance with the AI assistant

1. **AI Assistant** tab → choose one of the six tools.
2. For **"Analyze my compliance"**: click "Analyze" → the AI reads your data and returns a score, non-conformities and recommendations.
3. For **"Regulatory chat"**: ask a question; check "Include the context of my file" for an answer tailored to your records.
4. For **"Predict my renewals"**: get a 12-month calendar with the estimated budget.
5. For **"Generate a report"**: produce a structured report on screen (to copy elsewhere if needed — no export exists).
6. For **"Recommend training"**: add planned projects as needed, then run the team skills analysis.

Each call consumes credits; a depleted balance returns a 402 error in the red banner.

### 3.12 Read the dashboard and prioritize

1. **Dashboard** tab.
2. Check the **score** (badge at the top of the page) and the **alert indicators** ("To renew", "Expired").
3. Go through the **alerts list**: handle the HIGH-priority alerts first (already expired), then the MEDIUM ones (to renew).
4. Review the **breakdowns** to see the concentration by category, trade or type.
5. If needed, refer to the **reference organizations** and the **practical tips** at the bottom of the page.

---

## 4. Reference

### 4.1 The six tabs

| Internal key | Displayed label | Actual content |
|--------------|-----------------|----------------|
| `dashboard` | Dashboard | Summary (score, indicators, alerts, breakdowns, resources) |
| `rbq` | RBQ Licenses (N) | RBQ license register |
| `ccq` | CCQ Cards (N) | CCQ card register |
| `attestations` | Legal documents (N) | Attestation register |
| `verifications` | Audits and inspections | AI project verification |
| `assistant` | AI Assistant | Six AI tools |

### 4.2 API endpoints (31 in total, prefix `/api/erp/v1/conformite`)

**Metadata (2)** — read, any tenant user:

| Method + path | Role |
|---------------|------|
| `GET /constants` | Returns the enumerations and reference lists for the interface |
| `GET /resources` | Returns the 8 organizations and the 6 tip sections |

**RBQ licenses (6)**:

| Method + path | Guard |
|---------------|-------|
| `GET /licences` (status, category, search filters) | any user |
| `GET /licences/expiring?days=60` (bounds 1-365) | any user |
| `GET /licences/{id}` | any user |
| `POST /licences` | admin |
| `PUT /licences/{id}` | admin |
| `DELETE /licences/{id}` | admin |

**CCQ cards (6)**:

| Method + path | Guard |
|---------------|-------|
| `GET /cartes` (status, trade, search filters) | any user |
| `GET /cartes/expiring?days=60` (bounds 1-365) | any user |
| `GET /cartes/{id}` | any user |
| `POST /cartes` | admin |
| `PUT /cartes/{id}` | admin |
| `DELETE /cartes/{id}` | admin |

**Attestations (8)**:

| Method + path | Guard |
|---------------|-------|
| `GET /attestations` (status, type filters) | any user |
| `GET /attestations/expiring?days=30` (bounds 1-365) | any user |
| `GET /attestations/{id}` | any user |
| `POST /attestations` | admin or accountant |
| `PUT /attestations/{id}` | admin or accountant |
| `DELETE /attestations/{id}` | admin or accountant |
| `POST /attestations/{id}/upload` (PDF/JPG/PNG/WebP, 10 MB) | admin or accountant |
| `GET /attestations/{id}/download` | any user |

**Dashboard and alerts (2)**:

| Method + path | Role |
|---------------|------|
| `GET /statistics` | Indicators, score and 3 breakdowns |
| `GET /alertes` | 6 alert families (20 rows max each) |

**AI Assistant (7)** — any tenant user, actual gating by credits:

| Method + path | Role |
|---------------|------|
| `POST /ai/analyze` | Full analysis (score, risks, non-conformities) |
| `POST /ai/chat` | Regulatory chat (question 1-2000 characters, optional context) |
| `POST /ai/verify-project` | A project's regulatory requirements |
| `POST /ai/search-regulations` | Regulation search (query 1-1000 characters) |
| `POST /ai/predict-renewals` | 12-month renewal calendar |
| `POST /ai/generate-rapport` | Structured compliance report |
| `POST /ai/recommend-formations` | Training recommendations (up to 20 planned projects) |

> **Architecture note**: the module is **mounted directly** (without a defensive block) in `erp_api.py`. The reference data comes from `conformite_data.py`, which is a simple **data module** (no endpoint is defined in it). No compliance endpoint lives in `secondary.py`.

### 4.3 Statuses by entity

| Entity | Possible statuses | Colors |
|--------|-------------------|--------|
| RBQ license | ACTIVE · SUSPENDUE · EXPIREE · REVOQUEE | green · yellow · red · dark gray |
| CCQ card | ACTIVE · SUSPENDUE · EXPIREE | green · yellow · red |
| Attestation | VALIDE · EN_RENOUVELLEMENT · EXPIREE | green · yellow · red |
| Risk level (AI) | FAIBLE · MOYEN · ELEVE · CRITIQUE | green · yellow · orange · red |
| Priority (alert) | HAUTE · MOYENNE · BASSE | red · yellow · green |
| Severity (AI non-conformity) | MINEURE · MAJEURE · CRITIQUE | yellow · orange · red |

> Meaning of the status codes: ACTIVE (active), SUSPENDUE (suspended), EXPIREE (expired), REVOQUEE (revoked), VALIDE (valid), EN_RENOUVELLEMENT (under renewal); risk FAIBLE/MOYEN/ELEVE/CRITIQUE (low/medium/high/critical); priority HAUTE/MOYENNE/BASSE (high/medium/low); severity MINEURE/MAJEURE/CRITIQUE (minor/major/critical). These uppercase tokens are the literal values used by the system and returned by the AI.

> Attestation statuses are only **three** in the engine (VALIDE, EN_RENOUVELLEMENT, EXPIREE). Any other value that might appear in a dropdown (for example "lapsed" or "suspended") would be a translation leftover with no real effect.

### 4.4 The 27 RBQ categories (by subgroup)

Source: `CATEGORIES_RBQ`, 27 entries. (Several code comments say "26"; the actual list does contain **27**.)

**General (4)**
- `1.1` — Contractor for new residential buildings, class I
- `1.2` — Contractor for new residential buildings, class II
- `1.3` — Contractor for small buildings
- `16` — General contractor

**Mechanical (10)**
- `2` — Hot-air heating systems contractor
- `3` — Plumbing contractor
- `15.1` — Hot-water heating systems
- `15.2` — Steam heating systems
- `15.3` — Oil-burner systems
- `15.4` — Gas-burner systems
- `15.5` — Ventilation
- `15.6` — Air conditioning
- `15.7` — Refrigeration
- `15.8` — Fire protection

**Electrical (1)**
- `4` — Electrical contractor

**Civil engineering (2)**
- `5.1` — Excavation and earthwork
- `5.2` — Deep foundations

**Structure (6)**
- `6` — Framing and carpentry
- `11.1` — Concrete structures
- `11.2` — Precast concrete
- `12` — Reinforcement and rebar
- `13` — Steel structures and precast elements
- `14` — Masonry

**Envelope (3)**
- `7` — Exterior cladding
- `9` — Roofing
- `10` — Insulation, waterproofing, roofing and metal cladding

**Finishing (1)**
- `8` — Interior systems

### 4.5 The 28 CCQ trades

Source: `METIERS_CCQ`, 28 trades.

**Trades with multiple progression (5)**: Apprentice (4 periods), Crane operator (4 classes), Heavy equipment operator (4 classes), Welder (classes A/B/C), Pipe welder (classes A/B) — see the table in 2.3.

**Trades with a "Journeyman" qualification (23)**: Bricklayer-mason, Heat and frost insulator, Tile setter, Carpenter-joiner, Boilermaker, Cement finisher, Roofer, Electrician, Sheet metal worker, Reinforcing steel erector, Refrigeration mechanic, Elevator mechanic, Millwright, Fire protection mechanic, Assembler-erector, Glazier (assembler-mechanic), Power shovel operator, Painter, Plasterer, Plumber, Resilient flooring layer, Interior systems installer, Pipefitter.

### 4.6 The 5 attestation types

| Code | Label | Organization | Purpose |
|------|-------|--------------|---------|
| `REVENU_QUEBEC` | Revenu Québec attestation | Revenu Québec | Provincial tax compliance |
| `ARC` | Canada Revenue Agency attestation | Canada Revenue Agency | Federal tax compliance |
| `CNESST` | CNESST compliance attestation | CNESST | Occupational health and safety |
| `CCQ` | CCQ attestation - Statement of situation | Commission de la construction du Québec | Contribution status |
| `RBQ` | RBQ solvency attestation | Régie du bâtiment du Québec | Solvency and bonding |

### 4.7 Project verification parameters (AI)

- **Project types (7)**: Single-family residential, Multi-family residential, Commercial, Industrial, Institutional, Major renovation, Extension.
- **Regions (18)**: Bas-Saint-Laurent, Saguenay–Lac-Saint-Jean, Capitale-Nationale, Mauricie, Estrie, Montréal, Outaouais, Abitibi-Témiscamingue, Côte-Nord, Nord-du-Québec, Gaspésie–Îles-de-la-Madeleine, Chaudière-Appalaches, Laval, Lanaudière, Laurentides, Montérégie, Centre-du-Québec, Other region.
- **Work types (12)**: Foundation, Framing, Electrical, Plumbing, Heating/Ventilation, Roofing, Exterior cladding, Interior finishing, Masonry, Steel structure, Excavation, Swimming pool.
- **Project types for the training recommendation (5)**: Residential, Commercial, Industrial, Institutional, Infrastructure.

### 4.8 Compliance score (full scale)

The score starts at **100** then subtracts the following penalties:

| Situation | Penalty |
|-----------|---------|
| **Revoked** RBQ license | −15 |
| **Expired** RBQ license | −10 |
| **Suspended** RBQ license | −6 |
| **Expired** attestation | −8 |
| **Expired** CCQ card | −5 |
| **Suspended** CCQ card | −3 |

- The score is bounded between **0 and 100**.
- **No recorded data → score = 0** (suspended, revoked or under-renewal records still count as "data", to avoid wrongly showing 0 for a tenant that has only this type of record).
- Badge display: **green ≥ 80%**, **yellow from 50 to 79%**, **red < 50%**.

> Correction compared with earlier versions of this manual: **suspensions** and **revocations** do indeed penalize the score (they were not accounted for in the old documented scale).

### 4.9 Alert windows

**Dashboard "Compliance alerts" list** (`GET /alertes`) — 6 families, up to 20 rows each:

| Alert type | Condition | Priority |
|------------|-----------|----------|
| LICENCE_EXPIREE | expiration date passed | HIGH |
| LICENCE_EXPIRE_BIENTOT | expires within 60 days | MEDIUM |
| CARTE_EXPIREE | renewal passed | HIGH |
| CARTE_EXPIRE_BIENTOT | expires within 60 days | MEDIUM |
| ATTESTATION_EXPIREE | expiration date passed | HIGH |
| ATTESTATION_EXPIRE_BIENTOT | expires within 60 days | MEDIUM |

**Standalone "expiring" endpoints** (the `?days=` parameter is bounded to 1-365):

| Endpoint | Default |
|----------|---------|
| `GET /licences/expiring` | 60 days |
| `GET /cartes/expiring` | 60 days |
| `GET /attestations/expiring` | 30 days |

> The deadlines are computed on the **tenant's local date** (the company's time zone), not on the server's UTC time — so the evening "expired / valid" switch matches the local calendar.

### 4.10 PostgreSQL tables (tenant schema)

The three tables are created **on demand** (on the first request), not when the tenant is created.

| Table | Content and constraints |
|-------|-------------------------|
| `conformite_licences_rbq` | `numero_licence` **UNIQUE**, `categories` in JSONB, bond and insurance in numeric |
| `conformite_cartes_ccq` | `numero_carte` **UNIQUE**, `employee_id` (logical link to `employees.id`, validated if the table exists), `metiers_additionnels` in JSONB, `asp_construction` boolean |
| `conformite_attestations` | **UNIQUE (type, numero)**, attachment in `fichier_data` BYTEA + `fichier_nom`, `mime_type`, `taille` |

**Automatically created indexes**: `idx_conf_licences_expiration`, `idx_conf_licences_statut`, `idx_conf_cartes_renouvellement`, `idx_conf_cartes_employee`, `idx_conf_attestations_expiration`, `idx_conf_attestations_type`.

### 4.11 Validations and error codes

| Rule or limit | HTTP response |
|---------------|---------------|
| License number already in use | 409 (conflict) |
| Card number already in use | 409 |
| Attestation (type, number) pair already in use | 409 |
| Issue date later than expiration date | 422 |
| Bond or insurance outside 0 to 1,000,000,000 | 422 |
| Accumulated hours outside 0 to 1,000,000 | 422 |
| Notes longer than 5000 characters | 422 |
| More than 30 categories, or a code longer than 200 characters | 422 |
| Status outside the allowed list | 400 |
| Attestation type outside the list | 400 |
| RBQ category or CCQ trade outside the official list | 400 |
| Nonexistent employee (card creation) | 404 |
| Empty update body | 400 |
| File larger than 10 MB | 413 |
| MIME type outside PDF/JPG/PNG/WebP, or non-conforming header bytes | 415 |
| AI service unavailable | 503 |
| AI credits depleted | 402 |
| AI overloaded ("overload") | 503 |
| Empty or malformed AI response | 502 |

### 4.12 AI costs

- **Model**: `claude-opus-4-8`, 32,000 tokens maximum per call.
- **Base rates**: US$5 per million input tokens, US$25 per million output tokens, US$6.25 per million for cache writes, US$0.50 per million for cache reads.
- **Markup**: × 1.30 (30%).
- **The charge occurs AFTER the response is validated**: a call that fails, returns empty or malformed **is not billed**.
- **Dedicated rate limit**: 10 AI calls per minute per IP address on the `/conformite/ai/` paths (this is the most expensive endpoint class in the application).

### 4.13 Shortcuts and useful behaviors

- **Search**: 400 ms debounce before sending; special characters (`\`, `%`, `_`) are escaped server-side.
- **Regulatory chat**: Enter to send, Shift+Enter for a line break.
- **Category badges**: up to 3 shown on desktop (then "+N"), 4 on mobile.
- **AI lock**: a single AI call at a time (buttons are disabled during processing).

---

## 5. Integrations and FAQ

### 5.1 Integration with Module 10 (Employees)

- The `employees` table is queried to **validate the existence** of an employee when creating a CCQ card; the join displays the full name in the table.
- If the employee record does not exist yet (a very recent tenant), the join is skipped and the table shows "#id".
- **No automatic synchronization**: if an employee is deleted, their CCQ card remains. Best practice: delete the card at the same time.

### 5.2 Integration with Projects, Real Estate and Time Tracking

- **No automatic link.** RBQ licenses are not checked automatically when a project is created.
- The "Audits and inspections" tab (AI verification) is used **manually** before bidding.
- CCQ hours are not fed from **Module 12 (Time Tracking)**: the "accumulated hours" field is entered by hand.
- The phase compliance indicators in **Module 34 (Real Estate)** are separate and unrelated to the licenses recorded here.

### 5.3 Integration with Accounting and Grants

- **No automatic accounting entry.** Bonds and insurance are informational: they are not accounting liabilities or assets. To be recorded manually in **Module 14 (Accounting)**.
- The AI's training recommendations may mention subsidized programs, but with **no** automatic link to **Module 17 (Grants)**.

### 5.4 AI integration and credits

- The 7 AI tools go through the tenant's **prepaid credits** control (the same credits as the ERP's other AI features, see **Module 24**).
- The cost is logged after each successful call.
- User input is framed to prevent instruction injection; the links returned by the search are sanitized (only `http://` and `https://` are kept).

### 5.5 FAQ

**Q: What is the difference between the RBQ and the CCQ?**
A: The **RBQ** issues licenses to **companies** (one per legal entity). The **CCQ** issues skill cards to **individual workers** (the R-20 regime). A company has an RBQ license; each worker has a CCQ card.

**Q: Does the module verify my license numbers against the official RBQ registry?**
A: **No.** There is no connection to the RBQ or CCQ registries. All data entry is manual. For an official verification, use `rbq.gouv.qc.ca`.

**Q: How many RBQ categories does the module know?**
A: **27** subcategories, from code 1.1 to code 16, split across 7 subgroups (see 4.4). Some internal code comments say "26", but the actual list has 27.

**Q: Does the compliance score account for suspensions and revocations?**
A: **Yes.** A revoked license removes 15 points, an expired license 10, a suspended license 6, an expired attestation 8, an expired card 5, a suspended card 3. (This is a correction: the old scale only penalized expirations.)

**Q: Can I export my data to Excel, CSV or PDF?**
A: **No.** There is no export or printing of compliance data. The only download is an attestation's **attachment**. Even the **AI report** is displayed on screen only.

**Q: Does the module send email reminders before deadlines?**
A: **No.** Alerts are only visible in the dashboard. Get into the habit of checking it (deadlines are ideally monitored 60 to 90 days ahead).

**Q: Can I upload several files for the same attestation?**
A: **No.** A single attachment per attestation (PDF, JPG, PNG or WebP, 10 MB max), with no version management.

**Q: What happens if I upload a file larger than 10 MB?**
A: The server rejects it (413 error). Compress the PDF or reduce the image resolution. There is no automatic compression.

**Q: Does the module handle monthly CCQ hours declarations?**
A: **No.** The "accumulated hours" field is a manual running total. For declarations, use the CCQ employer portal.

**Q: Can I reassign a CCQ card to another employee?**
A: **No**, the employee is locked after creation. Delete the card and create a new one.

**Q: Can I record several RBQ licenses for the same company (parent company and subsidiaries)?**
A: **Yes.** Each license is independent; there is no limit.

**Q: Does the "Audits and inspections" tab serve to log my audits?**
A: **No**, despite its name. It is the **AI verification** tool for a project's requirements; it saves nothing. Likewise, the "Legal documents" tab contains exactly the **attestations**.

**Q: Are AI credits consumed if the AI response is bad?**
A: **No.** The charge occurs after the response is validated: a call that fails, returns empty or malformed is not billed.

**Q: Does the AI guarantee legal compliance?**
A: **No.** The diagnostic is indicative. The system prompt forbids inventing law references. For critical cases, consult the RBQ, the CCQ or a specialized advisor.

**Q: Are attachments served securely?**
A: **Yes.** The download forces the file to be saved, the name is sanitized, and the MIME type is revalidated (served as a generic binary stream if it falls outside the allowed list).

**Q: Is there a change log (who changed what)?**
A: The tables keep the creation and last-modification dates, but **not** a detailed per-user log. Use the "Notes" field to record important changes.

**Q: What should I do in the event of an RBQ or CNESST audit?**
A: Download your attachments one by one, generate an AI report if needed (to copy, since it is not exportable), and keep your documents for the legal retention periods applicable in Quebec.

---

## 6. Summary

- **Role**: a **manual** register of Quebec regulatory compliance — RBQ licenses, CCQ cards, attestations — with a dashboard and an AI assistant. **No** connection to the official registries.
- **Access**: sidebar → FIELD group → **RBQ/CCQ** (shield icon), route `/conformite`. Default open tab: **RBQ Licenses**.
- **Six tabs**: Dashboard · RBQ Licenses · CCQ Cards · **Legal documents** (= attestations) · **Audits and inspections** (= AI verification) · AI Assistant. Note: the two bold labels are misleading.
- **Three entities**: RBQ licenses (**27** categories split across 7 groups), CCQ cards (**28** trades with dynamic qualifications), attestations (**5** types, one PDF or image attachment of 10 MB max).
- **Permissions**: viewing for everyone; license and card writes reserved for the **admin**; attestation writes for the **admin or the accountant**; AI tools subject to **prepaid credits**.
- **Compliance score** (0-100): −15 revoked license, −10 expired license, −6 suspended license, −8 expired attestation, −5 expired card, −3 suspended card; 0 if no data; badge green ≥ 80%, yellow ≥ 50%, red < 50%.
- **Alerts**: in the dashboard only (licenses and cards at 60 days, attestations at 30 days via the dedicated endpoint); **no reminders** by email or calendar.
- **Seven AI tools** (Claude Opus 4.8, 32,000 tokens, 30% markup, not billed if the response fails): analyze · chat · verify a project · search · predict · report · training.
- **Key limits**: no export or printing (except an attestation's attachment), AI report on screen only, ephemeral AI results, no bulk import, a single attachment per attestation, employee locked after card creation, no payroll or CCQ declarations.
- **31 endpoints** under `/api/erp/v1/conformite`; **3 tables** per tenant created on demand.

---

**Documentation generated from the code (verified files)**:
- `ERP_REACT/backend/routers/conformite.py` (2531 lines, 31 endpoints including 7 AI tools)
- `ERP_REACT/backend/routers/conformite_data.py` (398 lines, static-data module — 27 RBQ categories, 28 CCQ trades, 5 attestation types, 18 regions, 8 organizations, 6 tip sections)
- `ERP_REACT/frontend/src/pages/ConformitePage.tsx` (3164 lines, 6 tabs)
- `ERP_REACT/frontend/src/api/conformite.ts` (587 lines)
- `ERP_REACT/frontend/src/store/useConformiteStore.ts` (740 lines)

**Related manuals**:
- Module 10 (Employees — create the employee record before the CCQ card) — `10-operations-employes.md`
- Module 12 (Time Tracking — hours are not synchronized here automatically) — `12-operations-pointage.md`
- Module 14 (Accounting — manual recording of bonds and insurance) — `14-operations-comptabilite.md`
- Module 17 (Grants — funded training programs) — `17-terrain-subventions.md`
- Module 34 (Real Estate — separate phase compliance indicators) — `34-terrain-immobilier.md`
- Module 24 (AI Assistant — AI credits and the general operation of the AI) — `24-communication-assistant-ia.md`
- Module 30 (Configuration — tenant and access management) — `30-configuration.md`
