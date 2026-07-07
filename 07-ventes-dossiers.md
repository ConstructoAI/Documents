# Module 07 — Dossiers (gestion documentaire)

> **Version** : 3.0 (refonte complète vérifiée contre le code source 2026-07)
> **Route frontend** : `/dossiers` (menu latéral « Dossiers », groupe « Gestion », icône `FolderOpen`, entre « Ventes » et « Soumissions »). Détail : `/dossier/:id` (**au singulier**). Page publique de partage : `/dossiers/public/:token` (sans authentification).
> **Préfixe API** : `/api/erp/v1` — attention : le nom de la route est `/dossiers` côté écran, mais **tous les endpoints sont sous `/api/erp/v1/documents`** (voir 1.7).
> **Code de référence (backend)** : `backend/routers/documents.py` (6138 lignes, 58 routes — dossiers, pièces jointes, répertoires, notes, liens, étapes, extras, partage, endpoints publics) · `backend/routers/dossiers_ai.py` (327 lignes, 1 route — assistant Dossiers **en lecture seule**) · `backend/routers/dossier_ai.py` (772 lignes, 2 routes — assistant **Extras/avenants** proposer→confirmer).
> **Code de référence (frontend)** : `frontend/src/pages/DossiersPage.tsx` (428 lignes — liste) · `frontend/src/pages/DossierDetailPage.tsx` (3911 lignes — Fiche 360, 13 onglets) · `frontend/src/pages/DossierPublicPage.tsx` (561 lignes — page publique) · `frontend/src/components/dossiers/DossiersAssistantTab.tsx` (123 lignes) · `frontend/src/components/dossiers/ExtrasAssistant.tsx` (321 lignes).
> **Clients API frontend** : `api/documents.ts` (949 lignes), `api/dossiersAi.ts` (35 lignes). Note : **`api/dossiers.ts` n'existe pas** — toutes les pages importent depuis `@/api/documents`.
> **Tables PostgreSQL (par locataire)** : `dossiers` (en-tête), `attachments` (fichiers stockés en `BYTEA`), `dossier_folders` (arborescence de répertoires), `dossier_notes`, `dossier_liens`, `dossier_etapes`, `dossier_extras` (avenants), et 5 tables de liaison `dossier_devis` / `dossier_projets` / `dossier_formulaires` / `dossier_achats` / `dossier_factures`. Facturation d'un extra : `factures` / `facture_lignes`. Table **partagée** : `public.dossiers_public_tokens` (jetons de partage, schéma `public`). Tables partagées (chemin IA) : `public.ai_prepaid_credits`, `public.ai_usage_tracking`.
> **Cadrage** : malgré son nom « gestion documentaire », ce module est en réalité une **Fiche 360° de projet**. Un dossier est un **carrefour** qui rassemble, autour d'un même chantier ou d'un même client, l'opportunité, les soumissions, le projet, les bons de travail, les achats, les demandes de prix, les factures, le pointage, la comptabilité (marge), une **arborescence de documents**, des **notes** (avec IA), des **liens** cliquables et des **extras/avenants** facturables. Chaque ligne de la liste porte d'ailleurs la mention « Fiche 360 ». Il ne remplace ni le module Soumissions (qui chiffre les devis), ni le module Comptabilité (qui émet les factures) : il les **relie** et donne une vue d'ensemble.

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

### 1.1 Mission

Le module Dossiers sert à **regrouper au même endroit tout ce qui touche un chantier ou un client**. Au lieu de chercher un devis dans un module, une facture dans un autre et les photos dans un troisième, vous ouvrez **un seul dossier** et vous voyez :

- les **soumissions**, le **projet**, les **bons de travail**, les **achats**, les **demandes de prix** et les **factures** rattachés ;
- le **pointage** des employés et la **comptabilité** du chantier (revenus, coûts, marge estimée) ;
- une **arborescence de répertoires et de documents** (plans, photos, contrats, correspondance) ;
- des **notes** de chantier, avec des outils d'IA pour les enrichir, analyser une photo ou résumer l'ensemble ;
- des **liens** externes utiles (permis en ligne, dossier infonuagique du client, etc.) ;
- les **extras (avenants)** au contrat, que vous pouvez suivre et **facturer** en un clic.

Le dossier est aussi votre **outil de partage** : vous pouvez générer un lien public sécurisé pour qu'un client ou un sous-traitant consulte (et, si vous l'autorisez, téléverse) des documents, sans avoir de compte dans l'ERP.

### 1.2 Accès par le menu latéral

Cliquez sur **Dossiers** dans le menu latéral (groupe « Gestion »). La page s'ouvre sur la **liste** des dossiers (`/dossiers`).

Deux liens rapides sont utiles :

- `app.constructoai.ca/dossiers?open=<id>` ouvre **directement** la Fiche 360 d'un dossier. C'est ce lien qu'utilisent les boutons « Voir le dossier » ailleurs dans l'ERP (module Ventes/CRM, par exemple). Le paramètre `open` est consommé puis retiré de l'adresse (`DossiersPage.tsx:142-152`).
- `app.constructoai.ca/dossiers/public/<token>` est la **page publique** partagée avec un tiers (voir 2.13).

> **Nuance d'adresse** : la Fiche 360 vit à `/dossier/<id>` (**au singulier**), tandis que la liste et la page publique sont sous `/dossiers` (au pluriel). C'est voulu dans le code (`App.tsx:221-222`).

### 1.3 Les trois écrans

| Écran | Route | Authentification | Rôle |
|-------|-------|------------------|------|
| **Liste des dossiers** | `/dossiers` | Oui | Rechercher, filtrer, créer, supprimer, ouvrir l'assistant IA |
| **Fiche 360 (détail)** | `/dossier/:id` | Oui | Les 13 onglets, le cœur du module |
| **Page publique** | `/dossiers/public/:token` | **Non** | Consultation (et téléversement selon le niveau) de documents partagés |

### 1.4 Numérotation automatique

À la création, chaque dossier reçoit un numéro **`DOS-AAAA-NNNNN`** (ex. `DOS-2026-00042`) : l'année en cours suivie de l'identifiant du dossier sur 5 chiffres (`documents.py:638`). Le numéro est généré de façon **infaillible même en cas de clics simultanés** : le dossier est d'abord inséré avec un numéro provisoire `TEMP`, puis mis à jour avec son numéro définitif dans la même transaction (`documents.py:615-640`). On n'utilise jamais un `COUNT + 1` qui pourrait produire des doublons.

Les **extras** suivent le même principe avec le format **`EXT-XXXX`** (ex. `EXT-0001`, `documents.py:5834`). La **facture** produite en facturant un extra reçoit un numéro **`FACT-AAAA-NNNNN`** géré par le module Comptabilité.

### 1.5 Statuts, types et priorités

**5 statuts de dossier** (constante `DOSSIER_STATUTS`, `documents.py:57`). Le statut est **forcé à `OUVERT` à la création** ; il évolue ensuite via le menu déroulant en haut de la Fiche 360.

| Valeur (base) | Libellé affiché | Couleur du badge |
|---------------|-----------------|------------------|
| `OUVERT` | Ouvert | Bleu |
| `EN_COURS` | En cours | Vert |
| `EN_ATTENTE` | En attente | Jaune |
| `TERMINE` | Terminé | Sarcelle |
| `ARCHIVE` | Archivé | Gris |

Dans les indicateurs de la liste, « Ouverts » regroupe `OUVERT + EN_COURS + EN_ATTENTE`, et « Terminés » regroupe `TERMINE + ARCHIVE` (`documents.py:472-473`).

**5 types de dossier** (menu déroulant de la modale de création) : **Projet**, **Client**, **Fournisseur**, **Administratif**, **Autre**.

**4 priorités** (constante `PRIORITES`, `documents.py:62`) — valeurs en base `BASSE` / `NORMAL` / `HAUTE` / `URGENT`, libellés affichés **Basse** / **Normal** / **Haute** / **Urgent**.

| Priorité | Couleur du badge |
|----------|------------------|
| Urgent | Rouge |
| Haute | Orange |
| Normal | Bleu |
| Basse | Gris |

### 1.6 Permissions et rôles

Le module est **volontairement ouvert à toute l'équipe** : presque tous les endpoints exigent simplement un compte ERP valide du locataire (`Depends(get_current_user)`). Autrement dit, tout utilisateur authentifié peut créer un dossier, y attacher des documents, écrire des notes, associer des éléments et gérer les partages. L'isolation entre entreprises reste garantie côté serveur (chaque requête est bornée au schéma du locataire, et chaque dossier est revérifié comme appartenant bien à ce locataire — protection contre l'accès à un identifiant d'autrui).

