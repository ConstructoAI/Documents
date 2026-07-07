# Module 15 — Comptabilité (grand livre, factures, paie, fonds de prévoyance)

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/ComptabilitePage.tsx` (≈ 6 437 lignes — page unique à **18 onglets fixes + 1 onglet conditionnel**), sous-composants `components/comptabilite/ComptesPayablesTab.tsx` (≈ 399 lignes), `DecomptesTab.tsx` (≈ 441 lignes), `WipTab.tsx` (≈ 293 lignes), `RetenuesTab.tsx` (≈ 302 lignes), `ComptaAssistantTab.tsx` (≈ 372 lignes) ; client API `frontend/src/api/accounting.ts` (≈ 1 527 lignes)
> - Backend : `backend/routers/accounting.py` (≈ 15 455 lignes — **93 points d'accès**, dont **16 réservés au rôle « comptable »**), `backend/routers/accounting_ai.py` (≈ 895 lignes — assistant comptable), `backend/routers/payroll.py` (≈ 1 789 lignes — **module Paie, à part**), `feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (feuillets fiscaux — **module Paie, à part**), `feuillets_common.py` (socle NAS/employeur), `fonds_prevoyance.py` (≈ 3 213 lignes — **volet Immobilier/Terrain, à part**)
> - Préfixe API : `/api/erp/v1` — la comptabilité répond sous `/accounting`, l'assistant sous `/accounting/ai`
> - Espace de traduction (i18n) : `compta` (`frontend/src/i18n/locales/{fr,en}/compta.json`)
> **Tables PostgreSQL (par tenant)** : `factures` (en-tête des factures clients et fournisseurs, décomptes et notes de crédit), `facture_lignes` (lignes de facture), `journal_entries` / `journal_lines` (grand livre en partie double), `plan_comptable` (comptes), `cost_centers` (centres de coûts), `periodes_comptables` (périodes ouvertes/clôturées), `retenues_chantier` (retenues de garantie), `cedule_postes` / `decompte_lignes` (décomptes progressifs CCDC), `immobilisations` / `amortissement_ecritures`, `factures_recurrentes`, `accounting_audit_log` (journal d'audit sept ans). Tables des modules connexes : `payroll_*` et `feuillets_fiscaux` (module Paie), `fp_*` (fonds de prévoyance, volet Immobilier).
> **Cadrage** : ce module est la **comptabilité générale et la facturation** d'une entreprise de construction. Une seule page à onglets réunit la **facturation** des clients et des fournisseurs, le **grand livre** (plan comptable et écritures en partie double), les **états financiers**, la **facturation de construction** (décomptes progressifs CCDC, travaux en cours, retenues de garantie 1150 / 2150), les **taxes** TPS/TVQ, les **immobilisations**, les **périodes comptables** et un **assistant IA comptable**. Le module fonctionne en **multidevise et multijuridiction** (Québec–Canada par défaut ; États-Unis de façon conditionnelle). **Point de vocabulaire important** : malgré le titre, la **paie**, les **feuillets fiscaux** (T4, RL-1, PD7A) et le **fonds de prévoyance** (Loi 16) **ne se trouvent PAS dans ce module**. Ils vivent dans d'autres pages du système (voir §1.2). Ce manuel les couvre au chapitre des intégrations (§5), parce qu'ils **alimentent** la comptabilité, mais il vous dit clairement **où** les utiliser.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Procédures pas à pas](#3-procédures-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module Comptabilité est le **grand livre unique** de l'entreprise. Il permet à une personne autorisée de :

- **facturer les clients** : créer une facture de vente numérotée (`FACT-2026-00031`), y saisir des lignes, appliquer la TPS et la TVQ, l'envoyer par courriel (avec un **lien public** de 90 jours), enregistrer les **paiements** partiels ou complets, produire une **note de crédit** conforme à la Loi sur la taxe de vente du Québec, mettre en place des **factures récurrentes** et relancer les mauvais payeurs par des **rappels** à quatre niveaux ;
- **saisir les dépenses fournisseurs** : entrer une facture d'achat (à la main ou par **scan IA**), la ventiler sur plusieurs comptes de charge, suivre les **comptes payables** (vieillissement des soldes) et enregistrer les décaissements ;
- **tenir le grand livre en partie double** : gérer le **plan comptable** (35 comptes préchargés au Québec), passer des **écritures de journal** équilibrées, les **valider**, consulter le **grand livre** compte par compte et les **transactions** ;
- **produire les états financiers** : bilan, état des résultats, flux de trésorerie et **déclaration de taxes** TPS/TVQ (ou taxe de vente américaine), tous exportables ;
- **facturer la construction** selon les usages du secteur : **décomptes progressifs** (méthode CCDC avec cédule de valeurs), suivi des **travaux en cours** (WIP, méthode coût à coût) et **retenues de garantie** (comptes 1150 pour les clients, 2150 pour les sous-traitants) ;
- **gérer les immobilisations** et leur amortissement (linéaire ou dégressif) ;
- **fermer les périodes comptables** et bloquer toute écriture rétroactive dans une période clôturée ;
- **exporter** vers QuickBooks (format IIF), Sage 50 (CSV) et en CSV/Excel/PDF ;
- interroger un **assistant IA comptable** qui **propose** des écritures ou des factures que vous **confirmez** avant qu'elles n'existent.

### 1.2 Où vivent réellement « paie » et « fonds de prévoyance »

Le titre de ce module mentionne la paie et le fonds de prévoyance parce qu'ils font partie de l'univers comptable de l'entreprise. **Mais leur interface n'est pas dans `/comptabilite`.** Voici la carte exacte, pour éviter toute recherche inutile :

| Thème du titre | Où l'utiliser dans le système | Manuel dédié |
|----------------|-------------------------------|--------------|
| **Grand livre** et **factures** | Menu **Comptabilité** (`/comptabilite`) — **c'est ce module** | Ce manuel (§2 à §4) |
| **Paie** (RRQ, RRQ2, RQAP, AE, FSS, CNESST, CCQ, périodes de paie, bulletins PDF) | Menu **Pointage** (`/pointage`), page **« Pointage & Paie »**, onglets **Paie** et **Paie CCQ** | Module 13 — Pointage et heures |
| **Feuillets fiscaux** (T4, RL-1, remise PD7A) | Menu **Pointage** (`/pointage`), onglet **Feuillets** | Module 13 — Pointage et heures |
| **Fonds de prévoyance** (Loi 16, études, projections sur 25 ans) | Volet **Immobilier / Terrain** (vitrine `/immo`, back-office promoteur) | Module 19 — Immobilier |

La comptabilité et ces modules **communiquent** : une paie génère automatiquement une écriture de journal en brouillon dans la comptabilité (§5.1), et les feuillets sont alimentés par les cumuls annuels de la paie (§5.2). Le fonds de prévoyance, lui, est un outil de gestion de copropriété **totalement indépendant** de la comptabilité de l'entrepreneur ; il n'y a **aucun écran de fonds de prévoyance dans `/comptabilite`** (§5.3).

### 1.3 Ce que le module ne fait PAS

- **Pas de paie ni de bulletins de salaire ici.** La paie québécoise complète est dans le module **Pointage** (`/pointage`). Aucun onglet de la Comptabilité ne calcule un salaire.
- **Pas de feuillets T4 / RL-1 / PD7A ici.** Ils sont aussi dans **Pointage**.
- **Pas de fonds de prévoyance ici.** C'est un outil du volet Immobilier.
- **On ne modifie pas les lignes d'une facture dans le panneau de détail.** Le détail affiche les lignes en **lecture seule** ; toute modification passe par la fenêtre **Modifier la facture**.
- **On ne fait pas passer une facture à « Payée » à la main.** Le statut d'une facture se change en ligne, **sauf** vers « Payée » ou « Partiellement payée » : ces deux-là ne s'obtiennent qu'en **enregistrant un paiement**.
- **On ne supprime pas n'importe quelle facture.** La suppression est réservée aux factures en **Brouillon** ou **Annulée**. Une facture comptabilisée se neutralise par **annulation** (contre-passation automatique), jamais par suppression de l'écriture.
- **On n'efface jamais une écriture de journal validée.** Une écriture **Validée** est figée pour toujours (norme comptable) ; on la corrige par une **contre-passation** (écriture inverse), jamais par un effacement.
- **La récurrence n'a pas de bouton « Nouvelle ».** Un modèle de facture récurrente se crée **depuis une facture existante** (bouton « Récurrence » du panneau de détail), pas depuis l'onglet Récurrence.
- **Le coût estimé des travaux en cours n'est pas toujours fiable.** L'onglet WIP affiche « à vérifier » ou « indisponible » quand la source du coût estimé (le devis accepté) n'a pas été propagée au projet.
- **Certains feuillets ne sont pas transmissibles en l'état.** Le RL-1 est **préliminaire et non officiel** (certification de Revenu Québec non obtenue) ; le T4 et le PD7A sont des **aides** dont la conformité aux gabarits officiels reste à finaliser (voir §5.2).

### 1.4 Accès par le menu latéral

- **Menu latéral** → **Comptabilité** (icône calculatrice `Calculator`).
- **Adresse** : `/comptabilite`.
- **Onglet ouvert par défaut** : **Vue Globale** (le tableau de bord financier).
- **Ouverture directe d'une facture** : un lien du type `/comptabilite?open=<id>` (par exemple depuis la fiche d'un dossier) bascule automatiquement sur l'onglet **Factures** et ouvre la fiche demandée.
- Page protégée : il faut être authentifié dans un tenant.

