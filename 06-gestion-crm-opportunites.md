# Module 06 — Ventes (CRM, opportunités, pipeline, B2B back-office)

> **Version** : 3.0 (refonte complète vérifiée contre le code source 2026-07)
> **Route frontend** : `/ventes` (menu latéral « Ventes », groupe « Gestion », icône `TrendingUp`). La route `/b2b` redirige vers `/ventes?tab=b2b`.
> **Préfixe API** : `/api/erp/v1`
> **Code de référence (backend)** : `backend/routers/crm.py` (2563 lignes, 25 routes CRM) · `backend/routers/ventes_ai.py` (479 lignes, 2 routes — assistant IA Ventes) · `backend/routers/b2b.py` (3017 lignes, 33 routes — back-office B2B) · `backend/routers/b2b_ai.py` (340 lignes — assistant IA B2B en lecture seule)
> **Code de référence (frontend)** : `frontend/src/pages/VentesPage.tsx` (2914 lignes, 8 onglets) · `frontend/src/pages/B2bPage.tsx` (1411 lignes, 10 sous-onglets) · `frontend/src/components/crm/BATQualificationForm.tsx` (289 lignes) · `frontend/src/components/ventes/VentesAssistantTab.tsx` (232 lignes) · `frontend/src/components/b2b/B2bAssistantTab.tsx` (152 lignes)
> **Clients API frontend** : `api/crm.ts` (384 lignes), `api/ventesAi.ts` (75 lignes), `api/b2b.ts` (529 lignes), `api/b2bAi.ts` (36 lignes). Note : **`api/ventes.ts` n'existe pas**.
> **Tables PostgreSQL (par locataire)** : `opportunities`, `interactions`, `crm_activities`, `prospect_qualifications` (B.A.T.), `opportunity_assignations`, `dossiers` (auto `DOS-OPP-…`), `devis` / `devis_lignes` (conversion), `b2b_clients`, `b2b_client_users`, `b2b_demandes`, `b2b_soumissions`, `b2b_contrats`, `b2b_commandes` (+ `b2b_commande_lignes`), `b2b_messages`, `b2b_favoris`, `b2b_notifications`. Tables partagées (chemin IA) : `public.ai_prepaid_credits`, `public.ai_usage_tracking`, `public.ai_credit_ledger`, `public.entreprises`.
> **Cadrage** : ce module est le **poste de travail commercial** de l'ERP. Il gère le pipeline d'opportunités (Kanban), les relances et le calendrier de suivi, la qualification (pointage automatique et grille B.A.T. manuelle), un assistant IA qui propose des opportunités sur confirmation, et — pour les administrateurs seulement — le **back-office B2B/B2C** (comptes clients du portail, demandes, soumissions, contrats, commandes, messagerie, catalogue). La gestion des **entreprises** et des **contacts** vit dans ses propres pages et manuels (modules 04 et 05). La conversion d'une opportunité produit un **devis brouillon** (module 08), pas un projet directement.

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

Le module Ventes centralise le **cycle commercial** d'une entreprise de construction, de la prise de contact jusqu'à la signature :

- suivre chaque affaire dans un **pipeline visuel** à 6 statuts ;
- **qualifier** les affaires (à chaud/froid par calcul automatique, ou en détail avec la grille B.A.T.) ;
- planifier et honorer les **relances** (appels, courriels, visites, tâches) ;
- **convertir** une opportunité gagnante en soumission (devis) en un clic ;
- gérer, du côté administrateur, tout le **back-office B2B** relié au portail client externe.

C'est un poste de pilotage : il **ne remplace pas** le module Soumissions (qui chiffre et émet les devis) ni le module Projets (qui exécute les chantiers). Il alimente le premier et prépare le second.

### 1.2 Accès par le menu latéral

Cliquez sur **Ventes** dans le menu latéral (groupe « Gestion »). La page s'ouvre sur l'onglet **Pipeline** par défaut.

Deux liens rapides existent :

- `app.constructoai.ca/ventes?open=<id>` ouvre directement une opportunité dans l'onglet **Opportunités** (utilisé par les boutons « Ouvrir » ailleurs dans l'ERP).
- `app.constructoai.ca/ventes?tab=b2b` (ou l'ancienne route `/b2b`) ouvre l'onglet **B2B/B2C**, si vous êtes administrateur.

### 1.3 Les deux périmètres et les 8 onglets

La page regroupe **deux périmètres distincts** sous un seul écran à onglets :

| # | Onglet | Périmètre | Visible par |
|---|--------|-----------|-------------|
| 1 | **Pipeline** | CRM — Kanban glisser-déposer des opportunités | Tous |
| 2 | **Relances** | CRM — file des tâches/relances par échéance | Tous |
| 3 | **Opportunités** | CRM — tableau paginé + panneau de détail | Tous |
| 4 | **Calendrier** | CRM — vue mensuelle des événements | Tous |
| 5 | **Historique** | CRM — fil chronologique interactions + activités | Tous |
| 6 | **Qualification** | CRM — pointage automatique + grille B.A.T. | Tous |
| 7 | **Assistant IA** | CRM — clavardage qui propose des opportunités sur confirmation | Tous |
| 8 | **B2B/B2C** | Back-office B2B (10 sous-onglets) | **Administrateurs seulement** |

L'onglet **B2B/B2C** est masqué dans la barre d'onglets ET non chargé pour un utilisateur non-administrateur (double protection, `VentesPage.tsx:276,313`).

Un **badge rouge** apparaît sur l'onglet **Relances** avec le nombre de relances dues (en retard + aujourd'hui), pour que vous sachiez d'un coup d'œil qu'il y a du suivi à faire.

### 1.4 Permissions et rôles

Le module applique **trois niveaux de permission** distincts. C'est une nuance importante : lire, écrire dans le CRM, et administrer le B2B ne demandent pas les mêmes droits.

| Niveau | Ce qui est protégé | Règle (`backend`) | Qui a le droit |
|--------|--------------------|--------------------|----------------|
| **Lecture** (CRM + B2B) | Voir opportunités, pipeline, relances, calendrier, statistiques, listes B2B | `get_current_user` (`erp_auth.py:475`) | Tout compte ERP valide. Les jetons de client B2B (portail externe) sont refusés (403). |
| **Écriture CRM** | Créer/modifier/supprimer opportunités, interactions, activités, qualification, conversion en devis | `require_crm_write` (`crm.py:40`) | `super_admin`, ou rôle dans `{admin, gestionnaire, contremaitre, user}`. **Le rôle `user` (opérateur principal) est autorisé.** `employee` et `comptable` sont **refusés** (403). |
| **Écriture B2B** | Créer clients/demandes/soumissions/contrats, approuver des accès, changer un statut de commande | `require_tenant_admin_or_role()` (`erp_auth.py:720`) | `is_admin` (relu côté serveur, infalsifiable) OU rôle `admin` OU `super_admin`. **Plus strict** : un `user` non-administrateur est **refusé**. |

Autrement dit : un employé avec le rôle `user` peut travailler tout le CRM (pipeline, relances, conversion en devis) mais **ne peut pas** entrer dans le back-office B2B ni administrer les accès au portail.

**Mode consultation (lecture seule).** Si l'abonnement Stripe du locataire est en souffrance, le compte peut basculer en mode **readonly** (les lectures passent, toute écriture renvoie 403) ou **blocked** (401). Ce contrôle est appliqué en amont, dans `get_current_user` (`erp_auth.py:520-530`, cache 60 s), et couvre **tous** les endpoints du module, y compris l'assistant IA. Un bandeau « Mode consultation » s'affiche alors dans l'interface.