**Une seule action est réservée** aux administrateurs et aux comptables : **facturer un extra**. L'endpoint `POST …/extras/{eid}/facturer` est protégé par `require_tenant_admin_or_role("comptable")` (`documents.py:6074`) — il faut être `is_admin` (relu côté serveur, infalsifiable), avoir le rôle `admin` ou `comptable`, ou être super-administrateur. La même règle s'applique à l'**auto-facturation** (passage d'un extra à « Approuvé ») et à la facturation demandée via l'assistant IA Extras.

| Action | Qui a le droit |
|--------|----------------|
| Consulter, créer, modifier, supprimer un dossier ; documents, notes, liens, étapes, associations, partages | Tout compte ERP valide du locataire |
| Créer / modifier / supprimer un **extra** | Tout compte ERP valide (mais la **facturation** est bloquée, voir ci-dessous) |
| **Facturer** un extra (ou approuver un extra qui déclenche l'auto-facture) | `admin`, `comptable`, `is_admin` ou super-administrateur |

**Mode consultation (lecture seule).** Si l'abonnement Stripe du locataire est en souffrance, le compte peut passer en mode **lecture seule** : les lectures fonctionnent, mais **toute écriture renvoie 403**. Ce contrôle est appliqué en amont, dans `get_current_user` (`erp_auth.py:520-528`), et couvre donc **tous** les endpoints authentifiés du module (création de dossier, téléversement, notes, extras, IA). Les **4 endpoints publics par jeton** ne passent pas par `get_current_user` : ils ne sont donc pas soumis au mode consultation, mais ils restent coupés si l'entreprise est désactivée (voir 2.13).

### 1.7 Pourquoi « Dossiers » à l'écran et « documents » dans l'adresse ?

C'est un héritage historique du code : le premier nom interne du module était « documents ». Le préfixe des endpoints est resté `/documents` (`documents.py:50` + `API_PREFIX`, donc `/api/erp/v1/documents/…`), alors que l'interface a été renommée « Dossiers » pour plus de clarté. Le seul endroit où le mot « dossiers » apparaît côté API est l'assistant en lecture seule (`/api/erp/v1/dossiers/ai/chat`). Aucun ancien fichier `dossier.py` n'existe : **le backend est bien `routers/documents.py`**.

### 1.8 Ce que le module ne fait pas (vérifié dans le code)

- **Pas d'onglet « Étapes »** dans la Fiche 360. Le backend sait pourtant gérer une liste d'étapes (`GET/POST …/etapes` + `…/toggle`), et l'API cliente existe, mais **aucun onglet ne l'affiche** dans la version actuelle. C'est une fonctionnalité dormante.
- **Pas de sélecteur de catégorie au téléversement** : les 10 catégories de documents (PLAN, PHOTO, CONTRAT…) existent côté serveur, mais l'interface de téléversement ne les propose pas. Un document importé reste sans catégorie explicite.
- **Pas de bouton « Archiver »** : « Archivé » est un **statut** (réglable par le menu déroulant de la Fiche 360), pas une action de la liste. La liste n'offre que Nouveau dossier, Actualiser, Assistant IA et Supprimer.
- **Pas de vue « Statistiques » séparée** : les 3 indicateurs (Total / Ouverts / Terminés) sont toujours affichés en haut de la liste ; il n'y a pas d'onglet à basculer.
- **Modale de création minimale** : seuls Nom, Type et Priorité sont demandés. La description, les notes et le statut ne sont pas exposés à la création (le statut est forcé à `OUVERT`).
- **Aucun export CSV/PDF de la liste, aucune impression, aucune action en lot.** Les seules sorties de fichiers sont le téléchargement d'une pièce jointe et le partage public par lien.
- **La facturation d'un extra produit une facture BROUILLON** (elle n'est pas envoyée automatiquement au client).
- **L'assistant IA Dossiers est en lecture seule** et **ne lit pas le contenu des fichiers joints** : il travaille sur les métadonnées (dossiers, notes, étapes, liens, extras, factures liées).

---

## 2. Interface

### 2.1 Écran Liste — `/dossiers`

**Barre de commande** (en haut) :

| Bouton | Icône | Effet |
|--------|-------|-------|
| **Nouveau dossier** | `Plus` (bleu) | Ouvre la modale de création (voir 2.2). |
| **Actualiser** | `RefreshCw` | Recharge la liste **et** les indicateurs. |
| **Assistant IA** | `Sparkles` | Ouvre une grande modale avec l'assistant Dossiers **en lecture seule** (voir 2.14). |

À droite de la barre : un **champ de recherche** (« Rechercher... ») et un **filtre de statut** (menu déroulant : Tous les statuts, Ouvert, En cours, En attente, Terminé, Archivé).

**Recherche et filtre.** La recherche est faite **côté serveur** avec un léger délai de 300 ms après la frappe (`DossiersPage.tsx:79-85`). Elle porte sur le **titre** et le **numéro** de dossier, sur **tout le locataire** (pas seulement la page affichée), et remet l'affichage à la page 1. Le filtre de statut fait de même.

**Indicateurs (toujours visibles quand ils sont chargés)** — source `GET /documents/statistics` :

| Indicateur | Signification |
|------------|---------------|
| **Total** | Nombre total de dossiers. |
| **Ouverts** (bleu) | Dossiers `OUVERT + EN_COURS + EN_ATTENTE`. |
| **Terminés** (vert) | Dossiers `TERMINE + ARCHIVE`. |

