# User Manual — Constructo AI ERP

Integrated management software for construction companies in Quebec.

> **Version**: 3.0 — complete overhaul verified against the source code
> **Date**: 2026-07-08
> **Audience**: end users (project managers, managers, accountants, field employees, administrators)
> **Languages**: English (this folder) and French (parent folder [`../README.md`](../README.md))

---

## Documentation status

This folder contains **38 user manuals**, one per module and sub-module of the Constructo AI ecosystem, in sidebar-menu order, followed by the co-hosted applications (Real Estate, B2B/B2C Portal, SEAOP, Estimation Express, Mobile Time Clock).

| Status | Meaning |
|--------|---------|
| ✅ v3.0 code-verified | Each manual was written from the **current source code** (backend routers, React pages, i18n files), reviewed linguistically, then translated into English. |

Every statement is traceable to the code (`ERP_REACT/backend/routers/*.py`, `frontend/src/pages/*.tsx`, `frontend/src/api/*.ts`, and the separate applications `IMMO_REACT`, `SEAOP_REACT`, `ESTIMATION_EXPRESS_REACT`, `MOBILE_REACT`).

**French version**: each manual has its French counterpart in the parent folder, under the same file name. See [`../README.md`](../README.md).

---

## Table of contents — by menu section

### Main

| # | Module | File |
|---|--------|------|
| 01 | Dashboard and statistics | [01-principal-tableau-de-bord.md](01-principal-tableau-de-bord.md) |
| 02 | Tracking (Kanban, Gantt, Calendar) | [02-suivi-gantt.md](02-suivi-gantt.md) |

### Management

| # | Module | File |
|---|--------|------|
| 03 | Companies (clients and suppliers) | [03-gestion-entreprises.md](03-gestion-entreprises.md) |
| 04 | Contacts | [04-gestion-contacts.md](04-gestion-contacts.md) |
| 05 | Sales (CRM, pipeline, B2B back-office) | [05-gestion-crm-opportunites.md](05-gestion-crm-opportunites.md) |
| 06 | Files (document management) | [06-ventes-dossiers.md](06-ventes-dossiers.md) |
| 07 | Quotes and estimates (manual, AI, import) | [07-ventes-soumissions.md](07-ventes-soumissions.md) |
| 08 | Construction projects | [08-ventes-projets.md](08-ventes-projets.md) |

### Operations

| # | Module | File |
|---|--------|------|
| 09 | Store (products, inventory, purchasing, RFQ) | [09-operations-magasin.md](09-operations-magasin.md) |
| 10 | Employees (human resources) | [10-operations-employes.md](10-operations-employes.md) |
| 11 | Work orders | [11-operations-bons-de-travail.md](11-operations-bons-de-travail.md) |
| 12 | Time tracking (+ CCQ payroll, T4/RL-1/PD7A slips) | [12-operations-pointage.md](12-operations-pointage.md) |
| 13 | Purchase orders (supplier purchasing) | [13-operations-bons-de-commande.md](13-operations-bons-de-commande.md) |
| 14 | Accounting (general ledger, invoices, WIP, holdbacks) | [14-operations-comptabilite.md](14-operations-comptabilite.md) |

### Field

| # | Module | File |
|---|--------|------|
| 15 | Site weather | [15-terrain-meteo-chantier.md](15-terrain-meteo-chantier.md) |
| 16 | RBQ / CCQ compliance | [16-terrain-conformite.md](16-terrain-conformite.md) |
| 17 | Grants and government assistance | [17-terrain-subventions.md](17-terrain-subventions.md) |
| 18 | Logistics and deliveries | [18-terrain-logistique.md](18-terrain-logistique.md) |
| 19 | Equipment and labour rental | [19-terrain-location.md](19-terrain-location.md) |
| 20 | Maintenance (preventive, corrective) | [20-terrain-maintenance.md](20-terrain-maintenance.md) |

### Communication

