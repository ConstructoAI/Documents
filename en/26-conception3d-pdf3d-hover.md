# Module 26 — PDF3D (plan to 3D)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Route**: `/hover` — left-side menu "PDF3D" (**3D DESIGN** section of the navigation bar, alongside "CAD" and "3D Render")
> **Reference code**: `backend/routers/hover.py` (1,932 lines; 13 endpoints including 1 public webhook), mounted defensively in `erp_api.py` under the real prefix `/api/erp/v1/hover/...`; `frontend/src/pages/HoverPage.tsx` (541 lines), `frontend/src/api/hover.ts` (182 lines; 12 functions), `frontend/src/components/hover/HoverEstimateModal.tsx` (293 lines) + `frontend/src/components/hover/hoverEstimate.ts` (composes measurements → quote), i18n `frontend/src/i18n/locales/{fr,en}/hover.json` (73 keys, FR/EN parity)
> **Shared PostgreSQL tables (`public`)**: `hover_oauth` (central account, singleton `id=1`), `hover_jobs` (plan submissions, isolated per company via the `tenant_schema` column), `hover_webhooks`. Billing relies on the shared tables `public.ai_prepaid_credits` (credit balance) and `public.ai_usage_tracking` (usage tracking).
> **Scope**: this module sends a **set of architect's plans** (PDF, DWG or DXF, with PNG / JPG supplements) to an external 3D-reconstruction service that, within about 24 hours, returns a **3D model and exterior measurements** (roof, walls, openings). For a completed model, you consult the **deliverables** (PDF reports, presentation image, external 3D viewer, CAD XML file) and generate a **complete-construction budgetary estimate per square foot**, converted into a **quote** (Sales module). The technical service is **Hover** (`hover.to`), but all visible text names it **"PDF3D"** (white-label). This is **not** the **CAD / 3D Modeling** module (manual drawing, module 25) nor the **Takeoff** module (quantity takeoff on PDF, module 32).

*Terminology used in this manual:* "endpoint" refers to an API endpoint; "company" or "tenant" refers to your account (each company has its own isolated data); "job" or "submission" refers to a 3D-reconstruction request submitted to the service; "white-label" means the vendor's name (Hover) is hidden and replaced everywhere in the interface with "PDF3D".

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

### 1.1 The module's mission

PDF3D turns a **set of paper or digital plans** into a **usable three-dimensional model**. In concrete terms, the module lets you:

- **Submit a set of plans** (four-side elevations + dimensioned floor plans) to the reconstruction service, in a single submission bundling up to 24 files;
- **Track the progress** of each submission (submitted, pending, complete, failed) in a "My uploads" table;
- **Consult the deliverables** of a completed model: a **roof report** (PDF), a **Pro Premium report** (PDF), a **presentation image**, a **link to an external 3D viewer** and the model's **CAD file** (XML);
- **Generate a budgetary estimate** for a **complete construction, per square foot** (all trades) from the model's measurements, then **convert it into a quote** in the Sales module, to be reviewed before it is sent to the client.

The service reconstructs the building in **about 24 hours** (asynchronous processing); you are notified upon completion. Each reconstruction is **billed per structure**.

### 1.2 How to access it

- Left-side navigation bar → **3D DESIGN** section → **PDF3D** (building icon).
- Address: `/hover` (protected page, authentication required).
- The same 3D DESIGN section also contains **CAD** (`/cao2`) and **3D Render** (`/rendu-3d`), which are separate modules.

### 1.3 Roles and permissions

Three access levels coexist in this module. This is the most important point to understand before using it.

| Action | Who can do it |
|--------|---------------|
| **Open the page, view the status, list the submissions, consult the deliverables, generate an estimate** | Any signed-in user of the company |
| **Connect / disconnect the central PDF3D account** | **Platform super-administrator only** |
| **Submit a plan** (billed action) | **Administrator or owner of your company** — the super-administrator is **excluded** |

**Important clarifications:**

