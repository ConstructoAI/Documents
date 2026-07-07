# Module 24 — Internal Messaging

> **Version**: 3.0 (line-by-line overhaul verified against the source code, 2026-07-07)
> **Menu**: "COMMUNICATION" section of the sidebar → **Messaging** (`MessageSquare` icon) — neighbors: **Emails**, **Voice agent**
> **Route**: `/messagerie`
> **Reference code (backend)**: `backend/routers/messaging.py` (1,137 lines, 13 endpoints — 6 channels/messages, 3 inactive direct messages, 4 notifications; prefix-less router mounted under `/api/erp/v1`); `backend/routers/messagerie_ai.py` (322 lines, 1 endpoint `POST /messagerie/ai/chat`, prefix `/messagerie/ai`)
> **Reference code (frontend)**: `frontend/src/pages/MessagingPage.tsx` (502 lines); `frontend/src/components/messaging/MessageAttachments.tsx` (263 lines); `frontend/src/components/messagerie/MessagerieAssistantTab.tsx` (122 lines); `frontend/src/api/messaging.ts` (133 lines) + `frontend/src/api/messagerieAi.ts`
> **Labels**: `i18n/locales/fr/messaging.json` (43 lines) + `i18n/locales/fr/messagerieAssistant.json` (15 lines)
> **PostgreSQL tables (per tenant)**: `conference_channels`, `conference_messages`, `conference_reactions`, `conference_members`, `conference_attachments`, `notifications`
> **Scope**: internal team messaging in the style of Teams / Slack, specific to each company (tenant). **Channels** (`#`) containing a chronological **message feed**, with **emoji reactions** and **attachments**. Added to this are a **read-only AI Assistant** (summarizing and searching messages) and a separate **notifications** system (the top-bar bell, outside this page). On the web side, it is above all a **lightweight client for viewing and participating**: creating private channels, sending attachments, and editing and deleting messages are handled by the **mobile app**, which shares the same tables.

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

Internal Messaging gives every user of the same company a space for quick exchanges, modeled on Microsoft Teams or Slack:

- **Topic channels** (`#general`, `#south-shore-jobsite`, `#bids`, etc.) containing text messages.
- **Emoji reactions** on each message (six quick emojis), toggled (added or removed with one click).
- **Attachments** (images and files) displayed in the feed, with a full-screen image viewer and file downloads.
- **Read-only AI Assistant**, opened in a modal window, that summarizes a channel, finds a message, or counts activity, drawing on the company's real messages.
- **Local search** within the current channel.

It is an **internal company** communication tool: it is not intended for clients. Exchanges with clients go through the CRM (interactions), Emails, and B2B portal modules.

### 1.2 Access and layout

- Sidebar → **COMMUNICATION** section → **Messaging** (`MessageSquare` icon), next to **Emails** and **Voice agent**.
- URL: `/messagerie`. The page is protected: you must be authenticated.
- **2-pane layout** (no tabs):
  1. On the left, the **"Channels"** bar (fixed width on a wide screen).
  2. On the right, the **message area** (channel header, feed, input area).
- Two modal windows overlay as needed: the **AI Assistant** and **New channel**.
- Adaptive height: `calc(100vh - 120px)` on mobile, `calc(100vh - 180px)` on desktop.
- **Mobile-responsive**: below the `md` breakpoint (768 px), the screen shows either the channel list or the feed of the selected channel, with a back button.

> **Important — no tabs.** The page has no tab system. It has two panes and two modals. Channels are not fixed tabs: they are **created by users** and are therefore fully dynamic, specific to each company.

### 1.3 Permissions

- Any authenticated user of the company can: list visible channels, create a channel, read messages, post a message, react with an emoji, download an attachment, and query the AI Assistant.
- **No moderator or channel-administrator role** is required: all users have the same access rights to public channels. There is no role guard (`is_admin`, `comptable`, `super_admin`) on the messaging endpoints.
- **Private channels**: access is restricted to their members. Membership of a private channel is managed by the mobile app (see §5.1). A web user with no linked employee record sees only the **public** channels — this is a safe degradation, never a leak.
- The **AI Assistant** is open to any authenticated user, but its use consumes prepaid AI credits (see §4.9).

### 1.4 Sub-modules (page surfaces)

