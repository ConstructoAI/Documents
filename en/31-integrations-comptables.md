# Module 31 — Accounting Integrations (QuickBooks, Sage)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Access**: the module has **no** standalone entry in the sidebar. You open it either via **Configuration → "Integrations" tab** (icon `Link2`, `ConfigurationPage.tsx:286-289`), or via the direct address **`/integration`** (`App.tsx:256`, protected page).
> **Reference code**:
> - Frontend: single page `frontend/src/pages/IntegrationPage.tsx` (1,608 lines, **7 sub-tabs**), API client `frontend/src/api/integration.ts` (269 lines) and `frontend/src/api/sage.ts` (149 lines), state store `frontend/src/store/useIntegrationStore.ts` (271 lines), translations `frontend/src/i18n/locales/{fr,en}/pages.json` (`integration` section).
> - Backend: `backend/routers/integration.py` (4,073 lines — **10 endpoints**; QuickBooks Online + Sage 50 ODBC) and `backend/routers/sage.py` (2,818 lines — **7 endpoints**; Sage Business Cloud Accounting). Shared actual prefix: `/api/erp/v1`. Access guards in `backend/erp_auth.py`.
> **PostgreSQL tables (per company, created on demand)**: `integrations` (connections), `integration_sync_logs` (synchronization log), `integration_entity_map` (actual 1 local ↔ 1 external mappings, populated only by synchronization). The webhooks of the tab of the same name live elsewhere, in the `{tenant}.webhooks` / `{tenant}.webhook_deliveries` tables of the Configuration module.
> **Scope**: this module connects the ERP to an **external accounting software** in order to **export** your customers, suppliers, invoices, and payments to it, or to **import** the same data from it. Four connectors coexist in the interface, but **only one is truly active today**: **QuickBooks Online** (OAuth 2.0 protocol). **Sage 50** (desktop software, ODBC connection) only lets you test the connection; **Sage Business Cloud** (full OAuth connector) remains **inert** as long as the server is not configured; the **Webhooks** and **Mapping** tabs are, respectively, a manual notification feature and a read-only reference table. The module is reserved for company **administrators**.

*A note on the terminology used in this manual:* "endpoint" refers to an API endpoint; "company" or "tenant" refers to your account (each company has its own isolated data); "connection" refers to a saved link to an accounting software; "synchronization" (or "sync") refers to the transfer of data between the ERP and that software; "export" = from the ERP **to** the accounting software; "import" = from the accounting software **to** the ERP; "connector" refers to one of the four systems (QuickBooks, Sage 50, Sage Business Cloud, Webhooks).

---

## Table of contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step procedures](#3-step-by-step-procedures)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Purpose of the module

The **Accounting Integrations** module avoids duplicate data entry between the Constructo AI ERP and your accounting software. Concretely, it lets you:

