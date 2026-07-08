# Module 04 — Contacts (individuals)

> **Version**: 3.0 (overhaul verified against the current source code, July 2026)
> **Reference code**:
> - `ERP_REACT/backend/routers/companies.py` — Contacts CRUD, served by the **same** router as Companies; 5 real endpoints under `/contacts`: list (`:621`), statistics (`:707`), creation (`:753`), update (`:829`), deletion (`:934`). There is **no** `routers/contacts.py` file.
> - `ERP_REACT/backend/routers/contacts_ai.py` — Contacts-specific AI assistant; 2 endpoints (`/contacts/ai/chat` `:284`, `/contacts/ai/confirm-action` `:423`); propose-then-confirm pattern.
> - `ERP_REACT/frontend/src/pages/ContactsPage.tsx` (523 lines) — `/contacts` page: list, statistics, search, creation and edit modals.
> - `ERP_REACT/frontend/src/components/contacts/ContactsAssistantTab.tsx` — AI assistant modal (chat + proposal cards).
> - `ERP_REACT/frontend/src/api/companies.ts` — contacts API client (`listContacts` `:128`, `getContactStats` `:143`, `createContact` `:148`, `updateContact` `:153`, `deleteContact` `:157`); `ERP_REACT/frontend/src/api/contactsAi.ts` — AI assistant client. There is **no** `api/contacts.ts` file.
> - Labels: `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json`, keys `contacts.*` (lines 1039-1118).
> **API prefix**: `/api/erp/v1` (`erp_config.py:9`). Real example: `GET /api/erp/v1/contacts`.
> **PostgreSQL table** (per tenant schema): `contacts` (main entity); the address columns and a few others are added on the fly by `_ensure_contact_address_cols` on first access; the AI assistant credits live on `public.ai_prepaid_credits`.
> **Scope**: this module manages the **address book of individuals** (natural persons), whether or not they are linked to a company. It is **distinct** from the Companies module (legal entities, manual 04) and from the CRM and opportunities module (the sales pipeline, manual 06). A contact is **not** an ERP user account.

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

The **Contacts** module is your **people address book**. It centralizes every individual your company does business with: the project manager at a client, a supplier's representative, a subcontractor's estimator, the architect, the engineer, the land surveyor, the bank advisor, the insurance broker, or any other person to reach.

A contact is **always a person** (a first name and a last name). It can be **linked to a company** — for example "Marie Gagnon, engineer at Construction ABC" — or **exist on its own**, without an employer, when you only know the person. Each record carries the contact details (email, landline, mobile), the role or position, the function, the department, a mailing address and notes.

Contacts serve as an **anchor point** across the rest of the ERP: they are selectable as points of contact in CRM opportunities, quotes, projects, contracts and emails. They also feed a company's **"primary contact"** (see manual 04).

> **Contacts and Companies are two twin modules.** They share the same server-side code (`companies.py`) and are tightly linked: a company's record shows its contacts, and a contact points to its company. But they are **two separate pages** in the menu (`/contacts` and `/entreprises`). The Contacts module manages **people**; the Companies module manages **organizations**.

### 1.2 Access from the sidebar

- **Sidebar** → **"MANAGEMENT"** group → **"Contacts"** (icon `Users`), located between "Companies" and "Sales" (`Sidebar.tsx:49`, key `nav.contacts` in `nav.json:8`).
- **Address**: `/contacts`.
- **Component**: `ContactsPage` (lazy-loaded, `App.tsx:100` and `App.tsx:224`), protected by authentication (`ProtectedRoute`).
- **Page title**: "Contacts" (`ContactsPage.tsx:216`).
- **A single view**: there are **no tabs** or sub-pages. All navigation happens through search, column sorting, pagination and the two modals (create, edit). There is **no** dedicated detail page of the `/contacts/{id}` type.

### 1.3 Permissions and roles

| Action | Server-side guard | Who can do it |
|---|---|---|
| View, search (read) | `get_current_user` | Any authenticated tenant user |
| Create a contact | `get_current_user` | Any authenticated tenant user |
| Edit a contact | `get_current_user` | Any authenticated tenant user |
| **Delete a contact** | **`require_tenant_admin_or_role()`** (`companies.py:935`) | **Tenant administrator** (or super-administrator) |
| AI assistant (chat + confirmation) | `get_current_user` + AI guard + credits | Any authenticated user, if the AI credit balance allows |