**Tableau (bureau)** — colonnes triables (clic sur l'en-tête) et redimensionnables :

| Colonne | Contenu |
|---------|---------|
| **Nom** | Icône dossier + titre + mention bleue « Fiche 360 ». |
| **Type** | Libellé traduit (Projet / Client / Fournisseur / Administratif / Autre). |
| **Statut** | Badge coloré. |
| **Priorité** | Badge coloré. |
| **Mis à jour** | Date de dernière modification. |
| **Actions** | Bouton **Supprimer** (corbeille). |

Cliquer n'importe où sur une ligne ouvre la Fiche 360 (`/dossier/:id`). Sur mobile, la même information est présentée en **cartes**. En bas : le total et la **pagination** (20 dossiers par page). Si la liste est vide : « Aucun dossier ».

**Suppression depuis la liste.** Le bouton corbeille demande une confirmation détaillée : la suppression efface les **pièces jointes, notes et étapes**, **détache** (sans les supprimer) les opportunités et projets liés, supprime les **dépenses en cascade**, et **est irréversible**. Un message de succès confirme « Dossier "…" supprimé ».

### 2.2 Modale « Nouveau dossier »

Trois champs seulement :

| Champ | Détail |
|-------|--------|
| **Nom** * | Obligatoire (texte libre, ex. « Rénovation cuisine Dupont »). |
| **Type** | Menu déroulant : Projet / Client / Fournisseur / Administratif / Autre. |
| **Priorité** | Menu déroulant : Basse / Normal / Haute / Urgent. |

Boutons **Annuler** / **Créer**. Un verrou empêche le double-envoi. En cas d'erreur, le message s'affiche **dans la modale**. Le statut n'est pas demandé (il vaut `OUVERT`). À la validation, le dossier `DOS-AAAA-NNNNN` est créé et vous êtes amené à sa Fiche 360.

### 2.3 Fiche 360 — en-tête

En haut de la Fiche 360 (`DossierDetailPage.tsx:216-308`) :

- Un bouton **retour** (flèche gauche).
- Le **titre**, **modifiable sur place** : cliquez sur le crayon, un champ apparaît (255 caractères max) avec les boutons valider/annuler ; **Entrée** enregistre, **Échap** annule.
- Un bouton **Supprimer le dossier** (corbeille), avec le même avertissement de cascade que la liste.
- Le **statut**, modifiable sur place par un menu déroulant des 5 valeurs, avec mise à jour immédiate à l'écran.
- Une ligne de métadonnées : le **numéro de dossier** (police à chasse fixe), un badge du **numéro d'opportunité** si le dossier provient du CRM, et le **nom du client**.

### 2.4 Fiche 360 — les 13 onglets

La barre de navigation compte **13 onglets** (`DossierDetailPage.tsx:62-76`). Certains affichent un **compteur** du nombre d'éléments liés.

| # | Onglet | Compteur ? | Ce qu'on y trouve |
|---|--------|-----------|-------------------|
| 1 | **Résumé** | — | Indicateurs financiers, opportunité liée, progression. |
| 2 | **Soumissions** | Oui | Devis rattachés. |
| 3 | **Projet** | Oui | Projet rattaché. |
| 4 | **Bons de travail** | Oui | BT rattachés. |
| 5 | **Achats** | Oui | Bons de commande rattachés. |
| 6 | **Demandes de prix** | Oui | Demandes de prix rattachées. |
| 7 | **Factures** | Oui | Factures rattachées. |
| 8 | **Pointage** | Oui | Heures et coûts par employé. |
| 9 | **Comptabilité** | — | Revenus, coûts et marge du dossier. |
| 10 | **Documents** | Oui | Arborescence de répertoires et de fichiers, partages. |
| 11 | **Notes** | — | Notes de chantier + outils IA. |
| 12 | **Liens** | — | Liens externes cliquables. |
| 13 | **Extras** | — | Avenants et leur facturation. |

> **Il n'y a pas d'onglet « Étapes »** malgré son existence côté serveur (voir 1.8). La **gestion des partages** n'est pas un onglet : elle est intégrée dans l'onglet **Documents** (boutons « Partager »).

### 2.5 Onglet Résumé

- **8 indicateurs** : Budget total, Facturé, Payé, Solde dû, Achats, Main-d'œuvre, Marge, et « BT / BC / DP » (compteurs de bons de travail, bons de commande et demandes de prix).
- Une **carte Opportunité** si le dossier provient du CRM : nom, client, montant, source, badge de statut.
- Une **carte Progression** : une frise en 5 étapes — Opportunité → Soumission → Projet → Travaux → Facturation — chaque étape se colorant en vert lorsqu'elle est atteinte.

### 2.6 Associer et retirer des éléments (onglets 2 à 7)

Les six onglets **Soumissions, Projet, Bons de travail, Achats, Demandes de prix, Factures** partagent un même mécanisme de rattachement (`LinkableSection`, `DossierDetailPage.tsx:459-581`). Chacun affiche, en plus de sa liste :

- un bouton **« + Associer {type} »** : il ouvre un menu déroulant des éléments **associables** (ceux qui existent dans le module concerné et ne sont pas déjà rattachés, chargés via `GET /documents/{id}/linkable`) ; le choix crée l'association ;
- une zone **« Retirer une association »** : des pastilles `#id` cliquables retirent le lien.

Retirer une association **ne supprime jamais l'élément lui-même** : le devis, le projet, la facture, etc. restent intacts ; seul le rattachement au dossier disparaît.

Le contenu de chaque onglet en **lecture** :

| Onglet | Colonnes / infos | Ouvre vers |
|--------|------------------|-----------|
| **Soumissions** | Numéro, statut, montant, date | `/devis?open=…` |
| **Projet** | Budget, dates prévue/réelle | `/projets?open=…` |
| **Bons de travail** | N°, nom, statut, priorité, montant, échéance | `/bons-travail?open=…` |
| **Achats** | N°, fournisseur, statut, montant, dates commande/livraison | `/magasin?open=…` |
| **Demandes de prix** | N°, nom, statut, priorité, montant, échéance | (numéro non cliquable) |
| **Factures** | N°, client, statut, montant TTC, payé, solde dû, date | `/comptabilite?open=…` |

Si un onglet n'a aucun élément, il affiche un message du type « Aucune soumission liée à ce dossier ».

### 2.7 Onglet Pointage

Deux tableaux (`PointageSection`) :

- **Sommaire par employé** : Employé, Heures, Coût, nombre d'entrées ;
- **Entrées récentes** (20 au maximum) : Employé, heure d'entrée, heure de sortie, Heures, Type.

### 2.8 Onglet Comptabilité

Vue financière agrégée du dossier, en deux cartes :

- **Revenus** : Budget total, Total facturé, Total payé, Solde à recevoir, Factures payées, Factures en retard ;
- **Coûts et marge** : Heures, Coût main-d'œuvre, Achats, Factures fournisseur, Total des coûts, **Marge estimée**, **% de marge**.

Ces chiffres sont calculés à la volée par la vue « 360 » du serveur (voir 4.9). Deux subtilités utiles : le coût de main-d'œuvre affiché est le **coût réel de la paie** dès qu'il existe (sinon une estimation à partir des taux horaires), et le calcul **évite les doubles comptages** (les achats déjà représentés par une facture fournisseur ne sont pas comptés deux fois ; les avoirs sont soustraits des revenus).

### 2.9 Onglet Documents (le plus riche)

C'est une véritable **arborescence de répertoires et de fichiers**, jusqu'à **5 niveaux de profondeur**.

**Barre d'en-tête** : un compteur « {n} documents » et trois boutons :

| Bouton | Icône | Effet |
|--------|-------|-------|
| **Nouveau répertoire / sous-répertoire** | `FolderPlus` | Crée un dossier ou sous-dossier (désactivé au 5ᵉ niveau). |
| **Partager** | `Share2` | Partage le **dossier entier** (voir 2.9.4). |
| **Ajouter un document** | `Upload` | Téléverse un ou plusieurs fichiers (jusqu'à **150 Mo** chacun). |

**Fil d'Ariane** : Racine → … permet de remonter dans l'arborescence.

**Glisser-déposer** : une zone de dépôt accepte des fichiers directement, avec une barre de progression (nom du fichier, « Fichier x/y », pourcentage).

**Affichage** : les **répertoires** apparaissent en premier, puis les **fichiers**.

#### 2.9.1 Actions sur un répertoire

Ouvrir · **Partager** (`Share2`) · **Renommer** (`Pencil`) · **Déplacer** (`FolderInput`) · **Copier** (`Copy`) · **Supprimer** (`Trash2`).

À la **suppression d'un répertoire**, une modale demande la stratégie :

- **« Remonter le contenu au parent »** : les sous-répertoires et documents sont rattachés au répertoire parent (ils ne sont pas perdus) ;
- **« Tout supprimer »** : le sous-arbre entier et ses fichiers sont effacés définitivement.

#### 2.9.2 Actions sur un fichier

Pour les fichiers que vous avez téléversés (source « pièces jointes ») : cliquer sur le nom ouvre l'**aperçu** ; les boutons offrent **Aperçu** (`Eye`), **Télécharger**, **Renommer** (`Pencil`, l'extension d'origine est conservée), **Déplacer** (`FolderInput`), **Copier** (`Copy`) et **Supprimer** (`Trash2`). Les documents provenant d'autres sources (par exemple d'anciennes tables) apparaissent en lecture seule (nom seulement).

L'**aperçu** s'ouvre dans une visionneuse intégrée pour les types pris en charge (images, PDF, texte).

#### 2.9.3 Déplacer / copier (y compris vers un autre dossier)

Une modale unifiée gère le déplacement et la copie. Elle permet de choisir **un autre dossier de destination** (recherche côté serveur, avec délai de frappe) puis un **répertoire cible** dans une arborescence indentée. Le sous-arbre que vous déplacez est exclu des destinations possibles (impossible de déplacer un répertoire dans lui-même). Un avertissement précise qu'un **déplacement vers un autre dossier révoque les partages** du répertoire concerné.

#### 2.9.4 Partager le dossier entier

Le bouton **Partager** (en-tête) génère un lien public de la forme `…/dossiers/public/{token}`. La modale affiche :

- des **statistiques** : nombre de consultations et de téléchargements, et les dernières dates ;
- les dates **Créé** et **Expire** (le lien du dossier entier expire au bout de **90 jours**) ;
- deux boutons : **Régénérer** (fait tourner le jeton — l'ancien lien cesse aussitôt de fonctionner) et **Révoquer** (désactive le partage).

Un lien de dossier entier donne accès, en lecture, à **tous** les documents du dossier (mode « liste à plat »).

#### 2.9.5 Partager un seul répertoire (partage ciblé)

Le bouton **Partager** d'un **répertoire** ouvre une modale plus fine, avec deux réglages :

- le **niveau d'accès** : **Lecteur seul** (`reader` — consultation en ligne, téléchargement refusé), **Lecture + téléchargement** (`downloader`), ou **Contributeur** (`contributor` — consultation, téléchargement **et** téléversement par l'invité) ;
- l'**expiration** : **30 jours**, **90 jours** ou **Jamais** (par défaut 90 jours).

La modale affiche ensuite le lien, ses statistiques, son niveau, sa date d'expiration, et permet de le révoquer. Un partage de répertoire ne donne accès qu'au **sous-arbre** de ce répertoire.

### 2.10 Onglet Notes

**Formulaire d'ajout** : une zone de texte, avec des outils :

- **Enrichir avec IA** (`Sparkles`) : réécrit et structure la note ;
- **Analyser photo IA** (`Image`) : téléverser une photo de chantier ; l'IA la décrit (constat, gravité, localisation, recommandations) ;
- **Résumé IA du dossier** (`Bot`) : synthèse de l'ensemble des notes ;
- **Joindre des fichiers** (`Paperclip`, plusieurs à la fois) ;
- **Enregistrement audio** (`Mic` / `Square`, avec minuterie mm:ss) — pratique pour dicter une note sur le chantier ;
- bouton **Ajouter la note**.

Après un enrichissement, un panneau **« Actions identifiées par l'IA »** peut apparaître. Le **Résumé IA** affiche, lui, un résumé, les **« Problèmes ouverts »**, les **« Actions en attente »** et le nombre de notes analysées.

**Liste des notes** : chaque note porte un **badge de catégorie** (defaut, observation, progression, decision, action, general), un indicateur d'**épinglage**, son auteur et sa date. Les pièces jointes s'affichent selon leur type :

- **images** en aperçu direct, avec agrandissement et téléchargement ;
- **audio** avec un lecteur intégré ;
- **autres fichiers** sous forme de boutons de téléchargement.

Par note : **Épingler / Désépingler** et **Supprimer**. Les notes épinglées remontent en tête.

> Les pièces jointes de notes sont limitées à **10 fichiers de 15 Mo** par ajout, et l'analyse de photo à **15 Mo**. C'est distinct du téléversement de documents (150 Mo).

### 2.11 Onglet Liens

**Formulaire** : une **URL** * (obligatoire, jusqu'à 2048 caractères, doit commencer par `http://` ou `https://` et ne pas contenir d'espace) et une **Description** (jusqu'à 1000 caractères, avec compteur). La **liste** montre chaque lien (ouverture dans un nouvel onglet, en toute sécurité), sa description et sa date d'ajout, avec édition sur place (`Pencil`) et suppression (`Trash2`).

Ces liens servent à pointer vers des ressources externes : dossier infonuagique du client, permis municipal en ligne, plans hébergés ailleurs, etc.

### 2.12 Onglet Extras (avenants)

L'onglet Extras gère les **avenants au contrat** — les travaux supplémentaires demandés en cours de chantier — et permet de les **facturer**.

**Bouton bascule « Assistant IA »** : ouvre l'assistant Extras (voir 2.14), qui peut proposer de créer/modifier/facturer un extra **sur confirmation**.

**4 cartes de totaux** : **Approuvés**, **À facturer**, **Facturés** (avec, le cas échéant, la note « − X en avoirs » et « + X en brouillon (à envoyer) »), et **Total**. Ces totaux tiennent compte des **avoirs** (notes de crédit) et distinguent les factures encore en **brouillon**.

**Formulaire d'ajout** : **Description** *, **Montant** (avant taxes, jusqu'à 10 000 000 $), **Date**. Bouton **Ajouter l'extra**.

**Liste des extras** : numéro (`EXT-XXXX`), badge de statut, montant, description, date, et « Facturé — {numéro} » le cas échéant.

| Statut d'extra | Libellé | Couleur |
|----------------|---------|---------|
| `PROPOSE` | Proposé | Jaune |
| `APPROUVE` | Approuvé | Bleu |
| `REFUSE` | Refusé | Rouge |
| `FACTURE` | Facturé | Vert |

**Actions par ligne** :

- un **menu déroulant de statut** : Proposé / Approuvé / Refusé ;
- un bouton **« Facturer »** (visible si l'extra est **Approuvé** et son montant positif) → il crée une **facture BROUILLON** liée au dossier ;
- **Modifier** / **Supprimer**.

Deux règles importantes :

- un extra **Facturé est verrouillé** : ni statut, ni description/montant, ni suppression (le serveur renvoie une erreur 409 si on essaie) ;
- passer un extra à **Approuvé** peut déclencher une **facturation immédiate** (« auto-facture ») **uniquement si vous avez le droit de facturer** (admin/comptable) **et** que le dossier a un client. Un utilisateur sans ce droit peut approuver un extra sans que rien ne soit facturé — c'est le bouton « Facturer » (ou un administrateur) qui déclenchera la facture plus tard.

### 2.13 Écran public — `/dossiers/public/:token`

C'est la page qu'un client ou un sous-traitant ouvre **sans compte**, à partir du lien que vous lui avez envoyé. Le serveur reconnaît le **jeton** et présente le contenu autorisé.

**Deux modes** selon le type de lien :

- **dossier entier** : la liste à plat de tous les documents ;
- **répertoire ciblé** : une navigation arborescente en lecture, limitée au sous-arbre partagé.

**En-tête** : le nom de l'entreprise, le titre du dossier, son numéro, un badge de statut, un compteur « Documents partagés » et une **bascule de langue FR / EN**.

**Bandeau de niveau d'accès** :

- niveau **Lecteur seul** : bandeau ambre « Consultation en ligne : téléchargement désactivé » ;
- niveau **Contributeur** : bandeau bleu invitant à glisser-déposer des fichiers.

**Par fichier** : un bouton **« Consulter »** (si le type est prévisualisable : image, texte ou PDF) et un bouton **« Télécharger »** (sauf au niveau Lecteur seul).

**Zone de téléversement invité** : seulement au niveau **Contributeur** — l'invité peut déposer des fichiers, qui atterrissent dans le répertoire partagé.

**Sécurité de la page publique** (transparente pour l'utilisateur, mais utile à connaître) : le lien est protégé contre les abus par des **limites de débit** (par adresse IP d'abord, puis par jeton), le téléversement invité est plafonné (5000 fichiers / 10 Go cumulés par dossier), les fichiers servis sont vérifiés par leur signature réelle (un fichier déguisé est neutralisé) et forcés en téléchargement, et **le lien cesse de fonctionner si l'abonnement de l'entreprise est désactivé**. Un pied de page rappelle « Lien sécurisé — {entreprise} · Documents en lecture seule ».

### 2.14 Les deux assistants IA

Le module comporte **deux assistants distincts** (ce ne sont pas deux variantes du même) :

| Assistant | Où | Nature |
|-----------|-----|--------|
| **Assistant IA Dossiers** | Bouton « Assistant IA » de la **liste** (modale) | **Lecture seule.** Il interroge vos dossiers, notes, étapes, liens, extras et factures liées et répond en langage naturel. Il **n'écrit rien** et **ne lit pas le contenu des fichiers joints**. |
| **Assistant IA Extras** | Bouton « Assistant IA » de l'**onglet Extras** | **Écriture sur confirmation.** Il peut proposer de créer un extra, de changer son statut, de le facturer, de le modifier ou de le supprimer. Chaque proposition s'affiche sous forme de **carte** (champs + totaux) que vous **confirmez** ou **annulez**. Rien n'est écrit tant que vous n'avez pas confirmé. |

Points communs : les deux répondent en français ou en anglais, gardent un historique borné (40 échanges), empêchent le double-envoi, et **consomment des crédits IA** (le module a donc un coût, voir 4.22). L'assistant Extras revérifie vos droits **au moment de confirmer** une facturation : un utilisateur sans droit de facturation ne peut pas contourner la règle en passant par l'IA.

---

## 3. Workflows pas à pas

### 3.1 Créer un dossier

1. Liste des dossiers → **Nouveau dossier**.
2. Saisir le **Nom** (obligatoire), choisir le **Type** et la **Priorité**.
3. **Créer** : le dossier `DOS-AAAA-NNNNN` est créé au statut **Ouvert** et sa Fiche 360 s'ouvre.

### 3.2 Renommer un dossier ou changer son statut

- **Renommer** : dans l'en-tête de la Fiche 360, cliquez sur le **crayon** à côté du titre, tapez le nouveau nom, faites **Entrée** (ou **Échap** pour annuler).
- **Changer le statut** : utilisez le **menu déroulant** de statut dans l'en-tête (Ouvert / En cours / En attente / Terminé / Archivé). Le changement est immédiat.

> Pour « archiver » un dossier, mettez simplement son statut à **Archivé**. Il n'y a pas de bouton « Archiver » distinct.

### 3.3 Rattacher un devis, un projet, une facture, etc.

1. Ouvrez l'onglet correspondant (Soumissions, Projet, Bons de travail, Achats, Demandes de prix ou Factures).
2. Cliquez **« + Associer {type} »**.
3. Choisissez l'élément dans le menu déroulant (seuls les éléments non encore rattachés apparaissent). Le lien est créé aussitôt.

Vous pouvez rattacher un même dossier à plusieurs devis, plusieurs factures, etc. Le rattachement est **idempotent** : associer deux fois le même élément ne crée pas de doublon.

### 3.4 Détacher un élément

1. Dans le même onglet, section **« Retirer une association »**.
2. Cliquez la pastille `#id` de l'élément à détacher.

L'élément d'origine (devis, projet…) reste intact ; seul le lien disparaît.

### 3.5 Organiser les documents en répertoires

1. Onglet **Documents** → **Nouveau répertoire** ; nommez-le.
2. Pour créer un sous-répertoire, ouvrez d'abord le répertoire parent, puis **Nouveau sous-répertoire** (jusqu'à 5 niveaux).
3. Pour **déplacer** un fichier ou un répertoire : bouton **Déplacer** → choisissez la destination (même dossier ou un autre dossier, puis le répertoire cible).
4. Pour **copier** : bouton **Copier** (un répertoire est copié avec tout son contenu).

### 3.6 Téléverser des documents

1. Onglet **Documents** → **Ajouter un document** (ou glissez les fichiers dans la zone de dépôt).
2. Sélectionnez un ou plusieurs fichiers (150 Mo maximum chacun).
3. La barre de progression indique l'avancement ; les fichiers apparaissent dans le répertoire courant.

> Le type de fichier est libre. Il n'y a pas de choix de catégorie à cette étape.

### 3.7 Télécharger, renommer ou supprimer un document

- **Télécharger** : bouton **Télécharger** sur la ligne du fichier.
- **Aperçu** : cliquez le nom, ou le bouton **Aperçu** (images, PDF, texte).
- **Renommer** : bouton **Renommer** ; l'extension d'origine est conservée automatiquement.
- **Supprimer** : bouton **Supprimer** (le fichier est effacé de la base).

### 3.8 Partager le dossier entier avec un client

1. Onglet **Documents** → **Partager** (en-tête).
2. Le lien `…/dossiers/public/{token}` est généré (valable **90 jours**). Copiez-le et envoyez-le au client.
3. Suivez les **consultations** et **téléchargements** dans la même modale.
4. Pour couper l'accès : **Révoquer**. Pour renouveler le lien (et invalider l'ancien) : **Régénérer**.

### 3.9 Partager un seul répertoire avec un niveau d'accès

1. Onglet **Documents** → bouton **Partager** sur la **ligne d'un répertoire**.
2. Choisissez le **niveau** :
   - **Lecteur seul** : le tiers consulte en ligne mais ne peut pas télécharger ;
   - **Lecture + téléchargement** : il consulte et télécharge ;
   - **Contributeur** : il consulte, télécharge **et** peut déposer ses propres fichiers.
3. Choisissez l'**expiration** (30 jours, 90 jours ou Jamais).
4. Copiez le lien et envoyez-le. Vous pouvez le révoquer à tout moment.

> Utilisez **Contributeur** pour recevoir des documents d'un sous-traitant (photos, fiches techniques) sans lui créer de compte.

### 3.10 Consulter un lien public (côté client, sans compte)

1. Le client ouvre le lien reçu.
2. Il choisit sa **langue** (FR / EN) au besoin.
3. Il **consulte** les documents ; il **télécharge** si le niveau l'autorise ; il **dépose** des fichiers s'il est Contributeur.

### 3.11 Écrire une note et l'enrichir avec l'IA

1. Onglet **Notes** → tapez votre note dans la zone de texte (ou dictez-la avec le micro).
2. Joignez des photos/fichiers au besoin (jusqu'à 10 × 15 Mo).
3. **Ajouter la note.**
4. Pour la structurer : **Enrichir avec IA**. Pour extraire un constat d'une photo : **Analyser photo IA**. Pour une synthèse de tout le dossier : **Résumé IA du dossier**.

> Chaque appel d'IA consomme des crédits (voir 4.22).

### 3.12 Ajouter un lien externe

1. Onglet **Liens** → collez l'**URL** (elle doit commencer par `http://` ou `https://`).
2. Ajoutez une **description** courte.
3. Validez. Le lien s'ouvre ensuite dans un nouvel onglet, en toute sécurité.

### 3.13 Créer, approuver et facturer un extra (avenant)

1. Onglet **Extras** → renseignez **Description**, **Montant** (avant taxes) et **Date** → **Ajouter l'extra**. Il naît au statut **Proposé**.
2. Faites-le valider par le client, puis passez-le à **Approuvé** (menu déroulant de statut).
3. **Facturer** :
   - si vous êtes **administrateur ou comptable** et que le dossier a un **client**, passer l'extra à « Approuvé » peut déclencher la facture **automatiquement** ;
   - sinon, cliquez le bouton **« Facturer »** sur la ligne de l'extra (réservé aux administrateurs/comptables).
4. Une **facture BROUILLON** `FACT-AAAA-NNNNN` est créée et liée au dossier ; l'extra passe à **Facturé** et se verrouille. Terminez l'envoi de la facture depuis le module Comptabilité.

> Un extra sans client rattaché au dossier **ne peut pas** être facturé (le système renvoie une erreur explicite). Rattachez d'abord un client au dossier.

### 3.14 Interroger l'assistant IA Dossiers (lecture)

1. Liste des dossiers → **Assistant IA**.
2. Posez une question sur vos dossiers (ex. « Quels dossiers ont un solde dû ? », « Résume les extras approuvés du chantier X »).
3. L'assistant lit vos données et répond. Il **n'écrit rien** et ne lit pas le contenu des fichiers.

### 3.15 Utiliser l'assistant IA Extras (proposer → confirmer)

1. Onglet **Extras** → bascule **Assistant IA**.
2. Décrivez ce que vous voulez (ex. « Ajoute un extra de 3 200 $ pour le drain français, daté d'aujourd'hui »).
3. L'IA affiche une **carte de proposition** (champs + total).
4. Vérifiez, puis **Confirmer** (l'extra est réellement créé/modifié/facturé) ou **Annuler**. Une facturation confirmée revérifie vos droits.

### 3.16 Supprimer un dossier

1. En-tête de la Fiche 360 (ou bouton corbeille de la liste) → **Supprimer**.
2. Confirmez l'avertissement.
3. Le serveur supprime **en cascade** les pièces jointes, notes, étapes, liens, extras et les 5 tables de liaison, purge les jetons de partage, puis **détache** (met à NULL) les opportunités et projets liés (ils ne sont pas supprimés). L'opération est **irréversible**.

> Les devis, projets, bons de commande et factures liés **restent en base** — ils perdent seulement leur rattachement au dossier.

---

## 4. Référence

> Rappel : la base de tous les endpoints est `/api/erp/v1`. « lecture » = `Depends(get_current_user)` (tout compte du locataire). Les écritures sont automatiquement bloquées en **mode consultation** (lecture seule, pilotée par Stripe).

### 4.1 Écrans et routes

| Écran | Route front | Composant |
|-------|-------------|-----------|
| Liste | `/dossiers` | `DossiersPage.tsx` |
| Détail (Fiche 360) | `/dossier/:id` | `DossierDetailPage.tsx` |
| Page publique | `/dossiers/public/:token` | `DossierPublicPage.tsx` |

### 4.2 Endpoints — dossier (CRUD)

| Méthode | Chemin | Garde | Référence |
|---------|--------|-------|-----------|
| GET | `/documents/statistics` | lecture | `documents.py:451` |
| GET | `/documents` | lecture | `documents.py:488` |
| GET | `/documents/{id}` | lecture | `documents.py:574` |
| POST | `/documents` | lecture | `documents.py:607` |
| PUT | `/documents/{id}` | lecture | `documents.py:663` |
| DELETE | `/documents/{id}` | lecture | `documents.py:706` |

> `PUT /documents/{id}` n'accepte que 4 champs : **titre**, **statut**, **priorite**, **notes** (liste blanche). `DELETE` verrouille le dossier, purge 13 tables enfant et les jetons de partage, puis met à NULL `opportunities.dossier_id` et `projects.dossier_id`.

### 4.3 Endpoints — pièces jointes (fichiers `BYTEA`)

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| POST | `/documents/{id}/attachments` | Téléverser (max **150 Mo** → 413) | `documents.py:1005` |
| GET | `/documents/{id}/attachments` | Lister | `documents.py:1090` |
| GET | `/documents/{id}/attachments/{att_id}/download` | Télécharger | `documents.py:1127` |
| GET | `/documents/{id}/attachments/{att_id}/preview` | Aperçu en ligne (durci) | `documents.py:1177` |
| DELETE | `/documents/{id}/attachments/{att_id}` | Supprimer | `documents.py:1287` |
| PATCH | `/documents/{id}/attachments/{att_id}/folder` | Déplacer (dossier/répertoire) | `documents.py:1937` |
| PATCH | `/documents/{id}/attachments/{att_id}` | Renommer (extension conservée) | `documents.py:1997` |
| POST | `/documents/{id}/attachments/{att_id}/copy` | Copier | `documents.py:2059` |

### 4.4 Endpoints — répertoires (arborescence, profondeur ≤ 5)

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| POST | `/documents/{id}/folders` | Créer | `documents.py:1509` |
| GET | `/documents/{id}/folders` | Lister | `documents.py:1571` |
| PUT | `/documents/{id}/folders/{fid}` | Renommer / déplacer | `documents.py:1607` |
| DELETE | `/documents/{id}/folders/{fid}` | Supprimer (`reparent` ou `cascade`) | `documents.py:1766` |
| PATCH | `/documents/{id}/folders/{fid}/move` | Déplacer le sous-arbre (cross-dossier) | `documents.py:2137` |
| POST | `/documents/{id}/folders/{fid}/copy` | Copier récursivement | `documents.py:2322` |

### 4.5 Endpoints — notes (+ IA)

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| GET | `/documents/{id}/notes` | Lister (100 max, épinglées d'abord) | `documents.py:2609` |
| POST | `/documents/{id}/notes` | Créer (1 à 20000 caractères) | `documents.py:2662` |
| POST | `/documents/{id}/notes-with-files` | Créer avec fichiers (**10 × 15 Mo**) | `documents.py:2700` |
| GET | `/documents/{id}/notes/{nid}/attachment/{idx}` | Télécharger une pièce jointe de note | `documents.py:2783` |
| DELETE | `/documents/{id}/notes/{nid}` | Supprimer | `documents.py:2832` |
| PATCH | `/documents/{id}/notes/{nid}/pin` | Épingler / désépingler | `documents.py:3468` |
| PATCH | `/documents/{id}/notes/{nid}/categorie` | Recatégoriser | `documents.py:3508` |
| POST | `/documents/{id}/notes/ai/enrich` | **IA** : enrichir (crédits) | `documents.py:3083` |
| POST | `/documents/{id}/notes/ai/analyze-photo` | **IA Vision** : analyser une photo (**15 Mo**, crédits) | `documents.py:3187` |
| POST | `/documents/{id}/notes/ai/summary` | **IA** : résumer (≤ 200 notes, crédits) | `documents.py:3326` |

### 4.6 Endpoints — liens, étapes, éléments liés, 360

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| GET/POST/PUT/DELETE | `/documents/{id}/liens[/{lid}]` | Liens externes (URL `http(s)` ≤ 2048) | `documents.py:2871`+ |
| GET/POST | `/documents/{id}/etapes` | Étapes (**pas d'onglet UI**) | `documents.py:2482`, `2518` |
| PUT | `/documents/{id}/etapes/{eid}/toggle` | Cocher/décocher une étape | `documents.py:2555` |
| GET | `/documents/{id}/linked` | Éléments liés (résumé) | `documents.py:3553` |
| GET | `/documents/{id}/360` | Vue agrégée (Résumé/Comptabilité) | `documents.py:3646` |

### 4.7 Endpoints — associer / dissocier des éléments

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| POST | `/documents/{id}/link` | Associer un élément | `documents.py:5488` |
| DELETE | `/documents/{id}/link/{type}/{item_id}` | Dissocier | `documents.py:5538` |
| GET | `/documents/{id}/linkable?item_type=` | Éléments associables | `documents.py:5577` |

**Types et tables de liaison** (`LINK_TABLES`, `documents.py:5446`) : `devis` → `dossier_devis` · `projet` → `dossier_projets` · `bon_travail` → `dossier_formulaires` (filtre `BON_TRAVAIL`) · `bon_commande` → `dossier_achats` · `facture` → `dossier_factures` · `demande_prix` → `dossier_formulaires` (filtre `DEMANDE_PRIX`).

### 4.8 Endpoints — partage par jeton

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| POST | `/documents/{id}/share` | Générer / régénérer le lien du dossier (**90 j**) | `documents.py:4492` |
| DELETE | `/documents/{id}/share` | Révoquer le lien du dossier | `documents.py:4530` |
| GET | `/documents/{id}/share-info` | Jeton + statistiques | `documents.py:4585` |
| POST | `/documents/{id}/folders/{fid}/share` | Lien d'un répertoire (niveau + expiration) | `documents.py:4621` |
| DELETE | `/documents/{id}/folders/{fid}/share` | Révoquer le lien d'un répertoire | `documents.py:4700` |
| GET | `/documents/{id}/folders/{fid}/share-info` | Jeton + statistiques (répertoire) | `documents.py:4760` |
| GET | `/documents/{id}/shares` | Lister tous les liens actifs | `documents.py:4814` |

### 4.9 Endpoints — page publique (sans authentification)

| Méthode | Chemin | Rôle | Référence |
|---------|--------|------|-----------|
| GET | `/documents/public/{token}` | Métadonnées + liste des documents | `documents.py:4934` |
| GET | `/documents/public/{token}/attachments/{att_id}` | Aperçu en ligne | `documents.py:5114` |
| GET | `/documents/public/{token}/attachments/{att_id}/download` | Télécharger (**refusé au niveau `reader`**) | `documents.py:5193` |
| POST | `/documents/public/{token}/upload` | Téléverser (**`contributor` seulement**) | `documents.py:5285` |

### 4.10 Endpoints — extras (avenants) et facturation

| Méthode | Chemin | Garde | Référence |
|---------|--------|-------|-----------|
| GET | `/documents/{id}/extras` | lecture | `documents.py:5694` |
| POST | `/documents/{id}/extras` | lecture (crée en **Proposé**) | `documents.py:5797` |
| PUT | `/documents/{id}/extras/{eid}` | lecture (verrou si facturé → 409 ; auto-facture si admin/comptable) | `documents.py:5859` |
| DELETE | `/documents/{id}/extras/{eid}` | lecture (409 si facturé) | `documents.py:5964` |
| POST | `/documents/{id}/extras/{eid}/facturer` | **`require_tenant_admin_or_role("comptable")`** | `documents.py:6072` |

> Facturer produit une **facture BROUILLON** via le module Comptabilité (`_create_invoice_from_source`), avec les taxes du locataire et un numéro `FACT-AAAA-NNNNN`, puis lie la facture au dossier et passe l'extra à **Facturé**.

### 4.11 Endpoints — assistants IA

| Assistant | Méthode | Chemin | Nature | Référence |
|-----------|---------|--------|--------|-----------|
| **Dossiers (lecture seule)** | POST | `/dossiers/ai/chat` | Interroge la base (SELECT en liste blanche) | `dossiers_ai.py:209` (monté `erp_api.py:1021`) |
| **Extras (proposer→confirmer)** | POST | `/documents/ai/chat` | Propose des actions sur les extras | `dossier_ai.py:490` (monté `erp_api.py:1137`) |
| **Extras (proposer→confirmer)** | POST | `/documents/ai/confirm-action` | Exécute l'action confirmée (revérifie les droits) | `dossier_ai.py:666` |

### 4.12 Statuts, types, priorités, catégories

| Ensemble | Valeurs (base) |
|----------|----------------|
| Statuts de dossier (`DOSSIER_STATUTS`, `documents.py:57`) | OUVERT · EN_COURS · EN_ATTENTE · TERMINE · ARCHIVE |
| Types de dossier (UI) | Projet · Client · Fournisseur · Administratif · Autre |
| Priorités (`PRIORITES`, `documents.py:62`) | BASSE · NORMAL · HAUTE · URGENT |
| Catégories de documents (`DOCUMENT_CATEGORIES`, `documents.py:52`) — **définies mais non exposées à l'écran** | PLAN · PHOTO · CONTRAT · FACTURE · CORRESPONDANCE · ADDENDA · FICHE_TECHNIQUE · SOUMISSION · DIRECTIVE_CHANTIER · AUTRE (défaut AUTRE) |
| Catégories de notes (`_NOTE_CATEGORIES`, `documents.py:39`) | defaut · observation · progression · decision · action · general (défaut general) |
| Statuts d'extra (`EXTRA_STATUTS`, `documents.py:112`) | PROPOSE · APPROUVE · REFUSE · FACTURE |
| Niveaux de partage de répertoire (`_VALID_PERMISSION_LEVELS`, `documents.py:140`) | reader · downloader · contributor |

### 4.13 Calcul de la comptabilité 360 (onglet Comptabilité / Résumé)

Source : `GET /documents/{id}/360` (`documents.py:3646`, bloc comptable `4050-4139`).

| Élément | Règle |
|---------|-------|
| **Périmètre** | Ancré sur le projet ou l'opportunité du dossier ; élargi au client **seulement** s'il n'y a aucun ancrage (évite de compter deux fois entre dossiers d'un même client). |
| **Coût main-d'œuvre** | Estimation `SUM(heures × taux)` **remplacée** par le coût réel de la paie (`payroll_entry_projets.cout_reparti`) dès qu'il est positif. |
| **Revenus** | Exclut les factures ANNULÉE et BROUILLON, exclut les factures fournisseur, **soustrait les avoirs** ; marge calculée sur la base **hors taxes**. |
| **Achats** | Bons de commande **non** déjà représentés par une facture fournisseur comptée (anti double-comptage). |
| **Marge estimée** | `total_facturé_HT − total_coûts`. |

### 4.14 Limites, bornes et défenses

| Élément | Valeur |
|---------|--------|
| Téléversement d'un document | **150 Mo** par fichier (→ 413) ; lecture par tranches de 64 Ko |
| Pièces jointes de note | **10 fichiers × 15 Mo** |
| Analyse de photo IA | **15 Mo** |
| Profondeur de l'arborescence | **5 niveaux** (racine = niveau 1) |
| Montant d'un extra | ≤ **10 000 000 $** |
| Note (texte) | 1 à 20000 caractères |
| Description d'un lien | ≤ 1000 caractères ; URL `http(s)` ≤ 2048 |
| Pagination de la liste | `page` ≥ 1, `per_page` 1-100 (20 à l'écran) |
| Recherche | LIKE échappé (`% _`), tronquée à 100 caractères |
| Expiration — lien de dossier | **90 jours** (fixe) |
| Expiration — lien de répertoire | 30 j / 90 j / **Jamais** (défaut 90 j, ≤ 3650 j) |
| Débit — lecture publique | 480 / 10 min par IP · 240 / 10 min par jeton |
| Débit — téléversement public | 300 / 10 min par IP · 200 / 10 min par jeton |
| Plafond cumulatif — invités (contributeur) | 5000 fichiers / 10 Go par dossier (→ 429) |

### 4.15 Sécurité des fichiers (auth et public)

- **Signature réelle vérifiée** (« magic bytes ») : un fichier déguisé (ex. exécutable renommé en `.pdf`) est servi en `octet-stream`, jamais interprété.
- **Aperçu en ligne restreint** à une liste blanche de types (image, PDF, texte), avec en-têtes `X-Content-Type-Options: nosniff` et une politique de contenu stricte.
- **Nom de fichier assaini** pour l'en-tête de téléchargement (pas d'injection de retour à la ligne).
- **Téléchargement forcé** (`Content-Disposition: attachment`) pour les téléchargements.
- **Jetons** générés aléatoirement (`secrets.token_urlsafe`) ; régénérer fait **tourner** le jeton de façon atomique (aucun lien fantôme).
- **Isolation** : chaque dossier, répertoire et pièce jointe est revérifié comme appartenant au bon locataire (protection contre l'accès par identifiant d'autrui).

### 4.16 Assistants IA — modèle, coût, débit

| Élément | Valeur |
|---------|--------|
| Modèle | `claude-sonnet-4-6` |
| Coût par échange | (entrée × 0,003 + sortie × 0,015) / 1000 × **1,30** (marge 30 %) |
| Débité de | `public.ai_prepaid_credits` (crédits prépayés du locataire) |
| Gardes | 503 si l'IA est indisponible · 403 si l'IA est bloquée · 402 si crédits insuffisants |
| Débit par IP — notes IA | 15 / min |
| Débit par IP — assistant Dossiers | 20 / min |
| Débit par IP — assistant Extras | 20 / min |
| Assistant Dossiers | **Lecture seule**, ne lit pas le contenu des fichiers |
| Assistant Extras | Écriture **sur confirmation** ; facturation revérifiée (admin/comptable) |

### 4.17 Raccourcis et gestes

| Action | Geste |
|--------|-------|
| Ouvrir un dossier | Clic sur la ligne (liste) ou `/dossiers?open=<id>` |
| Renommer le titre | Crayon → **Entrée** (valider) / **Échap** (annuler) |
| Téléverser | Bouton « Ajouter un document » **ou** glisser-déposer dans la zone |
| Envoyer un message à un assistant IA | **Entrée** |
| Régénérer un lien de partage | Bouton « Régénérer » (l'ancien lien cesse aussitôt) |

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|--------|------|
| **06 — Ventes (CRM)** | À la conversion d'une opportunité, un dossier peut être créé et relié (`opportunities.dossier_id`). Le bouton « Voir le dossier » du CRM ouvre `/dossiers?open=<id>`. À la suppression du dossier, l'opportunité est **détachée** (non supprimée). |
| **08 — Soumissions (devis)** | Les devis se rattachent au dossier (onglet Soumissions) et s'ouvrent via `/devis?open=…`. |
| **09 — Projets** | Un projet peut être relié (`projects.dossier_id`) ; il alimente le pointage et la comptabilité du dossier. Détaché à la suppression du dossier. |
| **12 — Bons de travail** | Rattachés via l'onglet Bons de travail ; ouverture via `/bons-travail?open=…`. |
| **13 — Pointage** | L'onglet Pointage agrège les heures et coûts des employés sur le projet du dossier. |
| **14 — Bons de commande / Magasin** | Les achats se rattachent (onglet Achats) et s'ouvrent via `/magasin?open=…`. |
| **15 — Comptabilité** | Les factures se rattachent (onglet Factures) ; **facturer un extra** y crée une facture BROUILLON. L'onglet Comptabilité calcule revenus, coûts et marge. |
| **25 — Assistant IA** | Les assistants Dossiers et Extras (et les outils IA des notes) consomment les **crédits prépayés** du locataire. |
| **28 — Configuration** | La configuration de taxes du locataire s'applique à la facture d'un extra. L'état de l'abonnement Stripe pilote le **mode consultation**. |
| **Application mobile** | La table des extras est **partagée** avec l'application mobile de chantier : un extra saisi sur le terrain apparaît dans l'onglet Extras du web (et inversement). |

### 5.2 FAQ

**Q1. Pourquoi l'adresse dit « documents » alors que le menu dit « Dossiers » ?**
C'est un héritage du code : le nom interne d'origine était « documents ». L'interface a été renommée « Dossiers », mais les endpoints restent sous `/api/erp/v1/documents`. C'est cosmétique et sans effet pour vous.

**Q2. Qui peut créer, modifier ou supprimer un dossier ?**
Tout utilisateur du locataire ayant un compte ERP valide. Il n'y a pas de rôle « gestionnaire de dossier ». La **seule** action réservée (aux administrateurs et comptables) est la **facturation d'un extra**.

**Q3. Où sont stockés les fichiers ? Y a-t-il une limite ?**
Les documents sont stockés **dans la base de données** (colonne `BYTEA`), pas sur un service infonuagique externe. La limite est de **150 Mo par fichier**. Les pièces jointes de notes sont limitées à 10 fichiers de 15 Mo.

**Q4. Comment archiver un dossier ?**
Mettez son statut à **Archivé** (menu déroulant de l'en-tête). Il n'y a pas de bouton « Archiver » distinct, et l'archivage ne supprime rien.

**Q5. Puis-je choisir une catégorie (Plan, Photo, Contrat…) au téléversement ?**
Non. Les catégories existent côté serveur mais ne sont pas proposées à l'écran. Organisez plutôt vos fichiers en **répertoires**.

**Q6. Quelle est la différence entre partager le dossier et partager un répertoire ?**
Le partage du **dossier entier** donne accès, en lecture, à tous les documents (lien de 90 jours). Le partage d'un **répertoire** est plus fin : vous choisissez le **niveau** (Lecteur seul / Lecture + téléchargement / Contributeur) et l'**expiration** (30 j, 90 j, Jamais), et l'accès se limite au sous-arbre partagé.

**Q7. Le client peut-il déposer des fichiers via le lien public ?**
Oui, mais **seulement** si vous lui donnez un lien de **répertoire** au niveau **Contributeur**. Les liens de dossier entier et les niveaux Lecteur seul / Lecture + téléchargement ne permettent pas le dépôt.

**Q8. Comment couper un accès partagé ?**
Ouvrez le partage concerné et cliquez **Révoquer**. Pour renouveler un lien (et invalider immédiatement l'ancien), cliquez **Régénérer**. Un lien cesse aussi de fonctionner à son expiration, ou si l'abonnement de l'entreprise est désactivé.

**Q9. Puis-je suivre qui a consulté ou téléchargé ?**
Oui : la modale de partage affiche le nombre de **consultations** et de **téléchargements**, ainsi que les dernières dates. Il n'y a pas de notification par courriel automatique.

**Q10. Que se passe-t-il si je supprime un dossier ?**
Ses pièces jointes, notes, étapes, liens et extras sont **supprimés** ; les jetons de partage sont purgés ; les opportunités et projets liés sont **détachés** (conservés). Les devis, bons de travail, bons de commande et factures liés **restent** en base — ils perdent seulement le rattachement. L'opération est **irréversible** : préférez l'archivage si vous hésitez.

**Q11. Qu'est-ce qu'un extra, et pourquoi certaines personnes ne voient pas de bouton « Facturer » ?**
Un extra est un **avenant** (travaux supplémentaires). N'importe qui peut le créer et le suivre, mais **seuls les administrateurs et comptables peuvent le facturer**. Pour les autres, le bouton « Facturer » n'apparaît pas.

**Q12. Facturer un extra envoie-t-il la facture au client ?**
Non. Cela crée une **facture BROUILLON** liée au dossier. Vous la vérifiez et l'envoyez depuis le module Comptabilité.

**Q13. Pourquoi mon extra ne peut-il pas être facturé ?**
Deux raisons fréquentes : le dossier n'a **pas de client** rattaché, ou l'extra n'est **pas au statut Approuvé** (ou son montant est nul). Un extra déjà **Facturé** est verrouillé et ne peut plus être modifié ou supprimé.

**Q14. Quelle est la différence entre les deux assistants IA ?**
L'**assistant Dossiers** (bouton de la liste) est en **lecture seule** : il répond à vos questions sans rien modifier, et ne lit pas le contenu des fichiers. L'**assistant Extras** (onglet Extras) peut **agir** sur les extras (créer, changer le statut, facturer…), mais **seulement après votre confirmation**.

**Q15. Les outils IA sont-ils facturés ?**
Oui. Enrichir une note, analyser une photo, résumer les notes, et les deux assistants consomment des **crédits IA prépayés** (coût réel du modèle + marge de 30 %). Sans crédits suffisants, l'action est refusée (402).

**Q16. Y a-t-il une liste de vérification des étapes dans le dossier ?**
Pas dans l'interface actuelle : le serveur sait gérer des étapes, mais aucun onglet ne les affiche. C'est une fonctionnalité dormante.

**Q17. Puis-je exporter la liste des dossiers en PDF ou CSV ?**
Non. Le module ne propose **aucun export, aucune impression, aucune action en lot**. Les seules sorties de fichiers sont le téléchargement d'une pièce jointe et le partage public par lien.

**Q18. Jusqu'à combien de niveaux de répertoires puis-je créer ?**
**5 niveaux** (la racine comptant pour le premier). Au 5ᵉ niveau, le bouton « Nouveau sous-répertoire » est désactivé.

**Q19. Un extra saisi sur le mobile apparaît-il ici ?**
Oui. La table des extras est **partagée** entre le web et l'application mobile de chantier : les deux voient les mêmes avenants.

**Q20. Mon compte est en « Mode consultation » — que puis-je faire ?**
Vous pouvez **tout consulter**, mais aucune écriture n'est acceptée (création, téléversement, notes, extras, IA renvoient 403). Régularisez l'abonnement Stripe (module Configuration) pour revenir en écriture. Les liens publics déjà partagés continuent de fonctionner tant que l'entreprise reste active.

---

## 6. Récapitulatif

- **Un dossier = une Fiche 360°** : autour d'un chantier/client, il relie soumissions, projet, bons de travail, achats, demandes de prix, factures, pointage, comptabilité, documents, notes, liens et **extras**.
- **Trois écrans** : liste (`/dossiers`), Fiche 360 (`/dossier/:id`, **au singulier**), page publique (`/dossiers/public/:token`, sans compte).
- **13 onglets** dans la Fiche 360 ; **pas** d'onglet « Étapes » (fonction dormante), la gestion des partages est dans l'onglet **Documents**.
- **Numérotation** : dossier `DOS-AAAA-NNNNN`, extra `EXT-XXXX`, facture d'extra `FACT-AAAA-NNNNN` — tous générés sans risque de doublon.
- **Permissions ouvertes** : tout compte du locataire gère les dossiers ; **seule** la **facturation d'un extra** est réservée aux administrateurs/comptables. Le **mode consultation** (Stripe) met tout le module en lecture seule.
- **Documents** : arborescence jusqu'à **5 niveaux**, fichiers **150 Mo** en base (`BYTEA`), déplacement/copie même entre dossiers, aperçu sécurisé.
- **Partage** : lien de **dossier entier** (90 j, lecture) ou lien de **répertoire** avec **niveau** (Lecteur seul / Lecture + téléchargement / Contributeur) et **expiration** (30 j / 90 j / Jamais). Le niveau **Contributeur** autorise le dépôt de fichiers par un invité.
- **Notes** : catégories, épinglage, pièces jointes (images/audio/fichiers), et **3 outils IA** (enrichir, analyser une photo, résumer).
- **Extras (avenants)** : cycle Proposé → Approuvé → Facturé ; **facturer** crée une facture BROUILLON liée au dossier ; un extra facturé est verrouillé ; totaux conscients des avoirs et des brouillons.
- **Deux assistants IA distincts** : Dossiers (**lecture seule**, ne lit pas les fichiers) et Extras (**proposer → confirmer**, droits revérifiés). Modèle `claude-sonnet-4-6`, crédits prépayés + marge 30 %.
- **Ce que le module ne fait pas** : pas de catégorie au téléversement, pas de bouton « Archiver », pas d'export/impression/lot, pas d'envoi automatique de facture, pas d'onglet Étapes.
- **Suppression en cascade** : efface pièces jointes/notes/étapes/liens/extras/liaisons, purge les partages, **détache** opportunités et projets ; les éléments liés (devis/BT/BC/factures) sont préservés.

---

*Sources vérifiées (2026-07)* : `backend/routers/documents.py` (6138 lignes, 58 routes) · `backend/routers/dossiers_ai.py` (327 lignes, 1 route) · `backend/routers/dossier_ai.py` (772 lignes, 2 routes) · montage `erp_api.py:993` (documents), `1021` (dossiers_ai), `1137-1138` (dossier_ai, bloc défensif) · `frontend/src/pages/DossiersPage.tsx` (428 lignes) · `frontend/src/pages/DossierDetailPage.tsx` (3911 lignes, 13 onglets) · `frontend/src/pages/DossierPublicPage.tsx` (561 lignes) · `frontend/src/components/dossiers/{DossiersAssistantTab.tsx, ExtrasAssistant.tsx}` · `frontend/src/api/{documents.ts, dossiersAi.ts}` · Sidebar `Sidebar.tsx:51` (groupe « Gestion ») · i18n `dossiers.json`, `dossiersAssistant.json`.

*Manuels liés* : 06 — Ventes (CRM) · 08 — Soumissions · 09 — Projets · 12 — Bons de travail · 13 — Pointage · 14 — Bons de commande · 15 — Comptabilité · 25 — Assistant IA · 28 — Configuration.

*Manuel ERP Constructo AI — Module 07 Dossiers (gestion documentaire / Fiche 360) — v3.0 vérifié — 2026-07*
