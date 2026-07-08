# Module 05 — Sales (CRM, opportunities, pipeline, B2B back-office)

> **Version**: 3.0 (complete overhaul verified against the source code, 2026-07)
> **Frontend route**: `/ventes` (side menu "Sales", "Management" group, `TrendingUp` icon). The `/b2b` route redirects to `/ventes?tab=b2b`.
> **API prefix**: `/api/erp/v1`
> **Reference code (backend)**: `backend/routers/crm.py` (2563 lines, 25 CRM routes) · `backend/routers/ventes_ai.py` (479 lines, 2 routes — Sales AI assistant) · `backend/routers/b2b.py` (3017 lines, 33 routes — B2B back-office) · `backend/routers/b2b_ai.py` (340 lines — read-only B2B AI assistant)
> **Reference code (frontend)**: `frontend/src/pages/VentesPage.tsx` (2914 lines, 8 tabs) · `frontend/src/pages/B2bPage.tsx` (1411 lines, 10 sub-tabs) · `frontend/src/components/crm/BATQualificationForm.tsx` (289 lines) · `frontend/src/components/ventes/VentesAssistantTab.tsx` (232 lines) · `frontend/src/components/b2b/B2bAssistantTab.tsx` (152 lines)
> **Frontend API clients**: `api/crm.ts` (384 lines), `api/ventesAi.ts` (75 lines), `api/b2b.ts` (529 lines), `api/b2bAi.ts` (36 lines). Note: **`api/ventes.ts` does not exist**.
> **PostgreSQL tables (per tenant)**: `opportunities`, `interactions`, `crm_activities`, `prospect_qualifications` (B.A.T.), `opportunity_assignations`, `dossiers` (auto `DOS-OPP-…`), `devis` / `devis_lignes` (conversion), `b2b_clients`, `b2b_client_users`, `b2b_demandes`, `b2b_soumissions`, `b2b_contrats`, `b2b_commandes` (+ `b2b_commande_lignes`), `b2b_messages`, `b2b_favoris`, `b2b_notifications`. Shared tables (AI path): `public.ai_prepaid_credits`, `public.ai_usage_tracking`, `public.ai_credit_ledger`, `public.entreprises`.
> **Scope**: this module is the ERP's **sales workstation**. It manages the opportunity pipeline (Kanban), the follow-ups and the tracking calendar, qualification (automatic scoring and the manual B.A.T. grid), an AI assistant that proposes opportunities upon confirmation, and — for administrators only — the **B2B/B2C back-office** (portal client accounts, requests, quotes, contracts, orders, messaging, catalogue). Managing **companies** and **contacts** lives in their own pages and manuals (modules 04 and 05). Converting an opportunity produces a **draft quote** (module 07), not a project directly.

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

### 1.1 Mission

The Sales module centralizes a construction company's **sales cycle**, from first contact to signing:

- track every deal in a **visual pipeline** with 6 statuses;
- **qualify** deals (hot/cold via automatic scoring, or in detail with the B.A.T. grid);
- schedule and complete **follow-ups** (calls, emails, visits, tasks);
- **convert** a winning opportunity into a quote in one click;
- manage, on the administrator side, the entire **B2B back-office** connected to the external client portal.

It is a control station: it **does not replace** the Quotes module (which prices and issues quotes) or the Projects module (which executes the job sites). It feeds the first and prepares the second.

### 1.2 Access from the side menu

Click **Sales** in the side menu ("Management" group). The page opens on the **Pipeline** tab by default.

Two quick links exist:

- `app.constructoai.ca/ventes?open=<id>` opens an opportunity directly in the **Opportunities** tab (used by the "Open" buttons elsewhere in the ERP).
- `app.constructoai.ca/ventes?tab=b2b` (or the legacy `/b2b` route) opens the **B2B/B2C** tab, if you are an administrator.

### 1.3 The two scopes and the 8 tabs

The page brings together **two distinct scopes** under a single tabbed screen:

| # | Tab | Scope | Visible to |
|---|--------|-----------|-------------|
| 1 | **Pipeline** | CRM — drag-and-drop opportunity Kanban | Everyone |
| 2 | **Follow-ups** | CRM — queue of tasks/follow-ups by due date | Everyone |
| 3 | **Opportunities** | CRM — paginated table + detail panel | Everyone |
| 4 | **Calendar** | CRM — monthly view of events | Everyone |
| 5 | **History** | CRM — chronological feed of interactions + activities | Everyone |
| 6 | **Qualification** | CRM — automatic scoring + B.A.T. grid | Everyone |
| 7 | **AI Assistant** | CRM — chat that proposes opportunities upon confirmation | Everyone |
| 8 | **B2B/B2C** | B2B back-office (10 sub-tabs) | **Administrators only** |

The **B2B/B2C** tab is hidden from the tab bar AND not loaded for a non-administrator user (double protection, `VentesPage.tsx:276,313`).

A **red badge** appears on the **Follow-ups** tab with the number of due follow-ups (overdue + today), so you can tell at a glance that there is follow-up work to do.

### 1.4 Permissions and roles

The module applies **three distinct permission levels**. This is an important nuance: reading, writing in the CRM, and administering B2B do not require the same rights.

| Level | What is protected | Rule (`backend`) | Who is allowed |
|--------|--------------------|--------------------|----------------|
| **Read** (CRM + B2B) | View opportunities, pipeline, follow-ups, calendar, statistics, B2B lists | `get_current_user` (`erp_auth.py:475`) | Any valid ERP account. B2B client tokens (external portal) are refused (403). |
| **CRM write** | Create/edit/delete opportunities, interactions, activities, qualification, conversion to quote | `require_crm_write` (`crm.py:40`) | `super_admin`, or a role in `{admin, gestionnaire, contremaitre, user}`. **The `user` role (primary operator) is allowed.** `employee` and `comptable` are **refused** (403). |
| **B2B write** | Create clients/requests/quotes/contracts, approve access, change an order status | `require_tenant_admin_or_role()` (`erp_auth.py:720`) | `is_admin` (re-read server-side, tamper-proof) OR the `admin` role OR `super_admin`. **Stricter**: a non-administrator `user` is **refused**. |

In other words: an employee with the `user` role can work the entire CRM (pipeline, follow-ups, conversion to quote) but **cannot** enter the B2B back-office or administer portal access.