- **Connect** your external accounting account (today **QuickBooks Online**) securely, without ever giving your password to the ERP (the connection goes through the provider's official authorization page);
- **Export** to that software your **customers**, **suppliers**, **invoices**, **purchase invoices**, and **payments** created in the ERP;
- **Import** from that software your **chart of accounts**, your **customers**, your **suppliers**, and your **invoices**;
- **Track** each transfer in a detailed **history** (date, direction, entity, status, message);
- **View** the **field mapping** (which ERP field feeds which field of the accounting software);
- **Create and test** **webhooks** (outgoing HTTP notifications to Zapier, n8n, or your own server).

Synchronization is **entirely manual**: it only triggers when you click a "Sync export" or "Sync import" button. There is **no automatic scheduling** in the interface.

### 1.2 How to access it

The module **does not appear** as a distinct entry in the sidebar (the sidebar contains only **"Configuration"**, `Sidebar.tsx:109`). Two paths lead to the module:

| Path | Detail |
|------|--------|
| **Configuration → "Integrations" tab** | Sidebar → **Configuration** → **Integrations** tab (icon `Link2`). The tab embeds the module's full page (`ConfigurationPage.tsx:286-289`). |
| **Direct address** `/integration` | Opens the same page full-screen (`App.tsx:256`). Protected page: authentication required. |

> **Important — access reserved for administrators.** The page itself opens for any logged-in user (it is a generic protected route), **but all server endpoints require administrator rights** (`require_tenant_admin_or_role`). An ordinary user who opens the page sees the interface, but the calls return an **authorization error** (403): the lists stay empty and the actions fail. See §1.3.

### 1.3 Roles and permissions

| Action | Who can do it |
|--------|---------------|
| **Open the page** | Any logged-in user (protected route) |
| **List / connect / synchronize / test / delete** a connection | Company **administrator** only |

Access control is identical on **all** endpoints of both routers (`integration.py` and `sage.py`): the `require_tenant_admin_or_role()` guard authorizes access if your account carries the **administrator** flag (`is_admin`, re-read on the server, therefore un-forgeable), **or** if your role is `admin`, **or** if you are a platform super-administrator. **No additional business role** opens the module (the "accountant" role is not enough here).

> **Read-only (view) mode.** If your company's subscription is suspended, the ERP switches to **read-only mode**: all the module's **writes** (create, modify, delete, test, connect, synchronize, choose a Sage company) are **blocked** (403). Only **reads** (connection list, history, statistics) remain possible.

### 1.4 The four connectors and their real state

This is the most important point to understand before using the module. Four connectors share the interface, but they are **not** at the same level of maturity.

| Connector | Tab | Technology | Real state |
|-----------|-----|------------|------------|
| **QuickBooks Online** | QuickBooks | OAuth 2.0 (Intuit API v3) | **Fully active**: connect, test, export **and** import. The only connector that actually synchronizes. |
| **Sage 50 (Simply Accounting)** | Sage 50 | ODBC (Pervasive/Actian driver, desktop software) | **Connect + test only.** Online synchronization **does not exist** (the server refuses it). An amber banner flags this. |
| **Sage Business Cloud** | Sage Business Cloud | OAuth 2.0 (Sage Accounting API v3.1) | **Full connector but inert** as long as the server does not have its activation keys. Likely state in production: **"not configured"**. |
| **Webhooks** | Webhooks | Outgoing HTTP POST (signed HMAC-SHA256) | **Manual creation and testing** only. Automatic triggering on business events is **not yet active**. |

> **In short:** to actually synchronize your accounting today, use **QuickBooks Online**. The other three connectors are either preparatory or awaiting a server configuration.

### 1.5 What the module does — and does NOT do

The module **does**: connect QuickBooks (OAuth), export/import accounting data, test a connection, log each transfer, display the field mapping, create and test webhooks, and track history.

The module **does NOT**:

- **No scheduling / sync frequency.** Sync is **100% manual** ("Sync export" / "Sync import" buttons). A frequency setting exists in the data, but **no control exposes it** in the interface.
- **No PDF or CSV export, no printing, no file upload.** The only function resembling an export is to **copy** the sample JSON payload template in the Webhooks tab.
- **No AI assistant.** This module uses no artificial intelligence and consumes **no AI credits**.
- **No Sage 50 synchronization.** The ODBC connection and test exist, but any transfer is **refused** by the server.
- **No Sage Business Cloud activity** as long as the server is not configured ("not configured" state).
- **No automatic webhook triggering.** They can be created and tested, but business events do not call them yet.
- **No editing of the field mapping.** It is a **fixed reference table** (not editable).
- **No QuickBooks import of purchase invoices or payments.** On import, only the chart of accounts, customers, suppliers, and sales invoices are supported (see §4.4).

### 1.6 The 7 sub-tabs

Source: `TAB_KEYS` (`IntegrationPage.tsx:44-52`). The tab bar can be driven with the mouse or the keyboard (arrows, Home, End).

| # | Tab | Icon | Role |
|---|-----|------|------|
| 1 | **Overview** | `Database` | Summary: indicators, provider cards, integration methods, Quebec tax table. |
| 2 | **QuickBooks** | `BookOpen` | **Active** connector: OAuth connection, export/import synchronization, test, delete. |
| 3 | **Sage 50** | `BookOpen` | ODBC (DSN) connection + test **only**. |
| 4 | **Sage Business Cloud** | `Link2` | Separate OAuth connector, inert without server configuration. |
| 5 | **Webhooks** | `Globe` | Creating and testing outgoing HTTP notifications. |
| 6 | **Mapping** | `ArrowRightLeft` | Field reference table (read-only). |
| 7 | **History** | `Clock` | Paginated log of all synchronizations. |

---

## 2. Interface

### 2.1 Header, tab bar, and global banners

At the top of the page: the title **"Accounting Integrations"** and the subtitle **"QuickBooks Online & Sage 50 — Data synchronization"** (`pages.json`, `integration` section).

Below the title, the **bar of 7 tabs** (see §1.6). Two **transient banners** can appear above the content:

- **Error banner** (red): error message returned by the server.
- **OAuth banner** (green on success, red on failure): confirms or reports the failure of a connection (for example "QuickBooks connected successfully!"). A **Close** button dismisses it.

A global **loading indicator** appears during loads.

**Status badges.** Throughout the module, the state of a connection or a transfer is shown by a **colored badge** with six values (colors compliant with accessibility standards, in both light and dark mode):

| Badge | Meaning |
|-------|---------|
| **Connected** | The connection is established and valid. |
| **Success** | The transfer completed successfully. |
| **Disconnected** | The connection is not established. |
| **Error** | An error occurred. |
| **Pending** | Connection started but not finalized, or transfer queued. |
| **Skipped** | Item deliberately skipped (for example, a taxed invoice whose tax code was not yet available; it goes out again on the next transfer). |

### 2.2 "Overview" tab

Summary screen, in four blocks.

**a) Four indicators (KPI cards):**

| Indicator | Content |
|-----------|---------|
| **Connections** | Total number of connections; subtext "{n} active". |
| **Total syncs** | Total number of synchronizations; subtext "{n} errors". |
| **Active webhooks** | Number of active webhooks; subtext "{n} configured". |
| **Last sync** | Date of the last synchronization, or **"None"**. |

**b) Two provider cards:**

- **QuickBooks Online** — "Integration with Intuit QuickBooks for accounting and invoicing." + status badge + last sync date.
- **Sage 50 (Simply Accounting)** — "ODBC connection via Pervasive/Actian for Sage 50 Canada." + status badge.