- **Deletion is reserved for administrators.** The `require_tenant_admin_or_role()` guard (`erp_auth.py:720-747`) authorizes the user whose `is_admin` flag is true — re-read server-side, therefore tamper-proof — or whose role is `admin`, or a platform super-administrator. An ordinary employee can **create and edit** a record, but **not delete** it.
- **Important detail about the Delete button**: the trash icon is **displayed to everyone** in the table and the cards (`ContactsPage.tsx:313-319` and `:372-378`), without role-based hiding. A non-administrator user can therefore click "Delete", but the ERP will then respond **403 (access denied)** and the contact will not be deleted. This is not a bug: security is enforced server-side, not by hiding the button.
- **Tenant isolation**: each request applies `db.set_tenant(conn, user.schema)`. Without a tenant context, the API returns **HTTP 400 "Missing tenant context"**. A contact from one tenant is **never** visible to another tenant; no tenant identifier is accepted from the browser (protection against indirect access to others' data).
- **Read-only mode (subscription)**: if the account is read-only — without an active Stripe subscription or after cancellation — all writes (`POST`, `PUT`, `DELETE`) return **403 "Read-only mode"**; only reads (`GET`) go through (`erp_auth.py:526-530`). If the tenant's company is outright deactivated, the session is cut off (**401**). This control applies to all the module's endpoints.

### 1.4 Module components

1. **Statistics banner** — 4 **tenant-wide** counters (Contacts, Companies, With email, With phone).
2. **Command bar** — creation, AI assistant, search with quick clear.
3. **Contacts list** — sortable and resizable table on desktop, stacked cards on mobile, with pagination (20 per page).
4. **"New Contact" modal** — full creation form.
5. **"Edit contact" modal** — same form, pre-filled, with protection of the already-linked company.
6. **AI Assistant — Contacts** — a chat that reads your real data and **proposes** creating a contact, which you confirm before execution.

### 1.5 What the module does not do (to set expectations)

- **No notion of "contact type" or "interaction".** A contact's record has **no** category field, no type field, and no log of calls / emails / meetings. Roles and positions are **free text** (`role_poste`, `fonction`, `departement`). Business activities and interactions (calls, emails, meetings with time tracking) belong to the **CRM and opportunities** module (manual 06), not here.
- **No export** (CSV or PDF), **no printing**, **no upload** of a file, photo or business card.
- **No bulk action**: no multi-selection, no group deletion or editing.
- **No dedicated detail page**: only the list plus the edit modal.
- **No duplicate detection**: two contacts with the same email or the same name can coexist; use search before creating.
- **No format validation**: the email, phone and postal code are **free-text fields**; the server checks only their **maximum length**, not their form (the email does not even require an "@").
- **Deletion is permanent** (see section 2.10): unlike companies, which are simply deactivated, a deleted contact is **physically erased** and cannot be recovered from the interface.
- **The AI assistant only creates** a contact (in version 1). It neither edits nor deletes any record.

---

## 2. Interface

### 2.1 General layout

The page stacks, from top to bottom (`ContactsPage.tsx`):

1. the alert banners — red on error, green on success (the success message disappears on its own after 3 seconds);
2. the "Contacts" title and the 4 statistics cards;
3. the command bar (creation, AI assistant, search);
4. the list — a **table** on large screens, **stacked cards** on mobile — with its pagination.

The modals (create, edit, AI assistant) open on top of the page.

### 2.2 Statistics banner (4 cards)

Four counters, computed over the **entire tenant** by the `GET /contacts/stats` endpoint — and **not** on the displayed page alone (`companies.py:707-737`):

| Card | Icon / color | Server-side computation |
|---|---|---|
| **Contacts** | `Users` / blue | `COUNT(*)` — total number of contacts |
| **Companies** | `Building2` / purple | `COUNT(DISTINCT company_id)` among contacts linked to a company |
| **With email** | `Mail` / green | Number of contacts whose email is filled in (non-empty) |
| **With phone** | `Phone` / yellow | Number of contacts whose phone is filled in (non-empty) |

> **Good to know.** These four figures are **global**: they do **not** change when you flip through pages or run a search. The "Companies" card counts the number of **distinct companies** represented by your contacts (a contact without an employer is not counted). The "With email" and "With phone" cards help you spot incomplete records.

### 2.3 Command bar

On the left (`ContactsPage.tsx:227-255`):

- **"New Contact"** (icon `Plus`, primary button) → opens the creation modal (section 2.8).
- **"AI Assistant"** (icon `Sparkles`) → opens the AI assistant modal (section 2.11).

On the right:

- A **search** field with the placeholder "**Search by name, email, company, role…**". The search is **debounced** by 300 milliseconds after your last keystroke (so as not to query the server on every letter) and **always returns to page 1**.
- A **"Clear"** button (icon `X`) appears as soon as the field contains text; it clears the search and returns to the first page.

> **The search covers four fields at once** (`companies.py:647-660`), exactly as the placeholder promises: the **full name** (first name + last name), the **email**, the linked **company's name**, and the **role or position**. It is case-insensitive and looks for the term anywhere in these fields. It is also **safe**: the special characters you type are treated as ordinary text, and the term is capped at 100 characters.
>
> The search runs **server-side** over the entire tenant, then returns a page of results. So it is not a mere filter of the current page: you find a contact even if it is on page 12.

### 2.4 Table (desktop, width `md` and up)

The table (`ContactsPage.tsx:263-343`) has **six columns**. Five of them are **sortable** (click the header to sort, click again to reverse) and **resizable** (drag the handle between two headers). The default widths are: Name 200 px, Company 180 px, Role / Position 160 px, Email 220 px, Phone 140 px (`ContactsPage.tsx:207`).

| Column | Sort key | Displayed content |
|---|---|---|
| **Name** | `prenom` | **Initials** badge (first letter of the first name + first letter of the last name) + "First Last" + blue **"Primary"** badge if the contact is marked as primary contact |
| **Company** | `companyNom` | The linked company's name, or `--` if the contact is standalone |
| **Role / Position** | `rolePoste` | The role or position, or `--` |
| **Email** | `email` | `Mail` icon + email address, or `--` |
| **Phone** | `telephone` | Formatted number (for example `(514) 555-1234`), or `--` |
| **Actions** | — | **Edit** button (pencil) + **Delete** button (trash) |

> **Sorting only reorders the displayed page.** The list is first sorted by the server on the **last name** (`companies.py:681`), then split into pages of 20. When you click a header, you reorder the **20 visible rows** of the current page. To browse the whole address book in a specific order, combine sorting and pagination, or use the search.

### 2.5 Mobile cards (width below `md`)

On phone and small screens (`ContactsPage.tsx:346-408`), the table gives way to **stacked cards**. Each card repeats the same content, condensed: initials badge, full name, "Primary" badge where applicable, role as a subtitle, then a secondary line with the company (icon `Building2`), the email (icon `Mail`) and the phone (icon `Phone`). The **Edit** and **Delete** buttons are present on each card.

### 2.6 Empty states

- **Search with no result**: the message "**No results for this search**" is displayed, together with a **"Clear"** button to go back to the full list.
- **Empty address book** (no contact): a `Users` icon and the message "**Start by adding a contact.**", with a **"New Contact"** button.

### 2.7 Pagination

Pagination appears only if there is **more than one page** of results. Each page contains **20 contacts** (`ContactsPage.tsx:61`). A safeguard prevents getting stuck on a page that has become empty: if the total drops below the current page (after deletions, for example), the display automatically returns to the last page that contains data (`ContactsPage.tsx:118-121`).

### 2.8 "New Contact" modal

Opened by the **"New Contact"** button (`ContactsPage.tsx:417-465`). The fields, in order:

| Field | Required | Length constraint |
|---|---|---|
| **First name** | **Yes** | 100 characters |
| **Last name** | **Yes** | 100 characters |
| **Email** | No | 255 characters |
| **Phone** | No | 50 characters |
| **Company** (dropdown) | No | empty "-- Select --" option by default |
| **Role / Position** | No | 150 characters |
| **Function** | No | 150 characters |
| **Department** | No | 150 characters |
| **Mobile** | No | 50 characters |
| **Address** (placeholder "123 Example St") | No | 300 characters |
| **City** | No | 120 characters |
| **Province** (placeholder "QC") | No | 80 characters |
| **Postal code** (placeholder "H0H 0H0") | No | 20 characters |
| **Notes** (text area, 3 lines) | No | 10,000 characters |
| **Primary contact** (checkbox) | No | boolean |

- A note at the bottom of the modal reminds you: "**\* Required fields**".
- The buttons are **"Cancel"** and **"Save"**. The Save button stays **disabled** as long as the first name **or** the last name is empty (`ContactsPage.tsx:460`).
- **The "Company" dropdown loads only the first 100 companies** (`listCompanies({ perPage: 100 })`, `ContactsPage.tsx:90`), sorted by name. If your target company is not in the list (beyond the first 100), first create the contact without a company, then link it from the edit modal (which does know how to preserve an existing link; see section 2.9).
- On success, a green banner shows "**Contact saved**" and the list as well as the statistics refresh.

> **Useful reminder.** Checking **"Primary contact"** at creation only has an effect if the contact is also **linked to a company**. In that case, the server automatically removes the "primary" mark from the other contacts of the same company, so that there is **only one primary contact per company** (`companies.py:795-799`).

### 2.9 "Edit contact" modal

Opened by a row's pencil button (`ContactsPage.tsx:468-514`). It repeats **all** the creation fields, in a slightly different layout (first name and last name on the same line, the last-name label simply becomes "Name"), and pre-fills each value from the existing record, including the notes, the address and the "Primary contact" state.

- The buttons are **"Cancel"** and **"Save"**; Save is disabled if the first name or the last name is empty.
- On success, a green banner shows "**Contact updated**".

> **Protection of the already-linked company (important).** Since the dropdown loads only 100 companies, it could happen that the company currently linked to the contact **is not among them** (for example the 150th company, or a deactivated company). To prevent saving from silently **reassigning** the contact to the first company in the list, the modal **injects** an option that preserves the existing link (`ContactsPage.tsx:480-482`). So you keep the right employer even if it does not appear among the 100 loaded. This is a behavioral difference from the creation modal, which does not yet have a link to preserve.

### 2.10 Deleting a contact

The **trash** button (in the table or a card) triggers a **browser confirmation**: "**Delete this contact?**" (`ContactsPage.tsx:151-162`). If you confirm, the ERP calls `DELETE /contacts/{id}`.

- **Deletion is permanent.** The server performs a physical erasure of the row (`DELETE FROM contacts`, `companies.py:945`). There is **no** trash bin and no reversible deactivation, unlike companies (which become "Inactive"). **You cannot undo** a deletion from the interface.
- **Contact linked elsewhere → clear refusal (409).** If the contact is referenced by a project, a contract, a purchase order or another document, the server **refuses** the deletion and returns the message: "**This contact is linked to a project, contract or document. Unlink it before deleting.**" (`companies.py:953-959`). In that case, first remove the contact from the documents concerned, then try again.
- **Contact not found → 404** ("Contact not found").
- **Non-administrator → 403**: the click ends in a refusal (see section 1.3).
- On success, the list and statistics refresh.

> **Best practice.** Since deletion is irreversible and blocked as soon as a link exists, often prefer **editing**: clear or correct the contact details rather than deleting, in order to preserve the history of documents that point to this contact.

### 2.11 AI Assistant — Contacts

Opened by the **"AI Assistant"** button, in a large modal titled "**AI Assistant — Contacts**", subtitled "Query your contacts and create new ones upon confirmation." (`ContactsPage.tsx:516-519`, `ContactsAssistantTab.tsx`).

**How it works (chat):**

- A message area displays the exchange as bubbles.
- An input area at the bottom: **Enter** sends the message, **Shift+Enter** inserts a line break. The **"Send"** button does the same.
- On opening, an empty state offers three examples to get you started:
  - "Who are my contacts at "Construction ABC"?"
  - "Create the contact Marie Gagnon, engineer, email marie@abc.com."
  - "List the contacts without an email address."
- During processing, the "**Analyzing…**" indicator is shown.

**What the assistant can do:**

1. **Read and answer**: it queries your real data (the contacts and the companies) to answer your questions.
2. **Propose creating a contact**: when it understands that you want to add someone, it **writes nothing right away**. It displays a **proposal card** with a preview of the fields (name, email, role…), marked "**Awaiting confirmation**", and two buttons: **"Confirm"** and **"Cancel"**. Only upon confirmation is the contact actually created, and the assistant then replies "**Done. Contact created.**" The list and statistics refresh immediately.

> **Safeguards.** The assistant **creates a contact only upon your explicit confirmation**; it neither edits nor deletes any record (version 1 is limited to **creation**). For reading, it is restricted to your contacts and companies tables alone and cannot reach sensitive data (payroll, employees, users, secrets, other tenants). See sections 4.8 and 5.5 for the settings, costs and security.

---

## 3. Step-by-step workflows

### 3.1 View and search for a contact

1. Open the sidebar → **MANAGEMENT** → **Contacts**.
2. The list is displayed (20 per page), sorted by last name. The 4 cards at the top give you the tenant totals.
3. To find someone, type in the search field: a **name**, an **email**, a **company name** or a **role**. The results refresh about 300 milliseconds after your last keystroke.
4. Click a column header to **sort** the displayed page; drag the handles to **widen** a column.
5. Use the pagination at the bottom to browse the pages. Click **"Clear"** to return to the full list.

### 3.2 Create a contact linked to a company

1. Click **"New Contact"**.
2. Enter the **First name** and the **Last name** (required).
3. In the **"Company"** dropdown, choose the employer (the first 100 companies, sorted by name).
4. As needed, fill in the email, phone, mobile, role / position, function, department and address.
5. Optionally check **"Primary contact"** to make this person the company's main point of contact.
6. Click **"Save"**. "Contact saved" banner; the list updates.

> If the company does not appear in the menu (beyond the first 100), see section 3.4.

### 3.3 Create a standalone contact (without a company)

1. Click **"New Contact"**.
2. Enter at least the **First name** and the **Last name**.
3. Leave the **"Company"** menu on "-- Select --".
4. Fill in the useful contact details, then **"Save"**.

The contact appears in the list with `--` in the Company column. You can link it later.

### 3.4 Link a contact to a company (or change its employer)

1. On the contact's row, click the **pencil** button (Edit).
2. In **"Company"**, choose the new company. If the contact was already linked to a company missing from the menu, that company remains **preserved** (it is re-injected into the list): you do not lose the existing link by accident.
3. Click **"Save"**.

**Server-side effects, automatic and within a single transaction** (`companies.py:860-905`):
- If the contact was the **primary contact** of its former company, the "primary contact" pointer of that **former** company is reset (so as not to leave an orphan pointer).
- If it remains marked "primary" **and** linked to its new company, the other contacts of that new company lose the "primary" mark, in order to keep **one primary per company**.

### 3.5 Designate a company's primary contact

There are **two angles** on the same idea of a "main point of contact":

- **From the Contacts module** — check **"Primary contact"** in the creation or edit modal. This activates the blue **"Primary"** badge in the list and guarantees uniqueness on the contacts side (the others are unmarked).
- **From the Companies module** (manual 04) — a company's record offers a **"Primary contact"** menu that stores a pointer on the company side (`companies.contact_principal_id`).

These two settings aim at the same goal but are **stored separately**. The "Primary" badge on the Contacts page reflects the checkbox ticked on the contact. For perfect consistency, designate the person on both sides. The server already takes care not to leave an orphan pointer when a contact changes company (see section 3.4).

### 3.6 Edit a contact's details

1. Click the **pencil** on the contact's row.
2. Edit the email, phone, mobile, address, notes, etc.
3. Click **"Save"**. "Contact updated" banner.

Only the allowed fields are taken into account; the server also records the last modification date (`updated_at`).

### 3.7 Delete a contact

1. Click the **trash** on the row (or the card).
2. Confirm "Delete this contact?".
3. Possible outcomes:
   - **Success**: the contact disappears from the list (**permanent** deletion).
   - **"This contact is linked to a project, contract or document…"**: first unlink it from the documents concerned, then try again.
   - **Refusal (403)**: you are not an administrator; ask a tenant administrator.

### 3.8 Create a contact with the AI assistant

1. Click **"AI Assistant"**.
2. Describe the person in natural language, for example: "Create the contact Marie Gagnon, engineer at Construction ABC, email marie@abc.com, mobile 514-555-1234."
3. The assistant displays a **proposal card** ("Awaiting confirmation") with a preview of the fields.
4. Check the values, then click **"Confirm"** (or **"Cancel"** to correct your request).
5. On confirmation, the contact is created ("Done. Contact created.") and the list refreshes.

> You can also **query** your contacts without creating anything: "Who are my contacts at Construction ABC?", "List the contacts without an email."

---

## 4. Reference

### 4.1 Data model — `contacts` table (per tenant)

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | integer (primary key) | — | Auto-incremented |
| `company_id` | integer, **nullable** | No | Foreign key to `companies.id`; `NULL` = standalone contact; a value ≤ 0 is normalized to `NULL` |
| `prenom` | text (≤ 100) | **Yes** | Rejected if empty or made up of spaces only |
| `nom_famille` | text (≤ 100) | **Yes** | Same |
| `email` | text (≤ 255) | No | **No** format validation |
| `telephone` | text (≤ 50) | No | Free text; formatted on display |
| `mobile` | text (≤ 50) | No | Distinct from the landline |
| `role_poste` | text (≤ 150) | No | E.g. "Purchasing Director" |
| `fonction` | text (≤ 150) | No | Free complement to the role |
| `departement` | text (≤ 150) | No | E.g. "Accounting" |
| `adresse` | text (≤ 300) | No | Added on the fly on older schemas |
| `ville` | text (≤ 120) | No | Same |
| `province` | text (≤ 80) | No | Same |
| `code_postal` | text (≤ 20) | No | Same |
| `est_principal` | boolean | No | Default `false`; unique per company (reconciled server-side) |
| `notes` | text (≤ 10,000) | No | Free text |
| `created_at` | timestamp | — | Set at creation |
| `updated_at` | timestamp | — | Updated on each modification |

> **Automatic migration.** On older tenants, some columns (`adresse`, `ville`, `province`, `code_postal`, `mobile`, `fonction`, `departement`, `est_principal`) may be missing. The `_ensure_contact_address_cols` function (`companies.py:36-71`) adds them **automatically** on first access, once per schema. You have nothing to do.

### 4.2 Name mapping (camelCase / snake_case)

The client automatically transforms names between the interface (camelCase) and the server (snake_case):

`companyId` ↔ `company_id`; `nomFamille` ↔ `nom_famille`; `rolePoste` ↔ `role_poste`; `codePostal` ↔ `code_postal`; `estPrincipal` ↔ `est_principal`; `companyNom` ↔ `company_nom` (the company name, computed by join); `createdAt` ↔ `created_at`. The other fields keep the same name (`prenom`, `email`, `telephone`, `mobile`, `fonction`, `departement`, `adresse`, `ville`, `province`, `notes`).

### 4.3 API endpoints

All are served under the `/api/erp/v1` prefix and require authentication plus a valid tenant context.

| Method and path | Function (file:line) | Guard | Role |
|---|---|---|---|
| `GET /contacts` | `list_contacts` (`companies.py:621`) | `get_current_user` | List and search (paginated) |
| `GET /contacts/stats` | `contacts_stats` (`companies.py:707`) | `get_current_user` | 4 global counters |
| `POST /contacts` | `create_contact` (`companies.py:753`) | `get_current_user` | Create a contact |
| `PUT /contacts/{id}` | `update_contact` (`companies.py:829`) | `get_current_user` | Edit a contact |
| `DELETE /contacts/{id}` | `delete_contact` (`companies.py:934`) | **`require_tenant_admin_or_role()`** | Delete (admin) |
| `POST /contacts/ai/chat` | `contacts_ai_chat` (`contacts_ai.py:284`) | `get_current_user` + AI guard + credits | Assistant chat |
| `POST /contacts/ai/confirm-action` | `confirm_contacts_action` (`contacts_ai.py:423`) | `get_current_user` + AI guard + credits | Confirm the proposed creation |

**`GET /contacts` parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `page` | integer ≥ 1 | 1 | Page number |
| `per_page` | integer 1-100 | 20 | Page size (the interface uses 20) |
| `search` | text | — | Search on name, email, company, role |
| `company_id` | integer | — | Filter on a specific company |

Response: `{ items, total, page, per_page }`, sorted by last name then identifier. Each item includes `company_nom` (the company name, via join). **Note**: there is **no** `limit` alias on this endpoint (unlike the companies list); use `per_page`.

**Notable response codes:**

| Code | Meaning |
|---|---|
| **200** | Success (with message: "Contact created", "Contact updated", "Contact deleted") |
| **400** | "Missing tenant context", "Company not found" (linked company nonexistent in the tenant), or "No field to update" (empty update) |
| **403** | Write in read-only mode, or deletion attempted by a non-administrator |
| **404** | "Contact not found" |
| **409** | Deletion refused: the contact is linked to a project, a contract or a document |
| **402** | AI credits exhausted (assistant endpoints) |
| **429** | Too many requests (rate limit exceeded) |
| **503** | AI service unavailable (Anthropic client not configured) |

### 4.4 Validations and bounds

| Level | Rule | Effect |
|---|---|---|
| Interface | "Save" button disabled if first name or last name empty | Prevents submission |
| Server (Pydantic) | `prenom` and `nom_famille` required, non-empty after trimming spaces | 422 otherwise |
| Server | Maximum lengths (see 4.1) | 422 if exceeded |
| Server | `company_id` ≤ 0 → `NULL` | Silent normalization (detached contact) |
| Server | Linked company must exist in the tenant | 400 "Company not found" |
| Server | Uniqueness of the primary contact per company | The others are unmarked, within a transaction |
| Server | `updated_at` set on each modification | Automatic timestamp |

> **The email is not validated**: the field accepts any text of 255 characters or fewer (no requirement for an "@" or a domain). Take care to enter it correctly, as it is used for automatic email matching (see section 5.4).

### 4.5 Statistics — calculation method

`GET /contacts/stats` aggregates the **entire tenant** in a single query (`companies.py:724-737`):
- **total** = `COUNT(*)`;
- **companies** = `COUNT(DISTINCT company_id)`, ignoring standalone contacts;
- **withEmail** = number of contacts with a non-empty email;
- **withPhone** = number of contacts with a non-empty phone.

These figures are **independent** of the search and pagination.

### 4.6 Search — exact scope

The search (`companies.py:647-660`) applies a case-insensitive "contains" on four fields, combined with "OR":
1. the full name (`prenom` + `nom_famille`);
2. the email (`email`);
3. the linked company's name (`companies.nom`);
4. the role or position (`role_poste`).

Empty fields are handled without error, special characters are neutralized, and the term is truncated to 100 characters. The total count takes the same join into account, so pagination stays exact.

### 4.7 Rate limits (per IP address, 60-second window)

| Endpoint | Limit |
|---|---|
| `POST /contacts/ai/chat` | 20 per minute |
| `POST /contacts/ai/confirm-action` | 30 per minute |
| CRUD `/contacts` (non-AI) | general bracket of 1500 per minute |

An excess returns **429** with a `Retry-After: 60` header.

### 4.8 AI Assistant — settings, security and costs

- **Model**: `claude-sonnet-4-6`, up to 8000 tokens per response, tool loop capped at 5 iterations (`contacts_ai.py`).
- **Two tools**:
  - `recherche_bd` — read-only, **at most 50 rows**, restricted to the `{contacts, companies}` whitelist. Sensitive tables (employees, payroll, users, emails, AI credits, secrets, tokens, other tenants…) are **blocked**, as are references to another schema. The underlying SQL engine allows only read-only `SELECT`s, with a maximum timeout.
  - `proposer_contact` — **writes nothing**; it validates the fields (same rules as `ContactCreate`) and returns a **proposal** to confirm.
- **Write only upon confirmation**: `POST /contacts/ai/confirm-action` fully **re-validates** the fields, then delegates to the usual `create_contact` function (same tenant and company guards). No tenant identifier is accepted from the browser (protection against indirect access to others' data).
- **Version 1 scope**: contact **creation** only. Editing by the AI is planned for later.
- **Costs (AI credits)**: each **chat turn** is billed to the tenant at the model's **real cost marked up by 30%** (`contacts_ai.py:399`), debited from the prepaid credits. The **creation confirmation**, however, is **not billed**: creating a contact via the assistant costs only the chat turn(s) that led to the proposal. If the credits are exhausted, the chat returns **402**; an automatic top-up may trigger depending on the account's configuration. The service returns **503** if the AI is not configured.

### 4.9 Interface components and shortcuts

- **Components**: `CommandBar`, `StatCard`, `Card`, `Modal` (size `lg` for records, `xl` for the assistant), `Input`, `Select`, `Textarea`, `Button`, `Badge`, `Pagination`, `SortableHeader`, `MessageBubble` (assistant).
- **Behaviors**: search debounced by 300 ms; client-side sorting by column (`useSortable`); column resizing (`useColumnResize`); race protection that ignores responses from stale requests; phone formatting (`formatPhone`).
- **Assistant**: **Enter** to send, **Shift+Enter** for a line break.

---

## 5. Integrations and FAQ

### 5.1 With the Companies module (manual 04)

- A contact points to a company via `company_id` (optional). A company's record shows **its** contacts, sorted with the primary contact first.
- A company can designate a **primary contact** (pointer `contact_principal_id`). This pointer and the contact's "Primary contact" checkbox are **two distinct** but complementary settings (see section 3.5). The server clears the former company's pointer when a contact moves.
- **Deleting a company** deactivates it (it becomes "Inactive"); its contacts **remain** in the address book. Their "Company" column may show the deactivated company.

### 5.2 With the CRM and opportunities (manual 06)

Contacts are selectable as points of contact for an opportunity. It is **there**, not in the Contacts module, that **interactions** and **activities** (calls, emails, meetings) and sales qualification live. The Contacts module keeps no interaction log.

### 5.3 With Quotes, Projects and Contracts (manuals 08, 09)

A contact can be the "client contact" of a quote, a project or a contract. These links explain the **409** refusal on deletion: as long as a document points to the contact, you must first unlink it.

### 5.4 With Emails (manual 23)

A contact's name and its **email** are used for **automatic matching** of incoming emails with the right person. Hence the importance of an accurate and correctly spelled email, since the module does not validate its format.

### 5.5 With the AI assistant (credits)

The Contacts module assistant shares the same **AI credit wallet** as the other ERP assistants (`public.ai_prepaid_credits`). The chat consumes credits; confirming a creation does not. See also manual 25 (AI Assistant — overview).

### 5.6 Frequently asked questions

**Does the module have "contact types" or an interaction log?**
No. There is **no** type or category field, and no log of calls / emails / meetings. The role, function and department are **free text**. Interactions and the sales pipeline are in the CRM module (manual 06).

**Can I export or print my contacts?**
No. The module offers neither export (CSV, PDF), nor printing, nor file upload. For a bulk extract, go through a database administrator.

**Can I select several contacts and delete them at once?**
No. There is no bulk action; each deletion is individual and confirmed.

**Why does my "Delete" button give an error?**
Two causes. Either you are **not an administrator** (403 refusal): ask an administrator. Or the contact is **linked** to a project, a contract or a document (409 refusal): unlink it first.

**Is deletion reversible?**
**No.** Unlike companies (deactivated and reversible), a deleted contact is **permanently erased**. When in doubt, edit the record rather than delete it.

**Why does a company not appear in the "Company" menu at creation?**
The menu loads only the **first 100** companies. Create the contact without a company, then link it from the **edit** modal, which knows how to preserve the link even outside this list of 100.

**Is a company's primary contact automatically pre-filled in a quote or an opportunity?**
No. No automatic pre-filling. You choose the point of contact when creating the document.

**Can I give a contact access to the ERP?**
No. A contact is **data**, not an account. User accounts are managed in Administration (manual 28 / Configuration).

**Can two contacts have the same email or the same name?**
Yes. There is **no** uniqueness constraint or duplicate detection. Search before creating to avoid duplicates.

**Does search by company or by role really work?**
Yes. The search does cover the **name**, the **email**, the **company name** and the **role / position**, as the placeholder indicates. (This was a limitation of older versions; it is no longer the case.)

**What is the difference between "Role / Position", "Function" and "Department"?**
There is no imposed rule: they are three free-text fields. Useful convention: `Role / Position` = the exact title ("Purchasing Director"); `Function` = a more general category ("Management"); `Department` = the unit ("Purchasing"). Only the role / position appears in the list; the function and the department are seen when opening the record.

**Does the "Primary" badge reflect the primary contact chosen on the company side?**
The badge reflects the "Primary contact" checkbox **ticked on the contact**. For perfect consistency with the company record, designate the person on both sides (section 3.5).

**How many contacts can I save?**
No fixed limit. Performance depends on the database; pagination stays at 20 per page.

**Is a contact's address used for billing or delivery?**
No. Billing and delivery addresses are managed at the company level or directly in the documents (quote, invoice, purchase order). The contact's address is purely informational.

---

## 6. Summary

- **Mission**: address book of **individuals**, linked or not to a company; twin module of Companies (manual 04), but a separate page.
- **Access**: sidebar → **MANAGEMENT** group → **Contacts** (icon `Users`), address `/contacts`.
- **Code**: no `routers/contacts.py` — the CRUD lives in **`companies.py`** (5 `/contacts` endpoints: list, stats, creation, update, deletion); the AI assistant is in **`contacts_ai.py`**; no `api/contacts.ts` — the client is in **`companies.ts`**.
- **Statistics**: 4 **tenant-wide** cards (Contacts, Companies, With email, With phone) via `GET /contacts/stats` — independent of the page and the search.
- **Search**: covers **name + email + company + role**, server-side, across the whole tenant.
- **List**: sortable table (by column, on the current page) and resizable; cards on mobile; **20 per page**; server sort by last name.
- **Fields**: first name and last name **required**; email, phone, mobile, role, function, department, address, notes; **"Primary contact"** checkbox present at both creation **and** editing.
- **Linking**: `company_id` optional; validated within the tenant; dropdown limited to **100** companies at creation, but the existing link is **preserved** on editing.
- **Primary contact**: **only one per company**, reconciled automatically by the server, within a transaction.
- **Deletion**: **permanent** (physical erasure), reserved for **administrators**; clear **409 refusal** if the contact is linked to a project, a contract or a document. The button stays visible for everyone (403 refusal server-side for a non-admin).
- **AI assistant**: reads your contacts / companies and **creates** a contact **upon confirmation** (version 1 = creation only); the chat consumes credits, the confirmation does not.
- **Security**: strict tenant isolation; read-only mode on an inactive subscription; no data from another tenant accessible.
- **Absences to know**: no type / interaction, no export or printing, no upload or photo, no bulk action, no detail page, no duplicate detection, no email format validation.

---

**Verified source files**:
- `ERP_REACT/backend/routers/companies.py` — contacts CRUD (shared with Companies): list, statistics, creation, update, deletion, guards and transactions.
- `ERP_REACT/backend/routers/contacts_ai.py` — AI assistant (propose / confirm pattern, read whitelist, credit billing).
- `ERP_REACT/frontend/src/pages/ContactsPage.tsx` — `/contacts` page (list, statistics, search, modals).
- `ERP_REACT/frontend/src/components/contacts/ContactsAssistantTab.tsx` — AI assistant modal.
- `ERP_REACT/frontend/src/api/companies.ts` (contacts) and `ERP_REACT/frontend/src/api/contactsAi.ts` (assistant) — API clients.
- `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json` — labels `contacts.*` (lines 1039-1118).

**Related manuals**:
- Manual 04 — Companies (clients and suppliers): `03-gestion-entreprises.md`
- Manual 06 — CRM and opportunities (interactions, pipeline): `05-gestion-crm-opportunites.md`
- Manual 08 — Quotes: `07-ventes-soumissions.md`
- Manual 09 — Projects: `08-ventes-projets.md`
- Manual 23 — Emails (automatic matching by email): `21-communication-emails.md`
- Manual 25 — AI Assistant (overview): `24-communication-assistant-ia.md`
- Manual 28 — Configuration (user accounts, categories): `30-configuration.md`

---

*Constructo AI ERP Manual — Module 04 Contacts — v3.0 verified against the source code — July 2026*