**View-only mode (read-only).** If the tenant's Stripe subscription is past due, the account can switch to **readonly** mode (reads pass, any write returns 403) or **blocked** (401). This control is applied upstream, in `get_current_user` (`erp_auth.py:520-530`, 60 s cache), and covers **all** the module's endpoints, including the AI assistant. A "View-only mode" banner is then displayed in the interface.

### 1.5 What the module does not do (verified in the code)

- **No PDF/CSV export, no printing, no file upload** anywhere in the Sales/B2B module (unlike the Tracking/Gantt module). Documents (PDF quotes, contracts) are generated from the Quotes/Files modules.
- **No automatic project creation**: conversion creates a draft quote; moving quote → project is handled by the Quotes/Projects module.
- **No automatic email or notification** triggered by an opportunity status change.
- **No commission calculation** or salesperson compensation.
- **No employee-to-opportunity assignment interface**: the functions exist on the API side (`crm.ts:353-365`) but are called nowhere in the Sales screen (dead surface, see FAQ).
- **No administrator-side order creation**: the B2B catalogue is **view-only**; orders are created from the **external client portal** (`/b2b-portal`).
- The Sales AI assistant **only creates opportunities** (no other action), and **only after your confirmation**.

---

## 2. Interface

### 2.1 Indicators header and tab bar

At the top of the page, **four indicators** (KPIs) remain visible regardless of the tab (`VentesPage.tsx:248-270`, source `GET /crm/stats`):

| Indicator | Meaning |
|------------|---------------|
| **Opportunities** | Total number of opportunities (with the "in progress" share). |
| **Conversion rate** | `won / (won + lost) × 100`. |
| **Amount won** | Sum of the `montant_estime` of Won opportunities. |
| **Average time** | Average duration (days) between creation and closing of Won opportunities. |

Below the indicators, the **tab bar** (the 8 tabs above). The active tab is highlighted.

### 2.2 Pipeline tab (Kanban)

Drag-and-drop Kanban view. There are **6 statuses** in total, but the display arranges them as follows:

- **4 active, draggable columns**: Prospecting → Qualification → Proposal → Negotiation.
- **2 summary cards** at the top of the screen: **Won** (green) and **Lost** (red). They show the total amount and the number of opportunities in that status, and also serve as **drop targets**: dragging a card onto them marks it won or lost.

> **Important nuance**: Won and Lost are not table columns, they are summary zones. The 6 statuses do exist in the database; only the first 4 have a column.

**Each column header**: colored status badge + opportunity counter + total amount of the stage (source `GET /crm/pipeline`, aggregated by status).

**Creation buttons**:

- **New opportunity** (above the Kanban) opens the blank creation modal.
- **Add** (at the bottom of each column) opens the same modal but **pre-fills the status** of the column.

**The opportunity card** (`PipelineCard`) shows:

- the deal **name** and its number (`OPP-00001`);
- the client **company** name;
- the **estimated amount** and the **probability** (%);
- the expected **closing date**;
- the **B.A.T. score**: colored badge `X/100` + category **A+/A/B/C/D**. If the opportunity comes from a voice-agent pre-qualification (internal status EN_COURS), the card shows "Preliminary B.A.T. / to complete" instead of a score;
- an **eye** button "Open detail";
- **quick-action chips**: "Advance to <next status>" (arrow), "Won", "Lost".

**Drag-and-drop**:

- **Between two columns** = status change. The card moves immediately (optimistic update); if saving fails, it returns to its place (automatic rollback).
- **Within the same column** = **reordering** of the display priority. The order is persisted via `PUT /crm/opportunities/reorder` (payload `{orderedIds}`). If it fails, the previous order is restored.

**Open detail**: double-click a card, or click the eye. The **detail modal** opens (see 2.5).

**Delete**: from the detail view, deletion asks for a native confirmation. If a quote or a project is linked, the warning specifies that it will be **detached** (not deleted).

### 2.3 Opportunities tab (table)

Paginated table view (20 per page) with a side detail panel.

**Command bar**:

- **New opportunity** button;
- **search** field ("Search...") — covers the name, source, notes; a sequence guard prevents a late response from overwriting a more recent search;
- **status** filter ("All statuses" or one of the 6 values).

**Columns (desktop)**: No. · Name (+ source) · Company · Amount · Prob. · Status (badge) · Closing. If the list is empty: "No opportunity found".

**Cards (mobile)**: name, status badge, company, amount, probability, date.

A **total counter** and **pagination** appear at the bottom.

**Detail panel** (on clicking a row, source `GET /crm/opportunities/{id}`): on desktop it occupies roughly 40% on the right; on mobile it becomes full screen with a **Back** button. It contains the **Edit** (pencil), **Delete** and **Close** buttons, as well as: company, contact, amount, probability, closing date, source, notes, "Created on" date, the **Create a quote** and **View file** buttons, the list of **interactions** and the **B.A.T. grid**.

**Edit mode** (Edit button): a form with Name, Status, Estimated amount, Probability (0-100), Closing date, Company (dropdown), Source, Notes → `PUT /crm/opportunities/{id}`. **Save** / **Cancel** buttons.

### 2.4 Opportunity creation modal

Opened by "New opportunity" (or "Add"). Two columns.

**Left column**:

| Field | Note |
|-------|------|
| **Opportunity name** * | Required (e.g. "Dupont kitchen renovation"). |
| Client PO No. | Client's purchase order number, optional. |
| Client (Company) | Dropdown of companies (loaded via `listCompanies`, 100 per page). |
| Client (Person) | Dropdown of contacts. |
| Manual entry | For a client outside the CRM (free-text name). |
| Status | Default Prospecting (or the column's status if opened via "Add"). |
| Priority | Low / Normal / High / Urgent. |

**Right column**:

| Field | Note |
|-------|------|
| Source | Free text (e.g. "Website, Referral, Trade show..."). |
| Quote deadline | Optional. |
| Planned work start / end | Optional. |
| Estimated amount ($) | 0 to 1,000,000,000. |
| Probability | Slider 0-100, step of 5. |
| Expected closing date | Optional. |
| Description | Text area. |
| Notes | Text area. |

A "* Required fields" note and the **Cancel** / **Save** buttons (→ `POST /crm/opportunities`) close the modal.

On creation, the system generates the number `OPP-00001`, along with a **client file** `DOS-OPP-…` of type CLIENT (best-effort, see module 06).

### 2.5 Detail modal (View / Edit / History / B.A.T.)

Accessible from the Pipeline (double-click/eye) and from Opportunities (row click). Large size. It has several zones:

**a) View** — number + status badge, name, company, contact, estimated amount, probability bar, closing date, source, notes. Two action buttons:

