# Module 23 — Emails (IMAP/SMTP)

> **Version**: 3.0 (complete overhaul verified against the source code, July 2026)
> **Route**: `/emails` — sidebar "Emails" menu (Communication section of the navigation bar)
> **Reference code**: `backend/routers/emails.py` (6,993 lines, 29 endpoints), `backend/routers/emails_ai.py` (332 lines, 1 endpoint), `frontend/src/pages/EmailsPage.tsx`, `frontend/src/components/emails/*` (`EmailAccountsPanel`, `EmailSyncPanel`, `EmailAIPanel`, `EmailAIComposeButton`, `EmailsAssistantTab`), `frontend/src/api/emails.ts`, `frontend/src/store/useEmailsStore.ts`
> **PostgreSQL tables (one copy per company / tenant)**: `email_accounts`, `emails`, `email_attachments`, `email_templates`, `email_threads`, `email_sync_log`
> **Scope**: this module is a **multi-account IMAP/SMTP email client** integrated into the ERP, Outlook-style. It handles **external emails** (Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, or any IMAP/SMTP server) received and sent from the user's mail servers, plus an **internal address** specific to each company. It is **not** instant messaging between users (see the Messaging module) or the ERP's notification system (see the Administration module).

*A note on the terminology used in this manual:* "endpoint" denotes an API endpoint; "tenant" denotes your company (each company has its own isolated data).

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

Manage your business emails without leaving the ERP:

- **Several accounts per company**: Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, or any other IMAP/SMTP server, used simultaneously.
- **Two connection methods**: app password (encrypted at rest by Fernet) or **OAuth2** (Google and Microsoft 365, no app password).
- **On-demand IMAP receiving** (three modes) and **automatic background synchronization**.
- **Real SMTP sending** with attachments, pre-filled templates, and signature.
- **Company-specific internal address**: emails sent to this address arrive in your inbox thanks to a webhook (n8n and Mailgun integration).
- **Reading, search, favorites (star), and trash**.
- **Two Claude artificial-intelligence functions**: one to **draft and reply** (assisted), one to **consult and summarize** your mailbox (read-only).
- **Automatic CRM linking**: on synchronization, an incoming email is attached to the corresponding contact or company (by exact address, otherwise by domain).

### 1.2 How to access it

- Sidebar navigation → **Communication** section → **Emails** (envelope icon). The immediate neighbors are **Messaging** and **Voice agent**.
- Address: `/emails`.
- The page is accessible to any logged-in user of the company.

### 1.3 Roles and permissions

- **Each email account belongs to the user who created it** (`user_id`). An account marked as "shared" (with no owner) remains visible to all users of the company; this is the case for the **internal address**, which is shared by nature.
- **No user sees another user's personal accounts** within the same company, not even an administrator. There is no administrator override on personal accounts.
- **Only one action is reserved for administrators**: the **"Restore"** button (reactivating deactivated accounts). All other actions are open to the account owner.
- In **consultation mode** (suspended subscription / read-only), write operations — sending, creating or editing an account, moving, synchronizing, replying automatically — are blocked by the ERP's global control; reading remains possible.

### 1.4 The sub-modules (tabs)

The module has **8 tabs** and **1 additional folder** (the Trash, accessible via the folder list, with no dedicated tab):

| # | Tab | Role |
|---|--------|------|
| 1 | **Inbox** | Read, search, star, and delete received emails; reply |
| 2 | **New message** | Opens the compose window (does not change tab) |
| 3 | **Sent** | Sent (or failed) emails |
| 4 | **Drafts** | See §2.1 and §5 — cannot be filled manually from the interface |
| 5 | **Templates** | View the pre-installed email templates (read-only) |
| 6 | **Configuration** | Manage IMAP/SMTP accounts and OAuth connections |
| 7 | **Synchronization** | Receive IMAP emails and view synchronization history |
| 8 | **AI Assistant** | Ask questions about your mailbox (read-only, no sending) |

*Note:* the internal code comments contradict each other (the file header states "8 tabs," a comment further down says "7 tabs"); the actual displayed count is indeed **8 tabs**. The **Trash** is a fourth **folder** in the left-hand list, not a tab.

### 1.5 What the module does — and does NOT do

The module **does**: configure several accounts, connect by app password or by OAuth, synchronize IMAP (on demand and automatically), actually send via SMTP with attachments, receive on an internal address via webhook, sanitize received HTML, link incoming emails to the CRM, draft and reply with AI, consult the mailbox with a read-only AI assistant.

The module **does NOT** (limits detailed in §5):

