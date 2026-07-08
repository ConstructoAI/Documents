# Module 02 — Suivi (Kanban, Gantt, Calendrier)

> **Version** : 3.0 (refonte vérifiée par rapport au code source actuel — remplace la v2.0 qui décrivait 3 onglets, 6 tableaux Kanban et 3 niveaux de zoom)
> **Code de référence** : `frontend/src/pages/SuiviPage.tsx` (8267 lignes, 4 onglets), `frontend/src/pages/suivi/SuiviAssistantTab.tsx`, `frontend/src/components/suivi/QuickCreateModal.tsx` (1301 lignes), `frontend/src/components/suivi/SuiviShareButton.tsx`
> **Backends** : `backend/routers/production.py` (cœur : Kanban, Gantt, Calendrier, opérations, assignations, partage), `backend/routers/projects.py` (Gantt « Projets » avec phases + assignations projet), `backend/routers/suivi_ai.py` (Assistant IA, lecture seule). Dépendances transversales : `crm.py` (Ventes/opportunités), `devis.py`, `suppliers.py`, `employees.py`, `ai.py` (widget Calendrier).
> **Route** : `/suivi` — menu latéral, section « Suivi », libellé « Suivi »
> **Libellés d'interface** : namespace i18n `crm.suivi.*` (fichier `crm.json`, lignes 367 à 1038). Quelques libellés d'accessibilité du Gantt sont sous `production.gantt.a11y.*`.
> **Tables PostgreSQL** : le module n'a **aucune table métier propre** ; il compose les entités d'autres modules. Tables techniques qu'il crée et entretient lui-même (auto-réparation) : `gantt_dependencies`, `gantt_task_baselines`, `calendar_notes`, `bt_assignations`, `devis_assignations`, `achat_assignations`, `project_assignments`, plus la table partagée `public.calendar_share_tokens` (partage public).
> **Cadrage** : poste de pilotage **transversal** en lecture et édition légère. Il agrège Ventes (CRM), Soumissions (devis), Projets, Bons de travail (BT), Opérations, Achats (bons de commande) et Factures sous quatre angles : **Kanban**, **Gantt**, **Calendrier** et **Assistant IA**. Il ne remplace pas les modules source : les actions à effet comptable (factures) ou d'inventaire (réception d'achat, annulation de BT) restent **bloquées** ici et renvoient au module dédié.

---

## Sommaire

