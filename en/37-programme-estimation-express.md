# Module 37 — Estimation Express (paid public sub-app)

> **Version**: 1.0 (initial draft verified against the source code, July 2026)
> **Intended audience**: this manual is addressed to the **users** of Estimation Express — the **contractors** and **construction workers** who want to quickly obtain a cost estimate, produce a professional quote, and have a client approve it, without having an ERP account. It is not addressed to developers.
> **What Estimation Express is**: a **paid, pay-as-you-go public web application** that puts Constructo AI's AI estimating within everyone's reach, **without a subscription and without an ERP account**. You describe your project (and attach your plans) in an **AI chat**, the AI replies with a priced estimate, you can turn the conversation into a **professional quote** (a themed HTML document that can be signed online), and you can even add a **photorealistic 3D render**. Everything is drawn from **prepaid credits** that you buy by card (Stripe), at the **actual cost** of the AI plus a markup.
> **Separate application**: Estimation Express is **not** an internal tab of the Constructo AI ERP. It is a **standalone sub-application** (folder `ESTIMATION_EXPRESS_REACT/frontend`, basename `/estimation` — `main.tsx:15`), served under `app.constructoai.ca/estimation`. No authentication, no password, no ERP token: your **only identity** is a **session token** kept in your browser (= your credit wallet).
> **API prefix**: `/api/estimation/v1` (`api/client.ts:20`). The module lives in **a single backend file**: `routers/estimation_express.py` (≈ 4,089 lines), which exposes **about thirty** public endpoints and reuses `devis.py` heavily (document rendering, quote-generation tool, multi-agent plan analysis), `gemini_image.py` (3D rendering), and `ai_profiles.py` (experts and text extraction).
> **Reference code (frontend, `ESTIMATION_EXPRESS_REACT/frontend/src/`)**: `main.tsx` (15, basename `/estimation`), `App.tsx` (3 routes), pages `ChatPage.tsx` (2,378 lines), `RenderPage.tsx` (840), `SoumissionPublicPage.tsx` (382); components `components/soumission/SoumissionRenderModal.tsx` (592), `components/SignatureCanvas.tsx`, `components/render/Rendu3DDropzone.tsx`, `components/render/Rendu3DControls.tsx`, `components/render/Rendu3DViewer.tsx`, `components/PlanCropper.tsx`; API `api/client.ts`, `api/estimation.ts`; translations `i18n.ts`, `locales/{fr,en}/translation.json`.
> **PostgreSQL tables (shared database, `public` schema, `express_*` prefix)**: `express_sessions` (the credit wallet), `express_topups` (Stripe top-ups), `express_messages` (chat history), `express_files` (attached plans), `express_holds` (credit holds), `express_soumissions` (generated documents), `express_login_codes` (one-time codes), `express_profiles` and `express_profile_documents` (custom AI profiles), `express_conversations` (history threads). The module also writes a usage trace to `public.ai_usage_tracking`.
> **Money scope**: unlike SEAOP (free), Estimation Express is **paid per use**. Every AI response, every generated quote, and every 3D render **draws credits** from your wallet — at the **actual cost of the AI tokens**, converted to Canadian dollars and **marked up** (see §4.3). The wallet is topped up by **card, via Stripe** ($25 / $50 / $100 of credits, taxes on top). You need at least **$2** of balance to act.

*A note on the terminology used in this manual:* "wallet" (or "balance", `express_sessions.balance_cad` in the code) refers to the amount of prepaid credits attached to your session; "session token" (or "token") refers to the unique identifier of your wallet, kept in your browser; "issuer" refers to the company that produces the quote (you); "recipient" (or "client") refers to the person to whom you send the quote for approval; "AI expert" refers to a specialist profile (electrician, plumber, general contractor, etc.) that steers the AI's answers; "top-up" (or "credit top-up") refers to buying credits by card; "endpoint" refers to an API endpoint; "public link" (or `public_token`) refers to the read-only address of a quote, distinct from your session token.

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

Estimation Express answers a simple need: **get a reliable construction estimate, right away, without subscribing to an ERP**. The main journey has four steps:

1. You land on the page, a **session** (empty wallet) is created automatically, and you **top up** a few credits by card.
2. You choose an **AI expert** (general contractor, electrician, plumber, roofing, etc.), then you **describe your project** in the chat — attaching your **plans** if needed. The AI replies with a priced estimate; the **actual cost** of its answer is deducted from your balance.
3. When you are happy with the exchange, you click **"Generate quote"**: the AI turns the conversation into a **professional quote document** (themed layout, logo, terms, exclusions), with a **public link** that you send to your client.
4. The client opens the link, reviews the quote, then **approves** it (with their **drawn signature**) or **declines** it. You are notified of their decision by email.

In addition, a **3D-render page** (`/estimation/rendu`) lets you produce photorealistic images from a 3D model, an image, or a PDF plan, paid from the **same wallet** as the chat.

> **Key takeaway.** Estimation Express is a "public clone" of the ERP's **AI Estimating** (module 24 / the estimating functions of `devis.py`). The difference is not quality — it's the **same engine** (Claude Opus, same expert profiles, same document generation) — but the **access model**: no account, no subscription, **pay-as-you-go** on prepaid credits.

### 1.2 A SEPARATE application (not to be confused with the ERP)

This is the first point to understand. Like SEAOP (module 36) and the B2B portal (module 35), **Estimation Express is not an internal page of the ERP**. It is a **separate application**, with:

- its own address (`app.constructoai.ca/estimation`);
- **no sign-in page** in the classic sense (no username, no password);
- its own **session token** kept in the browser — it does **not** open the ERP, and an ERP account does **not** open Estimation Express;
- its own header bar (no ERP-style navigation sidebar).

The header bar carries a **"Back to Constructo AI"** link (`chat.backToApp`) that **leaves** Estimation Express to return to the ERP's sign-in portal (`app.constructoai.ca/login`). It is a plain page redirect, not a session hand-off.

### 1.3 The business model: prepaid credits, pay-as-you-go

Estimation Express works like a **prepaid card**. You load credits, then each billable action consumes some, according to its **actual cost**.

