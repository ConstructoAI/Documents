# Module 15 — Site Weather

> **Version**: 3.0 (overhaul verified against the source code of July 7, 2026 — added the AI assistant and the 30-minute cache, and corrected the actual location of the endpoints)
> **Menu label**: "Weather" ("FIELD" group in the sidebar, `CloudSun` icon) — route `/meteo`
> **Reference code (backend)**: `ERP_REACT/backend/routers/secondary.py` lines 8653-8744 (2 Open-Meteo weather endpoints, ~92 lines); `ERP_REACT/backend/routers/meteo_ai.py` (165 lines, 1 endpoint `POST /meteo/ai/chat`, AI planning assistant)
> **Reference code (frontend)**: `ERP_REACT/frontend/src/pages/MeteoPage.tsx` (281 lines, single page); `ERP_REACT/frontend/src/components/meteo/MeteoAssistantTab.tsx` (122 lines, AI chat); `ERP_REACT/frontend/src/api/secondary.ts` lines 47-53 (2 weather functions); `ERP_REACT/frontend/src/api/meteoAi.ts` lines 25-37 (chat function)
> **External service**: Open-Meteo (`https://api.open-meteo.com/v1/forecast`) — public API, free, no key, no cost
> **PostgreSQL tables**: no tenant table. The module stores no forecast (30-minute in-memory cache only). Only the AI assistant writes to the shared tables `public.ai_usage_tracking` (usage tracking) and `public.ai_prepaid_credits` (credit debiting).
> **Scope**: a **passive-consultation** tool for 7-day forecasts across 7 urban stations in Quebec, with automatic highlighting of site risks (frost, rain, wind) and an **AI planning assistant** (an advisory chat, with no access to data). It **is not** a push-alert system: it generates no email, no notification and no calendar event, and has **no database link** to projects, phases or work orders.

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

Give field crews and planners a **quick view of the weather forecast over 7 days** for Quebec's main urban areas, with an **automatic reading of site risks** (frost, precipitation, wind) accompanied by **operational recommendations** (whether or not to pour concrete, asphalt paving, exterior painting, work at heights), plus an **AI assistant** that advises on how to organize the work according to the weather.

Concretely, the module answers three questions:

- "What will the weather be in Québec, Montréal or Saguenay over the next 7 days?"
- "Which site activities should I adapt, postpone or suspend based on this weather?"
- "How should I organize my work week given the forecast conditions?" (AI assistant)

### 1.2 What the module does (verified against the code)

- List **7 pre-configured Quebec weather stations** (Montréal, Québec, Gatineau, Trois-Rivières, Sherbrooke, Saguenay, Rimouski) via `GET /weather/stations` (`secondary.py:8653`).
- Retrieve the **7-day daily forecast** from Open-Meteo via `GET /weather/forecast?lat=X&lon=Y` (`secondary.py:8692`): maximum temperature, minimum temperature, total precipitation (mm) and maximum wind (km/h).
- Display **7 cards** (one per day) with a **dynamic icon** based on the dominant condition and a **badge** when a threshold is crossed: "Frost", "Rain" or "Strong wind".
- Mark the current day's card with a "Today" badge and a blue border (calendar date in Quebec time).
- Generate a consolidated **"Site impact" section** that lists the at-risk days with a **text recommendation** per event (6 levels, see section 4.5). If no day crosses a threshold: a green message "No weather alerts - favorable working conditions".
- Open an **AI planning assistant** (a natural-language chat) that advises on organizing the work according to the weather — with no access to the tenant's database.

### 1.3 What the module does not do

> **Important**: this module is **strictly consultative**. The forecast part is read-only; only the AI assistant consumes credits. The module **does not implement**:

- **No link to projects or sites.** Despite the name "Site Weather", the weather is provided by **fixed city** (7 stations) and never by site address or project. There is no project selection, no geolocation, no coordinate pulled from a record. The "weather-to-site" match is **human**: you choose the nearest station yourself.
- **No custom location entry in the screen.** The backend technically accepts an arbitrary latitude/longitude pair, but the interface sends **only** the coordinates of the 7 hard-coded stations. No address field, no GPS.
- **No persistence or history.** Forecasts come from Open-Meteo live; a 30-minute in-memory cache exists on the server side, but **no database table** stores the weather.
- **Daily forecast only** (7 days). No hourly forecast, no "feels like", no humidity, UV index, cloud cover, text description or snow accumulation. Only 4 measurements per day.
- **Purely visual alerts.** The frost, rain and wind thresholds are computed **client-side** on the displayed forecasts: **no push notification** (email, browser notification, text message, webhook), **no reminder**, **no configurable threshold** (the thresholds are hard-coded in `MeteoPage.tsx`).
- **No automatic blocking.** Even if the screen shows "STOP work at heights", no work order and no phase is suspended by the system.
- **No export** (PDF, CSV, iCal), no printing, no upload.
- **No simultaneous multi-station comparison** (only one station displayed at a time).
- **No civil emergency alert**: the module is not connected to Environment Canada, the MSC (Meteorological Service of Canada) or the Ministry of Public Security. The thresholds are internal benchmarks, not official warnings.

