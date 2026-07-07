# Module 04 — Companies (clients and suppliers)

> **Version**: 3.0 (overhaul verified against the current source code, July 2026)
> **Reference code**:
> - `ERP_REACT/backend/routers/companies.py` — Companies and contacts CRUD; 11 real endpoints (`/companies*` and `/contacts*`), company soft-delete, admin guard on deletion.
> - `ERP_REACT/backend/routers/entreprises_ai.py` — Companies AI assistant; 2 endpoints (`/entreprises/ai/chat`, `/entreprises/ai/confirm-action`), propose-then-confirm pattern.
> - `ERP_REACT/frontend/src/pages/CompaniesPage.tsx` — `/entreprises` page (list, detail panel, modals, statistics).
> - `ERP_REACT/frontend/src/components/entreprises/EntreprisesAssistantTab.tsx` — AI assistant modal.
> - `ERP_REACT/frontend/src/api/companies.ts` and `api/entreprisesAi.ts` — API clients.
> - Labels: `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json`, keys `entreprises.*` (lines 1119-1251), `contacts.*` (1039) and `stages.*` (1252-1277).
> **API prefix**: `/api/erp/v1` (`erp_config.py:9`). Real example: `GET /api/erp/v1/companies`.
> **PostgreSQL tables** (per tenant schema): `companies` (main entity), `contacts` (coupled entity, also documented in manual 05); AI assistant credits on `public.ai_prepaid_credits`.
> **Scope**: this module manages the tenant's **directory of third-party entities** (clients, suppliers, subcontractors, consultants, agencies). It **does not cover**: the detailed address book of individuals (manual 05 — Contacts, a separate menu module), the sales cycle and pipeline (manual 06 — CRM and opportunities), quotes (manual 08), projects (manual 09), invoicing (manual 15).

---

## Contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step workflows](#3-step-by-step-workflows)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The **Companies** module is the central address book of the **third-party organizations** your company does business with: clients (residential, commercial, industrial, municipalities), material suppliers, specialized subcontractors, consultants and engineers, architects, land surveyors, inspection bodies, financial institutions and insurers.

Each company record acts as an **anchor point** for the rest of the ERP. Once a company is created, it becomes selectable as a client in quotes and projects, and each record displays, read-only:

- its **linked contacts** (individuals);
- its **recent quotes** (the last 5);
- its **recent projects** (the last 5).

These linked documents are attached by **foreign key** (`client_company_id`), that is, by a real and reliable link, not by a mere name match (`CompaniesPage.tsx:315-319`).

### 1.2 Access from the sidebar

- **Sidebar** → **"MANAGEMENT"** group → **"Companies"** (icon `Building2`), located between "Dashboard / Tracking" and "Contacts" (`Sidebar.tsx:48`, `nav.json:6-8`).
- **Address**: `/entreprises`.
- **Component**: `CompaniesPage` (lazy-loaded, `App.tsx:99,223`), protected by authentication (`ProtectedRoute`).
- **Page title**: "Companies" (`CompaniesPage.tsx:638`).
- **A single main view**: there are no tabs. Navigation happens through search, the type filter, and the detail panel that opens on the side.

> **"Contacts" is a separate module.** The Companies page has no Contacts tab; individuals have their own page (`/contacts`, `Sidebar.tsx:49`, manual 05). The two modules remain closely linked: the detail panel shows a company's contacts, the creation modal offers a "Primary contact" dropdown, and the AI assistant can create both. See section 5.

### 1.3 Permissions and roles

| Action | Server-side guard | Who can do it |
|---|---|---|
| View, search (read) | `get_current_user` | Any authenticated tenant user |
| Create a company or a contact | `get_current_user` | Any authenticated tenant user |
| Edit a company or a contact | `get_current_user` | Any authenticated tenant user |
| **Delete (deactivate) a company** | **`require_tenant_admin_or_role()`** | **Tenant administrator or equivalent role** |
| **Delete a contact** | **`require_tenant_admin_or_role()`** | **Tenant administrator or equivalent role** |
| AI assistant (chat + confirmation) | `get_current_user` + AI guard + credits | Any authenticated user, if the AI credit balance allows |

- **Tenant isolation**: every request applies `db.set_tenant(conn, user.schema)`. Without a tenant context, the API returns **HTTP 400 "Missing tenant context"**. A company from one tenant is never visible to another tenant.
- Deletion requires an **administrator**: the `require_tenant_admin_or_role()` guard authorizes the user whose `is_admin` flag is true (re-read server-side, hence unforgeable), or whose role is `admin`, or a platform super-administrator. An ordinary employee can create and edit a record, but **cannot deactivate it**.
- **Read-only mode (subscription)**: if the account is read-only (no Stripe or a cancelled subscription), all writes (`POST`, `PUT`, `PATCH`, `DELETE`) return **403 "Read-only mode"**; only reads (`GET`) go through. If the tenant's company is deactivated, the session is cut (**401**). This control applies to every endpoint in the module.

