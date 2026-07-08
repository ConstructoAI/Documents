# Module 34 — Real Estate (developer and public storefront)

> **Version**: 3.0 (rewrite verified line by line against the source code of July 7, 2026 — the module is now a distinct application with **two surfaces**: a public storefront and a developer area; major corrections: the actual route is `/immo` and not `/immobilier`, **14 tabs** and not 13, addition of the Cadastre tab, of the publication flow to the storefront, of the AI assistant, of the actual AI models and of the actual permissions)
> **Access**: ERP login page → "Real Estate" tile (internal link), or directly at the URL `app.constructoai.ca/immo`. Return to the ERP via the "Constructo AI Portal" link in the top bar.
> **Reference code (application)**: **separate** React application `IMMO_REACT/frontend` (base `/immo`, `App.tsx:41`), served by the `constructo-erp-react` service (B2B/C2B model — no dedicated hosting service)
> **Reference code (backend)**: five routers mounted in `ERP_REACT/backend/erp_api.py` (defensive block) — `routers/immobilier.py` (developer area, ≈ 61 endpoints), `routers/immo_ai.py` (AI assistant, 5 endpoints, 9 tools), `routers/fonds_prevoyance.py` (Law 16, ≈ 31 endpoints), `IMMO_REACT/backend/routers/public.py` (public storefront, 4 endpoints), `IMMO_REACT/backend/routers/publish.py` (publication, 5 endpoints)
> **Actual API paths**: `/api/erp/v1/immobilier` (developer), `/api/erp/v1/immo/ai` (AI assistant), `/api/erp/v1/fonds-prevoyance` (Law 16), `/api/immo/v1/public` (storefront, **no authentication**), `/api/immo/v1/promoteur` (publish / unpublish)
> **PostgreSQL tables**: a set of `immo_*` tables **per tenant** (land parcels, projects, financing, units, inspections, payments, drawdowns, phases, marketing, deliveries, documents); a set of `fp_*` tables **per tenant** for Law 16; and **one shared table** `public.immo_listings` (the published listings, partitioned by the `tenant_schema` column)
> **AI model**: in-depth analyses = Claude Opus 4.8 (`claude-opus-4-8`); conversations, reports and suggestions = Claude Sonnet 4.6 (`claude-sonnet-4-6`). All AI consumption is billed to the tenant's **prepaid AI credits**, with a 30% markup.
> **Scope**: this module covers the **new real estate development cycle** (land → projects → financing → construction → units → marketing → delivery), the **co-ownership contingency fund (Law 16)**, a **cadastral analysis**, an **AI assistant**, and — a major new feature — a **public listings storefront** on which the developer publishes its units for sale. It is **not** a full rental-management tool (no leases, no tenant portal, no rent generation) nor a connector to Centris or an official registry.

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

Give a Quebec real estate developer or general contractor a single place to **run a new real estate project end to end** — from spotting a piece of land to handing over the keys — and to **publicly showcase the units for sale** on a storefront browsable by any buyer, without them having to log in.

The module answers concrete and distinct needs depending on the person:

- On the **buyer / visitor** side: "Which new properties are for sale, in which city, at what price, and how do I reach the developer?"
- On the **developer** side: "How do I quickly publish my units with photos? Where do my land parcels, my financing, my construction phases stand? What contribution to the contingency fund should I plan for my co-ownership? Is this land buildable?"

### 1.2 Two surfaces in a single application

The module is a standalone React application (`IMMO_REACT`) served under `app.constructoai.ca/immo`. It presents **two surfaces** sharing the same layout (dark sidebar + top bar):

| Surface | Authentication | Who it's for | Content |
|---------|----------------|--------------|---------|
| **Public storefront** | None (open to all) | Buyers, general public | Home, listing search, detail page with the developer's contact information |
| **Developer area** | SSO login with the ERP credentials | Company (tenant) | Listing publication, full management (14 tabs), Law 16 contingency fund, cadastre, AI assistant |

The **core of the module** is the **publication flow**: a unit created in the developer area becomes publishable (or removable) from the storefront via the "My listings" screen. Publishing creates or updates a row in the shared `public.immo_listings` table; the storefront only shows active listings.

### 1.3 What the module does (verified against the code)

- **Public storefront**: a catalog of listings with advanced search (city, type, price, area, bedrooms, bathrooms, status, sort), pagination (24 per page), a detail page with a photo gallery and an email contact button.
- **Publication**: publish or remove each unit, attach up to **12 photos** (actual upload, automatic compression, drag and drop, reordering, choice of cover photo).
- **Getting-started assistant** ("New project"): a single form creates a land parcel, a project and several units all at once.
- **Real estate management (14 tabs)**: dashboard, land, cadastre, projects, financing, construction (phases), units, marketing, delivery, inspections, payments, documents, calculators, contingency fund.
- **Six financial calculators**: monthly payment, amortization, interim interest, CMHC premium, ROI, total cost — with the semi-annual compounding specific to Canadian mortgages.
- **Automatic generation of the drawdowns** of a construction loan (7 steps according to progress).
- **Law 16 contingency fund**: co-ownerships, inventory of components, studies, maintenance logbook, 25-year projections (3 scenarios), sale certificates (art. 1069 C.c.Q.) and AI advice.
- **Cadastral analysis**: search for a lot by address or number, OpenStreetMap map, feasibility (zoning), constraints, with attachment to a project.
- **AI assistant**: a chat that **proposes** an action and only executes it after the user's **confirmation** (create a land parcel, a project, a unit, financing, change a status, publish a listing).
- **Bilingual** French / English (toggle in the top bar) and **light / dark theme**.

### 1.4 What the module does NOT do (important limits)

> **Read this before relying on the module.** Several natural expectations are **not** covered.

- **No rental management.** No screen for leases, tenants, rental expenses or a tenant portal. Translation labels inherited from the ERP exist for these notions, but **they are wired to no screen** in the Real Estate application: these modules do not exist here.
- **Inspections and Payments tabs are read-only.** You can view a project's inspections and payments, but the interface **does not allow** creating, editing or deleting them (no command bar, no entry dialog). This data comes from elsewhere (other flows or technical entry).
- **Documents tab without real upload.** The entry dialog only captures a **text "File path"** plus metadata; it uploads no file (unlike listing photos). Editing a document is impossible: you can only create and delete.
- **A single real file upload** in the entire application: the **listing photos**.
- **A single export**: the **contingency fund report** (Law 16), downloadable as a `.md` file. No other PDF or CSV export anywhere.
- **No connection to Centris, to an official registry or to the real-time assessment roll.** The cadastre relies on open data (OpenStreetMap) for indicative purposes.
- **AI assistant absent from the public storefront**: it only appears for a logged-in developer.
- **Areas in square meters (m²)** everywhere in the developer area (the Law 16 reconstruction-value calculation, however, uses the square foot).
- **No automatic reminder** by email, notification or calendar for deadlines (end of financing, expected delivery, expiration of a document).
- **No dedicated business roles** ("real estate director", "broker", "inspector"): see the permissions below.

