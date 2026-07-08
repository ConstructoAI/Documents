# Manuel utilisateur — ERP AI de Constructo AI

Logiciel de gestion intégré pour les entreprises de construction au Québec.

> **Version** : 3.0 — refonte complète vérifiée par rapport au code source
> **Date** : 2026-07-08
> **Public cible** : utilisateurs finaux (chefs de projet, gestionnaires, comptables, employés de terrain, administrateurs)
> **Langues** : français (ce dossier) et anglais (sous-dossier [`en/`](en/README.md))

---

## État de la documentation

Ce dossier contient **38 manuels d'utilisation**, un par module et sous-module de l'écosystème Constructo AI, dans l'ordre du menu latéral, suivis des applications co-hébergées (Immobilier, Portail B2B/B2C, SEAOP, Estimation Express, Pointeur mobile).

| Statut | Signification |
|--------|---------------|
| ✅ v3.0 vérifié code | Chaque manuel a été rédigé à partir du **code source actuel** (routers backend, pages React, fichiers i18n), corrigé linguistiquement (accents, orthographe, anglicismes — norme OQLF), puis traduit en anglais. |

Chaque affirmation est traçable jusqu'au code (`ERP_REACT/backend/routers/*.py`, `frontend/src/pages/*.tsx`, `frontend/src/api/*.ts`, applications séparées `IMMO_REACT`, `SEAOP_REACT`, `ESTIMATION_EXPRESS_REACT`, `MOBILE_REACT`).

**Version anglaise** : chaque manuel possède son équivalent anglais dans le sous-dossier [`en/`](en/), au même nom de fichier. Voir [`en/README.md`](en/README.md).

---

## Table des matières — par section du menu

### Principal

| # | Module | Fichier |
|---|--------|---------|
| 01 | Tableau de bord et statistiques | [01-principal-tableau-de-bord.md](01-principal-tableau-de-bord.md) |
| 02 | Suivi (Kanban, Gantt, Calendrier) | [02-suivi-gantt.md](02-suivi-gantt.md) |

### Gestion

| # | Module | Fichier |
|---|--------|---------|
| 03 | Entreprises (clients et fournisseurs) | [03-gestion-entreprises.md](03-gestion-entreprises.md) |
| 04 | Contacts | [04-gestion-contacts.md](04-gestion-contacts.md) |
| 05 | Ventes (CRM, pipeline, back-office B2B) | [05-gestion-crm-opportunites.md](05-gestion-crm-opportunites.md) |
| 06 | Dossiers (gestion documentaire) | [06-ventes-dossiers.md](06-ventes-dossiers.md) |
| 07 | Soumissions et devis (manuel, IA, import) | [07-ventes-soumissions.md](07-ventes-soumissions.md) |
| 08 | Projets de construction | [08-ventes-projets.md](08-ventes-projets.md) |

### Opérations

| # | Module | Fichier |
|---|--------|---------|
| 09 | Magasin (produits, inventaire, achats, RFQ) | [09-operations-magasin.md](09-operations-magasin.md) |
| 10 | Employés (ressources humaines) | [10-operations-employes.md](10-operations-employes.md) |
| 11 | Bons de travail | [11-operations-bons-de-travail.md](11-operations-bons-de-travail.md) |
| 12 | Pointage et heures (+ paie CCQ, feuillets T4/RL-1/PD7A) | [12-operations-pointage.md](12-operations-pointage.md) |
| 13 | Bons de commande (achats fournisseurs) | [13-operations-bons-de-commande.md](13-operations-bons-de-commande.md) |
| 14 | Comptabilité (grand livre, factures, WIP, retenues) | [14-operations-comptabilite.md](14-operations-comptabilite.md) |

### Terrain

| # | Module | Fichier |
|---|--------|---------|
| 15 | Météo chantier | [15-terrain-meteo-chantier.md](15-terrain-meteo-chantier.md) |
| 16 | Conformité RBQ / CCQ | [16-terrain-conformite.md](16-terrain-conformite.md) |
| 17 | Subventions et aides gouvernementales | [17-terrain-subventions.md](17-terrain-subventions.md) |
| 18 | Logistique et livraisons | [18-terrain-logistique.md](18-terrain-logistique.md) |
| 19 | Location d'équipements et main-d'œuvre | [19-terrain-location.md](19-terrain-location.md) |
| 20 | Maintenance (préventive, corrective) | [20-terrain-maintenance.md](20-terrain-maintenance.md) |

### Communication