### 1.4 Access via the sidebar

- Left sidebar → **FIELD** group (collapsible) → **Weather** (`CloudSun` icon). Ref. `Sidebar.tsx:71`.
- Direct URL: `/meteo`.
- Breadcrumb: "Weather".
- Single page, no tabs or sub-pages. The AI assistant opens in a modal window over the page.

### 1.5 Permissions and roles

- **Authentication required.** All 3 endpoints are guarded by `get_current_user`: any user logged into the tenant can view the weather and use the assistant.
- **No particular role required.** Neither administrator, nor super-administrator, nor business role: viewing is open to all members of the company.
- **No per-tenant restriction.** The 7 stations are identical for everyone; Open-Meteo is a global, public API.
- For the AI assistant, the only real gatekeeper is the company's **AI credit balance** (see section 4.6). An account with no credits receives a 402 error and cannot send a message, but weather viewing itself remains free.

### 1.6 Module components

| Component | File | Role |
|-----------|------|------|
| Weather page | `pages/MeteoPage.tsx` | Single screen: header bar, 7 cards, Site impact section, assistant open button |
| AI Assistant (chat) | `components/meteo/MeteoAssistantTab.tsx` | Modal conversation window with the planning assistant |
| Message bubble | `components/ai/MessageBubble.tsx` | Renders the chat messages (shared with the other AI assistants) |
| Weather API | `api/secondary.ts` (47-53) | `listWeatherStations()` and `getWeatherForecast(lat, lon)` |
| Chat API | `api/meteoAi.ts` (25-37) | `chatMeteo(message, history, language, weatherContext)` |
| Weather endpoints | `routers/secondary.py` (8653-8744) | Fixed stations + Open-Meteo proxy with cache |
| AI endpoint | `routers/meteo_ai.py` (65) | Planning chat, a single call to Claude |

### 1.7 Costs and external limits

- **Weather viewing: free.** Open-Meteo requires no key and tolerates about 10,000 requests per day per IP address. A 30-minute server cache greatly reduces the number of actual calls (see section 4.8).
- **AI assistant: paid.** Each message sent to the assistant consumes the company's **AI credits** (the `claude-sonnet-4-6` model rate plus a 30% markup). This is the **only** financial effect of the module. Calculation details in section 4.6.
- **Open-Meteo server timeout: 10 seconds.** Beyond that, the request fails gracefully and the screen shows a temporarily-unavailable-service message.

### 1.8 Technical architecture

```
Frontend MeteoPage.tsx ──> Backend secondary.py /weather/*  ──> Open-Meteo (public, free)
   listWeatherStations()      GET /weather/stations               api.open-meteo.com/v1/forecast
   getWeatherForecast()       GET /weather/forecast               (30-min server cache)

Frontend MeteoAssistantTab ─> Backend meteo_ai.py /meteo/ai/chat ─> Claude (sonnet-4-6)
   chatMeteo()                POST /meteo/ai/chat                  (no database, no tool)
                                                                   debit AI credits + usage tracking
```

No database on the forecast side. The AI assistant accesses no tenant data: it makes only a single call to the model and writes only the usage trace and the credit debit to the shared tables `public.ai_usage_tracking` and `public.ai_prepaid_credits`.

---

## 2. Interface

Source: `MeteoPage.tsx` (281 lines) and `MeteoAssistantTab.tsx` (122 lines).

### 2.1 General layout