### 1.4 Module components

1. **Statistics banner** — 4 tenant-wide counters (Total, Clients, Suppliers, Subcontractors).
2. **Command bar** — creation, AI assistant, refresh, search, filter by type.
3. **Company list** — table on desktop, cards on mobile, with pagination (20 per page).
4. **Side detail panel** — contact details, linked contacts, recent quotes and projects.
5. **Create / Edit modals** — a single shared form for both operations.
6. **AI Assistant — Companies** — a chat that reads real data and proposes creations upon confirmation.

### 1.5 What the module does not do (to set expectations)

- **No export** (CSV or PDF), printing, or upload of files, logos or documents on a company record.
- **No permanent deletion** from the interface: "deletion" is a reversible deactivation (see section 3.5). Reactivation is likewise not exposed in the interface.
- **The AI assistant only creates** (a company or a contact). It neither edits nor deletes any record (see section 2.8).
- **No duplicate detection**: two companies with the same name can coexist; search before you create.
- **No format validation** of tax numbers, postal code or phone: these are free-text fields.
- **No parent/subsidiary hierarchy**: a company is a flat record.

---

## 2. Interface

### 2.1 General layout

The page stacks, top to bottom (`CompaniesPage.tsx`):

1. the alert banners (error or success);
2. the "Companies" title and the 4 statistics cards;
3. the command bar;
4. the list (table on large screens, stacked cards on mobile) with its pagination.

Clicking a company opens a **detail panel** on the right (it takes up about 40% of the width on desktop); on mobile, it takes the full screen with a "Back to list" button.

### 2.2 Statistics banner (4 cards)

Four counters, computed over **the entire tenant** by the `GET /companies/stats` endpoint — and not on the displayed page alone (`CompaniesPage.tsx:641-646`, `companies.py:320-356`):

| Card | Color | Server-side computation |
|---|---|---|
| **Total** | blue | `COUNT(*)` of active companies |
| **Clients** | blue | `type_company` contains "Client" (`ILIKE '%Client%'`) |
| **Suppliers** | green | `type_company` contains "Fournisseur" (`ILIKE '%Fournisseur%'`) |
| **Subcontractors** | purple | `type_company` contains "Sous-traitant" (`ILIKE '%Sous-traitant%'`) |

> **Important — the type is stored in French, even in the English interface.** The dropdown shows English labels, but the value written to the database is the French canonical string (e.g., `Fournisseur matériaux`, `Sous-traitant spécialisé`, `Client résidentiel`; `CompaniesPage.tsx:65-80`, kept in French for DB compatibility). The statistics match on that stored value, which is why the `ILIKE '%Fournisseur%'` / `'%Sous-traitant%'` / `'%Client%'` patterns work in every UI language. As long as you pick from the dropdown, the counters stay correct.

> **Good to know — the 3 sub-cards do not cover the whole Total.** Categorization is done by substring match on the type (`companies.py:342-348`). A company of type "General contractor", "Architect", "Municipality", "Insurer"… counts toward **Total** but toward **none** of the three sub-cards. It is therefore normal for `Clients + Suppliers + Subcontractors` to be **less** than the Total. This is not a display error.
>
> Deactivated companies are **excluded** from these counters by default, as they are from the list (see section 3.5).

### 2.3 Command bar

On the left (`CompaniesPage.tsx:648-676`):

- **"New Company"** (icon `Plus`, primary button) → opens the creation modal.
- **"AI Assistant"** (icon `Sparkles`) → opens the AI assistant modal (section 2.8).
- **"Refresh"** (icon `RefreshCw`, which spins while loading) → reloads the list and the statistics.

On the right:

- **Search field** ("Search by name, email, city…") with a 300 ms debounce after typing before the request fires (`CompaniesPage.tsx:228-231`). The server searches on **name**, **email** and **city** (`companies.py:264-271`); the `%` and `_` wildcards are neutralized, and the search is case-insensitive.
- **Filter by type** (dropdown): "All types" plus the 14 company types (`FILTER_TYPE_OPTIONS`, `CompaniesPage.tsx:111-114`).

### 2.4 Company list