- **Create a quote** → `POST /crm/opportunities/{id}/create-devis`, then redirect to `/devis?open=<devisId>`. If the opportunity is already converted, the server replies "already converted to quote #X" and that detail is displayed.
- **View file** (if a file is linked) → `/dossiers?open=…`.

**b) Edit** — same fields as creation (2-column form). Edit / Delete buttons.

**c) Interactions + activities feed** — a **merged** chronological feed of the opportunity's interactions and activities. Two links at the top, **Interaction** and **Activity**, open an **inline form**:

| Field | Detail |
|-------|--------|
| Type | Dropdown (interaction or activity types). |
| Date | Event date. |
| Summary / Subject | Text. |
| Add | → `POST /crm/interactions` or `POST /crm/activities`. |

Each feed item shows: type badge + title + subtext + date + Inter./Act. badge + activity status.

**d) B.A.T. grid** — the manual qualification grid (see 2.9) is embedded directly in the modal.

**Deletion**: the Delete button triggers a native confirmation. Deletion **detaches** (does not delete) the linked quote/project; it cascade-deletes the opportunity's interactions, activities, assignments and qualifications.

### 2.6 Follow-ups tab

A tracking queue of your tasks and follow-ups, sorted by due date. Subtitle: "Your follow-ups and tasks to do, by due date." (source `GET /crm/relances?horizonDays=7`).

**Three buckets**:

| Bucket | Color | Content |
|---------|---------|---------|
| **Overdue** | Red | Follow-ups whose due date has passed. |
| **Today** | Blue | Today's follow-ups. |
| **Upcoming (7 days)** | Grey | Follow-ups for the next 7 days. |

Each **follow-up card** shows: activity-type badge + date + subject + linked opportunity/company + opportunity status badge. Three actions:

- **Done** → marks the activity completed (`PATCH /crm/activities/{id}`, status TERMINE);
- **Postpone** → an inline date field appears (Confirm / Cancel) to move the due date;
- **Open** → opens the opportunity (`/ventes?open=<id>`).

If nothing is scheduled: "No follow-up scheduled" with a tip.

### 2.7 Calendar tab

Monthly grid (Monday-to-Sunday week). **Previous month / next month** navigation + **Today** button (source `GET /crm/calendar?year&month`).

**Three event types** with a colored legend:

| Type | Color | Source |
|------|---------|--------|
| Interaction | Blue | Interaction date. |
| Activity | Purple | Activity date. |
| Opp. closing | Orange | An opportunity's expected closing date. |

Each **day cell** shows up to 3 events, then "+N more". Clicking a day opens a **side panel** listing all events for that day (type + title + sub-type badge).

### 2.8 History tab

Chronological feed of **all** the tenant's interactions and activities (source `GET /crm/timeline?limit=50`, max 200). A **company filter** (dropdown) lets you narrow the display.

Each card: icon by type/sub-type, Interaction/Activity badge, sub-type, company, title, date. A total counter appears at the top. If empty: "No event in the history".

> Interactions and activities are **entered** only from an opportunity's detail view (2.5). The History and Calendar are for viewing only.

### 2.9 Qualification tab and B.A.T. grid

Two qualification mechanisms coexist: **automatic scoring** (this tab) and the **manual B.A.T. grid** (embedded in the detail modals).

**Automatic scoring** (source `GET /crm/qualification`) — a score from 0 to 100 computed on the fly on **open** opportunities (neither WON nor LOST):

**Three summary cards** at the top, with counters: **Hot** (HOT, red), **Warm** (WARM, orange), **Cold** (COLD, blue).

**Table**: Opportunity · Score (bar) · Category (badge) · Amount · Probability · **Details** (bullets explaining the score: amount, linked company, linked contact, probability, interactions, source, recent, inactive). Clicking a row opens the opportunity. If empty: "No open opportunity to qualify".

**B.A.T. Scoring Grid (manual qualification).** A component embedded in the detail modals (Pipeline and Opportunities). Score out of 100, split into **4 collapsible sections** of 25 points each, **13 questions** in total (radio buttons):

| Section | Points | Questions | Icon |
|---------|--------|-----------|-------|
| **A. Budget** | 25 | A1 (10), A2 (10), A3 (5) | `DollarSign` |
| **B. Authority** | 25 | B1 (10), B2 (10), B3 (5) | `Users` |
| **C. Timing** | 25 | C1 (10), C2 (10), C3 (5) | `Clock` |
| **D. Compatibility** | 25 | D1 (10), D2 (5), D3 (5), D4 (5) | `Target` |

The total score gives a **category** and a **recommended action** (see 4.7). A **Qualification notes** field (optional) and the **Save qualification** button (→ `POST /crm/qualification/bat`) finalize the entry.

> **The server recalculates** the total score and the category from the submitted answers; it does not trust the values computed by the browser. The aggregated scores shown on the Kanban cards come from `GET /crm/qualification/bat/all`.

### 2.10 AI Assistant tab — Sales

A dedicated chat. Title "AI Assistant — Sales", subtitle "Analyzes your pipeline and creates opportunities upon confirmation." It uses `api/ventesAi.ts` (`POST /ventes/ai/chat` and `POST /ventes/ai/confirm-action`).

**Two-step operation**:

1. You ask a question or make a request. The AI **reads your real data** (opportunities, companies, contacts) and answers. If it proposes to create an opportunity, it shows a **proposal card** (title + field preview) marked "Awaiting confirmation".
2. You click **Confirm** (the opportunity is actually created) or **Cancel** (nothing happens). **Only clicking Confirm writes to the database.**

Three starter examples are offered. Input is sent with **Enter**; locks prevent double-sending. Each message shows its metadata ("Sales" profile, tokens, cost in USD, duration).