- No **"Save as draft"** button in the compose window.
- **Read-only templates**: you cannot create, edit, or delete a template from the interface.
- No **"Reply All"** or **"Forward"**; the **"Archive"** button was intentionally removed.
- No manual **"Mark as read / unread"** button: reading automatically marks the email as read.
- No **discussion-thread view** (grouped conversation).
- No **PDF or CSV export**, and no **printing** in this module.
- Local states (read, star, trash) are **not reflected** on the origin IMAP server.

---

## 2. Interface

### 2.1 Tab bar

At the top of the page, a bar presents the 8 tabs described in §1.4. Notable elements:

- A **blue** unread-email badge appears on the **Inbox** tab when there is at least one unread message.
- The **New message** button opens a modal window and **does not change tab** (you stay on the current tab).
- The **Inbox**, **Sent**, and **Drafts** tabs share the same **three-panel layout** (folders / list / reading). The **Trash** uses the same layout but is reached via the folder list in the left panel.

### 2.2 Three-panel layout (Inbox)

#### 2.2.1 Left panel — Folders

- Full-width **"New message"** button.
- **"Internal address"** banner (read-only) showing your company's internal address. Emails sent to this address will arrive in the inbox (the interface refreshes every 60 seconds).
- **Four folders**, each with a counter: **Inbox**, **Sent**, **Drafts**, **Trash**. The counter shows the number of unread messages (blue badge) or the total.
- On mobile, a header shows the current folder title and a "New" button.

#### 2.2.2 Middle panel — Message list

- **Search** field ("Search...") with a 400 ms debounce before triggering. The search covers the subject, sender, recipient, and text body.
- Each row shows: an **unread dot** (light blue), the **sender** (or the recipient in Sent and Drafts), the **relative time**, the **subject** (or "(no subject)"), and the **star** (favorite) and **paperclip** (attachments) icons.
- Distinct **empty states** depending on the folder: "No email received / sent", "No draft", "No email", with a reminder about the internal address.
- **Pagination** of 50 messages per page as soon as the total exceeds 50: **"Prev."** button, **"Page X / Y"** indicator, **"Next"** button.

#### 2.2.3 Right panel — Reading a message

- **Header**: subject, sender (name and address in angle brackets), recipients ("To:", "Cc:"), relative date.
- **Actions**: **star** button (add / remove the favorite), **trash** button (with confirmation "Do you want to move this message to trash?", or "Do you want to permanently delete this message?" if the email is already in the Trash), **"Reply"** button, close button.
- **Body**: the HTML is **sanitized** before display (see §4.10) or, failing that, the plain text ("(no content)" if empty).
- **AI panel**: shown **only for an inbox email**. It gives access to the Analyze, Suggest a reply, and Auto-reply functions (see §2.6).
- **Attachments**: "Attachments (N)" section with a download button per file (name and size in KB).
- **Empty state** (no message selected): "Select an email to read" and a reminder of your address.

### 2.3 "New message" window (compose)

The compose window presents, in order:

1. **Sender account** ("From"): dropdown with the **"Default account"** option and your active accounts (excluding the internal address). A "(default)" badge and the provider name accompany each account.
2. **Template**: dropdown ("Select a template...") that automatically fills the subject and body from the chosen template (the HTML is converted to text).
3. **To** (required): recipient(s), separated by commas.
4. **Cc** (carbon copy).
5. **Bcc** (blind carbon copy).
6. **"Draft with AI"** button (see §2.7).
7. **Subject**.
8. **Message** (8-line text area).
9. **Attach a file**: multiple file selection, **5 files maximum** on the interface side; each attachment appears as a chip (name, size, remove button).
10. **Actions**: **"Cancel"** and **"Send"** (the Send button is disabled while the "To" field is empty or during sending).

Important behaviors:

- The text body is converted to HTML (escaping and line breaks) before sending.
- If SMTP sending fails, the window **stays open** and retains your input, so you can retry.
- There is **no** "Save as draft" button (see §5).

### 2.4 "Configuration" tab (accounts)

Title: **"Email accounts"**, subtitle "Manage your IMAP/SMTP accounts, Gmail OAuth and Microsoft 365."

- **Header buttons**:
  - **"Restore"** — reactivates deactivated accounts that still have credentials (administrators only).
  - **"New account"** — opens the creation window.