```
+-----------------------------------------------------------------------+
| Weather        Updated at 14:05   [Sparkles AI Assistant] [⟳] [ v ]   |
+-----------------------------------------------------------------------+
|  [D1]   [D2]   [D3]   [D4]   [D5]   [D6]   [D7]   <- 7 daily cards     |
|                        [Today]                                        |
|                        [Frost]                                        |
+-----------------------------------------------------------------------+
| ShieldAlert  Site impact - Recommendations                            |
|  [Snowflake] Fri May 2 - Frost expected (-1 °C)       [Warning]       |
|              Protect fresh concrete with insulating blankets...       |
|  [Droplets]  Sat May 3 - Significant rain (14 mm)     [Warning]       |
+-----------------------------------------------------------------------+
```

The card grid is responsive: 1 column on a phone, 2 on a tablet, 4 on a medium screen, 7 on a large screen (1280 px and up).

### 2.2 Header bar

Located at the top of the page (`MeteoPage.tsx:114-151`), from left to right:

| Element | Behavior |
|---------|----------|
| **"Weather" title** | Always displayed. |
| **"Updated at HH:MM"** | Timestamp of the last successful load. Appears **only after a success**, and is hidden on small screens. A soft failure does not update this timestamp (to avoid showing a misleading time). |
| **"AI Assistant" button** (`Sparkles` icon) | Opens the planning assistant's modal window (see 2.6). |
| **Refresh button** (`RefreshCw` icon) | Reloads the current station's forecast. The icon spins during loading. The button is disabled during a load or if no station is selected. Tooltip "Refresh". |
| **Station drop-down menu** | Lists the 7 stations by name. Changing station automatically reloads the forecast. |

### 2.3 The 7 forecast cards

Each card (`MeteoPage.tsx:156-208`) presents one day. Detailed anatomy:

| Element | Source or logic |
|---------|-----------------|
| **Abbreviated day** | `formatWeekdayShort(f.date)` → format such as "Fri May 2" (abbreviated weekday, day of the month, abbreviated month), following the interface language (en-CA / fr-CA). |
| **"Today" badge** | Displayed, with a blue border around the card, if the card's date is today's in Quebec time (`America/Montreal`). |
| **Day icon** (dynamic) | `Snowflake` if cold (min < 0), otherwise `CloudRain` if rainy (precip > 5), otherwise `Wind` if windy (wind > 40), otherwise `CloudSun`. The color follows: blue if cold or rainy, orange if windy, yellow otherwise. |
| **"Max" line** | Maximum temperature, followed by the "°" symbol (pink tint). |
| **"Min" line** | Minimum temperature, followed by "°". Shown in blue if it is below 0. |
| **"Rain" line** | Precipitation in mm. Shown in blue if above 5 mm. |
| **"Wind" line** | Maximum wind in km/h. Shown in orange if above 40 km/h. |
| **Card border** | Yellow outline if the day crosses at least one of the three card thresholds (cold, rain or wind). |
| **Condition badge** | Displayed if a card threshold is crossed. **A single badge**, in priority order: "Frost" (blue) if cold; otherwise "Strong wind" (red) if windy; otherwise "Rain" (yellow). |

> **Safe display of missing values.** If Open-Meteo returns a null value, the line shows "--" instead of "null". Numbers are formatted per language (fr-CA or en-CA), with at most one decimal place.

> **Watch out for the two sets of thresholds.** The thresholds that trigger a **badge on the card** are more permissive than those of the "Site impact" section. A card can therefore carry a "Rain" badge (precip > 5 mm) without any recommendation appearing below (Site impact only reacts beyond 10 mm). See the comparison table in section 4.5.

### 2.4 The "Site impact" section

A block added below the grid, **only if at least one forecast is available** (`MeteoPage.tsx:214-268`). The thresholds are computed client-side, on the displayed forecasts. Two variants:

**Variant A — no alert** (no day crosses the strict thresholds):

```
[HardHat green]  Site impact
                 No weather alerts - favorable working conditions
```

**Variant B — alerts detected**:

```
[ShieldAlert]  Site impact - Recommendations
+---------------------------------------------------------------+
| [Icon]   day - short message                     [Badge]      |
|          detailed recommendation                              |
+---------------------------------------------------------------+
| ... one line per event ...                                    |
+---------------------------------------------------------------+
```

Each line carries an icon based on the type (`Snowflake` for frost, `Droplets` for rain, `Wind` for wind), the "day - message" label, a severity badge ("Critical" in red or "Warning" in yellow) and the recommendation text.

> **Stacking is possible.** Frost, rain and wind are evaluated independently: a single day can therefore generate **up to 3** distinct lines in Site impact (for example frost + rain + wind on the same day). Within a category, only the most severe line is shown (critical **or** warning, never both).