> The Sales assistant **only creates opportunities** — no other action. It applies the same rights as the CRM: the confirmed creation re-checks `require_crm_write` server-side, so an `employee`/`comptable` cannot create an opportunity even via the AI.

### 2.11 B2B/B2C tab (administrators only)

Title "B2B Space". This is the **back-office** of the external client portal (`/b2b-portal`). It has **10 sub-tabs**:

#### a) Dashboard

4 indicators (Active clients, New requests, Active contracts, Contract value) + "Requests by status" + "Summary" (totals) + "Recent activity" (source `GET /b2b/stats`). The "unread messages" counter counts messages **written by a client** (unread).

#### b) Access requests

Approval flow for portal sign-ups. Two sub-tabs, **Pending** / **Approved** (with counters). Table: Company · Contact · Email · Phone · City · Date · Actions.

- For a **pending** account: **Approve** (activates the account and reactivates the client company) or **Reject** (deletes the request).
- For an **approved** account: **Deactivate** (revokes access).

Endpoints: `PUT /b2b/client-users/{id}/approve | reject | deactivate`. Each action asks for confirmation.

#### c) Clients

Search + **New client** button. Table: Name · Contact · Email · Phone · **CRM company** · Status · Actions.

- **Link to a CRM company** (`Link2` icon): opens a company-search modal. This link feeds the **portal tracking** (the client sees their quotes/projects). The link is **always set manually** by the administrator — **never** auto-linked by email (protection against impersonation).
- **Deactivate** (trash) → `DELETE /b2b/clients/{id}` (logical, cascading deactivation of access).

Creation modal: Company name * · Contact name · Email · Phone · City · Sector (→ `POST /b2b/clients`).

#### d) Requests

Status filter (All / New / In progress / Submitted / Accepted / Declined / Cancelled) + **New request** button. Table: Title · Client · Budget · Status · Date. Clicking opens a **detail panel** (client, category, budget, priority, job site, description) with a **Create quote** button and the list of linked quotes.

Creation modal: Client * · Title * · Description · Category · Estimated budget · Deadline · Priority · Job site address/city (→ `POST /b2b/demandes`).
Quote modal: **Pre-tax amount** * · Description · Lead time (days) · Validity (days) · Payment terms · Warranties (→ `POST /b2b/soumissions`).

#### e) Quotes

Status filter (All / Draft / Submitted / Under evaluation / Accepted / Declined). Table: Request · Client · Amount · Lead time · Status · Actions.

- **Accept** (✓) → `PUT /b2b/soumissions/{id}/accepter`: marks the quote accepted, **automatically declines the other quotes** for the same request, and **creates an active contract**.
- **Decline** (✗) → `PUT /b2b/soumissions/{id}/refuser`.

These actions are unavailable if the quote is already Accepted / Declined / Expired.

#### f) Contracts

Status filter (All / Draft / Active / Completed / Cancelled). Table: Number · Title · Client · Amount · **Progress** (bar) · Status · Actions. The **Edit** button opens a modal: Status · Progress (%) · Amount paid · Internal notes (→ `PUT /b2b/contrats/{id}`).

#### g) Orders

Status filter (7 values). **List view**: Number · Total (incl. tax) · City · Status · Date · Action. **Detail view**: product lines, subtotal, GST (Goods and Services Tax), QST (Quebec Sales Tax), Total (incl. tax).

- **Advance to next status**: EN_ATTENTE → CONFIRMEE → EN_PREPARATION → EXPEDIEE → LIVREE (`PUT /b2b/commandes/{id}/statut`).
- **Cancel**: asks for confirmation ("reserved stock replenished"). Setting an order to ANNULEE **restores the stock**; ANNULEE is a **terminal** status.

> Orders are **created** from the external client portal, not here. This sub-tab only tracks and advances existing orders.

#### h) Messages

Two columns: the list of requests on the left, the conversation thread on the right. Bubbles distinguish **You** (the supplier) from **Client**. Input is sent with **Enter** (→ `POST /b2b/messages`); messages are marked read as soon as the thread is opened.

#### i) Catalogue

**View-only** (the old administrator cart has been removed). Search + category filter. Grid of product cards: name, code, description, category, price/unit, stock ("X in stock" / "Out of stock"). Source `GET /b2b/catalogue`.

#### j) AI Assistant — B2B Management

A **read-only** chat (no writes, no proposals). Title "AI Assistant — B2B Management". It uses `api/b2bAi.ts`. It **does not access** client accounts (passwords) or the **content of messages**. Three starter examples are offered.

---

## 3. Step-by-step workflows

### 3.1 Create an opportunity

1. **Pipeline** or **Opportunities** tab → **New opportunity** (or **Add** at the bottom of a column to set the status).
2. Enter at least the **Name** (required). Fill in the company/contact, the estimated amount, the probability, the source and the priority.
3. **Save**. The system creates opportunity `OPP-00001`, along with a **client file** `DOS-OPP-…` (best-effort).

### 3.2 Advance a deal through the pipeline

**By drag-and-drop**: in the Kanban, drag the card to another column (new status) or to the Won/Lost summary cards. The card moves right away; if saving fails, it returns to its place.

**By quick button**: on the card, click "Advance to <status>", "Won" or "Lost".

Transitions are **unrestricted**: any status can lead to any other. No rule requires qualifying before advancing.

### 3.3 Reorder the deals in a column

In the Kanban, drag a card **above/below** another **within the same column**. The new order is saved (`PUT /crm/opportunities/reorder`) and serves as display priority.

### 3.4 Qualify a deal

**Automatic**: open the **Qualification** tab. The score (Hot/Warm/Cold) and its reasons compute themselves from data already entered.

**Detailed (B.A.T.)**: open an opportunity's detail, expand the **B.A.T. Scoring Grid**, answer the 13 questions (4 sections), add notes, **Save qualification**. The B.A.T. score and its A+/A/B/C/D category then propagate to the Kanban card.

### 3.5 Log an interaction or an activity

1. Open the detail of the relevant opportunity.
2. In the **feed**, click **Interaction** (past event: call, email...) or **Activity** (planned task: follow-up, visit...).
3. Choose the **Type**, the **Date**, enter the **Summary/Subject**, then **Add**.

An **activity** is created with status PLANIFIE and feeds the **Follow-ups** and the **Calendar**.