### 1.5 Access

- **From the ERP**: login page → "Real Estate" tile.
- **Directly**: `app.constructoai.ca/immo`.
- **Default screen (visitor)**: the storefront Home.
- **Default screen (logged-in developer)**: "My listings".
- **Sidebar** — public section: **Home**, **Listings**; section visible depending on state:
  - Logged out: **Become a developer** (leads to ERP account creation), **Developer area** (login).
  - Logged in: collapsible section **Management** → **My listings**, **Real estate management**.
- **Top bar**: the **"Constructo AI Portal"** link (return to the ERP), the **FR / EN** toggle, the **light / dark theme** toggle, then the account menu (logged in) or the **Become a developer** + **Sign in** buttons (logged out).

### 1.6 Permissions and roles

> **Important security note.** Unlike most other ERP modules, the developer area **enforces no role**: any authenticated user of the tenant can create, edit, delete **and publish to the public storefront**.

| Action | Who can do it | Technical guard |
|--------|---------------|-----------------|
| Browse the storefront, search, open a detail page | **Anyone** (public) | None (public endpoints) |
| Sign in to the developer area | A tenant user (same credentials as the ERP) | Two-step ERP SSO |
| Manage the 14 tabs (land, projects, units, etc.) | Any logged-in tenant user | `get_current_user` (no `require_role`) |
| Publish / remove a listing, manage photos | Any logged-in tenant user | `get_current_promoteur` + `require_publish_access` |
| Use the AI assistant (propose → confirm) | Any logged-in user, **if the tenant has AI credits** | `get_current_user` + prepaid credits |
| Use the contingency fund AI tools | Any logged-in user, **if AI credits** | `get_current_user` + prepaid credits |

- **Tenant isolation**: all `immo_*` and `fp_*` data lives in the company's own PostgreSQL schema; the shared `public.immo_listings` table is always filtered by `tenant_schema`. No cross-tenant access is possible.
- **Consultation mode (inactive subscription)**: the developer's access derives from the subscription state (see §5.2). A **cancelled or absent** subscription puts the tenant in **consultation mode**: reads work, but **publishing or removing a listing is refused** with the message "Consultation mode: inactive subscription" (403 error). A **deactivated** company is simply logged out (401).
- **AI credits**: each call to the assistant or to the Law 16 advice first checks the service availability, then the prepaid credit balance; an exhausted balance returns a **402** error shown on screen. The super-administrator and exempt companies are not blocked.

### 1.7 The screens at a glance

| Surface | Screen | Route | Role |
|---------|--------|-------|------|
| Storefront | Home | `/` | Hero, search, indicators, recent listings |
| Storefront | Listings | `/annonces` | Advanced search + paginated grid |
| Storefront | Detail page | `/annonce/:id` | Gallery, features, contact |
| Developer | Sign in | `/promoteur/connexion` | ERP SSO |
| Developer | My listings | `/promoteur` | Publish / remove + photos |
| Developer | New project | `/promoteur/nouveau-projet` | Guided getting-started assistant |
| Developer | Real estate management | `/promoteur/immobilier` | 14 tabs (including Cadastre and Contingency fund) |

---

## 2. Interface

### 2.0 Elements common to both surfaces

- **Sidebar**: title "Real Estate", subtitle "Constructo AI", footer "Constructo AI © 2026". The entries change depending on whether you are logged in or not (see §1.5).
- **Top bar**: hamburger button (on mobile), "Constructo AI Portal" link (return to the ERP), title of the current page, FR / EN toggle, light / dark theme toggle.
- **Language**: the preference is kept locally (key `immo_lang`). The interface, and the AI assistant's replies, adapt to French or English.
- **Session**: the developer's login token is kept locally (key `immo_token`). If it expires or is revoked, a 401 error triggers an **automatic sign-out** and a return to the login page — there is no dedicated revocation warning screen.

---

### 2.1 Storefront — Home

The general public's entry page.

- **Hero**: title "Find your next property", a subtitle, and a **search bar** (placeholder "City, neighborhood, property type…") with the **Search** button, which redirects to the listings list applying the search.
- **Four key indicators** (fed by `GET /public/stats`): **Available listings**, **Cities served**, **Developers**, **Average price**. While loading, the value shows "…"; if unavailable, "—".
- **Recent listings**: a grid of six listing cards, with a "See all" link.
- **Developer call to action** (bottom of page): "Are you developing real estate projects?" with the **Become a developer** (account creation) and **I already have an account** (login) buttons.
- **States**: a loading indicator on startup; a "No listings yet" message if there is nothing to show.

**Listing card (element reused everywhere)**: cover photo (or an icon if no photo), a **status** badge in the top-left corner, a **type** badge in the top-right corner, the title, the city, a truncated description, bedroom / bathroom / area icons, the price (or "Price on request"), and "View details".

**Status badge**: Available (green), Reserved (amber), Sold (gray), Listing (blue, for any other case).

---

### 2.2 Storefront — Listings (advanced search)

A search screen comparable to a Centris-style portal. **All filters are driven by the address (URL)**, which makes a search shareable simply by copying and pasting the link.

**Base row (always visible)**

| Field | Type | Options |
|-------|------|---------|
| Search | Text | Free-text search |
| City | Drop-down menu | Actual cities present in the listings (`GET /public/filters`) |
| Property type | Drop-down menu | Condo, Apartment, House, Commercial, Penthouse, Other |
| Sort by | Drop-down menu | Most recent, Price ascending, Price descending, Largest area |

**Advanced search (expandable panel)**

| Field | Type |
|-------|------|
| Min price / Max price | Number ($) |
| Min area / Max area | Number (m²) |
| Bedrooms | 1+ to 5+ |
| Bathrooms | 1+ to 3+ |
| Status | All, Available, Reserved, Sold |

- **Apply** and **Reset** buttons.
- **Results**: an "N listings" counter, the card grid (24 per page) and **pagination**.
- **States**: "No results" when a filter returns nothing, "No listings" when the storefront is empty.

---

### 2.3 Storefront — Listing detail page

- "Back to listings" return link.
- **Gallery**: one large photo, plus clickable thumbnails if there are several photos; an icon if the listing has no photo.
- **"Description" card**: the listing text, or "No description provided".
- **Side card**: the title and its status badge, the address, the **price** (or "Price on request"), the bedroom / bathroom / area icons, and above all the means of contact:
  - **"Request information"** button: opens the email software addressed to the developer (falls back to `info@constructoai.ca` if no address is provided).
  - **Phone** button (shown only if a number is provided).
