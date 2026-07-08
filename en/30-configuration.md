# Module 30 — Configuration (company, users, subscription)

> **Version**: 3.0 (overhaul verified against the source code, July 2026)
> **Reference code**:
> - Frontend: `frontend/src/pages/ConfigurationPage.tsx` (4,108 lines — a single page with **11 tabs**), API client `frontend/src/api/config.ts` (661 lines) and `frontend/src/api/stripe.ts`, state stores `frontend/src/store/useConfigStore.ts` (209 lines), `frontend/src/store/useStripeStore.ts` (145 lines) and `frontend/src/store/useUiThemeStore.ts` (236 lines), translations `frontend/src/i18n/locales/{fr,en}/config.json` (441 lines each)
> - Backend: `backend/routers/config.py` (3,247 lines — **38 endpoints**, actual prefix `/api/erp/v1/config`), `backend/routers/stripe_routes.py` (379 lines — **6 endpoints**, prefix `/api/erp/v1/stripe`), `backend/routers/html_utils.py` (document theme), access guards `backend/erp_auth.py`
> - **Integrations** tab: separate module `frontend/src/pages/IntegrationPage.tsx` (1,608 lines — **7 sub-tabs**) + router `backend/routers/integration.py` (outside the present module; see the dedicated manual)
> **PostgreSQL tables**: `public.entreprises` (one row per tenant — subscription, taxes, currency, country, time zone, language, holdback, fiscal year), `{tenant}.entreprise_config` (JSON configuration: logo, contact details, document theme, supplier categories, free-form keys), `{tenant}.users` (accounts and roles), `{tenant}.payroll_config` (employer information), `{tenant}.webhooks` / `{tenant}.webhook_deliveries` (outbound webhooks, no UI). Billing goes through the shared Stripe tables and the prepaid AI-credits ledger.
> **Scope**: Configuration is your company's **settings hub** in the ERP. A single tabbed page brings together your personal **profile**, **user management** (accounts and rights), the **company identity** (logo, contact details, RBQ/NEQ/GST/QST numbers), the **appearance** of your documents and your interface, the **tax and jurisdiction settings** (country, currency, taxes, holdback, jurisdiction, fiscal year), the document **language**, the **time zone**, your **subscription** and your **AI credits** (Stripe), and access to the **accounting integrations**. It is an administration module: most of its tabs are reserved for the company **administrator**.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step procedures](#3-step-by-step-procedures)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The **Configuration** page lets the contractor set, in a single place, everything that governs how the ERP behaves for their company:

- **Their account** — change their name, email and password.
- **Users** — create accounts for their employees, assign a role, reset a password, deactivate an account.
- **Company identity** — upload a logo, enter the address and the RBQ (Régie du bâtiment du Québec, the Quebec building authority), NEQ (Quebec enterprise number), GST and QST numbers. This information then appears on every quote, invoice and order.
- **Appearance** — two separate systems: the colors of the **generated documents** (quotes, invoices, orders, emails) and the colors of the **ERP interface**.
- **Tax settings and jurisdiction** — country, currency, sales taxes (GST/QST or others), default holdback (retainage) rate, state or province, fiscal year, plus the employer payroll information and the supplier categories.
- **Document language** — French or English.
- **Time zone** — for timestamping time tracking, creation dates and due dates.
- **Subscription** — subscribe to, manage or cancel the subscription, view and recharge AI credits (through Stripe).
- **Accounting integrations** — connect QuickBooks Online or Sage (separate embedded module).

### 1.2 What the module does — and does not do

| The module **does** | The module **does not** |
|---|---|
| Create, edit and **deactivate** users | **Permanently delete** a user (deactivation only) |
| Grant administrator rights | Change the **username** after creation |
| Customize document colors (stored in the database) | Modify documents already sent (they keep the look they had when sent) |
| Customize interface colors (per browser) | Sync interface colors across your devices |
| Set taxes, currency, country, holdback, fiscal year | Change the **country** once you have invoices or accounting entries |
| Subscribe to, manage and cancel the Stripe subscription | Charge your card outside of Stripe |
| Recharge AI credits (real payment) | Refund AI credits from the ERP |
| Connect QuickBooks / Sage (Integrations tab) | Manage **webhooks** from Configuration (no active tab) |

### 1.3 Access from the sidebar

- **Sidebar** → **System** section → **Configuration** (icon `Settings`).
- **URL**: `/configuration` (`App.tsx:257`).
- **Default open tab**: **Profile**.
- **Direct link to the subscription**: the URL `/configuration?tab=abonnement` opens the **Subscription** tab directly (used when returning from a Stripe payment).
- Protected page: you must be authenticated in a tenant.

### 1.4 Who sees what: administrator or not

The module distinguishes two user profiles. The administrator signal is computed as follows (`ConfigurationPage.tsx:95`):

> You are considered an **administrator** if your account carries the `is_admin` flag (re-read on the server, hence unforgeable), **or** if your role is `admin`, **or** if you are a platform super-administrator.

Six tabs are **administrator-only** (`adminOnly`): **Users**, **Appearance**, **Jurisdiction & Currency**, **Taxes**, **Preferences**, **Timezone**. They are **hidden** from ordinary users.

| Profile | Visible tabs | Count |
|---|---|---|
| **Administrator** | All 11 tabs | **11** |
| **Ordinary user** | Profile, Company (read-only), Interface, Subscription, Integrations | **5** |

