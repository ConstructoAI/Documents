# Module 07 — Quotes and estimates (manual, AI, import)

> **Version**: 3.0 (complete overhaul verified against the source code, 2026-07)
> **Frontend route**: `/devis` (sidebar menu "Quotes", "Sales" group). The on-screen title is **"Quotes"** (`devis.title`), even though the URL and the internal code say "devis". Public client-validation page: `/devis/public/:token` (no authentication).
> **API prefix**: `/api/erp/v1`. Three routers are mounted under this prefix: `/devis/manuel-template` (Manual template catalog), `/devis` (the core of the module) and `/devis/ai` (read-only assistant). Mount order matters: `manuel-template` is mounted **before** `devis`, otherwise the dynamic route `/{devis_id}` would capture the word "manuel-template" (`erp_api.py:1000-1010`).
> **Reference code (backend)**: `backend/routers/devis.py` (13,085 lines — CRUD, lines, AI, sending, public page, project conversion) · `backend/routers/devis_ai.py` (344 lines — **read-only** Quotes assistant) · `backend/routers/devis_manuel_template.py` (663 lines — custom sections and lines of the Manual template). Total: **66 endpoints** (57 + 1 + 8).
> **Reference code (frontend)**: `frontend/src/pages/DevisPage.tsx` (3,052 lines — list, detail panel, modals) · `components/devis/EstimationIA.tsx` (1,728 lines) · `components/devis/ConstructionTemplate.tsx` (1,130 lines) · `pages/DevisPublicPage.tsx` (538 lines) · `components/devis/DevisRenderModal.tsx` (534 lines) · `components/devis/AiProfileManager.tsx` (410 lines) · `components/devis/DevisAssistantTab.tsx` (152 lines) · `components/devis/ClientInfoCard.tsx` · `components/devis/DevisConditionsEditor.tsx` · `components/devis/DevisFinancialSummary.tsx`. API clients: `api/devis.ts`, `api/devisAi.ts`.
> **PostgreSQL tables (per tenant)**: `devis` (header), `devis_lignes`, `devis_assignations`, `devis_ai_estimations`, `ai_profiles` + `ai_profile_documents`, `conversations` + `conversation_documents`, `manuel_custom_sections` + `manuel_custom_lignes`, plus writes to `companies`, `contacts`, `projects`, `opportunities`, `produits`, `employees`, `factures`. **Shared** table: `public.devis_public_tokens` (public tokens, 90-day validity). AI path: `public.ai_prepaid_credits`, `public.ai_usage_tracking`.
> **Scope**: this module is a company's **central quote editor**. It handles the whole cycle: build a quote in **three ways** (by hand, with **AI**, or by **importing** a plan or a takeoff), edit it online, apply a **markup** (cost-plus) per quote or per line, set the **terms and exclusions**, produce a **professional HTML document**, **send it to the client** through a signable link, **export** (Excel, QuickBooks CSV), **convert to a project** and **invoice**. It does not replace the **Takeoff** module (which measures quantities on a plan and sends them back here), nor the **Accounting** module (which issues invoices): it feeds and connects them.

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

The Quotes module is used to **prepare a price for a client, present it cleanly, and turn it into a project once accepted**. You start from a need (a kitchen renovation, a new build, an addition), you build the list of work items with their prices, you apply your margin, then you send the client a polished document that they can **accept by signing it online**. As soon as they accept, the ERP automatically creates the corresponding **project** and you can **invoice** it.

A quote is therefore a living document: a draft while you build it, sent when it goes to the client, accepted or declined depending on their decision.

### 1.2 The three ways to build a quote: manual, AI, import

This is the key to the module. You are never forced to type everything by hand.

| Way | Where | What you do | Result |
|-----|-------|-------------|--------|
| **Manual** | Detail panel ("Add" button) **or** the **"Manual"** tab (Quebec Construction Template) | You enter lines one by one, or you check items in a 9-section template with quantities and prices | Lines added directly to the quote |
| **AI** | The **"AI Estimation"** tab | You chat with a virtual expert (Claude), you describe the project, then you click "Generate quote" | The AI proposes a quote structured by trade, which you review and then add to the quote or use to create a new quote |
| **Import** | The **"AI Estimation"** tab (upload a plan or a document) **or** the **"Takeoff"** tab (measure on a PDF) | You provide a PDF plan, an image, a price list, or you measure quantities on a plan | The AI **reads** the document and produces an **analysis** (category, areas, trades) or a cost estimate; the Takeoff sends back priced **quantities** |

> **Understand this — "import" does not fill in the lines by itself.** When you upload a plan into AI Estimation, the system produces an **analysis** (text, diagnostic, areas). There is **no button that writes quote lines directly from a file**. You review the analysis, then you click "Generate quote" (AI Estimation) or "Apply to quote" (Takeoff) to create the lines. This is intentional: you always keep the final say over what goes into the price.

### 1.3 Access from the sidebar

Click **Quotes** in the sidebar. The page opens on the **list** of quotes (`/devis`). A direct link exists to open a specific quote: `app.constructoai.ca/devis?open=<id>` (used by the "View quote" buttons elsewhere in the ERP).

### 1.4 The six tabs

At the top of the page, six tabs (`DevisPage.tsx:1358`):

| Tab | Role | Access |
|-----|------|--------|
| **Quotes** | The list + the detail panel (the core of the module) | Everyone |
| **AI Estimation** | Chat with an AI expert and **generate** a quote | Everyone |
| **Takeoff** | Take quantities on a PDF plan (**separate module**, see Chapter 30) | Everyone |
| **Manual** | The "Quebec Construction Template" (9 sections to check) | Everyone |
| **Conditions** | Edit the company's **default** terms and exclusions | **Administrators only** |
| **AI Assistant** | Ask **read-only** questions about your quotes (the "sparkles" icon) | Everyone |

> **The "Conditions" tab does not appear** for a user who is not an administrator (`isAdmin`, `DevisPage.tsx:1363`). This is normal: it touches settings that apply to the entire company.
>
> **The "Takeoff" tab is a full module of its own** (loaded on demand, `components/metre-pdf/MetrePdf`). Its detailed operation is described in **Chapter 30 — Takeoff**. Here, only the **bridge** matters: "Apply to quote" / "Create quote".

### 1.5 Permissions and roles

The module is **deliberately open to the whole team**: create, edit, add lines, send, change a status, delete — all of this is allowed to **any authenticated user** of the tenant. The only reserved action is editing the company's **default terms/exclusions** (`PUT /devis/defaults` → `require_tenant_admin_or_role()`, `devis.py:2564`). This is the module's single admin-only entry point.