### 1.5 Ce que le module ne fait pas (vérifié dans le code)

- **Aucun export PDF/CSV, aucune impression, aucun téléversement de fichier** dans tout le module Ventes/B2B (contrairement au module Suivi/Gantt). Les documents (devis PDF, contrats) se génèrent depuis les modules Soumissions/Dossiers.
- **Aucune création automatique de projet** : la conversion crée un devis brouillon ; le passage devis → projet relève du module Soumissions/Projets.
- **Aucun envoi automatique de courriel ni de notification** déclenché par un changement de statut d'opportunité.
- **Aucun calcul de commission** ni de rémunération vendeur.
- **Aucune interface d'assignation d'employé à une opportunité** : les fonctions existent côté API (`crm.ts:353-365`) mais ne sont appelées nulle part dans l'écran Ventes (surface morte, voir FAQ).
- **Aucune création de commande côté administrateur** : le catalogue B2B est **consultatif** ; les commandes se créent depuis le **portail client externe** (`/b2b-portal`).
- L'assistant IA Ventes **ne crée que des opportunités** (aucune autre action), et **seulement après confirmation** de votre part.

---

## 2. Interface

### 2.1 En-tête des indicateurs et barre d'onglets

En haut de la page, **quatre indicateurs** (KPI) restent visibles quel que soit l'onglet (`VentesPage.tsx:248-270`, source `GET /crm/stats`) :

| Indicateur | Signification |
|------------|---------------|
| **Opportunités** | Nombre total d'opportunités (avec la part « en cours »). |
| **Taux de conversion** | `gagnées / (gagnées + perdues) × 100`. |
| **Montant gagné** | Somme des `montant_estime` des opportunités GAGNE. |
| **Délai moyen** | Durée moyenne (jours) entre création et clôture des opportunités GAGNE. |

Sous les indicateurs, la **barre d'onglets** (les 8 onglets ci-dessus). L'onglet actif est surligné.

### 2.2 Onglet Pipeline (Kanban)

Vue Kanban en glisser-déposer. Il existe **6 statuts** au total, mais l'affichage les répartit ainsi :

- **4 colonnes actives, glissables** : Prospection → Qualification → Proposition → Négociation.
- **2 cartes-résumé** en haut de l'écran : **Gagné** (vert) et **Perdu** (rouge). Elles affichent le montant total et le nombre d'opportunités du statut, et servent aussi de **cibles de dépôt** : glisser une carte dessus la marque gagnée ou perdue.

> **Nuance importante** : Gagné et Perdu ne sont pas des colonnes du tableau, ce sont des zones de synthèse. Les 6 statuts existent bien en base ; seuls les 4 premiers ont une colonne.

**En-tête de chaque colonne** : badge de statut coloré + compteur d'opportunités + montant total de l'étape (source `GET /crm/pipeline`, agrégé par statut).

**Boutons de création** :

- **Nouvelle opportunité** (au-dessus du Kanban) ouvre la modale de création vierge.
- **Ajouter** (en pied de chaque colonne) ouvre la même modale mais **préremplit le statut** de la colonne.

**La carte d'opportunité** (`PipelineCard`) affiche :

- le **nom** de l'affaire et son numéro (`OPP-00001`) ;
- le nom de l'**entreprise** cliente ;
- le **montant estimé** et la **probabilité** (%) ;
- la **date de clôture** prévue ;
- le **score B.A.T.** : badge coloré `X/100` + catégorie **A+/A/B/C/D**. Si l'opportunité provient d'une pré-qualification de l'agent vocal (statut interne EN_COURS), la carte affiche « B.A.T. préliminaire / à compléter » au lieu d'un score ;
- un bouton **œil** « Ouvrir le détail » ;
- des **pastilles d'action rapide** : « Avancer vers <statut suivant> » (flèche), « Gagné », « Perdu ».

**Glisser-déposer** :

- **Entre deux colonnes** = changement de statut. La carte bouge immédiatement (affichage optimiste) ; si l'enregistrement échoue, elle revient à sa place (retour arrière automatique).
- **Dans la même colonne** = **réordonnancement** de la priorité d'affichage. L'ordre est persisté via `PUT /crm/opportunities/reorder` (charge utile `{orderedIds}`). En cas d'échec, l'ordre précédent est restauré.

**Ouvrir le détail** : double-cliquez sur une carte, ou cliquez sur l'œil. La **modale de détail** s'ouvre (voir 2.5).

**Supprimer** : depuis le détail, la suppression demande une confirmation native. Si un devis ou un projet est lié, l'avertissement précise qu'il sera **détaché** (et non supprimé).

### 2.3 Onglet Opportunités (tableau)

Vue tableau paginée (20 par page) avec un panneau de détail latéral.

**Barre de commande** :

- bouton **Nouvelle opportunité** ;
- champ de **recherche** (« Rechercher... ») — porte sur le nom, la source, les notes ; une garde de séquence empêche qu'une réponse en retard écrase une recherche plus récente ;
- filtre **statut** (« Tous les statuts » ou l'une des 6 valeurs).

**Colonnes (bureau)** : No. · Nom (+ source) · Entreprise · Montant · Prob. · Statut (badge) · Fermeture. Si la liste est vide : « Aucune opportunité trouvée ».

**Cartes (mobile)** : nom, badge de statut, entreprise, montant, probabilité, date.

Un **compteur total** et la **pagination** figurent en bas.

**Panneau de détail** (au clic sur une ligne, source `GET /crm/opportunities/{id}`) : sur bureau, il occupe environ 40 % à droite ; sur mobile, il passe en plein écran avec un bouton **Retour**. On y trouve les boutons **Modifier** (crayon), **Supprimer**, **Fermer**, ainsi que : entreprise, contact, montant, probabilité, date de clôture, source, notes, date « Créé le », les boutons **Créer une soumission** et **Voir le dossier**, la liste des **interactions** et la **grille B.A.T.**

**Mode édition** (bouton Modifier) : formulaire avec Nom, Statut, Montant estimé, Probabilité (0-100), Date de fermeture, Entreprise (menu déroulant), Source, Notes → `PUT /crm/opportunities/{id}`. Boutons **Enregistrer** / **Annuler**.

### 2.4 Modale de création d'opportunité

Ouverte par « Nouvelle opportunité » (ou « Ajouter »). Deux colonnes.

**Colonne de gauche** :

| Champ | Note |
|-------|------|
| **Nom de l'opportunité** * | Obligatoire (ex. « Rénovation cuisine Dupont »). |
| No. PO Client | Numéro de bon de commande du client, optionnel. |
| Client (Entreprise) | Menu déroulant des entreprises (chargé via `listCompanies`, 100 par page). |
| Client (Personne) | Menu déroulant des contacts. |
| Saisie manuelle | Pour un client hors CRM (nom libre). |
| Statut | Défaut Prospection (ou le statut de la colonne si ouverte par « Ajouter »). |
| Priorité | Basse / Normale / Haute / Urgente. |

**Colonne de droite** :

| Champ | Note |
|-------|------|
| Source | Texte libre (ex. « Site web, Recommandation, Salon... »). |
| Date limite de soumission | Optionnelle. |
| Début / Fin prévue des travaux | Optionnelles. |
| Montant estimé ($) | 0 à 1 000 000 000. |
| Probabilité | Curseur 0-100, pas de 5. |
| Date de clôture prévue | Optionnelle. |
| Description | Zone de texte. |
| Notes | Zone de texte. |

Une note « * Champs obligatoires » et les boutons **Annuler** / **Enregistrer** (→ `POST /crm/opportunities`) ferment la modale.

À la création, le système génère le numéro `OPP-00001`, ainsi qu'un **dossier client** `DOS-OPP-…` de type CLIENT (au mieux, voir module 07).

### 2.5 Modale de détail (Vue / Édition / Historique / B.A.T.)

Accessible depuis le Pipeline (double-clic/œil) et depuis Opportunités (clic ligne). Taille large. Elle comporte plusieurs zones :

**a) Vue** — numéro + badge de statut, nom, entreprise, contact, montant estimé, barre de probabilité, date de clôture, source, notes. Deux boutons d'action :

