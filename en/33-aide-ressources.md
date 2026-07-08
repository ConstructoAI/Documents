# Module 33 — Help and resources

> **Version**: 3.0 (overhaul verified against the source code, July 2026)
> **Reference code**: `frontend/src/components/layout/Sidebar.tsx:244-297` (the sidebar's "Help & Resources" block — separator/divider 245, collapsible header 246-257, table of the **4 links** 259-262, rendering of each link 263-297); `frontend/src/components/layout/TopBar.tsx:312-320` (**AI Assistant** button, the top bar's only *in-app* help entry point); `Sidebar.tsx:382-389` (**mobile** drawer footer: email + phone + version); i18n `frontend/src/i18n/locales/fr/nav.json:41-47` and `en/nav.json:41-47`
> **PostgreSQL tables**: **none**. **No** FastAPI endpoint, **no** internal React route (`/aide` does not exist), **no** role guard, **no** money effect, **no** network call to the ERP server.
> **Scope**: "Help and resources" is **not a functional module**. It is a **static block of 4 links** rendered at the bottom of the left sidebar, below every navigation group and above the Super-Admin section. Three of these links **leave the application** (YouTube channel, two Markdown files on GitHub) and a fourth triggers the support **phone dialer**. All the logic lives on the client side; that is why this manual is legitimately shorter than those for the real modules. For **AI-assisted help inside** the ERP, this is not the section: it is the **AI Assistant** button in the top bar (a separate module, see `24-communication-assistant-ia.md`).

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

### 1.1 Section mission

To offer, from any screen in the ERP, a **single, always-visible entry point** to help and documentation, without building anything inside the application:

- **Videos** — Constructo AI's official YouTube channel (tutorials, demos, new features).
- **Manual** — the index of the user documentation (this repository), publicly hosted on GitHub.
- **Phone** — the support number, which opens the device's dialer directly.
- **Useful links** — a page of curated references for the Quebec contractor: RBQ (Régie du bâtiment du Québec, the provincial building authority), CCQ (Commission de la construction du Québec, the construction workforce commission), CNESST (the workplace health and safety board), Revenu Québec, etc.

It is a **signpost panel**, not a built-in knowledge base. It contains no help text of its own: it **points** to content that lives elsewhere (YouTube, GitHub, the support phone line).

### 1.2 What the section does — and does not do

| The section **does** | The section **does not** |
|---|---|
| Display 4 help shortcuts at the bottom of the sidebar | Host a help page *inside* the ERP (no `/aide` route) |
| Open the video channel and the documentation in a new tab | Search the documentation (no help search engine) |
| Trigger a call to support (a `tel:` link) | Open a contact form or a ticketing system |
| Stay visible for **every** signed-in user | Filter by role (an employee sees the same links as an admin) |
| Collapse / expand with one click on its header | Be configured through the interface (the URLs are hard-coded) |
| — | Show contextual help tooltips, a guided tour, or a "?" button |
| — | Track reading progress, save favorites, or log clicks |

### 1.3 An important point: this is not a "module" in the usual sense

Unlike the other chapters of this manual, there is **nothing to describe on the server side**:

- **0 FastAPI endpoints** — no `aide`, `help`, `support`, `contact`, or `feedback` router in `ERP_REACT/backend/`.
- **0 PostgreSQL tables or columns** touched by the section.
- **0 internal React routes** — the links are `<a href>` tags, **not** internal navigation links (`NavLink`).
- **0 permission guards** specific to the section (no `is_admin`, no `super_admin`, no consultation mode).
- **0 paid integrations** — no Stripe, no AI credits, no billing triggered by these 4 links.

The only hardening present is client-side: the `rel="noopener noreferrer"` attribute on the external links (protection against tab hijacking, *tabnabbing*).

### 1.4 Access

- **Location**: left sidebar, at the very bottom, after the last group of modules (see `Sidebar.tsx:244`).
- **Header**: the label "**HELP & RESOURCES**" (already uppercase in the translation, `en/nav.json:41`).
- **Always available**: the sidebar appears only in the protected shell (after sign-in), so the section is present on **every** page of the ERP.
- **No internal address**: each click opens a new tab to an external site (YouTube or GitHub) or launches the phone dialer — the ERP itself stays open behind it.

### 1.5 Permissions