| Action | Who is allowed |
|--------|----------------|
| View, create, edit a quote; add/edit/delete lines; send; change the status; convert to a project; invoice; export | Any valid tenant account |
| Delete a quote | Any valid account — **except** if the status is `Accepted` or `Completed` (400 refusal) |
| Edit the company's **default terms/exclusions** (Conditions tab) | **Administrators** (`is_admin`, re-checked server-side) |

**Read-only mode.** If the tenant's subscription is past due, the account can switch to **read-only**: reads work, but any **write** returns 403. This control is applied upstream in authentication and covers every endpoint in the module. Special case: HTML generation **skips the cache write** in read-only mode but remains viewable (`devis.py:11202`). Public token endpoints (client viewing, accepting, declining) do not pass through this control.

**AI credits.** The artificial-intelligence features (AI Estimation, document analysis, assistant, quote estimation, 3D render) **consume AI credits billed to the tenant**. Each AI endpoint first checks credits (402 if the balance is insufficient) and logs the spend. The balance is always shown in the AI Estimation toolbar (or "Unlimited" for exempt accounts).

**Isolation.** Every request is scoped to the tenant's PostgreSQL schema; one company never sees another's quotes. Documents attached to an AI conversation are additionally isolated **per user** within the tenant.

### 1.6 Automatic numbering

On creation, each quote receives a **`DEV-YYYY-NNN`** number (e.g., `DEV-2026-007`): the current year followed by the identifier. The number is produced in a way that is **fail-safe even under simultaneous clicks**: the quote is first inserted with a provisional number, then updated with its final number derived from its real identifier (`RETURNING id`, `devis.py:9084`). A `COUNT + 1` that could produce duplicates is never used.

### 1.7 Statuses and types

**Statuses.** A quote is born as a **Draft**. It moves to **Sent** when you send it, then to **Accepted** or **Declined** depending on the client's decision. Other values exist (`Validated`, `Pending`, `Completed`, `Cancelled`, `Expired`) and are reachable through bulk update or other flows. The list filter exposes only the six common statuses (see 2.3).

**Quote types.** **Detailed** (default) or **Budget**. The "Budget" type signals an approximate estimate.

**Project type.** Five choices at creation (`TYPE_PROJET_OPTIONS`, `DevisPage.tsx:593`): New residential, Residential renovation, New commercial, Commercial renovation, Institutional/Public.

**Priority.** Normal (default), High, Urgent.

### 1.8 The pricing model: markup is **embedded in the prices**

This is the most important point to understand, and the most counterintuitive.

Since a 2026 business decision, the quote operates on a **"markup embedded in the unit prices"** basis. Concretely:

- You enter your **base costs** (quantity × unit price) on each line.
- The system applies, on top, a **markup** made of three parts: **Administration** (3% by default), **Contingencies** (12% by default) and **Profit** (15% by default). Together, that is a factor of roughly **×1.30**.
- **The price shown to the client on each line already contains this markup.** The three "Administration / Contingencies / Profit" lines in the financial summary are **not** amounts added on top: they are an **informational breakdown** of the markup already included in the prices.

> **Do not be misled**: if you see "Administration 3%, Contingencies 12%, Profit 15%" under the subtotal, that **does not increase** the total. The "all taxes included" total the client sees is the **same** in the preview, in the Excel, and in the CSV. The breakdown exists only so you know how your price is composed. The summary reminds you of this with the note "Including markup added to unit prices:".

Two concrete cases:

- **Markup at 3/12/15**: your unit prices are your costs, and the margin is added for display. A gray callout details the share of each part.
- **Markup at 0/0/0**: a blue callout appears — "0% markup — overhead, contingencies and profit already included in the unit prices (all-inclusive, e.g. General Contractor estimate)". This is the "all-inclusive" mode, typical of a general contractor's estimate.

**The 15% profit: a default, not a lock.** The 15% is the **default** value (deterministic pricing tool, AI generation, default column value in the database). But you can enter **any profit percentage** on the quote (0 to 100%) and even a different percentage **per line**. The final document always uses the **actually recorded** percentage, not a hard-coded 15%.

**The per-sq.ft. price base comes from the expert, not from a constant.** There is **no fixed "$X/sq.ft." value** in the module. When the AI prices a build, **it** provides the per-square-foot rate (per floor), based on the tier and the region. The server only imposes the markup cascade (×1.30) and **soft validation ranges** (see 4.6) that flag an abnormal price without ever blocking it. The "Economy tier" used by default corresponds to the **bottom of the category's range** (for example the low end of $200–450/sq.ft. for a new residential build), not to a single figure.

### 1.9 What the module does not do (verified in the code)

- **No direct PDF export.** The PDF export was removed (`DevisPage.tsx:19`). To get a PDF, use **"Generate HTML"** then **Print** (the browser produces the PDF), or the client's public page ("Print" / "Download" button). Vectorial PDF export exists only in the **Takeoff/CAD** module, not here.
- **The "Conditions" tab is hidden from non-administrators.**
- **The AI Assistant (last tab) is read-only**: it answers questions about your data, but **creates and modifies no quote**. To generate, use the "AI Estimation" tab.
- **Import does not fill lines automatically** (see 1.2): it produces an analysis to review.
- **A single 3D render per quote** (replaceable or removable).
- **The client's signature is required to accept**: the name alone is not enough.
- **Lines require a quantity greater than 0.** The "administration / contingencies / profit" items and lines with quantity 0 are excluded from bulk adds.
- **The 3D render is paid**: generating it consumes the tenant's AI credits (×1.30); attaching or removing it is free.

---

## 2. Interface

### 2.1 The four indicators (KPIs)

Always visible at the top (`DevisPage.tsx:1334`), fed by `getDevisStatistics()`:

| Indicator | Meaning |
|-----------|---------|
| **Total quotes** | Total number of quotes for the tenant |
| **Drafts** | Quotes still in Draft status |
| **Sent** | Quotes sent to the client |
| **Acceptance rate** | Accepted ÷ (accepted + declined), as a percentage |

### 2.2 The tab bar

The six tabs described in 1.4. The active tab is highlighted. The "AI Assistant" tab carries the "sparkles" icon (Sparkles).

### 2.3 The "Quotes" tab — the list

**The command bar** (`DevisPage.tsx:1669`) contains:

- The main **"New quote"** button.
- Three view buttons: **List**, **Table**, **Cards**.
- A **Search…** field (on number and title).
- A **Status** filter: All, Draft, Sent, Accepted, Declined, Expired.
- A **Type** filter: All types, Detailed, Budget.

**The bulk-action bar** appears as soon as you check at least one quote (`DevisPage.tsx:1649`): "{n} quote(s) selected", a "Change status…" dropdown (Draft / Sent / Accepted / Declined / Expired) and a "Deselect" button.

