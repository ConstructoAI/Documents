# Module 07 — Dossiers (document management)

> **Version**: 3.0 (complete overhaul verified against the 2026-07 source code)
> **Frontend route**: `/dossiers` (sidebar menu "Dossiers", "Management" group, `FolderOpen` icon, between "Sales" and "Quotes"). Detail: `/dossier/:id` (**singular**). Public sharing page: `/dossiers/public/:token` (no authentication).
> **API prefix**: `/api/erp/v1` — note: the screen route is `/dossiers`, but **all endpoints live under `/api/erp/v1/documents`** (see 1.7).
> **Reference code (backend)**: `backend/routers/documents.py` (6138 lines, 58 routes — dossiers, attachments, folders, notes, links, steps, extras, sharing, public endpoints) · `backend/routers/dossiers_ai.py` (327 lines, 1 route — **read-only** Dossiers assistant) · `backend/routers/dossier_ai.py` (772 lines, 2 routes — **Extras/change-order** assistant, propose→confirm).
> **Reference code (frontend)**: `frontend/src/pages/DossiersPage.tsx` (428 lines — list) · `frontend/src/pages/DossierDetailPage.tsx` (3911 lines — 360° Record, 13 tabs) · `frontend/src/pages/DossierPublicPage.tsx` (561 lines — public page) · `frontend/src/components/dossiers/DossiersAssistantTab.tsx` (123 lines) · `frontend/src/components/dossiers/ExtrasAssistant.tsx` (321 lines).
> **Frontend API clients**: `api/documents.ts` (949 lines), `api/dossiersAi.ts` (35 lines). Note: **`api/dossiers.ts` does not exist** — every page imports from `@/api/documents`.
> **PostgreSQL tables (per tenant)**: `dossiers` (header), `attachments` (files stored as `BYTEA`), `dossier_folders` (folder tree), `dossier_notes`, `dossier_liens`, `dossier_etapes`, `dossier_extras` (change orders), and 5 link tables `dossier_devis` / `dossier_projets` / `dossier_formulaires` / `dossier_achats` / `dossier_factures`. Invoicing a change order: `factures` / `facture_lignes`. **Shared** table: `public.dossiers_public_tokens` (sharing tokens, `public` schema). Shared tables (AI path): `public.ai_prepaid_credits`, `public.ai_usage_tracking`.
> **Scope**: despite its "document management" name, this module is really a **project 360° Record**. A dossier is a **hub** that gathers, around a single jobsite or client, the opportunity, the quotes, the project, the work orders, the purchases, the price requests, the invoices, the time tracking, the accounting (margin), a **document tree**, **notes** (with AI), clickable **links**, and billable **extras/change orders**. Every list row even carries a "360° Record" label. It replaces neither the Quotes module (which prices the quotes) nor the Accounting module (which issues the invoices): it **connects** them and gives an overview.

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

The Dossiers module is used to **bring together in one place everything related to a jobsite or a client**. Instead of hunting for a quote in one module, an invoice in another and photos in a third, you open **a single dossier** and you see:

- the **quotes**, the **project**, the **work orders**, the **purchases**, the **price requests** and the **invoices** attached;
- employee **time tracking** and jobsite **accounting** (revenue, costs, estimated margin);
- a **tree of folders and documents** (plans, photos, contracts, correspondence);
- jobsite **notes**, with AI tools to enrich them, analyze a photo or summarize the whole set;
- useful external **links** (online permits, the client's cloud folder, etc.);
- contract **extras (change orders)**, which you can track and **invoice** in one click.

The dossier is also your **sharing tool**: you can generate a secure public link so a client or subcontractor can view (and, if you allow it, upload) documents, without having an ERP account.

### 1.2 Access from the sidebar

Click **Dossiers** in the sidebar ("Management" group). The page opens on the **list** of dossiers (`/dossiers`).

Two quick links are useful:

- `app.constructoai.ca/dossiers?open=<id>` opens the **360° Record** of a dossier **directly**. This is the link used by the "View dossier" buttons elsewhere in the ERP (the Sales/CRM module, for example). The `open` parameter is consumed then removed from the address (`DossiersPage.tsx:142-152`).
- `app.constructoai.ca/dossiers/public/<token>` is the **public page** shared with a third party (see 2.13).

> **Address nuance**: the 360° Record lives at `/dossier/<id>` (**singular**), whereas the list and the public page are under `/dossiers` (plural). This is intentional in the code (`App.tsx:221-222`).

### 1.3 The three screens

| Screen | Route | Authentication | Role |
|--------|-------|----------------|------|
| **Dossier list** | `/dossiers` | Yes | Search, filter, create, delete, open the AI assistant |
| **360° Record (detail)** | `/dossier/:id` | Yes | The 13 tabs, the heart of the module |
| **Public page** | `/dossiers/public/:token` | **No** | View (and, depending on the level, upload) shared documents |

### 1.4 Automatic numbering

On creation, each dossier receives a **`DOS-YYYY-NNNNN`** number (e.g. `DOS-2026-00042`): the current year followed by the dossier's 5-digit identifier (`documents.py:638`). The number is generated in a **fail-proof way even under simultaneous clicks**: the dossier is first inserted with a temporary `TEMP` number, then updated with its final number in the same transaction (`documents.py:615-640`). A `COUNT + 1` that could produce duplicates is never used.

**Extras** follow the same principle with the **`EXT-XXXX`** format (e.g. `EXT-0001`, `documents.py:5834`). The **invoice** produced when invoicing an extra receives a **`FACT-YYYY-NNNNN`** number managed by the Accounting module.

### 1.5 Statuses, types and priorities

**5 dossier statuses** (`DOSSIER_STATUTS` constant, `documents.py:57`). The status is **forced to `OUVERT` on creation**; it then changes via the dropdown at the top of the 360° Record.

| Value (database) | Displayed label | Badge colour |
|------------------|-----------------|--------------|
| `OUVERT` | Open | Blue |
| `EN_COURS` | In progress | Green |
| `EN_ATTENTE` | On hold | Yellow |
| `TERMINE` | Completed | Teal |
| `ARCHIVE` | Archived | Grey |

In the list indicators, "Open" groups `OUVERT + EN_COURS + EN_ATTENTE`, and "Completed" groups `TERMINE + ARCHIVE` (`documents.py:472-473`).

**5 dossier types** (creation modal dropdown): **Project**, **Client**, **Supplier**, **Administrative**, **Other**.

**4 priorities** (`PRIORITES` constant, `documents.py:62`) — database values `BASSE` / `NORMAL` / `HAUTE` / `URGENT`, displayed labels **Low** / **Normal** / **High** / **Urgent**.

| Priority | Badge colour |
|----------|--------------|
| Urgent | Red |
| High | Orange |
| Normal | Blue |
| Low | Grey |

### 1.6 Permissions and roles

The module is **deliberately open to the whole team**: almost every endpoint simply requires a valid tenant ERP account (`Depends(get_current_user)`). In other words, any authenticated user can create a dossier, attach documents to it, write notes, link items and manage shares. Isolation between companies remains guaranteed server-side (each request is scoped to the tenant's schema, and each dossier is re-checked as truly belonging to that tenant — protection against accessing someone else's identifier).

**Only one action is restricted** to administrators and accountants: **invoicing an extra**. The `POST …/extras/{eid}/facturer` endpoint is protected by `require_tenant_admin_or_role("comptable")` (`documents.py:6074`) — you must be `is_admin` (re-read server-side, tamper-proof), have the `admin` or `comptable` role, or be a super administrator. The same rule applies to **auto-invoicing** (moving an extra to "Approved") and to invoicing requested via the Extras AI assistant.

| Action | Who is allowed |
|--------|----------------|
| View, create, edit, delete a dossier; documents, notes, links, steps, associations, shares | Any valid tenant ERP account |
| Create / edit / delete an **extra** | Any valid ERP account (but **invoicing** is blocked, see below) |
| **Invoice** an extra (or approve an extra that triggers auto-invoicing) | `admin`, `comptable`, `is_admin` or super administrator |

**Read-only mode (view only).** If the tenant's Stripe subscription is past due, the account can switch to **read-only** mode: reads work, but **any write returns 403**. This control is applied upstream, in `get_current_user` (`erp_auth.py:520-528`), and therefore covers **all** the module's authenticated endpoints (dossier creation, upload, notes, extras, AI). The **4 public token endpoints** do not go through `get_current_user`: they are therefore not subject to read-only mode, but they still stop working if the company is deactivated (see 2.13).

### 1.7 Why "Dossiers" on screen and "documents" in the address?

This is a historical legacy of the code: the module's first internal name was "documents". The endpoint prefix stayed `/documents` (`documents.py:50` + `API_PREFIX`, hence `/api/erp/v1/documents/…`), while the interface was renamed "Dossiers" for clarity. The only place the word "dossiers" appears on the API side is the read-only assistant (`/api/erp/v1/dossiers/ai/chat`). No old `dossier.py` file exists: **the backend is indeed `routers/documents.py`**.

### 1.8 What the module does not do (verified in the code)

- **No "Steps" tab** in the 360° Record. The backend can nonetheless manage a list of steps (`GET/POST …/etapes` + `…/toggle`), and the client API exists, but **no tab displays it** in the current version. It is a dormant feature.
- **No category selector on upload**: the 10 document categories (PLAN, PHOTO, CONTRAT…) exist server-side, but the upload interface does not offer them. An imported document stays without an explicit category.
- **No "Archive" button**: "Archived" is a **status** (set via the 360° Record dropdown), not a list action. The list only offers New dossier, Refresh, AI Assistant and Delete.
- **No separate "Statistics" view**: the 3 indicators (Total / Open / Completed) are always shown at the top of the list; there is no tab to switch to.
- **Minimal creation modal**: only Name, Type and Priority are requested. Description, notes and status are not exposed on creation (the status is forced to `OUVERT`).
- **No CSV/PDF export of the list, no printing, no bulk action.** The only file outputs are downloading an attachment and public sharing by link.
- **Invoicing an extra produces a DRAFT invoice** (it is not automatically sent to the client).
- **The Dossiers AI assistant is read-only** and **does not read the content of attached files**: it works on metadata (dossiers, notes, steps, links, extras, linked invoices).

---

## 2. Interface

### 2.1 List screen — `/dossiers`

**Command bar** (top):

| Button | Icon | Effect |
|--------|------|--------|
| **New dossier** | `Plus` (blue) | Opens the creation modal (see 2.2). |
| **Refresh** | `RefreshCw` | Reloads the list **and** the indicators. |
| **AI Assistant** | `Sparkles` | Opens a large modal with the **read-only** Dossiers assistant (see 2.14). |

To the right of the bar: a **search field** ("Search...") and a **status filter** (dropdown: All statuses, Open, In progress, On hold, Completed, Archived).

**Search and filter.** Search is done **server-side** with a slight 300 ms delay after typing (`DossiersPage.tsx:79-85`). It covers the dossier **title** and **number**, across **the whole tenant** (not just the displayed page), and resets the display to page 1. The status filter does the same.

**Indicators (always visible once loaded)** — source `GET /documents/statistics`:

| Indicator | Meaning |
|-----------|---------|
| **Total** | Total number of dossiers. |
| **Open** (blue) | `OUVERT + EN_COURS + EN_ATTENTE` dossiers. |
| **Completed** (green) | `TERMINE + ARCHIVE` dossiers. |

**Table (desktop)** — sortable columns (click the header) and resizable:

| Column | Content |
|--------|---------|
| **Name** | Folder icon + title + blue "360° Record" mention. |
| **Type** | Translated label (Project / Client / Supplier / Administrative / Other). |
| **Status** | Coloured badge. |
| **Priority** | Coloured badge. |
| **Updated** | Last modification date. |
| **Actions** | **Delete** button (trash). |

Clicking anywhere on a row opens the 360° Record (`/dossier/:id`). On mobile, the same information is shown as **cards**. At the bottom: the total and **pagination** (20 dossiers per page). If the list is empty: "No dossiers".

**Deletion from the list.** The trash button asks for detailed confirmation: deletion erases **attachments, notes and steps**, **detaches** (without deleting them) the linked opportunities and projects, cascade-deletes the **expenses**, and **is irreversible**. A success message confirms that dossier "…" was deleted.

### 2.2 "New dossier" modal

Only three fields:

| Field | Detail |
|-------|--------|
| **Name** * | Required (free text, e.g. "Dupont kitchen renovation"). |
| **Type** | Dropdown: Project / Client / Supplier / Administrative / Other. |
| **Priority** | Dropdown: Low / Normal / High / Urgent. |

**Cancel** / **Create** buttons. A lock prevents double submission. On error, the message is shown **in the modal**. The status is not requested (it is `OUVERT`). On validation, the `DOS-YYYY-NNNNN` dossier is created and you are taken to its 360° Record.

### 2.3 360° Record — header

At the top of the 360° Record (`DossierDetailPage.tsx:216-308`):

- A **back** button (left arrow).
- The **title**, **editable in place**: click the pencil, a field appears (255 characters max) with confirm/cancel buttons; **Enter** saves, **Esc** cancels.
- A **Delete dossier** button (trash), with the same cascade warning as the list.
- The **status**, editable in place via a dropdown of the 5 values, with immediate on-screen update.
- A metadata line: the **dossier number** (monospace font), an **opportunity number** badge if the dossier comes from the CRM, and the **client name**.

### 2.4 360° Record — the 13 tabs

The navigation bar has **13 tabs** (`DossierDetailPage.tsx:62-76`). Some show a **counter** of the number of linked items.

| # | Tab | Counter? | What you find there |
|---|-----|----------|---------------------|
| 1 | **Summary** | — | Financial indicators, linked opportunity, progress. |
| 2 | **Quotes** | Yes | Attached quotes. |
| 3 | **Project** | Yes | Attached project. |
| 4 | **Work Orders** | Yes | Attached work orders. |
| 5 | **Purchases** | Yes | Attached purchase orders. |
| 6 | **Price Requests** | Yes | Attached price requests. |
| 7 | **Invoices** | Yes | Attached invoices. |
| 8 | **Time Tracking** | Yes | Hours and costs per employee. |
| 9 | **Accounting** | — | Dossier revenue, costs and margin. |
| 10 | **Documents** | Yes | Folder and file tree, shares. |
| 11 | **Notes** | — | Jobsite notes + AI tools. |
| 12 | **Links** | — | Clickable external links. |
| 13 | **Extras** | — | Change orders and their invoicing. |

> **There is no "Steps" tab** despite its existence server-side (see 1.8). **Share management** is not a tab: it is built into the **Documents** tab ("Share" buttons).

### 2.5 Summary tab

- **8 indicators**: Total budget, Invoiced, Paid, Balance due, Purchases, Labour, Margin, and "WO / PO / PR" (counters of work orders, purchase orders and price requests).
- An **Opportunity** card if the dossier comes from the CRM: name, client, amount, source, status badge.
- A **Progress** card: a 5-step timeline — Opportunity → Quote → Project → Work → Invoicing — each step turning green when reached.

### 2.6 Link and unlink items (tabs 2 to 7)

The six tabs **Quotes, Project, Work Orders, Purchases, Price Requests, Invoices** share the same linking mechanism (`LinkableSection`, `DossierDetailPage.tsx:459-581`). Each one shows, in addition to its list:

- a **"+ Link {type}"** button: it opens a dropdown of **linkable** items (those that exist in the relevant module and are not already linked, loaded via `GET /documents/{id}/linkable`); the choice creates the link;
- a **"Remove a link"** area: clickable `#id` chips remove the link.

Removing a link **never deletes the item itself**: the quote, project, invoice, etc. stay intact; only the link to the dossier disappears.

The content of each tab in **read** mode:

| Tab | Columns / info | Opens to |
|-----|----------------|----------|
| **Quotes** | Number, status, amount, date | `/devis?open=…` |
| **Project** | Budget, planned/actual dates | `/projets?open=…` |
| **Work Orders** | No., name, status, priority, amount, due date | `/bons-travail?open=…` |
| **Purchases** | No., supplier, status, amount, order/delivery dates | `/magasin?open=…` |
| **Price Requests** | No., name, status, priority, amount, due date | (number not clickable) |
| **Invoices** | No., client, status, amount incl. tax, paid, balance due, date | `/comptabilite?open=…` |

If a tab has no item, it shows a message like "No quote linked to this dossier".

### 2.7 Time Tracking tab

Two tables (`PointageSection`):

- **Summary per employee**: Employee, Hours, Cost, number of entries;
- **Recent entries** (20 maximum): Employee, clock-in time, clock-out time, Hours, Type.

### 2.8 Accounting tab

Aggregated financial view of the dossier, in two cards:

- **Revenue**: Total budget, Total invoiced, Total paid, Balance receivable, Paid invoices, Overdue invoices;
- **Costs and margin**: Hours, Labour cost, Purchases, Supplier invoices, Total costs, **Estimated margin**, **Margin %**.

These figures are computed on the fly by the server's "360" view (see 4.9). Two useful subtleties: the labour cost shown is the **actual payroll cost** as soon as it exists (otherwise an estimate from hourly rates), and the calculation **avoids double counting** (purchases already represented by a supplier invoice are not counted twice; credits are subtracted from revenue).

### 2.9 Documents tab (the richest)

This is a true **tree of folders and files**, up to **5 levels deep**.

**Header bar**: a "{n} documents" counter and three buttons:

| Button | Icon | Effect |
|--------|------|--------|
| **New folder / subfolder** | `FolderPlus` | Creates a folder or subfolder (disabled at level 5). |
| **Share** | `Share2` | Shares the **entire dossier** (see 2.9.4). |
| **Add a document** | `Upload` | Uploads one or more files (up to **150 MB** each). |

**Breadcrumb**: Root → … lets you go back up the tree.

**Drag and drop**: a drop zone accepts files directly, with a progress bar (file name, "File x/y", percentage).

**Display**: **folders** appear first, then **files**.

#### 2.9.1 Actions on a folder

Open · **Share** (`Share2`) · **Rename** (`Pencil`) · **Move** (`FolderInput`) · **Copy** (`Copy`) · **Delete** (`Trash2`).

When **deleting a folder**, a modal asks for the strategy:

- **"Move contents up to the parent"**: subfolders and documents are re-attached to the parent folder (they are not lost);
- **"Delete everything"**: the entire subtree and its files are permanently erased.

#### 2.9.2 Actions on a file

For files you uploaded (source "attachments"): clicking the name opens the **preview**; the buttons offer **Preview** (`Eye`), **Download**, **Rename** (`Pencil`, the original extension is kept), **Move** (`FolderInput`), **Copy** (`Copy`) and **Delete** (`Trash2`). Documents from other sources (for example old tables) appear read-only (name only).

The **preview** opens in a built-in viewer for supported types (images, PDF, text).

#### 2.9.3 Move / copy (including to another dossier)

A unified modal handles moving and copying. It lets you choose **another destination dossier** (server-side search, with a typing delay) then a **target folder** in an indented tree. The subtree you are moving is excluded from the possible destinations (you cannot move a folder into itself). A warning specifies that **moving to another dossier revokes the shares** of the affected folder.

#### 2.9.4 Share the entire dossier

The **Share** button (header) generates a public link of the form `…/dossiers/public/{token}`. The modal shows:

- **statistics**: number of views and downloads, and the latest dates;
- the **Created** and **Expires** dates (the entire-dossier link expires after **90 days**);
- two buttons: **Regenerate** (rotates the token — the old link stops working immediately) and **Revoke** (disables sharing).

An entire-dossier link gives read access to **all** the dossier's documents (flat-list mode).

#### 2.9.5 Share a single folder (targeted sharing)

The **Share** button of a **folder** opens a finer modal, with two settings:

- the **access level**: **Reader only** (`reader` — online viewing, download refused), **Read + download** (`downloader`), or **Contributor** (`contributor` — viewing, download **and** upload by the guest);
- the **expiration**: **30 days**, **90 days** or **Never** (default 90 days).

The modal then shows the link, its statistics, its level, its expiration date, and lets you revoke it. A folder share only gives access to that folder's **subtree**.

### 2.10 Notes tab

**Add form**: a text area, with tools:

- **Enhance with AI** (`Sparkles`): rewrites and structures the note;
- **AI photo analysis** (`Image`): upload a jobsite photo; the AI describes it (finding, severity, location, recommendations);
- **AI dossier summary** (`Bot`): a synthesis of all the notes;
- **Attach files** (`Paperclip`, several at once);
- **Audio recording** (`Mic` / `Square`, with an mm:ss timer) — handy for dictating a note on the jobsite;
- **Add note** button.

After an enhancement, an **"Actions identified by AI"** panel may appear. The **AI summary** shows a summary, the **"Open issues"**, the **"Pending actions"** and the number of analyzed notes.

**Notes list**: each note carries a **category badge** (defaut, observation, progression, decision, action, general), a **pin** indicator, its author and date. Attachments are displayed by type:

- **images** as a direct preview, with enlargement and download;
- **audio** with a built-in player;
- **other files** as download buttons.

Per note: **Pin / Unpin** and **Delete**. Pinned notes move to the top.

> Note attachments are limited to **10 files of 15 MB** per addition, and photo analysis to **15 MB**. This is separate from document upload (150 MB).

### 2.11 Links tab

**Form**: a **URL** * (required, up to 2048 characters, must start with `http://` or `https://` and contain no space) and a **Description** (up to 1000 characters, with a counter). The **list** shows each link (opens in a new tab, safely), its description and its addition date, with in-place editing (`Pencil`) and deletion (`Trash2`).

These links point to external resources: the client's cloud folder, an online municipal permit, plans hosted elsewhere, etc.

### 2.12 Extras tab (change orders)

The Extras tab manages **contract change orders** — the additional work requested during the job — and lets you **invoice** them.

**"AI Assistant" toggle button**: opens the Extras assistant (see 2.14), which can propose creating/editing/invoicing an extra **upon confirmation**.

**4 total cards**: **Approved**, **To invoice**, **Invoiced** (with, where applicable, the note "− X in credits" and "+ X in draft (to send)"), and **Total**. These totals account for **credits** (credit notes) and distinguish invoices still in **draft**.

**Add form**: **Description** *, **Amount** (before tax, up to $10,000,000), **Date**. **Add extra** button.

**Extras list**: number (`EXT-XXXX`), status badge, amount, description, date, and "Invoiced — {number}" where applicable.

| Extra status | Label | Colour |
|--------------|-------|--------|
| `PROPOSE` | Proposed | Yellow |
| `APPROUVE` | Approved | Blue |
| `REFUSE` | Refused | Red |
| `FACTURE` | Invoiced | Green |

**Per-row actions**:

- a **status dropdown**: Proposed / Approved / Refused;
- an **"Invoice"** button (visible if the extra is **Approved** and its amount positive) → it creates a **DRAFT invoice** linked to the dossier;
- **Edit** / **Delete**.

Two important rules:

- an **Invoiced extra is locked**: neither status, nor description/amount, nor deletion (the server returns a 409 error if you try);
- moving an extra to **Approved** may trigger **immediate invoicing** ("auto-invoice") **only if you have the right to invoice** (admin/accountant) **and** the dossier has a client. A user without that right can approve an extra without anything being invoiced — it is the "Invoice" button (or an administrator) that will trigger the invoice later.

### 2.13 Public screen — `/dossiers/public/:token`

This is the page a client or subcontractor opens **without an account**, from the link you sent them. The server recognizes the **token** and presents the authorized content.

**Two modes** depending on the link type:

- **entire dossier**: the flat list of all documents;
- **targeted folder**: read-only tree navigation, limited to the shared subtree.

**Header**: the company name, the dossier title, its number, a status badge, a "Shared documents" counter and a **FR / EN language toggle**.

**Access-level banner**:

- **Reader only** level: amber banner "Online viewing: download disabled";
- **Contributor** level: blue banner inviting drag-and-drop of files.

**Per file**: a **"View"** button (if the type is previewable: image, text or PDF) and a **"Download"** button (except at Reader only level).

**Guest upload area**: only at **Contributor** level — the guest can drop files, which land in the shared folder.

**Public page security** (transparent to the user, but useful to know): the link is protected against abuse by **rate limits** (by IP address first, then by token), guest upload is capped (5,000 files / 10 GB cumulative per dossier), served files are verified by their real signature (a disguised file is neutralized) and forced to download, and **the link stops working if the company's subscription is deactivated**. A footer reminds "Secure link — {company} · Read-only documents".

### 2.14 The two AI assistants

The module has **two distinct assistants** (they are not two variants of the same one):

| Assistant | Where | Nature |
|-----------|-------|--------|
| **Dossiers AI assistant** | "AI Assistant" button of the **list** (modal) | **Read-only.** It queries your dossiers, notes, steps, links, extras and linked invoices and answers in natural language. It **writes nothing** and **does not read the content of attached files**. |
| **Extras AI assistant** | "AI Assistant" button of the **Extras tab** | **Write on confirmation.** It can propose to create an extra, change its status, invoice it, edit it or delete it. Each proposal is shown as a **card** (fields + totals) that you **confirm** or **cancel**. Nothing is written until you confirm. |

Common points: both answer in French or English, keep a bounded history (40 exchanges), prevent double submission, and **consume AI credits** (the module therefore has a cost, see 4.16). The Extras assistant re-checks your rights **at confirmation time** for an invoicing action: a user without invoicing rights cannot bypass the rule through the AI.

---

## 3. Step-by-step workflows

### 3.1 Create a dossier

1. Dossier list → **New dossier**.
2. Enter the **Name** (required), choose the **Type** and the **Priority**.
3. **Create**: the `DOS-YYYY-NNNNN` dossier is created with status **Open** and its 360° Record opens.

### 3.2 Rename a dossier or change its status

- **Rename**: in the 360° Record header, click the **pencil** next to the title, type the new name, press **Enter** (or **Esc** to cancel).
- **Change status**: use the status **dropdown** in the header (Open / In progress / On hold / Completed / Archived). The change is immediate.

> To "archive" a dossier, simply set its status to **Archived**. There is no separate "Archive" button.

### 3.3 Attach a quote, a project, an invoice, etc.

1. Open the corresponding tab (Quotes, Project, Work Orders, Purchases, Price Requests or Invoices).
2. Click **"+ Link {type}"**.
3. Choose the item in the dropdown (only items not yet linked appear). The link is created immediately.

You can attach a single dossier to several quotes, several invoices, etc. Linking is **idempotent**: linking the same item twice does not create a duplicate.

### 3.4 Detach an item

1. In the same tab, **"Remove a link"** section.
2. Click the `#id` chip of the item to detach.

The original item (quote, project…) stays intact; only the link disappears.

### 3.5 Organize documents into folders

1. **Documents** tab → **New folder**; name it.
2. To create a subfolder, first open the parent folder, then **New subfolder** (up to 5 levels).
3. To **move** a file or folder: **Move** button → choose the destination (same dossier or another dossier, then the target folder).
4. To **copy**: **Copy** button (a folder is copied with all its content).

### 3.6 Upload documents

1. **Documents** tab → **Add a document** (or drag the files into the drop zone).
2. Select one or more files (150 MB max each).
3. The progress bar shows the progress; the files appear in the current folder.

> The file type is unrestricted. There is no category choice at this step.

### 3.7 Download, rename or delete a document

- **Download**: **Download** button on the file's row.
- **Preview**: click the name, or the **Preview** button (images, PDF, text).
- **Rename**: **Rename** button; the original extension is kept automatically.
- **Delete**: **Delete** button (the file is erased from the database).

### 3.8 Share the entire dossier with a client

1. **Documents** tab → **Share** (header).
2. The `…/dossiers/public/{token}` link is generated (valid for **90 days**). Copy it and send it to the client.
3. Track **views** and **downloads** in the same modal.
4. To cut off access: **Revoke**. To renew the link (and invalidate the old one): **Regenerate**.

### 3.9 Share a single folder with an access level

1. **Documents** tab → **Share** button on a **folder's row**.
2. Choose the **level**:
   - **Reader only**: the third party views online but cannot download;
   - **Read + download**: they view and download;
   - **Contributor**: they view, download **and** can drop their own files.
3. Choose the **expiration** (30 days, 90 days or Never).
4. Copy the link and send it. You can revoke it at any time.

> Use **Contributor** to receive documents from a subcontractor (photos, spec sheets) without creating an account for them.

### 3.10 View a public link (client side, no account)

1. The client opens the received link.
2. They choose their **language** (FR / EN) if needed.
3. They **view** the documents; they **download** if the level allows; they **drop** files if they are a Contributor.

### 3.11 Write a note and enhance it with AI

1. **Notes** tab → type your note in the text area (or dictate it with the mic).
2. Attach photos/files if needed (up to 10 × 15 MB).
3. **Add note.**
4. To structure it: **Enhance with AI**. To extract a finding from a photo: **AI photo analysis**. For a synthesis of the whole dossier: **AI dossier summary**.

> Each AI call consumes credits (see 4.16).

### 3.12 Add an external link

1. **Links** tab → paste the **URL** (it must start with `http://` or `https://`).
2. Add a short **description**.
3. Confirm. The link then opens in a new tab, safely.

### 3.13 Create, approve and invoice an extra (change order)

1. **Extras** tab → fill in **Description**, **Amount** (before tax) and **Date** → **Add extra**. It is born with status **Proposed**.
2. Have it validated by the client, then move it to **Approved** (status dropdown).
3. **Invoice**:
   - if you are an **administrator or accountant** and the dossier has a **client**, moving the extra to "Approved" may trigger the invoice **automatically**;
   - otherwise, click the **"Invoice"** button on the extra's row (reserved for administrators/accountants).
4. A **DRAFT invoice** `FACT-YYYY-NNNNN` is created and linked to the dossier; the extra moves to **Invoiced** and locks. Finish sending the invoice from the Accounting module.

> An extra with no client attached to the dossier **cannot** be invoiced (the system returns an explicit error). Attach a client to the dossier first.

### 3.14 Query the Dossiers AI assistant (read)

1. Dossier list → **AI Assistant**.
2. Ask a question about your dossiers (e.g. "Which dossiers have a balance due?", "Summarize the approved extras of jobsite X").
3. The assistant reads your data and answers. It **writes nothing** and does not read file content.

### 3.15 Use the Extras AI assistant (propose → confirm)

1. **Extras** tab → **AI Assistant** toggle.
2. Describe what you want (e.g. "Add a $3,200 extra for the French drain, dated today").
3. The AI shows a **proposal card** (fields + total).
4. Check it, then **Confirm** (the extra is actually created/edited/invoiced) or **Cancel**. A confirmed invoicing re-checks your rights.

### 3.16 Delete a dossier

1. 360° Record header (or the list's trash button) → **Delete**.
2. Confirm the warning.
3. The server **cascade-deletes** the attachments, notes, steps, links, extras and the 5 link tables, purges the sharing tokens, then **detaches** (sets to NULL) the linked opportunities and projects (they are not deleted). The operation is **irreversible**.

> Linked quotes, projects, purchase orders and invoices **stay in the database** — they only lose their link to the dossier.

---

## 4. Reference

> Reminder: the base of all endpoints is `/api/erp/v1`. "read" = `Depends(get_current_user)` (any tenant account). Writes are automatically blocked in **read-only mode** (view only, driven by Stripe).

### 4.1 Screens and routes

| Screen | Front route | Component |
|--------|-------------|-----------|
| List | `/dossiers` | `DossiersPage.tsx` |
| Detail (360° Record) | `/dossier/:id` | `DossierDetailPage.tsx` |
| Public page | `/dossiers/public/:token` | `DossierPublicPage.tsx` |

### 4.2 Endpoints — dossier (CRUD)

| Method | Path | Guard | Reference |
|--------|------|-------|-----------|
| GET | `/documents/statistics` | read | `documents.py:451` |
| GET | `/documents` | read | `documents.py:488` |
| GET | `/documents/{id}` | read | `documents.py:574` |
| POST | `/documents` | read | `documents.py:607` |
| PUT | `/documents/{id}` | read | `documents.py:663` |
| DELETE | `/documents/{id}` | read | `documents.py:706` |

> `PUT /documents/{id}` accepts only 4 fields: **titre**, **statut**, **priorite**, **notes** (allowlist). `DELETE` locks the dossier, purges 13 child tables and the sharing tokens, then sets `opportunities.dossier_id` and `projects.dossier_id` to NULL.

### 4.3 Endpoints — attachments (`BYTEA` files)

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| POST | `/documents/{id}/attachments` | Upload (max **150 MB** → 413) | `documents.py:1005` |
| GET | `/documents/{id}/attachments` | List | `documents.py:1090` |
| GET | `/documents/{id}/attachments/{att_id}/download` | Download | `documents.py:1127` |
| GET | `/documents/{id}/attachments/{att_id}/preview` | Inline preview (hardened) | `documents.py:1177` |
| DELETE | `/documents/{id}/attachments/{att_id}` | Delete | `documents.py:1287` |
| PATCH | `/documents/{id}/attachments/{att_id}/folder` | Move (dossier/folder) | `documents.py:1937` |
| PATCH | `/documents/{id}/attachments/{att_id}` | Rename (extension kept) | `documents.py:1997` |
| POST | `/documents/{id}/attachments/{att_id}/copy` | Copy | `documents.py:2059` |

### 4.4 Endpoints — folders (tree, depth ≤ 5)

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| POST | `/documents/{id}/folders` | Create | `documents.py:1509` |
| GET | `/documents/{id}/folders` | List | `documents.py:1571` |
| PUT | `/documents/{id}/folders/{fid}` | Rename / move | `documents.py:1607` |
| DELETE | `/documents/{id}/folders/{fid}` | Delete (`reparent` or `cascade`) | `documents.py:1766` |
| PATCH | `/documents/{id}/folders/{fid}/move` | Move the subtree (cross-dossier) | `documents.py:2137` |
| POST | `/documents/{id}/folders/{fid}/copy` | Copy recursively | `documents.py:2322` |

### 4.5 Endpoints — notes (+ AI)

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| GET | `/documents/{id}/notes` | List (100 max, pinned first) | `documents.py:2609` |
| POST | `/documents/{id}/notes` | Create (1 to 20,000 characters) | `documents.py:2662` |
| POST | `/documents/{id}/notes-with-files` | Create with files (**10 × 15 MB**) | `documents.py:2700` |
| GET | `/documents/{id}/notes/{nid}/attachment/{idx}` | Download a note attachment | `documents.py:2783` |
| DELETE | `/documents/{id}/notes/{nid}` | Delete | `documents.py:2832` |
| PATCH | `/documents/{id}/notes/{nid}/pin` | Pin / unpin | `documents.py:3468` |
| PATCH | `/documents/{id}/notes/{nid}/categorie` | Recategorize | `documents.py:3508` |
| POST | `/documents/{id}/notes/ai/enrich` | **AI**: enrich (credits) | `documents.py:3083` |
| POST | `/documents/{id}/notes/ai/analyze-photo` | **AI Vision**: analyze a photo (**15 MB**, credits) | `documents.py:3187` |
| POST | `/documents/{id}/notes/ai/summary` | **AI**: summarize (≤ 200 notes, credits) | `documents.py:3326` |

### 4.6 Endpoints — links, steps, linked items, 360

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| GET/POST/PUT/DELETE | `/documents/{id}/liens[/{lid}]` | External links (URL `http(s)` ≤ 2048) | `documents.py:2871`+ |
| GET/POST | `/documents/{id}/etapes` | Steps (**no UI tab**) | `documents.py:2482`, `2518` |
| PUT | `/documents/{id}/etapes/{eid}/toggle` | Check/uncheck a step | `documents.py:2555` |
| GET | `/documents/{id}/linked` | Linked items (summary) | `documents.py:3553` |
| GET | `/documents/{id}/360` | Aggregated view (Summary/Accounting) | `documents.py:3646` |

### 4.7 Endpoints — link / unlink items

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| POST | `/documents/{id}/link` | Link an item | `documents.py:5488` |
| DELETE | `/documents/{id}/link/{type}/{item_id}` | Unlink | `documents.py:5538` |
| GET | `/documents/{id}/linkable?item_type=` | Linkable items | `documents.py:5577` |

**Types and link tables** (`LINK_TABLES`, `documents.py:5446`): `devis` → `dossier_devis` · `projet` → `dossier_projets` · `bon_travail` → `dossier_formulaires` (filter `BON_TRAVAIL`) · `bon_commande` → `dossier_achats` · `facture` → `dossier_factures` · `demande_prix` → `dossier_formulaires` (filter `DEMANDE_PRIX`).

### 4.8 Endpoints — token sharing

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| POST | `/documents/{id}/share` | Generate / regenerate the dossier link (**90 d**) | `documents.py:4492` |
| DELETE | `/documents/{id}/share` | Revoke the dossier link | `documents.py:4530` |
| GET | `/documents/{id}/share-info` | Token + statistics | `documents.py:4585` |
| POST | `/documents/{id}/folders/{fid}/share` | Folder link (level + expiration) | `documents.py:4621` |
| DELETE | `/documents/{id}/folders/{fid}/share` | Revoke the folder link | `documents.py:4700` |
| GET | `/documents/{id}/folders/{fid}/share-info` | Token + statistics (folder) | `documents.py:4760` |
| GET | `/documents/{id}/shares` | List all active links | `documents.py:4814` |

### 4.9 Endpoints — public page (no authentication)

| Method | Path | Role | Reference |
|--------|------|------|-----------|
| GET | `/documents/public/{token}` | Metadata + document list | `documents.py:4934` |
| GET | `/documents/public/{token}/attachments/{att_id}` | Inline preview | `documents.py:5114` |
| GET | `/documents/public/{token}/attachments/{att_id}/download` | Download (**refused at `reader` level**) | `documents.py:5193` |
| POST | `/documents/public/{token}/upload` | Upload (**`contributor` only**) | `documents.py:5285` |

### 4.10 Endpoints — extras (change orders) and invoicing

| Method | Path | Guard | Reference |
|--------|------|-------|-----------|
| GET | `/documents/{id}/extras` | read | `documents.py:5694` |
| POST | `/documents/{id}/extras` | read (creates as **Proposed**) | `documents.py:5797` |
| PUT | `/documents/{id}/extras/{eid}` | read (lock if invoiced → 409; auto-invoice if admin/accountant) | `documents.py:5859` |
| DELETE | `/documents/{id}/extras/{eid}` | read (409 if invoiced) | `documents.py:5964` |
| POST | `/documents/{id}/extras/{eid}/facturer` | **`require_tenant_admin_or_role("comptable")`** | `documents.py:6072` |

> Invoicing produces a **DRAFT invoice** via the Accounting module (`_create_invoice_from_source`), with the tenant's taxes and a `FACT-YYYY-NNNNN` number, then links the invoice to the dossier and moves the extra to **Invoiced**.

### 4.11 Endpoints — AI assistants

| Assistant | Method | Path | Nature | Reference |
|-----------|--------|------|--------|-----------|
| **Dossiers (read-only)** | POST | `/dossiers/ai/chat` | Queries the database (allowlisted SELECT) | `dossiers_ai.py:209` (mounted `erp_api.py:1021`) |
| **Extras (propose→confirm)** | POST | `/documents/ai/chat` | Proposes actions on extras | `dossier_ai.py:490` (mounted `erp_api.py:1137`) |
| **Extras (propose→confirm)** | POST | `/documents/ai/confirm-action` | Executes the confirmed action (re-checks rights) | `dossier_ai.py:666` |

### 4.12 Statuses, types, priorities, categories

| Set | Values (database) |
|-----|-------------------|
| Dossier statuses (`DOSSIER_STATUTS`, `documents.py:57`) | OUVERT · EN_COURS · EN_ATTENTE · TERMINE · ARCHIVE |
| Dossier types (UI) | Project · Client · Supplier · Administrative · Other |
| Priorities (`PRIORITES`, `documents.py:62`) | BASSE · NORMAL · HAUTE · URGENT |
| Document categories (`DOCUMENT_CATEGORIES`, `documents.py:52`) — **defined but not exposed on screen** | PLAN · PHOTO · CONTRAT · FACTURE · CORRESPONDANCE · ADDENDA · FICHE_TECHNIQUE · SOUMISSION · DIRECTIVE_CHANTIER · AUTRE (default AUTRE) |
| Note categories (`_NOTE_CATEGORIES`, `documents.py:39`) | defaut · observation · progression · decision · action · general (default general) |
| Extra statuses (`EXTRA_STATUTS`, `documents.py:112`) | PROPOSE · APPROUVE · REFUSE · FACTURE |
| Folder sharing levels (`_VALID_PERMISSION_LEVELS`, `documents.py:140`) | reader · downloader · contributor |

### 4.13 360 accounting calculation (Accounting / Summary tab)

Source: `GET /documents/{id}/360` (`documents.py:3646`, accounting block `4050-4139`).

| Element | Rule |
|---------|------|
| **Scope** | Anchored on the dossier's project or opportunity; widened to the client **only** if there is no anchor (avoids double counting across dossiers of the same client). |
| **Labour cost** | `SUM(hours × rate)` estimate **replaced** by the actual payroll cost (`payroll_entry_projets.cout_reparti`) as soon as it is positive. |
| **Revenue** | Excludes CANCELLED and DRAFT invoices, excludes supplier invoices, **subtracts credits**; margin computed on a **pre-tax** basis. |
| **Purchases** | Purchase orders **not** already represented by a counted supplier invoice (anti double-counting). |
| **Estimated margin** | `total invoiced (pre-tax) − total costs`. |

### 4.14 Limits, bounds and defenses

| Element | Value |
|---------|-------|
| Document upload | **150 MB** per file (→ 413); read in 64 KB chunks |
| Note attachments | **10 files × 15 MB** |
| AI photo analysis | **15 MB** |
| Tree depth | **5 levels** (root = level 1) |
| Extra amount | ≤ **$10,000,000** |
| Note (text) | 1 to 20,000 characters |
| Link description | ≤ 1,000 characters; URL `http(s)` ≤ 2048 |
| List pagination | `page` ≥ 1, `per_page` 1-100 (20 on screen) |
| Search | escaped LIKE (`% _`), truncated to 100 characters |
| Expiration — dossier link | **90 days** (fixed) |
| Expiration — folder link | 30 d / 90 d / **Never** (default 90 d, ≤ 3,650 d) |
| Rate — public read | 480 / 10 min per IP · 240 / 10 min per token |
| Rate — public upload | 300 / 10 min per IP · 200 / 10 min per token |
| Cumulative cap — guests (contributor) | 5,000 files / 10 GB per dossier (→ 429) |

### 4.15 File security (auth and public)

- **Real signature verified** ("magic bytes"): a disguised file (e.g. an executable renamed to `.pdf`) is served as `octet-stream`, never interpreted.
- **Inline preview restricted** to an allowlist of types (image, PDF, text), with `X-Content-Type-Options: nosniff` headers and a strict content policy.
- **Sanitized file name** for the download header (no line-break injection).
- **Forced download** (`Content-Disposition: attachment`) for downloads.
- **Tokens** generated randomly (`secrets.token_urlsafe`); regenerating **rotates** the token atomically (no ghost link).
- **Isolation**: each dossier, folder and attachment is re-checked as belonging to the right tenant (protection against access via someone else's identifier).

### 4.16 AI assistants — model, cost, rate

| Element | Value |
|---------|-------|
| Model | `claude-sonnet-4-6` |
| Cost per exchange | (input × 0.003 + output × 0.015) / 1000 × **1.30** (30% margin) |
| Debited from | `public.ai_prepaid_credits` (the tenant's prepaid credits) |
| Guards | 503 if the AI is unavailable · 403 if the AI is blocked · 402 if credits insufficient |
| Rate per IP — note AI | 15 / min |
| Rate per IP — Dossiers assistant | 20 / min |
| Rate per IP — Extras assistant | 20 / min |
| Dossiers assistant | **Read-only**, does not read file content |
| Extras assistant | Write **on confirmation**; invoicing re-checked (admin/accountant) |

### 4.17 Shortcuts and gestures

| Action | Gesture |
|--------|---------|
| Open a dossier | Click the row (list) or `/dossiers?open=<id>` |
| Rename the title | Pencil → **Enter** (save) / **Esc** (cancel) |
| Upload | "Add a document" button **or** drag and drop into the zone |
| Send a message to an AI assistant | **Enter** |
| Regenerate a share link | "Regenerate" button (the old link stops immediately) |

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

| Module | Link |
|--------|------|
| **06 — Sales (CRM)** | On converting an opportunity, a dossier can be created and linked (`opportunities.dossier_id`). The CRM's "View dossier" button opens `/dossiers?open=<id>`. On dossier deletion, the opportunity is **detached** (not deleted). |
| **08 — Quotes** | Quotes attach to the dossier (Quotes tab) and open via `/devis?open=…`. |
| **09 — Projects** | A project can be linked (`projects.dossier_id`); it feeds the dossier's time tracking and accounting. Detached on dossier deletion. |
| **12 — Work Orders** | Attached via the Work Orders tab; open via `/bons-travail?open=…`. |
| **13 — Time Tracking** | The Time Tracking tab aggregates employee hours and costs on the dossier's project. |
| **14 — Purchase Orders / Inventory** | Purchases attach (Purchases tab) and open via `/magasin?open=…`. |
| **15 — Accounting** | Invoices attach (Invoices tab); **invoicing an extra** creates a DRAFT invoice there. The Accounting tab computes revenue, costs and margin. |
| **25 — AI Assistant** | The Dossiers and Extras assistants (and the note AI tools) consume the tenant's **prepaid credits**. |
| **28 — Configuration** | The tenant's tax configuration applies to an extra's invoice. The Stripe subscription state drives **read-only mode**. |
| **Mobile app** | The extras table is **shared** with the mobile jobsite app: an extra entered in the field appears in the web Extras tab (and vice versa). |

### 5.2 FAQ

**Q1. Why does the address say "documents" while the menu says "Dossiers"?**
It is a code legacy: the original internal name was "documents". The interface was renamed "Dossiers", but the endpoints stay under `/api/erp/v1/documents`. It is cosmetic and has no effect for you.

**Q2. Who can create, edit or delete a dossier?**
Any tenant user with a valid ERP account. There is no "dossier manager" role. The **only** restricted action (to administrators and accountants) is **invoicing an extra**.

**Q3. Where are the files stored? Is there a limit?**
Documents are stored **in the database** (`BYTEA` column), not on an external cloud service. The limit is **150 MB per file**. Note attachments are limited to 10 files of 15 MB.

**Q4. How do I archive a dossier?**
Set its status to **Archived** (header dropdown). There is no separate "Archive" button, and archiving deletes nothing.

**Q5. Can I choose a category (Plan, Photo, Contract…) on upload?**
No. The categories exist server-side but are not offered on screen. Instead, organize your files into **folders**.

**Q6. What is the difference between sharing the dossier and sharing a folder?**
Sharing the **entire dossier** gives read access to all documents (90-day link). Sharing a **folder** is finer: you choose the **level** (Reader only / Read + download / Contributor) and the **expiration** (30 d, 90 d, Never), and access is limited to the shared subtree.

**Q7. Can the client drop files via the public link?**
Yes, but **only** if you give them a **folder** link at **Contributor** level. Entire-dossier links and the Reader only / Read + download levels do not allow dropping.

**Q8. How do I cut off a shared access?**
Open the relevant share and click **Revoke**. To renew a link (and immediately invalidate the old one), click **Regenerate**. A link also stops working at its expiration, or if the company's subscription is deactivated.

**Q9. Can I track who viewed or downloaded?**
Yes: the share modal shows the number of **views** and **downloads**, as well as the latest dates. There is no automatic email notification.

**Q10. What happens if I delete a dossier?**
Its attachments, notes, steps, links and extras are **deleted**; the sharing tokens are purged; linked opportunities and projects are **detached** (kept). Linked quotes, work orders, purchase orders and invoices **stay** in the database — they only lose the link. The operation is **irreversible**: prefer archiving if in doubt.

**Q11. What is an extra, and why do some people not see an "Invoice" button?**
An extra is a **change order** (additional work). Anyone can create and track it, but **only administrators and accountants can invoice it**. For others, the "Invoice" button does not appear.

**Q12. Does invoicing an extra send the invoice to the client?**
No. It creates a **DRAFT invoice** linked to the dossier. You check it and send it from the Accounting module.

**Q13. Why can't my extra be invoiced?**
Two common reasons: the dossier has **no client** attached, or the extra is **not in Approved status** (or its amount is zero). An already **Invoiced** extra is locked and can no longer be edited or deleted.

**Q14. What is the difference between the two AI assistants?**
The **Dossiers assistant** (list button) is **read-only**: it answers your questions without changing anything, and does not read file content. The **Extras assistant** (Extras tab) can **act** on extras (create, change status, invoice…), but **only after your confirmation**.

**Q15. Are the AI tools billed?**
Yes. Enhancing a note, analyzing a photo, summarizing notes, and the two assistants consume **prepaid AI credits** (the model's actual cost + 30% margin). Without sufficient credits, the action is refused (402).

**Q16. Is there a step checklist in the dossier?**
Not in the current interface: the server can manage steps, but no tab displays them. It is a dormant feature.

**Q17. Can I export the dossier list to PDF or CSV?**
No. The module offers **no export, no printing, no bulk action**. The only file outputs are downloading an attachment and public sharing by link.

**Q18. How many folder levels can I create?**
**5 levels** (the root counting as the first). At level 5, the "New subfolder" button is disabled.

**Q19. Does an extra entered on mobile appear here?**
Yes. The extras table is **shared** between the web and the mobile jobsite app: both see the same change orders.

**Q20. My account is in "Read-only mode" — what can I do?**
You can **view everything**, but no write is accepted (creation, upload, notes, extras, AI return 403). Regularize the Stripe subscription (Configuration module) to return to write access. Already-shared public links keep working as long as the company stays active.

---

## 6. Summary

- **A dossier = a 360° Record**: around a jobsite/client, it connects quotes, project, work orders, purchases, price requests, invoices, time tracking, accounting, documents, notes, links and **extras**.
- **Three screens**: list (`/dossiers`), 360° Record (`/dossier/:id`, **singular**), public page (`/dossiers/public/:token`, no account).
- **13 tabs** in the 360° Record; **no** "Steps" tab (dormant feature), share management is in the **Documents** tab.
- **Numbering**: dossier `DOS-YYYY-NNNNN`, extra `EXT-XXXX`, extra invoice `FACT-YYYY-NNNNN` — all generated with no risk of duplicates.
- **Open permissions**: any tenant account manages dossiers; **only** **invoicing an extra** is reserved for administrators/accountants. **Read-only mode** (Stripe) puts the whole module in read-only.
- **Documents**: tree up to **5 levels**, **150 MB** files in the database (`BYTEA`), move/copy even between dossiers, secure preview.
- **Sharing**: **entire-dossier** link (90 d, read) or **folder** link with **level** (Reader only / Read + download / Contributor) and **expiration** (30 d / 90 d / Never). The **Contributor** level lets a guest upload files.
- **Notes**: categories, pinning, attachments (images/audio/files), and **3 AI tools** (enrich, analyze a photo, summarize).
- **Extras (change orders)**: Proposed → Approved → Invoiced cycle; **invoicing** creates a DRAFT invoice linked to the dossier; an invoiced extra is locked; totals aware of credits and drafts.
- **Two distinct AI assistants**: Dossiers (**read-only**, does not read files) and Extras (**propose → confirm**, rights re-checked). Model `claude-sonnet-4-6`, prepaid credits + 30% margin.
- **What the module does not do**: no category on upload, no "Archive" button, no export/printing/bulk, no automatic invoice sending, no Steps tab.
- **Cascade deletion**: erases attachments/notes/steps/links/extras/link tables, purges shares, **detaches** opportunities and projects; linked items (quotes/WO/PO/invoices) are preserved.

---

*Verified sources (2026-07)*: `backend/routers/documents.py` (6138 lines, 58 routes) · `backend/routers/dossiers_ai.py` (327 lines, 1 route) · `backend/routers/dossier_ai.py` (772 lines, 2 routes) · mounting `erp_api.py:993` (documents), `1021` (dossiers_ai), `1137-1138` (dossier_ai, defensive block) · `frontend/src/pages/DossiersPage.tsx` (428 lines) · `frontend/src/pages/DossierDetailPage.tsx` (3911 lines, 13 tabs) · `frontend/src/pages/DossierPublicPage.tsx` (561 lines) · `frontend/src/components/dossiers/{DossiersAssistantTab.tsx, ExtrasAssistant.tsx}` · `frontend/src/api/{documents.ts, dossiersAi.ts}` · Sidebar `Sidebar.tsx:51` ("Management" group) · i18n `dossiers.json`, `dossiersAssistant.json`.

*Related manuals*: 06 — Sales (CRM) · 08 — Quotes · 09 — Projects · 12 — Work Orders · 13 — Time Tracking · 14 — Purchase Orders · 15 — Accounting · 25 — AI Assistant · 28 — Configuration.

*Constructo AI ERP Manual — Module 07 Dossiers (document management / 360° Record) — v3.0 verified — 2026-07*