### 2.5 Loading, empty and error states

- **Loading** (station list or forecast in progress): a page skeleton (`SkeletonPage`) in place of the grid.
- **No forecast** (empty array without error): centered message "No forecast available". The Site impact section does not appear.
- **Error**: a red banner at the top of the page shows the message. Two cases: "Error loading weather" (failure to load the stations) or "Weather service temporarily unavailable. Please try again later." (forecast failure).
- **Race between requests**: if you switch stations quickly, a late response from a previous station is ignored (internal anti-race guard), so that the forecast of a city other than the one selected is never displayed.
- **Language change**: switching FR/EN does not re-trigger a network call and does not reset the chosen station.

### 2.6 The AI assistant modal

Opened by the "AI Assistant" button, title "**AI Assistant — Weather & planning**", large size (`MeteoPage.tsx:271-278`). It passes the assistant a context of the form "Weather station viewed: {station name}".

The chat (`MeteoAssistantTab.tsx`) includes:

- **Header**: `Sparkles` icon, title and subtitle "Job-site scheduling advice based on the weather.".
- **Empty state**: invitation message "Ask a job-site planning question based on the weather (concrete pouring, roofing, freeze/thaw, wind, rain). The assistant advises based on conditions; it does not read your internal data.", followed by **3 clickable examples**:
  - "Can I pour concrete tomorrow at these temperatures?"
  - "What precautions for roofing in strong wind?"
  - "How should I plan the week if rain is expected?"
- **Messages**: yours on the right, the assistant's on the left with a headset icon. Under each assistant reply, a bubble footer shows a "Weather" badge, the number of tokens consumed, the **cost in US dollars** (in orange, to 4 decimal places) and the reply time in seconds.
- **In progress**: indicator "Analyzing…".
- **Error**: a red banner under the thread; the message comes from the server if available, otherwise "An error occurred. Please try again.".
- **Input area**: a multi-line field with the placeholder "Ask your weather planning question…". The **Enter** key sends the message; **Shift+Enter** inserts a line break. "Send" button (paper-plane icon). An internal lock prevents double submission.

---

## 3. Step-by-step workflows

### 3.1 Check the weather for a site (main procedure)

1. Sidebar → **FIELD** group → **Weather**.
2. The page loads on Montréal by default (first station in the list), with 7 cards.
3. In the drop-down menu at the top right, choose the **city nearest the site**.
4. Scan the 7 cards: spot the days with a **yellow border** or a **badge** ("Frost", "Rain" or "Strong wind").
5. Scroll down to the **Site impact** section to read the recommendations specific to the at-risk days.
6. Decide manually (postpone a pour, move up a delivery, reorganize crews, cancel work at heights, etc.).
7. Communicate the decision to the crews through another channel (Messaging module, email, work order).

> **Reminder**: no automatic action. The weather writes to no table, no work order, no phase.

### 3.2 Check the weather window for pouring concrete

1. Select the station for the site's city.
2. Identify the days where the **minimum temperature stays above 0** over the 48 to 72 hours following the planned pour.
3. If the following night forecasts frost (min below 0), plan for insulating blankets and an antifreeze admixture — this is exactly the "Frost expected" recommendation in Site impact.
4. In the case of **"Severe frost"** (min below -10, "Critical" severity), the recommendation is **"Stop concrete pouring."** — strict application advised.
5. Also watch precipitation (avoid more than 10 mm in the 24 hours after the pour).

### 3.3 Plan asphalt paving and exterior painting

1. **Asphalt paving**: choose a window of 3 to 5 consecutive rain-free days (precipitation below 5 mm each day), ideally with a minimum temperature of at least 5.
2. **Exterior painting**: favor rain-free days (precip below 5 mm), with a minimum temperature of at least 10 and a maximum wind below 40 km/h.
3. The "Postpone exterior painting" recommendation appears automatically in Site impact as soon as a day exceeds 10 mm of rain.
4. There is no logic dedicated to these trades in the code: reading the table remains a human task. If needed, ask the AI assistant for advice (section 3.5).

### 3.4 Anticipate violent wind (work at heights)

1. Spot the "Strong wind" badges (wind above 40 km/h) on the 7 cards.
2. In Site impact, a **"Violent winds"** line (above 70 km/h, "Critical") carries the recommendation **"STOP work at heights. Lower crane. Secure all light materials and equipment."**
3. A **"Strong winds"** line (50 to 70 km/h, "Warning") recommends "Secure scaffolding and banners. Limit work at heights. Tie down light materials."
4. Coordinate with the superintendent and the crane operator — the decision to stop remains manual.

