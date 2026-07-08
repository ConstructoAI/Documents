# Module 09 — Store (products, inventory, purchasing)

> **Version**: 3.0 (complete overhaul verified against the actual source code)
> **Reference code**:
> - Frontend: `frontend/src/pages/MagasinPage.tsx` (≈ 3,425 lines, single 7-tab page), `frontend/src/components/rfq/RfqTab.tsx` (≈ 1,412 lines), `frontend/src/components/magasin/MaterialPricingTab.tsx` (≈ 502 lines), `frontend/src/components/magasin/MaterialWebSearch.tsx` (≈ 150 lines), `frontend/src/components/magasin/MagasinAssistantTab.tsx` (≈ 232 lines)
> - API client: `frontend/src/api/inventory.ts`, `suppliers.ts`, `rfq.ts`, `magasinAi.ts`, `materialsPricing.ts`
> - Backend: `backend/routers/inventory.py` (products + movements + bill of materials + statistics), `backend/routers/suppliers.py` (suppliers + purchase orders), `backend/routers/rfq.py` (requests for quote, ≈ 2,435 lines), `backend/routers/materials_pricing.py` (material prices), `backend/routers/magasin_ai.py` (AI assistant)
> - API prefix: `/api/erp/v1`
> **PostgreSQL tables (per tenant)**: `produits`, `mouvements_stock`, `produit_composants`, `fournisseurs`, `bons_commande`, `bon_commande_lignes`, `rfq_demandes`, `rfq_reponses`, `produit_fournisseurs`, `produit_historique_prix`; shared tables: `public.bc_public_tokens`
> **Scope**: this module brings together on **a single page** the entire supply and stock-management cycle of a construction company: the **product catalog** (materials, equipment, PPE), **stock level tracking** with full movement traceability, **purchase orders** to suppliers (with email sending and receiving that replenishes stock), **requests for quote** (bids to several suppliers, with a public portal), a **retailer price comparator** (Canac, BMR, Home Depot) and an **AI assistant**. It **does not manage** multiple warehouses or inter-site transfers, it **does not calculate** a moving weighted average cost on inventory value, and stock is **never edited by hand** (only through a movement).

---

## Table of contents

