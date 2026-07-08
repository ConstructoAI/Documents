# Module 23 — AI Voice Agent (virtual receptionist)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Route**: `/agent-vocal` — sidebar menu "Voice Agent" (**Communication** section of the navigation bar, alongside "Emails" and "Messaging")
> **Reference code**: `backend/routers/voice.py` (2,941 lines; 1 public webhook endpoint + 15 management endpoints), `backend/routers/voice_admin.py` (690 lines; 5 super-admin endpoints), `backend/voice_provider.py` (165 lines; Vapi adapter), `frontend/src/pages/AgentVocalPage.tsx` (1,559 lines), `frontend/src/api/voice.ts` (282 lines), `frontend/src/components/admin/AdminVoiceCallsTab.tsx`, i18n `frontend/src/i18n/locales/{fr,en}/voice.json`
> **PostgreSQL tables per company (tenant)**: `voice_agent_config`, `voice_calls`, `voice_qualification_questions`, `voice_knowledge_base`, `voice_lookup_attempts`
> **Shared PostgreSQL tables (`public`)**: `voice_phone_routing`, `voice_assistant_routing`, `voice_calls_index`, `voice_admin_access_log`
> **Scope**: this module is an **automated AI telephone receptionist**, named **"Clara"**, that answers your company's inbound calls, **qualifies new prospects** (and creates an opportunity in the Sales module), **reports the status of an existing file** (quote, invoice, project, work order) after identity verification, and can **transfer the call** to a person. The voice, the transcription, and the Claude model are provided by the **Vapi** platform; the ERP orchestrates the configuration, the security, and the call log. It is **not** an outbound dialer, nor a classic voicemail module.

*Terminology note used in this manual:* "endpoint" refers to an API endpoint (a URL that the software calls); "tenant" refers to your company (each company has its own isolated data); "Vapi" is the voice-agent platform that provides the telephony, the speech recognition, and the speech synthesis.

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

The AI voice agent answers the phone on your behalf, day and night, with a natural voice in French (Quebec) and English. Concretely, "Clara":

