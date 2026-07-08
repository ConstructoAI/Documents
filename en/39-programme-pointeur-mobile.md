# Module 39 — Mobile Time Clock (employee time-tracking PWA)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Intended audience**: this manual is addressed to the **users** of the Mobile Time Clock — the **contractors**, **foremen** and **construction workers** who clock their hours, chat about the jobsite, review their dossiers and manage inventory from their phone. It is not addressed to developers.
> **What the "Mobile Time Clock" is**: despite its name, it is not a simple punch clock. It is a **full mobile application** (a "PWA," a web app installable on your phone) that brings a good part of the Constructo AI ERP directly into the field: **clocking your hours** with automatic geolocation and weather, jobsite **messaging** (channels and direct messages), **360° dossiers** (notes, photos, steps, extras), **business documents** (quotes, invoices, work orders, purchase orders), a **barcode scanner** for inventory, an **AI assistant**, **jobsite weather**, a **construction calculator** and more. Time tracking is only one of the six big families of features.
> **SEPARATE application (not a tab of the Web ERP)**: the Mobile Time Clock is a **standalone application** (folder `MOBILE_REACT/frontend`, React 18 + Vite + Zustand + Tailwind), **co-hosted** by the ERP and served under **`app.constructoai.ca/mobile`**. It has its **own login system** (a token separate from the ERP's): you sign in with the **company email**, then pick **your name** from the employee list, then enter your **4-digit PIN**. An ERP account does not open the mobile app, and the reverse is also true.
> **API prefix**: `/api/mobile/v1` (plus `/api/mobile/v1/attachments` for attachments).
> **Reference code (backend, `MOBILE_REACT/backend/`)**: `mobile_api.py` (5,278 lines, **118 endpoints**), `attachments_api.py` (361 lines, **7 endpoints**), `mobile_database.py` (12,822 lines — business logic and database), `mobile_auth.py` (266 lines — login, tokens, signed URLs), `mobile_models.py` (1,390 lines), `mobile_pdf_service.py` (635 lines — PDF), `mobile_attachments_service.py` (434 lines — attachments), `mobile_idempotency.py` (264 lines — **present but not enabled**, see §4.10).
> **Reference code (frontend, `MOBILE_REACT/frontend/src/`)**: `App.tsx` (**24 routes**), `components/layout/MobileLayout.tsx` (256 lines — the shell), pages `LoginPage.tsx`, `DashboardPage.tsx`, `PunchPage.tsx` (504 lines), `CrewPage.tsx`, `MessagesPage.tsx` / `ChannelChatPage.tsx` / `DmChatPage.tsx`, `DossiersPage.tsx` / `DossierDetailPage.tsx` (1,471 lines), `DocumentsHubPage.tsx` / `DocumentListPage.tsx` / `DocumentDetailPage.tsx` / `DocumentFormPage.tsx`, `AiAssistantPage.tsx`, `MeteoPage.tsx`, `ScannerPage.tsx` (1,123 lines), `ReceptionPage.tsx`, `CountingPage.tsx`, `CalculatricePage.tsx`, `RemindersPage.tsx`, `AuditLogPage.tsx`, `AppearancePage.tsx`, `PhotoUploadPage.tsx`.
> **PostgreSQL tables (database shared with the ERP)**: time-tracking, dossier, document and inventory data live in **your company's schema** (the same one as the Web ERP); some cross-cutting features use `public`-schema tables prefixed `mobile_*` (`mobile_punch_photos`, `mobile_push_subscriptions`, `mobile_auth_rate_limit`, `mobile_audit_events`, `mobile_idempotency`, `mobile_recurrent_invoices_config`, etc.). Time tracking writes to `time_entries`, stock movements to `mouvements_stock` (**the same tables as the ERP**): what you enter on site is immediately visible at the office, and vice versa.
> **Important scope**: the mobile app **shares the ERP's database**. It is therefore not a parallel system: it is a **field window onto the same data**. Permissions are controlled by **four roles** (ADMIN, MANAGER, EMPLOYE, APPRENTI) that decide what each person sees and can do.

*A note on the terminology used in this manual:* "PWA" (Progressive Web App) refers to a web application that installs and behaves like a native app on the phone; "tenant" (or "company") refers to your organization in Constructo AI; "clock in / clock out" (punch in / punch out) refers to starting or stopping the timer for your hours; "WO" refers to a **work order**; "PO" a **purchase order**; "360° Record" refers to the detailed view of a dossier that gathers all its information; "role" refers to your access level (ADMIN, MANAGER, EMPLOYE, APPRENTI); "endpoint" refers to an API endpoint; "offline" refers to operating without an Internet connection; "attachment" refers to an attached file (photo, PDF, document); "PIN" refers to your personal 4-digit code (labelled "NIP" in the French interface).

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

### 1.1 The application's mission

The Mobile Time Clock answers a simple need: **give the people on the jobsite access to the essentials of the ERP, on their phone, even without a network**. An employee arrives at a site, opens the app, picks their work order and **clocks in**; their location and the weather are recorded **automatically**. At the end of the day, they **clock out** and can, if needed, have a **supervisor** present on site **sign**. In between, they check jobsite **messages**, open the project **dossier**, take a **note photo**, ask a question to the **AI assistant** or check the **weather** for the coming days.

The typical daily journey comes down to a few gestures:

1. **Sign in** once (company email, choose your name, PIN) — the session stays open for 24 hours.
2. **Clock in** on the day's work order; **clock out** at the end of the shift.
3. **Communicate** with the team (channels, direct messages) and **review dossiers**.
4. Depending on your role, **manage inventory** (scan, receiving, counting), **produce documents** (quotes, invoices, orders) or **track overdue invoices**.

> **Keep in mind.** Time tracking is the module's historical core, but the app is really a **jobsite mini-ERP**. What you do in it (clock, move stock, create a document, add a note) is written to **the same database** as the Web ERP: no re-entry at the office.

### 1.2 A SEPARATE application (not to be confused with the Web ERP)

This is the first point to understand. The Mobile Time Clock **is not an internal page of the ERP**. It is a **distinct application**, with:

- its own address: **`app.constructoai.ca/mobile`**;
- its **own login** — an authentication token **separate** from the ERP's (being logged into one does not open the other);
- its **own** streamlined design, built for the thumb and full sunlight: a bar at the top, a navigation bar at the bottom, no desktop-style sidebar.

The old dedicated address `mobile.constructoai.ca` now automatically **redirects** to `app.constructoai.ca/mobile`.

> **Practical consequence of the move to `/mobile`.** If you had installed the app or enabled notifications at the old address, you must **reinstall** the app and **re-enable notifications** (see §2.17). The old subscriptions are no longer valid.

### 1.3 How to access it and install the app

**Open the app.** In the phone's browser, go to **`app.constructoai.ca/mobile`**. The login page appears.

**Install to the home screen (recommended).** Like any PWA, the app can be added to the home screen to open full-screen, like a real app:

- **iPhone / iPad (Safari)**: **Share** button → **"Add to Home Screen."**
- **Android (Chrome)**: **⋮** menu → **"Install app"** (or the offered banner).

Once installed, the app works **offline** for the essentials (see §1.7) and can receive **push notifications** (§2.17).

### 1.4 The three-step login

The app does **not** ask for a personal username and password like the ERP. Login happens in **three stages** (see the detail in §2.2):

1. **The company** — you enter the **company email** and its **password**. It is the same company (the same "tenant") as in the ERP.
2. **Who are you?** — the app shows the **list of employees** of the company; you **tap your name**.
3. **Your PIN** — you enter your **personal 4-digit code**. On the fourth digit, login happens **automatically**.

Once signed in, your session stays valid for **24 hours**. Your role, the company's time zone and your permissions are stored on the device.

> **Shared device, safely.** If several employees use the same phone (common on a jobsite), at login the app **clears** any pending data and cached attachments from **another** employee. Each person sees only their own data.

### 1.5 The four roles (what each can do)

Everything you see and can do depends on your **role**, set by your company. There are **four**:

| Role | For whom | Typical access |
|---|---|---|
| **ADMIN** | Owner, administrator | **Everything**: time tracking, messaging, dossiers, **all** documents (quotes / invoices / orders), inventory, AI assistant with ERP tools, **overdue reminders**, **audit log** |
| **MANAGER** | Foreman, project manager | Like ADMIN **except the audit log**; approves the team's hours, manages inventory and financial documents |
| **EMPLOYE** | Field employee (default) | Time tracking, messaging, dossiers, **work orders only** (not quotes / invoices / purchase orders), **general** AI assistant (no ERP tools), calculator, weather |
| **APPRENTI** | Apprentice | Like EMPLOYE |

In addition to the role, a **stock-management permission** ("can manage stock") can be granted individually: it unlocks the **movement scanner**, **receiving** and **counting** even for an employee who is not ADMIN/MANAGER (the ADMIN and MANAGER roles have it by default).

> **Keep in mind.** The role is **checked server-side**: the interface hides unauthorized features, but even by bypassing the display, the server refuses (403 error) any forbidden action. An employee can therefore **never** see a quote or an invoice, nor approve their own hours.

### 1.6 The six families of features

The app brings together about twenty screens, which can be grouped into **six families**:

| Family | Screens | For whom |
|---|---|---|
| **1. Time tracking & hours** | Home, Time tracking (clock in / out, history, weekly summary), Crew on site | Everyone; hours approval = foremen |
| **2. Communication** | Messages (channels + direct messages), AI Assistant, Notifications | Everyone |
| **3. Project dossiers** | Dossiers, 360° Record (14 tabs: notes, steps, extras, documents, etc.) | Everyone; financial section restricted to ADMIN/MANAGER |
| **4. Business documents** | Documents (quotes, invoices, work orders, purchase orders), Overdue reminders | Work orders = everyone; the rest = ADMIN/MANAGER |
| **5. Inventory / store** | Scanner (barcode), Receiving, Counting | Stock management (ADMIN/MANAGER or permission) |
| **6. Field tools** | Site Weather, Construction Calculator, Appearance, Audit Log | Everyone; audit log = ADMIN |

### 1.7 Offline mode (jobsite without a network)

The jobsite is often out of network range. The app is designed to **keep working without a connection**:

- your essential data (current punch, work orders, history, dossiers) is **preloaded** while you are online and stays **viewable offline**;
- when you **act** offline (clock in/out, add a note, create a document, confirm a stock cart, etc.), the action is **queued** on the phone;
- as soon as the network returns, everything **syncs automatically**; an "Offline / N pending actions" indicator and a **"Sync now"** button keep you informed.

**18 types of actions** can be deferred and then replayed (see the full list in §4.10). Two notable exceptions: inventory **receiving** and **counting** require a connection, and **editing an existing document** is disabled offline (to avoid overwriting data changed at the office in the meantime).

> **Keep in mind.** You can clock in and out **without a network**: the exact time of your gesture is recorded locally, and the punch uploads as-is once you reconnect. You do not lose your hours because the jobsite has no Wi-Fi.

### 1.8 Bilingual, light/dark theme and appearance

- **French / English** — an **FR / EN** toggle in the top bar switches the entire interface. Your choice is stored on the device.
- **Light / dark theme** — a sun / moon toggle adapts the display to conditions (full sun, night work).
- **Custom appearance** — the **Appearance** screen (§2.16) lets you choose the interface **colors** (preset themes or custom colors). This setting is **local to the device**.

---

## 2. Interface

### 2.1 The shell: navigation bars (`MobileLayout.tsx`)

Once signed in, all screens (except login) display inside a common **shell** made up of three zones.

**Top bar (at the top).** From left to right:

- a **"C" chip** and your **First Last name**, with the **company name** below;
- the **language toggle FR / EN**;
- the **notifications bell** (enable / disable push notifications, §2.17);
- the **theme toggle** (sun = switch to light, moon = switch to dark).

**Bottom navigation bar (at the bottom) — 5 tabs.** This is the heart of navigation:

| Tab | Screen | Note |
|---|---|---|
| **Home** | Dashboard | Overview of the day |
| **Time Tracking** | Clock in / out and history | The module's core |
| **Messages** | Messaging | **Red badge** = number of unread messages (refreshed every 30 seconds) |
| **Dossiers** | Dossier list | 360° Records of the projects |
| **More** | Slide-up menu | Opens the menu of the other features |

**"More" menu (slide-up sheet).** Tapping **More** slides a menu up from the bottom:

- **Crew** (foreman view);
- **AI Assistant**;
- **Site Weather**;
- **Calculator**;
- **Appearance**;
- **Invoice reminders** — *visible only to ADMIN and MANAGER*;
- **Audit Log** — *visible only to ADMIN*;
- **Log out**.

> **Where are the other screens?** Some screens (Documents, Scanner, Receiving, Counting) are neither in the bottom bar nor in the More menu: they are reached from the Home **quick-actions grid** (§2.3), depending on your role and permissions.

### 2.2 Login screen (`LoginPage.tsx`)

The login screen, outside the shell, shows a navy banner, the logo, the title **"Constructo AI"** and the subtitle **"Mobile Application."** Login unfolds in **three steps**, with the ability to go back at each one.

**Step 1 — Company.** Title **"Company login."** Two fields:

- **Company email** (example shown: `info@monentreprise.ca`);
- **Password**.

**"Continue"** button. This is the **company account** (the tenant), not a personal account.

**Step 2 — "Who are you?"** The app shows the **list of active employees** of the company: for each, an **avatar**, the **First Last name** and the **position**. You **tap your name**. If the list is empty: "No employees found."

**Step 3 — PIN.** Title **"Enter your 4-digit PIN."** Four numeric boxes with **auto-advance**; as soon as the **fourth digit is entered**, login happens on its own (no button needed). An arrow lets you go back and pick a different employee.

The footer shows the version: **"Constructo AI Mobile v1.0."**

> **PIN security.** The PIN is **verified on the server** (never in the phone). It is protected against repeated attempts: beyond **6 failed tries** for the same employee (over 5 minutes), attempts are **temporarily blocked**. A forgotten code is reset by your administrator.

### 2.3 Home / Dashboard (`DashboardPage.tsx`)

The Home screen sums up the day:

- **Greeting** — "Hello, {first name}!" and today's **long date**.
- **Punch status card** (clickable, leads to Time Tracking) — shows **"On the clock"** or **"Off the clock,"** the **elapsed time** ("For {duration}") or "No active punch," an **animated green dot** when you are clocked in, and, where applicable, the current **project** and **work order number**.
- **Two stat tiles** — **"Hours this week"** (total in hours) and **"Unread messages"** (counter).
- **Quick-actions grid** — shortcuts to: **Time Tracking, Crew, Messages** (with badge), **Dossiers, Documents, Weather, AI Assistant, Calculator, Scanner**, and — **only if you manage stock** — **Receiving** and **Counting**.

> **Note.** The **Photo Upload** (`/photo`), **Reminders**, **Audit Log**, **Appearance** and **Counting** screens do not appear in the Home grid: some are reached through the More menu (depending on the role), and `/photo` is not linked to any navigation (see §2.18).

### 2.4 Time tracking (`PunchPage.tsx`) — the module's core

The Time tracking screen shows a **clock** at the top and the **"On the clock" / "Off the clock"** state. Its content changes depending on whether you are clocked in or not.

**When you are NOT clocked in:**

- **"Work order" selector** — the list of your available work orders, each shown as `number - description | project`. If the list is empty: "No work order available."
- **Details of the selected work order** — number, project, and the jobsite's **address + city** (with a pin icon).
- **"Operation (optional)" selector** — appears if the work order has operations, to specify the task.
- **"Notes (optional)" area** — to note planned tasks, conditions, etc. (suggested example: "Planned tasks, weather conditions…").
- **Green "Clock in" button** — disabled as long as no work order is chosen.

**When you ARE clocked in:**

- a **green card** recalls the current work order, operation and project;
- the **clock-in time** ("In at {time}," in the company's time zone);
- a **live timer** (updated every second);
- a **weather badge** captured at clock-in, if available;
- a **clock-out notes** area;
- two buttons: **"Clock out"** (red) and **"Clock out + client signature."**

**Geolocation (GPS) — automatic and invisible.** At clock-in **and** clock-out, the app captures your position **in the background, silently** (with a maximum delay of 6 seconds). **There is no GPS button and no map**: you never see your coordinates. If GPS is unavailable, the server uses the work order's **jobsite address** as a fallback position.

**Weather — captured by the server.** At punch time, the server records a **weather snapshot** (via Open-Meteo) attached to your punch — first based on your GPS position, otherwise based on the jobsite address. This weather then displays as a **read-only badge** (you never enter it). This snapshot is **distinct** from the Site Weather screen (§2.12), which gives the **upcoming** forecast.

**External signature (supervisor on site).** The **"Clock out + client signature"** button opens a **"Supervisor signature"** modal meant for a **supervisor present on the jobsite** (who has no account):

- **"Signer's name"** field (at least 2 characters);
- a **touch canvas** where the supervisor signs **with a finger or a stylus**;
- a **"Clear"** button to start over;
- a reminder of the work order and the **punched duration**;
- **"Cancel"** and **"Confirm"** buttons.

> **The punch signature is that of an external supervisor, not yours.** It serves as proof of on-site presence validated in the field. It is distinct from **PIN approval** done by a foreman (§2.5).

**Built-in history ("My hours").** At the bottom of the Time tracking screen, a section with **two tabs**:

- **"History" tab** — your punches **grouped by date** (company time zone). Each card shows the work order, project, operation, clock-in / clock-out times (or "In progress") and the total. **Badges** flag the status: **"Invoiced"** (locked) or **"Approved"** (with "Approved by {name} on {date}"). As long as a punch is **neither invoiced nor approved**, you can **Edit** it (the notes only) or **Delete** it (with confirmation).
- **"Weekly summary" tab** — navigation from one **week** to another, the **week's total** (orange card if there is overtime), a **"+ {h} OT (> 40 h)"** badge, and a **bar chart by day** with an 8-hour threshold, distinguishing regular hours from overtime.

> **Overtime rule.** The summary treats as overtime any hours **beyond 8 h in a day** and **beyond 40 h in a week** (constants `OVERTIME_DAILY = 8` and `OVERTIME_WEEKLY = 40`).

### 2.5 Crew on site (`CrewPage.tsx`) — foreman view

Reached through the **More → Crew** menu, this screen gives the foreman a view of **who is present**. Title **"Crew on site,"** **"Refresh"** button.

- The list is **grouped by project** (accordion). Each project shows a **present / assigned** counter and, where applicable, an **"Approver"** badge.
- Expanding a project shows its members: **green dot** (clocked in) or **grey** (absent), the position, the **work order number** and a **live timer** for those who are clocked in.
- If you are an **approver**, an **"Approve"** button validates an employee's hours; otherwise, an **"Approved"** badge shows the status.

> **PIN approval vs external signature.** The approval done here by a foreman (with their account) is **recorded under their name**. It is different from the signature of an external supervisor taken at punch time (§2.4), which has no account.

### 2.6 Messaging (`MessagesPage.tsx`, `ChannelChatPage.tsx`, `DmChatPage.tsx`)

**Message center.** Title **"Messages,"** two tabs: **Channels** and **Direct messages**.

- **Channels tab** — the list of channels (**lock** icon if private, **#** otherwise), with name, **unread badge**, description and number of members. A floating **+** button opens the **"New channel"** modal (required **Channel name** field and **Description**). Empty: "No channel available."
- **Direct messages tab** — your private conversations (avatar, name, preview of the last message, relative time, unread badge). The **+** button starts a new conversation.

**In a channel (`ChannelChatPage.tsx`).** The conversation thread shows each message (colored avatar, name, time, "(edited)" or "Message deleted" mention). You can:

- **react** with an emoji (👍 ❤️ ✅ 😀 🎉 🙏);
- **attach photos** (image grid);
- write and **send** a message.

The input bar offers **attach a photo** / **take a photo** buttons, a text field and send. Photos are **compressed and stripped of their EXIF data** (including GPS position) **before sending**, in compliance with Law 25 (Quebec's privacy law). Messages being sent show a **"Pending"** or **"Failed"** status.

**In a direct message (`DmChatPage.tsx`).** A compose mode lets you **search for an employee** ("Search for an employee…"). Bubbles align (yours on the right), with **read receipts** (✓ / ✓✓) and photo attachments.

> **What the mobile app exposes (and not quite).** The server supports **private channels**, **per-member membership** and **discussion threads** (threaded replies). But the current mobile interface **does not offer**, when creating a channel, a choice of private/public or member selection (it sends only the name and the description), and there is **no screen to reply in a thread** within a channel (only reactions and photos). These capabilities exist behind the scenes but are not all exposed here.

### 2.7 Dossiers and 360° Record (`DossiersPage.tsx`, `DossierDetailPage.tsx`)

**Dossier list.** Title **"Dossiers,"** a **search** (title / number / project) and **status filter chips** (All + the statuses). Each card shows the number, the title, **status** and **priority** badges, the project, the **due date** and a **steps progress bar** ("{done}/{total} ({percent} %)").

**360° Record (dossier detail).** Header with title, number and status badge, then **14 scrollable tabs**:

| Tab | Content |
|---|---|
| **Summary** | Key info (project, client, owner, dates), progress, 4 indicators (Budget / Invoiced / Paid / Balance due), activity counters |
| **Quotes** | Linked quotes, with statuses and amounts |
| **Projects** | Attached projects |
| **WO** | Work orders |
| **Purchases** | Purchase requisitions |
| **Requests** | Price requests |
| **Invoices** | Linked invoices |
| **Time tracking** | Time entries of the dossier (in / out / hours, "Approved" badge) |
| **Accounting** | Revenue / billing, costs and margin (%) |
| **Steps** | Sorted steps; you change the status (To do / In progress / Done) via a drop-down menu |
| **Notes** | Notes with photos; add a note, AI enhancement and analysis, AI summary |
| **Docs** | Attachments (upload, preview, download, rename, delete) |
| **Links** | Web links (URL + description) |
| **Extras** | Extras (change orders); creation and AI assistant |

**Restricted financial section.** For an **EMPLOYE**, the dossier's financial data (quotes, invoices, purchase orders, margin) is **hidden**: they see the operational side (steps, notes, time tracking) but not the figures.

**Notes tab in detail.** An **"Add a note"** button opens a modal where you enter the text and where you can **"Enhance with AI," "Analyze photo with AI"** and **add photos** (compressed, EXIF stripped). A **"AI summary"** button produces a summary of the dossier (open points and pending actions).

**Extras tab (change orders).** An **"AI Assistant"** button lets you describe an extra in plain language: the AI **proposes**, you **confirm**. An **"Add an extra"** form asks for a **Description** and an **Amount before tax**. Each extra carries a badge (Proposed / Approved / Rejected / Invoiced). A note reminds you: **"Invoicing is done at the office (web)."**

> **Extras are created on the jobsite, but invoiced at the office.** The mobile app lets you **enter** and get an extra **approved**, but **turning it into an invoice happens in the Web ERP** (or by an ADMIN/MANAGER through the dedicated invoicing action). This is intentional: invoicing stays centralized.

### 2.8 Business documents (`DocumentsHubPage.tsx` and following)

The app embeds a real **business-document manager**, with four types: **Quotes, Invoices, Work orders, Purchase orders**.

**Document center.** For each type, a stats card (Total / Draft / In progress / Done). **Important role restriction:**

- an **EMPLOYE** sees only **"Work orders"**;
- **Quotes, Invoices and Purchase orders** are reserved for **ADMIN and MANAGER**.

A floating **"Scan a receipt"** button (managers only) opens the receipt scanner: you photograph a receipt, the **AI extracts** the supplier, the line items, the taxes and the total, and pre-fills a **purchase order** to confirm.

**List of a type (`DocumentListPage.tsx`).** Search, an **"Add a {type}"** button, **"Export to CSV,"** cards with status and client, and a context menu **Edit / Duplicate / Delete**. Indicators flag items created offline.

**Document detail (`DocumentDetailPage.tsx`).** An action bar brings together:

- **"Download PDF"**;
- **"Send by email"** (recipient, CC, subject, message);
- **"Stripe payment link"** (invoices) — generates a card-payment link;
- **"Duplicate"**;
- **"Make recurring"** (invoices) — weekly, monthly, quarterly or yearly frequency;
- **"Request signature"** (quotes and invoices) — electronic client signature.

The document's **line items** are added, edited and deleted (Description / Quantity / Unit / Price), with subtotal, **GST 5%**, **QST 9.975%** and total.

**Form (`DocumentFormPage.tsx`).** At creation / edit: Project name, Name (required), Client or Supplier, Project, Status, Priority, due / delivery dates, Description, Notes. **Offline editing is disabled** (a message flags it).

> **Professional numbering.** Each document receives a sequential number of the form `PREFIX-YYYY-NNN` (for example `DEV-2026-001`), generated safely on the server (no duplicates even under simultaneous use).

### 2.9 Barcode scanner / POS (`ScannerPage.tsx`)

The Scanner turns the phone into an **inventory terminal**. It uses the **camera** (EAN, UPC, Code128, QR formats), with a fallback to **manual entry** and a **flashlight**.

A **mode selector** (visible if you manage stock) offers:

- **Lookup** — scan a product to see its **sheet** (name, codes, **stock**, location, category, brand, price). If several products match, a disambiguation appears. An ADMIN/MANAGER can **"Link a barcode"** to an unrecognized product. A **"Stock movement"** panel allows a single-unit adjustment (In / Out / Adjustment + quantity + note).
- **In / Out / Adjustment** — **cart mode**: scan continuously, items stack up, quantities adjust (±), you add a note, then **"Confirm"** everything at once.

The cart is **offline-resistant**: confirmations are queued with a **uniqueness key** that guarantees the same confirmation is **never counted twice**, even on a retry (see §4.10). The cart survives a page refresh.

> **The three movement types.** **In** adds to stock, **Out** removes (refused if stock is insufficient), **Adjustment** sets the quantity to an **absolute value** (physical count). These movements are written to **the same table as the ERP** (`mouvements_stock`).

### 2.10 Receiving and Counting (`ReceptionPage.tsx`, `CountingPage.tsx`)

Two inventory screens, reserved for those who **manage stock**, and which **require a connection** (no offline mode).

**Receiving.** The list of **purchase orders** to receive (Sent / Confirmed / In progress). Opening an order shows its lines; **scanning an item automatically checks off** the matching line (a visual aid, not a lock). The **"Confirm receipt"** button records the **full receipt** and updates stock.

**Counting.** **Cycle-count sessions**: you scan the items, enter the **counted quantity** against the theoretical quantity, the app computes the **variances**, then you **"Close"** (adjustments are applied to stock) or **"Cancel"** the session.

### 2.11 AI Assistant (`AiAssistantPage.tsx`)

A conversational assistant reached through the **More → AI Assistant** menu. On the left, the list of your **conversations** (create, delete); on the right, the conversation thread.

- **Input** — text, **attachments** (camera or file, up to 5, compressed), **voice dictation** (Canadian French recognition) and **voice playback** of the answers.
- **Pending actions** — when the AI proposes a **write** to the database (create, edit, delete a record), it presents a card with **"Confirm"** or **"Cancel,"** and a status (Executed / Cancelled / Expired / Rejected / Failed). **Nothing is written without your confirmation.**

**Major difference by role.** For an **ADMIN or MANAGER**, the assistant has **ERP tools**: it can list and create invoices, quotes and orders, record a payment, search the database, compute taxes, propose a ledger entry, etc. For an **EMPLOYE**, the assistant stays **general**: it answers questions but **has no access to financial data or ERP writes**.

> **Assistant billing.** The AI assistant draws on your company's **prepaid AI credits** (the same as the ERP). Each exchange consumes credits based on the **actual cost** of the model's tokens (Claude Sonnet). If the company's credit balance is depleted, the assistant is temporarily unavailable (see §4.5).

### 2.12 Site Weather (`MeteoPage.tsx`)

Reached through **More → Site Weather**. A **station selector** (7 Quebec stations), the **7-day forecast** (a "Today" card plus 6 days: High / Low / Rain / Wind) and **color-coded alerts** (Frost, High wind, Rain).

An **"Jobsite impact"** section translates the weather into **concrete recommendations** — for example, in high wind: **"STOP work at height."** The alert thresholds: frost below 0 / below −10 °C, rain above 10 / 20 mm, wind above 50 / 70 km/h.

> **Two "weathers" not to be confused.** This screen gives the **forecast** (upcoming days). The **punch weather badge** (§2.4), on the other hand, is a **past snapshot** recorded at the precise moment you clocked.

### 2.13 Construction calculator (`CalculatricePage.tsx`)

A **jobsite calculator** inspired by the Construction Master Pro, with a display-style screen, two tabs **Framing** and **Conversions**, and support for the **physical keyboard**.

- **Feet-inches-fractions entry** (with presets) and **unit conversions** (m, cm, mm, ft, in, yards, kg, lb…).
- **Framing functions**: pitch, rise, run, diagonal, hip; miter, stair, arc, circle; length, width, height; board feet, studs, etc.
- **Detailed results panel** (stair: risers, treads, stringer, Blondel formula; circle; arc; polygon…).
- **History** of calculations.

### 2.14 Overdue reminders (`RemindersPage.tsx`) — ADMIN / MANAGER

Reached through **More → Invoice reminders** (visible only to ADMIN and MANAGER). The screen sorts **overdue invoices** by **aging bucket** (30, 60, 90 days, over 90 days).

A **"Send reminders"** button opens a modal: selecting the buckets, **simulation mode** (dry run, nothing sent), **test email**. On confirmation, **reminder emails** (tailored to the age, from the most courteous to the final notice) are sent, PDF attached.

### 2.15 Audit Log (`AuditLogPage.tsx`) — ADMIN only

Reached through **More → Audit Log** (ADMIN only, Law 25 compliance). It lists the recorded **events**, with **filters**:

- **Entity**: Invoices, Quotes, Work orders, Purchase orders, Authentication, Attachments;
- **Action**: Create, Edit, Delete, Login, Signature, Email sent, Payment received;
- **Dates**.

Each event keeps a **context** (IP address, device, "Before / After" values), with pagination.

### 2.16 Appearance (`AppearancePage.tsx`)

Reached through **More → Appearance**. Customizes the app's interface **colors** (setting **local to the device**):

- **preset themes**: Constructo, Forest, Brick, Anthracite, Burgundy, Ocean;
- **custom colors**: primary color, hover, top bar;
- **live preview** and a **contrast warning**;
- **Save / Cancel / Reset** buttons.

### 2.17 Push notifications (the bell)

The top bar's **bell** enables or disables **push notifications** (new message, etc.). Enabling is **opt-in**: tapping the bell requests the browser's permission, then subscribes the device. The icon reflects the state (filled bell = enabled, crossed-out = disabled).

> **Limits to know about.** Notifications are **hidden** on browsers that do not support them (notably **iOS before version 16.4**). And, because of the app's move to `/mobile` (§1.2), employees who enabled notifications at the old address must **re-enable** them here.

### 2.18 "Photo Upload" screen (`/photo`) — not wired up

The app contains a **"Photo Upload"** screen (take or choose a photo, then upload it) and its **server storage** works. **But no button or menu leads to this screen** in the current interface: it is absent from the Home grid, the More menu and the bottom bar.

**In practice**, to attach a photo on the jobsite, you therefore go through **dossier notes** (§2.7) or **messaging** (§2.6), which are fully accessible. This standalone upload screen should be considered **unavailable** as long as nothing links to it.

---

## 3. Step-by-step workflows

### 3.1 Signing in the first time

1. Open **`app.constructoai.ca/mobile`** in the phone's browser.
2. *(Recommended)* Add the app to the home screen (§1.3).
3. **Company step**: enter the company **email** and **password**, then **"Continue."**
4. **"Who are you?" step**: tap **your name** in the list.
5. **PIN step**: enter your **4-digit code** — login happens on the fourth digit.
6. You land on **Home**. Your session lasts 24 hours.

> **Forgot your PIN?** Ask your administrator to reset it. After several failed tries, wait a few minutes before trying again.

### 3.2 Clocking in and out

1. **Time Tracking** tab.
2. **Choose the day's work order** (and the **operation** if requested).
3. *(Optional)* Add a **note**.
4. Tap **"Clock in."** Your position and the weather are recorded **automatically**; the **timer** starts.
5. At the end of the shift, go back to Time tracking, add a **clock-out note** if needed, then **"Clock out."**

> **No network?** Clock anyway: the time of your gesture is recorded on the phone and will upload once you reconnect. Punches are **capped at 16 hours**: if you forget to clock out, the system automatically bounds the duration (it will not count 24 h or more).

### 3.3 Clocking out with a supervisor's signature

1. While clocked in, tap **"Clock out + client signature."**
2. Hand the phone to the **supervisor present on site**.
3. They enter their **name** (at least 2 characters) and **sign with a finger** on the canvas (**"Clear"** button to start over).
4. They tap **"Confirm."** The clock-out is recorded **with the signature** as proof of presence.

### 3.4 Correcting or deleting a punch

1. **Time Tracking** tab → **"My hours"** section → **History** tab.
2. Locate the punch. If it is **neither invoiced nor approved**, two actions are possible:
   - **Edit** — adjusts **the notes** (the hours cannot be changed here);
   - **Delete** — with confirmation.
3. An **"Invoiced"** or **"Approved"** punch is **locked**: contact a manager at the office.

### 3.5 Reviewing your hours for the week

1. **Time Tracking** tab → **"My hours"** → **Weekly summary** tab.
2. Navigate from one **week** to another.
3. Read the **total**, the **overtime** badge (beyond 40 h) and the **chart by day** (8-hour threshold).

### 3.6 Approving your team's hours (foreman)

1. **More → Crew** menu.
2. Expand the desired **project**; locate the **clocked-in** members (green dot).
3. If you are an **approver**, tap **"Approve"** to validate an employee's hours. The punch turns to **"Approved"** (under your name).

### 3.7 Writing in a channel or in a direct message

1. **Messages** tab.
2. **Channel**: **Channels** tab → open a channel → write, **react** with an emoji or **attach a photo** → send.
3. **Direct message**: **Direct messages** tab → **+** → **search for the employee** → write and send.
4. To **create a channel**: Channels tab → **+** → enter a **Name** (and a **Description**) → **Create**.

> **Photos and privacy.** Sent photos are **compressed** and **stripped of their EXIF data** (including location) before sending.

### 3.8 Reviewing a dossier and adding a note (with AI)

1. **Dossiers** tab → search / filter → open a dossier (**360° Record**).
2. Browse the **14 tabs** (Summary, Steps, Time tracking, etc.).
3. **Notes** tab → **"Add a note"** → enter the text.
4. *(Optional)* **"Enhance with AI," "Analyze photo with AI,"** or add **photos**.
5. Save. The **"AI summary"** button generates a dossier **summary** at any time.

### 3.9 Adding an extra (change order) to a dossier

1. 360° Record → **Extras** tab.
2. **"Add an extra"** → **Description** + **Amount before tax**, or use **"AI Assistant"** (describe, the AI proposes, you confirm).
3. The extra appears with a badge (Proposed / Approved…).
4. **Invoicing is done at the office (web)** — the mobile app creates the extra but does not invoice it.

### 3.10 Creating a work order (everyone) or a quote / invoice (ADMIN / MANAGER)

1. Home → **Documents** tile (or context menu depending on the screen).
2. Choose the **type** — an **EMPLOYE** sees only **Work orders**; Quotes / Invoices / Purchase orders require the ADMIN or MANAGER role.
3. **"Add a {type}"** → fill in the form (Name required, client, project, status…).
4. Open the document → **add line items** (Description / Quantity / Unit / Price); the taxes (GST 5%, QST 9.975%) are calculated automatically.

### 3.11 Sending a document by email or generating a payment link

1. Open the **document** (§3.10).
2. **"Download PDF"** to get it, or **"Send by email"** (recipient, CC, subject, message — the PDF is attached).
3. For an **invoice**: **"Stripe payment link"** creates a card-payment link to send to the client. Once paid, the invoice automatically turns to **"Paid."**

### 3.12 Scanning a product and recording a stock movement

1. Home → **Scanner** tile (if you manage stock).
2. **Lookup mode**: scan to see the product's **sheet**; if needed, make a **single-unit movement** (In / Out / Adjustment + quantity + note).
3. **In / Out / Adjustment mode** (cart): scan several items, adjust the quantities, then **"Confirm"** the cart.

> **No network?** The cart is queued with a **uniqueness key**: on reconnection, it is confirmed **once only**, never twice.

### 3.13 Receiving a purchase order

1. Home → **Receiving** tile (connection required).
2. Open the **purchase order** to receive.
3. **Scan** the received items (the lines get **checked off**).
4. **"Confirm receipt"** — stock is updated.

### 3.14 Doing an inventory count

1. Home → **Counting** tile (connection required).
2. Start a **session**; scan the items and enter the **counted quantity**.
3. Review the **variances**, then **"Close"** (applies the adjustments) or **"Cancel."**

### 3.15 Asking the AI assistant a question

1. **More → AI Assistant** menu.
2. Type (or **dictate**) your question; attach a photo if needed.
3. If the AI proposes a **write** (ADMIN / MANAGER), a card appears: **"Confirm"** or **"Cancel."** Nothing is written without your consent.

### 3.16 Enabling notifications and working offline

1. **Notifications**: tap the **bell** (top bar) and accept the browser's request. *(Unavailable on iOS before 16.4.)*
2. **Offline**: act normally; actions are queued (an "N pending actions" indicator).
3. On reconnection, everything **syncs**. You can force it with **"Sync now."**

---

## 4. Reference

### 4.1 Screens and routes (24)

All routes are served under **`app.constructoai.ca/mobile`**.

| Screen | Route | File | Access |
|---|---|---|---|
| Login | `/login` | `LoginPage.tsx` | Public |
| Home / Dashboard | `/` | `DashboardPage.tsx` | Signed in |
| Time tracking | `/pointage` | `PunchPage.tsx` | Signed in |
| Crew on site | `/equipe` | `CrewPage.tsx` | Signed in |
| Messages (hub) | `/messages` | `MessagesPage.tsx` | Signed in |
| Channel | `/messages/channel/:id` | `ChannelChatPage.tsx` | Signed in |
| Direct message | `/messages/dm/:id` | `DmChatPage.tsx` | Signed in |
| Dossiers (list) | `/dossiers` | `DossiersPage.tsx` | Signed in |
| 360° Record | `/dossiers/:id` | `DossierDetailPage.tsx` | Signed in |
| Documents (hub) | `/documents` | `DocumentsHubPage.tsx` | Work orders = everyone; rest = ADMIN/MANAGER |
| Documents (list) | `/documents/:docType` | `DocumentListPage.tsx` | Same |
| New document | `/documents/:docType/nouveau` | `DocumentFormPage.tsx` | Same |
| Document detail | `/documents/:docType/:docId` | `DocumentDetailPage.tsx` | Same |
| Edit document | `/documents/:docType/:docId/modifier` | `DocumentFormPage.tsx` | Same |
| AI Assistant | `/assistant` | `AiAssistantPage.tsx` | Signed in (ERP tools = ADMIN/MANAGER) |
| Site Weather | `/meteo` | `MeteoPage.tsx` | Signed in |
| Photo Upload | `/photo` | `PhotoUploadPage.tsx` | **Not wired up** (see §2.18) |
| Scanner | `/scanner` | `ScannerPage.tsx` | Stock management |
| Receiving | `/reception` | `ReceptionPage.tsx` | Stock management (online) |
| Counting | `/comptage` | `CountingPage.tsx` | Stock management (online) |
| Calculator | `/calculatrice` | `CalculatricePage.tsx` | Signed in |
| Invoice reminders | `/reminders` | `RemindersPage.tsx` | ADMIN / MANAGER |
| Audit Log | `/audit` | `AuditLogPage.tsx` | ADMIN |
| Appearance | `/apparence` | `AppearancePage.tsx` | Signed in |

*(Any other address redirects to Home.)*

### 4.2 Roles and permissions

| Feature | ADMIN | MANAGER | EMPLOYE | APPRENTI |
|---|:---:|:---:|:---:|:---:|
| Clock, review your hours | ✅ | ✅ | ✅ | ✅ |
| Approve the team's hours | ✅ | ✅ | per approver | per approver |
| Messaging (channels, direct messages) | ✅ | ✅ | ✅ | ✅ |
| Dossiers — operational section | ✅ | ✅ | ✅ | ✅ |
| Dossiers — financial section | ✅ | ✅ | ❌ | ❌ |
| Documents — **Work orders** | ✅ | ✅ | ✅ | ✅ |
| Documents — **Quotes / Invoices / Purchase orders** | ✅ | ✅ | ❌ | ❌ |
| Scan a receipt (→ purchase order) | ✅ | ✅ | ❌ | ❌ |
| Inventory (scanner, receiving, counting) | ✅ | ✅ | per stock permission | per stock permission |
| AI Assistant — **general** | ✅ | ✅ | ✅ | ✅ |
| AI Assistant — **ERP tools (money)** | ✅ | ✅ | ❌ | ❌ |
| Overdue reminders | ✅ | ✅ | ❌ | ❌ |
| Recurring invoices — trigger | ✅ | ❌ | ❌ | ❌ |
| Audit log | ✅ | ❌ | ❌ | ❌ |

> The **"can manage stock"** permission can be granted individually to an EMPLOYE / APPRENTI to unlock inventory, without giving them the MANAGER role.

### 4.3 API endpoints (main ones)

All prefixed with **`/api/mobile/v1`** (and `/api/mobile/v1/attachments` for attachments). The app has **118 endpoints** in `mobile_api.py` and **7** in `attachments_api.py`. Here are the main ones, by domain.

**Login and session**

| Method | Path | Role |
|---|---|---|
| POST | `/auth/tenant` | Step 1: company email + password → list of employees |
| POST | `/auth/pin` | Step 2: employee + PIN → session token (24 h) |
| GET | `/me` | Refreshes role, time zone and stock permission |
| POST | `/auth/signed-url` | Generates a temporary signed URL (images, attachments) |

**Time tracking**

| Method | Path | Role |
|---|---|---|
| GET | `/punch/status` | Active punch and elapsed time |
| POST | `/punch/in` | Clock in (work order, operation, notes, GPS) |
| POST | `/punch/out` | Clock out |
| GET | `/history` | Punch history |
| GET | `/weekly-summary` | Weekly summary (overtime) |
| PUT / DELETE | `/time-entries/{id}` | Edit notes / delete (if not approved) |
| GET | `/crew` | Crew-on-site view |
| POST | `/punch/{id}/approve` | Approve a punch |
| POST | `/punch/{id}/signature-externe` | External supervisor signature |
| GET | `/work-orders` | Available work orders |

**Messaging**

| Method | Path | Role |
|---|---|---|
| GET / POST | `/channels` | List / create a channel |
| GET / POST | `/channels/{id}/messages` | Read / write in a channel |
| POST | `/messages/{id}/reactions` | React (emoji) |
| GET / POST | `/dm/...` | Direct messages (conversations, sending) |
| GET | `/messaging/unread` | Unread counter |

**Dossiers and extras**

| Method | Path | Role |
|---|---|---|
| GET | `/dossiers`, `/dossiers/{id}` | List / 360° Record |
| POST | `/dossiers/{id}/notes` | Add a note (text + photos) |
| PATCH | `/dossiers/{id}/etapes/{id}` | Change a step's status |
| GET/POST/PUT/DELETE | `/dossiers/{id}/liens` | Web links |
| GET/POST/PUT/DELETE | `/dossiers/{id}/extras` | Extras (change orders) |
| POST | `/dossiers/{id}/extras/{id}/facturer` | Invoice an extra (ADMIN/MANAGER) |

**Documents**

| Method | Path | Role |
|---|---|---|
| GET | `/documents/{type}`, `/documents/{type}/{id}` | List / detail |
| POST | `/documents/{type}` | Create |
| POST | `/documents/{type}/{id}/pdf` | Generate the PDF |
| POST | `/documents/{type}/{id}/email` | Send by email |
| GET | `/documents/{type}/export.csv` | CSV export |
| GET/POST | `/documents/{type}/{id}/signature` | Electronic signature (quotes / invoices) |
| POST | `/documents/factures/{id}/payment-link` | Stripe payment link |
| GET | `/factures/overdue` | Overdue invoices (ADMIN/MANAGER) |
| POST | `/factures/send-reminders` | Send reminders |
| POST | `/factures/{id}/recurrent` | Make an invoice recurring |

**Inventory**

| Method | Path | Role |
|---|---|---|
| GET | `/products/scan`, `/products/search` | Search for a product |
| POST | `/stock-movements`, `/stock-movements/bulk` | Single-unit / cart movement (idempotent) |
| GET / POST | `/purchase-orders`, `/purchase-orders/{id}/receive` | Receiving |
| POST | `/inventory-counts` (+ `/lines`, `/close`, `/cancel`) | Counting |

**AI Assistant, weather, notifications, audit**

| Method | Path | Role |
|---|---|---|
| POST | `/ai/chat` | One exchange with the assistant |
| POST | `/ai/pending-actions/{id}/confirm` / `/cancel` | Confirm / cancel a proposed write |
| GET | `/ai/quota` | AI credit balance |
| GET | `/weather/stations`, `/weather/forecast` | Weather |
| GET/POST | `/push/vapid-public-key`, `/push/subscribe`, `/push/unsubscribe` | Notifications |
| GET | `/audit/events` | Audit log (ADMIN) |
| GET | `/health` | Service health |

### 4.4 Time tracking — calculations and bounds

| Item | Rule |
|---|---|
| Only one open punch at a time | A server lock prevents two simultaneous clock-ins (protects payroll) |
| Recorded time | The time of the **gesture** (tap), even offline, in the company's time zone |
| Clock tolerance (offline) | Up to **15 min in the future** / **7 days in the past**; beyond that, the server time is used |
| Maximum punch duration | **16 hours** (automatic cap if clock-out is forgotten) |
| Hourly rate | Employee's annual salary ÷ 2080 hours |
| Overtime — daily | Beyond **8 h** |
| Overtime — weekly | Beyond **40 h** |
| GPS | Captured automatically on clock-in **and** clock-out (max 6 s delay); **invisible**; fallback = jobsite address |
| Weather | Snapshot recorded by the server at punch time (read-only) |

### 4.5 AI assistant billing

| Item | Value |
|---|---|
| Credit source | Company's **prepaid AI credits** (`ai_prepaid_credits`), shared with the ERP |
| AI model (text + tools) | Claude Sonnet — **$3 / million** input tokens, **$15 / million** output tokens |
| AI model (image analysis) | Claude Opus (vision) |
| What is charged | The **actual cost** of each exchange's tokens |
| Markup | **None** (the raw cost is charged, with no added margin) |
| Balance depleted | The assistant becomes unavailable (quota error) until topped up |
| Write confirmation | Any write action (create / edit / delete) requires a **confirmation**; it **expires** after 30 minutes |

> **Pricing-policy note.** Unlike other AI modules in the ecosystem (which apply a markup), the mobile assistant charges the **raw** token cost. This is a configuration point that may change; refer to the balance shown by **`/ai/quota`**.

### 4.6 Taxes and document numbering

| Item | Value |
|---|---|
| GST | **5%** |
| QST | **9.975%** |
| Numbering | `PREFIX-YYYY-NNN` (e.g. `DEV-2026-001`), sequential and safe |
| Document types | `devis` · `factures` · `bons-travail` · `bons-commande` |
| CSV export | UTF-8 with `;` separator (Excel Quebec compatible), up to 5000 rows |
| PDF | Letter format, with the company's branding |
| Payment link | Stripe, in **CAD**, tax-inclusive amount |

### 4.7 Limits and bounds (files, sizes)

| Item | Limit |
|---|---|
| Session duration | **24 hours** |
| PIN | **4 digits**; max 6 failed attempts per employee (5 min) |
| Punch photo (`/photo`) | **5 MB**, JPEG / PNG / GIF / WebP |
| Photos per dossier note | **10 photos**, 5 MB each |
| Photos per message | Compression + EXIF stripping before sending |
| Document attachments | **10 MB**, JPEG / PNG / WebP / HEIC / PDF / DOCX / XLSX |
| AI attachments | 5 max; ~5 MB per image, ~32 MB per PDF |
| Receipt OCR | **10 MB** (HEIC accepted) |
| AI Assistant — max tokens | 32,000 per response |
| 3D files in a render | *(not applicable: see module 34)* |

### 4.8 Statuses

| Domain | Values |
|---|---|
| **Punch** | active (in progress) · completed · **approved** (validated) · **invoiced** (locked) |
| **Dossier step** | To do · In progress · Done |
| **Extra (change order)** | Proposed · Approved · Rejected · Invoiced |
| **Document** | Draft · In progress · Done (+ Paid for an invoice) |
| **Purchase order (receiving)** | Sent · Confirmed · In progress · Received |
| **Count session** | In progress · Closed · Cancelled |
| **Pending AI action** | Pending · Executed · Cancelled · Expired · Rejected · Failed |

### 4.9 Security and privacy

- **Own login** — token separate from the ERP, valid for 24 hours; the PIN is verified server-side, protected against repeated attempts.
- **Permissions checked server-side** — the interface hides, but the server **refuses** any unauthorized action (403 error). An employee can never see a quote / an invoice nor approve their own hours.
- **Per-company isolation** — each company sees only its own data (another company's photos and attachments are inaccessible).
- **Law 25** — photos are **stripped of their EXIF data** (including position) before sending; an **audit log** (ADMIN) traces sensitive actions (create, edit, delete, login, signature, email sent, payment received).
- **Anti-abuse limits** — calls to the AI and the OCR are capped (for example 20 AI-assistant exchanges per minute).

### 4.10 Offline mode — the 18 deferrable actions

The following actions can be **queued** offline, then **replayed** on reconnection:

1. Clock in · 2. Clock out · 3. Approve a punch · 4. Edit a punch · 5. Delete a punch · 6. Create a document · 7. Edit a document · 8. Delete a document · 9. Create a document line · 10. Edit a line · 11. Delete a line · 12. Add a project note · 13. Upload a photo · 14. Upload an attachment · 15. Send a message · 16. Mark a message read · 17. Update a dossier step · 18. Confirm a stock cart.

**Idempotency (no duplicates).**

- The **stock cart** has a real **uniqueness key**: even replayed several times, it is counted **only once**.
- A **general** idempotency mechanism (`Idempotency-Key` header) exists in the code (`mobile_idempotency.py`) **but is not enabled** in production to date. In other words, only the stock-cart confirmation currently enjoys guaranteed server-side duplicate protection.

> **What is NOT available offline**: inventory **receiving** and **counting** (connection required), and **editing an existing document** (disabled to avoid overwriting data changed at the office).

### 4.11 Pitfalls and things to know

- **The "Photo Upload" screen (`/photo`) is not linked to anything**: use **dossier notes** or **messaging** to attach photos (§2.18).
- **No GPS button, no map**: the position is captured **silently** at punch time; you never see it.
- **The punch weather is a past snapshot** (read-only), not to be confused with the **forecast** on the Site Weather screen.
- **The punch signature is that of an external supervisor**, not yours.
- **An EMPLOYE sees only Work orders** in Documents; Quotes / Invoices / Purchase orders + the receipt scanner = ADMIN/MANAGER.
- **Extras are created on the jobsite but invoiced at the office (web).**
- **Simplified channel creation**: no private/public choice or member selection at creation; **no threaded replies** (threads) in the interface (reactions and photos only).
- **Receiving / Counting require a connection**; no offline mode.
- **Notifications** unavailable on iOS before 16.4, and to be **re-enabled** after the move to `/mobile`.
- **Recurring invoices**: there is **no automatic scheduling**; their generation is **triggered manually** (by an ADMIN).
- **General idempotency not active**: apart from the stock cart, avoid knowingly resending the same action twice.

---

## 5. Integrations and FAQ

### 5.1 Links with the rest of Constructo AI

- **Web ERP (shared database).** The mobile app **shares the ERP's database**. Time tracking writes to **`time_entries`**, stock movements to **`mouvements_stock`** — **the same tables** as the ERP. What you enter on site appears at the office, and vice versa, in real time.
- **Time Tracking module (13) and Work Orders (12).** Mobile time tracking feeds the ERP's **Time Tracking** module; the work orders you clock against come from the **Work Orders** module.
- **Dossiers module (07).** The mobile 360° Records are the field version of the ERP's **dossiers** (notes, steps, extras, documents).
- **Store module (10) and Purchase Orders (14).** The scanner, receiving and counting act on the **same inventory** as the store; receiving closes out **purchase orders**.
- **Accounting module (15).** The invoices created, sent and paid on mobile are the **same** as those in accounting; Stripe payments are reflected there.
- **AI Assistant (25) and AI credits.** The mobile assistant draws on the **company's AI credits**, like the ERP's assistant.
- **Site Weather (16).** The mobile weather screen reuses the Open-Meteo **forecast** from the Weather module.
- **Stripe.** Used for invoice **payment links** (client card payment). The Stripe key is server-side, never exposed.
- **Emails (SMTP).** Used to **send documents** (PDF attached) and for **overdue reminders**.

### 5.2 What is NOT possible

- **No login with personal email / password**: login is by company + choice of employee + **PIN**.
- **No display of geolocation**: no screen shows your coordinates or a map.
- **No weather entry**: it is captured automatically.
- **No invoicing of extras on mobile**: invoicing is done at the office.
- **No threaded replies or fine channel-member management** in the mobile interface.
- **No offline receiving / counting**, nor **offline document editing**.
- **No automatic scheduling** of recurring invoices (manual trigger).
- **Standalone photo-upload screen not accessible** (route `/photo` not linked).
- **Quotes / Invoices / Purchase orders invisible to an EMPLOYE.**

### 5.3 Frequently asked questions

**Is it the same app as the ERP?**
No. It is a **separate mobile app**, served under `app.constructoai.ca/mobile`, with its own login. It does, however, share the ERP's **database**: the data is common.

**How do I sign in?**
In three steps: **company email + password**, then **your name** in the list, then your **4-digit PIN**. No personal account to create.

**I forgot my PIN.**
Ask your administrator to reset it. After several failed tries, wait a few minutes.

**Can I clock without a network?**
Yes. The time of your gesture is recorded locally and uploads on reconnection. A punch with no clock-out is **capped at 16 hours**.

**Can my boss see where I am?**
Your position is captured at punch time to attach your hours to the right jobsite, but **it is not displayed anywhere** in the app. Photos, for their part, are **stripped of their location** before sending.

**Why don't I see quotes or invoices?**
Because you have the **EMPLOYE** role: only **work orders** are visible to you. Financial documents are reserved for ADMIN and MANAGER.

**How do I attach a photo to a project?**
Through a **dossier note** (360° Record → Notes → add photos) or through **messaging**. The standalone photo-upload screen is not accessible right now.

**I'm a foreman: how do I approve hours?**
**More → Crew** menu, expand the project, then **"Approve"** for each employee (if you are an approver).

**Does the AI assistant cost anything?**
It consumes the **company's AI credits**, at the **actual cost** of the tokens. If the balance is depleted, the assistant becomes unavailable until topped up.

**Notifications won't enable.**
On iPhone, you need **iOS 16.4 or newer**. If you had enabled notifications at the old address, **re-enable them** here (tap the bell).

**How does a client pay an invoice?**
Open the invoice → **"Stripe payment link"** → send the link. Once paid, the invoice turns to **"Paid"** automatically.

**Can I invoice an extra from my phone?**
You can **create** it and get it **approved**, but **invoicing is done at the office (web)**.

**Is the app bilingual?**
Yes. The **FR / EN** toggle (top bar) switches the entire interface. Your choice is stored.

### 5.4 Common troubleshooting

| Symptom | Lead |
|---|---|
| Cannot clock in | Choose a **work order** first; the button is greyed out otherwise |
| "Already clocked in" when trying to clock in | A punch is already open: clock **out** first |
| The PIN is refused | Check the code; after several tries, **wait** a few minutes |
| My hours don't upload | You were **offline**: reconnect, the action syncs (**"Sync now"** button) |
| I can't edit a punch | It is **approved** or **invoiced** (locked); see with the office |
| The "Reminders" / "Audit Log" button is missing | Reserved respectively for **ADMIN/MANAGER** and **ADMIN** |
| The Receiving / Counting tiles don't appear | You do not have the **stock permission** |
| Receiving / Counting refused offline | These screens **require a connection** |
| Cannot edit a document | **Offline editing is disabled**; reconnect |
| Notifications don't work | iOS < 16.4 not supported; otherwise **re-enable** the bell |
| The AI assistant replies "unavailable" | The company's **AI credit balance** may be depleted |
| I can't find the "photo" screen | Normal: use **dossier notes** or **messaging** |

---

## 6. Summary

- **The Mobile Time Clock is a standalone application** (an installable PWA), served under **`app.constructoai.ca/mobile`**, with its **own login** (company email → choice of employee → **4-digit PIN**, 24-hour session). It is not a tab of the Web ERP, but it **shares the same database**.
- **It is a jobsite mini-ERP**, not just a punch clock: **time tracking** (automatic GPS and weather, supervisor signature, history and weekly summary), **messaging** (channels and direct messages), **360° dossiers** (notes, steps, extras), **documents** (quotes, invoices, work orders, purchase orders, PDF, email, Stripe payment, reminders), **inventory** (scanner, receiving, counting) and **tools** (AI assistant, weather, calculator).
- **Four roles** (ADMIN, MANAGER, EMPLOYE, APPRENTI) decide what each person sees and does, **checked server-side**. An EMPLOYE sees only **work orders** on the documents side and has **no** ERP tools in the assistant; the sensitive features (reminders, audit log, recurring invoices) are reserved for managers.
- **Time tracking** records the time of your gesture (even offline), captures the **position** and the **weather** in the background, caps at **16 h**, and computes overtime **beyond 8 h/day and 40 h/week**.
- **Offline mode** queues **18 types of actions** then syncs them; only **receiving**, **counting** and **document editing** require a connection. Only the **stock cart** is protected against duplicates (general idempotency is present in the code but **not enabled**).
- **Points to watch**: the **photo-upload screen `/photo` is not linked to any navigation** (use notes or messaging); **channel creation** and **discussion threads** are only partially exposed; **extras are invoiced at the office**; **notifications** require iOS ≥ 16.4 and a **re-enable** after the move to `/mobile`.
- **Technical volume**: backend `mobile_api.py` (**5,278 lines, 118 endpoints**) + `attachments_api.py` (**7 endpoints**) + `mobile_database.py` (12,822 lines), under **`/api/mobile/v1`**; frontend `MOBILE_REACT/frontend` (**24 routes**), co-hosted by the ERP.

---

*Verified source files:* backend (`MOBILE_REACT/backend/`) — `mobile_api.py` (5,278 lines; 118 route decorators; time tracking `punch_in`/`punch_out`, thresholds `OVERTIME_DAILY = 8` / `OVERTIME_WEEKLY = 40` at `mobile_api.py:1295-1296`; document types `VALID_DOC_TYPES` at `:3281`), `attachments_api.py` (361 lines, 7 endpoints), `mobile_database.py` (12,822 lines; roles `VALID_ROLES_MOBILE` at `:229`; 16 h punch cap at `:3018`; Sonnet AI rates $3/$15 and `balance_usd` charge with no markup at `:8213-8214` and `:8318`), `mobile_auth.py` (266 lines; 24 h JWT, signed URLs), `mobile_models.py` (1,390 lines), `mobile_pdf_service.py` (635 lines), `mobile_attachments_service.py` (434 lines), `mobile_idempotency.py` (264 lines; `install(app)` defined at `:260` but **never called** = dormant middleware). Frontend (`MOBILE_REACT/frontend/src/`) — `App.tsx` (24 routes), `components/layout/MobileLayout.tsx` (256 lines; 5 tabs + More menu, guards `canAccessReminders`/`canAccessAudit` at `:47-49`), `pages/PunchPage.tsx` (504 lines), `pages/DossierDetailPage.tsx` (1,471 lines), `pages/ScannerPage.tsx` (1,123 lines), and the Login / Dashboard / Crew / Messages / Dossiers / Documents / AiAssistant / Meteo / Reception / Counting / Calculatrice / Reminders / AuditLog / Appearance / PhotoUpload pages. Co-host mounting by the ERP: `ERP_REACT/backend/erp_api.py:1339-1379` (mounted routers + SPA under `/mobile`). Orphan `/photo` route confirmed by search (no `navigate('/photo')` or `to="/photo"` outside `App.tsx:110`).

*Related manuals:* `13-operations-pointage.md` (time tracking on the ERP side, which shares `time_entries`), `12-operations-bons-de-travail.md` (the work orders you clock against), `07-ventes-dossiers.md` (the dossiers / 360° Record), `10-operations-magasin.md` and `14-operations-bons-de-commande.md` (the inventory, receiving and purchase orders), `15-operations-comptabilite.md` (invoices and payments), `25-communication-assistant-ia.md` (the AI assistant), `24-communication-messagerie.md` (messaging), `16-terrain-meteo-chantier.md` (weather), `26-outils-calculateurs.md` (the calculator).

*Constructo AI ERP Manual — Module 39 "Mobile Time Clock (employee time-tracking PWA)" — v1.0 verified — 2026-07.*