| # | Module | File |
|---|--------|------|
| 21 | Email (IMAP/SMTP) | [21-communication-emails.md](21-communication-emails.md) |
| 22 | Internal messaging | [22-communication-messagerie.md](22-communication-messagerie.md) |
| 23 | AI voice agent (virtual receptionist) | [23-communication-agent-vocal.md](23-communication-agent-vocal.md) |
| 24 | AI Assistant (general AI engine) | [24-communication-assistant-ia.md](24-communication-assistant-ia.md) |

### 3D Design

| # | Module | File |
|---|--------|------|
| 25 | CAD / 3D modelling | [25-outils-dao-modelisation.md](25-outils-dao-modelisation.md) |
| 26 | PDF3D (plan to 3D) | [26-conception3d-pdf3d-hover.md](26-conception3d-pdf3d-hover.md) |
| 27 | Photorealistic 3D rendering | [27-conception3d-rendu-3d.md](27-conception3d-rendu-3d.md) |

### Tools

| # | Module | File |
|---|--------|------|
| 28 | Construction calculators | [28-outils-calculateurs.md](28-outils-calculateurs.md) |
| 29 | Web (real-time search) | [29-outils-web.md](29-outils-web.md) |
| 32 | Takeoff (quantity takeoff on PDF) | [32-metre-pdf.md](32-metre-pdf.md) |

### System

| # | Module | File |
|---|--------|------|
| 30 | Configuration (company, users, subscription) | [30-configuration.md](30-configuration.md) |
| 31 | Accounting integrations (QuickBooks, Sage) | [31-integrations-comptables.md](31-integrations-comptables.md) |
| 33 | Help and resources | [33-aide-ressources.md](33-aide-ressources.md) |

### Programs (co-hosted applications)

| # | Program | File |
|---|---------|------|
| 34 | Real estate (developer and public showcase) | [34-terrain-immobilier.md](34-terrain-immobilier.md) |
| 35 | B2B / B2C portal (clients and suppliers) | [35-programme-portail-b2b.md](35-programme-portail-b2b.md) |
| 36 | SEAOP (Quebec public tenders) | [36-programme-seaop.md](36-programme-seaop.md) |
| 37 | Estimation Express (paid public sub-application) | [37-programme-estimation-express.md](37-programme-estimation-express.md) |
| 38 | Mobile time clock (PWA app) | [38-programme-pointeur-mobile.md](38-programme-pointeur-mobile.md) |

---

## Getting started

### 1. Sign in

1. Open the application in your browser.
2. Enter your company email.
3. Enter your username and password.
4. Click "Sign in".

### 2. Navigation

- Use the **left sidebar** to move between modules.
- The **dashboard** shows your key indicators as soon as you sign in.
- The **AI Assistant** (top-right button) answers your questions in natural language.

### 3. Help

- Each module has its manual in this folder (English) and in the parent folder (French).
- Support: **info@constructoai.ca** | **1 (936) 587-1141**.

---

## System highlights

- **Native Quebec compliance**: RBQ, CCQ, GST/QST, CNESST, Bill 16, Bill 25.
- **Artificial intelligence**: specialized assistants (Claude), estimating, vision, 3D rendering, voice agent.
- **Trade calculators**: tools compliant with CCQ, CSA, NBC, ASTM and AWS standards.
- **Quebec payroll**: QPP, QPIP, EI, HSF, CNESST, CCQ, T4 and RL-1 slips.
- **Accounting integrations**: QuickBooks Online (OAuth 2.0), Sage.
- **B2B/B2C client portal**, **SEAOP public tenders**, **public estimating** and a **mobile** time-clock app.
- **Multi-company (multi-tenant) architecture**.

---

## Support

- **AI Assistant**: available inside the application.
- **Email**: info@constructoai.ca
- **Phone**: 1 (936) 587-1141

---

© 2026 Constructo AI — All rights reserved | Developed by Sylvain Leduc