- **Greets the caller** in your company's name and introduces herself as a virtual assistant (transparency required by Law 25 (Quebec's privacy law)).
- **Qualifies a new prospect**: she asks your questions (nature of the project, type of work, region, timeline, budget, financing, decision-maker, etc.), confirms the answers, then announces that a representative will call back. At the end of the call, the ERP **automatically creates an opportunity** in the Sales module ("Qualification" stage), with a **preliminary B.A.T. pre-qualification** computed from the call.
- **Reports the status of an existing file**: quote, invoice, project, or work order. The caller gives the **document number** and their **name** (or their company's name); the ERP verifies the identity server-side, then announces **the status only**. An amount, a balance, or a total is **never** disclosed over the phone.
- **Transfers the call** to a human number if you configure one and the caller asks for it.
- **Logs every call**: date, duration, language, summary, transcript, recording (if consented), and outcome. You review all of this in the **call log**.
- **Notifies you by email** (optional) after each relevant call.
- **Tracks usage and cost** (number of calls, minutes, estimated cost) in a **Statistics** tab.

### 1.2 How to access it

- Sidebar → **Communication** section → **Voice Agent** (phone icon).
- Address: `/agent-vocal`.
- The page is accessible to **any logged-in user of the company** (see §1.3 for an important note about permissions).

### 1.3 Roles and permissions

- **Opening the page** only requires being logged into the ERP (standard protected route).
- **Point of attention (security).** Unlike several other ERP modules (Accounting, Configuration, Store…) where management is reserved for administrators, **all of the voice agent's management endpoints** (`/voice/config`, `/voice/questions`, `/voice/knowledge`, `/voice/config/sync`, `/voice/calls`, `/voice/stats`) are protected **by authentication only** (`get_current_user`), **with no** administrator-role check. In practice, **any logged-in user of your company** can read and modify the voice configuration and the knowledge base, activate or deactivate the agent, view the call log, and trigger synchronization to Vapi. If you want to restrict these actions, take this into account when assigning accounts; as of today, the interface imposes no administrator safeguard.
- **The cross-company centralized view** (all tenants) is, for its part, strictly reserved for the **platform super-administrator** (see §2.6 and §4.3). A company administrator has **no** access to it.
- In **consultation mode** (suspended subscription / read-only), the module's writes (saving the configuration, activating, editing the questions, synchronizing) are blocked by the ERP's global control; reading remains possible.

### 1.4 The sub-modules (tabs)

The page permanently displays a **status banner** (status, connected number, language, activation button), then a bar of **four tabs**:

| # | Tab | Role |
|---|-----|------|
| 1 | **Call log** *(default)* | List of received calls + detail of a call (summary, transcript, recording) |
| 2 | **Configuration** | Telephony, voice, greeting messages, transfer, notifications, synchronization **+ qualification questions** |
| 3 | **Knowledge base** | Information cards (title / content / language) provided to the agent to answer callers |
| 4 | **Statistics** | Usage and cost: calls, minutes, estimated cost, billable amount, qualification and transfer rates |

### 1.5 What the module does — and does NOT do

The module **does**: answer **inbound** calls, introduce itself in the company's name, qualify a prospect according to your questions, create an opportunity and an interaction in the CRM, pre-fill a B.A.T. qualification, report the status of a file after identity verification, transfer to a human, record and transcribe the call (with consent for the audio), send a summary by email, track cost and usage, and give the super-administrator a centralized view of all calls.

The module **does NOT do**:

- **No outbound calls**: the agent answers; it never dials numbers to prospect.
- **No disclosure of any amount** over the phone (price, balance, total) — this is a strict security rule, enforced server-side.
- **No contractual commitment or firm price** given verbally.
- **No automatic credit debit and no Stripe billing** in this module (see §1.6 and §4.5). The cost is **tracked**, not charged.
- **No Google knowledge base**: the knowledge cards are stored in your company database, not in an external service (see §5, FAQ).
- **No automatic appointment booking in a calendar**: the agent gathers availability, but it is a representative who calls back.

### 1.6 Two operating modes

The agent covers two use cases, both **fully implemented**:

- **Mode A — Qualification (new prospect).** Clara asks your questions, the call is summarized, and an **opportunity** is created at the end of the call if the threshold is reached (a name + at least one useful piece of information).
- **Mode B — Status of an existing file.** Clara asks for the **number** of the document and the **name**, calls one of the four lookup tools, and announces **the status only**. All identity verification, anti-enumeration, and anti-abuse lockout live **server-side** (the agent never decides). Only **usage-based billing** is deferred; the mode itself is complete.

---

## 2. Interface

### 2.1 Header and status banner

At the top of the page: a phone icon, the title **"Voice Agent"** and the subtitle "AI phone assistant — call reception and qualification".

Just below it, a **status banner** that is always visible (regardless of the tab) shows:

- **Status**: a green dot ("Active") or red dot ("Inactive"), accompanied by a badge.
- **Connected number**: the phone number associated with the agent (formatted), or "No number".
- **Default language**: "French" or "English".
- **Activation button** on the right: **"Activate"** (blue) when the agent is inactive, **"Deactivate"** (grey) when it is active. The button shows a loading indicator while saving.

*Note:* the Activate / Deactivate toggle only writes the "active" field; it touches no other setting. The "Active" status signals your intent on the ERP side; whether calls are actually received also depends on the number being connected at Vapi and on the synchronization (see §3.1).

### 2.2 "Call log" tab

This is the tab opened by default. It displays the list of received calls, from most recent to oldest.

**Table columns:**

| Column | Content |
|--------|---------|
| **Date / time** | Start of the call (or record date as a fallback) |
| **Caller number** | The caller's number, formatted, or "--" if withheld |
| **Duration** | Format `minutes:seconds` (e.g. `4:32`), or "--" |
| **Language** | "French" or "English" (detected language) |
| **Outcome** | Colored badge: **Qualified** (green), **Transferred** (blue), **Message** (amber), **Dropped** (grey) |
| **Opportunity** | Link **"Opp. #N"** to the Sales module if an opportunity was created, otherwise "--" |

**Interactions:**

- **Clicking a row** opens the **call detail** (modal window, see §2.3).
- **Clicking the "Opp. #N" link** (Opportunity column) opens the **Sales** module without opening the detail.
- A **"Load more"** button appears when there are more than 50 calls; it appends the next page to the list.
- **Empty state**: "No calls yet."

*The log displays no financial data* (neither cost nor amount): this information lives in the Statistics tab.

### 2.3 Call detail (modal window)

Opened by clicking a row in the log, the "Call detail" window shows:

- **Metadata** (in two columns): Caller number, Start, Duration, Language, Outcome (badge), and **Recording consent** ("Yes" / "No" / "--").
- **Opportunity link**: "Opp. #N" to the Sales module, where applicable.
- **Summary**: the AI-generated call summary (in the language of the call), or "No summary available."
- **Recording**: a native audio player if the call was recorded **and** consent was given; otherwise "No recording available." The audio is loaded only on playback.
- **Transcript**: either a list of speaking turns (speaker + text), or the raw text, or a readable JSON display depending on the format received; "No transcript available." as a fallback.

### 2.4 "Configuration" tab

This tab gathers all of the agent's settings, in sections, followed by the management of the **qualification questions**. On success or error, an alert bar appears at the top.

#### 2.4.1 "Telephony" section

- **Vapi assistant ID**: your assistant's identifier at Vapi (e.g. `161ceedc-ce68-4005-ae8d-473a5d22254a`). Required for web or test calls (with no called number): it links the call to your company. Visible in the Vapi dashboard.
- **Phone number**: the connected number, in international format (e.g. `+1 514 555 0199`).
- **Default language**: dropdown menu "French" / "English".
- **Bilingual agent** (checkbox): indicates that the agent detects and switches to the caller's language. *(In practice, the agent's script always prompts it to switch language if the caller changes; the speech recognition is set to the default language chosen above.)*