> **Important**: hiding a tab is a display convenience, not a security measure. The real protection is **server-side**: every write (creating a user, changing a tax, etc.) requires administrator rights, checked by the `require_tenant_admin_or_role` guard on the server. An ordinary user who reached one of these endpoints would have the write refused.

### 1.5 User roles

When you create an account, you choose a **role** from five (`ConfigurationPage.tsx:72-78`):

| Role | Label | Typical use |
|---|---|---|
| `admin` | **Administrator** | Full access to the company |
| `user` | **User** | Standard business access |
| `employee` | **Employee** | Often tied to an employee record |
| `comptable` | **Accountant** | Accounting- and payroll-oriented |
| `gestionnaire` | **Manager** | Oversight-oriented |

> **Good to know**: these five roles are mainly for display and a few targeted guards (for example, the **accountant** role can read the employer information, and it unlocks the sensitive Accounting functions). The **real** access control in Configuration rests on the **Administrator** flag (`is_admin`), not on the role label. In other words, check "Administrator" to grant full rights, whatever the displayed role. The server does, however, **refuse** to assign the reserved platform "super-administrator" role (422 error message): that status cannot be granted from within a tenant.

### 1.6 The 11 tabs

Source: the `TABS` array (`ConfigurationPage.tsx:55-69`).

| # | Tab | Admin-only? | Role |
|---|---|---|---|
| 1 | **Profile** | No (everyone their own) | Name, email, password of the current user |
| 2 | **Users** | **Yes** | Employee accounts and rights |
| 3 | **Company** | No (editing admin-only) | Logo, contact details, numbers; system configuration |
| 4 | **Appearance** | **Yes** | Colors of the generated documents |
| 5 | **Interface** | No (per user) | Colors of the ERP interface |
| 6 | **Jurisdiction & Currency** | **Yes** | Country, currency, holdback, state/province, fiscal year, employer, categories |
| 7 | **Taxes** | **Yes** | Two configurable sales taxes |
| 8 | **Preferences** | **Yes** | Document language (FR / EN) |
| 9 | **Timezone** | **Yes** | Tenant time zone |
| 10 | **Subscription** | No | Stripe subscription and AI credits |
| 11 | **Integrations** | No | QuickBooks / Sage (separate module) |

### 1.7 Two color systems not to be confused

The module has **two** color tabs, independent of each other:

| Aspect | **Appearance** tab | **Interface** tab |
|---|---|---|
| What it changes | The generated **documents** (quotes, invoices, orders, customer emails) | The ERP **interface** (sidebar, top bar, buttons) |
| Scope | The whole company (all users) | **Your account, on this browser** only |
| Where it is stored | In the database (`document_theme`) | In the browser (local storage), **without** a server call |
| Who can change it | Administrator only | Each user for themselves |
| Number of colors | **8** | **4** |

> Neither one touches the system light/dark mode, which is a separate setting in your browser.

### 1.8 Consultation mode (read-only)

Subscription payment governs writing throughout the ERP, including Configuration. A tenant **without an active Stripe subscription** (canceled subscription, no card on file) switches to **consultation mode**: reading remains possible, but **all writes to `/config/*` are blocked** ("403 — Consultation mode" response).

Two exceptions escape the block, precisely so you can put things right: the **re-subscription and payment** calls (`/stripe/*`) and **sign-out** (`/auth/*`) remain allowed. That is why, even in consultation mode, the **Subscription** tab stays fully functional so you can subscribe again or recharge.

> This mechanism is global to the ERP (defined in `erp_auth.py` and `erp_stripe.py`), not specific to Configuration. It has three states: **`full`** (everything allowed), **`readonly`** (read-only + re-subscription) and **`blocked`** (company deactivated → session cut). On an internal instance where billing is disabled (`ERP_BILLING_ENABLED=false`) or in development mode, everything is `full`.

---

## 2. Interface

### 2.1 General layout

At the top, the title **"Configuration"**. Just below, a horizontal scrolling **tab bar**; the administrator-only tabs do not appear there for an ordinary user. The content changes with the active tab.

Two **banners** can appear above the content:
- a **red** banner on error;
- a **green** banner on success, which **disappears on its own after 3 seconds**.

On the **Subscription** tab, these banners come from the Stripe subsystem (they reflect payment messages).

### 2.2 "Profile" tab

*Open to everyone.* Two cards side by side.

**"Personal information" card**

| Element | Detail |
|---|---|
| Role badge + `@username` | Read-only (the username cannot be changed). |
| **Full name** | Editable text field. |
| **Email** | Editable email field. |
| **Save** button | Saves the name and email. Protected against double-click (spinner while sending). |

**"Change password" card**

| Element | Detail |
|---|---|
| **New password** | Hint shown: "Min. 6 characters". |
| **Confirm password** | Must match the previous field. |
| **Change password** button | Disabled while both fields are empty. |

Input checks: fewer than 6 characters → "Password must be at least 6 characters"; the two fields differ → "Passwords do not match". During the initial load, the page shows "Loading profile…".

### 2.3 "Users" tab

*Administrator-only.* Card header: **"Users ({{count}})"** with two buttons — **Refresh** and **New user**.

**Table (desktop view)** — columns:

| Column | Content |
|---|---|
| **User** | Username; a purple `Shield` icon marks an administrator. |
| **Name** | Full name. |
| **Email** | Address. |
| **Role** | Badge (Administrator / User / Employee / Accountant / Manager). |
| **Status** | **Active** or **Inactive** badge. |
| **Actions** | Icons (see below). |