### 3.6 Handle your follow-ups

1. **Follow-ups** tab (the red badge shows the overdue + today count).
2. For each card: **Done** (completed), **Postpone** (choose a new date), or **Open** (go to the opportunity).

### 3.7 Convert an opportunity into a quote

1. Open the opportunity's detail (Pipeline or Opportunities).
2. Click **Create a quote** (→ `POST /crm/opportunities/{id}/create-devis`).
3. You are redirected to the created quote (`/devis?open=<id>`), in **Draft** status, **Detailed** type.

What the server does (`create_devis_from_opportunity`, `crm.py:1232`):

- it **locks** the opportunity (`FOR UPDATE`): two simultaneous clicks create only one quote; if a quote already exists, it replies **400 "already converted to quote #X"**;
- it applies the **Administration 3% / Contingencies 12% / Profit 15%** cascade on the estimated amount (the 15% profit is **fixed**, consistent with the ERP's cost-plus model);
- it computes the **taxes according to the tenant's configuration** (`resolve_document_tax_config`) — in Quebec, GST 5% and QST 9.975%, but a tenant configured elsewhere will have its own rates;
- it numbers the quote **`DEV-{year}-{id:03d}`**, seeds an estimate line into it (quantity 1, unit "global") if the amount is positive;
- it moves the opportunity to **PROPOSITION** status and links it to the quote, then links the file to the quote.

> Conversion creates a **quote**, not a project. Moving quote → project happens in the Quotes/Projects module upon acceptance.

### 3.8 Delete an opportunity

1. Opportunity detail → **Delete** → confirm.
2. The server cascade-deletes the **interactions, activities, assignments, qualifications**; it **detaches** (sets to NULL) the linked quote, project and emails (the history is preserved); it deletes the auto-created file **only if it is empty**.

### 3.9 Use the Sales AI assistant

1. **AI Assistant** tab. Make your request (e.g. "Create an opportunity for the renovation at 12 rue Principale, budget $80,000").
2. The AI reads your data and, if relevant, shows a **proposal card**.
3. Check the fields, then **Confirm** (actual creation) or **Cancel**.

### 3.10 B2B — Approve portal access

1. A client signs up on the **external portal** (`/b2b-portal`); their account appears in **B2B → Access requests → Pending**.
2. **Approve**: the account becomes active and the client company is reactivated. (Or **Reject** to delete the request.)
3. Optional but recommended: in **Clients**, **link** the client to a **CRM company** so they can track their quotes/projects in the portal.

### 3.11 B2B — From request to contract

1. **Requests** → **New request** (client, title, budget, job site...).
2. In the request's detail → **Create quote** (Pre-tax amount, lead time, validity...). The request moves from NOUVELLE to EN_COURS.
3. **Quotes** → **Accept**: the quote moves to ACCEPTEE, the **other quotes** for the request are declined, and an active **contract** `CTR-YYYYMM-0001` is generated.
4. **Contracts** → **Edit** to track progress (%) and amounts paid.

### 3.12 B2B — Track and cancel an order

1. **Orders**: open the order (created via the portal).
2. **Advance** to the next status (EN_ATTENTE → ... → LIVREE) as you go.
3. **Cancel** if necessary: the **reserved stock is replenished** and the order becomes ANNULEE (final).

---

## 4. Reference

### 4.1 CRM endpoints (`/api/erp/v1/crm`)

| Method | Path | Guard | Reference |
|---------|--------|-------|-----------|
| GET | `/crm/opportunities` | read | `crm.py:331` |
| GET | `/crm/opportunities/{id}` | read | `crm.py:442` |
| POST | `/crm/opportunities` | `require_crm_write` | `crm.py:526` |
| PUT | `/crm/opportunities/reorder` | `require_crm_write` | `crm.py:719` |
| PUT | `/crm/opportunities/{id}` | `require_crm_write` | `crm.py:787` |
| DELETE | `/crm/opportunities/{id}` | `require_crm_write` | `crm.py:876` |
| POST | `/crm/opportunities/{id}/create-devis` | `require_crm_write` | `crm.py:1232` |
| GET | `/crm/opportunities/{id}/assignations` | read | `crm.py:2433` |
| POST | `/crm/opportunities/{id}/assignations` | `require_crm_write` | `crm.py:2474` |
| DELETE | `/crm/opportunities/{id}/assignations/{aid}` | `require_crm_write` | `crm.py:2534` |
| GET | `/crm/interactions` | read | `crm.py:1037` |
| POST | `/crm/interactions` | `require_crm_write` | `crm.py:1117` |
| GET | `/crm/activities` | read | `crm.py:1525` |
| POST | `/crm/activities` | `require_crm_write` | `crm.py:1582` |
| PATCH | `/crm/activities/{id}` | `require_crm_write` | `crm.py:1644` |
| GET | `/crm/pipeline` | read | `crm.py:1178` |
| GET | `/crm/stats` | read | `crm.py:1416` |
| GET | `/crm/relances` | read | `crm.py:1718` |
| GET | `/crm/calendar` | read | `crm.py:1824` |
| GET | `/crm/timeline` | read | `crm.py:1928` |
| GET | `/crm/qualification` | read | `crm.py:2023` |
| GET | `/crm/qualification/bat/all` | read | `crm.py:2213` |
| GET | `/crm/qualification/bat/{id}` | read | `crm.py:2259` |
| POST | `/crm/qualification/bat` | `require_crm_write` | `crm.py:2298` |

### 4.2 Sales AI Assistant endpoints (`/api/erp/v1/ventes/ai`)

| Method | Path | Effect | Reference |
|---------|--------|-------|-----------|
| POST | `/ventes/ai/chat` | Reads the data, proposes (creates nothing) | `ventes_ai.py:302` |
| POST | `/ventes/ai/confirm-action` | Creates the confirmed opportunity (re-checks `require_crm_write`) | `ventes_ai.py:442` |

### 4.3 B2B back-office endpoints (`/api/erp/v1/b2b`) — main ones

| Method | Path | Guard | Reference |
|---------|--------|-------|-----------|
| GET | `/b2b/stats` | read | `b2b.py:722` |
| GET | `/b2b/clients` · `/b2b/clients/{id}` | read | `b2b.py:821` · `881` |
| POST | `/b2b/clients` | admin | `b2b.py:922` |
| PUT | `/b2b/clients/{id}` | admin | `b2b.py:965` |
| DELETE | `/b2b/clients/{id}` | admin | `b2b.py:1027` |
| POST | `/b2b/client-users` | admin | `b2b.py:1111` |
| GET | `/b2b/client-users` | read | `b2b.py:1166` |
| PUT | `/b2b/client-users/{id}/approve` | admin | `b2b.py:1213` |
| PUT | `/b2b/client-users/{id}/reject` | admin | `b2b.py:1308` |
| PUT | `/b2b/client-users/{id}/deactivate` | admin | `b2b.py:1370` |
| GET | `/b2b/demandes` · `/b2b/demandes/{id}` | read | `b2b.py:1435` · `1504` |
| POST | `/b2b/demandes` | admin | `b2b.py:1558` |
| PUT | `/b2b/demandes/{id}` | admin | `b2b.py:1608` |
| GET | `/b2b/soumissions` | read | `b2b.py:1670` |
| POST | `/b2b/soumissions` | admin | `b2b.py:1731` |
| PUT | `/b2b/soumissions/{id}` | admin | `b2b.py:1808` |
| PUT | `/b2b/soumissions/{id}/accepter` | admin | `b2b.py:1866` |
| PUT | `/b2b/soumissions/{id}/refuser` | admin | `b2b.py:1978` |
| GET | `/b2b/contrats` · `/b2b/contrats/{id}` | read | `b2b.py:2028` · `2084` |
| PUT | `/b2b/contrats/{id}` | admin | `b2b.py:2128` |
| GET | `/b2b/commandes` · `/b2b/commandes/{id}` | read | `b2b.py:2190` · `2247` |
| PUT | `/b2b/commandes/{id}/statut` | admin | `b2b.py:2290` |
| GET | `/b2b/catalogue` | read | `b2b.py:2451` |
| GET/POST/DELETE | `/b2b/favoris[/{produit_id}]` | read | `b2b.py:2544` · `2586` · `2635` |
| GET | `/b2b/messages` | read | `b2b.py:2679` |
| POST | `/b2b/messages` | read (get_current_user) | `b2b.py:2730` |
| PUT | `/b2b/messages/read` | read | `b2b.py:2773` |
| GET | `/b2b/notifications` | read | `b2b.py:2829` |
| PUT | `/b2b/notifications/{id}/read` | read | `b2b.py:2877` |
| GET | `/b2b/categories` | **PUBLIC (no auth)** | `b2b.py:3014` |
| POST | `/b2b/ai/chat` | read (B2B assistant) | `b2b_ai.py:221` |

> `GET /b2b/categories` is the **only unauthenticated endpoint** in the module: it returns a static dictionary of ~140 Quebec construction categories, with no tenant data whatsoever.

### 4.4 Opportunity statuses (`OPPORTUNITY_STATUSES`, `crm.py:51`)

| Value (database) | Displayed label | Color | Kanban column? |
|---------------|-----------------|---------|------------------|
| PROSPECTION | Prospecting | Blue | Column 1 |
| QUALIFICATION | Qualification | Yellow | Column 2 |
| PROPOSITION | Proposal | Purple | Column 3 |
| NEGOCIATION | Negotiation | Orange | Column 4 |
| GAGNE | Won | Green | Summary card (drop target) |
| PERDU | Lost | Red | Summary card (drop target) |

The values are stored in uppercase ASCII; an idempotent migration (`_ensure_opportunities_statut_check`) resynchronizes the constraint for older tenants. The displayed labels come from `crm.json` (namespace `ventes.statusLabels.*`).

### 4.5 Interaction / activity types

| Constant (`crm.py`) | Values |
|----------------------|---------|
| `INTERACTION_TYPES` (`:52`) | APPEL, EMAIL, REUNION, VISITE, NOTE |
| `ACTIVITY_TYPES` (`:271`) | **TACHE**, APPEL, EMAIL, REUNION, VISITE, RELANCE, NOTE |
| `ACTIVITY_STATUSES` (`:264`) | PLANIFIE, TERMINE, ANNULE |

> `TACHE` is a **valid** activity value (`crm.py:271`). The old manual reported a "TACHE rejected" bug: it is **fixed**.

### 4.6 Opportunity priorities

Low · Normal · High · Urgent (`crm.json` labels). Default: Normal.

### 4.7 Qualification scales

**Automatic scoring** (`GET /crm/qualification`, on open opportunities):

| Criterion | Points |
|---------|--------|
| `montant_estime > 0` | +20 |
| Company filled in | +15 |
| Contact filled in | +10 |
| Probability > 50 | +20 |
| At least 1 interaction | +15 |
| Source filled in | +10 |
| Updated < 30 days | +10 |

Categories: **HOT** ≥ 70 · **WARM** ≥ 40 · **COLD** below.

**Manual B.A.T. grid** (the server recalculates the total and the category):

| Total score | Category | Color | Recommended action |
|-------------|-----------|---------|--------------------|
| ≥ 90 | A+ | Green | Top priority — visit within 48-72 h |
| 75-89 | A | Green | High priority |
| 50-74 | B | Yellow | Potential — to explore further |
| 25-49 | C | Orange | Warm — keep in touch |
| < 25 | D | Grey | Cold — not a priority |

> Two **distinct** systems: automatic scoring (3 levels HOT/WARM/COLD) and the B.A.T. grid (5 categories A+/A/B/C/D). They can diverge.

### 4.8 Conversion to quote — cascade and taxes

| Element | Value |
|---------|--------|
| Administration | amount × 3% |
| Contingencies | amount × 12% |
| Profit | amount × 15% (**fixed**, cost-plus model) |
| Taxes | Per the tenant's configuration (`resolve_document_tax_config`) — QC: GST 5%, QST 9.975% |
| Clamped amount | `max(0, min(amount, 1,000,000,000))` |
| Quote created | Draft status, Detailed type, number `DEV-{year}-{id:03d}`, 1 initial estimate line |
| Effect on the opportunity | moves to PROPOSITION + link to the quote |
| Protection | `FOR UPDATE` + re-check → 400 if already converted |

### 4.9 Automatic numbering

| Entity | Format | Example |
|--------|--------|---------|
| Opportunity | `OPP-{id:05d}` | OPP-00042 |
| Auto-created file | `DOS-OPP-…` | DOS-OPP-00042 |
| Converted quote | `DEV-{year}-{id:03d}` | DEV-2026-137 |
| B2B contract | `CTR-{YYYYMM}-{id:04d}` | CTR-202607-0009 |

All numbers are generated by INSERT-RETURNING-id (never by COUNT+1), guaranteeing uniqueness even in the case of simultaneous clicks.

### 4.10 B2B statuses

| Constant (`b2b.py`) | Values |
|----------------------|---------|
| `DEMANDE_STATUTS` (`:31`) | NOUVELLE, EN_COURS, SOUMISE, ACCEPTEE, REFUSEE, ANNULEE |
| `SOUMISSION_STATUTS` (`:34`) | BROUILLON, SOUMISE, EN_EVALUATION, ACCEPTEE, REFUSEE, EXPIREE |
| `CONTRAT_STATUTS` (`:32`) | BROUILLON, ACTIF, EN_COURS, TERMINE, ANNULE, SUSPENDU |
| `COMMANDE_STATUTS` (`:33`) | EN_ATTENTE, CONFIRMEE, EN_PREPARATION, EXPEDIEE, LIVREE, ANNULEE |

> The `b2b_commandes` table is **shared** with the legacy Streamlit application (lowercase statuses). The module handles casing transparently (`upper(...)` and CHECK-constraint repair). Any change to the statuses must remain compatible with both applications.

### 4.11 B2B taxes

GST 5% (`TPS_RATE = 0.05`) and QST 9.975% (`TVQ_RATE = 0.09975`), hard-coded in `b2b.py:28-29`. For a quote: if the **Pre-tax amount** is provided, GST and QST are added on top; otherwise the pre-tax amount is derived from the tax-included total. Expiry date = today's date + validity (default 30 days).

### 4.12 Limits, bounds and defenses

| Element | Value |
|---------|--------|
| Opportunity pagination | `page` ≥ 1, `per_page` 1-200 (20 by default on screen) |
| Search | LIKE with `% _ \` escaping, truncated to 100 characters |
| Estimated amount | 0 to 1,000,000,000 |
| Probability | 0 to 100 |
| Reordering | up to 10,000 identifiers per call |
| B.A.T. grid | axes 0-25, total 0-100, notes ≤ 10,000 characters |
| Follow-ups | horizon 0-90 days (7 by default) |
| Calendar | year 1900-2200, month 1-12 |
| History (timeline) | limit ≤ 200 |
| Assignment | UNIQUE (opportunity, employee) → 409 if duplicate |

### 4.13 AI Assistant — model, cost, rate limits

| Element | Sales | B2B |
|---------|--------|-----|
| Model | `claude-sonnet-4-6` | `claude-sonnet-4-6` |
| Write | Yes, opportunity **upon confirmation** | **No** (read-only) |
| Read tools | `recherche_bd` — tables `{opportunities, companies, contacts}` | B2B tables (excluding accounts/messages) + `projects`, `companies` |
| Cost | (input × 0.003 + output × 0.015) / 1000 × **1.30** (30% margin) | same |
| Charged to | `public.ai_prepaid_credits` (auto Stripe top-up of $10 below the threshold) | same |
| Rate limit (per IP) | chat 20/min, confirmation 30/min | chat 20/min |

The real credit check is `_check_credits` (fail-closed: blocks on error). Technical note: the Sales chat charges **without an idempotency key** — a network retry can double-charge (minor; it is a chat, not a money mutation).

### 4.14 Shortcuts

| Action | Gesture |
|--------|-------|
| Open a card's detail | Double-click (or eye button) |
| Change status | Drag the card to another column / summary card |
| Reorder | Drag the card within the same column |
| Send a chat message | Enter |
| Open B2B directly | `/ventes?tab=b2b` (admin) |
| Open an opportunity directly | `/ventes?open=<id>` |

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

| Module | Link |
|--------|------|
| **04 — Companies** | Opportunities and B2B clients attach to a company. Creating/managing companies lives in its own module. |
| **05 — Contacts** | An opportunity's "Client (Person)" field points to a contact. Contact management: module 04. |
| **07 — Files** | Each opportunity generates a `DOS-OPP-…` file; conversion links the file to the quote. |
| **08 — Quotes** | "Create a quote" produces a `DEV-…` draft quote and redirects to its editor. |
| **09 — Projects** | A project can reference the opportunity (`projects.opportunity_id`). Promoting quote → project happens on the Quotes/Projects side. |
| **10 — Store / Inventory** | The B2B catalogue reads the `produits`; cancelling a B2B order **restores the stock**. |
| **External B2B portal** (`/b2b-portal`) | Clients sign up there, send requests and **place orders**. This module's B2B back-office is the internal counterpart. |
| **25 — AI Assistant** | The Sales and B2B assistants share the same AI credit engine (`ai_prepaid_credits`) and Stripe billing. |
| **28 — Configuration** | The tenant's tax configuration drives the taxes of the conversion to quote. The Stripe subscription state drives view-only mode. |

### 5.2 FAQ

**Q1. Why don't I see the B2B/B2C tab?**
It is reserved for administrators (role `admin`, `is_admin`, or super-administrator). An account with the `user` role has access to the entire CRM but not the B2B back-office.

**Q2. Can an employee work in the pipeline?**
Yes, if their role is `user`, `gestionnaire`, `contremaitre`, `admin` or super-administrator. The `employee` and `comptable` roles are **read-only** on the CRM side (any write returns 403).

**Q3. Why does the pipeline have only 4 columns when there are 6 statuses?**
Won and Lost are not columns, but two **summary cards** at the top (amount + count), which also serve as drop targets. The 4 columns cover the "in progress" work.

**Q4. Does conversion create a project?**
No: it creates a **quote** in Draft status. The process is Opportunity → Quote → Acceptance → Project; the last step is handled by the Quotes/Projects module.

**Q5. Are the 3 / 12 / 15% margins configurable?**
The **15% profit is fixed** (the ERP's cost-plus model), as are Administration 3% and Contingencies 12%. To adjust a specific case, edit the quote after it is created (Quotes module).

**Q6. Are the conversion taxes always 5% / 9.975%?**
They follow the **tenant's configuration** (`resolve_document_tax_config`). In Quebec, they are indeed GST 5% and QST 9.975%, but a tenant configured elsewhere (e.g. another province, United States) will have its own rates. This is an improvement over the old hard-coded QC rate.

**Q7. Can I export the pipeline to PDF or CSV?**
No. The Sales/B2B module offers **no export, no printing, no upload**. Documents are produced from Quotes/Files.

**Q8. Can an employee be assigned to an opportunity?**
Not from the interface: the functions exist on the API side (`crm.ts:353-365`, endpoints `crm.py:2433/2474/2534`) but **are wired to no screen** in the module. There is therefore no assignment interface.

**Q9. Can the AI assistant edit or delete data?**
No. The **Sales** assistant can only **propose an opportunity**, created only after your confirmation (and it re-checks your write rights server-side). The **B2B** assistant is **read-only** and accesses neither client accounts nor message content.

**Q10. What happens if I double-click "Create a quote"?**
Nothing dangerous: the opportunity is locked (`FOR UPDATE`) during the conversion, so a single quote is created. The second click receives "already converted to quote #X".

**Q11. Difference between an interaction and an activity?**
An **interaction** is a **past** event (call received, email sent). An **activity** is a **planned task** (follow-up, visit, task), with a PLANIFIE/TERMINE/ANNULE status; activities are what feed the Follow-ups and the Calendar.

**Q12. Is the B.A.T. grid mandatory to advance a deal?**
No. No rule ties the status to the score. Qualification is a decision-support tool, not a lock.

**Q13. Why does the B.A.T. score I computed sometimes differ from the one displayed?**
The **server always recalculates** the total and the category from your answers; that value is authoritative.

**Q14. How does a client get portal access?**
They sign up on the external portal; their request appears in **B2B → Access requests → Pending**. An administrator **approves** it (which activates the account and the company) or **rejects** it.

**Q15. Why link a B2B client to a CRM company?**
The link lets the client **track their quotes and projects** in the portal. It is **always set manually** by an administrator — never automatically by email, to prevent any impersonation.

**Q16. Why doesn't the B2B catalogue allow ordering?**
The administrator cart has been **removed**; the catalogue is now view-only. Orders are created on the client side, in the external portal. The administrator **tracks** them and **advances** them (or cancels them) from the Orders sub-tab.

**Q17. What happens to stock when I cancel a B2B order?**
The reserved stock is **replenished** (an inbound movement, reason ANNULATION), and the order moves to ANNULEE, which is a **terminal** status.

**Q18. My account is in "View-only mode" — why?**
The tenant's Stripe subscription is not up to date. In readonly, you can view everything but no write is accepted. Bring the subscription current (Configuration module) to return to write access.

**Q19. Does the AI assistant bill me?**
Each exchange consumes prepaid AI credits (the model's real cost + 30% margin), charged to `ai_prepaid_credits`. An automatic Stripe top-up of $10 triggers below the threshold. Super-administrators and exempted accounts are not charged.

**Q20. Where are companies and contacts managed?**
In their dedicated modules: **04 — Companies** and **05 — Contacts**. This module only **references** them in the dropdowns of opportunities and B2B clients.

---

## 6. Summary

- **One screen, two scopes**: the CRM (pipeline, follow-ups, calendar, history, qualification, AI assistant) for everyone, and the **B2B back-office** (10 sub-tabs) reserved for administrators.
- **8 tabs**: Pipeline · Follow-ups · Opportunities · Calendar · History · Qualification · AI Assistant · B2B/B2C.
- **6 opportunity statuses**, but **4 draggable** Kanban columns; Won/Lost are summary cards and drop targets.
- **Three permission levels**: read (any ERP account), CRM write (`require_crm_write`, includes the `user` role), B2B write (`require_tenant_admin_or_role`, administrators only). The Stripe **view-only mode** can put the entire module in read-only.
- **Two qualifications**: automatic scoring (HOT/WARM/COLD) and the manual B.A.T. grid (A+/A/B/C/D, recalculated server-side).
- **One-click conversion to quote**: 3% / 12% / **15% fixed profit** cascade + taxes **per the tenant's configuration**, `DEV-{year}-{id:03d}` draft quote, opportunity → PROPOSITION, anti-double-conversion locking.
- **Numbering**: `OPP-{id:05d}`, `DOS-OPP-…`, `DEV-{year}-{id:03d}`, `CTR-{YYYYMM}-{id:04d}`.
- **Two AI assistants**: Sales (proposes an opportunity upon confirmation) and B2B (read-only). Model `claude-sonnet-4-6`, prepaid credits + 30% margin.
- **B2B**: portal access approval, request → quote → contract (the contract is created upon acceptance), order tracking (cancellation = stock restoration), messaging, view-only catalogue.
- **What the module does not do**: no export/printing/upload, no project creation, no automatic email, no commission calculation, no employee assignment (dead API surface), no administrator-side order creation.
- **Single public endpoint**: `GET /b2b/categories` (static category dictionary, no tenant data).

---

*Verified sources (2026-07)*: `backend/routers/crm.py` (2563 lines) · `backend/routers/ventes_ai.py` (479 lines) · `backend/routers/b2b.py` (3017 lines) · `backend/routers/b2b_ai.py` (340 lines) · `frontend/src/pages/VentesPage.tsx` (2914 lines) · `frontend/src/pages/B2bPage.tsx` (1411 lines) · `frontend/src/components/crm/BATQualificationForm.tsx` · `frontend/src/components/ventes/VentesAssistantTab.tsx` · `frontend/src/components/b2b/B2bAssistantTab.tsx` · `frontend/src/api/{crm.ts, ventesAi.ts, b2b.ts, b2bAi.ts}` · i18n `crm.json` (namespace `ventes.*`), `b2b.json`, `b2bAssistant.json`.

*Related manuals*: 04 — Companies · 05 — Contacts · 07 — Files · 08 — Quotes · 09 — Projects · 10 — Store / Inventory · 25 — AI Assistant · 28 — Configuration.

*Constructo AI ERP Manual — Module 05 Sales (CRM, opportunities, pipeline, B2B back-office) — v3.0 verified — 2026-07*
