# Module 28 — Construction Calculators

> **Version**: 3.0 (overhaul verified against the source code, July 2026)
> **Reference code**: `frontend/src/pages/CalculateursPage.tsx` (2,269 lines, **5 tabs**, **15 tiles**), `frontend/src/components/calculateurs/MasterProCalculatorBody.tsx` (903 lines, jobsite calculator), `frontend/src/components/calculateurs/MursParametriquePanel.tsx` (2,676 lines, wall framing), `frontend/src/api/calculators.ts` (1,642 lines), `backend/routers/calculators.py` (3,681 lines, **59 endpoints**, actual prefix `/api/erp/v1/calculators`), `backend/routers/calculators_data.py` (807 lines — tables of constants, standards, prices, the AI system prompt; **a data module, NOT a router**)
> **PostgreSQL table (per tenant)**: `calculator_history` (created on demand on the first call — a single table)
> **Scope**: a suite of **13 server-side construction calculators** (concrete, stairs, structural analysis, roofing, painting, electrical, plumbing, HVAC, welding, bending, metal weight, taxes, payroll) + **2 browser-side tools** (the jobsite **Calculator** and the **Parametric Walls**) = **15 tiles**, plus a **Claude AI Assistant** and a per-tenant persistent **history**. This module is an **estimating and preliminary-sizing aid**: it is **NOT** official structural-design software, **NOT** a BIM, **NOT** a CAD tool (see the *3D Modeling / CAD* module), and it replaces **neither** an engineer's seal **nor** the real payroll of the *Time Tracking and Payroll* module.

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

Give estimators, foremen, contractors and tradespeople a **professional calculation toolbox** tailored to Quebec, covering the main trades (structure, envelope, mechanical, metal) and financial calculations (taxes, payroll costs). Each calculator:

- encodes **standards-based formulas** (CSA — Canadian Standards Association, CNB — Canada's National Building Code, CCQ — the Quebec Construction Code, CCE — the Canadian Electrical Code, CNP — the National Plumbing Code, ASHRAE, AWS — American Welding Society, IIW — International Institute of Welding, Blondel, Hazen-Williams, Magnus);
- returns **material quantities**, **indicative costs in CAD** and **compliance verdicts** (*Compliant* / *Non-compliant* badge);
- can be complemented by the **AI Assistant** (Claude) for explanation, recommendation and diagnosis.

### 1.2 The two calculation types

The module blends two families of tools. The distinction matters to the user because it explains offline behavior, AI billing and saving.

| Type | Where the calculation runs | Tiles concerned | Connection required |
|---|---|---|---|
| **Server-side** (backend) | Python FastAPI, on Constructo's servers | The **13** construction calculators (see 1.3) | Yes — a valid session token |
| **Browser-side** (client) | Directly in your browser, in JavaScript | The **Calculator** (`master-pro`) and the **Parametric Walls** (`murs-parametrique`) | The calculation itself is local; saving a walls project goes through the server |

> The two browser-side tools have **no server endpoint** and are **excluded from the AI Assistant** (the AI has no context for them). This is intentional: they are instant calculators.

### 1.3 Inventory of the 15 tiles (6 categories)

Source: `CALC_DEFS` (`CalculateursPage.tsx:52-68`) and `CALCULATEURS_LISTE` (`calculators_data.py:694-708`).

| # | Category | Identifier | Display name | Calculation | Main standard |
|---|-----------|-------------|-------------|--------|------------------|
| 1 | Multi-purpose | `master-pro` | **Calculator** | Browser | Feet-inches math (Construction Master Pro style) |
| 2 | Structure | `concrete` | **Concrete** | Server | CSA A23.1 / ACI (American Concrete Institute) 209 / CNESST (Quebec's occupational health & safety board) |
| 3 | Structure | `stairs` | **Stairs** | Server | CCQ 9.8 / 3.4 / Blondel |
| 4 | Structure | `murs-parametrique` | **Parametric Walls (In development)** | Browser | Light wall framing (imperial) |
| 5 | Structure | `charge-tributaire-complete` | **Structural Analysis** | Server | CNB / CSA O86 |
| 6 | Envelope | `roofing` | **Roofing** | Server | CCQ 9.19 / CNB 4.1.6.2 |
| 7 | Envelope | `painting` | **Painting** | Server | Coverage / DFT / dew point (Magnus) |
| 8 | Mechanical | `electrical` | **Electrical** | Server | CCE 4-004 / 8-200 |
| 9 | Mechanical | `plumbing` | **Plumbing** | Server | CNP / Hazen-Williams |
| 10 | Mechanical | `hvac` | **HVAC** | Server | ASHRAE (U·A·ΔT) / 62.2 |
| 11 | Metal | `welding` | **Welding** | Server | CSA W47.1 / W59 / AWS D1.1 / IIW |
| 12 | Metal | `bending` | **Metal Bending** | Server | K-factor / air bending |
| 13 | Metal | `metal-weight` | **Metal Weight** | Server | Densities + AISC (American Institute of Steel Construction) W/C shapes |
| 14 | Finance | `taxes` | **Quebec Taxes** | Server | GST (Goods and Services Tax) 5% + QST (Quebec Sales Tax) 9.975% |
| 15 | Finance | `charge-tributaire` | **Employee Payroll** | Server | RRQ / RQAP / AE / CNESST / FSS / CCQ |

> **Number recap**: **15 tiles** = **13** computed server-side + **2** browser-side. The dashboard shows "15" (it counts the tiles), whereas internal technical documentation sometimes speaks of "13 calculators" (it counts the server-side families). Both are correct; they simply do not count the same thing.

### 1.4 Access

- Side menu → **Tools** → **Calculators** (calculator icon).
- Address: `/calculateurs`.
- Tab open by default: **Dashboard**.
- 5 top-level tabs (see [section 2](#2-interface)).

### 1.5 Permissions and roles

- **No particular role is required** to run a calculation. Any authenticated user of the tenant (active account) can use any calculator. There is no "engineer" or "estimator" role, and no read-only-mode restriction on calculations (they are pure calculations that do not touch the database).
- The **history** (`calculator_history`) is **shared at the tenant level**: all users of the same company see and manage the same saved calculations. It requires a valid tenant context (otherwise *error 400 — missing tenant context*).
- The **AI Assistant** adds one further gate: the AI service must be available and the tenant must have **prepaid AI credits** (see 1.6).

### 1.6 AI model and pricing

The AI Assistant calls Anthropic's Claude large language model. The actual parameters from the code:

| Item | Value (source `calculators.py:121-132`) |
|---|---|
| Model | `claude-opus-4-8` |
| Maximum response tokens | 32,000 |
| Input price | US$5 / million tokens |
| Output price | US$25 / million tokens |
| Cache write | US$6.25 / million tokens |
| Cache read | US$0.50 / million tokens |
| Markup billed to the tenant | × 1.30 |

Every AI message is **debited from the tenant's prepaid AI credits** (the same wallet as the general AI Assistant and AI Estimation), at **actual cost × 1.30**. Billing details:

- The charge happens **after** the response. If you **leave the AI tab while Claude is answering**, the request is **canceled** and **no credit is debited** (the code detects the disconnection — `abortAi`, `CalculateursPage.tsx:2039`).
- A **super-administrator** account is never billed.
- If credits are exhausted: *error 402*. If the AI is disabled for the tenant: *error 403*. If the Claude service is unavailable: *error 503*.

> **Version note**: some internal code comments still mention "Claude Opus 4.7" or "4.6". This is a trace of outdated documentation: the model actually executed is **`claude-opus-4-8`** with **32,000** response tokens. This manual describes the actual behavior.

---

## 2. Interface

### 2.1 The 5 top-level tabs

Source: `CalculateursPage.tsx:725-736`.

| # | Tab | Icon | Content |
|---|--------|-------|---------|
| 1 | **Dashboard** | Chart | 4 indicators (KPIs) + tile grids by category |
| 2 | **Calculators** | Calculator | Sidebar of the 15 tiles + panel for the chosen calculator |
| 3 | **Structural Analysis** | Ruler | Beam/lintel check (CNB/CSA O86) + diagram |
| 4 | **AI Assistant** | Sparkles | 4 AI sub-tools (chat, recommendations, explain a standard, diagnosis) |
| 5 | **History** | Clock | List of saved calculations + statistics |

At the top of the page: the title **"Calculators"** and the subtitle **"Construction calculation tools"**. Two status banners (red for errors, green for successes) appear under the title and are dismissible.

> **There is no separate "Conversions" tab.** A "Conversions" translation key exists in the code, but the tab is **not mounted** in the bar. Unit conversions are done in the **Calculator** (internal *Conversions* tab and conversion sheet — see 2.4).

### 2.2 The "Dashboard" tab

Four indicator cards at the top (`CalculateursPage.tsx:756-761`):

| Indicator | Displayed value |
|---|---|
| **Calculators** | **15** (number of tiles) |
| **Saved calculations** | Actual number of calculations in your history |
| **Quebec standards** | **"10+"** (fixed, indicative value) |
| **Claude AI** | **"4 tools"** (reflects the 4 sub-tools of the AI tab) |

Below the indicators, **6 sections** (one per category: Multi-purpose, Structure, Envelope, Mechanical, Metal, Finance) each present their tiles as buttons (colored icon + name + short description + chevron). **Clicking a tile** clears the previous results and switches to the **Calculators** tab with that calculator already open.

### 2.3 The "Calculators" tab

**Sidebar + panel** layout.

- **On desktop**: a sticky sidebar (on the left) lists the 15 tiles; the panel on the right shows the chosen calculator.
- **On mobile**: a **"Choose a calculator"** button opens a full-screen drawer with the list.
- **With no selection**: an empty card prompts you to "Select a calculator from the list on the left."

**Pattern common to all server-side panels**: a row of **sub-tabs** (pills) at the top, a **left card** with the form (each field = a label + an input field or a dropdown), a **"Calculate"** button (with a waiting indicator), and a **right "Results" card** that shows the values (some highlighted) and the **compliance badges** *Compliant* / *Non-compliant*. Changing calculator automatically **clears** the displayed results (to avoid showing an old result that no longer matches).

Two tiles escape this pattern: the **Calculator** (2.4) and the **Parametric Walls** (3.14), which have their own full interface. The **Structural Analysis** tile opens the same screen as the tab of the same name (2.5) — it is a dual entry point to the same tool.

The detail of the sub-tabs and fields of each of the 13 server-side calculators is given in [section 3](#3-step-by-step-workflows).

### 2.4 The Calculator (`master-pro`) — browser-side

A faithful replica of a physical **Construction Master Pro**-type jobsite calculator, shared with the Takeoff module. **100% local** calculation (no server call; works even offline once the page is loaded). No data is sent to Claude.

- **Two internal tabs**: **Framing** and **Conversions**.
- **Display** structured in feet-inches-sixteenths (FEET / INCH markers, fractions to 1/16) or in decimal, with status indicators (CONV, memory, Rise/Run/Pitch, L/L/H).
- **Function grid (20 buttons)** with a primary label and a "secondary" label (reached via the *Conv/Shift* key): *Pitch, Rise, Run, Diag, Hip-V; Miter, Stair, Arc, Circ, Jack; Length, Width, Height, %; Yds, Feet, Inch, Clear*.
- **Numeric pad (20 buttons)**: *Conv* toggle, digits with conversions as their secondary function (cm, board-foot "Bd Ft", mm, lb, studs "Studs", tons, kg, acre, metric tonnes, cost, degrees-minutes-seconds), memory (Store / Rcl / M+) and the ÷ × − = + operators.
- **Context detail panel**: depending on the last function, shows for example the risers and the run (going) of a staircase (with a Blondel check), the chords of a circle, an arc, a polygon, a miter/bevel, studs spaced at 40 or 60 cm.
- **Calculator footer** (4 buttons): *FT/IN* (feet-inches entry sheet with preset values), *Conversion* (m / cm / mm / ft / in / yd sheet), *History* (with a counter), *Clear all*.
- **Physical keyboard supported**: digits, operators, *Enter* (=), *Esc* (clear all), *f/i/y* for the units, *c* for the secondary function.
- **No server save**: the calculator's history lives in the current session only.

### 2.5 The "Structural Analysis" tab

Check of a member in bending (beam or lintel) per the CNB load combinations and the CSA O86 resistances. Accessible **either** via the top-level tab **or** via the "Structural Analysis" tile (both open the same screen).

**Input fields**:

| Field | Detail |
|---|---|
| Member type | **Beam** or **Lintel** only |
| Material type | **Dimensional lumber** (SPF) or **LVL** |
| Section | Loaded from the server according to the material (e.g., `2x10`) |
| Number of plies (*ply*) | 1 to 6 |
| Span | in mm |
| Tributary width | in m |
| Dead / live / snow load | in kPa |
| Use | Floor (deflection L/360) / Roof (L/180) / Lintel (L/360) |

**"Analyze"** button. **Results**: a verdict (compliant / non-compliant), a **diagram** of the beam, the maximum moment (M max), the maximum shear (V max), the deflection (Δ), the factored resistances Mr and Vr (CSA O86), the factored load at the ultimate limit state, and **3 checks** as percentages with a green check or red cross: **Bending**, **Shear**, **Deflection**.

> **The "Column" option is not offered.** The menu only offers Beam and Lintel. A column analysis (axial compression, buckling) is **not implemented** and is refused server-side. See the [limits](#44-limits-and-known-simplifications).

### 2.6 The "AI Assistant" tab

Four sub-tools (pills). Each call consumes AI credits (see 1.6).

| Sub-tool | What you provide | What the AI returns |
|---|---|---|
| **Expert Chat** | A calculator (optional) + your question | An answer in French, citing the standards; Your / Assistant history; *Clear* button |
| **Recommendations** | A calculator (among the 13 eligible) + a goal + constraints (optional) | Approach, steps, estimated costs |
| **Explain a standard** | A standard or an article (e.g., "CCQ 9.8", "CSA A23.1", "CCE 8-200") | Official title, body, main requirements, note |
| **Diagnosis** | A calculator + the problem + the symptoms | Diagnosis, urgency, probable causes, recommendation for professional intervention |

> **Two AI functions exist server-side but have no button in the interface**: *Analyze* (score a calculation from 0 to 100) and *Optimize* (suggest savings). They are therefore not usable from the current screen — only the 4 sub-tools above are exposed. That is why the dashboard indicator shows "4 tools".

### 2.7 The "History" tab

Three statistics at the top (**Total calculations**, **Calculators used**, **Last 30 days**), a per-calculator filter, a **"Clear all"** button (with confirmation) and the list of saved calculations. Each row shows the icon, the label and the date (`fr-CA` format), a **"Details"** button (expands the inputs and results in plain text) and a trash icon to delete an entry.

> **What the history actually contains.** In the current interface, **only the Parametric Walls write to the history** (that is how their wall projects are saved and reloaded). The 13 server-side calculators **have no wired "Save" button**: their results are ephemeral and **do not populate** the history automatically. The history therefore remains mainly a log of your wall projects (plus, possibly, entries created through integration). The persistence itself is real and per-tenant (table `calculator_history`).

### 2.8 Components present but not wired up

The `components/calculateurs/` folder contains **7 built but unconnected components** (no screen opens them): an advanced roofing panel, a floor panel, a cladding panel, a floor-plan editor, a reports window, a house-levels selector and a layers manager. They appear to be the remains of a partially integrated **multi-level house designer**. **They are not accessible** today and have no effect. The only truly active roofing calculator is the **Roofing** panel described in 3.7 (server-side).

---

## 3. Step-by-step workflows

Each calculator below lists its **sub-tabs**, its main **inputs**, the **formula/standard** applied and its **outputs**. The pattern is always the same: fill the left card → **Calculate** → read the right card.

### 3.1 Estimate the concrete for a slab (Concrete calculator)

The **Concrete** calculator has **7 sub-tabs**: *Volume · CSA mix · Rebar · ACI 209 curing · Excavation · CNESST slope · Concrete stairs*.

**Example — 6 m × 8 m garage slab, 100 mm thick:**

1. Open **Calculators** → **Concrete** → **Volume** sub-tab.
2. Enter length `6`, width `8`, thickness `0.1` (in meters), waste `10`%, and choose the **concrete class** (e.g., *C-2 (25 MPa exterior)*).
3. Click **Calculate**.
4. Read on the right: total volume (m³), area, cement / sand / gravel / water, **number of 30 kg bags**, and the number of 4×8 formwork sheets.

**Other sub-tabs:**

- **CSA A23.1 mix**: volume + target strength (15 to 40 MPa) → material quantities, mix ratio, water/cement ratio (W/C), 30 and 40 kg bags. The mix depends on the **exposure class** (the code maps class → strength → mix).
- **Rebar (CSA G30.18)**: dimensions, cover, spacing, bar type (10M to 55M), number of layers → number of bars in each direction, total length, cutting into standard 6 m bars, total mass (kg and lb). *Safeguard*: if the slab is thinner than twice the cover, the calculation is refused (negative effective dimensions).
- **Curing (ACI 209)**: final strength, age, temperature, cement type (GU / HE / MS / HS) → strength reached at the given age, percentage of the final strength, maturity factor, minimum curing duration. Formula `f(t) = f28 · t / (a + b·t)` with a maturity factor based on temperature.
- **Excavation**: dimensions + soil type (swell factor) → compact volume, swelled volume (m³ and cu yd), number of 12 cu yd trucks, estimated weight.
- **CNESST slope**: depth + soil type → slope angle, horizontal/vertical ratio, horizontal clearance distance, and the **list of CNESST requirements** (daily inspection beyond 1.2 m, engineer analysis recommended beyond 3 m, mandatory beyond 6 m).
- **Concrete stairs (Blondel)**: total height, width, slab thickness, target run and riser → Blondel *OK / Out* badge, number of steps, `2R + G` value, volume and materials.

### 3.2 Size a staircase (Stairs calculator)

**2 sub-tabs**: *CCQ dimensions · Guardrail*.

1. **Dimensions (CCQ 9.8 / 3.4)**: enter the total height to climb, the target run (going) and riser, the width, and the **use** (*Residential — CCQ 9.8* or *Commercial — CCQ 3.4*). **Calculate** → CCQ compliance badge (with the reference article), number of steps, actual run, `2R + G`, slope in degrees, walking line, and a **comfort evaluation**.
2. **Guardrail (CCQ 9.8.7)**: length, height, baluster spacing, use → two badges (Height and Balusters), number of balusters, number of posts, handrail length.

### 3.3 Check a beam or lintel (Structural Analysis)

1. **Structural Analysis** tab (or the tile of the same name).
2. Choose the **member type** (Beam or Lintel) and the **material** (SPF dimensional lumber or LVL); the list of **sections** then populates from the server.
3. Choose the section, the number of plies, enter the span (mm) and the tributary width (m).
4. Enter the **dead**, **live** and **snow** loads (kPa) and the **use** (Floor / Roof / Lintel — which sets the deflection limit).
5. Click **Analyze**.
6. Read the verdict, the diagram, M max / V max / deflection, Mr / Vr and the 3 checks (bending, shear, deflection).

> **Preliminary** result: the resistances use the CSA O86 resistance factor φ = 0.9, but with adjustment factors Kd = Kl = 1.0 and no size effect (Kz) — this is **deliberately conservative**. An engineer must validate and seal any official calculation.

### 3.4 Calculate a roof (Roofing calculator)

**4 sub-tabs**: *Area + shingles · Ventilation · Gutters · Snow load*.

- **Area + shingles**: dimensions, pitch (x:12), overhang, waste, material (shingles, membranes, sheet metal) → actual area (pitch factor `√(1 + (pitch/12)²)`), number of "squares", bundles, underlayment, ice-and-water membrane, and **material and total costs** in CAD.
- **Ventilation (CCQ 9.19.1)**: attic area + presence of a vapor barrier → ventilation ratio (1:300 with a vapor barrier, otherwise 1:150), net free ventilation area, soffit length, number of ridge vents or turbines (50/50 intake/exhaust split).
- **Gutters (CCQ 9.14.6)**: roof area, perimeter, size (4 to 7 in) → number of downspouts, length, hangers, corners, end caps.
- **Snow load (CNB 4.1.6.2)**: province (QC / ON / BC / AB), **city** (free-text field), roofing type, pitch, exposure (Normal / Exposed) → ground snow (Ss), slope factor (Cs), roof snow load (kPa and lb/ft²), dead load, design load. The full CNB formula is applied: `S = Is · [Ss · (Cb·Cw·Cs·Ca) + Sr]`. If the entered city is not listed, a default value is used.

### 3.5 Calculate paint (Painting calculator)

**3 sub-tabs**: *Quantity + cost · DFT · Dew point*.

- **Quantity + cost**: room dimensions and height, number of doors and windows, number of coats, paint type, surface type, application method (roller, brush, airless, HVLP) → net area (openings deducted), liters and gallons, theoretical dry film thickness, **cost before and after taxes** (GST + QST), cost per m², recoat time. Coverage is adjusted by absorption factors (based on the surface) and transfer efficiency (based on the method).
- **DFT (dry film)**: volume (mL), solids percentage, area → dry film thickness (µm and mils), coverage, evaluation.
- **Dew point (Magnus)**: air temperature, relative humidity, surface temperature → *Application OK / Condensation risk* badge, dew point, safety margin, recommendation. Used to confirm that a surface is warm enough before painting (common rule: surface ≥ dew point + 3°C).

### 3.6 Electrical (Electrical calculator)

**4 sub-tabs**: *Cable · Residential load · Lighting · Grounding*.

- **Cable (CCE 4-004)**: power (W), voltage, length, power factor, maximum voltage drop, conductor (copper / aluminum), circuit (single-phase / three-phase) → current (A), AWG gauge, cross-section (mm²), actual drop, ampacity (75°C), breaker, compliance. The chosen cross-section is the maximum of the voltage-drop criterion and the ampacity criterion (× 1.25).
- **Residential load (CCE 8-200)**: living area + loads (heating, air conditioning, range, dryer, water heater, others, in kW) → total demand (kW), current at 240 V, recommended service size (A), reference article. Base 5 kW + 1 kW per 90 m² increment.
- **Lighting (lumens)**: area, room type, luminaire flux, utilization and maintenance factors → number of luminaires, required illuminance (lux), layout, spacing.
- **Grounding**: soil resistivity, rod length and diameter, number of rods → *< 25 ohms (Hydro-Québec)* badge, total resistance and per rod.

### 3.7 Plumbing (Plumbing calculator)

**4 sub-tabs**: *DFU + WSFU · Hazen-Williams · Water heater · Drain slope*.

- **DFU + WSFU (CNP)**: number of fixtures (toilets, sinks, showers, bathtubs, kitchen sink, dishwasher, washer, floor drain) → total drainage fixture units (DFU) and water supply fixture units (WSFU), flow (GPM and LPM), recommended drain diameter.
- **Hazen-Williams**: flow, length (ft), diameter (in), material (C coefficient) → head loss (psi/ft), velocity, C coefficient.
- **Water heater**: number of bedrooms, bathrooms and people → *Adequate / Undersized* badge, capacity (gal and L), minimum first-hour rating (FHR).
- **Drain slope**: diameter, length, slope → *CNP compliant* badge, fall (m and in), recommended slope.

### 3.8 HVAC (HVAC calculator)

**5 sub-tabs**: *Heat load · Ducts · CFM per room · HRV/ERV · Cooling*.

- **Heat load (ASHRAE)**: area, ceiling height, insulation level, climate zone (8 reference cities) → design heat loss (W), BTU/h, tonnage, ventilation airflow (CFM), BTU/ft². The calculation relies on the **U·A·ΔT envelope method** (estimated envelope area + infiltration at 0.5 air changes per hour), not a simple per-m² rule.
- **Ducts**: airflow (CFM) + circuit type → *Velocity OK* badge, standard diameter (in), actual and recommended velocity.
- **CFM per room**: volume + room type (air changes per hour) → required airflow, ACH.
- **HRV/ERV (ASHRAE 62.2)**: area, number of bedrooms, number of occupants → recommended airflow, unit size, minimum 62.2 airflow.
- **Cooling (solar gains)**: glazing area, orientation, solar heat gain coefficient (SHGC), radiation, occupants, equipment → total gain (BTU/h), tonnage.

### 3.9 Welding (Welding calculator)

**3 sub-tabs**: *Fillet weld · Heat input · CE preheat*.

- **Fillet weld (CSA W47.1)**: joint type, thickness, length, process (SMAW / GMAW / FCAW / GTAW / SAW) → throat (`0.707 × leg`), leg, volume, deposited weld-metal weight, electrode consumption (with a loss factor), deposition rate.
- **Heat input (kJ/mm)**: voltage, amperage, speed, process → net heat input (accounting for arc efficiency) and evaluations for carbon steel and for stainless/aluminum.
- **Preheat (IIW carbon equivalent)**: composition (% C, Mn, Cr, Mo, V, Ni, Cu) + thickness → carbon equivalent, cracking risk, recommended preheat temperature, and the formula used, displayed.

### 3.10 Metal bending (Metal Bending calculator)

**3 sub-tabs**: *Development + tonnage · Springback · Minimum radius*.

- **Development + tonnage (air bending)**: length, thickness, angle, inside radius, width, material → developed length, K-factor, bend allowance and bend deduction, required tonnage (kN), V-die opening, minimum radius, springback. An **alert** appears if the requested radius risks cracking the material. Tonnage follows the formula `1.42 · UTS · t² · L / (V · 1000)`.
- **Springback**: desired angle + material → angle to bend (compensated) and springback.
- **Minimum radius**: thickness + material → minimum radius (mm and in) and factor.

### 3.11 Metal weight (Metal Weight calculator)

A single form. Choose the **shape** (plate, round tube, square tube, round bar, square bar, L-angle, I-beam, AISC W-shape, C-channel UPN), the **material** (about twenty, loaded from the server) and the **dimensions** (fields adapt to the shape). For W and C shapes, a section selector replaces manual dimensions. **Outputs**: weight (kg and lb), volume, density, estimated cost (CAD), section designation.

### 3.12 Quebec Taxes (Quebec Taxes calculator)

Enter the **pre-tax amount** ($) → **Calculate** → net amount, **GST (5%)**, **QST (9.975%)**, gross total.

### 3.13 Employee Payroll (Employee Payroll calculator)

Enter the **gross wage** and the **employee type** (*Regular* or *CCQ Construction*) → two blocks:

- **Employee deductions**: RRQ (Quebec Pension Plan), RQAP (Quebec Parental Insurance Plan), AE (Employment Insurance), total;
- **Employer contributions**: CNESST, FSS (Health Services Fund), CCQ (construction-industry levy, if applicable), total;
- plus the **net pay** and the **total employer cost**.

> The rates come from the **central payroll module** (`payroll_rates`), synchronized with the *Time Tracking and Payroll* module to avoid discrepancies. Watch for the **simplifications**: federal and provincial income tax is estimated at 15% / 15% (not the real progressive brackets), the CNESST is a **default flat rate** (the real rate depends on the trade's risk class), and the CCQ contribution is a **flat indicative rate** (the real contribution is in $/hour per trade). This calculator gives an **order of magnitude**, not an official payroll.

### 3.14 Design a wall and send it to a quote (Parametric Walls)

A **light wall framing** tool in **imperial** units, computed **browser-side** (inspired by the *Wall / Rake / Tall Wall Builder* generators). Explicitly marked **"In development"**: useful, but still evolving.

- **Project bar**: project name (with an identifier), **New**, **Load** (saved projects), **Save** buttons, plus **Send to quote** (creates one or more quote lines) and **Create Takeoff BOM** (produces an assembly for the Takeoff module).
- **Modes**: Standard / Gable / Tall Wall.
- **Views**: Front / Back / **3D**, with undo/redo (up to 50 steps) and zoom.
- **4 tabs**: **Wall** (length, stud size 2×4/2×6/2×8, heights, spacing, first stud, direction, doubling, blocking, top plate, sheathing + a collapsible **Quebec general-contractor compliance** panel); **Openings** (window/door, rectangular or arched shape, position, width, height, sill height); **Cut** (cut list exportable to **CSV** and **PDF**); **Details** (king studs, jack studs, cripple studs, headers, plates, blocking).

**Example:**

1. Open **Calculators** → **Parametric Walls (In development)**.
2. Set the wall properties (**Wall** tab): length, stud size and height, spacing.
3. Add the necessary openings (**Openings** tab).
4. Check the cut list (**Cut** tab) and export it to CSV or PDF if needed.
5. Click **Save** to keep the project (it will appear in the History tab).
6. Click **Send to quote** to transfer the quantities into a bid, **or Create Takeoff BOM** to feed an assembly in the Takeoff module.

### 3.15 Use the AI Assistant

1. **AI Assistant** tab.
2. Choose the sub-tool: **Expert Chat**, **Recommendations**, **Explain a standard** or **Diagnosis**.
3. For the chat: choose a calculator (optional), type the question, **Send**. For the others: fill in the fields (goal, standard, problem/symptoms as applicable).
4. Read the answer. Each call debits the AI credits at actual cost × 1.30. If you leave the tab during generation, the call is canceled and **nothing is billed**.

### 3.16 View and manage history

1. **History** tab.
2. Filter by calculator if needed.
3. On a row, click **Details** to see the inputs and results.
4. Delete an entry (trash icon) or clear everything (**Clear all**, with confirmation).

> Reminder: today, it is mainly the **Parametric Walls** projects that populate this list (see 2.7).

---

## 4. Reference

### 4.1 The 59 endpoints (prefix `/api/erp/v1/calculators`)

All require an authenticated session. The calculation endpoints are `POST`; the reference data and history combine `GET`, `POST` and `DELETE`.

| Domain | Endpoints |
|---|---|
| **Concrete** (8) | `/concrete`, `/concrete/dosage`, `/concrete/rebar`, `/concrete/cure`, `/concrete/formwork`, `/concrete/excavation`, `/concrete/talus`, `/concrete/stairs` |
| **Stairs** (3) | `/stairs`, `/stairs/materials`, `/stairs/garde-corps` |
| **Electrical** (4) | `/electrical`, `/electrical/residential`, `/electrical/lighting`, `/electrical/grounding` |
| **Roofing** (4) | `/roofing`, `/roofing/ventilation`, `/roofing/gutters`, `/roofing/snow-load` |
| **Painting** (3) | `/painting`, `/painting/dft`, `/painting/dew-point` |
| **Plumbing** (4) | `/plumbing`, `/plumbing/hazen-williams`, `/plumbing/water-heater`, `/plumbing/drain-slope` |
| **HVAC** (5) | `/hvac`, `/hvac/duct`, `/hvac/cfm`, `/hvac/hrv`, `/hvac/cooling` |
| **Welding** (4) | `/welding`, `/welding/heat-input`, `/welding/preheat`, `/welding/consumable` |
| **Bending** (3) | `/bending`, `/bending/springback`, `/bending/min-radius` |
| **Metal weight** (1) | `/metal-weight` |
| **Taxes** (1) | `/taxes` |
| **Payroll** (1) | `/charge-tributaire` |
| **Structural Analysis** (3) | `POST /charge-tributaire-complete`, `GET /charge-tributaire-complete/materials`, `GET /charge-tributaire-complete/snow-loads` |
| **Conversions** (1) | `GET /conversions` (reference data — no dedicated tab in the interface) |
| **History** (5) | `GET /history`, `POST /history`, `DELETE /history/{id}`, `DELETE /history`, `GET /history/stats` |
| **AI Assistant** (6) | `/ai/chat`, `/ai/analyze`, `/ai/recommend`, `/ai/explain-norm`, `/ai/diagnose`, `/ai/optimize` |
| **References** (3) | `GET /constants`, `GET /resources`, `GET /` (list of calculators) |

**Total: 59 endpoints.** Breakdown: 42 calculation `POST` + 2 structural-data `GET` + 1 conversions `GET` + 5 history + 6 AI + 3 references.

> Two server endpoints have **no button** in the interface: `/concrete/formwork` (formwork is already given in the *Volume* result) and `/ai/analyze` + `/ai/optimize` (2 of the 6 AI functions). They exist but are not reachable from the current screen.

### 4.2 Applied standards and formulas (actual code behavior)

| Standard / method | Calculator | What the code applies |
|---|---|---|
| **CSA A23.1** | Concrete | Mixes 15 to 40 MPa based on the exposure class |
| **CSA G30.18** | Rebar | 10M to 55M bars, linear masses, 6 m cutting |
| **ACI 209** | Concrete curing | `f(t) = f28 · t / (a + b·t)` + maturity factor by temperature |
| **CNESST** | Excavation slope | H:V slopes by soil type + requirements at the 1.2 / 3 / 6 m thresholds |
| **Blondel** | Stairs | `2R + G` within the comfort range |
| **CCQ 9.8 / 3.4** | Stairs | Riser, run, width (residential / commercial) |
| **CCQ 9.8.7** | Guardrail | Height, baluster spacing, handrail |
| **CCQ 9.19.1** | Attic ventilation | 1:300 with a vapor barrier, 1:150 without |
| **CCQ 9.14.6** | Gutters | Drainage capacity by size |
| **CNB 4.1.6.2** | Snow load | `S = Is · [Ss · (Cb·Cw·Cs·Ca) + Sr]` (full formula) |
| **CNB (combinations)** | Structural Analysis | `1.4D` / `1.25D+1.5L` / `+1.5S` |
| **CSA O86** | Structural Analysis (wood) | `Mr = φ·Fb·S·Kd·Kl` and `Vr`, with **φ = 0.9**, **Kd = Kl = 1.0**, Kz omitted (conservative) |
| **CCE 4-004** | Cable | Cross-section = max(voltage drop, ampacity × 1.25) |
| **CCE 8-200** | Residential load | Base 5 kW + 1 kW per 90 m² |
| **CNP** | Plumbing | DFU / WSFU, drain diameter by load |
| **Hazen-Williams** | Plumbing | Head loss `hf = 10.44·L·Q^1.852 / (C^1.852·d^4.87)` |
| **ASHRAE (U·A·ΔT)** | HVAC | Envelope + infiltration 0.5 ACH |
| **ASHRAE 62.2** | HVAC (HRV/ERV) | Minimum residential ventilation airflow |
| **CSA W47.1 / W59 / AWS D1.1** | Welding | Throat `0.707 × leg`, net heat input (arc efficiency) |
| **IIW** | Welding | `CE = C + Mn/6 + (Cr+Mo+V)/5 + (Ni+Cu)/15` |
| **Magnus** | Painting | Dew point |
| **AISC / CISC** (Canadian Institute of Steel Construction) | Metal weight | Library of W / C shapes |
| **GST / QST** | Taxes | 5% + 9.975% |
| **RRQ / RQAP / AE / CNESST / FSS / CCQ** | Payroll | Rates from the central `payroll_rates` module |

### 4.3 Useful HTTP response codes

| Code | Meaning for the user |
|---|---|
| 200 | Calculation successful |
| 400 | Invalid input (e.g., slab too thin for rebar, unknown shape section, **"Column" type refused**, unknown snow city) or **missing tenant context** for the history |
| 402 | AI credits exhausted (AI Assistant) |
| 403 | AI disabled for the tenant |
| 404 | History item not found on deletion |
| 413 | AI request too large |
| 502 | Unusable AI response |
| 503 | AI service unavailable |

### 4.4 Limits and known simplifications

The calculator **always produces a result**, but some models are deliberately simplified. Good to know:

| Topic | Current behavior |
|---|---|
| **Structural Analysis (wood)** | φ = 0.9 but Kd = Kl = 1.0 and size effect (Kz) omitted → **conservative** preliminary sizing, to be validated by an engineer |
| **Column / compression** | Not implemented: the option is not offered and is refused server-side (error 400) |
| **Lintel** | Treated as a beam (deflection L/360), without a dedicated bearing/support check |
| **Structural steel (CSA S16)** | Not implemented: only wood (CSA O86) is computed in structural analysis |
| **Seismic loads** | Not included (no spectrum, no ductility check) |
| **Payroll income tax** | 15% / 15% flat, not the real progressive brackets |
| **CNESST** | Default flat rate; the real rate depends on the trade's risk class |
| **CCQ (payroll)** | Flat indicative rate; the real contribution is in $/hour per trade |
| **Snow load** | Default value if the city is not listed |
| **Bending** | One bend per calculation (add up for a multi-bend part) |
| **CAD prices** | Indicative (internal reference tables); for a bid, use the supplier prices of the *Store / Inventory* module |

> **Good to know**: two formerly simplified models are **now complete** — the **snow load** applies the full CNB 4.1.6.2 formula, and the **HVAC load** uses the U·A·ΔT envelope method (with infiltration). Older internal notes that listed them "to be reworked" are outdated.

### 4.5 Rate limiting and availability

- The **6 AI endpoints** (`/ai/*`) are limited to **10 requests per minute** (per address). The ~50 other calculation endpoints fall back on the general, much higher limit (1,500/min).
- All calculations run **server-side** (except the Calculator and the Parametric Walls): a network connection and a valid session are required.

### 4.6 Keyboard shortcuts (Calculator)

Valid in the **Calculator** tile (`master-pro`):

| Key | Action |
|---|---|
| `0`–`9` | Digit entry |
| `+ − × ÷` | Operators |
| `Enter` | Equals (=) |
| `Esc` | Clear all |
| `f` | Feet |
| `i` | Inches |
| `y` | Yards |
| `c` | Secondary function (*Conv / Shift*) |

### 4.7 History persistence

Table `calculator_history`, created **on demand** in the tenant's schema on first use (columns: `id`, `calculator_id`, `subcalc_id`, `label`, `inputs` JSONB, `results` JSONB, `notes`, `user_id`, `created_at`). Two indexes speed up reads. No automatic purge: entries remain until manually deleted.

---

## 5. Integrations and FAQ

### 5.1 Integrations with other modules

| Module | Link | Detail |
|---|---|---|
| **Quotes / Bids** | Partial | The **Parametric Walls** have a **Send to quote** button that creates lines. The 13 server-side calculators, however, **do not inject** automatically: you copy the results by hand. |
| **Takeoff** | Partial | The **Parametric Walls** can **Create a Takeoff BOM** (assembly). |
| **Store / Inventory** | None | The calculators neither consume nor reserve stock; quantities are transcribed manually. Use this module's real supplier prices rather than the indicative prices. |
| **Purchase orders** | None | Copy the material quantities by hand into the order. |
| **Accounting / Payroll** | None (informational) | The *Taxes* and *Employee Payroll* calculators give indicative figures; they create no invoice or entry. For real payroll, see the *Time Tracking and Payroll* module. |
| **AI Assistant** | Shared credits | The calculators' AI draws on the **same prepaid AI credit wallet** as the other AI modules. |

### 5.2 FAQ

**Are the results worth a signed engineer's calculation?**
No. These are **estimating and preliminary-sizing** tools. For a permit or a sealed plan, an engineer (OIQ — Ordre des ingénieurs du Québec, Quebec's professional engineers order) must validate and sign. Structural Analysis is especially conservative (Kd = Kl = 1.0).

**Why does the dashboard show 15 when we sometimes speak of 13 calculators?**
The **15** count the visible tiles (13 server-side + the Calculator + the Parametric Walls). The **13** count only the server-side calculators. Both figures are correct.

**Does the Calculator work offline?**
Yes, once the page is loaded: it calculates in the browser, with no server call. Same for the calculation side of the Parametric Walls (saving a project, however, requires the network). The 13 other calculators do need the server.

**Where did the Conversions go?**
There is no longer a separate "Conversions" tab. Unit conversions are done in the **Calculator**: internal *Conversions* tab and conversion sheet (m, cm, mm, ft, in, yd).

**Why doesn't my concrete calculation appear in the History?**
Because the 13 server-side calculators have no wired "Save" button in the current interface: their results are ephemeral. Only the **Parametric Walls** projects are saved to the history. To keep a calculation, note it down or copy it into a quote.

**Can I analyze a column (compression, buckling)?**
No. Structural Analysis only offers **Beam** and **Lintel**. The column mode is not implemented and is refused.

**What happens if the AI service is down?**
The 4 AI sub-tools return a 503 error, but **all the other calculators keep working** (they are deterministic, AI-free).

**Can the CCQ / CNESST rates be customized per company?**
Not in this version. The payroll rates come from the central `payroll_rates` module (current year). For customization, contact the Constructo administrator.

**How do I export a calculation to PDF for a client?**
There is no official PDF export for the 13 calculators (use the browser's print, or the History tab → Details). The **Parametric Walls**, on the other hand, export their cut list to **CSV** and **PDF**.

**Does the welding calculator prescribe non-destructive testing?**
No. It sizes the joint (throat, leg, consumption, preheat) but does not prescribe the inspections (radiography, dye penetrant, magnetic particle). Follow CSA W59 / AWS D1.1 according to the class of work.

**Are the floor / cladding / house-designer calculators available?**
No. Seven components of that kind exist in the code but **are connected to no screen**: they are inaccessible today. The only active roofing panel is the **Roofing** calculator described in 3.7.

**Are the snow loads up to date with the latest edition of the CNB?**
The ground-snow values by city are built into the code (article 4.1.6.2). Verify the edition applicable to your project and request an update if a new edition changes the values.

---

## 6. Summary

- **Module**: a suite of **15** construction-calculation **tiles** (Quebec) = **13 server-side calculators** + **2 browser-side tools** (Calculator, Parametric Walls), plus an AI Assistant and a per-tenant history.
- **5 tabs**: Dashboard · Calculators · Structural Analysis · AI Assistant · History. **No** Conversions tab (conversions live in the Calculator).
- **6 categories**: Multi-purpose (1) · Structure (4) · Envelope (2) · Mechanical (3) · Metal (3) · Finance (2).
- **59 endpoints** under `/api/erp/v1/calculators`: 42 calculations + 2 structural data + 1 conversions + 5 history + 6 AI + 3 references.
- **Access**: side menu → Tools → Calculators. **No particular role required**; the history is shared per tenant; the AI requires prepaid credits.
- **Concrete**: volume, CSA A23.1 mix, rebar, ACI 209 curing, excavation, CNESST slope, Blondel stairs (7 sub-tabs).
- **Stairs**: CCQ 9.8/3.4 dimensions + 9.8.7 guardrail.
- **Structural Analysis**: Beam / Lintel, SPF wood or LVL, CNB combinations, CSA O86 Mr/Vr, deflection L/360 or L/180, diagram. **Conservative** (Kd = Kl = 1.0), **no column**.
- **Roofing**: area/shingles, ventilation 1:300/1:150, gutters, snow load **full CNB 4.1.6.2**.
- **Painting**: quantity + cost (GST/QST), dry film thickness, Magnus dew point.
- **Electrical**: AWG cable (CCE 4-004), residential load (CCE 8-200), lighting, grounding.
- **Plumbing**: DFU/WSFU (CNP), Hazen-Williams, water heater, drain slope.
- **HVAC**: **U·A·ΔT** heat load (ASHRAE), ducts, CFM/room, HRV/ERV (62.2), cooling.
- **Welding**: fillet (CSA W47.1), heat input, preheat (IIW carbon equivalent).
- **Bending**: development + tonnage (air bending), springback, minimum radius.
- **Metal weight**: 9 shapes × ~20 materials + AISC W/C shapes.
- **Taxes**: GST 5% + QST 9.975%. **Payroll**: employee deductions + employer contributions (central-module rates), with simplified income tax and CNESST.
- **Calculator**: feet-inches jobsite calculator (Construction Master Pro style), 100% local, with conversions.
- **Parametric Walls (In development)**: imperial wall framing, 3D view, CSV/PDF cut list, **Send to quote** and **Create Takeoff BOM**.
- **AI Assistant**: **4 sub-tools** (Chat, Recommendations, Explain a standard, Diagnosis) on **Claude Opus 4.8** (32,000 tokens), billed at actual cost × 1.30 on prepaid credits; canceled free of charge if you leave the tab.
- **Limits**: preliminary sizing only, no engineer's seal, no structural steel, no column, indicative payroll, indicative prices.

---

**Verified source files**: `frontend/src/pages/CalculateursPage.tsx` (2,269 lines), `frontend/src/components/calculateurs/MasterProCalculatorBody.tsx` (903 lines), `frontend/src/components/calculateurs/MursParametriquePanel.tsx` (2,676 lines), `frontend/src/api/calculators.ts` (1,642 lines), `backend/routers/calculators.py` (3,681 lines, 59 endpoints), `backend/routers/calculators_data.py` (807 lines).

**Related manuals**:
- Module 07 — Quotes (copy or receive quantities) — `07-ventes-soumissions.md`
- Module 09 — Store / Inventory (real supplier prices) — `09-operations-magasin.md`
- Module 12 — Time Tracking (real payroll vs indicative calculation) — `12-operations-pointage.md`
- Module 13 — Purchase Orders (buying materials) — `13-operations-bons-de-commande.md`
- Module 24 — AI Assistant (prepaid AI credits) — `24-communication-assistant-ia.md`
- Module 32 — Takeoff (BOM assemblies, quantity takeoff) — `32-metre-pdf.md`
- Module 25 — 3D Modeling / CAD — `25-outils-dao-modelisation.md`