| # | Module | Fichier |
|---|--------|---------|
| 21 | Courriels (IMAP/SMTP) | [21-communication-emails.md](21-communication-emails.md) |
| 22 | Messagerie interne | [22-communication-messagerie.md](22-communication-messagerie.md) |
| 23 | Agent vocal IA (standardiste virtuelle) | [23-communication-agent-vocal.md](23-communication-agent-vocal.md) |
| 24 | Assistant IA (moteur IA général) | [24-communication-assistant-ia.md](24-communication-assistant-ia.md) |

### Conception 3D

| # | Module | Fichier |
|---|--------|---------|
| 25 | DAO / Modélisation 3D | [25-outils-dao-modelisation.md](25-outils-dao-modelisation.md) |
| 26 | PDF3D (plan vers 3D) | [26-conception3d-pdf3d-hover.md](26-conception3d-pdf3d-hover.md) |
| 27 | Rendu 3D photoréaliste | [27-conception3d-rendu-3d.md](27-conception3d-rendu-3d.md) |

### Outils

| # | Module | Fichier |
|---|--------|---------|
| 28 | Calculateurs de construction | [28-outils-calculateurs.md](28-outils-calculateurs.md) |
| 29 | Web (recherche temps réel) | [29-outils-web.md](29-outils-web.md) |
| 32 | Métré (prise de quantités sur PDF) | [32-metre-pdf.md](32-metre-pdf.md) |

### Système

| # | Module | Fichier |
|---|--------|---------|
| 30 | Configuration (entreprise, utilisateurs, abonnement) | [30-configuration.md](30-configuration.md) |
| 31 | Intégrations comptables (QuickBooks, Sage) | [31-integrations-comptables.md](31-integrations-comptables.md) |
| 33 | Aide et ressources | [33-aide-ressources.md](33-aide-ressources.md) |

### Programmes (applications co-hébergées)

| # | Programme | Fichier |
|---|-----------|---------|
| 34 | Immobilier (promoteur et vitrine publique) | [34-terrain-immobilier.md](34-terrain-immobilier.md) |
| 35 | Portail B2B / B2C (clients et fournisseurs) | [35-programme-portail-b2b.md](35-programme-portail-b2b.md) |
| 36 | SEAOP (appels d'offres publics du Québec) | [36-programme-seaop.md](36-programme-seaop.md) |
| 37 | Estimation Express (sous-application publique payante) | [37-programme-estimation-express.md](37-programme-estimation-express.md) |
| 38 | Pointeur mobile (application PWA de pointage) | [38-programme-pointeur-mobile.md](38-programme-pointeur-mobile.md) |

---

## Premiers pas

### 1. Connexion

1. Accédez à l'application dans votre navigateur.
2. Entrez le courriel de votre entreprise.
3. Saisissez votre nom d'utilisateur et votre mot de passe.
4. Cliquez sur « Se connecter ».

### 2. Navigation

- Utilisez le **menu latéral gauche** pour naviguer entre les modules.
- Le **tableau de bord** affiche vos indicateurs clés dès la connexion.
- L'**Assistant IA** (bouton en haut à droite) répond à vos questions en langage naturel.

### 3. Aide

- Chaque module dispose de son manuel dans ce dossier (français) et dans [`en/`](en/) (anglais).
- Voir aussi le fichier [liens-utiles.md](liens-utiles.md) pour des ressources externes (RBQ, CCQ, CNESST, programmes de subventions, etc.).
- Support : **info@constructoai.ca** | **1 (936) 587-1141**.

---

## Points forts du système

- **Conformité québécoise native** : RBQ, CCQ, TPS/TVQ, CNESST, Loi 16, Loi 25.
- **Intelligence artificielle** : assistants spécialisés (Claude), estimation, vision, rendu 3D, agent vocal.
- **Calculateurs métier** : outils conformes aux normes CCQ, CSA, CNB, ASTM, AWS.
- **Paie Québec** : RRQ, RQAP, AE, FSS, CNESST, CCQ, feuillets T4 et Relevé 1.
- **Intégrations comptables** : QuickBooks Online (OAuth 2.0), Sage.
- **Portail client B2B/B2C**, **appels d'offres SEAOP**, **estimation publique** et **application mobile** de pointage.
- **Architecture multi-entreprises** (multi-tenant).

---

## Support

- **Assistant IA** : disponible dans l'application.
- **Courriel** : info@constructoai.ca
- **Téléphone** : 1 (936) 587-1141

---

© 2026 Constructo AI — Tous droits réservés | Développé par Sylvain Leduc