**None.** The 4 links are rendered **outside** the role-filtering loop (`canSeeItem`, `Sidebar.tsx:144-152`), which applies only to the navigation groups (`NAV_GROUPS`). As a result: **every authenticated user** — administrator, owner, accountant, unprivileged employee, super-administrator — sees exactly the same 4 resources. The targets (YouTube, GitHub) are public.

---

## 2. Interface

The section's entire interface fits at the bottom of the sidebar (`Sidebar.tsx:244-297`), with behavior that varies by display mode.

### 2.1 The "HELP & RESOURCES" header

Source: `Sidebar.tsx:245-257`.

- A horizontal **divider** (`border-t border-white/10`) marks the start of the block, just above the title (`Sidebar.tsx:245`).
- The title is a **collapsible button** (`<button>`): clicking it collapses or expands the list of 4 links (`toggleGroup('nav.aideRessources')`, `Sidebar.tsx:247-248`).
- Style: small light-gray uppercase text with letter-spacing (`text-[11px] font-semibold uppercase tracking-wider text-white/60`, `Sidebar.tsx:249`) — consistent with the sidebar's dark-navy theme (`erp-sidebar`).
- To the right of the title, a **chevron** (`ChevronDown`, size 12) indicates the state: it rotates a quarter turn (`-rotate-90`) when the section is collapsed (`Sidebar.tsx:252-255`).
- **Default state = expanded**: on first display, the `nav.aideRessources` key is undefined in local state, which yields "not collapsed" (`?? false`, `Sidebar.tsx:254, 258`). The 4 links are therefore visible right away.
- This collapse state is a **session display preference** (local React state, `Sidebar.tsx:120`): it holds as long as the application stays open, but **reverts to "expanded" after a full page reload** (it is not persisted on the workstation).

### 2.2 The 4 links

Source: `Sidebar.tsx:259-262`. Each entry is an `<a href>` tag (so it **leaves** the application; it is not internal navigation).

| Order | Label (EN) | i18n key | Destination | Type | Icon | Trailing element |
|---|---|---|---|---|---|---|
| 1 | **Videos** | `nav.helpResources.videos` | `https://www.youtube.com/channel/UC3EGXYQNj5UYGiyNfiiom_A` | External (new tab) | `Video` (18) | "External link" glyph (`ExternalLink`, 10, dimmed) |
| 2 | **Manual** | `nav.helpResources.manuel` | `https://github.com/ConstructoAI/Documents/blob/main/README.md` | External (new tab) | `BookOpen` (18) | "External link" glyph |
| 3 | **Phone** | `nav.helpResources.telephone` | `tel:+15148201972` | **Non-external** (dialer) | `Phone` (18) | **Text** `1 514 820-1972` |
| 4 | **Useful links** | `nav.helpResources.usefulLinks` | `https://github.com/ConstructoAI/Documents/blob/main/liens-utiles.md` | External (new tab) | `ExternalLink` (18) | "External link" glyph |

Rendering rules (`Sidebar.tsx:263-295`):

- **Label**: comes from the translation, `t(link.labelKey)` (`Sidebar.tsx:264`) — it is no longer hard-coded in the component.
- **New tab**: only for external links. `target="_blank"` and `rel="noopener noreferrer"` are applied **only** when `external` is true (`Sidebar.tsx:269-270`). The Phone link (`external:false`) therefore opens **no** tab.
- **Trailing element** (`Sidebar.tsx:288-292`): if the link is external, a small dimmed "external link" glyph appears on the right; otherwise, if it has trailing text (`trailingText`), that text is shown (the Phone case, which displays the number `1 514 820-1972`); otherwise, nothing.
- None of these links go through the role filter: they are rendered **unconditionally** for every signed-in user.

#### 2.2.1 Videos

Constructo AI's official YouTube channel. It hosts the getting-started tutorials, per-module demos, and new features. The link opens YouTube in a new tab; the ERP stays open behind it.

#### 2.2.2 Manual

Points to the **README.md** of the public `ConstructoAI/Documents` repository — the **index** of the user documentation (the same repository that contains this file). This README lists all the manuals (Main, Tracking, Management, Sales, Operations, Field, Communication, Tools, Configuration, Help). It is a **lightweight** resource: the README is a table of contents from which you open the manual for the module you want.