### 1.5 Permissions et rôles

Deux niveaux de protection se croisent dans ce module.

**a) Gardes d'affichage (dans l'écran).** Deux onglets n'apparaissent que dans certaines conditions :

| Onglet | Condition d'affichage |
|--------|-----------------------|
| **Conditions** | Réservé aux **administrateurs** (`role == 'admin'` ou `is_admin`) |
| **Tax USA** | Affiché uniquement pour les entreprises dont le pays est **les États-Unis** (`tenantCountry == 'US'`) |

**b) Gardes d'écriture (côté serveur).** La **consultation** de la comptabilité est ouverte à tout utilisateur authentifié du tenant. Les **opérations sensibles** exigent le rôle **comptable** (garde `require_tenant_admin_or_role('comptable')`, qui admet aussi d'office tout administrateur ou super-administrateur, même si son rôle nominal est « utilisateur »). Seize points d'accès sont protégés ainsi :

- création, modification et désactivation d'un **compte** du plan comptable ;
- création, ajout de lignes et **validation** d'une **écriture de journal** ;
- **suppression**, **paiement** et **note de crédit** d'une facture ;
- création et **libération** d'une **retenue de garantie** ;
- génération d'un **amortissement** ;
- lecture des **travaux en cours** (WIP) ;
- **clôture** et **réouverture** d'une période comptable.

> **À savoir.** La **création** d'une facture, l'ajout de lignes et la comptabilisation qui survient quand une facture **sort du brouillon** sont ouvertes à tout utilisateur authentifié — ce n'est pas réservé au comptable. Autrement dit, un utilisateur ordinaire peut facturer et déclencher l'écriture de vente au grand livre ; les gestes plus délicats (supprimer, encaisser, créditer, clôturer) restent, eux, réservés au rôle comptable ou à un administrateur.

**c) Mode consultation (lecture seule).** Si l'abonnement du tenant est absent ou annulé, le système bascule en **mode consultation** : la lecture reste possible partout, mais **toute écriture est refusée** (erreur 403). C'est une protection globale, indépendante du module.

### 1.6 Cartes de résumé et bouton « Synchroniser »

Dès qu'un résumé est chargé, **quatre cartes chiffrées** coiffent la page (elles concernent l'ensemble de la facturation) :

| Carte | Ce qu'elle montre |
|-------|-------------------|
| **CA total** | Chiffre d'affaires facturé aux clients (net des notes de crédit) |
| **Encaissé** | Total des paiements clients reçus |
| **Solde dû** | Comptes clients restant à percevoir |
| **Factures en retard** | Nombre et montant des factures dépassant leur échéance |

En haut à droite, le bouton **« Synchroniser »** lance `POST /accounting/sync-all` : il **génère rétroactivement** les écritures de grand livre manquantes à partir des factures, des paiements, des bons de commande reçus et des heures d'employés, puis recharge l'onglet courant et le résumé. L'opération est **idempotente** (elle ne crée jamais de doublon).

### 1.7 Les 18 onglets (plus un pour les États-Unis)

Les onglets sont regroupés par thème, avec un séparateur visuel entre chaque groupe.

