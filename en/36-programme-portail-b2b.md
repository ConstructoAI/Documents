# Module 36 — B2B / B2C Portal (clients and suppliers)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Intended audience**: this manual is addressed to the **client** of a supplier (the contractor or company that buys, requests quotes, and tracks its files), not to the supplier that operates the ERP. The supplier's **back-office** (access approval, catalogue management, quote pricing, order status changes) lives elsewhere, in module **06 — Sales (B2B/B2C tab)**, reserved for administrators.
> **Frontend route**: `/b2b-portal` (standalone application served **by** the ERP). Public screens `/b2b-portal/login` and `/b2b-portal/register`; protected screens `/b2b-portal/dashboard`, `/catalogue`, `/panier`, `/commandes`, `/suivi`, `/contrats`, `/demandes`, `/messages` (`App.tsx:173-184`). The access tile, on the ERP login page, is called **"B2B / C2B — Client portal"** (`LoginPage.tsx:119-128`).
> **API prefix**: `/api/erp/v1`. Three routers serve the B2B world, but **only one concerns this manual**: the client portal `/b2b-portal` (the other two, `/b2b` and `/b2b/ai`, are on the supplier side — see §1.2).
> **Reference code (backend)**: `backend/routers/b2b_portal.py` (1,321 lines — **22 endpoints** on the client side) · `backend/routers/auth.py` (4 B2B endpoints: supplier lookup, login, registration, profile) · access guards in `backend/erp_auth.py` (`get_current_b2b_client`, `erp_auth.py:581`). The `b2b_*` tables are created and repaired on demand by `_ensure_b2b_tables` (`backend/routers/b2b.py:227`).
> **Reference code (frontend)**: `frontend/src/pages/b2b-portal/` — `B2bLoginPage.tsx` (208 lines), `B2bRegisterPage.tsx` (391 lines), `B2bDashboardPage.tsx` (97 lines), `B2bCataloguePage.tsx` (134 lines), `B2bPanierPage.tsx` (154 lines), `B2bCommandesPage.tsx` (119 lines), `B2bSuiviPage.tsx` (230 lines), `B2bContratsPage.tsx` (99 lines), `B2bDemandesPage.tsx` (172 lines), `B2bMessagesPage.tsx` (152 lines). Layout: `components/layout/B2bPortalLayout.tsx` (183 lines) + `B2bProtectedRoute.tsx` (25 lines). API clients: `api/b2b-portal.ts` (277 lines), `api/b2b-portal-auth.ts` (175 lines). State stores: `store/useB2bPortalStore.ts` (252 lines), `store/useB2bAuthStore.ts` (98 lines). Translations: `i18n/locales/{fr,en}/b2b.json` (`portal` section) + `layout.json` (`b2bPortalLayout` section) + `auth.json` (`b2b` section).
> **PostgreSQL tables (per supplier company)**: `b2b_clients`, `b2b_client_users` (sign-in accounts), `b2b_paniers`, `b2b_panier_lignes`, `b2b_commandes`, `b2b_commande_lignes`, `b2b_favoris`, `b2b_demandes`, `b2b_soumissions`, `b2b_contrats`, `b2b_messages`, `b2b_notifications`. Shared tables read by the portal: `produits` and `mouvements_stock` (Store module), `devis` and `projects` (Tracking), `companies` (CRM client record).
> **Scope**: the portal is an **online space that the supplier opens to its clients**. Once their access is approved, a client can **browse the supplier's catalogue**, fill a **cart**, place an **order** (with GST and QST), **track their orders**, **track (read-only) their real quotes and projects** in the supplier's ERP, view their **contracts**, submit **quote requests**, and **exchange messages** with the supplier. The portal **does not replace** the ERP: it is its client-facing storefront. It **collects no online payment**, contains **no AI assistant**, and lets the client modify **only** their cart, their orders (at creation), their requests, and their messages.

*A note on the terminology used in this manual:* "supplier" (or "company") refers to the company that operates the ERP and has granted you access; "client" refers to you, the portal user; "endpoint" refers to an API endpoint; "quote" (or "bid") refers to a price prepared by the supplier; "request" refers to a **quote request** that you send to the supplier; "order" refers to a purchase of catalogue products.

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

### 1.1 Purpose of the portal

The **B2B / B2C Portal** is the online space that a supplier (a company that is itself a Constructo AI customer) makes available to **its own clients**. Concretely, once your access is approved, it lets you:

- **Browse the supplier's product catalogue** (name, code, description, category, price, unit) and mark your **favorites**;
- **Fill a cart** and **place an order** online, with automatic calculation of **GST (Goods and Services Tax, 5%)** and **QST (Quebec Sales Tax, 9.975%)**;
- **Track your orders** (number, date, status, line details);
- **Track, read-only, your real quotes and projects** as they exist in the supplier's ERP (provided the supplier has linked your account to your record — see §1.5);
- **View your contracts** (amount, amount paid, progress);
- **Submit quote requests** (describe a project and receive offers from the supplier);
- **Exchange messages** with the supplier, one conversation thread per request.

The portal is an **application separate** from the ERP. You do not need an ERP account: you sign in with a **client account** that the supplier approves. Your session, your data, and your navigation are **entirely separate** from those of the ERP.

### 1.2 Two separate worlds: client portal ↔ supplier back-office

This is the single most important point to understand. Around the same "B2B" acronym, **two completely distinct applications** coexist:

| | **Client portal** (this manual) | **Supplier back-office** (module 06) |
|---|---|---|
| Who uses it | **You**, the client | The supplier (ERP administrator) |
| Address | `/b2b-portal` | `/ventes?tab=b2b` (B2B/B2C tab) |
| Account type | Client account (`b2b_client`) | ERP administrator account |
| What you do there | Order, request, track, chat | Approve access, manage the catalogue, **price** quotes, create contracts, **advance** orders, reply to messages |
| AI assistant | **None** | Yes ("AI Assistant — B2B Management", read-only, administrator-only) |