> **Note:** the Overview shows **only** QuickBooks and Sage 50. **Sage Business Cloud does not appear here**; it is driven only from its own tab.

**c) "Supported integration methods" card** — three informational tiles, no action:

| Method | Description | Label |
|--------|-------------|-------|
| **Zapier** | No-code connector, simple. Ideal to get started. | **Recommended** |
| **n8n** | Open-source, self-hosted workflow, free. | **Free** |
| **Direct API** | Custom Python/REST integration. | **Advanced** |

**d) "Quebec tax configuration (GST/QST)" card** — reference table (GST = federal Goods and Services Tax; QST = Quebec Sales Tax):

| Tax | Rate | Authority | Constructo field | QuickBooks |
|-----|------|-----------|------------------|------------|
| **GST** | 5% | Canada Revenue Agency | `tps` | `TxnTaxDetail.TaxLine[0]` |
| **QST** | 9.975% | Revenu Québec | `tvq` | `TxnTaxDetail.TaxLine[1]` |
| **Combined** | 14.975% | — | `montant_ttc` | `TotalAmt` |

Displayed note: "QST is calculated on the pre-tax amount only, not on pre-tax + GST."

### 2.3 "QuickBooks" tab — fully active connector

Title: **"QuickBooks Online"**.

**Connection card.** QB logo, title **"Connect your QuickBooks account"** and a security reminder (the connection uses OAuth 2.0; your QuickBooks credentials are never shared with the ERP).

- If no connection exists: **"Connect QuickBooks"** button (green). A click creates the connection, fetches the authorization address, and **redirects you to the Intuit site** to authorize access. An internal guard prevents double-clicking.

**Connection list.** Each row shows: QB logo, connection name, status badge, last sync date. Buttons **contextual to the status**:

| Situation | Available buttons |
|-----------|-------------------|
| Status **≠ connected** | **"Reconnect"** (restarts the OAuth authorization). |
| Status **= connected** | **"Sync export"** (transfers ERP → QuickBooks) + **"Sync import"** (transfers QuickBooks → ERP). |
| Always | **Test** icon (checks that the connection responds) + **Delete** icon (with confirmation "Are you sure you want to delete this connection?"). |

**Result banner** (green / red) after a test or a synchronization. Success is **strict**: it requires zero errors **and** a success status. Otherwise, a **warning** details the outcome: "Synchronization completed with errors: {n} succeeded, {n} failed.", followed by a few details.

**"Syncable data" card** (specific to QuickBooks) — four tiles:

| Data | Direction |
|------|-----------|
| **Customers / Suppliers** | Bidirectional |
| **Invoices** | Export |
| **Payments** | Export |
| **Projects** | Export (metadata) |

> **Environment.** By default, the server points to QuickBooks' **sandbox** environment (`QB_ENVIRONMENT`, `integration.py:125`), i.e., a **test** environment. Switching to **production** is a server setting (`QUICKBOOKS_ENVIRONMENT` variable). If your synchronizations do not show up in your real QuickBooks account, that is the first thing to check with the infrastructure administrator.

### 2.4 "Sage 50" tab — connection and test only

Title: **"Sage 50 (Simply Accounting)"**.

**Amber banner** (permanent): **"Online Sage 50 synchronization is not yet available. Connection and test only."**

**Card.** S50 logo, title **"Connect Sage 50"** and the explanation: "Connect Sage 50 (Simply Accounting) via ODBC. Your IT technician must first configure a DSN on the workstation where Sage 50 is installed."

- **"Add a Sage 50 connection"** button → shows a small form:

| Field | Detail |
|-------|--------|
| **Name (optional)** | Free-form label. Example: "Sage 50 Desktop". |
| **DSN (Data Source Name)** | Exact name of the ODBC data source configured in Windows. Example: "Sage50_MyCompany". Displayed help: "The exact name of the DSN configured in the Windows ODBC data sources." |

- **"Connect"** (disabled while the DSN is empty) and **"Cancel"** buttons.

**Connection list**: identical to QuickBooks', but **without** a synchronization button, **without** "Reconnect", and **without** the "Syncable data" card. Only **Test** and **Delete** are present.

> **What a DSN is.** A DSN (Data Source Name) is an ODBC connection name declared in Windows, on the **workstation where Sage 50 is installed**. It is your **IT technician** who creates it. The ERP simply uses this name to reach the Sage 50 database. The DSN field is validated to block dangerous characters (`; { } =`) and is limited to 64 characters.
>
> **Reminder:** even once the connection is created and tested, **no data can be synchronized** with Sage 50 (see §5.2). The product direction chosen by the publisher is to migrate customers toward **Sage Business Cloud**.

### 2.5 "Sage Business Cloud" tab — separate OAuth connector

Title: **"Sage Business Cloud"**. This tab works **independently** (it does not share the QuickBooks / Sage 50 tab mechanics) and shows one of **three states**:

**a) Loading** — a loading indicator while the server is queried.