- **Expandable OAuth block** ("Quick connection with OAuth (recommended)"): **"Connect Gmail (OAuth)"** and **"Connect Microsoft 365 (OAuth)"** buttons. Each button is **grayed out** if the server credentials are not configured (the tooltip then indicates "GOOGLE_CLIENT_ID/SECRET not configured on Render" or "MS_CLIENT_ID/SECRET not configured on Render"). A click redirects to the provider's authorization page.
- **Account list** (one card per account): name, address, IMAP/SMTP line (server and port), last synchronization with its status and any error. Possible **badges**: **Default** (star), provider, **OAuth** (green), **Password** (padlock), **Auto-sync** (green lightning bolt) or **Sync off** (gray).
- **Per-account actions**: **Test** (checks IMAP and SMTP separately and displays the result), **Edit** (pencil), **Deactivate** (trash).
- **Empty state**: "No account configured" and 'Click "New account" to get started.'

#### 2.4.1 Account creation / edit window

Title "New email account" or "Edit account". Fields:

- **Account name** (required, e.g. "Main Gmail").
- **Email address** (required; not editable when editing). When you leave this field, the **provider is automatically detected** from the domain.
- **Provider**: dropdown (Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, Other). Shows "(detecting...)" during detection, plus instructions and a "Setup guide" link.
- **IMAP block (receiving)**: Server (required), Port, User (default = the email address), **SSL** checkbox (default: port 993, enabled).
- **SMTP block (sending)**: Server (required), Port, User, **STARTTLS** checkbox (default: port 587, enabled).
- **App password**: the label adapts to the context —
  - "App password *" (creation),
  - "New password (leave blank to keep)" (editing an already-authenticated account),
  - "App password * (required to activate the account)" (editing an account without authentication).
  An alert box appears if the account has no saved authentication.
- **HTML signature** and **Text signature** (text areas).
- **Auto-sync** and **Default account** checkboxes.
- **Folders to sync** field (default "INBOX").
- A **warning** appears if you create an account without a password ("Use OAuth or enter an app password").
- **Cancel** and **Create** buttons (or **Save** when editing).

#### 2.4.2 Deactivation confirmation

Title "Deactivate this account?", message: "The account will be deactivated (soft-delete). Emails remain accessible for reading, but the account will no longer be able to send or synchronize." **Cancel** and **Deactivate** buttons.

*Deletion is always a deactivation (soft-delete):* no permanent account deletion. Emails already received remain readable.

### 2.5 "Synchronization" tab

Title "Synchronization", subtitle "Receive IMAP emails from your external accounts."

- **Synchronize all accounts** — three buttons:
  - **"New only"** (mode `new`): unread messages since the last synchronization.
  - **"Last 50 emails"** (mode `recent`): initial catch-up (read and unread).
  - **"All (max 200)"** (mode `all`): up to 200 emails per account; this button asks for confirmation before running.
  Explanatory notes accompany each button. A warning appears if there is no external account (synchronization does not apply to the internal address).
- **Synchronize a specific account** — per account: **"New"** and **"Last 50"** buttons.
- **Synchronization history** — **"Refresh"** button, then one line per run: status badge (**OK** in green, **Error** in red, **In progress** in blue), account name, "+N email(s)", "N error(s)", start and end times, optional error message. Empty state: "No synchronization recorded".

### 2.6 AI panel under a received email (drafting and replying)

Visible only for an **inbox** email. Title "Construction AI Assistant", "Claude" badge. Options: **Tone** (Professional / Friendly / Formal), **Sender account** (for the automatic reply), and **Additional context** (optional). Three buttons:

- **"Analyze"**: returns the **urgency** (high / medium / low), the type, the sentiment, a summary, the list of **required actions** (action and due date) and **alerts**.
- **"Suggest a reply"**: proposes 1 or 2 reply drafts (title, length, body), with a "CRM data used" badge, the client context, a **"Use"** button (which fills the compose window) and "To include" / "To avoid" lists.
- **"Auto-reply"** (alert button): opens a confirmation ("The AI will **generate AND send** a reply..." with the warning "This action is irreversible"), then **generates and sends the reply without manual validation**. The result shows "Reply sent automatically" or "Send failed", a confidence score, the subject, the body, an AI note, and the optional SMTP error.

*Auto-reply safeguards:* it only works on an **incoming** email; it **refuses** to act (409 error) if an automatic reply has already been sent in the same thread or to the same sender within the last 24 hours (anti-loop).

### 2.7 "Draft with AI" button (in compose)

In the compose window, the **"Draft with AI"** button opens a "Draft with Construction AI" window: **Instructions** field (at least 5 characters), **Recipient** (optional, to personalize via the CRM) and **Tone**. The **"Generate"** button produces a subject and a body, plus advice on the best time to send. Two buttons apply the result to the compose window: **"Use this version"** and **"Use the short version"**.

