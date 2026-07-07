# Module 09 — Projets de construction

> **Version** : 3.0 (refonte vérifiée ligne à ligne contre le code source du 2026-07-07)
> **Code de référence** : `backend/routers/projects.py` (2037 lignes, 20 endpoints), `backend/routers/projets_ai.py` (460 lignes, 2 endpoints), `frontend/src/pages/ProjectsPage.tsx` (1249 lignes), `frontend/src/components/projets/ProjetsAssistantTab.tsx` (231 lignes), `frontend/src/api/projects.ts`, `frontend/src/api/projetsAi.ts`
> **Endpoints connexes** : `backend/routers/production.py:2810` (génération de cédule), `backend/routers/devis.py:11884` (conversion automatique d'un devis accepté en projet)
> **Tables PostgreSQL** : `projects`, `project_phases`, `project_notes`, `project_assignments`, `dossier_projets` (association) ; agrégats en lecture seule depuis `devis`, `factures`, `bons_commande`, `time_entries`, `companies`, `contacts`
> **Route et menu** : `/projets` — libellé « Projets » (icône `Briefcase`), section « Gestion » de la barre latérale
> **Cadrage** : ce module gère la **fiche projet** (le dossier maître d'un chantier) : liste, création, édition, duplication, suppression, statistiques, notes catégorisées par IA, synthèse financière des coûts réels et passerelles vers le Dossier 360, la cédule (Bon de travail) et la soumission liée. Il **ne fait pas** l'ordonnancement Gantt interactif (celui-ci vit dans le module **Suivi**), ni la gestion des phases en écriture depuis l'écran, ni la promotion immobilière (Module 11).

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

Le module **Projets de construction** est le dossier maître de chaque chantier. Il permet à un entrepreneur ou à un employé de :

- Tenir une **liste paginée** de tous les projets, avec recherche et filtres par statut et priorité.
- **Créer**, **modifier**, **dupliquer** et **supprimer** un projet.
- Suivre quatre **indicateurs clés (KPI)** : nombre total de projets, projets en cours, projets terminés, budget total cumulé.
- Consulter un **panneau de détail** riche : métadonnées du chantier, phases (en lecture seule), soumission liée, synthèse financière des coûts réels, notes.
- **Catégoriser une note par IA** (Claude) selon une grille propre à la construction.
- **Générer une cédule** (un Bon de travail pré-rempli d'opérations standard) en un clic.
- **Naviguer** vers le Dossier 360 lié.
- **Exporter** la liste complète au format CSV.
- Appliquer une **mise à jour en lot** (statut ou priorité) sur plusieurs projets cochés.
- Dialoguer avec un **assistant IA dédié** qui lit les données réelles et propose la création d'un projet sur confirmation.

Un projet peut naître de deux façons : **manuellement** (bouton « Nouveau projet » ou assistant IA), ou **automatiquement** lorsqu'un devis passe au statut « Accepté » (voir §5.2). Dans les deux cas, l'écran Projets sert ensuite à piloter le dossier.

### 1.2 Accès et prérequis

| Prérequis | Détail |
|---|---|
| **Menu** | Barre latérale → section **Gestion** → **Projets** (icône `Briefcase`). Route `/projets`. |
| **Authentification** | Session ouverte. Chaque appel passe par `Depends(get_current_user)`. |
| **Contexte tenant** | L'utilisateur doit être rattaché à une entreprise (`user.schema`). Sinon : `400 Contexte tenant manquant`. |
| **Catégorisation IA et assistant IA** | Service IA disponible, garde-fou IA franchi et **crédits IA prépayés suffisants**. Sinon : `503` (service indisponible), `403` (accès IA refusé) ou `402` (crédits épuisés). |

### 1.3 Rôles et permissions

Point important à connaître : **le module Projets n'impose aucun contrôle de rôle**. Il n'y a pas de garde `require_role` ni `require_tenant_admin` sur les endpoints de `projects.py`. Tout utilisateur authentifié du tenant — y compris un compte au rôle « user » — peut créer, modifier, dupliquer et supprimer des projets et des notes.

Deux nuances encadrent tout de même les écritures :

1. **Mode consultation (lecture seule)** — Un tenant dont l'abonnement est suspendu ou annulé bascule en mode consultation. Ce filtre est appliqué en amont, dans `get_current_user`, et bloque toutes les écritures avec un `403` avant même d'atteindre le module.
2. **Génération de cédule** — Le bouton « Générer la cédule » appelle un endpoint qui, lui, **est protégé** par `require_tenant_admin_or_role(*BT_WRITE_ROLES)` (il vit dans le module Production/Bons de travail). Un utilisateur sans droit d'écriture sur les Bons de travail recevra un refus sur cette action précise.

### 1.4 Sous-modules et écrans

| Écran / composant | Contenu |
|---|---|
| **Liste des projets** (3 modes : Liste, Tableau, Cartes) | Colonne gauche : recherche, filtres, tri, édition en ligne, sélection en lot. |
| **Panneau de détail** | Colonne droite (~45 %) : métadonnées, phases, soumission liée, finances, notes, boutons Dossier 360 et Cédule. |
| **Modales** | Nouveau projet, Modifier le projet, Ajouter une note, Assistant IA. |
| **Assistant IA — Projets** | Chat en langage naturel (lecture des données + proposition de création sur confirmation). |

---

## 2. Interface

### 2.1 Disposition générale

```
+----------------------------------------------------------------------+
|  Projets                                                             |
+----------------------------------------------------------------------+
|  [ Total ]  [ En cours ]  [ Terminés ]  [ Budget total ]   4 KPI     |
+----------------------------------------------------------------------+
|  [ Liste ]  [ Tableau ]  [ Cartes ]        Sélecteur de mode         |
+----------------------------------------------------------------------+
|  (si au moins un projet coché)                                       |
|  N projet(s) sélectionné(s)  [ Changer le statut... ] [ Désélect. ]  |
+----------------------------------------------------------------------+
|  [ + Nouveau projet ]  [ Assistant IA ]  [ Exporter CSV ]            |
|                                  [ Rechercher... ]  [ Statut : Tous ] |
+----------------------------------------------------------------------+
|  ZONE LISTE (gauche, 100 % ou ~55 % si un détail est ouvert)         |
|  Numéro | Nom | Client | Budget | Statut | Priorité | Début | Fin    |
+----------------------------------------------------------------------+
|  ZONE DÉTAIL (droite, ~45 %, si un projet est sélectionné)           |
|  Nom + badges + [ Dupliquer ] [ Modifier ] [ Fermer ]                |
|  [ Voir le Dossier 360 ]  [ Générer la cédule ]                      |
|  Client | Budget | Description | Adresse | Dates                     |
|  Phases (N)   ·   Soumission NUMÉRO (si devis lié)                   |
|  Finances [ Afficher / Masquer ]  ·  Notes (N) [ + Ajouter ]         |
+----------------------------------------------------------------------+
```

### 2.2 En-tête de page (toujours visible)

**Titre** : « Projets ».

**Quatre cartes KPI** (alimentées par `GET /projects/statistics`) :

| Carte | Contenu | Couleur |
|---|---|---|
| **Total** | Nombre total de projets du tenant. | Neutre |
| **En cours** | Projets au statut « En cours ». | Bleu |
| **Terminés** | Projets au statut « Terminé ». | Vert |
| **Budget total** | Somme des budgets (`budget_total`), formatée en dollars. | Neutre |

**Sélecteur de mode d'affichage** : trois boutons — **Liste** / **Tableau** / **Cartes**.

**Barre de commande** :

| Bouton | Icône | Effet |
|---|---|---|
| **Nouveau projet** | `Plus` | Ouvre la modale de création. |
| **Assistant IA** | `Sparkles` | Ouvre la modale de l'assistant IA. |
| **Exporter CSV** | `Download` | Télécharge le fichier `projets_export.csv`. |
| **Rechercher…** (champ) | — | Filtre sur le nom, la description et le numéro de projet. |
| **Statut** (menu déroulant) | — | Filtre : Tous / En attente / En cours / Terminé / Annulé / Suspendu. |

**Bandeaux de rétroaction** : une alerte rouge en cas d'erreur ; une alerte verte de succès, effacée automatiquement après 4 secondes.

### 2.3 Barre d'actions en lot (conditionnelle)

Elle apparaît dès qu'au moins un projet est coché :

- Compteur « N projet(s) sélectionné(s) ».
- Menu déroulant **« Changer le statut… »** (En attente / En cours / Terminé / Annulé / Suspendu) → applique le nouveau statut à toute la sélection après une **confirmation**.
- Bouton **« Désélectionner »**.

### 2.4 Les trois modes d'affichage de la liste

#### 2.4.1 Mode Liste (par défaut)

Tableau dont les colonnes sont **triables** et **redimensionnables** :

| Colonne | Note |
|---|---|
| Case à cocher | Sélection individuelle ou tout / rien. |
| **Numéro** | `numero_projet` (format `PROJ-AAAA-NNNNN`), en bleu. |
| **Nom** | Nom du projet. |
| **Client** | Nom résolu du client (entreprise, contact ou saisie manuelle). |
| **Budget** | Aligné à droite. |
| **Statut** | Badge coloré — **édition en ligne** (un clic ouvre un menu déroulant). |
| **Priorité** | Badge coloré — **édition en ligne**. |
| **Début prévu** | Date — **édition en ligne**. |
| **Date Fin** | Date — **édition en ligne**. |
| Action | Bouton **Supprimer** (icône `Trash2`, avec confirmation). |

Un clic sur une ligne (hors cellule éditable) ouvre le panneau de détail. La ligne sélectionnée est surlignée. Sur téléphone, la liste se transforme en cartes compactes (nom, badge de statut, client, budget, date de création, bouton supprimer). Message si vide : « Aucun projet ».

#### 2.4.2 Mode Tableau (compact)

Tableau plus dense, avec des colonnes supplémentaires : case à cocher, **ID**, Nom, Client, **Type**, Budget, Statut, Priorité, Début, Fin, **Ville**. Les dates y restent éditables en ligne.

#### 2.4.3 Mode Cartes

Grille de cartes : case à cocher, badges de statut et de priorité, nom, client, budget, date de fin, ville (icône `MapPin`), bouton supprimer.

**Pagination** : 20 projets par page, en bas de la liste, dans les trois modes.

### 2.5 Panneau de détail (colonne droite)

Il s'ouvre au clic sur un projet. À l'ouverture, l'application charge en parallèle le projet, ses notes, le dossier lié et la soumission liée. Sur téléphone, un bouton « Retour à la liste » remplace la vue de détail.

**En-tête du panneau** : nom du projet, suivi de trois boutons :

| Bouton | Icône | Effet |
|---|---|---|
| **Dupliquer** | `Copy` | Crée une copie du projet (protection contre le double-clic). |
| **Modifier** | `Pencil` | Ouvre la modale d'édition. |
| **Fermer** | `X` | Referme le panneau. |

**Badges** : statut et priorité.

**Boutons contextuels** :

- **« Voir le Dossier 360 »** (bleu, icône `FolderOpen`) — visible **seulement** si un dossier est lié. Navigue vers `/dossier/{id}`.
- **« Générer la cédule »** (vert, icône `CalendarClock`) — crée un Bon de travail pré-rempli d'opérations standard selon le type de projet. L'opération est **idempotente** : si une cédule automatique existe déjà, le bouton la retourne sans en créer une deuxième. Un état « Génération… » s'affiche pendant le traitement.

**Métadonnées** : Client, Budget, Description, Adresse et Ville du chantier (icône `MapPin`), Début, Fin.

**Section Phases** — **lecture seule**. Pour chaque phase : son nom et sa progression en pourcentage. Titre « Phases (N) ». Il n'y a **aucun** bouton d'ajout, d'édition ou de suppression de phase dans cet écran (voir §5.3).

**Section Soumission liée** — **lecture seule**, affichée si un devis est rattaché au projet. Elle reprend les lignes du devis avec une **majoration recalculée côté client** : administration 3 %, contingences 12 %, profit 15 % par défaut (chaque ligne peut surcharger ces pourcentages). Elle affiche le sous-total, les taxes (TPS / TVQ) et le total avec taxes. Titre « Soumission {numéro} (N lignes) ». L'édition du devis se fait dans le module **Soumissions**, pas ici.

**Section Finances** — repliable (bouton Afficher / Masquer). Elle appelle `GET /projects/{id}/financials` et présente les **coûts réels calculés** (voir §4.4) :

- Quatre cartes : **Revenus** (vert), **Dépenses** (rouge), **Marge** (bleu ou orange, avec le pourcentage), **Budget** (gris, si supérieur à 0).
- Détail des revenus : devis acceptés (liste informative), factures client (liste, montant encaissé, badge de statut).
- Détail des dépenses : matériaux (bons de commande, icône `Package`), main-d'œuvre (heures × coût, icône `Users`), factures fournisseur.
- État vide : « Aucune donnée financière pour ce projet ».

**Section Notes** — titre « Notes (N) » et bouton **« Ajouter »**. Pour chaque note : titre, badge de catégorie (si présent), pourcentage d'importance (si présent), contenu (deux lignes), date, et bouton **« Catégoriser IA »** (icône `Bot`). L'analyse affiche un état « Analyse… ». État vide : « Aucune note ».

**Ligne de bas** : « Créé le {date} ».

### 2.6 Modales

#### 2.6.1 Nouveau projet (grande taille, deux colonnes)

| Champ | Obligatoire | Note |
|---|---|---|
| **Nom du projet** | Oui | Rejeté si vide ou composé uniquement d'espaces. |
| **No. PO Client** | Non | Numéro de bon de commande du client. |
| **Client (Entreprise)** | Non | Menu déroulant alimenté par le CRM (100 entreprises maximum). |
| **Client (Personne)** | Non | Menu déroulant des contacts. |
| **Saisie manuelle** | Non | Nom de client libre si le client n'est pas dans le CRM. |
| **Type de projet (modèle de cédule)** | Non | Menu déroulant de 5 types (voir §4.3). Pilote le gabarit d'opérations de la cédule. |
| **Statut** | Non | Par défaut « En attente ». |
| **Priorité** | Non | Par défaut « Moyenne ». |
| **Début prévu des travaux** | Non | Date. |
| **Fin prévue des travaux** | Non | Date. |
| **Budget ($)** | Non | Nombre (pas de saisie négative). |
| **Adresse chantier** | Non | |
| **Ville chantier** | Non | |
| **Description** | Non | Zone de texte. |

Une note rappelle que le champ marqué d'un astérisque est obligatoire. Le bouton **Créer** reste désactivé tant que le nom est vide. **Tous ces champs sont enregistrés** (y compris le PO client, le contact et la saisie manuelle — voir §4.6).

#### 2.6.2 Ajouter une note

Trois champs : **Titre** (obligatoire), **Contenu** (obligatoire, zone de texte) et **Catégorie** (facultative, exemple : « Technique, Sécurité, Budget… »). Boutons Annuler / Ajouter.

#### 2.6.3 Modifier le projet (taille moyenne)

Champs : Nom (obligatoire), Description, Type de projet, Statut, Priorité, Date de début, Date de fin, Budget, Adresse chantier, Ville chantier. Boutons Annuler / **Enregistrer**. Vider le champ Budget efface la valeur (envoie `null`). Une alerte s'affiche en cas d'erreur d'édition.

#### 2.6.4 Assistant IA

Ouvre le composant de chat `ProjetsAssistantTab` (voir §2.7).

### 2.7 Assistant IA — Projets

L'assistant suit un modèle **propose → confirme** : il peut lire les données et proposer la création d'un projet, mais il **n'écrit jamais** directement en base. C'est l'utilisateur qui confirme.

**En-tête** : « Assistant IA — Projets » et un sous-titre. L'écran d'accueil propose trois exemples de questions :

- « Quels projets sont en cours et pour quel budget total ? »
- « Crée un projet … »
- « Quels projets se terminent ce mois-ci ? »

**Capacités de la version 1** :

1. **Lecture** — L'outil `recherche_bd` interroge une liste blanche stricte de trois tables : `projects`, `companies`, `contacts`. Une garde bloque toute table sensible (employés, paie, salaires, NAS, utilisateurs, crédits IA, etc.). Maximum 50 lignes par requête.
2. **Action** — Un seul type d'action : la **création de projet**. L'IA appelle l'outil `proposer_projet`, ce qui affiche une **carte de proposition** (icône `Briefcase`) montrant les champs proposés. L'utilisateur clique **Confirmer** (bleu, icône `CheckCircle`) ou **Annuler** (icône `X`). Seul le bouton Confirmer déclenche réellement la création. Un badge « En attente de confirmation » accompagne la carte.

> **La modification et la suppression par l'IA ne sont pas implémentées** (fonctionnalité prévue mais non livrée). L'assistant ne sait que lire et proposer une création.

**Zone de saisie** : une zone de texte (Entrée = envoyer, Maj+Entrée = nouvelle ligne) et un bouton **Envoyer**. Chaque bulle de message affiche des métadonnées (profil, jetons, coût en dollars, durée). Des verrous synchrones empêchent le double-envoi et la double-confirmation.

---

## 3. Workflows pas à pas

### 3.1 Créer un projet manuellement

1. Cliquer sur **« Nouveau projet »**.
2. Saisir le **Nom** (obligatoire).
3. Choisir le client : soit **Client (Entreprise)**, soit **Client (Personne)**, soit **Saisie manuelle** si le client n'est pas au CRM. Ces trois champs sont enregistrés.
4. Renseigner les champs souhaités : No. PO Client, Type de projet, Statut, Priorité, dates, Budget, Adresse, Ville, Description.
5. Cliquer sur **Créer**.

Le projet reçoit un numéro au format `PROJ-AAAA-NNNNN` et, par défaut, le statut « En attente » et un type vide.

### 3.2 Rechercher et filtrer

- **Recherche** : tape un terme dans le champ « Rechercher… ». Le filtre porte sur le **nom**, la **description** et le **numéro de projet** (insensible à la casse).
- **Filtre Statut** : Tous, En attente, En cours, Terminé, Annulé, Suspendu.

### 3.3 Changer de mode d'affichage

Cliquer sur **Liste**, **Tableau** ou **Cartes** dans le sélecteur de l'en-tête.

### 3.4 Ouvrir un projet

Cliquer sur une ligne (ou une carte) pour ouvrir le panneau de détail. Le projet, ses notes, son dossier lié et sa soumission liée se chargent en parallèle.

### 3.5 Éditer en ligne (statut, priorité, dates)

- **Statut ou priorité** : dans la vue Liste, cliquer sur le badge. Un menu déroulant apparaît. Le changement est appliqué immédiatement (mise à jour optimiste, avec retour en arrière automatique en cas d'échec et garde contre les doubles envois).
- **Dates** : cliquer sur la cellule de date (Début ou Fin), choisir une date. L'enregistrement est immédiat. Vider la date l'efface (écrit `null`).

### 3.6 Modifier un projet

Cliquer sur **Modifier** (crayon) dans le panneau de détail, ajuster les champs, puis **Enregistrer**. Contrairement à l'ancienne version, tous les champs de cette modale sont bien persistés (le type de projet est modifiable).

### 3.7 Dupliquer un projet

Cliquer sur **Dupliquer** (icône `Copy`). Le nouveau projet est nommé « Copie de … », reçoit un nouveau numéro et le statut forcé « En attente ». La duplication **ne recopie pas** les phases, les notes ni les assignations. Un message confirme la création avec le nouvel identifiant.

### 3.8 Supprimer un projet

Cliquer sur **Supprimer** (icône `Trash2`) puis confirmer. La suppression détruit en cascade les données propres au projet (phases, notes, opérations, matériaux, journaux de chantier, etc.) et **détache** (met à `NULL`, sans les détruire) les documents comptables et de vente : écritures de journal, dépenses, devis, factures, bons de commande, feuilles de temps, dossiers, courriels.

> **Garde** : un projet dont le statut commence par « termin » (Terminé et ses variantes) **ne peut pas être supprimé** (`400`). Il faut d'abord le repasser dans un autre statut.

### 3.9 Mettre à jour plusieurs projets en lot

1. Cocher les projets voulus. La barre d'actions en lot apparaît.
2. Choisir un statut dans **« Changer le statut… »**.
3. Confirmer. Le nouveau statut est appliqué à toute la sélection.

### 3.10 Exporter la liste en CSV

Cliquer sur **« Exporter CSV »**. Le fichier `projets_export.csv` (jusqu'à 20 000 lignes) est téléchargé. Ses colonnes : ID, Numéro Projet, Nom Projet, Statut, Priorité, Type, Client, Date Début, Date Fin, Budget Total, Description, Adresse Chantier, Ville Chantier, Créé le, Modifié le. Les cellules texte sont protégées contre l'injection de formule. Il n'y a **pas** d'export PDF ni d'impression dans ce module.

### 3.11 Générer une cédule (Bon de travail)

Dans le panneau de détail, cliquer sur **« Générer la cédule »**. Le système crée un Bon de travail pré-rempli d'opérations standard, choisies selon le **type de projet** (résidentiel, rénovation, commercial, institutionnel, public). Utile surtout pour les projets créés directement, sans passer par un devis. L'action est idempotente (une seule cédule automatique par projet). Cette action requiert un droit d'écriture sur les Bons de travail.

### 3.12 Naviguer vers le Dossier 360

Si un dossier est lié, cliquer sur **« Voir le Dossier 360 »** pour ouvrir `/dossier/{id}` : documents, communications, extras et vue à 360 degrés du chantier.

### 3.13 Consulter la synthèse financière

Ouvrir la **section Finances** du panneau (bouton Afficher). Les quatre cartes (Revenus, Dépenses, Marge, Budget) et leurs détails se calculent à la volée à partir des factures, bons de commande et feuilles de temps rattachés au projet.

### 3.14 Ajouter une note

Section Notes → **« Ajouter »** → saisir Titre, Contenu et, au besoin, une Catégorie → **Ajouter**.

### 3.15 Catégoriser une note par IA

Cliquer sur **« Catégoriser IA »** (icône `Bot`) sous une note. Claude (Opus 4.8) classe la note dans l'une des dix catégories construction : Technique, Sécurité, Budget, Planning, Qualité, Communication, Environnement, RH, Approvisionnement, Autre. Il attribue aussi un niveau d'importance. Le coût est déduit des crédits IA prépayés du tenant.

### 3.16 Utiliser l'assistant IA

1. Cliquer sur **« Assistant IA »** dans la barre de commande.
2. Poser une question sur les projets (avancement, budgets, échéances) ou demander la création d'un projet.
3. Pour une lecture, l'assistant répond directement à partir des données réelles.
4. Pour une création, l'assistant affiche une **carte de proposition**. Vérifier les champs, puis cliquer sur **Confirmer** (le projet est créé) ou **Annuler**.

### 3.17 Ouvrir un projet depuis un lien externe

Un lien de la forme `/projets?open={id}` (par exemple depuis le calendrier) ouvre automatiquement le panneau de détail du projet concerné.

---

## 4. Référence

### 4.1 Endpoints (22 au total)

Préfixe réel des chemins : `/api/erp/v1`. Le CRUD est sous `/projects` (anglais), l'assistant sous `/projets/ai` (français) — incohérence de nommage assumée dans le code.

**CRUD et lecture — `routers/projects.py`**

| Méthode | Chemin | Fonction (ligne) | Rôle |
|---|---|---|---|
| GET | `/projects` | list_projects (431) | Liste paginée + filtres (recherche nom/description/numéro, statut, priorité, entreprise). |
| GET | `/projects/statistics` | get_project_statistics (515) | KPI : total, en cours, terminés, budget total. |
| POST | `/projects/duplicate/{id}` | duplicate_project (564) | Duplique un projet (statut forcé « En attente »). |
| GET | `/projects/export-csv` | export_projects_csv (674) | Export CSV (20 000 lignes maximum). |
| POST | `/projects/batch-update` | batch_update_projects (757) | Mise à jour en lot du statut ou de la priorité (1 à 1000 projets). |
| GET | `/projects/gantt` | get_gantt_data (800) | Données Gantt (projets + phases). **Consommé par le module Suivi.** |
| GET | `/projects/{id}` | get_project (882) | Détail : projet + client + phases + assignations. |
| GET | `/projects/{id}/financials` | get_project_financials (983) | Synthèse financière des coûts réels. |
| POST | `/projects` | create_project (1266) | Créer un projet. |
| PUT | `/projects/{id}` | update_project (1384) | Modifier un projet (partiel). |
| GET | `/projects/{id}/dossier` | get_project_dossier (1445) | Dossier 360 lié. |
| POST | `/projects/{id}/phases` | create_phase (1489) | Créer une phase (via API seulement). |
| PUT | `/projects/{id}/phases/{phaseId}` | update_phase (1538) | Modifier une phase (via API seulement). |
| GET | `/projects/{id}/notes` | list_project_notes (1582) | Lister les notes. |
| POST | `/projects/{id}/notes` | create_project_note (1623) | Créer une note. |
| POST | `/projects/{id}/notes/{noteId}/categorize` | categorize_project_note (1668) | Catégoriser une note par IA (facturé). |
| GET | `/projects/{id}/assignments` | list_project_assignments (1794) | Lister les assignations d'employés (via API seulement). |
| POST | `/projects/{id}/assignments` | add_project_assignment (1834) | Assigner un employé (via API seulement). |
| DELETE | `/projects/{id}/assignments/{assignmentId}` | remove_project_assignment (1883) | Retirer une assignation (via API seulement). |
| DELETE | `/projects/{id}` | delete_project (1921) | Supprimer un projet (garde « terminé »). |

**Assistant IA — `routers/projets_ai.py`**

| Méthode | Chemin | Fonction (ligne) | Rôle |
|---|---|---|---|
| POST | `/projets/ai/chat` | projets_ai_chat (289) | Chat IA : lecture + proposition de projet (n'écrit rien). Facturé. |
| POST | `/projets/ai/confirm-action` | confirm_projets_action (428) | Exécute une proposition confirmée → délègue à `create_project`. |

**Endpoint connexe (hors module, appelé par le bouton « Générer la cédule »)**

| Méthode | Chemin | Fichier | Rôle |
|---|---|---|---|
| POST | `/production/projects/{id}/generate-cedule` | production.py:2810 | Crée un Bon de travail idempotent depuis un gabarit d'opérations. Protégé par rôle d'écriture BT. |

> **Ce qui n'existe pas** : aucun endpoint DELETE pour une phase (seulement POST et PUT). Aucun endpoint PUT ni DELETE pour une note (seulement GET, POST et la catégorisation). Une phase créée ou une note ajoutée ne peut donc pas être supprimée par l'interface ni par l'API (elles ne disparaissent qu'à la suppression du projet, qui les efface en cascade).

### 4.2 Statuts et priorités

**Statuts** (valeur canonique en base : `En attente`, `En cours`, `Termine`, `Annule`, `Suspendu`) :

| Affichage | Couleur du badge |
|---|---|
| En attente | Jaune |
| En cours | Bleu |
| Terminé | Vert |
| Suspendu | Ambre |
| Annulé | Rouge |

**Priorités** : Basse, Moyenne, Haute, Urgente. Le système tolère aussi d'anciennes valeurs héritées d'un devis (NORMAL, URGENT, CRITIQUE) et les ramène vers la forme canonique.

> **Normalisation tolérante** : à l'écriture (création, édition, lot), les variantes de casse, d'accents ou d'ancien format (par exemple `EN_ATTENTE`) sont converties vers la forme canonique. Il n'existe **aucune machine à états** : on peut passer librement d'un statut à l'autre. La seule règle liée au statut est l'interdiction de supprimer un projet terminé.

### 4.3 Types de projet et modèle de cédule

Le type de projet pilote le gabarit d'opérations utilisé par « Générer la cédule » :

| Type (menu) | Effet |
|---|---|
| *(vide)* | Résidentiel par défaut. |
| Résidentiel — Construction neuve | Gabarit résidentiel neuf. |
| Résidentiel — Rénovation | Gabarit rénovation. |
| Commercial — Construction neuve | Gabarit commercial neuf. |
| Commercial — Rénovation | Gabarit commercial rénovation. |
| Institutionnel | Gabarit institutionnel. |
| Public | Gabarit public. |

### 4.4 Calculs

**Statistiques (KPI)** — `GET /projects/statistics` regroupe par statut :
- `total` = somme de tous les projets ;
- `en_cours` = nombre au statut exact « En cours » ;
- `termines` = nombre au statut exact « Termine » ;
- `budget_total` = somme des `budget_total`.

> Nuance : ce regroupement se fait sur le statut **brut** (non normalisé rétroactivement). Un très ancien projet resté dans un format non canonique en base formerait sa propre tranche et ne serait pas compté dans « en cours » ou « terminés ».

**Synthèse financière (coûts réels)** — `GET /projects/{id}/financials`. Le budget est un champ **stocké** ; les coûts réels ne sont **pas** stockés, ils sont recalculés à chaque appel :

| Bloc | Source | Règle |
|---|---|---|
| Budget | `projects.budget_total` | Retourné tel quel. Aucun calcul d'écart budget / réel. |
| Revenus — devis | Devis acceptés liés | Informatif seulement (non compté dans le total des revenus). |
| Revenus — factures | Factures **client** (hors Annulée et Brouillon) | Base **HT** (hors taxes). Une note de crédit (avoir) est soustraite. |
| Dépenses — matériaux | Bons de commande (hors annulé et brouillon) | Exclus s'ils sont déjà facturés par le fournisseur (anti double-comptage). |
| Dépenses — main-d'œuvre | Feuilles de temps pointées | Heures × (taux horaire, ou salaire ÷ 2080). |
| Dépenses — fournisseur | Factures fournisseur (inclut les brouillons) | Base HT, avoir soustrait. |
| **Total revenus** | = total des factures client. | Les devis n'y sont pas comptés. |
| **Total dépenses** | = matériaux + main-d'œuvre + factures fournisseur. | |
| **Marge** | = revenus − dépenses. | |
| **Marge %** | = marge ÷ revenus × 100 (0 si revenus ≤ 0). | |

**Progression Gantt** (module Suivi) — la progression d'un projet est **dérivée** : c'est la moyenne des progressions de ses phases (arrondie à une décimale), ou 0 s'il n'a aucune phase. Elle n'est jamais stockée sur le projet.

**Majoration de la soumission liée** (panneau de détail, affichage) — multiplicateur = 1 + (administration + contingences + profit) ÷ 100, avec 3 %, 12 % et 15 % par défaut, surchargeables ligne par ligne. Ce calcul est un simple affichage : il ne touche pas au devis.

### 4.5 Limites et bornes

| Élément | Limite |
|---|---|
| Pagination | 1 à 100 par page (défaut 20). |
| Recherche | Sous-chaîne insensible à la casse sur nom + description + numéro. |
| Gantt | 500 projets, exclut les projets annulés. |
| Export CSV | 20 000 lignes. |
| Mise à jour en lot | 1 à 1000 projets par appel. |
| Nom du projet | 300 caractères, non vide. |
| Description | 20 000 caractères. |
| Budget | 0 à 999 999 999 999,99 (borne anti-débordement). |
| Note — titre / contenu | 300 / 20 000 caractères. |
| Message de l'assistant IA | 1 à 8000 caractères. |
| Débit de l'assistant IA | 20 requêtes/minute (chat), 30/minute (confirmation), par adresse IP. |

### 4.6 Modèles IA et coûts

| Surface | Modèle | Jetons max | Tarif (avant marge) | Marge |
|---|---|---|---|---|
| Catégorisation de note | `claude-opus-4-8` | 32 000 | 5 $ / 25 $ par million (entrée / sortie) | +30 % |
| Assistant IA (chat) | `claude-sonnet-4-6` | 8 000 | 3 $ / 15 $ par million | +30 % |

Les deux surfaces sont facturées aux **crédits prépayés** du tenant, protégées par le garde-fou IA et la vérification de solde (`402` si épuisé). Chaque usage est journalisé.

### 4.7 Persistance des champs à la création

| Champ de la modale | Enregistré ? |
|---|---|
| Nom du projet | Oui |
| No. PO Client | **Oui** |
| Client (Entreprise) | Oui |
| Client (Personne) | **Oui** |
| Saisie manuelle | **Oui** (fige le nom du client) |
| Type de projet | Oui |
| Statut, Priorité | Oui |
| Début, Fin, Budget | Oui |
| Adresse, Ville, Description | Oui |

> **Correction d'une information périmée** : les anciennes versions de ce manuel affirmaient que le No. PO Client, le Client (Personne) et la Saisie manuelle étaient « silencieusement ignorés ». C'est **faux** dans le code actuel : `create_project` insère bien `po_client`, `client_contact_id` et `client_nom_direct`, et résout le nom du client à figer. De même, la modale Modifier n'a plus de champ « Gestionnaire » ou « Notes » : tous ses champs sont persistés.

### 4.8 Codes d'erreur

| Code | Cause |
|---|---|
| 400 | Contexte tenant manquant / nom vide / suppression d'un projet terminé. |
| 402 | Crédits IA épuisés. |
| 403 | Accès IA refusé, ou écriture bloquée en mode consultation. |
| 404 | Projet, phase, note ou assignation introuvable. |
| 409 | Employé déjà assigné (assignation en double). |
| 500 | Erreur interne. |
| 503 | Service IA non disponible ou temporairement surchargé. |

---

## 5. Intégrations et FAQ

### 5.1 Cartographie des intégrations

| Module lié | Nature du lien |
|---|---|
| **CRM (Entreprises / Contacts)** | Alimente les listes Client (Entreprise) et Client (Personne) à la création. Le nom du client est figé (`client_nom_cache`) puis résolu à l'affichage. |
| **Soumissions (Devis)** | Deux liens. **Entrant** : l'acceptation d'un devis crée automatiquement un projet (voir §5.2). **Sortant** : le panneau de détail affiche la soumission liée en lecture seule. |
| **Dossier 360** | Table d'association `dossier_projets` → bouton « Voir le Dossier 360 ». |
| **Bons de travail / Production** | Bouton « Générer la cédule » → Bon de travail pré-rempli d'opérations selon le type de projet. |
| **Comptabilité** | La section Finances agrège les factures client, les bons de commande, les feuilles de temps et les factures fournisseur pour calculer les coûts réels et la marge. |
| **Suivi** | Consomme `GET /projects/gantt` pour afficher le Gantt et la progression. Le Gantt **n'est pas** rendu dans l'écran Projets. |
| **Calendrier** | Un lien `?open={id}` ouvre directement un projet. |

### 5.2 Conversion automatique d'un devis en projet

La création d'un projet à partir d'un devis se déclenche **du côté Devis**, pas depuis cet écran (il n'y a aucun bouton « Convertir » dans le module Projets). Trois chemins déclenchent le helper `_create_project_from_devis` (devis.py:11884) :

1. Un devis passe au statut « Accepté » (création automatique).
2. L'endpoint explicite `POST /devis/{id}/convert-to-project` (idempotent).
3. L'acceptation publique d'un devis par le client, via son lien signé (sans authentification).

Particularités d'un projet issu d'un devis, qui le distinguent d'une création manuelle :

- Le **budget** est le montant d'investissement total (avec taxes) du devis, recalculé si nécessaire.
- Le **statut** est forcé à « En cours » (et non « En attente ») ; le **type** par défaut est « Construction ».
- Le projet est relié à l'opportunité CRM gagnée, les pièces jointes du devis sont recopiées, et un Bon de travail est généré au passage (au mieux, sans jamais bloquer la conversion).

### 5.3 Ce qui n'est pas disponible dans cet écran

- **Pas d'onglets Kanban / Gantt / Calendrier / Statistiques** : l'écran propose trois **modes d'affichage** (Liste / Tableau / Cartes), pas des onglets de planification. Le Gantt interactif vit dans le module **Suivi**.
- **Gestion des phases en écriture** : les phases sont affichées en lecture seule. Les capacités de création et d'édition existent dans l'API, mais aucun bouton ne les expose ici, et la création d'un projet ne génère aucune phase automatiquement.
- **Gestion des assignations d'employés** : l'API sait lister, ajouter et retirer des assignations, mais aucun élément de l'écran Projets ne les utilise.
- **Pas d'export PDF, pas d'impression, pas de téléversement de fichiers** : les documents passent par le Dossier 360 lié.
- **L'assistant IA ne modifie ni ne supprime** : il ne fait que lire et proposer une création.
- **La soumission liée est en lecture seule** : elle s'édite dans le module Soumissions.

### 5.4 FAQ

**Q : Où se trouve le module dans le menu ?**
R : Barre latérale, section **Gestion**, entrée **Projets** (icône `Briefcase`). Ce n'est pas dans la section « Outils ».

**Q : Comment un projet est-il créé automatiquement ?**
R : En acceptant un devis. Le projet naît alors avec le statut « En cours » et le budget du devis. Voir §5.2.

**Q : Le numéro de projet, d'où vient-il ?**
R : Il est généré au format `PROJ-AAAA-NNNNN` à partir de l'identifiant interne et de l'année. Les anciens projets sans numéro sont complétés automatiquement à la première consultation.

**Q : Puis-je supprimer une phase ou une note ?**
R : Non. Il n'existe aucun endpoint de suppression pour les phases ni pour les notes. Elles ne sont effacées qu'à la suppression complète du projet (cascade). Une phase peut cependant être modifiée par l'API (`PUT`).

**Q : Pourquoi ne puis-je pas supprimer un projet « Terminé » ?**
R : Une garde métier l'interdit (retour `400`). Repassez d'abord le projet dans un autre statut.

**Q : Les devis acceptés comptent-ils dans les revenus de la section Finances ?**
R : Non. Le total des revenus provient uniquement des factures client (base hors taxes). Les devis acceptés sont affichés à titre informatif.

**Q : Pourquoi la marge se calcule-t-elle sur des montants hors taxes ?**
R : Pour comparer du hors taxes à du hors taxes. Les taxes perçues (TPS, TVQ) ne sont pas un revenu ; les inclure gonflerait la marge d'environ 13 %.

**Q : La section Finances me montre-t-elle l'écart entre mon budget et mes coûts réels ?**
R : Non. Elle affiche le budget et les coûts réels côte à côte, mais ne calcule aucun pourcentage d'avancement budgétaire ni écart automatique.

**Q : Qui peut créer, modifier ou supprimer un projet ?**
R : Tout utilisateur authentifié du tenant, quel que soit son rôle. Seule exception : le mode consultation (abonnement suspendu) bloque les écritures, et la génération de cédule exige un droit d'écriture sur les Bons de travail.

**Q : Quel est le format du fichier CSV exporté ?**
R : `projets_export.csv`, encodage UTF-8, séparateur virgule, 15 colonnes. La colonne « Notes » qui dupliquait la description dans l'ancienne version a été retirée.

**Q : Combien coûte la catégorisation d'une note par IA ?**
R : Elle utilise Claude Opus 4.8 et se facture aux crédits prépayés selon les jetons consommés (tarif 5 $ / 25 $ par million, plus une marge de 30 %). En pratique, une classification produit très peu de texte, donc le coût par note est minime.

**Q : L'assistant IA peut-il modifier un projet existant ?**
R : Non. La version 1 se limite à la lecture et à la proposition de création. La modification par l'IA est prévue mais non implémentée.

---

## 6. Récapitulatif

- **Mission** : gérer la fiche projet (dossier maître du chantier) — liste, création, édition, duplication, suppression, KPI, notes catégorisées par IA, synthèse financière des coûts réels, passerelles Dossier 360 / Cédule / Soumission.
- **Accès** : barre latérale → **Gestion** → **Projets** (route `/projets`).
- **Code source** : `projects.py` (2037 lignes, 20 endpoints), `projets_ai.py` (460 lignes, 2 endpoints), `ProjectsPage.tsx` (1249 lignes), `ProjetsAssistantTab.tsx` (231 lignes).
- **Trois modes d'affichage** : Liste (édition en ligne du statut, de la priorité et des dates), Tableau (compact), Cartes. Pas d'onglets Kanban / Gantt / Calendrier.
- **Statuts** : En attente / En cours / Terminé / Suspendu / Annulé. **Priorités** : Basse / Moyenne / Haute / Urgente. Normalisation tolérante des anciennes valeurs.
- **Deux origines d'un projet** : création manuelle (statut « En attente ») ou conversion automatique d'un devis accepté (statut forcé « En cours », budget du devis).
- **Finances** : coûts réels calculés à la volée (factures client HT, bons de commande, main-d'œuvre, factures fournisseur) ; budget stocké ; aucun calcul d'écart budget / réel.
- **IA** : catégorisation de note (Opus 4.8) et assistant chat (Sonnet 4.6), facturés aux crédits prépayés, modèle propose → confirme, création de projet seulement.
- **Permissions** : aucun contrôle de rôle sur le module (tout utilisateur du tenant peut tout faire), sauf le mode consultation et le droit d'écriture BT pour la cédule.
- **Ce qui n'existe pas** : suppression de phase ou de note ; édition des phases et des assignations depuis l'écran ; export PDF ou impression ; modification par l'IA ; onglets de planification (le Gantt vit dans le module Suivi).

---

*Sources vérifiées le 2026-07-07 : `ERP_REACT/backend/routers/projects.py`, `ERP_REACT/backend/routers/projets_ai.py`, `ERP_REACT/backend/routers/production.py` (génération de cédule), `ERP_REACT/backend/routers/devis.py` (conversion devis → projet), `ERP_REACT/frontend/src/pages/ProjectsPage.tsx`, `ERP_REACT/frontend/src/components/projets/ProjetsAssistantTab.tsx`, `ERP_REACT/frontend/src/api/projects.ts`, `ERP_REACT/frontend/src/api/projetsAi.ts`.*
*Manuels liés : Module 07 — Soumissions (devis) · Module 08 — Suivi (Gantt / Kanban) · Module 10 — Bons de travail · Module 11 — Immobilier · Module Comptabilité · Module Dossiers (Dossier 360).*
*Manuel ERP Constructo AI — Module 09 — Projets de construction — v3.0 — 2026-07-07*