**The List view** (`DevisPage.tsx:1707`) shows a table on desktop and cards on mobile. Columns are sortable and resizable:

| Column | Content |
|--------|---------|
| (checkbox) | Selection for bulk actions |
| **Number** | `DEV-YYYY-NNN` |
| **Title** | Project name |
| **Client** | Name (company, contact, or manual entry) |
| **Amount** | Total, all taxes included |
| **Status** | Colored badge + inline editing via dropdown |
| **Type** | Detailed / Budget, editable inline |
| **Planned start** | Date, editable inline |
| **End date** | Date, editable inline |
| **Created** | Creation date |
| (trash) | Delete |

> **Inline editing of the status to "Accepted"**: the system **confirms first** ("Marking quote '…' as Accepted will automatically create a new project. Continue?"), because acceptance triggers project creation on the server.

**The Table view** (`DevisPage.tsx:1844`) adds the detailed financial columns: **Subtotal / GST / QST / Total**.

**The Cards view** (`DevisPage.tsx:1936`) presents a grid of cards (number, status badge, name, client, amount in green, trash).

**Status colors**: Draft = gray, Sent = indigo, Accepted/Completed = green, Declined/Cancelled = red, Expired = amber.

### 2.4 The detail panel

Click a quote to open its **detail panel** (right-hand column on desktop, full screen on mobile). At the top: the number, the opportunity number if there is a link (blue badge), the project name, an **Edit** pencil and the close button. Then the status badge, the client and the description.

#### A. The financial summary (editable)

`DevisFinancialSummary` component (`DevisPage.tsx:383`). This is where you set the markup and the presentation.

- **Subtotal**, followed by the note "Including markup added to unit prices:".
- If Administration = Contingencies = Profit = 0, a blue callout explains the "all-inclusive" mode.
- **Three lines Administration / Contingencies / Profit.** Each offers:
  - an **eye** to show or hide it in the document;
  - an **editable label** (e.g., rename "Administration" to "Overhead");
  - a **percentage** field (0 to 100, step 0.5);
  - a **dollar amount** field (entering it recalculates the matching percentage).
  - Default values: Administration 3%, Contingencies 12%, Profit 15%.
- The **tax lines** (GST 5%, QST 9.975% in Quebec by default) then the **Grand total** in bold.
- A **"Quote columns"** section: five toggles that decide which columns are visible in the exported document — **Unit**, **Quantity**, **Unit price**, **Line amount**, **Labor and Material**.

#### B. Terms and exclusions

`DevisConditionsEditor` component (`DevisPage.tsx:223`), collapsible. Two text areas (Terms / Exclusions), each with an **eye** to show or hide it, a **"Reset"** button (returns to the company defaults) and sample text. A badge indicates "Defaults" or "Custom". One line per term or exclusion; bullets and numbering are added automatically in the final document.

#### C. The action buttons

Up to eight buttons (`DevisPage.tsx:2036`) depending on the quote's state:

| Button | Effect |
|--------|--------|
| **Generate HTML** | Produces the professional HTML document |
| **Preview** | Opens the document in a window (iframe) |
| **Add a 3D render** | Opens the photorealistic render window (see 2.9) |
| **Send to client** (primary) | Opens the email + public-link sending window |
| **Convert to project** (green) | Visible if Accepted/Completed and no linked project yet |
| **Create invoice** | Visible if Accepted/Completed — creates a draft invoice |
| **QuickBooks CSV** | Downloads a QuickBooks-compatible CSV |
| **Copy CSV** | Copies the CSV to the clipboard |
| **Excel (.xlsx)** | Downloads the Excel file |

> The QuickBooks CSV contains the columns Item, Description, Category, Quantity, Unit, Unit Price, Amount, Tax Code, MO %, MO $, MAT %, MAT $. Prices in it are **already marked up** (line factor), to stay consistent with the HTML preview and the Excel.

#### D. The lines

Title "Lines ({n})" and an **"Add"** button. Each line is read and edited in place.