**Row actions:**
- **Edit** (`Edit3` icon) — opens the edit dialog.
- **Change password** (`Key` icon).
- **Deactivate** (`XCircle` icon) — shown **only** if the row is not your own and the account is active. A confirmation appears: "Deactivate user {{name}}?".

On mobile, each user is shown as an equivalent card. If the list is empty: "No users".

**Three dialogs (modals):**

**a) New user**

| Field | Required | Detail |
|---|---|---|
| **Username** | Yes | Example shown: "e.g. jdupont". Unique within the company. |
| **Password** | Yes | At least 6 characters. |
| **Full name** | No | |
| **Email** | No | |
| **Role** | No | Five-value drop-down. |
| **Administrator** | No | Checkbox (grants full rights). |

**Cancel** / **Create** buttons (the Create button stays disabled while the username or the password is empty). The account is created **active**.

**b) Edit: {{name}}** — Full name, Email, Role, **Administrator** checkbox. **Cancel** / **Save** buttons. *The username is not shown: it cannot be changed.*

**c) Change password** — New password, Confirm. Same checks (6 characters, match). **Cancel** / **Change** buttons.

**Two server safeguards** prevent you from locking yourself out:
- You cannot **remove administrator status** from the **last administrator** of the company.
- You cannot **deactivate the last active administrator**.

In both cases the operation is refused with an explicit message. Deactivation is a **soft delete** (the account becomes "Inactive", it is never erased); you also cannot deactivate your own account.

### 2.4 "Company" tab

*Visible to everyone; editable by administrators only.* Two sub-tabs: **Company information** (default) and **System configuration**.

For a non-administrator user, an amber band appears: **"Read-only: only a company administrator can edit this information."**; all fields are then locked.

**a) Company information**

**Logo.** Preview (image or empty thumbnail), **Upload** / **Change** and **Remove** buttons. Selection constraints: **maximum size 1 MB**, formats **PNG, JPG, GIF, SVG, WEBP**. Note shown: "PNG, JPG, SVG. Max 1 MB. Recommended: 500x200px, transparent background." The logo is converted to base64 and saved in the configuration; it is then reused on your documents.

**Twelve text fields** (`ENTREPRISE_INFO_FIELDS`):

| Field | Example shown |
|---|---|
| **Company name** | Construction ABC Inc. |
| **Address** | |
| **City** | |
| **Province** | |
| **Postal code** | H1A 2B3 |
| **Phone** | |
| **Email** | |
| **Website** | |
| **RBQ number** | 5734-1234-01 |
| **NEQ number** | |
| **GST number** | 123456789 RT0001 |
| **QST number** | 1234567890 TQ0001 |

The **Save** button activates only when there are changes; only the modified fields are saved (one by one). Success message: "Information saved".

**b) System configuration**

Advanced editor for the key/value configuration entries, grouped by category (**General**, **Billing**, **AI**, **Notifications**). Each row shows the key (in code style), a category badge, an optional description, a text field and a **Save** button (which shows a green check 2 seconds after saving). If there is nothing: "No configuration found. Entries will be created automatically by the system."

> **Two protected keys**: `document_theme` and `supplier_categories` **cannot** be edited here (the server refuses with a redirect message). Use the **Appearance** tab instead (document theme) and the **Supplier categories** card (Jurisdiction & Currency tab).

### 2.5 "Appearance" tab (document colors)

*Administrator-only.* Title **"Document appearance"**. Sets the colors of every generated **HTML document**: quotes, invoices, purchase orders, work orders and emails sent to customers.

**Six preset themes**, clickable (a check marks the active theme):

| Theme | Primary color |
|---|---|
| **Constructo Blue** (default) | `#1F4E79` |
| **Forest Green** | `#166534` |
| **Brick Red** | `#991B1B` |
| **Anthracite** | `#1F2937` |
| **Burgundy** | `#7F1D1D` |
| **Ocean** | `#0C4A6E` |

**Advanced customization — 8 colors**, each with a color picker, a hexadecimal field and a usage hint:

| Color | Technical key | Use |
|---|---|---|
| **Primary color** | `primary` | Headers, title banner, table headers |
| **Primary — dark** | `primary_dark` | Dark variant (hover, important borders) |
| **Accent** | `accent` | Subtitles, left border of info boxes |
| **Accent — light** | `accent_light` | Document number on the header |
| **Header text** | `header_text` | Text on the primary background |
| **Alternating rows** | `table_row_alt` | Background of even table rows |
| **Info section background** | `info_bg` | Background of info boxes and totals |
| **Borders** | `border` | Thin rules (table lines, separators) |

Each field validates the hexadecimal format (`#RGB` or `#RRGGBB`); a red border flags an incorrect value. A **live preview** shows a mock quote (header, info boxes, table, total) that updates as you change the colors.

Actions: **Save** (active when there are changes and all colors are valid), **Discard changes**, and the **Reset to defaults** link (with confirmation).

### 2.6 "Interface" tab (ERP colors)

*Open to everyone, per-user setting.* Title **"Interface colors"**, with a subtitle reminding you that "this choice applies to your account on this browser." This setting is saved **locally in the browser** (no server call) and applied **immediately**.

A **contrast warning** (non-blocking) appears if your colors make the text hard to read (WCAG accessibility check).

**Six preset themes** bearing the same names as for documents (Constructo Blue, Forest Green, Brick Red, Anthracite, Burgundy, Ocean).

**Four colors**:

| Color | Default (D365 theme) | Use |
|---|---|---|
| **Primary color** | `#0078D4` | Buttons, active tab, links |
| **Primary — hover** | `#005EA2` | Hover of the primary color |
| **Sidebar** | `#002050` | Sidebar background |
| **Top bar** | `#002B6B` | Top bar background |