| Surface | Where | Role |
|---|---|---|
| "Channels" bar | Left pane | Channel list, creation, opening the AI Assistant |
| Message area | Right pane | Header, chronological feed, reactions, attachments, input |
| "New channel" modal | Overlaid | Create a channel (name + description) |
| "AI Assistant — Messaging" modal | Overlaid | Read-only AI dialogue (summary, search) |
| Notifications (bell) | **Outside this page**, in the top bar | Alerts generated by other modules |

---

## 2. Interface

### 2.1 "Channels" sidebar (left pane)

**Header.** The title **"Channels"**, followed by two action buttons:

- **AI Assistant** (`Sparkles` icon): opens the AI Assistant modal. The tooltip shows **"AI Assistant"**.
- **New channel** (`Plus` icon): opens the creation modal. The tooltip shows **"New channel"**.

**Channel list.** Each row shows:

- The `#` (hash) icon followed by the **channel name** (truncated if long).
- On the right, two subtle counters:
  - The **member count** (`Users` icon + number), shown **only if greater than 0**. Tooltip: "N member(s)".
  - The **message count** (number only), shown only if greater than 0.
- The active channel is highlighted in the accent color.

> **Why "0 members" on a channel I just created?** A **public** channel created from the web registers no member row: its member counter therefore stays at 0, even though the whole company can write in it. The interface simply hides the badge when the counter is 0. Channels whose member count is visible are generally **private** channels managed by the mobile app.

**Empty state.** If no channel exists yet: **"No channels"**.

**Loading.** On first display, a loading spinner fills the screen, then the **first channel** in the list (alphabetical order) is **selected automatically**.

### 2.2 Active channel header (right pane)

- On mobile, a back button (`ChevronLeft`, tooltip **"Back to channels"**) returns to the list.
- The `#` icon + the **channel name**.
- A gray **"N members"** badge if the channel has at least one member.
- A **search field** (`Search` icon, placeholder **"Search..."**) with an `X` clear button (tooltip **"Clear search"**).
- The channel **description**, shown under the header if it was filled in.

### 2.3 Message feed

The feed is **chronological and flat**: the oldest messages at the top, the most recent at the bottom. There is no threaded (conversation-thread) view and no reply collapsing.

**Loaded window.** The web loads the channel's **100 most recent messages** (constant `MSG_FETCH_LIMIT = 100`). A new incoming message pushes the oldest one out of the window.

**Automatic refresh.** As long as a channel is open, the feed refreshes **every 30 seconds** in the background (silent periodic polling). If a colleague writes while you are reading, their message appears in under 30 seconds without any action on your part.

**Automatic scrolling.** When a channel is opened, the feed instantly scrolls to the last message. When a new message arrives at the bottom, scrolling is smooth. If you scroll up to read old messages and no new one arrives, the view is not forced back down.

**Message content.**