- **"Features" card**: Type, Area, Bedrooms, Bathrooms, Postal code, Developer, **Published on**.
- **State**: "Listing not found" if the listing does not exist or is no longer active.

> **Good to know (privacy)**: the public detail page displays the developer's **email** and **phone** in plain text. This is a deliberate choice to allow direct contact; only provide contact details you are willing to make public.

---

### 2.4 Developer area — Sign in

Title "Developer area".

| Field | Description |
|-------|-------------|
| Company email | The same as for the ERP |
| Password | The same as for the ERP |

- **Sign in** button. Note shown: "Same credentials as your Constructo AI ERP".
- "Create a Constructo AI account" link (leads to ERP registration).
- **How it works**: sign-in happens in two steps against the ERP (identify the company, then the user). The token obtained is reused for all developer-area calls — it is genuine single sign-on (SSO) with the ERP.

---

### 2.5 Developer area — My listings

The developer's central screen: this is where you **publish** and where you **manage photos**.

- **Header**: title "My listings", the developer's email address and the "N published listing(s)" count.
- **Buttons**: **New project**, **Real estate management**, **View the storefront**.
- **List**: one card per tenant unit, with:
  - the project name and the unit number;
  - a **Published** (green) or **Not published** (gray) badge;
  - a type / area / price line;
  - a **Photos** button;
  - a **Publish** button (if not published) or **Remove** button (if published).
- **Empty state**: "No units to publish" with a "Create a project" call to action.
- **Behaviors**:
  - On publication, the **listing's contact email is pre-filled** with that of the logged-in developer.
  - In consultation mode (inactive subscription), publishing or removing returns a shown error: "Consultation mode: inactive subscription".

**"Listing photos" dialog** (Photos button)

- **Up to 12 photos**; the **first is the cover**.
- **Accepted formats**: PNG, JPG, WEBP, GIF. **Limits**: 10 MB per photo, 60 MB combined.
- **Automatic browser-side compression**: large images are reduced (longest side brought down to 1920 px, converted to JPEG) before sending, to stay under the limits.
- **Adding**: a **drag-and-drop** zone or an **Add photos** button.
- **Per thumbnail**: **Set as cover** (star icon), **Remove** (X), and **drag reordering**. The cover carries a "Cover" badge.
- **Buttons**: **Cancel** and, depending on state, **Save** (if the unit is already published) or **Save and publish** (with the note "Saving will also publish this unit to the storefront").

> **Good to know**: photos and contact details set manually on a listing are **preserved** if you republish the unit from the ERP without redefining them (see §5, preservation via COALESCE). On the other hand, the **price, status, area, type and address** are **refreshed** from the unit's record at each publication.

---

### 2.6 Developer area — New project (getting-started assistant)

A single form that, in one submission, creates a land parcel, then a project, then as many units as you add. It is the fast track to publication.

**Section 1 — "Your project"**

| Field | Required | Details |
|-------|:--------:|---------|
| Project name | Yes | Example placeholder: "Le Griffin - phase 2" |
| Project type | — | Residential, Commercial, Mixed |
| Address | Yes | Example: "1200 rue Ottawa" |
| City | Yes | — |
| Postal code | — | — |
| Description | — | Text area |

> The address and city are used to **locate the listings**: they are carried by the land parcel the assistant creates.

**Section 2 — "Your units"** (repeatable rows)

| Field | Details |
|-------|---------|
| Number | Unit identifier |
| Type | Condo, Apartment, House, Penthouse |
| Area (m²) | — |
| Bedrooms | — |
| Bathrooms | — |
| Sale price | — |

- **Add a unit** and **Remove** buttons.
- **Create the project** button → success screen "Project created!" ("N unit(s) added") with the **Publish my units** and **Create another project** buttons.
- **Validations**: the name is required, the address and city are required, and at least one unit is needed.

> **A nuance to know**: this assistant uses **different** type lists than detailed management (see §4.3). Here "Project type" offers Residential / Commercial / Mixed; in the Projects dialog of detailed management, it offers Condos / Rental / Mixed / Commercial / Houses.

---

### 2.7 Developer area — Real estate management (14 tabs)

The tab bar shows the icon alone and a short label on mobile. Here are the 14 tabs, in order.

| # | Tab | Type | Summary |
|---|-----|------|---------|
| 1 | Dashboard | View | Indicators + quick calculator |
| 2 | Land | Full CRUD | Prospecting → acquisition |
| 3 | Cadastre | Tool | Lot analysis (map) |
| 4 | Projects | Full CRUD | Development projects |
| 5 | Financing | Full CRUD | Bank loans |
| 6 | Construction | Full CRUD | Construction phases |
| 7 | Units | Full CRUD | Dwellings / spaces |
| 8 | Marketing | Full CRUD | Sales strategy |
| 9 | Delivery | Full CRUD | Handover to recipients |
| 10 | Inspections | **Read-only** | Inspections + compliance |
| 11 | Payments | **Read-only** | Financial transactions |
| 12 | Documents | Create + delete | Documentary references |
| 13 | Calculators | Tools | 6 financial calculations |
| 14 | Contingency Fund | CRUD + AI | Law 16 (co-ownerships) |

> **Major correction versus the old manual**: the module has **14 tabs**, not 13 — the **Cadastre** tab was added. The internal comment of the page file, moreover, still wrongly says "13 tabs".

#### 2.7.1 Dashboard tab

- **Four key indicators**: Total land parcels, Total projects, Approved financing, Units sold.
- **Two lists**: Land parcels by status, Projects by status (with colored badges).
- **Built-in quick calculator**: Principal / Annual rate / Term → Monthly payment, Total cost, Total interest.

#### 2.7.2 Land tab (CRUD)

- **Command bar**: **New land parcel** button, search, **Status** filter (Prospecting, Offer in progress, Acquired, In development, Rejected).
- **Table**: Number, Address, City, Area, Zoning, Asking price, Status, Actions (edit / delete).
- **Entry dialog**: Address\*, City\*, Postal code, Area (m²), **Zoning** (Residential, Commercial, Mixed, Industrial), Owner (name), Asking price ($), Notes.

#### 2.7.3 Cadastre tab

A lot analysis tool, entirely browser-side (OpenStreetMap map), with no dedicated endpoint in the Real Estate backend. See the detail in §2.7.15.

#### 2.7.4 Projects tab (CRUD)

- **Command bar**: **New project** button, search, **Status** filter (Planning, In progress, Construction, Completed, Cancelled).
- **Table**: Number, Name, Type, Dwellings, Budget, ROI %, Status, Actions.
- **Entry dialog**: Project name\*, **Project type** (Condos, Rental, Mixed, Commercial, Houses), Number of dwellings, Total budget ($), Land cost ($), Construction cost ($), Estimated sales revenue ($), Start date, End date, Description, Notes.