**On desktop** — table (`CompaniesPage.tsx:687-768`). Columns are sortable (sorting is done client-side, over the 20 rows of the displayed page) and resizable:

| Column | Content | Sort key |
|---|---|---|
| **Name** | company name, with the email on a sub-line | `nom` |
| **Type** | colored badge by type (`TYPE_COLORS`, `:116-135`) | `typeCompany` |
| **Contact** | the formatted **phone** (see note) | `telephone` |
| **City** | city | `ville` |
| **Actions** | Edit (pencil) and Delete (trash) buttons | — |

> **Mind the "Contact" column label.** It actually shows the company's **phone number** (sort key `telephone`), not the name of a primary contact. The label can be misleading.

- Clicking a row opens the detail panel.
- **Pagination**: 20 companies per page (`CompaniesPage.tsx:190`). A pagination control appears beyond one page. A line shows the total count: "X company/companies".

**On mobile** — stacked cards (`:771-824`): name, type badge, Delete button, then three lines email / phone / city with their icons.

**Empty states** (`:749-763`; mobile `:810-823`):

- With an active filter or search → "No results for these filters" plus a **"Reset filters"** button.
- With no data at all → "Start by adding a company." plus a **"New Company"** button.

**Banners** (`:635-636`): an error banner (which you can close) and a success banner that auto-clears after 3 seconds (`:242-246`).

### 2.5 Detail panel

Opened by clicking a company (`renderDetailPanel`, `CompaniesPage.tsx:473-631`).

**Header**: name, type badge, business sector, plus three buttons: **Edit**, **Delete**, **Close**.

**Contact details** (shown when filled in):

- email (icon `Mail`);
- formatted phone (icon `Phone`);
- concatenated address "address, city, province, country" (icon `MapPin`);
- website (icon `Globe`);
- "Payment: {terms}";
- notes (in a box);
- "Created on {date}".

**Linked contacts** (`:554-579`): title "Contacts ({count})"; for each contact, a circle with its initials, its first and last name, a **"Primary"** badge if it is the primary contact, then a "role - email" line.

**Recent quotes** (`:582-605`, icon `FileText`): up to 5 quotes (`GET /devis?clientCompanyId={id}&perPage=5`). Each row shows the quote number, the project name and a status badge. If there are none: "No quote".

**Recent projects** (`:607-628`, icon `FolderKanban`): up to 5 projects (`GET /projects?clientCompanyId={id}&perPage=5`). Each row shows the project name and a status badge. If there are none: "No project".

> These two lists are filtered by the **foreign-key link** `clientCompanyId`: they show only the documents actually attached to this company, not a fuzzy name search.

### 2.6 Create / Edit modal

Both operations share the same form (`renderFormFields`, `CompaniesPage.tsx:395-470`). The modals are titled "New Company" or "Edit Company". At the bottom: **Cancel** and **Save** (the latter is disabled while the name is empty, and protected against double-click).

Fields, in order:

1. **Company name** * — required.
2. **Company type** * — dropdown, 14 values (section 2.7).
3. **Construction sector** — dropdown, 18 default values (section 2.7).
4. **Address** (subheading), **Address (street, number)**, **City**.
5. **Province/State**, **Postal code**.
6. **Country**.
7. **Website**.
8. **Email** (email type), **Phone**.
9. **GST number**, **QST number**.
10. **Payment terms**.
11. **Primary contact** — dropdown: "None" plus all the tenant's contacts (up to 100), in the "first last (company)" format (`CompaniesPage.tsx:219-224`).
12. **Notes on the company** — a 3-line text area.

A note reminds "* Required fields".

**Default values suggested at creation** (`:167-172,276`): type = "General contractor", province = "Québec", country = "Canada". Server-side, payment terms default to "Net 30" if you do not change them (`companies.py:160`).

Error messages appear **inside the modal** (above the buttons), so they stay visible without being hidden behind the background (`:865,878`).

> **Correction versus older versions.** Email, phone, the GST and QST numbers, and payment terms **are now in the form**: you no longer need to go through the API to enter them.

### 2.7 Company types and business sectors

**14 company types** (`TYPE_ENTREPRISE_OPTIONS`, `CompaniesPage.tsx:65-80`):

General contractor · Specialized subcontractor · Real estate developer · Materials supplier · Consultant/Engineer · Architect · Land surveyor · Inspection body · Financial institution · Insurer · Residential client · Commercial client · Industrial client · Municipality.

**18 default business sectors** (`SECTEUR_OPTIONS`, `CompaniesPage.tsx:83-103`):