#### 2.4.2 "Voice and greeting" section

- **Voice ID**: the provider's voice identifier. Leave blank for the default voice (Azure Neural, the Quebec voice **Sylvie**). An identifier of the form `fr-CA-…` is interpreted as an Azure voice; any other identifier is treated as a native Vapi voice.
- **Greeting message (French)** and **Greeting message (English)**: the first message spoken when the call is answered. If you leave them blank, a default message compliant with Law 25 is used (see §3.1 and §4.6).

#### 2.4.3 "Call transfer" section

- **Transfer number**: qualified calls, or calls that explicitly ask for a person, are transferred to this number. Leave blank to disable transfer (the agent will then take a message). The number is normalized to international format; if it is invalid, transfer is **cleanly disabled** (a wrong number is never dialed).

#### 2.4.4 "Post-call notification" section

- **"Notify me by email after each relevant call" checkbox**: enables the sending of a summary.
- **Notification email**: the address that will receive the summary (outcome, caller, duration, summary). The field is active only if the box above is checked. The email is sent only for calls that carry content (an opportunity was created **or** a summary is present).

#### 2.4.5 "Synchronization" section

- **"Agent managed directly in Vapi (do not synchronize)" checkbox**: check it if this agent's profile (script, voice, greeting) is configured directly in the Vapi dashboard. Synchronization from the ERP is then **disabled** so as not to overwrite that external configuration. This is the case, for example, for a sales profile connected to a demo number.

#### 2.4.6 Action buttons

- **"Save"**: saves all the settings above. A cleared field is genuinely erased (set to empty), not silently retained.
- **"Sync the agent"**: pushes the configuration **and** the active questions **and** the knowledge base to your Vapi assistant. The button is **greyed out** if the "Agent managed directly in Vapi" box is checked. A help text explains the effect, or the locked state, of the synchronization.

#### 2.4.7 "Qualification questions" section

Below the settings, a card lists the questions the agent asks to qualify a call. **The order defines the flow** of the interview.

- **"Add a question" button** (top right): opens the add window (see §2.4.8).
- Each question displays:
  - **up / down arrows** (reordering, saved immediately);
  - the **French text** of the question;
  - an **"Associated field" badge** (e.g. "Project type", "Region"…);
  - an **"Inactive"** badge if the question is disabled;
  - a **"Required"** checkbox (toggle saved immediately);
  - the **Edit** (pencil) and **Delete** (trash, with confirmation) buttons.
- **Empty state**: "No qualification questions. Add one to get started." — but in practice, **13 default questions** are created automatically on first opening (see §4.7).

#### 2.4.8 Add / edit question window

- **Question (French)**: the spoken text (required).
- **Question (English)**: the English variant (optional).
- **Associated field**: a dropdown among **14 predefined keys** (see §4.7). This is the data the question aims to collect; the agent maps it to the correct qualification field.
- **"Required" checkbox**.
- **"Save"** / **"Cancel"** buttons.

### 2.5 "Knowledge base" tab

Information cards the agent uses to answer callers' questions (business hours, service areas, specialties, warranty policy, etc.).

**Table columns:** Title, Content (two-line preview), Language, Status ("Active" / "Inactive"), and the Edit / Delete actions.

- **"Add an item" button**: opens the add window.
- **Add / edit window**: **Title** (required), **Content** (required), **Language** ("French" / "English").
- **Deletion** with confirmation.
- **Empty state**: "No knowledge item. Add one to enrich the agent."

*Only cards marked "Active" are injected into the agent's script during a call.*

### 2.6 "Statistics" tab

Three usage-tracking cards (read-only; amounts are in US dollars, see §4.5).

**Current month** (with the period shown):