**Preview**: an interface mock (sidebar, top bar, active tab, button, link). Actions: **Save**, **Discard changes**, **Reset to defaults** (with confirmation).

> This setting is **not** shared: it does not follow your account from one device or browser to another. On a new device, you start again from the default colors.

### 2.7 "Jurisdiction & Currency" tab

*Administrator-only.* This tab stacks **five jurisdiction cards**, then the **Employer payroll information** card and the **Supplier categories** card. Each card has its own error and success messages and its own save button.

**Card 1 — Jurisdiction & Currency**
- **Country**: Canada or United States.
- **Currency**: Canadian dollar (CAD) or US dollar (USD).
- Shows the saved ISO code. **Save jurisdiction** button.

> **Warning**: changing the **country** is **refused** if your company already has invoices or accounting entries ("409" message). Reason: the country determines the tax labels (GST/QST in Canada, Sales Tax in the United States), and the history already produced must not be made inconsistent. Set the country **before** you start invoicing.

**Card 2 — Default holdback rate**
- Numeric field **Default holdback rate (%)** (0 to 100, step 0.5). Hint: "CA standard: 10% (LCS Quebec)…". **Save rate** button. This rate feeds, by default, the holdbacks in Accounting and in progress claims.

**Card 3 — Jurisdiction (US State / CA Province)**
- Drop-down with the "— Not specified —" option, then two groups: **US states** (50 + District of Columbia) and **Canadian provinces** (13), shown in full. **Save jurisdiction** button.

**Card 4 — Fiscal year**
- **Start month** (12-month menu) and **Start day** (the maximum depends on the month; February 29 is accepted by recurring convention). A preview shows "Fiscal year: {{start}} to {{end}}". By default, the fiscal year follows the calendar year (January 1). **Save fiscal year** button.

**Card 5 — Important note**: four reminders (the country affects the chart of accounts; taxes are set separately in the Taxes tab; the currency; these values apply to the whole company).

**"Employer payroll information" card** — a prerequisite for producing the year-end T4 and RL-1 tax slips. Nine fields, **all optional** (CNESST = Quebec workplace health and safety board; CCQ = the Quebec construction commission, Commission de la construction du Québec):

| Field | Example |
|---|---|
| **CRA payroll account number** | 123456789RP0001 |
| **Revenu Quebec identification number** | 1234567890RS0001 |
| **CNESST number** | |
| **CNESST classification** | |
| **CCQ number** | If subject to CCQ |
| **Address** (employer legal) | |
| **City** | |
| **Postal code** | H2X 1Y4 |
| **Province** | QC (2 letters) |

**Save** / **Discard changes** buttons. *Reading this card is also open to the **accountant** role.*

**"Supplier categories" card** — the list of categories offered in the "Business sector" field when you create a supplier (to classify your expenses). A field + **Add** button (duplicates, case-insensitive, are ignored), an inline-editable list with deletion (`Trash2` icon), a "{{n}} category(ies)" counter, then **Save** / **Reset to default values**. A list of 30 categories is provided by default.

### 2.8 "Taxes" tab

*Administrator-only.* Title **"Tax configuration"**. Quebec defaults: GST 5% and QST 9.975%. **An empty label or a rate of 0 hides the tax**; documents already produced keep their historical taxes.

Four fields:

| Field | Detail |
|---|---|
| **Tax 1 — Label** | Example: "GST / Sales Tax (empty to hide)". Maximum 50 characters. |
| **Tax 1 — Rate (%)** | Number from 0 to 100 (step 0.001). |
| **Tax 2 — Label** | Example: "QST / PST…". |
| **Tax 2 — Rate (%)** | Number from 0 to 100. |

Rates are clamped to the 0–100 range and rounded to 3 decimals. **Save** button (active when there are changes).

**"Example configurations" card** — a reference table:

| Jurisdiction | Tax 1 | Tax 2 |
|---|---|---|
| Quebec | GST 5% | QST 9.975% |
| Ontario | HST 13% | — |
| British Columbia | GST 5% | PST 7% |
| Alberta | GST 5% | — |
| USA (e.g. California) | Sales Tax 8.25% | — |
| Tax-exempt | — | — |

### 2.9 "Preferences" tab (document language)

*Administrator-only.* Title **"Language of generated documents"**. Two radio buttons presented as cards:

- **French** — "Default for Quebec… Currency format: 1 234,56 $".
- **English** — "Recommended for the United States… Currency format: $1,234.56".

**Save** button (active on change). **"Important note"** card with three reminders: historical documents are regenerated on the fly in the new language; the tax labels stay independent; the texts you have customized are not translated automatically.

### 2.10 "Timezone" tab

*Administrator-only.* Title **"Tenant time zone"**. Used to timestamp **time tracking**, **creation dates**, **due dates** and **reports**.

Drop-down of **13 time zones** (IANA zones), with the saved value shown below:

| Group | Time zones |
|---|---|
| **Canada** (6) | Toronto/Montreal, Halifax, St. John's, Winnipeg, Edmonton, Vancouver |
| **United States** (6) | New York, Chicago, Denver, Phoenix (no DST), Los Angeles, Anchorage |
| **Pacific** (1) | Honolulu |

**Save** button (active on change). **"Important note"** card: dates are stored in universal time (UTC) in the database; the effect is immediate; special reminders for Phoenix and Saskatchewan (no daylight saving).