Residential construction · Commercial construction · Industrial construction · Residential renovation · Commercial renovation · Excavation and earthworks · Specialized foundations · General carpentry · Roofing · Plumbing and heating · Building electrical · Insulation and waterproofing · Exterior cladding · Interior finishing · Landscaping · Demolition · Equipment rental · Construction transport.

> **Sectors are configurable per tenant.** The dropdown list comes from your configuration (`GET /config/supplier-categories`, `CompaniesPage.tsx:257-266`; set in **Configuration → Supplier categories**, manual 28). If none is configured, the 18 sectors above apply. If a record already carries a sector that is not in the current list (an inherited value), the dropdown **keeps** that value so as not to overwrite it (`:406-409,419-423`).
>
> **Technical note.** Server-side, the type and the sector are **free-text strings**, bounded only in length. There is no strict validation against the 14 types or the 18 sectors — those lists exist only in the interface, and their stored values are the French canonical strings (see section 2.2). In practice, stick to the dropdowns so the filter and statistics stay consistent.

### 2.8 AI Assistant — Companies

Opened by the "AI Assistant" button in the command bar; "AI Assistant — Companies" modal (`CompaniesPage.tsx:888`, component `EntreprisesAssistantTab.tsx`, backend `entreprises_ai.py`). It is an assistant **with human confirmation**: it **proposes**, you **confirm** (`entreprises_ai.py:13-19`).

**Two capabilities:**

1. **Read your data** (tool `recherche_bd`). The assistant can query, read-only, a restricted set of tables: `companies`, `contacts`, `projects`, `opportunities`, `devis`, `factures`. Sensitive data (payroll, HR, employees, user accounts, Stripe, emails…) is **refused** by a dedicated safeguard (`_ENT_SENSITIVE_RE`, `entreprises_ai.py:79-86`).
2. **Create upon confirmation** (tools `proposer_entreprise` and `proposer_contact`). The assistant **never writes directly**. It displays a **proposal card** (with a preview of the fields). You click **Confirm** or **Cancel**. Only confirmation (`POST /entreprises/ai/confirm-action`) executes the creation, after a fresh server-side validation.

**Chat flow** (`EntreprisesAssistantTab.tsx`): an "AI Assistant — Companies" header and a subtitle; a welcome state with 3 example questions:

- "How many suppliers do I have in Quebec City?"
- "Create the company \"Construction ABC\", client, in Montreal."
- "Add a contact Jean Tremblay, director, for this company."

You type your request (Enter to send, Shift+Enter for a new line). The reply bubbles show metadata (profile, tokens, cost, time). Under each proposal, two buttons **Confirm** and **Cancel**, with the note "Awaiting confirmation". Locks prevent double-send and double-confirmation. After a successful confirmation, the page's list and statistics refresh automatically (`:889`).

**Assistant limits:**

- **Creation only** (company and contact). **Editing** an existing record via AI **is not implemented** (`entreprises_ai.py:21-23`).
- To attach a contact to a company, the assistant must first find the company's identifier via `recherche_bd`.
- The type vocabulary the AI suggests may differ from the dropdown labels (for example "Supplier" or "Client" instead of "Materials supplier" or "Residential client"). Since the stored value is the French canonical type, check and adjust the type after creation if needed.

**A separate assistant exists for the Contacts page** (`/contacts`, backend `contacts_ai.py`); it is independent of the Companies assistant.

---

## 3. Step-by-step workflows

### 3.1 Create a company

1. Sidebar → **Companies** → **"New Company"** button.
2. Enter at least the **Name** (required). Choose the **Type** (default "General contractor") according to the nature of the relationship.
3. Complete, as needed: business sector, address, city, province ("Québec" by default), postal code, country ("Canada" by default), website, email, phone, GST and QST numbers, payment terms, primary contact, notes.
4. Click **Save** → `POST /companies` → "Company created" message → the list and statistics reload.

> No number is generated automatically (unlike quotes `DEV-` or folders `DOS-`). The company is immediately usable as a client in quotes and projects.

### 3.2 View and edit a company

1. Click the company's row in the list → the **detail panel** opens (contact details, contacts, recent quotes and projects).
2. Click **Edit** (pencil) → the modal opens, pre-filled.
3. Adjust the desired fields.
4. Click **Save** → `PUT /companies/{id}` → "Company updated" message.

> The update is partial: only the changed fields are sent. Submitting with no change returns "No field to modify". The name cannot be emptied.

### 3.3 Search, filter and sort

1. Command bar → **search field**: type a fragment of a name, an email or a city. The request fires 300 ms after the last keystroke.
2. **Filter by type**: choose one of the 14 types or "All types". Search and filter combine.
3. **Sort**: click a column header (Name, Type, Contact/phone, City). The sort applies **to the 20 rows of the displayed page**, not to the entire tenant.
4. Navigate with the pagination (20 per page).

