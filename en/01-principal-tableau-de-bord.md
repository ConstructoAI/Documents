# Module 01 — Dashboard and statistics

> **Version**: 3.0 (overhaul verified against the source code — Dashboard and Analytics are now **merged** into a single six-tab page)
> **Reference code**: `backend/routers/dashboard.py` (285 lines, 5 endpoints), `backend/routers/analytics.py` (1553 lines, 25 endpoints including 20 active + 5 orphaned), `backend/routers/dashboard_ai.py` (456 lines, 1 endpoint — read-only BI assistant), `frontend/src/pages/AnalyticsPage.tsx` (1456 lines, 6 tabs), `frontend/src/components/analytics/DashboardAssistantTab.tsx` (166 lines)
> **Route**: `/dashboard` (renders `AnalyticsPage.tsx`; the former `/` and `/analyses` routes redirect here)
> **Charting library**: Recharts (AreaChart, BarChart, PieChart/Donut, ResponsiveContainer)

---

## Contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step workflows](#3-step-by-step-workflows)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The Dashboard is the company's control center. In a single place, and **read-only**, it brings together the key performance indicators (KPIs), charts and tables that answer the contractor's daily questions:

- Where do my projects, quotes and invoices stand?
- How much have I collected, and how much is still owed to me?
- Are my projects profitable?
- Are my employees well utilized?
- Is my stock sufficient, and what is it worth?
- What is the state of my sales pipeline?

Since version 3.0, a sixth tab also lets you **query this data in natural language** using an AI analysis assistant (see section 2.8).

The module is used only to **view**: no business data is entered or modified here. All displayed values come from the other modules (Projects, Quotes, Accounting, Employees, Store, CRM, Work Orders, Purchase Orders).

### 1.2 One page, six tabs (major change)

> **Important** — If you knew an earlier version of the ERP: the "Dashboard" and the "Analytics" page are **no longer two separate pages**. They are **merged** into a single six-tab page.

- The `/dashboard` route now renders the `AnalyticsPage.tsx` component (`App.tsx:200-206`).
- The former `DashboardPage.tsx` page **no longer exists** in the software.
- The old addresses redirect automatically to the merged page: `/` → `/dashboard` (`App.tsx:198`) and `/analyses` → `/dashboard` (`App.tsx:218-219`).
- Only **one entry** remains in the sidebar: **"Dashboard"** (`LayoutDashboard` icon, "Main" section, `Sidebar.tsx:40`). The "Analytics" entry is gone.
- The **title shown at the top of the page stays "Dashboard"** on every tab — there is no tab-specific title (`AnalyticsPage.tsx:440`).

### 1.3 Access from the sidebar

- Sidebar → **Main** section → **Dashboard** (`LayoutDashboard` icon).
- Address: `app.constructoai.ca/dashboard`.
- This is very often the home screen shown right after login (the root `/` redirects here).

### 1.4 Permissions and roles

| Rule | Detail |
|---|---|
| **No role restriction** | Any authenticated user of the tenant sees the page. There is no role check (`require_role`, `is_admin`, etc.) on `dashboard.py` or `analytics.py` — only `Depends(get_current_user)` protects the endpoints. |
| **Tenant-wide view** | All users see the **same** figures (those of the entire company). There is no "my projects" or "my hours" filtering per user. |
| **Sensitive data visible to all** | Since there is no role lock, **any employee of the tenant** can read revenue, margins, the sales pipeline and even the **average employee salary** aggregate (`employes_salaire_moyen`, exposed by `GET /dashboard`). Keep this in mind if field employees have ERP access. |
| **Platform super-admin** | A super-administrator without a tenant receives **empty / zero** responses (never an error). |
| **Consultation mode (inactive subscription)** | The consultation tabs remain accessible (these are reads). However, **the AI Assistant is blocked** (see section 5.2): sending a message is technically a write, which is refused in consultation mode. |

### 1.5 The six tabs (sub-modules)

| Tab | Label (desktop) | Label (mobile) | Icon | Content |
|---|---|---|---|---|
| `vue_globale` | **Global View** | Global | Eye | 16 KPI cards + 5 charts + 2 tables |
| `projets` | **Projects** | Projects | BarChart3 | Profitability, progress, creation per month, breakdown by status |
| `finances` | **Finances** | Finances | DollarSign | Revenue vs. expenses, accounts-receivable aging, pipeline, top clients |
| `rh` | **HR** | HR | Users | Hours worked, departments, productivity per employee |
| `stock` | **Stock** | Stock | Boxes | Stock value, restocking alerts, top suppliers |
| `assistant` | **AI Assistant** | AI | Sparkles | Natural-language analysis chat (read-only) |

The default tab is **Global View** (`AnalyticsPage.tsx:241`). On mobile, the tab bar scrolls horizontally and shows only the icon + a short label.

### 1.6 Two figure sources coexist