- **Calls** (number);
- **Minutes** (total);
- **Estimated cost** (Vapi provider cost);
- **Billable amount** (cost × margin; the note "Cost × 1.3" is shown).

**Totals** (entire history):

- **Total calls**, **Qualification rate** (with the count of qualified), **Transfer rate**, **Total minutes**, **Total estimated cost**, **Total billable**, **Language (FR / EN)**, **Messages only**.

**Monthly trend**: a 6-month table (Month, Calls, Minutes, Estimated cost).

A card footnote reminds you: "Amounts in USD (Vapi provider cost). Automatic credit billing is disabled."

### 2.7 Super-administrator view (outside the tenant page)

This view is **not** in the `/agent-vocal` page. It lives in the **Super-Admin** module (`/admin`), the **"Voice calls"** tab (component `AdminVoiceCallsTab`), and is visible only to the platform super-administrator. It provides a **centralized, cross-company** way to review all the calls of all tenants: filters (company, outcome, language, dates, search), aggregates (total, qualified, transferred, total cost), full detail of a call (transcript, recording if consented), and an **audit log** of the accesses (Law 25 traceability). See §4.3.

---

## 3. Step-by-step workflows

### 3.1 Putting the agent into service for the first time

Prerequisite: your assistant and your number must exist at **Vapi** (done by the Constructo AI team during setup; the `VAPI_API_KEY` and `VAPI_WEBHOOK_SECRET` keys are set on the server side).

1. Open **Voice Agent** → **Configuration** tab.
2. **Telephony** section: enter the **Vapi assistant ID** and the connected **Phone number**. Choose the **Default language** and check **Bilingual agent** if you want the greeting in both languages.
3. **Voice and greeting** section: leave the **Voice ID** blank for the default voice, or enter a specific voice. Write your **greeting messages** (French and English) if you do not want the default message.
4. **Call transfer** section: enter the **Transfer number** to which calls that ask for a person should be routed (optional).
5. **Post-call notification** section: check the box and enter an **email** if you want to receive a summary after each relevant call.
6. Click **"Save"**.
7. Click **"Sync the agent"**: the ERP builds the profile (qualification script from your questions + knowledge base + status tools + greeting) and pushes it to Vapi. *The voice set at Vapi is not overwritten by the synchronization.*
8. Go back up to the **status banner** and click **"Activate"**.

*Tip:* run **"Sync the agent"** again every time you change your questions, your knowledge base, the greeting, or the transfer number, so that Vapi reflects your changes.

### 3.2 Customizing the qualification questions

1. **Configuration** tab → **Qualification questions** section.
2. To **add**: **"Add a question"** button, enter the text (French, and English if desired), choose the **Associated field** from the list, check **Required** if needed, **Save**.
3. To **reorder**: use the **up / down** arrows; the order is saved immediately. This is the order in which Clara will ask the questions.
4. To **make a question required**: check its **"Required"** box directly in the list.
5. To **edit** or **delete**: pencil / trash buttons.
6. Click **"Sync the agent"** (Configuration section) to apply your changes at Vapi.

*Good to know:* the **13 default questions** already cover a complete construction qualification interview (nature of the project, work, building type, region, timeline, motivation, decision-maker, availability, plans/budget, financing, competing bids, call-back contact details, source). Adapt them rather than starting from scratch.

### 3.3 Feeding the knowledge base

1. **Knowledge base** tab → **"Add an item"**.
2. Give a clear **Title** (e.g. "Service areas"), a concise **Content** (e.g. "Greater Montreal, Laval, North Shore and South Shore"), and the **Language**.
3. **Save**. Repeat for each frequent topic (business hours, lead times, warranties, types of work, etc.).
4. Click **"Sync the agent"** so that Clara can use them.

*Only active cards are used.* Keep the content short and factual: it is injected into the agent's script.

### 3.4 What happens when a client calls (new prospect)

This flow is automatic; you have nothing to do during the call.

1. Vapi receives the call on your number and queries the ERP to obtain the agent's profile. The ERP identifies **your company by the called number**, builds Clara (greeting, script, active questions, knowledge base, tools) and returns her in under 7 seconds.
2. Clara greets the caller, introduces herself as an AI virtual assistant (Law 25), asks for the name, then asks your questions **one at a time**.
3. At the end of the call, Vapi sends the report to the ERP. If the threshold is reached (**a name + at least one useful piece of information**), the ERP creates:
   - an **opportunity** in the Sales module (number `OPP-NNNNN`, "Qualification" stage, source "agent_vocal", **amount not estimated**);
   - an **interaction** of type "Call" linked to the opportunity;
   - a preliminary **B.A.T. pre-qualification** (Budget / Authority / Timing, status "In progress") to be reviewed and completed in the CRM grid.