#### 2.7.5 Financing tab (CRUD)

- **Filter**: "Filter by project".
- **Table**: Number, Bank, Loan type, Requested amount, Approved amount, Rate %, Status, Actions.
- **Entry dialog**: Project (drop-down menu), Bank\*, **Loan type** (Mortgage, Construction, Bridge, Line of credit), Requested amount ($), Annual interest rate (%), Amortization period (years), Down payment (%), Notes.

#### 2.7.6 Construction tab — phases (CRUD)

- **Prerequisite**: a **Select a project** menu is mandatory; without a chosen project, a message and an icon invite you to select one.
- **Table**: #, Phase, Status, **Completion** (progress bar), Planned budget, Actual cost, Delay (days), Actions.
- **Entry dialog**: Phase name\* (choice among suggested standard phases, or free text), Phase number, **Status** (Upcoming, In progress, Delayed, Completed, Suspended), Completion (%), Planned budget ($), Planned start date, Planned end date, Delay (days), Reason for delay, plus the check boxes **Inspection required**, **NBC-compliant** (National Building Code), **Materials ordered**, **Materials received**, and Notes.

#### 2.7.7 Units tab (CRUD)

- **Prerequisite**: selecting a project is mandatory.
- **Table**: Number, Type, Area, Bedrooms, Bath, Floor, Sale price, Status, Actions.
- **Entry dialog**: Unit number\*, **Type** (Condo, Apartment, Commercial, House, Penthouse), Area (m²), Bedrooms, Bathrooms, Floor, Sale price ($), Monthly rent ($), Notes.

> It is the unit created here (or by the New project assistant) that becomes **publishable** in "My listings".

#### 2.7.8 Marketing tab (CRUD)

- **Prerequisite**: selecting a project.
- **Four indicators**: Units sold, Units rented, Average sale price, Pre-sales rate.
- **Table**: Strategy, Average price, Average rent, Pre-sales target, Marketing budget, Broker, Launch, Actions.
- **Entry dialog**: **Sales strategy** (Pre-sale, Direct sale, Rental, Mixed), Launch date, Average sale price ($), Average rent ($), Pre-sales target (%), Marketing budget ($), Website, Broker (name), Broker commission (%), plus the boxes **Brochure ready**, **Sales plans ready**, **3D model**, and Notes.

#### 2.7.9 Delivery tab (CRUD)

- **Prerequisite**: selecting a project.
- **Table**: Unit, Recipient, Type, Delivery date, Keys (Yes / No badge), Satisfaction (n/10), Claims, Actions.
- **Entry dialog**: Unit ID, Recipient name, **Recipient type** (Buyer, Tenant), Planned delivery date, Warranty period (months), Deficiency list;
  - **Documents delivered** group (boxes): Keys handed over, Deed of sale signed, Lease signed, Co-ownership manual, As-built plans, Certificate of compliance;
  - **Warranties** group (boxes): Pre-delivery inspection, Deficiencies corrected, Legal warranty (latent defects), GCR warranty (Garantie de construction résidentielle, Quebec's residential construction warranty);
  - Satisfaction rating (1 to 10), Client comments, Notes.

#### 2.7.10 Inspections tab (READ-ONLY)

- **Prerequisite**: selecting a project.
- **Table**: Type, Category, Inspector, **Score** (colored bar), Deficiencies, Status, **Compliance** (NBC / CCE / CSST badges), Date.
- **No** create, edit or delete **button**: this tab is **consultative**.

#### 2.7.11 Payments tab (READ-ONLY)

- **Prerequisite**: selecting a project.
- **Table**: Type, Category, Amount, Recipient, Description, Date, Status.
- **No action button**: this tab is **consultative**.

#### 2.7.12 Documents tab (create + delete)

- **Prerequisite / filters**: selecting a project + **Category** filter.
- **Table**: Name, Category, File type, Document date, **Confidential** (red badge), Status, Actions (delete).
- **"New document" dialog**: Document name\*, **Category** (Contracts, Permits, Plans and drawings, Technical studies, Financing, Insurance, Correspondence, Inspection reports, Photos, Other), **File type** (PDF, Image, Word, Excel, CAD, Other), **File path** (text), Description, Document date, Expiration date, **Confidential** (box).

> **Important limitation**: this tab **does not host** files. The "File path" field is only a text reference (for example a link to an existing folder). To attach an actual file, use the ERP's Dossiers (document management) module, then paste the link here.

#### 2.7.13 Calculators tab (6 sub-tabs)

All calculations are done server-side; nothing is saved (each calculation is one-off).

| Sub-tab | Inputs | Outputs |
|---------|--------|---------|
| **Monthly payment** | Principal ($), Annual rate (%), Term (years) | Monthly payment, Total cost of credit, Total interest |
| **Amortization** | + **Frequency** (Monthly, Bi-weekly, Weekly) | Summary (monthly payment, total interest, total cost) + table (Period, Payment, Principal, Interest, Balance) — first 24 rows |
| **Interim interest** | Amount borrowed, Annual rate, Construction period (months) | Total interest + month-by-month table (Drawdown, Cumulative balance, Interest) |
| **CMHC premium** | Loan amount, Property value | Loan-to-value ratio, Premium %, Premium in $, Total loan; + Tax on the premium (Quebec 9%); "not insurable" warning if the ratio exceeds 95% |
| **ROI** | Total investment, Annual revenue, Annual expenses, Term | ROI, Annual net profit, Payback period |
| **Total cost** | Principal, Annual rate, Term | Monthly payment, Total cost, Total interest, Principal |

See §4.4 for the exact formulas (semi-annual compounding, CMHC scale).

#### 2.7.14 Contingency Fund tab (Law 16)

See the detailed section §2.7.16.

#### 2.7.15 Cadastre — land analysis (detail)

- **Search** by **civic address** or **lot number** → a list of candidates (lot number, municipality, area).
- **Lot record**:
  - **Lot identity**: lot number, municipality, area, assessed value, CUBF code;
  - **Interactive map** (OpenStreetMap, polygon outline);
  - **Feasibility (zoning)**: zone code, permitted uses;
  - **Constraints**: present or absent, with a percentage;
  - **Proximity**, **Access and logistics**, **Warnings**.
- **"Attach to a project"** button to link the analysis to an existing project.

> **Good to know**: the cadastre relies on open data and serves as decision support. It **does not replace** a certificate of location or an official municipal zoning check.

#### 2.7.16 Contingency fund — Law 16

Reminder banner: "Contingency Fund (Quebec's Law 16) — Study mandatory every 5 years · Maintenance logbook mandatory · Minimum period 25 years". A shared **co-ownership selector** applies to all sub-tabs. Seven sub-tabs:

**a) Co-ownerships** (CRUD)
- Columns: Name, Address, Year, Units, Reconstruction value, Components, Studies.
- The entry dialog offers a **"Calculate automatically (Quebec 2025)"** button for the reconstruction value (see §4.5).