- **When reading**: the description (with a "%" badge if the line has its own markup), quantity × price (marked up), the amount (marked up), a visibility **eye**, an **edit** pencil, a **trash**. If Labor/Material display is on, a sub-line shows the "Labor x% / Material y%" split.
- **When editing** (`startEditLine`, `DevisPage.tsx:1116`): description, quantity, unit, unit price, a **custom Labor/Material ratio** (two fields that complete each other, "Auto" button to return to keyword detection), and a **custom per-line markup** (Admin / Conting. / Profit — leave empty to inherit the quote's percentages; a "Custom" badge and an "Inherit from quote" link appear). A live preview shows "= {amount} (markup x%)".

At the bottom, the **Labor/Material totals** if display is on, and the note **"Validity: {date}"**.

> **Automatic Labor/Material detection.** From keywords in the description, the system guesses the labor / material split (20 rules covering Quebec trades, `MO_MAT_RULES`, `DevisPage.tsx:52`). Examples: painting 70/30, demolition 65/35, drywall 60/40, electrical 55/45, plumbing 50/50, roofing 45/55, concrete/foundation 40/60, insulation 35/65, excavation 30/70, cabinets 30/70, doors and windows 30/70. With no recognized keyword: 50/50. You can always force your own percentages.

### 2.5 The panel's modals

- **New quote** (`DevisPage.tsx:2491`) — two columns. Left: Project name (required), Client PO #, Client (Company), Client (Person), Manual entry, Status, Priority, Project type. Right: Current task (among 25 production tasks), Quote deadline, Planned start, Planned end, Price ($). Bottom: Description. A guard prevents double submission.
- **Edit quote** (`DevisPage.tsx:2555`) — the same fields, plus the Quote type.
- **Add a line** (`DevisPage.tsx:2628`) — Description (required), Quantity, Unit, Unit price, with a tax preview (Subtotal, taxes, Grand total).
- **HTML preview** (`DevisPage.tsx:2668`) — the document iframe, with "Open in new tab" and "Close".
- **Send to client** (`DevisPage.tsx:2746`) — a "Client email" field, an intro message explaining that the status will move to "Sent" and that a public validation link will be generated. After sending: an alert (success or warning depending on whether the email went out), the **public validation link** and a "Copy link" button.

### 2.6 The "AI Estimation" tab

`EstimationIA.tsx` component (1,728 lines). This is the conversation with an expert that **generates** a quote.

**The toolbar** (`EstimationIA.tsx:1064`):

- An **expert profile selector**, split into "My profiles" and "System profiles". The number of system profiles is **dynamic** (loaded at startup): it covers dozens of trades. Next to it, a **gear** opens the custom-profile manager.
- A **Document** button to upload 1 to 5 files (`.pdf, .png, .jpg, .jpeg, .txt, .csv, .xlsx, .docx`, max 32 MB).
- A **History** button, a **New** button, and an **AI-credit indicator** (balance in US dollars or "Unlimited").

**Automatic profile detection.** Depending on the jurisdiction (remembered per tenant), a suitable profile is preselected. The first document upload switches to the **General Contractor** profile (category diagnostic).

**The progress bar** shows the upload phases then the analysis phases. With several documents, the system uses "one AI agent per document + a coordinating chief".

**The History panel** lists saved conversations (auto-saved after each answer), with inline rename and delete. Restoring a conversation re-establishes the client info card and the generated quote.

**The linked-quote banner** shows in blue "Linked quote: {name} — items added to this quote" or in amber "No quote selected…". If no quote is linked, a **client info card** (see 2.8) appears to enter the information.

**The diagnostic banner** (General Contractor mode) displays the detected category and subcategory, and the breakdown by zone (To estimate / Renovation / Expansion / Existing kept, in sq.ft.).

**The document thumbnails** (persisted) let you, for each file, download it, activate or deactivate it from the AI context, or delete it.

**The conversation area** shows the messages (Markdown rendering), a "{profile} is thinking…" indicator, and a starting state with a clickable sample question and a three-step guide ("1. Choose an expert · 2. Describe your project · 3. (optional) Attach a plan"). The input field lets you attach files (paperclip, max 5), type, then send.

**The "Generate quote" button** (`EstimationIA.tsx:1491`) launches the structuring. The result appears in a **generated-quote table** (`EstimationIA.tsx:1502`):

- A header "Generated quote — {n} items" with the buttons **HTML** (local export, complete company-branded document, with schedule and signature area), **Add to quote** (if a quote is linked) or **Create new quote**.
- The table is **grouped by trade** (colored sections, per-section total), with items **editable inline** (pencil, trash, bounds on quantity and price).
- The totals: Subtotal, Administration (x%), Contingencies (x%), Profit (x%), Subtotal (pre-tax), taxes, **Grand total**.
- An **"Estimated schedule" Gantt chart** appears if there is more than one section.

**The profile manager** (`AiProfileManager.tsx`) is a window to create and edit your **custom AI profiles**: a **Name**, **Instructions** (personality and expertise), and a **knowledge base** (upload documents `.pdf, .txt, .csv, .xlsx, .docx, .md, .tsv`, max 20 MB — the text is extracted and injected into the AI's context).

### 2.7 The "Manual" tab — the Quebec Construction Template

`ConstructionTemplate.tsx` component (1,130 lines). A banner shows whether a quote is linked. If none, a **client info card** appears. Then the template itself, with three sub-tabs (`ConstructionTemplate.tsx:583`):

- **Works** — **9 fixed sections** numbered 0.0 to 8.0: Site Preparation and Demolition, Foundation (Infrastructure and Services), Structure and Framing, Exterior Envelope, Mechanical and Electrical Systems, Insulation and Waterproofing, Interior Finishes, Exterior Landscaping and Garage, Machinery. Each item is a **checkbox** that opens the Quantity / Unit / Unit price / Amount fields. Nine units are offered (lump sum, sq.ft., lin.ft., unit, hour, day, m², lin.m, cubic yard). You can add **custom lines** per section and **custom sections** (numbered 9.0 and up, renamable and removable). A warning appears if you use a reserved name (administration, contingencies, profit).
- **Recap** — the items grouped by section, with the financial summary (Total works, Administration x%, Contingencies x%, Profit x%, Subtotal before taxes, taxes, GRAND TOTAL).
- **Config** — three sliders Administration / Contingencies / Profit and the "total markup" display.

At the bottom, the **"Apply to quote '{name}' ({n} items — {total})"** button (if a quote is linked) or **"Create new quote"**. The system excludes the administration/contingencies/profit categories and adds only lines with a quantity greater than 0. A **"Preview Quote HTML"** button lets you see the rendering **without saving anything**.

### 2.8 The client info card (shared)

`ClientInfoCard.tsx` component, reused by AI Estimation, Takeoff and Manual when no quote is linked. A collapsible "Client info / Quote information" card with five sections: **Project** (Name), **Client** (Company / Person / Manual entry), **Schedule** (Quote deadline / Planned start), **References** (PO # / Priority: Normal, High, Urgent), **Notes** (Description).

### 2.9 The 3D render (optional, paid)

`DevisRenderModal.tsx` component (534 lines). Adds a **photorealistic image** at the bottom of the quote. The flow has five steps:

1. **Upload** an image or a PDF (no 3D file).
2. **Crop** the area to render.
3. **Set** the parameters (Details, Quality: pro / standard / fast, Resolution: 2K / 4K).
4. **Render**: preview + cost shown in US dollars, with "Attach" or "Redo".
5. **Attached**: "Replace", "Remove" or "Done".

> **Generation is billed** (tenant credits ×1.30, via the `/cao/render` render module). **Attaching or removing is free.** Locks prevent double-clicking. The render then appears in the HTML preview **and** on the client's public page. **A single render per quote.**

### 2.10 The "Conditions" tab (administrators)

`DevisDefaultsTab` component (`DevisPage.tsx:2826`). Reserved for administrators. It edits the company's **default Terms and Exclusions** (two text areas, Save, Reset to system values). These texts apply to **new** quotes; existing quotes are not affected, and each quote can then be customized individually.

### 2.11 The "AI Assistant" tab (read-only)

`DevisAssistantTab.tsx` component (152 lines). A conversational assistant that queries your **real quote data** (amounts, statuses, taxes, clients) and answers in natural language. Title "AI Assistant — Quotes", subtitle reminding that it is **read-only**. It offers three sample questions. Each answer shows metadata (tokens, cost, duration).

> **This assistant creates and modifies nothing.** It is distinct from AI Estimation (which does generate). It consults, summarizes, compares. To act, go back to AI Estimation or the editor.

### 2.12 The "Takeoff" tab (the bridge)

The "Takeoff" tab loads the on-plan quantity-takeoff module (Chapter 30). From that module, two buttons bring the result back here: **"Apply to quote"** (adds the priced lines to the linked quote) and **"Create quote"** (creates a new quote from the takeoff). This is the second major "import" path: quantities measured on a real plan become priced lines.

### 2.13 The public page (client side)

`DevisPublicPage.tsx` component (538 lines), at `/devis/public/:token`, **without authentication**. This is what your client sees when they click the link received by email.

- **Header**: the company's contact details, the quote's number and title, then the full document in an iframe.
- **Toolbar**: a **zoom** (50 to 200%, the mobile version starts at 60%), a **Print** button and a **Download** button (HTML file). A "Powered by Constructo AI" footer.
- **Two decision buttons**: **Decline** and **Accept quote**.
  - **Accept** opens a form: "Your full name" **and** a **drawn signature** (in a canvas). The confirmation button stays disabled until both the name **and** the signature are provided. The confirmation screen then shows "Signed by: {name}" with the signature image.
  - **Decline** lets the client give a reason (optional).
- Possible states: loading, ready, accepted, declined, error, or "already decided" (if the client comes back later).

---

## 3. Step-by-step workflows

### 3.1 Create a quote manually, line by line

1. **Quotes** tab → **"New quote"** button.
2. Enter at minimum the **Project name** (required).
3. Choose the client: a **company** from the CRM, a **person**, or a **manual entry** if the client is not yet registered.
4. Set the status, priority, project type, dates and PO # as needed.
5. Click **Create**. The `DEV-YYYY-NNN` number is assigned automatically.
6. Open the quote, then, in the **Lines** section, click **"Add"**.
7. For each line: description, quantity, unit, unit price. Repeat.
8. Set the **markup** and the **terms** in the financial summary (see 3.7 and 3.8).

### 3.2 Build with the Quebec Construction Template (Manual tab)

1. First link a quote (select it in the Quotes tab) — or let the system create one.
2. **Manual** tab → **Works** sub-tab.
3. Go through the **9 sections**, **check** the items that apply, enter Quantity / Unit / Unit price.
4. Add **custom lines** or **custom sections** as needed.
5. **Config** sub-tab: set Administration / Contingencies / Profit.
6. **Recap** sub-tab: check the total.
7. Click **"Preview Quote HTML"** to see the rendering without saving anything.
8. Click **"Apply to quote"** (or **"Create new quote"**). Only lines with a quantity greater than 0 are transferred.

### 3.3 Estimate with AI (AI Estimation tab)

1. **AI Estimation** tab.
2. Choose a suitable **expert profile** (or keep the preselection).
3. Fill in the **client info card** (if no quote is linked).
4. In the conversation, **describe the project** (type, area, tier, constraints). You can attach a plan (paperclip).
5. Chat with the expert until you reach a good understanding of the project.
6. Click **"Generate quote"**. The AI structures the lines **by trade** and proposes the markup (profit is brought back to 15% by default).
7. **Review** the generated table (edit, delete items as needed).
8. Click **"Add to quote"** (linked quote) or **"Create new quote"**.

> A build's price is computed **deterministically**: the AI supplies the floors and their areas, and the server applies the "base × 1.30 × taxes" formula (see 4.5). This avoids omissions (garage, missing area) and makes the pricing reproducible.

### 3.4 Import a plan or a document for AI analysis

1. **AI Estimation** tab → **Document** button (or the conversation paperclip).
2. Select 1 to 5 files (PDF, image, `.txt`, `.csv`, `.xlsx`, `.docx`), 32 MB maximum, PDF up to 100 pages.
3. The system uploads then **analyzes** (one agent per document + a coordinator for several files).
4. Read the **diagnostic**: category, tier, areas by zone, trades.
5. Continue the conversation to refine, then **generate the quote** as in 3.3.

> **Reminder**: the analysis does not create lines on its own. You are the one who triggers generation once the analysis is validated.

### 3.5 Estimate an existing quote (text or plan)

Two specialized endpoints work on an **already open** quote:

- **From the text** (`POST /devis/{id}/ai-estimate`): the AI analyzes the quote's content. In **precision mode** (default), it "thinks" harder; it can consult the tenant's **product catalog** to anchor the prices.
- **From a plan** (`POST /devis/{id}/ai-estimate-with-plan`): you upload a plan, the AI reads it (vision) and estimates. 32 MB cap, PDF up to 100 pages, automatic compression of large images, additional context up to 20,000 characters.

Each estimate is **archived** (`devis_ai_estimations`) and can be viewed / deleted.

### 3.6 Create and manage an AI expert profile

1. **AI Estimation** tab → **gear** next to the profile selector.
2. Click **"Create a profile"**.
3. Give a **Name** and **Instructions** (personality, expertise, pricing rules, standards to cite).
4. In **Knowledge base**, add your documents (price lists, specifications, catalogs), 20 MB maximum each. The text is extracted and injected into the context.
5. Save. The profile now appears under "My profiles" in the selector.

### 3.7 Adjust the markup (per quote and per line)

**At the quote level** (financial summary):

1. Set the **percentage** for Administration, Contingencies and Profit, or enter a **dollar amount** directly (the percentage recalculates).
2. Rename the labels if you wish (e.g., "Overhead").
3. Hide a part with its **eye** if you do not want to show it to the client.

**At the line level** (line editing): enter Admin / Conting. / Profit **specific to that line**. Leave empty to inherit the quote's percentages.

> **Caution**: changing a percentage **at the quote level clears** any override of the same part **on the lines** (`devis.py:9250`). So set the quote level first, then the per-line exceptions.

### 3.8 Customize the terms and exclusions

1. In the detail panel, expand **Terms & Exclusions**.
2. Write **one line per term** and **one line per exclusion** (bullets and numbering are automatic).
3. Use the **eye** to hide a section, or **"Reset"** to return to the company defaults.
4. These texts replace the defaults for **this** quote only.

### 3.9 Generate and check the HTML rendering

1. Detail panel → **"Generate HTML"**, then **"Preview"**.
2. Check the layout, prices, schedule and terms in the iframe.
3. If needed, **"Open in new tab"** for a full-page view.

> The document is a **premium three-page** rendering in Letter format (8.5 × 11 in), in the company's colors (logo included), with schedule/Gantt and price banners (area, $/sq.ft., taxes, validity).

### 3.10 Add a 3D render

1. Detail panel → **"Add a 3D render"**.
2. Upload an image or a PDF, **crop**, set the quality and resolution.
3. Launch the render (it **consumes credits**), check the preview, then **"Attach"**.
4. The render appears in the HTML preview and on the public page. You can **Replace** or **Remove** it (free).

### 3.11 Send the quote to the client

1. Detail panel → **"Send to client"**.
2. Enter the client's **email address**.
3. Click send. The system:
   - moves the status to **Sent**;
   - generates a **public token** (if none exists) valid for **90 days**;
   - sends a **company-branded email** with the link `/devis/public/{token}`;
   - records the recipient and the send date.
4. Copy the **public validation link** shown if you also want to share it another way.

### 3.12 The client accepts (with signature) or declines

On the client side, on the public page:

- **Accept**: they enter their **full name**, **draw their signature**, then confirm. Through an **atomic** update, the server moves the status to **Accepted** (a single "winner" in case of a double click), and records the name, the signature and the date. Then, in the background and without ever rolling back the acceptance even in case of a hiccup: **project creation**, copying of attachments, creation of work orders, moving the linked opportunity to "Won".
- **Decline**: they can give a **reason** (optional). The status moves to **Declined** and the reason is recorded.

In both cases, the contractor is notified.

### 3.13 Convert to a project (manually)

If a quote is **Accepted** or **Completed** but does not yet have a project (for example, accepted offline):

1. Detail panel → **"Convert to project"** (green button).
2. The operation is **idempotent**: if a project already exists, it returns the existing identifier without creating a second one.

### 3.14 Invoice

1. Detail panel → **"Create invoice"** (visible if Accepted/Completed).
2. Confirm. An **invoice** is created from the quote (Accounting module).

### 3.15 Export (Excel, QuickBooks CSV)

- **Excel (.xlsx)**: dedicated button, direct download (client header, lines, subtotals, taxes, grand total; protection against formula injection).
- **QuickBooks CSV**: the "QuickBooks CSV" button (download) or "Copy CSV" (clipboard).
- **PDF**: there is **no** direct PDF export. Do **Generate HTML → Preview → Print** (choose "Save as PDF" in the print window), or let the client print/download from the public page.

### 3.16 Change the status of several quotes at once

1. In the list, **check** several quotes.
2. In the bulk-action bar, choose the new status in "Change status…".

### 3.17 Assign employees to a quote

Three endpoints let you link employees to a quote (`GET/POST/DELETE /devis/{id}/assignments`), with a role. A duplicate is refused (409).

### 3.18 Calculate a CCQ or CNESST contribution

Two small calculators are exposed (**pure-calculation** endpoints, without authentication):

- **CCQ** (Commission de la construction du Québec — Quebec construction commission) (`POST /devis/calculate-ccq`): from a labor amount and a list of trades; per-trade rates are hard-coded (about 11.8 to 12.5%, default 12.5%). This is not an "hours × hourly rate" calculation, but "amount × trade rate".
- **CNESST** (Commission des normes, de l'équité, de la santé et de la sécurité du travail — Quebec workplace health and safety board) (`POST /devis/calculate-cnesst`): contribution = labor × rate (`taux_unite` parameter, default 1.80%).

### 3.19 Change the company's default terms (administrators)

1. **Conditions** tab (visible only to administrators).
2. Edit the default Terms and Exclusions.
3. **Save** (or **Reset** to system values). New quotes inherit them; old ones do not change.

### 3.20 Delete a quote

1. Trash in the list or the detail panel, then confirm.
2. **Refusal (400)** if the status is **Accepted** or **Completed**. To delete, change the status first.
3. Deletion removes the lines, assignments and other linked items, and **detaches** (sets to NULL) the attached invoices, projects and opportunities.

---

## 4. Reference

### 4.1 Statuses

| Status (displayed) | Badge color | Meaning |
|--------------------|-------------|---------|
| Draft | Gray | Being prepared |
| Validated | (internal) | Checked internally |
| Sent | Indigo | Sent to the client |
| Pending | (internal) | Received, decision pending |
| Accepted | Green | Signed by the client → project created |
| Declined | Red | Declined by the client |
| Completed | Green | Cycle completed |
| Cancelled | Red | Cancelled internally |
| Expired | Amber | Validity passed |

The list filter exposes six statuses (All, Draft, Sent, Accepted, Declined, Expired). Key transitions: send → **Sent**; public acceptance → **Accepted**; public decline → **Declined**. Bulk update allows any status. Server-side, public access is **deny by default**: a status is only viewable or "decidable" by the client if it appears on an allowlist (`devis.py:12421` / `12427`).

### 4.2 Types and priorities

- **Quote type**: Detailed (default), Budget.
- **Project type**: New residential, Residential renovation, New commercial, Commercial renovation, Institutional/Public.
- **Priority**: Normal (default), High, Urgent.

### 4.3 The cost-plus pricing model (cascade)

The default values are Administration **3%**, Contingencies **12%**, Profit **15%** (`devis.py:1289-1291`), i.e., a total markup of **×1.30**.

```
base                 = sum of (quantity × unit price) across the lines
administration       = base × adm%
contingencies        = base × con%
profit               = base × pro%
subtotal before taxes = base + administration + contingencies + profit
GST (5%)             = subtotal before taxes × 0.05
QST (9.975%)         = subtotal before taxes × 0.09975
GRAND TOTAL          = subtotal before taxes + GST + QST
```

**Two representations of the markup**:

- **In the database**, `devis_lignes.montant_ligne` = pure base cost (`quantity × price`, no markup). The quote's `administration / contingencies / profit` aggregates are recomputed on every write.
- **In the rendered document** ("markup embedded" model), each line shows `montant_ligne × line_factor`, where `line_factor = 1 + adm + con + pro` (with any per-line overrides). The summary **does not add** the markup: it shows the "Subtotal" (markup already included) then **breaks it down** in gray.

### 4.4 Per-line markup overrides

A line can carry its own percentages: `admin_pct_ligne` and `contingence_pct_ligne` (0 to 100), `profit_pct_ligne` (−100 to 999, to allow rebates or special cases). An empty value (NULL) = the line **inherits** the quote's percentages. These overrides are an **internal** tool: they are **never** shown to the client (the public page strips these fields).

### 4.5 The deterministic tool `calculer_prix_construction`

To price a build, the AI does not invent the total: it calls a server-side **calculation tool** (`devis.py:736`). It provides the list of **floors** (gross area in sq.ft., with a weight per floor: ground floor 1.0, top floor 0.85, intermediate floors 0.80) and the **reduced zones** (unheated garage, basement), along with a **per-sq.ft. rate** (base cost). The server then applies the "base × 1.30 × taxes" cascade — **exactly** the same formula as the HTML document. Result: a reproducible cost estimate, with no missing surface.

> This is where the per-sq.ft. rate lives: it is **supplied by the AI / expert profile**, not set by the server. No "$X/sq.ft." constant exists in the module.

### 4.6 APCHQ validation ranges (soft)

After a generation, the server checks that the per-sq.ft. price stays within a **reasonable range** for the category, and flags (without blocking) the outliers (`_validate_estimation_items`, `devis.py:6413`). APCHQ = Association des professionnels de la construction et de l'habitation du Québec (Quebec home-building industry association). Examples:

| Category | Range $/sq.ft. |
|----------|----------------|
| New residential build | 200 – 450 |
| Residential expansion | 250 – 500 |
| Major renovation | 150 – 400 |
| Basement | 80 – 200 |
| Kitchen renovation | 200 – 1000 |
| (default) | 50 – 2000 |

These checks are **non-blocking**: they produce warnings ([SOFT] or [CRITICAL]) but let the quote through. Other signals: price outside $0.01–$1M, quantity outside 0.001–100,000, total above $10M, one category concentrated above 40%, duplicates. The **default tier is "Economy"** = the bottom of the category's range.

### 4.7 Fields of a line

| Field | Required | Notes |
|-------|----------|-------|
| `description` | Yes | — |
| `quantite` | Yes | Must be greater than 0 for bulk adds |
| `unite` | No | Free text (default "unité") |
| `prix_unitaire` | Yes | ≥ 0 |
| `montant_ligne` | Auto | `round(quantity × price, 2)` = **base cost, no markup** |
| `categorie` | No | Trade (grouping) |
| `notes_ligne` | No | Item detail (bullets) |
| `visible` | No | Default true; false = excluded from the document |
| `mo_pct` / `mat_pct` | No | Labor / material split |
| `admin_pct_ligne` / `contingence_pct_ligne` / `profit_pct_ligne` | No | Markup overrides (see 4.4) |
| `sequence_ligne` | Auto | Order |

### 4.8 Units and categories

- **Manual template units** (9): lump sum, sq.ft., lin.ft., unit, hour, day, m², lin.m, cubic yard.
- **Quote categories**: 21 reference trades (`_SOUMISSION_CATEGORIES`), each with an English equivalent. The AI tool schema switches automatically based on the tenant document's language.
- **Default terms and exclusions**: 5 terms (30-day validity, 30/40/30 payment schedule, 1-year warranty, RBQ mention) and 15 hard-coded exclusions, overridable per quote then per company. (RBQ = Régie du bâtiment du Québec — Quebec building authority.)

### 4.9 AI models and pricing

| Feature | Model | Max tokens | Cost (per million, before margin) |
|---------|-------|------------|-----------------------------------|
| AI Estimation (conversation, generation, analysis, plan) | `claude-opus-4-8` | 32,000 | input $5, output $25, cache write $10, cache read $0.50 |
| **Read-only** AI Assistant (`/devis/ai/chat`) | Sonnet (`AI_MODEL`) | 8,000 | ≈ $0.003/1k input, $0.015/1k output |

In both cases, the real cost is **marked up 30%** and deducted from the tenant's prepaid credits, then logged (`track_ai_usage`). Calls to Claude are offloaded off the event loop so they do not freeze the shared ERP. Prompt caching (1 h) and the files API reduce costs on long conversations.

### 4.10 Limits and caps

| Item | Limit |
|------|-------|
| Public token validity | 90 days |
| Signature (data-URL image) | ≤ 500,000 characters |
| Document / plan analysis | ≤ 32 MB, PDF ≤ 100 pages |
| Conversation with files | ≤ 5 files, ≤ 10 MB each |
| Multi-document analysis | ≤ 5 files |
| A profile's knowledge base | ≤ 20 MB per document |
| A profile's instructions | ≤ 200,000 characters |
| Terms / exclusions | ≤ 10,000 characters, ≤ 200 lines |
| AI messages (anti-abuse) | ≤ 400 messages, ≤ 1,500,000 characters |
| `/devis/ai/chat` rate | 20 requests/min per IP |
| `/devis/public/` rate | 60 requests/min per IP |
| Deletion | Forbidden if Accepted/Completed |

### 4.11 Endpoint table

**Main router `/api/erp/v1/devis`** (57 endpoints — a selection of the most used):

| Method | Path | Role |
|--------|------|------|
| GET | `/devis` | Paginated list (status, type, client filters) |
| GET | `/devis/statistics` | Indicators (KPIs) |
| POST | `/devis` | Create (Draft status, auto number) |
| GET | `/devis/{id}` | Detail + lines |
| PUT | `/devis/{id}` | Edit (cascade recompute, auto project creation if Accepted) |
| DELETE | `/devis/{id}` | Delete (refused if Accepted/Completed) |
| POST | `/devis/batch-update` | Change status in bulk |
| POST | `/devis/{id}/lignes` · `/lignes/batch` | Add one / several lines |
| PUT · PATCH · DELETE | `/devis/{id}/lignes/{lid}` [`/visibility`] | Edit / hide / delete a line |
| POST | `/devis/{id}/preview-html-with-items` | Preview with in-memory lines (0 writes) |
| POST | `/devis/{id}/generate-html` | Generate and cache the document |
| POST · DELETE | `/devis/{id}/render` | Attach / remove a 3D render |
| GET | `/devis/{id}/export-xlsx` | Excel export |
| POST | `/devis/{id}/send` | Send (status → Sent, email + token) |
| POST | `/devis/{id}/convert-to-project` | Convert to a project (idempotent) |
| POST | `/devis/{id}/ai-estimate` · `/ai-estimate-with-plan` | Estimate an existing quote (text / plan) |
| GET · DELETE | `/devis/{id}/ai-estimations[/{eid}]` | Estimate history |
| GET · POST · DELETE | `/devis/{id}/assignments[/{aid}]` | Employee assignments |
| POST | `/devis/ai-chat` · `/ai-chat-with-files` | Expert conversation (with files) |
| POST | `/devis/ai-generate-soumission` | Structure a quote |
| POST | `/devis/ai-analyze-document` · `/ai-analyze-documents` | Analyze 1 / up to 5 documents |
| GET · POST · PUT · DELETE | `/devis/ai-profiles[/{id}][/documents]` | Custom AI profiles |
| GET | `/devis/expert-profiles` | List of system + custom profiles |
| GET · POST · PUT · PATCH · DELETE | `/devis/conversations[/{id}][/documents]` | AI conversation history |
| GET · PUT | `/devis/defaults` | Default terms/exclusions (**admin** for writes) |
| POST | `/devis/calculate-ccq` · `/calculate-cnesst` | Calculators (without authentication) |
| GET | `/devis/public/{token}` | Public view (without authentication) |
| POST | `/devis/public/{token}/accept` · `/refuse` | Client acceptance (signature) / decline |

**Assistant router `/api/erp/v1/devis/ai`** (1 endpoint): `POST /chat` — read-only assistant.

**Manual template catalog router `/api/erp/v1/devis/manuel-template`** (8 endpoints): `GET/POST /sections`, `PUT/DELETE /sections/{id}`, `GET/POST /lignes`, `PUT/DELETE /lignes/{id}`.

### 4.12 PostgreSQL tables

`devis`, `devis_lignes`, `devis_assignations`, `devis_ai_estimations`, `ai_profiles`, `ai_profile_documents`, `conversations`, `conversation_documents`, `manuel_custom_sections`, `manuel_custom_lignes` (per tenant); `public.devis_public_tokens` (shared). The module also writes to `companies`, `contacts`, `projects`, `opportunities`, `produits`, `employees`, `factures`. Several columns are created "on demand" (idempotent lazy migrations), with schema-qualified DDL to avoid accidentally writing into the shared schema.

---

## 5. Integrations and FAQ

### 5.1 Links with the other modules

| Module | Link |
|--------|------|
| **CRM / Opportunities** (ch. 06) | An opportunity can spawn a quote; on acceptance, the linked opportunity moves to "Won". The opportunity-number badge appears at the top of the detail panel. |
| **Projects** (ch. 09) | On acceptance (or through manual conversion), a **project** is created and linked (concurrency-safe and idempotent operation). The starting work orders are generated. |
| **Companies / Contacts** (ch. 04-05) | A quote's client references a company, a contact, or a manually entered name; the name is cached. |
| **Dossiers** (ch. 07) | On acceptance, the attachments are copied to the project; the quote appears in the 360° Record. |
| **Accounting** (ch. 15) | The **Create invoice** button creates a (draft) invoice from the quote. |
| **Takeoff** (ch. 30) | The takeoff measures quantities on a plan and sends them back here ("Apply to quote" / "Create quote"). |
| **CAD / 3D Render** (ch. 31) | The photorealistic 3D render attached to the quote comes from the render engine (billed to credits). |
| **Store / Products** (ch. 10) | Estimating an existing quote can consult the **product catalog** to anchor the prices. |
| **Employees** (ch. 11) | Assigning employees to a quote. |

### 5.2 FAQ

**How do I get a PDF of my quote?**
There is no direct PDF export. Do **Generate HTML → Preview → Print** and choose "Save as PDF", or let the client download/print from the public page.

**Does the "Profit 15%" line add to my total?**
No. With the "markup embedded" model, the line prices **already contain** the administration, contingencies and profit. The three summary lines only **break down** that markup. The total does not change whether they are shown or hidden.

**Can I change the 15% profit?**
Yes. The 15% is a **default**. You can enter any profit percentage (0 to 100%) on the quote, and even a different percentage per line. The document always uses the recorded value.

**What per-sq.ft. price does the AI use?**
There is no fixed value in the system. It is the **AI expert** (and its profile) that supplies the per-sq.ft. rate, per floor, based on the tier and the region. The server only applies the ×1.30 markup and checks that the result stays within a reasonable range (soft, non-blocking validation).

**Does "importing" a plan create the lines automatically?**
No. Import **analyzes** the document (category, areas, trades). You review, then click "Generate quote" to create the lines. The Takeoff, for its part, produces quantities that you transfer with "Apply to quote".

**Why doesn't the "Conditions" tab appear for me?**
It is reserved for **administrators**: it sets the whole company's default terms/exclusions.

**Is the 3D render free?**
No for **generation** (tenant AI credits ×1.30). Yes for **attaching** or **removing** an already-generated render.

**What is the difference between "AI Estimation" and "AI Assistant"?**
**AI Estimation** (tab 2) **generates** quotes. The **AI Assistant** (last tab) is **read-only**: it answers questions about your data but creates and modifies nothing.

**How long does the public link stay valid?**
90 days. After that, resend the quote to regenerate a link.

**Can the client accept without signing?**
No. Both the name **and** the drawn signature are required to accept.

**Can I delete an accepted quote?**
No (400 refusal). Change its status first, then delete.

**What exactly does the client see on the public page?**
The complete document, with zoom, print and download — but **without** the sensitive information (per-line markups, Labor/Material split, token, internal notes, metadata), which is stripped before sending.

**Do the CCQ and CNESST calculators account for hours?**
No. They work on a labor **amount** × a **rate** (per trade for the CCQ, a unit rate for the CNESST).

**Is there a "Duplicate" function?**
There is no duplication of a full quote. To start from a base, use the **Manual** tab (template) or **AI Estimation**, or recreate the quote and transfer lines.

**Does the system handle multiple currencies?**
Taxes and labels are the tenant's (by default GST 5% / QST 9.975% in Quebec). There is no multi-currency conversion in this module.

---

## 6. Summary

- **On-screen title: "Quotes"** (route `/devis`). Six tabs: Quotes, AI Estimation, Takeoff, Manual, Conditions (administrators), AI Assistant (read-only).
- **Three ways to build**: **manual** (line by line or the 9-section Quebec Construction Template), **AI** (expert conversation → "Generate quote"), **import** (plan/document analyzed by the AI, or Takeoff quantities). Import **never writes** the lines on its own: you review, then you generate.
- **"Markup embedded" pricing model**: the line prices already contain Administration (3%) + Contingencies (12%) + Profit (15%) ≈ ×1.30. The three summary lines are an **informational breakdown**, not amounts added on top. The **15% profit** is an **editable default**, not a lock. The **per-sq.ft. rate** comes from the AI, not from a constant.
- **Numbering** `DEV-YYYY-NNN`, fail-safe even under simultaneous clicks.
- **Sending**: status → Sent + 90-day public link + company-branded email. **Acceptance** by the client with a **mandatory drawn signature** → automatic creation of the **project** and the opportunity moving to "Won".
- **Exports**: Excel (.xlsx), QuickBooks CSV. **No direct PDF** — use Generate HTML + Print, or the public page.
- **3D render** optional and **paid** (generation billed to credits; attaching is free; a single one per quote).
- **AI**: Opus 4.8 for estimating (32,000 tokens), Sonnet for the read-only assistant; real cost ×1.30 deducted from credits, with a possible charge before the action (402 if the balance is insufficient).
- **Permissions**: everything is open to the team, except editing the company's **default terms** (administrators). **Deletion forbidden** if Accepted/Completed. **Read-only** mode if the subscription is past due.

---

*Verified source files: `backend/routers/devis.py` (13,085 lines, 57 endpoints) · `backend/routers/devis_ai.py` (344 lines) · `backend/routers/devis_manuel_template.py` (663 lines) · `frontend/src/pages/DevisPage.tsx` (3,052 lines) · `pages/DevisPublicPage.tsx` (538 lines) · `components/devis/EstimationIA.tsx` (1,728 lines) · `ConstructionTemplate.tsx` (1,130 lines) · `DevisRenderModal.tsx` (534 lines) · `AiProfileManager.tsx` (410 lines) · `DevisAssistantTab.tsx` (152 lines) · `api/devis.ts` · `api/devisAi.ts` · i18n `en/{devis,devisAssistant,devisRender}.json`.*

*Related manuals: 06 — CRM / Opportunities · 07 — Dossiers · 09 — Projects · 15 — Accounting · 30 — Takeoff · 31 — CAD / 3D Modeling.*

*Constructo AI ERP Manual — Module 07 "Quotes and estimates (manual, AI, import)" — v3.0 verified — 2026-07.*