### 3.5 Use the AI planning assistant

1. Click **"AI Assistant"** in the header bar. The window opens, indicating the currently viewed station as context.
2. Type a planning question (for example: "With -8 at night and 12 during the day, is it safe to strip the formwork off a slab tomorrow?").
3. Press **Enter** to send (or **Shift+Enter** for a new line). The assistant thinks, then replies with cautious advice suited to Quebec's climate.
4. Chain your questions: the assistant takes into account the last 12 exchanges of the conversation.
5. Each reply shows its cost in US dollars: keep an eye on your AI credit balance.

> **What the assistant knows and does not know.** It **does not read** your internal data (neither projects, nor employees, nor stored forecasts: nothing is stored). It reasons from the weather you describe and from its general knowledge. The station name passed as context is treated as a **mere unverified hint**, never as an official measurement. For precise advice, state the temperatures, precipitation and wind yourself in your question.

### 3.6 Refresh, change station and handle errors

1. The **Refresh** button re-runs the call for the current station (useful because forecasts are cached for 30 minutes on the server side).
2. Changing station in the drop-down menu automatically reloads the page.
3. If Open-Meteo is unavailable: after 10 seconds at most, a red banner shows "Weather service temporarily unavailable. Please try again later." and the grid stays empty.
4. Workaround: try again later, or check Environment Canada or MétéoMédia (The Weather Network) in parallel for critical decisions.

---

## 4. Reference

### 4.1 The 7 stations (hard-coded)

Source: `secondary.py:8656-8664`.

| Code | City | Latitude | Longitude |
|------|------|----------|-----------|
| YUL | Montréal | 45.5017 | -73.5673 |
| YQB | Québec | 46.8139 | -71.2080 |
| YOW | Gatineau | 45.4765 | -75.7013 |
| YQT | Trois-Rivières | 46.3432 | -72.5419 |
| YSH | Sherbrooke | 45.4010 | -71.8884 |
| YSB | Saguenay | 48.4279 | -71.0685 |
| YRI | Rimouski | 48.4489 | -68.5243 |

> **Note.** The codes look like airport codes, but they are only labels: the coordinates point to the city center. Only the latitude/longitude pairs are sent to Open-Meteo. They are therefore **not** real Environment Canada stations.

### 4.2 Endpoints (3 in total)

Mount prefix: `/api/erp/v1`.

| Method | Full path | File:line | Authentication | Response |
|--------|-----------|-----------|----------------|----------|
| GET | `/api/erp/v1/weather/stations` | `secondary.py:8653` | `get_current_user` | `{ stations: [ {code, name, lat, lon} × 7 ] }` |
| GET | `/api/erp/v1/weather/forecast?lat=X&lon=Y` | `secondary.py:8692` | `get_current_user` | `{ forecasts: [...7], latitude, longitude }` or `{ forecasts: [], error }` |
| POST | `/api/erp/v1/meteo/ai/chat` | `meteo_ai.py:65` | `get_current_user` + AI credits | `{ response, input_tokens, output_tokens, cost_usd, credit_balance, elapsed_seconds }` |

Default values for `/weather/forecast` if `lat`/`lon` are absent: Montréal (`lat=45.5017`, `lon=-73.5673`). Accepted bounds: latitude from -90 to 90, longitude from -180 to 180.

### 4.3 Format of a forecast

Backend response (Python, lowercase with underscores):

```json
{
  "date": "2026-05-02",
  "temp_max": 18.4,
  "temp_min": 5.1,
  "precipitation": 2.3,
  "wind_max": 22.0
}
```

As seen by the interface (TypeScript, camelCase after transformation by the API client): `date`, `tempMax`, `tempMin`, `precipitation`, `windMax`. Each numeric value can be null; the interface then shows "--". The backend truncates all arrays to the shortest one so it never chokes on partial data (`secondary.py:8718`).

### 4.4 Card badge thresholds

Source: `MeteoPage.tsx:158-160`. These thresholds trigger the card's yellow border and badge.

| Condition | Threshold | Badge | Color |
|-----------|-----------|-------|-------|
| Cold | minimum temperature below 0 | "Frost" | blue |
| Rain | precipitation above 5 mm | "Rain" | yellow |
| Wind | maximum wind above 40 km/h | "Strong wind" | red |