- A round **avatar** (the first letter of the author's name).
- The **author's name**. Failing that, the label **"User"**.
- The **relative date** ("5 min ago", "yesterday"...).
- The **"(edited)"** note if the message was edited (edit done from mobile).
- The message **text**, which preserves line breaks.
- Any **attachments** (see §2.5).
- The **reactions row** (see §2.4).

**Special states.**

- Channel open but empty: `MessageSquare` icon + **"No messages in #channel"** + **"Be the first to write!"**.
- No channel selected: `Hash` icon + **"Select a channel"**.

### 2.4 Emoji reactions

Below each message are **chips** showing `emoji + count`. A click toggles your reaction:

- If you had not yet reacted with this emoji, it is **added** (tooltip **"Add a reaction"**).
- If you had already reacted, it is **removed** (tooltip **"Remove my reaction"**). Your own reactions are highlighted in the accent color.

When you hover over a message, a **quick picker** shows the emojis **not yet used** on that message, translucent, so you can react in one click.

**Available emojis (six, fixed)**: 👍 ❤️ 😄 🎉 🤔 👀. The same palette serves both the reactions and the composer's emoji picker. There is no free-form emoji picker.

A safeguard prevents accidental double-clicks on the same reaction (synchronous UI-side lock), and the database enforces uniqueness `(message, user, emoji)`: you cannot place the same emoji twice on a message.

### 2.5 Attachments

Attachments are **written by the mobile app**; the web **displays and downloads** them (read-only). The component fetches each file through an authenticated call, with no public URL.

- **Images**: 96 × 96 pixel thumbnails. A click opens a **full-screen viewer** with:
  - **Previous / Next** navigation between the message's images (on-screen arrows or the ← and → keys).
  - Closing via the `X` button or the Esc key.
  - An "i / N" counter when there are several images.
  - On load failure: **"Image unavailable"**.
- **Non-image files** (PDF, Word, Excel...): a downloadable card showing an icon by type (`FileText` for PDF and Word, `FileSpreadsheet` for Excel, generic icon otherwise), the file name and its size ("B", "KB", "MB"), with a `Download` icon. A click downloads the file.

> **Download limit: 25 MB** per attachment. Images of a safe type (JPEG, PNG, GIF, WebP, AVIF) are served for inline display; everything else is forced to download, with the appropriate security headers (MIME-sniffing protection, restrictive content security policy).

### 2.6 Message composer

At the bottom of the right pane:

- An **emoji button** (`Smile` icon, tooltip **"Add an emoji"**) opens a small picker of the six emojis. The chosen emoji is inserted **at the cursor position** in the text. The picker closes on a click outside or the Esc key.
- A **text field** (placeholder **"Message #channel..."**). Sending happens with the **Enter** key (without Shift).
- A **send button** (`Send` icon, tooltip **"Send"**), disabled if the field is empty or if a send is already in progress. A double-send safeguard prevents a duplicate send on a repeated press.

> **Only one type of send from the web: text (and emojis).** You cannot attach a file from the web: attachments are added by the mobile app. See §5.1.

### 2.7 "New channel" modal

Opened by the `Plus` button. Two fields:

- **"Channel name *"** (required, placeholder "general"). Maximum length: 100 characters.
- **"Description"** (optional, placeholder "Channel description..."). Maximum length: 2,000 characters.

Buttons: **"Cancel"** and **"Create"**. A double-creation safeguard prevents creating two identical channels on a double-click.

> **No "private" or "type" option in the web form.** The channel created from the web is always **public**: the form sends only the name and the description. Private channels are created in the mobile app.

### 2.8 "AI Assistant — Messaging" modal

Opened by the `Sparkles` button in the "Channels" bar. It is a **modal window** (not a tab, despite its internal label).

- **Title**: **"AI Assistant — Messaging"**.
- **Subtitle**: **"View, summarize, and find your messages and channels (read-only)."**
- **Welcome screen**: a prompt text specifying that the assistant reads your real messages, sends no message, and accesses neither user accounts nor payroll or human resources data. Three **clickable examples** are offered:
  1. "Summarize the latest messages in the Project channel."
  2. "Are there any messages mentioning a delivery?"
  3. "How many channels and messages do we have?"
- **Input**: a text area (placeholder "Ask your question about messaging…"). Sending happens with **Enter**; **Shift + Enter** inserts a line break.
- During processing: the **"Analyzing…"** indicator.
- On a problem: **"An error occurred. Try again."**
- **Answers**: presented in bubbles, with metadata (profile "Messaging", token count, cost, duration).

The assistant is **read-only**: it can read the content of messages and channels, but it writes nothing and sends no message. Its read scope is strictly limited to channels, messages, members, and reactions; it has access to neither accounts, nor payroll, nor sensitive data (see §4.8).

### 2.9 Notifications (the top-bar bell — outside this page)

Messaging **does not display the bell itself**: the bell lives in the top bar (`TopBar`), on every page. The Messaging module nevertheless provides the endpoints that feed it:

- A **bell** (`Bell`) with a red badge showing the number of unread notifications (capped in the display at "99+").
- A dropdown menu titled **"Notifications"**, with a **"Mark all as read"** button and the list of the latest notifications (title, message, relative date, dot for unread ones).
- The counter is re-polled **every 60 seconds**; the list (20 items maximum) is loaded when the menu opens. A click marks the notification as read and navigates to the associated link.

> **Warning — notifications do NOT concern channel messages.** The bell is a generic system fed by **other modules** (quotes, invoices, emails). **Posting a message in a channel generates no notification**, and there is no `@user` mention system. See §5.2.

---

## 3. Step-by-step workflows

### 3.1 Open a channel and read messages

1. Open **Messaging** (side menu, COMMUNICATION section).
2. On load, the first channel is selected automatically; its feed appears.
3. To switch channels, click another channel in the left bar. The search field resets and the feed scrolls to the last message.
4. The feed refreshes on its own every 30 seconds.

### 3.2 Send a message

1. Select the desired channel.
2. Click in the **"Message #channel..."** field and type the text.
3. (Optional) Click the `Smile` icon to insert an emoji at the cursor position.
4. Press **Enter** (or click the `Send` button).
5. The message goes to `POST /channels/{id}/messages`; the feed refreshes silently and shows your message.

> Limit: a message is at most 10,000 characters (beyond that, sending is rejected with a validation error).

### 3.3 React to a message

1. Hover over the message: the not-yet-used emojis appear translucent.
2. Click an emoji to **add** your reaction, or click an already-highlighted chip to **remove** yours.
3. The counter updates at the next refresh.

### 3.4 Create a channel

1. Click the `Plus` button in the "Channels" bar.
2. Enter a **Name** (for example `south-shore-jobsite`).
3. (Optional) Enter a **Description** (for example "Daily tracking of the South Shore project").
4. Click **"Create"**.
5. The channel, **public**, appears in the list. All company users see it and can write in it; no invitation is required.

> Tip: for a one-on-one exchange, simply create a dedicated channel (for example `coord-marie-john`). Private direct messages are not offered from the web (see §5.4).

### 3.5 Search within a channel

1. Select a channel.
2. Type a keyword in the **"Search..."** field of the header.
3. The feed filters **instantly**. An "N result(s) for "..."" counter appears, under the label **"Search within loaded messages"**.
4. Click the `X` button to return to the full feed.

> **Search is local**: it covers only the already-loaded messages (the window of the 100 most recent). It does not go back through the channel's entire history.

### 3.6 View an attachment

1. In the feed, spot the image thumbnails or the file cards under a message.
2. Click a thumbnail to open the **full-screen viewer**; navigate with the ← / → arrows; close with Esc.
3. Click a file card to **download** the document (PDF, Word, Excel...).

### 3.7 Query the AI Assistant

1. Click the `Sparkles` button in the "Channels" bar.
2. In the modal, type a question (or click an example).
3. Press **Enter**. The "Analyzing…" indicator appears.
4. The assistant answers based on your company's real messages. Each answer consumes AI credits (see §4.9).

> Useful examples: "Summarize what was said this week in #south-shore-jobsite", "Find the messages about concrete", "How many messages have we exchanged in total?"

### 3.8 Navigate on mobile

1. On open, the channel list fills the whole screen.
2. Clicking a channel switches to its feed, full screen.
3. The back button (`ChevronLeft`, "Back to channels") returns to the list.

### 3.9 Handle a notification (top-bar bell)

1. Click the bell in the top bar (present on all pages).
2. The "Notifications" menu lists the latest alerts.
3. Clicking a notification marks it as read and opens the linked item (a quote, an invoice, an email...).
4. The "Mark all as read" button clears the badge at once.

---

## 4. Reference

### 4.1 Endpoints — channels and messages (`messaging.py`)

All require a valid JWT token and a tenant context (`user.schema`); otherwise **400 "Missing tenant context"**. None imposes a particular role. Global prefix: `/api/erp/v1`.

| Method | URL | Role |
|---|---|---|
| GET | `/channels` | Lists active channels; hides private channels the caller is not a member of; returns the actual member and message counts |
| POST | `/channels` | Creates a channel (atomic transaction; persists `channel_type` and `is_private`) |
| GET | `/channels/{id}/messages` | Lists messages (pagination `page` ≥ 1, `per_page` default 50, **max 200**); enriches each message with its reactions and attachments |
| GET | `/channels/attachments/{id}` | Downloads an attachment's binary (read-only) |
| POST | `/channels/{id}/messages` | Posts a message (validates that `parent_message_id`, if provided, belongs to the same channel) |
| POST | `/channels/{id}/messages/{mid}/reactions` | Toggles an emoji reaction (add or remove) |

> The web loads 100 messages per call; the server maximum is 200 per page. There is no "Load more old messages" button in the web interface.

### 4.2 Endpoints — direct messages (inactive)

| Method | URL | Actual behavior |
|---|---|---|
| GET | `/direct-messages` | Always returns `{ items: [], unread_count: 0 }` (table absent) |
| POST | `/direct-messages` | **503** "Direct messaging service temporarily unavailable." |
| PUT | `/direct-messages/{id}/read` | **503** (same message) |

> Private direct messages are **not functional**. Returning a 503 code (rather than a fake 200) avoids giving the impression of delivery. There is no direct-message interface in the web.

### 4.3 Endpoints — notifications (top-bar bell)

| Method | URL | Role |
|---|---|---|
| GET | `/notifications` | Lists the user's notifications (`unread_only` optional; `limit` default 20, **max 50**); returns an empty list if the table does not exist |
| GET | `/notifications/count` | Unread count (for the bell) |
| PUT | `/notifications/{id}/read` | Marks a notification as read |
| PUT | `/notifications/read-all` | Marks all notifications as read |

### 4.4 Endpoint — AI Assistant (`messagerie_ai.py`)

| Method | URL | Role |
|---|---|---|
| POST | `/messagerie/ai/chat` | Read-only AI Assistant: answers from the real messages; writes nothing, sends no message |

Body parameters: `message` (1 to 8,000 characters), `history` (40 items maximum, truncated to 12 conversation turns server-side), `language` (optional, FR or EN). Model used: `claude-sonnet-4-6`. Generation cap: 8,000 tokens. Up to 5 tool round-trips per answer.

### 4.5 Input limits (validation → 422)

| Field | Rule |
|---|---|
| Channel name | 1 to 100 characters (required) |
| Channel description | 2,000 characters maximum |
| Channel type | 50 characters maximum ("general" by default) |
| Message text | 1 to 10,000 characters |
| Reaction emoji | 10 characters maximum (`VARCHAR(10)` column) |
| Question to the AI Assistant | 1 to 8,000 characters |

### 4.6 HTTP status codes

| Code | Meaning in this module |
|---|---|
| 200 | Success |
| 400 | Missing tenant context, invalid tenant schema, or empty channel name |
| 402 | AI credits exhausted (AI Assistant) |
| 403 | Private channel, caller not a member (read, send, react); or AI access denied |
| 404 | Message or channel not found; attachment of a private channel not accessible (hidden so as not to reveal its existence) |
| 413 | Attachment exceeding 25 MB on download |
| 422 | Out-of-bounds data (see §4.5) |
| 503 | Direct messages (always); or AI service momentarily unavailable |

### 4.7 Emojis, refresh, and constants

| Item | Value | Source |
|---|---|---|
| Emojis (reactions and input) | 👍 ❤️ 😄 🎉 🤔 👀 (six, fixed) | `MessagingPage.tsx:24` |
| Messages loaded by the web | 100 (most recent) | `MSG_FETCH_LIMIT`, `MessagingPage.tsx:28` |
| Feed refresh | 30 seconds | `MessagingPage.tsx:108` |
| Bell counter refresh | 60 seconds | `TopBar.tsx` |
| Server-side message pagination | default 50, max 200 | `messaging.py:372` |
| Attachment download limit | 25 MB | `messaging.py:60` |
| AI Assistant rate limit | 20 requests per IP per window | `erp_api.py` |

### 4.8 PostgreSQL tables (per tenant)

| Table | Role | Who writes |
|---|---|---|
| `conference_channels` | Channels: name, description, type, `is_private`, `is_active`, creator, date | Web and mobile |
| `conference_messages` | Messages: channel, author, text, parent message, edited/deleted flags, dates | Web (sending) and mobile |
| `conference_reactions` | Reactions: message, user, emoji; uniqueness `(message, user, emoji)` | Web and mobile (toggle) |
| `conference_members` | Members of private channels; membership is indexed by employee ID | Mobile |
| `conference_attachments` | Attachments (binary in database) | Mobile (writing); web (read-only) |
| `notifications` | Generic notifications (bell) fed by other modules | Other modules (quotes, invoices, emails) |

> **Provisioning.** The `conference_*` tables are not created by the ERP initialization: they come from the mobile app and historical migrations. The ERP merely auto-repairs `conference_attachments` (created on the fly if absent). The `notifications` table is created by no ERP component: the endpoints check for its existence and return an empty list if it is missing. Web messaging therefore depends on a schema provisioned elsewhere.

**AI Assistant read allowlist**: only `conference_channels`, `conference_messages`, `conference_members`, `conference_reactions`. Any query naming `users`, `employees`, payroll, salaries, the SIN (Social Insurance Number), emails, tokens, Stripe, etc. is **denied** by a safeguard. The read engine runs in a read-only transaction, with a maximum timeout, an automatic result limit, and sensitive-column masking. The assistant **can read message content** (a business decision), but nothing else.

### 4.9 AI Assistant cost calculation

- AI access is open to any authenticated user; the real lock is the **prepaid AI credits balance**: if the balance is empty, the request is rejected with **402 "AI credits exhausted"**.
- Cost charged per request: `(input tokens × 0.003 + output tokens × 0.015) / 1000 × 1.30`, i.e., the reference rate marked up by **30%**. The cost is debited from the balance after each successful answer.
- The debit has **no idempotency key**: on a network retry or a double submission, the request may be **charged twice**. The double-send protection exists only on the UI side. A per-IP rate limit (20 requests per window) reduces this risk.
- If the subscription provides for it, a low balance triggers an automatic top-up via Stripe.

### 4.10 Keyboard shortcuts

| Context | Key | Effect |
|---|---|---|
| Message field | Enter | Send the message |
| Emoji picker | Esc | Close the picker |
| AI Assistant area | Enter | Send the question |
| AI Assistant area | Shift + Enter | New line |
| Image viewer | ← / → | Previous / next image |
| Image viewer | Esc | Close the viewer |

---

## 5. Integrations and FAQ

### 5.1 Integration with the mobile app

The mobile app is the **primary client** of the messaging system and shares the same tables. Several features exist only there:

- **Private channels**: their creation and member management happen on mobile. Membership is indexed by employee ID. A web user without a linked employee record sees only public channels.
- **Attachments**: adding them happens on mobile; the web displays and downloads them.
- **Editing and deleting messages**: available on mobile; the web shows the "(edited)" note but offers no edit or delete action.
- **Conversation threads**: the parent-message link is stored and validated server-side, but the web shows a flat feed and does not use threaded replies.

### 5.2 Integration with notifications (top-bar bell)

The `notifications` table is **shared** and fed by other modules: a new quote, an invoice, or an incoming email each drop an alert there, with a clickable link. The Messaging module only **reads, counts, and marks as read** these notifications. It **creates none**: posting a channel message does not ring the bell, and there is no `@user` mention.

### 5.3 Integration with the Employees module

The name shown as a message's author comes from a join with the company's employee records and user accounts (joins strictly limited to the tenant's schema, to prevent any leak of names between companies). If no name is found, the label "User" is shown.

### 5.4 What is not possible from the web

- **Private direct messages**: not functional (return an empty list or a 503 code). Workaround: create a channel dedicated to two people.
- **Sending attachments**: mobile only.
- **Editing or deleting messages**: mobile only.
- **Creating private channels**: mobile only.
- **`@user` mentions**: nonexistent.
- **Export (PDF, CSV) or printing** of a channel: none.
- **Presence status** ("online", "typing"), **read receipts**, **audio/video calls**, **browser or email notifications** on a new message: none.
- **Search across the entire history**: web search covers only the 100 loaded messages.

### 5.5 FAQ

**Q: Are there tabs in Messaging?**
A: No. The page has two panes (channels on the left, messages on the right) and two modal windows (New channel, AI Assistant). Channels are created by users; they are not fixed tabs.

**Q: Who sees the channel I create from the web?**
A: Everyone in the company. Channels created from the web are always public. Private channels are created in the mobile app.

**Q: Why do some channels show "0 members"?**
A: A public channel created from the web registers no member: its counter stays at 0 (the badge is then hidden), even though the whole company can write in it. The member count is mostly relevant for private channels managed on mobile.

**Q: Can I edit or delete a message from the web?**
A: No. Editing and deleting are done from the mobile app. The web shows the "(edited)" note when a message has been edited.

**Q: Can I attach a file to a message from the web?**
A: No. You can only view and download attachments (added on mobile). The web input area accepts only text and emojis.

**Q: Does the search go back through the entire history?**
A: No. It filters only the already-loaded messages (the 100 most recent in the current channel).

**Q: Does posting a message notify my colleagues?**
A: No, not via the notification bell or by email. Your colleagues will see the message by opening the channel; the feed refreshes on its own every 30 seconds. There is no `@user` mention.

**Q: Can I write `@Marie` to alert her?**
A: You can write it, but it is only text: no notification is triggered. The mention system does not exist.

**Q: Do direct (private, one-to-one) messages work?**
A: Not from the web. For a restricted exchange, create a dedicated channel.

**Q: How many reaction emojis are available?**
A: Six, fixed: 👍 ❤️ 😄 🎉 🤔 👀. You cannot place the same emoji twice on a message.

**Q: Can the AI Assistant read my messages? And see salaries or passwords?**
A: It can read the content of your company's messages and channels, to summarize or find them. It **cannot** access user accounts, passwords, payroll, or human resources: its reading is locked to the messaging tables only.

**Q: Can the AI Assistant write or send a message on my behalf?**
A: No. It is strictly read-only: it writes nothing and sends no message.

**Q: Does the AI Assistant cost anything?**
A: Yes. Each answer consumes prepaid AI credits (reference rate marked up by 30%). If the balance is exhausted, the assistant replies "AI credits exhausted". Avoid quickly resending the same question: a retry may be charged a second time.

**Q: Are messages kept for a long time?**
A: Yes, indefinitely; there is no automatic purge.

**Q: Does messaging work on a phone?**
A: The web page is mobile-friendly (channel list, switch to the feed, back button). There is also a dedicated **mobile app**, which is the primary client and offers the advanced features (private channels, attachments, editing, conversation threads).

**Q: Can I export or print a channel's history?**
A: No, no export or printing function is provided.

---

## 6. Summary

- **Internal team messaging** in the style of Teams / Slack, specific to each company: channels, a message feed, emoji reactions, attachments, a read-only AI Assistant.
- **Access**: side menu, COMMUNICATION section → Messaging; route `/messagerie`.
- **2-pane layout** (channels + messages), **no tabs**, plus 2 modals (New channel, AI Assistant). Channels are dynamic, created by users.
- **Six fixed emojis**: 👍 ❤️ 😄 🎉 🤔 👀; toggled reactions; uniqueness `(message, user, emoji)`.
- **The web loads 100 messages** per channel; feed refresh every 30 seconds; **local** search over the loaded window.
- **Read-only attachments** on the web side (written by mobile): full-screen image viewer, file download, 25 MB limit.
- **AI Assistant** in a modal (`Sparkles` button): read-only, reads message content but nothing sensitive; `claude-sonnet-4-6` model; cost = reference rate + 30%; 402 rejection if credits exhausted; no idempotency key (risk of double charge on a retry).
- **Notifications (bell)**: a **separate** system, in the top bar, fed by other modules (quotes, invoices, emails). Posting a message creates no notification; no `@user` mention.
- **Mobile only**: private channels, sending attachments, editing and deleting messages, conversation threads.
- **Inactive**: private direct messages (empty list or 503).
- **Permissions**: every authenticated user has the same rights; private channels are visible only to their members (membership managed on mobile).

---

**Documentation generated from verified code**:
- `backend/routers/messaging.py` (1,137 lines, 13 endpoints)
- `backend/routers/messagerie_ai.py` (322 lines, 1 endpoint)
- `frontend/src/pages/MessagingPage.tsx` (502 lines)
- `frontend/src/components/messaging/MessageAttachments.tsx` (263 lines)
- `frontend/src/components/messagerie/MessagerieAssistantTab.tsx` (122 lines)
- `frontend/src/api/messaging.ts` (133 lines) + `frontend/src/api/messagerieAi.ts`
- `frontend/src/i18n/locales/fr/messaging.json` (43 lines) + `messagerieAssistant.json` (15 lines)

**Related manuals**:
- Module 9 (Employees — author name resolution) — `09-employes.md`
- Module 23 (Emails — distinct from internal messaging) — `25-emails.md`
- Module 25 (AI Assistant — shared credit engine) — `12-ia.md`
- Module 28 (Administration — shared `notifications` table, top-bar bell) — `14-administration.md`
