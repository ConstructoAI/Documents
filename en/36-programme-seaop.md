# Module 36 — SEAOP (Quebec public tenders)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Intended audience**: this manual is addressed to the **users** of the SEAOP platform — the **project owners** (individuals, companies, and organizations that have work done) and the **contractors / construction workers** (who submit bids and make themselves known). It also covers, at the end of the document, the **Administration** functions (moderation and monitoring). It is not addressed to developers.
> **What SEAOP is**: "Système Électronique d'Appel d'Offres Public" (Electronic Public Tendering System). It is a **free matchmaking platform** between project owners and construction contractors in Quebec. A project owner publishes a tender, contractors submit bids, the client compares, exchanges messages, awards the contract, and then rates. Around this orbit three public **directories**, a **paid** estimate-request **service**, a community **Chat Room**, an **AI Assistant**, and an **Administration**.
> **Separate application**: SEAOP is **not** an internal tab of the Constructo AI ERP. It is a **standalone application** (basename `/seaop` — `App.tsx:37`), served under `app.constructoai.ca/seaop`, with its own sign-in, its own accounts, and its own session token (distinct from the ERP's). In production, it is co-hosted in the same service as the ERP (Render service `constructo-seaop`), but it remains a separate experience.
> **API prefix**: `/api/seaop/v1` (`seaop_config.py:12`). **14 routers** mounted (`seaop_api.py:351-398`), **77 endpoints** total.
> **Reference code (backend, `SEAOP_REACT/backend/`)**: `seaop_api.py` (429 lines, mounting) · `seaop_config.py` (175) · `seaop_auth.py` (390, JWT/session guards) · `seaop_database.py` (1,904, data access) · `seaop_models.py` (389, Pydantic validations) · `seaop_email.py` (408, 8 emails). Routers (`backend/routers/`): `auth.py` (529, 9 endpoints) · `leads.py` (489, 7) · `soumissions.py` (538, 7) · `messages.py` (255, 4) · `evaluations.py` (162, 3) · `notifications.py` (139, 4) · `chat_room.py` (392, 9) · `uploads.py` (240, 2) · `services.py` (975, 10 — estimation) · `admin.py` (153, 5) · `repertoire.py` (287, 3 — RBQ) · `professionnels.py` (535, 6) · `ouvriers.py` (641, 7) · `ai.py` (663, 1). Table schema: `modules/seaop/seaop_db_postgres.py` (721, `init_seaop_tables`).
> **Reference code (frontend, `SEAOP_REACT/frontend/src/`)**: `App.tsx` (105, routes) · pages `AccueilPage.tsx` (161), `NouveauProjetPage.tsx` (181), `EspaceEntrepreneurPage.tsx` (425), `LeadDetailPage.tsx` (754), `MesProjetsPage.tsx` (410), `EntrepreneurMessagesPage.tsx` (145), `ServiceEstimationPage.tsx` (1,292), `RepertoirePage.tsx` (292), `ProfessionnelsPage.tsx` (793), `OuvriersPage.tsx` (896), `AdminPage.tsx` (149), `NotificationsPage.tsx` (51), `ChatRoomPage.tsx` (10), `LoginPage.tsx` (102), `RegisterPage.tsx` (16). Layout: `components/layout/` — `Sidebar.tsx` (210), `TopBar.tsx` (215), `AppLayout.tsx` (42). Key components: `auth/LoginForm.tsx` (490), `auth/RegisterForm.tsx` (545), `auth/ProfileChoice.tsx` (120), `leads/LeadForm.tsx` (673), `leads/LeadCard.tsx` (210), `leads/LeadFilters.tsx` (274), `soumissions/SoumissionForm.tsx` (540), `soumissions/SoumissionCard.tsx` (318), `soumissions/SoumissionList.tsx` (148), `aiAssistant/SeaopAiPanel.tsx` (382), `chatRoom/ChatRoomPanel.tsx` (199), `messages/ChatThread.tsx` (237), `messages/ConversationList.tsx` (144), `evaluations/EvaluationForm.tsx` (86). Administration: `admin/DashboardStats.tsx` (227), `admin/EntrepreneurTable.tsx` (469), `admin/SoumissionTable.tsx` (193), `admin/ServiceTabs.tsx` (940), `admin/OuvriersAdminTab.tsx` (274), `admin/ProfessionnelsAdminTab.tsx` (243), `admin/RepertoireAdminCard.tsx` (141). Constants: `utils/constants.ts` (301). Translations: `i18n/locales/{fr,en}/*.json`.
> **PostgreSQL tables (shared database `public.seaop_*`, NO per-company multi-schema)**: `seaop_leads`, `seaop_entrepreneurs`, `seaop_soumissions`, `seaop_messages`, `seaop_evaluations`, `seaop_notifications`, `seaop_addenda`, `seaop_chat_room` (+ `_likes`, `_online`), `seaop_demandes_estimation`, `seaop_rbq_entrepreneurs`, `seaop_professionnels`, `seaop_ouvriers`, `seaop_ai_usage`. Platform administration accounts: `public.super_admins`.
> **Scope**: the **tendering platform is entirely free** — no payment, no billed credit, no Stripe gateway on the public side. The **estimate-request service** ($200 / $275 / $350) is a **paid** professional service, but billing happens **off-platform** (by email, on delivery of the quote): no payment is taken online in the code.

*A note on the terminology used in this manual:* "project owner" (or "client", `client` in the code) refers to the person or organization that **publishes** a tender and receives bids; "contractor" refers to the construction company that **submits** a bid; "tender" (or "project", or "lead" in the code) refers to the request published by a project owner; "bid" refers to a contractor's priced offer in response to a tender; "directory" refers to one of the three public lists of companies and workers; "endpoint" refers to an API endpoint; "administration" refers to the moderation and monitoring functions reserved for the platform team.

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

### 1.1 The platform's mission

SEAOP is an online marketplace that **connects** those who have work to be done with those who do it. The main journey is simple:

1. A **project owner** describes their project and **publishes** it as a tender (public, free, no account).
2. **Contractors** browse the tenders, filter by region and type of work, then **submit a bid** (price, timeframe, inclusions, conditions).
3. The project owner **compares** the bids received, **exchanges messages** with the contractors, then **awards** the contract to one of them.
4. Once the work is done, the project owner **rates** the contractor (1-to-5 rating + comment), which feeds their public reputation.

Around this journey orbit complementary services: **three public directories** (RBQ contractors, professionals, skilled workers), a **paid estimate-request service**, a community **Chat Room**, an **AI Assistant** (for signed-in users), and an **Administration** (moderation, RBQ-licence verification, estimate-request monitoring).

### 1.2 SEAOP is a SEPARATE application (not to be confused with the ERP)

This is the first point to understand. Unlike the other modules in this manual, **SEAOP is not an internal page of the Constructo AI ERP**. It is a **distinct application**, with:

- its own address (`app.constructoai.ca/seaop`);
- its own sign-in page and its own accounts;
- its own session token (the SEAOP token does **not** open the ERP for you, and the ERP token does **not** open SEAOP);
- its own sidebar and its own top bar.

> **Key point.** SEAOP's top bar permanently carries a **"Constructo AI sign-in"** link (`TopBar.tsx`) that **leaves** SEAOP to return to the ERP portal. It is the only bridge between the two worlds, and it is purely a page redirect (not a session share).

### 1.3 Two business models coexist: free and paid

SEAOP has **two clearly distinct money models** coexisting. Do not confuse them.

| | **Tendering platform** | **Estimate-request service** |
|---|---|---|
| Purpose | Publish a project, receive bids, award | Have a **professional estimate** produced by the Constructo AI team |
| Price | **Free** ($0), for everyone | **$200 / $275 / $350** depending on complexity |
| Account required | No, to publish; a contractor account to bid | No account: public form |
| Online payment | None | **None either** — billing **off-platform**, by email, **on delivery** of the quote |
| Where | "Post a project" / "Tenders" tabs | "Services" menu → "Estimate request" |

> **Key point.** Even the **paid** service takes **no online payment**: there is no Stripe, no card, no gateway in SEAOP's code. You submit a request, the team produces the estimate, and then bills you, on delivery, by email. As for the "credits" and "subscriptions" that sometimes appear on contractor accounts, these are columns **never billed or decremented** (see §4.6): the tendering platform stays free from end to end.

### 1.4 The four roles and the three ways to sign in

SEAOP distinguishes **four types of authenticated users** (`seaop_auth.py:264-269`) but shows only **three sign-in tabs** (`LoginForm.tsx:30-38`).

| Role (code) | Who | Sign-in tab | How you identify | Where you land |
|---|---|---|---|---|
| **Project owner** (`client`) | The one who publishes projects | **"Project owner"** | Two ways: (A) **magic link** received by email, or (B) **email + reference number** of their tender (`SEAOP-YYYYMMDD-XXXXXXXX`) | `/mes-projets` |
| **Contractor** (`entrepreneur`) | The company that bids | **"Contractor"** | **Email + password** (account to be created) | `/appels-offres` |
| **Administration** (`super_admin`) | The platform team | **"Administration"** | **Username + password** (individual accounts) | `/administration` |
| **Admin** (`admin`) | Internal fallback access | *(hidden from the interface)* | Shared password, server-side only | — |

> **The project owner has no password.** They do not "create an account" in the classic sense. Their identity rests on their **email** and their project's **reference number** (or on a **magic link** the platform sends them by email). This is intended: publishing a project must stay frictionless.
>
> **The `admin` role (shared password) no longer appears in the interface** (`LoginForm.tsx:16-18`). It remains server-side as a fallback access, but all administration now goes through **individual accounts** (the "Administration" tab). Do not document it as a normal access path.

### 1.5 The three directories (distinct from the roles above)

Do not confuse the **roles** (who sign in) with the **directories** (public listings that you consult). SEAOP publishes **three directories**:

| Directory | Menu | Content | Source | Contact details |
|---|---|---|---|---|
| **Contractors (RBQ)** | Services → Contractor directory | ~53,000 holders of an **active RBQ licence** | **Données Québec open data** (imported, no registration) | Public (phone / email from the licence) |
| **Professionals** | Services → Professionals directory | Technologists, architects, structural engineers | **Moderated self-registration** (validated before publication) | **Hidden** (revealed after contact, Law 25) |
| **Skilled workers** | Services → Skilled workers directory | Self-employed workers by trade | **Moderated self-registration** | **Hidden** (revealed after contact, Law 25) |

The **RBQ contractors** directory is thus an **imported official list**: you do not register in it. The **professionals** and **workers** directories, on the other hand, are fed by the interested parties themselves (self-registration), then **moderated** by administration before appearing to the public. The workers directory is the most recent of the three.

### 1.6 How to access SEAOP and find your way around

**Access.** Open `app.constructoai.ca/seaop`, or go through the "Public tenders / SEAOP" tile on the Constructo AI portal. The home page, the tenders, project posting, and the three directories are **public** (no sign-in required).

**The sidebar** (`Sidebar.tsx`) shows, under the "SEAOP / Tenders" brand:

- **Main items**: "Home", "Post a project", "Tenders", "My projects" *(project owner only)*, "My messages" *(contractor only)*, "Chat Room";
- **"Services" section** (collapsible): "Estimate request", "Contractor directory", "Professionals directory", "Skilled workers directory";
- **"Administration"** *(administration only)*.

**The top bar** (`TopBar.tsx`) contains: the hamburger button (on mobile), the **"Constructo AI sign-in"** back link, the page title, the **FR / EN language toggle**, the **notification bell** (red chip with the unread count), the light / dark **theme toggle**, and the user menu (name, email, "Sign out") or a "Sign in" button.

**The AI Assistant** (floating button + side panel) is mounted **on all pages** (`AppLayout.tsx:39`), but it is usable **only once signed in**.

### 1.7 Privacy, moderation, and contact-detail protection (Law 25)

SEAOP applies a **systematic protection of contact details**:

- On the **public tender list**, the project owner's **email and phone** are **hidden** from everyone except the project owner and administration (`leads.py`, `_strip_lead_contact`).
- On the **professionals** and **workers** directories, contact details are **never** shown upfront: they are revealed **only after** you send a **contact form** (name, email, message). This is not a bug; it is an anti-harvesting measure compliant with **Law 25** (Quebec's privacy law) (see §2.14 and §2.15).
- The **Chat Room** never shows participants' emails; the AI Assistant also hides third-party contact details.

The **professionals** and **workers** directories go through a **moderation queue**: a submitted listing stays "Pending" until administration **approves** or **rejects** it.

### 1.8 The "development" mode (DEV_MODE)

A `DEV_MODE` flag (variable `VITE_DEV_MODE` on the interface side, `SEAOP_DEV_MODE` on the server side) makes it possible to **lock the platform** during a preparation phase: in this mode, **only the "Administration" tab is visible** at sign-in, and a "Development mode" banner appears (`LoginPage.tsx`). In production, `DEV_MODE` must be `false`: the three public tabs then appear normally. If you only see the "Administration" tab, the platform is in development mode.

---

## 2. Interface

### 2.1 General layout (sidebar, top bar, AI Assistant)

All pages share the same layout (`AppLayout.tsx`): **sidebar** on the left (fixed on large screens, a drawer on mobile), **top bar** at the top, content in the center, and a **floating AI Assistant button** at the bottom right.

- **Language toggle.** The FR / EN button in the top bar translates the whole interface. **Known exception**: the "Estimate request" screen is written in hard-coded French (see §2.12) — its English version is incomplete.
- **Light / dark theme.** The theme toggle remembers your choice.
- **Notification bell.** It shows the number of unread notifications and updates periodically (polling every 60 seconds). A click opens the Notifications page (§2.16).
- **User menu.** Once signed in, your name and email are shown, with the "Sign out" option.

### 2.2 Home (`AccueilPage.tsx`) — public

The home page shows:

- a **main banner**: title "SEAOP", sub-title "Electronic Public Tendering System", and two action buttons — **"Post a project"** and **"View tenders"**;
- a **row of statistics** (4 cards): "Active projects" (number of visible tenders), "Contractors", "Bids", "Compliance rate" (these last three show a "New" label);
- a **"Recent projects"** section: up to 6 tender cards, with a "View all tenders" link. If nothing is published: "No project published yet.".

### 2.3 Post a tender (`NouveauProjetPage.tsx` + `LeadForm.tsx`) — public

This is the form by which a **project owner** publishes their project. No sign-in is required. Header: "Post a tender".

**Input convenience:**

- **Auto-saved draft.** The form saves your input in the browser. If you come back, a "Draft available" banner offers to **Restore** or to start fresh.
- **Live urgency indicator.** A badge (Low / Normal / High / Critical) is calculated in real time from your two dates (see §4.5).

**Form fields** (`leadForm.json`):

| Field | Required | Detail |
|---|:---:|---|
| **Project name** | Yes | Project title |
| **Email** | Yes | Used to contact you again and to sign in later |
| **Phone** | Yes | — |
| **Postal code** | Yes | Used for regional classification |
| **Project type** | Yes | 11 choices (see §4.4) |
| **Budget** | Yes | 6 ranges (see §4.4) |
| **Completion timeframe** | Yes | 6 choices (see §4.4) |
| **Bid deadline** | No | Defaults to 14 days out. Determines urgency. |
| **Desired start date** | No | Defaults to 30 days out. |
| **Description** | Yes | Minimum 20 characters (counter shown) |
| **Documents and plans** | No | Up to 5 files, 150 MB max per file. Formats: PDF, JPG, PNG, DOC, DOCX, XLS, XLSX |

**Collapsible section "Legal requirements and insurance"** (badge "Recommended") — to require certain guarantees from bidders:

- **RBQ licence required** (+ **Accepted RBQ categories**);
- **CNESST attestation required**;
- **Civil liability insurance required** (+ **Minimum amount** in $);
- **Bid bond required** (+ **Percentage** in %).

**Publishing.** The **"Publish the tender"** button saves the project and shows a **success screen**: the **reference number** (`SEAOP-YYYYMMDD-XXXXXXXX`, with one-click copy), the note "A confirmation email has been sent", and two buttons ("Back to home" / "View other tenders").

> **Keep your reference number.** Together with your email, it is what lets you **come back to review your bids** and sign in as a project owner (§3.2).

### 2.4 Tenders / Contractor workspace (`EspaceEntrepreneurPage.tsx`) — public + contractor

This page is **public** (everyone sees the tender list) but becomes the **contractor's workspace** once signed in.

**Header.**

- Anonymous visitor: title "Tenders" + "Post a project" button.
- Signed-in contractor: "Welcome, {company}" + three stat cards (Available projects, My bids, Average rating).

**Tabs** (visible **only** for the signed-in contractor):

- **"Tenders"** — the filterable list of open projects;
- **"My bids"** — the list of the bids you have submitted;
- **"My profile"** — your read-only record (Company, Contact, Email, Phone, RBQ no., Service areas, Project types, Certifications, Subscription, Status, Rating, Registration date).

**List filters** (`LeadFilters.tsx`): text search, **Project type**, **Region** (administrative regions of Quebec), **Sort by** (date / number of bids / urgency), a **"My service areas"** chip (to filter on your declared regions), and a **Reset** button. The list is shown as **tender cards** (§2.5).

### 2.5 Tender card (`LeadCard.tsx`)

Each project appears as a card: **badges** (Awarded / Closed / Cancelled status, or urgency level, + project type), the **title**, a **truncated description**, a grid (Budget, Timeframe, Postal code, Deadline), a **colour-coded days-remaining indicator**, a **bid counter**, and two buttons — **"View details"** (to the detail page, §2.6) and **"Bid"**. The "Bid" button is **hidden** if the project is already awarded, closed, or cancelled.

### 2.6 Tender detail (`LeadDetailPage.tsx`, route `/projet/:id`) — public

The detail page gathers everything about a project:

- back bar + status / urgency badges;
- **Header**: project type, reference number, title, publication date;
- Full **Description**;
- **Attached documents**: each item (Document / Plan / Photo) is downloadable;
- **Details**: Budget, Timeframe, Start date, Deadline (+ days remaining), Postal code; the **phone and email are visible only to the owner or administration**;
- **Compliance requirements**: RBQ / CNESST / Insurance / Bond badges with the required amounts;
- **Addenda**: the list of published addenda. An **"Add an addendum"** form (Title + Description) is reserved for the **project owner or administration**; publishing an addendum **notifies and emails all contractors** that have already bid;
- **Received bids**: visible **only once signed in** (bid cards, §2.8); for the project owner, the **"Award"** action appears here;
- a **fixed "Submit a proposal" button** (for the contractor) that opens the bid form.

### 2.7 Bid form (`SoumissionForm.tsx`) — contractor

Opened from a card or the detail page, this form lets the contractor **price their offer**.

**Built-in Quebec tax calculator.** You enter an amount, and the form shows a live summary of **pre-tax / GST (federal sales tax) 5% / QST (Quebec sales tax) 9.975% / tax-included total**. A **pre-tax / tax-included toggle** lets you indicate whether your amount is before or after tax. A **draft** is auto-saved per project.

**Fields** (`soumissionForm.json`):

| Field | Required | Detail |
|---|:---:|---|
| **Amount** | Yes | Pre-tax / tax-included toggle; capped (see §4.7) |
| **Completion time** | Yes | e.g. "6 weeks" |
| **Bid validity** | Yes | Defaults to "30 days" |
| **Work description** | Yes | Minimum 50 characters |
| **Inclusions** | No | What is included |
| **Exclusions** | No | What is not included |
| **Conditions** | No | Specific terms |
| **Attached documents** | No | Up to 3 files, 150 MB max per file |

**Bid bond** (optional section): an explanatory box, then a **"I include a bond"** checkbox that reveals a **Type** (Certified cheque / Bank letter of guarantee / Surety bond) and an **Amount** in $.

The **"Submit the proposal"** button files the bid. The project owner is then **notified** (in-app + email).

> **One bid per tender.** A contractor may submit **only one** proposal per tender (a second attempt is refused). To correct your offer, **edit** the existing bid — but only while it is in "sent" status and the deadline has not passed (see §3.4).

### 2.8 Bid card (`SoumissionCard.tsx`)

Each bid appears as a card: banner of the linked project (contractor view), company and contact name, **"Viewed" badge** (if the client has looked at it), status badge, compliance badges (**RBQ verified**, **RBQ no. …**, **Insured**, **Bonded**), a prominent **amount**, description, timeframe + validity, the contractor's **rating stars**, and the date.

For the **project owner**, two actions appear (with confirmation): **"Award"** and **"Decline"**. Possible statuses: sent → viewed → under review → accepted / declined.

### 2.9 My projects (`MesProjetsPage.tsx`, route `/mes-projets`) — project owner

This is the project owner's dashboard. Header "My tenders" + "Messages" button. Three stat cards (Total projects, With bids, In progress).

- **Project grid**: each card leads to that project's **bid detail** (via `SoumissionList`), with the **Accept / Decline / Award / Details** actions.
- **Sliding messaging panel**: list of conversations (`ConversationList`) + discussion thread (`ChatThread`).
- **Rating modal** (`EvaluationForm`): on an **accepted** bid, you can rate the contractor (1-to-5 stars + comment).
- Empty state: "You have not published a project yet".

### 2.10 My messages (`EntrepreneurMessagesPage.tsx`, route `/messages`) — contractor

The contractor's messaging space. Header "My messages". Two-column layout: the **list of conversations** (`ConversationList`) on the left, the **discussion thread** (`ChatThread`) on the right. Empty state: "No message yet…".

**The discussion thread (`ChatThread.tsx`)**: bubbles aligned by sender, sender name (Contractor / Client), relative timestamp, read receipts (one check = sent, two checks = read), input area (the **Enter** key sends), anti-double-send lock.

> **Who can write to whom.** A project owner may only write **to a contractor that has actually bid on their own project** (server-side anti-harassment rule, `messages.py`). There is no "open" messaging between strangers.

### 2.11 Chat Room (`ChatRoomPanel.tsx`, route `/chat-room`) — public (read), write if signed in

A **community chat room** that anyone can read. It shows: an "Online users" sidebar (with statistics), an orange banner of **pinned messages**, and the **message feed** (each message can be liked, replied to, and administration can delete). The room refreshes periodically (every 30 seconds). The input area (max 5,000 characters) appears only if you are signed in; otherwise: "Sign in to join the discussion." Emails are **never** shown in the room.

### 2.12 Estimate request (`ServiceEstimationPage.tsx`, route `/services/estimation`) — paid service, public

This is the **paid professional service**. Here you request a detailed estimate produced by the Constructo AI team. No account is required.

> **Technical particularity.** This screen is written in **hard-coded French** (it does not use the translation system). Its English version is therefore **incomplete**: the content stays in French even in EN mode.

**Pricing banner** — three tiers:

| Tier | Price | Note |
|---|---|---|
| **SIMPLE** | **$200** | — |
| **MOYEN** (Medium) | **$275** | "Most common" badge |
| **COMPLEXE** (Complex) | **$350** | — |

Three notes accompany the prices, including **"Billing only upon delivery of the completed quote"**. Contacts shown: `info@constructoai.ca` and 1 (936) 587-1141.

**4-step wizard:**

1. **Project** — **Trade** (21 choices, see §4.4), **Sector** (Residential / Commercial / Institutional / Industrial), **Project type**.
2. **Need** — **Description** (10 to 5,000 characters), **Documents** by drag-and-drop (images ≤ 5 MB, PDF ≤ 150 MB, up to 10 files combined, with a progress bar), **Urgency level** (Normal / Urgent), **Availability** (As soon as possible / Specific date).
3. **Details** (all optional) — Postal code, Area / dimensions, Location / address, Estimated budget, Desired timeframe.
4. **Contact details** — First name, Last name, Email, Phone, Company (optional) + summary.

**Success screen**: "Request received" + **reference number** + the promise of a **detailed estimate by email within 24 to 48 business hours**.

### 2.13 Contractor directory (`RepertoirePage.tsx`, route `/services/repertoire`) — public

Directory of **holders of an active RBQ (Régie du bâtiment du Québec, Quebec's building authority) licence**, fed by the **Données Québec open data** (Quebec's open-data portal). **No one registers here**: it is an imported official list.

**Filters**: Region, Trade (RBQ subcategory), Search (name / city / licence no., with a short 400 ms typing debounce). **Cards**: name, "RBQ licence no. …", "Restricted licence" badge where applicable, municipality / region, subcategories (up to 6), phone / email links. **Pagination**: 20 per page.

### 2.14 Professionals directory (`ProfessionnelsPage.tsx`, route `/services/professionnels`) — public

**Self-registered and moderated** directory of professional technologists, architects, and structural engineers. A banner reminds that listings are "validated by SEAOP before publication".

**Filters**: Type (Professional technologist / Architect / Structural engineer), Region (administrative regions of Quebec), Search. **Cards**: name, company, type badge, Member no., region, specialty, description, **"Contact"** and "Website" buttons.

- **"Contact" modal (protected, Law 25).** Contact details are **hidden** by default. They are revealed **only after** you send a form: **Your name**, **Your email**, **Phone** (optional), **Message** (≥ 10 characters). Once sent, the professional's phone and email are shown, and your message is relayed to them.
- **"Register" modal.** Type, Full name, Company, Member no., Email, Phone, Region, Municipality, Website, Specialty, Description. The listing is published **after validation** by administration.

### 2.15 Skilled workers directory (`OuvriersPage.tsx`, route `/services/ouvriers`) — public

The third and most recent directory: **self-employed workers by trade**. Same **moderated self-registration** and **protected contact details** logic as for professionals.

**Filters**: Trade (21 choices), Region, Search. **Cards**: name, company, trade badge, region + service areas, metadata (years of experience, CCQ (Commission de la construction du Québec) card, RBQ, Self-employed, rate $/h, availability), certifications, description, **Past work** (portfolio thumbnails), **"Contact"** buttons (protected, same mechanism as professionals) and "Website".

- **"Register" modal.** Full name, Trade, Company, Email, Phone, Website, Region, Municipality, Service areas, Years of experience, CCQ competency card, Min / max hourly rate, Availability, RBQ licence, NEQ (Quebec enterprise number), "I am self-employed" checkbox, Certifications, Description. **At least one of email or phone is required.** The portfolio accepts up to 4 images.

### 2.16 Notifications (`NotificationsPage.tsx`, route `/notifications`) — signed in

The list of your notifications, with an icon per type (bid, message, rating, status, alert, information). A click **marks as read**; a **"Mark all as read"** button clears the counter. Each row shows a relative time. The top-bar **bell** reflects the unread count (updated every 60 seconds).

### 2.17 Sign in and registration

**Sign-in screen (`LoginPage.tsx` + `LoginForm.tsx`).** It starts with a **profile choice** (`ProfileChoice.tsx`): two large cards — **"Contractor"** (leads to account creation) and **"Project owner"** (leads to posting a project) — plus a discreet **"Administration sign-in"** link. Then comes the **three-tab** sign-in form (§1.4). The screen also handles **magic links**: if you arrive with a link received by email (`?magic=…`), you are signed in automatically as a project owner.

**Contractor registration screen (`RegisterForm.tsx`).** In **two steps**:

- **Step 1 "Account"**: Company name, Contact name, Email, Phone, Password (**≥ 8 characters**), Confirmation. All required.
- **Step 2 "Profile" (optional)**: RBQ no. (format `XXXX-XXXX-XX`, with an RBQ verification link), RBQ categories, Insurance checkbox + Amount, Service areas, Project types, Certifications. A **"Skip and create my account"** button lets you skip this step.

### 2.18 Administration (`AdminPage.tsx`, route `/administration`)

Reserved for administration. Header "Administration". The **tabs**:

| Tab | Content | Access |
|---|---|---|
| **Overview** | 4 indicators (Projects, Contractors, Bids, Total revenue), "Monthly trend" chart, "Top 5 contractors" table (Bids / Accepted / Revenue / Rating) | admin + super_admin |
| **Contractors** | Table filterable by status (Active / Inactive / Suspended): Company, Contact, Email, RBQ (+ verified badge), Subscription, Credits, Status, Rating. Actions: **Suspend / Activate**, **Verify RBQ**, **Credits** | admin + super_admin |
| **Bids** | "Recent bids" (Project ref., Contractor, Amount, Status, Date) | admin + super_admin |
| **Services** | (a) **RBQ directory** card (counter, last import, **"Refresh the directory"** button — super_admin); (b) **Email / SMTP diagnostics** (status + test email); (c) **estimate requests** table with a detail modal (client info, attachments, internal notes, actions **Send back to client / Reject / Mark as sent**) | admin + super_admin |
| **Workers** | Moderation queue (Pending / Approved / Rejected), actions **Approve / Reject** | admin + super_admin |
| **Professionals** | Identical moderation queue | **super_admin only** |

> **A permission asymmetry to know.** The **"Professionals" tab is reserved for super_admin**; the **"Workers" tab is open to admin and super_admin**. Likewise, the **RBQ directory refresh** (Données Québec import) is reserved for super_admin.

### 2.19 AI Assistant (`SeaopAiPanel.tsx`)

A **floating button** opens a conversational-assistant **side panel**, available **on all pages but reserved for signed-in users**. Sub-title: "Tenders & QC compliance".

- **What it can do**: **read** your data (tenders, your bids, your profile, your projects, a contractor's reputation), **draft** documents (a tender for a client, a bid for a contractor), and **analyze images** (Vision: reading plans and photos).
- **Attachments**: up to 3 files (JPEG, PNG, WEBP, GIF, PDF).
- **Contextual suggestions** tailored to your role (contractor / client / general).
- **Privacy**: the assistant can access **only** your own data (the scope is **enforced by the server**, never chosen by the AI); third-party contact details are hidden except for the owner or administration.

> **The assistant never modifies the database.** It **reads** and **drafts**; it does not publish, bid, or award on your behalf. It is up to you to take its text and use it in the right form.

---

## 3. Step-by-step workflows

### 3.1 Publish a tender (project owner)

1. Without signing in, click **"Post a project"** (home or sidebar).
2. Fill in the required fields: Project name, Email, Phone, Postal code, Project type, Budget, Timeframe, Description (≥ 20 characters).
3. Adjust if needed the **Bid deadline** (default +14 days) and the **Start date** (+30 days). The deadline sets the **urgency level** shown to contractors.
4. Attach your **plans / documents** (up to 5 files).
5. If necessary, expand **"Legal requirements and insurance"** and check RBQ / CNESST / Insurance / Bond, with the desired amounts.
6. Click **"Publish the tender"**.
7. **Write down and keep the reference number** shown (`SEAOP-YYYYMMDD-XXXXXXXX`). A confirmation email is sent to you.

### 3.2 Come back to review your bids (project owner)

The project owner has no password: two paths are available.

**Path A — magic link:**
1. **"Project owner"** tab of the sign-in.
2. Request a **sign-in link** to your email.
3. Open the email received and click the link; you land directly on **"My projects"**. *(The link is valid for 30 minutes.)*

**Path B — email + reference number:**
1. **"Project owner"** tab.
2. Enter your **email** and your project's **reference number** (`SEAOP-YYYYMMDD-XXXXXXXX`).
3. You reach "My projects".

Once signed in, in "My projects", open a project to see its **received bids**, compare amounts, **exchange messages**, and then **award** (§3.5).

### 3.3 Create a contractor account

1. Sign-in page → **"Contractor"** card (or "Create an account" link).
2. **Step 1**: Company name, Contact name, Email, Phone, Password (**≥ 8 characters**), Confirmation. Continue.
3. **Step 2 (optional)**: enter your RBQ no., your categories, your insurance, your **service areas**, and your **project types** (these last two feed the "My service areas" filter and tender relevance). Or click **"Skip and create my account"**.
4. Then sign in through the **"Contractor"** tab (email + password). You land on **"Tenders"**.

### 3.4 Find a tender and bid (contractor)

1. **"Tenders"** tab. Filter by **Type**, **Region**, or enable **"My service areas"**. Sort by date, number of bids, or urgency.
2. Open a project (**"View details"**) to read the description, download the **plans**, and check the **compliance requirements** (RBQ / CNESST / insurance / bond).
3. Click **"Bid"** / **"Submit a proposal"**.
4. Enter the **Amount** (pre-tax / tax-included toggle — the calculator shows GST / QST / total), **Timeframe**, **Validity**, **Description** (≥ 50 characters), and if needed Inclusions / Exclusions / Conditions and a **bond**.
5. Attach up to 3 documents, then **"Submit the proposal"**. The client is notified.
6. **To correct** your offer: go back to **"My bids"** and edit it — possible **only** while it is "sent" and the **deadline** has not passed.

> You may submit **only one** bid per project. After the deadline, submitting and editing are **closed**.

### 3.5 Compare bids and award (project owner)

1. "My projects" → open the relevant project to see the list of **received bids**.
2. Compare amounts, timeframes, inclusions / exclusions, compliance badges (RBQ verified, Insured, Bonded), and each contractor's **rating**.
3. If needed, open **messaging** to ask questions.
4. Click **"Award"** on the selected bid, then confirm.

> **What awarding triggers automatically.** The chosen bid becomes **"accepted"**, the project becomes **"awarded"**, it **stops accepting new bids**, and **all other bids are declined** — each contractor (winner and losers alike) is **notified** (in-app + email).

### 3.6 Exchange messages

- **Contractor**: **"My messages"** tab → choose a conversation → write to the project owner (Enter sends).
- **Project owner**: "My projects" → **"Messages"** button (sliding panel) → choose the conversation with the desired contractor.

Reminder: a client can only write to a contractor **that has bid on their project**. When a thread is opened, received messages are marked as read (the bell counter goes down).

### 3.7 Rate a contractor (project owner)

1. "My projects" → the project whose bid is **accepted**.
2. On that bid, open the **rating modal**.
3. Give a **1-to-5 star rating** + a **comment**, then confirm.

> **You can only rate an ACCEPTED bid.** This is a safeguard: you cannot sabotage the reputation of a contractor you did not select. A rating is **unique** per bid and per rater (a new rating replaces the previous one). The contractor's average is recalculated automatically.

### 3.8 Publish an addendum (project owner)

1. Open your project's **detail** (`/projet/:id`).
2. **Addenda** section → **"Add an addendum"**: Title + Description.
3. Publish. All contractors that have **already bid** receive a **notification and an email**. Previous addenda remain listed and numbered.

### 3.9 Request a professional estimate (paid service)

1. **Services → "Estimate request"** menu.
2. Choose the matching tier (SIMPLE $200 / MOYEN $275 / COMPLEXE $350), for reference.
3. **Project** step: Trade, Sector, Project type.
4. **Need** step: Description (≥ 10 characters), **documents** (drag-and-drop), urgency, availability.
5. **Details** step (optional): postal code, area, address, budget, timeframe.
6. **Contact details** step: First name, Last name, Email, Phone, Company.
7. Submit. You receive a **reference number** (`EST-…`) and the promise of a reply **by email within 24 to 48 business hours**.

> **No payment at this stage.** The team produces the estimate, then **bills you on delivery**, off-platform.

### 3.10 Register in a directory (professional or worker)

1. **Services** menu → **"Professionals directory"** or **"Skilled workers directory"**.
2. Open the **"Register"** modal and fill in your listing (for a worker: at least one of email **or** phone; a portfolio of 4 images maximum).
3. Submit. Your listing goes out **"Pending"**: it appears to the public **only after approval** by administration.

### 3.11 Get connected through a directory (Law 25 protection)

1. Open the directory (professionals or workers) and find a listing.
2. Click **"Contact"**. The contact details are **hidden**.
3. Fill in the **contact form** (your name, your email, your message ≥ 10 characters).
4. After sending, the **contact details are shown** and your message is **relayed** to the professional / worker.

> This detour through a form is **normal and intended** (anti-email-harvesting, Law 25 compliance). It is not a malfunction.

### 3.12 (Administration) moderate, verify, process

1. **"Workers" / "Professionals" tab**: browse the queue (Pending / Approved / Rejected), open a listing (contact details visible here), then **Approve** or **Reject**.
2. **"Contractors" tab**: filter by status, **Suspend / Activate** an account, **Verify** its RBQ licence (adds the "RBQ verified" badge and notifies), or adjust its **Credits**.
3. **"Services" tab**: refresh the **RBQ directory** (super_admin), test the **email / SMTP** configuration, and process the **estimate requests** (open the modal, download the attachments, change the status: Send back to client / Reject / Mark as sent).

---

## 4. Reference

### 4.1 SEAOP screens

| Screen | File | Access | Role |
|---|---|---|---|
| Home | `AccueilPage.tsx` | Public | Showcase + recent projects |
| Post a tender | `NouveauProjetPage.tsx` / `LeadForm.tsx` | Public | Create a project |
| Tenders / Contractor workspace | `EspaceEntrepreneurPage.tsx` | Public + contractor | List, My bids, My profile |
| Project detail | `LeadDetailPage.tsx` | Public | Detail + addenda + bids |
| My projects | `MesProjetsPage.tsx` | Project owner | Tracking, messaging, awarding, rating |
| My messages | `EntrepreneurMessagesPage.tsx` | Contractor | Messaging |
| Chat Room | `ChatRoomPanel.tsx` | Public (read) | Community room |
| Estimate request | `ServiceEstimationPage.tsx` | Public | Paid service (4 steps) |
| Contractor directory | `RepertoirePage.tsx` | Public | RBQ directory |
| Professionals directory | `ProfessionnelsPage.tsx` | Public | Moderated directory |
| Workers directory | `OuvriersPage.tsx` | Public | Moderated directory |
| Notifications | `NotificationsPage.tsx` | Signed in | Alerts |
| Sign in / Registration | `LoginPage.tsx` / `RegisterForm.tsx` | Public | 3 tabs, contractor registration |
| Administration | `AdminPage.tsx` | admin / super_admin | 6 tabs |
| AI Assistant | `SeaopAiPanel.tsx` | Signed in | Read + draft + Vision |

### 4.2 API endpoints (77 total)

All prefixed with `/api/seaop/v1`.

**Authentication (`auth.py`, 9):**

| Method | Path | Access |
|---|---|---|
| POST | `/auth/entrepreneur/login` | Public |
| POST | `/auth/entrepreneur/register` | Public |
| POST | `/auth/client/login` | Public (email + reference no.) |
| POST | `/auth/client/request-link` | Public (magic link) |
| POST | `/auth/client/verify-link` | Public (magic-token exchange) |
| POST | `/auth/admin/login` | Public (shared password, hidden from the UI) |
| POST | `/auth/super-admin/login` | Public (individual accounts) |
| POST | `/auth/logout` | Signed in |
| GET | `/auth/me` | Signed in |

**Tenders (`leads.py`, 7):**

| Method | Path | Access |
|---|---|---|
| GET | `/leads` | Public (contacts hidden) |
| GET | `/leads/mes-projets` | Project owner |
| GET | `/leads/{id}` | Public |
| POST | `/leads` | Public (5 / IP / h) |
| PUT | `/leads/{id}` | Owner or admin |
| GET | `/leads/{id}/addenda` | Public |
| POST | `/leads/{id}/addenda` | Owner or admin |

**Bids (`soumissions.py`, 7):**

| Method | Path | Access |
|---|---|---|
| POST | `/soumissions` | Contractor |
| GET | `/soumissions/lead/{id}` | Owner client or admin |
| GET | `/soumissions/mes-soumissions` | Contractor |
| GET | `/soumissions/{id}` | Participant or admin |
| PUT | `/soumissions/{id}` | Issuing contractor (if "sent") |
| PUT | `/soumissions/{id}/statut` | Owner client or admin |
| POST | `/soumissions/{id}/attribuer` | Owner client or admin |

**Messages (`messages.py`, 4) · Ratings (`evaluations.py`, 3) · Notifications (`notifications.py`, 4):**

| Method | Path | Access |
|---|---|---|
| POST | `/messages` | Participant |
| GET | `/messages/conversations` | Signed in |
| GET | `/messages/conversation/{lead}/{entrepreneur}` | Participant |
| PUT | `/messages/mark-read/{lead}/{entrepreneur}` | Participant |
| POST | `/evaluations` | Client (accepted bid only) |
| GET | `/evaluations/entrepreneur/{id}` | Public |
| GET | `/evaluations/soumission/{id}` | Participant / admin |
| GET | `/notifications` | Signed in |
| GET | `/notifications/count` | Signed in |
| PUT | `/notifications/read-all` | Signed in |
| PUT | `/notifications/{id}/read` | Signed in |

**Chat Room (`chat_room.py`, 9):** `GET /chat-room/messages` (public), `POST /chat-room/messages`, `PUT /chat-room/messages/{id}` (author), `DELETE /chat-room/messages/{id}` (author or admin), `POST /chat-room/messages/{id}/like`, `PUT /chat-room/messages/{id}/pin` (admin), `GET /chat-room/online` (public), `POST /chat-room/heartbeat`, `GET /chat-room/stats` (public).

**Uploads (`uploads.py`, 2):** `POST /uploads` (public, 150 MB, 30 / IP / h), `POST /uploads/multi` (public, ≤ 5 files).

**Estimate request (`services.py`, 10):** `POST /services/estimation` (public, 5 / IP / h), `GET /services/estimation/meta` (public), `POST /services/estimation/plans` (public, PDF 150 MB), `GET /services/estimation/admin`, `GET /services/estimation/admin/{id}`, `PUT /services/estimation/admin/{id}`, `GET /services/estimation/admin/email-status`, `POST /services/estimation/admin/test-email`, `GET /services/estimation/admin/{id}/plans/{plan_id}`, `POST /services/estimation/admin/{id}/resend-client-email` (the last 7: admin / super_admin).

**Administration (`admin.py`, 5):** `GET /admin/stats`, `GET /admin/entrepreneurs`, `PUT /admin/entrepreneurs/{id}`, `GET /admin/soumissions`, `PUT /admin/entrepreneurs/{id}/verify-rbq` (admin / super_admin).

**RBQ directory (`repertoire.py`, 3):** `GET /services/repertoire` (public), `GET /services/repertoire/meta` (public), `POST /services/repertoire/admin/refresh` (**super_admin**).

**Professionals (`professionnels.py`, 6):** `POST /professionnels/inscription` (public, 5 / IP / h), `GET /professionnels` (public, no contact details), `GET /professionnels/meta` (public), `POST /professionnels/{id}/contact` (public, protected, 10 / IP / h), `GET /professionnels/admin/queue` (**super_admin**), `POST /professionnels/admin/{id}/moderate` (**super_admin**).

**Workers (`ouvriers.py`, 7):** `POST /ouvriers/inscription` (public, 5 / IP / h), `GET /ouvriers` (public, no contact details), `GET /ouvriers/meta` (public), `GET /ouvriers/{id}` (public, with portfolio), `POST /ouvriers/{id}/contact` (public, protected, 10 / IP / h), `GET /ouvriers/admin/queue` (admin / super_admin), `POST /ouvriers/admin/{id}/moderate` (admin / super_admin).

**AI Assistant (`ai.py`, 1):** `POST /ai/chat` (signed in).

### 4.3 Statuses

The values below are the internal status codes stored in the database (kept verbatim).

| Domain | Values |
|---|---|
| **Tender** | nouveau · en_cours · ferme · attribue · annule |
| **Bid** | envoyee · vue · en_evaluation · acceptee · refusee |
| **Estimate request** | nouvelle · en_analyse · estimation_envoyee · refusee · archivee |
| **Moderation (professionals / workers)** | EN_ATTENTE · APPROUVE · REJETE |
| **Contractor account** | actif · inactif · suspendu (+ bloqué / banni / rejeté: token refused) |
| **Bond (bid)** | Certified cheque · Bank letter of guarantee · Surety bond |

### 4.4 Enumerations (choice lists)

**Project types (11)** — `Construction work`, `Public building renovation`, `Road infrastructure`, `Urban development`, `IT systems`, `Professional services`, `Supplies and equipment`, `Maintenance services`, `Engineering works`, `Specialized consulting`, `Other`.

**Budget ranges (6)** — `Under $25,000`, `$25,000 – $100,000`, `$100,000 – $500,000`, `$500,000 – $1,000,000`, `Over $1,000,000`, `To be determined by bids`.

**Completion timeframes (6)** — `Urgent (less than 1 month)`, `Short term (1–3 months)`, `Medium term (3–6 months)`, `Long term (6–12 months)`, `Multi-year (more than 12 months)`, `Per project schedule`.

**Trades for estimation and workers (21)** — General contractor · Electrical · Plumbing · HVAC (heating / ventilation / air conditioning) · Roofing · Framing / Carpentry · Interior finishing · Painting · Flooring (ceramic, wood, vinyl) · Windows / Doors · Insulation · Drywall / Taping · Concrete / Foundation · Excavation / Earthwork · Demolition · Masonry / Stone · Exterior cladding · Paving / Asphalt · Landscaping · Pool / Spa · Other.

**Sectors (4)** — Residential · Commercial · Institutional · Industrial.

**Regions** — the filters offer the administrative regions of Quebec (the tenders filter lists 18 entries; the professionals and workers directory filters list 17). Regional classification is based on the **postal code**.

### 4.5 Automatic urgency calculation

A tender's urgency level is **calculated** from the number of days remaining before the **bid deadline** (`leads.py`):

| Days remaining | Urgency |
|---|---|
| ≤ 3 | **Critical** |
| ≤ 7 | **High** |
| ≤ 14 | **Normal** |
| > 14 | **Low** |

### 4.6 Money, taxes, and free-of-charge

- **Tendering platform = free.** No payment, no gateway. The `abonnement` (default "gratuit") and `credits_restants` (default 5) columns on contractor accounts exist but are **never decremented or required**: submitting a bid costs nothing and consumes no credit.
- **Estimate service = paid, offline.** Tiers **$200 / $275 / $350**, **billed on delivery** of the quote, by email. **No online payment** in the code.
- **AI Assistant = free for the user.** Its cost (Claude model) is **logged for internal audit** but **never billed**.
- **Tax calculator (bid)**: **GST 5%** and **QST 9.975%** applied to the amount, with a pre-tax / GST / QST / total summary. It is an **input-assist tool** for the contractor (the retained amount stays the one they enter).

### 4.7 Limits and bounds

| Item | Limit |
|---|---|
| Contractor password | **≥ 8 characters** |
| Tender description | **≥ 20 characters** |
| Bid description | **≥ 50 characters** |
| Estimate description | 10 to 5,000 characters |
| Contact message (directories) | **≥ 10 characters** |
| Chat Room message | ≤ 5,000 characters |
| Bid amount | > 0 and ≤ $1,000,000,000 |
| Documents — tender | 5 files (PDF / images / Office) |
| Documents — bid | 3 files |
| Documents — estimate | 10 files combined (images ≤ 5 MB, PDF ≤ 150 MB) |
| Generic upload (`/uploads`) | 150 MB per file, magic bytes verified |
| Worker portfolio | 4 images |
| Session — contractor / super_admin | 7 days / 24 hours |
| Session — project owner | 24 hours; magic link valid 30 minutes |
| Session — admin | 2 hours |
| AI Assistant | 20 messages per 10 minutes; 3 attachments |

### 4.8 Reference numbers

- **Tender**: `SEAOP-YYYYMMDD-XXXXXXXX` (8 hexadecimal characters). Keep it: it is used for the project owner's sign-in.
- **Estimate request**: `EST-YYYYMMDD-XXXXXXXX`.

These numbers are generated in a **unique and non-sequential** way (no duplicate possible, no leak of the total number of projects).

### 4.9 Pitfalls and things to know

- **The project owner has no password**: sign in by magic link **or** email + reference number. Losing both = no longer able to sign back in (republish if needed).
- **One bid per project**, editable only if "sent" and before the deadline.
- **You can only rate an accepted bid**; a rating replaces the previous one.
- **Contact details hidden** on public lists and the professionals / workers directories: this is intended (Law 25).
- **The "Estimate request" screen stays in French** even in English (hard-coded text).
- **Admin "Professionals" tab reserved for super_admin**; "Workers" open to admin as well.
- **The bid amount is stored in single precision (REAL / float32)**: on very large amounts, a minimal rounding is theoretically possible in display or sorting. No practical effect at usual amounts.
- **Unused legacy tables** in the application (`seaop_attributions`, `seaop_estimations`, old technologist / architecture / engineer requests): awarding now goes through the project and bid statuses; a **single** estimate service remains.

---

## 5. Integrations and FAQ

### 5.1 Links with the rest of Constructo AI

- **Constructo AI portal (ERP).** SEAOP is **separate** from the ERP. The only bridge is the **"Constructo AI sign-in"** link in the top bar, which **leaves** SEAOP. Accounts and tokens are **not** shared.
- **Données Québec (RBQ licences).** The contractor directory is fed by an **import** of the public RBQ-licence dataset (refreshed by the super_admin).
- **Emails (SMTP).** SEAOP sends 8 types of emails (tender confirmation, new bid, status change, message, addendum, estimate on the admin side and the client side, magic link). The Administration → Services tab lets you **test** the configuration.
- **AI Assistant (Claude).** SEAOP's assistant is **specific to SEAOP**; it is not the ERP's AI Assistant (module 24). It reads only SEAOP data, with a scope enforced by the server.
- **No Stripe, QuickBooks, or accounting integration.** SEAOP collects nothing and syncs with no accounting software.

### 5.2 Frequently asked questions

**Is SEAOP the same thing as the Constructo AI ERP?**
No. It is a **separate application** (address `/seaop`), with its own accounts and its own session. The "Constructo AI sign-in" link is only used to return to the ERP.

**How much does it cost to publish a tender?**
**Nothing.** The tendering platform is entirely free, for project owners and contractors alike.

**So what is paid?**
Only the **estimate-request service** ($200 / $275 / $350), and even then: **nothing is paid online**. You are billed **on delivery** of the quote, by email.

**I am a project owner: how do I sign back in?**
By a **magic link** sent to your email, **or** with your **email + your project's reference number** (`SEAOP-…`). There is no password for this role.

**I lost my reference number.**
Request a **magic link** to your email (Path A). If you no longer have access to the email either, you will have to republish the project.

**Can a contractor bid twice on the same project?**
No. One bid per project. You can **edit** it while it is "sent" and the deadline has not passed.

**Why can't I see a professional's or a worker's contact details?**
It is **normal**: they are revealed after you send the **contact form** (Law 25 protection). Your message is relayed to them and their contact details are shown.

**How does my company get into the contractor directory?**
This directory is an **official list imported** from active RBQ licences: you do not register in it. However, you can **register** in the **professionals** or **workers** directory (listing moderated before publication).

**Why doesn't my directory listing appear right away?**
It goes out **"Pending"** and is published only after **approval** by administration.

**Can the AI Assistant publish a project or bid on my behalf?**
No. It **reads** your data and **drafts** documents, but never modifies the database. You take its text into the right form.

**Do I pay to use the AI Assistant?**
No. Its use is free (limited to 20 messages per 10-minute window to prevent abuse).

**Does the estimate service take my card?**
No. No online payment. Billing on delivery, off-platform.

**What happens when I award a contract?**
The chosen bid becomes "accepted", the project becomes "awarded", it **stops accepting bids**, and **all the others are declined** — everyone is notified.

**Can I rate a contractor I did not select?**
No. You can rate **only** the **accepted** bid, to protect reputations.

**The estimate screen stays in French even when I switch to English. A bug?**
No, a known limitation: this screen is written in hard-coded French. Its English translation is incomplete.

**I only see the "Administration" tab on the sign-in page.**
The platform is in **development mode** (`DEV_MODE`). In production, the three public tabs appear.

### 5.3 Common troubleshooting

| Symptom | What to check |
|---|---|
| "Project owner" sign-in refused | Check the **email + reference number pair**, or use the **magic link**. The magic link expires after 30 minutes. |
| "Bid" button missing | The project is **awarded, closed, or cancelled** — submissions are closed. |
| Cannot edit a bid | It is no longer "sent" (already viewed / under review / awarded) **or** the **deadline** has passed. |
| "You have already bid" | One bid per project: **edit** the existing one. |
| Contact details not found in a directory | Normal (Law 25): click **"Contact"** and send the form. |
| My directory listing is not visible | It is **"Pending"** moderation. |
| Cannot rate a contractor | The bid is not **accepted**. |
| No email notifications | Check your spam; on the admin side, test **SMTP** (Administration → Services). |
| Only the "Administration" tab at sign-in | **Development mode** active — wait for the production release. |
| Estimate interface in French in EN mode | Known limitation (hard-coded text on this screen). |

---

## 6. Summary

- **SEAOP is a free matchmaking platform** between project owners and construction contractors in Quebec. A project owner publishes a tender, contractors bid, the client compares, exchanges, **awards**, and then **rates**.
- **It is a separate application** from the Constructo AI ERP (address `/seaop`, its own accounts and session). The "Constructo AI sign-in" link only **leaves** SEAOP.
- **Four roles** (project owner, contractor, admin, super_admin) but **three sign-in tabs**: the `admin` role (shared password) is **hidden**. The **project owner has no password** (magic link or email + reference number).
- **Three directories distinct from the roles**: **RBQ** contractors (imported, not registerable), **professionals** and **workers** (**moderated** self-registration). On the last two, **contact details are hidden** and are revealed after a contact form (**Law 25**).
- **Two business models**: the tendering platform is **free**; the **estimate service** is **paid** ($200 / $275 / $350) but **with no online payment** — billed **on delivery**, by email. Contractor "credits" and "subscriptions" are **never** billed.
- **Key automatic behaviors**: urgency calculated from the deadline; awarding that **declines all other bids** and notifies everyone; rating **reserved for an accepted bid**; **one** bid per project.
- **Administration** in 6 tabs (Overview, Contractors, Bids, Services, Workers, Professionals); "Professionals" and the RBQ refresh are **reserved for super_admin**.
- **AI Assistant** (signed-in users): **read + draft + Vision**, scope enforced by the server, **never** writes to the database, **never** billed.
- **Technical volume**: **14 routers**, **77 endpoints**, `public.seaop_*` tables in a shared database (no multi-schema).

---

*Verified source files:* backend (`SEAOP_REACT/backend/`) — `seaop_api.py` (429), `seaop_config.py` (175), `seaop_auth.py` (390), `seaop_database.py` (1,904), `seaop_models.py` (389), `seaop_email.py` (408); routers `routers/` — `auth.py` (529, 9 endpoints), `leads.py` (489, 7), `soumissions.py` (538, 7), `messages.py` (255, 4), `evaluations.py` (162, 3), `notifications.py` (139, 4), `chat_room.py` (392, 9), `uploads.py` (240, 2), `services.py` (975, 10), `admin.py` (153, 5), `repertoire.py` (287, 3), `professionnels.py` (535, 6), `ouvriers.py` (641, 7), `ai.py` (663, 1); DDL `modules/seaop/seaop_db_postgres.py` (721). Frontend (`SEAOP_REACT/frontend/src/`) — `App.tsx` (105), pages (`AccueilPage` 161, `NouveauProjetPage` 181, `EspaceEntrepreneurPage` 425, `LeadDetailPage` 754, `MesProjetsPage` 410, `EntrepreneurMessagesPage` 145, `ServiceEstimationPage` 1,292, `RepertoirePage` 292, `ProfessionnelsPage` 793, `OuvriersPage` 896, `AdminPage` 149, `NotificationsPage` 51, `LoginPage` 102), components (`Sidebar` 210, `TopBar` 215, `LoginForm` 490, `RegisterForm` 545, `ProfileChoice` 120, `LeadForm` 673, `LeadCard` 210, `LeadFilters` 274, `SoumissionForm` 540, `SoumissionCard` 318, `SeaopAiPanel` 382, `ChatRoomPanel` 199, `ChatThread` 237, `EvaluationForm` 86; administration `DashboardStats` 227, `EntrepreneurTable` 469, `ServiceTabs` 940, `OuvriersAdminTab` 274, `ProfessionnelsAdminTab` 243, `RepertoireAdminCard` 141, `SoumissionTable` 193), constants `utils/constants.ts` (301). Production mounting: `ERP_REACT/backend/erp_api.py` (routers under `/api/seaop/v1`, SPA under `/seaop`).

*Related manuals:* `24-communication-assistant-ia.md` (the ERP's AI Assistant, not to be confused with SEAOP's), `07-ventes-soumissions.md` (the ERP's internal quotes), `35-programme-portail-b2b.md` (the other standalone application served by the ERP), `16-terrain-conformite.md` (RBQ, CNESST, bonding — Quebec compliance context).

*Constructo AI ERP Manual — Module 36 "SEAOP (Quebec public tenders)" — v1.0 verified — 2026-07.*
