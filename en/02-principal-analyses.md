# Module 02 — Analytics (merged into the Dashboard)

> **Version**: 3.0 — This module no longer exists as a separate page.
> **See**: [Module 01 — Dashboard and statistics](01-principal-tableau-de-bord.md)

---

## Analytics now lives in the Dashboard

Following the ERP overhaul, the "Analytics" page is **no longer a separate module**. The Dashboard and Analytics have been **merged** into a single six-tab page, reachable from the sidebar → **Dashboard** (route `/dashboard`).

- The former `/analyses` address **redirects automatically** to `/dashboard` (`App.tsx:218-219`).
- There is no longer an "Analytics" entry in the sidebar: only a single **"Dashboard"** entry remains (`Sidebar.tsx:40`).
- All analytical content — the **Global View**, **Projects**, **Finances**, **HR**, **Stock** and **AI Assistant** tabs — is documented in **Module 01**.

See **[Module 01 — Dashboard and statistics](01-principal-tableau-de-bord.md)** for the full detail of the indicators, charts, tables, the period selector and the natural-language analysis assistant.

---

**Related manuals**:
- [Module 01 — Dashboard and statistics](01-principal-tableau-de-bord.md)