> The manual's content **leaves the ERP**: it lives on GitHub, requires an Internet connection, and is not rendered in an in-app viewer.

#### 2.2.3 Phone

`tel:+15148201972` link. It is the section's **only non-external link** (`external:false`, `Sidebar.tsx:261`): it opens no tab; it asks the operating system to dial the number (useful mainly on mobile, or with softphone software on a computer). The number appears as text to the right of the label: **1 514 820-1972**.

#### 2.2.4 Useful links

Points to **liens-utiles.md** in the same GitHub repository. It is a page of **curated** references for the Quebec contractor, organized by theme: regulations and licensing (RBQ, GCR — Garantie de construction résidentielle, the residential construction warranty), labor and collective agreements (CCQ), health and safety (CNESST), taxation (Revenu Québec, CRA — the Canada Revenue Agency), public procurement and calls for tenders, grants and financing, codes and standards (CNB — the National Building Code, CSA), integrity registers, associations and professional orders, reference legislation, industry watch, and online calculators.

> Like the Manual, this page **leaves the ERP** to GitHub. Its content evolves independently of the application (the URLs there were validated as of its own revision date).

### 2.3 Behavior by display mode

Source: `Sidebar.tsx:246, 258, 271-280`.

| Mode | "HELP & RESOURCES" header | The 4 links |
|---|---|---|
| **Desktop, expanded sidebar** (`lg:w-56`) | Visible and collapsible | Icon + label + trailing element |
| **Desktop, collapsed sidebar** (`lg:w-14`) | **Hidden** (the header shows only in expanded mode) | Rendered **as icons only**; the label (and the number, for Phone) appears as a **tooltip** on hover (`title` attribute, `Sidebar.tsx:271`) |
| **Mobile** (full-screen drawer) | Visible and collapsible | Icon + label, with enlarged touch targets (`min-h-[44px]`, `Sidebar.tsx:277`) |

Details:

- **Collapsed sidebar**: even when collapsed, the section shows its 4 icons (`Video`, `BookOpen`, `Phone`, `ExternalLink`), each with a tooltip. This is the only way to keep help accessible when the sidebar is in icon mode.
- **Mobile**: the drawer closes automatically on *internal* navigation, but these external links do not change route (they open a tab or the dialer), so the drawer behaves as expected.

### 2.4 The mobile drawer footer (support contact details)

Source: `Sidebar.tsx:382-389`. **Mobile mode only.**

At the bottom of the mobile drawer — and **nowhere on desktop** — there is a footer that shows the real support contact details:

- **Version**: "Constructo AI ERP AI v1.0" (`Sidebar.tsx:383`).
- **Email**: `mailto:info@constructoai.ca`, displayed as **info@constructoai.ca** (`Sidebar.tsx:385`). This is the **only place in the sidebar** that exposes the email address as a clickable link.
- **Phone**: `tel:+15148201972`, displayed as **1 (514) 820-1972** (`Sidebar.tsx:387`).

> **Note**: on **desktop**, the sidebar footer (`Sidebar.tsx:344-353`) contains **only** the button to collapse / expand the sidebar — no email, no version. The only contact present in the desktop sidebar is therefore the **Phone** link in the Help section.

### 2.5 The in-app help button: AI Assistant (top bar)

Source: `TopBar.tsx:312-320`.

**In-app assisted** help is not in the sidebar but in the **top bar**: a sparkles-shaped button (`Sparkles`, size 18) that leads to the AI Assistant (`navigate('/assistant-ia')`). Its label, its tooltip, and its accessibility label are all "**AI Assistant**" (`t('nav.assistantIa')`).

- **Global access**: this button is present on every page, to the left of the search bar. On mobile it is a 44 × 44 icon; the "AI Assistant" text appears only from a very large screen up (the `xl` breakpoint).
- **Single entry point**: the AI Assistant is **not** in the sidebar (absent from `NAV_GROUPS`). This top-bar button is the **only** visible way to open `/assistant-ia`.
- **Separate module**: the AI Assistant has its own server (`routers/ai.py`) and its own AI-credit billing — it is **not** part of the Help and resources section. See `24-communication-assistant-ia.md`.

> There is **no dedicated "?" or "Help" button** in the top bar. The only other utility-looking button is the gear (`Settings`) that leads to Configuration (`TopBar.tsx:539-544`) — unrelated to help.