### 3.4 Designate a primary contact

The primary contact is a link that goes **from the company to a contact**.

1. First create the contact (Contacts page, manual 05), or via the AI assistant.
2. Open the company → **Edit** → **"Primary contact"** dropdown → select the person.
3. **Save.** In the detail panel, this contact will carry the **"Primary"** badge.

> **A single primary contact per company** is enforced by the server: designating a new primary automatically removes the flag from the previous one (`companies.py:794-799, 888-895`).

### 3.5 Deactivate a company (soft delete)

1. Detail panel or table → **Delete** button (trash).
2. Confirmation dialog: "Do you really want to delete this company?". Confirm.
3. Result: `DELETE /companies/{id}` performs a **deactivation** (`active = FALSE` and `statut = 'Inactif'`, `companies.py:556-600`). The success message shown is **"Company deactivated"**.

Consequences:

- The record is **never physically erased**. All quotes, projects, invoices and opportunities already attached remain intact.
- The company **disappears from the list and the statistics** by default: these exclude deactivated companies (`COALESCE(active, TRUE) = TRUE`, `companies.py:125-138`).
- **Only an administrator** can perform this action (guard `require_tenant_admin_or_role()`). An ordinary employee will not see the operation succeed.

> **Reactivation**: it is **not exposed in the interface**. There is no reactivation button, and no "include inactive" checkbox in the list. To reactivate a deactivated record, contact your administrator (an operation to perform via the API or direct database access).

### 3.6 View linked quotes and projects

1. Click the company → the detail panel automatically loads the **last 5 quotes** and the **last 5 projects** actually attached (by foreign key).
2. For the full list, open the Quotes (manual 08) or Projects (manual 09) modules and filter by this client.

### 3.7 Query your data with the AI assistant

1. Command bar → **"AI Assistant"**.
2. Ask a natural-language question, for example "How many suppliers do I have in Quebec City?" or "List my clients in Montreal".
3. The assistant queries your data (read-only, authorized tables only) and answers. This query **consumes AI credits** (see section 4.6).

### 3.8 Create a company or a contact with the AI assistant

1. Open the AI assistant.
2. Phrase the request, for example "Create the company \"Construction ABC\", client, in Montreal." or "Add a contact Jean Tremblay, director, for Construction ABC.".
3. The assistant displays a **proposal card** with the field preview. **Nothing is saved yet.**
4. Review, then click **Confirm** (or **Cancel**).
5. Upon confirmation, the entity is created and the page refreshes.

> Confirmation does not re-invoke the AI model: it performs a pure database write. It **does not consume** any additional credit (only the chat consumes, see section 4.6).

### 3.9 Configure the sector list

The sectors offered in the form come from **Configuration → Supplier categories** (manual 28). Add your own sectors there so they appear in the company form's dropdown. Otherwise, the 18 default sectors apply.

---

## 4. Reference

### 4.1 API endpoints

All prefixed with `/api/erp/v1`. Unless noted, the guard is `get_current_user` (authenticated user, limited to their tenant).

| Method | Path | Role | Guard / notes |
|---|---|---|---|
| GET | `/companies` | Paginated list + search + type filter | `list_companies` (`companies.py:234`) |
| GET | `/companies/stats` | Total / Clients / Suppliers / Subcontractors counters | `companies_stats` (`:320`); declared before `/{id}` |
| GET | `/companies/{id}` | One company + its `contacts[]` array | `get_company` (`:372`); 404 if not found |
| POST | `/companies` | Create a company | `create_company` (`:428`) |
| PUT | `/companies/{id}` | Partial update | `update_company` (`:490`); 404 if no row |
| DELETE | `/companies/{id}` | **Deactivation** (soft delete) | `delete_company` (`:557`) — **`require_tenant_admin_or_role()`** |
| GET | `/contacts` | List of contacts (filter `company_id`, search) | `list_contacts` (`:621`) |
| GET | `/contacts/stats` | Contact counters | `contacts_stats` (`:707`) |
| POST | `/contacts` | Create a contact | `create_contact` (`:753`) |
| PUT | `/contacts/{id}` | Update a contact | `update_contact` (`:829`) |
| DELETE | `/contacts/{id}` | **Permanently delete** a contact | `delete_contact` (`:934`) — **`require_tenant_admin_or_role()`**; 409 if referenced |
| POST | `/entreprises/ai/chat` | AI assistant: reads + proposals | `entreprises_ai_chat` (`:356`); **debits credits** |
| POST | `/entreprises/ai/confirm-action` | AI assistant: executes the confirmed creation | `confirm_entreprises_action` (`:506`); **debits nothing** |