> **Remember:** there is **no AI assistant in the client portal**. The "B2B Management" assistant that the supplier may use lives in the ERP back-office (module 06) and is **never** accessible from your portal. No button, no portal feature consumes any artificial-intelligence credits.

Many portal actions are **deliberately read-only** on your side, because the decision belongs to the supplier: it is the supplier who **accepts or declines** a quote, who **advances or cancels** an order, who **drafts** a contract. You initiate (a request, an order, a message); the supplier decides.

### 1.3 How to access it

Three paths lead to the portal:

| Path | Detail |
|------|--------|
| **Tile on the ERP login page** | On `app.constructoai.ca`, the login page shows a **"B2B / C2B — Client portal"** tile (`LoginPage.tsx:119-128`) that opens `/b2b-portal/login`. |
| **Direct address** | Type `app.constructoai.ca/b2b-portal` directly. The `/b2b-portal` root redirects you to the dashboard if you are signed in, otherwise to the login page. |
| **Link received from the supplier** | The supplier can simply send you the portal address and the **contact email** to use to identify it (see §3.1). |

From within the portal, the top bar always carries a **"Constructo AI Login"** link (back arrow) that **leaves** the portal and returns to the ERP root (`B2bPortalLayout.tsx:87-94`). It is the reverse path: from the portal to the ERP.

### 1.4 Roles, permissions, and security

**One account type, one access level.** Unlike the ERP (which distinguishes administrators, accountants, employees), the portal has **no internal role**. Every active, approved client account has exactly the same access. There is no "portal administrator" on the client side.

**Every account is isolated.** You see **only your own** orders, requests, contracts, messages, and files. The portal systematically scopes every request to your client identifier and to your supplier's schema (`get_current_b2b_client`, `erp_auth.py:581`; `client_company_id` / `client_id` filter on every query). You can never view the data of another client of the same supplier, nor that of another supplier.

**Two watertight worlds.** A client portal token **cannot** open the ERP, and an ERP token **cannot** open the portal: each world rejects the other's token (403). Your portal session uses its own token (stored under `b2b_token`), its own network instance, and its own login page.

**Immediate revocation.** Even though your session lasts several days (the token is valid for 7 days), the server rechecks on **every request** that your account and your client record are still active. If the supplier disables your access, you are signed out on the very next action (401).

**The portal is NOT subject to "read-only mode".** When an ERP supplier's subscription falls into arrears, its ERP may switch to read-only; **this restriction does not apply to the client portal** (`erp_auth.py:581` does not apply the read-only-mode logic). Your portal keeps working normally (reading **and** writing the cart / orders).

### 1.5 The "tracking" of your files depends on a link set by the supplier