---

## 3. Step-by-step workflows

These procedures follow directly from the behavior described in section 2. None of them triggers a call to the ERP server.

### 3.1 Open the video tutorials

1. In the left sidebar, scroll down to the "HELP & RESOURCES" section.
2. Click **Videos**.
3. A new tab opens on Constructo AI's YouTube channel. The ERP stays open in the previous tab.

### 3.2 Consult the user manual

1. "HELP & RESOURCES" section → **Manual**.
2. A new tab opens on the documentation README (GitHub).
3. From the table of contents, open the manual for the module you need. Tip: use the browser's search (`Ctrl+F` / `Cmd+F`) on the GitHub page to find a keyword.

> An Internet connection is required (the content is hosted on GitHub, outside the ERP).

### 3.3 Call support

1. "HELP & RESOURCES" section → **Phone** (number displayed: 1 514 820-1972).
2. On a device with telephony (mobile, or calling software on a computer), the dialer opens with the number pre-filled. No new tab opens.
3. On a workstation without a telephony application, the click may do nothing: dial the number manually in that case.

### 3.4 Email support

Email is **not** one of the section's 4 links; it is found elsewhere in the application:

- **On mobile**: open the navigation drawer (menu), scroll to the footer; tap **info@constructoai.ca** (`Sidebar.tsx:385`). The email application opens.
- **Before sign-in**: the address `info@constructoai.ca` also appears on the login and registration pages (`LoginPage.tsx:173`, `RegisterPage.tsx:113`, `B2bLoginPage.tsx:76`, `B2bRegisterPage.tsx:156`).
- **On desktop, after signing in**: the address does not appear in the sidebar. Copy it from this manual or from the mobile footer: **info@constructoai.ca**.

### 3.5 Open the useful links (Quebec organizations)

1. "HELP & RESOURCES" section → **Useful links**.
2. A new tab opens on the GitHub references page (RBQ, CCQ, CNESST, Revenu Québec, grants, standards, etc.).
3. Click the link for the organization you want, which then opens in turn.

### 3.6 Collapse or expand the section

1. Click the "HELP & RESOURCES" header.
2. The chevron rotates and the list of 4 links hides (collapsed) or reappears (expanded).
3. This setting applies to the current session; a full page reload returns the section to the "expanded" state.

### 3.7 Find help when the sidebar is in icon mode

1. If the sidebar is collapsed (icons only), the "HELP & RESOURCES" header is hidden, but the 4 icons stay visible.
2. Hover over an icon to show its tooltip (for example "Phone 1 514 820-1972").
3. Click the icon you want — the behavior (new tab or dialer) is identical to expanded mode.

### 3.8 Get AI-assisted help in the ERP (AI Assistant)

1. In the **top bar**, click the **sparkles** icon (AI Assistant), to the left of the search.
2. The `/assistant-ia` page opens inside the ERP.
3. Ask your question in natural language. The Assistant can consult your data (projects, invoices, clients) and propose actions.

> This is a **separate module**, billed in AI credits. It is not an official channel for reporting a bug: for that, use support email or phone.

### 3.9 Choose the right channel for your need

| Need | Recommended channel |
|---|---|
| Learn how to use a module | **Videos** or **Manual** |
| Find an official reference (RBQ, CCQ, CNESST, taxes) | **Useful links** |
| Quick question about my data in the ERP | **AI Assistant** (top bar) |
| Technical problem, bug, billing, integration | **Email** info@constructoai.ca or **Phone** 1 514 820-1972 |
| Discovery / questions before subscribing | Public "Sylvain" chatbot on the login page (see 5.2) |

---

## 4. Reference

### 4.1 Server endpoints

**None.** The section calls no FastAPI endpoint. There is no `aide`, `help`, `support`, `contact`, or `feedback` router in `ERP_REACT/backend/`. Each click is handled **entirely in the browser**: opening an external tab or triggering the dialer (`tel:`).

| Item | Status |
|---|---|
| Endpoints (method + path) | None |
| Mount prefix | Not applicable |
| PostgreSQL tables / columns | None |
| Validations / bounds / quotas | Not applicable |
| Required roles / guards (`is_admin`, `super_admin`, consultation mode) | None — visible to every signed-in user |
| Integrations (Stripe, QuickBooks, Vapi, AI) | None in the section |
| Money effects (billing, credits) | None |
| Workflows / statuses / calculations | None |