- **A single central PDF3D account exists for the whole platform.** It is connected **once** by the super-administrator (see §1.4). You do **not** have to create or connect a PDF3D account yourself: once the central account is plugged in, all companies use it.
- **The super-administrator cannot submit a plan.** They serve only to plug in the central account. Technically, they have no company schema, so submitting is refused for them server-side; the interface therefore hides the submission form from them (`HoverPage.tsx:52`, `hover.py:878`).
- **Only an administrator or owner of a company sees the submission form** and can incur the cost. An ordinary user (without administrative rights) sees the status and the list of submissions, but **not** the submission form.
- In **consultation mode** (suspended subscription / read-only), **all** the module's write actions (connect, disconnect, submit a plan, refresh a submission, generate an estimate) are **blocked** by the ERP's global control. Reading statuses and deliverables remains possible.

### 1.4 The central PDF3D account (connection model)

PDF3D works with **a single service account** shared by the whole platform, not one account per company:

- The **super-administrator** clicks once on **"Connect PDF3D"**, authenticates with the service (OAuth protocol), and the access token is **encrypted** then stored in a single database row (`public.hover_oauth`, `id=1`).
- After that, **every plan submission** from **every company** goes through this central account. The service does not distinguish between companies; it is the ERP that **ties each submission to your company** (the `tenant_schema` column of `public.hover_jobs`) and that **bills you** the cost.
- If the central account is **not** connected, the page shows a "not connected" status and submitting is impossible. If the server has **not** been configured at all (missing environment variables), the page shows "PDF3D is not enabled on this server yet."

### 1.5 Billing: two distinct costs

The module spends real money from **your prepaid credit balance** (the same balance as the 3D Render module and AI estimates). There are **two separate costs**:

| Cost | When | Amount | Idempotent |
|------|------|--------|------------|
| **A — Plan submission** | On send confirmation | Fixed: **about US$390** (service's real cost × margin) | **Yes** — the same submission is never billed twice |
| **B — Estimate** | On measurement reading (in the estimate modal) | AI's real cost (variable, small) × margin | **No** — each re-reading of the measurements is billed |

- **Cost A** is **US$299 × 1.30 = $388.70** (shown as "≈ US$390" in the interface). It is **configurable** by the server administrator; set to 0, submission billing is **disabled**.
- The **super-administrator is exempt** from billing A (but cannot submit a plan anyway).
- Before each send, the ERP **checks your balance**: if it is insufficient, the send is **refused** (error 402) **before** the external service is called — so no surprise charges.
- **Cost B** is the cost of the artificial intelligence (Claude Opus model) reading the PDF reports. It is small but **repeats every time** you click "Read model measurements".

> **Bill this cost back to your client.** The warning shown in the submission form reminds you: "Make sure you have the balance and bill this cost back to your client."

### 1.6 What the module does — and does NOT do

The module **does**: submit a set of plans, track the state of submissions, display the deliverables of a completed model (PDF reports, image, 3D viewer link, CAD XML file), read the measurements with AI, produce a complete-construction estimate per square foot, and create a budgetary quote in the Sales module.

The module does **NOT**:

- **No built-in 3D viewer.** "View in 3D" opens the service's **external viewer** in a new tab. An internal viewer had been built then **removed** (render deemed insufficient). Do not expect a 3D display in the page itself.
- **No automatic import into the CAD module.** The CAD file (XML) is **downloaded**; it is **not** automatically re-injected into the CAD / cao2 module. A manual or future import remains to be done ("to import into the CAD if needed").
- **No display of raw measurements (JSON).** The detailed measurements exist server-side but are **not** displayed as-is. The estimate relies on the AI's **reading of the PDF reports**, not on this raw data.
- **No instant result.** As long as the model is not **complete** (~24 h), no deliverable is available; the report buttons return a "not yet available" error.
- **No per-submission entry of the exact cost.** The amount billed for a submission is **fixed** (configured by the server), because the service does not return the real cost of each reconstruction.
- **No calling the service by an ordinary user.** Only a company administrator incurs the expense.

### 1.7 The page's three zones

The page consists of **three stacked cards** and **one modal window**:

| # | Zone | Role | Visible to |
|---|------|------|------------|
| A | **Connection** | Central account status, connect / disconnect buttons | Always |
| B | **Plan submission** | Form: name, files, cost warning, two-step send | Company admin/owner only, if the account is connected |
| C | **My uploads** | Submissions table + deliverables of completed models | As soon as the account is connected |
| — | **Estimate modal** | Measurements → estimate → quote (three steps) | Opened from a completed submission |

---

## 2. Interface

### 2.1 Header and banners

At the top of the page: a building icon, the title **"PDF3D — Plan to 3D"** and the subtitle "Turn a set of architectural plans (elevations + dimensioned floor plans) into a 3D model and measurements."

On load, a **loading indicator** ("Loading...") appears while the connection status is fetched.

Two **transient banners** appear as needed:

- **Error** (red, alert icon): the detailed error message returned by the server, or otherwise "An error occurred."
- **Success** (blue, check icon): confirmation of an action (for example "PDF3D account connected successfully." or "Plan sent.").

### 2.2 Card A — Connection

This card is **always visible**. It shows the state of the **central account** and offers the connection actions according to your role.

**Status indicator:**

- Green check + **"PDF3D account connected"** if the central account is plugged in;
- Amber icon + **"PDF3D account not connected"** otherwise.
- If the account is connected and a name is available, an **"Account"** line shows the service account name.

**Actions by situation:**

| Situation | What you see |
|-----------|--------------|
| **Server not configured** | Gray text: "PDF3D is not enabled on this server yet (missing environment variables). Contact the administrator." No button. |
| **Super-administrator, account not connected** | **"Connect PDF3D"** button (blue, link icon). During the call it shows "Redirecting..." then redirects you to the service's authorization page. |
| **Super-administrator, account connected** | **"Disconnect"** button (broken-link icon). Disconnecting clears the tokens but **keeps** the submission history. |
| **Non-super-administrator user, account not connected** | Gray text: "A super-administrator must connect the PDF3D account." No action available to you. |

> **Authorization return.** After authorizing access on the service's site, you are brought back to the `/hover` page with a confirmation code; the ERP automatically finalizes the connection and shows "PDF3D account connected successfully." or, on failure, "PDF3D connection failed. Please try again."

### 2.3 Card B — Plan submission

This card appears **only if** the central account is connected **and** you are an **administrator or owner** of your company (the super-administrator never sees it).

**Title and help:** "Send a plan" — "Attach the plan set (PDF, DWG or DXF) containing the 4-side elevations and the dimensioned floor plans."

**Fields:**

| Field | Detail |
|-------|--------|
| **Project / structure name** | Text field. Suggested example: "e.g. Tremblay Residence - lot 42". Required to enable the send button. |
| **Plan files** | **Multiple**-file picker. Accepted formats: **PDF, DWG, DXF, PNG, JPG**. Help shown: "Multiple files accepted (max 24, 80 MB total)." and "Formats: PDF, DWG, DXF, PNG, JPG." |

**Cost warning (amber box, triangle icon):** "Cost: about US$390 is deducted from your credit balance on send. Make sure you have the balance and bill this cost back to your client."

**Two-step send (anti-error):**

1. The **"Send"** button (upload icon) is **disabled** while the name is empty or no file is attached. A click switches the card to **confirmation mode**.
2. In confirmation, two buttons appear: **"Confirm send (≈ US$390)"** (amber) and **"Cancel"**. Only on the confirmation click does the send (and billing) happen. During the operation, the button shows "Sending...".

**Card footer note (clock icon):** "The 3D model and measurements are generated by PDF3D within about 24h (billed per structure). You will be notified on completion."

> **Double-send protection.** The interface generates a **unique submission key** the moment you prepare the form and reuses it on any retry (flaky network, double-click). The server recognizes this key and **never bills the same submission twice**. The key is reset only after a successful send.

### 2.4 Card C — My uploads

This card appears as soon as the central account is connected. It lists **your** submissions (your company's only).

**Title:** "My uploads", with a **"Refresh"** link (refresh icon) that reloads the list.

**Empty state:** "No plans sent yet."

**Table columns:**

| Column | Content |
|--------|---------|
| **Name** | Project name. If the submission is **complete**, a chevron lets you **expand** its deliverables; otherwise, plain text. |
| **Status** | Label: **Submitted**, **Pending**, **Complete**, **Failed**, or "—" if unknown. |
| **Requested by** | Name of the person who made the submission, or "—". |
| **Date** | Date and time of the submission, or "—". |
| *(action)* | Per-row **Refresh** icon: re-queries the service for that specific submission. |

**Refreshing a submission** queries the service to update the status. A status **can never go backward**: a "Complete" or "Failed" submission never reverts to "Pending", even if a message arrives late.

### 2.5 Expanded sub-row of a completed submission

Clicking the chevron of a **completed** submission opens a sub-row showing the model's **deliverables**.

- **Presentation image**: loaded on demand (via a secure channel). A placeholder shows while it loads.
- **Help**: "Deliverables of the completed model:".

**Deliverable buttons:**

| Button | Effect |
|--------|--------|
| **Roof report (PDF)** | Opens the roof report in a new tab. |
| **Pro Premium report (PDF)** | Opens the full report (façades, openings, materials) in a new tab. |
| **View in 3D** | Opens the service's **external 3D viewer** in a new tab. *Appears only if a viewer link is available.* |
| **Download CAD (XML)** | Downloads the model's CAD file (sanitized filename `{name}.xml`). |
| **Get estimate** | Opens the **Estimate modal** (see §2.6). |

**Explanatory note:** "\"View in 3D\" opens the PDF3D viewer (realistic render). \"Download CAD\" provides the model's XML file to import into the DAO if needed."

If there is a problem with a specific deliverable, a red error message appears below the buttons.

### 2.6 Estimate modal (measurements → quote, in three steps)

Opened by the **"Get estimate"** button of a completed submission. Title: **"Estimate — {project name}"**, with a close button (X). The modal follows a **state machine** across several phases, with **locks** that prevent any double-click.

**Phase 1 — Introduction (`idle`).**
Text: "Generates a budgetary COMPLETE construction estimate ($/sq ft, all trades) from the model measurements. You confirm the gross floor area, then the AI produces the quote lines. Review before sending." Amber box: "AI cost billed to your credit balance (measurement reading + estimate)." Button: **"Read model measurements"** (ruler icon).

**Phase 2 — Reading measurements (`loadingMeasures`).**
Loading indicator: "Reading model measurements...". In the background, the AI reads the model's PDF reports and pre-fills a **takeoff description** and an estimated **floor area**. **This reading is billed** (cost B).

**Phase 3 — Confirm the floor area (`measures`).**
- **Number field**: "Gross floor area (sq ft)" (ruler icon, pre-filled value, editable; example "1414").
- **Help**: "Footprint × number of storeys. Adjust if needed (e.g. 1414 = 707 main + 707 upper). Drives a COMPLETE construction $/sq ft estimate (structure, interior, plumbing/electrical/HVAC, finishes included)."
- **Expandable block** "View extracted measurements": shows the raw description produced by the AI.
- **Button**: **"Generate estimate"** (sparkles icon).

**Phase 4 — Estimating (`estimating`).**
Indicator: "Reading measurements and estimating...". The ERP's estimating AI converts the description + the confirmed floor area into **quote lines**. The target is about **$265/sq ft** for a complete, finished residential build (APCHQ — Quebec construction and housing professionals' association — range of $250–500/sq ft).

**Phase 5 — Preview (`preview`).**
A **line-items table** appears:

| Column | Content |
|--------|---------|
| **Description** | Line label (+ category in gray) |
| **Qty** | Quantity |
| **Unit** | Unit of measure |
| **Unit price** | Unit price |
| **Amount** | Line amount |

At the bottom: **"Estimated total"** (formatted in Canadian dollars) and the disclaimer: "Preliminary budgetary COMPLETE-construction estimate ($/sq ft, all trades; PDF3D exterior measurements as support). Review before sending to the client."

**Phase 6 — Create the quote (`creating` → `done`).**
The **"Create quote"** button creates a **budgetary** quote of **Construction** project type, named "Estimation - {project name}", with the markups (administration / contingencies / profit) from the AI, then injects the lines into it. At the end (`done`), a green "Quote created." banner shows the quote number, and the button becomes **"Open quote"** (which takes you to the Sales module with the quote open). The **"Close"** button is always present.

> **Idempotent creation.** If injecting the lines fails and you retry, the ERP **reuses the same quote** instead of creating a duplicate. Lines with no quantity (section headers) are automatically dropped, and each amount is recomputed and bounded to avoid any overflow.

---

## 3. Step-by-step workflows

### 3.1 Enable PDF3D (super-administrator, one time only)

**Precondition:** the server must be configured (service environment variables set by the infrastructure administrator). Otherwise the Connection card shows "PDF3D is not enabled on this server yet".

1. Sign in as **platform super-administrator**.
2. Open **3D DESIGN → PDF3D**.
3. In the **Connection** card, click **"Connect PDF3D"**.
4. You are redirected to the service's authorization page; authorize access.
5. You automatically return to `/hover`; the "PDF3D account connected successfully." message confirms the connection.

From then on, **all companies** can submit plans. You no longer need to intervene, except to disconnect the account.

### 3.2 Submit a plan set (company administrator)

**Precondition:** central account connected; sufficient credit balance (~US$390); you are an administrator or owner of your company.

1. Open **3D DESIGN → PDF3D**.
2. Prepare your **plan set**: ideally the **four-side elevations** and the **dimensioned floor plans**, in PDF, DWG or DXF (PNG / JPG supplements accepted). Maximum **24 files**, **80 MB total**.
3. In the **Plan submission** card, enter a clear **Project / structure name** (it will identify the submission in "My uploads").
4. Click **Plan files** and select your files.
5. Read the **cost warning** (~US$390 deducted on send).
6. Click **"Send"**, then **"Confirm send (≈ US$390)"**.
7. The "Plan sent." message confirms the submission; it appears in **My uploads** with status **Submitted**.

**What happens next?** The service reconstructs the building in **about 24 hours**. You have nothing else to do; come back later and click **Refresh** (or the row's refresh icon) to see the status change to **Complete**.

> **If the balance is insufficient**, the send is **refused before** any charge (error message indicating the required amount and your balance). Top up your credit balance, then start over.

### 3.3 Track and retrieve a completed model

1. In **My uploads**, click **Refresh** (or a row's refresh icon) to update statuses.
2. When a submission shows **Complete**, click the **chevron** to the left of its name to **expand** its **deliverables**.
3. Review:
   - **Roof report (PDF)** and **Pro Premium report (PDF)** — open in a new tab;
   - **View in 3D** — opens the external 3D viewer (if a link is available);
   - **Download CAD (XML)** — saves the model file to your computer.

### 3.4 Produce an estimate, then a quote

**Precondition:** a submission with status **Complete**; sufficient credit balance for the AI reading (small but real cost).

1. Expand the completed submission and click **"Get estimate"**.
2. In the modal, click **"Read model measurements"**. The AI reads the PDF reports (**billed**) and pre-fills the floor area.
3. **Check and correct the gross floor area** (footprint × number of storeys). If needed, expand "View extracted measurements" to check what the AI read.
4. Click **"Generate estimate"**. The estimating AI produces the lines (target ~$265/sq ft).
5. Review the **line-items table** and the **Estimated total**.
6. Click **"Create quote"**. A budgetary quote is created in the Sales module.
7. Click **"Open quote"** to finalize it (adjust prices, structure, markups) **before** sending it to the client.

> **Reminder.** This estimate is **preliminary** and relies on the model's exterior measurements plus a per-square-foot target. It **must be reviewed** by an estimator before any contractual send.

### 3.5 Import the model into the CAD module (manual)

The module does **not** automatically re-import the model into the CAD module. If you want to rework the geometry:

1. In the deliverables, click **"Download CAD (XML)"**.
2. Keep this file. Importing it into the CAD / cao2 module is a manual step (or a future feature); as of today, no "import into the CAD" button exists.

### 3.6 Disconnect the central account (super-administrator)

1. As super-administrator, open **3D DESIGN → PDF3D**.
2. In the **Connection** card, click **"Disconnect"**.
3. The central account tokens are cleared; **the submission history is kept**. No company can submit a plan until the account is reconnected.

---

## 4. Reference

### 4.1 API endpoints

All prefixed with `/api/erp/v1`. The "Guard" column indicates the access control applied server-side.

| Method | Path | Guard | Role |
|--------|------|-------|------|
| GET | `/hover/status` | Authenticated | Integration status (configured / connected / can connect) `hover.py:442` |
| POST | `/hover/oauth/connect` | Super-admin | Generates the authorization URL + anti-CSRF token `hover.py:485` |
| POST | `/hover/oauth/callback` | Super-admin | Exchanges the code for tokens, stores them encrypted `hover.py:529` |
| POST | `/hover/disconnect` | Super-admin | Clears the tokens (keeps the submissions) `hover.py:613` |
| POST | `/hover/blueprints` | Company admin/owner | **Plan submission → 3D (cost A)** `hover.py:864` |
| GET | `/hover/jobs` | Authenticated | List of the company's submissions `hover.py:1062` |
| GET | `/hover/jobs/{id}/measurements` | Authenticated | JSON measurements (defined but **not shown** in the interface) `hover.py:1108` |
| GET | `/hover/jobs/{id}/report/{kind}` | Authenticated | PDF report (`roof` or `pro_premium`) `hover.py:1151` |
| GET | `/hover/jobs/{id}/image` | Authenticated | Presentation image `hover.py:1204` |
| GET | `/hover/jobs/{id}/cad` | Authenticated | CAD file (XML) `hover.py:1240` |
| POST | `/hover/jobs/{id}/measures` | Authenticated | **AI measurement reading (cost B)** `hover.py:1331` |
| POST | `/hover/jobs/{id}/refresh` | Authenticated | Refreshes the status from the service `hover.py:1774` |
| POST | `/hover/webhook/{secret}` | **Public** (no auth) | Receives notifications from the service `hover.py:1790` |

### 4.2 Submission statuses

| Displayed status | Internal value | Meaning |
|------------------|----------------|---------|
| **Submitted** | `submitted` | The plan was accepted by the service; reconstruction in progress. |
| **Pending** | `pending` | Submission recorded, awaiting processing. |
| **Complete** | `complete` | The 3D model and measurements are available; the deliverables appear. |
| **Failed** | `failed` | The service could not reconstruct the model. |
| *(internal)* | `error` | Error at submission time (allows a legitimate retry). |

A status **never goes backward**: a completed or failed submission stays in its final state even if a late notification arrives.

### 4.3 Submission cost calculation (cost A)

| Item | Value |
|------|-------|
| Service base cost (configurable) | **US$299** by default |
| Margin applied | **× 1.30** |
| **Amount billed** | **$388.70** (shown as "≈ US$390") |
| Set to 0 | Submission billing **disabled** |
| Super-administrator | **Exempt** (and cannot submit anyway) |
| Debited from | `public.ai_prepaid_credits` (prepaid credit balance, in US dollars) |
| Balance guard | Check **before** send → **402** if insufficient |
| Idempotency | Yes (`hover_job:{id}` key + send header) — never double-billed |

### 4.4 Estimate cost calculation (cost B)

| Item | Value |
|------|-------|
| AI model (reading the PDF reports) | **Claude Opus 4.8** |
| Rate applied | Real token cost (input / output) **× 1.30** |
| Idempotency | **No** — each "Read measurements" is billed |
| Guard | AI authorization + credit balance → **402** if exhausted |

The estimate itself (generating the quote lines) then goes through the Sales module's estimating AI, also billed in credits.

### 4.5 Limits, bounds and quotas

| Item | Value |
|------|-------|
| Files per submission | **24** maximum |
| Total size per submission | **80 MB** (beyond: 413) |
| Accepted formats | **PDF, DWG, DXF, PNG, JPG** |
| Submissions per company / 24 h | **10** (beyond: 429) |
| Submissions for the whole platform / 24 h | **50** (beyond: 429) |
| Report / image / CAD (proxy) | **25 MB** maximum |
| Reports read by the AI (estimate) | **12 MB** maximum |
| Stored JSON measurements | **2 MB** maximum |
| AI model for measurement reading | `claude-opus-4-8`, `max_tokens` = 4000 |
| Estimate target (per square foot) | **~$265/sq ft** (APCHQ range $250–500/sq ft) |
| Reconstruction time | **~24 h** (asynchronous) |
| Anti-CSRF token validity (OAuth) | **10 minutes** |

**Per-IP rate limits (per minute):** send = 12, webhook = 120, refresh = 30, measurement reading = 6.

### 4.6 Deliverable types (deliverable_id)

The service reconstructs according to a **deliverable type**. PDF3D uses the **complete** type by default.

| Identifier | Type | Content |
|------------|------|---------|
| 2 | Roof (roof) | Roof measurements only |
| **3** | **Complete (default)** | **3D model + roof AND wall measurements** |
| 8 | Interior (interior) | Interior reconstruction |

### 4.7 Data model (shared `public` tables)

- **`hover_oauth`** — the single **central account** (`id=1`): access and refresh tokens (**encrypted**), expiry date, account name, temporary anti-CSRF token, status.
- **`hover_jobs`** — one record per submission: owning company (`tenant_schema`), requester, service identifiers, `submission_uuid`, name, deliverable type, status, measurements (JSON), viewer link, presentation image, billed amount, idempotency key. Each company sees only **its** rows.
- **`hover_webhooks`** — record of the service's notification URLs and their verification state.

### 4.8 Security and defensive behaviors

- **Encrypted central account.** The service tokens are encrypted (Fernet) before storage; the server **refuses** to store a token if it cannot encrypt it.
- **Token leak prevention.** Deliverables are fetched only from the service's official domain (`https://hover.to/`); any other address is refused.
- **Fail-closed webhook.** Receiving notifications requires a **secret in the address**; with no secret configured, the webhook is **inert** (no action). A forged notification can **never** create a submission or trigger billing; it can only update an **existing** row.
- **Per-company isolation.** Each submission is tied to your company schema; one tenant never sees another's submissions.
- **Generic error messages.** The technical detail of errors is never exposed to the user.
- **Consultation mode.** No module write is allowed in read-only (suspended subscription).

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

- **Sales / Quotes (module 07).** The estimate creates a **budgetary quote** (project "Construction") that you open and finalize in the Sales module. The "Open quote" button leads there directly.
- **CAD / 3D Modeling (module 25).** The CAD file (XML) is meant to be **imported as needed** into the CAD module; this import remains **manual** (no automatic bridge as of today).
- **3D Render (module) and AI estimates.** PDF3D **shares the same prepaid credit balance** (`public.ai_prepaid_credits`) as 3D Render and AI estimates. A submission or a measurement reading draws down this common balance.
- **Configuration / Subscription (module 30).** In **consultation mode** (suspended subscription), all PDF3D writes are blocked.
- **Super-Admin.** Connecting / disconnecting the central account is a **super-administrator** action; it gates the module's use for **all** companies.

### 5.2 Frequently asked questions

**Do I need to create a PDF3D account for my company?**
No. A **single central account** is connected once by the super-administrator, and all companies use it. You have nothing to connect yourself.

**Why don't I see the submission form?**
Three possible reasons: (1) the central account is not connected; (2) you are not an administrator or owner of your company (submitting is restricted to that role); (3) you are a super-administrator, who is **excluded** from submitting.

**Can the super-administrator submit a plan?**
No. They **connect** the central account, but submitting is refused for them (they have no company schema). Only a company administrator can submit and is billed.

**How much does a submission cost?**
About **US$390** (precisely US$299 × 1.30 = $388.70), debited from your credit balance on confirmation. This amount is **configurable** by the server and can be set to 0 to disable billing. **Bill this cost back to your client.**

**What if my balance is insufficient?**
The send is **refused before** any charge, with a message indicating the required amount and your balance. Top up, then start over. No debt, no surprise.

**Will I be billed twice if I click several times?**
Not for the **submission**: a unique key guarantees a single charge per submission. However, each **"Read model measurements"** (in the estimate modal) is billed separately — don't repeat it needlessly.

**How long until I get my 3D model?**
**About 24 hours.** Processing is asynchronous. Come back later and click "Refresh"; the status will change to "Complete".

**Can I view the 3D model directly in the ERP?**
No. "View in 3D" opens the service's **external viewer** in a new tab. An internal viewer was tried and then removed. You can also download the CAD file (XML) for external use.

**Does the CAD file import automatically into the CAD module?**
No. It **downloads**; importing it into the CAD / cao2 module remains manual (or forthcoming).

**Why is the billed amount in US dollars?**
Because the service bills in US dollars. The debit is taken from your prepaid credit balance (in US dollars), with a 30% margin.

**Can I adjust the floor area the AI proposes?**
Yes, and it's recommended. The "Gross floor area (sq ft)" field is **editable**. Correct it (footprint × number of storeys): it directly drives the per-square-foot estimate.

**Is the estimate reliable for a contract?**
Not as-is. It is a **preliminary budgetary estimate** (target ~$265/sq ft, all trades). It **must be reviewed** by an estimator before any send to the client.

**What if a submission stays "Pending" for a long time?**
Click the row's **Refresh** icon to re-query the service. Since processing can take ~24 h, some delay is normal. A complete or failed status will never revert to "Pending".

### 5.3 Common troubleshooting

| Symptom | Lead |
|---------|------|
| "PDF3D is not enabled on this server yet" | The service's environment variables are missing. This is an **infrastructure administrator** task. |
| "A super-administrator must connect the PDF3D account" | The central account is not plugged in: a **super-administrator** must click "Connect PDF3D". |
| The submission form does not appear | Account not connected, or you are not a company admin / owner (or you are a super-administrator, excluded from submitting). |
| Send refused (insufficient credits) | Top up your credit balance (~US$390 required), then start over. |
| Send refused (daily limit) | Cap of **10 submissions / company / 24 h** (or 50 for the platform) reached. Try again the next day. |
| "Plans too large" | You exceed **80 MB total** or **24 files**. Reduce the plan set. |
| A report returns a "not yet available" error | The model is not **Complete** yet (~24 h). Wait and refresh. |
| "View in 3D" is missing | The service did not provide a viewer link for this submission. |
| The estimate produces no lines | Message "The AI produced no usable line items. Please try again." — re-run "Generate estimate", adjusting the floor area if needed. |

---

## 6. Summary

- **PDF3D turns a set of plans (PDF / DWG / DXF, + PNG / JPG) into a 3D model and measurements** via the external service **Hover**, presented everywhere under the **white-label "PDF3D"**. Access: **3D DESIGN → PDF3D** menu (`/hover`).
- **A single central account** is connected **once** by the **super-administrator**; all companies then use it. The super-administrator **cannot submit a plan**.
- **Only a company administrator / owner** sees the submission form and incurs the expense. An ordinary user views statuses and deliverables.
- **Three zones**: Connection (central account status), Plan submission (name + files + two-step confirmation), My uploads (table + deliverables of completed models), plus a three-step **Estimate modal**.
- **Two distinct costs**, debited from the prepaid credit balance: **A — the submission** (~US$390, fixed, **idempotent**, refused if the balance is insufficient) and **B — the AI measurement reading** (variable, **billed on each re-reading**). To be **billed back to the client**.
- **Reconstruction in ~24 h** (asynchronous); no deliverable before the **Complete** status. A status never goes backward.
- **Deliverables** of a completed model: roof report (PDF), Pro Premium report (PDF), presentation image, **external 3D viewer link** (no built-in viewer) and **CAD file (XML)** to download.
- **Estimate → quote**: AI measurement reading → **floor-area** confirmation → line generation (target ~$265/sq ft, complete build) → **budgetary quote** in the Sales module, **to be reviewed before sending**.
- **Limits**: 24 files / 80 MB per submission; 10 submissions / company / 24 h (50 for the platform). **Importing the CAD file into the CAD module remains manual**; the JSON measurements are not shown in the interface.
- **Security**: central-account tokens encrypted, deliverables restricted to the official domain, fail-closed webhook (secret in the address), strict per-company isolation, writes blocked in consultation mode.

---

*Verified source files:* `backend/routers/hover.py` (1,932 lines, 13 endpoints), mounted in `backend/erp_api.py` under `/api/erp/v1/hover`; `frontend/src/pages/HoverPage.tsx` (541 lines), `frontend/src/api/hover.ts` (182 lines), `frontend/src/components/hover/HoverEstimateModal.tsx` (293 lines), `frontend/src/components/hover/hoverEstimate.ts`; `frontend/src/i18n/locales/fr/hover.json` and `frontend/src/i18n/locales/en/hover.json` (73 keys); `frontend/src/i18n/locales/fr/nav.json` (the "3D DESIGN" section, "PDF3D" entry).

*Related manuals:* `07-ventes-soumissions.md` (budgetary quote produced by the estimate), `25-outils-dao-modelisation.md` (CAD, same 3D DESIGN section, target of a future CAD import), `24-communication-assistant-ia.md` (AI estimating engine and prepaid credits), `30-configuration.md` (subscription and consultation mode).