### 2.11 "Subscription" tab

*Open to everyone.* Driven by Stripe. Two cards.

**"Subscription" card** (**Refresh** button). If a subscription exists, three tiles:

| Tile | Content |
|---|---|
| **Status** | Colored badge; "Will be canceled on {{date}}" note if a cancellation is scheduled. |
| **Plan** | Plan name + "{{amount}} $ / month" or "/ year". |
| **Renewal** | Date; "Trial ends: {{date}}" note if you are in a trial period. |

Buttons: **Manage my subscription** (opens the **Stripe customer portal**: card, invoices, plan change), **Cancel** (if the subscription is active and not already being canceled). If there is **no** subscription: "No active subscription" + "Subscribe to a plan…" + **Subscribe now** button (Stripe payment for the `pro` plan).

**"AI Credits" card** — three tiles:

| Tile | Content |
|---|---|
| **Current balance** | Amount in $, or **"Unlimited"** (in green) if your plan is exempt. |
| **Usage this month** | Amount consumed. |
| **Plan type** | Plan name; "Unlimited credits" badge if exempt. |

If you are not exempt, warnings appear: **"Low balance…"** (under $2) and **"AI credits depleted…"** (at 0 or below), with a **Recharge** button.

**Cancellation dialog** — title "Cancel subscription", "Warning" box, message "…will remain active until the end of the period ({{date}})…", **Keep** / **Confirm cancellation** buttons. Cancellation takes effect at the **end of the current period** (you keep access until then).

**Recharge dialog** — title "Recharge AI credits", six quick amounts (**$10 / $25 / $50 / $100 / $200 / $500**), a "Custom amount ($)" field (hint "Min. 5.00"), a **Recharge ${{amount}}** button. Accepted amount: **between $5 and $500**.

> **Recharging is a real payment.** It immediately charges the card on file in your Stripe account (default currency: Canadian dollar). The system **charges first, then credits** idempotently (never a double credit). If the card is declined, a clear message appears ("402" error) and **no credit is added**.

**Possible subscription statuses**: Active, Free trial, Past due, Canceled, Cancellation pending, Incomplete, Expired, Unpaid.

### 2.12 "Integrations" tab

*Open to everyone.* This tab **embeds a separate module** (loaded on demand): accounting-integration management. It has its **own seven sub-tabs**:

| Sub-tab | Role |
|---|---|
| **Overview** | Connection status. |
| **QuickBooks** | OAuth connection to QuickBooks Online + syncs. |
| **Sage 50** | Connection to the Sage 50 connector. |
| **Sage** | Sage Business Cloud Accounting (REST/OAuth). |
| **Webhooks** | Outbound webhooks **of the Integrations module** (this is **where** the visible webhooks are). |
| **Mappings** | Account mapping between the ERP and the accounting software. |
| **History** | Sync logs. |

> The details of these sub-tabs belong to the **Integrations manual**. Just remember that Configuration merely **hosts** this page; it adds no settings of its own.

---

## 3. Step-by-step procedures

### 3.1 Change your name or email

1. **Configuration** → **Profile** tab.
2. In "Personal information", edit **Full name** and/or **Email**.
3. Click **Save**. Green confirmation banner.

### 3.2 Change your own password

1. **Profile** tab → "Change password" card.
2. Enter the **New password** (at least 6 characters) then **Confirm**.
3. Click **Change password**.

### 3.3 Create an account for an employee

1. **Users** tab (administrator) → **New user** button.
2. Enter the **Username** (unique) and a **Password** (6 characters minimum) — both are required.
3. Fill in the **Full name** and the **Email** (optional).
4. Choose a **Role**; check **Administrator** if the person must have full rights.
5. Click **Create**. The account appears in the list, in the **Active** state.

### 3.4 Change a user's role or rights

1. **Users** tab → **Edit** icon on the desired row.
2. Adjust the **Role** and/or the **Administrator** checkbox.
3. Click **Save**.

> If you try to remove administrator status from the **last** administrator, the operation is refused: add another administrator first.

### 3.5 Reset an employee's password

1. **Users** tab → **Change password** icon on the row.
2. Enter and confirm the new password (6 characters minimum).
3. Click **Change**. Then share the password with the employee securely.

### 3.6 Deactivate a user who is leaving the company

1. **Users** tab → **Deactivate** icon on the row (absent on your own row).
2. Confirm "Deactivate user {{name}}?".
3. The account becomes **Inactive**; it can no longer sign in, but its history stays intact.

> The last active administrator cannot be deactivated. There is no permanent deletion: deactivation is reversible (reactivate by editing the account).

### 3.7 Upload the logo and enter the contact details

1. **Company** tab → **Company information** sub-tab (administrator).
2. Click **Upload** (or **Change**) and choose a **PNG/JPG/GIF/SVG/WEBP** image of **1 MB maximum** (ideally 500 x 200 px, transparent background).
3. Fill in the 12 fields (name, address, phone, RBQ, NEQ, GST, QST, etc.).
4. Click **Save**. This information will appear on your quotes, invoices and orders.

### 3.8 Customize the document colors

1. **Appearance** tab (administrator).
2. Click a **preset theme**, or adjust the **8 colors** by hand (picker or hex code).
3. Check the result in the **live preview**.
4. Click **Save**. To return to the original theme, use **Reset to defaults**.

### 3.9 Customize the interface colors

1. **Interface** tab (each person for themselves).
2. Choose a theme or adjust the **4 colors**. Heed the contrast warning if it appears.
3. Click **Save**. The change is immediate and **specific to this browser**.