A single badge is displayed, in priority **Frost > Strong wind > Rain**.

### 4.5 Site impact thresholds and recommendations

Source: `MeteoPage.tsx:218-232` and `terrain.json:42-53`. All thresholds are hard-coded.

| Type | Trigger threshold | Severity | Message | Recommendation |
|------|-------------------|----------|---------|----------------|
| Frost | min below -10 | **Critical** | Severe frost (X °C) | Stop concrete pouring. Protect pipes. Plan heating of work areas. |
| Frost | min below 0 (and -10 or above) | Warning | Frost expected (X °C) | Protect fresh concrete with insulating blankets. Use antifreeze additives. Check pipe protection. |
| Rain | precip above 20 mm | **Critical** | Heavy precipitation (X mm) | Postpone outdoor work. Secure excavations against flooding. Check drainage pumps. |
| Rain | precip above 10 mm (and 20 or below) | Warning | Significant rain (X mm) | Protect moisture-sensitive materials. Plan tarps for work areas. Postpone exterior painting. |
| Wind | wind above 70 km/h | **Critical** | Violent winds (X km/h) | STOP work at heights. Lower crane. Secure all light materials and equipment. |
| Wind | wind above 50 km/h (and 70 or below) | Warning | Strong winds (X km/h) | Secure scaffolding and banners. Limit work at heights. Tie down light materials. |

> **Comparison of the two threshold sets.** The cards (badges) react earlier than Site impact:
> - Card: frost below 0, rain beyond 5 mm, wind beyond 40 km/h.
> - Site impact: frost below 0 or below -10, rain beyond 10 mm or 20 mm, wind beyond 50 km/h or 70 km/h.
>
> A card can therefore carry a "Rain" badge (for example 6 mm) without generating a line in Site impact.

### 4.6 AI assistant — parameters, guards and cost

Source: `meteo_ai.py`.

| Aspect | Detail |
|--------|--------|
| Model | `claude-sonnet-4-6` |
| Maximum response length | 4000 tokens |
| Database / tools | **none**: a single call to the model, with no access to tenant data |
| Message length | from 1 to 8000 characters |
| History kept | the last 12 turns (roles "user" and "assistant"), each truncated to 8000 characters |
| Weather context | at most 4000 characters, treated as an **untrusted hint** (prompt-injection protection) |
| Language | Quebec French by default, English if the language starts with "en" (instruction repeated at the start and end of the prompt) |

**Guards, in order** (`meteo_ai.py:69-80`):

1. AI service not configured → **503** "AI service unavailable".
2. No tenant context → **400** "Missing tenant context".
3. AI access control (`check_ai_guard`) → **403** if refused. In practice this control lets any authenticated user through after a few exemptions: the real gatekeeper is the credit balance.
4. Credits exhausted → **402** "AI credits exhausted. Please top up your balance to continue.".

**Cost and billing**: `cost = (input_tokens × 0.003 + output_tokens × 0.015) ÷ 1000 × 1.30`. In other words, the model rate ($3 US per million input tokens, $15 US per million output tokens) **plus a 30% markup**. The amount is debited from the prepaid credits (`public.ai_prepaid_credits`, product "ERP") and tracked under the `meteo_chat` feature in `public.ai_usage_tracking`. As a rough guide, a typical reply of 500 input tokens and 600 output tokens costs about $0.0137 US.

**Errors** (`meteo_ai.py:157-164`): an overload or a timeout on the model side returns **503** "AI service temporarily unavailable."; any other error returns **500** "Internal error in the weather assistant.".

### 4.7 Rate limits (per IP address)

Ref. `erp_api.py`.

| Endpoint | Limit | Scope |
|----------|-------|-------|
| `POST /meteo/ai/chat` | 20 per minute | ERP |
| `GET /weather/forecast` | 40 per minute | **shared** between the ERP and the mobile app (same egress to Open-Meteo, common quota) |
| `GET /weather/stations` | no dedicated limit | hard-coded list, no external call |

### 4.8 The Open-Meteo external service