### 4.2 The 4 links (exact definition)

Source: `Sidebar.tsx:259-262`.

| # | Label FR / EN | i18n key | `href` | `external` | Icon (lucide) | Trailing |
|---|---|---|---|---|---|---|
| 1 | Vidéos / Videos | `nav.helpResources.videos` | `https://www.youtube.com/channel/UC3EGXYQNj5UYGiyNfiiom_A` | `true` | `Video` (18) | external glyph |
| 2 | Manuel / Manual | `nav.helpResources.manuel` | `https://github.com/ConstructoAI/Documents/blob/main/README.md` | `true` | `BookOpen` (18) | external glyph |
| 3 | Téléphone / Phone | `nav.helpResources.telephone` | `tel:+15148201972` | `false` | `Phone` (18) | text `1 514 820-1972` |
| 4 | Liens utiles / Useful links | `nav.helpResources.usefulLinks` | `https://github.com/ConstructoAI/Documents/blob/main/liens-utiles.md` | `true` | `ExternalLink` (18) | external glyph |

### 4.3 Translations (i18n)

Source: `fr/nav.json:41-47` and `en/nav.json:41-47`. Full FR/EN parity (the title + 4 labels). No server-side strings.

| Key | French | English |
|---|---|---|
| `nav.aideRessources` | AIDE & RESSOURCES | HELP & RESOURCES |
| `nav.helpResources.videos` | Vidéos | Videos |
| `nav.helpResources.manuel` | Manuel | Manual |
| `nav.helpResources.telephone` | Téléphone | Phone |
| `nav.helpResources.usefulLinks` | Liens utiles | Useful links |

### 4.4 Behavior by display mode

| Aspect | Expanded sidebar (desktop) | Collapsed sidebar (desktop) | Mobile drawer |
|---|---|---|---|
| "HELP & RESOURCES" header | Visible, collapsible | Hidden | Visible, collapsible |
| Links | Icon + label + trailing | Icon only + tooltip | Icon + label (44 px target) |
| Footer with email + version | No | No | **Yes** (`Sidebar.tsx:382-389`) |
| Width | `lg:w-56` | `lg:w-14` | `85vw` (max 280 px) |

### 4.5 Support contact details and channels

| Channel | Value | Where to find it | ERP server involved? |
|---|---|---|---|
| Videos (YouTube) | Official channel | Link 1 of the section | No (external) |
| Manual (GitHub) | `ConstructoAI/Documents` README | Link 2 of the section | No (public GitHub) |
| Useful links (GitHub) | `ConstructoAI/Documents` liens-utiles.md | Link 4 of the section | No |
| Phone | **1 514 820-1972** (`tel:+15148201972`) | Link 3 of the section + mobile footer | No (system dialer) |
| Email | **info@constructoai.ca** | Mobile footer + login/registration pages | No (email application) |
| AI Assistant (in-app) | `/assistant-ia` | Sparkles button in the top bar | Yes — **separate module** (`routers/ai.py`) |
| "Sylvain" chatbot (before sign-in) | Public login page | `app.constructoai.ca` | Yes — **separate module** (`routers/voice.py`) |

> **Number format**: same number everywhere (`+15148201972`), but shown differently — the section link shows `1 514 820-1972` (`Sidebar.tsx:261`), while the mobile footer and the public chatbot display `1 (514) 820-1972` (`Sidebar.tsx:387`). A simple presentation difference.

### 4.6 Configurability

| Setting | Value | User-editable? |
|---|---|---|
| URLs of the 4 links | Hard-coded (`Sidebar.tsx:259-262`) | No |
| Phone number | `+15148201972` (hard-coded) | No |
| Email address | `info@constructoai.ca` (hard-coded) | No |
| Labels | `nav.helpResources.*` translations | No (requires updating the language files) |
| Order / number of links | 4, fixed | No |

> Changing a link or a label = editing the code (`Sidebar.tsx` or the `nav.json` files), rebuilding the interface, then redeploying. **No administration interface** lets you change these values.

### 4.7 Security