| Onglet | Libellé court | Groupe | À quoi il sert |
|--------|---------------|--------|----------------|
| **Vue Globale** | Vue gl. | Aperçu | Tableau de bord : chiffre d'affaires, dépenses, profit, ventilation mensuelle |
| **Factures** | Factures | Facturation | Factures clients et fournisseurs (l'onglet le plus riche) |
| **Comptes payables** | À payer | Facturation | Ce que l'entreprise doit à ses fournisseurs, par échéance |
| **Conditions** *(admin)* | Conditions | Facturation | Conditions de facturation et coordonnées bancaires par défaut |
| **Récurrence** | Récur. | Facturation | Modèles de factures répétées automatiquement |
| **Décomptes** | Décomptes | Construction | Décomptes progressifs CCDC (cédule de valeurs) |
| **Travaux en cours** | WIP | Construction | Avancement et écarts de facturation par projet |
| **Retenues** | Ret. | Construction | Retenues de garantie clients et sous-traitants |
| **Plan comptable** | Plan | Grand livre | Liste des comptes |
| **Journal** | Journal | Grand livre | Écritures en partie double |
| **Transactions** | Trans. | Grand livre | Vue agrégée revenus / dépenses / notes de crédit |
| **Grand Livre** | G.Livre | Grand livre | Mouvements d'un compte avec solde cumulatif |
| **États Financiers** | États | Rapports | Bilan, Résultats, Flux de trésorerie, Taxes |
| **Centres de Coûts** | Coûts | Rapports | Budgets et dépenses par centre |
| **Périodes** | Périodes | Gestion | Ouverture et clôture des périodes comptables |
| **Immobilisations** | Immo. | Gestion | Actifs et amortissement |
| **Assistant IA** | IA | Assistant | Assistant comptable (proposer → confirmer) |
| **Tax USA** *(É.-U.)* | Tax US | Taxe | Feuillets 1099-NEC et formulaires W-9 |

### 1.8 Concepts clés

- **Partie double.** Chaque opération produit une **écriture de journal** dont la somme des **débits** égale la somme des **crédits** (tolérance d'un cent). Une écriture non équilibrée est refusée.
- **HT et TTC.** Les montants « HT » sont hors taxes ; « TTC » inclut la TPS et la TVQ. Les taxes sont calculées **au niveau de la facture**, pas ligne par ligne.
- **Instantané des taxes.** Chaque facture **fige** ses libellés et taux de taxe à la création (par exemple TPS 5 %, TVQ 9,975 % au Québec). Modifier plus tard la configuration de l'entreprise ne change pas les factures déjà émises.
- **Statut « Brouillon » puis comptabilisation.** Une facture en **Brouillon** n'a pas encore d'écriture au grand livre. Dès qu'elle **sort du brouillon** (Envoyée, etc.), le système génère automatiquement l'écriture de vente (ou d'achat).
- **Contre-passation.** On n'efface jamais une écriture validée : pour annuler son effet, le système crée une écriture **inverse** (débits et crédits permutés). C'est ce qui se produit quand vous annulez une facture, créditez une facture ou ramenez une facture comptabilisée en brouillon.
- **Période comptable.** Un intervalle de dates qu'on peut **clôturer**. Une fois clôturée, **aucune écriture** ne peut plus être passée à une date comprise dans la période.
- **Numéros de documents.** Ils sont attribués de façon **sûre face à la concurrence** (jamais par comptage) : factures `FACT-2026-00031`, notes de crédit `AV-2026-00007`, écritures `JE-VTE-00042`, retenues `JE-RET`, etc.

---

## 2. Interface

Cette section décrit chaque onglet et chaque fenêtre du module `/comptabilite`. Les intégrations Paie, Feuillets et Fonds de prévoyance (qui ne sont pas dans cet écran) sont traitées au §5.

### 2.1 Onglet « Vue Globale » (tableau de bord financier)

C'est l'onglet ouvert par défaut. Il présente **trois cartes** — **Chiffre d'affaires**, **Dépenses totales**, **Profit net** — puis une carte **« Ventilation mensuelle »** : un tableau Mois / Revenus / Dépenses / Profit avec une **ligne de total**. Si aucune donnée n'existe encore, un message d'invite s'affiche. Source : `GET /accounting/dashboard`.

### 2.2 Onglet « Factures »

L'onglet le plus complet. Vue **maître-détail** : la liste des factures à gauche, le détail à droite.

#### 2.2.1 Barre d'actions

- **Nouvelle facture** : ouvre la fenêtre de création d'une facture **client** (revenu).
- **Nouvelle dépense** : ouvre la même fenêtre, préréglée en facture **fournisseur** (dépense).
- **Scanner facture IA** (bouton violet, icône appareil photo) : ouvre un sélecteur de fichier (image ou PDF) ; l'IA lit la facture fournisseur et pré-remplit les champs (voir §2.19 et §3.11).
- **Filtre de statut** (menu déroulant) : Tous, Brouillon, Envoyée, Partiellement payée, Payée, En retard, Annulée.
- **Recherche** : par numéro ou par nom de client/fournisseur.

#### 2.2.2 Liste des factures

Sur ordinateur, un tableau à quatre colonnes : **Numéro**, **Client**, **Statut**, **Total TTC**. Le **statut** est un badge coloré **modifiable en ligne** par un menu déroulant — **sauf** les statuts « Payée » et « Partiellement payée », qui ne s'obtiennent que par un paiement. Sur téléphone, chaque facture devient une carte (numéro, badge, client, date, total). Une liste vide affiche un message d'invite.

#### 2.2.3 Panneau de détail

En sélectionnant une facture, le panneau de droite (plein écran sur téléphone) affiche :

- **En-tête** : numéro, nom du client, bouton **crayon** (Modifier — masqué si la facture est Payée ou Annulée), bouton de fermeture.
- **Badge de statut**, dates (facture / échéance), **projet associé**, **notes** et **notes internes**.
- **Bloc des montants**, avec des **libellés de taxe dynamiques** (tirés de la facture ; par défaut TPS 5 % et TVQ 9,975 %) : Sous-total HT, taxe 1, taxe 2, **Total TTC**, Payé, **Solde dû**.
- **Barre d'actions**, dont les boutons apparaissent selon le statut :

| Bouton | Quand il apparaît | Effet |
|--------|-------------------|-------|
| **Envoyer** | Brouillon | Fenêtre de confirmation puis comptabilisation et passage à Envoyée |
| **Courriel** | Sauf Payée / Annulée | Envoi de la facture par courriel (PDF joint, lien public 90 jours) |
| **Paiement** | Sauf Payée / Annulée / Avoir | Enregistre un encaissement |
| **Note de crédit** | Sauf Brouillon / Annulée / Avoir | Émet un avoir |
| **Récurrence** | Sauf Annulée / Avoir | Crée un modèle de facture récurrente |
| **Rappel** | Envoyée / Partiellement payée / En retard | Envoie une relance |
| **Historique des rappels** | Si au moins un rappel a été envoyé | Liste des rappels |
| **Aperçu HTML** | Toujours | Aperçu de la facture mise en page |
| **PDF** | Toujours | Télécharge le PDF (moteur WeasyPrint) |
| **Imprimer** | Toujours | Ouvre le HTML pour impression |
| **Excel** | Toujours | Télécharge le fichier `.xlsx` |
| **CSV QuickBooks** | Toujours | Télécharge un CSV compatible QuickBooks (et « Copier CSV » vers le presse-papiers) |
| **Supprimer** | Brouillon ou Annulée | Supprime la facture (contre-passation automatique si une écriture est liée) |

- **Section Lignes** (en **lecture seule**) : description, quantité × prix unitaire, montant. Pour changer une ligne, utilisez la fenêtre **Modifier la facture**.

### 2.3 Onglet « Comptes payables »

Vue autonome de ce que l'entreprise **doit** à ses fournisseurs. Source : `GET /accounting/payables/summary`.

- **Trois indicateurs** : **Total à payer**, **En retard**, **Fournisseurs** (nombre).
- Carte **« Vieillissement des soldes (par échéance) »** : cinq tranches — **Non échu**, **1-30 jours**, **31-60 jours**, **61-90 jours**, **90+ jours**.
- **Tableau par fournisseur** : Fournisseur, Factures, Total dû, En retard, Plus ancienne échéance. Chaque ligne est **cliquable** : elle ouvre le **détail** des factures impayées du fournisseur (numéro, date, échéance, solde dû, statut) avec un bouton **Payer**.
- **Fenêtre « Enregistrer un décaissement »** : montant, date, mode (Virement, Chèque, Carte, Comptant, Autre). Elle passe le paiement au fournisseur (débit du compte 2100 Fournisseurs, crédit du compte 1010 Encaisse).

### 2.4 Onglet « Conditions » (administrateurs seulement)

Deux cartes de réglages qui alimentent le **bas des factures** et leur PDF. Source : `GET /accounting/factures/defaults`.

- **« Conditions de facturation »** : une zone de texte (une condition par ligne), avec les boutons **Insérer le modèle** (le modèle par défaut du pays), **Réinitialiser** et **Enregistrer**.
- **« Coordonnées bancaires (dépôt direct) »** : Institution financière, Numéro d'institution (3 chiffres), Transit (5 chiffres), Numéro de compte (5 à 20 chiffres), Bénéficiaire, et une **case « Afficher sur les factures »** (désactivée tant qu'aucun champ n'est rempli). Un **aperçu en direct** montre la section Paiement telle qu'elle apparaîtra. Les mêmes validations sont appliquées côté serveur.

### 2.5 Onglet « Récurrence »

Liste des **modèles de factures récurrentes**. Un **filtre de statut** (Tous, Actifs, En pause, Terminés, Annulés). Le tableau montre : Nom, Client, **Fréquence** (avec son multiplicateur), Prochaine génération, Générées (sur maximum), Statut, Envoi automatique par courriel, Actions. Actions par ligne : **Pause**, **Réactiver**, **Générer immédiatement**, **Annuler** le modèle.

> **Comment créer un modèle ?** Il n'y a **pas** de bouton « Nouveau » ici. On crée un modèle depuis une **facture existante**, par le bouton **Récurrence** de son panneau de détail (voir §2.2.3 et §3.7).

### 2.6 Onglet « Décomptes » (facturation progressive CCDC)

Facturation par avancement, à la manière des décomptes progressifs du secteur de la construction (usage CCDC). On choisit d'abord un **projet**, puis :

- **Carte « Cédule de valeurs »** : la répartition du contrat en **postes**. Si la cédule est vide, un champ **Numéro de devis** et un bouton **Importer du devis** permettent de la créer automatiquement à partir d'un devis accepté. Le tableau montre, par poste : le libellé (et le pourcentage déjà facturé), la **Valeur au contrat**, le **Déjà facturé**, un champ **% complété à date** (modifiable) et le **Ce décompte** (le montant calculé). Une ligne de **total** et un rappel du taux de retenue complètent la carte. Le bouton **Créer le décompte** génère la facture de décompte (désactivé tant que le total est nul).
- **Carte « Décomptes émis »** : les décomptes déjà produits (Numéro `#n`, Date, Total TTC, Retenue, **Net à payer**, Statut).
- **Fenêtre « poste »** (ajout ou modification) : **Description** et **Valeur au contrat**.

> **Limite d'interface.** La fenêtre de poste n'expose que la description et la valeur. Les champs techniques `code_poste`, `categorie` et `sequence_poste` existent dans l'API mais ne sont **pas** offerts à l'écran.

### 2.7 Onglet « Travaux en cours » (WIP)

Suivi de l'avancement financier des chantiers par la méthode **coût à coût**, en lecture seule. Source : `GET /accounting/wip`. On choisit une date « Au ». Cinq indicateurs coiffent l'onglet : **Valeur du contrat**, **Revenu reconnu**, **Facturé**, **Surfacturation**, **Sous-facturation**. Puis un tableau par projet :

- **Avancement** (pourcentage et barre ; rouge en cas de dépassement ; un triangle d'alerte signale un coût estimé non fiable ou indisponible) ;
- **Coûts encourus**, **Valeur du contrat**, **Revenu reconnu**, **Facturé** ;
- **Écart** : bleu = **surfacturation** (facturé au-delà de l'avancement, un passif) ; ambre = **sous-facturation** (facturé en deçà, un actif).

Chaque ligne se **déplie** pour montrer les sources : coût de main-d'œuvre (avec la mention « paie réelle » ou « estimation »), coût des matériaux, coût estimé (badge « À vérifier » si la fiabilité est douteuse) et la source de la valeur du contrat.

> **Pourquoi « à vérifier » ?** Le coût estimé provient du **devis accepté**. Quand ce coût n'a pas été propagé au projet, l'onglet le signale honnêtement plutôt que d'afficher un avancement faux.

### 2.8 Onglet « Retenues »

Gestion des **retenues de garantie**, pour les **clients** (compte 1150, « Retenues à recevoir ») comme pour les **sous-traitants** (compte 2150, « Retenues sur sous-traitants à payer »). Le type est **déduit de la facture** liée, pas choisi à la main.

- **Barre d'outils** : filtre **Toutes / Clients / Sous-traitants** et bouton **Nouvelle retenue**.
- **Tableau** : Type (badge Client ou Sous-traitant), Facture, Client, Taux, Montant retenu, Fin des travaux, Libération, Statut (**Retenue** en jaune, **Libérée** en vert), et un bouton **Libérer** (avec un avertissement : l'écriture de libération est **validée et irréversible**).
- **Fenêtre « Nouvelle retenue de garantie »** : choix d'une **facture** (émise, hors brouillon et annulée), **Taux (%)** (vide = taux par défaut du tenant), **Date de fin des travaux**, **Notes**.

### 2.9 Onglet « Plan comptable »

La liste des **comptes**. Barre d'outils : **Export CSV** et **Nouveau compte**. Une **case « Afficher les comptes inactifs »**. Le tableau montre : **Code** (indenté selon le niveau), **Nom**, **Type**, **Solde normal**, **Actif**, et des actions (**Modifier**, **Désactiver / Réactiver**). Au premier affichage, le plan est **préchargé automatiquement** selon le pays du tenant (35 comptes au Québec ; voir §4.4).

### 2.10 Onglet « Journal »

Les **écritures de journal** en partie double. Barre d'outils : **Export CSV**, **QuickBooks IIF**, **Sage 50 CSV**, **Nouvelle écriture**. Le tableau montre : Numéro d'écriture, Date, Description, **Type** (Ventes, Achats, Encaissement, Décaissement, Paie, Amortissement, etc.), Montant, Statut (**Brouillon**, **Validée**, **Annulée**), et un bouton **Valider** sur les écritures en brouillon.

> Une écriture **Validée** est **figée** : on ne peut plus lui ajouter de ligne. Pour la corriger, on passe une écriture d'ajustement (contre-passation).

### 2.11 Onglet « Transactions »

Vue agrégée et lisible des mouvements. Filtre par type : **Tous / Revenu / Dépense**. Le tableau montre : **Type** (badge vert pour un revenu, rouge pour une dépense, gris pour une note de crédit), Référence, Date, Description, Montant (signé), Statut. Les avoirs apparaissent en revenu négatif (gris).

### 2.12 Onglet « Grand Livre »

Le détail des mouvements **d'un compte**. On choisit un **compte** dans un menu déroulant, puis on clique **Charger**. Le tableau affiche : Date, Numéro d'écriture, Libellé, **Débit**, **Crédit** et **Solde cumulatif**. Seules les écritures **validées** sont prises en compte.

### 2.13 Onglet « États Financiers »

Quatre **sous-onglets**. Tous ne comptent que les écritures **validées**.

- **Bilan** : filtre « Date du bilan (au) », bouton **Exercice courant** et **Appliquer**. Sections **Actifs** (court et long terme), **Passifs** (court et long terme), **Capitaux propres**, totaux et un indicateur **Équilibré / écart**. Le **résultat net de l'exercice** est injecté dans les capitaux propres.
- **Résultats** : Revenus, Coûts des ventes, **Marge brute**, Frais d'exploitation, **Résultat net**.
- **Flux de trésorerie** : un tableau Mois / Entrées / Sorties / Net. *(Ce sous-onglet n'a pas de filtre de dates propre, contrairement au Bilan et aux Taxes.)*
- **Taxes** : filtre **Du / Au**, boutons **Calculer** et **Export CSV**. Trois cartes de net — **taxe 1** (TPS), **taxe 2** (TVQ) et **Total net dû** (« À remettre » ou « Remboursement ») — puis un tableau mensuel Collectée / Payée / Net par taxe. Le calcul est **conscient des avoirs** (les taxes des notes de crédit sont soustraites).

### 2.14 Onglet « Centres de Coûts »

Barre d'outils : **Nouveau centre de coûts**. Le tableau des centres montre : Code, Nom, Type (Projet, Département, Activité, Autre), Budget annuel. Une carte **« Résumé des coûts par centre »** compare Budget, Dépenses et **Écart** (vert ou rouge). Un centre de coûts de projet porte le code `PRJ-00007`.

### 2.15 Onglet « Périodes »

Barre d'outils : **Nouvelle période**. Le tableau montre : Nom, Début, Fin, Statut (**Ouverte** / **Clôturée**), Clôturée par, et un bouton **Clôturer** (avec avertissement d'irréversibilité) ou **Réouvrir** (réservé aux comptables et administrateurs).

> **Effet réel de la clôture.** Contrairement aux anciennes versions du système, la clôture **bloque effectivement** toute écriture datée dans la période : le serveur refuse d'y comptabiliser quoi que ce soit. C'est un verrou, pas un simple marqueur.

### 2.16 Onglet « Immobilisations »

Quatre indicateurs : **Nombre d'actifs**, **Coût d'acquisition**, **Amortissement cumulé**, **Valeur nette**. Barre d'outils : un champ **mois**, un bouton **Générer amortissement** et un bouton **Nouvel actif**. Le tableau montre : Nom, Catégorie, Date d'acquisition, Coût, Amortissement cumulé, Valeur nette, Méthode (et durée en mois).

### 2.17 Onglet « Assistant IA »

Un **assistant comptable** conversationnel. L'en-tête affiche « Assistant IA comptable » et trois exemples de questions. L'assistant fonctionne selon le principe **proposer → confirmer** : il peut **rechercher** dans la base (en lecture seule et sur une liste blanche stricte de tables comptables — jamais la paie ni les données RH), puis **proposer** soit une **écriture de journal**, soit une **facture**. Chaque proposition s'affiche dans une **carte** (avec ses lignes de débit/crédit ou de description/quantité/prix et ses totaux) et des boutons **Confirmer** / **Annuler**. **Aucune écriture n'est créée sans votre confirmation.** Des verrous empêchent un double envoi ou une double confirmation, et les métadonnées d'usage (jetons, coût, temps) sont affichées. Source : `POST /accounting/ai/chat` et `POST /accounting/ai/confirm-action`.

### 2.18 Onglet « Tax USA » (entreprises américaines seulement)

Deux sections, visibles uniquement si le pays du tenant est les États-Unis.

- **1099-NEC Contractors** : un sélecteur d'année fiscale ; un tableau avec cases à cocher, Contractor, TIN (masqué), Cumul payé (USD), présence d'un **W-9** (badge Oui/Non), et la **retenue de 24 %** (backup withholding). Le bouton **Générer 1099-NEC** ouvre une fenêtre (format PDF, CSV IRIS, ou les deux ; liste et liens de téléchargement).
- **W-9 Contractors** : un bouton **Demander W-9** (choix d'un fournisseur ; envoi d'un lien public). Le tableau montre : Nom, Statut (En attente, Envoyé, Reçu, Vérifié, Expiré), TIN, Date de signature, et les actions **Vérifier** et **PDF**. Le destinataire remplit un **formulaire W-9 public** complet.

### 2.19 Les fenêtres (modales)

Les fenêtres suivantes s'ouvrent depuis les onglets ci-dessus :

| Fenêtre | Ouverte depuis | Champs principaux |
|---------|----------------|-------------------|
| **Nouvelle facture (client / fournisseur)** | Factures | Bascule **Client (revenu) / Fournisseur (dépense)** ; client OU (fournisseur + numéro de facture fournisseur + compte de charge) ; dates d'émission et d'échéance ; projet ; conditions ; **lignes** (description, quantité, prix, montant ; pour un fournisseur, un **compte de charge par ligne** permet une ventilation multipostes) ; totaux en direct ; notes et notes internes |
| **Modifier la facture** | Factures | Mêmes sections + une section **Statut** (Marquer en retard, Annuler la facture) |
| **Scanner une facture fournisseur (IA)** | Factures | Aperçu des données extraites (fournisseur, numéro, dates, HT, TPS, TVQ, TTC, nombre de lignes, niveau de confiance) puis **Créer la facture fournisseur** |
| **Confirmer l'envoi** | Détail facture | Aperçu de l'**écriture comptable** qui sera générée (Ventes ou Achats) |
| **Envoyer par courriel** | Détail facture | Destinataire, CC, sujet, message ; mention du **lien public partageable (90 jours)** |
| **Enregistrer un paiement** | Détail facture | Total TTC, déjà payé, solde dû, montant, date, **mode**, référence, **compte de trésorerie** (par défaut Encaisse 1010) |
| **Note de crédit** | Détail facture | Facture d'origine, **raison**, montant total de l'avoir (TTC), date, notes internes (conformité art. 350 LTVQ) |
| **Facture récurrente** | Détail facture | Nom, **fréquence** (hebdomadaire, bimensuel, mensuel, bimestriel, trimestriel, semestriel, annuel), multiplicateur, date de première génération, date de fin, nombre maximal d'occurrences, statut initial, envoi automatique par courriel |
| **Rappel de paiement** | Détail facture | Quatre niveaux (Courtois J+3, Ferme J+15, Insistant J+30, Mise en demeure J+60), destinataire, message |
| **Nouvelle écriture de journal** | Journal | Description, **type**, lignes (compte, libellé, débit, crédit), indicateur **Équilibré / écart** |
| **Nouveau / Modifier un compte** | Plan comptable | **Code** (4 à 10 chiffres, immuable), nom, type, classe, solde normal, description, actif |
| **Nouvelle période comptable** | Périodes | Année fiscale, période, dates de début et de fin, nom |
| **Nouveau centre de coûts** | Centres de coûts | Code, nom, type, budget annuel |
| **Nouvelle immobilisation** | Immobilisations | Nom, catégorie, date d'acquisition, coût, durée de vie (mois), méthode (linéaire / dégressif), valeur résiduelle |

---

## 3. Procédures pas à pas

### 3.1 Créer et envoyer une facture client

1. Onglet **Factures** → bouton **Nouvelle facture**.
2. Laissez la bascule sur **Client (revenu)**. Choisissez le **client**, la **date d'émission** (l'échéance se remplit automatiquement), au besoin un **projet** et les **conditions**.
3. Ajoutez les **lignes** : description, quantité, prix unitaire. Le sous-total, les taxes (TPS/TVQ) et le total se calculent en direct.
4. Au besoin, saisissez des **notes** (visibles par le client) et des **notes internes**.
5. **Enregistrer**. La facture apparaît en **Brouillon** (aucune écriture au grand livre pour l'instant).
6. Dans le panneau de détail, cliquez **Envoyer**. Une fenêtre montre l'**écriture de vente** qui sera générée : **débit 1100** (clients, TTC), **crédit 4100** (revenus, HT), **crédit 2200** (TPS), **crédit 2210** (TVQ). Confirmez : la facture passe à **Envoyée** et l'écriture est comptabilisée.
7. Pour l'expédier : bouton **Courriel** (le PDF est joint et un **lien public de 90 jours** est créé), **PDF** (téléchargement) ou **Imprimer**.

### 3.2 Saisir une facture fournisseur (dépense)

1. Onglet **Factures** → bouton **Nouvelle dépense** (ou la bascule **Fournisseur** dans la fenêtre de création).
2. Choisissez le **fournisseur**, entrez son **numéro de facture** et un **compte de charge** (par exemple 5100 Coût des matériaux).
3. Ajoutez les **lignes**. Pour ventiler la dépense sur **plusieurs comptes**, indiquez un **compte de charge par ligne** : la comptabilisation créera une ligne de grand livre par compte.
4. **Enregistrer**, puis **Envoyer**. L'écriture d'achat générée est : **débit 5100** (ou vos comptes de charge, HT), **débit 1200** (TPS remboursable / CTI), **débit 1210** (TVQ remboursable / RTI), **crédit 2100** (fournisseurs, TTC).

### 3.3 Numériser une facture fournisseur avec l'IA

1. Onglet **Factures** → bouton **Scanner facture IA**.
2. Choisissez une **image** ou un **PDF** (jusqu'à 20 Mo).
3. L'IA lit le document et affiche les données extraites : fournisseur, numéro, dates, HT, TPS, TVQ, TTC, nombre de lignes et **niveau de confiance**.
4. Vérifiez, corrigez au besoin, puis **Créer la facture fournisseur**. La facture est créée en brouillon comme au §3.2.

> Le scan consomme des **crédits IA** du tenant (coût réel des jetons, majoré de 30 %). Vérifiez toujours le résultat : l'IA propose, vous validez.

### 3.4 Enregistrer un paiement

1. Ouvrez la facture → bouton **Paiement**.
2. Saisissez le **montant** (partiel ou total), la **date**, le **mode** (Virement, Chèque, Carte, Comptant, Autre), une **référence** et, au besoin, le **compte de trésorerie** (par défaut Encaisse 1010).
3. **Enregistrer**. Le système :
   - refuse un **surpaiement** (montant supérieur au solde) ;
   - tient compte des **notes de crédit** actives (le solde net = TTC − avoirs) ;
   - met le statut à **Payée** si le solde tombe à zéro, sinon à **Partiellement payée** ;
   - comptabilise l'**encaissement** : **débit 1010** (encaisse), **crédit 1100** (clients).

Pour une facture fournisseur, l'opération est un **décaissement** : **débit 2100**, **crédit 1010**. On peut aussi la lancer depuis l'onglet **Comptes payables** (bouton **Payer** ou **Enregistrer un décaissement**).

### 3.5 Émettre une note de crédit (avoir)

1. Ouvrez une facture **envoyée ou payée** → bouton **Note de crédit**.
2. Indiquez la **raison**, le **montant total de l'avoir** (TTC), la **date** et des **notes internes**.
3. **Enregistrer**. L'avoir est **créé puis appliqué immédiatement** : une écriture de **contre-passation** équilibrée est passée et le **solde de la facture d'origine** est recalculé. Le cumul des avoirs ne peut jamais dépasser le TTC d'origine. Le numéro suit le format `AV-2026-00007`.

> Conformité : la note de crédit respecte l'article 350 de la Loi sur la taxe de vente du Québec. Les taxes de l'avoir sont **héritées** de la facture d'origine pour garantir l'équilibre.

### 3.6 Relancer un mauvais payeur

1. Ouvrez une facture **envoyée / partiellement payée / en retard** → bouton **Rappel**.
2. Choisissez le **niveau** : Courtois (J+3), Ferme (J+15), Insistant (J+30) ou Mise en demeure (J+60). Ajustez le destinataire et le message.
3. **Envoyer**. L'historique est consultable par le bouton **Historique des rappels**.

> Les rappels peuvent aussi partir **automatiquement** chaque jour (voir §5.7 sur la tâche quotidienne).

### 3.7 Mettre en place une facturation récurrente

1. Ouvrez une facture existante qui servira de **modèle** → bouton **Récurrence**.
2. Réglez le **nom**, la **fréquence** (mensuel, trimestriel, etc.) et son **multiplicateur**, la **date de première génération** (elle doit être dans le futur), la **date de fin** ou le **nombre maximal d'occurrences**, le **statut** des factures produites (Brouillon ou Envoyée) et l'**envoi automatique par courriel**.
3. **Enregistrer**. Le modèle apparaît dans l'onglet **Récurrence**, où vous pouvez le **mettre en pause**, le **réactiver**, **générer immédiatement** une occurrence ou l'**annuler**.

### 3.8 Passer une écriture de journal manuelle

1. Onglet **Journal** → bouton **Nouvelle écriture**.
2. Saisissez la **description**, le **type** (Vente, Achat, Salaire, Ajustement, Autre) et au moins **deux lignes** (compte, libellé, et un **débit** ou un **crédit** — jamais les deux sur la même ligne).
3. L'indicateur **Équilibré / écart** doit être vert (débits = crédits, à un cent près). **Enregistrer** : l'écriture naît en **Brouillon**.
4. Quand elle est prête, cliquez **Valider**. Le système revérifie l'équilibre puis fige l'écriture (**Validée**).

### 3.9 Produire un décompte progressif (CCDC)

1. Onglet **Décomptes** → choisissez le **projet**.
2. Si la **cédule de valeurs** est vide, entrez un **numéro de devis** et cliquez **Importer du devis** (ou ajoutez les **postes** un à un). La valeur d'un poste importé est le **prix de vente** (le montant du devis majoré de l'administration, des contingences et du profit).
3. Pour chaque poste, saisissez le **% complété à date**. La colonne **Ce décompte** affiche le montant à facturer (avancement × valeur au contrat, moins ce qui est déjà facturé).
4. Cliquez **Créer le décompte**. Le système :
   - ne facture que les postes dont l'avancement a **augmenté** ;
   - calcule la **retenue de garantie** (HT × taux, par défaut 10 %) et l'inscrit sur le décompte ;
   - crée une facture de type **Décompte** en **Brouillon**, numérotée par ordre dans le projet ;
   - protège l'opération par un **verrou par projet** (pas de double facturation).
5. Le **Net à payer** = Total TTC − Retenue. Le décompte apparaît dans « Décomptes émis ».

### 3.10 Créer et libérer une retenue de garantie

1. Onglet **Retenues** → bouton **Nouvelle retenue**.
2. Choisissez une **facture** (émise), un **taux** (vide = taux du tenant), la **date de fin des travaux** et des **notes**. **Enregistrer**.
   - Pour un **client** : **débit 1150** (retenues à recevoir), **crédit 1100** (clients).
   - Pour un **sous-traitant** : **débit 2100** (fournisseurs), **crédit 2150** (retenues à payer).
   - La base de calcul est le montant **HT**.
3. À la fin de la garantie, sélectionnez la retenue et cliquez **Libérer**.
   - Pour un **client** : **débit 1010** (encaisse), **crédit 1150**.
   - Pour un **sous-traitant** : **débit 2150**, **crédit 1010**.
   - L'écriture de libération est **validée et irréversible**.

### 3.11 Clôturer une période comptable

1. Onglet **Périodes** → bouton **Nouvelle période** (année fiscale, dates, nom), au besoin.
2. Sur une période **Ouverte**, cliquez **Clôturer** et confirmez.
3. Résultat : **aucune écriture** ne peut plus être passée à une date de la période. Toute tentative de comptabilisation (facture, paiement, décompte, contre-passation) est **refusée**.
4. Un administrateur ou un comptable peut **Réouvrir** une période si nécessaire.

### 3.12 Enregistrer et amortir une immobilisation

1. Onglet **Immobilisations** → bouton **Nouvel actif** : nom, catégorie, date d'acquisition, **coût**, **durée de vie** (en mois), **méthode** (linéaire ou dégressif), valeur résiduelle.
2. Chaque mois, indiquez le **mois** dans la barre d'outils et cliquez **Générer amortissement** : le système calcule la dotation (linéaire = (coût − résiduel) / durée ; dégressif = valeur nette × taux / 12) et l'inscrit au grand livre. La dernière période solde exactement la valeur résiduelle.

### 3.13 Exporter les données comptables

- **Onglet Journal** : **Export CSV**, **QuickBooks IIF**, **Sage 50 CSV**.
- **Onglet Plan comptable** : **Export CSV**.
- **Onglet Grand Livre** : export du grand livre en CSV.
- **États Financiers → Taxes** : **Export CSV** de la déclaration.
- **Par facture** (panneau de détail) : **PDF**, **Excel (.xlsx)**, **CSV QuickBooks** (et « Copier CSV »).

Tous les CSV sont encodés en UTF-8 avec un BOM (pour un affichage correct des accents dans Excel) et protégés contre l'injection de formules.

### 3.14 Utiliser l'assistant IA comptable

1. Onglet **Assistant IA**. Posez votre demande en langage naturel (par exemple « passe l'écriture de l'assurance de 1 200 $ payée comptant »).
2. L'assistant peut **consulter** vos données comptables, puis **proposer** une écriture ou une facture sous forme de **carte** détaillée.
3. Vérifiez la proposition, puis cliquez **Confirmer** (l'écriture est alors créée) ou **Annuler**. Rien n'est écrit tant que vous n'avez pas confirmé.

---

## 4. Référence

### 4.1 Statuts de facture

| Statut | Couleur | Comment on l'obtient |
|--------|---------|----------------------|
| **Brouillon** | gris | À la création. Aucune écriture au grand livre |
| **Envoyée** | indigo | En quittant le brouillon (déclenche la comptabilisation) |
| **Partiellement payée** | ambre | **Automatique** après un paiement partiel |
| **Payée** | vert | **Automatique** quand le solde tombe à zéro |
| **En retard** | rouge | Manuel, ou **automatique** par la tâche quotidienne (échéance dépassée, solde restant) |
| **Annulée** | gris | Par annulation (contre-passation automatique de l'écriture liée) |

Types de document : **Facture**, **Avoir** (note de crédit), **Décompte** (facturation progressive).

### 4.2 Écritures de grand livre générées automatiquement

| Opération | Débit | Crédit | Type de journal |
|-----------|-------|--------|-----------------|
| Facture **client** (vente) | 1100 Clients (TTC) | 4100 Revenus (HT) + 2200 TPS + 2210 TVQ | Ventes |
| Note de crédit **client** | inverse de la vente | inverse | Ventes (avoir) |
| Facture **fournisseur** (dépense) | 5100 (ou comptes de charge, HT) + 1200 TPS + 1210 TVQ | 2100 Fournisseurs (TTC) | Achats |
| Note de crédit **fournisseur** | inverse de l'achat | inverse | Achats (avoir) |
| **Encaissement** client | 1010 Encaisse | 1100 Clients | Encaissement |
| **Décaissement** fournisseur | 2100 Fournisseurs | 1010 Encaisse | Décaissement |
| **Retenue** client | 1150 Retenues à recevoir | 1100 Clients | JE-RET |
| **Retenue** sous-traitant | 2100 Fournisseurs | 2150 Retenues à payer | JE-RET |
| **Libération** retenue client | 1010 Encaisse | 1150 | JE-LIB |
| **Libération** retenue sous-traitant | 2150 | 1010 Encaisse | JE-LIB |
| **Paie** (depuis le module Pointage) | 6100 Salaires (brut + charges) | 2300 Net + 2310 Retenues à la source + 2320 Charges | Paie (brouillon) |
| **Amortissement** | 6900 Amortissements | 1510 Amortissement cumulé | Amortissement |

Avant tout enregistrement, le système **vérifie l'équilibre** (|débits − crédits| ≤ 0,01 $) et refuse l'écriture sinon. En multijuridiction (États-Unis), la ligne de deuxième taxe (TVQ) n'est **pas** insérée quand le tenant n'a pas de deuxième taxe.

### 4.3 Types de journal et préfixes de numéro

| Type | Préfixe du numéro |
|------|-------------------|
| Ventes | `JE-VTE-…` |
| Achats | `JE-ACH-…` |
| Banque | `JE-BNQ-…` |
| Encaissement | `JE-ENC-…` |
| Paie | `JE-PAI-…` |
| Stock | `JE-STK-…` |
| Opérations diverses | `JE-OD-…` |
| Amortissement | `JE-AMO-…` |
| Général | `JE-GEN-…` |
| Contre-passation | `JE-CP-…` |

> Les types sont stockés au **pluriel** (« Ventes », « Achats »). L'affichage reconnaît aussi les anciennes formes au singulier, par souci de compatibilité.

### 4.4 Plan comptable québécois (35 comptes préchargés — extraits)

| Code | Nom | Type |
|------|-----|------|
| 1010 | Encaisse | Actif |
| 1100 | Clients | Actif |
| 1150 | Retenues à recevoir | Actif |
| 1200 | TPS à recevoir (CTI) | Actif |
| 1210 | TVQ à recevoir (RTI) | Actif |
| 1500 / 1510 | Immobilisations / Amortissement cumulé | Actif |
| 2100 | Fournisseurs | Passif |
| 2150 | Retenues sur sous-traitants à payer | Passif |
| 2200 | TPS à payer | Passif |
| 2210 | TVQ à payer | Passif |
| 2300 | Salaires à payer | Passif |
| 2310 | Retenues à la source | Passif |
| 2320 | Charges sociales (CNESST, etc.) | Passif |
| 4100 | Revenus de construction | Revenu |
| 5100 – 5500 | Coûts directs (matériaux, main-d'œuvre, sous-traitance, équipement, chantier) | Charge |
| 6100 – 6900 | Frais (administration, loyer, amortissements) | Charge |

Le plan est préchargé selon le pays : **Québec 35 comptes**, **États-Unis 36** (avec un compte d'impôt d'État), **Canada standard 35**. Le préchargement est **idempotent** (il ne double jamais les comptes). Vous pouvez ajouter vos propres comptes ; le **code** (4 à 10 chiffres) est immuable après création, et un compte déjà utilisé dans le grand livre ne peut plus être **reclassé** (type, classe, solde normal).

### 4.5 Principaux points d'accès (API)

Préfixe commun : `/api/erp/v1/accounting`. « comptable » = garde `require_tenant_admin_or_role('comptable')`.

| Méthode et chemin | Rôle | Rôle métier |
|-------------------|------|-------------|
| `GET /invoices` · `GET /invoices/{id}` | authentifié | Lister / ouvrir les factures |
| `POST /invoices` | authentifié | Créer une facture (client ou fournisseur) |
| `PUT /invoices/{id}` | authentifié | Modifier (comptabilise en quittant le brouillon ; contre-passe si annulée ou ramenée en brouillon) |
| `DELETE /invoices/{id}` | **comptable** | Supprimer (brouillon ou annulée seulement) |
| `POST /invoices/{id}/payment` | **comptable** | Enregistrer un paiement |
| `POST /invoices/{id}/credit-note` | **comptable** | Émettre une note de crédit |
| `POST /invoices/{id}/send` · `/generate-html` · `/pdf` | authentifié | Envoyer / aperçu / PDF |
| `POST /invoices/ai/scan` | authentifié | Scan IA d'une facture fournisseur |
| `GET /invoices/public/{token}` | **public** | Consultation sans compte (lien 90 jours) |
| `GET/POST /chart-of-accounts` · `PUT/DELETE /chart-of-accounts/{id}` | lecture : authentifié ; écriture : **comptable** | Plan comptable |
| `POST /journal` · `/with-lines` · `/{id}/lines` · `PUT /{id}/validate` | **comptable** | Écritures de journal |
| `GET /journal` · `/ledger` · `/trial-balance` | authentifié | Consultations |
| `GET /balance-sheet` · `/income-statement` · `/cash-flow` · `/tax-declaration` | authentifié | États financiers |
| `POST /cedule/from-devis/{id}` · `GET /cedule` · `POST /decomptes` | authentifié | Décomptes CCDC |
| `GET /wip` | **comptable** | Travaux en cours |
| `GET /holdbacks` · `POST /holdbacks` · `PUT /holdbacks/{id}/release` | lecture : authentifié ; écriture : **comptable** | Retenues |
| `GET/POST /periods` · `PUT /{id}/close` · `/{id}/reopen` | lecture/création : authentifié ; clôture/réouverture : **comptable** | Périodes |
| `GET/POST /fixed-assets` · `POST /generate-depreciation` | lecture : authentifié ; amortissement : **comptable** | Immobilisations |
| `POST /sync-all` · `/sync-factures` · `/sync-depenses` | authentifié | Synchronisation rétroactive |
| `POST /ai/chat` · `/ai/confirm-action` | authentifié (+ crédits IA) | Assistant comptable |
| `POST /cron/daily` | **jeton de tâche** (pas de session) | Maintenance quotidienne automatique |

### 4.6 Calculs à connaître

- **Taxes de facture** : Total TTC = HT + TPS (5 %) + TVQ (9,975 %) au Québec. Les taux réels sont ceux **figés** sur la facture.
- **Retenue de garantie** : Montant retenu = **HT × taux** (par défaut 10 %).
- **Décompte** : Ce décompte = (% complété × valeur au contrat) − déjà facturé, uniquement si positif.
- **Travaux en cours (WIP)** : % d'achèvement = coûts encourus / coût estimé ; Revenu reconnu = min(%, 100 %) × valeur du contrat ; Écart = Facturé − Revenu reconnu (positif = surfacturation, négatif = sous-facturation). Tout est en **HT**.
- **Amortissement** : linéaire = (coût − valeur résiduelle) / durée en mois ; dégressif = valeur nette comptable × taux / 12.
- **Coût de l'IA** : coût réel des jetons **× 1,30** (majoration de 30 %), débité des crédits prépayés du tenant.

### 4.7 Limites connues

- Lignes de facture **en lecture seule** dans le détail (modification par la fenêtre Modifier).
- Passage à **Payée / Partiellement payée** impossible à la main (uniquement par un paiement).
- **Suppression** de facture limitée aux statuts Brouillon et Annulée.
- **Récurrence** : pas de bouton de création directe (on part d'une facture existante).
- **Cédule de décompte** : la fenêtre de poste n'expose que la description et la valeur.
- **Flux de trésorerie** : pas de filtre de dates propre.
- **WIP** : coût estimé signalé « à vérifier / indisponible » quand la source n'est pas fiable.
- Le **lien public** de facture est limité en débit par une protection de fréquence (efficace par instance de serveur).

---

## 5. Intégrations et FAQ

### 5.1 La paie et son lien avec la comptabilité (module 13 — Pointage)

La **paie québécoise complète** n'est **pas** dans ce module : elle se trouve dans le menu **Pointage** (`/pointage`, page « Pointage & Paie »), onglets **Paie** et **Paie CCQ**. Son moteur (`payroll.py`, garde `require_payroll_access`) calcule, pour chaque employé et chaque période :

- le **salaire brut** (salaire fixe ou heures régulières + heures supplémentaires × 1,5) et les **vacances** (4 % par défaut au Québec) ;
- l'**impôt** progressif fédéral et provincial ;
- les cotisations **RRQ** (6,30 %) et **RRQ2** (4 % sur la tranche supérieure), **RQAP** (0,430 % employé / 0,602 % employeur), **AE** (1,30 % / 1,82 %) ;
- les **charges de l'employeur** : **CNESST** (1,54 % par défaut, **à configurer par entreprise**), **FSS** (1,65 %) et **CCQ** (taux de financement + taux horaire selon la convention).

Les périodes se comptent 52 fois l'an (hebdomadaire), 26 (aux deux semaines) ou 12 (mensuel). Quand on génère une paie, le système la produit en brouillon, verrouille la période contre les doublons et recalcule les cumuls annuels.

**Le pont avec la comptabilité** : une paie (un « payroll run ») génère automatiquement une **écriture de journal en brouillon** (validation manuelle exigée, sur avis de la comptable) : **débit 6100** (salaires + charges, ventilé par projet), **crédit 2300** (net à payer), **crédit 2310** (retenues à la source), **crédit 2320** (charges de l'employeur). Vous la retrouvez dans l'onglet **Journal** de la Comptabilité, où vous la **validez**.

> **Taux à valider.** Les taux CNESST et CCQ sont des **réglages** (des estimations par défaut), pas des taux légaux figés. Configurez-les pour votre entreprise. Les barèmes de l'année sont marqués « à valider ».

### 5.2 Les feuillets fiscaux T4 / RL-1 / PD7A (module 13 — Pointage)

Les feuillets de fin d'année sont eux aussi dans le menu **Pointage**, onglet **Feuillets**. Ils sont alimentés par les **cumuls annuels** de la paie :

- **T4** (fédéral, ARC) : cases 14 (revenus), 16/17 (RRQ), 18 (AE), 22 (impôt), 55 (RQAP), etc.
- **RL-1** (Québec, Revenu Québec) : cases A (revenus), B (RRQ), C (AE), E (impôt), G, H, I.
- **PD7A** : aide au calcul de la **remise mensuelle des retenues à la source** (bloc ARC = impôt fédéral + AE ; bloc Revenu Québec = impôt provincial + RRQ + RQAP + FSS + CNESST).

Le **numéro d'assurance sociale** est déchiffré de façon **auditée** (conformité Loi 25) chaque fois qu'un feuillet est produit.

> **Mise en garde importante.** Le **RL-1 est préliminaire et non officiel** (certification de Revenu Québec non obtenue ; le PDF porte un filigrane). Le **T4** et le **PD7A** sont des **aides** : leurs fichiers XML et bordereaux ne sont **pas** garantis conformes aux gabarits officiels de transmission. Par ailleurs, la deuxième cotisation **RRQ2 n'est pas encore suivie par fiche de paie**, elle vaut donc 0 dans la remise PD7A. En clair : **ces feuillets ne remplacent pas une paie certifiée transmissible.** Faites-les valider par votre comptable.

### 5.3 Le fonds de prévoyance (Loi 16) — volet Immobilier / Terrain

Le **fonds de prévoyance** est un outil de gestion de **copropriété** (étude du fonds de prévoyance selon la **Loi 16** du Québec). Il **n'a aucun écran dans `/comptabilite`** : il vit dans le **volet Immobilier / Terrain** (vitrine `/immo`, back-office promoteur). Son moteur (`fonds_prevoyance.py`) permet de :

- décrire une **copropriété** et ses **composantes de bâtiment** (enveloppe, structure, systèmes mécaniques, aménagements) ;
- lancer une **étude** et générer des **projections sur 25 ans** avec **trois scénarios** de contribution (uniforme, progressif +3 %/an, variable avec marge de sécurité), en tenant compte des remplacements cycliques et de l'inflation ;
- tenir un **carnet d'entretien** et produire des **attestations** de vente ;
- calculer une **valeur de reconstruction** (taux Québec 2025 : Économique 250 $, Base 325 $, Moyenne 387 $, Haut de gamme 487 $ le pied carré) ;
- s'appuyer sur des analyses **IA** (score de santé, conformité Loi 16, suggestion de contribution).

Toutes ces fonctions sont ouvertes à tout utilisateur authentifié du tenant (elles n'exigent pas le rôle comptable). Pour l'utiliser, reportez-vous au **Module 19 — Immobilier**.

### 5.4 Devis, dossiers et CRM

- À la création d'une facture client, un **projet** peut être associé ; la facture hérite alors des **taxes du devis** si un devis est lié.
- Les **décomptes** importent leur cédule de valeurs **depuis un devis accepté** (le poste vaut le **prix de vente** du devis).
- Une facture liée à un dossier d'opportunité s'y **rattache automatiquement**, et apparaît dans la fiche 360 du dossier.

### 5.5 Bons de commande (module 14 — Magasin)

Une facture fournisseur peut être **créée depuis un bon de commande** reçu. La **synchronisation comptable** (`sync-depenses`) génère des écritures d'achat pour les bons transmis, et des écritures de salaire à partir des heures d'employés. Voir le **Module 14 — Bons de commande**.

### 5.6 Portail client B2B et Stripe

- Les factures **ne sont pas** affichées dans le portail client B2B ; pour transmettre une facture, utilisez le **courriel** (avec le lien public de 90 jours) ou le **PDF**.
- Il n'y a **pas** de paiement de facture par Stripe dans ce module. Stripe sert à l'abonnement du tenant et aux crédits IA, pas au règlement des factures clients.

### 5.7 Maintenance quotidienne automatique

Une tâche planifiée (`POST /accounting/cron/daily`, protégée par un **jeton de tâche** et non par une session utilisateur) passe chaque nuit chez tous les tenants actifs pour : (1) faire basculer en **En retard** les factures échues encore impayées ; (2) **générer** les factures récurrentes dues ; (3) envoyer les **rappels automatiques**. C'est pourquoi une facture peut passer « En retard » toute seule.

### 5.8 Assistant IA : ce qu'il peut et ne peut pas faire

- Il **propose**, vous **confirmez**. Aucune écriture ni facture n'est créée sans votre clic **Confirmer**.
- Il ne lit que des **tables comptables** (liste blanche stricte) : il **ne voit pas** la paie, les salaires ni les données RH.
- La création d'une **écriture** par l'IA exige, au moment de la confirmation, un rôle **comptable** ou **administrateur**. La création d'une **facture** par l'IA suit la même règle que la création manuelle (ouverte à tout utilisateur authentifié).
- Chaque échange consomme des **crédits IA** (coût réel majoré de 30 %).

### 5.9 Foire aux questions

**Où est la paie ? Je ne la trouve pas dans Comptabilité.**
Elle est dans le menu **Pointage** (`/pointage`), onglets **Paie** et **Paie CCQ**. Voir le Module 13. La comptabilité ne fait que **recevoir** l'écriture de paie (en brouillon) et vous laisser la valider.

**Et les T4 / RL-1 ?**
Aussi dans **Pointage**, onglet **Feuillets**. Attention : le RL-1 est **non officiel** et les fichiers ne sont **pas** garantis transmissibles (voir §5.2).

**Où est le fonds de prévoyance annoncé dans le titre ?**
Dans le volet **Immobilier / Terrain** (Module 19), pas ici. Aucun onglet de `/comptabilite` n'y donne accès.

**Pourquoi ma facture n'a-t-elle pas d'écriture au grand livre ?**
Parce qu'elle est encore en **Brouillon**. L'écriture est générée quand la facture **quitte le brouillon** (Envoyée). Cliquez **Envoyer**.

**Puis-je remettre une facture comptabilisée en Brouillon ?**
Oui, et dans ce cas le système **contre-passe** automatiquement l'écriture de vente et la délie de la facture. C'est un comportement récent, sûr sur le plan comptable.

**Comment corriger une facture déjà payée ?**
On ne la modifie pas directement. Émettez une **note de crédit** (avoir) pour neutraliser tout ou partie, puis refacturez au besoin.

**Comment annuler une écriture validée ?**
Impossible de l'effacer. Passez une **écriture d'ajustement** inverse (contre-passation), ou laissez le système le faire via l'annulation d'une facture.

**Les écritures en Brouillon apparaissent-elles dans les états financiers ?**
Non. Le bilan, les résultats, le grand livre et la déclaration de taxes ne comptent que les écritures **Validées**.

**Je suis propriétaire mais mon rôle est « utilisateur ». Suis-je bloqué des actions comptables ?**
Non. La garde reconnaît le drapeau **administrateur** (`is_admin`), infalsifiable et relu côté serveur. Un propriétaire administrateur a accès aux opérations comptables même si son rôle nominal est « utilisateur ».

**Mon écran est en lecture seule et je ne peux rien enregistrer. Pourquoi ?**
Le tenant est probablement en **mode consultation** (abonnement absent ou annulé). Régularisez l'abonnement pour retrouver l'écriture.

**Puis-je facturer un client hors Québec (TVH, taxe américaine) ?**
Oui : les taxes sont **multijuridiction**. Une facture fige ses propres libellés et taux ; pour les entreprises américaines, l'onglet **Tax USA** et la logique de taxe unique s'appliquent (pas de deuxième taxe).

**L'export QuickBooks fonctionne-t-il avec QuickBooks en ligne ?**
L'export **IIF** vise QuickBooks Desktop. Pour QuickBooks en ligne, utilisez le **CSV QuickBooks** par facture, ou le connecteur OAuth du module Intégrations.

**Le décompte crée une facture : dois-je faire quelque chose de plus ?**
La facture de décompte naît en **Brouillon**. Envoyez-la comme une facture ordinaire pour la comptabiliser (la retenue est déjà calculée et déduite du net à payer).

### 5.10 Ce qui n'existe pas (limites)

- Pas de paie, pas de bulletins de salaire, pas de feuillets fiscaux **dans ce module** (ils sont dans Pointage).
- Pas de fonds de prévoyance **dans ce module** (volet Immobilier).
- Pas de paiement de facture par Stripe, pas d'affichage des factures dans le portail B2B.
- Pas de modification de ligne dans le détail (fenêtre Modifier), pas de passage manuel à « Payée ».
- Pas de suppression d'une facture autre que Brouillon ou Annulée, pas d'effacement d'une écriture validée.
- Pas de deuxième taxe pour un tenant à taxe unique (États-Unis).

---

## 6. Récapitulatif

- Le module **Comptabilité** (`/comptabilite`, icône calculatrice) est le **grand livre et la facturation** de l'entreprise. Onglet ouvert par défaut : **Vue Globale**.
- **Malgré le titre, la paie, les feuillets et le fonds de prévoyance ne sont PAS ici** : la **paie** et les **feuillets** (T4, RL-1, PD7A) sont dans **Pointage** (Module 13) ; le **fonds de prévoyance** (Loi 16) est dans **Immobilier** (Module 19). La comptabilité **reçoit** l'écriture de paie (en brouillon) mais ne calcule aucun salaire.
- **18 onglets** (plus **Tax USA** pour les entreprises américaines), en six groupes : Aperçu, Facturation, Construction, Grand livre, Rapports, Gestion, plus l'Assistant IA. L'onglet **Conditions** n'est visible que des administrateurs.
- **Factures** : création client/fournisseur, **scan IA**, envoi par courriel (lien public 90 jours), **paiements** avant/après avoirs, **notes de crédit** (art. 350 LTVQ), **récurrence** (depuis une facture existante), **rappels** à quatre niveaux, PDF / Excel / CSV QuickBooks.
- **Grand livre en partie double** : plan comptable (35 comptes au Québec), écritures **Brouillon → Validée** (figées), grand livre, transactions, et **états financiers** (Bilan, Résultats, Flux, Taxes) qui ne comptent que les écritures validées.
- **Facturation de construction** : **décomptes progressifs CCDC** (cédule de valeurs importée d'un devis), **travaux en cours** (WIP, coût à coût, en lecture seule), **retenues de garantie** (1150 clients, 2150 sous-traitants, base HT).
- **Comptabilisation automatique** : une facture qui quitte le brouillon génère son écriture ; l'annulation ou le retour en brouillon **contre-passe** automatiquement (jamais d'effacement d'écriture validée).
- **Périodes** : la clôture **bloque effectivement** toute écriture datée dans la période.
- **Permissions** : consultation ouverte à tous ; opérations sensibles (supprimer, encaisser, créditer, valider une écriture, gérer les comptes, libérer une retenue, amortir, clôturer) réservées au rôle **comptable** ou à un **administrateur**. Le drapeau `is_admin` prime sur le rôle nominal. **Mode consultation** (lecture seule) si l'abonnement est absent ou annulé.
- **Effet argent** : aucun paiement Stripe direct ici ; les seuls coûts facturés au tenant sont les **crédits IA** (scan de facture et assistant comptable, coût réel majoré de 30 %).
- **Exports** : QuickBooks IIF, Sage 50 CSV, plan comptable / journal / grand livre / balance / déclaration de taxes en CSV, factures en PDF / Excel / CSV.

---

**Documentation générée à partir du code source** : `frontend/src/pages/ComptabilitePage.tsx` (18 onglets), sous-composants `ComptesPayablesTab.tsx`, `DecomptesTab.tsx`, `WipTab.tsx`, `RetenuesTab.tsx`, `ComptaAssistantTab.tsx`, `api/accounting.ts` ; `backend/routers/accounting.py` (93 points d'accès), `accounting_ai.py` (assistant), `payroll.py` / `feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` / `feuillets_common.py` (module Paie), `fonds_prevoyance.py` (volet Immobilier).

**Manuels liés** :
- Module 13 — Pointage et heures (paie CCQ, feuillets T4 / RL-1 / PD7A) — `13-operations-pointage.md`
- Module 14 — Bons de commande (factures fournisseurs, comptabilisation des achats) — `14-operations-bons-de-commande.md`
- Module 08 — Soumissions (devis à l'origine des décomptes) — `08-ventes-soumissions.md`
- Module 09 — Projets (rattachement des factures et des travaux en cours) — `09-ventes-projets.md`
- Module 07 — Dossiers / Fiche 360 (ouverture directe d'une facture) — `07-ventes-dossiers.md`
- Module 19 — Immobilier (fonds de prévoyance, Loi 16) — `19-terrain-immobilier.md`
- Module 25 — Assistant IA (crédits IA du scan et de l'assistant comptable) — `25-communication-assistant-ia.md`
- Module 28 — Configuration (taxes, thème des documents, coordonnées bancaires) — `28-configuration.md`