### 2.8 "AI Assistant" tab (consultation, read-only)

Title "AI Assistant — Emails", subtitle "Consult, summarize, and find your emails (read-only, no sending)." A chat area with three starter examples:

- "How many unread emails do I have?"
- "Summarize the latest emails received from this client."
- "Find the emails that mention a quote."

Multiline input (Enter to send, Shift+Enter for a line break), **Send** button, "Analyzing…" indicator.

*Strict scope:* this assistant **only reads** your emails; it writes nothing and sends no email. It has **no** access to accounts (IMAP/SMTP credentials, OAuth tokens) or to any HR, payroll, or security data — its reading is limited to the `emails`, `email_threads`, and `email_templates` tables. Not to be confused with the assisted drafting of §2.6 and §2.7, which are separate functions.

### 2.9 "Templates" tab

Title "Email templates". **Read-only list**: for each template, the name, a category badge, "code: ...", the subject (in italics), and the **available variables** in `{{variable}}` format. Empty state: "No template". These templates feed the compose dropdown (§2.3). No template creation, editing, or deletion is possible from the interface (see §5).

---

## 3. Step-by-step workflows

### 3.1 Set up an account with an app password (Gmail example)

1. **Configuration** tab → **"New account"**.
2. Enter a **name** (e.g. "Main Gmail") then the **email address**. The provider is detected automatically when you leave the address field (the IMAP and SMTP servers pre-fill).
3. Generate an **app password** with your provider (for Gmail with two-step verification: `myaccount.google.com/apppasswords`) and paste it into **App password**. Do **not** use your usual login password.
4. Check the **SSL** (IMAP, port 993) and **STARTTLS** (SMTP, port 587) checkboxes, adjusting servers and ports if needed.
5. Click **Create**. The password is encrypted at rest (see §4.10).
6. On the account card, click **Test** to validate IMAP and SMTP, then go to the **Synchronization** tab to receive emails.

### 3.2 Set up an account via OAuth (Gmail or Microsoft 365)

1. **Configuration** tab → expand the **"Quick connection with OAuth (recommended)"** block.
2. Click **"Connect Gmail (OAuth)"** or **"Connect Microsoft 365 (OAuth)"**. If the button is grayed out, the server credentials are not configured — contact your administrator.
3. Authorize access with the provider (Google requests mail access; Microsoft requests IMAP and SMTP).
4. On return, you are redirected to the Emails tab with a success message. The account appears with the **OAuth** (green) badge. No app password is required; the token is refreshed automatically.

*OAuth advantage:* no app password to manage, and the access token renews on its own (the refresh token is also encrypted at rest).

### 3.3 Set up an account manually (company domain or "Other" provider)

1. **Configuration** tab → **"New account"** → enter the address. A company domain (for example hosted at GoDaddy or Microsoft 365) may be detected as **Other**.
2. Manually choose the correct **provider** in the dropdown, or directly enter the IMAP and SMTP servers and ports provided by your host.
3. Fill in the **User** if different from the address, the **password**, then **Create**.
4. **Test**, then **Synchronize**.

### 3.4 Test a connection

On an account card, the **Test** button checks **IMAP then SMTP separately** and displays for each "OK" or the error encountered. No email is modified by the test (the OAuth token may be refreshed along the way). Use this button whenever a synchronization or send fails, to isolate the problem.

### 3.5 Receive your emails (synchronization)

1. **Synchronization** tab.
2. For a first load, click **"Last 50 emails"** (catch-up). After that, **"New only"** is enough day-to-day. To import everything, **"All (max 200)"** (with confirmation).
3. The emails appear in the **Inbox**. Each run is traced in the **history** (status, number of new emails, errors).

*Automatic synchronization:* in addition to the manual buttons, a background process periodically synchronizes the accounts whose **Auto-sync** is enabled ("New only" mode). The default interval is 15 minutes (configurable server-side). The internal address is not affected by IMAP synchronization.

### 3.6 Understanding the internal address

Each company has an **internal address** created automatically, displayed read-only above the folder list. Emails sent to this address arrive in the inbox via a **webhook** (n8n and Mailgun integration); the interface refreshes every 60 seconds. This address notably serves as a fallback destination when you send without having configured an external account (see §3.8).

### 3.7 Read an email

Clicking a row in the list opens the message in the right panel. The email is **automatically marked as read** (the list and badges update immediately). The HTML body is sanitized before display. For an inbox email, the **AI panel** (Analyze / Suggest a reply / Auto-reply) appears below the message.