- **Anti-tab-hijacking**: `rel="noopener noreferrer"` is applied to the 3 external links (`Sidebar.tsx:270`) — the target page cannot access the originating window.
- **No tracking**: the click triggers no analytics event on the ERP side.
- **No error, loading, or retry state**: the links are static. The only possible "failure" is browser-side (external site unavailable, or no telephony / email application present).

### 4.8 What does not exist (cheat sheet)

| Feature | Present? |
|---|---|
| Internal `/aide` route or help page in the ERP | No |
| Built-in FAQ / searchable knowledge base | No |
| In-app manual viewer | No (the manual is on GitHub) |
| Contact form / ticketing system | No |
| Live chat in the shell | No |
| "?" button in the top bar | No |
| Centralized contextual help tooltips / guided tour | No |
| "Email" link among the section's 4 links | No (email is in the mobile footer) |
| Email in the **desktop** sidebar | No (mobile only) |
| Role-based filtering of the links | No (everyone sees them) |

---

## 5. Integrations and FAQ

### 5.1 Relationship to the AI Assistant

The AI Assistant is the only **in-app assisted** help in the ERP, but it is not part of this section: it is a standalone module (server `routers/ai.py`), reachable via the sparkles button in the top bar (`TopBar.tsx:312-320`). It consumes **AI credits** and can consult your data. Details: `24-communication-assistant-ia.md`.

### 5.2 Relationship to the public "Sylvain" chatbot

Before sign-in, the public login page (`app.constructoai.ca`) offers a conversational assistant — "Sylvain" (system prompt `backend/sylvain_prompt.py`, server `routers/voice.py`, prefixes `/voice` and `/api/voice`). Its role is to guide visitors and to **redirect technical or integration questions** to email **info@constructoai.ca** or phone **1 (514) 820-1972**, and to the YouTube **videos** for discovery. It **normalizes** the number's display (`+1 514-820-1972` → `15148201972`, `voice.py:356`). This chatbot is **separate** from the Help and resources section and is not accessible from the signed-in shell.

### 5.3 The `ConstructoAI/Documents` GitHub repository

The **Manual** and **Useful links** links both point to the **public** repository `github.com/ConstructoAI/Documents` — the same repository that hosts this manual. Practical consequences:

- **Public**: anyone with the link can read these pages, without an account.
- **External**: the content lives outside the ERP; it requires the Internet and cannot be verified from the application code.
- **Evolving**: the documentation can be updated independently of ERP releases.
- To report an error in the documentation: email **info@constructoai.ca** (or open an issue on the repository).

### 5.4 What is NOT possible

- **No internal help page**: no `/aide` route, no built-in FAQ, no documentation search, no in-app manual viewer.
- **No ticketing system, contact form, or live chat**: human support comes down to **email** and **phone**.
- **No "?" button**, no guided tour, no centralized contextual help tooltip in this section.
- **No configuration through the interface**: the 4 URLs, the number, and the email are hard-coded; changing them requires a code change and a redeployment.
- **No role-based filtering**: an unprivileged employee sees exactly the same resources as an administrator.
- **No "Email" link** among the 4 links; on desktop, email appears nowhere in the sidebar (mobile and login pages only).

### 5.5 FAQ

**Is "Help and resources" a module like the others?**
No. It is a static block of 4 links in the sidebar (`Sidebar.tsx:244-297`). It has no internal page, no server, and no database.

**Why do the links open a new tab?**
To preserve the ERP's state (a form in progress, navigation) while you consult the external resource. The Phone link, for its part, only opens the dialer (no tab).

**Where is the manual: in the application or online?**
Online. The "Manual" link opens the README of the `ConstructoAI/Documents` GitHub repository. There is no built-in viewer; an Internet connection is required.

**The "Manual" link seems shorter than the other resources. Is that normal?**
Yes. The link points to the **README** (the documentation index), not to a specific module. From this index, you open the detailed manual for the module you want.

**How do I reach a human at support?**
By **email** (info@constructoai.ca) or by **phone** (1 514 820-1972). There is no live chat and no ticketing system.

**I don't see the support email on my computer.**
On desktop, email is not in the sidebar: it appears in the **mobile drawer footer** and on the **login pages**. Address: info@constructoai.ca. The only contact present in the desktop sidebar is the **Phone** link.

**The number is displayed differently in two places — is it the same one?**
Yes. `1 514 820-1972` (section link) and `1 (514) 820-1972` (mobile footer) both dial `+15148201972`. A simple presentation difference.