**b) Not configured (likely state in production)** — amber banner: **"The Sage Business Cloud connector is not configured on the server. Contact your administrator to enable the integration."** This is the default state as long as the server does not have its activation keys (`SAGE_CLIENT_ID` / `SAGE_CLIENT_SECRET` variables). No action is possible.

**c) Configured** — card with "SBC" logo, account name, status badge, and description: "Synchronize your accounting data with Sage Business Cloud Accounting via OAuth 2.0." Depending on the situation:

| Sub-state | What you see |
|-----------|--------------|
| **Connected** | "Connected to {name}" + last sync date + **"Sync export"**, **"Sync import"**, **"Test"**, and **"Disconnect"** buttons (with confirmation "Are you sure you want to disconnect Sage Business Cloud?"). |
| **Pending, multiple companies** | "Your Sage account gives access to several companies. Choose the one to connect to Constructo AI." + a **"Sage company"** dropdown + **"Choose"** button. |
| **Not connected** | **"Connect Sage"** button → redirects to Sage's OAuth authorization. |

**Result banner** (green / red) after an action. The **authorization return** (after authorizing access on the Sage site) is handled automatically: you come back to the page, the ERP finalizes the connection and displays "Sage Business Cloud connected successfully!" or prompts you to choose the company if your account manages several.

### 2.6 "Webhooks" tab — manual creation and testing

Header: **"Webhooks"** + **"New webhook"** button.

**Blue box (information):** "Webhooks send an HTTP POST notification to your URL every time an event occurs. Use them with Zapier, n8n, or your own server… Each payload is signed with HMAC-SHA256 for integrity verification."

**Amber banner (important):** **"Note: automatic delivery of business events is not yet active. Webhooks can be created and tested ('Test' button), but events do not fire automatically yet."**

**Creation form** ("New webhook" button):

| Field | Detail |
|-------|--------|
| **Destination URL** | Address that will receive the notification. Example: "https://hooks.zapier.com/hooks/catch/…". |
| **Description** | Free-form label. Example: "Sync invoices to QuickBooks". |
| **Events** | Checkboxes grouped by category: **4 categories, 13 events** (see §4.7). |
| Buttons | **"Create"** (disabled if the URL is empty) and **"Cancel"**. |

**Empty state:** "No webhook configured" + "Create a webhook to trigger synchronizations".

**Webhook list.** Each row: expand chevron, description (or URL), **"Active" / "Inactive"** badge, URL, event chips. **Test** and **Delete** buttons (confirmation "Are you sure you want to delete this webhook?").

- **Expanded "Recent deliveries" area:** "Loading…", "No delivery", or the list of the 10 most recent deliveries (success/failure icon, event type, HTTP response code, date).

**"Webhook payload example" card:** a formatted JSON block (example of an `invoice.created` event with number, pre-tax amount, GST, QST, tax-included amount, status…) and a **copy button** (a green checkmark confirms the copy).

> **Technical note.** The Webhooks tab does **not** go through this module's routers. It relies on the ERP's **generic outgoing webhooks** (`/config/webhooks` endpoints, managed by the Configuration module). So these are **not** inbound "QuickBooks" or "Sage" webhooks: they are notifications that **your** ERP sends to a URL of **your** choice.

### 2.7 "Mapping" tab — read-only reference table

Header: **"Field mapping"**.

**Two filters:** a **provider** dropdown (**QuickBooks** / **Sage 50**) and an **entity type** dropdown (**All entities**, or Company, Invoice, Invoice line, Payment).

If **Sage 50** is selected, an amber banner reminds you: "Informational / upcoming: the Sage 50 mapping is provided for reference. Online synchronization is not yet available."

**Table**, columns: **Entity / Constructo AI field / {provider} field / Direction** (Bidirectional, Export, or Import, with an icon). The data is **hard-coded in the software** (not editable): **16 rows** for QuickBooks, **14 rows** for Sage 50. QuickBooks example: the company name (`nom`) maps to `DisplayName / CompanyName` (bidirectional), the tax-included amount (`montant_ttc`) maps to `TotalAmt` (export), the line quantity (`quantite`) maps to `SalesItemLineDetail.Qty`, etc.

> **What is this table for?** It documents **which ERP field feeds which field of the accounting software**, and in which direction. It is a comprehension aid: you do not edit it. The **actual** mappings between an ERP record and its twin in QuickBooks/Sage are created automatically by synchronization (they are not editable either).

### 2.8 "History" tab

Header: **"Synchronization history"** + "{n} entries" counter.

**Three filters:**

| Filter | Values |
|--------|--------|
| **Provider** | All providers / QuickBooks / Sage 50 |
| **Status** | All statuses / Success / Error / Pending / Skipped |
| **Entity type** | All entities / Invoices / Companies / Payments / Projects |

**Empty state:** "No synchronization history" + "Synchronizations will appear here once configured".

**Table**, columns: **Date / Provider / Direction / Entity / Status / Details** (the direction carries an export or import icon; the status is a badge; the details show the error message or a note, truncated if needed).

**Pagination:** "{from}–{to} of {total}", **"Previous"** / **"Next"** buttons, "Page {n} / {total}". By default, **25 entries per page**.

> **Note:** the history **combines both providers**. Sage Business Cloud synchronizations also appear here (provider "sage"), even though they have no dedicated button in the tab.