**b) Components** (building inventory)
- Columns: Subcategory, Description, Quantity, Condition, Remaining life, Total cost.
- **Alerts** flag urgency: CRITICAL, URGENT, WARNING, OK.

**c) Studies**
- Columns: Date, Professional, Professional order, Current fund, Recommended, Annual contribution, Compliant.
- Assumption fields: inflation rate, rate of return, contingency.

**d) Maintenance logbook**
- Columns: Description, Type, Planned date, Completed date, Planned cost, Actual cost, Contractor.

**e) Projections**
- **"Generate the 3 scenarios"** button (Uniform, Progressive, Variable).
- **Chart** of the fund balance's evolution.
- **"Save the selected scenario"** button.
- Year-by-year table, with a **shortfall warning** if the fund goes negative in a given year.

**f) Certificates** (art. 1069 C.c.Q. — sale of a unit)
- Columns: Unit, Seller, Buyer, Request date, Issue date, Fund, Arrears, Status.

**g) AI advice** (4 sub-tabs)
- **Full analysis**: health score, risk level, points of attention, recommendations, Law 16 compliance, expert advice.
- **Expert chat**: ask a question, with an "Include the co-ownership context" box.
- **Full report**: **Generate** button, then **Download (.md)** — this is the **only** file export in the entire module.
- **Contribution suggestion**: from the replacement cost, the number of units, the horizon and the current balance → proposes a uniform contribution, a contribution per unit per month, and a progressive contribution by phase.

---

### 2.8 Real Estate AI Assistant

- **Floating "AI Assistant" button**, bottom right, **visible only to the logged-in developer** — never on the public storefront. It opens a sliding side panel.
- **Title**: "Real Estate Assistant — AI Expert: land, projects, units, financing".
- **Panel header**: **New conversation**, **History**, Close.
- **Chat**: user / assistant bubbles (Markdown rich text), 3 starter examples to get going.
- **"Propose → confirm" principle**: the AI never modifies your data directly. It shows a **proposal card** (field-by-field preview) that you validate with **Confirm** or reject with **Cancel**.
- **Six types of confirmable actions**: create a **land parcel**, a **project**, a **unit**, **financing**; perform a **status change**; **publish a listing**. This last case is handled separately: it shows a warning ("publishing makes this listing visible to the public… immediately") and an amber **Publish** button.
- **History**: the list of conversations is **kept server-side**; you can resume or delete a conversation.

> **Good to know**: the assistant only executes after your confirmation, and it acts **only on your tenant** — it cannot target another company. The internal search it uses is isolated and protected against access to sensitive tables (employees, payroll, users, etc.).

---

## 3. Step-by-step workflows

### 3.1 Visitor: from search to contact

1. Open `app.constructoai.ca/immo` (Home).
2. Enter a city or a type in the search bar, then **Search** (or go to **Listings**).
3. Refine with the **advanced search** (price, area, bedrooms, status) and a **sort**.
4. Click a card to open the **detail page**.
5. Browse the gallery and the features.
6. Click **"Request information"** (opens an email to the developer) or use the **phone button**.

### 3.2 Developer: publish quickly (express route)

1. **Sign in** to the developer area (ERP credentials).
2. **New project**: fill in the "Your project" section (name, address, city) and add one or more units (number, type, area, bedrooms, bathrooms, price).
3. **Create the project** → success screen.
4. Click **Publish my units** (or go to **My listings**).
5. For each unit, click **Photos**, add up to 12 images (drag and drop), choose the **cover**, reorder as needed, then **Save and publish**.
6. Check the result with **View the storefront**.

### 3.3 Publish, remove and manage the photos of an existing unit

1. **My listings**: locate the unit (Published / Not published badge).
2. **Publish**: makes the listing visible (the contact email is pre-filled with yours).
3. **Photos**: manage the gallery at any time; "Set as cover" (star), "Remove" (X), drag to reorder.
4. **Remove**: hides the listing from the storefront **without deleting** its data or its photos (you can republish later with the same gallery).
5. If a "Consultation mode: inactive subscription" message appears, reactivate the company's subscription (see §5.2).

### 3.4 Developer: fine management of a project (full cycle)

1. **Land** → New land parcel (address, city, area, zoning). Move the **status** along: Prospecting → Offer in progress → Acquired → In development.
2. (Optional) **Cadastre** → analyze the lot, then **Attach to a project**.
3. **Projects** → New project (type, dwellings, budgets, estimated revenue, dates).
4. **Financing** → New financing (bank, loan type, requested amount, rate, term); set the approved amount once the agreement is obtained.
5. **Drawdowns** → generate them automatically (see §3.5).
6. **Construction** → select the project, create the **phases** (status, completion, planned budget, compliance and materials boxes).
7. **Units** → select the project, create each unit (type, area, bedrooms, price).
8. **Marketing** → define the strategy, the pre-sales target, the broker, the marketing budget.
9. **Delivery** → at handover, enter the recipient, check the documents delivered and the warranties, record the satisfaction.
10. **Inspections** and **Payments** → review the progress (read-only).

### 3.5 Automatically generate the drawdowns of a financing

1. Make sure the financing carries an **approved amount (commitment)**.
2. Launch the automatic generation (dedicated button or the AI assistant).
3. The system creates **7 drawdown steps**, distributed according to a typical construction site's progress: **10%, 15%, 25%, 15%, 20%, 10%, 5%** (total 100%).
4. **Safeguards**: the operation is **atomic** and **idempotent** — if drawdowns already exist, it is refused (409) to avoid duplicates; if the total would exceed the commitment, it is refused (400); the last step absorbs the rounding so the sum comes out exact.
5. Then track each drawdown individually.

### 3.6 Law 16 contingency fund: from the co-ownership to the projection

1. **Contingency Fund** tab → **Co-ownerships** sub-tab → create the co-ownership (name, address, year, units). Use **"Calculate automatically (Quebec 2025)"** for the reconstruction value.
2. **Components** sub-tab → inventory roofs, elevators, windows, etc. (quantity, unit cost, condition, remaining life). The total cost and the remaining life can be calculated automatically.
3. **Studies** sub-tab → create a fund study (professional, professional order, inflation, return and contingency assumptions).
4. **Projections** sub-tab → **Generate the 3 scenarios** (Uniform, Progressive, Variable), examine the chart and the year-by-year table, watch for any **shortfall warning**, then **Save the selected scenario**.
5. **Maintenance logbook** sub-tab → plan and track the work (planned / completed dates, costs, contractor).
6. **Certificates** sub-tab → on the sale of a unit, issue the certificate (art. 1069 C.c.Q.) with the fund state and the arrears.
7. **AI advice** sub-tab → run a **Full analysis**, chat with the **expert**, or generate a **Full report** (downloadable as `.md`).