- **Créer une soumission** → `POST /crm/opportunities/{id}/create-devis`, puis redirection vers `/devis?open=<devisId>`. Si l'opportunité est déjà convertie, le serveur répond « déjà convertie en devis #X » et ce détail s'affiche.
- **Voir le dossier** (si un dossier est lié) → `/dossiers?open=…`.

**b) Édition** — mêmes champs que la création (formulaire 2 colonnes). Boutons Modifier / Supprimer.

**c) Fil interactions + activités** — fil chronologique **fusionné** des interactions et des activités de l'opportunité. Deux liens en tête, **Interaction** et **Activité**, ouvrent un **formulaire en ligne** :

| Champ | Détail |
|-------|--------|
| Type | Menu déroulant (types d'interaction ou d'activité). |
| Date | Date de l'événement. |
| Résumé / Sujet | Texte. |
| Ajouter | → `POST /crm/interactions` ou `POST /crm/activities`. |

Chaque élément du fil affiche : badge de type + titre + sous-texte + date + badge Inter./Act. + statut de l'activité.

**d) Grille B.A.T.** — la grille de qualification manuelle (voir 2.9) est intégrée directement dans la modale.

**Suppression** : le bouton Supprimer déclenche une confirmation native. La suppression **détache** (ne supprime pas) le devis/projet lié ; elle supprime en cascade les interactions, activités, assignations et qualifications de l'opportunité.

### 2.6 Onglet Relances

File de suivi de vos tâches et relances, triée par échéance. Sous-titre : « Vos relances et tâches à faire, par échéance. » (source `GET /crm/relances?horizonDays=7`).

**Trois tranches** :

| Tranche | Couleur | Contenu |
|---------|---------|---------|
| **En retard** | Rouge | Relances dont l'échéance est passée. |
| **Aujourd'hui** | Bleu | Relances du jour. |
| **À venir (7 jours)** | Gris | Relances des 7 prochains jours. |

Chaque **carte de relance** affiche : badge du type d'activité + date + sujet + opportunité/entreprise liée + badge de statut de l'opportunité. Trois actions :

- **Fait** → marque l'activité terminée (`PATCH /crm/activities/{id}`, statut TERMINE) ;
- **Reporter** → un champ de date en ligne apparaît (Confirmer / Annuler) pour déplacer l'échéance ;
- **Ouvrir** → ouvre l'opportunité (`/ventes?open=<id>`).

Si rien n'est planifié : « Aucune relance planifiée » avec une astuce.

### 2.7 Onglet Calendrier

Grille mensuelle (semaine du lundi au dimanche). Navigation **mois précédent / mois suivant** + bouton **Aujourd'hui** (source `GET /crm/calendar?year&month`).

**Trois types d'événements** avec légende colorée :

| Type | Couleur | Source |
|------|---------|--------|
| Interaction | Bleu | Date de l'interaction. |
| Activité | Violet | Date de l'activité. |
| Clôture opp. | Orange | Date de clôture prévue d'une opportunité. |

Chaque **cellule-jour** affiche jusqu'à 3 événements, puis « +N autres ». Cliquer sur un jour ouvre un **panneau latéral** listant tous les événements de ce jour (type + titre + badge de sous-type).

### 2.8 Onglet Historique

Fil chronologique de **toutes** les interactions et activités du locataire (source `GET /crm/timeline?limit=50`, max 200). Un **filtre par entreprise** (menu déroulant) permet de restreindre l'affichage.

Chaque carte : icône selon le type/sous-type, badge Interaction/Activité, sous-type, entreprise, titre, date. Un compteur total figure en tête. Si vide : « Aucun événement dans l'historique ».

> Les interactions et activités se **saisissent** uniquement depuis le détail d'une opportunité (2.5). L'Historique et le Calendrier sont des vues de consultation.

### 2.9 Onglet Qualification et grille B.A.T.

Deux mécanismes de qualification cohabitent : le **pointage automatique** (cet onglet) et la **grille B.A.T. manuelle** (intégrée aux modales de détail).

**Pointage automatique** (source `GET /crm/qualification`) — score de 0 à 100 calculé à la volée sur les opportunités **ouvertes** (ni GAGNE ni PERDU) :

**Trois cartes-résumé** en tête, avec compteur : **Chaud** (HOT, rouge), **Tiède** (WARM, orange), **Froid** (COLD, bleu).

**Tableau** : Opportunité · Score (barre) · Catégorie (badge) · Montant · Probabilité · **Détails** (puces expliquant le score : montant, entreprise liée, contact lié, probabilité, interactions, source, récent, inactif). Cliquer une ligne ouvre l'opportunité. Si vide : « Aucune opportunité ouverte à qualifier ».

**Grille de Pointage B.A.T. (qualification manuelle).** Composant intégré dans les modales de détail (Pipeline et Opportunités). Score sur 100, réparti en **4 sections repliables** de 25 points chacune, **13 questions** au total (boutons radio) :

| Section | Points | Questions | Icône |
|---------|--------|-----------|-------|
| **A. Budget** | 25 | A1 (10), A2 (10), A3 (5) | `DollarSign` |
| **B. Autorité** | 25 | B1 (10), B2 (10), B3 (5) | `Users` |
| **C. Timing** | 25 | C1 (10), C2 (10), C3 (5) | `Clock` |
| **D. Compatibilité** | 25 | D1 (10), D2 (5), D3 (5), D4 (5) | `Target` |

Le score total donne une **catégorie** et une **action recommandée** (voir 4.7). Un champ **Notes de qualification** (optionnel) et le bouton **Enregistrer la qualification** (→ `POST /crm/qualification/bat`) finalisent la saisie.

> **Le serveur recalcule** le score total et la catégorie à partir des réponses envoyées ; il ne fait pas confiance aux valeurs calculées par le navigateur. Les scores agrégés affichés sur les cartes du Kanban proviennent de `GET /crm/qualification/bat/all`.

### 2.10 Onglet Assistant IA — Ventes

Clavardage dédié. Titre « Assistant IA — Ventes », sous-titre « Analyse ton pipeline et crée des opportunités sur confirmation. ». Il utilise `api/ventesAi.ts` (`POST /ventes/ai/chat` et `POST /ventes/ai/confirm-action`).

**Fonctionnement en deux temps** :