---

## 3. Step-by-step procedures

### 3.1 Connect QuickBooks Online

**Prerequisite:** be a company **administrator**; the server must have been configured for QuickBooks (Intuit keys).

1. Open **Configuration → Integrations** (or the `/integration` address).
2. Go to the **QuickBooks** tab.
3. Click **"Connect QuickBooks"**.
4. You are redirected to the **Intuit** site: sign in to your QuickBooks account and **authorize** access.
5. You automatically return to the page; the green banner **"QuickBooks connected successfully!"** confirms the connection. The connection appears in the list with the **connected** status.

> **On failure:** the message "Unable to start the connection. Check that QuickBooks is configured on the server." indicates that the activation keys are missing on the server side (an infrastructure administrator task). The message "QuickBooks connection failed. Try again." simply invites you to restart the authorization.

### 3.2 Synchronize with QuickBooks (export or import)

**Prerequisite:** a QuickBooks connection with the **connected** status.

1. **QuickBooks** tab → find the connected connection.
2. To **send your data from the ERP to QuickBooks**, click **"Sync export"**. To **bring QuickBooks data into the ERP**, click **"Sync import"**.
3. Wait: the transfer processes entities **in dependency order** (first the chart of accounts, then customers and suppliers, then invoices, purchase invoices, and payments).
4. Read the **result banner**: a strict success ("… successfully"), or an outcome "{n} succeeded, {n} failed".
5. Open the **History** tab for the line-by-line detail (each entity, its direction, its status, its message).

> **What is transferred.** On **export**: customers, suppliers, sales invoices, purchase invoices, payments. On **import**: chart of accounts, customers, suppliers, and sales invoices. **Purchase invoices and payments are not imported** (they are counted as errors and the whole run ends with a "partial" status — see §4.4).
>
> **No duplicate.** A record already synchronized is **never recreated**: the ERP remembers the mapping between each local record and its external twin, and it updates the existing one rather than creating a second.

### 3.3 Connect Sage 50 (ODBC) and test

**Prerequisite:** Sage 50 installed on a workstation, and an **ODBC DSN** already created by your IT technician on that workstation.

1. **Sage 50** tab → **"Add a Sage 50 connection"**.
2. Enter a **Name** (optional) and, most importantly, the exact **DSN**.
3. Click **"Connect"**.
4. Click the **Test** icon to verify that the ERP can reach the Sage 50 database.

> **Important limitation.** You will **not** be able to synchronize data with Sage 50: there is **neither** "Sync export" **nor** "Sync import" on this tab, and the server refuses any transfer. In addition, the required ODBC driver is **likely not** present on the cloud hosting (Linux): the test may return "pyodbc ODBC driver not installed". So, in practice, Sage 50 remains a **demonstration** connector.

### 3.4 Connect Sage Business Cloud

**Prerequisite:** the server must be configured for Sage (activation keys). Otherwise, the tab shows "not configured" and nothing is possible.

1. **Sage Business Cloud** tab.
2. If the screen shows "not configured", **stop**: ask your administrator to enable the integration on the server side.
3. Otherwise, click **"Connect Sage"** and **authorize** access on the Sage site.
4. Upon return:
   - If your Sage account manages **only one** company, the connection finalizes automatically (**connected** status).
   - If your account manages **several** companies, choose the right one in the **"Sage company"** dropdown, then click **"Choose"**.
5. Click **"Test"** to validate, then use **"Sync export"** / **"Sync import"** as needed.

> **Changing the Sage company** clears the previous mappings (to start cleanly on the new company). This is intentional: one ERP must not mix two Sage sets of books.

### 3.5 Create and test a webhook

1. **Webhooks** tab → **"New webhook"**.
2. Enter the **destination URL** (for example your Zapier or n8n entry point) and a **Description**.
3. Check the **events** you are interested in (Invoices, Payments, Projects, Companies).
4. Click **"Create"**.
5. On the created row, click **Test**: the ERP sends a test notification and shows the result under **"Recent deliveries"**.

> **Reminder:** only **manual testing** works. Business events (an invoice actually created, a payment actually received, etc.) do **not yet** trigger these webhooks automatically. Create and test them now, but do not expect automatic deliveries.

### 3.6 Review a transfer's history

1. **History** tab.
2. Filter as needed by **provider**, **status**, or **entity type**.
3. Browse the table (25 rows per page; **Previous** / **Next** buttons).
4. For a row in **error**, read the **Details** column: it contains the returned message (for example a total discrepancy detected by the reconciliation guard).

---

## 4. Reference

### 4.1 Endpoints — QuickBooks & Sage 50 (`integration.py`)

All prefixed with `/api/erp/v1`. All protected by the **administrator** guard (`require_tenant_admin_or_role`).