> **An asymmetry to know**: deleting a **company** is a reversible deactivation (`active = FALSE`), whereas deleting a **contact** is a **permanent deletion** (blocked with a 409 if the contact is still linked to a project, a contract or a document).

### 4.2 Company form fields and validations

| Field (form) | Column | Required | Max length | Default |
|---|---|---|---|---|
| Company name | `nom` | Yes | 200 | — |
| Company type | `type_company` | Yes (dropdown) | 100 | "General contractor" |
| Business sector | `secteur_activite` | No | 150 | — |
| Address (street, number) | `adresse` | No | 300 | — |
| City | `ville` | No | 120 | — |
| Province/State | `province` | No | 80 | "Québec" |
| Postal code | `code_postal` | No | 20 | — |
| Country | `pays` | No | 80 | "Canada" |
| Website | `site_web` | No | 300 | — |
| Email | `email` | No | 255 | — |
| Phone | `telephone` | No | 50 | — |
| GST number | `numero_tps` | No | 50 | — |
| QST number | `numero_tvq` | No | 50 | — |
| Payment terms | `payment_terms` | No | 100 | "Net 30" |
| Primary contact | `contact_principal_id` | No | — | "None" |
| Notes | `notes` | No | 10000 | — |

- The **name** is validated server-side: an empty or whitespace-only name is refused (HTTP 422).
- The **type** and the **sector** are not validated against a closed list (length-bounded free text).
- A non-existent **primary contact** is cleanly refused (HTTP 400), with no technical error.

### 4.3 The 14 company types

| # | Type | Typical use |
|---|---|---|
| 1 | General contractor | Default value; a company that orchestrates subcontractors |
| 2 | Specialized subcontractor | Roofer, plumber, electrician… |
| 3 | Real estate developer | Project developer (condos, rentals) |
| 4 | Materials supplier | Hardware, concrete, wood, steel |
| 5 | Consultant/Engineer | Civil, structural, mechanical engineering |
| 6 | Architect | Architecture firm |
| 7 | Land surveyor | Surveying, location certificate |
| 8 | Inspection body | RBQ (Régie du bâtiment du Québec — Québec building authority), CNESST (Québec workplace health, safety and labour board), quality control |
| 9 | Financial institution | Bank, credit union (caisse), mortgage broker |
| 10 | Insurer | Site insurance, civil liability |
| 11 | Residential client | Individual homeowner |
| 12 | Commercial client | Business or store |
| 13 | Industrial client | Factory, industrial park |
| 14 | Municipality | City, township, MRC (regional county municipality) — a public project owner |

> Only types containing "Client", "Fournisseur" or "Sous-traitant" (matched against the stored French value) feed the statistics sub-cards (see 2.2).

### 4.4 The 18 default business sectors

Residential construction · Commercial construction · Industrial construction · Residential renovation · Commercial renovation · Excavation and earthworks · Specialized foundations · General carpentry · Roofing · Plumbing and heating · Building electrical · Insulation and waterproofing · Exterior cladding · Interior finishing · Landscaping · Demolition · Equipment rental · Construction transport.

A list you can enrich per tenant in Configuration (section 3.9).

### 4.5 Statistics computation

| Counter | Formula (`companies.py:342-348`) |
|---|---|
| Total | `COUNT(*)` of non-deactivated companies |
| Clients | `COUNT(*) FILTER (WHERE type_company ILIKE '%Client%')` |
| Suppliers | `COUNT(*) FILTER (WHERE type_company ILIKE '%Fournisseur%')` |
| Subcontractors | `COUNT(*) FILTER (WHERE type_company ILIKE '%Sous-traitant%')` |

Consequence: `Total ≠ Clients + Suppliers + Subcontractors` whenever there are types outside these three families (General contractor, Architect, Municipality, Insurer, etc.).

### 4.6 AI assistant — bounds and billing

| Parameter | Value |
|---|---|
| Model | `claude-sonnet-4-6` |
| Max output tokens | 8000 |
| Tool iterations per turn | 5 maximum |
| History kept | 12 turns (body bounded to 40 messages) |
| Readable tables | `companies`, `contacts`, `projects`, `opportunities`, `devis`, `factures` |
| Rate limit (per IP) | Chat: 20; confirmation: 30 |
| Chat cost | `(input_tokens × 0.003 + output_tokens × 0.015) / 1000 × 1.30` (30% markup), in USD |
| Credits | Debited from `public.ai_prepaid_credits`; **402** if the balance is exhausted |
| Action confirmation | Does not re-invoke the model: **no credit debited** |