### 3.8 Compose and send an email

1. **New message**.
2. Choose the **sender account** ("From"). If you leave **"Default account"** and you have no external account, sending goes through the **platform relay**: the email then leaves from the service address `info@constructoai.ca` **on behalf of your company**, and the reply address (Reply-To) is your internal address. To send from your own address, select one of your configured accounts.
3. (Optional) Choose a **template**, or click **"Draft with AI"** (§2.7).
4. Fill in **To**, optionally **Cc** and **Bcc**, the **Subject** and the **Message**.
5. (Optional) **Attach a file**: up to 5 files (size limits in §4.3).
6. **Send**. The email is saved to **Sent** on success. On SMTP failure, the window stays open and retains your input.

### 3.9 Reply to an email

In the reading panel, **"Reply"** opens the pre-filled compose window: **To** = the sender, **Subject** = "Re: ...", **body** = quote of the original message. The receiving account is preselected as the sender (excluding the internal address). There is no "Reply All" or "Forward" (see §5).

### 3.10 Draft an email with AI

1. In **New message**, click **"Draft with AI"**.
2. Describe what you want to communicate (at least 5 characters). Adding a **recipient** lets the AI personalize via the CRM. Choose a **tone**.
3. **Generate**, then apply with **"Use this version"** or **"Use the short version"**.
4. Review and adjust before sending.

### 3.11 Analyze, suggest a reply, or reply automatically

Under a received email (AI panel, §2.6):

- **Analyze** to obtain urgency, type, sentiment, summary, required actions, and alerts.
- **Suggest a reply** to obtain drafts; **"Use"** fills the compose window so that you validate and send it yourself.
- **Auto-reply** so the AI **drafts and sends** immediately (after confirmation). Reserve for simple cases: the action is irreversible, and the anti-loop only prevents repeated sends within the same thread.

### 3.12 Query your mailbox with the AI Assistant

**AI Assistant** tab → ask a question in natural language ("How many unread emails?", "Summarize this client's latest emails", "Find the emails that mention a quote"). The assistant reads your emails and answers; it **sends nothing** and **does not access** account credentials.

### 3.13 Star, delete, and empty the Trash

- **Star**: star button in the reading panel to mark / remove a favorite.
- **Delete**: trash button → the message goes to the **Trash** (with confirmation). A message **already in the Trash** is **permanently deleted** on the second pass (with purging of attachments).
- The **Trash** opens from the folder list in the left panel.

*Reminder:* these actions are local to the ERP; they are not reflected on the origin IMAP server.

### 3.14 Download an attachment

In the reading panel, **"Attachments (N)"** section, click a file to download it (it is served from the database; maximum download size: 25 MB).

### 3.15 View the synchronization history

**Synchronization** tab → **"Synchronization history"** section → **"Refresh"**. Each line indicates the status, the account, the number of new emails, the errors, and the start and end times.

---

## 4. Reference

### 4.1 Endpoints (30 total)

Two routers, mounted under the `/api/erp/v1` prefix: `emails.py` (29 endpoints, base `/emails`) and `emails_ai.py` (1 endpoint, base `/emails/ai`).

**Accounts (8)**

| Method and path | Purpose |
|---|---|
| GET `/accounts` | List of active accounts; creates the internal address if no account |
| GET `/providers` | Pre-filled settings per provider + OAuth availability |
| GET `/providers/detect?email=` | Detects the provider from the domain |
| POST `/accounts` | Creates an IMAP/SMTP account (encrypted password) |
| PUT `/accounts/{id}` | Edits an account (empty password = kept) |
| DELETE `/accounts/{id}` | Deactivates an account (soft-delete) |
| POST `/accounts/restore-legacy` | Reactivates deactivated accounts (administrator) |
| POST `/accounts/{id}/test` | Tests IMAP and SMTP separately |

**Messages (7)**

| Method and path | Purpose |
|---|---|
| GET `/messages` | Paginated list (folder, search, filters, ≤ 200/page) |
| GET `/messages/{id}` | Email detail; marks it as read |
| PUT `/messages/{id}/read` | Marks as read |
| PUT `/messages/{id}/star` | Toggles the star |
| PUT `/messages/{id}/move` | Moves to an allowed folder |
| DELETE `/messages/{id}` | Trash, or permanent deletion if already in Trash |
| POST `/messages/send` | Send (multipart, attachments, templates) |

**Others (14)**