### 3.7 Use the financial calculators

1. **Calculators** tab → choose the desired sub-tab.
2. **Monthly payment / Total cost**: Principal, Annual rate, Term → monthly payment and interest. The formula uses Canadian **semi-annual compounding** (see §4.4).
3. **Amortization**: add the **Frequency** (monthly, bi-weekly, weekly) to obtain the detailed table.
4. **Interim interest**: Amount, Rate, Construction period → interest to capitalize during the work.
5. **CMHC premium**: Loan amount, Property value → ratio, premium, Quebec 9% tax; beyond 95%, the loan is **not insurable**.
6. **ROI**: Investment, Revenue, Expenses, Term → return and payback period.

> The calculators **save nothing**. To keep a result, copy it into a financing's notes or into a project document.

### 3.8 Analyze a lot on the cadastre and attach it

1. **Cadastre** tab → enter a civic address or a lot number → **Search**.
2. Choose a candidate from the list.
3. Examine the **lot identity**, the **map**, the **feasibility (zoning)**, the **constraints**, the **proximity** and the **warnings**.
4. Click **"Attach to a project"** to link the analysis.

### 3.9 Steer with the AI assistant (propose → confirm)

1. Open the **AI Assistant** (floating button, logged-in developer).
2. Phrase your request in natural language — for example "how many unsold units?" or "create a land parcel in Granby".
3. For a **read** (dashboard, search), the answer displays directly.
4. For an **action** (create, change a status, publish), the AI presents a **proposal card**: check the preview, then **Confirm** or **Cancel**.
5. For a **publication**, read the warning (immediate public visibility) before clicking **Publish**.
6. Resume an earlier discussion via **History**, or start fresh with **New conversation**.

---

## 4. Reference

### 4.1 Endpoints by backend

**a) Developer area — `routers/immobilier.py` (`/api/erp/v1/immobilier`, ≈ 61 endpoints, ERP authentication)**

| Entity | Endpoints |
|--------|-----------|
| Dashboard | `GET /dashboard` |
| Land | `GET /terrains`, `GET /terrains/{id}`, `POST`, `PUT`, `DELETE` |
| Projects | `GET /projets`, `GET /projets/{id}`, `POST`, `PUT`, `DELETE` |
| Financing | `GET /financements`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Units | `GET /unites` (**`projet_id` required**), `POST`, `PUT`, `DELETE` |
| Inspections | `GET /inspections`, `POST`, `PUT` (no delete) |
| Payments | `GET /paiements`, `POST` (no edit or delete) |
| Drawdowns | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE`, `POST /deblocages/generer-auto` |
| Phases | `GET`, `GET /phases/types`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Marketing | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Deliveries | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Documents | `GET`, `GET /{id}`, `POST`, `DELETE` |
| Calculators | `POST /calculer-mensualite`, `/calculer-amortissement`, `/calculer-interets-intercalaires`, `/calculer-prime-schl`, `/calculer-roi`, `/calculer-cout-total` |
| AI | `POST /ia/analyser-projet` (Opus), `/ia/chat`, `/ia/rapport-financement`, `/ia/optimiser-financement` (Sonnet) |

**b) AI Assistant — `routers/immo_ai.py` (`/api/erp/v1/immo/ai`, 5 endpoints, 9 tools)**

| Endpoint | Role |
|----------|------|
| `POST /chat` | Conversation (tool loop, persisted history) |
| `POST /confirm-action` | Executes a proposal **after confirmation** (fully revalidated server-side) |
| `GET /conversations` | List of conversations |
| `GET /conversations/{id}` | Detail (404 if it does not belong to the user) |
| `DELETE /conversations/{id}` | Deletion |

The 9 tools: `recherche_bd` (restricted read-only SQL), `tableau_de_bord_immo`, `calculer_financier_immo`, `proposer_terrain`, `proposer_projet`, `proposer_unite`, `proposer_financement`, `proposer_changement_statut`, `proposer_publication_annonce`.

> **Nuance**: the product memory spoke of "9 tools"; on the interface side, only **6 types of actions** are *confirmable* (land parcel, project, unit, financing, status change, publication). The other three tools are reads that do not require confirmation.

**c) Contingency fund — `routers/fonds_prevoyance.py` (`/api/erp/v1/fonds-prevoyance`, ≈ 31 endpoints, ERP authentication)**

| Entity | Endpoints |
|--------|-----------|
| Reference | `GET /reference` |
| Co-ownerships | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE`, `GET /{id}/statistiques` |
| Components | `GET /coproprietes/{id}/composantes`, `POST`, `PUT`, `DELETE` |
| Studies | `GET /coproprietes/{id}/etudes`, `GET /etudes/{id}`, `POST`, `PUT`, `DELETE` |
| Projections | `POST /etudes/{id}/generer-projections`, `GET /etudes/{id}/projections` |
| Maintenance logbook | `GET /coproprietes/{id}/entretiens`, `POST`, `PUT`, `DELETE` |
| Certificates | `GET /coproprietes/{id}/attestations`, `POST`, `PUT`, `DELETE` |
| AI | `POST /ia/analyze-copropriete` (Opus), `/ia/chat`, `/ia/suggest-contribution`, `/ia/rapport-recommandations` (Sonnet) |
| Calculation | `POST /calculer-valeur-reconstruction` |

**d) Public storefront — `IMMO_REACT/backend/routers/public.py` (`/api/immo/v1/public`, 4 endpoints, NO authentication)**

| Endpoint | Role |
|----------|------|
| `GET /listings` | Advanced search (city, type, price, area, bedrooms, bathrooms, status, sort, text; 100 max per page) |
| `GET /filters` | Cities and values available for the menus |
| `GET /stats` | Home indicators |
| `GET /listings/{id}` | Detail page (404 if the listing is inactive) |

**e) Publication — `IMMO_REACT/backend/routers/publish.py` (`/api/immo/v1/promoteur`, 5 endpoints, developer authentication)**

| Endpoint | Role |
|----------|------|
| `GET /me` | Developer profile |
| `GET /units` | Tenant units + publication state |
| `GET /units/{id}/photos` | Read the photos |
| `POST /units/{id}/publish` | Publish (validates the photos, refused 403 in consultation mode) |
| `POST /units/{id}/unpublish` | Remove from the storefront |

### 4.2 Statuses by entity