### 3.10 Configure country, currency and taxes (do this first)

1. **Jurisdiction & Currency** tab → card 1: choose **Country** and **Currency**, then **Save jurisdiction**.
2. **Taxes** tab: enter the **label** and the **rate** of each tax (for example GST 5% and QST 9.975%), then **Save**. Leave a label empty to hide a tax.
3. Do this **before** issuing your first invoices: the country can no longer change afterward.

### 3.11 Set the holdback rate

1. **Jurisdiction & Currency** tab → card 2.
2. Enter the **Default holdback rate (%)** (10% is the Quebec standard).
3. Click **Save rate**.

### 3.12 Define the fiscal year

1. **Jurisdiction & Currency** tab → card 4.
2. Choose the **Start month** and the **Start day**; check the "Fiscal year: … to …" preview.
3. Click **Save fiscal year**.

### 3.13 Enter the employer information (slip preparation)

1. **Jurisdiction & Currency** tab → **Employer payroll information** card.
2. Fill in the known numbers (CRA, Revenu Québec, CNESST, CCQ) and the legal address. Everything is optional and can be completed later.
3. Click **Save**. This data is used to produce the T4 and RL-1 slips (Time Tracking and Payroll module).

### 3.14 Manage the supplier categories

1. **Jurisdiction & Currency** tab → **Supplier categories** card.
2. Enter a category and click **Add** (duplicates are ignored); delete the ones you don't need.
3. Click **Save**. To start over, **Reset to default values**.

### 3.15 Choose the document language

1. **Preferences** tab (administrator).
2. Select **French** or **English**.
3. Click **Save**. Documents will now be generated in that language.

### 3.16 Set the time zone

1. **Timezone** tab (administrator).
2. Choose the zone (for example "Toronto/Montreal").
3. Click **Save**. The effect on timestamps is immediate.

### 3.17 Subscribe to a plan

1. **Subscription** tab. If there is no active subscription, click **Subscribe now**.
2. You are redirected to the **Stripe** payment page; pay for the plan.
3. On return, the ERP reopens the **Subscription** tab; click **Refresh** if needed to see the updated status.

### 3.18 Manage or cancel the subscription

1. **Subscription** tab → **Manage my subscription** to open the **Stripe portal** (change card, download invoices, change plan).
2. To stop: **Cancel** button → **Confirm cancellation**. Access stays active until the **end of the period** already paid, then the company switches to **consultation mode**.

### 3.19 Recharge AI credits

1. **Subscription** tab → **AI Credits** card → **Recharge**.
2. Choose a quick amount ($10 to $500) or enter a **custom amount** (between $5 and $500).
3. Click **Recharge ${{amount}}**. The card on file is charged, then the balance is credited. If the card is declined, no credit is added.

### 3.20 Connect an accounting software

1. **Integrations** tab → **QuickBooks** sub-tab (or **Sage 50** / **Sage**).
2. Start the connection (OAuth for QuickBooks and Sage Business Cloud) and follow the account mappings.
3. Check the **History** to verify the syncs. (Details in the Integrations manual.)

---

## 4. Reference

### 4.1 Configuration endpoints (`/api/erp/v1/config`)

Unless stated otherwise, **reading** (GET) is open to any user of the tenant and **writing** requires administrator rights (`require_tenant_admin_or_role`).

| Method | URL | Role | Purpose |
|---|---|---|---|
| GET | `/config/profile` | Self | Read your profile |
| PUT | `/config/profile` | Self | Change name / email |
| GET | `/config/users` | Admin | List of users |
| POST | `/config/users` | Admin | Create a user |
| PUT | `/config/users/{id}` | Admin | Edit (role, rights, email) |
| PUT | `/config/users/{id}/password` | Self **or** admin | Change a password |
| DELETE | `/config/users/{id}` | Admin | Deactivate (soft delete) |
| GET | `/config/entreprise` | Everyone | Read the configuration |
| PUT | `/config/entreprise/{cle}` | Admin | Write a key (logo, contact details, free-form keys) |
| GET / PUT / DELETE | `/config/document-theme` | GET everyone / write admin | Document theme |
| GET / PUT | `/config/tax-config` | GET everyone / PUT admin | Sales taxes |
| GET / PUT | `/config/document-language` | GET everyone / PUT admin | Language (fr/en) |
| GET / PUT | `/config/timezone` | GET everyone / PUT admin | Time zone |
| GET / PUT | `/config/country` | GET everyone / PUT admin | Country (CA/US) |
| GET / PUT | `/config/currency` | GET everyone / PUT admin | Currency (CAD/USD) |
| GET / PUT | `/config/retainage` | GET everyone / PUT admin | Holdback rate |
| GET / PUT | `/config/fiscal-year` | GET everyone / PUT admin | Fiscal year |
| GET / PUT | `/config/juridiction` | GET everyone / PUT admin | US state / CA province |
| GET / PUT | `/config/supplier-categories` | GET everyone / PUT admin | Supplier categories |
| GET / PUT | `/config/employer-payroll` | GET admin **or accountant** / PUT admin | Employer information |
| GET POST PUT DELETE | `/config/webhooks[...]` | Admin | Outbound webhooks — **no UI** (see 5.4) |

### 4.2 Subscription endpoints (`/api/erp/v1/stripe`)

All require only an authenticated session (the company is resolved from your account).