> **A point to watch (billing).** The chat debits credits **without a dedicated idempotency key** (`entreprises_ai.py:482`): an accidental resend or a double-click can debit twice. The only protection is the per-IP rate limit. Avoid resending the same question in bursts.

### 4.7 Guards and read-only mode

- **Read, create, edit**: any authenticated tenant user.
- **Deletion (company and contact)**: tenant administrator or equivalent role (`require_tenant_admin_or_role()`).
- **Missing tenant context** → HTTP 400 "Missing tenant context".
- **Read-only mode** (read-only subscription) → any write returns **403 "Read-only mode"**; deactivated tenant company → **401** (session cut).
- **No dedicated rate limit** on the `/companies` and `/contacts` CRUD endpoints (only the two AI endpoints have one).

### 4.8 Limits and what does not exist

| Topic | State |
|---|---|
| CSV / PDF export, printing | Absent from this module |
| File / logo / document upload | Absent |
| Permanent deletion of a company | Impossible from the interface (deactivation only) |
| Reactivation of a company | Not exposed in the interface |
| Global server-side sorting | No: client-side sort on the displayed page (20 rows) |
| Editing / deletion via AI | No: the assistant only creates |
| AI access to payroll / HR / employees | Refused (anti-exfiltration safeguard) |
| Duplicate detection | Absent |
| Format validation (taxes, postal code, phone) | Absent (free text) |
| Status editing outside deactivation | No dedicated control in the form |
| Parent/subsidiary hierarchy | Absent |

### 4.9 Fields present in the database but hidden from the interface

The `companies` record carries, in the database, columns deliberately **not displayed and not editable** here (allowlist `_PUBLIC_COMPANY_COLS`, `companies.py:82-92`):

- `mot_de_passe_hash` — login password for **B2B / C2B portal** clients (the `companies` table also serves as the portal identity);
- `qbo_*` — **QuickBooks** secrets and sync state;
- `credit_limit`, `ca_total` — internal commercial data;
- `tax_number_*`, `representant_code`, `source_acquisition` — internal fields.

Any new column added to this table is **excluded by default** from the interface until it is added to the allowlist (safe behavior in the face of schema evolution).

---

## 5. Integrations and FAQ

### 5.1 Links to other modules

| Linked module | Technical link | What it means |
|---|---|---|
| **Contacts** (manual 05) | `contacts.company_id` ↔ `companies.contact_principal_id` | A company has N contacts; it designates one primary |
| **CRM and opportunities** (manual 06) | `opportunities.company_id` | The sales pipeline relies on the company as the client |
| **Quotes** (manual 08) | `devis.client_company_id` | The last 5 quotes appear in the detail panel |
| **Projects** (manual 09) | `projects.client_company_id` | The last 5 projects appear in the detail panel |
| **Accounting and invoices** (manual 15) | `factures.client_company_id` | Invoicing targets the client company |
| **Purchase orders** (manual 14) | supplier = company | A supplier is a company from this directory |
| **B2B / C2B portal** | `companies.mot_de_passe_hash` | The same table serves as the portal login identity |
| **QuickBooks** (Configuration, manual 28) | `qbo_*` columns, canonical `active` column | Sync reads `active` (hence the importance of the soft delete) |

### 5.2 Frequently asked questions

**Why does my Total not match the sum of Clients + Suppliers + Subcontractors?**
That is normal. The three sub-cards cover only the types containing "Client", "Fournisseur" or "Sous-traitant". A "General contractor", an "Architect" or a "Municipality" counts toward the Total but toward none of the sub-cards (section 2.2).

**The "Contact" column shows a phone number, not a name. Is that a bug?**
No, it is the intended behavior: this column shows the company's phone. The label is simply not very explicit. The primary contact's name, for its part, appears in the detail panel.

**I "deleted" a company but it did not really disappear?**
Deletion is a reversible **deactivation**: the record moves to `active = FALSE` / `statut = 'Inactif'` and drops out of the list and the statistics, but it stays in the database and all its linked documents are preserved. The message shown is, in fact, "Company deactivated".

**How do I reactivate a deactivated company?**
The interface does not offer this action. Reach out to your administrator; reactivation is done at the API or database level.

**I am not an administrator and the Delete button does nothing?**
Deactivation requires an administrator role. An ordinary employee can create and edit a record, but not deactivate it.