- **URL**: `https://api.open-meteo.com/v1/forecast`
- **Parameters**: `latitude`, `longitude`, `daily=temperature_2m_max,temperature_2m_min,precipitation_sum,wind_speed_10m_max`, `timezone=America/Montreal`, `forecast_days=7`.
- **Authentication**: none (free public API).
- **Server cache**: 30 minutes, up to 128 entries, key = latitude/longitude rounded to 2 decimals (the 7 stations therefore occupy 7 entries). Only successful, non-empty results are cached. The blocking call runs off the event loop so as not to freeze the shared server (`secondary.py:8708`).
- **Behavior on failure** (an HTTP 200 response nonetheless): an HTTP, network or timeout error → `{ forecasts: [], error: "Service meteo temporairement indisponible" }`; an unexpected error → `{ forecasts: [], error: "Erreur interne" }`.
- **Official documentation**: `https://open-meteo.com/en/docs`.

### 4.9 Interface text (internationalization)

- All language files are merged into a **single namespace**, each file being indexed by its name.
- The weather screen's labels therefore live under **`terrain.meteo.*`** (the `meteo` sub-section of the `terrain.json` file, 31 keys) — and **not** in a dedicated `meteo.json` file. The `terrain.json` file is in fact the **shared** bundle of the secondary modules (Weather, Compliance, Grants, Real Estate, Rental, Maintenance, Logistics).
- The assistant's labels live under **`meteoAssistant.*`** (the `meteoAssistant.json` file, 13 keys). The interface is bilingual (French/English).
- The menu entry and the breadcrumb use `nav.meteo` and `breadcrumb.meteo` = "Weather".

### 4.10 Keyboard shortcuts (AI assistant)

| Key | Effect |
|-----|--------|
| Enter | Send the message |
| Shift + Enter | Insert a line break |

---

## 5. Integrations and FAQ

### 5.1 Projects, phases, work orders, Gantt tracking

> **No direct integration.** The Weather module is a **standalone silo**:

- No foreign key `projet_id`, `phase_id` or `bt_id` exists in this module.
- No "See this project's weather" button in the Projects or Tracking modules.
- A work order scheduled on a severe-frost day is **not** flagged automatically: the superintendent postpones or suspends it manually.
- To mentally link a weather forecast to a site, you remember the city yourself and select the matching station.

### 5.2 Mobile app (weather at clock-in)

The only weather actually **tied to a site location** exists in the mobile time-tracking app, not in this ERP screen:

- At clock-in and clock-out, the mobile app captures a weather snapshot from the site address and stores it on the time record (`time_entries`).
- The mobile app also exposes its own `weather/stations` and `weather/forecast` endpoints (same 7 stations, same Open-Meteo API).
- This mobile weather is attached to the **time entries**, never to projects. It is independent of the ERP `/meteo` page.

### 5.3 The Field/Cadastre module is unrelated

The code file `terrain.py` carries the same prefix as the language-bundle name, but it **has nothing to do with the weather**: it is a **cadastre** module (a land record by lot number, zoning, environmental constraints, heritage proximity). It is currently **dormant** (no screen uses it). The weather endpoints, for their part, live in `secondary.py` and `meteo_ai.py`. Likewise, the GPS module concerns **vehicle** tracking, not the weather.

### 5.4 AI credits and billing