| Method | Path | Role | Line |
|--------|------|------|------|
| GET | `/integrations` | List of connections (tokens masked) | `integration.py:804` |
| POST | `/integrations` | Create a connection (**quickbooks** or **sage50** only) | `:833` |
| PUT | `/integrations/{id}` | Update (name, status, frequency, config) | `:871` |
| DELETE | `/integrations/{id}` | Delete (first revokes the token at Intuit) | `:928` |
| GET | `/integrations/quickbooks/auth-url` | Generate the OAuth authorization address | `:997` |
| POST | `/integrations/quickbooks/callback` | Finalize authorization (exchange the code for the tokens) | `:1045` |
| POST | `/integrations/{id}/test` | Test the connection (QuickBooks or Sage 50) | `:1228` |
| POST | `/integrations/{id}/sync` | Trigger synchronization (**QuickBooks only**) | `:3685` |
| GET | `/integrations/sync-history` | Paginated history | `:3943` |
| GET | `/integrations/sync-stats` | Counters (total / success / errors, by provider, by entity) | `:4013` |

> A **Sage Business Cloud** connection is **not** created here: `POST /integrations` only accepts `quickbooks` and `sage50`. The Sage Cloud connection is born from its own OAuth flow (§4.2).

### 4.2 Endpoints — Sage Business Cloud (`sage.py`)

All prefixed with `/api/erp/v1`, protected by the **administrator** guard, and **blocked (503)** as long as the server is not configured.

| Method | Path | Role | Line |
|--------|------|------|------|
| GET | `/sage/connections` | List of Sage connections (+ "configured" indicator) | `sage.py:292` |
| GET | `/sage/auth-url` | Prepare OAuth authorization (a single Sage connection per company) | `:336` |
| POST | `/sage/callback` | Finalize authorization, retrieve the Sage companies | `:408` |
| POST | `/sage/connections/{id}/business` | Choose the Sage company | `:551` |
| POST | `/sage/connections/{id}/test` | Test the connection | `:649` |
| DELETE | `/sage/connections/{id}` | Delete the connection | `:709` |
| POST | `/sage/connections/{id}/sync` | Trigger synchronization | `:2537` |

### 4.3 Statuses and badges

| Badge | Internal value | Where |
|-------|----------------|-------|
| **Connected** | `connected` | Connection established |
| **Success** | `success` | Successful transfer (history) |
| **Disconnected** | `disconnected` | Connection not established |
| **Error** | `error` | Connection or transfer error |
| **Pending** | `pending` | Connection started / transfer queued |
| **Skipped** | `skipped` | Item deliberately skipped (resumed on the next transfer) |

### 4.4 Synchronized entities and directions

**QuickBooks** — processing order (dependencies respected): chart of accounts → customers → suppliers → sales invoices → purchase invoices → customer payments → supplier payments.

| Entity | Export (ERP → QB) | Import (QB → ERP) |
|--------|:---:|:---:|
| Chart of accounts | Implicit import (one-way) | Yes |
| Customers | Yes | Yes |
| Suppliers | Yes | Yes |
| Sales invoices | Yes | Yes |
| Purchase invoices | Yes | **No** |
| Customer payments | Yes | **No** |
| Supplier payments | Yes | **No** |

> Purchase invoices and payments **are not imported**: on import, these entities are counted as errors and the operation ends with a **"partial"** status. **Credit-note** invoices and **supplier** invoices are **excluded** from the sales-invoice export.

**Sage Business Cloud** — order: chart of accounts → contacts (customers/suppliers) → sales invoices → purchase invoices. Sage Cloud synchronization is **bidirectional** on these four entities (import: chart of accounts, contacts, sales and purchase invoices; export: contacts, sales and purchase invoices; the chart of accounts is implicit import).

### 4.5 The "money path": taxes and reconciliation

The module handles real amounts; several safeguards prevent silently creating a wrong total in your books.

- **Quebec taxes.** On export, the ERP finds the matching **tax codes** in the accounting software (GST ≈ 5%, QST ≈ 9.975%, combined ≈ 14.975%). A **tolerance of 0.02** distinguishes the Quebec combined rate (14.975%) from HST (Harmonized Sales Tax, 15%). If a code matches, the invoice is sent **tax-exclusive** and the software recomputes the tax; otherwise, the ERP supplies the total tax amount.
- **Taxed invoice with no available code.** If the tax codes are not (yet) readable, the invoice is **deferred** (status **skipped**) and goes out on the next transfer, rather than being exported with an incorrect tax.
- **Reconciliation guard (post-send).** After creating an invoice or a purchase invoice in the accounting software, the ERP **compares the returned total to the ERP total**, with a tolerance of **max($0.05, 1% of the total)**. If there is a discrepancy on a taxed invoice, the event is logged **as an error** — **without** re-sending the invoice (the mapping is preserved, so **no duplicate**).
- **On import**, the ERP reconstructs the GST/QST breakdown only if the combined rate is close to 14.975%; otherwise the tax stays "unallocated". **Import never overwrites** the local name of a customer or supplier (a rename done in the ERP takes precedence).
- **Anti-duplicate.** The mapping (local ↔ external) is written **before** any send decision; combined with a uniqueness index, it guarantees that the same record is never created twice. Sage Cloud reinforces this with a **deterministic idempotency key** per record; QuickBooks relies on the mapping and the uniqueness constraint.

### 4.6 US suppliers (US context)

For a tenant configured in the United States, exporting a supplier can include its **W-9 tax number** (TIN). This number is **encrypted** and is only decrypted after **mandatory audit logging** (compliance): if the audit fails, the number is **not** transmitted. The log always masks the TIN digits.