**Does an employee see fewer resources than an administrator?**
No. The 4 links are subject to no role filtering: every signed-in user sees exactly the same thing.

**Can I change these links from the administration interface?**
No. They are hard-coded in `Sidebar.tsx`. Changing them requires a code change, a rebuild, and a redeployment.

**Is there contextual help (tooltips) on each field?**
No centralized system here. Some screens have occasional tooltips, but there is no universal per-field "?" button.

**Is the AI Assistant the support channel for bugs?**
No. The AI Assistant (sparkles button in the top bar) answers business questions about your data, but for a bug or a technical/billing question, go through email or phone.

**Where is the AI Assistant? I don't see it in the left menu.**
It is **not** in the sidebar. Its only entry point is the **sparkles** icon in the top bar, to the left of the search.

**Will this section grow over time?**
Any evolution would be documented in a future version of this manual. As it stands, it is a block of 4 static links.

---

## 6. Summary

- **Scope**: "Help and resources" (`nav.aideRessources` = "HELP & RESOURCES") is **not a functional module** — it is a **static block of 4 links** at the bottom of the sidebar (`Sidebar.tsx:244-297`). No internal route, no server, no database.
- **No backend**: 0 FastAPI endpoints, 0 PostgreSQL tables, 0 role guards, 0 money effects, 0 network calls to the ERP.
- **The 4 links**: **Videos** (YouTube), **Manual** (GitHub README), **Phone** (`tel:+15148201972`, displayed as 1 514 820-1972), **Useful links** (GitHub liens-utiles.md). Three are external (`target="_blank"` + `rel="noopener noreferrer"`); Phone opens the dialer (`external:false`).
- **Always visible**: the section is not filtered by role — administrator and employee alike see the same resources.
- **Collapsible header**, expanded by default; the collapse state is a session preference, reset on a full reload.
- **Display modes**: expanded sidebar (icon + label); collapsed sidebar (icons + tooltips, header hidden); mobile (44 px targets + **footer with email info@constructoai.ca, phone 1 (514) 820-1972, and version v1.0**).
- **Support email**: present **in the mobile footer** and on the **login pages** — **not** in the desktop sidebar, and **not** among the 4 links.
- **In-ERP assisted help**: the **AI Assistant** button (sparkles) in the top bar (`TopBar.tsx:312-320`) — a **separate module**, the only entry point to `/assistant-ia`.
- **Public "Sylvain" chatbot** (before sign-in) guides visitors and redirects technical questions to email / phone — outside the section.
- **Does not exist**: internal help page, built-in FAQ, documentation search, contact form, ticketing system, live chat, "?" button, guided tour, configuration through the interface.
- **Not configurable**: the 4 URLs, the number, and the email are hard-coded; changing them = edit the code + rebuild + redeploy.

---

**Verified sources**: `frontend/src/components/layout/Sidebar.tsx` ("Help & Resources" block 244-297: divider 245, collapsible header 246-257, table of the 4 links 259-262, rendering 263-297; `canSeeItem` role filter 144-152, from which the section is excluded; mobile footer 382-389; desktop footer 344-353), `frontend/src/components/layout/TopBar.tsx:312-320` (AI Assistant button), `frontend/src/i18n/locales/fr/nav.json:41-47` and `en/nav.json:41-47` (translations), `backend/` (no `aide`/`help`/`support`/`contact`/`feedback` router — verified), `backend/sylvain_prompt.py` + `backend/routers/voice.py` (public "Sylvain" chatbot, prefixes `/voice` and `/api/voice`, number normalization `voice.py:356`), `frontend/src/pages/LoginPage.tsx:173`, `RegisterPage.tsx:113`, `B2bLoginPage.tsx:76`, `B2bRegisterPage.tsx:156` (pre-sign-in support email).

**Related manuals**:
- AI Assistant (in-app assisted help, AI credits) — `24-communication-assistant-ia.md`
- Voice agent (telephony, voice chatbot) — `23-communication-agent-vocal.md`
- Configuration (subscription, payment method, preferences) — `30-configuration.md`
- Web (real-time search + Quebec useful links) — `29-outils-web.md`

---

*Manual 29 — Help and resources — Constructo AI ERP — revision 3.0, July 2026*