1. [Vue d'ensemble et accès](#1-vue-densemble-et-acces)
2. [Interface](#2-interface)
3. [Processus pas-à-pas](#3-processus-pas-a-pas)
4. [Référence](#4-reference)
5. [Intégrations et FAQ](#5-integrations-et-faq)
6. [Récapitulatif](#6-recapitulatif)

---

## 1. Vue d'ensemble et accès

### 1.1 À quoi sert le module Suivi

Le module **Suivi** est le tableau de bord d'un chef de projet ou d'un gestionnaire : il rassemble sur une seule page, et sous plusieurs formes, l'ensemble du travail en cours de l'entreprise. Plutôt que d'ouvrir successivement les modules Ventes, Soumissions, Projets, Bons de travail, Achats et Comptabilité, l'utilisateur voit tout au même endroit et peut agir directement :

- **Kanban** — glisser les cartes d'une colonne à l'autre pour faire avancer un dossier, réordonner les priorités à l'intérieur d'une colonne, assigner des employés.
- **Gantt** — visualiser l'échéancier de chantier sous forme de barres, déplacer et redimensionner les dates, relier les tâches par des dépendances, tracer un chemin critique et une ligne de base, exporter l'échéancier.
- **Calendrier** — voir les échéances par jour, semaine, mois ou en agenda, reprogrammer un événement par glissement, poser des notes, réassigner les opérations, obtenir un résumé par l'assistant Claude.
- **Assistant IA** — poser des questions en langage naturel sur l'avancement (« quels projets sont en retard ? ») et obtenir une réponse fondée sur les données réelles du tenant, en **lecture seule**.

### 1.2 Accès et prérequis

- **Menu latéral** : section « Suivi » -> **Suivi**.
- **Adresse (URL)** : `/suivi`.
- **Authentification** requise (`Depends(get_current_user)`) et **contexte tenant** (votre entreprise). Sans contexte tenant, l'API renvoie `400`.
- **Mode consultation (lecture seule)** : si votre abonnement est en mode consultation, toutes les écritures (glisser-déposer, édition de dates, assignations, notes, partage) sont bloquées en amont ; la lecture reste possible.
- **Assistant IA** : nécessite une clé Anthropic active, le garde-fou IA au vert et des **crédits IA** suffisants. Sinon : `503` (service indisponible), `403` (accès IA refusé) ou `402` (crédits épuisés).

### 1.3 Les quatre onglets

Le sélecteur d'onglets en haut de la page (`SuiviPage.tsx:418-443`, clé `MainTab`) propose, dans l'ordre :

| Onglet | Clé interne | Icône | Rôle |
|---|---|---|---|
| **Kanban** | `kanban` | Kanban | Tableaux de cartes par statut (7 tableaux) |
| **Gantt** | `gantt` | BarChart3 | Diagramme d'échéancier (6 sources) |
| **Calendrier** | `calendrier` | Calendar | Agenda mensuel/hebdomadaire/journalier |
| **Assistant IA** | `assistant` | Sparkles | Clavardage de suivi en lecture seule |

Kanban est l'onglet affiché par défaut.

### 1.4 Rôles et permissions (vue d'ensemble)

Le module applique volontairement des gardes différenciées : **voir** est ouvert à tout membre du tenant, **modifier** est réservé aux rôles opérationnels.

| Action | Qui peut la faire |
|---|---|
| Consulter Kanban, Gantt, Calendrier, Assistant | Tout membre authentifié du tenant |
| **Réordonner** les cartes Kanban (vertical) | Tout membre (geste purement cosmétique) |
| **Changer le statut** par glisser-déposer (Kanban), éditer les dates/statut/assigné du Gantt, créer/supprimer des dépendances, définir une ligne de base, assigner un employé à un BT ou à un achat, réassigner une opération | `admin`, `super_admin`, `gestionnaire`, `contremaitre` (rôles `BT_WRITE_ROLES`) |
| Créer/modifier/supprimer une **note** de calendrier | Tout membre ; une note **partagée** est aussi modifiable par un admin ; seul l'auteur peut changer la portée personnelle/partagée |
| Créer/modifier des **phases** ou **assignations de projet** | Tout membre (via le Gantt « Projets ») |
| **Générer / révoquer** le lien de partage public | `admin` ou `super_admin` uniquement (les autres voient et copient le lien existant, mais ne peuvent ni le créer ni le révoquer) |
| Consulter la **page publique** de partage | N'importe qui possédant le lien, **sans connexion** |

> Le rôle `super_admin` contourne la machine à états des bons de travail et la garde « BT terminé seulement si opérations terminées ».

---

## 2. Interface

### 2.1 Disposition générale

```
+----------------------------------------------------------------------+
|  Suivi                                                                |
+----------------------------------------------------------------------+
|  [ Kanban ] [ Gantt ] [ Calendrier ] [ Assistant IA ]                |
+----------------------------------------------------------------------+
|                                                                      |
|   Contenu de l'onglet actif                                          |
|                                                                      |
+----------------------------------------------------------------------+
```

Les onglets Gantt et Calendrier passent en pleine hauteur (mise en page dense) ; Kanban et Assistant restent en défilement standard. Toute erreur de chargement s'affiche dans un bandeau rouge fermable en haut de la zone de contenu.

### 2.2 Onglet Kanban

L'onglet Kanban (`KanbanTab`, `SuiviPage.tsx:631-1913`) présente **7 tableaux** sélectionnables par des pastilles, chacun avec ses propres colonnes de statut.

#### Les 7 tableaux et leurs colonnes

| Tableau | Libellé | Colonnes (de gauche à droite) | Source |
|---|---|---|---|
| Ventes | **Ventes** | `PROSPECTION`, `QUALIFICATION`, `PROPOSITION`, `NEGOCIATION` (les gagnées/perdues sont résumées dans un encart) | Opportunités CRM |
| Soumissions | **Soumissions** | `Brouillon`, `Valide`, `Envoye`, `En attente`, `Accepte`, `Refuse`, `Termine` | Devis |
| Projets | **Projets** | `En attente`, `En cours`, `Suspendu`, `Termine` | Projets |
| Achats | **Achats** | `Brouillon`, `Envoye`, `Confirme`, `En cours`, `Recu`, `Facture`, `Annule` | Bons de commande |
| BT | **BT** | `BROUILLON`, `EN_COURS`, `EN_PAUSE`, `TERMINE` | Bons de travail |
| Opérations | **Opérations** | `En attente`, `En cours`, `Termine`, `Annule` | Opérations des BT |
| Factures | **Factures** | `BROUILLON`, `ENVOYEE`, `PARTIELLEMENT_PAYEE`, `PAYEE`, `EN_RETARD` | Factures (**lecture seule**) |

> Les colonnes couvrent volontairement **tous** les statuts possibles de chaque entité : ainsi, aucune carte ne « disparaît » du tableau parce que son statut ne correspondrait à aucune colonne.

#### Barre supérieure

Au-dessus des colonnes : bouton **Rafraîchir**, bouton **Partager** (voir 2.7), bouton **Créer** (ouvre la modale de création rapide — voir 2.6). Le bouton **Créer** est masqué sur le tableau Factures (les factures ne se créent pas ici).

#### Panneau latéral gauche (poste de travail, sur ordinateur)

Une colonne de 288 px présente des cartes de synthèse pour le tableau actif :

- **Vue active** — le nom du tableau courant.
- **Information** — barre de progression globale (%) et trois compteurs : « À faire », « En cours », « Terminés ».
- **Budget total** — somme des montants des cartes affichées, formatée en dollars (par exemple `125 000,00 $`).
- **Par statut** — pour chaque colonne, le nombre de cartes et une mini-barre proportionnelle.
- **Par employé** — la charge de travail répartie par employé assigné. **Cliquer sur un employé filtre le tableau** pour ne montrer que ses cartes ; une ligne « Non assigné » regroupe les cartes sans assigné. Cette carte est **absente** pour le tableau Factures.

Sur mobile, le panneau latéral est remplacé par une barre compacte de statistiques ; les colonnes défilent horizontalement avec des flèches ‹ › et des pastilles de position.

#### Anatomie d'une carte

```
+------------------------------------------+
|  Titre (nom / numéro)              [⋮]   |
|  Entreprise                    Montant   |   <- Ventes
|  ███████░░░  Probabilité 65 %            |   <- Ventes
|  Fournisseur : ...                       |   <- Achats
|  Début : 12 janv. 2026  Fin : 30 janv.   |
|  (avatars des assignés)   [+]  [Échéance]|
+------------------------------------------+
```

Selon le tableau, une carte affiche : le titre (nom, numéro ou titre), l'entreprise et le montant (Ventes), une barre de probabilité (Ventes), le fournisseur (Achats), les dates de début et de fin, un **badge d'échéance** (« En retard », « Aujourd'hui » ou « {n}j restants »), les **avatars** des employés assignés (photo ou initiales, avec un « +N » au-delà d'un certain nombre), un bouton **+** pour ajouter un assigné, et un badge de **priorité** (URGENT / HAUTE) le cas échéant.

- **Un clic** ouvre la modale de détail.
- **Un double-clic** ouvre la fiche source dans son module : Ventes -> `/ventes`, Soumissions -> `/devis`, Projets -> `/projets`, BT et Opérations -> `/bons-travail`, Achats -> `/magasin`, Factures -> `/comptabilite`.
- Le bouton **+** (masqué sur Factures) ouvre la fenêtre d'assignation d'un employé.

#### Modales et états

- **Détail de carte** — statut, priorité, montant, date de début, échéance (avec badge), fournisseur, « Équipe assignée » avec bouton **Ajouter** (masqué sur Factures) et un bouton **Fermer**.
- **Assigner un employé** — champ de recherche et liste des employés (jusqu'à 100 chargés).
- **Feuille mobile « Déplacer vers »** — sur mobile, remplace le glisser-déposer : elle liste les colonnes cibles.
- **Notifications** — bandeau vert (« Statut mis à jour avec succès », « Employé assigné avec succès ») ou bandeau rouge flottant en cas d'échec.

### 2.3 Onglet Gantt

L'onglet Gantt (`GanttTab`, `SuiviPage.tsx:1926-5426`) affiche un diagramme d'échéancier de type MS Project : un tableau à gauche, une frise chronologique à droite.

#### Les 6 sources

Sélectionnables par pastilles : **Ventes**, **Soumissions** (devis), **Projets**, **Achats** (bons de commande), **BT**, **Opérations**. Les Projets et les BT affichent des **sous-tâches** repliables (phases de projet ; opérations d'un BT) via un chevron.

#### Légende

Quatre couleurs de statut : **En attente**, **En cours**, **Terminé**, **Annulé**.

#### Barre d'outils (à droite)

| Contrôle | Effet |
|---|---|
| **Zoom** (5 niveaux) | `24h`, `3 jours`, `Semaine`, `2 Sem`, `Mois`. Sur mobile, un menu déroulant remplace les pastilles. |
| Sélecteur de **date** + ‹ › + **Aujourd'hui** | Visible en zoom `24h` / `3 jours` (« vue chantier » en heures) pour se déplacer heure par heure. |
| **Recherche** | Filtre les lignes par texte. |
| **Dépendances** | Affiche ou masque les flèches de liaison entre barres. |
| **Chemin critique** | Trace le chemin critique (tâches sans marge, en jours ouvrés) ; anneau rouge sur les barres critiques. |
| **Ligne de base** | Affiche le plan figé (barre grise) et l'écart d'échéancier ; panneau Définir / Mettre à jour / Effacer. |
| **Exporter CSV** | Télécharge l'échéancier en CSV. **Désactivé pour la source Opérations.** |
| **Imprimer** | Lance l'impression du navigateur (`window.print()`). |
| **Rafraîchir**, **Partager**, **Créer** | Recharge, partage public, création rapide. |

#### Panneau de gauche (colonnes de type MS Project)

Colonnes **triables** (clic sur l'en-tête) et **redimensionnables** (poignées) : **Numéro**, **Nom**, **Projet**, **Montant**, **Priorité**, **Statut**, **Assigné**, **Fournisseur**, **Début**, **Durée**, **Fin**, **%**. Toutes les colonnes ne s'affichent pas pour toutes les sources (par exemple, Projet et Fournisseur n'apparaissent que pour BT, Achats et Opérations ; la Priorité est absente des Soumissions et des Achats).

Cellules **modifiables directement** (double-clic) :

- **Statut** — menu déroulant. Pour les BT, seules les transitions valides depuis le statut courant sont proposées (voir la machine à états en 4.4).
- **Assigné** — menu déroulant des employés.
- **Fournisseur** — modifiable uniquement pour les Opérations et les sous-lignes de BT (options : « Interne » + fournisseurs du Magasin).
- **Début** et **Fin** — champs de date.
- **Durée** et **%** sont en **lecture seule** (le % est calculé automatiquement).

#### Frise chronologique (les barres)

- Barres colorées par statut ; **glisser** pour déplacer, **redimensionner** par le bord gauche ou droit (ce qui modifie les dates).
- **Poignée de préhension** à gauche de chaque ligne pour réordonner verticalement.
- **Jalons** représentés par un losange lorsque la durée est d'un jour ou moins.
- **Ligne « maintenant »** rouge (avec l'heure exacte en vue heures).
- **Barre grise** de ligne de base et **anneau rouge** du chemin critique lorsqu'ils sont activés.
- Lignes parentes (BT -> opérations, Projet -> phases) repliables.

#### Dépendances

Panneau tableau : **Source**, **Cible**, **Type** (menu déroulant FS / SS / FF / SF), **Délai (j)** et suppression. Les liens apparaissent aussi sous forme de **flèches** entre les barres. On crée une dépendance en **glissant depuis le bord droit** d'une barre vers une autre (mode liaison, bandeau « Mode liaison », `Échap` pour annuler). Un clic sur une flèche ouvre une confirmation de suppression. Les flèches sont **masquées** en vue heures (24h / 3 jours).

#### Infobulle

Au survol/clic d'une barre : titre, statut, Début, Fin, Progression %, Budget, Gestionnaire. Un double-clic sur la barre ou le numéro ouvre la fiche source.

#### Préférences mémorisées

Zoom, source, affichage des dépendances, chemin critique, ligne de base, date affichée, largeur du panneau, largeurs des colonnes et lignes repliées sont conservés dans le navigateur (clé `erp.suivi.gantt.prefs.v1`) et restaurés à la prochaine ouverture.

### 2.4 Onglet Calendrier

L'onglet Calendrier (`CalendarTab`, `SuiviPage.tsx:5537-8267`) est un agenda complet.

#### Quatre modes de vue

**Mois**, **Semaine**, **Jour**, **Agenda**. Raccourcis clavier : `Maj+M`, `Maj+S`, `Maj+J`, `Maj+A` ; `Maj+T` revient à aujourd'hui.

#### En-tête

Navigation ‹ › + titre de la période + bouton **Aujourd'hui** ; bascule de vue ; champ de **Recherche** (raccourci `/`) ; bouton **Partager** (lien public en lecture seule) ; bouton **Claude** / « Demander à Claude » (voir plus bas).

#### Filtres

- **Employé** — menu déroulant « Tous les employés » + liste ; filtre le calendrier sur un employé.
- **10 types d'événements** en pastilles à activer/désactiver : Opportunité, Soumission, Projet, Bon de commande, Bon de travail, Facture, Interaction, Activité CRM, Note, Opération.
- Un compteur de résultats de recherche et un bouton **Réinitialiser**.

Les préférences (vue, filtres) sont mémorisées dans le navigateur (`erp.suivi.calendar.prefs.v1`).

#### Vue Mois

Grille de 7 colonnes avec **numéros de semaine ISO** et **jours fériés du Québec** (2024-2030, drapeau rouge). Chaque case montre les événements et les notes du jour. Au survol (sur ordinateur), deux boutons apparaissent : **ajouter une note** et **créer un élément**. Un clic sur une case sélectionne le jour et ouvre un panneau latéral (ordinateur) ou une feuille inférieure (mobile). **Glisser un événement** vers une autre case le **reprogramme** (décale ses dates).

#### Vue Semaine

7 colonnes-jours, en-têtes avec fériés, événements par colonne, bouton de création par jour, glissement pour reprogrammer.

#### Vues Jour et Agenda

Liste/échéancier des événements du jour ou de la période, avec navigation vers la fiche source.

#### Notes libres (style pense-bête)

On crée et modifie des notes directement dans une case (surtout en **vue Mois** ; les vues Semaine et Jour affichent l'indice « ouvrir en vue Mois pour modifier »). Chaque note a une **portée** :

- **Personnelle** — visible par vous seul.
- **Partagée** — visible par toute l'entreprise.

La note affiche son auteur (« par … ») et un badge Personnelle/Partagée. La suppression demande une confirmation. **Seul l'auteur** peut basculer une note de personnelle à partagée (et inversement) ; une note partagée peut aussi être modifiée ou supprimée par un administrateur.

#### Opérations dans le calendrier

Pour les événements de type Opération, on peut réassigner **directement** l'**employé** et le **sous-traitant/fournisseur**. Ce droit est réservé aux rôles `admin`, `super_admin`, `gestionnaire`, `contremaitre` ; les autres voient l'assignation en lecture seule.

#### Assistant du calendrier — « Demander à Claude »

Le bouton **Claude** ouvre la modale « Assistant Claude — Calendrier » : suggestions cliquables, compteur de caractères (1000 max), note de confidentialité (« Aucune donnée hors de votre tenant »), bouton **Nouveau** (efface la conversation) et **Envoyer**.

> **Important** — ce widget est **distinct** de l'onglet Assistant IA (voir 2.5). Il utilise l'assistant **général** de l'ERP avec un **contexte calendrier compact** (jusqu'à 50 événements visibles) ; c'est un outil de résumé/suggestion. Il **débite** aussi les crédits IA.

### 2.5 Onglet Assistant IA

L'onglet **Assistant IA** (`SuiviAssistantTab.tsx`) est un **clavardage en lecture seule** dédié au Suivi. En-tête « Assistant IA — Suivi », sous-titre « Suis l'avancement à partir de tes données réelles (lecture seule). ». L'état vide propose trois exemples de questions :

- « Quels projets sont en cours et lesquels sont en retard ? »
- « Quel est l'état de mon pipeline de ventes par statut ? »
- « Quels bons de travail ne sont pas encore terminés ? »

L'assistant interroge les données réelles (ventes, projets, devis, BT, achats, opérations) et répond sur l'avancement et l'état. Il **n'écrit rien** : aucune carte « proposer/confirmer », aucune création ni modification. Chaque réponse affiche des métadonnées (jetons, coût, durée). Un verrou empêche l'envoi de deux questions simultanées.

### 2.6 Élément partagé — la modale « Créer »

Le bouton **Créer** des onglets Kanban, Gantt et Calendrier ouvre la même modale de création rapide (`QuickCreateModal.tsx`). Le type proposé par défaut suit le tableau/onglet actif.

**6 types** : **Projet**, **Opportunité**, **Soumission** (devis), **Bon de travail**, **Bon de commande**, **Opération**.

Champs selon le type : Type, Nom (obligatoire sauf pour un bon de commande), BT parent (obligatoire pour une opération), Employé et Fournisseur (opération), Fournisseur (obligatoire pour un bon de commande), Statut, Priorité (Basse / Normale / Haute / Urgente — absente des bons de commande et des opérations), Client (projet/opportunité/devis), Projet associé (devis/BT/BC), dates de début et de fin, date d'échéance/clôture/livraison/prévue (le libellé varie), Montant/Budget/Prix estimé, Probabilité % (opportunité), Source (opportunité), Type de soumission Détaillée/Budgétaire (devis), PO Client, Adresse chantier (projet), Description et Notes.

**Opérations en lot** (bons de travail seulement) : une section « Opérations » permet d'ajouter plusieurs opérations d'un coup (Nom obligatoire, Description, Quantité, Heures prévues, Statut, Fournisseur Interne/Externe, Employé, dates, Poste de travail). Elles sont créées à la suite du BT ; en cas d'échec partiel, un message signale le nombre d'opérations en erreur.

Deux boutons de soumission : **Créer** (reste sur la vue et rafraîchit) et **Plus de détails ->** (crée puis ouvre la page complète de la fiche ; masqué pour les BT).

### 2.7 Élément partagé — le partage public

Le bouton **Partager** (présent dans les trois vues) donne accès à **un seul lien public en lecture seule par entreprise**. Le paramètre `?view=` ouvre la bonne vue (calendrier, Gantt ou Kanban) à partir du même lien.

- **Générer** et **Révoquer** le lien sont réservés aux administrateurs ; tout autre utilisateur peut voir et copier un lien déjà actif, mais pas en créer.
- **Portée** (rappelée dans l'interface) : le lien n'affiche **que l'échéancier** — projets, bons de travail et **notes partagées** (leurs intitulés et dates). Il n'expose **jamais** de devis, facture, montant, opportunité ni note personnelle. Le lien reste valide **jusqu'à révocation**.

> Conseil : puisque les titres sont visibles publiquement, évitez d'y inscrire des données sensibles.

---

## 3. Processus pas-à-pas

### 3.1 Faire avancer une carte Kanban (changement de statut)

1. Ouvrir l'onglet **Kanban** et choisir le tableau (Ventes, Soumissions, Projets, Achats, BT, Opérations).
2. Sur ordinateur, **glisser** une carte vers une autre colonne ; un emplacement pointillé apparaît dans la colonne cible.
3. Au dépôt, la carte change de colonne immédiatement (mise à jour optimiste), puis l'API enregistre le nouveau statut : `PUT /production/kanban/update-status` (ou `crmApi.updateOpportunity` pour les Ventes, `updateOperation` pour les Opérations).
4. Si le serveur refuse (transition interdite, action réservée à un autre module), la carte **revient à sa place** et un message rouge s'affiche.
5. Sur mobile : le glisser-déposer est remplacé par le bouton **Déplacer** de la carte, qui ouvre la liste des colonnes cibles.

> **Blocages volontaires** (le glisser-déposer n'est pas un raccourci universel) : impossible de changer le statut d'une **facture** ici (piloté par la Comptabilité) ; impossible de faire passer un **achat** à « Reçu » ou « Facturé » (réception = module Magasin) ni de dé-valider un BC déjà reçu/facturé ; impossible de faire passer un **devis** à « Accepté » (passe par Soumissions, qui crée le projet lié) ; impossible d'**annuler** un BT depuis le Kanban (annulation = module BT, pour restaurer le stock). Voir 4.3.

### 3.2 Réordonner les cartes dans une colonne

1. Glisser une carte **verticalement** à l'intérieur de sa colonne.
2. L'ordre est enregistré via `PUT /production/kanban/reorder` (ou `crmApi.reorderOpportunities` pour les Ventes).
3. Ce geste est **purement cosmétique** : il n'a aucun effet métier et est donc autorisé à tout membre du tenant, y compris pour les Factures et les Opérations.

### 3.3 Assigner un employé à une carte

1. Cliquer le bouton **+** à côté des avatars d'une carte (sur mobile, passer par la modale de détail).
2. Dans la fenêtre **Assigner un employé**, rechercher par nom et cliquer sur l'employé voulu.
3. Un message vert confirme (« Employé assigné avec succès ») et le tableau se recharge.

> Le tableau **Factures** ne permet pas l'assignation (aucun bouton **+**).

### 3.4 Filtrer un tableau Kanban par employé

1. Dans le panneau latéral, carte **Par employé**, cliquer sur un employé.
2. Le tableau ne montre plus que ses cartes ; la ligne « Non assigné » isole les cartes sans assigné.
3. Cliquer de nouveau (ou « Afficher tout ») pour retirer le filtre.

### 3.5 Déplacer ou redimensionner une barre du Gantt

1. Ouvrir l'onglet **Gantt** et choisir la source.
2. **Glisser le centre** d'une barre pour la déplacer, ou **saisir un bord** (gauche/droit) pour la redimensionner.
3. Au relâchement, les nouvelles dates sont enregistrées dans l'entité correspondante (projet, BT, opération, devis, bon de commande, opportunité).
4. Pour réordonner une ligne, utiliser la **poignée de préhension** à gauche.

> En vue **24h** ou **3 jours**, les barres sont cliquables (infobulle) mais **non déplaçables**, et les flèches de dépendance sont masquées.

### 3.6 Modifier une cellule du Gantt (statut, assigné, fournisseur, dates)

1. Double-cliquer la cellule à modifier dans le panneau de gauche.
2. Choisir/saisir la valeur : **Statut** (menu ; transitions valides pour les BT), **Assigné** (employés), **Fournisseur** (Opérations et sous-lignes de BT seulement), **Début** / **Fin** (dates).
3. La valeur est enregistrée immédiatement. Durée et % restent en lecture seule.

### 3.7 Créer une dépendance entre deux barres

1. Activer l'affichage des **Dépendances**.
2. **Glisser depuis le bord droit** d'une barre (mode liaison, bandeau « Mode liaison ») vers la barre cible ; `Échap` annule.
3. Choisir le **Type** (FS / SS / FF / SF) et le **Délai (j)** dans le panneau des dépendances au besoin.

> Le serveur **refuse** une dépendance qui créerait un **cycle** (« Cette dépendance créerait un cycle ») et interdit de relier une entité à elle-même. Le délai est borné à ±3650 jours.

### 3.8 Supprimer une dépendance

1. Cliquer sur la **flèche** de la dépendance dans la frise (ou la ligne dans le panneau).
2. Confirmer dans la fenêtre « Supprimer cette dépendance ? ».

### 3.9 Définir, mettre à jour ou effacer une ligne de base

1. Activer **Ligne de base**.
2. **Définir la ligne de base** fige les dates actuelles (plan de référence) ; une barre grise apparaît sous chaque barre.
3. **Mettre à jour** re-fige à l'état courant ; **Effacer** retire la ligne de base — attention, l'effacement s'applique à **toutes les vues** et demande une confirmation (« irréversible »).

### 3.10 Afficher le chemin critique

1. Activer **Chemin critique**. Les tâches sans marge sont marquées d'un anneau rouge et d'une étiquette « critique » ; les autres affichent leur marge (« M:{n} j »).
2. S'il n'y a **aucune dépendance** dans la vue, le chemin critique se limite à la tâche la plus longue ; reliez des tâches pour un ordonnancement réel.

### 3.11 Exporter le Gantt en CSV / imprimer

1. **Exporter CSV** agrège Projets + BT + Devis + Bons de commande + Dépendances dans un fichier unique (avec en-tête UTF-8 pour Excel Québec).
2. **Imprimer** lance la boîte d'impression du navigateur (choisir « Enregistrer en PDF » au besoin).

> L'exportation CSV est **désactivée** quand la source active est **Opérations**.

### 3.12 Naviguer et changer de vue dans le calendrier

1. Utiliser ‹ › pour changer de période, **Aujourd'hui** pour revenir à la date du jour.
2. Basculer entre **Mois / Semaine / Jour / Agenda** (ou `Maj+M/S/J/A`).
3. Utiliser la **Recherche** (`/`) et les **filtres de type** pour cibler ; **Réinitialiser** remet tout à zéro.

### 3.13 Créer un élément depuis un jour du calendrier

1. En vue Mois ou Semaine, survoler une case et cliquer **créer un élément** (ou le bouton **Créer**).
2. La modale de création rapide s'ouvre avec la date pré-remplie ; compléter et **Créer** (ou **Plus de détails ->**).

### 3.14 Reprogrammer un événement par glissement

1. En vue Mois ou Semaine, **glisser** un événement vers un autre jour.
2. Les dates de l'entité sont décalées automatiquement.

> Certains types ne sont pas déplaçables depuis le calendrier ; un message l'indique (« Les éléments de type … ne peuvent pas être déplacés depuis le calendrier »).

### 3.15 Ajouter une note au calendrier

1. En **vue Mois**, cliquer **Ajouter une note** sur la case du jour (ou double-cliquer la case).
2. Saisir le texte (Entrée pour enregistrer, Échap pour annuler ; 500 caractères max).
3. Choisir la portée : **Rendre partagée** (visible par toute l'entreprise) ou **Rendre personnelle** (vous seul).
4. Pour modifier/supprimer plus tard : rouvrir en vue Mois. Un administrateur peut aussi gérer les notes **partagées**.

### 3.16 Réassigner une opération depuis le calendrier

1. Sur un événement de type **Opération**, ouvrir les sélecteurs d'assignation.
2. Choisir un **employé** et/ou un **sous-traitant/fournisseur**.
3. Action réservée aux rôles `admin`, `super_admin`, `gestionnaire`, `contremaitre`.

### 3.17 Interroger l'Assistant IA du Suivi

1. Ouvrir l'onglet **Assistant IA**.
2. Poser une question sur l'avancement (les exemples proposés sont cliquables).
3. Lire la réponse (fondée sur vos données réelles). L'assistant ne modifie rien ; chaque réponse indique le coût en crédits IA.

### 3.18 Demander un résumé au calendrier (widget Claude)

1. Dans l'onglet **Calendrier**, cliquer **Claude**.
2. Poser une question ou choisir une suggestion (« Quels événements sont en retard ? », « Résume ma charge pour cette période »).
3. **Nouveau** efface la conversation. Ce widget résume les événements **visibles** ; il débite les crédits IA.

### 3.19 Partager le suivi (lien public)

1. Cliquer **Partager**.
2. Si vous êtes administrateur : **Générer un lien public** (ou révoquer un lien existant).
3. **Copier** le lien et le transmettre. Le destinataire voit l'échéancier en lecture seule, sans connexion. Le paramètre `?view=` (calendar / gantt / kanban) choisit la vue affichée.

---

## 4. Référence

### 4.1 Onglets et raccourcis

| Onglet | Clé | Raccourci de vue (Calendrier) |
|---|---|---|
| Kanban | `kanban` | — |
| Gantt | `gantt` | — |
| Calendrier | `calendrier` | `Maj+M` Mois · `Maj+S` Semaine · `Maj+J` Jour · `Maj+A` Agenda · `Maj+T` Aujourd'hui · `/` Recherche |
| Assistant IA | `assistant` | — |

### 4.2 Tableaux Kanban et colonnes

| Tableau | Colonnes | Édition du statut par glisser |
|---|---|---|
| Ventes | PROSPECTION · QUALIFICATION · PROPOSITION · NEGOCIATION (+ GAGNE/PERDU en encart) | Oui |
| Soumissions | Brouillon · Valide · Envoye · En attente · Accepte · Refuse · Termine | Oui, **sauf** vers Accepté (bloqué -> Soumissions) |
| Projets | En attente · En cours · Suspendu · Termine | Oui |
| Achats | Brouillon · Envoye · Confirme · En cours · Recu · Facture · Annule | Oui, **sauf** vers Reçu/Facturé (bloqué -> Magasin) |
| BT | BROUILLON · EN_COURS · EN_PAUSE · TERMINE | Oui (machine à états), **sauf** vers Annulé (bloqué -> module BT) |
| Opérations | En attente · En cours · Termine · Annule | Oui |
| Factures | BROUILLON · ENVOYEE · PARTIELLEMENT_PAYEE · PAYEE · EN_RETARD | **Non** (lecture seule) |

### 4.3 Règles de blocage du glisser-déposer Kanban

Le serveur (`PUT /production/kanban/update-status`) protège les effets comptables et d'inventaire :

| Cas | Résultat | Raison |
|---|---|---|
| Toute carte **Facture** | `400` | Statut piloté par la Comptabilité (paiements, écritures) |
| **Achat** vers `Recu` ou `Facture` | `400` | Réception/facturation = module Magasin (mouvement de stock) |
| Dé-valider un **achat** déjà `Recu`/`Facture` | `400` | Cohérence de stock |
| **Devis** vers `Accepte` | `400` | Passe par Soumissions (crée le projet lié + notifie) |
| **BT** vers `Annule` | `400` | Annulation = module BT (restauration du stock) |
| **BT** vers `Termine` avec des opérations non terminées | `400` | Toutes les opérations doivent être terminées ou annulées (sauf `super_admin`) |

> Le **réordonnancement** (`reorder`), lui, autorise **toutes** les entités, y compris Factures et Opérations, car il n'a aucun effet métier.

### 4.4 Machine à états des bons de travail (BT)

Transitions autorisées (le `super_admin` les contourne) :

| Depuis | Vers |
|---|---|
| BROUILLON | BROUILLON, EN_COURS, ANNULE |
| EN_COURS | EN_COURS, EN_PAUSE, TERMINE, ANNULE |
| EN_PAUSE | EN_PAUSE, EN_COURS, ANNULE |
| TERMINE | TERMINE (terminal) |
| ANNULE | ANNULE (terminal) |

Le Gantt ne propose dans le menu déroulant de statut que les transitions valides depuis le statut courant (les variantes accentuées/espacées héritées sont normalisées automatiquement).

### 4.5 Sources et endpoints du Gantt

| Source | Endpoint de données | Notes |
|---|---|---|
| Ventes | `crmApi.listOpportunities` | Opportunités CRM |
| Soumissions | `GET /production/gantt/devis` | Exclut Annulé et Refusé |
| Projets | `GET /projects/gantt` | Projets non annulés (**LIMIT 500**) + phases en sous-tâches ; progression = moyenne des phases |
| Achats | `GET /production/gantt/bons-commande` | Début = date de commande, fin = date de livraison prévue |
| BT | `GET /production/gantt/bons-travail` | BT + opérations en sous-tâches ; fournisseur agrégé des sous-traitants |
| Opérations | `GET /production/gantt/operations` | Une ligne par opération (les opérations orphelines, sans BT, sont masquées) |

> Un endpoint `GET /production/gantt/projects` existe côté serveur mais n'est **pas** utilisé par l'interface ; la vue « Projets » passe par `GET /projects/gantt` (celui qui porte les phases).

### 4.6 Types de dépendances

| Code | Libellé |
|---|---|
| `finish_to_start` | Fin -> Début (FS) |
| `start_to_start` | Début -> Début (SS) |
| `finish_to_finish` | Fin -> Fin (FF) |
| `start_to_finish` | Début -> Fin (SF) |

Délai (`lag_days`) : entier borné **-3650 à +3650**. Types d'entités pouvant être reliées : projet, BT, devis, bon de commande, opération, opportunité. Détection de cycle **fail-closed** (refuse en cas de doute).

### 4.7 Calendrier — types d'événements et vues

**Modes de vue** : Mois, Semaine, Jour, Agenda.

**10 types filtrables** : Opportunité, Soumission, Projet (+ sous-type « Début projet »), Bon de commande, Bon de travail, Facture, Interaction, Activité CRM, Note, Opération. Les événements proviennent de l'agrégateur `GET /production/calendar-events` (projets, BT, opérations, devis, bons de commande, factures, interactions, activités), des **notes** (`GET /production/calendar-notes`) et des **opportunités** du CRM.

### 4.8 Endpoints (référence complète)

Tous préfixés par `/api/erp/v1`.

| Domaine | Méthode + chemin | Accès |
|---|---|---|
| Kanban | `GET /production/kanban` | Lecture |
| Kanban | `GET /production/kanban/achats` | Lecture |
| Kanban | `PUT /production/kanban/update-status` | Écriture (rôles) |
| Kanban | `PUT /production/kanban/reorder` | Lecture (cosmétique) |
| Gantt données | `GET /production/gantt/{bons-travail,devis,bons-commande,operations}` | Lecture |
| Gantt données | `GET /projects/gantt` | Lecture |
| Gantt dépendances | `GET/POST/PUT/DELETE /production/gantt/dependencies` | Lecture (GET) / Écriture (rôles) |
| Gantt ligne de base | `GET/POST/DELETE /production/gantt/baselines` | Lecture (GET) / Écriture (rôles) |
| Gantt exportation | `GET /production/gantt/export-csv` | Lecture |
| Calendrier | `GET /production/calendar-events?year&month` | Lecture |
| Calendrier notes | `GET/POST/PUT/DELETE /production/calendar-notes` | Lecture ; écriture par l'auteur (ou admin si partagée) |
| Partage | `POST/DELETE /production/calendar/share` | Admin uniquement |
| Partage | `GET /production/calendar/share` | Lecture (jeton masqué aux non-admins) |
| Partage public | `GET /production/calendar/public/{token}?view=` | **Sans authentification** |
| Édition opération | `PUT /production/work-orders/{bt_id}/operations/{op_id}` | Écriture (rôles) |
| Assignation BT | `POST /production/work-orders/{bt_id}/assignations` | Écriture (rôles) |
| Assignation achat | `POST /production/achats/{achat_id}/assignations` | Écriture (rôles) |
| Dates BT | `PUT /production/work-orders/{bt_id}` | Écriture (rôles) |
| Phases projet | `POST/PUT /projects/{project_id}/phases` | Tout membre |
| Assignations projet | `GET/POST/DELETE /projects/{project_id}/assignments` | Tout membre |
| Dates projet | `PUT /projects/{project_id}` | Tout membre |
| Assistant IA | `POST /suivi/ai/chat` | Lecture (débite les crédits IA) |

### 4.9 Permissions et gardes

- **Lecture** (`get_current_user`, tout membre) : toutes les vues de données, l'exportation CSV et le **réordonnancement** Kanban.
- **Écriture** (`require_tenant_admin_or_role` avec `BT_WRITE_ROLES = admin, super_admin, gestionnaire, contremaitre`) : changement de statut Kanban, dépendances, ligne de base, édition d'opérations, assignations BT/achat.
- **Admin uniquement** : générer/révoquer le partage public (et le GET masque le jeton aux non-admins).
- **Sans garde de rôle mais avec propriété** : notes de calendrier (auteur, ou admin si partagée ; seul l'auteur change la portée), phases et assignations de projet.
- **Public** : la page de partage, sans authentification.

### 4.10 Limites, plafonds et fréquences

| Élément | Limite |
|---|---|
| Kanban — cartes par entité | 50 |
| Kanban — réordonnancement (`ordered_ids`) | 10 000 max |
| Gantt — Projets | 500 lignes |
| Ligne de base — items | 5 000 max |
| Dépendances — délai | -3650 à +3650 jours |
| Dépendances — détection de cycle | 1000 itérations (fail-closed) |
| Note de calendrier — longueur | 500 caractères |
| Partage — lien | permanent (jusqu'à révocation), 1 par tenant |
| Assistant IA — message | 1 à 8000 caractères |
| Assistant IA — historique | 40 tours envoyés (re-tronqués à 12) |
| Assistant IA — jetons de réponse | 8000 max ; boucle d'outils : 5 itérations |
| Assistant IA — fréquence | 20 requêtes/min par adresse IP |
| Page publique — fréquence | 60 requêtes/min par adresse IP |
| Format des dates échangées | `AAAA-MM-JJ` |

### 4.11 Calculs

| Élément | Règle |
|---|---|
| **Progression automatique** (Gantt, %) | 0 avant la date de début, 100 à partir de la date de fin, sinon `arrondi(temps écoulé / durée totale × 100)` |
| **Progression d'une opération** (BT) | `min(heures réelles / heures prévues × 100, 100)` |
| **Progression d'un projet** (Gantt) | moyenne des progressions de ses phases |
| **Badge d'échéance** | échéance dépassée -> « En retard » (rouge) ; jour même -> « Aujourd'hui » (bleu) ; dans 3 jours ou moins -> « {n}j restants » (jaune) ; au-delà -> aucun badge |
| **Budget total** (panneau Kanban) | somme des montants des cartes affichées |

### 4.12 Jours fériés du Québec (calendrier)

Le calendrier surligne les jours fériés du Québec de **2024 à 2030** (Jour de l'An, Vendredi saint, Lundi de Pâques, Journée nationale des patriotes, Fête nationale du Québec, Fête du Canada, Fête du Travail, Action de grâce, Noël, Lendemain de Noël) avec un drapeau rouge.

### 4.13 Effet sur les crédits IA

L'Assistant IA du Suivi (`POST /suivi/ai/chat`) et le widget « Demander à Claude » du calendrier **débitent les crédits IA prépayés** de votre entreprise. Le coût facturé correspond au coût réel des jetons Anthropic **majoré de 30 %**. En cas de solde épuisé : `402` (« Crédits IA épuisés »). Aucune autre intégration payante (Stripe, QuickBooks, etc.) n'intervient dans ce module.

---

## 5. Intégrations et FAQ

### 5.1 Intégrations avec les autres modules

Le Suivi est une **vue transversale** : il lit et modifie légèrement les entités d'autres modules, sans table métier propre.

| Module | Rôle dans le Suivi | Manuel |
|---|---|---|
| **CRM / Opportunités** | Tableau et source Gantt « Ventes » ; réordonnancement et changement de statut des opportunités | [05-gestion-crm-opportunites.md](./05-gestion-crm-opportunites.md) |
| **Soumissions (Devis)** | Tableau et source Gantt « Soumissions » ; dates du Gantt | [07-ventes-soumissions.md](./07-ventes-soumissions.md) |
| **Projets** | Tableau et source Gantt « Projets » (avec phases) ; assignations et dates | [08-ventes-projets.md](./08-ventes-projets.md) |
| **Bons de travail** | Tableau et source « BT » ; opérations en sous-tâches ; assignations ; machine à états | [11-operations-bons-de-travail.md](./11-operations-bons-de-travail.md) |
| **Bons de commande (Achats)** | Tableau et source « Achats » ; dates de livraison ; assignations | [13-operations-bons-de-commande.md](./13-operations-bons-de-commande.md) |
| **Comptabilité (Factures)** | Tableau « Factures » en lecture seule | [14-operations-comptabilite.md](./14-operations-comptabilite.md) |
| **Magasin (Inventaire)** | Fournisseurs proposés à l'assignation des opérations ; réception des achats | [09-operations-magasin.md](./09-operations-magasin.md) |
| **Employés** | Liste des employés pour les assignations | [10-operations-employes.md](./10-operations-employes.md) |

Double-clic depuis n'importe quelle vue : Ventes -> `/ventes`, Soumissions -> `/devis`, Projets -> `/projets`, BT et Opérations -> `/bons-travail`, Achats -> `/magasin`, Factures -> `/comptabilite`.

### 5.2 Les deux systèmes d'intelligence artificielle

Ne pas les confondre :

| | Onglet « Assistant IA » | Widget « Demander à Claude » (Calendrier) |
|---|---|---|
| Emplacement | Onglet dédié du Suivi | Bouton « Claude » de l'onglet Calendrier |
| Nature | Clavardage de suivi **spécialisé** | Assistant **général** avec contexte calendrier |
| Données | Interroge les tables du Suivi (liste blanche stricte, liste noire des données sensibles) | Résume les événements **visibles** (jusqu'à 50) |
| Écriture | Aucune (lecture seule) | Aucune (résumé/suggestion) |
| Crédits IA | Débités (majoration 30 %) | Débités |

### 5.3 Ce qui n'existe pas / limites à connaître

- Le tableau **Factures est en lecture seule** : ni changement de statut par glissement, ni assignation, ni création.
- L'**exportation CSV du Gantt n'est pas disponible** pour la source « Opérations » (bouton désactivé).
- En vues **24h / 3 jours**, les barres du Gantt ne sont **ni déplaçables ni redimensionnables** et les flèches de dépendance sont masquées.
- Les **opérations n'ont pas de page dédiée** : un double-clic ouvre toujours le **bon de travail parent**.
- L'onglet **Assistant IA est en lecture seule** — il ne crée ni ne modifie rien, sans confirmation d'action.
- Les **notes** se modifient surtout en **vue Mois** (les vues Semaine/Jour renvoient vers la vue Mois).
- La **réassignation d'opérations dans le calendrier** est réservée aux rôles `admin`, `super_admin`, `gestionnaire`, `contremaitre`.
- Le **partage public** n'expose que l'échéancier (projets, BT, notes partagées) : jamais de montants, devis, factures, opportunités ni notes personnelles.
- La **priorité** est absente des Soumissions et des Bons de commande.

### 5.4 FAQ

**Q : Pourquoi je ne peux pas passer une facture à « Payée » en glissant sa carte ?**
R : Par conception. Le statut des factures est piloté par la Comptabilité (paiements, écritures comptables). Le tableau Factures du Suivi est en lecture seule.

**Q : J'ai essayé de faire passer un achat à « Reçu » dans le Kanban, et ça a échoué.**
R : Normal. La réception d'un achat entraîne un mouvement de stock ; elle se fait dans le module Magasin, pas ici.

**Q : Pourquoi mon devis ne veut pas passer à « Accepté » depuis le Kanban ?**
R : L'acceptation d'un devis crée le projet lié et déclenche des notifications ; elle passe par le module Soumissions.

**Q : Je n'arrive pas à faire glisser un BT vers « Terminé ».**
R : Un BT ne peut être terminé que si **toutes** ses opérations sont terminées ou annulées. Terminez d'abord les opérations restantes (un `super_admin` peut contourner cette règle).

**Q : Le réordonnancement vertical des cartes est-il enregistré ?**
R : Oui, l'ordre est enregistré (`reorder`). C'est un geste cosmétique autorisé à tout membre, y compris sur les tableaux Factures et Opérations.

**Q : Quelle est la différence entre l'onglet « Assistant IA » et le bouton « Claude » du calendrier ?**
R : L'onglet Assistant IA est un clavardage de suivi spécialisé (données réelles, lecture seule). Le bouton « Claude » du calendrier est l'assistant général de l'ERP, alimenté par les événements visibles du calendrier, pour un résumé rapide. Les deux débitent les crédits IA.

**Q : Pourquoi le Gantt « Projets » ne montre-t-il pas tous mes projets ?**
R : La requête limite à 500 projets et exclut les projets annulés.

**Q : Pourquoi certaines barres du Gantt ne bougent-elles pas ?**
R : En vue 24h ou 3 jours (« vue chantier » en heures), les barres sont figées : cliquez pour l'infobulle, mais utilisez un zoom Semaine/2 Sem/Mois pour déplacer les dates.

**Q : Comment relier deux tâches ?**
R : Activez les Dépendances, puis glissez depuis le bord droit d'une barre vers une autre. Le système refuse toute liaison qui créerait une boucle.

**Q : À quoi sert la ligne de base ?**
R : À figer un plan de référence. La barre grise montre ensuite l'écart entre le plan initial et les dates actuelles. L'effacement s'applique à toutes les vues.

**Q : Qui peut créer le lien de partage public ?**
R : Un administrateur (ou super-admin). Les autres membres peuvent copier un lien déjà généré, mais pas en créer ni le révoquer. Le lien ne montre que l'échéancier, sans aucun montant.

**Q : Une note du calendrier peut-elle être vue par mes collègues ?**
R : Seulement si vous la rendez **partagée**. Une note personnelle n'est visible que par vous. Seul l'auteur peut changer cette portée ; un administrateur peut gérer les notes partagées.

**Q : Le double-clic sur une opération n'ouvre pas une page d'opération.**
R : Les opérations n'ont pas de page dédiée ; le double-clic ouvre leur bon de travail parent.

**Q : Le module Suivi a-t-il ses propres données ?**
R : Non. Il compose les entités des autres modules. Il ne crée que des tables techniques (dépendances, ligne de base, notes, assignations, jetons de partage).

---

## 6. Récapitulatif

| Élément | Détail |
|---|---|
| **Mission** | Poste de pilotage transversal : Kanban, Gantt, Calendrier et Assistant IA sur Ventes, Soumissions, Projets, BT, Opérations, Achats et Factures. |
| **Route / menu** | `/suivi` — menu latéral « Suivi ». |
| **Code source** | `SuiviPage.tsx` (8267 lignes), `SuiviAssistantTab.tsx`, `QuickCreateModal.tsx` (1301 lignes), `SuiviShareButton.tsx` ; backends `production.py` (cœur), `projects.py` (Gantt Projets), `suivi_ai.py` (Assistant). |
| **4 onglets** | Kanban · Gantt · Calendrier · Assistant IA. |
| **Kanban** | 7 tableaux (Ventes, Soumissions, Projets, Achats, BT, Opérations, Factures) ; glisser = changer de statut (avec blocages métier) ; glisser vertical = réordonner ; assignation d'employés. Factures = lecture seule. |
| **Gantt** | 6 sources ; 5 niveaux de zoom (24h, 3 jours, Semaine, 2 Sem, Mois) ; barres déplaçables/redimensionnables (sauf en heures) ; dépendances FS/SS/FF/SF ; chemin critique ; ligne de base ; exportation CSV (sauf Opérations) ; impression. |
| **Calendrier** | 4 vues (Mois, Semaine, Jour, Agenda) ; jours fériés QC ; notes personnelles/partagées ; reprogrammation par glissement ; réassignation d'opérations (rôles) ; widget « Claude ». |
| **Assistant IA** | Clavardage de suivi en lecture seule ; liste blanche de tables, aucune écriture ; débite les crédits IA (majoration 30 %). |
| **Créer** | Modale partagée, 6 types (Projet, Opportunité, Soumission, BT, Bon de commande, Opération) + opérations en lot pour les BT. |
| **Partage** | 1 lien public en lecture seule par tenant (`?view=` calendar/gantt/kanban) ; génération/révocation admin ; n'expose que l'échéancier. |
| **Permissions** | Lecture = tout membre ; écriture = admin/super_admin/gestionnaire/contremaitre ; partage = admin ; notes = auteur/admin ; page publique = sans authentification. |
| **Blocages volontaires** | Facture non modifiable ; achat non « Reçu/Facturé » ; devis non « Accepté » ; BT non « Annulé » ; BT « Terminé » seulement si opérations terminées. |
| **Pas dans ce module** | Pas de page d'opération dédiée ; pas d'exportation CSV pour les Opérations ; pas d'édition de barres en vue heures ; pas d'écriture par l'Assistant IA ; pas de table métier propre. |

---

*Manuel ERP Constructo — Module 02 Suivi (Kanban, Gantt, Calendrier) — v3.0 vérifié par rapport au code source — 2026-07-07*

**Sources vérifiées** : `frontend/src/pages/SuiviPage.tsx`, `frontend/src/pages/suivi/SuiviAssistantTab.tsx`, `frontend/src/components/suivi/QuickCreateModal.tsx`, `frontend/src/components/suivi/SuiviShareButton.tsx`, `frontend/src/api/production.ts`, `frontend/src/api/suiviAi.ts`, `frontend/src/i18n/locales/fr/crm.json` (bloc `crm.suivi.*`, lignes 367-1038) ; backends `backend/routers/production.py`, `backend/routers/projects.py`, `backend/routers/suivi_ai.py`.

**Manuels liés** :
- Module 05 — CRM / Opportunités : [05-gestion-crm-opportunites.md](./05-gestion-crm-opportunites.md)
- Module 07 — Soumissions (Devis) : [07-ventes-soumissions.md](./07-ventes-soumissions.md)
- Module 08 — Projets : [08-ventes-projets.md](./08-ventes-projets.md)
- Module 11 — Bons de Travail : [11-operations-bons-de-travail.md](./11-operations-bons-de-travail.md)
- Module 13 — Bons de Commande : [13-operations-bons-de-commande.md](./13-operations-bons-de-commande.md)
- Module 14 — Comptabilité (Factures) : [14-operations-comptabilite.md](./14-operations-comptabilite.md)
- Module 24 — Assistant IA : [24-communication-assistant-ia.md](./24-communication-assistant-ia.md)