It helps to understand that the page draws its indicators from **two different mechanisms** — this explains some of the period selector's behavior (see section 4.4):

| Source | Nature | Reacts to the period? | Feeds |
|---|---|---|---|
| `GET /analytics/kpis` | 16 KPIs **computed over the chosen period** | Yes | "Row 1" and "Row 2" KPI cards of Global View, and the cards of the other tabs |
| `GET /dashboard` (`DashboardStats`, ~35 fields) | **Absolute** counters (current totals) | No (loaded once) | "Row 3" and "Row 4" KPI cards + the 2 tables of Global View |

---

## 2. Interface

### 2.1 Overall page layout

From top to bottom (`AnalyticsPage.tsx:436-455`):

1. **Header**: the "Dashboard" title on the left, and on the right the **Period** selector (drop-down menu).
2. **Six-tab bar**: Global View · Projects · Finances · HR · Stock · AI Assistant (horizontal scrolling on mobile).
3. **Content area**: displays the active tab.

The page has three global display states (see section 2.9): loading, load error, and normal content.

### 2.2 Period selector

Drop-down menu at the top right, four choices (`AnalyticsPage.tsx:39-44`):

| Label | Value (days) | Default |
|---|---|---|
| **30 days** | 30 | ✅ |
| **90 days** | 90 | |
| **6 months** | 180 | |
| **1 year** | 365 | |

> **Good to know**: this selector has a **partial** and non-obvious effect. It refreshes only part of the page's figures (the period KPI cards, project profitability, HR productivity and departments). Several charts are locked to 1 year, and **the charts specific to each tab do not refresh** when you change the period. The full details are given in section 4.4. This is not a display bug on your end: it is the software's current behavior.

### 2.3 "Global View" tab

This is the summary dashboard: 16 KPI cards spread over 4 rows, then charts, then two tables.

#### 2.3.1 Row 1 — period indicators (`AnalyticsPage.tsx:472-501`)

| Card | Value | Color | Subtitle / trend |
|---|---|---|---|
| **Revenue (completed)** | Net revenue for the period | green | "% vs. prev. month" trend |
| **Quotes sent** | Number of quotes sent | blue | "of N total" |
| **Active projects** | Number of active projects | purple | "N total" |
| **Active employees** | Number of employees with ACTIVE status | teal | — |

#### 2.3.2 Row 2 — period indicators (`AnalyticsPage.tsx:504-530`)

| Card | Value | Color |
|---|---|---|
| **Sales pipeline** | Total value of opportunities + "N opportunities" | purple |
| **Stock alerts** | Number of products below threshold | red if > 0, otherwise green |
| **Revenue collected** | Amount actually collected | green |
| **Balance due (invoices)** | Amount still to be collected | red if > 0 |

#### 2.3.3 Row 3 — absolute counters (`AnalyticsPage.tsx:536-561`)

> These four cards come from the former dashboard. They are **loaded once** when the page opens and **do not depend on the selected period**.

| Card | Value | Icon |
|---|---|---|
| **Companies** | Total number of companies (clients + suppliers) | Building2 |
| **Invoices** | Total number of invoices | Receipt |
| **Products** | Total number of products | Package |
| **Suppliers** | Total number of suppliers | Truck |

#### 2.3.4 Row 4 — operations and alerts (`AnalyticsPage.tsx:564-591`)

| Card | Value | Color / subtitle |
|---|---|---|
| **Work orders** | Total number of work orders | "N in progress" |
| **Completed projects** | Number of completed projects | green |
| **Draft quotes** | Number of draft quotes | amber — "To finalize" |
| **Alerts** | Number of active alerts | red if > 0 |

#### 2.3.5 Global View charts

| Chart | Type | Content |
|---|---|---|
| **Monthly revenue** | AreaChart (gradient) | Net revenue per month (`AnalyticsPage.tsx:594-614`) |
| **Revenue vs. Expenses** | AreaChart (2 areas) | Revenue and Expenses overlaid (`:618-644`) |
| **Project evolution** | Stacked AreaChart | In progress / Completed / Pending per month (`:646-677`) |
| **Invoice distribution** | Donut | Breakdown of invoices by status (`:682-688`) |
| **Work orders by status** | Donut | Breakdown of work orders by status (`:690-696`) |

Each chart shows "No data available" when there is nothing to plot.

#### 2.3.6 Global View tables

**Projects by status** (`AnalyticsPage.tsx:704-740`) — columns:
- **Status**
- **Count**
- **%** (with progress bar)

**Top 5 suppliers** (`AnalyticsPage.tsx:742-767`) — columns:
- **Supplier**
- **Orders** (number of purchase orders)
- **Total amount**

> These two tables, like rows 3 and 4, are loaded once and ignore the period selector.

### 2.4 "Projects" tab

#### 2.4.1 KPI cards (`AnalyticsPage.tsx:776-786`)

**Total projects** · **In progress** · **Completed** · **Total budget** (sum of the budgets of the analyzed projects).