| Entity | Statuses |
|--------|----------|
| Land parcel | Prospecting, Offer in progress, Acquired, In development, Rejected |
| Project | Planning, In progress, Construction, Completed, Cancelled |
| Construction phase | Upcoming, In progress, Delayed, Completed, Suspended |
| Unit / Listing | Available, Reserved, Sold |
| Recipient (delivery) | Buyer, Tenant |

> Listing statuses are normalized server-side into `disponible` / `reserve` / `vendu` (no other value is ever accepted at publication).

### 4.3 Taxonomies (types)

| Field | Values |
|-------|--------|
| Zoning (land parcel) | Residential, Commercial, Mixed, Industrial |
| Project type (detailed management) | Condos, Rental, Mixed, Commercial, Houses |
| Project type (New project assistant) | Residential, Commercial, Mixed |
| Loan type | Mortgage, Construction, Bridge, Line of credit |
| Unit type (detailed management) | Condo, Apartment, Commercial, House, Penthouse |
| Unit type (New project assistant) | Condo, Apartment, House, Penthouse |
| Property type (storefront search) | Condo, Apartment, House, Commercial, Penthouse, Other |
| Document category | Contracts, Permits, Plans and drawings, Technical studies, Financing, Insurance, Correspondence, Inspection reports, Photos, Other |
| Sales strategy | Pre-sale, Direct sale, Rental, Mixed |

> **Two type lists coexist** (New project assistant versus detailed management): this is a fact of the code, not a data-entry error. Choose the value that makes sense; it remains editable later in the corresponding tab.

### 4.4 Financial calculations (actual formulas)

- **Periodic rate (Canadian mortgage)**: **semi-annual** compounding, so the per-period rate is `(1 + i/2)^(2/p) − 1` (and not `i/p`), where `i` is the annual rate and `p` the number of periods per year. This is the legal convention in Canada.
- **Monthly payment**: `M = P · [ r (1+r)^n ] / [ (1+r)^n − 1 ]`, with `r` the periodic rate and `n` the number of payments. The last period absorbs the rounding residual.
- **Amortization frequencies**: Monthly = 12 payments/year, Bi-weekly = 26, Weekly = 52.
- **Interim interest**: drawdown assumed **linear** (`amount / term in months`), monthly interest `(rate/100)/12` applied to the cumulative balance.
- **ROI**: `(annual net profit × term / investment) × 100`. Payback period = `investment / annual net profit` (undefined if the profit is nil or negative).
- **CMHC premium** (based on the loan-to-value ratio, LTV):

| LTV | Premium | Note |
|-----|---------|------|
| > 95% | — | **Not insurable** |
| 90.01% to 95% | 4.00% | — |
| 85.01% to 90% | 3.10% | — |
| 80.01% to 85% | 2.80% | — |
| ≤ 80% | 0% | Conventional loan (no premium) |

  The **Quebec 9% tax** on the premium is payable **in cash** (it is not added to the loan).

- **Automatic drawdowns**: 7 steps of **10 / 15 / 25 / 15 / 20 / 10 / 5%** (= 100%).

### 4.5 Contingency fund — Law 16 scales

- **Reconstruction value (Quebec 2025)**: rate per square foot × type factor × age factor.

| Quality | $/sq ft | | Type | Factor | | Building age | Factor |
|---------|--------:|---|------|-------:|---|--------------|-------:|
| Economy | 250 | | Residential | 1.00 | | > 50 years | 0.85 |
| Base | 325 | | Commercial | 1.15 | | … | … |
| Mid-range | 387 | | Mixed | 1.08 | | ≤ 15 years | 1.00 |
| High-end | 487 | | Industrial | 0.95 | | | |

- **Component condition factors** (affect remaining life): Excellent 1.10 · Good 1.00 · Fair 0.85 · Poor 0.70 · **Critical 0.00** (nil remaining life → immediate replacement).
- **Projection period**: 25 years by default (legal minimum), configurable from 1 to 100 years.
- **Three contribution scenarios**:
  - **Uniform**: a constant annual contribution computed by annuity (net present value of future expenses).
  - **Progressive**: a contribution that rises 3%/year, adjusted by a solver to balance the fund.
  - **Variable**: a contribution that targets, each year, a reserve of at least the next major expense plus 15% (with a floor of $50,000).
- **Cyclical re-inflation**: each component is replaced at the end of its remaining life, then at each theoretical life cycle; its cost is **re-inflated** at each replacement.
- **Intermediate shortfall alert**: even if the final balance is positive, the system flags the year and the amount of the **maximum shortfall** along the way (the balance may dip below zero in a given year).

### 4.6 Generated reference numbers

Entities receive a unique number at creation (for example `TER-#####` for a land parcel, `IMMO-#####` for a unit, `DEB-#####` for a drawdown, `FIN-…` for a financing). These numbers are generated reliably (never by a simple counter), with a retry in case of collision.

### 4.7 Listing photos — limits

| Rule | Value |
|------|-------|
| Maximum number | 12 photos (the 1st = cover) |
| Accepted formats | PNG, JPG, WEBP, GIF (and `https://` links) |
| Rejected format | SVG (protection against malicious scripts) |
| Size per photo | ~ 10 MB |
| Combined size | 60 MB |
| Compression | Automatic browser-side (1920 px max, JPEG) |

### 4.8 Useful limits and validations