| | **Estimation Express** |
|---|---|
| Account required | **None** (automatic anonymous session) |
| Subscription | **None** |
| Payment | **Prepaid credits**, purchased by card via **Stripe** |
| Top-ups | **$25 / $50 / $100** of credits (taxes on top, see §4.4) |
| What is billed | Each **AI response**, each **generated quote**, each **3D render** |
| How it is billed | At the **actual cost** of the AI tokens, converted to CAD (× 1.38) and marked up (× 3) |
| Minimum balance to act | **$2** (below this, actions are blocked) |
| What is free | Viewing the chat, **attaching** a render to a quote, the client-side **approval** |

The internal mechanics are cautious: on each action, an amount is **held** on your balance (a "hold"), then, once the answer is produced, the **actual cost** is charged and the **remainder is refunded to you** immediately. If an operation fails (AI unavailable, network error), the hold is **fully refunded** — you are never charged for an answer you did not receive. A safety net automatically recovers holds that got stuck (for example after a server restart).

> **Key takeaway.** You only pay for **what the AI actually consumes**. A short question costs a few cents; analyzing large plans or a long quote costs more. The exact cost of each answer is shown **under** the answer ("Cost: $X.XX") and your balance updates live.

### 1.4 No account, no password: the session IS the wallet

Estimation Express has **neither username nor password**. On your first visit, the application creates a **session** (a unique token) and saves it in your browser (`localStorage`, key `estimation_token`). This token **is** your wallet: everything you top up is attached to it.

Important consequence: if you switch devices, or if you clear your browser data, you **lose access** to that wallet — **unless** you have **linked it to an email**. That is the role of the **"My Account"** button:

- you enter your **email**;
- you receive a **6-digit code** (one-time, valid for 15 minutes);
- you enter it, and your wallet is now **recoverable** from any device (email + a new code).

> **No password.** Multi-device access happens only through a **one-time code** sent by email (`account.secureNote`: "No password — access via one-time code.").
>
> **1 email = 1 wallet.** Balances **never merge**. If you link an email that **already** has a wallet, the application switches to **that** wallet and your current anonymous balance is **no longer accessible** on this device (a warning tells you so before you continue). This is intentional, to prevent any double-crediting.

### 1.5 The three screens

The whole application lives in **three screens** (`App.tsx`):

| Route (address) | Screen | Purpose |
|---|---|---|
| `/estimation` | **Chat** (`ChatPage.tsx`) | Main screen: balance, top-up, AI expert, estimating chat, quote generation |
| `/estimation/rendu` | **3D Render** (`RenderPage.tsx`) | Pay-as-you-go photorealistic renders (wallet **shared** with the chat) |
| `/estimation/s/:ptoken` | **Public approval** (`SoumissionPublicPage.tsx`) | Read-only page where **the client** views and approves / declines a quote |

Any other address automatically redirects to the chat.

### 1.6 A single wallet, shared between the chat and the 3D render

The chat (`/estimation`) and the 3D render (`/estimation/rendu`) share **strictly the same wallet** (same `estimation_token` key in the browser, `RenderPage.tsx:43-44`). You top up once, and you spend interchangeably on estimates, quotes, or renders. A top-up started from the render page **brings you back there** after payment; a top-up started from the chat brings you back to the chat.

### 1.7 How to access Estimation Express and find your way around

**Access.** Open `app.constructoai.ca/estimation`, or go through the "Estimation Express" link on Constructo AI's public home page. No sign-in is required: the session creates itself.

**The header bar** (common to the chat and the render) contains, from left to right: the **logo** and the title **"Estimation Express"** with the subtitle **"AI construction estimating, pay-as-you-go"**; a **"Back to Constructo AI"** link; a **"3D Render"** link (or **"Estimation"** on the render page); the **FR / EN language toggle**; the **"Balance"** panel (colored by state); the **"My Account"** button; the **"Top up"** button.

**Under the chat header**, an **action bar** groups the **AI expert** selector, then the buttons **"AI Profiles"**, **"Appearance"**, **"New conversation"**, **"History"** (visible only once linked to an email) and **"Generate quote"**.

### 1.8 Bilingual French / English

The header **FR / EN** toggle changes, at once, the **interface**, the **language of the AI's answers**, the **language of the quote document**, and the **default texts** (terms, exclusions). Your choice is remembered in the browser (`localStorage`, key `estimation_lang`). The **public** approval page displays in the language you had chosen when you generated that specific document.

---

## 2. Interface

### 2.1 General layout

The chat and the 3D render share the same **header bar** (§1.7). The rest of the page differs by screen. There is **no** navigation sidebar: Estimation Express is deliberately minimalist, focused on one task at a time.

The header's **"Balance"** panel changes color according to the state of your wallet (`balanceColor`):

- **red**: balance at zero or negative;
- **amber**: low balance (below the $2 minimum, or close to running out);
- **green**: comfortable balance.

When the balance is blocking, the **"Top up"** button is wrapped in a **pulsing amber ring** to catch your eye.

### 2.2 Chat screen — header (`ChatPage.tsx`)

In addition to the common elements, the chat header gives access to:

- **"Back to Constructo AI"** (arrow icon) — leaves the application for `app.constructoai.ca/login`;
- **"3D Render"** (image icon) — opens the 3D-render page (§2.13);
- **FR / EN** — language toggle;
- **"Balance"** — your wallet, in Canadian dollars;
- **"My Account"** (person icon) — opens the email-linking modal (§2.7);
- **"Top up"** (wallet icon) — opens the top-up modal (§2.6).

### 2.3 Chat screen — action bar

Under the header, the action bar brings together:

- **"AI expert" selector** — a dropdown that chooses the specialist who will answer. It offers the neutral option **"Generic Assistant"**, a **"My Profiles"** group (your custom experts, if any) and an **"Experts"** group (the experts supplied by the platform, see §4.7). By default, the Quebec general contractor is preselected if available. While the list loads, the label reads **"Loading experts…"**.
- **"AI Profiles"** (robot icon) — opens the management of your custom experts (§2.9).
- **"Appearance"** (palette icon) — opens the visual customization of the quote document (§2.10).
- **"New conversation"** (speech-bubble + icon) — starts from a blank conversation. Disabled if there is no message. A confirmation message reminds you that **the current conversation will be cleared but your balance is preserved**.
- **"History"** (clock icon) — opens the list of your past conversations (§2.8). **Visible only if you have linked your wallet to an email.**
- **"Generate quote"** (signed-document icon) — opens the document-generation modal (§2.11). Disabled if there is no message.