- The weather assistant shares the same **AI credit wallet** as the ERP's other artificial-intelligence features (product "ERP").
- Each message is tracked under the `meteo_chat` feature, visible in AI usage tracking (the super-administrator's AI Usage tab).
- Weather viewing (cards and Site impact) remains **entirely free**: only chat messages are billed.

### 5.5 CNESST and safety

Although the wind thresholds (beyond 50, then 70 km/h) overlap with best practices for work at heights, the module **does not incorporate** the legal obligations and produces **no** compliant work stoppage. The recommendations are indicative. Any decision to halt work must be validated by the person in charge, in accordance with the requirements of the CNESST (Quebec's workplace health and safety board) and applicable laws and regulations.

### 5.6 Frequently asked questions

**Can I add my own city (Drummondville, Joliette, etc.)?**
No, not from the interface. The 7 stations are hard-coded (`secondary.py:8656-8664`). Workaround: choose the nearest station (for example Trois-Rivières for Drummondville).

**Can I see more than 7 days or an hourly forecast?**
No. The horizon is fixed at 7 days and only daily values are requested. For hourly data, check Environment Canada or MétéoMédia directly.

**Does the module keep a history of the forecasts viewed?**
No. No persistence: each opening calls Open-Meteo live again (with a 30-minute server cache). To archive, take a screenshot.

**What happens if Open-Meteo is down?**
After 10 seconds at most, the page shows "Weather service temporarily unavailable. Please try again later." and the grid stays empty. Try again later.

**Are the frost, rain and wind thresholds configurable?**
No. They are hard-coded in the interface (`MeteoPage.tsx`) and identical for all tenants.

**Does the module send emails or block work orders on a critical alert?**
No. No push-notification system, no scheduled task, no webhook, no write to other tables. The recommendations are textual; the superintendent acts manually.

**Can I compare two cities side by side?**
No. Only one station is displayed at a time. Workaround: open two browser tabs.

**Is there a risk of exhausting the Open-Meteo quota?**
Low. The 30-minute server cache greatly reduces calls, and Open-Meteo tolerates about 10,000 requests per day per IP address. The rate limit of `/weather/forecast` is, however, **shared** with the mobile app.

**Does the AI assistant know my displayed weather and my projects?**
It only receives the **name of the station** viewed, as a mere unverified hint. It has access to **no** tenant data (neither projects nor stored forecasts). For precise advice, state the conditions yourself (temperatures, precipitation, wind) in your question.

**How much does a question to the assistant cost?**
The `claude-sonnet-4-6` model rate plus a 30% markup, debited from your AI credits. A typical reply costs a few US cents. The exact cost is shown under each reply.

**Is Open-Meteo reliable enough to make construction decisions?**
Open-Meteo aggregates several models. Accuracy is reasonable at 3-5 days and more uncertain at 7 days. For a critical decision (a major pour, a crane lift), always cross-check with Environment Canada and a field supervisor.

**Are the units metric?**
Yes: degrees Celsius, millimeters, kilometers per hour.

---

## 6. Summary

- **Purpose**: viewing 7-day forecasts for 7 Quebec cities, with automatic site recommendations and an AI planning assistant.
- **Access**: sidebar → **FIELD** group → **Weather** (`CloudSun` icon), route `/meteo`. Authentication required, no particular role.
- **3 endpoints**: `GET /weather/stations`, `GET /weather/forecast` (in `secondary.py`), `POST /meteo/ai/chat` (in `meteo_ai.py`). **No tenant table.**
- **Data source**: Open-Meteo (free, no key), 30-minute server cache. This is **not** Environment Canada.
- **7 stations**: Montréal, Québec, Gatineau, Trois-Rivières, Sherbrooke, Saguenay, Rimouski (hard-coded city-center coordinates).
- **4 measurements per day**: maximum temperature, minimum temperature, precipitation, maximum wind. Daily forecast only.
- **3 card badges**: "Frost" (min below 0), "Rain" (precip beyond 5 mm), "Strong wind" (wind beyond 40 km/h).
- **6 Site impact recommendations**: Severe frost, Frost expected, Heavy precipitation, Significant rain, Violent winds, Strong winds — computed client-side, stackable (up to 3 lines per day).
- **AI assistant**: `claude-sonnet-4-6` planning chat, **with no database or tool**, 4000 tokens maximum, billed to AI credits (model rate plus a 30% markup, `meteo_chat` feature).
- **What it does not do**: no project/site/work-order link, no geolocation, no city addition, no push notification, no history, no export, no configurable threshold.
- **On failure**: message "Weather service temporarily unavailable. Please try again later.", empty grid.

---

**Documentation generated from the source code**: `ERP_REACT/backend/routers/secondary.py` (8653-8744, weather endpoints + cache); `ERP_REACT/backend/routers/meteo_ai.py` (165 lines, AI assistant); `ERP_REACT/frontend/src/pages/MeteoPage.tsx` (281 lines); `ERP_REACT/frontend/src/components/meteo/MeteoAssistantTab.tsx` (122 lines); `ERP_REACT/frontend/src/components/ai/MessageBubble.tsx`; `ERP_REACT/frontend/src/api/secondary.ts` (47-53); `ERP_REACT/frontend/src/api/meteoAi.ts` (25-37); text under `i18n/locales/fr/terrain.json` (`meteo.*`) and `i18n/locales/fr/meteoAssistant.json`.

**Related manuals**:
- Module 08 — Construction Projects (geographic reference of the site, no technical link) — `08-ventes-projets.md`
- Module 02 — Tracking / Gantt (construction phases, no weather integration) — `02-suivi-gantt.md`
- Module 11 — Work Orders (to suspend manually on a critical alert) — `11-operations-bons-de-travail.md`
- Module 24 — AI Assistant (credit wallet shared with the weather assistant) — `24-communication-assistant-ia.md`