1. [Overview](#1-overview)
2. [Interface](#2-interface)
3. [Step-by-step processes](#3-step-by-step-processes)
4. [Reference](#4-reference)
5. [Integrations and FAQ](#5-integrations-and-faq)
6. [Summary](#6-summary)

---

## 1. Overview

### 1.1 Module mission

The Store is the procurement control desk. It lets you:

- maintain a **catalog** of products (name, code, selling price, cost price, unit, supplier, location);
- track the **available stock** of each product and be alerted when a product drops below its minimum threshold;
- trace **every stock movement** (in, out, adjustment) along with the associated employee, project, purchase order and cost;
- issue numbered **purchase orders** (POs) to suppliers, **send them by email** (with a public link), and **receive** the goods, which **automatically increases stock**;
- launch **requests for quote** (RFQs) to several suppliers, **compare** their offers and **award** the order (which generates the POs);
- **compare prices** for a list of materials at major retailers;
- declare the **bill of materials** (BOM) of an assembled product;
- rely on an **AI assistant** to query the data and create products.

### 1.2 Access

- **Sidebar** → **Operations** section → **Store** (cart icon).
- **Address**: `/magasin`.
- Protected page: you must be authenticated in a tenant.
- **Tab open by default**: **Orders** (purchase orders).
- **Directly opening a purchase order**: a link of the form `/magasin?open=<id>` (for example from the 360 view of a case file) automatically opens the relevant PO.

### 1.3 The 7 tabs (sub-modules)

The page is made up of seven tabs, in this order:

| # | Tab | Role | Icon |
|---|-----|------|------|
| 1 | **Orders** | Purchase orders to suppliers (creation, sending, receiving, invoicing) | document |
| 2 | **Requests for Quote** | Bids to several suppliers, comparison, award | scale |
| 3 | **Material Prices** | Canac / BMR / Home Depot price comparator + web search | dollar |
| 4 | **Movements** | History and creation of stock movements | arrows |
| 5 | **Products** | Catalog + bill of materials (BOM) + barcode labels | box |
| 6 | **Supplier** | Supplier records | truck |
| 7 | **AI Assistant** | Chat that reads the data and offers to create products | stars |

> The **search** field and the **pagination** (20 rows per page) are shared at the top of the page.

### 1.4 Permissions and roles

**Viewing** all tabs is open to any authenticated user of the tenant. **Writing** depends on the role. Three families of rights coexist (all of them account for the owner's administrator status, so as never to lock out a boss whose role happens to be "user"):

| Right | What it allows | Allowed roles |
|-------|----------------|---------------|
| **Inventory write** (`require_inventory_write`) | Create/edit a product, create/cancel a movement, manage the bill of materials (BOM), print a label | admin, super-admin, **manager**, **storekeeper**, or any administrator (`is_admin`) |
| **Purchasing write** (`require_purchase_write`) | Create/edit a supplier, create/edit/send/receive/delete a purchase order, scan a PO | admin, super-admin, manager, storekeeper, **user**, or administrator |
| **RFQ write** (`require_rfq_write`) | Create/edit an RFQ, invite suppliers, enter prices, award | admin, super-admin, manager, storekeeper, or administrator |

> **Key nuance**: a plain **user** (non-administrator "user" role) can create and manage **purchase orders**, but **cannot** edit the product catalog, create a movement, or manage a request for quote. These last actions require a **manager** or **storekeeper** role (or administrator).

**Hiding the cost price**: the purchase cost (`cout_revient`, `cout_unitaire`) is **hidden** from field roles. Only management roles see it (admin, super-admin, manager, storekeeper, user, foreman, accountant). An employee with the "employee" role sees products and stock, **but not costs**.

### 1.5 Statistics banner (4 cards)

Four figure cards permanently sit atop the page (source: `GET /inventory/stats`):

| Card | Content | Color |
|------|---------|-------|
| **Products** | Number of active products | blue |
| **Minimum threshold alert** | Number of products below their threshold | red if > 0, otherwise green |
| **Stock value** | Total inventory value in dollars | green |
| **Categories** | Number of distinct categories | purple |

### 1.6 Key concepts

- **Product**: a catalog item. Two prices are **entered by hand**: the **selling price** and the **cost price**. There is **no** automatic average cost calculation.
- **Available stock**: the quantity on hand. It is **never** corrected directly on the record: only through a **movement**.
- **Stock movement**: a dated, traced operation that changes stock. Only three types: **In**, **Out**, **Adjustment**. Each movement keeps the before/after stock, the employee, the project, the purchase order, the cost, and the reason.
- **Purchase order (PO)**: a purchase commitment with a supplier, numbered `BC-00001`. When you move it to **Received**, the stock of the linked products **increases automatically** and an inbound movement is created.
- **Request for quote (RFQ)**: a call for bids. You describe the items **without prices**, invite suppliers, each one **quotes** (through a public link or internal entry), you **compare**, then you **award** — which generates one purchase order per selected supplier.
- **Bill of materials (BOM)**: the composition of an assembled product (its child components). Declarative: creating an assembly does not consume the components automatically.

---

## 2. Interface

### 2.1 General layout

```
+------------------------------------------------------------------+
|  "Store" title                                                   |
+------------------------------------------------------------------+
|  [Products]  [Threshold alert]  [Stock value]  [Categories]      |  <- 4 KPI cards
+------------------------------------------------------------------+
|  Orders | Requests for Quote | Material Prices | Movements |     |  <- tab bar
|  Products | Supplier | AI Assistant                             |
+------------------------------------------------------------------+
|  Action bar (buttons + search)                                  |
+----------------------------+-------------------------------------+
|  List (paginated table)    |  Detail panel (on the right)        |
+----------------------------+-------------------------------------+
```

Each tab offers its own **action bar** (create buttons, filters, search) and, for the Orders / Requests for Quote / Products / Supplier tabs, a **master-detail** view: the list on the left, the detail panel on the right when an item is selected. On a phone, the tables collapse into simplified cards.

### 2.2 "Orders" tab (purchase orders)

#### 2.2.1 Action bar

- **Supplier order** (main button): opens the purchase order creation window.
- **Scan a PO (AI)**: upload an image or a PDF of a supplier's purchase order; the AI extracts the line items. During processing: "Analysis in progress…".
- **Search** (number, supplier, project).

#### 2.2.2 Purchase order table

Columns (sortable and resizable):

| Column | Content |
|--------|---------|
| **Number** | `BC-00001` |
| **Supplier** | Supplier name |
| **Project** | Associated project (if provided) |
| **Amount** | PO total |
| **Order Date** | Editable directly on click (inline date field) |
| **Expected Delivery** | Editable directly on click |
| **Status** | Colored badge |

A **trash** icon lets you delete a PO (with confirmation). If no order exists: "No purchase order".

#### 2.2.3 The 7 purchase order statuses

| Status | Color | Meaning |
|--------|-------|---------|
| **Draft** | gray | Created, not yet sent |
| **Sent** | indigo | Sent to the supplier |
| **Confirmed** | blue | The supplier has confirmed |
| **In progress** | purple | Delivery underway |
| **Received** | green | Goods received → **stock has been replenished** |
| **Invoiced** | teal | Supplier invoice recorded |
| **Cancelled** | red | Order cancelled |

#### 2.2.4 Purchase order detail panel

When you select a PO, a panel opens on the right. Its sections:

- **Header**: number + supplier name + colored **status dropdown** (if the PO carries a status outside the list, it is still displayed).
- **Information**: Order date, Expected delivery date, Responsible employee (among active employees), Associated project.
- **Delivery**: Delivery address, Delivery terms.
- **Items**: a **"Select inventory item"** menu (which automatically fills in the description, the unit and the price), then Description, Quantity, Unit price, and the **"Add line"** button. Each line shows its amount (quantity × price) and can be deleted.
- **Totals**: Subtotal (before tax), **GST/QST** (Goods and Services Tax / Quebec Sales Tax) calculated according to the tax configuration specific to the PO (Canada or United States; by default in Quebec, GST 5%, QST 9.975%), then Total (tax included).
- **Internal notes**.
- **Actions (first row)**:
  - **Generate HTML**: downloads the purchase order as a printable `.html` file.
  - **Preview**: opens the document in an embedded window.
  - **Send to supplier**: opens the email sending window.
  - **QuickBooks CSV**: downloads a QuickBooks-compatible CSV file.
  - **Copy CSV**: copies the CSV to the clipboard.
  - **Excel (.xlsx)**: downloads the PO as a spreadsheet.
  - **Create invoice**: creates a (draft) supplier invoice from the PO. Disabled if the PO is already **Invoiced**.
- **Actions (second row)**: **Delete** and **Save** (the latter stays disabled until a field has been changed).

> **Targeted save**: the panel sends only **the fields that were actually modified** to the server. This avoids accidentally overwriting the status of a PO that a colleague moved to "Received" in the meantime.

#### 2.2.5 "Create a purchase order" window

Five sections: **Supplier** (required) · **Assignment** (Project, Employee, Delivery date) · **Items** (one line per item: choose a product from the catalog **or** enter a description and a unit by hand; real-time tax summary) · **Delivery** (address, terms) · **Internal notes**. **Cancel** / **Create the purchase order** buttons.

#### 2.2.6 "Scan a PO (AI)" window

After analyzing an image or a PDF, a banner appears in the creation window: "{n} line(s) extracted by the AI", a **confidence** indicator (high/medium/low), the extracted **total**, and an alert if the supplier was not recognized. The user validates or corrects, then creates the PO. The scan is **read-only** (it writes nothing until you click "Create").

#### 2.2.7 "Send the purchase order" window

Enter the supplier's email address, then send. The system:

- generates a **shareable public link** valid for **90 days** (copyable), which displays the PO without authentication;
- moves the PO to **Sent** status;
- sends an email themed in the company's colors.

### 2.3 "Requests for Quote" tab (RFQ)

This tab manages the full cycle of supplier bids. Action bar: **New request** and **Refresh**. Master-detail view.

#### 2.3.1 Request list

Columns: **Number** (`RFQ-00001`) · **Title** · **Project** · **Status** · **Deadline** · **Responses** (number of suppliers who responded out of the number invited).

**Five request statuses**: Draft · Sent · Closed · Awarded · Cancelled.

#### 2.3.2 Request detail panel

- **Header**: number + title + status badge + **Edit** (pencil), **Delete** (impossible if already **Awarded**) and close buttons.
- **Information**: Project, Responsible employee, Deadline.
- **Delivery**: address and terms (if provided).
- **Items to quote**: the list of items **without prices** (the suppliers will do the pricing), with an add form (Description, Quantity, Unit).
- **Invited suppliers**: each supplier carries a status (**Invited**, **Responded**, **Selected**, **Not selected**). Per line: **Enter price** (enter a quote received offline), **Send** (transmit the public link) and **Remove**. A "Choose a supplier" menu + **Invite** adds suppliers (those already invited are filtered out).
- **Comparison**: the **"View comparison"** button (disabled until there is a response) shows a table of prices per supplier, the **best price per line in green**, the subtotals, the **coverage** (number of lines quoted out of the total), the lead time and quality, **"Select"** checkboxes, and the **"Award → generate PO"** button that creates one purchase order per selected supplier.
- **Actions**: Send (to several suppliers), Generate HTML, Preview, Excel, Copy comparison CSV.

#### 2.3.3 Request for quote windows

- **Create** (five sections: Information, Items, Suppliers to invite, Delivery, Notes).
- **HTML preview** of the request.
- **Bulk send**: selection of several suppliers, with a send report (sent / email failures / failures).
- **Edit the header**.
- **Enter prices — {supplier}**: internal entry of each line's price, plus lead time, terms and note (for a quote received by phone or in person).

#### 2.3.4 Public supplier portal

An invited supplier receives a **secure link** (signed token). Opening it, they see the request, enter their prices, lead time and terms, then submit. The portal is **public** (no account required) but protected: signed token with a limited lifetime, a cap on the number of requests per hour, refusal if the request is closed or expired, one token = one supplier.

### 2.4 "Material Prices" tab

An assistant that compares the price of a list of materials at major Quebec retailers. A toggle separates two modes: **Comparator** and **Web search**.

#### 2.4.1 Comparator mode

You paste a list of materials into the chat window. The system queries **Canac**, **BMR** and **Home Depot** in parallel and returns:

- a **comparison table** per item (best price checked in green, link to the retailer's listing);
- the **total per supplier**;
- the **"Cheapest plan"** (buy everything at the lowest price, item by item);
- a **store-optimized purchase plan** (grouping purchases by banner), with the savings achieved in dollars and as a percentage.

A **"Create a purchase order (draft)"** button turns a plan into a PO: you choose the retailer and the corresponding supplier, and a draft PO is generated.

> **External dependency**: this comparator queries **unofficial** web services of the retailers. Home Depot is particularly fragile (often blocked). Partial results are therefore normal. This tab may even be **absent** if the service could not start on the server.

#### 2.4.2 Web search mode

A free-form query launches a **web search** (Claude with web search) and returns a synthesis text, the **cited sources**, as well as the number of searches, the duration and the cost.

### 2.5 "Movements" tab

Full stock traceability.

#### 2.5.1 Action bar and filters

- **New movement**.
- **Filters**: Type (All / In / Out / Adjustment), Product, Project, Employee, Date from, Date to, **Search** (product, reference, reason) and **Reset** (visible when filters are active).

#### 2.5.2 Movements table

Columns: **Product** (with the "Cancelled" badge where applicable) · **Type** (green badge for In, red for Out, blue for Adjustment) · **Quantity** (with the unit) · **Stock before → after** · **Employee** · **Project / PO** · **Cost** · **Reference** · **Date**. Clicking a row opens the detail.

#### 2.5.3 "New movement" window

Five sections:

1. **Type**:
   - **In** (receipt, purchase, return) → increases stock;
   - **Out** (send to the job site, use) → decreases stock;
   - **Adjustment** (inventory correction) → sets stock to the counted value.
2. **Item**: selection menu with the stock displayed.
3. **Stock and quantity**: live preview of **current stock → after**, with a red **"Insufficient stock for this outbound movement!"** alert if the outbound exceeds stock.
4. **Details**: Reference, Movement date (today by default, **backdating possible**), Reason.
5. **Advanced**: Responsible employee, Associated project, Associated PO, Unit cost (with live total calculation).

The final button changes with the type: "Record inbound", "Record outbound" or "Adjust stock".

#### 2.5.4 Movement detail window

Shows the badges (type, cancelled, reversing movement), the item, the before/after quantities, the cost, the context (reference, reason, employee, project, PO), the dates, and the **"Cancel this movement"** button. Cancelling creates an **opposite reversing movement** (an In cancels an Out and vice versa). The button is disabled if the movement is already cancelled or is itself a reversing movement.

### 2.6 "Products" tab

Catalog of items and bills of materials.

#### 2.6.1 Action bar

- **Add a New Item**.
- **Low stock** (toggle): shows only the products below their threshold.
- **Category filter** (categories actually present in the database).
- **Search** (name, code).

#### 2.6.2 Products table

Columns (sortable, resizable): **Product** (name + code) · **Category** · **Stock** (with the unit) · **Threshold** · **Selling price** · **Status** (red "Low" badge if stock is at or below the threshold and the threshold is greater than 0, otherwise green "OK") · **Actions** (**Label** barcode and **Edit**). Clicking a row opens the bill of materials panel. If no product: "No product".

#### 2.6.3 Bill of materials (BOM) panel

- **Components**: the list of child products (product, quantity, unit, unit price, stock, notes) with deletion, plus an add form (choose a component product, Quantity, Unit, Notes).
- **Used in**: the list of parent products that use this product as a component (reverse dependencies).

The server refuses a self-reference, a duplicate and **any circular reference** (A that would contain B that would contain A).

#### 2.6.4 Barcode label

The **Label** button downloads a **Code128 PDF** label. If the product does not yet have a code, the server generates one. The label opens in a new tab (with a download fallback).

#### 2.6.5 "Create / Edit an item" window

| Field | Notes |
|-------|-------|
| **Name** | Required |
| **Initial quantity** (creation) / **Current quantity** (editing) | When editing, this field is **read-only** — a hint points to creating an **adjustment** movement |
| **Internal code** | Automatically generated (`ART-00001`) if left empty |
| **Minimum limit** | Alert threshold |
| **Barcode** | EAN / UPC |
| **Product type** | 13 choices (see §4.7) |
| **Sales unit** | `un`, `m²`, `pi²`, `kg`, etc. |
| **Main supplier** | Free text |
| **Stock location** | Free text |
| **Selling price ($)** | Entered by hand |
| **Cost price ($)** | Entered by hand |
| **Applicable standard** | 7 choices (see §4.7) |
| **Description** | Free text |
| **Notes** | Free text |

> **Stock cannot be edited here when editing.** To correct a quantity, you must go through an **adjustment** movement (Movements tab). This is the only path that leaves an audit trail.

### 2.7 "Supplier" tab

#### 2.7.1 Action bar

- **New Supplier**.
- **Supplier order** (shortcut to creating a PO).
- **Search**.

#### 2.7.2 Suppliers table

Columns: **Supplier** (name + email) · **Category** · **Contact** · **City** · **Rating** (score out of 5) · **Status** (Active / Inactive) · **Actions** (Edit). Double-clicking opens editing. If no supplier: "No supplier".

#### 2.7.3 "Create a supplier" window

Fields: **Company** (required — chosen among the companies in the directory) · Supplier Code · Payment Terms (default "Net 30 days") · Product Category (11 choices) · Sales Contact · Delivery Lead Time (days) · Technical Contact · **Quality Rating /10** (slider) · **Construction Certifications** (10 checkboxes: **RBQ** (Quebec building authority), **CCQ** (Quebec construction commission), **CNESST** (Quebec occupational health & safety board), **ISO 9001**, **BNQ** (Quebec standards bureau), **CSA**, **LEED**, **GCR Warranty** (residential construction warranty), **ACQ** (Quebec Construction Association), **APCHQ** (Quebec home builders' association)) · Evaluation Notes.

#### 2.7.4 "Edit a supplier" window

Name, Category, Payment terms, Contacts, Lead time, Rating /10, Notes, Evaluation notes, and the **"Active supplier"** checkbox (uncheck to remove the supplier from the menus while keeping it).

### 2.8 "AI Assistant" tab

An "AI Assistant — Store" chat window that:

- **reads** the real Store data (number of products, stockouts, categories, stock value, etc.);
- can **offer to create a product**: the AI displays a **proposal card** that the user must **Confirm** (or Cancel) — the write happens only on confirmation.

Each message shows the number of tokens, the cost and the duration. Examples of suggested questions: "which products are below their threshold?", "create a product…", "what is the stock value by category?".

> **Deliberately restricted scope**: the Store assistant creates **only products**. It touches **neither stock nor purchase orders** (more sensitive operations, reserved for the dedicated screens). Confirmation re-checks that the user does indeed have the inventory write right.

---

## 3. Step-by-step processes

### 3.1 Create a product

1. **Products** tab → **Add a New Item**.
2. Fill in at least the **Name**. Optional: initial quantity, threshold, code (generated if empty), unit, selling price, cost price, category, standard.
3. **Save**. The product appears in the table. Its badge is "OK" (or "Low" if the initial stock is already at the threshold).

### 3.2 Correct a product's stock (physical inventory)

Stock is never edited on the product record. To adjust it:

1. **Movements** tab → **New movement**.
2. **Adjustment** type, choose the item.
3. Enter the **actual counted quantity** (absolute value, not a variance).
4. Optional: reason ("Inventory 2026-Q1"), employee, date.
5. **Adjust stock**. The movement records the stock before, the stock after and the variance, for audit purposes.

### 3.3 Create a supplier

1. **Supplier** tab → **New Supplier**.
2. Choose the **Company** (required). Fill in the payment terms, the category, the contacts, the lead time, the rating, the certifications.
3. **Save**. The supplier is **Active** by default.

### 3.4 Create a purchase order

1. **Orders** tab → **Supplier order** (or **Supplier** tab → **Supplier order**).
2. Choose the **supplier** (required), the project, the employee, the delivery date.
3. Add the **items**: choose a product from the catalog (description, unit and price fill in) or enter a line by hand. Taxes are calculated live.
4. Fill in the delivery address and terms, and internal notes.
5. **Create the purchase order**. It appears in **Draft** status, numbered `BC-00001`.

### 3.5 Scan a supplier purchase order (AI)

1. **Orders** tab → **Scan a PO (AI)**.
2. Choose an **image** or a **PDF** of the purchase order (up to 20 MB).
3. Wait for the analysis. A banner shows the number of extracted lines, the confidence and the total.
4. Check / correct the proposed lines, choose the supplier if not recognized.
5. **Create the purchase order**.

> The scan consumes **AI credits** (see §4.6). It creates nothing until you have confirmed.

### 3.6 Send a purchase order to the supplier

1. Open the PO → **Send to supplier**.
2. Enter the supplier's email address, then send.
3. The system creates a **public link** (valid 90 days), moves the PO to **Sent** and dispatches the email. You can copy the link to share it another way.

### 3.7 Receive a purchase order (replenish stock)

1. Open the PO → in the header, change the status to **Received**.
2. The system, **automatically and atomically**:
   - for each line linked to a catalog product, **increases stock** by the received quantity;
   - creates an **inbound movement** per product (reference "Réception BC-xxxxx"), with the unit cost.
3. The PO now shows **Received**, and the movements appear in the Movements tab.

> **Final receipt.** Once **Received**, the PO can no longer be **reverted** to another status (the system returns an error to protect inventory). If you made a mistake, **cancel the generated stock movements** (Movements tab, "Cancel this movement" button).

### 3.8 Create a supplier invoice from a purchase order

1. Open the PO → **Create invoice**.
2. A **draft supplier invoice** is created in the Accounting module, carrying over the PO lines.
3. The button is disabled if the PO is already **Invoiced**.

### 3.9 Launch a request for quote (call for bids)

1. **Requests for Quote** tab → **New request**.
2. Fill in the title, the project, the employee, the deadline.
3. Add the **items to quote** (description, quantity, unit — **without prices**).
4. Choose the **suppliers to invite**.
5. **Create**. The request is numbered `RFQ-00001`.
6. **Invite / Send**: per supplier, click **Send** to transmit the public link, or use bulk send.

### 3.10 Collect prices and award

1. When suppliers respond (through the public link or through **Enter price** internally), their status changes to **Responded**.
2. Click **View comparison**: the best price per line stands out in green, with the subtotals and the quoting coverage.
3. Check **Select** for the winning supplier(s).
4. Click **Award → generate PO**. The system:
   - keeps only the **quoted lines** (refuses a $0 PO);
   - creates **one purchase order per selected supplier**;
   - marks the winners **Selected**, the others **Not selected**, and the request **Awarded**;
   - updates the supplier's purchase price in the catalog (price history).

> An **Awarded** request can no longer be deleted.

### 3.11 Compare retailer prices and generate a PO

1. **Material Prices** tab → **Comparator** mode.
2. Paste the list of materials into the chat window.
3. Read the comparison: best price per item, total per supplier, cheapest plan, store-optimized plan.
4. To order: **Create a purchase order (draft)**, choose the retailer and the corresponding supplier. A draft PO is generated in the Orders tab.

### 3.12 Create a manual stock movement

- **In** (receipt outside a PO, return from the job site): In type, item, quantity, reference, cost. Stock increases.
- **Out** (send to the job site, loss): Out type, item, quantity. The system refuses if stock is insufficient.
- **Adjustment** (inventory): see §3.2.

### 3.13 Cancel a stock movement

1. **Movements** tab → click the movement → **Cancel this movement**.
2. The system creates a **reversing movement** (reference prefixed "ANNUL-") and marks the original as cancelled.
3. You cannot cancel an already-cancelled movement, or a reversing movement.

### 3.14 Declare a bill of materials (BOM)

1. **Products** tab → click the **parent product** (the assembly).
2. In **Components**, choose a child product, its quantity, its unit, a note, then add.
3. Repeat for each component. The child's **Used in** section will show the parent.

> The bill of materials is **declarative**: assembling a product does not consume the components in stock. For that, make an outbound movement of each component and an inbound movement of the assembly (or go through a work order).

### 3.15 Print a barcode label

1. **Products** tab → on the product's row, click **Label**.
2. A **Code128 PDF** label opens in a new tab, ready to print.

### 3.16 Use the AI assistant

1. **AI Assistant** tab.
2. Ask a question ("which products are below their threshold?") or ask to create a product.
3. For a creation, the AI displays a **proposal card**: check it, then **Confirm**. The product is created.

---

## 4. Reference

### 4.1 Main endpoints (API)

All prefixed with `/api/erp/v1`.

**Products and statistics** (`inventory.py`)

| Method + path | Role | Right |
|---|---|---|
| GET `/products` | List (filters: search, category, low stock) | read |
| GET `/products/categories` | Distinct categories | read |
| GET `/products/scan` | Resolution by barcode then product code | read |
| GET `/products/{id}` | Detail | read |
| POST `/products` | Create | inventory write |
| PUT `/products/{id}` | Edit (stock is excluded) | inventory write |
| GET `/products/{id}/label` | Code128 PDF label | inventory write |
| GET `/inventory/stats` | 4 KPI cards | read |

**Movements** (`inventory.py`)

| Method + path | Role | Right |
|---|---|---|
| POST `/stock-movements` | Create a movement | inventory write |
| GET `/stock-movements` | List + filters | read |
| GET `/stock-movements/{id}` | Detail | read |
| POST `/stock-movements/{id}/cancel` | Cancel (reversing movement) | inventory write |

**Bill of materials (BOM)** (`inventory.py`)

| Method + path | Role | Right |
|---|---|---|
| GET `/products/{id}/composants` | Components + "used in" | read |
| POST `/products/{id}/composants` | Add | inventory write |
| PUT `/products/{id}/composants/{cid}` | Edit | inventory write |
| DELETE `/products/{id}/composants/{cid}` | Remove | inventory write |

**Suppliers and purchase orders** (`suppliers.py`)

| Method + path | Role | Right |
|---|---|---|
| GET `/suppliers` | List | read |
| GET `/suppliers/{id}` | Detail (+ last 20 POs) | read |
| POST `/suppliers` | Create | purchasing write |
| PUT `/suppliers/{id}` | Edit | purchasing write |
| GET `/suppliers/purchase-orders` | All POs | read |
| POST `/suppliers/{id}/orders` | Create a PO | purchasing write |
| POST `/suppliers/orders/ai/scan` | Scan a PO (Vision AI) | purchasing write |
| PUT `/suppliers/purchase-orders/{id}` | Edit a PO | purchasing write |
| PUT `/suppliers/purchase-orders/{id}/dates` | Edit the dates | purchasing write |
| PUT `/suppliers/purchase-orders/{id}/status` | Edit the status | purchasing write |
| POST `/suppliers/orders/{id}/lines` | Add a line | purchasing write |
| DELETE `/suppliers/orders/{id}/lines/{lid}` | Delete a line | purchasing write |
| DELETE `/suppliers/purchase-orders/{id}` | Delete the PO | purchasing write |
| POST `/suppliers/orders/{id}/generate-html` | Printable HTML | read |
| POST `/suppliers/orders/{id}/send` | Send by email | purchasing write |
| GET `/suppliers/orders/public/{token}` | Public view (no account) | token |
| GET `/suppliers/orders/{id}/export-xlsx` | Excel export | read |

**Requests for quote (RFQ)** (`rfq.py`): `/rfq/demandes` (create, list, detail, edit, delete), `/rfq/demandes/{id}/lignes`, `/rfq/demandes/{id}/fournisseurs` (invite, remove, send), `/rfq/demandes/{id}/reponses/{rid}/lignes` (enter/read prices), `/rfq/demandes/{id}/comparatif`, `/rfq/demandes/{id}/octroi`, `/rfq/demandes/{id}/generate-html`, `/rfq/demandes/{id}/export-xlsx`. **Public portal**: `/public/rfq/{token}` (load) and `/public/rfq/{token}/submit` (submit) — without authentication, protected by a signed token.

**Material prices** (`materials_pricing.py`): `/materials/pricing/search`, `/materials/pricing/ai/chat`, `/materials/pricing/web-search`. **No database writes.**

**AI assistant** (`magasin_ai.py`): `/magasin/ai/chat` (query + propose), `/magasin/ai/confirm-action` (execute the confirmed creation).

### 4.2 Statuses

| Object | Statuses |
|---|---|
| **Purchase order** | Draft · Sent · Confirmed · In progress · Received · Invoiced · Cancelled |
| **Request for quote** | Draft · Sent · Closed · Awarded · Cancelled |
| **Supplier response (RFQ)** | Invited · Responded · Selected · Not selected |

### 4.3 Stock movement types

| Type | Effect on stock | Rule |
|---|---|---|
| **In** | `after = before + quantity` | quantity > 0 |
| **Out** | `after = before − quantity` | quantity > 0 **and** cannot exceed stock (otherwise refused) |
| **Adjustment** | `after = quantity` (target value) | quantity ≥ 0 |

> There is **no** "Transfer" type (the module does not manage multiple sites).

### 4.4 Calculations

| Element | Formula |
|---|---|
| **"Low" status** | `stock_disponible ≤ stock_minimum` **and** `stock_minimum > 0` |
| **Stock value** (KPI) | `Σ [ max(stock, 0) × (cost price, otherwise selling price, otherwise 0) ]` |
| **PO line amount** | `round(quantity × unit price, 2)` |
| **PO subtotal** (before tax) | `Σ of the line amounts` |
| **Tax 1 / Tax 2 on the PO** | `round(subtotal × rate / 100, 2)` (rates specific to the PO) |
| **PO total** (tax included) | `subtotal + tax 1 + tax 2` |
| **Cost of a receipt** | Per product: `total amount of the lines / total quantity` (weighted average **of the lines of that receipt**) |

> **Important note on cost.** The receipt's "weighted average" aggregates only the lines of the **same purchase order** for a given product. It feeds the **movement cost** (audit and accounting) but **does not recalculate** the product's cost price as a moving average over the total stock value. **There is no moving weighted average cost**: a product's cost price remains the value **entered by hand** on its record.

### 4.5 Taxes

A purchase order's taxes are **frozen at creation** according to the tenant's configuration (multi-province Canada or United States). Default values in Quebec: **GST 5%**, **QST 9.975%**. An empty and legitimate tax label (exemption) is preserved. Taxes are recalculated on display from the subtotal.

### 4.6 Money effect and AI credits

The Store module **does not bill anything through Stripe or QuickBooks directly**. The **only monetary effect** is the **debit of the tenant's prepaid AI credits**, for four functions:

| Function | AI model |
|---|---|
| Store AI Assistant (chat) | Sonnet |
| Material Prices (comparator chat) | Sonnet |
| Material Prices (web search) | Opus (web search) |
| PO scan (Vision) | Sonnet |

The cost billed to the tenant is the **actual token cost** ($0.003 per thousand input, $0.015 per thousand output) **marked up by 30%**. If the credits are exhausted, the call is refused. The service is unavailable if the AI is not configured.

### 4.7 Value lists

**13 product types** ("Product type" field):

```
Concrete and cement · Wood and framing · Steel and metal · Plumbing ·
Electrical · Insulation · Roofing · Paint and finishing ·
Hardware · Cladding · Tools · PPE / Safety · Other
```

**11 product categories** (supplier record): the same as above, **plus** "Equipment rental", **without** "Steel and metal" (slightly different order) — namely: Concrete and cement · Wood and framing · Steel and metal · Plumbing · Electrical · Insulation · Roofing · Paint and finishing · Hardware · Equipment rental · Other.

**7 applicable standards** ("Applicable standard" field):

```
CSA · ASTM · BNQ · ULC · ISO · LEED · Other
```

**10 supplier certifications** (checkboxes):

```
RBQ · CCQ · CNESST · ISO 9001 · BNQ · CSA · LEED ·
GCR Warranty · ACQ · APCHQ
```

### 4.8 Limits and safeguards

| Rule | Effect |
|---|---|
| Out greater than stock (manual movement) | Refused |
| Revert a PO already "Received" | Refused — cancel the stock movements instead |
| Edit a line of a "Received" or "Invoiced" PO | Refused |
| Delete a "Received", "Invoiced" or already-posted PO | Refused |
| Delete an "Awarded" request for quote | Refused |
| Award an RFQ with no quoted line | Refused (no $0 PO) |
| Edit stock directly on the product record | Impossible — go through an adjustment movement |
| Create a looping BOM component (circular reference) | Refused |
| Out-of-bounds amounts (numeric overflow) | Cleanly refused (error message, no crash) |
| PO scan file larger than 20 MB | Refused |
| RFQ public portal: too many requests / hour / address | Rate-limited |

### 4.9 PostgreSQL tables (per tenant)

| Table | Role |
|---|---|
| `produits` | Catalog (name, code, price, cost, stock, threshold, category, standard, location, active) |
| `mouvements_stock` | Movement history (type, before/after quantities, cost, employee, project, PO, cancellation) |
| `produit_composants` | Parent-child bill of materials (unique per pair) |
| `fournisseurs` | Supplier records (linked to a company in the directory) |
| `bons_commande` | PO headers (number, supplier, project, status, taxes, totals, public token) |
| `bon_commande_lignes` | PO lines (also shared by the requests for quote, via role columns) |
| `rfq_demandes` / `rfq_reponses` | Request for quote headers and supplier responses |
| `produit_fournisseurs` / `produit_historique_prix` | Purchase price per supplier and history (fed when an RFQ is awarded) |
| `public.bc_public_tokens` | Public PO link tokens (90 days) |

---

## 5. Integrations and FAQ

### 5.1 Links with other modules

| Module | Link |
|---|---|
| **Case files (360 view)** | A PO can be opened directly from a case file (`/magasin?open=<id>`); awarding an RFQ and creating a PO can be attached to the project's case file. |
| **Projects** | A PO, a movement and a request for quote can be associated with a project. |
| **Work orders** | Material issues to the job site often go through work orders; this module traces the stock counterpart. |
| **Accounting** | The **Create invoice** button generates a draft supplier invoice; receiving a PO triggers, through the stock movement, the general ledger entry (triggered by the database). |
| **Configuration** | The tax rates (GST/QST, provinces, United States), the logo and the document theme come from the company's Configuration. |
| **AI credits** | The Store's AI functions consume the tenant's prepaid credits (see §4.6). |
| **Employees** | The responsible person for a PO and the author of a movement are chosen among active employees. |

### 5.2 Recommended purchasing cycle

1. (Optional) **Request for quote** to several suppliers → comparison → **award** (generates the POs).
2. **Purchase order**: complete it, **send** it to the supplier.
3. On delivery: move the PO to **Received** → **stock is replenished** automatically.
4. (Optional) **Create** a supplier **invoice** → Accounting.

### 5.3 FAQ

**Does receiving a purchase order update stock automatically?**
Yes. Moving a PO to **Received** increases the stock of the linked products and creates an inbound movement per product. (This is a major change from older versions, where receiving had no effect on stock.)

**Can I cancel a receipt made by mistake?**
You cannot move the PO back (it stays "Received" for audit). The correction is done by **cancelling the generated stock movements** (Movements tab → "Cancel this movement"), which recreates an opposite reversing movement.

**How do I correct a stock quantity?**
Only through an **adjustment** movement. The "Current quantity" field on the product record is read-only when editing.

**Is there an automatic weighted average cost?**
No. The **selling price** and the **cost price** are entered by hand on the product record. The cost of a receipt movement (average of that PO's lines) is used for audit and accounting, but does not update the product's cost price.

**Can I manage several warehouses or make inter-site transfers?**
No. Stock is **global per product**. The "Stock location" field is a simple informational text. There are no multiple warehouses, no transfers, and no "Transfer" movement type.

**Is there a dashboard or dedicated category management in the Store?**
No. Some internal labels (warehouses, dashboard, categories, transfers) remain in the code as vestiges of an older version **but are rendered nowhere**. The only real tabs are the 7 described here.

**Is there a "purchase vouchers" or "requisitions" tab?**
No. There is **no** separate "purchase voucher" document in the Store. The closest thing to a requisition is the **Request for quote** (RFQ). (Some "purchase voucher" orders exist in external technical tools, but they do not correspond to any tab on this page.)

**Does the supplier need an account to view a PO or respond to a request for quote?**
No. The sent PO generates a **public link** (90 days). The request for quote generates a **secure link** per supplier. No account is required, but access is protected by a signed token with a limited lifetime.

**Does a field employee see the purchase costs?**
No. The cost price and the movement costs are **hidden** from field roles (the "employee" role). Only management roles see them.

**Is the price comparator (Canac / BMR / Home Depot) 100% reliable?**
No — it queries **unofficial** web services of the retailers. Partial results are normal (Home Depot is often blocked). The tab may even be absent if the service did not start. Always check prices before ordering.

**Does the PO scan directly create an order?**
No. It **extracts** the lines into the creation window; you must check them and then **Create the purchase order**. It consumes AI credits.

**Can I delete a purchase order?**
Yes, as long as it is neither **Received**, nor **Invoiced**, nor already posted. Otherwise deletion is refused.

**Can the AI assistant modify stock or create a purchase order?**
No. Its only write action is **product creation**, and only after human confirmation.

**Can I bulk import a catalog (CSV/Excel)?**
There is no bulk import button on this page. You can export a PO (Excel, QuickBooks CSV) and an RFQ comparison, but catalog import goes through other tools.

### 5.4 What does not exist (known limits)

- No multiple warehouses, no inter-site transfers, no "Transfer" movement.
- No dedicated dashboard, no dedicated category management in the Store.
- No "purchase vouchers / requisitions" tab (the RFQ fills that role).
- No moving weighted average cost on inventory value.
- No direct stock editing on the record (only through a movement).
- No lot tracking, expiry dates, or nested units (e.g. "1 bag = 25 kg").
- No dedicated print button (printing goes through the HTML preview "Open in a new tab").
- No bulk catalog import from this page.

---

## 6. Summary

- The **Store** (`/magasin`, Operations section) brings together **7 tabs**: **Orders**, **Requests for Quote**, **Material Prices**, **Movements**, **Products**, **Supplier**, **AI Assistant**. Default tab: **Orders**.
- **4 KPI cards** at all times: Products, Minimum threshold alert, Stock value, Categories.
- **Purchase orders**: 7 statuses (Draft → Sent → Confirmed → In progress → **Received** → Invoiced, plus Cancelled), email sending with a **90-day public link**, AI PO scanning, HTML / Excel / QuickBooks CSV export, supplier invoice creation.
- **Receiving a PO (status "Received") automatically replenishes stock** and creates an inbound movement per product. **Final receipt**: to correct, you cancel the movements.
- **Movements**: 3 types (In, Out, Adjustment — **no** Transfer), full traceability (stock before/after, employee, project, PO, cost), cancellation via a reversing movement, backdating possible.
- **Stock is never edited by hand**: only through an **adjustment** movement.
- **Selling price and cost price are entered manually**: **no moving weighted average cost**.
- **Requests for quote (RFQ)**: call for bids to several suppliers, **public portal** via a signed token, **comparison** (best price per line, coverage), **award** that generates one PO per selected supplier.
- **Material Prices**: Canac / BMR / Home Depot comparator (unofficial services, partial results possible) + web search, with draft PO creation.
- **AI Assistant**: reads the data, **creates products on confirmation** (nothing else).
- **Permissions**: open reading; inventory write (manager / storekeeper / admin); purchasing write (includes the user role); RFQ write (manager / storekeeper / admin). **Costs hidden** from field roles.
- **Money effect** limited to **prepaid AI credits** (4 AI functions, actual token cost marked up by 30%).
- **Does not exist**: multiple warehouses, transfers, dedicated dashboard, purchase vouchers, moving average cost, lot/expiry tracking, bulk catalog import.

---

**Documentation generated from the source code**: `MagasinPage.tsx`, `RfqTab.tsx`, `MaterialPricingTab.tsx`, `MaterialWebSearch.tsx`, `MagasinAssistantTab.tsx`, `api/inventory.ts`, `api/suppliers.ts`, `api/rfq.ts`, `api/magasinAi.ts`; `backend/routers/inventory.py`, `suppliers.py`, `rfq.py`, `materials_pricing.py`, `magasin_ai.py`.

**Related manuals**:
- Module 11 — Work Orders (material issues) — `11-operations-bons-de-travail.md`
- Module 13 — Purchase Orders / Purchasing (complementary view) — `13-operations-bons-de-commande.md`
- Module 14 — Accounting (supplier invoices, general ledger) — `14-operations-comptabilite.md`
- Module 08 — Projects (association of POs and movements) — `08-ventes-projets.md`
- Module 06 — Case Files / 360 View (direct opening of a PO) — `06-ventes-dossiers.md`
- Module 30 — Configuration (taxes, document theme) — `30-configuration.md`
- Module 24 — AI Assistant (AI credits) — `24-communication-assistant-ia.md`