| Method | URL | Purpose |
|---|---|---|
| POST | `/stripe/checkout` | Create a subscription payment session |
| GET | `/stripe/subscription` | Subscription details (Stripe source) |
| POST | `/stripe/portal` | Open the Stripe customer portal |
| POST | `/stripe/cancel` | Schedule cancellation at end of period |
| GET | `/stripe/credits` | AI-credit balance + month's usage |
| POST | `/stripe/credits/recharge` | Recharge (real payment, $5–500) |

### 4.3 Subscription statuses

| Status | Meaning | ERP access |
|---|---|---|
| **Active** | Subscription paid and in good standing | Full |
| **Free trial** | Trial period in progress | Full |
| **Past due** | Charge failed, grace period | Full (to be settled) |
| **Cancellation pending** | Cancellation scheduled at end of period | Full until the date |
| **Canceled** | Subscription ended | **Consultation mode** |
| **Incomplete** | Initial payment not finalized | **Consultation mode** |
| **Expired** / **Unpaid** | Access ended | **Consultation mode** |

### 4.4 Limits and validations

| Rule | Value / effect |
|---|---|
| Password | **6 characters minimum** (creation and change) |
| Username | Unique within the company (otherwise refused); **cannot be changed** after creation |
| Last administrator | Can be **neither** demoted **nor** deactivated |
| Self-deactivation | Forbidden (you cannot deactivate your own account) |
| "Super-administrator" role | Refused on create/edit (422 message) |
| Logo | **1 MB maximum**; PNG, JPG, GIF, SVG, WEBP |
| Theme color | Hexadecimal format `#RGB` or `#RRGGBB` |
| Tax rate | 0 to 100, rounded to 3 decimals; label ≤ 50 characters |
| Holdback rate | 0 to 100% |
| Fiscal year | Month 1–12, valid day for the month (February 29 tolerated) |
| Country | CA or US; **change refused** if invoices/entries exist |
| Currency | CAD or USD |
| Time zone | One of the 13 allowed zones |
| Supplier categories | 80 characters per entry, 200 entries maximum |
| Credit recharge | **$5 to $500**; real payment, idempotent |
| Language | `fr` or `en` |

### 4.5 The 8 document colors (Appearance tab)

Default values = **Constructo Blue** theme (source `html_utils.py`).

| Key | Default | Role |
|---|---|---|
| `primary` | `#1F4E79` | Headers, title banner, table headers |
| `primary_dark` | `#163A5C` | Dark variant |
| `accent` | `#2563EB` | Subtitles, info-box border |
| `accent_light` | `#93C5FD` | Document number on the header |
| `header_text` | `#FFFFFF` | Text on the primary background |
| `table_row_alt` | `#F8F9FA` | Table row alternation |
| `info_bg` | `#F8FAFC` | Background of info boxes and totals |
| `border` | `#E9ECEF` | Thin rules |

### 4.6 The 4 interface colors (Interface tab)

Default values = **D365** theme (source `useUiThemeStore.ts`), saved in the browser.

| Color | Default | Role |
|---|---|---|
| Primary color | `#0078D4` | Buttons, active tab, links |
| Primary — hover | `#005EA2` | Hover |
| Sidebar | `#002050` | Sidebar background |
| Top bar | `#002B6B` | Top bar background |

### 4.7 Server defensive behaviors (good to know)

- **Reads always tolerant**: configuration GETs never crash; they return default values if something goes wrong.
- **Column not yet migrated**: some settings (taxes, language, time zone, country, currency, holdback, jurisdiction) may return "503" with the name of the migration script if the database does not yet have the column. The fiscal year and the employer information, however, **repair themselves** (automatic creation of the column or table).
- **Concurrent writes**: changes to the JSON configuration are serialized (lock), so that one save does not overwrite another.

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

| Configuration setting | Where it acts |
|---|---|
| Logo, contact details, RBQ, GST, QST | Headers of **quotes**, **invoices**, **purchase orders**, **work orders** and **emails** |
| Document theme (Appearance) | Rendering of all generated **HTML/PDF documents** |
| Taxes (GST/QST or others) | **Sales**, **Quotes**, **Accounting** (tax calculation) |
| Country / currency / jurisdiction | **Accounting** (chart of accounts, tax filing), documents |
| Holdback rate | **Accounting** (holdbacks), **Progress claims** |
| Fiscal year | **Accounting** (periods, financial statements) |
| Employer information | **Time Tracking and Payroll** (T4, RL-1, PD7A slips) |
| Supplier categories | **Suppliers / Purchasing**, **expense** breakdown |
| Time zone | **Time tracking**, creation dates, due dates, reports |
| Users and roles | The whole platform (access and permissions) |
| Subscription and AI credits | All **AI features** (Assistant, Estimation, Web, etc.) |

### 5.2 The case of webhooks

You may hear about "webhooks". Two things must be distinguished:

- **In Configuration**: there is **no active Webhooks tab**. The server does have the corresponding functions (create, test, history, with HMAC signature and hardened anti-SSRF protection), but **no interface is wired to them** on this page. As a user, disregard them.
- **In the Integrations tab**: the **Webhooks** sub-tab you see belongs to the **Integrations module** (a different system, tied to accounting). That is the one that is functional.

### 5.3 Security and best practices

- **Grant Administrator status only to trusted people**: it opens all the sensitive settings (taxes, users, subscription).
- **Set country, currency and taxes before the first invoice.** The country locks as soon as invoices or entries exist.
- **Always keep at least two administrators**: the system prevents you from removing the last one, but a second administrator avoids a lockout if one leaves.
- **Credit recharging is a real payment**: check the amount before confirming.

### 5.4 Frequently asked questions