| Method and path | Purpose |
|---|---|
| GET `/templates` | List of templates (read-only) |
| GET `/attachments/{id}/download` | Downloads an attachment (≤ 25 MB) |
| GET `/threads/{thread_id}` | Messages of a thread (not exposed in the interface) |
| GET `/stats` | Unread / total counters per folder |
| POST `/webhook/inbound` | Receiving (public; n8n Bearer or Mailgun HMAC) |
| GET `/oauth/{provider}/auth-url` | Starts the OAuth connection |
| GET `/oauth/{provider}/callback` | OAuth callback (public; HMAC-signed state) |
| POST `/accounts/{id}/sync` | Synchronizes an account |
| POST `/sync/all` | Synchronizes all accounts |
| GET `/sync-history` | Synchronization history |
| POST `/ai/suggest-reply` | AI: suggests a reply |
| POST `/ai/analyze` | AI: analyzes an email |
| POST `/ai/draft` | AI: drafts a message |
| POST `/ai/auto-reply` | AI: **generates and sends** a reply |
| POST `/emails/ai/chat` | Consultation AI Assistant (read-only) |

### 4.2 Folders and tabs

- **Tabs (8)**: Inbox, New message, Sent, Drafts, Templates, Configuration, Synchronization, AI Assistant.
- **Sidebar folders (4)**: Inbox, Sent, Drafts, Trash.
- **Server-side recognized folders (6, fixed list)**: `inbox`, `sent`, `drafts`, `trash`, `archive`, `spam`. Only the first four are displayed; `archive` and `spam` exist but have no navigation button. There are no custom folders.

### 4.3 Limits and quotas

| Item | Limit |
|---|---|
| Attachments when sending | **5 files**, **10 MB per file**, **20 MB total** |
| Subject of a sent email | 998 characters |
| Body of a sent email | 5,000,000 characters |
| Attachment download | 25 MB |
| Received attachment (kept on synchronization) | 25 MB |
| Received HTML body (sanitized) | 30 MB |
| List pagination | 50 per page (up to 200 per API request) |

### 4.4 Synchronization modes

| Mode | Button | What is retrieved |
|---|---|---|
| `new` | "New only" | New since the last synchronization (on the very first pass: up to 100) |
| `recent` | "Last 50 emails" | The 50 most recent (read and unread) |
| `all` | "All (max 200)" | Up to 200 emails per account |

Duplicate prevention: the same email is never imported twice (identified by the account and the message ID).

### 4.5 Supported providers

**Pre-filled settings**: Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, and **Other** (manual configuration). The provider is detected from the address domain; a custom company domain may be classified as "Other" and must then be chosen manually.

**OAuth available** for **Google (Gmail)** and **Microsoft 365**. The OAuth buttons remain grayed out as long as the credentials are not configured server-side (`GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`, `MS_CLIENT_ID` / `MS_CLIENT_SECRET`, plus the callback address). Other providers use an app password.

### 4.6 Pre-installed templates

Six templates are installed by default (system emails, not editable from the interface); each contains `{{...}}` variables replaced at send time:

| Code | Usage |
|---|---|
| `devis_envoye` | Sending a quote / estimate |
| `facture_envoyee` | Sending an invoice |
| `facture_rappel` | Payment reminder / follow-up |
| `projet_update` | Project update |
| `demande_prix` | Price request (supplier) |
| `inscription_bienvenue` | Welcome email |

The exact variables of each template are visible in the **Templates** tab (§2.9). At send time, the variables are substituted and any unresolved `{{...}}` markers are removed.

### 4.7 Artificial-intelligence functions (billing)

- **Two distinct surfaces**:
  - **Drafting / replying** (`/ai/analyze`, `/ai/suggest-reply`, `/ai/draft`, `/ai/auto-reply`) — the last one **generates and sends**.
  - **Consultation** (`/emails/ai/chat`) — **read-only**, no sending.
- **Model**: Claude Sonnet. **Billed cost** = (input tokens × 0.003 + output tokens × 0.015) / 1000 × **1.30** (30% markup), debited from the company's **prepaid AI credits**. Each assistant response returns its cost and the remaining balance.
- **Actual blocking**: access is not blocked by role (the `check_ai_guard` guard lets any authenticated user through). The only real gate is the **credit balance**: if the balance is insufficient, the call is refused (402 error). A debit failure afterward is logged but does not block the response.
- **AI security**: the drafting functions run read-only on the company's database (no write action is allowed to the AI, as a defense against injection through the content of an external email). The consultation AI Assistant is additionally limited to three tables (`emails`, `email_threads`, `email_templates`) and **excludes** accounts (credentials, tokens) and any HR / security data.