### 4.7 Webhook events (13, in 4 categories)

| Category | Events |
|----------|--------|
| **Invoices** | Invoice created, Invoice updated, Invoice sent, Invoice paid, Invoice overdue, Invoice canceled |
| **Payments** | Payment received, Refund |
| **Projects** | Project created, Project updated, Project status changed |
| **Companies** | Company created, Company updated |

> These events are **defined** and selectable, but **do not fire automatically yet**. Only the **Test** button triggers a delivery.

### 4.8 Limits, quotas, and security

| Element | Value |
|---------|-------|
| **Synchronization rate** | **5 requests / 60 seconds** per IP address (applies to the **Sync** and **Test** buttons, both QuickBooks and Sage Cloud). Beyond that: 429. |
| **Concurrent processing** | 2 parallel synchronizations maximum (QuickBooks); same for Sage Cloud. |
| **Anti-double-sync lock** | Only one synchronization at a time **per connection**. A second attempt returns **409 "A synchronization is already in progress for this connection. Try again in a few minutes."**. A lock "stuck" for more than **30 minutes** is released automatically. |
| **History** | 25 entries per page (adjustable from 1 to 100 on the server). |
| **Anti-CSRF token validity (OAuth)** | 10 minutes. |
| **Sage 50 DSN length** | 64 characters; the characters `; { } =` are forbidden. |

**Security:**

- **Encrypted tokens.** The OAuth access tokens (QuickBooks and Sage Cloud) are encrypted (Fernet) before storage; the server **refuses** to save a token if it cannot encrypt it.
- **Anti-CSRF protection.** The OAuth authorization return is validated by a `state` token with a limited lifetime (10 min); a missing timestamp or an inconsistent clock causes the finalization to fail (fail-closed).
- **Anti-substitution.** QuickBooks: the already-linked account (realm) cannot be replaced by another on return. Sage: the chosen company must be in the authorized list returned at connection time.
- **Anti-SSRF (Sage).** Calls to the Sage API use only relative paths under the official domain; the token is never sent to an arbitrary address.
- **Signed payloads.** Each webhook is signed with HMAC-SHA256 so the recipient can verify its integrity.
- **Per-company isolation.** Each connection, log, and mapping is walled off to your company.
- **Generic messages.** The technical detail of errors is never exposed in the interface.

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

- **Accounting (module 14).** Import populates your **chart of accounts** and creates **invoices** in the ERP; export sends your invoices and payments to the accounting software. It is the module whose data flows the most through this connector.
- **Sales / Quotes and Projects (modules 08 and 09).** The **sales invoices** derived from quotes and projects are what export pushes to QuickBooks/Sage; **projects** are exported as metadata.
- **Company management (module 03).** Your **customers** and **suppliers** are the "bidirectional" entities: they go to the accounting software and come back, without ever overwriting a locally modified name.
- **Configuration (module 30).** This module **is** a tab of Configuration. Configuration also hosts the **webhooks** used by the tab of the same name, and manages the **subscription**: a suspended subscription switches the module into **read-only (view) mode** (writes blocked). The **taxes** (GST/QST) set in Configuration determine the exported breakdown.
- **Store / Purchase orders (modules 10 and 14).** The **suppliers** exported to the accounting software also come from these modules.

### 5.2 Frequently asked questions

**Where is the module in the menu?**
It has no entry of its own. Open **Configuration** (sidebar), then the **Integrations** tab. The direct address `/integration` leads to the same place.

**Why is the page empty or showing errors even though I'm logged in?**
Because the module is **reserved for administrators**. An ordinary user sees the page, but the server refuses the data (403). Request administrator rights, or have an administrator configure the integration.

**Which connector actually works?**
**QuickBooks Online**, today, is the only one that actually synchronizes (export **and** import). Sage 50 is limited to connection and test; Sage Business Cloud is ready but **inert** as long as the server is not configured; webhooks do not fire automatically yet.

**Can I schedule an automatic synchronization (every night, for example)?**
No. Synchronization is **100% manual**: it only starts when you click "Sync export" or "Sync import". No frequency setting is offered in the interface.

**My QuickBooks data doesn't show up in my real account. Why?**
The server points by default to QuickBooks' **test (sandbox)** environment. Switching to **production** is an infrastructure administrator setting (`QUICKBOOKS_ENVIRONMENT` variable).

**Will I create duplicates if I synchronize several times?**
No. The ERP remembers the mapping between each local record and its external twin: an item already transferred is **updated**, never recreated. The same transfer also cannot run twice in parallel on one connection (lock, 409 message).

**Can I synchronize Sage 50?**
No. The ODBC connection and test exist, but **any transfer is refused**. In addition, the required ODBC driver is likely not installed on the cloud hosting. The recommended path is **Sage Business Cloud**.

**The Sage Business Cloud tab shows "not configured". What should I do?**
The server does not have its Sage activation keys. This is an **infrastructure administrator** task; you cannot do anything from the interface until it is enabled.

**Do webhooks fire when I create an invoice?**
Not yet. You can **create** and **test** a webhook, but business events do **not** call it automatically for now.

