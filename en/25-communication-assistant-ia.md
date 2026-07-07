# Module 25 — AI Assistant (Claude, expert profiles)

> **Version**: 3.0 (line-by-line overhaul verified against the source code, July 2026)
> **Access**: the ERP's **top bar** → "sparkle" icon button (`Sparkles`) labeled **"AI Assistant"**. **This module is NOT in the left sidebar** (see §1.2). Direct address: `/assistant-ia`.
> **Reference code (backend)**: `backend/routers/ai.py` (3,541 lines, 12 endpoints; the ERP's central AI engine); `backend/routers/ai_profiles.py` (804 lines, 8 endpoints; serves **AI Estimation** and **PDF Takeoff**, not this module — see §1.7); `backend/routers/data_import.py` (684 lines; client import)
> **Reference code (frontend)**: `frontend/src/pages/AssistantIAPage.tsx` (716 lines); `frontend/src/components/ai/ClientImportModal.tsx` (243 lines); `frontend/src/api/ai.ts` (175 lines) + `frontend/src/api/dataImport.ts` (76 lines)
> **Labels**: `i18n/locales/fr/ai.json` (126 lines) + `i18n/locales/en/ai.json`
> **Shared PostgreSQL tables (`public`)**: `ai_prepaid_credits`, `ai_usage_tracking`, `ai_credit_applied_invoices`, `ai_credit_ledger`
> **Per-company (tenant) PostgreSQL tables**: `conversations`, `conversation_messages`, `ai_profiles`, `ai_profile_documents`, `companies` (import target)
> **Scope**: a conversational assistant powered by **Claude (Anthropic)**, **connected to your company's data**. It **reads** your database (projects, invoices, employees, quotes, inventory, etc.) and can also **write** to it (create, modify, delete records) by means of two SQL tools. It **analyzes a document**, **analyzes plans** (image reading / vision) and drives a **client-import assistant** from a CSV or Excel file. **This module is not designed to produce cost estimates**: for that, use the **Quotes → AI Estimation** module (see §1.5 and §5.1).

*Note on the terminology used in this manual:* "endpoint" refers to an application-programming-interface endpoint; "tenant" refers to your company (each company has its own isolated data); "token" is the AI's billing unit (one word is worth roughly 1.3 tokens); "prepaid credits" refers to the dollar balance your company draws down on each call to the AI.

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

The AI Assistant is a **conversational agent** built into the ERP. You write it a question or an instruction in English (or French), and it answers by drawing on **your company's real data**. Concretely, it can:

- **Answer questions about your data**: "How many projects are in progress?", "Which invoices exceed $5,000 and are unpaid?", "Which employee has the most hours this month?". It queries the database using the **`recherche_bd`** tool (read-only) and phrases an answer in plain language.
- **Create or modify records** on request: "Create a project note…", "Mark this work order as done…". It uses the **`executer_action`** tool (direct write). *This is the ERP's only assistant with write access to your database — read the caution in §1.3 and §3.2 carefully.*
- **Give expert construction advice**: technical tips, standards reminders, leads for solving a jobsite problem. The assistant automatically adapts its tone and expertise to the subject detected in your question (electrical, plumbing, structure, roofing, insulation, safety, legal matters, accounting, etc.), without you having to choose anything (see §4.3).
- **Analyze a document** you provide: contract, received quote, spec sheet, spreadsheet. One file at a time (see §2.6).
- **Analyze plans** (up to ten image or PDF files) by image reading: identifying elements, estimating areas, required trades (see §2.7).
- **Import a client list** from a CSV or Excel file exported from another program, with column matching proposed by the AI (see §2.8 and §3.6).
- **Track your consumption**: credit balance, cost over the last 30 days, usage by feature (see §2.9).

The assistant keeps **the history of your conversations**: you can reopen them, continue them, or delete them.

### 1.2 How to access it

- **From the ERP's top bar** (the one that stays visible at the top no matter which module is open): click the **"AI Assistant"** button, recognizable by its purple sparkle icon (`Sparkles`). This button is defined in `TopBar.tsx` (lines 312-320) and takes you to the `/assistant-ia` page.
- **By direct address**: `/assistant-ia`.

> **Important — this module is NOT in the left sidebar.** Unlike most modules (Sales, Accounting, Store, etc.), the AI Assistant does not appear in the side menu. You open it only through the "sparkle" button in the top bar (or via the address). If you look for it in the left bar, you will not find it there: that is normal.

The page is **protected**: you must be logged in to the ERP.

### 1.3 Roles and permissions

- **Opening the page and chatting**: available to **any logged-in user** of the company. There is **no role check** (neither administrator, nor accountant, nor super-administrator) on this module's endpoints; they all rely solely on authentication.
- **Caution (security).** Since there is no role guard, **any logged-in employee** can ask the assistant to **modify or delete data** (`executer_action` tool), without going through the usual screens or an administrator's approval. Take this into account when assigning accounts. The only technical safeguards are: refusal of destructive keywords, a mandatory `WHERE` clause on any modification or deletion (it is impossible to "erase everything" at once), a maximum 10-second limit per operation, and a server-side audit log (see §4.3).
- **Credits.** The module's real "lock" is **financial**, not role-based: if the company's prepaid-credit balance is exhausted, the assistant stops answering until it is topped up (see §1.6 and §4.5). The platform super-administrator and a few internal accounts are exempt from billing (unlimited usage).
- **Consultation mode (read-only).** When the company's subscription is suspended, the ERP switches to consultation mode and blocks writes globally. Reading remains possible.

### 1.4 The page's surfaces

The page is laid out in **two columns**: on the left, the **chat area** (the wider one); on the right, a **statistics sidebar** that appears on demand. Added to these are collapsible panels and a modal window:

| Surface | Where | Role |
|---|---|---|
| **Chat area** | Left column | Conversation thread, message input |
| **Conversations panel** | Unfolds beneath the header | List of saved conversations (open, new, delete) |
| **"Analyze a document" panel** | Unfolds above the input | Upload a file + optional instructions |
| **"Analyze a plan" panel** | Unfolds above the input | Upload up to 10 plans |
| **"Import clients" modal** | Overlay window | Three-step CSV / Excel import assistant |
| **"Stats" sidebar** | Right column (on demand) | Credits, daily costs, 30-day usage |

### 1.5 What the module does — and does NOT do

The module **does**: chat in natural language, consult your data (read), write to your database (framed creation / modification / deletion), give expert construction advice, analyze a document, analyze plans by image reading, import clients, keep conversation history, and track consumption and cost.

The module **does NOT**:

- **It does not produce project cost estimates.** An on-screen warning box says so: "For an **estimate**, go instead to the **Quotes → AI Estimation** module. The AI Assistant is not designed to produce cost estimates." AI Estimation uses a more powerful model, specialized expert profiles and deterministic price calculation — which the AI Assistant does not have.
- **It offers no expert-profile selector.** The assistant always runs on a single profile (see §1.7). You do not choose "Electrician" or "Plumber" on this screen.
- **It does not display the answer as it types** (no "typewriter" effect): the answer appears all at once, once complete. There is **no "Stop" button** during generation.
- **It does not let you rename a conversation** (only create, open, delete).
- **It does not export** conversations or analyses (no PDF, no dedicated printing). Workaround: select-and-copy the text, or use the browser's print function.
- **It does not search the Internet** and calls no external services: it limits itself to your data and its general knowledge.

### 1.6 The AI engine and pay-as-you-go billing (in brief)

- **Model used**: `claude-sonnet-4-6` (Anthropic's "Sonnet"). It is a fast, economical model — **not** the "Opus" model, which is reserved for AI Estimation. Answer capped at 32,000 tokens (`AI_MAX_TOKENS`, `ai.py:45`).
- **Billing.** Each exchange consumes the company's **prepaid credits**, calculated from the number of tokens read and produced, marked up by 30% (see the formula in §4.4). The balance is shown at the top of the page.
- **Automatic top-up.** When the balance drops below $0.10, the ERP attempts an automatic top-up via Stripe (default amount $10.00). If the card is declined or the balance is exhausted without a top-up, the assistant returns a "Credits exhausted" error (code 402) and displays a banner with a **Top up** button (see §3.9).
- **Exempt accounts**: the super-administrator and a few internal companies (IDs 1, 105 and 172) have **unlimited** usage, with no charge.

### 1.7 An important nuance: "expert profiles", two systems not to be confused

The module's name mentions "expert profiles". In the ERP, that term covers **two distinct mechanisms** — and **only one** concerns this page:

1. **Six internal profiles** (`AI_PROFILES`, `ai.py:737-790`): `general` ("Constructo AI Assistant"), `expert_construction`, `estimateur`, `comptable`, `juridique`, `securite`. **The AI Assistant always uses the `general` profile.** The code hard-codes this choice (`AssistantIAPage.tsx:38`: `const selectedProfile = 'general';`) and **no selector is offered** on screen. So you never change profiles here. The `GET /ai/profiles` endpoint would indeed return the six profiles, but the page does not call it.
2. **About 66 file-based expert profiles** (Architect, Electrician, Plumber, Roofing, RBQ and CCQ [Quebec's building authority and construction commission], General contractor QC / CA / US, etc.) plus **custom profiles** stored per company. They are managed by `ai_profiles.py` and served by `GET /ai-profiles/experts`. **These profiles do NOT feed the AI Assistant**: they are consumed by the **AI Estimation** module (Quotes) and by **PDF Takeoff**. If you are trying to choose a specific expert (for a quote, say), it is in those modules, not here.

In short: on this screen there is **a single brain** (`general`), which **adapts its expertise automatically** to the subject of your question (see §4.3), with no selection menu.

---

## 2. Interface

### 2.1 Page header

At the top of the chat area:

- A purple sparkle icon (`Sparkles`), the title **"AI Assistant"** and the subtitle **"Versatile construction expert — Connected to your data"**.
- The **"Conversations"** button (`MessageSquare` icon): opens or closes the panel of saved conversations. A counter `(n)` appears next to the label when conversations exist.
- The **balance indicator** (`CreditCard` icon): displays the remaining amount as `$X.XX`, or **"Unlimited"** if your company is exempt. The color reflects the balance state: **green** above $5, **yellow** between $0 and $5, **red** at $0 or below.
- The **"Stats"** button (`BarChart3` icon): shows or hides the statistics sidebar (see §2.9).

### 2.2 "Conversations" panel

This panel unfolds when you click **"Conversations"**.

- At the top: the title **"Conversations"** and a **"New"** button (`Plus` icon) that clears the screen to start a blank thread.
- **Conversation list** (vertical scroll): each row shows an icon, the conversation's **name** and a **"{n} messages"** counter. The active conversation is highlighted in blue.
  - **Clicking a row** loads the full conversation into the chat area.
  - **The trash button** (`Trash2` icon, "Delete" tooltip) permanently deletes the conversation.
- **Empty state**: "No conversation".

> **Reminder of the limits.** You can **create**, **open** and **delete** a conversation, but **not rename it**: the name is assigned automatically. The list shows the user's 30 most recent conversations.

### 2.3 "AI credits exhausted" banner

When the balance is at $0 or below (and the company is not exempt), a red banner appears at the top of the chat area:

- `AlertTriangle` icon, title **"AI credits exhausted"** and the message **"Top up your balance to keep using the AI assistant."**
- **"Top up"** button (`ExternalLink` icon): opens, in a new tab, the Stripe billing portal (`https://billing.stripe.com/p/login/constructoai`). There you manage your card and your balance (see §3.9).

As long as the credits are exhausted, message input is disabled.

### 2.4 Chat area

**Empty state (no message).** In the center: a sparkle icon, the title **"Ask your question"** and an inviting subtitle. Just below, an **amber warning box**:

> For an **estimate**, go instead to the **Quotes → AI Estimation** module. The AI Assistant is not designed to produce cost estimates.

This box also contains two quick-start buttons: **"Analyze a document"** and **"Analyze a plan"**, which open the corresponding panels (see §2.6 and §2.7).

**Message bubbles.**

- Your message appears in a **blue bubble aligned to the right** ("person" avatar).
- The assistant's answer appears in a **light bubble with a blue border**, with the "hard hat" avatar (`HardHat`). The text is formatted (Markdown): headings, lists, **bold**, and above all styled **tables**.
- Below each assistant answer, a bubble footer shows indicators: a **profile badge** ("Constructo AI Assistant", or "Document Analysis", or "Plan Analysis" depending on the type), a **type badge** (Document in blue, Plan in green), the **token count**, the **cost** in dollars (shown to four decimals, in orange) and the answer's **duration** in seconds.

**While waiting for an answer**, an animated indicator (sparkle avatar + loading spinner) temporarily replaces the future bubble. The answer appears **all at once** once ready (no progressive display).

### 2.5 Input bar

At the bottom of the chat area:

- **Three icon buttons** to the left of the field:
  - `FileUp` — **"Analyze a document"**: opens the document-analysis panel (§2.6).
  - `Image` — **"Analyze a plan"**: opens the plan-analysis panel (§2.7).
  - `Database` — **"Import clients (CSV/Excel)"**: opens the import modal (§2.8).
- **The input field**, with the placeholder **"Ask the AI expert your question…"** (or "Credits exhausted" if the balance is empty). **The Enter key sends the message** (without `Shift`). The field is disabled while an answer is being generated or if credits are exhausted.
- **The "Send" button** (`Send` icon): disabled if the field is empty, if an answer is in progress, or if credits are exhausted.

> **Note.** There is **no "Stop" button** to interrupt an answer in progress, and **no as-you-type display**. A long answer may take a few seconds; wait until it is fully displayed.

### 2.6 "Analyze a document" panel

This panel unfolds above the input bar.

- Title: **"Analyze a document"**.
- Upload area: **a single file**, maximum size **50 MB**. Accepted formats: **PDF, Word (.docx), Excel (.xlsx), CSV, TXT, Markdown, JSON, HTML** and **images** (JPG, PNG). The label reads "Document (PDF, DOCX, XLSX, CSV, TXT, Images)".
- **"Specific instructions (optional)"** field: state what you want (for example "Extract all amounts and deadlines", "Summarize this contract in ten points").
- **"Analyze the document"** button: launches the analysis. The result is inserted into the thread as an assistant message preceded by "**Document type:** … | **Pages:** …".

Under the hood: text documents are read and **truncated to 100,000 characters**; images are resized to 1568 pixels before reading.

### 2.7 "Analyze a plan" panel

Also above the input bar.

- Title: **"Analyze a plan"**.
- Upload area: **up to 10 files**, maximum size 50 MB each. Formats: **JPG, PNG, PDF**. The label reads "Plans (JPG, PNG, PDF) — up to 10 files".
- **"Analyze the plans"** button: launches the image reading. The result is inserted into the thread, preceded by "**Plan type:** … | **Files analyzed:** …".

Under the hood: for a PDF, only **the first five pages** are converted to images (at double resolution) and read. The assistant identifies the plan's elements, estimates areas and suggests the trades.

### 2.8 "Import clients" modal

Opened by the `Database` button on the input bar, this window unrolls a **three-step** assistant.

**Step 1 — File choice.**
- Drop zone: **"Choose a file (.csv or .xlsx)"** button.
- Help: "Accepted formats: CSV, Excel (.xlsx). Maximum 8 MB / 5000 rows."
- **"Cancel"** and **"Analyze the file"** buttons. The analysis is **read-only**: it changes nothing, it prepares the preview.

**Step 2 — Preview and column matching.**
- Four counters: **"n to create"**, **"n to update"**, **"n in error"**, **"n row(s) total"**.
- A **matching** table: for each column in your file, a **dropdown** lets you choose the target client column. The AI already proposes a match; you can correct it. Possible targets: **Name (required)**, Type, Industry, Email, Phone, Address, City, Province, Postal code, Country, Website, GST No., QST No., Payment terms, Notes — plus the **"— Ignore —"** option to skip importing a column.
- Safeguard: the **"Confirm import"** button stays **disabled** as long as the **Name** column is not mapped. The message "The 'Name' column is required: map it to a column in the file." reminds you of this.
- **"Back"** and **"Confirm import"** buttons. Confirmation **actually writes** the clients into your database.

**Step 3 — Report.**
- Title **"Import complete"**, with the count **"n created"**, **"n updated"**, **"n error(s)"** and the error details, row by row.
- **"Close"** button.

> **Scope of the import.** The import assistant populates only the **clients / companies** table (`companies`). The **Name** column is required; the file is capped at **8 MB** and **5000 rows**.

### 2.9 "Stats" sidebar

Shown by the **"Stats"** button in the header, it stacks three cards.

**"AI Credits" card.**
- If the company is exempt: the note **"Unlimited"** and an **"Exempt"** badge.
- Otherwise: the **balance** in US dollars, the line **"Used this month: $X.XXXX"**, and, if automatic top-up is active, **"Auto top-up: $X"**. A **"Top up credits"** button (link to the Stripe portal) appears when the balance is zero or negative.

**"Daily costs (30d)" card.**
- A small **bar chart** of 30 bars (one per day), with a tooltip per day (date, cost, number of requests), a "30d / Today" legend and the **30-day total**.

**"Usage (30 days)" card.**
- The totals **Requests / Tokens / Cost** over 30 days, then a **breakdown by feature** (for example `chat_general`, `analyze_document`, `analyze_plan`) with each one's request count.

These cards are fed by the `GET /ai/credits`, `GET /ai/usage` and `GET /ai/usage/daily` endpoints.

---

## 3. Step-by-step workflows

### 3.1 Ask a question about your data

1. Open the AI Assistant (the "sparkle" button in the top bar).
2. Write your question, for example: "List the projects in progress with their planned end date" or "What are my three oldest unpaid invoices?".
3. Press **Enter** (or click **Send**).
4. The assistant thinks, consults your database as needed (`recherche_bd` tool, read-only, 50 rows maximum per query), then answers in plain language, often as a **table**.
5. The exchange's cost and the token count appear below the answer; the balance at the top of the page is updated.

> **Tip.** You can request a format: "Show me this as a table with columns Client, Amount, Days overdue." The assistant can also chain steps: the conversation keeps the context of previous exchanges.

### 3.2 Ask the AI to create or modify data

The assistant can **write** to your database (`executer_action` tool). Examples: "Add a note to the 'South Shore Renovation' project: inspection scheduled for Friday", "Move work order BT-00042 to Done status".

1. State the request clearly, unambiguously identifying the target record (number, exact name).
2. Send the message.
3. The assistant performs the operation **immediately** and confirms the result (for example "Record created, ID 128" or "1 row modified").

> **Major caution — the write is immediate and has no confirmation step.** In this module there is **no "I propose, you confirm" mechanism** on the server side: if the assistant decides to act, the action is applied and committed on the spot. The technical safeguards are: destructive keywords (DROP, TRUNCATE, ALTER, etc.) are blocked; **any modification or deletion must include a `WHERE` condition** (it is impossible to wipe an entire table at once); every action is logged. But **there is no automatic undo**. Be precise, and avoid vague requests like "delete the old files": prefer "delete file number 57". When in doubt, first ask the assistant to **show you** the affected records (read) before asking it to act.

### 3.3 Resume or manage a conversation

1. Click **"Conversations"** to unfold the list.
2. **To resume**: click a conversation; its full thread reloads, and you can continue the dialogue.
3. **To start fresh**: click **"New"**; the screen clears (the previous conversation stays saved).
4. **To delete**: click the **trash** on the relevant row.

Conversations are **saved automatically** after each exchange; you have nothing to save manually.

### 3.4 Analyze a document

1. Click the **"Analyze a document"** button (`FileUp` icon) or the button of the same name in the empty-state box.
2. Upload **one** file (PDF, Word, Excel, CSV, TXT, image, etc.), 50 MB maximum.
3. Optional: state your **instructions** ("Check the penalty clauses", "Give me the list of materials and quantities").
4. Click **"Analyze the document"**.
5. The answer is added to the thread, preceded by the document type and the number of pages.

### 3.5 Analyze plans

1. Click **"Analyze a plan"** (`Image` icon).
2. Upload **up to 10** files (JPG, PNG or PDF).
3. Click **"Analyze the plans"**.
4. The answer (elements spotted, estimated areas, suggested trades) is added to the thread.

> **Good to know.** For a PDF, the assistant reads **the first five pages** only. If your plan has more useful sheets, split it or upload the relevant pages as images.

### 3.6 Import clients from a CSV or Excel file

1. Click **"Import clients (CSV/Excel)"** (`Database` icon).
2. **Step 1**: choose your file (8 MB / 5000 rows maximum), then **"Analyze the file"**.
3. **Step 2**: check the **column matching** proposed by the AI. Make sure the **Name** column is properly mapped (otherwise the import stays blocked). Adjust the other columns or set them to "— Ignore —". Check the counters (to create / to update / in error).
4. Click **"Confirm import"**.
5. **Step 3**: read the **report** (created / updated / errors). Fix your file if needed and start over for the rows in error, then **"Close"**.

### 3.7 Monitor consumption and balance

1. Click **"Stats"** in the header.
2. Check the **"AI Credits"** card (balance, used this month, auto top-up).
3. Check the **daily-cost bar chart** (30 days) and the **"Usage"** card (requests, tokens, cost, breakdown by feature).

### 3.8 Top up credits

1. Click **"Top up"** (red banner) or **"Top up credits"** (Stats card).
2. The **Stripe billing portal** opens in a new tab (`https://billing.stripe.com/p/login/constructoai`).
3. Manage your card, your balance and your history there. Top-up happens **outside the ERP**, in the interface hosted by Stripe.

> **Good to know.** There is no "amount to top up" field in the ERP: top-up goes through Stripe. If **automatic top-up** is active, the ERP tops up on its own (default amount $10.00) as soon as the balance falls below $0.10.

### 3.9 What to do when the assistant shows "Credits exhausted"

1. The input field is locked and a red banner appears.
2. Click **"Top up"** and complete the operation in Stripe.
3. Return to the ERP and refresh the page (or wait for the balance to update). As soon as the balance turns positive again, the assistant answers once more.

If automatic top-up fails (card declined), the Stripe error message states the cause; update your payment method in the portal.

---

## 4. Reference

### 4.1 AI-engine endpoints (`ai.py`, prefix `/api/erp/v1/ai`)

All require being logged in; none has an additional role guard.

| # | Method + path | Role | Notes |
|---|---|---|---|
| 1 | `POST /ai/chat` | Main chat (tool loop) | Core of the module; used by the page |
| 2 | `GET /ai/conversations` | Conversation list (`subject='assistant_ia'`) | 30 maximum, per user |
| 3 | `GET /ai/conversations/{id}` | Conversation detail (messages) | |
| 4 | `DELETE /ai/conversations/{id}` | Deletion | 404 if already gone |
| 5 | `GET /ai/profiles` | List of the 6 internal profiles | **Not called** by this page |
| 6 | `GET /ai/usage` | Usage statistics (period 1-365 d, default 30) | Super-admin = global totals |
| 7 | `GET /ai/credits` | Prepaid-credit balance | Creates a row at 0 if absent |
| 8 | `GET /ai/quota` | Quota indicator (`allowed` **always true**) | **Not called** by this page |
| 9 | `GET /ai/usage/daily` | Daily breakdown (1-90 d, default 30) | Feeds the bar chart |
| 10 | `GET /ai/usage/monthly` | Monthly breakdown (1-24 months, default 6) | **Not called** by this page |
| 11 | `POST /ai/analyze-document` | Document analysis | 1 file, 50 MB |
| 12 | `POST /ai/analyze-plan` | Plan analysis (image reading) | 10 files maximum |

### 4.2 Expert-profile endpoints (`ai_profiles.py`, prefix `/api/erp/v1/ai-profiles`)

> **Reminder**: these eight endpoints serve **AI Estimation** and **PDF Takeoff**, **not** the AI Assistant. They are listed here to clear up the confusion about "expert profiles". None has an administrator guard: any logged-in user of the tenant can create / delete a custom profile and upload documents to it (up to 20 MB).

| # | Method + path | Role |
|---|---|---|
| 1 | `GET /ai-profiles/` | List of the tenant's custom profiles |
| 2 | `POST /ai-profiles/` | Create a custom profile (name, instructions) |
| 3 | `GET /ai-profiles/experts` | **≈ 66 file-based profiles** + custom profiles |
| 4 | `GET /ai-profiles/{id}` | Custom-profile detail + documents |
| 5 | `PUT /ai-profiles/{id}` | Modify a custom profile |
| 6 | `DELETE /ai-profiles/{id}` | Delete (with its documents) |
| 7 | `POST /ai-profiles/{id}/documents` | Add a reference document (20 MB max) |
| 8 | `DELETE /ai-profiles/{id}/documents/{doc_id}` | Remove a document |

### 4.3 The assistant's two SQL tools

These tools are provided to the model only if the user is attached to a company. The tool loop runs at most **five times** per question.

**`recherche_bd` — read-only.**
- Only `SELECT` / `WITH` queries, **50 rows maximum**, in a **read-only** transaction with a 10-second timeout.
- Blocked keywords: `DROP, TRUNCATE, ALTER, CREATE, GRANT, REVOKE, SET ROLE, SET SESSION, COPY, LOCK, VACUUM, TABLE`. Semicolons forbidden (no chaining of queries). File-reading / cross-database access functions blocked.
- **Protected sensitive data**: the users table and sensitive columns (password, SIN — Social Insurance Number, token, secret, credit limit, etc.) are refused in the query **and** masked in the result, even on a `SELECT *`.

**`executer_action` — write.**
- `INSERT` / `UPDATE` / `DELETE` on **any table in your company** (no table allowlist).
- Safeguards: same blocked keywords; **`UPDATE` and `DELETE` without a `WHERE` refused**; 10-second timeout; validation and audit logging.
- **No server-side confirmation step**: execution is immediate (see the caution in §3.2).

**Automatic expertise adaptation.** For each question, the assistant **detects the subject** (among 18 intents: advice, finances, team, materials, schedule, electrical, plumbing, structure, roofing, insulation, safety, legal, accounting, creation, modification, etc.) and adjusts its framing. This setting is **invisible** and **automatic**; there is nothing to select.

### 4.4 AI model and cost calculation

| Item | Value |
|---|---|
| Model | `claude-sonnet-4-6` (Sonnet) |
| Maximum tokens per answer | 32,000 |
| Input rate | $0.003 / 1000 tokens read |
| Output rate | $0.015 / 1000 tokens produced |
| Markup applied | × 1.30 (30%) |

**Formula** (identical for chat, document analysis and plan analysis):

```
cost_$ = ( input_tokens × 0.003 + output_tokens × 0.015 ) / 1000 × 1.30
```

*Examples.* A simple question (≈ 500 tokens read + 300 produced) costs about $0.008 (less than a cent). A heavier plan analysis (≈ 4000 + 2000 tokens) costs about $0.055. With a $10 balance, that represents hundreds to over a thousand exchanges depending on their complexity.

### 4.5 Credits, top-up and thresholds

| Parameter | Value | Effect |
|---|---|---|
| Billing active | yes (by default) | Otherwise, unlimited internal usage on the customer's key |
| Exempt accounts | super-admin + companies 1, 105, 172 | Unlimited usage, no charge |
| Auto-top-up threshold | balance < $0.10 | Triggers a Stripe top-up |
| Default top-up amount | $10.00 | Configurable on the platform side |
| Monthly spending cap | **none** | Product decision: never a hard block by usage |
| Negative balance | allowed | Actual usage is always consumed and tracked |
| On a verification error | block (fail-closed) | The AI stops answering out of caution |

> **Caution (billing).** The `/ai/chat`, `/ai/analyze-document` and `/ai/analyze-plan` endpoints **charge without an idempotency key**. In plain terms: if the network retries the request (double submission, reconnection), the charge may **repeat**. Avoid clicking "Send" several times or refreshing the page during an answer.

### 4.6 Limits and caps

| Function | Limit |
|---|---|
| Tool loop per question | 5 iterations |
| `recherche_bd` | 50 rows per query, read-only, 10 s timeout |
| `executer_action` | `WHERE` required on modify/delete, 10 s timeout |
| Document analysis | 1 file, 50 MB, text truncated to 100,000 characters |
| Plan analysis | 10 files, PDF read over its first 5 pages, images reduced to 1568 px |
| Client import | clients table only, Name column required, 8 MB / 5000 rows |
| Conversation history | 30 recent conversations shown |
| Rate limit (per IP) | 1500 requests / 60 s (the ERP's general limit) |

> **Technical note.** Unlike the mini-assistants of the other modules (capped at 20 requests/minute), the AI Assistant **has no dedicated rate limit**: it falls back to the general limit of 1500 requests per minute per IP address. The real brake remains the credit balance.

### 4.7 Response codes and error messages

| Code | Meaning | What you see |
|---|---|---|
| 200 | Success | The answer displays |
| 402 | Credits exhausted / card declined | "AI credits exhausted" banner + Top up button |
| 403 | AI service access denied | "AI service access denied." message |
| 413 | File or conversation too large | Dedicated message |
| 429 | Too many requests (rate exceeded) | Dedicated message |
| 503 | AI service missing or overloaded | Dedicated message; try again later |
| 500 | Internal error | Generic message |

On an error, the page shows an alert box and an "Error: {message}" bubble. The precise message comes from the server (for example the reason for the card decline).

### 4.8 PostgreSQL tables

| Table | Location | Role |
|---|---|---|
| `ai_prepaid_credits` | `public` (shared) | Credit balance per company and per month, Stripe info, auto top-up |
| `ai_usage_tracking` | `public` (shared) | Log of each call: user, feature, model, tokens, cost, duration |
| `ai_credit_applied_invoices` | `public` (shared) | Anti-duplication of Stripe top-ups |
| `ai_credit_ledger` | `public` (shared) | Debit idempotency ledger (not used by the 3 chat endpoints — see §4.5) |
| `conversations` | per tenant | AI Assistant conversations (`subject='assistant_ia'`) |
| `conversation_messages` | per tenant | Conversation messages |
| `ai_profiles` | per tenant | **Custom** expert profiles (AI Estimation / Takeoff module) |
| `ai_profile_documents` | per tenant | Reference documents of the custom profiles |
| `companies` | per tenant | Clients / companies — **import target** |

> The `public` schema's `ai_*` tables are **shared** across all companies to centralize AI billing; conversations, however, remain **isolated** per company and per user.

### 4.9 What does not exist in this module (recap of limits)

- No **expert-profile selector** (fixed `general` profile).
- No **as-you-type display** and no **"Stop"** button.
- No conversation **renaming**.
- No **export** or dedicated printing.
- No **Internet search** and no call to external services.
- No **project cost estimate** (redirect to Quotes → AI Estimation).

---

## 5. Integrations and FAQ

### 5.1 Links with the other modules

| Module | Link with the AI Assistant |
|---|---|
| **Quotes → AI Estimation** | The **real** cost estimate. More powerful model, selectable expert profiles, deterministic price calculation. The AI Assistant explicitly redirects there. |
| **PDF Takeoff** | Also consumes the 66 file-based expert profiles (via `ai_profiles.py`). |
| **Sales / Contacts** | The import assistant populates the clients table (`companies`), then visible in Contacts / Sales. |
| **Configuration / Administration** | The credit balance, automatic top-up and Stripe billing are managed at the company level; the super-administrator sees the usage totals of all companies. |
| **Other AI mini-assistants** (Accounting, Real Estate, CAD, Messaging, Takeoff, Extras, etc.) | They reuse the **same engine** (`ai.py`) for billing and the call to Claude, but each has its own scope and its own safeguards. Unlike the AI Assistant, several of them apply an "I propose, you confirm" pattern before writing. |

### 5.2 Data isolation and confidentiality

- Each of the assistant's requests runs **in your company's schema**: the `recherche_bd` and `executer_action` tools see **only your data**. A leak of data from one company to another through these tools is blocked by PostgreSQL's per-schema partitioning.
- Data sent to the Claude API passes through Anthropic's servers. Anthropic undertakes **not to use** API data for training (barring explicit agreement). If you handle personal information, take your obligations into account (Quebec's Law 25 / Canada's PIPEDA).

### 5.3 FAQ

**Where is the AI Assistant? I don't see it in the left menu.**
It is not there. You open it via the **"AI Assistant"** button (sparkle icon) in the **top bar**, present everywhere in the ERP, or via the `/assistant-ia` address.

**Can I choose an "Electrician" or "Plumber" profile here?**
No. This screen runs on a single profile (`general`) that **automatically adapts** its expertise to the subject of your question. The named expert profiles (Electrician, Plumber, Architect, RBQ and CCQ, etc.) serve the **AI Estimation** module and **PDF Takeoff**, not the AI Assistant.

**Can the AI really modify or delete my data?**
Yes, through the `executer_action` tool (create / modify / delete). **The action is immediate, with no confirmation step** in this module. The protections are: destructive keywords blocked, mandatory `WHERE` condition, audit log — but **no automatic undo**. Be precise in your requests (see §3.2).

**How much does a conversation cost?**
A few fractions of a cent for a simple question; about $0.05 for a complex plan analysis. The exact cost is shown below each answer and accumulates in the Stats tab.

**Why does the assistant tell me to go to "AI Estimation"?**
Because the AI Assistant **is not designed to price a project**. AI Estimation (Quotes module) uses a more powerful model, expert profiles and controlled price calculation, tailored to the Quebec market.

**Can I export a conversation to PDF?**
No dedicated button. Workaround: select-and-copy the text, or use the browser's print function.

**Does the chat display word by word?**
No. The answer appears all at once, once complete. There is also no "Stop" button.

**What happens if my balance is exhausted?**
The assistant returns a "Credits exhausted" error (402) and input locks. Top up via the Stripe portal (**Top up** button). If automatic top-up is active, the ERP tops up on its own as soon as the balance falls below $0.10.

**Can I upload several documents to analyze?**
For **document analysis**, one file at a time (50 MB). For **plan analysis**, up to ten files. A plan PDF is read over its **first five pages** only.

**Can the client import modify existing clients?**
Yes: the preview distinguishes "to create" from "to update". Confirmation applies both. The **Name** column is required; the file is capped at 8 MB / 5000 rows.

**Can an employee without administrator rights use the assistant?**
Yes. Any logged-in user of the company has access, **including** the ability to make the AI write to the database. There is no role check on this module; the only brake is the credit balance.

**Are my conversations private to my account?**
Yes, they are tied to your user and your company; the 30 most recent appear in the Conversations panel.

**Can the AI search the Internet?**
No. It limits itself to your data and its general knowledge. For web search, see the ERP's **Web** module.

---

## 6. Summary

- **Access via the top bar** ("sparkle" **AI Assistant** button), **not** via the side menu; address `/assistant-ia`.
- **A single active profile** (`general`), **with no selector**; expertise adapts automatically to the subject. The "66 expert profiles" belong to **AI Estimation** and **PDF Takeoff**, not to this module.
- **Model** `claude-sonnet-4-6` (Sonnet), 32,000 tokens maximum — not Opus.
- **Two SQL tools**: `recherche_bd` (read, 50 rows) and `executer_action` (**immediate write, no server confirmation**; `WHERE` required). It is the **only** ERP assistant with write access — use with caution.
- **Document analysis** (1 file, 50 MB) and **plan analysis** (10 files, PDF over 5 pages) by image reading.
- **Client-import assistant** CSV / Excel in three steps (preview, matching, report), **Name column required**, 8 MB / 5000 rows.
- **Conversations** saved automatically: create, open, delete — **no renaming, no export**.
- **No as-you-type display, no "Stop" button**.
- **Pay-as-you-go billing**: cost = (input × 0.003 + output × 0.015) / 1000 × 1.30; auto Stripe top-up below $0.10 (default $10.00); no hard monthly cap; super-admin and companies 1/105/172 exempt.
- **Not an estimation tool**: the screen redirects to **Quotes → AI Estimation**.
- **No role check** on the module: any logged-in user can chat and trigger writes; the brake is the credit balance.

---

**Documentation generated from verified source code**: `backend/routers/ai.py` (3,541 lines, 12 endpoints), `backend/routers/ai_profiles.py` (804 lines, 8 endpoints), `backend/routers/data_import.py` (684 lines), `frontend/src/pages/AssistantIAPage.tsx` (716 lines), `frontend/src/components/ai/ClientImportModal.tsx` (243 lines), `frontend/src/api/ai.ts`, `frontend/src/api/dataImport.ts`, `frontend/src/components/layout/TopBar.tsx` (access button, lines 312-320), `i18n/locales/fr/ai.json`.

**Related manuals**:
- Module 08 (Sales — Quotes / **AI Estimation**) — `08-ventes-soumissions.md`
- Module 05 (Contact management — imported clients) — `05-gestion-contacts.md`
- Module 19 (Field — Real Estate, AI mini-assistant) — `19-terrain-immobilier.md`
- Module 24 (Communication — Messaging, AI mini-assistant) — `24-communication-messagerie.md`
- Module 28 (Configuration — AI credits, Stripe) — `28-configuration.md`
- Module 30 (PDF Takeoff — expert profiles) — `30-metre-pdf.md`