4. The call appears in the **log** with the outcome **"Qualified"** and the link to the opportunity.
5. If you enabled notifications, an **email summary** is sent to the configured address.

*Special cases:* if the call does not reach the threshold, it is still logged (outcome "Message"). If the caller asks for a person and a transfer number is configured, the call is transferred (outcome "Transferred").

### 3.5 What happens when a client asks for the status of their file (Mode B)

1. Clara asks for the **document number** (quote, invoice, project, or work order) and has the caller **repeat or spell it out** to confirm.
2. She asks for the person's **full name** **or** their company's name.
3. The ERP verifies **server-side** that the number really designates a document belonging to your company **and** that the name matches the client on file.
4. If everything matches, Clara announces **the status** (e.g. "your quote DEV-2026-043 has been sent and is awaiting a response"), then reminds the caller that it **cannot** disclose any amount over the phone and that a representative can provide the details.
5. If the file cannot be found **or** the name does not match, Clara gives **exactly the same neutral response** ("I could not confirm this file…") and offers a call-back — the agent never confirms the existence of a file to an unverified person.

*Automatic protections:* after **3 failures** on the same document, **5 failures** from the same caller number, or **30 failures** in total within one hour, verification is temporarily locked (anti-guessing). An overly generic name alone ("Construction", "Inc", the name of a city) is **never** enough to verify an identity; the caller must say all the distinctive words of the name on file.

### 3.6 Reviewing a call and listening to the recording

1. **Call log** tab: click the desired row.
2. Read the metadata, the **summary** and the **transcript**.
3. If a **recording** is present (consent given), use the audio player to listen to it.
4. If an **opportunity** was created, click the link to open it in **Sales**.

*The audio recording is kept only if the caller consented.* Without consent, no file is stored.

### 3.7 Enabling email notices

1. **Configuration** tab → **Post-call notification** section.
2. Check **"Notify me by email after each relevant call"**.
3. Enter the **notification email** (e.g. `ventes@entreprise.com`).
4. **Save**.

After each call that carries content, you will receive an email summarizing the outcome, the caller, the duration and the summary — with, where applicable, the number of the created opportunity.

### 3.8 Reading the usage statistics

1. **Statistics** tab.
2. Review the **current month** (calls, minutes, cost, billable amount), the **totals** (including qualification and transfer rates, FR/EN split) and the **monthly trend**.
3. Remember that the amounts are **in US dollars** and **indicative**: automatic credit billing is disabled in this module (see §4.5).

### 3.9 (Super-administrator) Reviewing the calls of all companies

1. Open the **Super-Admin** module (`/admin`) → **"Voice calls"** tab.
2. Filter by company, outcome, language, dates, or keyword.
3. Open a call to see the full detail; where applicable, listen to the recording (the access is **logged**).
4. Review the **audit log** of the accesses (Law 25 traceability).
5. If technically needed, **reindexing** rebuilds the centralized view from each company's data.

---

## 4. Reference

### 4.1 Endpoints — public webhook (Vapi → ERP)