### 2.4 Chat screen — conversation area

**On opening (no message)**, a welcome screen appears: a hard-hat icon, the title **"Welcome to Estimation Express"**, an introductory text, a note reminding you that "Each answer deducts the actual cost of the AI from your balance", and a clickable **"Example to try"** card that pre-fills the input box with a concrete example ("Can you give me an estimate for the addition ONLY, at ECONOMY pricing…").

**The message thread** shows your messages (blue bubbles, plain text) and the AI's answers (formatted in **Markdown**: headings, lists, tables, bold). Under each AI answer, a **"Cost: $X.XX"** note indicates what was charged for that specific answer.

While the AI is thinking, a **"Thinking…"** indicator (with animation) is shown.

### 2.5 Chat screen — input bar

At the bottom of the screen:

- **Blocking balance guard** — if your balance is exhausted or insufficient, an amber panel replaces the input ("Top up your credits to get started" or "Low balance — top up to continue") with a **"Top up"** button.
- **Non-blocking reminder** — if the balance becomes low (but stays sufficient), a small note invites you to top up soon, without preventing you from sending.
- **Attached-file chips** — each added plan appears with its name, its size in megabytes, and a button to remove it.
- **Attach button** (paperclip icon) — adds plans. Accepted formats: **PDF, PNG, JPEG, WEBP**. **Maximum 5 files** per message, **32 MB per file** (limit aligned with Anthropic's API).
- **Text area** — describe your project or ask a question. **Enter** sends the message; **Shift + Enter** inserts a line break.
- **"Send" button** (paper-plane icon / sending animation).
- **Permanent disclaimer**: "Indicative estimates generated by AI — to be validated by a professional."

> **Two ways to query the AI.** If you **attach plans**, the application launches a **multi-agent analysis**: an estimator reads each plan in parallel, then a coordinator synthesizes them. If you write **text only**, it is a **conversational exchange** with the chosen expert, who keeps the last 16 turns of the conversation in memory. Both are billed at the actual cost of the tokens.

### 2.6 "Top up credits" modal

Opened by the **"Top up"** button, it presents the available **top-up amounts** (by default **$25 / $50 / $100** of credits). For each option, the application shows the **credit amount** added to your balance and, next to it, the **total to pay including taxes** (GST 5% and QST 9.975% are added **on top of** the credit amount — for example, **$25 of credits = $28.74 to pay**, see §4.4).

A click on an option **redirects you to Stripe's secure payment page**. After payment, Stripe **brings you back** to Estimation Express, which **automatically confirms** the top-up and **credits** your balance. A modal footer reminds you: **"Secure payment by Stripe"**.

> **The credit appears after the payment is confirmed.** Estimation Express **queries Stripe** on return (and a safety net at server startup catches any payment that was confirmed but not yet credited). If your balance did not move immediately, refresh the page: the check is **idempotent** (never a double credit).

### 2.7 "My Account" modal (email + one-time code)

This is where you **link your wallet to an email** so you can find it anywhere.

- **Email step** — enter your address, then **"Send code"**. A **6-digit** code is emailed to you.
- **Code step** — enter the 6 digits, then **"Verify and access my balance"**. A **"Change email / resend a code"** link lets you start over.
- If your wallet is already linked, a **"Balance linked to {email}"** banner is shown.
- **Switch warning** — if you link an email that **already** has a wallet while your anonymous session holds a balance, the application warns you that this anonymous balance will **no longer be accessible** here (balances do not merge).
- **"Sign out"** — starts a **new blank session** on this device. If your current balance is not linked to any email, a warning notes that it will become **inaccessible**.
- Modal footer: **"No password — access via one-time code."**

### 2.8 "History" modal

**Visible only once linked to an email.** It lists your past conversations: title (or **"Untitled conversation"**), date, number of messages, and the **"current"** tag for the active conversation. For each row:

- **Rename** (pencil icon) — in-place editing; **Enter** confirms, **Esc** cancels;
- **Delete** (trash icon) — with confirmation (irreversible action).

If no conversation is saved: "No conversations saved yet."

> **Anonymous or signed in.** As long as you have **not** linked an email, you have **only one** "default" conversation: clicking **"New conversation"** clears it. Once **linked** to an email, each "New conversation" creates a **separate thread**, kept and recoverable in History across all your devices.

### 2.9 "AI Profiles" (custom) modal

You can create your **own AI experts**, reusable in the chat and in quote generation.

**List** — a **"New profile"** button, then your existing profiles (name + number of reference documents), each with **Edit** and **Delete** (with confirmation). Empty: "No custom profiles yet." **Maximum 20 profiles** per wallet.

**Form** — two fields:

- **"Profile name"** (max 120 characters);
- **"Instructions (the AI's role and expertise)"** (text area, **max 50,000 characters**, with a counter) — describe the role, the expertise, your pricing rules, the expected tone.

**Reference documents** (available **only after saving** the profile) — an **"Add"** button. Formats: **PDF, TXT, CSV, TSV, MD, XLSX, DOCX** (the text is **extracted** and provided to the AI). **Maximum 5 documents**, **20 MB per document**. The content of these documents serves as reference to the AI when this profile is active.

> **A balance guard**: creating a profile or uploading a document requires at least **$2** of balance (an anti-abuse measure, since these operations use resources).

### 2.10 "Document appearance" modal

Customizes the **appearance of your quotes** (colors and default texts). It applies to all your **new** generated documents. Settings are kept locally in the browser.

- **Preset themes** — 6 ready-made palettes: **Constructo Blue, Forest Green, Brick Red, Charcoal, Burgundy, Ocean**.
- **Advanced customization** — 8 adjustable colors (picker + hex code): **Primary color, Primary dark, Accent, Light accent, Header text, Alternating rows, Info-section background, Borders**.
- **Preview** — a themed mini-quote updates live.
- **Default terms** and **Default exclusions** — two text areas (one line per item; bullets and numbering added automatically; max 10,000 characters each). Left empty, they fall back to the platform's default texts, adapted to the language.
- Buttons **Reset**, **Cancel**, **Save**.

### 2.11 "Generate quote" modal

Turns your conversation into a **professional quote document**.

**Form:**

- **"Project name"**;
- **"Your company (issuer)"** section — **Company name (required)**, Email, Phone, Address, City, Postal code, RBQ (Régie du bâtiment du Québec — Quebec building authority; optional), and a **Logo** (PNG / JPG / WEBP, **max 600 KB**);
- **"Your client (recipient)"** section — Client name, Client email (with the note: if you provide the client's email, they **receive the approval link by email**, with a copy to you).

The **"Generate"** button produces the document. This generation **deducts an amount from your balance** based on AI usage (it re-reads the whole conversation and drafts the quote's line items).

**Result screen:**

- **"Quote {number} created."**;
- the **public link** of the quote, with a **"Copy"** button and **"Open quote"** (new tab);
- if you had entered the client's email: the confirmation that **an email with the approval link was sent to them** (copy to you); otherwise, the invitation to **share the link** yourself;
- an **"Add a 3D render"** / **"Manage 3D render"** button (§2.12);
- a **"Done"** button.

Your issuer details and your logo are **remembered locally** for next time.

> **The financial percentages cannot be changed on this screen.** The quote automatically applies a standard markup (administration, contingencies, **fixed 15% profit**) on top of the base cost, then the taxes (see §4.3). These percentages are **fixed server-side** and **do not appear** as adjustable fields in the current interface.

### 2.12 "Quote 3D render" modal

Optional, it adds **one** render image to the bottom of the quote. A **5-step** flow:

1. **Upload** — import a **plan (PDF) or an image**. 3D files **are not accepted** in this context.
2. **Crop** — select the **area to illustrate** with a rectangle (the cropping tool).
3. **Settings** — preview of the area, a **Details** field (max 2,000 characters), **Quality** (Pro / Standard / Fast) and **Resolution** (2K / 4K; 4K unavailable in Fast quality), then **"Generate render"** (deducts an amount from the balance).
4. **Generated** — the image + its cost, with **"Attach to quote"** (free) or **"Redo"**.
5. **Attached** — confirmation, with **Replace** or **Remove** (free) and **Done**.

> **One render per quote**, replaceable or removable at any time. The attached render appears at the bottom of the document, in the preview **and** on the public page viewed by the client.

### 2.13 Standalone "3D Render" screen (`/estimation/rendu`)

Public version of Constructo AI's "3D Render" module (module 27), with the **same wallet** as the chat. Identical header (the back link is called **"Estimation"** here). Body in **three columns**:

- **Left — drop zone.** Drag-and-drop or browse. Three file families:
  - **3D**: GLB, GLTF, OBJ, STL, FBX (**max 50 MB**);
  - **Image**: PNG, JPG, WEBP (resized if necessary);
  - **PDF**: a single page is taken as-is; a **multi-page** PDF opens a **page selector** (preview + Previous / Next + "Use this page").
- **Center — the viewer.** A 3D model appears in an orbitable viewer (rotate it to frame the desired angle); an image or a PDF appears as a preview.
- **Right — the control panel:**
  - **Render type**: **Exterior** (building seen from outside), **Interior** (room or fit-out), **Object** (isolated product). The application suggests "Object" for a 3D file, otherwise "Exterior".
  - **Details and description (optional)** — text area (max 2,000 characters) to specify materials, lighting, mood.
  - **Quality**: Pro / Standard / Fast.
  - **Resolution**: 2K / 4K (4K disabled in Fast quality).
  - **"Generate render"** — deducts the actual cost from the shared balance. Blocked if the balance is below the minimum.
  - **Result** — the generated image, its cost, an optional low-balance reminder, then **"Download"** (PNG) or **"Regenerate"**.

A banner appears if the balance is exhausted, with a **"Top up"** button (which then brings you back to the render page).

### 2.14 Public approval page (`/estimation/s/:ptoken`)

This is the page **your client** opens to view and decide. It is **read-only** and **opens no access** to your balance or your chat: its address rests on a **distinct public token** (`ptoken`).

- **Header** — the quote number, the issuer's name, **zoom** controls (zoom out / **clickable percentage = fit to screen** / zoom in, between 30% and 200%) and **"Print / PDF"** (printing at 100%).
- **Document** — the quote appears in a secure frame, width-fitted on mobile, with manual zoom.
- **State banners** — "This quote has been approved" or "This quote has been declined", where applicable.
- **Client actions** (as long as no decision has been made):
  - **"Decline"** and **"Approve quote"** buttons;
  - **approval panel**: a **"Your full name"** field (minimum 2 characters) + **"Your signature"** drawn by hand in a **signature canvas** (with a **"Clear"** button), then **"Confirm approval"**;
  - **decline confirmation**: a confirmation message, then **"Confirm decline"**.
- Footer: **"Secure viewing and approval — Constructo AI"**.

> **A drawn signature is required to approve**: the name alone is not enough. Once the decision is made, it is **final** (a second attempt on an already-decided document is refused). The issuer is **notified by email** of both approval and decline.

---

## 3. Step-by-step workflows

### 3.1 First visit: create a session and top up

1. Open `app.constructoai.ca/estimation`. A **session** (empty wallet) is created automatically — nothing to do.
2. Click **"Top up"**.
3. Choose a credit amount (**$25 / $50 / $100**). The total to pay, **taxes included**, is shown (for example $28.74 for $25 of credits).
4. You are redirected to **Stripe**: pay by card.
5. Back on Estimation Express, your **balance** is credited automatically. You are ready to estimate.

> **Tip.** Before topping up, consider **linking your email** (§3.2): your wallet will then be recoverable if you switch devices.

### 3.2 Link your wallet to an email (multi-device access)

1. Click **"My Account"**.
2. Enter your **email**, then **"Send code"**.
3. Read the **6-digit code** received by email (valid for 15 minutes).
4. Enter it, then **"Verify and access my balance"**.
5. Your wallet is now **recoverable everywhere**: on another device, do "My Account" again with the same email and a new code.

> If the email **already** has a wallet, the application switches to it. An unlinked anonymous balance present on the current device will **no longer be accessible** (warning shown) — balances do not merge.

### 3.3 Estimate a project (chat + plans)

1. In the **"AI expert"** selector, choose the specialist you want (general contractor, electrician, plumber, roofing, etc.) or leave **"Generic Assistant"**.
2. **Describe your project** in the input box: nature of the work, area, finish level (economy / standard / high-end), constraints, desired schedule. The more precise you are, the better the estimate.
3. If needed, **attach your plans** (paperclip icon): up to **5 files**, PDF or images, **32 MB** each.
4. Press **Enter** (or **"Send"**). The AI replies with a priced estimate (often as a table: items, quantities, labor, materials). The **cost** of the answer is shown below and your balance drops accordingly.
5. **Iterate**: ask for adjustments ("redo it in an economy version", "add the demolition", "break down the electrical"). Each exchange refines the estimate.

> **Cost tip.** A targeted question costs less than a long plan analysis. Keep an eye on the **"Balance"** panel (green / amber / red). Below **$2**, sending is blocked: top up to continue.

### 3.4 Create a custom AI expert

1. Click **"AI Profiles"** → **"New profile"**.
2. Give it a **name** (e.g., "General contractor – residential renovation").
3. Write the **instructions**: role, expertise, pricing rules, tone. Be explicit (up to 50,000 characters).
4. **Save**.
5. (Optional) Reopen the profile and **add reference documents** (PDF, TXT, CSV, TSV, MD, XLSX, DOCX; 5 max, 20 MB each): price schedules, quote templates, technical notes. The AI extracts the text and uses it when this profile is active.
6. Then select this profile in the **"AI expert"** menu (**"My Profiles"** group) before writing.

### 3.5 Customize the document's appearance

1. Click **"Appearance"**.
2. Choose a **preset theme** (Constructo Blue, Forest Green, Brick Red, Charcoal, Burgundy, Ocean) or adjust the **8 colors** by hand.
3. If needed, write your **Default terms** and **Default exclusions** (one line per item).
4. Check the **preview**, then **Save**. Your next quotes will adopt this style.

### 3.6 Generate a quote

1. After **at least one exchange** with the AI, click **"Generate quote"**.
2. Fill in the **Project name**.
3. Fill in **"Your company (issuer)"** — the **Company name is required**; add email, phone, address, RBQ, and your **logo** as needed (max 600 KB).
4. Fill in **"Your client (recipient)"** — name, and **email** if you want the approval link to be **sent automatically** to them.
5. Click **"Generate"** (an amount is deducted from your balance based on AI usage).
6. On the result screen: **copy the public link**, **open** the quote to check it, or let the email go to your client.

> **How the price is built.** The AI prices the **base costs** (materials, labor, equipment), then the document applies a **markup** on top (administration, contingencies, **15% profit**) and the **taxes** (GST 5%, QST 9.975%). These percentages are **standard and fixed** — they cannot be set in the generation screen.

### 3.7 Add a 3D render to the quote

1. From the quote result screen, click **"Add a 3D render"**.
2. **Upload** a plan (PDF) or an image (3D files are not accepted here).
3. **Crop** the area to illustrate.
4. Set **Details**, **Quality**, and **Resolution**, then **"Generate render"** (deducted from the balance).
5. **"Attach to quote"** (free). The render appears at the bottom of the document, including on the client's public page.

> **One** render per quote. You can **Replace** or **Remove** it at any time (both actions are free).

### 3.8 Have the client approve the quote (public page)

On **your client's** side:

1. They open the **public link** (received by email or that you forwarded to them).
2. They **review** the quote (zoom, print / PDF possible).
3. To accept: they click **"Approve quote"**, enter their **full name**, **draw their signature**, then **"Confirm approval"**.
4. To decline: they click **"Decline"**, then **"Confirm decline"**.

On **your** side: you receive an **email** informing you of the decision. The decision is **final** (no going back on the same document).

### 3.9 Generate a standalone 3D render

1. Click **"3D Render"** in the header (or open `app.constructoai.ca/estimation/rendu`).
2. **Import** a 3D model (GLB / GLTF / OBJ / STL / FBX, max 50 MB), an image (PNG / JPG / WEBP) or a PDF plan (choose the desired page if the PDF has several).
3. For a 3D file: **orient** the model to frame the desired angle.
4. Choose the **Type** (Exterior / Interior / Object), write **Details** if needed, set **Quality** and **Resolution**.
5. **"Generate render"** (deducted from the shared wallet).
6. **"Download"** the image (PNG) or **"Regenerate"**.

### 3.10 Manage conversation history

*(Requires having linked an email — §3.2.)*

1. Click **"History"**.
2. **Reopen** a conversation by selecting it, **Rename** it (pencil icon) or **Delete** it (trash icon, irreversible).
3. Create new threads with **"New conversation"**: each is kept separately and recoverable across all your devices.

### 3.11 Sign out or change email

1. **"My Account"** → **"Sign out"** starts a **new blank session** on the device.
2. If your balance was **not** linked to an email, a warning notes that it will become **inaccessible** — link it first if you want to keep it.
3. To **change email**, relaunch "My Account" and the **"Change email / resend a code"** link.

---

## 4. Reference

### 4.1 Screens and routes

| Screen | Route | File | Access |
|---|---|---|---|
| Chat (main) | `/estimation` | `ChatPage.tsx` | Public (auto session) |
| Standalone 3D render | `/estimation/rendu` | `RenderPage.tsx` | Public (shared wallet) |
| Public approval | `/estimation/s/:ptoken` | `SoumissionPublicPage.tsx` | Public (read-only, distinct token) |
| *(any other address)* | `*` | — | Redirects to `/estimation` |

### 4.2 API endpoints (≈ 30)

All prefixed with `/api/estimation/v1`. **No authentication**; ownership is verified by the **session token** (the wallet) or, for a shared quote, by the **public token** (read + decision only).

**Configuration and static data**

| Method | Path | Purpose |
|---|---|---|
| GET | `/config` | Top-up amounts, taxes, minimum balance, file count |
| GET | `/experts` | List of system experts |
| GET | `/document-defaults?lang=` | Default texts (terms / exclusions) |

**Session, account, and credits**

| Method | Path | Purpose |
|---|---|---|
| POST | `/session` | Creates a session (wallet, balance 0) |
| POST | `/account/request-code` | Sends a 6-digit code by email |
| POST | `/account/verify-code` | Verifies the code → restores / attaches / creates the wallet |
| POST | `/topup` | Creates the Stripe payment session ($25 / $50 / $100) |
| POST | `/topup/confirm/{token}` | Verifies the payment and credits (idempotent) |
| GET | `/balance/{token}` | Balance, email, total spent |

**Chat and conversations**

| Method | Path | Purpose |
|---|---|---|
| GET | `/history/{token}` | Messages of the active conversation |
| POST | `/chat/{token}` | One chat turn (message, expert, language, files) |
| POST | `/chat/{token}/reset` | New conversation (signed in) / clear (anonymous) |
| GET | `/conversations/{token}` | List of history threads |
| PATCH | `/conversations/{token}/{id}` | Rename a conversation |
| DELETE | `/conversations/{token}/{id}` | Delete a conversation |

**Custom AI profiles** (private to the wallet)

| Method | Path | Purpose |
|---|---|---|
| GET | `/profiles/{token}` | List your profiles |
| POST | `/profiles/{token}` | Create a profile (balance ≥ $2, max 20) |
| PUT | `/profiles/{token}/{id}` | Edit name / instructions |
| DELETE | `/profiles/{token}/{id}` | Delete (+ documents) |
| GET | `/profiles/{token}/{id}/documents` | List the documents |
| POST | `/profiles/{token}/{id}/documents` | Upload a document (balance ≥ $2, max 5, 20 MB) |
| DELETE | `/profiles/{token}/{id}/documents/{docId}` | Delete a document |

**Quote**

| Method | Path | Purpose |
|---|---|---|
| POST | `/soumission/{token}` | Generate the document + the public link (charges) |
| POST | `/soumission/{token}/{id}/render` | Attach a 3D render (free) |
| DELETE | `/soumission/{token}/{id}/render` | Remove the 3D render |
| GET | `/soumission/public/{ptoken}` | Public view (read-only) |
| POST | `/soumission/public/{ptoken}/accept` | The client approves (name + signature) |
| POST | `/soumission/public/{ptoken}/refuse` | The client declines |
| GET | `/soumissions/{token}` | List of the wallet's quotes *(API only, not displayed, see §4.9)* |

**3D render**

| Method | Path | Purpose |
|---|---|---|
| POST | `/render/{token}` | Generates a render (charges the actual cost) |

### 4.3 Billing model (how the cost is calculated)

Estimation Express bills you the **actual cost** of the AI, never a flat fee. Two formulas:

| Action | Cost charged |
|---|---|
| **Chat response** and **quote generation** | Actual cost of the Claude Opus tokens (in USD) **× 1.38** (USD → CAD conversion) **× 3.0** (markup) |
| **3D render** (standalone or for a quote) | Actual cost of the Gemini image (in USD) **× 1.38 × 3.0** |

- **Hold then settle.** On sending, an amount is **held**; after the answer, the **actual cost** is charged and the **surplus refunded**. You are never charged beyond the hold, nor for a failed operation (full refund).
- **The cost is shown** under each answer ("Cost: $X.XX") and for each render.
- **Minimum balance: $2.** Below this, sending, generating, and rendering are **blocked** (grayed-out button). There is **no free** production action (only viewing, attaching a render, and client-side approval cost nothing).

**Building a quote's price.** The document does not simply copy the costs: on the **sum of the base costs** (materials + labor + equipment, **without markup**), it applies a markup of **≈ 30%** — administration (≈ 3%), contingencies (≈ 12%) and a **fixed 15% profit** — then the **taxes** (GST 5%, QST 9.975%). These percentages are **fixed server-side** and **cannot be changed** in the current interface (see §4.9).

### 4.4 Top-ups and taxes

| Credits added to balance | Taxes (on top) | Total to pay |
|---|---|---|
| **$25** | GST 5% + QST 9.975% | **$28.74** |
| **$50** | GST 5% + QST 9.975% | **≈ $57.49** |
| **$100** | GST 5% + QST 9.975% | **≈ $114.98** |

- The **credits** match the nominal amount ($25 / $50 / $100); the **taxes are added on top** at payment time.
- Payment by **card, via Stripe Checkout** (CAD currency, Canadian-French interface).
- Crediting the balance is **idempotent**: verified on return from Stripe and caught up by a safety net at startup — **never a double credit**.
- There is **no** Stripe webhook for Express: confirmation happens by **polling** on return from payment (refresh the page if the balance is slow to update).

### 4.5 Statuses

| Domain | Values |
|---|---|
| **Quote** | envoye (sent) · accepte (accepted) · refuse (declined) |
| **Top-up** | not credited → credited (`credited`) |
| **Credit hold** | held · settled · refunded |
| **Login code (OTP)** | active → consumed (one-time, 15 min, 5 attempts max) |

### 4.6 Limits and bounds

| Item | Limit |
|---|---|
| Minimum balance to act | **$2** |
| Chat message | 1 to 12,000 characters |
| Attached plans — per message | **5 files**, PDF / PNG / JPEG / WEBP, **32 MB** per file |
| Attached plans — total per message | 40 MB |
| Attached plans — cumulative per wallet | 15 files |
| Conversation context retained by the AI | last 16 turns |
| Custom AI profiles | **20** per wallet |
| Profile name / instructions | 120 / 50,000 characters |
| Reference documents per profile | **5**, PDF / TXT / CSV / TSV / MD / XLSX / DOCX, **20 MB** each |
| Conversations kept | 300 per wallet |
| Quote logo | **600 KB**, PNG / JPG / WEBP |
| Approval signature | drawing required (image), name ≥ 2 characters |
| Render details | 2,000 characters |
| Standalone 3D render — 3D | GLB / GLTF / OBJ / STL / FBX, **50 MB** |
| Default terms / exclusions | 10,000 characters each |
| Account email | 254 characters |

### 4.7 The AI experts (about sixty)

The **"AI expert"** selector offers:

- **"Generic Assistant"** — a general-purpose estimator (the default fallback);
- your custom **"My Profiles"** (§2.9), if any;
- an **"Experts"** group — **about sixty** specialists supplied by the platform.

These system experts are **loaded dynamically** from the platform's profile files (`ai_profiles`); their **exact number depends on the deployed files** (it is not hard-coded). Among them are, for instance: **Architect, Electrician, Plumber, HVAC (heating, ventilation, air conditioning), Roofing, Foundations, Excavation, Masonry, Structural engineer, Construction accountant, RBQ and CCQ (Commission de la construction du Québec), Framing / wood structure**, and many other trades.

> **Key takeaway.** Choosing the right expert **steers** the AI's vocabulary, units, and pricing rules. For a multi-trade estimate, the **general contractor** or the **Generic Assistant** are a good fit; for a specific scope (electrical, roofing), choose the corresponding specialist.

### 4.8 Rate limits (anti-abuse)

To protect the service, each type of action is **capped per hour**, both per IP address and per wallet:

| Action | Indicative cap |
|---|---|
| Chat | 60/h per IP · 40/h per wallet |
| 3D render | 30/h · 20/h |
| Quote generation | 20/h · 15/h |
| Top-up / confirmation | 30/h |
| Sending a code by email | 15/h per IP · 5 per 30 min per email |
| Profiles — read / write | 180/h · 60/h |
| Session creation | 30/h |

If you hit a limit, wait a moment before trying again ("Too many requests. Please wait…").

### 4.9 Pitfalls and things to know

- **Your wallet lives in the browser.** Without an **email link** (§3.2), clearing your browsing data or switching devices makes you **lose access** to it.
- **1 email = 1 wallet**, never a merge: linking an already-used email switches to its wallet and makes the current anonymous balance **inaccessible** (warning shown).
- **Balance below $2 = everything is blocked**: there is no free production action.
- **The Administration / Contingencies / Profit percentages are NOT adjustable** in the quote-generation screen. Although labels and technical support exist behind the scenes, **no field** makes them editable in the current interface: the server applies standard values (fixed **15%** profit). Do not count on setting them per quote.
- **No quote-tracking dashboard in the application.** There is **no screen** listing your past quotes and their approval status. You learn of the client's decision by **email**. Keep the **public link** of each generated quote.
- **Quote render = image / PDF only**, **one** render, replaceable or removable. 3D files are accepted only on the **standalone** render page (`/estimation/rendu`).
- **4K unavailable in "Fast" quality** (Fast favors speed; use Standard or Pro for 4K).
- **History only appears once linked to an email.** When anonymous, you have only one "default" conversation, cleared by "New conversation".
- **The drawn signature is required** to approve on the client side; the name alone is not enough.
- **Indicative estimates.** As the disclaimer recalls, these are estimates **generated by AI, to be validated by a professional** before any contractual commitment.

---

## 5. Integrations and FAQ

### 5.1 Links with the rest of Constructo AI

- **ERP AI Estimating (module 24 / `devis.py`).** Estimation Express is the **public clone** of that function. Same engine (Claude Opus), same expert profiles, same document generator — but **without an ERP account**, on a **pay-as-you-go** basis. Conversely, in the ERP, AI estimating is included in the tenant's subscription.
- **3D Render (module 27).** The `/estimation/rendu` page is the **public version** of the render module, with the **same wallet** as the chat. Image engine: **Google Gemini**.
- **SEAOP (module 36).** Not to be confused: SEAOP is **free** and serves **public tenders**; its professional estimate service is produced **by the team** and billed offline. Estimation Express, on the other hand, is **self-service, pay-as-you-go**, and the estimate is produced **by the AI** live.
- **Stripe.** Serves only the **top-ups** (card payment). The Stripe key is **shared with the ERP** server-side and is **never** exposed. No subscription: only one-off credit purchases.
- **Claude (Anthropic) and Gemini (Google).** The AI engines. You are charged only their **actual cost** of tokens / image, converted and marked up (§4.3).
- **Email (system SMTP).** Serves the **login codes** (OTP), the **sending of the quote link** to the client, and the **decision notification** to the issuer. These sends are "best-effort" (they never block an operation).

### 5.2 What is NOT possible

- **No account or password.** Only a browser token and, optionally, a link via a **one-time code**.
- **No tracking dashboard** for quotes or approvals in the application (tracking goes through the decision email and your kept public links).
- **No setting of the financial percentages** (administration, contingencies, profit) in the quote screen.
- **No 3D render from a 3D file in a quote context** (image / PDF only); nor more than **one** render per quote.
- **No 4K in Fast quality.**
- **No wallet merging** between two emails.
- **No subscription**: only prepaid credits.

### 5.3 Frequently asked questions

**Do I need to create an account to use Estimation Express?**
No. A session is created automatically when you arrive. You may optionally **link an email** ("My Account") to recover your wallet on other devices — but always **without a password**, via a one-time code.

**How much does it cost?**
You buy **credits** ($25 / $50 / $100, taxes on top). Each AI response, each quote, and each render **charges the actual cost** of the AI, marked up (× 1.38 × 3). The cost is shown under each answer. You need at least **$2** of balance to act.

**Why do $25 of credits cost me $28.74?**
The **taxes** (GST 5% and QST 9.975%) are added **on top of** the credit amount. You do get $25 of credits, and you pay $28.74 including taxes.

**I topped up but my balance did not move.**
The credit is confirmed **on return from Stripe**. **Refresh the page**: the check is automatic and cannot credit twice. A safety net also catches confirmed payments at server startup.

**I'm switching computers: will I recover my balance?**
Only if you **linked it to an email**. Otherwise, the wallet stays attached to the original browser.

**Can I merge two balances?**
No. **1 email = 1 wallet**, no merge. Switching to an already-used email makes the current anonymous balance inaccessible (the application warns you).

**Is the estimate contractual?**
No. These are **indicative estimates generated by AI**, to be **validated by a professional** before any commitment.

**How does my client approve?**
They open the quote's **public link**, review it, then click **"Approve"**, enter their **name** and **draw their signature**, or click **"Decline"**. You are notified of their decision **by email**.

**Can I change the profit or the contingencies of a quote?**
Not in the current interface. The document applies **standard fixed percentages** (15% profit) on top of the base costs.

**Where is the list of my quotes?**
There is **no** tracking screen in Estimation Express. Keep the **public link** of each quote; the client's decision reaches you by **email**.

**What is the difference with SEAOP's estimating?**
SEAOP is **free** and connects parties; its "estimate request" is produced **by the team** and billed offline. Estimation Express is **self-service, pay-as-you-go**, and the estimate is produced **by the AI** immediately.

**Can I upload a 3D model into a quote?**
No: in a quote context, only **images and PDFs** are accepted. For a render from a **3D** file, use the standalone **3D Render** page (`/estimation/rendu`).

**Is the service bilingual?**
Yes. The **FR / EN** toggle changes the interface, the **language of the AI's answers**, and the **language of the document**. The public page displays in the language chosen when that document was generated.

### 5.4 Common troubleshooting

| Symptom | What to check |
|---|---|
| Cannot send a message | Balance below **$2**: top up. |
| Balance unchanged after payment | **Refresh** the page (confirmation on return from Stripe, idempotent). |
| "History" button missing | It only appears once the wallet is **linked to an email** ("My Account"). |
| "Too many requests" | Rate limit reached: wait a moment (§4.8). |
| Anonymous balance "gone" after linking an email | You switched to **another** wallet (1 email = 1 wallet, no merge). |
| File refused in the chat | Check the format (PDF / PNG / JPEG / WEBP), the size (**32 MB**) and the count (**5** max). |
| 3D file refused in a quote | Normal: image / PDF only here. Use `/estimation/rendu` for a 3D file. |
| 4K grayed out | You are in **Fast** quality: choose Standard or Pro. |
| The client cannot approve | The **drawn signature** is required (the name alone is not enough). |
| "The AI service is temporarily unavailable" | Try again: **no credit was deducted** (automatic refund). |
| I can't find my wallet | Without an email link, it stays attached to the original browser. |

---

## 6. Summary

- **Estimation Express is a paid, pay-as-you-go public application** that puts Constructo AI's AI estimating within everyone's reach, **without an ERP account or subscription**. It is the **public clone** of the ERP's AI Estimating (same engine), served under `app.constructoai.ca/estimation`.
- **The model is that of a prepaid card**: you buy **credits** ($25 / $50 / $100, taxes on top, via **Stripe**), and each **AI response**, **quote**, or **3D render** charges the **actual cost** of the AI, converted (× 1.38) and marked up (× 3). Minimum balance to act: **$2**. Hold then settle, with a **refund of the surplus** and of failed operations.
- **No password.** Identity is a **browser token** (= the wallet); it can be **linked to an email** via a **one-time code** to recover it on any device. **1 email = 1 wallet**, no merge.
- **Three screens**: the estimating **chat** (with experts, custom profiles, appearance, history, and quote generation), the standalone **3D render** (shared wallet), and the **public** approval page (the client views and **signs**).
- **The quote** is a themed professional document, with logo, terms, and exclusions; its price applies a standard markup (**fixed 15% profit**) and the taxes. The client approves it with a **drawn signature**; the issuer is **notified by email**.
- **About sixty AI experts** (loaded dynamically, number not fixed) steer the pricing; you can create your **own profiles** (instructions + reference documents).
- **Key limits to know**: the **financial percentages cannot be changed** in the interface, there is **no tracking dashboard** for quotes (decision by email), the quote render is limited to **one** image / PDF, and **4K** is unavailable in **Fast** quality.
- **Technical volume**: a **single backend file** (`estimation_express.py`, ≈ 4,089 lines, ≈ 30 endpoints under `/api/estimation/v1`), `public.express_*` tables, reusing `devis.py`, `gemini_image.py`, and `ai_profiles.py`.

---

*Verified source files:* backend — `ERP_REACT/backend/routers/estimation_express.py` (≈ 4,089 lines, the module's single file; mounted at `erp_api.py:1399` under `/api/estimation/v1`, SPA at `erp_api.py:1408`). Frontend (`ESTIMATION_EXPRESS_REACT/frontend/src/`) — `main.tsx` (15, basename `/estimation`), `App.tsx` (routes), `pages/ChatPage.tsx` (2,378), `pages/RenderPage.tsx` (840), `pages/SoumissionPublicPage.tsx` (382), `components/soumission/SoumissionRenderModal.tsx` (592), `components/SignatureCanvas.tsx`, `components/render/Rendu3DDropzone.tsx`, `components/render/Rendu3DControls.tsx`, `components/render/Rendu3DViewer.tsx`, `components/PlanCropper.tsx`, `api/client.ts`, `api/estimation.ts`, `i18n.ts`, `locales/{fr,en}/translation.json`. Verified billing constants: `USD_TO_CAD = 1.38` (`estimation_express.py:91`), `MARKUP = 3.0` (`:92`), `RENDER_MARKUP = 3.0` (`:3878`), `_MIN_BALANCE_CAD = 2.0` (`:101`), `_TOPUP_AMOUNTS = [25, 50, 100]` (`:95`), `TPS_RATE = 0.05` / `TVQ_RATE = 0.09975` (`:96-97`), `_MAX_FILES = 5` / `_MAX_FILE_BYTES = 32 MB` (`:110-112`).

*Related manuals:* `24-communication-assistant-ia.md` (the ERP's internal AI estimating, of which Estimation Express is the public clone), `27-conception3d-rendu-3d.md` (the 3D-rendering module from which `/estimation/rendu` derives), `07-ventes-soumissions.md` (the ERP's internal quotes), `36-programme-seaop.md` and `35-programme-portail-b2b.md` (the other standalone applications served by the ERP).

*Constructo AI ERP Manual — Module 37 "Estimation Express (paid public sub-app)" — v1.0 verified — 2026-07.*