#### 2.4.2 Project profitability — Budget vs. Actual cost (`AnalyticsPage.tsx:789-861`)

A horizontal bar chart (Budget vs. Actual cost, top 5 to 10) together with a **detailed table**:

| Column | Detail |
|---|---|
| **Project** | Project name |
| **Status** | Colored badge |
| **Budget** | Planned budget |
| **Cost** | Actual cost (labor + materials) |
| **Margin** | Budget − Cost (color depends on sign) |
| **%** | Margin in % — **green badge if ≥ 20**, **yellow if ≥ 0**, **red if < 0** |

A **Total row** closes the table.

#### 2.4.3 Other sections

- **Project progress** (`:864-890`): a list of progress bars, one per project, with the completion percentage. Bar color: **green if ≥ 100%**, **blue if ≥ 50%**, **amber** otherwise.
- **Projects created per month** (`:894-914`): AreaChart of the number of projects created per month.
- **Breakdown by status** (`:916-931`): Donut (In progress / Completed / Pending), computed client-side.

### 2.5 "Finances" tab

#### 2.5.1 KPI cards (`AnalyticsPage.tsx:940-970`)

| Card | Detail |
|---|---|
| **Revenue collected** | With "vs. prev. month" trend |
| **Balance due** | Red if > 0 — subtitle "N invoices" |
| **Quote conversion rate** | Accepted quotes ÷ total quotes — subtitle "A/T accepted" |
| **Sales pipeline** | Subtitle "N opportunities" |

#### 2.5.2 Sections

- **Revenue vs. Expenses (12 months)** (`:973-1004`): AreaChart with 3 areas — Revenue / Expenses / Margin.
- **Invoice distribution** (`:1008-1014`): Donut by status.
- **Accounts-receivable aging** (`:1016-1034`): BarChart by age bracket, cells colored by age. Brackets: **0-30 days / 31-60 days / 61-90 days / 90+ days**. Empty state: "No overdue invoices".
- **Sales pipeline** (`:1038-1093`): BarChart by stage + table **Stage (color dot) / Count / Amount** with a Total row.
- **Top clients (by revenue)** (`:1096-1142`): horizontal BarChart + table **Client / Type / Total revenue / Projects** with a Total row.

### 2.6 "HR" tab

#### 2.6.1 KPI cards (`AnalyticsPage.tsx:1150-1176`)

**Active employees** · **Total hours** (format `Xh`) · **Average hours/day** · **Departments** (count — subtitle "N active employees").

#### 2.6.2 Sections

- **Hours-worked trend (12 months)** (`:1179-1206`): **dual-axis** AreaChart — Hours (left axis) and Employees (right axis).
- **Breakdown by department** (`:1210-1218`): Donut by total hours.
- **Hours per employee** (`:1220-1238`): horizontal BarChart (top 5 to 8).
- **Detailed productivity** (`:1242-1290`): full table.

**Detailed productivity** table — columns:

| Column | Detail |
|---|---|
| **Employee** | First + last name |
| **Position** | Job function |
| **Dept.** | Department |
| **Days** | Days worked |
| **Hours** | Total hours |
| **h/day** | Hours per day — color depends on the 7.5 and 6 thresholds |
| **Projects** | Number of projects touched |

A **Total / Average** row closes the table.

### 2.7 "Stock" tab

> This is the only tab that does not need the period KPIs to display: it relies on its own stock summary.

#### 2.7.1 KPI cards (`AnalyticsPage.tsx:1298-1324`)

**Active products** (subtitle "N total") · **Stock alerts** (red if > 0) · **Total value** · **Categories** (count).

#### 2.7.2 Sections

- **Stock value by category** (`:1327-1351`): a BarChart **and** a Donut side by side.
- **Stock alerts (N)** (`:1354-1395`): table **Product / Category / Stock (+ unit) / Threshold / Level**. The **Level** column shows a bar + a percentage badge, colored **red if < 25%**, **yellow if < 50%**, **green** otherwise. Empty state: "No stock alerts".
- **Top suppliers** (`:1398-1448`): horizontal BarChart **and** table **Supplier / Ord. / Amount** with a Total row. This section appears only if there is at least one supplier with purchases.

### 2.8 "AI Assistant" tab (analysis, read-only)

This tab opens an **analysis chat** that answers your questions from your **real data** (`DashboardAssistantTab.tsx`).

#### 2.8.1 What you see

- **Header**: Sparkles icon, title **"AI Assistant — Dashboard"**, subtitle "Analyzes your indicators from your real data (read-only)."
- **Welcome screen** (before the first question): a welcome text + **three clickable example questions**:
  - "What is the total of my unpaid invoices?"
  - "Which projects are in progress and for what amount?"
  - "Show me my sales pipeline by status."