**Can I modify the field mapping?**
No. It is a **fixed reference table** (16 rows for QuickBooks, 14 rows for Sage 50). The actual mappings are created by synchronization and are not editable either.

**Can I export the history to PDF or CSV, or print it?**
No. The module offers neither file export, nor printing, nor upload. The only function resembling an export is to **copy** the sample JSON template in the Webhooks tab.

**Does the module consume AI credits?**
No. This module uses no artificial intelligence and consumes no credits.

**Why does an invoice appear as "skipped" in the history?**
Most often, because the matching **tax code** was not yet available in the accounting software at the time of the transfer. The invoice is not sent with an incorrect tax; it goes out again automatically on the next transfer.

**What does a "partial" status mean?**
That some of the requested entities could not be processed — typically an **import** of QuickBooks purchase invoices or payments, which **is not supported**. Check the history for details.

### 5.3 Common troubleshooting

| Symptom | What to check |
|---------|---------------|
| Empty page / 403 errors | You are not an **administrator**. The module is reserved for administrators. |
| "Unable to start the connection. Check that QuickBooks is configured on the server." | The Intuit keys are missing on the server side (infrastructure administrator). |
| Data doesn't go to the real QuickBooks account | **Sandbox** environment active; switch to production on the server side. |
| "A synchronization is already in progress for this connection." | A sync is already running. Wait a few minutes (lock released after 30 min if stuck). |
| 429 error after several clicks | The limit of **5 requests / 60 s** is reached. Wait one minute. |
| Sage 50 test: "pyodbc ODBC driver not installed" | The ODBC driver is not present on the host (expected in the cloud). Sage 50 remains a demonstration connector. |
| Sage Business Cloud tab "not configured" | Sage keys missing on the server side (infrastructure administrator). |
| Webhook created but never called | Normal: automatic delivery is not active. Use **Test**. |
| Writes refused (read-only mode) | Suspended subscription → read-only. Bring the subscription up to date (Configuration module). |
| History error with "total discrepancy" | The reconciliation guard detected a discrepancy on a taxed invoice; check the invoice's taxes. The invoice is **not** duplicated. |

---

## 6. Summary

- **The module connects the ERP to an external accounting software** to avoid duplicate data entry (export/import of customers, suppliers, invoices, and payments). Access: **Configuration → "Integrations" tab** or the `/integration` address. **No** entry of its own in the sidebar.
- **Reserved for administrators.** An ordinary user sees the page but receives 403 errors. In **read-only (view) mode** (suspended subscription), all writes are blocked.
- **Seven tabs:** Overview, QuickBooks, Sage 50, Sage Business Cloud, Webhooks, **Mapping**, History.
- **Only one fully active connector: QuickBooks Online** (OAuth 2.0), with export **and** import. By default, it points to the **test (sandbox)** environment — switch to production on the server side.
- **Sage 50** = ODBC (DSN) connection + **test only**; no synchronization. **Sage Business Cloud** = full OAuth connector but **inert** without server configuration ("not configured" state).
- **Fully manual synchronization** ("Sync export" / "Sync import" buttons); **no scheduling**. Dependency-respecting order: chart of accounts → customers/suppliers → invoices → purchase invoices → payments.
- **On QuickBooks import**, purchase invoices and payments **are not supported** ("partial" status).
- **Money safeguards:** defer taxed invoices with no tax code ("skipped"), reconcile the total after sending (discrepancy → "error" without re-sending), mapping written before any send (**no duplicate**), anti-double-sync lock (409), limit of 5 requests / 60 s.
- **Webhooks:** creation and **manual testing** only (automatic delivery not active); they rely on the Configuration module's generic webhooks (`/config/webhooks`), signed with HMAC-SHA256.
- **Field mapping:** fixed reference table (16 rows for QuickBooks, 14 rows for Sage 50), not editable.
- **No AI, no PDF/CSV export, no printing, no upload.** Security: encrypted OAuth tokens (Fernet), anti-CSRF (`state`), anti-SSRF (Sage), strict per-company isolation.

---

*Verified source files:* `backend/routers/integration.py` (4,073 lines, 10 endpoints; QuickBooks Online + Sage 50 ODBC), `backend/routers/sage.py` (2,818 lines, 7 endpoints; Sage Business Cloud), mounted in `backend/erp_api.py` under `/api/erp/v1`; `frontend/src/pages/IntegrationPage.tsx` (1,608 lines, 7 sub-tabs), `frontend/src/api/integration.ts` (269 lines), `frontend/src/api/sage.ts` (149 lines), `frontend/src/store/useIntegrationStore.ts` (271 lines); `frontend/src/i18n/locales/fr/pages.json` (`integration` section); Webhooks tab served by `backend/routers/config.py` (`/config/webhooks`). Access guards: `backend/erp_auth.py` (`require_tenant_admin_or_role`).

*Related manuals:* `30-configuration.md` (parent module: Integrations tab, webhooks, subscription, and read-only mode), `14-operations-comptabilite.md` (chart of accounts and invoices exchanged), `07-ventes-soumissions.md` and `08-ventes-projets.md` (exported sales invoices), `03-gestion-entreprises.md` (bidirectional customers and suppliers).