**Can I export the list of companies to CSV or PDF?**
Not from this module: no export or printing is provided here.

**Can I attach a logo or documents to a record?**
No, no upload is possible on a company record.

**Where do I put the NEQ (numéro d'entreprise du Québec — Québec business number) or an RBQ licence number?**
There is no dedicated field in this module. Record them in the **Notes**. RBQ / CCQ (Commission de la construction du Québec — Québec construction commission) attestations belong to the Compliance module (manual 17).

**Can the same organization be both a client and a supplier?**
Yes. A single `companies` record can be referenced as a client (invoices, projects) and as a supplier (purchase orders). Choose the most representative type and note the other role in the Notes.

**Can the AI assistant edit or delete a record?**
No. It only **creates** (a company or a contact), and always after your confirmation. It also does not access payroll, HR or employee data.

**Does column sorting cover all my companies?**
No: it sorts only the 20 rows of the displayed page. For a global sort, first narrow down with the search and the type filter.

**Which fields does the search cover?**
Only the name, the email and the city. It does not search the phone, the postal code, the notes or the tax numbers.

**Can I create two companies with the same name?**
Yes, there is no uniqueness constraint and no duplicate detection. Search before creating to avoid duplicates.

**Does renaming a company update quotes and invoices already issued?**
Documents already issued may keep the name as it was at their creation time. The attachment link (foreign key) stays correct, but re-issue the document if its display must reflect the new name.

---

## 6. Summary

- **Mission**: central directory of the tenant's third-party companies (clients, suppliers, subcontractors, consultants, agencies).
- **Access**: sidebar, MANAGEMENT group → **Companies** (icon `Building2`), address `/entreprises`.
- **Statistics**: 4 **tenant-wide** cards (Total, Clients, Suppliers, Subcontractors) via `GET /companies/stats`; the 3 sub-cards do not cover the whole Total (categorization by substring of the type).
- **List**: sortable table (current page), pagination 20 per page; the "Contact" column shows the **phone**.
- **Detail panel**: contact details, linked contacts, the last 5 quotes and the last 5 projects **attached by foreign key** (reliable).
- **Form**: name required; email, phone, GST, QST and payment terms **are in the form**; defaults "General contractor" / "Québec" / "Canada" / "Net 30".
- **14 types** of company and **18 default sectors** (sectors **configurable per tenant**).
- **Deletion = deactivation** (`active = FALSE`, message "Company deactivated"), **reserved for administrators**; deactivated records are **excluded** from the list and the statistics; no permanent deletion and no reactivation in the interface.
- **AI assistant**: reads your data (restricted tables) and **creates** companies and contacts **upon confirmation**; it neither edits nor deletes anything. The chat consumes AI credits; the confirmation does not.
- **Contacts** is a **separate module** (manual 05), served by the same backend and tightly linked to Companies.
- **Security**: strict tenant isolation; read-only mode on an inactive subscription; sensitive data (B2B password, QuickBooks secrets, internal commercial data) hidden from the interface.
- **Absences to know**: no export / printing, no upload, no duplicate detection, no format validation, no hierarchy, no global server-side sorting.

---

**Verified source files**:
- `ERP_REACT/backend/routers/companies.py` (companies and contacts CRUD, soft delete, statistics, guards)
- `ERP_REACT/backend/routers/entreprises_ai.py` (AI assistant, propose / confirm pattern, credit billing)
- `ERP_REACT/frontend/src/pages/CompaniesPage.tsx` (`/entreprises` page)
- `ERP_REACT/frontend/src/components/entreprises/EntreprisesAssistantTab.tsx` (assistant modal)
- `ERP_REACT/frontend/src/api/companies.ts` and `ERP_REACT/frontend/src/api/entreprisesAi.ts` (API clients)
- `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json` (keys `entreprises.*`, `contacts.*`, `stages.*`)

**Related manuals**:
- Manual 05 — Contacts (individuals): `05-gestion-contacts.md`
- Manual 06 — CRM and opportunities: `06-gestion-crm-opportunites.md`
- Manual 08 — Quotes: `08-ventes-soumissions.md`
- Manual 09 — Projects: `09-ventes-projets.md`
- Manual 14 — Purchase orders (suppliers): `14-operations-bons-de-commande.md`
- Manual 15 — Accounting and invoices: `15-operations-comptabilite.md`
- Manual 17 — Compliance (RBQ / CCQ / CNESST): `17-terrain-conformite.md`
- Manual 25 — AI Assistant (overview): `25-communication-assistant-ia.md`
- Manual 28 — Configuration (supplier categories, QuickBooks): `28-configuration.md`