**Q: I only see 5 tabs, not 11. Why?**
A: You are not an administrator. The Users, Appearance, Jurisdiction & Currency, Taxes, Preferences and Timezone tabs are administrator-only. Ask an administrator to check "Administrator" on your account.

**Q: I can't edit the company information.**
A: The Company tab is **read-only** for non-administrators (an amber band tells you so). Only an administrator can write there.

**Q: Can I change an employee's username?**
A: No. The username is set at creation and cannot be changed. Create a new account if necessary (and deactivate the old one).

**Q: Can I permanently delete an account?**
A: No. You **deactivate** an account (it becomes Inactive); it is never erased, to preserve history. Deactivation is reversible.

**Q: The system refuses to deactivate an administrator.**
A: It is the last active administrator. Create or promote another administrator, then try again.

**Q: I changed the interface colors on my computer, but not on my tablet.**
A: That is normal. The **Interface** colors are saved **in each browser** and do not sync. Redo the setting on the other device. (The **Document** colors, on the other hand, are shared by the whole company.)

**Q: The system won't let me change country.**
A: You already have invoices or accounting entries. The country determines the tax labels (GST/QST or Sales Tax); changing it would distort the history. This setting is done before you start invoicing.

**Q: I set a tax to 0% and it disappeared from the documents.**
A: That is intended. A **rate of 0** or an **empty label** hides the tax on new documents. Put a label and a rate back to show it again. Documents already produced keep their original taxes.

**Q: Where are the T4 and RL-1 slips?**
A: Not here. Configuration is only for entering the **employer information** (numbers, address). The slips are produced in the **Time Tracking and Payroll** module.

**Q: My credit recharge was charged but the balance didn't move right away.**
A: The system charges first, then credits. In case of a hiccup, a safety mechanism (the Stripe webhook) re-credits automatically. Refresh the AI Credits card. If in doubt, contact support.

**Q: My company switched to "consultation mode". What should I do?**
A: Your subscription is no longer active (canceled, or payment not finalized). Reading remains possible, but writes are blocked. Go to the **Subscription** tab (which stays accessible) and **subscribe again** or settle the payment.

**Q: Can I create new custom roles?**
A: No. The five roles (Administrator, User, Employee, Accountant, Manager) are fixed. What really matters for rights is the **Administrator** checkbox.

**Q: Is there an AI assistant, PDF export or printing in Configuration?**
A: No. Configuration only sets parameters. AI credits are only **viewed and recharged** here, not spent.

**Q: Can I manage webhooks from Configuration?**
A: No, there is no active Webhooks tab in Configuration. The available webhooks are in the **Webhooks** sub-tab of the **Integrations** module.

---

## 6. Summary

- **An 11-tab settings hub**: Profile, Users, Company, Appearance, Interface, Jurisdiction & Currency, Taxes, Preferences, Timezone, Subscription, Integrations.
- **Two profiles**: the **administrator** sees all 11 tabs; the **ordinary user** sees 5 (Profile, Company read-only, Interface, Subscription, Integrations).
- **The real right is the "Administrator" checkbox** (`is_admin`), not the role label. The five roles are mostly labels; only "super-administrator" is forbidden.
- **Two distinct color systems**: **Appearance** (documents, in the database, whole company, 8 colors) and **Interface** (ERP, in the browser, per user, 4 colors).
- **Company identity**: logo (≤ 1 MB) + 12 fields (RBQ, NEQ, GST, QST…) reused on all documents.
- **Tax settings**: country and currency (the country locks at the first invoice), 2 configurable taxes (0 = hidden), holdback rate, state/province, fiscal year, employer information (T4/RL-1), supplier categories.
- **Users**: creation, roles, password reset (6 characters minimum), **deactivation** (never deletion); the **last administrator** is protected.
- **Subscription (Stripe)**: subscribe, manage via the portal, cancel at end of period; **AI credits** rechargeable from $5 to $500 (real payment, idempotent).
- **Consultation mode**: without an active subscription, all Configuration writes are blocked, except re-subscription and sign-out.
- **Webhooks**: no active tab in Configuration; they live in the **Integrations** module.

---

**Verified source files**: `frontend/src/pages/ConfigurationPage.tsx` (4,108 lines, 11 tabs), `frontend/src/api/config.ts` (661 lines), `frontend/src/api/stripe.ts`, `frontend/src/store/useConfigStore.ts` (209 lines), `frontend/src/store/useStripeStore.ts` (145 lines), `frontend/src/store/useUiThemeStore.ts` (236 lines), `frontend/src/i18n/locales/{fr,en}/config.json` (441 lines), `backend/routers/config.py` (3,247 lines, 38 endpoints), `backend/routers/stripe_routes.py` (379 lines, 6 endpoints), `backend/routers/html_utils.py` (document theme), `backend/erp_auth.py` (access guards), `frontend/src/pages/IntegrationPage.tsx` (1,608 lines, 7 sub-tabs — separate module).

**Related manuals**:
- Module 10 (Employees — records linked to user accounts) — `10-operations-employes.md`
- Module 12 (Time Tracking and Payroll — T4/RL-1/PD7A slips fed by the employer information) — `12-operations-pointage.md`
- Module 14 (Accounting — taxes, holdback, country/currency, fiscal year) — `14-operations-comptabilite.md`
- Module 24 (AI Assistant — AI credits recharged here) — `24-communication-assistant-ia.md`
- Integrations manual (QuickBooks / Sage — Integrations tab) — see the Integrations module sheet
- Manuals overview — `README.md`