- **Units**: the list requires a `projet_id` (all the tenant's units are never listed at once); creating a unit validates the existence of the parent project (404 otherwise).
- **Calculators**: rate capped at 100%, term at 100 years, construction period at 600 months (overflow protections).
- **Empty date fields**: automatically converted to "empty" to avoid database errors.
- **Cascading deletions**: deleting a **project** removes its linked entities (financing, units, phases, etc.) **and deactivates its public listings**; deleting a **unit** also deactivates its listing (otherwise it would remain displayed forever); deleting a **land parcel** unlinks the projects that referenced it.

---

## 5. Integrations and FAQ

### 5.1 Single sign-on (SSO) with the ERP

The developer area shares the ERP's authentication: same credentials, same token. Signing in to Real Estate means signing in to the company. Signing out, or the token expiring, brings you back to the login page.

### 5.2 Stripe and consultation mode

The developer's access depends on the company's subscription state (queried through the ERP):

| Level | Condition | Effect |
|-------|-----------|--------|
| **Full** | Subscription active, in trial, being cancelled, past due or unpaid | Read and write |
| **Consultation** | No Stripe subscription or cancelled subscription | Read-only; **publish / remove refused (403)** |
| **Blocked** | Deactivated company | Sign-out (401) |

In practice, if you see "Consultation mode: inactive subscription" while trying to publish, it means the company's subscription must be brought up to date (ERP Configuration / subscription module).

### 5.3 AI credits

The Real Estate assistant and the contingency fund's AI advice **consume the tenant's prepaid AI credits** (the same as the rest of the ERP), with a 30% markup on the actual cost. An exhausted balance returns a 402 error. In-depth analyses use Claude Opus 4.8; conversations, reports and suggestions use Claude Sonnet 4.6.

### 5.4 Public storefront and privacy

The storefront is **open to all**, without login. It never exposes the internal tenant identifier, but it **deliberately** displays the developer's email and phone on each detail page, to allow contact. A per-IP request cap limits automated harvesting.

### 5.5 Link with other modules

- **ERP Dossiers (document management) module**: to be used to host actual files, since the Real Estate Documents tab only stores a text path.
- **Accounting module**: the payments in the Payments tab are **not** posted automatically to the general ledger. Enter them manually as needed.
- **ERP Projects module**: real estate projects (`immo_projets`) are **distinct** from the ERP's construction projects; there is no automatic link between the two.
- **Law 16**: the contingency fund is a full sub-module in its own right, with its own tables and its own AI assistant.

### 5.6 FAQ

**Q: Where did the Properties, Tenants, Leases and Expenses tabs from the old manual go?**
A: They do not exist here. The Real Estate application is a **development** and **storefront** tool, not a rental-management tool. Translation labels linger in the language files, but no screen displays them.

**Q: Can I create an inspection or a payment from the interface?**
A: No. The Inspections and Payments tabs are **read-only**: no create button, no entry dialog. You can only view.

**Q: How do I attach an actual PDF to a project document?**
A: The Documents tab does not upload a file; it only keeps a text "File path". Host the file in the ERP's Dossiers module, then paste its link into this field. The **only** real upload in the application is that of the **listing photos**.

**Q: If I republish a unit from the ERP, do I lose the photos and the text I fine-tuned on the listing?**
A: No for the photos, the title, the description and the contact details: they are **preserved** if you do not redefine them. **Yes**, however, for the **price, the status, the area, the type and the address**, which are **refreshed** from the unit's record at each publication. If you had adjusted the price directly on the listing, it will be overwritten by the unit's sale price.

**Q: Does removing a listing delete its photos?**
A: No. "Remove" hides the listing (it becomes inactive) but keeps all its data and its gallery. You can republish later without redoing everything.

**Q: Can the AI assistant modify my data without my approval?**
A: No. It **proposes** an action and waits for your **confirmation**. It acts only on your company, and it is protected against access to sensitive data (payroll, employees, users).

**Q: Is there a PDF or CSV export of the listings, projects or financing?**
A: No. The **only** file export in the entire module is the **contingency fund report** as `.md` (Contingency Fund tab → AI advice → Full report).

**Q: Does the cadastre replace a certificate of location?**
A: No. It gives an indicative reading (open data, OpenStreetMap map) to help assess a lot. Always validate the zoning and the constraints with the municipality.

**Q: Do the calculators save their results?**
A: No. Each calculation is one-off. Copy the results into a financing's notes or into a document if you want to keep them.

**Q: Can everyone in the company publish to the public storefront?**
A: Yes. The developer area enforces no role: any logged-in tenant user can create, edit, delete and **publish**. If control is a concern, govern this usage internally.

**Q: Why can't I publish even though I am properly logged in?**
A: Probably **consultation mode**: the company's subscription is inactive. The "Consultation mode: inactive subscription" message indicates it. Bring the subscription up to date to re-enable publication.

**Q: How many units can I create per project?**
A: No fixed limit. In practice, the unit list requires choosing a project, which keeps the display manageable even with many units.

**Q: Does the contingency fund guarantee that the fund will never go into a shortfall?**
A: No, and the system is transparent about it: even when the final balance is positive, it flags the **year and the amount of the maximum shortfall** along the way, for three contribution scenarios.

---

## 6. Summary

- **Two surfaces**: a **public storefront** without login (Home, Listings, Detail page) and a **developer area** in ERP SSO.
- **Actual route**: `/immo` (separate `IMMO_REACT` application), not `/immobilier`.
- **Core of the module**: the **publication flow** — a unit from the developer area becomes a public listing (shared `public.immo_listings` table) from "My listings".
- **Express route**: the "New project" assistant creates land + project + units in one submission, then "Publish my units".
- **Listing photos**: up to 12, drag and drop, reordering, cover, automatic compression — the **only** real file upload.
- **Detailed management: 14 tabs** (including **Cadastre**, added) — Dashboard, Land, Cadastre, Projects, Financing, Construction, Units, Marketing, Delivery, Inspections, Payments, Documents, Calculators, Contingency Fund.
- **Inspections and Payments = read-only**; **Documents = text path** (no real file); create and delete only.
- **Six calculators** with Canadian **semi-annual compounding** and the **CMHC** scale (not insurable > 95%, Quebec 9% tax).
- **Automatic drawdowns**: 7 steps (10/15/25/15/20/10/5%), atomic and without duplicates.
- **Law 16 contingency fund**: co-ownerships, components, studies, logbook, **25-year projections with 3 scenarios**, certificates (art. 1069 C.c.Q.), AI advice + **`.md` report** (the only export).
- **Cadastre**: lot search, OpenStreetMap map, feasibility, constraints, attachment to a project (indicative data).
- **AI assistant** (developer only): **propose → confirm**, six confirmable actions including publication, kept history.
- **AI models**: Opus 4.8 (analyses), Sonnet 4.6 (chat, reports, suggestions), billed to prepaid credits (+30%).
- **Permissions**: no dedicated role — any tenant user can do everything, including publish; **consultation mode** (inactive subscription) blocks publication (403).
- **What the module does not do**: no rental management, no Centris, no exports (except the Law 16 `.md`), no automatic posting, areas in m².

---

**Documentation generated from the code** (July 7, 2026):
- `IMMO_REACT/frontend` (React application, base `/immo`) — Home, Listings, Detail page, Sign in, My listings, New project, Real estate management (14 tabs), Cadastre, AI Assistant panel pages.
- `ERP_REACT/backend/routers/immobilier.py` (developer area, ≈ 61 endpoints), `routers/immo_ai.py` (AI assistant, 5 endpoints, 9 tools), `routers/fonds_prevoyance.py` (Law 16, ≈ 31 endpoints).
- `IMMO_REACT/backend/routers/public.py` (storefront, 4 endpoints), `routers/publish.py` (publication, 5 endpoints), `immo_database.py` (shared `public.immo_listings` table), `immo_auth.py` (authentication and consultation mode).

**Related manuals**:
- Module 06 (Dossiers — to host actual files) — `06-ventes-dossiers.md`
- Module 08 (Projects — distinct from real estate projects) — `08-ventes-projets.md`
- Module 14 (Accounting — manual entries) — `14-operations-comptabilite.md`
- Module 24 (AI Assistant — shared AI credits) — `24-communication-assistant-ia.md`
- Module 30 (Configuration — subscription and consultation mode) — `30-configuration.md`
