# Module 29 — Web (real-time search)

> **Version**: 3.0 (rebuilt and verified against the source code, July 2026)
> **Reference code**: `backend/routers/web.py` (572 lines, **4 endpoints**, actual prefix `/api/erp/v1/web`, tag "Recherche Web"), `frontend/src/pages/WebPage.tsx` (736 lines, **5 tabs** — single page, no `components/web` folder), `frontend/src/store/useWebStore.ts`, `frontend/src/api/web.ts`, `frontend/src/i18n/locales/{fr,en}/web.json`
> **PostgreSQL tables**: `web_search_history` (in each tenant's schema, **created on demand** on first use — no official migration). Billing via the shared tables `public.ai_prepaid_credits` (balance), `public.ai_credit_ledger` (idempotency), and `public.ai_usage_tracking` (tracking).
> **Scope**: a **real-time web search** and **page analysis** interface powered exclusively by **Claude's server-side tools (Anthropic)** — `web_search_20260209` and `web_fetch_20260209`. There is **NO** direct integration with Google, Bing, SerpAPI, Brave, or DuckDuckGo: all searching runs on Anthropic's side. The module uses the **Claude Opus** model (`claude-opus-4-8`), the most powerful and most expensive in the ERP, and debits the tenant's **prepaid AI credits**. To query your **internal data** (projects, invoices, clients), this is not the module: use the **AI Assistant** and its `recherche_bd` tool.

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

### 1.1 Module mission

Give the contractor and their employees **live access to information from the web**, without leaving the ERP, with written answers and **cited sources**:

- **Web search** — ask a question in natural language; Claude queries the web and writes a structured answer with the list of sources.
- **Page analysis** — provide the address (URL) of a web page or PDF; Claude retrieves its content, summarizes it, and draws out the key points.
- **Search + analysis** — Claude first looks for sources, keeps the 1 or 2 most relevant, then analyzes them in depth before synthesizing.
- **History** — the list of **your** recent searches, kept per tenant.
- **Useful Links** — 8 shortcuts to Quebec's official construction bodies — CCQ (construction industry commission), RBQ (building authority), CNESST (workplace health, safety and labour-standards board), etc.

It is a tool for **monitoring and documentary research** (new standards, market prices, regulations, product spec sheets). It is **not** a search engine for your own data, nor an extractor for files you upload.

### 1.2 What the module does — and does not do

| The module **does** | The module **does not** |
|---|---|
| Search the public web and cite sources | Search your projects/invoices/clients (see the AI Assistant) |
| Read a page or PDF **accessible online** by its URL | Analyze a file you **upload** from your computer |
| Synthesize several sources | Export the answer to PDF or CSV, or print it via a button |
| Keep a read-only history | Re-run, edit, or delete a history entry |
| Filter by allowed/blocked domains | Guarantee the accuracy of sources (to be validated by you) |

### 1.3 The provider: Anthropic only (no Google or Bing)

> **Important.** This module **relies on no traditional search engine** (Google Custom Search, Bing, SerpAPI, Brave, DuckDuckGo). It uses **exclusively Claude's two native web tools**, run on Anthropic's servers:
> - `web_search_20260209` — real-time web search and citations (`web.py:236`, `web.py:431`);
> - `web_fetch_20260209` — retrieval and analysis of a URL's content (`web.py:338`, `web.py:443`).

The access key used is `ANTHROPIC_API_KEY` (environment variable, shared with the AI Assistant). **No fallback** exists: if the Anthropic service is unavailable or if the key is missing, the module returns **503 "AI service unavailable"** (`web.py:217`), with no other engine to take over.

The ERP server **never visits** third-party sites itself: it is Anthropic that runs the search and loads the pages. This is a useful point for compliance (no outbound requests from your servers to the analyzed sites).

### 1.4 The model and its cost

Source: `web.py:32-33`, `web.py:70-88`.

| Parameter | Actual value |
|---|---|
| Model | **`claude-opus-4-8`** (Opus — the premium model) |
| Response `max_tokens` | **32,000** |
| Call | In **streaming**, offloaded off the shared event loop so it does not slow down the ERP (`web.py:91-101`, `asyncio.to_thread`) |
| Input rate | US$5 per million tokens |
| Output rate | US$25 per million tokens |
| Cache-write rate | US$6.25 per million tokens |
| Cache-read rate | US$0.50 per million tokens |
| Internal markup | **× 1.30** (30% margin) |

> The Web module uses **Opus, not Sonnet** (the ERP default for the AI Assistant is `claude-sonnet-4-6`). This is a deliberate choice for synthesis quality on open-ended searches — but it makes it **the most expensive AI function** on the platform. A former overcharge ("legacy" Opus rates of US$15/US$75 per million, i.e. ×3) has been fixed; the rates above are the exact ones for `claude-opus-4-8` (`web.py:70-72`).

**There is no temperature set** in the call (contrary to what an earlier version of this manual stated): the model runs at its default value. The exact cost formula is given in [4.4](#44-model-rates-and-cost-formula).

### 1.5 Geolocation locked to Quebec

All searches (the **Web Search** and **Search + Analysis** tabs) pass Claude a **hard-coded** user location: Montreal / Quebec / CA / `America/Montreal` time zone (`web.py:239-245` and `web.py:434-440`). This steers results toward local, French-language sources. This setting **cannot be changed** from the interface. The **Page Analysis** tab does not use it (it targets a specific URL you provide).

### 1.6 Access, permissions, and read-only mode

- **Where to find it**: sidebar, **Tools** section → **Web** (`Globe` icon), path `/web` (`Sidebar.tsx:98-103`, `App.tsx:126`). The default open tab is **Web Search**.
- **Who can use it**: **any authenticated user** of the tenant. All four endpoints are protected by the session check alone (`Depends(get_current_user)`); **no particular role is required** (neither administrator nor accountant). There is no restriction by trade.
- **The real safeguard is credit.** On every paid search, the server checks the prepaid AI credit balance; if the balance is exhausted, it returns **402 "Insufficient AI credits"** (`web.py:228-230`). (A second internal check, `check_ai_guard`, exists but **blocks no one** in the current configuration — it lets any authenticated user through; the effective block comes solely from credit.)
- **Read-only mode.** A tenant without an active Stripe subscription (cancelled subscription, no card on file) switches to **read-only mode** across the whole ERP. In this state, the **three search functions** (which are write requests) are **blocked with 403 "Read-only mode"**, while the **History** tab (read) and the **Useful Links** tab remain accessible.
- **Internal instance.** If billing is disabled at the server level (`ERP_BILLING_ENABLED=false`), there is **neither credit check nor debit**: search is free. Platform super-administrators are also exempt from debit.

### 1.7 The 5 tabs

Source: the `TABS` array (`WebPage.tsx:261-267`), ARIA tab bar (`WebPage.tsx:374-402`).

| # | Tab | Icon | Nature | Endpoint |
|---|---|---|---|---|
| 1 | **Web Search** | `Search` | Paid AI function | `POST /web/search` |
| 2 | **Page Analysis** | `FileText` | Paid AI function | `POST /web/fetch` |
| 3 | **Search + Analysis** | `Zap` | Paid AI function | `POST /web/search-fetch` |
| 4 | **History** | `History` | Read-only | `GET /web/history` |
| 5 | **Useful Links** | `ExternalLink` | Static (no server) | — |

Three tabs consume credits, one shows the history, and a last one is just a list of shortcuts.

---

## 2. Interface

### 2.1 General layout

At the top of the page: the title **"Web - Search and Analysis"** with the blue `Globe` icon (`WebPage.tsx:360-363`). Just below, an **error banner** appears if the last action failed; it is **manually dismissible** and does not disappear on its own (see [2.9](#29-error-messages-displayed)). Then the **bar of 5 tabs**. The content changes with the active tab; the default open tab is **Web Search**.

### 2.2 "Web Search" tab

Source: `WebPage.tsx:405-475`.

| Element | Detail |
|---|---|
| **Info box** (blue) | "**Real-time web search** - Claude searches the internet and provides an answer with cited sources." |
| **Input field** | A 3-line text area. Displayed example: "e.g., What are the latest innovations in modular construction in Quebec?". |
| **"Max searches" slider** | Range **1 to 10, default 5**. This is the maximum number of web searches Claude may launch to answer. Higher = more thorough, but slower and more expensive. The chosen value is shown on the right. |
| **Domain filtering** | Shared component (see [2.8](#28-the-domain-filter-shared-component)): three modes **None / Allow / Block** + a domains field. |
| **"Search" button** | Launches the search. While processing, it becomes "Searching..." with a spinner, and a hint appears: "Web search in progress, this may take a few seconds...". |
| **"Clear" button** | Appears **only** once a result is obtained; it clears the displayed result. |
| **Result** | Common result card (see [2.5](#25-the-common-result-card)), titled "Search result". |

An **anti-double-click lock** prevents billing the same search twice if you click quickly (`WebPage.tsx:317-328`).

### 2.3 "Page Analysis" tab

Source: `WebPage.tsx:478-563`.

| Element | Detail |
|---|---|
| **Info box** (blue) | "**Web page or PDF analysis** - Claude retrieves and analyzes in detail the content of a specific URL." |
| **URL field** | A URL-type field with a link icon. Example: "https://example.com/page-to-analyze". If you enter an address **without** `http://` or `https://` (e.g., `rbq.gouv.qc.ca`), the **`https://` prefix is added automatically** (`WebPage.tsx:335`). |
| **"Max tokens" slider** | Range **10,000 to 200,000, in steps of 10,000, default 100,000**; shown in "K". This is the cap on page content that Claude ingests. A large page at the cap costs more. |
| **"Citations" checkbox** | **Checked by default.** Asks Claude to cite passages precisely. |
| **Domain filtering** | Same as the Search tab (None / Allow / Block). |
| **"Analyze" button** | Launches the analysis; becomes "Analyzing..." with the hint "Retrieving and analyzing the page...". |
| **"Clear" button** | Clears the displayed result. |
| **Result** | Common card, titled "Page analysis". |

The server requires Claude to produce a **4-point structured analysis** (`web.py:349-359`): **1. Summary** — **2. Key points** — **3. Context and implications** — **4. Recommendations or conclusions**.

> This tab's maximum number of fetches is **locked to 5** on the app side (`WebPage.tsx:338`) and is **not** user-adjustable.

### 2.4 "Search + Analysis" tab

Source: `WebPage.tsx:566-662`.

| Element | Detail |
|---|---|
| **Info box** (blue) | "**Web search + in-depth analysis** - Claude first searches for relevant information, then analyzes in detail the best sources found." |
| **Warning** (amber) | "This function uses more resources because it combines web search AND detailed source analysis. Longer response time (30-60 seconds)." |
| **Input field** | A 3-line text area. Example: "e.g., Detailed analysis of seismic construction standards in Quebec in 2025". |
| **"Max searches" slider** | Range **1 to 5, default 3**. |
| **"Max analyses" slider** | Range **1 to 5, default 2**. |
| **Allowed domains** | A **single field** "Allowed domains (optional)". **No** None/Allow/Block toggle here: this tab supports **only** "allow" filtering (endpoint limitation, `web.py:450-452`). |
| **"Search and Analyze" button** | Launches processing; becomes "In-depth search in progress..." with the hint "In-depth search in progress, this may take 30-60 seconds...". |
| **"Clear" button** | Clears the displayed result. |
| **Result** | Common card, titled "In-depth analysis". |

The server chains the two tools (search **then** analysis) and gives Claude a **4-step process** (`web.py:454-465`): search → identify the 1-2 best sources → retrieve their full content → synthesize. In this mode, the content loaded per page is **capped at 50,000 tokens** (hard-coded, `web.py:447`) — lower than the Page Analysis tab's cap (up to 200,000), because two expensive operations are combined here.

### 2.5 The common result card

Source: `ResultCard` (`WebPage.tsx:102-162`). The three paid tabs display the same result block.

**Statistics bar** (small grey text), shown according to what happened:
- **"{n} search(es)"** — number of web searches performed (if greater than 0);
- **"{n} analysis(es)"** — number of pages analyzed (if greater than 0);
- **"{s}s"** — call duration, in seconds (clock icon);
- **"{n} tokens"** — total input + output tokens;
- **"{x.xxxx} USD"** — the **actually billed cost** of this call, to **4 decimals**.

**Text block**: Claude's answer, displayed as-is (line breaks are preserved, but there is **no** rich Markdown formatting). If Claude returned nothing textual: "No textual result."

**"Sources" block** (blue box): title "Sources ({n})" then the list of cited links, each opening the source in a **new browser tab** (external-link icon). Duplicate URLs are removed server-side (`web.py:133`, `web.py:145`).

> There is **no** "Copy", "Share", "Save", or "Print" button on the result card. To keep an answer, select the text and copy it (see the FAQ).

### 2.6 "History" tab

Source: `WebPage.tsx:665-718`, server `web.py:514-571`.

- **Automatic loading** as soon as you open the tab; it requests the **50 most recent entries** (`WebPage.tsx:299-303`).
- **Subtitle**: "History of your recent web searches." + a **"Refresh"** button to reload.
- **If empty**: history icon + "No searches in history." + "Your searches will appear here after use."
- **Each entry** (card) shows:
  - a coloured **type badge**: **"Search"** (blue), **"Analysis"** (green), or **"Search + Analysis"** (purple);
  - the localized **date** (`fr-CA` format);
  - **"{n} source(s)"** if sources were cited;
  - the **query** (or URL) in bold;
  - a **preview** of the result (2 lines maximum).

> **History is read-only.** There is **no** "Re-run" button, **no** "View", **no** deletion, **no** export. To redo a search, copy the query into the appropriate tab.
>
> **History is personal.** Each user sees **only their own** searches (filtered by user ID, `web.py:539`) — it is **not** shared among colleagues in the same tenant. It is also bounded to **50** displayed entries.

### 2.7 "Useful Links" tab

Source: `LIENS_UTILES` (`WebPage.tsx:38-95`), text in `web.json`.

Intro: "Government bodies, online tools, and essential resources for Quebec's construction sector." Then a grid of **8 cards** (title + description + coloured icon), each opening the site in a **new tab**:

| # | Organization | Address |
|---|---|---|
| 1 | Commission de la construction du Québec (CCQ) | `https://www.ccq.org` |
| 2 | Régie du bâtiment du Québec (RBQ) | `https://www.rbq.gouv.qc.ca` |
| 3 | CNESST | `https://www.cnesst.gouv.qc.ca` |
| 4 | Revenu Québec | `https://www.revenuquebec.ca` |
| 5 | Quebec Construction Code | `https://www.rbq.gouv.qc.ca/lois-reglements-et-codes` |
| 6 | Registre des entreprises du Québec (REQ) | `https://www.registreentreprises.gouv.qc.ca` |
| 7 | RBQ Licence Checker | `https://www.rbq.gouv.qc.ca/services-en-ligne/licence/registre-des-detenteurs-de-licence` |
| 8 | Plan Québec | `https://www.quebec.ca/gouvernement/politiques-orientations/plan-quebecois-infrastructures` |

This list is **fixed**; it is not configurable from the interface and consumes no credit.

### 2.8 The domain filter (shared component)

Source: the `DomainFilter` component (`WebPage.tsx:194-255`).

Present on **Web Search** and **Page Analysis**. Title "**Domain filtering**", three pill buttons:

- **"None"** — no restriction (default);
- **"Allow"** (green) — Claude **only consults** the listed domains;
- **"Block"** (red) — Claude **excludes** the listed domains.

As soon as you choose Allow or Block, a field appears, with the example "e.g., quebec.ca, canada.ca, rbq.gouv.qc.ca". Separate domains with **commas**.

Rules to know:
- **Allow and Block are mutually exclusive**: you cannot combine the two (`web.py:248-251`).
- You can enter up to **20** domains, but the server keeps only the **first 10** (`web.py:249`, `web.py:251`).
- The **Search + Analysis** tab only offers "Allowed domains" (no Block, no None) — see [2.4](#24-search--analysis-tab).

### 2.9 Error messages displayed

Source: `useWebStore.ts:35-56`, `web.json:134-137`.

On failure, a red banner appears at the top of the page. The message comes first from the detail returned by the server (e.g., "Insufficient AI credits"). Two generic messages exist on the app side:

- **Network / timeout** (502-504 errors, network drop, no response): "The search timed out or the connection was interrupted. Try again, ideally with fewer searches or analyses."
- **Unknown error** (fallback): "An error occurred. Please try again."

The banner **stays displayed** until you close it (the X); launching a new search clears it automatically (`WebPage.tsx:305-307`).

---

## 3. Step-by-step workflows

### 3.1 Run a simple web search

1. Sidebar → **Tools** → **Web** → **Web Search** tab.
2. Enter the question (e.g., "New RBQ requirements 2026 for residential building permits").
3. (Optional) Adjust the **Max searches** slider (1 to 10). Higher = more thorough answer, but more expensive.
4. (Optional) Enable a **domain filter**: **Allow** to limit to certain sources, **Block** to exclude some.
5. Click **Search**.
6. Wait a few seconds. Behind the scenes: the server checks the credit, prepares the `web_search` tool with the Quebec location, calls Claude Opus in streaming mode, extracts the text, sources, and counters, computes the cost, debits the credit, logs the usage (`web_search` feature), and records the history entry (type "search").
7. Read the answer, check the cited **sources** (they open in a new tab), note the **cost in USD** displayed.

### 3.2 Analyze a web page or PDF

1. **Page Analysis** tab.
2. Paste the **full URL** (e.g., `https://www.cnesst.gouv.qc.ca/.../guide-EPI.pdf`). Without a scheme, `https://` is added for you.
3. (Optional) Set **Max tokens** (10,000 to 200,000), uncheck **Citations**, or filter by domains.
4. Click **Analyze**.
5. Behind the scenes: the server validates the URL (it must start with `http://` or `https://`), prepares the `web_fetch` tool (citations enabled, content cap), requests the 4-point analysis, debits the credit, records the history (type "fetch").
6. Read the structured analysis (Summary / Key points / Context / Recommendations).

> Useful for: dissecting an online supplier quote, summarizing a long regulatory guide in PDF, extracting the essentials of a product spec sheet.

### 3.3 Combined search + analysis

1. **Search + Analysis** tab.
2. Enter an investigation question (e.g., "Detailed analysis of seismic standards in Quebec in 2026").
3. Set **Max searches** (1 to 5, default 3) and **Max analyses** (1 to 5, default 2).
4. (Optional) Enter **allowed domains** ("allow" mode only).
5. Click **Search and Analyze**. Allow **30 to 60 seconds**.
6. Behind the scenes: Claude searches, keeps the best sources, retrieves their content (capped at 50,000 tokens per page), then synthesizes. Credit debit, history (type "search_fetch").
7. Read the in-depth synthesis and its sources.

> This is the **most expensive** mode: reserve it for questions that truly require reading sources in full. For a quick answer, the **Web Search** tab is often enough.

### 3.4 Restrict or exclude domains

**Limit to Quebec government sources** (Web Search or Page Analysis tab): **Allow** mode → enter, for example, `quebec.ca, gouv.qc.ca, ccq.org, rbq.gouv.qc.ca, cnesst.gouv.qc.ca`. Reminder: 10 effective domains maximum.

**Exclude sources deemed unreliable**: **Block** mode → enter, for example, `reddit.com, quora.com`.

**On Search + Analysis**: only the "Allowed domains" field is available; there is no "block" mode.

### 3.5 View the history

**History** tab → the list loads on its own (50 most recent). **Refresh** button to reload. To redo a search, **copy** the question into the desired tab: there is no one-click re-run. Reminder: you see only **your** searches, not your colleagues'.

### 3.6 Access an official body

**Useful Links** tab → click a card (CCQ, RBQ, CNESST, Revenu Québec, REQ, etc.) → the site opens in a new tab. Tip: copy a specific URL from the official site then paste it into the **Page Analysis** tab to have its content summarized.

### 3.7 Track and control costs

The Web module does **not** have its own cost dashboard. Three habits:

1. **Look at the displayed cost** on each result card ("{x.xxxx} USD") — it is the exact amount debited.
2. **Choose the right tab**: Web Search (the cheapest) for a quick answer; reserve Search + Analysis for cases that require it.
3. **Lower the sliders** (Max searches, Max analyses, Max tokens) when you do not need exhaustiveness.

For aggregate tracking of AI consumption (across all functions), refer to the **AI Assistant** module: the `web_search`, `web_fetch`, and `web_search_fetch` features appear there in the usage tracking.

### 3.8 What to do if "Insufficient AI credits" or "Read-only mode"

- **"Insufficient AI credits" (402)**: the tenant's prepaid credit balance is exhausted. An **automatic top-up** of US$10 via Stripe normally triggers when the balance drops below US$0.10; if it did not happen (card missing or declined), the tenant administrator must fix the payment method. Then try again.
- **"Read-only mode" (403)**: the tenant is read-only (cancelled subscription or no card). Searches are blocked; **History** and **Useful Links** remain viewable. The subscription must be reactivated to regain the paid functions.
- **"AI service unavailable" (503)**: the Anthropic service is unreachable. Try again later; there is no fallback engine.
- **Timeout / network message**: try again, ideally with **fewer** searches or analyses (lower sliders).

---

## 4. Reference

### 4.1 Web router endpoints

Full prefix: `/api/erp/v1/web` (`erp_api.py:1047`, `web.py:20`). All protected by the session check (`get_current_user`), with no other role guard.

| Method | Path | Tracked feature | Purpose |
|---|---|---|---|
| POST | `/web/search` | `web_search` | Real-time web search (`web.py:212`) |
| POST | `/web/fetch` | `web_fetch` | Retrieves and analyzes a URL (`web.py:312`) |
| POST | `/web/search-fetch` | `web_search_fetch` | Searches then analyzes the best sources (`web.py:407`) |
| GET | `/web/history` | — (read) | Current **user's** history (`web.py:514`) |

### 4.2 Request bodies (Pydantic validation)

Source: `web.py:43-64`.

| Request | Fields and bounds |
|---|---|
| **WebSearchRequest** | `query` (required, ≤ 4000 characters); `max_uses` (1-10, default 5); `allowed_domains` / `blocked_domains` (≤ 20 client-side, **truncated to 10** server-side; mutually exclusive) |
| **WebFetchRequest** | `url` (required, ≤ 2048 characters, must start with `http(s)://`); `max_uses` (1-10, but **locked to 5** by the UI); `enable_citations` (default **true**); `max_content_tokens` (1000-200000, default 100000); `allowed_domains` / `blocked_domains` (≤ 20 → 10) |
| **WebSearchFetchRequest** | `query` (required, ≤ 4000); `max_search_uses` (1-5, default 3); `max_fetch_uses` (1-5, default 2); `allowed_domains` **only** (≤ 20 → 10) |
| **GET /web/history** | `limit` (1-100, default 20) — the UI requests 50 |

### 4.3 Response format (the three POSTs)

The keys are converted to `camelCase` by the interceptor (`api/web.ts:15-25`).

```json
{
  "text": "Answer written by Claude, with its sections.",
  "citations": [{ "title": "...", "url": "https://..." }],
  "search_count": 3,
  "fetch_count": 1,
  "input_tokens": 14832,
  "output_tokens": 2571,
  "cost_usd": 0.1783,
  "elapsed_seconds": 22.4,
  "credit_balance": 9.4217
}
```

`credit_balance` is the estimated balance **after** this call. `GET /web/history` returns `{ "items": [ { id, user_id, search_type, query, result_preview, citations_count, created_at } ] }`.

### 4.4 Model, rates, and cost formula

Source: `web.py:32-33`, `web.py:80-88`.

| Parameter | Value |
|---|---|
| `WEB_AI_MODEL` | `claude-opus-4-8` |
| `WEB_AI_MAX_TOKENS` | `32000` |
| Temperature | None (model default) |
| Input rate | US$5 per million tokens |
| Output rate | US$25 per million tokens |
| Cache write | US$6.25 per million tokens |
| Cache read | US$0.50 per million tokens |
| Markup | × 1.30 |

**Formula** (`_compute_web_cost`, `web.py:80-88`):

```
cost_USD = ( input       × 5    / 1,000,000
           + output      × 25   / 1,000,000
           + cache_write × 6.25 / 1,000,000
           + cache_read  × 0.50 / 1,000,000 ) × 1.30
```

**Indicative orders of magnitude** (the real cost depends mainly on how much web content is ingested; rely on the amount shown on the card):

| Action | Approx. tokens (input / output) | Approx. cost |
|---|---|---|
| Simple web search | ~15,000 / ~2,500 | ~US$0.18 |
| Analysis of a large page (100K) | ~100,000 / ~3,000 | ~US$0.75 |
| Search + analysis | ~30,000 / ~5,000 | ~US$0.36 |

### 4.5 Limits and caps

| Limit | Value |
|---|---|
| `max_uses` — Web Search / Page Analysis | 10 (Page Analysis locked to 5) |
| `max_search_uses` / `max_fetch_uses` — Search + Analysis | 5 each |
| `max_content_tokens` — Page Analysis | 200,000 |
| `max_content_tokens` — Search + Analysis | 50,000 (hard-coded) |
| Allowed / blocked domains | 20 entered, **10 kept** |
| Query length | 4,000 characters |
| URL length | 2,048 characters |
| Model response | 32,000 tokens |
| History displayed | 50 entries (server bound 100) |
| Database truncation (`query`, `result_preview`) | 500 characters each |

### 4.6 Error codes and statuses

Source: `web.py`, `erp_auth.py`.

| Case | Status | Message |
|---|---|---|
| Anthropic SDK / key missing | 503 | "AI service unavailable" |
| Empty query | 400 | "The search query is empty" |
| Empty URL | 400 | "The URL is empty" |
| URL without `http(s)://` | 400 | "The URL must start with http:// or https://" |
| AI guard rejects (blocks no one today) | 403 | "AI access denied" |
| Credit balance exhausted | 402 | "Insufficient AI credits" |
| Tenant is read-only | 403 | "Read-only mode" |
| Tenant blocked | 401 | (access denied) |
| Too many requests (see 4.7) | 429 | (with `Retry-After: 60` header) |
| Internal error (search) | 500 | "Error during web search" |
| Internal error (analysis) | 500 | "Error while analyzing the page" |
| Internal error (search + analysis) | 500 | "Error during in-depth search" |
| History loading error | 500 | "Error loading history" |

### 4.7 Rate limiting

Source: `erp_api.py:365`, `erp_api.py:473`, `erp_api.py:691-693`.

- **10 requests per 60 seconds per IP address** across the **three** search functions (shared key `{ip}:web`). Rationale: Opus at 32,000 tokens plus per-search server costs = the most expensive class.
- The **History** tab (`GET /web/history`) is **not** subject to this dedicated limit; it falls back on the general cap (1,500 requests/minute).
- If exceeded: **429** response with `Retry-After: 60`. Wait one minute.

### 4.8 The `web_search_history` table (tenant schema)

Created **on demand** (`CREATE TABLE IF NOT EXISTS`) on the first record, then cached per process to avoid a lock on every search (`web.py:173-186`). No official migration.

```sql
CREATE TABLE IF NOT EXISTS web_search_history (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    search_type VARCHAR(30) NOT NULL,   -- 'search' | 'fetch' | 'search_fetch'
    query TEXT NOT NULL,                 -- truncated to 500 characters
    result_preview TEXT,                 -- truncated to 500 characters
    citations_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- Read **filtered by `user_id`**: **personal** history, not shared (`web.py:539`).
- **Best-effort** write: if the record fails, the search response is **not** compromised (`web.py:193-205`).
- No dedicated index, no foreign key to `users`, **no automatic purge**.

### 4.9 Billing and idempotency

Identical sequence on the three POSTs: `check_ai_guard` → `_check_credits` (402 if exhausted) → Claude call → `_compute_web_cost` → `track_ai_usage` → `_deduct_credits` (idempotent debit) → `_save_search_history`.

- **Debit** on `public.ai_prepaid_credits` (the balance may go negative; the next check is what triggers the US$10 Stripe top-up below the US$0.10 threshold). Super-administrators and no-billing instance: **no debit**.
- **Idempotency**: `Idempotency-Key` header (the app generates a unique identifier per request, `api/web.ts:49-51`). The server transforms it into `web:{key}:{fingerprint of the query or URL}` and records it in `public.ai_credit_ledger` (`INSERT ... ON CONFLICT DO NOTHING`). Result: a **double-click** or a **retry** with the same key **does not debit twice**, but reusing a key on a **different** request does **not** let you dodge billing.

### 4.10 Unexposed / hard-coded settings

| Setting | Value | User-adjustable? |
|---|---|---|
| Search location | Montreal / Quebec / CA | No |
| Model | `claude-opus-4-8` | No |
| Temperature | Model default | No |
| Page Analysis `max_uses` | 5 | No |
| Search + Analysis `max_content_tokens` | 50,000 | No |
| Cost markup | × 1.30 | No |
| Effective domains | 10 (of 20 entered) | Partially |

---

## 5. Integrations and FAQ

### 5.1 Link with the AI Assistant (credits and tracking)

The Web module **reuses** the AI Assistant's credit machinery: `check_ai_guard`, `_check_credits`, `_deduct_credits`, `track_ai_usage` (imported from `ai.py`). Its three features — `web_search`, `web_fetch`, `web_search_fetch` — appear in the shared **AI usage tracking** (`public.ai_usage_tracking`), alongside the ERP's other AI consumption. See the **AI Assistant** manual (`24-communication-assistant-ia.md`).

### 5.2 Prepaid credits and Stripe

Same credits as the whole ERP (`public.ai_prepaid_credits`). **Automatic top-up** of US$10 when the balance drops below US$0.10. **Internal instances** (`ERP_BILLING_ENABLED=false`) and **super-administrators**: no debit. Managing the payment method and the subscription is handled by the **Configuration** module (`30-configuration.md`).

### 5.3 Do not confuse: web search ≠ internal search ≠ materials search

| Need | The right tool |
|---|---|
| Search **the web** (standards, prices, regulations) | **This module** (Web) |
| Search **your data** (projects, invoices, clients) | **AI Assistant** → `recherche_bd` tool (`24-communication-assistant-ia.md`) |
| Search for a **material / supplier** online from the Store | **Materials web search** component of the **Store** module (`components/magasin/MaterialWebSearch.tsx`) — a **separate** tool, modeled on this one but focused on materials, **outside** the `/web` route (`09-operations-magasin.md`) |

### 5.4 What is NOT possible

- **No export** (PDF, CSV, Markdown), **no print button**, **no file upload** in this module.
- **No re-run** or deletion from history; **no sharing** between users (personal history).
- **No rich formatting** of the result (plain text, line breaks preserved).
- **No changing** the location (Quebec), the model, or the temperature.
- **No scheduled search** (no automation / timer): everything is on demand.
- **No fallback engine** if Anthropic is unavailable.
- On **Search + Analysis**: no "Block" or "None" mode (only "allow").

### 5.5 FAQ

**Which search engine is used?** Claude's native tools (`web_search_20260209` + `web_fetch_20260209`), run at Anthropic. No Google, Bing, SerpAPI, Brave, or DuckDuckGo integration.

**Why Opus and not Sonnet?** For synthesis quality on open-ended searches. In return, it is the most expensive AI function in the ERP.

**How much does a search cost?** The exact amount appears on the result card. Ballpark: simple search ~US$0.18, large-page analysis ~US$0.75, search + analysis ~US$0.36. The cost varies mainly with the amount of web content read.

**Can I have a file from my computer analyzed?** No. The Page Analysis tab only accepts a publicly accessible **URL**. For a local document, host it online first (or use another module).

**Can the Montreal/Quebec geolocation be changed?** No, it is hard-coded. Claude remains multilingual and can search in English, but results are steered toward Quebec / French-language sources.

**Can a colleague see my searches?** No. History is filtered by user: each person sees only their own.

**Can I re-run a search from history?** No, it is read-only. Copy the question into the desired tab.

**Is streaming visible on screen?** No. The server uses streaming (required by Anthropic for long tool operations), but the interface waits for the complete response, then displays it all at once.

**Is there a result cache?** No. Each search is re-run (and re-billed). Check the history first to avoid paying twice for the same thing.

**Are the sources reliable?** The URLs come from Anthropic; the ERP does not validate them. **Verify them** before relying on them or clicking.

**Does a double-click bill me twice?** No: an anti-double-click lock and an idempotency key prevent double-debiting the same request.

**What happens if I exceed a slider?** Nothing: the sliders are already bounded, and the server re-caps the values as a safeguard.

**Are my searches used to train the AI?** They pass through the Anthropic API. Anthropic does not use API data for training unless explicitly agreed. When personal information is involved, keep Quebec's Law 25 and PIPEDA (federal privacy law) in mind.

---

## 6. Summary

- **Scope**: live web search + page analysis, via Claude's `web_search_20260209` and `web_fetch_20260209` tools. **Single provider: Anthropic** — no Google/Bing/SerpAPI; **503** if unavailable, with no fallback.
- **Model**: `claude-opus-4-8`, response up to 32,000 tokens, **no temperature set**. It is the **most expensive** AI function in the ERP.
- **5 tabs**: Web Search · Page Analysis · Search + Analysis (the three paid) · History (read-only, **personal**) · Useful Links (8 static shortcuts).
- **4 endpoints**: `POST /web/search`, `POST /web/fetch`, `POST /web/search-fetch`, `GET /web/history`.
- **Geolocation** locked to Montreal/Quebec. **Domain filtering**: None / Allow / Block (mutually exclusive, 20 entered → 10 kept); **Search + Analysis** = "allow" only.
- **Caps**: `max_uses` 10 (Page Analysis locked to 5) or 5 (combined mode); page content 200,000 tokens (50,000 in combined mode).
- **Billing**: prepaid AI credits, Opus rates (US$5/25/6.25/0.50 per million) **× 1.30**. Automatic Stripe top-up of US$10 below US$0.10. **Idempotency** in place (no double debit). Internal instance / super-admin: no debit.
- **Access**: any authenticated user; no role restriction. A tenant in **read-only mode** has its 3 searches blocked with **403**, but keeps history and links.
- **Call rate**: 10 requests / 60 s per IP on searches; 429 with `Retry-After` if exceeded.
- **History**: `web_search_history` table per tenant, created on demand, filtered by user, 50 entries displayed, **without** re-run/deletion/export.
- **Limits**: no export, no printing, no upload, no rich formatting, no cache, no automation. For your **internal data**, use the **AI Assistant** (`recherche_bd`).

---

**Verified sources**: `backend/routers/web.py` (572 lines, 4 endpoints), `frontend/src/pages/WebPage.tsx` (736 lines, 5 tabs), `frontend/src/store/useWebStore.ts`, `frontend/src/api/web.ts`, `frontend/src/i18n/locales/fr/web.json` (138 lines), `backend/erp_api.py` (mounting + rate limiting), `backend/erp_auth.py` (read-only mode), `backend/routers/ai.py` (credits, tracking, idempotency).

**Related manuals**:
- AI Assistant (prepaid credits, tracking, `recherche_bd`) — `24-communication-assistant-ia.md`
- Store (materials web search, separate tool) — `09-operations-magasin.md`
- Configuration (subscription, payment method, credits) — `30-configuration.md`
- Sales — Files (keep a result in a file) — `06-ventes-dossiers.md`
