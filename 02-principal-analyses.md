# Module 02 — Analyses (fusionné dans le Tableau de bord)

> **Version** : 3.0 — Ce module n'existe plus comme page distincte.
> **Voir** : [Module 01 — Tableau de bord et statistiques](01-principal-tableau-de-bord.md)

---

## Les Analyses sont désormais dans le Tableau de bord

Depuis la refonte de l'ERP, la page « Analyses » n'est **plus un module séparé**. Le Tableau de bord et les Analyses ont été **fusionnés** en une seule page à six onglets, accessible par le menu latéral → **Tableau de Bord** (route `/dashboard`).

- L'ancienne adresse `/analyses` **redirige automatiquement** vers `/dashboard` (`App.tsx:218-219`).
- Il n'y a plus d'entrée « Analyses » dans le menu latéral : une seule entrée **« Tableau de Bord »** subsiste (`Sidebar.tsx:40`).
- Tout le contenu analytique — les onglets **Vue Globale**, **Projets**, **Finances**, **RH**, **Stock** et **Assistant IA** — est documenté dans le **module 01**.

Consultez le **[Module 01 — Tableau de bord et statistiques](01-principal-tableau-de-bord.md)** pour le détail complet des indicateurs, graphiques, tableaux, du sélecteur de période et de l'assistant d'analyse en langage naturel.

---

**Manuels liés** :
- [Module 01 — Tableau de bord et statistiques](01-principal-tableau-de-bord.md)