1. Vous posez une question ou une demande. L'IA **lit vos données réelles** (opportunités, entreprises, contacts) et répond. Si elle propose de créer une opportunité, elle affiche une **carte de proposition** (titre + aperçu des champs) avec la mention « En attente de confirmation ».
2. Vous cliquez **Confirmer** (l'opportunité est réellement créée) ou **Annuler** (rien ne se passe). **Seul le clic Confirmer écrit en base.**

Trois exemples d'amorce sont proposés. La saisie envoie avec **Entrée** ; des verrous empêchent le double-envoi. Chaque message affiche ses métadonnées (profil « Ventes », jetons, coût en USD, durée).

> L'assistant Ventes **ne crée que des opportunités** — aucune autre action. Il applique les mêmes droits que le CRM : la création confirmée revérifie `require_crm_write` côté serveur, donc un `employee`/`comptable` ne peut pas créer d'opportunité même via l'IA.

### 2.11 Onglet B2B/B2C (administrateurs seulement)

Titre « Espace B2B ». C'est le **back-office** du portail client externe (`/b2b-portal`). Il comporte **10 sous-onglets** :

#### a) Tableau de bord

4 indicateurs (Clients actifs, Demandes nouvelles, Contrats actifs, Valeur contrats) + « Demandes par statut » + « Résumé » (totaux) + « Activité récente » (source `GET /b2b/stats`). Le compteur de « messages non lus » compte les messages **écrits par un client** (non lus).

#### b) Demandes d'accès

Flux d'approbation des inscriptions au portail. Deux sous-onglets, **En attente** / **Approuvés** (avec compteurs). Tableau : Entreprise · Contact · Courriel · Téléphone · Ville · Date · Actions.

- Pour un compte **en attente** : **Approuver** (active le compte et réactive l'entreprise cliente) ou **Rejeter** (supprime la demande).
- Pour un compte **approuvé** : **Désactiver** (révoque l'accès).

Endpoints : `PUT /b2b/client-users/{id}/approve | reject | deactivate`. Chaque action demande une confirmation.

#### c) Clients

Recherche + bouton **Nouveau client**. Tableau : Nom · Contact · Courriel · Téléphone · **Entreprise CRM** · Statut · Actions.

- **Lier à une entreprise CRM** (icône `Link2`) : ouvre une modale de recherche d'entreprise. Ce lien alimente le **suivi du portail** (le client voit ses devis/projets). Le lien est **toujours posé manuellement** par l'administrateur — **jamais** d'auto-liaison par courriel (protection contre l'usurpation).
- **Désactiver** (corbeille) → `DELETE /b2b/clients/{id}` (désactivation logique en cascade sur les accès).

Modale de création : Nom entreprise * · Nom contact · Courriel · Téléphone · Ville · Secteur (→ `POST /b2b/clients`).

#### d) Demandes

Filtre de statut (Tous / Nouvelle / En cours / Soumise / Acceptée / Refusée / Annulée) + bouton **Nouvelle demande**. Tableau : Titre · Client · Budget · Statut · Date. Le clic ouvre un **panneau de détail** (client, catégorie, budget, priorité, chantier, description) avec un bouton **Créer soumission** et la liste des soumissions liées.

Modale de création : Client * · Titre * · Description · Catégorie · Budget estimé · Date limite · Priorité · Adresse/Ville du chantier (→ `POST /b2b/demandes`).
Modale de soumission : **Montant HT** * · Description · Délai (jours) · Validité (jours) · Conditions de paiement · Garanties (→ `POST /b2b/soumissions`).

#### e) Soumissions

Filtre de statut (Tous / Brouillon / Soumise / En évaluation / Acceptée / Refusée). Tableau : Demande · Client · Montant · Délai · Statut · Actions.

- **Accepter** (✓) → `PUT /b2b/soumissions/{id}/accepter` : marque la soumission acceptée, **refuse automatiquement les autres soumissions** de la même demande, et **crée un contrat** actif.
- **Refuser** (✗) → `PUT /b2b/soumissions/{id}/refuser`.

Ces actions sont indisponibles si la soumission est déjà Acceptée / Refusée / Expirée.

#### f) Contrats

Filtre de statut (Tous / Brouillon / Actif / Terminé / Annulé). Tableau : Numéro · Titre · Client · Montant · **Avancement** (barre) · Statut · Actions. Le bouton **Modifier** ouvre une modale : Statut · Avancement (%) · Montant payé · Notes internes (→ `PUT /b2b/contrats/{id}`).

#### g) Commandes

Filtre de statut (7 valeurs). **Vue liste** : Numéro · Total TTC · Ville · Statut · Date · Action. **Vue détail** : lignes de produits, sous-total, TPS, TVQ, Total TTC.

- **Avancer au statut suivant** : EN_ATTENTE → CONFIRMEE → EN_PREPARATION → EXPEDIEE → LIVREE (`PUT /b2b/commandes/{id}/statut`).
- **Annuler** : demande une confirmation (« stock réservé réapprovisionné »). Passer une commande à ANNULEE **restitue le stock** ; ANNULEE est un statut **terminal**.

> Les commandes se **créent** depuis le portail client externe, pas ici. Ce sous-onglet ne fait que suivre et faire avancer les commandes existantes.

#### h) Messages

Deux colonnes : la liste des demandes à gauche, le fil de conversation à droite. Les bulles distinguent **Vous** (le fournisseur) et **Client**. La saisie envoie avec **Entrée** (→ `POST /b2b/messages`) ; les messages sont marqués lus dès l'ouverture du fil.

#### i) Catalogue

**Consultatif uniquement** (l'ancien panier administrateur a été retiré). Recherche + filtre par catégorie. Grille de cartes-produit : nom, code, description, catégorie, prix/unité, stock (« X en stock » / « Rupture de stock »). Source `GET /b2b/catalogue`.

#### j) Assistant IA — Gestion B2B

Clavardage en **lecture seule** (aucune écriture, aucune proposition). Titre « Assistant IA — Gestion B2B ». Il utilise `api/b2bAi.ts`. Il **n'accède pas** aux comptes clients (mots de passe) ni au **contenu des messages**. Trois exemples d'amorce sont proposés.

---

## 3. Workflows pas à pas

### 3.1 Créer une opportunité

1. Onglet **Pipeline** ou **Opportunités** → **Nouvelle opportunité** (ou **Ajouter** en pied d'une colonne pour fixer le statut).
2. Saisir au minimum le **Nom** (obligatoire). Renseigner l'entreprise/contact, le montant estimé, la probabilité, la source et la priorité.
3. **Enregistrer**. Le système crée l'opportunité `OPP-00001`, ainsi qu'un **dossier client** `DOS-OPP-…` (au mieux).

### 3.2 Faire avancer une affaire dans le pipeline

**Par glisser-déposer** : dans le Kanban, tirez la carte vers une autre colonne (nouveau statut) ou vers les cartes-résumé Gagné/Perdu. La carte se déplace tout de suite ; si l'enregistrement échoue, elle revient à sa place.

**Par bouton rapide** : sur la carte, cliquez « Avancer vers <statut> », « Gagné » ou « Perdu ».

Les transitions sont **libres** : n'importe quel statut peut mener à n'importe quel autre. Aucune règle n'oblige à qualifier avant d'avancer.

### 3.3 Réordonner les affaires d'une colonne

Dans le Kanban, glissez une carte **au-dessus/en dessous** d'une autre **dans la même colonne**. Le nouvel ordre est enregistré (`PUT /crm/opportunities/reorder`) et sert de priorité d'affichage.

### 3.4 Qualifier une affaire

**Automatique** : ouvrez l'onglet **Qualification**. Le score (Chaud/Tiède/Froid) et ses raisons se calculent seuls à partir des données déjà saisies.

**Détaillée (B.A.T.)** : ouvrez le détail d'une opportunité, déroulez la **Grille de Pointage B.A.T.**, répondez aux 13 questions (4 sections), ajoutez des notes, **Enregistrer la qualification**. Le score B.A.T. et sa catégorie A+/A/B/C/D remontent ensuite sur la carte du Kanban.

### 3.5 Journaliser une interaction ou une activité

1. Ouvrez le détail de l'opportunité concernée.
2. Dans le **fil**, cliquez **Interaction** (événement passé : appel, courriel...) ou **Activité** (tâche planifiée : relance, visite...).
3. Choisissez le **Type**, la **Date**, saisissez le **Résumé/Sujet**, puis **Ajouter**.

Une **activité** est créée au statut PLANIFIE et alimente les **Relances** et le **Calendrier**.

### 3.6 Traiter ses relances

1. Onglet **Relances** (le badge rouge indique le nombre en retard + aujourd'hui).
2. Pour chaque carte : **Fait** (terminé), **Reporter** (choisir une nouvelle date), ou **Ouvrir** (aller à l'opportunité).

### 3.7 Convertir une opportunité en soumission (devis)

1. Ouvrez le détail de l'opportunité (Pipeline ou Opportunités).
2. Cliquez **Créer une soumission** (→ `POST /crm/opportunities/{id}/create-devis`).
3. Vous êtes redirigé vers le devis créé (`/devis?open=<id>`), au statut **Brouillon**, type **Détaillée**.

Ce que fait le serveur (`create_devis_from_opportunity`, `crm.py:1232`) :

- il **verrouille** l'opportunité (`FOR UPDATE`) : deux clics simultanés ne créent qu'un seul devis ; si un devis existe déjà, il répond **400 « déjà convertie en devis #X »** ;
- il applique la cascade **Administration 3 % / Contingences 12 % / Profit 15 %** sur le montant estimé (le profit 15 % est **fixe**, cohérent avec le modèle cost-plus de l'ERP) ;
- il calcule les **taxes selon la configuration du locataire** (`resolve_document_tax_config`) — au Québec, TPS 5 % et TVQ 9,975 %, mais un locataire configuré ailleurs aura ses propres taux ;
- il numérote le devis **`DEV-{année}-{id:03d}`**, y sème une ligne d'estimation (quantité 1, unité « global ») si le montant est positif ;
- il fait passer l'opportunité au statut **PROPOSITION** et la relie au devis, puis lie le dossier au devis.

> La conversion crée un **devis**, pas un projet. Le passage devis → projet se fait dans le module Soumissions/Projets à l'acceptation.

### 3.8 Supprimer une opportunité

1. Détail de l'opportunité → **Supprimer** → confirmez.
2. Le serveur supprime en cascade les **interactions, activités, assignations, qualifications** ; il **détache** (met à NULL) le devis, le projet et les courriels liés (l'historique est préservé) ; il supprime le dossier auto-créé **seulement s'il est vide**.

### 3.9 Utiliser l'assistant IA Ventes

1. Onglet **Assistant IA**. Posez votre demande (ex. « Crée une opportunité pour la rénovation du 12 rue Principale, budget 80 000 $ »).
2. L'IA lit vos données et, si pertinent, affiche une **carte de proposition**.
3. Vérifiez les champs, puis **Confirmer** (création réelle) ou **Annuler**.

### 3.10 B2B — Approuver un accès au portail

1. Un client s'inscrit sur le **portail externe** (`/b2b-portal`) ; son compte apparaît dans **B2B → Demandes d'accès → En attente**.
2. **Approuver** : le compte devient actif et l'entreprise cliente est réactivée. (Ou **Rejeter** pour supprimer la demande.)
3. Optionnel mais recommandé : dans **Clients**, **liez** le client à une **entreprise CRM** pour qu'il suive ses devis/projets dans le portail.

### 3.11 B2B — De la demande au contrat

1. **Demandes** → **Nouvelle demande** (client, titre, budget, chantier...).
2. Dans le détail de la demande → **Créer soumission** (Montant HT, délai, validité...). La demande passe de NOUVELLE à EN_COURS.
3. **Soumissions** → **Accepter** : la soumission passe à ACCEPTEE, les **autres soumissions** de la demande sont refusées, et un **contrat** actif `CTR-AAAAMM-0001` est généré.
4. **Contrats** → **Modifier** pour suivre l'avancement (%) et les montants payés.

### 3.12 B2B — Suivre et annuler une commande

1. **Commandes** : ouvrez la commande (créée via le portail).
2. **Avancer** au statut suivant (EN_ATTENTE → ... → LIVREE) au fur et à mesure.
3. **Annuler** si nécessaire : le **stock réservé est réapprovisionné** et la commande devient ANNULEE (définitif).

---

## 4. Référence

### 4.1 Endpoints CRM (`/api/erp/v1/crm`)

| Méthode | Chemin | Garde | Référence |
|---------|--------|-------|-----------|
| GET | `/crm/opportunities` | lecture | `crm.py:331` |
| GET | `/crm/opportunities/{id}` | lecture | `crm.py:442` |
| POST | `/crm/opportunities` | `require_crm_write` | `crm.py:526` |
| PUT | `/crm/opportunities/reorder` | `require_crm_write` | `crm.py:719` |
| PUT | `/crm/opportunities/{id}` | `require_crm_write` | `crm.py:787` |
| DELETE | `/crm/opportunities/{id}` | `require_crm_write` | `crm.py:876` |
| POST | `/crm/opportunities/{id}/create-devis` | `require_crm_write` | `crm.py:1232` |
| GET | `/crm/opportunities/{id}/assignations` | lecture | `crm.py:2433` |
| POST | `/crm/opportunities/{id}/assignations` | `require_crm_write` | `crm.py:2474` |
| DELETE | `/crm/opportunities/{id}/assignations/{aid}` | `require_crm_write` | `crm.py:2534` |
| GET | `/crm/interactions` | lecture | `crm.py:1037` |
| POST | `/crm/interactions` | `require_crm_write` | `crm.py:1117` |
| GET | `/crm/activities` | lecture | `crm.py:1525` |
| POST | `/crm/activities` | `require_crm_write` | `crm.py:1582` |
| PATCH | `/crm/activities/{id}` | `require_crm_write` | `crm.py:1644` |
| GET | `/crm/pipeline` | lecture | `crm.py:1178` |
| GET | `/crm/stats` | lecture | `crm.py:1416` |
| GET | `/crm/relances` | lecture | `crm.py:1718` |
| GET | `/crm/calendar` | lecture | `crm.py:1824` |
| GET | `/crm/timeline` | lecture | `crm.py:1928` |
| GET | `/crm/qualification` | lecture | `crm.py:2023` |
| GET | `/crm/qualification/bat/all` | lecture | `crm.py:2213` |
| GET | `/crm/qualification/bat/{id}` | lecture | `crm.py:2259` |
| POST | `/crm/qualification/bat` | `require_crm_write` | `crm.py:2298` |

### 4.2 Endpoints Assistant IA Ventes (`/api/erp/v1/ventes/ai`)

| Méthode | Chemin | Effet | Référence |
|---------|--------|-------|-----------|
| POST | `/ventes/ai/chat` | Lit les données, propose (ne crée rien) | `ventes_ai.py:302` |
| POST | `/ventes/ai/confirm-action` | Crée l'opportunité confirmée (revérifie `require_crm_write`) | `ventes_ai.py:442` |

### 4.3 Endpoints Back-office B2B (`/api/erp/v1/b2b`) — principaux

| Méthode | Chemin | Garde | Référence |
|---------|--------|-------|-----------|
| GET | `/b2b/stats` | lecture | `b2b.py:722` |
| GET | `/b2b/clients` · `/b2b/clients/{id}` | lecture | `b2b.py:821` · `881` |
| POST | `/b2b/clients` | admin | `b2b.py:922` |
| PUT | `/b2b/clients/{id}` | admin | `b2b.py:965` |
| DELETE | `/b2b/clients/{id}` | admin | `b2b.py:1027` |
| POST | `/b2b/client-users` | admin | `b2b.py:1111` |
| GET | `/b2b/client-users` | lecture | `b2b.py:1166` |
| PUT | `/b2b/client-users/{id}/approve` | admin | `b2b.py:1213` |
| PUT | `/b2b/client-users/{id}/reject` | admin | `b2b.py:1308` |
| PUT | `/b2b/client-users/{id}/deactivate` | admin | `b2b.py:1370` |
| GET | `/b2b/demandes` · `/b2b/demandes/{id}` | lecture | `b2b.py:1435` · `1504` |
| POST | `/b2b/demandes` | admin | `b2b.py:1558` |
| PUT | `/b2b/demandes/{id}` | admin | `b2b.py:1608` |
| GET | `/b2b/soumissions` | lecture | `b2b.py:1670` |
| POST | `/b2b/soumissions` | admin | `b2b.py:1731` |
| PUT | `/b2b/soumissions/{id}` | admin | `b2b.py:1808` |
| PUT | `/b2b/soumissions/{id}/accepter` | admin | `b2b.py:1866` |
| PUT | `/b2b/soumissions/{id}/refuser` | admin | `b2b.py:1978` |
| GET | `/b2b/contrats` · `/b2b/contrats/{id}` | lecture | `b2b.py:2028` · `2084` |
| PUT | `/b2b/contrats/{id}` | admin | `b2b.py:2128` |
| GET | `/b2b/commandes` · `/b2b/commandes/{id}` | lecture | `b2b.py:2190` · `2247` |
| PUT | `/b2b/commandes/{id}/statut` | admin | `b2b.py:2290` |
| GET | `/b2b/catalogue` | lecture | `b2b.py:2451` |
| GET/POST/DELETE | `/b2b/favoris[/{produit_id}]` | lecture | `b2b.py:2544` · `2586` · `2635` |
| GET | `/b2b/messages` | lecture | `b2b.py:2679` |
| POST | `/b2b/messages` | lecture (get_current_user) | `b2b.py:2730` |
| PUT | `/b2b/messages/read` | lecture | `b2b.py:2773` |
| GET | `/b2b/notifications` | lecture | `b2b.py:2829` |
| PUT | `/b2b/notifications/{id}/read` | lecture | `b2b.py:2877` |
| GET | `/b2b/categories` | **PUBLIC (aucune auth)** | `b2b.py:3014` |
| POST | `/b2b/ai/chat` | lecture (assistant B2B) | `b2b_ai.py:221` |

> `GET /b2b/categories` est le **seul endpoint non authentifié** du module : il renvoie un dictionnaire statique de ~140 catégories de construction du Québec, sans aucune donnée de locataire.

### 4.4 Statuts d'opportunité (`OPPORTUNITY_STATUSES`, `crm.py:51`)

| Valeur (base) | Libellé affiché | Couleur | Colonne Kanban ? |
|---------------|-----------------|---------|------------------|
| PROSPECTION | Prospection | Bleu | Colonne 1 |
| QUALIFICATION | Qualification | Jaune | Colonne 2 |
| PROPOSITION | Proposition | Violet | Colonne 3 |
| NEGOCIATION | Négociation | Orange | Colonne 4 |
| GAGNE | Gagné | Vert | Carte-résumé (cible de dépôt) |
| PERDU | Perdu | Rouge | Carte-résumé (cible de dépôt) |

Les valeurs sont stockées en majuscules ASCII ; une migration idempotente (`_ensure_opportunities_statut_check`) resynchronise la contrainte des locataires plus anciens. Les libellés affichés proviennent de `crm.json` (namespace `ventes.statusLabels.*`).

### 4.5 Types d'interaction / d'activité

| Constante (`crm.py`) | Valeurs |
|----------------------|---------|
| `INTERACTION_TYPES` (`:52`) | APPEL, EMAIL, REUNION, VISITE, NOTE |
| `ACTIVITY_TYPES` (`:271`) | **TACHE**, APPEL, EMAIL, REUNION, VISITE, RELANCE, NOTE |
| `ACTIVITY_STATUSES` (`:264`) | PLANIFIE, TERMINE, ANNULE |

> `TACHE` est une valeur d'activité **valide** (`crm.py:271`). L'ancien manuel signalait un bogue « TACHE rejeté » : il est **corrigé**.

### 4.6 Priorités d'opportunité

Basse · Normale · Haute · Urgente (libellés `crm.json`). Défaut : Normale.

### 4.7 Barèmes de qualification

**Pointage automatique** (`GET /crm/qualification`, sur opportunités ouvertes) :

| Critère | Points |
|---------|--------|
| `montant_estime > 0` | +20 |
| Entreprise renseignée | +15 |
| Contact renseigné | +10 |
| Probabilité > 50 | +20 |
| Au moins 1 interaction | +15 |
| Source renseignée | +10 |
| Mise à jour < 30 jours | +10 |

Catégories : **HOT** ≥ 70 · **WARM** ≥ 40 · **COLD** en dessous.

**Grille B.A.T. manuelle** (le serveur recalcule le total et la catégorie) :

| Score total | Catégorie | Couleur | Action recommandée |
|-------------|-----------|---------|--------------------|
| ≥ 90 | A+ | Vert | Priorité maximale — visite 48-72 h |
| 75-89 | A | Vert | Priorité haute |
| 50-74 | B | Jaune | Potentiel — à approfondir |
| 25-49 | C | Orange | Tiède — maintenir le contact |
| < 25 | D | Gris | Froid — pas prioritaire |

> Deux systèmes **distincts** : le pointage automatique (3 niveaux HOT/WARM/COLD) et la grille B.A.T. (5 catégories A+/A/B/C/D). Ils peuvent diverger.

### 4.8 Conversion en devis — cascade et taxes

| Élément | Valeur |
|---------|--------|
| Administration | montant × 3 % |
| Contingences | montant × 12 % |
| Profit | montant × 15 % (**fixe**, modèle cost-plus) |
| Taxes | Selon la configuration du locataire (`resolve_document_tax_config`) — QC : TPS 5 %, TVQ 9,975 % |
| Montant borné | `max(0, min(montant, 1 000 000 000))` |
| Devis créé | statut Brouillon, type Détaillée, numéro `DEV-{année}-{id:03d}`, 1 ligne d'estimation initiale |
| Effet sur l'opportunité | passe à PROPOSITION + lien vers le devis |
| Protection | `FOR UPDATE` + revérification → 400 si déjà convertie |

### 4.9 Numérotation automatique

| Entité | Format | Exemple |
|--------|--------|---------|
| Opportunité | `OPP-{id:05d}` | OPP-00042 |
| Dossier auto-créé | `DOS-OPP-…` | DOS-OPP-00042 |
| Devis converti | `DEV-{année}-{id:03d}` | DEV-2026-137 |
| Contrat B2B | `CTR-{AAAAMM}-{id:04d}` | CTR-202607-0009 |

Tous les numéros sont générés par INSERT-RETURNING-id (jamais par COUNT+1), garantissant l'unicité même en cas de clics simultanés.

### 4.10 Statuts B2B

| Constante (`b2b.py`) | Valeurs |
|----------------------|---------|
| `DEMANDE_STATUTS` (`:31`) | NOUVELLE, EN_COURS, SOUMISE, ACCEPTEE, REFUSEE, ANNULEE |
| `SOUMISSION_STATUTS` (`:34`) | BROUILLON, SOUMISE, EN_EVALUATION, ACCEPTEE, REFUSEE, EXPIREE |
| `CONTRAT_STATUTS` (`:32`) | BROUILLON, ACTIF, EN_COURS, TERMINE, ANNULE, SUSPENDU |
| `COMMANDE_STATUTS` (`:33`) | EN_ATTENTE, CONFIRMEE, EN_PREPARATION, EXPEDIEE, LIVREE, ANNULEE |

> La table `b2b_commandes` est **partagée** avec l'ancienne application Streamlit (statuts en minuscules). Le module gère la casse de façon transparente (`upper(...)` et réparation de la contrainte CHECK). Toute évolution des statuts doit rester compatible avec les deux applications.

### 4.11 Taxes B2B

TPS 5 % (`TPS_RATE = 0.05`) et TVQ 9,975 % (`TVQ_RATE = 0.09975`), en dur dans `b2b.py:28-29`. Pour une soumission : si le **Montant HT** est fourni, TPS et TVQ s'ajoutent dessus ; sinon le HT est déduit du TTC. Date d'expiration = date du jour + validité (défaut 30 jours).

### 4.12 Limites, bornes et défenses

| Élément | Valeur |
|---------|--------|
| Pagination opportunités | `page` ≥ 1, `per_page` 1-200 (20 par défaut à l'écran) |
| Recherche | LIKE avec échappement `% _ \`, tronquée à 100 caractères |
| Montant estimé | 0 à 1 000 000 000 |
| Probabilité | 0 à 100 |
| Réordonnancement | jusqu'à 10 000 identifiants par appel |
| Grille B.A.T. | axes 0-25, total 0-100, notes ≤ 10 000 caractères |
| Relances | horizon 0-90 jours (7 par défaut) |
| Calendrier | année 1900-2200, mois 1-12 |
| Historique (timeline) | limite ≤ 200 |
| Assignation | UNIQUE (opportunité, employé) → 409 si doublon |

### 4.13 Assistant IA — modèle, coût, limites de débit

| Élément | Ventes | B2B |
|---------|--------|-----|
| Modèle | `claude-sonnet-4-6` | `claude-sonnet-4-6` |
| Écriture | Oui, opportunité **sur confirmation** | **Non** (lecture seule) |
| Outils de lecture | `recherche_bd` — tables `{opportunities, companies, contacts}` | tables B2B (hors comptes/messages) + `projects`, `companies` |
| Coût | (entrée × 0,003 + sortie × 0,015) / 1000 × **1,30** (marge 30 %) | idem |
| Débité de | `public.ai_prepaid_credits` (recharge auto Stripe 10 $ sous le seuil) | idem |
| Limite de débit (par IP) | clavardage 20/min, confirmation 30/min | clavardage 20/min |

Le vrai contrôle de crédit est `_check_credits` (fail-closed : bloque en cas d'erreur). Note technique : le clavardage Ventes débite **sans clé d'idempotence** — un retour réseau/réessai peut redébiter (mineur, il s'agit d'un clavardage, pas d'une mutation d'argent).

### 4.14 Raccourcis

| Action | Geste |
|--------|-------|
| Ouvrir le détail d'une carte | Double-clic (ou bouton œil) |
| Changer de statut | Glisser la carte vers une autre colonne / carte-résumé |
| Réordonner | Glisser la carte dans la même colonne |
| Envoyer un message au clavardage | Entrée |
| Ouvrir le B2B en direct | `/ventes?tab=b2b` (admin) |
| Ouvrir une opportunité en direct | `/ventes?open=<id>` |

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|--------|------|
| **04 — Entreprises** | Les opportunités et clients B2B se rattachent à une entreprise. La création/gestion des entreprises vit dans son propre module. |
| **05 — Contacts** | Le champ « Client (Personne) » d'une opportunité pointe vers un contact. Gestion des contacts : module 05. |
| **07 — Dossiers** | Chaque opportunité génère un dossier `DOS-OPP-…` ; la conversion lie le dossier au devis. |
| **08 — Soumissions (devis)** | « Créer une soumission » produit un devis brouillon `DEV-…` et redirige vers son éditeur. |
| **09 — Projets** | Un projet peut référencer l'opportunité (`projects.opportunity_id`). La promotion devis → projet se fait côté Soumissions/Projets. |
| **10 — Magasin / inventaire** | Le catalogue B2B lit les `produits` ; annuler une commande B2B **restitue le stock**. |
| **Portail B2B externe** (`/b2b-portal`) | Les clients s'y inscrivent, envoient des demandes et **passent les commandes**. Le back-office B2B de ce module en est la contrepartie interne. |
| **25 — Assistant IA** | Les assistants Ventes et B2B partagent le même moteur de crédits IA (`ai_prepaid_credits`) et la facturation Stripe. |
| **28 — Configuration** | La configuration de taxes du locataire pilote les taxes de la conversion en devis. L'état de l'abonnement Stripe pilote le mode consultation. |

### 5.2 FAQ

**Q1. Pourquoi je ne vois pas l'onglet B2B/B2C ?**
Il est réservé aux administrateurs (rôle `admin`, `is_admin`, ou super-administrateur). Un compte au rôle `user` a accès à tout le CRM mais pas au back-office B2B.

**Q2. Un employé peut-il travailler dans le pipeline ?**
Oui si son rôle est `user`, `gestionnaire`, `contremaitre`, `admin` ou super-administrateur. Les rôles `employee` et `comptable` sont en **lecture seule** côté CRM (toute écriture renvoie 403).

**Q3. Pourquoi le pipeline n'a-t-il que 4 colonnes alors qu'il y a 6 statuts ?**
Gagné et Perdu ne sont pas des colonnes, mais deux **cartes-résumé** en haut (montant + nombre), qui servent aussi de cibles de dépôt. Les 4 colonnes couvrent le travail « en cours ».

**Q4. La conversion crée-t-elle un projet ?**
Non : elle crée un **devis** au statut Brouillon. Le processus est Opportunité → Devis → Acceptation → Projet ; la dernière étape relève du module Soumissions/Projets.

**Q5. Les marges 3 / 12 / 15 % sont-elles configurables ?**
Le **profit 15 % est fixe** (modèle cost-plus de l'ERP), tout comme Administration 3 % et Contingences 12 %. Pour ajuster un cas particulier, modifiez le devis après sa création (module Soumissions).

**Q6. Les taxes de la conversion sont-elles toujours 5 % / 9,975 % ?**
Elles suivent la **configuration du locataire** (`resolve_document_tax_config`). Au Québec, ce sont bien TPS 5 % et TVQ 9,975 %, mais un locataire configuré ailleurs (ex. autre province, États-Unis) aura ses propres taux. C'est une amélioration par rapport à l'ancien taux QC en dur.

**Q7. Puis-je exporter le pipeline en PDF ou CSV ?**
Non. Le module Ventes/B2B ne propose **aucun export, aucune impression, aucun téléversement**. Les documents se produisent depuis Soumissions/Dossiers.

**Q8. Peut-on assigner un employé à une opportunité ?**
Pas depuis l'interface : les fonctions existent côté API (`crm.ts:353-365`, endpoints `crm.py:2433/2474/2534`) mais **ne sont branchées à aucun écran** du module. Il n'y a donc pas d'interface d'assignation.

**Q9. L'assistant IA peut-il modifier ou supprimer des données ?**
Non. L'assistant **Ventes** ne peut que **proposer une opportunité**, créée seulement après votre confirmation (et il revérifie vos droits d'écriture côté serveur). L'assistant **B2B** est en **lecture seule** et n'accède ni aux comptes clients ni au contenu des messages.

**Q10. Que se passe-t-il si je double-clique sur « Créer une soumission » ?**
Rien de dangereux : l'opportunité est verrouillée (`FOR UPDATE`) le temps de la conversion, donc un seul devis est créé. Le second clic reçoit « déjà convertie en devis #X ».

**Q11. Différence entre une interaction et une activité ?**
Une **interaction** est un événement **passé** (appel reçu, courriel envoyé). Une **activité** est une **tâche planifiée** (relance, visite, tâche), avec un statut PLANIFIE/TERMINE/ANNULE ; ce sont les activités qui alimentent les Relances et le Calendrier.

**Q12. La grille B.A.T. est-elle obligatoire pour avancer une affaire ?**
Non. Aucune règle ne lie le statut au score. La qualification est un outil d'aide à la décision, pas un verrou.

**Q13. Pourquoi le score B.A.T. que j'ai calculé diffère parfois de celui affiché ?**
Le **serveur recalcule** toujours le total et la catégorie à partir de vos réponses ; c'est cette valeur qui fait foi.

**Q14. Comment un client obtient-il un accès au portail ?**
Il s'inscrit sur le portail externe ; sa demande apparaît dans **B2B → Demandes d'accès → En attente**. Un administrateur l'**approuve** (ce qui active le compte et l'entreprise) ou le **rejette**.

**Q15. Pourquoi lier un client B2B à une entreprise CRM ?**
Le lien permet au client de **suivre ses devis et projets** dans le portail. Il est **toujours posé manuellement** par un administrateur — jamais automatiquement par courriel, pour éviter toute usurpation.

**Q16. Pourquoi le catalogue B2B ne permet-il pas de commander ?**
Le panier administrateur a été **retiré** ; le catalogue est désormais consultatif. Les commandes se créent côté client, dans le portail externe. L'administrateur les **suit** et les **fait avancer** (ou annule) depuis le sous-onglet Commandes.

**Q17. Qu'arrive-t-il au stock quand j'annule une commande B2B ?**
Le stock réservé est **réapprovisionné** (mouvement d'entrée, motif ANNULATION), et la commande passe à ANNULEE, qui est un statut **terminal**.

**Q18. Mon compte est en « Mode consultation » — pourquoi ?**
L'abonnement Stripe du locataire n'est pas à jour. En readonly, vous pouvez tout consulter mais aucune écriture n'est acceptée. Régularisez l'abonnement (module Configuration) pour revenir en écriture.

**Q19. L'assistant IA me facture-t-il ?**
Chaque échange consomme des crédits IA prépayés (coût réel du modèle + marge 30 %), débités de `ai_prepaid_credits`. Une recharge Stripe automatique de 10 $ se déclenche sous le seuil. Les super-administrateurs et les comptes exemptés ne sont pas débités.

**Q20. Où sont gérés les entreprises et les contacts ?**
Dans leurs modules dédiés : **04 — Entreprises** et **05 — Contacts**. Ce module ne fait que les **référencer** dans les menus déroulants des opportunités et clients B2B.

---

## 6. Récapitulatif

- **Un écran, deux périmètres** : le CRM (pipeline, relances, calendrier, historique, qualification, assistant IA) pour tous, et le **back-office B2B** (10 sous-onglets) réservé aux administrateurs.
- **8 onglets** : Pipeline · Relances · Opportunités · Calendrier · Historique · Qualification · Assistant IA · B2B/B2C.
- **6 statuts d'opportunité**, mais **4 colonnes** Kanban glissables ; Gagné/Perdu sont des cartes-résumé et cibles de dépôt.
- **Trois niveaux de permission** : lecture (tout compte ERP), écriture CRM (`require_crm_write`, inclut le rôle `user`), écriture B2B (`require_tenant_admin_or_role`, administrateurs seulement). Le **mode consultation** Stripe peut mettre tout le module en lecture seule.
- **Deux qualifications** : pointage automatique (HOT/WARM/COLD) et grille B.A.T. manuelle (A+/A/B/C/D, recalculée côté serveur).
- **Conversion en devis** en un clic : cascade 3 % / 12 % / **profit 15 % fixe** + taxes **selon la configuration du locataire**, devis `DEV-{année}-{id:03d}` brouillon, opportunité → PROPOSITION, verrouillage anti double-conversion.
- **Numérotation** : `OPP-{id:05d}`, `DOS-OPP-…`, `DEV-{année}-{id:03d}`, `CTR-{AAAAMM}-{id:04d}`.
- **Deux assistants IA** : Ventes (propose une opportunité sur confirmation) et B2B (lecture seule). Modèle `claude-sonnet-4-6`, crédits prépayés + marge 30 %.
- **B2B** : approbation des accès portail, demande → soumission → contrat (le contrat est créé à l'acceptation), suivi des commandes (annulation = restitution du stock), messagerie, catalogue consultatif.
- **Ce que le module ne fait pas** : aucun export/impression/téléversement, aucune création de projet, aucun courriel automatique, aucun calcul de commission, aucune assignation d'employé (surface API morte), aucune création de commande côté administrateur.
- **Endpoint public unique** : `GET /b2b/categories` (dictionnaire statique de catégories, aucune donnée de locataire).

---

*Sources vérifiées (2026-07)* : `backend/routers/crm.py` (2563 lignes) · `backend/routers/ventes_ai.py` (479 lignes) · `backend/routers/b2b.py` (3017 lignes) · `backend/routers/b2b_ai.py` (340 lignes) · `frontend/src/pages/VentesPage.tsx` (2914 lignes) · `frontend/src/pages/B2bPage.tsx` (1411 lignes) · `frontend/src/components/crm/BATQualificationForm.tsx` · `frontend/src/components/ventes/VentesAssistantTab.tsx` · `frontend/src/components/b2b/B2bAssistantTab.tsx` · `frontend/src/api/{crm.ts, ventesAi.ts, b2b.ts, b2bAi.ts}` · i18n `crm.json` (namespace `ventes.*`), `b2b.json`, `b2bAssistant.json`.

*Manuels liés* : 04 — Entreprises · 05 — Contacts · 07 — Dossiers · 08 — Soumissions · 09 — Projets · 10 — Magasin · 25 — Assistant IA · 28 — Configuration.

*Manuel ERP Constructo AI — Module 06 Ventes (CRM, opportunités, pipeline, B2B back-office) — v3.0 vérifié — 2026-07*