### 4.8 Rate limits (per IP address)

| Endpoint | Limit |
|---|---|
| `/emails/ai/chat` (AI Assistant) | 20 per minute |
| `/emails/messages/send` (sending) | 30 per minute |
| `/emails/ai/analyze`, `/suggest-reply`, `/draft`, `/auto-reply` | No dedicated limit (general limit of 1500/min) |

*Note:* the four AI drafting functions have no dedicated cap, unlike the other AI chats in the ERP. Use them with discernment.

### 4.9 Statuses, badges, and indicators

| Item | Meaning |
|---|---|
| Blue dot (list) | Unread email |
| Star | Favorite |
| Paperclip | Contains attachments |
| Default badge (account) | Default sender account |
| OAuth badge (green) | Account connected via OAuth |
| Password badge (padlock) | Account connected via app password |
| Auto-sync badge (green lightning bolt) / Sync off (gray) | Automatic synchronization enabled / disabled |
| History: OK / Error / In progress | Result of a synchronization |

### 4.10 Encryption and security

- **Encryption at rest (Fernet)**: the IMAP/SMTP app passwords **and** the OAuth tokens (access and refresh) are encrypted in the database, with a key derived from a server variable. The API never exposes secrets (it only returns "has a password" / "has an OAuth" indicators).
- **Sanitization of received HTML**: the content is cleaned at display time (real HTML parser, allowlist of tags, attributes, and URL schemes, forcing `rel="noopener"` against tab hijacking) and already cleaned server-side on receipt. The content of dangerous tags (script, style, etc.) is removed, text included.
- **Receiving (webhook)**: public endpoint but cryptographically protected — Bearer token (n8n) or HMAC signature (Mailgun, with replay protection). It always returns success to avoid retry storms.
- **OAuth callback**: public endpoint protected by a time-limited HMAC-signed state; the account is attached to the original user (account-theft protection).
- **SSRF protection**: the connection refuses any IMAP/SMTP server that would point to an internal network address.
- **Isolation**: each request is scoped to your company; accounts are filtered by owner.

---

## 5. Integrations and FAQ

### 5.1 Integration with the CRM

On synchronization (and on webhook receipt), an incoming email is **automatically linked** to the CRM: first to the **contact** whose address matches exactly, otherwise to the **company** whose domain matches. Public domains (gmail, outlook, yahoo, etc.) are ignored to avoid false matches. The client context also serves to personalize the AI's suggestions.

### 5.2 Integration with notifications

When an email is received on the **internal address** (webhook), a **notification** is sent to the company's administrators, with a link to the Emails module.

### 5.3 Distinction from the other communication modules

- **Messaging** (internal, between ERP users): different from this module, which handles external emails.
- **Voice agent**: neighbor in the sidebar, with no functional link to emails.
- The **Quotes**, **Invoices**, and other modules have their own sending; this module does not replace them.

### 5.4 What is not possible (known limits)

- **Save a draft from the compose window**: there is no "Save as draft" button. The Drafts folder therefore does not fill on demand from the interface (it can receive a failed send, or drafts synchronized from the server).
- **Manage templates**: templates are read-only; no adding, editing, duplicating, or deleting from the interface.
- **Reply All / Forward**: not available (only "Reply" exists). **Archive** was intentionally removed.
- **Manually mark as read / unread**: reading automatically marks as read; there is no reverse button.
- **Discussion thread**: no grouped conversation view.
- **Advanced filters** (by read / unread, by favorite): not exposed in the interface (only the folder and text search are).
- **Send "on behalf of" the internal address**: the internal address is not offered as a sender; the fallback to the internal relay is done via "Default account".
- **Export / printing**: no PDF or CSV export, no printing in this module.
- **Two-way synchronization**: local actions (read, star, trash) are not reflected on the origin server.

### 5.5 Frequently asked questions

**Q: Why don't my new emails arrive right away?**
A: Use the **Synchronization** tab ("New only"). Automatic synchronization exists for accounts with "Auto-sync" enabled (default interval of 15 minutes); the internal address, for its part, receives continuously via webhook (interface refresh every 60 seconds).

**Q: My Gmail / Outlook / iCloud password is refused.**
A: With two-step verification, you need an **app password** (Gmail: `myaccount.google.com/apppasswords`; Outlook and iCloud: account security section). Or use **OAuth**.

**Q: After a successful Microsoft 365 OAuth connection, synchronization fails.**
A: The Azure tenant must authorize IMAP and SMTP for OAuth (Microsoft disabled basic authentication by default). Your organization's administrator must enable them.

