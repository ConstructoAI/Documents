# Module 01 — Tableau de bord et statistiques

> **Version** : 3.0 (refonte vérifiée par rapport au code source — Tableau de bord et Analyses désormais **fusionnés** en une seule page à six onglets)
> **Code de référence** : `backend/routers/dashboard.py` (285 lignes, 5 points de terminaison), `backend/routers/analytics.py` (1553 lignes, 25 points de terminaison dont 20 actifs + 5 orphelins), `backend/routers/dashboard_ai.py` (456 lignes, 1 point de terminaison — assistant BI en lecture seule), `frontend/src/pages/AnalyticsPage.tsx` (1456 lignes, 6 onglets), `frontend/src/components/analytics/DashboardAssistantTab.tsx` (166 lignes)
> **Route** : `/dashboard` (affiche `AnalyticsPage.tsx` ; les anciennes routes `/` et `/analyses` redirigent ici)
> **Bibliothèque graphique** : Recharts (AreaChart, BarChart, PieChart/Donut, ResponsiveContainer)

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas à pas](#3-workflows-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le Tableau de bord est le poste de pilotage de l'entreprise. Il rassemble en un seul endroit, **en lecture seule**, les indicateurs clés (KPI), les graphiques et les tableaux qui répondent aux questions quotidiennes de l'entrepreneur :

- Où en sont mes projets, mes soumissions, mes factures ?
- Combien ai-je encaissé, combien me reste-t-il à percevoir ?
- Mes projets sont-ils rentables ?
- Mes employés sont-ils bien occupés ?
- Mon stock est-il suffisant, quelle est sa valeur ?
- Quel est l'état de mon pipeline de ventes ?

Depuis la version 3.0, un sixième onglet permet aussi d'**interroger ces données en langage naturel** grâce à un assistant IA d'analyse (voir section 2.8).

Le module ne sert qu'à **consulter** : aucune donnée métier ne s'y saisit ni ne s'y modifie. Toutes les valeurs affichées proviennent des autres modules (Projets, Soumissions, Comptabilité, Employés, Magasin, CRM, Bons de travail, Bons de commande).

### 1.2 Une seule page, six onglets (changement majeur)

> **Important** — Si vous avez connu une version antérieure de l'ERP : le « Tableau de bord » et la page « Analyses » ne sont **plus deux pages séparées**. Ils sont **fusionnés** en une seule page à six onglets.

- La route `/dashboard` affiche désormais le composant `AnalyticsPage.tsx` (`App.tsx:200-206`).
- L'ancienne page `DashboardPage.tsx` **n'existe plus** dans le logiciel.
- Les anciennes adresses redirigent automatiquement vers la page fusionnée : `/` → `/dashboard` (`App.tsx:198`) et `/analyses` → `/dashboard` (`App.tsx:218-219`).
- Il ne reste **qu'une seule entrée** dans le menu latéral : **« Tableau de Bord »** (icône `LayoutDashboard`, section « Principal », `Sidebar.tsx:40`). L'entrée « Analyses » a disparu.
- Le **titre affiché en haut de la page reste « Tableau de Bord »** sur tous les onglets — il n'y a pas de titre différent selon l'onglet actif (`AnalyticsPage.tsx:440`).

### 1.3 Accès par le menu latéral

- Menu latéral → section **Principal** → **Tableau de Bord** (icône `LayoutDashboard`).
- Adresse : `app.constructoai.ca/dashboard`.
- C'est très souvent l'écran d'accueil affiché juste après la connexion (la racine `/` y redirige).

### 1.4 Permissions et rôles

| Règle | Détail |
|---|---|
| **Aucune restriction par rôle** | Tout utilisateur authentifié du tenant voit la page. Il n'y a aucun contrôle de rôle (`require_role`, `is_admin`, etc.) sur `dashboard.py` ni `analytics.py` — seul `Depends(get_current_user)` protège les points de terminaison. |
| **Vue globale du tenant** | Tous les utilisateurs voient les **mêmes** chiffres (ceux de l'entreprise entière). Il n'y a pas de filtrage « mes projets » ou « mes heures » par utilisateur. |
| **Donnée sensible visible par tous** | Comme il n'y a pas de verrou de rôle, **tout employé du tenant** peut lire les revenus, les marges, le pipeline commercial et même l'agrégat **salaire moyen des employés** (`employes_salaire_moyen`, exposé par `GET /dashboard`). À garder en tête si des employés terrain ont un accès ERP. |
| **Super-admin plateforme** | Un super-administrateur sans tenant reçoit des réponses **vides / à zéro** (jamais d'erreur). |
| **Mode consultation (abonnement inactif)** | Les onglets de consultation restent accessibles (ce sont des lectures). En revanche, **l'Assistant IA est bloqué** (voir section 5.2) : envoyer un message est techniquement une écriture, refusée en mode consultation. |

### 1.5 Les six onglets (sous-modules)

| Onglet | Libellé (bureau) | Libellé (mobile) | Icône | Contenu |
|---|---|---|---|---|
| `vue_globale` | **Vue Globale** | Global | Eye | 16 cartes KPI + 5 graphiques + 2 tableaux |
| `projets` | **Projets** | Projets | BarChart3 | Rentabilité, progression, création par mois, répartition par statut |
| `finances` | **Finances** | Finances | DollarSign | Revenus vs dépenses, vieillissement des comptes clients, pipeline, top clients |
| `rh` | **RH** | RH | Users | Heures travaillées, départements, productivité par employé |
| `stock` | **Stock** | Stock | Boxes | Valeur du stock, alertes de réapprovisionnement, top fournisseurs |
| `assistant` | **Assistant IA** | IA | Sparkles | Chat d'analyse en langage naturel (lecture seule) |

L'onglet ouvert par défaut est **Vue Globale** (`AnalyticsPage.tsx:241`). Sur mobile, la barre d'onglets défile horizontalement et n'affiche que l'icône + un libellé court.

### 1.6 Deux sources de chiffres coexistent

Il est utile de comprendre que la page tire ses indicateurs de **deux mécanismes différents** — c'est ce qui explique certains comportements du sélecteur de période (voir section 4.4) :

| Source | Nature | Réagit à la période ? | Alimente |
|---|---|---|---|
| `GET /analytics/kpis` | 16 KPI **calculés sur la période** choisie | Oui | Cartes KPI « Rangée 1 » et « Rangée 2 » de Vue Globale, et les cartes des autres onglets |
| `GET /dashboard` (`DashboardStats`, ~35 champs) | Compteurs **absolus** (totaux courants) | Non (chargés une seule fois) | Cartes KPI « Rangée 3 » et « Rangée 4 » + les 2 tableaux de Vue Globale |

---

## 2. Interface

### 2.1 Disposition générale de la page

De haut en bas (`AnalyticsPage.tsx:436-455`) :

1. **En-tête** : le titre « Tableau de Bord » à gauche, et à droite le **sélecteur de Période** (menu déroulant).
2. **Barre des six onglets** : Vue Globale · Projets · Finances · RH · Stock · Assistant IA (défilement horizontal sur mobile).
3. **Zone de contenu** : affiche l'onglet actif.

La page a trois états d'affichage globaux (voir section 2.9) : chargement, erreur de chargement, et contenu normal.

### 2.2 Sélecteur de Période

Menu déroulant en haut à droite, quatre choix (`AnalyticsPage.tsx:39-44`) :

| Libellé | Valeur (jours) | Défaut |
|---|---|---|
| **30 jours** | 30 | ✅ |
| **90 jours** | 90 | |
| **6 mois** | 180 | |
| **1 an** | 365 | |

> **À savoir** : ce sélecteur a un **effet partiel** et non évident. Il ne rafraîchit qu'une partie des chiffres de la page (les cartes KPI de période, la rentabilité des projets, la productivité RH et les départements). Plusieurs graphiques sont figés sur 1 an, et **les graphiques propres à chaque onglet ne se rafraîchissent pas** quand on change la période. Le détail complet est donné en section 4.4. Ce n'est pas un bogue d'affichage de votre côté : c'est le comportement actuel du logiciel.

### 2.3 Onglet « Vue Globale »

C'est le tableau de bord de synthèse : 16 cartes KPI réparties en 4 rangées, puis des graphiques, puis deux tableaux.

#### 2.3.1 Rangée 1 — indicateurs de la période (`AnalyticsPage.tsx:472-501`)

| Carte | Valeur | Couleur | Sous-titre / tendance |
|---|---|---|---|
| **Revenus (terminés)** | Revenu net de la période | vert | Tendance en % « vs mois préc. » |
| **Soumissions envoyées** | Nombre de devis envoyés | bleu | « sur N au total » |
| **Projets actifs** | Nombre de projets actifs | violet | « N total » |
| **Employés actifs** | Nombre d'employés au statut ACTIF | sarcelle | — |

#### 2.3.2 Rangée 2 — indicateurs de la période (`AnalyticsPage.tsx:504-530`)

| Carte | Valeur | Couleur |
|---|---|---|
| **Pipeline commercial** | Valeur totale des opportunités + « N opportunités » | violet |
| **Alertes stock** | Nombre de produits sous le seuil | rouge si > 0, sinon vert |
| **Revenus encaissés** | Montant réellement encaissé | vert |
| **Solde dû (factures)** | Montant restant à percevoir | rouge si > 0 |

#### 2.3.3 Rangée 3 — compteurs absolus (`AnalyticsPage.tsx:536-561`)

> Ces quatre cartes viennent de l'ancien tableau de bord. Elles sont **chargées une seule fois** à l'ouverture de la page et **ne dépendent pas de la période** choisie.

| Carte | Valeur | Icône |
|---|---|---|
| **Entreprises** | Nombre total d'entreprises (clients + fournisseurs) | Building2 |
| **Factures** | Nombre total de factures | Receipt |
| **Produits** | Nombre total de produits | Package |
| **Fournisseurs** | Nombre total de fournisseurs | Truck |

#### 2.3.4 Rangée 4 — opérations et alertes (`AnalyticsPage.tsx:564-591`)

| Carte | Valeur | Couleur / sous-titre |
|---|---|---|
| **Bons de travail** | Nombre total de BT | « N en cours » |
| **Projets terminés** | Nombre de projets terminés | vert |
| **Soumissions brouillon** | Nombre de devis en brouillon | ambre — « À finaliser » |
| **Alertes** | Nombre d'alertes actives | rouge si > 0 |

#### 2.3.5 Graphiques de Vue Globale

| Graphique | Type | Contenu |
|---|---|---|
| **Revenus mensuels** | AreaChart (dégradé) | Revenu net par mois (`AnalyticsPage.tsx:594-614`) |
| **Revenus vs Dépenses** | AreaChart (2 aires) | Revenus et Dépenses superposés (`:618-644`) |
| **Évolution des projets** | AreaChart empilé | En cours / Terminés / En attente par mois (`:646-677`) |
| **Distribution des factures** | Donut | Répartition des factures par statut (`:682-688`) |
| **Bons de travail par statut** | Donut | Répartition des BT par statut (`:690-696`) |

Chaque graphique affiche « Aucune donnée disponible » lorsqu'il n'y a rien à tracer.

#### 2.3.6 Tableaux de Vue Globale

**Projets par statut** (`AnalyticsPage.tsx:704-740`) — colonnes :
- **Statut**
- **Nombre**
- **%** (avec barre de progression)

**Top 5 fournisseurs** (`AnalyticsPage.tsx:742-767`) — colonnes :
- **Fournisseur**
- **Commandes** (nombre de bons de commande)
- **Montant total**

> Ces deux tableaux, comme les rangées 3 et 4, sont chargés une seule fois et ignorent le sélecteur de période.

### 2.4 Onglet « Projets »

#### 2.4.1 Cartes KPI (`AnalyticsPage.tsx:776-786`)

**Projets total** · **En cours** · **Terminés** · **Budget total** (somme des budgets des projets analysés).

#### 2.4.2 Rentabilité des projets — Budget vs Coût réel (`AnalyticsPage.tsx:789-861`)

Un graphique à barres horizontales (Budget vs Coût réel, top 5 à 10) accompagné d'un **tableau détaillé** :

| Colonne | Détail |
|---|---|
| **Projet** | Nom du projet |
| **Statut** | Badge de couleur |
| **Budget** | Budget prévu |
| **Coût** | Coût réel (main-d'œuvre + matériaux) |
| **Marge** | Budget − Coût (couleur selon le signe) |
| **%** | Marge en % — badge **vert si ≥ 20**, **jaune si ≥ 0**, **rouge si < 0** |

Une **ligne Total** clôt le tableau.

#### 2.4.3 Autres sections

- **Progression des projets** (`:864-890`) : une liste de barres de progression, une par projet, avec le pourcentage d'avancement. Couleur de la barre : **vert si ≥ 100 %**, **bleu si ≥ 50 %**, **ambre** sinon.
- **Création de projets par mois** (`:894-914`) : AreaChart du nombre de projets créés par mois.
- **Répartition par statut** (`:916-931`) : Donut (En cours / Terminés / En attente), calculé côté écran.

### 2.5 Onglet « Finances »

#### 2.5.1 Cartes KPI (`AnalyticsPage.tsx:940-970`)

| Carte | Détail |
|---|---|
| **Revenus encaissés** | Avec tendance « vs mois préc. » |
| **Solde dû** | Rouge si > 0 — sous-titre « N factures » |
| **Taux conversion devis** | Devis acceptés ÷ devis total — sous-titre « A/T acceptés » |
| **Pipeline commercial** | Sous-titre « N opportunités » |

#### 2.5.2 Sections

- **Revenus vs Dépenses (12 mois)** (`:973-1004`) : AreaChart à 3 aires — Revenus / Dépenses / Marge.
- **Distribution des factures** (`:1008-1014`) : Donut par statut.
- **Vieillissement des comptes clients** (`:1016-1034`) : BarChart par tranche d'âge, cellules colorées selon l'ancienneté. Tranches : **0-30 jours / 31-60 jours / 61-90 jours / 90+ jours**. État vide : « Aucune facture en souffrance ».
- **Pipeline commercial** (`:1038-1093`) : BarChart par étape + tableau **Étape (pastille couleur) / Nombre / Montant** avec ligne Total.
- **Top clients (par chiffre d'affaires)** (`:1096-1142`) : BarChart horizontal + tableau **Client / Type / CA Total / Projets** avec ligne Total.

### 2.6 Onglet « RH »

#### 2.6.1 Cartes KPI (`AnalyticsPage.tsx:1150-1176`)

**Employés actifs** · **Heures totales** (format `Xh`) · **Heures/jour moyen** · **Départements** (nombre — sous-titre « N employés actifs »).

#### 2.6.2 Sections

- **Tendance des heures travaillées (12 mois)** (`:1179-1206`) : AreaChart à **double axe** — Heures (axe gauche) et Employés (axe droit).
- **Répartition par département** (`:1210-1218`) : Donut par heures totales.
- **Heures par employé** (`:1220-1238`) : BarChart horizontal (top 5 à 8).
- **Productivité détaillée** (`:1242-1290`) : tableau complet.

Tableau **Productivité détaillée** — colonnes :

| Colonne | Détail |
|---|---|
| **Employé** | Prénom + nom |
| **Poste** | Fonction |
| **Dépt.** | Département |
| **Jours** | Jours travaillés |
| **Heures** | Heures totales |
| **h/jour** | Heures par jour — couleur selon les seuils 7,5 et 6 |
| **Projets** | Nombre de projets touchés |

Une ligne **Total / Moyenne** clôt le tableau.

### 2.7 Onglet « Stock »

> C'est le seul onglet qui n'a pas besoin des KPI de période pour s'afficher : il repose sur son propre résumé de stock.

#### 2.7.1 Cartes KPI (`AnalyticsPage.tsx:1298-1324`)

**Produits actifs** (sous-titre « N total ») · **Alertes stock** (rouge si > 0) · **Valeur totale** · **Catégories** (nombre).

#### 2.7.2 Sections

- **Valeur du stock par catégorie** (`:1327-1351`) : un BarChart **et** un Donut côte à côte.
- **Alertes stock (N)** (`:1354-1395`) : tableau **Produit / Catégorie / Stock (+ unité) / Seuil / Niveau**. La colonne **Niveau** montre une barre + un badge en pourcentage, coloré **rouge si < 25 %**, **jaune si < 50 %**, **vert** sinon. État vide : « Aucune alerte de stock ».
- **Top fournisseurs** (`:1398-1448`) : BarChart horizontal **et** tableau **Fournisseur / Cmd. / Montant** avec ligne Total. Cette section n'apparaît que s'il existe au moins un fournisseur avec des achats.

### 2.8 Onglet « Assistant IA » (analyse, lecture seule)

Cet onglet ouvre un **chat d'analyse** qui répond à vos questions à partir de vos **vraies données** (`DashboardAssistantTab.tsx`).

#### 2.8.1 Ce que vous voyez

- **En-tête** : icône Sparkles, titre **« Assistant IA — Tableau de bord »**, sous-titre « Analyse tes indicateurs à partir de tes données réelles (lecture seule). »
- **Écran d'accueil** (avant la première question) : un texte de bienvenue + **trois exemples de questions** cliquables :
  - « Quel est le total de mes factures impayées ? »
  - « Quels projets sont en cours et pour quel montant ? »
  - « Montre-moi mon pipeline de ventes par statut. »
- **Fil de conversation** : vos messages et les réponses de l'assistant, avec défilement automatique. Pendant le traitement, un indicateur « Analyse en cours… » s'affiche.
- **Bandeau d'erreur** (si problème) : affiche le message renvoyé par le serveur, sinon « Une erreur est survenue. Réessaie. »
- **Zone de saisie** : un champ de texte (invite « Pose ta question sur le tableau de bord… ») et un bouton **Envoyer**. **Entrée** envoie le message ; **Maj+Entrée** insère un saut de ligne.
- **Détails sous chaque réponse** : profil « Analyste », nombre de jetons, coût et temps de la réponse.

#### 2.8.2 Ce que l'assistant peut et ne peut pas faire

| Peut | Ne peut pas |
|---|---|
| Lire et résumer vos finances, devis, projets, opportunités, produits, fournisseurs, bons de travail, entreprises et contacts. | **Écrire** quoi que ce soit : il n'y a aucune action à confirmer, aucune création ni modification (contrairement à l'assistant du module Comptabilité). |
| Répondre en langage naturel avec des chiffres exacts tirés de la base. | **Lire la paie, les heures individuelles, les salaires, le NAS, les utilisateurs, les crédits IA, les courriels** (exclusion volontaire — voir section 4.11 et 5.2). |
| Enchaîner quelques requêtes internes pour bâtir sa réponse. | Sortir du périmètre de votre entreprise (aucun accès inter-tenant). |

#### 2.8.3 Coût

Chaque question consomme des **crédits IA** prépayés (voir section 4.11). C'est la **seule fonction de tout le module qui a un effet monétaire** ; les autres onglets sont des lectures gratuites.

### 2.9 États de la page

| État | Affichage |
|---|---|
| **Chargement initial** | Squelette de page (`SkeletonPage`) tant que les indicateurs ne sont pas arrivés. |
| **Erreur de chargement** | Écran centré avec une icône d'alerte, le message d'erreur (ou « Impossible de charger les indicateurs. Réessayez. ») et un bouton **« Réessayer »**. |
| **Section vide** | Le composant « EmptyState » affiche « Aucune donnée disponible » (ou un message plus précis selon la section). |
| **Changements rapides** | Si vous changez vite de période ou d'onglet, la page ignore les réponses qui arrivent en retard (garde anti-réponse obsolète) : vous voyez toujours les données de la sélection courante. |

---

## 3. Workflows pas à pas

### 3.1 Consulter la vue d'ensemble quotidienne

1. Menu latéral → **Tableau de Bord** (souvent déjà affiché après la connexion).
2. La page s'ouvre sur l'onglet **Vue Globale** et charge en parallèle : les 16 KPI de période (`/analytics/kpis`), les compteurs absolus et alertes (`/dashboard`), les graphiques (`/dashboard/charts`) et le top fournisseurs (`/dashboard/top-suppliers`).
3. Lisez d'abord les rangées 1 et 2 (revenus, encaissé, solde dû, pipeline, alertes stock) pour la photo du moment.
4. Vérifiez la carte **Alertes** (rangée 4) : un chiffre en rouge signale des devis urgents, du stock bas ou des factures en retard.

> Aucun rafraîchissement automatique : les chiffres sont chargés à l'ouverture. Pour actualiser, rechargez la page (F5) ou rouvrez l'onglet.

### 3.2 Changer l'horizon temporel

1. En haut à droite, ouvrez le sélecteur de **Période** et choisissez 30 jours / 90 jours / 6 mois / 1 an.
2. **Effet réel** (voir la matrice en 4.4) : se mettent à jour les cartes KPI des rangées 1 et 2, la **rentabilité des projets**, la **productivité RH** et la **répartition par département**.
3. **Ne changent pas** avec la période : les graphiques propres à chaque onglet (revenus mensuels, distribution des factures, vieillissement, pipeline, valeur du stock, etc.), plusieurs séries figées sur 1 an, ainsi que les rangées 3-4 et les 2 tableaux de Vue Globale.

> Si un graphique ne bouge pas quand vous changez la période, c'est normal : reportez-vous à la section 4.4 pour savoir ce qui répond réellement au sélecteur.

### 3.3 Analyser la rentabilité des projets

1. Onglet **Projets**.
2. Consultez le tableau **Rentabilité des projets — Budget vs Coût réel**.
3. Pour chaque projet, comparez **Budget** et **Coût**, puis lisez la **Marge** et le **%**.
4. Repérez les badges **rouges** (marge négative) et **jaunes** (marge faible, entre 0 et 20 %) : ce sont vos projets à surveiller.
5. La **ligne Total** donne la marge globale sur les projets analysés.

### 3.4 Suivre la productivité RH

1. Onglet **RH**.
2. Lisez le graphique **Tendance des heures travaillées** (heures à gauche, nombre d'employés à droite).
3. Dans le tableau **Productivité détaillée**, repérez la colonne **h/jour** :
   - une valeur élevée peut signaler une surcharge ;
   - une valeur basse peut signaler une sous-utilisation.
4. La ligne **Total / Moyenne** donne la moyenne de l'équipe.

> L'onglet RH montre des **agrégats** (heures par employé, par département). Pour le détail individuel, la fiche employé et le calcul de paie, passez par les modules **Employés (11)** et **Pointage (13)**.

### 3.5 Gérer les alertes de stock

1. Onglet **Vue Globale** : la carte **Alertes stock** donne le nombre de produits sous le seuil.
2. Onglet **Stock** → tableau **Alertes stock** pour le détail.
3. Pour chaque produit, lisez la colonne **Niveau** : une barre **rouge (< 25 %)** est critique, **jaune (< 50 %)** demande attention.
4. Réapprovisionnez via le module **Bons de commande (14)** auprès du fournisseur concerné.

### 3.6 Piloter le pipeline commercial

1. Onglet **Finances** → section **Pipeline commercial**.
2. Lisez le montant et le nombre d'opportunités par étape.
3. Repérez les étapes qui bloquent (beaucoup de valeur immobilisée à une même étape).
4. Agissez dans le module **CRM (06)** pour faire avancer les opportunités.

### 3.7 Surveiller les factures en retard

1. Onglet **Finances** → section **Vieillissement des comptes clients**.
2. Analysez les quatre tranches (0-30 / 31-60 / 61-90 / 90+ jours). Les tranches les plus âgées sont les plus à risque.
3. La section **Distribution des factures** montre la répartition par statut.
4. Relancez et encaissez dans le module **Comptabilité (15)**.

### 3.8 Identifier les meilleurs clients

1. Onglet **Finances** → section **Top clients (par chiffre d'affaires)**.
2. Lisez le CA total et le nombre de projets par client.
3. La ligne **Total** situe la contribution de vos plus gros clients.

### 3.9 Analyser la valeur de l'inventaire

1. Onglet **Stock** → carte **Valeur totale**.
2. Section **Valeur du stock par catégorie** : repérez les catégories qui immobilisent le plus de capital.
3. Croisez avec les **Alertes stock** pour équilibrer sur-stock et ruptures.

### 3.10 Interroger vos données en langage naturel (Assistant IA)

1. Onglet **Assistant IA**.
2. Cliquez un exemple ou saisissez votre question (« Quelles factures sont impayées et pour combien ? »).
3. Appuyez sur **Entrée** (ou le bouton **Envoyer**).
4. Lisez la réponse ; les détails (coût, jetons, temps) apparaissent dessous.
5. Poursuivez la conversation : l'assistant garde le contexte des échanges récents.

> Chaque question consomme des **crédits IA**. Si votre abonnement est en **mode consultation**, l'assistant est bloqué (erreur 403) — voir section 5.2.

### 3.11 Récupérer après une erreur de chargement

1. Si l'écran d'erreur s'affiche (« Impossible de charger les indicateurs. »), cliquez **« Réessayer »**.
2. Si l'erreur persiste, rechargez la page (F5). En dernier recours, vérifiez votre connexion ou contactez votre administrateur.

---

## 4. Référence

### 4.1 Points de terminaison — Tableau de bord (`/api/erp/v1/dashboard`)

`routers/dashboard.py` — protégés par `Depends(get_current_user)`, sans rôle.

| Méthode + chemin | Rôle | Consommé par l'interface ? |
|---|---|---|
| `GET /dashboard` | Compteurs consolidés (`DashboardStats`, ~35 champs) + alertes | Oui (rangées 3-4, tableaux) |
| `GET /dashboard/alerts` | 3 alertes : devis urgents, stock bas, factures en retard | Oui |
| `GET /dashboard/charts` | 3 séries : projets par statut, revenus mensuels (6 mois), BT par statut | Oui (Vue Globale) |
| `GET /dashboard/top-suppliers` | Top 5 fournisseurs par volume d'achats | Oui (Vue Globale, Stock) |
| `GET /dashboard/activity` | Activité récente (20 derniers projets) | **Non** — orphelin |

### 4.2 Points de terminaison — Analyses actifs (`/api/erp/v1/analytics`)

`routers/analytics.py` — 25 points de terminaison au total, dont **20 réellement utilisés** par l'interface.

| Chemin | Contenu | Paramètre de période |
|---|---|---|
| `GET /analytics/kpis` | 16 KPI de synthèse | `period_days` 1-365 (défaut 30) |
| `GET /analytics/projects/profitability` | Budget vs coûts réels | `period_days` 1-730 (défaut 90) |
| `GET /analytics/projects/evolution` | Projets par mois et par statut | `period_days` 30-730 (défaut 365) |
| `GET /analytics/commercial/pipeline` | Entonnoir des opportunités (hors PERDU) | — |
| `GET /analytics/hr/productivity` | Productivité par employé | `period_days` 1-365 (défaut 30) |
| `GET /analytics/hr/departments` | Heures par département | `period_days` 1-365 (défaut 30) |
| `GET /analytics/finance/revenue-expenses` | Revenus vs dépenses + marge par mois | `period_days` 30-730 (défaut 365) |
| `GET /analytics/inventory/alerts` | Produits sous le seuil | — |
| `GET /analytics/top-clients` | Top clients par budget de projet | `period_days` 30-730 (défaut 365) |
| `GET /analytics/project-progress` | % d'avancement (projets non annulés) | — |
| `GET /analytics/sales-pipeline` | Opportunités par statut (inclut PERDU) | — |
| `GET /analytics/top-suppliers` | Fournisseurs par volume d'achats | — |
| `GET /analytics/monthly-revenue` | Revenu mensuel (12 mois) | — |
| `GET /analytics/stock-value` | Valeur du stock par catégorie | — |
| `GET /analytics/trends` | Mois courant vs mois précédent (%) | — |
| `GET /analytics/invoices-by-status` | Donut des factures (clients) | — |
| `GET /analytics/bt-by-status` | Donut des bons de travail | — |
| `GET /analytics/hours-trend` | Heures travaillées par mois | `period_days` 30-730 (défaut 365) |
| `GET /analytics/factures-aging` | Vieillissement 0-30 / 31-60 / 61-90 / 91+ | — |
| `GET /analytics/stock-summary` | Résumé stock (total, actifs, catégories, valeur, alertes) | — |

### 4.3 Points de terminaison présents mais inutilisés (orphelins)

Ces routes existent encore côté serveur mais **ne sont plus appelées** par l'interface (elles faisaient double emploi ou divergeaient d'un calcul officiel). Elles n'apparaissent nulle part dans l'écran :

- `GET /analytics/project-profitability` (doublon divergent de la rentabilité)
- `GET /analytics/workstation-load`
- `GET /analytics/top-clients-revenue`
- `GET /analytics/employee-productivity` (doublon « tout temps » de `/hr/productivity`)
- `GET /analytics/stock-alerts` (doublon de `/inventory/alerts`)
- `GET /dashboard/activity`

### 4.4 Le sélecteur de Période — matrice d'effet réel

> C'est le point le plus contre-intuitif du module. Voici précisément ce qui réagit ou non au sélecteur.

| Élément affiché | Réagit à la période ? | Pourquoi |
|---|---|---|
| Cartes KPI rangées 1 et 2 (Vue Globale) | ✅ Oui | `/analytics/kpis` reçoit la période |
| Cartes KPI des onglets Projets / Finances / RH (issues des KPI) | ✅ Oui | même source |
| Rentabilité des projets (Projets) | ✅ Oui | `profitability` reçoit la période |
| Productivité RH + Répartition par département (RH) | ✅ Oui | `hr/productivity` et `hr/departments` reçoivent la période |
| Évolution des projets / Création par mois | ❌ Non | figé à 365 jours |
| Revenus vs Dépenses | ❌ Non | figé à 365 jours |
| Top clients | ❌ Non | figé à 365 jours |
| Tendance des heures (RH) | ❌ Non | figé à 365 jours |
| Pipeline commercial / Alertes stock | ❌ Non | la période est ignorée |
| **Tous les graphiques d'onglet** (revenus mensuels, distribution factures, BT par statut, vieillissement, pipeline ventes, valeur stock…) | ❌ Non | le chargement par onglet ne dépend pas de la période |
| Rangées 3-4 + tableaux « Projets par statut » et « Top 5 fournisseurs » (Vue Globale) | ❌ Non | compteurs absolus, chargés une seule fois |

**En résumé** : pour une lecture *par période*, fiez-vous aux **cartes KPI (rangées 1-2)**, à la **rentabilité**, à la **productivité** et aux **départements**. Les graphiques historiques sont, eux, sur un horizon fixe (le plus souvent 12 mois).

### 4.5 Les 16 KPI de période (`/analytics/kpis`)

Champs calculés et bornés à la période (`analytics.py:80-230`) : `revenus_total`, `projets_actifs`, `projets_termines`, `projets_total`, `employes_actifs`, `alertes_stock`, `opportunites_pipeline`, `valeur_pipeline`, `devis_total`, `devis_acceptes`, `devis_envoyes`, `devis_valeur_totale`, `factures_total`, `factures_solde_du`, `revenus_encaisses` (+ tendances associées).

### 4.6 Les compteurs absolus (`DashboardStats`)

`GET /dashboard` renvoie ~35 compteurs courants (indépendants de la période), dont : entreprises, factures, produits, fournisseurs, bons de travail (total + en cours), projets terminés, devis en brouillon, valeur du pipeline, cinq compteurs d'inventaire, et l'agrégat `employes_salaire_moyen`.

### 4.7 Calculs clés

| Indicateur | Règle |
|---|---|
| **Revenu « avoir-aware »** | Une note de crédit (AVOIR, stockée en positif) est **soustraite** : revenu net = ventes − retours. |
| **Factures clients seulement** | Les revenus et le solde dû ne comptent que `type_destinataire` = `client` (ou nul) — les factures **fournisseurs** sont exclues. |
| **Statuts exclus des revenus** | Sont ignorés : `ANNULEE`, `BROUILLON` (revenus) ; `PAYEE`, `ANNULEE`, `BROUILLON` (solde dû). |
| **Coût de main-d'œuvre** | `heures × taux_horaire`, avec repli sur `salaire_annuel / 2080 h` puis 0 si aucun taux. |
| **Marge de projet** | `Budget − (coût main-d'œuvre + coût matériaux)`. |
| **Marge en %** | `Marge ÷ Budget × 100` (si budget > 0). |
| **Vieillissement** | Tranches selon `date du jour − date de facture` : ≤ 30 / ≤ 60 / ≤ 90 / sinon 91+. |
| **Tendance (trends)** | `(courant − précédent) ÷ précédent × 100` ; 0 si le précédent vaut 0. |
| **Taux de conversion des devis** | `devis acceptés ÷ devis total × 100`. |
| **Valeur du stock** | `stock × COALESCE(coût de revient, prix unitaire, 0)`. |
| **Mois manquants** | Les mois sans donnée sont comblés par des zéros (courbes continues). |

### 4.8 Les trois alertes du tableau de bord (`/dashboard/alerts`)

| Alerte | Condition |
|---|---|
| **Devis urgents** | Échéance ≤ 7 jours et statut « Envoyé » ou « En attente ». |
| **Stock bas** | Produit dont le stock disponible ≤ stock minimum (minimum > 0). |
| **Factures en retard** | Facture cliente échue et non payée. |

### 4.9 Codes couleur

| Contexte | Vert | Jaune / Ambre | Rouge / Bleu |
|---|---|---|---|
| **Badge % de marge** (Projets) | ≥ 20 % | ≥ 0 % | < 0 % (rouge) |
| **Barre de progression** (Projets) | ≥ 100 % | < 50 % (ambre) | ≥ 50 % (bleu) |
| **Niveau de stock** (Stock) | ≥ 50 % | < 50 % | < 25 % (rouge) |
| **Cartes « > 0 »** (alertes stock, solde dû) | 0 (vert) | — | > 0 (rouge) |

### 4.10 Périodes et bornes

- Quatre choix seulement : **30 / 90 / 180 / 365 jours**. Aucune plage personnalisée.
- Bornes serveur selon le point de terminaison : `period_days` accepté entre 1 et 365, ou 30 et 730 (une valeur hors bornes est refusée avec un code 422).
- Les données historiques ne sont **jamais purgées** : la limite de période ne concerne que l'affichage.

### 4.11 Assistant IA — périmètre, coût et sécurité (`/dashboard/ai/chat`)

| Aspect | Détail |
|---|---|
| **Point de terminaison** | `POST /api/erp/v1/dashboard/ai/chat` |
| **Nature** | Chat d'analyse **en lecture seule** — aucune écriture, aucune action à confirmer. |
| **Tables lisibles** | factures, lignes de factures, paiements reçus, devis, lignes de devis, projets, opportunités, produits, matériaux, fournisseurs, formulaires (BT), lignes de formulaires, entreprises, contacts. |
| **Tables interdites** | paie, heures individuelles, salaires, NAS, utilisateurs, crédits IA, courriels, entreprises de la plateforme, données Stripe/Vapi/Hover — exclusion volontaire (conformité **Loi 25**). |
| **Isolement** | Refus des références à un autre schéma : impossible de lire les données d'un autre tenant. |
| **Coût** | Coût réel du modèle Claude **× 1,30** (majoration 30 %), débité des **crédits IA** prépayés. |
| **Modèle** | `AI_MODEL` (Sonnet), `max_tokens = 8000`, jusqu'à 5 itérations internes. |
| **Historique** | Jusqu'à 40 échanges envoyés ; le serveur en conserve 40 et n'exploite que les 12 derniers. |
| **Codes d'erreur** | 503 si l'IA est indisponible, 402 si les crédits sont épuisés, 403 en mode consultation. |

### 4.12 Limitation du nombre d'appels

| Requête | Limite |
|---|---|
| `GET /dashboard` et `GET /dashboard/alerts` | **Non limitées** (exemptées). |
| Autres lectures analytics (`/charts`, `/top-suppliers`, `/analytics/*`) | Intervalle général (~1500 requêtes/min par IP). |
| `POST /dashboard/ai/chat` | **Intervalle dédié** : 20 requêtes par fenêtre et par IP. |

### 4.13 Ce qui n'existe pas dans ce module

- **Aucun export PDF ni Excel**, aucune impression. (Les libellés « Exporter PDF / Excel / Actualiser » de l'ancienne page Analyses subsistent dans les traductions mais ne sont **plus rattachés à aucun bouton**.)
- **Aucun bouton « Actualiser » manuel** : le rafraîchissement se fait à l'ouverture, au changement de période ou d'onglet.
- **Aucune plage de dates personnalisée** : seulement les 4 périodes prédéfinies.
- **Aucun forage (navigation détaillée)** : les cartes KPI, les lignes de tableaux et les segments de graphiques **ne sont pas cliquables**.
- **Aucune personnalisation** de la disposition (pas de widgets déplaçables).
- **Aucune différenciation par rôle** : tous les utilisateurs du tenant voient les mêmes chiffres.
- **Aucune actualisation automatique** (pas d'interrogation périodique, pas de temps réel).
- **L'assistant IA n'écrit jamais** et **ne lit pas la paie / RH / NAS / crédits**.

---

## 5. Intégrations et FAQ

### 5.1 Sources de données (tous les modules)

Le Tableau de bord ne stocke rien : il **agrège** les données produites ailleurs.

| Module source | Données consommées |
|---|---|
| **Projets (09)** | Statuts, budgets, dates, progression, coûts |
| **CRM (06)** | Opportunités, étapes, montants estimés (pipeline) |
| **Soumissions / Devis (08)** | Nombre, montants, taux de conversion |
| **Bons de travail (12)** | Nombre par statut, matériaux → coûts projet |
| **Bons de commande (14)** | Volumes d'achat, top fournisseurs |
| **Comptabilité (15)** | Factures (statut, vieillissement, revenus, solde dû) |
| **Employés (11)** + **Pointage (13)** | Employés actifs, heures, productivité, coût de main-d'œuvre |
| **Magasin (10)** | Stock, seuils, valeur, catégories |
| **Entreprises (04)** / **Contacts (05)** | Résolution des clients et fournisseurs |

### 5.2 Mode consultation (abonnement inactif)

Si l'abonnement du tenant est inactif (mode « consultation » / lecture seule) :

- **Tous les onglets de consultation restent accessibles** : ce sont des lectures, elles passent.
- **L'Assistant IA est bloqué (403)** : envoyer un message au chat compte comme une écriture, et les écritures sont refusées en mode consultation. Pour le réactiver, il faut régulariser l'abonnement (module **Configuration (28)** / Stripe).

### 5.3 Questions fréquentes

**Q : Pourquoi ai-je maintenant une seule page au lieu du Tableau de bord et de la page Analyses ?**
R : Les deux ont été **fusionnés** en une page unique à six onglets. Les anciennes adresses `/` et `/analyses` redirigent automatiquement vers `/dashboard`. C'est voulu.

**Q : Je change la période mais un graphique ne bouge pas. Est-ce un bogue ?**
R : Non. Le sélecteur n'a qu'un **effet partiel** (section 4.4). Seuls les KPI des rangées 1-2, la rentabilité, la productivité et les départements réagissent. Les graphiques historiques sont sur un horizon fixe (souvent 12 mois), et les graphiques d'onglet ne se rafraîchissent pas au changement de période.

**Q : Puis-je cliquer sur un indicateur pour voir le détail ?**
R : Non, il n'y a pas de forage. Notez le chiffre, puis ouvrez le module concerné (Comptabilité, Projets, Magasin…) pour le détail.

**Q : Comment exporter le tableau de bord en PDF ou en Excel ?**
R : Ce n'est pas possible depuis ce module. Solution de contournement : capture d'écran, ou copier-coller des tableaux. Pour des rapports formels, passez par les modules concernés.

**Q : Le « Solde dû » inclut-il les factures des fournisseurs ?**
R : Non. Seules les factures **clientes** (`type_destinataire` = `client`) sont comptées. Les achats fournisseurs relèvent du passif dans la Comptabilité.

**Q : Une note de crédit (avoir) est-elle bien déduite des revenus ?**
R : Oui. Le calcul est « avoir-aware » : un avoir est soustrait, donnant le revenu net (ventes − retours).

**Q : Tous mes employés voient-ils les revenus et les marges ?**
R : Oui. Il n'y a **aucune restriction par rôle** sur ce module : tout utilisateur ERP du tenant voit les agrégats financiers, et même le salaire moyen. Tenez-en compte avant de donner un accès ERP à des employés terrain.

**Q : L'assistant IA peut-il consulter les salaires ou la paie ?**
R : Non, jamais. Ces données (paie, heures individuelles, salaires, NAS, crédits, courriels) sont **exclues** de son périmètre, conformément à la Loi 25. Pour les questions de paie, utilisez l'assistant du module **Employés (11)**.

**Q : L'assistant IA peut-il créer une facture ou modifier un projet ?**
R : Non. Il est strictement en **lecture seule** : aucune écriture, aucune action à confirmer.

**Q : Pourquoi mon assistant IA renvoie-t-il une erreur 403 ?**
R : Probablement parce que votre abonnement est en **mode consultation**. Envoyer un message est considéré comme une écriture, donc bloqué. Régularisez l'abonnement pour le réactiver.

**Q : Combien coûte une question à l'assistant ?**
R : Le coût réel du modèle Claude, majoré de 30 %, débité de vos crédits IA. C'est la seule fonction payante du module.

**Q : Les chiffres sont-ils en temps réel ?**
R : Ils reflètent l'état exact au moment du **chargement** de la page. Il n'y a pas de rafraîchissement automatique : rechargez pour actualiser.

**Q : Puis-je choisir une plage de dates précise (ex. du 1er au 15 mars) ?**
R : Non. Seules quatre périodes prédéfinies existent (30 / 90 / 180 / 365 jours).

**Q : Pourquoi vois-je « Aucune donnée disponible » dans une section ?**
R : Il n'y a rien à afficher pour la sélection courante (pas de projet, pas de facture, pas de pointage sur l'horizon concerné). Ce n'est pas une erreur.

**Q : Le titre affiche « Tableau de Bord » même quand je suis sur l'onglet Finances. Normal ?**
R : Oui. Le titre de la page est fixe ; seul l'onglet actif change le contenu.

---

## 6. Récapitulatif

- **Une seule page à six onglets** : Vue Globale · Projets · Finances · RH · Stock · Assistant IA. Route `/dashboard` (affiche `AnalyticsPage.tsx`).
- **Fusion Tableau de bord + Analyses** : `DashboardPage.tsx` n'existe plus ; `/` et `/analyses` redirigent vers `/dashboard` ; une seule entrée « Tableau de Bord » dans le menu.
- **Module en lecture seule** : aucune saisie de donnée métier ; seule exception, l'Assistant IA (chat lecture seule, payant en crédits).
- **Deux sources de chiffres** : les KPI de période (`/analytics/kpis`, 16 indicateurs) et les compteurs absolus (`/dashboard`, ~35 champs).
- **Sélecteur de période à effet partiel** : ne rafraîchit que les KPI (rangées 1-2), la rentabilité, la productivité et les départements ; le reste est figé ou insensible (section 4.4).
- **Vue Globale** : 16 cartes KPI (4 rangées), 5 graphiques, 2 tableaux.
- **Bibliothèque** : Recharts (AreaChart, BarChart, Donut).
- **Assistant IA** : lecture seule, périmètre BI restreint, **exclut paie/RH/NAS/crédits (Loi 25)**, coût réel × 1,30, bloqué en mode consultation.
- **Alertes** : devis urgents, stock bas, factures en retard.
- **Codes couleur** : marge (vert ≥ 20 / jaune ≥ 0 / rouge < 0), progression (vert ≥ 100 / bleu ≥ 50 / ambre), stock (rouge < 25 / jaune < 50 / vert).
- **Aucune restriction par rôle** : tous les utilisateurs du tenant voient revenus, marges et salaire moyen.
- **Absences assumées** : pas d'export PDF/Excel, pas d'impression, pas de bouton Actualiser, pas de plage personnalisée, pas de forage, pas de personnalisation, pas de temps réel.
- **Limitation d'appels** : `GET /dashboard` et `/dashboard/alerts` non limités ; le reste sous l'intervalle général ; l'assistant IA plafonné à 20 requêtes par fenêtre et par IP.

---

**Fichiers sources vérifiés** :
- `ERP_REACT/frontend/src/pages/AnalyticsPage.tsx` (1456 lignes, 6 onglets)
- `ERP_REACT/frontend/src/components/analytics/DashboardAssistantTab.tsx` (166 lignes)
- `ERP_REACT/frontend/src/api/analytics.ts` (210 lignes), `api/dashboard.ts` (24 lignes), `api/dashboardAi.ts` (43 lignes)
- `ERP_REACT/backend/routers/dashboard.py` (285 lignes, 5 points de terminaison)
- `ERP_REACT/backend/routers/analytics.py` (1553 lignes, 25 points de terminaison — 20 actifs, 5 orphelins)
- `ERP_REACT/backend/routers/dashboard_ai.py` (456 lignes, 1 point de terminaison)
- `ERP_REACT/backend/erp_models.py` (`DashboardStats`, `DashboardAlert`, `DashboardResponse`)
- `ERP_REACT/frontend/src/App.tsx` (routage `/`, `/analyses` → `/dashboard`), `Sidebar.tsx:40`

**Manuels liés** :
- Module 02 — Analyses (`02-principal-analyses.md`) : ancienne page désormais fusionnée dans le présent module.
- Module 06 — CRM (`06-gestion-crm-opportunites.md`) : source du pipeline commercial.
- Module 08 — Soumissions (`08-ventes-soumissions.md`) : source du taux de conversion des devis.
- Module 09 — Projets (`09-ventes-projets.md`) : source de la rentabilité et de la progression.
- Module 10 — Magasin (`10-operations-magasin.md`) : source des alertes et de la valeur de stock.
- Module 11 — Employés (`11-operations-employes.md`) + Module 13 — Pointage (`13-operations-pointage.md`) : source de la productivité RH.
- Module 14 — Bons de commande (`14-operations-bons-de-commande.md`) : source du top fournisseurs.
- Module 15 — Comptabilité (`15-operations-comptabilite.md`) : source des factures, revenus, solde dû et vieillissement.
- Module 25 — Assistant IA (`25-communication-assistant-ia.md`) : assistant conversationnel complet (l'assistant du tableau de bord en est une variante BI en lecture seule).