The **Tracking** tab (your real quotes and projects) shows data **only if** the supplier has **linked** your portal account to your **company record** in its CRM. This link is **explicit** and **set only by the supplier** (in module 06, Clients tab). There is **never** any automatic association by email: this is a deliberate security decision (since the sign-up email is not verified, using it would allow impersonation of another company's quotes and projects, `b2b_portal.py:54-88`).

Until this link is set, the Tracking tab shows a yellow banner **"Account not linked yet"** and stays empty. The other tabs (catalogue, cart, orders, requests, contracts, messages) work **without** this link.

### 1.6 What the portal does — and does NOT do

The portal **does**: sign up and sign in per supplier, browse the catalogue and manage favorites, fill a cart, place a taxed order, track orders, track real quotes and projects (if linked), view contracts, submit quote requests, exchange messages tied to a request.

The portal **does NOT**:

- **No online payment.** Placing an order **charges no card**. The order is born "unpaid"; settlement happens **outside the portal**, per your agreement with the supplier (see §4.4). There is **neither** Stripe **nor** any payment gateway in the portal.
- **No AI assistant**, no credit consumption (see §1.2).
- **No order cancellation** on your side. Once the order is placed, only the supplier can advance or cancel it. Your view of orders is **read-only**.
- **No accepting or declining a quote** in the portal. When the supplier prices your request, you **see** the received quotes, but the decision to accept/decline is settled with the supplier (on the back-office side).
- **No "Profile" or "My account" page.** The user menu offers only **"Sign out"**. You cannot change your profile or your password from the portal (labels "Profile" / "Settings" exist in the translations but are wired to **no** screen — see §4.6).
- **No PDF/CSV export or printing** of its own. The only document you can view is your **official quote**, via an external link to its public page (Tracking tab → Quotes, see §2.8).
- **No attachment** in messaging, **no display of stock** in the catalogue, **no visible pagination** of the catalogue (see the limits in §4.5).

### 1.7 The eight tabs (plus two public screens)

After signing in, the top bar presents **eight tabs** (`B2bPortalLayout.tsx:21-30`). Two public screens (Sign in, Register) precede login, outside the navigation bar. Total: **ten screens**.

| # | Tab | Route | Icon | Role |
|---|-----|-------|------|------|
| 1 | **Home** | `/dashboard` | `LayoutDashboard` | Dashboard: 4 indicators + 3 quick actions |
| 2 | **Catalogue** | `/catalogue` | `ShoppingBag` | Supplier's products, search, favorites, add to cart |
| 3 | **Cart** | `/panier` | `ShoppingCart` | Items, quantities, taxes, order placement (item-count badge) |
| 4 | **Orders** | `/commandes` | `Package` | Order history + detail |
| 5 | **Tracking** | `/suivi` | `ClipboardList` | Your real **quotes** and **projects** (read-only) |
| 6 | **Contracts** | `/contrats` | `Handshake` | Your contracts (read-only) |
| 7 | **Requests** | `/demandes` | `FileText` | Your quote requests + quotes received |
| 8 | **Messages** | `/messages` | `MessageSquare` | Conversation threads, one per request |

---

## 2. Interface

### 2.1 General layout (top bar, navigation, footer)

All protected pages share the same **layout** (`B2bPortalLayout.tsx`). From left to right, the **top bar** (fixed at the top) contains:

- **Hamburger button** (on small screens, < lg): opens the eight tabs;
- **"Constructo AI Login" link** (back arrow): leaves the portal for the ERP root. On a narrow screen, only the icon remains visible;
- **"B2B Portal" brand** accompanied by the **name of your company** (below the brand);
- **Navigation** (the eight tabs, on large screens): each tab lights up blue when active. The **Cart** tab shows a **badge** with the item count as soon as there is at least one item;
- **User menu** (on the right): your name and email, then a single option, **"Sign out"**.

At the bottom of the page, a discreet **footer** shows "Constructo AI · B2B Client Portal · 2026".

> **Theme.** The portal always displays with Constructo AI's **default theme** (D365 blue). It **does not inherit** any custom theme the supplier may have configured for its ERP (`B2bPortalLayout.tsx:43`). This is intentional: the portal is a neutral space, common to all clients.

### 2.2 Public screen — Sign in (two steps)

Signing in (`B2bLoginPage.tsx`) happens in **two steps**, because the same email could exist with several suppliers: you must first **identify the supplier**.

**Step 1 — "B2B Client Sign-in".** A single field: **"Supplier email"** (helper example: `info@supplier.com`). You enter the email **of your supplier** (not your own), then **"Continue"** (the button shows "Searching..." during the check). Two secondary links: **"No account? Create one"** (to registration) and **"ERP Sign-in"** (to the ERP login, if you are in fact an ERP user).

- If the supplier is not found or is inactive, a message says so ("Supplier not found. Check the email.").

**Step 2 — "Client Sign-in".** The name of the identified supplier appears at the top (with an arrow to go back). Two fields: **"Your email"** (yours, this time) and **"Password"**. The **"Sign in"** button ("Signing in..." during submission) leads you to the dashboard on success.

On a large screen, a left panel introduces the portal: title **"Client Portal"**, text "Browse the catalog, place orders, submit quote requests and track your projects.", and the version mention "Constructo AI B2B v1.0". A red error banner (with a close button) appears if there is a problem.

> **Security — deliberately vague error message.** Whether the account does not exist or the password is wrong, the message is the **same** (generic 401). This protects against account enumeration: no one can guess, from the login page, which emails are registered.

### 2.3 Public screen — Register (identify the supplier, then your information)

Registration (`B2bRegisterPage.tsx`) lets you **request** access. It runs in **two entry steps**, followed by a **confirmation screen**.

**Step 1 — "Create a client account" (Identify your supplier).** Field **"Supplier email"**, with the helper "Ask your supplier contact for this email." Button **"Continue"** (disabled while the field is empty). Link "Already have an account? Sign in". (The screen is labeled "Step 1 of 2".)

**Step 2 — "Your information".** A form:

| Field | Required | Detail |
|-------|:--------:|--------|
| **Your company name** | Yes | e.g. "Acme Construction Inc." |
| **Your full name** | Yes | e.g. "John Smith" |
| **Your email** | Yes | Your sign-in address |
| **Phone** | No | e.g. "(514) 555-0100" |
| **City** | No | e.g. "Montreal" |
| **Postal code** | No | e.g. "H1A 1A1" |
| **Password** | Yes | **Minimum 6 characters** |
| **Confirm password** | Yes | Must match the previous field |

Button **"Create my account"** ("Creating..." during submission). Before even sending, the portal checks, on the client side, that the password is at least 6 characters and that the confirmation matches.

> **The province is fixed to "Quebec".** The form does not offer a province choice: it is recorded as "Quebec" (`B2bRegisterPage.tsx:47`). If you are outside Quebec, tell your supplier, who will adjust your record on its side.

**Confirmation screen — "Request sent!".** A success icon, then "Your access request has been sent to **{supplier}**. You will receive access once the supplier has approved your request." and "You can close this page. Try signing in later." Button **"Back to sign-in"**.

> **Your account is created INACTIVE.** On registration, the account is saved **disabled**: you **do not receive** a session, and you **cannot** sign in until the supplier has **approved** your request (in its back-office, module 06 → B2B → Access requests). The supplier receives a notification. There is **no** confirmation email to click: activation is **manual**, performed by someone at the supplier.

### 2.4 Home / Dashboard

The home page (`B2bDashboardPage.tsx`) greets you with **"Welcome, {your name}"** followed by your company name, then presents **four clickable indicators** and **three quick actions**.

**The four indicators** (each leads to the matching tab):

| Card | What it actually counts | Leads to |
|------|-------------------------|----------|
| **Orders in progress** | Your orders whose status is **neither Delivered nor Cancelled** | Orders |
| **Pending requests** | Your requests with status **New** or **In progress** | Requests |
| **Active contracts** | Your contracts with status **Active** or **In progress** | Contracts |
| **Unread messages** | Messages **written by the supplier** that you have not read yet | Messages |

Until the data has loaded, each card shows `--`.

> **A label nuance.** The **"Pending requests"** card (shown on screen as "Pending inquiries") actually counts the **"New"** or **"In progress"** requests (the server's `demandes_en_cours` counter, `b2b_portal.py:180`). The label speaks of "pending", the data speaks of "in progress": it is the **same** set of open requests (those that are neither accepted, nor declined, nor cancelled). Do not look for a distinction between the two words.

**The three quick actions**: **"Catalogue"** (browse products), **"New request"** (submit a quote request), **"New message"** (go to messaging).

### 2.5 Catalogue

The catalogue (`B2bCataloguePage.tsx`) presents the **supplier's products** as a grid of cards.

**Filters, at the top:**

- **Search field** (magnifier icon, "Search a product...") : filters on the **name**, the **product code**, or the **description**. The search triggers **after a short typing pause** (400 ms) so as not to overload the server;
- **Category dropdown**: "All categories" by default, or one of the categories actually present at the supplier.

**Each product card** shows:

- the product **name**;
- its **product code** (if any);
- a **heart button** (favorite): click to add it to your favorites (the heart turns red) or remove it (it turns gray again);
- a **description** (two lines maximum);
- a **category badge**;
- the **unit price** (e.g. "$12.50", or "--" if no price is set) and the **unit**;
- an **"Add to cart"** button: on click, it turns green for about a second and a half to confirm the addition. A safeguard prevents stacked additions on a double-click.

A loading indicator appears while loading; "No products" if the catalogue is empty or if no product matches the filter; a red banner in case of error.

> **What the catalogue does NOT show.** The **available stock** is **not displayed** to the client (the information travels on the server side but stays hidden in the card). There is **no** visible pagination: the page shows the first set of products (up to 20). If the catalogue is very large, refine your search or your category to find a specific product. Finally, **favorites** have **no** dedicated page: the heart only colors the product in the catalogue (see §4.6).

### 2.6 Cart and order

The cart (`B2bPanierPage.tsx`) gathers the chosen products and is used to **place the order**.

**Empty cart:** message "Your cart is empty" and a "Browse catalog" button.

**Left column — the items.** For each line:

- the product **name** (or "Product #{id}" if the name is no longer available) and its **code**;
- the **unit price** and the **unit**;
- **quantity** controls **−** / **+** (the "−" is disabled when the quantity reaches 1; a safeguard prevents stacking rapid actions on the same item);
- the **line subtotal**;
- a **trash** icon to remove the item.

**Right column — the summary:**

| Line | Calculation |
|------|-------------|
| **Subtotal** | Sum of (price × quantity) |
| **GST (5%)** | 5% of the subtotal |
| **QST (9.975%)** | 9.975% of the subtotal |
| **Total incl. tax** | Subtotal + GST + QST |

A **"Place order"** button unfolds the **order form**: **Shipping address**, **City**, **Postal code**, **Notes (optional)**, then **"Confirm order"** (green, "Processing..." during submission) and **"Cancel"**.

**After confirmation**, a **full-screen** screen confirms: green icon, "Order confirmed!", the order **number** and the **total**, then a "Back to cart" button.

> **What happens on confirmation (server side).** The portal checks that **stock is sufficient** for each product before creating the order; if a product is short, the order is **refused** with a "Insufficient stock for: ..." message and **nothing** is deducted. If stock is sufficient, the order is created, the **stock is decremented**, and a stock movement is recorded for traceability. Two simultaneous confirmations (two tabs, a double-click) are **serialized**: the second fails cleanly, without creating a duplicate order (`b2b_portal.py:474-694`). **No amount is charged**: the order is created "unpaid" (see §4.4).

### 2.7 Orders

The Orders tab (`B2bCommandesPage.tsx`) lists your orders, from most recent to oldest. Empty: "No orders".

**The list** shows, per line: the **number**, the **date**, a colored **status badge**, and the **total incl. tax** (the latter from medium screens up).

**Order statuses:**

| Status | Meaning |
|--------|---------|
| **Pending** | Order received, not yet confirmed by the supplier |
| **Confirmed** | The supplier has confirmed the order |
| **In preparation** | The order is being prepared |
| **Shipped** | The order has left the supplier |
| **Delivered** | The order has been delivered to you |
| **Cancelled** | The order has been cancelled |

**The detail** (on clicking an order) shows: a "Back to orders" button, the header (number + date + status badge), a **line table** (columns **Product / Qty / Unit price / Total**), then **Subtotal / GST (5%) / QST (9.975%) / Total incl. tax**.

> **The view is read-only.** You **cannot** cancel or modify an order from the portal. Progression (Confirmed → In preparation → Shipped → Delivered) and any cancellation are driven by the supplier in its back-office. For any change, contact the supplier (through messaging if needed, §2.11).

### 2.8 Tracking (Quotes / Projects)

The Tracking tab (`B2bSuiviPage.tsx`) shows **your real files in the supplier's ERP**: "Tracking my files" / "Track the status of your quotes and the progress of your projects." **Everything here is read-only.**

> **"Account not linked yet" banner.** If the supplier has not yet linked your account to your company record (see §1.5), a yellow banner appears: "Your files are not yet linked to your record at the company. Contact the company to enable tracking." The two sub-tabs below then stay empty.

**Two sub-tabs:**

**"Quotes".** Your **real quotes** (the ones the supplier prepared for you in its Quotes module). Each card shows: the **project name** or the **quote number**, the **date**, a **status badge** (Sent, Pending, In revision, Accepted, Declined, Expired, Completed…), and the **amount**. If a quote has a **public link**, the card becomes an **external link** (link icon) to the **official document** (public page `/devis/public/{token}`): this is the only document you can view/print from the portal. The supplier's **drafts** are **never** shown (prices still confidential).

**"Projects".** Your **real projects**. Each card shows: the project **name** or **number**, the job-site **city**, a **status badge**, a **progress bar** in percent, and, when available, the **Start** / **Expected end** dates and the **Job site** address.

> **The internal budget is NEVER shown.** The Projects view exposes progress, dates, and the job site, but **not** the project's internal budget (`b2b_portal.py:838`). That is management data belonging to the supplier, outside your portal.

### 2.9 Contracts

The Contracts tab (`B2bContratsPage.tsx`) lists your contracts. Empty: "No contracts". **Read-only.**

**The list**: title / number, date, **status badge**, and amount. Possible **statuses**: **Draft**, **Active**, **In progress**, **Completed**, **Cancelled**, **Suspended**.

**The detail** (on click): title + contract number, status badge, **Amount**, **Amount paid**, **Start date**, **Expected end date**, and a **progress bar** in percent. No action is possible: a contract is **created by the supplier** (usually when it **accepts** one of your quotes).

### 2.10 Requests

The Requests tab (`B2bDemandesPage.tsx`) is where you **request a quote** from the supplier. Heading "My inquiries", with a **"New inquiry"** button.

> **Wording note.** The top-bar tab is labeled **Requests**, but the screens use the word *inquiry* ("My inquiries", "New inquiry"), and the dashboard card reads *"Pending inquiries"*. They all designate the same thing: your quote requests.

**The new-request form** ("New quote request"):

| Field | Required | Detail |
|-------|:--------:|--------|
| **Project title** | Yes | Short summary of the need |
| **Detailed description** | No | The detail of what you want priced |
| **Category (e.g. renovation)** | No | Type of work |
| **Estimated budget ($)** | No | Indicative amount |
| **Deadline** | No | Desired due date |
| **Priority** | No | **Low**, **Normal**, **High**, or **Urgent** (Normal by default) |

Buttons **"Submit"** ("Sending..." during submission) and **"Cancel"**. A success message confirms: "Request submitted successfully".

**The request list**: title, **number of quotes received**, and date, **status badge** (New, In progress, Submitted, Accepted, Declined, Cancelled…). Empty: "No inquiries" and "Submit your first quote request".

**The detail** (on click): title + status badge, description, then **Category / Budget / Priority / Deadline**, and finally the **"Received quotes (N)"** section. Each received quote shows: its **amount**, a **status badge** (Draft, Submitted, Under review, Accepted, Declined, Expired), its **description**, and the **delay in days**. If there are none yet: "No quotes received yet."

> **You do not decide in the portal.** You **see** the quotes the supplier sends you, but you **cannot** accept or decline them from the portal: the decision is made with the supplier (on the back-office side). Moreover, the supplier's quote **drafts** are **not** shown to you (prices still being prepared). Finally, the job-site **address/city** field is **not** offered in this form (see §4.6): state the location in the **description** if needed.

### 2.11 Messages

The Messages tab (`B2bMessagesPage.tsx`) is your **conversation thread** with the supplier. Important particularity: **each message is attached to a request**. There is no "free" message without a request.

**Left column — "My inquiries".** The list of your requests, which serves as the **conversation list**. If you have none: "No inquiries. Create one to chat with your supplier."

**Right column — the conversation.** As long as no request is chosen: "Select an inquiry to view the conversation." Once a request is chosen, the messages appear as bubbles: **"You"** (on the right) and **"Supplier"** (on the left), with the timestamp. If the thread is empty: "No messages yet. Start the conversation."

**Input area.** A "Your message..." field and a **"Send"** button (disabled if the field is empty). When a thread opens, the supplier's messages are automatically **marked as read** (which lowers the dashboard's "Unread messages" counter).

> **Messaging limits.** No **attachment**, no editable **subject**, and the conversation is always **attached to a request** (messages tied to a contract are handled by the server, but no portal screen creates any). To send a document, use another channel agreed with your supplier.

---

## 3. Step-by-step workflows

### 3.1 Get access to the portal (first time)

1. Ask your contact at the supplier for **the email** that identifies its company in Constructo AI (e.g. `info@supplier.com`) and the portal address (`app.constructoai.ca/b2b-portal`).
2. Open `/b2b-portal/register`. **Step 1:** enter the **supplier email**, then "Continue".
3. **Step 2:** fill in your information (company name, your name, your email, a password of at least 6 characters, and optionally phone/city/postal code), then **"Create my account"**.
4. The "Request sent!" screen confirms the submission. **Your account is inactive**: wait for the supplier to **approve** your request.
5. Once notified of approval, return to `/b2b-portal/login`, identify the supplier (Step 1), then sign in with **your** email and password (Step 2).

> If sign-in fails right after registration, it is probably because approval has not been done yet: try again later or follow up with your contact.

### 3.2 Place a first order

1. **Catalogue** tab. Search for a product (search bar) or filter by category.
2. On each desired product, click **"Add to cart"** (the button turns green for a moment). Repeat for the other products.
3. **Cart** tab (the badge shows the item count). Adjust the **quantities** with **−** / **+**, remove an item if needed (trash).
4. Check the **summary**: Subtotal, GST (5%), QST (9.975%), Total incl. tax.
5. Click **"Place order"**, fill in **Shipping address**, **City**, **Postal code** and, if needed, **Notes**.
6. Click **"Confirm order"**. The "Order confirmed!" screen shows the **number** and the **total**.
7. Find the order in the **Orders** tab. The supplier will then advance it (Confirmed → … → Delivered).

> **No payment is requested at this step.** Settlement happens per your agreement with the supplier, outside the portal.

### 3.3 Track an order

1. **Orders** tab. Locate the order by its **number**, its **date**, or its **status badge**.
2. Click the line to open the **detail**: product lines (Qty, Unit price, Total) and the taxed summary.
3. The **status** tells you about progress (Pending, Confirmed, In preparation, Shipped, Delivered). You cannot cancel it yourself: contact the supplier if needed.

### 3.4 Request a quote and exchange messages

1. **Requests** tab → **"New inquiry"**.
2. Fill in at least the **Project title**; add description, category, budget, deadline, and priority as needed. State the **job-site location in the description** if necessary (the dedicated field is not in this form).
3. **"Submit"**. The request appears in the list (status "New").
4. **Messages** tab: select your request on the left, write to the supplier to clarify the need, then **"Send"**.
5. When the supplier prices it, return to the **request detail**: the **"Received quotes"** section shows amount, status, description, and delay. Discuss acceptance directly with the supplier.

### 3.5 Track your real quotes and projects

1. **Tracking** tab. If a yellow **"Account not linked yet"** banner appears, ask the supplier to **link** your account to your company record (see §3.7). Without this link, nothing shows.
2. Once linked, **"Quotes"** sub-tab: view your real quotes (status, amount). If a card is a **link** (icon), click it to open the **official document** (public page) — you can view, print, or download it from that external page.
3. **"Projects"** sub-tab: track the **progress** (percentage bar), the **dates**, and the **job site** of your projects.

### 3.6 View a contract

1. **Contracts** tab. Locate the contract by its title/number and its **status badge**.
2. Click to see the **detail**: Amount, Amount paid, Start and Expected end dates, progress. No action: the contract is managed by the supplier.

### 3.7 (For the supplier) Make tracking visible to a client

This workflow runs on the **supplier side** (module 06 — Sales → B2B/B2C tab), not in the client portal. It is described here so the client knows what to ask for:

1. In the B2B back-office, open the **client** record (`b2b_clients`).
2. **Link** this client to the matching **CRM company** (`companies`): it is this explicit link (`company_id`) that enables Tracking on the portal side.
3. **Never** rely on an automatic association by email: it does not exist (a security decision). The link must be **set manually** by the supplier.

As soon as this link is set, the quotes (non-draft) and projects of that company appear in the client's Tracking tab.

---

## 4. Reference

### 4.1 Portal screens

| Screen | File | Access | Read/Write |
|--------|------|--------|------------|
| Sign in (2 steps) | `B2bLoginPage.tsx` | Public | — |
| Register (2 steps + confirmation) | `B2bRegisterPage.tsx` | Public | Creates an **inactive** account |
| Home / Dashboard | `B2bDashboardPage.tsx` | Protected | Read |
| Catalogue | `B2bCataloguePage.tsx` | Protected | Read + favorites + add to cart |
| Cart & order | `B2bPanierPage.tsx` | Protected | Write (cart, order creation) |
| Orders | `B2bCommandesPage.tsx` | Protected | Read-only |
| Tracking (Quotes / Projects) | `B2bSuiviPage.tsx` | Protected | Read-only |
| Contracts | `B2bContratsPage.tsx` | Protected | Read-only |
| Requests | `B2bDemandesPage.tsx` | Protected | Read + request creation |
| Messages | `B2bMessagesPage.tsx` | Protected | Read + send message |

The protected screens go through `B2bProtectedRoute.tsx`: without a valid session, redirect to `/b2b-portal/login`.

### 4.2 API endpoints (26 in total)

All prefixed with `/api/erp/v1`. The 22 portal endpoints require a client session (`get_current_b2b_client`); the 4 authentication endpoints are public (except the profile).

**Authentication (`routers/auth.py`):**

| Method | Path | Access | Role |
|--------|------|--------|------|
| POST | `/auth/b2b-tenant-lookup` | Public | Step 1: supplier email → name + schema identifier. Refuses an inactive company. |
| POST | `/auth/b2b-client-login` | Public | Step 2: email + password + supplier → session (7-day token). |
| POST | `/auth/b2b-client-register` | Public | Registration: creates an **inactive** account awaiting approval. |
| GET | `/auth/b2b-me` | Client | Profile of the signed-in client. |

**Portal (`routers/b2b_portal.py`, 22 endpoints):**

| # | Method | Path | Role |
|---|--------|------|------|
| 1 | GET | `/b2b-portal/dashboard` | The 4 dashboard indicators |
| 2 | GET | `/b2b-portal/catalogue` | Products (search, category, server pagination 20/page) |
| 3 | GET | `/b2b-portal/panier` | The active cart (with taxes and item count) |
| 4 | POST | `/b2b-portal/panier/items` | Add a product to the cart |
| 5 | PUT | `/b2b-portal/panier/items/{id}` | Change the quantity (0 = remove) |
| 6 | DELETE | `/b2b-portal/panier/items/{id}` | Remove an item |
| 7 | POST | `/b2b-portal/panier/commander` | **Place the order** (stock check, decrement, taxes) |
| 8 | GET | `/b2b-portal/commandes` | List of orders |
| 9 | GET | `/b2b-portal/commandes/{id}` | Detail of an order |
| 10 | GET | `/b2b-portal/suivi/soumissions` | Your real quotes (non-draft) |
| 11 | GET | `/b2b-portal/suivi/projets` | Your real projects (without the internal budget) |
| 12 | GET | `/b2b-portal/demandes` | List of requests + number of quotes |
| 13 | POST | `/b2b-portal/demandes` | Create a quote request |
| 14 | GET | `/b2b-portal/demandes/{id}` | Detail + quotes received (non-draft) |
| 15 | GET | `/b2b-portal/contrats` | List of contracts |
| 16 | GET | `/b2b-portal/contrats/{id}` | Detail of a contract |
| 17 | GET | `/b2b-portal/messages` | Conversation threads (by request or contract) |
| 18 | POST | `/b2b-portal/messages` | Send a message (attached to a request/contract) |
| 19 | PUT | `/b2b-portal/messages/read` | Mark the supplier's messages as read |
| 20 | GET | `/b2b-portal/favoris` | List of favorites |
| 21 | POST | `/b2b-portal/favoris/{produit_id}` | Add a favorite (idempotent) |
| 22 | DELETE | `/b2b-portal/favoris/{produit_id}` | Remove a favorite |

### 4.3 Statuses

| Domain | Values (display) |
|--------|------------------|
| **Order** | Pending · Confirmed · In preparation · Shipped · Delivered · Cancelled |
| **Order payment** | Unpaid · Paid · Refunded · Pending (an order is born **Unpaid**) |
| **Request** | New · In progress · Submitted · Accepted · Declined · Cancelled |
| **Received quote** | Draft *(never shown)* · Submitted · Under review · Accepted · Declined · Expired |
| **Contract** | Draft · Active · In progress · Completed · Cancelled · Suspended |
| **Quote (Tracking)** | Sent · Pending · In revision · Accepted · Approved · Won · Declined · Expired · Completed *(draft excluded)* |
| **Project (Tracking)** | Pending · To do · Planned · In progress · Completed · Complete · Cancelled · Suspended · Paused |

### 4.4 Calculations and money

- **Taxes.** GST = **5%** and QST = **9.975%**, applied to the **subtotal** of the cart and the order (`b2b_portal.py:23-24`). On screen, the exact labels are "GST (5%)" and "QST (9.975%)".
- **Subtotal** = sum of (unit price × quantity) of each line. **Total incl. tax** = Subtotal + GST + QST.
- **Price frozen at addition.** An item's price is the product's price **at the moment you add it to the cart**. If the supplier later changes its price, your cart keeps the original price.
- **No online settlement.** Placing an order **charges no card** and calls **no** payment gateway. The order is created with the payment status **"Unpaid"**. The only concrete effect on the supplier side is the **stock decrement** (`produits.stock_disponible`) and the writing of an outbound **stock movement**, for traceability. Payment is settled outside the portal, per your agreement.
- **Stock check at order time.** If a product in the cart no longer has enough stock, the entire order is **refused** ("Insufficient stock for: ...") and nothing is deducted. Adjust the quantities or remove the item, then try again.
- **Order number**: of the form **`CMD-YYYYMMDD-NNNN`** (derived from the order's real identifier, never from a mere counter — no duplicate possible).

### 4.5 Limits and bounds

| Element | Limit |
|---------|-------|
| Password length (registration) | **6 characters minimum** |
| Quantity per item (add / modify) | from **1** to **1,000,000** |
| Catalogue — results per page (server) | 20 (pagination is **not** exposed in the interface) |
| Messages returned per thread | 100 maximum |
| Shipping address / Order notes | bounded (500 and 5,000 characters respectively) |
| Session duration (token) | 7 days (revocable immediately by the supplier) |

### 4.6 Pitfalls and non-exposed elements (worth knowing)

- **No "Profile / My account" page.** Labels "Profile", "Settings", and "My account" exist in the translations but are wired to **no** screen. The user menu offers only **"Sign out"**. You cannot change your profile or your password from the portal.
- **Favorites with no dedicated page.** Favorites are set in the catalogue (heart button) but there is **no** "My favorites" view: the favorite only colors the product.
- **Fields present in the API but absent from the screen.** Request creation technically accepts a job-site **address/city**, but the form does not offer them (put the location in the description). Sending a message technically accepts a **subject** and an attachment to a **contract**, neither exposed in the screen (which only handles the per-request thread). The catalogue technically receives the **available stock**, not displayed.
- **Technical ambiguity of the `client_company_id` name** *(for advanced administrators)*. In the **cart, orders, contracts, and favorites** tables, the `client_company_id` column actually contains the **client's** identifier (`b2b_clients.id`), **not** that of the CRM company. In contrast, in the **quotes** and **projects** tables (Tracking), the `client_company_id` field does contain the identifier of the **CRM company** (`companies.id`), resolved from the `b2b_clients.company_id` link. Two meanings for the same name: not to be confused when reading the database directly.
- **Tables shared with the legacy application.** The `b2b_*` tables are common to the React ERP and the supplier's legacy Streamlit application. The server repairs and harmonizes their constraints on demand (`_ensure_b2b_tables`, `b2b.py:227`) so the two worlds coexist (status casing, inherited primary keys). This is transparent to you.

---

## 5. Integrations and FAQ

### 5.1 Links with the other modules

- **Sales / B2B back-office (module 06).** This is the **supplier counterpart** of the portal. There the supplier **approves** your access, **links** your account to your company record (enables Tracking), **prices** your requests into quotes, **creates** your contracts, **advances** your orders, and **replies** to your messages. The "AI Assistant — B2B Management" also lives there, on the supplier side only.
- **Companies (module 04).** Your **company record** in the supplier's CRM (`companies`) is what your portal account must be **linked** to so that Tracking shows your quotes and projects.
- **Quotes / Estimates (module 08).** The **real quotes** visible in Tracking → Quotes come from this module. A card's **external link** opens the quote's **public page** (`/devis/public/{token}`), the only document you can view and print from the portal.
- **Projects (module 09).** The **real projects** visible in Tracking → Projects come from this module (progress, dates, job site — without the budget).
- **Store / Inventory (module 10).** The portal **catalogue** reads the **products** of this module (name, code, price, category). Placing an order **decrements the stock** and writes a **stock movement** here.
- **Accounting (module 15).** The portal **generates no invoice**: orders are born "unpaid". Billing and collection are handled on the supplier side.
- **Configuration (module 28).** The supplier's subscription is managed there. Good to know: **the portal is not affected** by the read-only mode that Configuration can impose on the supplier's ERP.

### 5.2 Frequently asked questions

**Do I need an ERP account to use the portal?**
No. You use a separate **client account**. You sign up, then the supplier **approves** your access.

**Why must I enter the supplier's email before my own?**
Because the same email could exist with several suppliers. The first step **identifies the company**; the second signs you in.

**I just signed up and can't sign in. Why?**
Your account is created **inactive**. You will only be able to sign in after **approval** by the supplier (a manual action on its side). There is no confirmation email to click.

**Does placing an order charge me online?**
No. **No payment** is taken in the portal. The order is created "unpaid"; you settle per your agreement with the supplier. The only immediate effect is the **stock reservation** (decrement).

**Can I cancel or modify an order I have already placed?**
No, not from the portal. Contact the supplier: only it can advance or cancel an order.

**Can I accept or decline a quote in the portal?**
No. You **see** the received quotes, but the decision is made with the supplier (on the back-office side).

**My Tracking tab shows "Account not linked yet". What should I do?**
Ask the supplier to **link** your account to your company record in its CRM. This link is **manual** and **mandatory**; without it, no quote or project shows.

**Where are my favorites?**
There is **no** "Favorites" page. The heart button simply colors the product in the catalogue.

**Why don't I see a product's available stock?**
Stock is **not** shown to clients. The check happens at order time: if stock is short, the order is refused with a clear message.

**The catalogue is huge: where is the pagination?**
The interface shows the first set of products (20). There is no visible pagination. Use the **search** or the **category filter** to find a specific product.

**Can I attach a file to a message?**
No. Messaging handles no attachment and is limited to a **thread per request**. Use another agreed channel to send a document.

**Can I change my password or my profile in the portal?**
No. There is **no** profile/settings page on the client side. Ask the supplier if a change is needed.

**Does the portal use artificial intelligence? Is it billed?**
No. The client portal **contains no AI assistant** and **consumes no credits**. The "B2B Management" assistant is reserved for the supplier (module 06).

**Can I export my orders to PDF or CSV?**
No. The portal has **neither** export **nor** printing of its own. Only your **official quote** can be viewed/printed, via its external link (Tracking → Quotes).

**Does my session expire?**
It lasts 7 days, but the supplier can **revoke** your access immediately (you are then signed out on the very next action).

**Does the portal stay open if the supplier's subscription is suspended?**
Yes, the client portal **is not** subject to read-only mode. It keeps working normally (unlike the supplier's ERP, which may switch to read-only).

### 5.3 Common troubleshooting

| Symptom | Lead |
|---------|------|
| "Supplier not found" at sign-in | The supplier email is wrong, or its company is inactive. Confirm the address with your contact. |
| Sign-in refused right after registration | Your account is not yet **approved**. Try again later or follow up with the supplier. |
| Same error message whatever the field | Normal: the sign-in message is **deliberately generic** (anti-enumeration protection). Check the supplier, email, and password. |
| Empty Tracking tab + yellow banner | **Account not linked**: ask the supplier to set the link to your company record. |
| "Insufficient stock for: ..." at order time | A product no longer has enough stock. Reduce the quantity or remove the item, then reorder. |
| A cart item's price no longer matches the catalogue | The price is **frozen at addition**. Remove then re-add the item to take the new price. |
| Cannot find a product in a large catalogue | No visible pagination: use the **search** or the **category filter**. |
| Sudden sign-out | Session expired (7 days) **or** access revoked by the supplier. Sign in again; if access is revoked, contact the supplier. |
| No button to accept a quote / cancel an order | Normal: those decisions are on the supplier side. Use messaging or a phone call. |

---

## 6. Summary

- **The B2B / B2C Portal is the online space a supplier opens to its clients.** Address: `/b2b-portal` ("B2B / C2B — Client portal" tile on the ERP login page). A **standalone** application, separate from the ERP.
- **Two distinct worlds**: the **client portal** (this manual) and the **supplier back-office** (module 06, `/ventes?tab=b2b`, administrators). It is the supplier who **approves** access, **prices** quotes, **creates** contracts, and **advances** orders.
- **Eight tabs**: Home, Catalogue, Cart, Orders, Tracking, Contracts, Requests, Messages; plus two public screens (2-step Sign in, Register).
- **Registration = access request.** The account is born **inactive** and is usable only after the supplier's **manual approval** (no confirmation email). Province fixed to "Quebec".
- **Ordering ≠ paying.** No online payment: the order is born **"Unpaid"**. Only concrete effects: **stock decrement** + stock movement. Taxes **GST 5%** and **QST 9.975%** on the subtotal. Stock check at order time (refused if insufficient), with no double-order possible.
- **Conditional tracking.** The **Quotes** and **Projects** tabs show your real files **only if** the supplier has **linked** your account to your company record (an explicit link, never automatic). Project **budgets** and quote **drafts** stay hidden.
- **Much is read-only on the client side**: orders (after creation), tracking, contracts, received quotes. You **initiate** (request, order, message); the supplier **decides**.
- **What the portal does NOT have**: no AI assistant, no AI credits, no online payment, no profile/password page, no favorites page, no export/printing of its own, no attachment, no visible catalogue pagination, no stock display, no order cancellation, no quote acceptance.
- **Security**: watertight ERP and portal worlds (each rejects the other's token); strict per-client isolation; generic sign-in message (anti-enumeration); immediate access revocation; **the portal is not** subject to the supplier's Stripe read-only mode.
- **26 client endpoints**: 4 for authentication (`auth.py`) + 22 for the portal (`b2b_portal.py`).

---

*Verified source files:* `backend/routers/b2b_portal.py` (1,321 lines, 22 endpoints); `backend/routers/auth.py` (4 B2B endpoints: `b2b-tenant-lookup`, `b2b-client-login`, `b2b-client-register`, `b2b-me`); guards `backend/erp_auth.py` (`get_current_b2b_client`, `create_b2b_client_jwt`); DDL and repairs `backend/routers/b2b.py` (`_ensure_b2b_tables`, shared `b2b_*` tables). Frontend: `frontend/src/pages/b2b-portal/` (`B2bLoginPage.tsx` 208, `B2bRegisterPage.tsx` 391, `B2bDashboardPage.tsx` 97, `B2bCataloguePage.tsx` 134, `B2bPanierPage.tsx` 154, `B2bCommandesPage.tsx` 119, `B2bSuiviPage.tsx` 230, `B2bContratsPage.tsx` 99, `B2bDemandesPage.tsx` 172, `B2bMessagesPage.tsx` 152); `components/layout/B2bPortalLayout.tsx` (183), `B2bProtectedRoute.tsx` (25); `api/b2b-portal.ts` (277), `api/b2b-portal-auth.ts` (175); `store/useB2bPortalStore.ts` (252), `store/useB2bAuthStore.ts` (98); translations `i18n/locales/fr/b2b.json` (`portal` section), `layout.json` (`b2bPortalLayout`), `auth.json` (`b2b`). Mounting: `backend/erp_api.py` (`/api/erp/v1/b2b-portal`).

*Related manuals:* `06-gestion-crm-opportunites.md` (supplier B2B back-office: access approval, record linking, pricing, contracts, order statuses, B2B AI assistant), `04-gestion-entreprises.md` (CRM company record linked to Tracking), `08-ventes-soumissions.md` (real quotes and the document's public page), `09-ventes-projets.md` (real Tracking projects), `10-operations-magasin.md` (catalogue products and stock), `15-operations-comptabilite.md` (billing, outside the portal), `28-configuration.md` (supplier's subscription; the portal is not subject to read-only mode).

*Constructo AI ERP Manual — Module 36 "B2B / B2C Portal (clients and suppliers)" — v1.0 verified — 2026-07.*