**Q: The OAuth button is grayed out.**
A: The server credentials are not configured (see §4.5). Contact your administrator.

**Q: If I delete an email in the ERP Trash, is it also deleted on Gmail?**
A: No. Actions are local to the ERP; there is no two-way synchronization.

**Q: How do I send from my own address and not from `info@constructoai.ca`?**
A: Set up an external account (app password or OAuth) and choose it as the sender. Without an external account, sending goes through the platform relay (sender `info@constructoai.ca` on behalf of your company, reply directed to your internal address).

**Q: Can I attach files?**
A: Yes, up to **5 files**, **10 MB each** and **20 MB total**.

**Q: Can the AI send an email on my behalf?**
A: Yes, with **"Auto-reply"** (it generates and sends after confirmation). The other functions (Analyze, Suggest, Draft with AI) only prepare text that you validate. The **AI Assistant** tab never sends anything.

**Q: Can the AI Assistant see my account passwords?**
A: No. It is limited to reading emails, threads, and templates; it has no access to accounts, tokens, or HR or security data.

**Q: How much do the AI functions cost?**
A: They consume **prepaid AI credits** (actual model cost plus a 30% markup). Each assistant response shows its cost and the remaining balance. If the balance is insufficient, the call is refused.

**Q: Can I see the emails of another user in my company?**
A: No. Each account belongs to its creator; only shared accounts (like the internal address) are visible to everyone.

**Q: Can a deleted account be recovered?**
A: Yes. Deletion is a deactivation. An administrator can reactivate it with **"Restore"**, and the emails remain readable.

**Q: Are attachments stored for a long time?**
A: They are kept in the database. When an email is permanently deleted, its attachments are purged.

**Q: Is the search comprehensive?**
A: It covers the subject, sender, recipient, and text body. There are no advanced filters in the interface.

---

## 6. Summary

- **Multi-account IMAP/SMTP client** integrated into the ERP, Outlook-style, with an **internal address** per company (receiving via n8n / Mailgun webhook).
- **8 tabs** (Inbox, New message, Sent, Drafts, Templates, Configuration, Synchronization, AI Assistant) + the **Trash** as a 4th folder.
- **30 endpoints**: 29 in `emails.py`, 1 in `emails_ai.py`.
- **Two connection methods**: app password (Fernet-encrypted) or **OAuth** (Google and Microsoft 365); the **OAuth tokens are also encrypted at rest**.
- **Three synchronization modes** (New / Last 50 / All ≤ 200) + **automatic background synchronization** ("Auto-sync" accounts, default 15 min).
- **Real sending** with attachments (5 files, 10 MB/file, 20 MB total), templates, and signature; fallback via the platform relay when there is no external account.
- **Two AI surfaces not to be confused**: drafting / replying (including **Auto-reply**, which sends) in `emails.py`, and a **read-only consultation AI Assistant** in `emails_ai.py` (limited to 3 tables, with no access to accounts).
- **AI billing** via prepaid credits (model cost × 1.30); the real gate is the balance, not the role.
- **6 pre-installed** read-only templates, with `{{...}}` variables substituted at send time.
- **Security**: encryption at rest, HTML sanitization (real HTML parser, anti-tab-hijacking), webhook and OAuth callback cryptographically protected, SSRF protection, isolation by company and by owner.
- **Does not do**: manual draft, template management, "Reply All" / "Forward" / "Archive", manual read/unread marking, thread view, advanced filters, export / printing, two-way synchronization.

---

**Documentation generated from verified source code**: `ERP_REACT/backend/routers/emails.py` (6,993 lines, 29 endpoints), `ERP_REACT/backend/routers/emails_ai.py` (332 lines, 1 endpoint), `ERP_REACT/frontend/src/pages/EmailsPage.tsx`, `ERP_REACT/frontend/src/components/emails/EmailAccountsPanel.tsx`, `EmailSyncPanel.tsx`, `EmailAIPanel.tsx`, `EmailAIComposeButton.tsx`, `EmailsAssistantTab.tsx`, `ERP_REACT/frontend/src/api/emails.ts`, `ERP_REACT/frontend/src/store/useEmailsStore.ts`, translation files `i18n/locales/fr/emails.json` and `emailsAssistant.json`.

**Related manuals**:
- CRM module (automatic matching of contacts and companies)
- Quotes / Bids module (their own sending)
- Invoices module (their own sending)
- AI / Assistant module (shared AI credits)
- Messaging module (internal messaging between users — distinct from this module)
- Administration module (server variables, OAuth, encryption key)
