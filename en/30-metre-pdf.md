# Module 30 — Takeoff (quantity takeoff on PDF)

> **Version**: 2.0 (fully rewritten from the actual source code, revision 2026-07)
> **Access**: the module has **no** entry in the left sidebar and **no** `/metre` address. You open it through **Sales → Quotes → "Takeoff" tab** (a lazy-loaded tab `metre-pdf`, `DevisPage.tsx:49,1361,1472`). So you must first open the Quotes page.
> **Reference code**:
> - Frontend: the `frontend/src/components/metre-pdf/` folder (~42,250 TS/TSX lines) — main component `MetrePdf.tsx`, state manager `store.ts`, drawing canvas `components/MeasurementCanvas.tsx` (**Fabric.js** library), plan viewer `components/PDFViewer.tsx` (**pdf.js**), bars and panels (`TopToolbar`, `LeftPanel`, `RightPanel`, `BomEstimationPanel`, `BottomBar`, `MetreSavedBar`, `PageNavigator`, `AiCountPanel`, `MetreAssistantPanel`), `utils/` helpers (including `canvasRotation.ts` for measuring on a rotated plan), API client `components/metre-pdf/api.ts` (prefix `/api/erp/v1/metre`) and `components/metre-pdf/api/metreAi.ts`.
> - Backend: `backend/routers/metre_pdf.py` (4,777 lines, **45 endpoints**; persistence + rendering + exports), `backend/routers/metre_ai_chat.py` (2,154 lines, **1 endpoint**; streaming assistant chat), `backend/routers/metre_ai_conversations.py` (1,075 lines, **4 endpoints**; conversations + action confirmation), `backend/routers/metre_ai_detect.py` (291 lines, **1 endpoint**; AI vision auto-count), `backend/routers/metre_ai_tools.py` (2,625 lines; the assistant's **19 tools**, not a router). **51 endpoints in total**, common actual prefix `/api/erp/v1/metre`.
> **PostgreSQL tables (per company, created on demand)**: `metre_projects`, `metre_documents` (the PDF is stored as **BYTEA**), `metre_calibrations`, `metre_layers`, `metre_layer_groups`, `metre_measurements` (JSONB columns `points` and `metadata_json`), `metre_products`, `metre_product_components`, `metre_ai_detections`, `metre_ai_conversations`, `metre_ai_messages`, `metre_ai_tool_executions` (12 tables). They are created **the first time** you use the module (lazy `_ensure_tables` mechanism, `metre_pdf.py:1222`), not when the account is created.
> **Scope**: this module is a **quantity-takeoff tool on a PDF or image plan**. You load a plan, **calibrate the scale page by page**, **draw measurements** (distances, areas, perimeters, counts, angles, circles, plus annotations), **associate them with catalog products** (materials + CCQ (Commission de la construction du Québec) labor) or with **parametric assemblies (BOM)**, then **generate a bid** or exportable **schedules** (CSV, PDF, HTML, DXF, PNG) that you can **send straight back into a quote**. A **conversational AI assistant** reads the takeoff and suggests actions, and an **AI auto-count** counts the repeating elements in an area. This module is **not** 3D modeling software (see module **31 — CAD / 3D Modeling**), it **does not replace** the Quotes module (it feeds it), and it **does not** fully read the plan automatically (only OpenCV snapping and the AI auto-count over a drawn area assist data entry).

*Terminology used in this manual:* "endpoint" means an API endpoint; "company" or "tenant" means your account (each company's data is isolated by a PostgreSQL schema); "takeoff" means a quantity-takeoff project (a plan + its measurements + its layers + its calibrations); "document" means the loaded PDF or image file; "calibration" means the link between the plan's pixels and real-world units; "measurement" means a priced drawing; "layer" groups measurements; "composite" or "assembly (BOM)" means a product made of sub-products computed by formula; "bid" or "schedule" means the priced output. "Upload" = send a file to the server; "download" = retrieve a file from the server.

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

Turn a **PDF plan** (or an image) into **priced quantities**. In concrete terms, you:

- **load** a plan on screen (one or more pages);
- **calibrate** each page's scale (you draw a line over a known dimension and enter its true length);
- **draw** your measurements over the plan (walls, areas, perimeters, counts, etc.) — the module automatically computes the real lengths and areas;
- **organize** your measurements into **layers** (for example "Basement walls", "Ground-floor slab") and **groups**;
- **associate** a **product** (price, unit, waste percentage) and, if needed, **labor** (CCQ trade) with each measurement, or link a layer to a **parametric assembly (BOM)** that fills itself in;
- **produce** a **bid** (materials + labor + taxes) and **schedules** that you **export** (CSV, PDF, HTML, DXF, PNG) or **send back into a quote**, whether existing or new.

An **AI assistant** can read your takeoff and run actions (create a layer, search a product, etc.) after your confirmation, and an **AI counting tool** counts the repeating elements in an area you box in.

### 1.2 How to access it

The Takeoff is **not** a standalone page. It is a **tab** of the Quotes page.

| Step | Detail |
|-------|--------|
| 1 | Left sidebar → **Sales** → **Quotes** (the Quotes page, `DevisPage.tsx`). |
| 2 | In the page's tab bar (Quote · AI Estimation · **Takeoff** · Manual · Terms · Assistant), click **"Takeoff"** (`DevisPage.tsx:1361`, label `devis.tabs.metre`). |
| 3 | The module loads on demand (a brief "Loading Takeoff..." message shows while the module loads). |

Practical consequences of this integration:

- The module **receives the context of the quote** currently selected on the page. If a quote is open, a **blue "Connected quote" banner** shows it and the takeoff items can be added to it. Otherwise, a collapsible **client card** appears to pre-fill a future bid.
- Closing the tab (the module's **X** button) **collapses** the Takeoff and unmounts it; your work is kept on the server (see §3.15).

### 1.3 Roles and permissions

- **No module-specific role restriction.** Every Takeoff endpoint requires only that you be an **authenticated user of your company** (`_require_tenant` guard, `metre_pdf.py:44`). There is **no "administrator", "accountant" or "super-administrator" guard**: any tenant member has **full control** (create, measure, price, manage catalogs, export, delete).
- **Strict per-company isolation.** Your takeoffs, your catalogs (products, labor) and your assemblies are shared **within your company**, never visible to another (PostgreSQL schema isolation).
- **View-only mode (read-only).** If your company's subscription is suspended, the ERP switches to view-only mode at the global authentication level: writes are then blocked. This behavior is not overridden in the module; it is inherited from the account.
- **AI assistant and credits.** The conversational assistant and the auto-count consume **AI credits** billed to your company (see §1.7). The platform's super-administrator is exempt.

### 1.4 Key concepts

- **Takeoff (project)**: the unit of work. Groups one or more **documents**, all the **measurements**, the **layers**, the **calibrations** and an optional **link to a quote**. Table `metre_projects`.
- **Document**: the loaded plan (PDF or image). Stored in the database as **BYTEA** (it survives server redeployments). Multi-page documents are supported. Table `metre_documents`.
- **Calibration**: the pixels ↔ reality link, defined **per page**. You draw a line over a known dimension and enter its real length; the module derives the **scale factor** (real unit per pixel). Table `metre_calibrations` (one per page, uniqueness constraint on document + page).
- **Measurement**: a typed drawing (distance, area, count, etc.) with a **value** computed in its page's **calibration unit**. Each measurement carries points (JSONB), a color, a layer, an optional product, a slope, deductions and labor. Table `metre_measurements`.
- **Layer**: groups measurements and carries a color, a visibility state and a lock. A layer can be **linked to an assembly (composite)** to feed a BOM. Table `metre_layers`.
- **Layer group**: a collapsible section that groups several layers. Table `metre_layer_groups`.
- **Product**: an item from the **catalog** (price, price unit, waste percentage). **Simple** or **composite**. Table `metre_products`.
- **Composite / assembly (BOM)**: a product made of **sub-products** whose quantity comes from a **parametric formula** evaluated over **input variables** (fed by the measurements). Table `metre_product_components`.
- **Deduction**: a "subtracting" measurement (for example an opening) attached to a parent measurement; the **net** quantity = parent − sum of deductions (never negative).
- **Bid / schedule**: the priced output (materials + labor + taxes), exportable or convertible into a quote.

### 1.5 Measurement types

| Tool | Shortcut | Purpose | Computed value |
|---|---|---|---|
| **Distance** | D | Length of a segment (2 points) | length (linear unit) |
| **Area** | A | Area of a polygon (shoelace formula) | area (unit²) |
| **Rectangle** | R | Area of a rectangle (2 corners) | area |
| **Perimeter** | P | Closed outline | outline length |
| **Polyline** | L | Open chain of segments | sum of lengths |
| **Angle** | N | Angle between 3 points | degrees |
| **Count** | C | Number of elements (one click per element) | count |
| **AI Count** | U | Automatic count over a boxed area (see §3.11) | count |
| **Circle** | I | Area of a disk (center + radius) | area (π·r²) |
| **Dimension** | X | Annotated architectural dimension | length |
| **Wall (keyboard entry)** | J | Segment placed at an exact keyboard-entered length (see §3.4) | length |

**Annotations** (not priced): Text (T), Arrow (W), Revision cloud (Q), Freehand (F), Highlighter (G), Note (E), Text bubble (B), plus the **architectural symbols** (dedicated catalog). They document the plan, they do not price it.

Tool source: `TopToolbar.tsx:63-87`.

### 1.6 Units and imperial entry

- **Calibration / measurement units**: `m`, `cm`, `mm`, `ft` (feet), `in` (inches). These are the five units accepted at calibration (`metre_pdf.py`, `CalibrationCreate.unit`) and for measurements created by the AI assistant (`ALLOWED_MEASUREMENT_UNITS`, `metre_ai_tools.py:63`).
- **Imperial entry**: feet-inches-sixteenths format. Recognized examples: `20-06-08` (20 ft 6 in 8/16), `200608` (same value, no dashes), or a decimal value (`3.048`). Imperial detection is automatic and shows the interpretation.
- **Product price units**: `pi` / `m` (linear), `pi²` / `m²` / `vg²` (area), `m³` (volume), `unité` / `feuille` / `sac` / `boîte` (count), `heure`. See the conversion table in §4.4.
- **Automatic measurement ↔ price reconciliation** (the money path): if your plan is calibrated in **meters** but the product is priced per **pi²** (sq ft), the quantity is **converted** before pricing. The factor follows the dimension: linear → linear = the conversion factor; area → area = the factor squared; **cross-dimension** (for example an area linked to a linear price, or a volume) → the factor stays **1** and the measurement is flagged **"incompatible unit"** (quantity not converted, warning shown). This conversion fixes the classic ×10.76 under-billing when a metric plan feeds a per-sq-ft catalog (`_billable_factor`, `metre_pdf.py:3870`).

### 1.7 AI models and billing

Two functions consume your company's **AI credits**, rebilled at the **actual cost plus 30% (× 1.30)**, with a **cap of US$2.00 per call**:

| Function | Model | Type | Ref. |
|---|---|---|---|
| **Takeoff Assistant** (chat) | Claude Opus 4.8 (`claude-opus-4-8`) | Streaming response, tool loop (max 8 iterations), cap of 200 messages / conversation | `metre_ai_chat.py:112` |
| **AI Count** (vision) | Claude Opus 4.8 (`claude-opus-4-8`) | Occurrence detection over a boxed area, no database write | `metre_ai_detect.py:47` |

- **Snap-point detection** uses **OpenCV** on the server; it is local processing, **with no AI cost**.
- **Confirming a write action** from the assistant carries a **fixed, nominal cost** of US$0.001 (`_METRE_TOOL_CONFIRM_COST_USD`, `metre_ai_conversations.py:67`).
- The API key stays **on the server**; the client only sends its authentication token. In self-hosted mode (billing disabled) or for a super-administrator, these calls remain **free but tracked** (`track_ai_usage`).

### 1.8 What the module does — and does not do

The module **does**: load a PDF/image plan, calibrate per page, draw 11 measurement types + 7 annotations + symbols, organize into layers/groups, associate products and labor, build parametric BOM assemblies, count with AI, chat with an AI assistant, generate a bid, export (CSV, PDF, HTML, DXF, PNG) and feed a quote.

The module **does not**:

- **No `/metre` route and no menu entry.** Access is **only** through the "Takeoff" tab of the Quotes page.
- **No AI assistant until the plan is saved.** The assistant button stays **grayed out** until the document has a server ID (see §3.12). One active conversation per document.
- **No auto-count on a rotated plan.** The plan must be **straight (0°)**; otherwise a message prompts you to reset the rotation.
- **No automatic correction of doubtful quantities.** Incompatible units, self-crossing outlines ("self-intersecting"), and formula-based composites linked live are **flagged** but not corrected.
- **No assemblies/BOM outside ERP mode.** Composites are available only when the module is connected to the ERP (message `productCatalog.add.compositeOnlyInErp`).
- **The "Automatic BOM bid" does not include** the manual values entered in the BOM panel, and does **not** generate labor lines.
- **Full screen is display-only** (CSS). The **Esc** key does not exit it (it cancels the tool or deselects).
- **The direct server-side "takeoff → quote" import is frozen.** The `import-to-devis` endpoint returns 503 by default; the real path goes through the **Bid modal** (see §3.13 and §4.2).

### 1.9 Sub-modules and panels

| Element | Component | Purpose |
|---|---|---|
| **"Saved takeoff" bar** | `MetreSavedBar` | Name of the active takeoff, save state, open / new / rename / close. |
| **Toolbar** | `TopToolbar` | File, measurement and annotation tools, transforms, zoom, drawing aids, catalogs, AI, exports, full screen, undo/redo. |
| **Left panel** | `LeftPanel` | Layers, measurements (per page), consolidated (all pages). |
| **Center area** | `PDFViewer` + `MeasurementCanvas` | Plan + drawing layer (Fabric.js) + zoom + page navigation + AI auto-count overlay. |
| **Right panel** | `RightPanel` | Measurement properties, calibration, catalogs, cost summary, document. |
| **BOM panel** | `BomEstimationPanel` | Live estimate of assemblies, schedules, CSV exports. |
| **Bottom bar** | `BottomBar` | Active tool, cursor position, live value, units, calibration state. |
| **AI Assistant** | `MetreAssistantPanel` | Chat, conversations, profiles, action confirmation. |
| **Auto-count** | `AiCountPanel` | Vision counting over a boxed area. |

---

## 2. Interface

### 2.1 General layout

Top to bottom (`MetrePdf.tsx:1346-1500`):

```
+---------------------------------------------------------------------------+
| "SAVED TAKEOFF" BAR: name · save state · Open · New                        |
+---------------------------------------------------------------------------+
| "Connected quote: {name}" BANNER   OR   CLIENT CARD (collapsible)          |
+---------------------------------------------------------------------------+
| TOOLBAR: File | Measurements | Annotations | Transform | Zoom |            |
|          Drawing aids | Catalogs | AI | Exports | Full screen              |
+----------+------------------------------------------+----------+-----------+
| LEFT     |   VIEWER (PDF/image plan)                | RIGHT    |  BOM      |
| PANEL    |   + measurement canvas (Fabric.js)       | PANEL    |  PANEL    |
| Layers   |   + zoom / rotation controls             | Proper-  |  (collap- |
| Measure- |   + AI Auto-count overlay                | ties     |  sed by   |
| ments    |                                          | Costs    |  default) |
| Consolid.|                                          |          |           |
+----------+------------------------------------------+----------+-----------+
| PAGE NAVIGATION: <  page N / total  >                                      |
+---------------------------------------------------------------------------+
| BOTTOM BAR: tool · cursor · value · SNAP · units · calibration             |
+---------------------------------------------------------------------------+
```

The rendering stacks **three layers**: the plan (rendered by **pdf.js**, cached at high resolution), the measurements (drawn by **Fabric.js**, `MeasurementCanvas.tsx:4`), and an interaction layer that captures the mouse (snapping, click, drag). The module handles **measuring on a rotated plan**: if you turn the plan, the drawings stay consistent thanks to a rotation matrix (`utils/canvasRotation.ts`).

### 2.2 "Saved takeoff" bar (`MetreSavedBar`)

Two states.

- **No takeoff open**: an alert icon, the message "**No takeoff open.** Create or open a takeoff...", and two buttons: **"Open"** and **"New takeoff"**.
- **Active takeoff**: a pulsing dot, the "TAKEOFF" label, the takeoff's **name** and description, then on the right:
  - a **counter** "{n} measurements";
  - a **save indicator** refreshed every 30 seconds: "Saved", "Saved N s / min / h ago" or "Saved on {date}"; or a red **"Save failed"** badge (clickable to dismiss); or "PDF not loaded";
  - actions: **pencil** (rename), **"Open"** (another takeoff), **"New"**, **X** (close).

### 2.3 Quote banner / client card

- If a quote is open in the Quotes page: a blue banner **"Connected quote: {name} — the takeoff items will be added to this quote"** (`MetrePdf.tsx:1356`).
- Otherwise: a collapsible **client card** (`ClientInfoCard`) that pre-fills the future bid (company client, contact; lists loaded from the ERP).

### 2.4 Top toolbar (`TopToolbar`)

Groups are separated by thin vertical rules. Each tool shows its **shortcut** in parentheses in its tooltip.

**1. File.**
- **Open a plan — PDF or image (Ctrl+O)**: accepts `.pdf, .png, .jpg, .jpeg, .bmp, .tiff, .tif, .webp` (`TopToolbar.tsx:272`).
- **New blank plan (ARCH D 36"×24")**: generates a blank ARCH D PDF to draw freehand or paste a screenshot.

**2. Measurement tools**: Calibrate (K), Select (V), Distance (D), Area (A), Rectangle (R), Perimeter (P), Polyline (L), Angle (N), Count (C), **AI Count (U)**, Circle (I), Dimension (X), Pan / hand (H).

**3. Annotation tools**: Text (T), Arrow (W), Revision cloud (Q), Freehand (F), Highlighter (G), Note (E), Text bubble (B).

**4. Transform** (active only when there is a selection): **Rotate 45°**, **Horizontal mirror copy (M)**, **Vertical mirror copy (Shift+M)**.

**5. Zoom**: Zoom out (−), displayed percentage, Zoom in (+), **Fit to page** (clamped between 0.1× and 10×).

**6. Drawing aids** (toggles): **Snap (F3)**, **Ortho mode (F8)**, **Grid (F7)**.

**7. Panels and catalogs** (toggles): **Construction Calculator**, **Slope Converter**, **Product Catalog**, **CCQ trades (labor)**, **Wall — keyboard measurement (J)**, **Architectural Symbols**.

**8. AI Assistant**: a button that opens/closes the chat panel. It is **disabled** until a document with a server ID is loaded (tooltip "Save the takeoff first").

**9. Summary and exports**: **Multi-page summary**, **Export to PNG** (3 files: annotated plan + product detail + BOM detail), **Generate a bid** (measurements with an associated product), **Generate an automatic BOM bid** (layers linked to composites).

**10. On the right**: **Full screen (F11)**, **Undo (Ctrl+Z)**, **Redo (Ctrl+Y)**.

### 2.5 Left panel (`LeftPanel`)

Three collapsible sections.

**A. Layers.**
- Header: **Create a group**, **Add a layer** (inline name entry + OK).
- **Layer groups**: collapsible header, rename (double-click), an eye to "hide/show all layers in the section", delete (with confirmation), a "Drag layers here" drop zone, drag-and-drop of a layer into a group or into "No group".
- **Per layer**: color dot (picker), name (double-click to rename, max 100 characters), measurement counter, **eye** (show/hide), **padlock** (lock/unlock), **BOM link** (opens the "BOM linking" sub-panel), **trash** (delete the layer and its measurements, with confirmation). Dragging a measurement onto a layer moves it.
- **"BOM linking" sub-panel**: a menu to pick a composite ("-- None --" + list); then, for each variable of the assembly: either an **"auto (derived from the drawing)"** or **"auto (layer measurements)"** badge (read-only), or a manual field, or an imperial field (feet-inches-sixteenths).

**B. Measurements (page N).**
- An orange **"{n} to fix"** badge if some measurements have no product (toggles a "show only these" filter).
- Measurements **grouped by layer** (collapsible, state remembered locally). Per measurement: color dot, type icon, label, **formatted value**, **↑ / ↓** arrows (copy to the neighboring layer), **Duplicate** (Ctrl+D), **Move to...** (layer search), **Delete**. Line 2: associated product, or **"No product"** (orange), or **"Product not found"** (product removed from the catalog).
- Selection: click; Ctrl+click (add/remove); Shift+click (range).

**C. Consolidated (all pages).**
- **"Details"** link (opens the multi-page summary).
- Totals **by type** (icon + count + formatted total) and **by product** (name, quantity + unit, cost $), plus an overall **Total**. The cost follows the formula `net value × slope × waste × price`, converted to the price unit.

### 2.6 Center area (viewer)

- **Viewer (`PDFViewer`)**: renders the current page. When empty: "Drag and drop a PDF or an image directly here, or use the button..." + "New blank plan (ARCH D 36"×24")".
- **Measurement canvas (`MeasurementCanvas`, Fabric.js)**: drawing, selection, moving; handles plan rotation.
- **Zoom controls**: Zoom in/out, percentage, Fit to page, **Rotate −90° / +90°**, **Reset rotation**.
- **Auto-count (`AiCountPanel`)**: a purple overlay visible when the "AI Count" tool is active (see §3.11).
- **Page navigation (`PageNavigator`)**: ‹ previous page, an editable "page / total" field, next page › (hidden when there is no document).
- **Wall entry (`MurInput`)**: a small dialog to type a dimension (format `FF-II-SS` or `FFIISS`, for example "20-06-08" or "200608"; the arrows set the direction, Enter confirms, Esc cancels).

### 2.7 Right panel (`RightPanel`)

Collapsed by default (a vertical "Properties" strip). Seven blocks.

**A. Measurement properties** — three modes:
- *No selection*: "Select a measurement to see its properties".
- *Multi-selection*: "{n} measurements selected" + **Selection total** + batch editing (color, stroke width 1-10, group, opacity, slope factor, **mark/unmark deductions**, associated product, paste properties, duplicate, delete).
- *Single selection*: duplicate; copy/paste properties; **display order** (to back / backward / forward / to front); **label** (with BOM suggestions); text content (note/bubble); group; type; **value** (with a "self-intersecting" warning if the outline crosses itself); **points** (P1, P2...); **segments** (per-edge dimensions + total perimeter/length, with a "plan not calibrated" warning); stroke width; font size; rotation + scale (symbol); color (8 choices); opacity; **slope factor (roofing)** (presets Flat, 2:12 ... 12:12 + custom + horizontal → real area computation); **deduction** (checkbox + choice of the parent measurement); a **Gross / − deductions / Net** block; **associated product** (menu grouped by category; cost block: editable gross quantity, net, waste %, price, **total cost $**, "incompatible unit" warning); **labor** (CCQ trade, hours × people → labor cost); **TOTAL (materials + labor)**.

**B. Calibration**: Scale (1 px = X unit), reference, pixels — or "Not calibrated - use the Calibrate tool".

**C. Product Catalog**: counter + **"Manage Catalog"**.

**D. CCQ trades**: counter + **"Manage Labor"**.

**E. Architectural symbols**: counter + **"Manage Symbols"**.

**F. Cost summary** (if some measurements have a product): a per-measurement list (cost $), **Materials / Labor** subtotals, **Total $**, an incompatible-units warning, and **five export buttons**: Bid PDF, CSV, HTML Estimate (opens a tab), Download HTML, DXF (AutoCAD).

**G. Document**: file + page count, or "No document loaded".

### 2.8 Live BOM panel (`BomEstimationPanel`)

Docked on the right, **collapsed by default** (a thin strip). It shows the **live estimate of assemblies**:

- "{n} active assemblies out of {total}", with **orphan** measurements (no assembly) flagged;
- the **Active assemblies** list (checkbox, **auto** or **manual** badge, "Enable/disable all" per section) and a **"Reset auto"** button;
- a **Job-site data** block (fields to fill in);
- a **Detailed schedule** (Section / Product / Qty / Unit / Price / Total);
- an **All-assemblies rollup** (unique products, with the number of assemblies that use them);
- the **Project estimate total**;
- two exports: **Supplier CSV** (detail + rollup) and **Estimate CSV** (sheet with time and cost).

> Keep in mind: the **"Automatic BOM bid"** (toolbar button) and the **BOM panel** do not share manual values. Data entered by hand in this panel is **not** picked up by the automatic BOM bid.

### 2.9 Bottom status bar (`BottomBar`)

Shows, left to right: the **active tool**; the cursor **coordinates** (in real units if calibrated, otherwise in pixels); the **live value** of the measurement in progress; the **SNAP** indicator; a clipboard counter; "{n} measurements"; the **Imperial (ft) / Metric (m) toggle (Ctrl+U)**; the **"Calibrated ({unit})"** or **"Not calibrated"** state; and, if the plan is calibrated and contains measurements, a **"Recalibrate"** link.

### 2.10 Modals

| Modal | Component | Main contents |
|---|---|---|
| **Calibration** | `CalibrationModal` | "Real length of the drawn reference" (e.g. "3.048 or 10-0-0 or 160608"), unit (m/cm/mm/ft/in), automatic imperial detection, a **"Recompute the N existing measurements at the new scale"** checkbox (checked by default). Calibration is **per page**. |
| **Bid** | `SoumissionModal` | An amber warnings banner (incompatible units / self-intersecting areas / formula composites linked live). **Section 1** — client card (Project name*, company client, contact, PO number, dates, priority, description). **Section 2** — consolidation toggle (Detailed / Product + layer / Product only) + **"Include labor"** checkbox, table by category, subtotals, dynamic taxes (tenant/quote, QC fallback), **Total incl. tax**. **Section 3** — Cancel, HTML Bid preview, Apply to quote, Create a quote. |
| **Takeoff library** | `MetreLibraryModal` | Search by name/description/PDF, cards (measurements, layers, pages, "linked to a quote", timestamp), delete (with confirmation), open. |
| **New takeoff / Rename** | `SaveMetreModal` | Name* + description. |
| **Product Catalog** | `ProductCatalog` | Products / Add / Import-Export tabs; search, categories, "Composite" badge, inline editing (Edit / Delete / Components), Waste % field, creating a **composite**; JSON import/export, empty the catalog. |
| **Composite editor** | `CompositeEditor` | Display mode, price override, **sub-products** (product + Qty/unit + **Formula**), **input variables (BOM inputs)** (lowercase name with underscores, unit, default value, description), validation. |
| **CCQ trades 2026** | `LaborCatalogPanel` | Trade, specialty, sector, hourly rate, number of people, productivity + unit, color; reset to the CCQ 2026 rates. |
| **Architectural symbols** | `SymbolCatalogPanel` | Categories, views (Plan/Elevation/Side), width/height in inches, JSON import/export, reset. |
| **Construction Master Pro calculator** | `CalculatorPanel` | Feet-inches entry, unit conversion, history. |
| **Slope converter** | `SlopeConverterPanel` | Slope (x:12) ↔ Degrees ↔ Percentage + reference table. |
| **Multi-page summary** | `SummaryPanel` | Client / Project / Takeoff sections, By group, By type, By product, By page; exports Bid PDF / CSV / HTML Estimate / Download HTML / DXF. |

---

## 3. Step-by-step workflows

### 3.1 Create or open a takeoff

1. The **Takeoff** tab. If no takeoff is open, the top bar offers **"New takeoff"** and **"Open"**.
2. **New takeoff**: enter a **name** (required) and a description, then **Create**. An empty project is created on the server (`POST /projects`).
3. **Open**: the **library** lists your takeoffs (search by name/description/PDF). Click a card to open it; you can also delete a takeoff (with confirmation).

> You can start loading a plan even before naming the takeoff: the PDF is cached locally, then **uploaded automatically** on the first save (`uploadCachedPdfForProject`).

### 3.2 Load a plan

1. **File → Open a plan (Ctrl+O)**, or **drag and drop** a PDF/image into the center area.
2. Accepted formats: PDF, PNG, JPG, JPEG, BMP, TIFF, WEBP. Maximum PDF size: **150 MB** (`METRE_MAX_FILE_SIZE_MB`).
3. The plan is displayed; navigate between pages if the document has several pages.
4. Tip: need a blank sheet to sketch on? **File → New blank plan (ARCH D 36"×24")**.

Server checks on upload: `.pdf` extension, size ≤ 150 MB (otherwise a 413 error), non-empty file, verified **`%PDF-` signature**, and **MIME type forced to `application/pdf`** (protection against booby-trapped files).

### 3.3 Calibrate the scale (required before pricing)

1. The **Calibrate (K)** tool. Draw a line over a **known dimension** on the plan (for example a wall dimensioned at 40 ft, or a scale bar).
2. In the modal, enter the **real length** and the **unit** (imperial `40-00-00` or decimal `12.192`). Imperial detection shows the interpretation.
3. Leave **"Recompute the N existing measurements at the new scale"** checked if you are correcting a calibration after already drawing (the values are reprojected).
4. **Calibrate**. The scale factor is saved **for this page** (`POST /documents/{id}/calibrate`). Repeat on each page if the scales differ.

> Tip: choose a **long**, clearly legible dimension. A reference that is too short amplifies the imprecision across every measurement on the page.

### 3.4 Draw measurements

- **Distance (D)**: two clicks.
- **Wall — keyboard measurement (J)**: place the first point, choose the **direction** with the arrows, type the **exact length** on the keyboard (feet-inches-sixteenths), then press **Enter** to place the segment at the desired length. Ideal when the dimension is written down but mouse drawing lacks precision.
- **Area (A) / Perimeter (P) / Polyline (L)**: successive clicks; double-click or **Enter** to close/finish.
- **Rectangle (R)**: two opposite corners. **Circle (I)**: center then radius.
- **Count (C)**: one click per element; the total increments.
- **Angle (N)**: three points. **Dimension (X)**: annotated dimension.
- **Drawing aids**: **snapping (F3)** magnetizes the cursor to the **endpoints**, **midpoints** and **intersections** of nearby measurements; **ortho mode (F8)** and the **grid (F7)** make straight drawing easier.

> The module **warns** you if an area outline crosses itself ("self-intersecting polygon"): the area and perimeter then become unreliable. Redraw the outline cleanly.

### 3.5 Organize: layers and groups

1. Left panel → **Add a layer** (for example "Basement walls", "Ground-floor slab"), choose the color.
2. Gather several layers into **collapsible groups** ("Create a group", then drag the layers into it).
3. **Lock** a layer (padlock) to avoid changing it by mistake; **hide** it (eye) to declutter the display.
4. Drag a measurement onto another layer to move it, or use the ↑ / ↓ arrows and "Move to..." in the measurement list.

### 3.6 Associate a product and price it

1. Select a measurement. The right panel opens on its properties.
2. Choose a **product** from the catalog (menu grouped by category), or create one via **"Manage Catalog"**.
3. If needed: **manual quantity** (overrides the computed value), **waste %**, **slope factor** (roofing), **deductions** (see §3.7), **labor** (see §3.9).
4. The **cost** appears: `net quantity × (1 + waste %) × price`, **after conversion to the product's price unit**. An **"incompatible unit"** alert appears if the measurement cannot be converted (for example an area linked to a linear price).

### 3.7 Deductions (openings)

1. Draw the parent measurement (for example a wall's area).
2. Draw the **opening** (window, door) as a separate measurement, then check **Deduction** in its properties and choose the **parent measurement**.
3. The parent's **net** quantity = gross − sum of deductions (never negative). The **Gross / − deductions / Net** block shows this.

### 3.8 Slope factor (roofing)

1. On an **area** measurement (or a circle), open **Slope factor**.
2. Choose a preset (**Flat**, **2:12** ... **12:12**) or a custom slope.
3. The module converts the **horizontal area** (measured on the plan) into the **real (sloped) roof area**. The factor enters the cost computation.

> The **Slope Converter** (toolbar) maps Slope (x:12) ↔ Degrees ↔ Percentage and provides a reference table.

### 3.9 Labor (CCQ trades)

1. Open **"Manage Labor"** to review/complete your trade catalog (trade, specialty, sector, **hourly rate**, number of people, productivity). A button resets to the **CCQ 2026 rates**.
2. On a measurement, the **Labor** section: choose the trade, enter **Hours × People**. The **labor cost** is added to the material cost to give the line's **TOTAL**.

### 3.10 Parametric assemblies (BOM / composites)

For repetitive work (2×4 wall, flooring, etc.), instead of associating a product with each measurement, link a **layer** to an **assembly** that computes itself.

1. **Manage Catalog → create a composite product** (assembly). *Available only in connected-ERP mode.*
2. In the **Composite editor**:
   - declare **input variables** (for example `surface_mur`, `longueur_mur`, `nombre_coins`) with their **unit** and a default value; the name must be lowercase with underscores (`^[a-z][a-z0-9_]*$`);
   - add **sub-products**, each with a **quantity per unit** or a **formula** (for example `surface_mur / 32` for 4×8 sheets, or `IF(surface_ss > 800, 3, 2)`).
3. **Link a layer** to the assembly (the layer's BOM-link icon). Geometric variables fill in **automatically** from the layer's measurements ("auto" badges), converted into the variable's declared unit. The other variables are entered by hand.
4. The **BOM panel** shows the schedule (sections + rollup by product) and the **project total**. Export as **Supplier CSV** or **Estimate CSV**.

> **Formula safety.** Formulas are evaluated by a **dedicated interpreter** (never `eval`), with an allowlist of functions — `CEIL, IF, MIN, MAX, ROUND, SUM, ABS, FLOOR` — and of characters; length ≤ 500 characters; every variable cited in a formula **must exist** among the input variables, otherwise the formula is rejected (avoids the "the line always yields 0" trap). References: `_validate_formula` (`metre_pdf.py:532`), `_FORMULA_DSL_FUNCTIONS` (`metre_ai_tools.py:778`).

### 3.11 AI auto-count ("AI Count")

Automatically counts repeating elements (outlets, doors, windows, light fixtures...) in an area you box in.

1. **Make sure the plan is straight (0°)** — reset the rotation if needed.
2. The **AI Count (U)** tool. The purple panel opens.
3. Enter the **element to count** (for example "electrical outlet").
4. **Draw a rectangle** around the area to analyze.
5. The AI returns "{n} occurrence(s) detected", an average **confidence** as a percentage, and a **cost** in dollars.
6. **Confirm** creates **one** count-type measurement with N points on the page; **Cancel** or **New region** to start over.

Possible error messages: "Set the plan straight (0°) before counting", "AI credits exhausted", "Region too large", "AI service busy".

*Under the hood*: the `POST /metre/ai/detect` endpoint (`metre_ai_detect.py:157`), Claude Opus 4.8 model, **no database write** (the measurement is created client-side after your confirmation). Billed at **actual cost × 1.30** (cap US$2/call), deducted **only after success**. A maximum of two simultaneous calls on the server (anti-overload).

### 3.12 Conversational AI assistant

A chat panel (480 px, on the right) where you describe what you want in plain language; the AI reads your takeoff and suggests actions.

1. **Save the takeoff first**: the Assistant button stays **grayed out** until the document has a server ID.
2. Open the assistant. Choose an **AI profile** (your profiles or the system profiles) and, if needed, a **new conversation** (the conversation list is in the panel's sidebar).
3. Type your request (for example "list the measurements on page 2", "create a Ground-floor walls layer", "search for a 5/8 gypsum product"). The response arrives **as a stream** (streaming), showing the executed tools and the costs.
4. The **8 read tools** (list measurements/layers, calibration, summary, search products, composite details, active composites, past projects) run **automatically**.
5. The **11 write tools** (create a layer/a measurement, link a composite, delete a measurement, and 7 catalog actions) show a **confirmation card**: nothing is applied until you click **Confirm** (confirmation is **server-side**, not bypassable). You can **reject** an action.
6. The panel's footer shows the **cumulative cost**, the **conversation cost** and a **message counter {n}/200** (warning at 180, critical at 195).

Points to watch:
- **Page lock**: if the AI answers about the page you sent but you have navigated elsewhere, a "Stop and refocus" banner warns you.
- **Only one active conversation at a time per document**, and **200 messages maximum** per conversation.
- **Editing before confirmation**: you can adjust the parameters of the **4 measurement tools** before confirming, but **not those of the 7 catalog tools** (create/edit/delete a product, composite variables and lines) — for the latter, you confirm as-is or reject.

*Under the hood*: `POST /documents/{id}/assistant-chat` (`metre_ai_chat.py:1229`, SSE streaming), Claude Opus 4.8 model, conversations managed by `metre_ai_conversations.py`, 19 tools in `metre_ai_tools.py`. A lock prevents two simultaneous streams on the same conversation (message "already streaming").

### 3.13 Generate the bid

1. Toolbar → **Generate a bid** (measurements with an associated product), or **Generate an automatic BOM bid** (layers linked to composites). The Bid modal opens.
2. Read the amber **warnings banner** (incompatible units, self-intersecting areas, formula composites linked live) and fix them if needed.
3. **Section 1** — fill in the client card (Project name required, client, contact, PO number, dates, priority, description).
4. **Section 2** — choose the **consolidation** (Detailed / Product + layer / Product only) and check **"Include labor"** if you want. Review the table (by category), the Materials / Labor subtotals, the **taxes** (the tenant's or the quote's; fallback **GST 5% / QST 9.975%**) and the **Total incl. tax**.
5. **Section 3** — three outcomes:
   - **HTML Bid preview** (if a quote is connected): previews the rendering;
   - **Apply to quote**: adds the lines to the connected quote;
   - **Create a quote**: generates a new quote from the takeoff.

> **What crosses over to the quote.** Lines in the **Administration / Contingencies / Profit** categories are **filtered out** on the way to the quote (to avoid double-counting: the quote applies its own markups), and only lines with **quantity > 0** are sent (the quote server rejects a zero quantity). The amount is **recomputed server-side** (`amount = round(quantity × price)`), and the preview already uses these same rounded values to avoid any discrepancy (`MetrePdf.tsx:1037-1057`).

> **Technical note.** The `POST /projects/{id}/import-to-devis/{devis_id}` server endpoint exists but is **frozen** (it returns 503 unless explicitly enabled by the `METRE_IMPORT_TO_DEVIS_ENABLED` environment variable). The path used by the interface is the **Bid modal** described above (`onApplyToDevis` / `onCreateDevis`, `DevisPage.tsx:1477`), not this endpoint.

### 3.14 Export

| Output | Where | Contents |
|---|---|---|
| **Supplier CSV** | BOM panel | Materials schedule (detail + rollup) for ordering. |
| **Estimate CSV** | BOM panel | Estimate sheet (time + cost). |
| **CSV** (product / summary) | Right panel, Multi-page summary | Tables by product / by measurement. |
| **Bid PDF** | Right panel, Summary | Laid-out bid (materials, labor, taxes). |
| **HTML Estimate** | Right panel, Summary | HTML bid (open in a tab or download). |
| **PNG** | Toolbar | 3 files: annotated plan + product detail + BOM detail (empty files are skipped). |
| **DXF (AutoCAD)** | Right panel, Summary | Measurement geometry for a CAD program. |

> **CSV and Excel.** Files are **UTF-8 with BOM** and use the **document-language separator**: **semicolon (`;`) in French** (Quebec Excel), **comma (`,`) in English** (`csvSeparatorForLang`, `RightPanel.tsx:397`). Each cell is wrapped in quotes and formula injection is neutralized (a name starting with `=`, `+`, `-` or `@` is prefixed). **Printing** is done via the bid PDF.

### 3.15 Save, rename, reopen

- **Saving**: an open takeoff is saved automatically (local autosave with ~1 s debounce + server sync on each change in ERP mode). The top-bar indicator shows the state ("Saved N s ago", or "Save failed" in red).
- **First save**: the cached PDF is **uploaded** automatically (BYTEA on the server).
- **Rename**: the pencil in the top bar.
- **Reopen / delete**: the **"Open"** button → takeoff library.

### 3.16 Recalibrate

1. Bottom bar → **"Recalibrate"** (visible if the page is calibrated and contains measurements), or relaunch the **Calibrate** tool on an already-calibrated page.
2. Draw the new reference, enter the true length, leave **"Recompute the existing measurements"** checked.
3. The server applies an **atomic recalibration**: the calibration and the **measurement values of this page only** are updated in a single transaction (other pages do not move). Cap: **5,000 measurements** per recalibration (`POST /documents/{id}/recalibrate`).

---

## 4. Reference

### 4.1 Keyboard shortcuts

| Key | Action | | Key | Action |
|---|---|---|---|---|
| **K** | Calibrate | | **X** | Dimension |
| **V** | Select | | **H** | Pan / hand |
| **D** | Distance | | **J** | Wall (keyboard measurement) |
| **A** | Area | | **T** | Text |
| **R** | Rectangle *(or Rotate 45° if there is a selection)* | | **W** | Arrow |
| **P** | Perimeter | | **Q** | Revision cloud |
| **L** | Polyline | | **F** | Freehand |
| **N** | Angle | | **G** | Highlighter |
| **C** | Count | | **E** | Note |
| **U** | AI Count | | **B** | Text bubble |
| **I** | Circle | | **M** / **Shift+M** | Horizontal / vertical mirror copy |
| **Ctrl+O** | Open a plan | | **Ctrl+D** | Duplicate the selection |
| **Ctrl+Z / Ctrl+Y** | Undo / Redo | | **Ctrl+U** | Imperial / metric toggle |
| **Ctrl+Shift+C / V** | Copy / paste properties | | **F3 / F8 / F7** | Snap / Ortho / Grid |
| **F11** | Full screen (Esc does not exit it) | | | |

### 4.2 Backend endpoints (prefix `/api/erp/v1/metre`)

All protected by a JWT token and isolated by company schema; **no particular role required** beyond tenant membership.

| Group | Method + route | Purpose |
|---|---|---|
| **Projects** | `POST /projects` · `GET /projects` · `GET /projects/{id}` · `PUT /projects/{id}` · `DELETE /projects/{id}` | CRUD of takeoffs |
| | `GET /metres-library` | Aggregated library view (measurements, layers, pages, author) |
| **Documents** | `POST /projects/{id}/documents/upload` | Upload a PDF (verified signature, BYTEA, ≤ 150 MB) |
| | `GET /projects/{id}/documents` · `GET /documents/{id}` | List / metadata |
| | `GET /documents/{id}/file` | Download the PDF |
| | `GET /documents/{id}/page/{page}?zoom` | PNG render of a page (cached, zoom 0.1–10) |
| | `DELETE /documents/{id}` | Delete a document |
| **Calibrations** | `POST /documents/{id}/calibrate` | Set a page's scale |
| | `POST /documents/{id}/recalibrate` | Atomic recalibration (≤ 5,000 measurements) |
| | `GET /documents/{id}/calibrations` | All calibrated pages (feeds the client control) |
| | `GET /documents/{id}/calibration/{page}` · `DELETE /documents/{id}/calibration/{page}` | Read / delete a calibration |
| **Measurements** | `GET /documents/{id}/measurements?page&layer_id` · `POST /documents/{id}/measurements` | List / create |
| | `GET /measurements/{id}` · `PUT /measurements/{id}` · `DELETE /measurements/{id}` | Read / edit / delete |
| | `GET /documents/{id}/measurements/export?format` | Export CSV or JSON |
| **Layers** | `GET/POST /documents/{id}/layers` · `GET/PUT/DELETE /layers/{id}` | CRUD of layers |
| **Groups** | `GET/POST /documents/{id}/layer-groups` · `PUT/DELETE /layer-groups/{id}` | CRUD of layer groups |
| **Products** | `GET /products?category` · `POST /products` · `POST /products/bulk-import` | List / create / bulk import |
| | `GET/PUT/DELETE /products/{id}` | Read / edit / delete |
| **Components** | `GET/POST /products/{id}/components` · `PUT/DELETE /products/{id}/components/{cid}` | Sub-products of an assembly |
| **Snapping** | `POST /documents/{id}/snap-points` | OpenCV detection (≤ 4,000 px) |
| **Summary** | `GET /documents/{id}/summary` | Totals by type / layer |
| **Quote (frozen)** | `POST /projects/{id}/import-to-devis/{devis_id}` | Server-side import — **disabled (503)** by default |
| **Assistant** | `POST /documents/{id}/assistant-chat` | Streaming AI chat (SSE) |
| **Conversations** | `GET /documents/{id}/conversations` · `GET/DELETE /conversations/{id}` | List / detail / archive |
| | `POST /conversations/{id}/tool-executions/{eid}/confirm` | Confirm/reject a write action |
| **Auto-count** | `POST /metre/ai/detect` | Vision counting over an area (no write) |

### 4.3 AI assistant tools (19)

**8 read tools** (run automatically): `lister_mesures`, `lister_calques`, `obtenir_calibration`, `obtenir_summary`, `chercher_produits_bom`, `obtenir_composite_details`, `lister_composites_actifs`, `lister_projets_passes`.

**11 write tools** (confirmation required, `TOOLS_REQUIRING_CONFIRMATION`, `metre_ai_tools.py:2521`):
- Takeoff (4): `creer_calque`, `creer_mesure`, `lier_composite_calque`, `supprimer_mesure`.
- Catalog (7): `creer_produit`, `modifier_produit`, `supprimer_produit`, `definir_variables_composite`, `ajouter_ligne_composite`, `modifier_ligne_composite`, `supprimer_ligne_composite`.

Every write follows the **preview → server confirmation → execution** pattern: at confirmation time, the server re-checks credits **before** any effect, re-validates the references (message "The references have changed" if they have moved), sets an anti-double-click lock, then executes and records the result.

### 4.4 Key calculations (the money path)

**Cost of a measurement line**: `net quantity × slope factor × (1 + waste %) × price`, where the quantity is first **converted** to the product's price unit.

**Unit conversion (billable factor)** — `_billable_factor` (`metre_pdf.py:3870`):

| Measurement unit | 1 unit in meters | | Price dimension | Behavior |
|---|---|---|---|---|
| mm | 0.001 | | linear → linear | factor = conversion |
| cm | 0.01 | | area → area | factor = conversion² |
| m | 1 | | count (unité, feuille, sac, boîte) | no conversion |
| ft | 0.3048 | | **cross** (area priced per linear, volume) | factor = 1 + **"incompatible"** |
| in | 0.0254 | | | |
| yd | 0.9144 | | | |

This mechanism fixes the ×10.76 under-billing (m² → sq ft). In a 100% imperial flow (the usual Quebec case), no conversion occurs (identity).

**AI cost**: Opus 4.8 = US$5/M input tokens, US$25/M output, US$10/M cache write, US$0.50/M cache read, all **× 1.30** (margin), capped at **US$2 per message/call**.

### 4.5 Bounds, validations and limits

| Element | Value / rule |
|---|---|
| PDF size | ≤ **150 MB** (`METRE_MAX_FILE_SIZE_MB`), mandatory `%PDF-` signature, MIME forced to `application/pdf` |
| A measurement's `metadata_json` | ≤ **64 KB** |
| A layer's `composite_inputs` | ≤ **16 KB** |
| Recalibration | ≤ **5,000 measurements** per page |
| Snap region | ≤ **4,000 px** |
| Composite formula | ≤ **500 characters**, allowlisted functions, variables must exist |
| Points of an AI measurement | ≤ **5,000** |
| Assistant: messages | ≤ **200** per conversation; one active conversation per document |
| Assistant: tool loop | ≤ **8 iterations**; cost ≤ US$2/message |
| Auto-count | straight plan (0°) mandatory; ≤ 1,000 detections; request ≤ 20 MB |
| Calibratable units | `m`, `cm`, `mm`, `ft`, `in` |
| Full screen | display-only (Esc does not exit it) |

Fundamental limits to know:
- **Per-page calibration**: a plan with several scales must be calibrated page by page.
- **Isotropic area**: area assumes the same scale in X and Y; a scanned and stretched plan can distort areas.
- **Simple outline**: a self-intersecting polygon gives a misleading area; the module **warns** but does not correct.
- **Rotation**: auto-count requires the plan to be straight; manual measurement, however, handles a rotated plan.

### 4.6 Data model (12 tables per company)

| Table | Contents |
|---|---|
| `metre_projects` | The takeoff (name, description, `company_id`, `devis_id`, author). |
| `metre_documents` | The plan: **PDF as BYTEA** + `mime_type`, page count. |
| `metre_calibrations` | Scale per page — uniqueness constraint (document, page). |
| `metre_layers` | Layers (color, visibility, lock, `composite_id`, `composite_inputs`, `group_id`, order). |
| `metre_layer_groups` | Layer groups (name, order, collapsed). |
| `metre_measurements` | Measurements (`points` JSONB, `metadata_json` JSONB, type, value, unit, quantity, product, deduction, slope, labor; links `ai_detection_id`, `devis_ligne_id`). |
| `metre_products` | Catalog (price, unit, waste %, composite, `bom_inputs`, labor fields, section). |
| `metre_product_components` | Sub-products of an assembly (quantity or formula). |
| `metre_ai_detections` | AI auto-count traces (tokens, cost, status). |
| `metre_ai_conversations` | Assistant conversations (one per document + user). |
| `metre_ai_messages` | Messages (role, content blocks). |
| `metre_ai_tool_executions` | Assistant write actions (status: pending / confirmed / rejected / executed / failed). |

These tables are created **on demand** on first use (`_ensure_tables`, `metre_pdf.py:1222`), not when the account is created.

### 4.7 Common messages and states

| Situation | Message |
|---|---|
| No takeoff open | "No takeoff open. Create or open a takeoff..." |
| Save failed | "Save failed" (clickable red badge) |
| Measurement with no product | "No product" (orange) / "{n} to fix" badge |
| Product removed from the catalog | "Product not found" |
| Self-crossing outline | "self-intersecting" warning on the value |
| Uncalibrated plan | "Not calibrated - use the Calibrate tool" |
| Measurement unit ≠ price unit | "incompatible unit" (quantity not converted) |
| Assistant unavailable | button grayed out until the PDF is saved |
| Auto-count on a rotated plan | "Set the plan straight (0°) before counting" |
| AI credits exhausted | "AI credits exhausted" |

---

## 5. Integrations and FAQ

### 5.1 Module 08 — Quotes

The Takeoff **lives inside** the Quotes page and **feeds** it. From the Bid modal: **"Apply to quote"** adds the lines to the connected quote, **"Create a quote"** generates a new one. The link is remembered (`metre_projects.devis_id`). The Administration / Contingencies / Profit categories are filtered out along the way (the quote applies its own markups) and each line's amount is recomputed server-side.

### 5.2 Shared catalogs

The **products** and the **labor** (CCQ trades) are shared at your company level. A product created in the Takeoff serves the assemblies and the bids; the CCQ 2026 rates are provided by default and can be reset.

### 5.3 Module 31 — CAD / 3D Modeling

A **separate** module. The Takeoff works **on a PDF plan (2D)** for quantity takeoff; CAD **builds a 3D model** (walls, roofs, furniture) and produces its own 3D takeoff. The two share utilities (imperial entry, dimensioning) but share neither the same page nor the same tables.

### 5.4 Module 25 — AI Assistant (general)

The **Takeoff** assistant is **dedicated to takeoff** (tools typed to measurements and catalogs) and separate from the ERP's general AI Assistant. It bills **AI credits** to your company and requires a **confirmation** for any write.

### 5.5 Module 28 — Configuration

The **taxes** applied to the bid come from your company's configuration (or the quote's), with a fallback of **GST 5% / QST 9.975%**. The account's **language** determines the **CSV separator** (`;` in French, `,` in English).

### 5.6 FAQ

- **I can't find "Takeoff" in the menu.** That's normal: it is not a menu entry or a `/metre` address. Go to **Sales → Quotes**, then the **"Takeoff"** tab.
- **The AI Assistant button is grayed out.** Save the takeoff first (name it and let the PDF upload). The assistant requires a document already persisted on the server.
- **My quantity looks 10× too low.** Check the **calibration unit ↔ price unit** agreement. A plan in **meters** with a product priced per **sq ft** is converted automatically; the "incompatible unit" alert flags the genuinely non-convertible cases (area linked to a linear price, volume).
- **My accents are wrong in the CSV (`boîte` → `boÃ®te`).** The CSVs are UTF-8 with BOM and do not use a `sep=` directive (which broke the encoding in Excel). Re-open the re-exported file; in English Excel, switch the account language to English to get a comma separator.
- **My CSV columns are shifted or show `#NAME?`.** Each cell is wrapped in quotes and values starting with `=`, `+`, `-` or `@` are neutralized: re-export.
- **Auto-count refuses to count.** The plan must be **straight (0°)**: reset the rotation. Also check your AI credits and shrink the area if it is too large.
- **The AI suggested an action but nothing happened.** Write actions require your **confirmation** in the card shown; until you confirm (or if you reject), nothing is applied.
- **I want to change a parameter before confirming a product creation.** That is not possible for the 7 **catalog** actions (product, composite variables/lines): confirm as-is or reject, then ask again with the right parameters. The 4 **measurement** actions (layer, measurement, composite link, deletion), however, do accept an adjustment before confirmation.
- **Full screen won't close with Esc.** Full screen is a display mode: press **F11** again (or the button). Esc cancels the tool or deselects.
- **The "Automatic BOM bid" ignores my values entered in the BOM panel.** That's intentional: this process does not pick up the panel's manual values and does not generate labor lines. Use the standard bid ("Generate a bid") if you have adjusted values by hand.
- **Can I import the takeoff directly into a quote without going through the modal?** No: the direct-import server endpoint is **frozen** (503). Always go through the Bid modal.
- **Can a non-administrator colleague edit my takeoffs?** Yes: the module does not restrict by role. Any authenticated member of your company has full control (the data stays isolated from other companies).

---

## 6. Summary

- **Access**: **Sales → Quotes → "Takeoff" tab** (no `/metre` route, no menu entry). The module loads on demand and receives the current quote.
- **Purpose**: quantity takeoff on a PDF/image → pricing → bid and schedules, re-injectable into a quote.
- **Key steps**: open/create a takeoff → load the plan → **calibrate (per page)** → draw → associate products / assemblies → bid → export or send to the quote. **Always calibrate before pricing.**
- **Measurements**: Distance, Area, Rectangle, Perimeter, Polyline, Angle, Count, **AI Count**, Circle, Dimension, **Wall (keyboard)** + annotations and symbols.
- **Units**: m/cm/mm/ft/in; imperial entry to 1/16"; **automatic conversion** to the price unit (fixes the ×10.76 m²↔sq ft trap); "incompatible unit" alert.
- **Pricing**: `net quantity × slope × (1 + waste) × price`; deductions; CCQ labor; QC taxes.
- **BOM**: parametric assemblies (variables + safe formulas, no `eval`) fed automatically by the layers; available **in ERP mode only**.
- **AI**: **auto-count** (vision, straight 0° plan) and **conversational assistant** (Opus 4.8), writes **confirmed server-side**, billed in credits (actual cost × 1.30, cap US$2).
- **Exports**: Supplier / Estimate CSV (UTF-8 + language-dependent separator), PDF, HTML, PNG (3 files), DXF.
- **Persistence**: PDF as **BYTEA**; ~1 s autosave + server sync; takeoff library; **12 `metre_*` tables** created on demand.
- **Permissions**: **no role required** beyond company membership; strict per-company isolation; view-only mode inherited from the account.
- **Pitfalls to avoid**: forgetting to calibrate; linking an area to a linear price; AI-counting on a rotated plan; expecting the AI assistant before saving the takeoff.

---

> Manual written from the actual source code (revision 2026-07). **Files verified**: `frontend/src/components/metre-pdf/` (`MetrePdf.tsx`, `store.ts`, `components/MeasurementCanvas.tsx`, `components/PDFViewer.tsx`, `components/TopToolbar.tsx`, `components/LeftPanel.tsx`, `components/RightPanel.tsx`, `components/BomEstimationPanel.tsx`, `components/BottomBar.tsx`, `components/AiCountPanel.tsx`, `components/MetreAssistantPanel.tsx`, `api.ts`, `api/metreAi.ts`, `utils/canvasRotation.ts`, `useAiCount.ts`), `frontend/src/pages/DevisPage.tsx` (host, `metre-pdf` tab); `backend/routers/metre_pdf.py` (45 endpoints), `metre_ai_chat.py` (1), `metre_ai_conversations.py` (4), `metre_ai_detect.py` (1), `metre_ai_tools.py` (19 tools). **Related manuals**: **08 — Quotes** (destination of the lines), **31 — CAD / 3D Modeling** (modeling, separate module), **25 — AI Assistant** (general assistant), **28 — Configuration** (taxes, language).