This endpoint is **public** (outside the ERP prefix) because it is Vapi that calls it. Its security rests entirely on the **signature**, verified before any processing.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/voice/webhook` | Vapi's single entry point. Verifies the signature (`x-vapi-secret` header) **before** any parsing → **401** if invalid; rejects a body larger than **5 MB** → **413**; **400** if the JSON is invalid. Then routes according to `message.type` (see §4.4). |

### 4.2 Endpoints — management (tenant, authentication required)

All mounted under `/api/erp/v1/voice/*`, protected by ERP authentication (but **without** an administrator guard, see §1.3).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/voice/config` | Agent state + configuration (one row, created on demand) |
| PATCH | `/voice/config` | Updates the configuration; handles number→company and assistant→company routing (see §4.8) |
| GET | `/voice/calls` | Call log (without transcript or amount), paginated (default 50, max 200) |
| GET | `/voice/calls/{call_id}` | Detail of a call (transcript, summary, recording if consented) |
| GET | `/voice/questions` | Qualification questions (creates the 13 default questions if empty) |
| POST | `/voice/questions` | Create a question (associated field validated against the allow-list) |
| PUT | `/voice/questions/reorder` | Reorder a batch of questions |
| PUT | `/voice/questions/{id}` | Edit a question (partial fields) |
| DELETE | `/voice/questions/{id}` | Delete a question |
| GET | `/voice/knowledge` | List the knowledge base |
| POST | `/voice/knowledge` | Create a card |
| PUT | `/voice/knowledge/{id}` | Edit a card |
| DELETE | `/voice/knowledge/{id}` | Delete a card |
| POST | `/voice/config/sync` | Push the configuration to Vapi; **409** if "managed directly in Vapi", **400** without an assistant, **502** if Vapi refuses |
| GET | `/voice/stats` | Usage statistics (read-only; default 6 months, max 24) |

### 4.3 Endpoints — super-administrator (cross-company)

All mounted under `/api/erp/v1/admin/voice/*`, protected by `require_super_admin` (a strong guard on the account type, **tamper-proof**; a company administrator never accesses it).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/voice/calls` | Paginated cross-company list (company / outcome / language / dates / search filters + aggregates). Every access is logged. |
| GET | `/admin/voice/calls/{schema}/{call_id}` | Full detail of a call from a company (transcript, recording if consented). Logged. |
| POST | `/admin/voice/calls/{schema}/{call_id}/listen` | Logs the act of **listening** to a recording (returns neither a URL nor a file). **404** without a recording. |
| GET | `/admin/voice/access-log` | Audit log of the accesses (Law 25 transparency) |
| POST | `/admin/voice/reindex` | Rebuilds the centralized view from all companies (maintenance) |

### 4.4 Vapi message types and handling

| `message.type` | Handling |
|----------------|----------|
| `assistant-request` | Builds the "Clara" assistant according to your company (greeting, script, questions, knowledge base, tools) and returns it (< 7 s) |
| `tool-calls` | **Mode B**: status lookup (allow-list of 4 tools, server-side identity verification) |
| `end-of-call-report` | Logs the call, creates the opportunity + interaction + B.A.T. pre-qualification if the threshold is reached, sends the email summary |
| `status-update` and others | Ignored (neutral response) |

### 4.5 Call outcomes and cost calculation

**Possible outcomes** ("Outcome" column of the log):

| Value | Label | Badge | Meaning |
|-------|-------|-------|---------|
| `qualified` | Qualified | Green | Opportunity created (threshold reached) |
| `transferred` | Transferred | Blue | Call transferred to a human |
| `message_only` | Message | Amber | Contact details / message taken, without an opportunity |
| `dropped` | Dropped | Grey | Call dropped |

**Cost and billing:**

- The **actual cost** of the call (orchestration + speech recognition + Claude model + speech synthesis + telephony) is billed **by Vapi**, in **US dollars**. The ERP retrieves it and only **tracks** it.
- The **billable amount** displayed = cost × **1.30** (display margin `_VOICE_BILLING_MARGIN`).
- **No automatic debit** of credits and no Stripe billing is performed by this module: automatic billing is **explicitly deferred** (a USD/CAD currency decision is still to be made). The figures in the Statistics tab are **indicative**.

### 4.6 Default greeting message (Law 25)

If you do not enter a greeting message, Clara uses this default message, which discloses its AI nature and the possibility of recording:

> "Hello, this is Clara, the virtual assistant for [your company]. I am a voice generated by artificial intelligence, and this call may be recorded. How may I help you?"

An equivalent French variant is used for a French-language call. The message that **you** configure always takes priority over this default message.

### 4.7 Default questions and associated fields

**The 13 questions created automatically** (in order) cover: (1) nature of the project, (2) type of work, (3) building type, (4) **region** *(required)*, (5) timeline, (6) motivation / urgency, (7) decision-maker, (8) availability for a meeting, (9) plans / budget, (10) financing, (11) competing bids, (12) **best number and time to call back** *(required)*, (13) source.

**The 14 associated fields** available (strict allow-list; any other key is refused):

| Key | Label |
|-----|-------|
| `project_type` | Project type |
| `work_type` | Type of work |
| `building_type` | Building type |
| `region` | Region |
| `timeline` | Timeline |
| `decision_maker` | Decision-maker |
| `has_plans_budget` | Plans / budget |
| `competing_bids` | Competing bids |
| `contact_info` | Contact details |
| `source` | Source |
| `contact_phone` | Contact phone |
| `urgency` | Urgency |
| `visit_availability` | Availability for a visit |
| `financing_status` | Financing status |

### 4.8 Company resolution and anti-hijacking

- On every call, the company is resolved **by the called number** (shared table `voice_phone_routing`), then, as a fallback, **by the assistant identifier** (table `voice_assistant_routing`, useful for web/test calls with no number). Both of these keys come from Vapi's **signed** message, never from a parameter supplied by the caller.
- When you save a **number** or an **assistant ID** in the configuration, the ERP accepts it only if it is **free** or **already owned by your company**; otherwise the save fails with a **409** ("already associated with another account"). This prevents hijacking a competitor's inbound calls. Clearing the field removes the corresponding routing.

### 4.9 Limits, thresholds and default values

| Item | Value |
|------|-------|
| Maximum webhook body size | **5 MB** (beyond: 413) |
| Maximum call duration | **3,200 s** (~53 min) pushed to Vapi (`maxDurationSeconds`) |
| Log pagination (tenant) | 50 per page (max 200) |
| Mode B lockout window | **1** rolling **hour** |
| Lockout — per document | **3** failures |
| Lockout — per caller number | **5** failures |
| Lockout — company cap | **30** failures |
| Default AI model (at Vapi) | provider **Anthropic**, model **`claude-sonnet-4-6`** |
| Default voice | Azure Neural **`fr-CA-SylvieNeural`** |
| Speech recognition | Deepgram **nova-2** |
| Default language | **`fr-CA`** |
| Cost display margin | **1.30** |
| Cost-tracking currency | **USD** |

### 4.10 Data model

**Your company's tables (created on demand):**

- `voice_agent_config` — the agent's configuration (provider, assistant ID, number, active, language, bilingual, voice, FR/EN greetings, transfer rules, notification email, "managed in Vapi" lock…).
- `voice_calls` — one record per call (unique provider identifier, caller number, timestamps, duration, language, transcript, summary, recording URL, outcome, consent, linked opportunity and interaction, estimated cost).
- `voice_qualification_questions` — your questions (order, FR/EN texts, associated field, required, active).
- `voice_knowledge_base` — your cards (title, content, language, active).
- `voice_lookup_attempts` — log of the identity-verification attempts (Mode B, for the anti-abuse lockout).

**Shared tables (`public`):**

- `voice_phone_routing` — number → company mapping.
- `voice_assistant_routing` — Vapi assistant → company mapping.
- `voice_calls_index` — **cross-company metadata mirror** (no transcript, no audio), for the super-admin view at scale.
- `voice_admin_access_log` — audit log of the super-admin accesses (Law 25).

**Tables from other modules that are populated:** `opportunities`, `interactions`, `prospect_qualifications` (Sales / CRM module). **Tables read in Mode B:** `devis`, `factures`, `projects`, `formulaires`, with `companies` and `contacts`.

### 4.11 Privacy protection (Law 25)

- **Transparency**: Clara introduces herself as an AI and signals the possibility of recording right from the greeting.
- **Consent**: the **audio recording is kept only if the caller consents**; otherwise no file is stored and the URL is hidden even from the super-administrator.
- **Minimization**: the cross-company mirror contains neither transcript nor audio.
- **Traceability**: every super-admin access (list, detail, listen, reindexing) is **logged in the same operation** as the read; no trace, no disclosure.

---

## 5. Integrations and FAQ

### 5.1 Links with the other modules

- **Sales / CRM** — each qualified call creates an **opportunity** ("Qualification" stage, source "agent_vocal"), an **interaction** of type "Call" and a preliminary **B.A.T. pre-qualification**. The log's "Opp. #N" link opens the **Sales** module.
- **Quotes, Invoicing, Projects, Work Orders** — **Mode B** reads the status of these documents (never an amount) after identity verification.
- **Emails** — the **post-call summary** is sent via the ERP's internal SMTP relay (the same one as the Emails module).
- **Super-Admin** — the **"Voice calls"** tab provides the cross-company centralized view and the audit log.
- **Configuration / Subscription** — in **consultation mode** (suspended subscription), the module's writes are blocked.

### 5.2 Frequently asked questions

**Can the agent call my clients (outbound calls)?**
No. The agent answers only **inbound** calls. It never dials numbers.

**Can it give a price, a balance, or a total over the phone?**
No, never. This is a security rule enforced server-side: even in Mode B, only the **status** is announced, never an amount. The agent invites the caller to request a call-back for financial details.

**Who can modify the agent's configuration?**
Currently, **any logged-in user of your company** (there is no administrator guard on this module — see §1.3). Take this into account when assigning accounts.

**Is the knowledge base stored in Google or an external service?**
No. The cards live in your company's `voice_knowledge_base` table, in the ERP database. The module has **no** Google integration. *(If you have ever heard of "Google knowledge bases" for a voice agent, that concerns a demonstration sales agent configured directly in Vapi, outside this module.)*

**Is "Clara" a single agent shared across all companies?**
No. **Clara is the default name**, personalized in the name of **each company** (the greeting and the script say "the virtual assistant for [your company]"). Each company has its own configuration and its own Vapi assistant. There is no separate "clone" object: one company = one configuration row + its routed assistant.

**Is the demo number (for example +1 936 587-1141) hard-coded?**
No. The number → company routing is **data-driven** (table `voice_phone_routing`). A "demo number" is simply a row in this table, not an application constant.

**What happens if Vapi replays the same end-of-call report twice?**
Nothing untoward: the handling is **idempotent** (uniqueness on the provider's call identifier). The call is not duplicated, and an already-created opportunity is never recreated or downgraded.

**Why does the Statistics tab show amounts in USD?**
Because Vapi bills in US dollars. The amounts are **indicative**; automatic credit billing is disabled in this module.

**I checked "Agent managed directly in Vapi": why is "Sync" greyed out?**
That's intentional. This lock prevents the ERP from overwriting a profile that you maintain directly in the Vapi dashboard. Uncheck it to resume synchronization from the ERP.

**Are some settings displayed but have no effect?**
Yes, a few fields are saved but **not yet used** in building the agent: the **business hours** (`business_hours`) and the **AI provider / model** fields (`llm_provider` / `llm_model`). The applied model always remains **Anthropic's `claude-sonnet-4-6`** by default. So do not expect any functional effect from them for now.

**Can a caller access someone else's file in Mode B?**
No. They must provide the **exact number** of the document **and** a name that matches the client on file (all the distinctive words). "Not found" and "name does not match" give the **same** neutral response, and anti-guessing locks trigger after a few failures.

### 5.3 Common troubleshooting

| Symptom | What to check |
|---------|---------------|
| The agent doesn't answer calls | Check that the status is **Active**, that the **number** is correctly entered and connected at Vapi, and run **"Sync the agent"** again. |
| A call doesn't appear in the log | The end-of-call report may not have been received yet, or the called number is not routed to any company. |
| "Sync" returns an error | **409** = "managed in Vapi" box checked; **400** = no assistant ID; **502** = Vapi refused (retry, check the assistant ID). |
| Audio recording missing | The caller did not consent: in that case, no audio is kept (Law 25). |
| Cannot save a number | **409** "already associated with another account": the number or the assistant ID is owned by another company. |
| A new question isn't being asked | Make sure it is **active**, then **sync the agent**. |

---

## 6. Summary

- **AI virtual receptionist "Clara"** for **inbound** calls: greeting in your company's name, prospect qualification, file status, transfer to a human — in French and English.
- **Access**: **Communication → Voice Agent** menu (`/agent-vocal`). Four tabs: **Call log**, **Configuration** (+ questions), **Knowledge base**, **Statistics**, topped by a **status banner** with an Activate / Deactivate toggle.
- **Mode A (qualification)**: at the end of a relevant call, an **opportunity** (Sales, Qualification stage), an **interaction** and a **B.A.T. pre-qualification** are created automatically; the amount is never guessed.
- **Mode B (file status)**: quote / invoice / project / work order, with **100% server-side identity verification**, anti-enumeration and anti-abuse lockout; **never any amount** disclosed.
- **Security**: Vapi signature verified **before** any processing, company resolved by the **called number**, anti-hijacking protection on the routing, script injection neutralized.
- **Law 25**: AI transparency at the greeting, **audio kept only with consent**, cross-company mirror without transcript or audio, super-admin accesses **logged**.
- **Money**: the Vapi cost (USD) is **tracked**, not charged; **no credit debit and no Stripe** in this module; billable amount displayed = cost × 1.30 (indicative).
- **Points of attention**: management is **not** reserved for administrators (any logged-in user can modify); the **business hours** and the **AI provider/model** fields are stored but **not used** (the model remains `claude-sonnet-4-6`).
- **Super-admin view**: "Voice calls" tab of the Super-Admin module, cross-company, with an audit log and reindexing.

---

*Verified source files:* `backend/routers/voice.py` (2,941 lines), `backend/routers/voice_admin.py` (690 lines), `backend/voice_provider.py` (165 lines), `frontend/src/pages/AgentVocalPage.tsx` (1,559 lines), `frontend/src/api/voice.ts` (282 lines), `frontend/src/components/admin/AdminVoiceCallsTab.tsx`, `frontend/src/components/layout/Sidebar.tsx`, `frontend/src/i18n/locales/fr/voice.json`.

*Related manuals:* `05-gestion-crm-opportunites.md` (opportunities and B.A.T.), `07-ventes-soumissions.md` and `08-ventes-projets.md` (documents read in Mode B), `11-operations-bons-de-travail.md`, `14-operations-comptabilite.md` (invoices), `21-communication-emails.md` (email relay), `22-communication-messagerie.md`, `30-configuration.md` (subscription and consultation mode).