- **Conversation thread**: your messages and the assistant's replies, with automatic scrolling. While processing, an "Analysis in progress…" indicator appears.
- **Error banner** (if a problem occurs): shows the message returned by the server, otherwise "An error occurred. Try again."
- **Input area**: a text field (placeholder "Ask your question about the dashboard…") and a **Send** button. **Enter** sends the message; **Shift+Enter** inserts a line break.
- **Details under each reply**: "Analyst" profile, token count, cost and response time.

#### 2.8.2 What the assistant can and cannot do

| Can | Cannot |
|---|---|
| Read and summarize your finances, quotes, projects, opportunities, products, suppliers, work orders, companies and contacts. | **Write** anything: there is no action to confirm, no creation or modification (unlike the Accounting module's assistant). |
| Answer in natural language with exact figures pulled from the database. | **Read payroll, individual hours, salaries, SIN (Social Insurance Number), users, AI credits, emails** (deliberate exclusion — see sections 4.11 and 5.2). |
| Chain a few internal queries to build its answer. | Go outside your company's scope (no cross-tenant access). |

#### 2.8.3 Cost

Each question consumes prepaid **AI credits** (see section 4.11). This is the **only feature in the entire module that has a monetary effect**; the other tabs are free reads.

### 2.9 Page states

| State | Display |
|---|---|
| **Initial loading** | Page skeleton (`SkeletonPage`) until the indicators arrive. |
| **Load error** | Centered screen with an alert icon, the error message (or "Unable to load the indicators. Try again.") and a **"Retry"** button. |
| **Empty section** | The "EmptyState" component shows "No data available" (or a more specific message depending on the section). |
| **Rapid changes** | If you quickly switch period or tab, the page discards responses that arrive late (stale-response guard): you always see the data for the current selection. |

---

## 3. Step-by-step workflows

### 3.1 View the daily overview

1. Sidebar → **Dashboard** (often already shown after login).
2. The page opens on the **Global View** tab and loads in parallel: the 16 period KPIs (`/analytics/kpis`), the absolute counters and alerts (`/dashboard`), the charts (`/dashboard/charts`) and the top suppliers (`/dashboard/top-suppliers`).
3. Read rows 1 and 2 first (revenue, collected, balance due, pipeline, stock alerts) for the current snapshot.
4. Check the **Alerts** card (row 4): a red figure signals urgent quotes, low stock or overdue invoices.

> No automatic refresh: the figures are loaded when the page opens. To update, reload the page (F5) or reopen the tab.

### 3.2 Change the time horizon

1. At the top right, open the **Period** selector and choose 30 days / 90 days / 6 months / 1 year.
2. **Actual effect** (see the matrix in 4.4): the row 1 and 2 KPI cards, **project profitability**, **HR productivity** and **breakdown by department** update.
3. **Do not change** with the period: the charts specific to each tab (monthly revenue, invoice distribution, aging, pipeline, stock value, etc.), several series locked to 1 year, as well as rows 3-4 and the 2 Global View tables.

> If a chart does not move when you change the period, that is normal: refer to section 4.4 to see what actually responds to the selector.

### 3.3 Analyze project profitability

1. **Projects** tab.
2. Review the **Project profitability — Budget vs. Actual cost** table.
3. For each project, compare **Budget** and **Cost**, then read the **Margin** and the **%**.
4. Spot the **red** badges (negative margin) and **yellow** badges (low margin, between 0 and 20%): these are the projects to watch.
5. The **Total row** gives the overall margin across the analyzed projects.

### 3.4 Track HR productivity

1. **HR** tab.
2. Read the **Hours-worked trend** chart (hours on the left, number of employees on the right).
3. In the **Detailed productivity** table, look at the **h/day** column:
   - a high value may signal overload;
   - a low value may signal underutilization.
4. The **Total / Average** row gives the team average.

> The HR tab shows **aggregates** (hours per employee, per department). For individual detail, the employee record and payroll calculation, go to the **Employees (11)** and **Time Tracking (13)** modules.

### 3.5 Manage stock alerts

1. **Global View** tab: the **Stock alerts** card gives the number of products below threshold.
2. **Stock** tab → **Stock alerts** table for the detail.
3. For each product, read the **Level** column: a **red bar (< 25%)** is critical, **yellow (< 50%)** needs attention.
4. Restock via the **Purchase Orders (14)** module with the supplier concerned.

### 3.6 Steer the sales pipeline

1. **Finances** tab → **Sales pipeline** section.
2. Read the amount and number of opportunities per stage.
3. Spot the stages that are stuck (a lot of value tied up at the same stage).
4. Act in the **CRM (06)** module to move opportunities forward.

### 3.7 Monitor overdue invoices

1. **Finances** tab → **Accounts-receivable aging** section.
2. Analyze the four brackets (0-30 / 31-60 / 61-90 / 90+ days). The oldest brackets are the most at risk.
3. The **Invoice distribution** section shows the breakdown by status.
4. Follow up and collect in the **Accounting (15)** module.

### 3.8 Identify your best clients

1. **Finances** tab → **Top clients (by revenue)** section.
2. Read the total revenue and number of projects per client.
3. The **Total** row shows the contribution of your largest clients.

### 3.9 Analyze inventory value

1. **Stock** tab → **Total value** card.
2. **Stock value by category** section: spot the categories that tie up the most capital.
3. Cross-reference with the **Stock alerts** to balance overstock and shortages.

### 3.10 Query your data in natural language (AI Assistant)

1. **AI Assistant** tab.
2. Click an example or type your question ("Which invoices are unpaid and for how much?").
3. Press **Enter** (or the **Send** button).
4. Read the reply; the details (cost, tokens, time) appear below.
5. Continue the conversation: the assistant keeps the context of recent exchanges.

> Each question consumes **AI credits**. If your subscription is in **consultation mode**, the assistant is blocked (403 error) — see section 5.2.

### 3.11 Recover from a load error

1. If the error screen appears ("Unable to load the indicators."), click **"Retry"**.
2. If the error persists, reload the page (F5). As a last resort, check your connection or contact your administrator.

---

## 4. Reference

### 4.1 Endpoints — Dashboard (`/api/erp/v1/dashboard`)

`routers/dashboard.py` — protected by `Depends(get_current_user)`, without a role.

| Method + path | Purpose | Consumed by the interface? |
|---|---|---|
| `GET /dashboard` | Consolidated counters (`DashboardStats`, ~35 fields) + alerts | Yes (rows 3-4, tables) |
| `GET /dashboard/alerts` | 3 alerts: urgent quotes, low stock, overdue invoices | Yes |
| `GET /dashboard/charts` | 3 series: projects by status, monthly revenue (6 months), work orders by status | Yes (Global View) |
| `GET /dashboard/top-suppliers` | Top 5 suppliers by purchase volume | Yes (Global View, Stock) |
| `GET /dashboard/activity` | Recent activity (last 20 projects) | **No** — orphaned |

### 4.2 Endpoints — Active analytics (`/api/erp/v1/analytics`)

`routers/analytics.py` — 25 endpoints in total, of which **20 are actually used** by the interface.

| Path | Content | Period parameter |
|---|---|---|
| `GET /analytics/kpis` | 16 summary KPIs | `period_days` 1-365 (default 30) |
| `GET /analytics/projects/profitability` | Budget vs. actual costs | `period_days` 1-730 (default 90) |
| `GET /analytics/projects/evolution` | Projects per month and per status | `period_days` 30-730 (default 365) |
| `GET /analytics/commercial/pipeline` | Opportunity funnel (excluding LOST) | — |
| `GET /analytics/hr/productivity` | Productivity per employee | `period_days` 1-365 (default 30) |
| `GET /analytics/hr/departments` | Hours per department | `period_days` 1-365 (default 30) |
| `GET /analytics/finance/revenue-expenses` | Revenue vs. expenses + margin per month | `period_days` 30-730 (default 365) |
| `GET /analytics/inventory/alerts` | Products below threshold | — |
| `GET /analytics/top-clients` | Top clients by project budget | `period_days` 30-730 (default 365) |
| `GET /analytics/project-progress` | % completion (non-cancelled projects) | — |
| `GET /analytics/sales-pipeline` | Opportunities by status (includes LOST) | — |
| `GET /analytics/top-suppliers` | Suppliers by purchase volume | — |
| `GET /analytics/monthly-revenue` | Monthly revenue (12 months) | — |
| `GET /analytics/stock-value` | Stock value by category | — |
| `GET /analytics/trends` | Current month vs. previous month (%) | — |
| `GET /analytics/invoices-by-status` | Invoice donut (clients) | — |
| `GET /analytics/bt-by-status` | Work orders donut | — |
| `GET /analytics/hours-trend` | Hours worked per month | `period_days` 30-730 (default 365) |
| `GET /analytics/factures-aging` | Aging 0-30 / 31-60 / 61-90 / 91+ | — |
| `GET /analytics/stock-summary` | Stock summary (total, active, categories, value, alerts) | — |

### 4.3 Endpoints present but unused (orphaned)

These routes still exist on the server side but **are no longer called** by the interface (they were redundant or diverged from an official calculation). They appear nowhere on the screen:

- `GET /analytics/project-profitability` (divergent duplicate of profitability)
- `GET /analytics/workstation-load`
- `GET /analytics/top-clients-revenue`
- `GET /analytics/employee-productivity` ("all-time" duplicate of `/hr/productivity`)
- `GET /analytics/stock-alerts` (duplicate of `/inventory/alerts`)
- `GET /dashboard/activity`

### 4.4 The Period selector — actual-effect matrix

> This is the most counter-intuitive point of the module. Here is precisely what does and does not respond to the selector.

| Displayed element | Reacts to the period? | Why |
|---|---|---|
| Row 1 and 2 KPI cards (Global View) | ✅ Yes | `/analytics/kpis` receives the period |
| KPI cards of the Projects / Finances / HR tabs (from the KPIs) | ✅ Yes | same source |
| Project profitability (Projects) | ✅ Yes | `profitability` receives the period |
| HR productivity + Breakdown by department (HR) | ✅ Yes | `hr/productivity` and `hr/departments` receive the period |
| Project evolution / Creation per month | ❌ No | locked to 365 days |
| Revenue vs. Expenses | ❌ No | locked to 365 days |
| Top clients | ❌ No | locked to 365 days |
| Hours trend (HR) | ❌ No | locked to 365 days |
| Sales pipeline / Stock alerts | ❌ No | the period is ignored |
| **All tab charts** (monthly revenue, invoice distribution, work orders by status, aging, sales pipeline, stock value…) | ❌ No | per-tab loading does not depend on the period |
| Rows 3-4 + "Projects by status" and "Top 5 suppliers" tables (Global View) | ❌ No | absolute counters, loaded once |

**In summary**: for a reading *by period*, rely on the **KPI cards (rows 1-2)**, **profitability**, **productivity** and **departments**. The historical charts, for their part, are on a fixed horizon (most often 12 months).

### 4.5 The 16 period KPIs (`/analytics/kpis`)

Fields computed and bounded to the period (`analytics.py:80-230`): `revenus_total`, `projets_actifs`, `projets_termines`, `projets_total`, `employes_actifs`, `alertes_stock`, `opportunites_pipeline`, `valeur_pipeline`, `devis_total`, `devis_acceptes`, `devis_envoyes`, `devis_valeur_totale`, `factures_total`, `factures_solde_du`, `revenus_encaisses` (+ associated trends).

### 4.6 The absolute counters (`DashboardStats`)

`GET /dashboard` returns ~35 current counters (independent of the period), including: companies, invoices, products, suppliers, work orders (total + in progress), completed projects, draft quotes, pipeline value, five inventory counters, and the `employes_salaire_moyen` aggregate.

### 4.7 Key calculations

| Indicator | Rule |
|---|---|
| **Credit-note-aware revenue** | A credit note (`AVOIR`, stored as a positive value) is **subtracted**: net revenue = sales − returns. |
| **Client invoices only** | Revenue and balance due count only `type_destinataire` = `client` (or null) — **supplier** invoices are excluded. |
| **Statuses excluded from revenue** | Ignored: `ANNULEE`, `BROUILLON` (revenue); `PAYEE`, `ANNULEE`, `BROUILLON` (balance due). |
| **Labor cost** | `hours × hourly_rate`, falling back to `annual_salary / 2080 h`, then 0 if no rate. |
| **Project margin** | `Budget − (labor cost + materials cost)`. |
| **Margin in %** | `Margin ÷ Budget × 100` (if budget > 0). |
| **Aging** | Brackets based on `today's date − invoice date`: ≤ 30 / ≤ 60 / ≤ 90 / otherwise 91+. |
| **Trend (trends)** | `(current − previous) ÷ previous × 100`; 0 if the previous value is 0. |
| **Quote conversion rate** | `accepted quotes ÷ total quotes × 100`. |
| **Stock value** | `stock × COALESCE(cost price, unit price, 0)`. |
| **Missing months** | Months without data are filled with zeros (continuous curves). |

### 4.8 The three dashboard alerts (`/dashboard/alerts`)

| Alert | Condition |
|---|---|
| **Urgent quotes** | Due date ≤ 7 days and status "Sent" or "Pending". |
| **Low stock** | Product whose available stock ≤ minimum stock (minimum > 0). |
| **Overdue invoices** | Client invoice past due and unpaid. |

### 4.9 Color codes

| Context | Green | Yellow / Amber | Red / Blue |
|---|---|---|---|
| **Margin % badge** (Projects) | ≥ 20% | ≥ 0% | < 0% (red) |
| **Progress bar** (Projects) | ≥ 100% | < 50% (amber) | ≥ 50% (blue) |
| **Stock level** (Stock) | ≥ 50% | < 50% | < 25% (red) |
| **">0" cards** (stock alerts, balance due) | 0 (green) | — | > 0 (red) |

### 4.10 Periods and bounds

- Only four choices: **30 / 90 / 180 / 365 days**. No custom range.
- Server bounds depend on the endpoint: `period_days` accepted between 1 and 365, or 30 and 730 (an out-of-bounds value is refused with a 422 code).
- Historical data is **never purged**: the period limit only affects the display.

### 4.11 AI Assistant — scope, cost and security (`/dashboard/ai/chat`)

| Aspect | Detail |
|---|---|
| **Endpoint** | `POST /api/erp/v1/dashboard/ai/chat` |
| **Nature** | **Read-only** analysis chat — no writes, no action to confirm. |
| **Readable tables** | invoices, invoice lines, payments received, quotes, quote lines, projects, opportunities, products, materials, suppliers, forms (work orders), form lines, companies, contacts. |
| **Forbidden tables** | payroll, individual hours, salaries, SIN, users, AI credits, emails, platform companies, Stripe/Vapi/Hover data — deliberate exclusion (**Law 25** — Quebec privacy law — compliance). |
| **Isolation** | Refusal of references to another schema: impossible to read another tenant's data. |
| **Cost** | Actual cost of the Claude model **× 1.30** (30% markup), debited from prepaid **AI credits**. |
| **Model** | `AI_MODEL` (Sonnet), `max_tokens = 8000`, up to 5 internal iterations. |
| **History** | Up to 40 exchanges sent; the server keeps 40 and uses only the last 12. |
| **Error codes** | 503 if the AI is unavailable, 402 if credits are exhausted, 403 in consultation mode. |

### 4.12 Rate limiting

| Request | Limit |
|---|---|
| `GET /dashboard` and `GET /dashboard/alerts` | **Not limited** (exempt). |
| Other analytics reads (`/charts`, `/top-suppliers`, `/analytics/*`) | General interval (~1500 requests/min per IP). |
| `POST /dashboard/ai/chat` | **Dedicated interval**: 20 requests per window and per IP. |

### 4.13 What does not exist in this module

- **No PDF or Excel export**, no printing. (The "Export PDF / Excel / Refresh" labels from the old Analytics page remain in the translations but are **no longer attached to any button**.)
- **No manual "Refresh" button**: refreshing happens when the page opens, when the period changes, or when the tab changes.
- **No custom date range**: only the 4 predefined periods.
- **No drill-down (detailed navigation)**: the KPI cards, table rows and chart segments **are not clickable**.
- **No customization** of the layout (no draggable widgets).
- **No role differentiation**: all tenant users see the same figures.
- **No automatic refresh** (no periodic polling, no real time).
- **The AI assistant never writes** and **does not read payroll / HR / SIN / credits**.

---

## 5. Integrations and FAQ

### 5.1 Data sources (all modules)

The Dashboard stores nothing: it **aggregates** data produced elsewhere.

| Source module | Data consumed |
|---|---|
| **Projects (09)** | Statuses, budgets, dates, progress, costs |
| **CRM (06)** | Opportunities, stages, estimated amounts (pipeline) |
| **Quotes (08)** | Count, amounts, conversion rate |
| **Work Orders (12)** | Count by status, materials → project costs |
| **Purchase Orders (14)** | Purchase volumes, top suppliers |
| **Accounting (15)** | Invoices (status, aging, revenue, balance due) |
| **Employees (11)** + **Time Tracking (13)** | Active employees, hours, productivity, labor cost |
| **Store (10)** | Stock, thresholds, value, categories |
| **Companies (04)** / **Contacts (05)** | Client and supplier resolution |

### 5.2 Consultation mode (inactive subscription)

If the tenant's subscription is inactive ("consultation" / read-only mode):

- **All consultation tabs remain accessible**: these are reads, they go through.
- **The AI Assistant is blocked (403)**: sending a message to the chat counts as a write, and writes are refused in consultation mode. To re-enable it, the subscription must be brought up to date (**Configuration (28)** module / Stripe).

### 5.3 Frequently asked questions

**Q: Why do I now have a single page instead of the Dashboard and the Analytics page?**
A: The two have been **merged** into a single six-tab page. The old `/` and `/analyses` addresses redirect automatically to `/dashboard`. This is intentional.

**Q: I change the period but a chart does not move. Is this a bug?**
A: No. The selector has only a **partial effect** (section 4.4). Only the row 1-2 KPIs, profitability, productivity and departments react. The historical charts are on a fixed horizon (often 12 months), and the tab charts do not refresh when the period changes.

**Q: Can I click an indicator to see the detail?**
A: No, there is no drill-down. Note the figure, then open the relevant module (Accounting, Projects, Store…) for the detail.

**Q: How do I export the dashboard to PDF or Excel?**
A: This is not possible from this module. Workaround: screenshot, or copy-paste the tables. For formal reports, go through the relevant modules.

**Q: Does the "Balance due" include supplier invoices?**
A: No. Only **client** invoices (`type_destinataire` = `client`) are counted. Supplier purchases fall under liabilities in Accounting.

**Q: Is a credit note properly deducted from revenue?**
A: Yes. The calculation is credit-note-aware: a credit note is subtracted, giving net revenue (sales − returns).

**Q: Do all my employees see revenue and margins?**
A: Yes. There is **no role restriction** on this module: any tenant ERP user sees the financial aggregates, and even the average salary. Bear this in mind before granting ERP access to field employees.

**Q: Can the AI assistant view salaries or payroll?**
A: No, never. This data (payroll, individual hours, salaries, SIN, credits, emails) is **excluded** from its scope, in accordance with Law 25. For payroll questions, use the **Employees (11)** module's assistant.

**Q: Can the AI assistant create an invoice or modify a project?**
A: No. It is strictly **read-only**: no writes, no action to confirm.

**Q: Why does my AI assistant return a 403 error?**
A: Probably because your subscription is in **consultation mode**. Sending a message is considered a write, so it is blocked. Bring the subscription up to date to re-enable it.

**Q: How much does a question to the assistant cost?**
A: The actual cost of the Claude model, marked up 30%, debited from your AI credits. It is the only paid feature of the module.

**Q: Are the figures real-time?**
A: They reflect the exact state at the moment the page **loads**. There is no automatic refresh: reload to update.

**Q: Can I choose a precise date range (e.g., March 1 to 15)?**
A: No. Only four predefined periods exist (30 / 90 / 180 / 365 days).

**Q: Why do I see "No data available" in a section?**
A: There is nothing to show for the current selection (no project, no invoice, no time entry in the relevant horizon). This is not an error.

**Q: The title shows "Dashboard" even when I am on the Finances tab. Is that normal?**
A: Yes. The page title is fixed; only the active tab changes the content.

---

## 6. Summary

- **A single six-tab page**: Global View · Projects · Finances · HR · Stock · AI Assistant. Route `/dashboard` (renders `AnalyticsPage.tsx`).
- **Dashboard + Analytics merger**: `DashboardPage.tsx` no longer exists; `/` and `/analyses` redirect to `/dashboard`; a single "Dashboard" entry in the menu.
- **Read-only module**: no business data entry; the only exception, the AI Assistant (read-only chat, paid in credits).
- **Two figure sources**: the period KPIs (`/analytics/kpis`, 16 indicators) and the absolute counters (`/dashboard`, ~35 fields).
- **Period selector with partial effect**: refreshes only the KPIs (rows 1-2), profitability, productivity and departments; the rest is locked or insensitive (section 4.4).
- **Global View**: 16 KPI cards (4 rows), 5 charts, 2 tables.
- **Library**: Recharts (AreaChart, BarChart, Donut).
- **AI Assistant**: read-only, restricted BI scope, **excludes payroll/HR/SIN/credits (Law 25)**, actual cost × 1.30, blocked in consultation mode.
- **Alerts**: urgent quotes, low stock, overdue invoices.
- **Color codes**: margin (green ≥ 20 / yellow ≥ 0 / red < 0), progress (green ≥ 100 / blue ≥ 50 / amber), stock (red < 25 / yellow < 50 / green).
- **No role restriction**: all tenant users see revenue, margins and average salary.
- **Assumed absences**: no PDF/Excel export, no printing, no Refresh button, no custom range, no drill-down, no customization, no real time.
- **Rate limiting**: `GET /dashboard` and `/dashboard/alerts` not limited; the rest under the general interval; the AI assistant capped at 20 requests per window and per IP.

---

**Verified source files**:
- `ERP_REACT/frontend/src/pages/AnalyticsPage.tsx` (1456 lines, 6 tabs)
- `ERP_REACT/frontend/src/components/analytics/DashboardAssistantTab.tsx` (166 lines)
- `ERP_REACT/frontend/src/api/analytics.ts` (210 lines), `api/dashboard.ts` (24 lines), `api/dashboardAi.ts` (43 lines)
- `ERP_REACT/backend/routers/dashboard.py` (285 lines, 5 endpoints)
- `ERP_REACT/backend/routers/analytics.py` (1553 lines, 25 endpoints — 20 active, 5 orphaned)
- `ERP_REACT/backend/routers/dashboard_ai.py` (456 lines, 1 endpoint)
- `ERP_REACT/backend/erp_models.py` (`DashboardStats`, `DashboardAlert`, `DashboardResponse`)
- `ERP_REACT/frontend/src/App.tsx` (routing `/`, `/analyses` → `/dashboard`), `Sidebar.tsx:40`

**Related manuals**:
- Module 01 — Analytics (`01-principal-tableau-de-bord.md`): former page now merged into the present module.
- Module 05 — CRM (`05-gestion-crm-opportunites.md`): source of the sales pipeline.
- Module 07 — Quotes (`07-ventes-soumissions.md`): source of the quote conversion rate.
- Module 08 — Projects (`08-ventes-projets.md`): source of profitability and progress.
- Module 09 — Store (`09-operations-magasin.md`): source of stock alerts and value.
- Module 10 — Employees (`10-operations-employes.md`) + Module 12 — Time Tracking (`12-operations-pointage.md`): source of HR productivity.
- Module 13 — Purchase Orders (`13-operations-bons-de-commande.md`): source of the top suppliers.
- Module 14 — Accounting (`14-operations-comptabilite.md`): source of invoices, revenue, balance due and aging.
- Module 24 — AI Assistant (`24-communication-assistant-ia.md`): full conversational assistant (the dashboard's assistant is a read-only BI variant of it).
