# Module 20 — Maintenance (preventive, corrective, predictive)

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — corrections majeures par rapport à la v2.0 : il y a désormais **11 onglets** et non 9, car **Compteurs** est maintenant un véritable écran de saisie et **Assistant IA** est un onglet à part entière ; le décompte réel est de **37 points d'accès** (31 de gestion et statistiques + 6 pour l'IA), et non 41 ; les modèles IA sont **Claude Sonnet 4-6** (question libre, liste de vérification) et **Claude Opus 4-8** (diagnostic, plan préventif, analyse d'intervention, estimation de coût), et non « opus-4-7 » ; le numéro `MR-#####` est **généré de façon unique** par sondage contre la contrainte d'unicité, et non par tirage aléatoire non protégé ; les relevés de compteurs (heures / kilomètres) **déclenchent réellement** des alertes d'usage ; **aucun** des 37 points d'accès n'a de garde de rôle — tout employé authentifié peut créer, modifier et supprimer ; le contrôle d'accès IA `check_ai_guard` est **neutre**, seul le solde de crédits bloque (erreur 402) ; le sous-titre du module n'est **jamais** affiché.)
> **Libellé dans le menu** : « Maintenance » (groupe « TERRAIN » de la barre latérale, icône `Wrench`) — route `/maintenance`. Réf. `Sidebar.tsx:68,76`, `nav.json:20,26`.
> **Titre affiché de la page** : « Maintenance » (`MaintenancePage.tsx:166`). Le sous-titre i18n « Gestion de la maintenance préventive et corrective » existe mais **n'est jamais rendu**.
> **Code de référence (côté serveur)** : tout le module vit dans **`ERP_REACT/backend/routers/secondary.py`** (routeur combiné « Secondary Modules » : Immobilier, Logistique, Location, **Maintenance**, Météo, Conformité, Subventions). La section Maintenance occupe les lignes **6180 → 8646** : **37 points d'accès** sous **`/maintenance/*`** (31 de gestion et statistiques + 6 pour l'assistant IA), la DDL des 8 tables (1736-1898) et son assistant `_ensure_maintenance_tables` (1899-1948), l'invite système de l'IA (1664-1723). **Il n'existe aucun fichier `routers/maintenance.py` ni `routers/maintenance_ai.py`.**
> **Ne pas confondre avec `routers/terrain.py`** : c'est un module **séparé** d'analyse de terrain (cadastre), sans aucun rapport avec la maintenance des équipements.
> **Chemins d'API réels** : préfixe `/api/erp/v1` (`erp_config.py:9`, monté `erp_api.py:1025`) — donc `/api/erp/v1/maintenance/*`. Le nom « `/maintenance` » désigne à la fois la route React (à l'écran) et le préfixe des appels serveur : ici les deux coïncident.
> **Code de référence (côté client)** : `ERP_REACT/frontend/src/pages/MaintenancePage.tsx` (**1960 lignes, un seul fichier**, 11 onglets, tous les sous-composants en ligne) ; `frontend/src/api/maintenance.ts` (445 lignes, 36 fonctions) ; magasin d'état `store/useMaintenanceStore.ts` ; textes sous `i18n/locales/fr/terrain.json` (sous-section `terrain.maintenance.*`).
> **Tables PostgreSQL** (une série **par tenant**, **créées à la demande**) : `maintenance_types`, `maintenance_planification`, `maintenance_demandes`, `maintenance_interventions`, `maintenance_pieces`, `maintenance_historique`, `maintenance_compteurs`, `maintenance_alertes`.
> **Cadrage** : gestion de maintenance assistée par ordinateur (GMAO) légère pour l'entretien des équipements de construction (parc de chantier, flotte, outillage). Le module catalogue des **types** de maintenance (préventive / corrective / prédictive), **planifie** des échéances récurrentes, gère des **demandes** (bons de travail) qui donnent lieu à des **interventions** et à la consommation de **pièces**, produit des **alertes**, tient un **historique** et des **relevés de compteurs** (heures / kilomètres / cycles) qui déclenchent des alertes « à l'usage », affiche des **statistiques**, et offre un **assistant IA** (5 outils). Ce **n'est pas** un module documentaire : il ne produit **aucun** bon d'intervention imprimable, **aucun** PDF, **aucun** CSV, et n'accepte **aucune** photo ni pièce jointe.

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

Donner à une entreprise de construction québécoise un **registre unique de l'entretien de ses équipements**, sous trois angles complémentaires :

- **Maintenance préventive** : définir des procédures-types (le catalogue « Types »), puis planifier des échéances récurrentes sur un équipement (tous les 30 jours, toutes les 250 heures d'utilisation, tous les 5000 km…). Quand une échéance approche, le module génère une **alerte**.
- **Maintenance corrective** : ouvrir une **demande** (un ticket de panne ou de réparation) numérotée `MR-#####`, la faire avancer par des **statuts**, y consigner une ou plusieurs **interventions**, y rattacher les **pièces** consommées, puis la clôturer — ce qui alimente automatiquement l'**historique** de l'équipement.
- **Maintenance « prédictive » / à l'usage** : saisir des **relevés de compteurs** (heures moteur, kilométrage) ; lorsqu'un équipement franchit l'intervalle prévu par une planification « à l'usage », le module crée une alerte automatiquement.

Le module répond concrètement à des questions comme :

- « Quelles maintenances préventives sont dues ou en retard cette semaine ? »
- « Cette excavatrice est tombée en panne : ouvrons une demande, notons les symptômes, l'intervention, les pièces, et clôturons. »
- « Cette génératrice a passé 250 heures depuis son dernier entretien : faut-il déclencher une alerte ? »
- « Quel est le coût total de maintenance ce mois-ci ? Combien de demandes en cours, en attente, terminées ? »
- « Devant ces symptômes, quel diagnostic probable, quelles pièces et quel coût estimer ? » (assistant IA)

> **Précision sur le mot « prédictive ».** Dans ce module, « prédictive » n'est **qu'une étiquette de catégorie** dans le catalogue des types (`PREDICTIVE`). Il n'y a **aucun moteur d'analyse prédictive** (pas d'apprentissage, pas de tendance de capteurs). La seule automatisation « fondée sur l'état » réelle, ce sont les **alertes à l'usage** déclenchées par les relevés de compteurs (voir 3.9 et 4.6).

### 1.2 Ce que le module fait (vérifié contre le code)

- **Catalogue des types** : créer, lister, rechercher, filtrer par catégorie, modifier et désactiver (suppression douce) des procédures-types (nom, catégorie, fréquence en jours, durée estimée, coût estimé, compétences requises).
- **Planifications préventives** : créer, lister, rechercher, filtrer par priorité, modifier et supprimer des échéances récurrentes par équipement, avec un **calcul automatique de la prochaine échéance** pour les fréquences en jours, semaines et mois.
- **Demandes de maintenance** : créer (numéro `MR-#####` généré par le serveur, statut initial `DEMANDE`), lister, filtrer par statut, ouvrir un **détail** complet, changer le statut, saisir un coût réel et une solution, et supprimer (seulement hors des états `EN_COURS` et `TERMINE`).
- **Interventions** : créer une intervention depuis le détail d'une demande (ce qui fait passer la demande à `EN_COURS`), puis, depuis l'onglet Interventions, éditer son statut, son temps passé et son rapport. Passer une intervention à `TERMINE` **clôture automatiquement** la demande parente et inscrit une ligne d'historique.
- **Pièces** : ajouter, depuis le détail d'une demande, les pièces consommées (nom, référence, quantité, coût unitaire) ; le coût total se calcule et, si la pièce est reliée à un article d'inventaire, **le stock est décrémenté**. Vue globale en lecture dans l'onglet Pièces.
- **Alertes** : générer en un clic les alertes des planifications dont l'échéance approche ou est dépassée, filtrer par priorité, les marquer « lue » puis « traitée ».
- **Historique** : consigner manuellement des événements par équipement (mise en service, panne, inspection, remplacement…) et lire l'historique chronologique. Une ligne est aussi **ajoutée automatiquement** à la clôture d'une demande.
- **Compteurs** : saisir des relevés d'heures / kilomètres / cycles ; un relevé peut **déclencher une alerte** si un intervalle de planification « à l'usage » est atteint.
- **Statistiques** : dix indicateurs (demandes, coûts, planifications, alertes, interventions) plus des répartitions par statut et par priorité.
- **Assistant IA** : cinq outils (discussion, diagnostic, plan préventif, liste de vérification, estimation de coût) qui produisent des conseils rédigés par Claude, facturés aux crédits IA de l'entreprise.

### 1.3 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes, et certaines étiquettes visibles ailleurs dans l'interface ne correspondent à aucun écran.

- **Aucun équipement relié.** Il n'existe **aucune fiche d'équipement**, **aucun menu déroulant** vers l'inventaire, la location ou les véhicules, et **aucune validation** de l'identifiant. Partout (Planification, Demandes, Historique, Compteurs), un équipement se désigne par un **couple** : un **Type d'équipement** (Inventaire / Location / Véhicule) et un **ID** tapé à la main sous forme de nombre brut. L'écran affiche « Inventaire #42 » sans jamais rechercher le nom réel de l'équipement 42.
- **Aucune exportation, aucune impression, aucun téléversement, aucune photo.** Pas de PDF, pas de CSV, pas de bon d'intervention imprimable, pas de pièce jointe. Le module ne produit **aucun** document remis à un client ou à un technicien.
- **Aucun onglet « Ordres de maintenance », « Équipements » ni « Planning » (calendrier).** Des libellés de traduction existent pour ces noms (`tabs.ordres`, `tabs.equipements`, `tabs.planning`), mais **le code n'a que 11 onglets** et aucun de ces trois n'y figure. La planification est une simple **liste**, pas une vue calendrier ni un diagramme de Gantt.
- **Les interventions ne se créent pas depuis leur onglet.** L'onglet Interventions ne fait qu'**éditer** et **supprimer**. La **création** passe obligatoirement par le détail d'une demande (voir 3.6).
- **Historique et Compteurs sont en ajout seul.** L'interface ne permet **ni de modifier ni de supprimer** une entrée d'historique ou un relevé de compteur. Pour corriger, ajoutez une nouvelle entrée.
- **Aucune notification automatique** (courriel, message texte, notification navigateur) sur les échéances, les retards ou les alertes. Les alertes vivent uniquement dans la table `maintenance_alertes` et l'onglet Alertes.
- **Aucun cron d'alertes.** Les alertes d'échéance ne se génèrent **pas** toutes seules : il faut cliquer sur **« Générer les alertes »** (voir 3.8). Les alertes d'usage, elles, naissent au moment d'un relevé de compteur.
- **Aucune écriture comptable.** Le coût réel d'une demande reste dans la table de maintenance ; rien n'est reporté vers la Comptabilité (Module 14).
- **Pas de champ technicien structuré.** Le technicien est un simple texte libre (dans l'historique) ; les colonnes serveur `technicien_interne_id` et `fournisseur_externe_id` existent mais **ne sont pas exposées** dans les formulaires.
- **Certains filtres restent en français même en anglais.** Les menus de filtre de statut de demande, de statut d'intervention et de type d'événement sont **codés en dur** (non traduits) : ils s'affichent en français quelle que soit la langue de l'interface.

### 1.4 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **Maintenance** (icône `Wrench`). Réf. `Sidebar.tsx:68,76`.
- URL directe : `/maintenance`.
- Titre de la page : **« Maintenance »**.
- **Onglet par défaut** : Tableau de bord.

> **Un tenant qui n'a jamais ouvert la page a des tables vides.** Les 8 tables `maintenance_*` sont créées **à la première utilisation** (à la volée, en tête de chaque point d'accès). Un tenant neuf voit des compteurs à **0** et des listes vides : c'est normal, pas une erreur. Ces tables ne sont **pas** créées à l'ouverture du tenant ni par la réparation automatique de démarrage (même patron que le module Métré).

### 1.5 Permissions et rôles

> **Attention — écriture ouverte à tout le tenant.** Contrairement aux autres modules du même fichier (Logistique, Location…), **aucun** des 37 points d'accès de la Maintenance n'a de garde de rôle. La seule protection est l'authentification et le **mode consultation** global.

| Action | Qui peut la faire |
|--------|-------------------|
| **Consulter** (tous les onglets, tous les GET) | Tout utilisateur authentifié du tenant (`get_current_user`). |
| **Créer / modifier / supprimer** un type, une planification, une demande, une intervention, une pièce, une alerte, un relevé, une entrée d'historique | **Tout utilisateur authentifié du tenant** — il n'y a **aucun** `require_role` ni `require_tenant_admin_or_role` sur ces points d'accès (`secondary.py:6208-7993`). |
| **Utiliser l'assistant IA** | Tout utilisateur authentifié **ayant des crédits IA** (voir 1.7). |

> **Rappel de contexte.** Le module Logistique, dans le **même fichier**, protège sa propre maintenance d'équipement par `require_tenant_admin_or_role(*LOGISTICS_WRITE_ROLES)`. Ici, ce n'est pas le cas : un simple employé peut supprimer une demande ou une planification. Tenez-en compte dans votre organisation du travail.

> **Le super-administrateur de la plateforme ne peut pas utiliser le CRUD.** Chaque point d'accès exige un **contexte tenant** (`if not user.schema → 400 « Contexte tenant manquant »`). Un super-administrateur de la plateforme (sans schéma de tenant) reçoit donc **400** sur les opérations de gestion ; il reste toutefois exempté côté crédits pour l'assistant IA.

> **Mode consultation (lecture seule) à l'échelle du tenant.** Si l'entreprise n'a pas d'abonnement Stripe actif (annulé ou absent), tout le tenant passe en **mode consultation** : les lectures restent permises, mais **toute** requête d'écriture (POST / PUT / DELETE) renvoie **403** (`erp_auth.py:526`). Une entreprise **désactivée** renvoie **401**. Comme les cinq points d'accès de l'assistant IA sont des POST, ils tombent aussi sous cette règle : **l'assistant IA n'est pas utilisable en mode consultation.** Cette garde est **globale** (dans `get_current_user`), ce qui compense l'absence de garde de rôle locale sur l'écriture.

### 1.6 Les 11 onglets

Source : `MaintenancePage.tsx:150-162` (tableau `tabs`) et `:188-198` (rendu). Les valeurs entre parenthèses sont des compteurs dynamiques.

| # | Clé interne | Libellé affiché | Icône | Compteur | Contenu réel |
|---|-------------|-----------------|-------|----------|--------------|
| 1 | `dashboard` | Tableau de bord | `BarChart3` | — | Indicateurs + demandes urgentes + planifications dues + dernières alertes |
| 2 | `types` | Types | `Settings` | — | Catalogue des procédures-types de maintenance |
| 3 | `planification` | Planification (N) | `Calendar` | planifications en retard | Échéances récurrentes par équipement (liste) |
| 4 | `demandes` | Demandes (N) | `ClipboardList` | demandes en attente | Demandes (tickets) + détail (mise à jour, pièces, interventions) |
| 5 | `interventions` | Interventions (N) | `Wrench` | interventions en cours | Interventions exécutées (édition et suppression) |
| 6 | `pieces` | Pièces | `Package` | — | Vue globale des pièces consommées (lecture) |
| 7 | `alertes` | Alertes (N) | `Bell` | alertes non lues | Alertes préventives + bouton « Générer les alertes » |
| 8 | `historique` | Historique | `History` | — | Historique chronologique par équipement |
| 9 | `compteurs` | Compteurs | `Gauge` | — | Relevés d'heures / kilomètres / cycles |
| 10 | `stats` | Statistiques | `BarChart3` | — | Dix indicateurs + répartitions statut / priorité |
| 11 | `ia` | Assistant IA | `Sparkles` | — | 5 outils d'aide (payants aux crédits IA) |

> **Badges dynamiques.** Le badge `(N)` de Planification affiche le nombre de planifications en retard ; celui de Demandes, le nombre de demandes en attente ; celui d'Interventions, le nombre d'interventions en cours ; celui d'Alertes, le nombre d'alertes non lues. Un badge n'apparaît que si le nombre est supérieur à zéro.

### 1.7 Coûts et facturation (assistant IA seulement)

- **Toute la partie registre est gratuite** : gérer les types, les planifications, les demandes, les interventions, les pièces, les alertes, l'historique et les compteurs ne consomme aucun crédit.
- **Seul l'assistant IA est payant.** Chaque appel d'un des outils consomme des **crédits IA prépayés** de l'entreprise (portefeuille partagé avec les autres assistants de l'ERP). Le coût est le tarif du modèle utilisé **majoré de 30 %** :
  - Outils **Claude Sonnet 4-6** (discussion, liste de vérification) : environ 3 $ US par million de jetons en entrée, 15 $ US par million en sortie, × 1,30.
  - Outils **Claude Opus 4-8** (diagnostic, plan préventif, estimation de coût — et le point d'accès dormant d'analyse d'intervention) : environ 5 $ US par million de jetons en entrée, 25 $ US par million en sortie, × 1,30 (plus l'éventuelle mise en cache).
- L'usage est tracé sous une fonctionnalité `maintenance_chat`, `maintenance_diagnose`, `maintenance_preventive`, `maintenance_checklist` ou `maintenance_estimate_cost` dans le suivi d'usage IA du super-administrateur.
- Un compte **sans crédits** reçoit une erreur **402** « Crédits IA insuffisants » et ne peut pas lancer l'outil ; le registre de maintenance, lui, reste gratuit.

> **Attention — pas d'idempotence sur le débit IA.** Le débit des crédits est effectué **sans clé d'idempotence** : un **double-clic** ou une **reprise réseau** sur un même appel peut **débiter deux fois** (le tarif étant déjà majoré de 30 %). De plus, un échec de débit est **silencieusement ignoré** (la réponse est quand même renvoyée). Attendez la réponse (ou le message d'erreur) avant de relancer un outil.

### 1.8 Architecture technique

```
Frontend  MaintenancePage.tsx (1960 lignes, 11 onglets, un seul fichier)
    │
    ├── Tableau de bord / Types / Planification / Demandes / Interventions /
    │   Pièces / Alertes / Historique / Compteurs / Statistiques
    │        └─ api/maintenance.ts ──> secondary.py  /api/erp/v1/maintenance/*   (31 endpoints gestion + statistiques)
    │                                   tables maintenance_* (8, créées à la demande, par tenant)
    │
    └── onglet Assistant IA
             └─ api/maintenance.ts ──> secondary.py  /api/erp/v1/maintenance/ia/*  (6 endpoints POST, 5 câblés)
                                        Claude sonnet-4-6 (chat / checklist)
                                        Claude opus-4-8  (diagnose / preventive / analyze-intervention / estimate-cost)
                                        débit des crédits IA prépayés + traçage de l'usage
```

> **Point d'attention pour un tenant neuf.** Les 8 tables `maintenance_*` sont créées **en tête de chaque point d'accès** (`_ensure_maintenance_tables`, `secondary.py:1899`). Elles ne sont **pas** créées à l'ouverture du tenant ni par la réparation automatique de démarrage. Concrètement : dès la première action (par exemple créer un type), tout se met en place ; avant cela, les compteurs restent à zéro.

---

## 2. Interface

Source : `MaintenancePage.tsx` (1960 lignes, sous-composants en ligne).

### 2.1 Disposition générale

- **Titre** « Maintenance » toujours affiché en haut (`:166`). Le sous-titre du module n'est jamais rendu.
- **Barre d'onglets** défilable horizontalement sur petit écran ; l'onglet actif est souligné en bleu.
- **Bandeau d'erreur** (rouge, fermable) au-dessus des onglets après une action ratée ; il se réinitialise au changement d'onglet.
- Chaque onglet possède, en règle générale, une **barre de commandes** (bouton d'action principal à gauche) et, à droite, un **champ de recherche** et un ou plusieurs **filtres**. La recherche est **locale** (elle filtre la page déjà chargée), sauf le filtre de statut des Demandes qui est appliqué **côté serveur**.
- **Adaptatif** : les tableaux (affichage bureau) se transforment en cartes empilées sur téléphone.

### 2.2 Onglet Tableau de bord

Source : `:207-305`. Nourri par `GET /maintenance/statistics` et les listes déjà chargées.

**Quatre cartes d'indicateurs** :

| Carte | Valeur | Couleur |
|-------|--------|---------|
| Interventions | Interventions du mois | bleu |
| En cours | Demandes au statut `EN_COURS` | vert |
| Demande | Demandes en attente | jaune |
| Alertes non lues | Alertes ni lues ni traitées | rouge |

**Trois cartes de listes (les 5 premiers éléments)** :

- **Demandes** (icône `AlertTriangle`) : jusqu'à 5 demandes **urgentes** (priorité `CRITIQUE` ou `HAUTE`), avec le titre, « {numéro} - {description} » et des badges de priorité et de statut. Vide : « Aucune donnée ».
- **Planifications dues** : jusqu'à 5 planifications actives dont la prochaine échéance est déjà atteinte, avec le nom, « Prévue : {date} » et un badge de priorité. Vide : « Aucune planification en retard ».
- **Dernières alertes** : jusqu'à 5 alertes récentes, avec le titre, le message et un badge de priorité. Vide : « Aucune alerte active ».

### 2.3 Onglet Types

Source : `:311-488`. Catalogue des **procédures-types** de maintenance (par exemple « Vidange moteur 250 h », « Inspection électrique annuelle »).

**Barre de commandes** : bouton **« Nouveau type »**. À droite : recherche (« Rechercher... ») et filtre **Catégorie** (Toutes / Préventive / Corrective / Prédictive, filtre local).

**Tableau « Types de maintenance »**, colonnes :

| Colonne | Contenu |
|---------|---------|
| Nom | Nom du type |
| Catégorie | Badge : `CORRECTIVE` (jaune), `PREDICTIVE` (sarcelle), sinon bleu (`PREVENTIVE`) |
| Fréquence | « {n} jours » ou « - » |
| Durée est. | « {n} h » ou « - » |
| Coût est. | Montant en dollars ou « - » |
| Actions | Éditer · Supprimer |

Liste vide : « Aucun type de maintenance. Créez-en un pour commencer. »

**Fenêtre « Nouveau type » / « Modifier type »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Nom | texte | **Oui** (seul champ obligatoire) |
| Description | zone de texte | non |
| Catégorie | menu (Préventive / Corrective / Prédictive) | non |
| Fréquence (jours) · Durée estimée (h) | nombre · nombre | non |
| Coût estimé ($) | nombre | non |
| Compétences requises | zone de texte | non |

Le bouton Créer / Mettre à jour est **désactivé tant que le nom est vide** (avec une garde contre le double envoi).

> **Champs cachés.** Les colonnes serveur `checklist_json` et `pieces_requises_json` existent (et l'API les accepte, `api/maintenance.ts:38,42`) mais **ne figurent pas** dans le formulaire.

> **Suppression = désactivation.** La suppression demande « Désactiver ce type de maintenance ? » et effectue une **suppression douce** (`actif = FALSE`) : le type disparaît de la liste (filtre par défaut sur les actifs) mais reste en base et référençable par les planifications existantes.

### 2.4 Onglet Planification

Source : `:494-731`. Échéances de maintenance **récurrentes** par équipement.

**Barre de commandes** : bouton **« Nouvelle planification »**. À droite : recherche et filtre **Priorité** (Toutes / Basse / Normale / Haute / Critique).

**Tableau « Planifications »**, colonnes :

| Colonne | Contenu |
|---------|---------|
| Nom | Nom de la planification |
| Équipement | « {type} #{id} » |
| Fréquence | « {valeur} {type} » (jours / semaines / mois / heures d'utilisation / kilomètres) |
| Prochaine échéance | Date ; **en rouge** avec badge **« En retard »** si elle est passée |
| Priorité | Badge |
| Actions | Éditer · Supprimer |

Liste vide : « Aucune planification. »

**Fenêtre « Nouvelle planification »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Nom | texte | **Oui** |
| Type d'équipement | menu (Inventaire / Location / Véhicule) | **Oui** |
| ID équipement | nombre | **Oui** |
| Type de maintenance | menu (« Tous » + les types créés) | non |
| Description | zone de texte | non |
| Type de fréquence | menu (Jours / Semaines / Mois / Heures utilisation / Kilomètres) | non |
| Valeur | nombre (≥ 1) | non |
| Date de début · Prochaine échéance | date · date | non |
| Seuil alerte (jours) · Priorité | nombre · menu | non |

Le bouton Créer est **désactivé tant que le nom ou l'ID d'équipement est manquant**.

> **Calcul automatique de la prochaine échéance** (à la création, si le champ est laissé vide) : pour `JOURS`, `date de début + valeur jours` ; pour `SEMAINES`, `+ valeur × 7 jours` ; pour `MOIS`, `+ valeur × 30 jours` (**approximation** — ce n'est pas un vrai mois calendaire). Pour `HEURES_UTILISATION` et `KILOMETRES`, aucune date n'est calculée (ces fréquences sont gérées par la voie « usage » : voir 3.9 et 4.6).

> **Validation.** Une valeur de fréquence `≤ 0` est refusée (**400**). La suppression demande « Supprimer cette planification ? » et est **définitive** (suppression douce côté serveur, disparition de la liste).

### 2.5 Onglet Demandes

Source : `:737-923`. Demandes de maintenance (les « tickets » : corrective, préventive ou urgente).

**Barre de commandes** : bouton **« Nouvelle demande »**. À droite : recherche et filtre **Statut** — ce filtre est appliqué **côté serveur** (il relance `GET /maintenance/requests`).

**Tableau « Demandes de maintenance »**, colonnes :

| Colonne | Contenu |
|---------|---------|
| Numéro | `MR-#####` (police à chasse fixe) |
| Titre | Titre de la demande |
| Type | Type de maintenance |
| Priorité | Badge |
| Statut | Badge (voir 4.1) |
| Date | Date de la demande |
| Actions | Œil (Détail) · Supprimer |

Liste vide : « Aucune demande. »

**Fenêtre « Nouvelle demande »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Titre | texte | non (auto-généré des 80 premiers caractères de la description si vide) |
| Description | zone de texte | **Oui** |
| Symptômes | zone de texte | non |
| Type de maintenance | menu (Corrective / Préventive / Urgente) | non |
| Priorité | menu | non |
| Type d'équipement · ID équipement | menu · nombre | non |
| Coût estimé ($) | nombre | non |

Le bouton Créer est **désactivé tant que la description est vide**. À la création, le serveur attribue le numéro `MR-#####` et pose le statut `DEMANDE`.

> **Suppression conditionnelle.** La suppression demande « Supprimer cette demande ? ». Le serveur **refuse (400)** de supprimer une demande **`EN_COURS`** ou **`TERMINE`**. Pour supprimer, ramenez d'abord la demande à un autre statut (`DEMANDE`, `APPROUVE`, `PLANIFIE`, `EN_ATTENTE_PIECES` ou `ANNULE`). La suppression retire en cascade les pièces et les interventions de la demande.

#### Fenêtre « Demande {numéro} » (détail)

Source : `RequestDetailModal`, `:929-1114` (grande taille). C'est le cœur du travail correctif. Elle contient quatre cartes.

1. **Infos (lecture seule)** : Titre, Priorité (badge), Type, Équipement (« #id »), Description et, s'ils sont présents, les Symptômes.
2. **Mise à jour** (grille de trois éléments) : un menu **Statut**, un champ **Coût réel ($)** et un bouton **Enregistrer** (qui n'envoie que les champs modifiés) ; plus une zone **Solution**.
3. **Pièces ({n}) — Total : {montant}** : un bouton **Ajouter** ouvre un formulaire **Nom pièce** (obligatoire) + Référence + Quantité + Coût unitaire + Enregistrer. Le tableau liste Nom, Référence, Quantité, Coût et un bouton **✕** pour retirer une ligne. Le coût total d'une ligne = quantité × coût unitaire.
4. **Interventions ({n})** : un bouton **Nouvelle** ouvre un formulaire **Type** (« Ex. : Révision, Réparation... ») + **Description** (obligatoire) + Enregistrer. La liste affiche le type (ou « Intervention »), un badge de statut, le descriptif des travaux et la date. Vide : « Aucune intervention enregistrée ».

> **La saisie des pièces et des interventions se fait ici, pas ailleurs.** Les onglets Pièces et Interventions ne servent qu'à consulter (et, pour les interventions, à éditer / supprimer). Le point d'entrée de création, c'est cette fenêtre de détail.

### 2.6 Onglet Interventions

Source : `:1120-1259`. Interventions exécutées, rattachées à une demande.

**Barre de commandes** : **aucun bouton d'action** (la création se fait depuis le détail d'une demande). À droite : recherche et filtre **Statut** (Tous / En cours / Terminé / Reporté — libellés codés en dur).

**Tableau « Interventions »**, colonnes :

| Colonne | Contenu |
|---------|---------|
| Demande | Numéro de la demande parente (ou « #id ») |
| Type | Type d'intervention (texte libre) |
| Description | Descriptif des travaux |
| Durée | « {n} h » ou « - » |
| Statut | Badge (En cours / Terminé / Reporté) |
| Date | Date d'intervention |
| Actions | Éditer · Supprimer |

Liste vide : « Aucune intervention. »

**Fenêtre « Modifier intervention »** (édition seule) : un menu **Statut** (En cours / Terminé / Reporté), un champ **Temps passé (h)** et une zone **Rapport**.

> **Effet automatique majeur.** Passer une intervention à **`TERMINE`** clôture automatiquement la **demande parente** (statut `TERMINE`, date de fin posée) et **inscrit une ligne d'historique** (voir 4.7). La suppression demande « Supprimer cette intervention ? » et retire en cascade les pièces rattachées à l'intervention.

### 2.7 Onglet Pièces

Source : `:1265-1350`. **Vue globale en lecture** des pièces consommées.

**Barre de commandes** : **aucun bouton**, seulement la recherche.

**Carte « Pièces utilisées »** avec, en haut à droite, « Coût total : {somme} » (somme des coûts totaux des lignes filtrées). Colonnes du tableau : **Pièce, Référence, Demande (#id), Quantité, Coût unit., Coût total**, plus une action Supprimer. Liste vide : « Aucune pièce enregistrée. »

> **Pas de création ici.** Pour ajouter une pièce, ouvrez le détail de la demande concernée (2.5). La suppression demande « Supprimer cette pièce ? ».

### 2.8 Onglet Alertes

Source : `:1356-1452`. Alertes préventives issues des planifications dont l'échéance approche ou est dépassée.

**Barre de commandes** : bouton **« Générer les alertes »** (icône `Zap`, appelle `POST /maintenance/alertes/generate`). À droite : case à cocher **« Non lues »** et filtre **Priorité**. Après génération, un bandeau vert annonce « {n} alerte(s) générée(s) ».

**Liste (cartes, pas tableau)** : pour chaque alerte, un badge de priorité, le titre (avec un badge **« Terminé »** si elle est traitée), le message, « {type d'alerte} - {date} » et, le cas échéant, « Prochaine échéance : {date} ». Le fond est **ambré** tant que l'alerte n'est pas lue. Boutons par alerte : **« Marquer comme lue »** (si non lue) et **« Marquer comme traitée »** (si non traitée). Liste vide : « Aucune alerte. »

> **Ce que fait « Générer les alertes ».** Le serveur parcourt les planifications actives dont la prochaine échéance tombe dans la fenêtre `≤ aujourd'hui + seuil d'alerte` (défaut 7 jours) et qui n'ont pas déjà une alerte non traitée. Pour chacune, il crée une alerte **`MAINTENANCE_RETARD`** si l'échéance est passée, sinon **`MAINTENANCE_DUE`**. L'opération est **idempotente** (un index unique partiel empêche les doublons) et plafonnée à 500 alertes par appel.

### 2.9 Onglet Historique

Source : `:1458-1587`. Historique chronologique des événements par équipement.

**Barre de commandes** : bouton **« Historique »** (icône `Plus`, ouvre le formulaire d'ajout). À droite : recherche et filtre **Type d'événement** (Tous + Maintenance / Panne / Inspection / Remplacement / Mise en service / Mise hors service — libellés codés en dur).

**Liste (cartes)** : pour chaque événement, un badge de type (`PANNE` en rouge, `INSPECTION` en bleu, sinon vert), « {type} #{id} », la description, la date, « Technicien : {nom} », le coût et « {n} h ». Liste vide : « Aucun historique. »

**Fenêtre (ajout d'entrée)** : Type d'équipement + ID équipement ; **Type d'événement** (obligatoire) ; Description ; Coût ($) + Technicien (texte libre). Le bouton est **désactivé tant que l'ID d'équipement est manquant**.

> **Lecture seule après coup.** L'historique n'a **ni édition ni suppression** dans l'interface. Une entrée `MAINTENANCE` s'y ajoute **automatiquement** à la clôture d'une demande (voir 4.7). Pour corriger une erreur, ajoutez une entrée complémentaire.

### 2.10 Onglet Compteurs

Source : `:1645-1769`. Relevés d'usage (heures moteur, kilométrage, cycles) — le moteur de la maintenance « à l'usage ».

**Barre de commandes** : bouton **« Nouveau relevé »**. À droite : recherche. Après création, un bandeau annonce « Relevé enregistré. » ou, si un seuil de planification « à l'usage » est atteint, « Relevé enregistré. {n} alerte(s) de maintenance générée(s). »

**Carte « Relevés de compteurs »** avec la note « Saisissez les relevés d'heures / kilomètres. Les planifications « à l'usage » génèrent une alerte quand l'intervalle est atteint. » Colonnes du tableau : **Équipement, Type de compteur, Valeur** (« {n} {unité} », km / h / vide), **Date du relevé, Notes**. Liste vide : « Aucun relevé enregistré. »

**Fenêtre « Relevé de compteur »** : Type d'équipement ; ID équipement ; **Type de compteur** (Heures / Kilomètres / Cycles) ; Valeur ; Date du relevé ; Notes. Le bouton est **désactivé tant que l'ID d'équipement est manquant**.

> **Ajout seul.** Comme l'historique, les relevés ne se modifient ni ne se suppriment depuis l'interface.

> **Attention aux compteurs de type « Cycles ».** Le mécanisme d'alerte d'usage ne reconnaît que **Heures** (→ fréquence « heures d'utilisation ») et **Kilomètres** (→ fréquence « kilomètres »). Un compteur de type **Cycles n'est associé à aucune fréquence** et ne déclenchera donc **jamais** d'alerte automatique (voir 4.6). Vous pouvez tout de même l'enregistrer pour le suivi.

### 2.11 Onglet Statistiques

Source : `:1593-1639`. Nourri par `GET /maintenance/statistics` (`secondary.py:7909`).

**Dix cartes d'indicateurs** : Total demandes, En cours, En attente, Terminé (terminées ce mois), Coût total (coût réel), Coût estimé, Planification (planifications actives), En retard (planifications en retard), Alertes non lues, Interventions (du mois).

**Deux cartes de répartition** : **Par statut** (un badge de statut et son compte) et **Par priorité** (un badge de priorité et son compte). En l'absence de statistiques, un squelette s'affiche.

> Les indicateurs sont calculés **à la volée** à chaque appel (agrégations SQL), sans cache.

### 2.12 Onglet Assistant IA

Source : `IaAssistantTab`, `:1801-1959`. **Cinq outils** exposés, chacun avec un bouton **« Lancer »** désactivé tant que les champs requis ne sont pas remplis, un indicateur « Analyse en cours... », et un résultat affiché par un afficheur récursif (`IaJsonView`) qui met en forme le JSON structuré renvoyé. Un **avertissement permanent** est affiché : « Réponses générées par IA — à valider par un technicien qualifié. »

| Outil | Modèle | Ce qu'on saisit | Point d'accès |
|-------|--------|-----------------|---------------|
| **Discussion** | Claude Sonnet 4-6 | Une question libre (clavardage, `Entrée` = envoyer) | `POST /maintenance/ia/chat` |
| **Diagnostic** | Claude Opus 4-8 | Équipement + Symptômes observés (+ Historique, optionnel) | `POST /maintenance/ia/diagnose` |
| **Plan préventif** | Claude Opus 4-8 | Équipement + Utilisation (+ Dernière maintenance, optionnel) | `POST /maintenance/ia/preventive` |
| **Liste de vérification** | Claude Sonnet 4-6 | Type de maintenance + Équipement | `POST /maintenance/ia/checklist` |
| **Estimation de coût** | Claude Opus 4-8 | Équipement + Problème (+ Urgence, optionnel) | `POST /maintenance/ia/estimate-cost` |

Pour l'outil **Discussion**, la zone conserve l'historique des échanges de la session (messages utilisateur et assistant) ; vide, elle affiche « Posez une question sur la maintenance d'un équipement. ».

> **Un sixième point d'accès existe mais n'est pas branché.** Le serveur possède aussi `POST /maintenance/ia/analyze-intervention` (`secondary.py:8294`, fonction cliente `iaAnalyzeIntervention`, `api/maintenance.ts:415`), qui analyse la qualité d'une intervention. **Aucun bouton de l'interface ne l'appelle** : l'onglet Assistant IA n'expose donc que **5 outils sur 6**.

> **Ce que l'assistant est et n'est pas.** C'est un **conseiller** : il rédige des diagnostics probables, des plans préventifs, des listes de vérification (ÉPI, cadenassage, inspections…) et des estimations de coût à partir de ce que vous saisissez. Il **n'écrit rien** dans vos données : il ne crée aucune demande, aucune intervention, aucune pièce. Les résultats sont des **brouillons de réflexion**, à valider par un technicien qualifié — surtout les coûts et la sécurité.

> **Erreurs possibles.** 402 « Crédits IA insuffisants » (rechargez le solde) ; 503 si le service IA n'est pas configuré ; 413 si la demande est trop volumineuse ; 403 en mode consultation (voir 1.5).

### 2.13 Éléments transverses

- **Recherche** : toujours **locale** (filtre la page déjà chargée), sauf le filtre de statut des Demandes qui interroge le serveur.
- **Filtres codés en dur** : les menus de statut de demande, de statut d'intervention et de type d'événement restent en **français** même quand l'interface est en anglais.
- **Aucune exportation, impression, CSV, PDF, téléversement, photo ni action groupée** nulle part.

---

## 3. Workflows pas à pas

### 3.1 Définir un type de maintenance

1. Onglet **Types** → **« Nouveau type »**.
2. Saisir le **Nom** (obligatoire) ; au besoin la description, la **catégorie** (Préventive / Corrective / Prédictive), la fréquence en jours, la durée estimée, le coût estimé et les compétences requises.
3. **Créer** (`POST /maintenance/types`). Le type devient référençable depuis les planifications.

### 3.2 Planifier une maintenance préventive récurrente

1. Onglet **Planification** → **« Nouvelle planification »**.
2. Saisir le **Nom** (obligatoire), le **Type d'équipement** (Inventaire / Location / Véhicule) et l'**ID équipement** (obligatoire) ; choisir un **Type de maintenance** (optionnel), un **Type de fréquence** et une **Valeur** ; renseigner la **Date de début** et, si vous le souhaitez, la **Prochaine échéance** (sinon elle sera calculée), le **Seuil d'alerte** (défaut 7 jours) et la **Priorité**.
3. **Créer** (`POST /maintenance/planification`). Pour une fréquence en jours / semaines / mois, la prochaine échéance se calcule automatiquement si vous l'avez laissée vide ; pour heures d'utilisation / kilomètres, l'échéance sera pilotée par les compteurs (voir 3.9).

### 3.3 Ouvrir une demande corrective (panne)

1. Onglet **Demandes** → **« Nouvelle demande »**.
2. Saisir la **Description** (obligatoire ; le Titre se remplit tout seul à partir d'elle si vous le laissez vide) ; au besoin les Symptômes, le **Type de maintenance**, la **Priorité**, l'équipement (Type + ID) et un coût estimé.
3. **Créer** (`POST /maintenance/requests`). Le serveur attribue le numéro `MR-#####` et pose le statut `DEMANDE`.

### 3.4 Faire avancer une demande (statut, coût, solution)

1. Onglet **Demandes** → cliquer sur l'**Œil** de la ligne pour ouvrir le détail.
2. Dans la carte **Mise à jour**, choisir le nouveau **Statut** (`DEMANDE` → `APPROUVE` → `PLANIFIE`… ou `EN_ATTENTE_PIECES`, `ANNULE`), saisir au besoin le **Coût réel** et la **Solution**, puis **Enregistrer** (`PUT /maintenance/requests/{id}` — seuls les champs modifiés sont envoyés).

> **Aucune règle de transition.** Le serveur accepte n'importe quel statut de la liste ; c'est l'interface qui suggère l'ordre logique. Clôturer une demande en la passant directement à `TERMINE` inscrit une ligne d'historique (voir 4.7).

### 3.5 Ajouter une pièce consommée

1. Ouvrir le **détail** de la demande → carte **Pièces** → **Ajouter**.
2. Saisir le **Nom pièce** (obligatoire) ; au besoin la référence, la quantité et le coût unitaire.
3. **Enregistrer** (`POST /maintenance/pieces`). Le coût total se calcule (quantité × coût unitaire). Si la pièce est reliée à un article d'inventaire, **le stock est décrémenté** (voir 5.1).

### 3.6 Consigner une intervention

1. Ouvrir le **détail** de la demande → carte **Interventions** → **Nouvelle**.
2. Saisir le **Type** (par exemple « Révision », « Réparation ») et la **Description** des travaux (obligatoire).
3. **Enregistrer** (`POST /maintenance/interventions`). L'intervention naît au statut `EN_COURS` et, si la demande était `DEMANDE` / `APPROUVE` / `PLANIFIE`, **elle passe automatiquement à `EN_COURS`** avec une date de début.

### 3.7 Clôturer une intervention (et la demande)

1. Onglet **Interventions** → **Éditer** la ligne.
2. Passer le **Statut** à `TERMINE` (ou `REPORTE`), renseigner le **Temps passé (h)** et le **Rapport**.
3. **Enregistrer** (`PUT /maintenance/interventions/{id}`). Au passage à `TERMINE`, la **demande parente est clôturée automatiquement** (`TERMINE`, date de fin) et une **ligne d'historique** est inscrite (type `MAINTENANCE`, avec l'équipement, la description, le coût réel et le temps réel).

### 3.8 Générer et traiter les alertes préventives

1. Onglet **Alertes** → **« Générer les alertes »**.
2. Le serveur crée les alertes des planifications dues ou en retard (voir 2.8) et affiche « {n} alerte(s) générée(s) ».
3. Sur chaque carte : **« Marquer comme lue »** puis, une fois l'action faite, **« Marquer comme traitée »**. Une alerte traitée sort du compteur « Alertes non lues » et affiche le badge « Terminé ».

> **À déclencher régulièrement.** Il n'y a **aucun cron** : rien ne génère les alertes d'échéance automatiquement. Prenez l'habitude de cliquer sur « Générer les alertes » (par exemple une fois par jour ouvré). Tant qu'une alerte non traitée existe pour une planification, une nouvelle exécution **ne créera pas de doublon**.

### 3.9 Suivre l'usage et déclencher une alerte à l'usage

1. **Prérequis** : une planification dont le **Type de fréquence** est **Heures utilisation** ou **Kilomètres**, sur l'équipement visé (voir 3.2).
2. Onglet **Compteurs** → **« Nouveau relevé »** → saisir le Type d'équipement, l'ID, le **Type de compteur** (Heures ou Kilomètres), la **Valeur** relevée et la date.
3. **Enregistrer** (`POST /maintenance/compteurs`). Le **premier** relevé fixe la valeur de référence (aucune alerte). Ensuite, dès que `valeur − référence ≥ intervalle` de la planification, une alerte **`MAINTENANCE_DUE`** est créée et la référence est réavancée — le bandeau indique alors « {n} alerte(s) de maintenance générée(s) ».

> **Rappel : les compteurs « Cycles » ne déclenchent rien.** Seuls Heures et Kilomètres sont reliés à une fréquence (voir 4.6). Un relevé en Cycles est conservé mais ne produit jamais d'alerte.

### 3.10 Saisir manuellement un événement d'historique

1. Onglet **Historique** → **« Historique »** (bouton `+`).
2. Saisir le Type d'équipement et l'**ID** (obligatoire), le **Type d'événement** (Maintenance / Panne / Inspection / Remplacement / Mise en service / Mise hors service), la description, le coût et le technicien.
3. **Enregistrer** (`POST /maintenance/historique`). Cas d'usage typiques : mise en service ou hors service d'un équipement, panne constatée hors du processus formel, inspection externe.

### 3.11 Utiliser l'assistant IA

1. Onglet **Assistant IA** → choisir l'outil (Discussion / Diagnostic / Plan préventif / Liste de vérification / Estimation de coût).
2. Renseigner les champs requis (par exemple, pour le Diagnostic : l'équipement et les symptômes observés).
3. **Lancer**. La réponse s'affiche (texte ou fiche structurée) ; le **coût** est débité des crédits IA de l'entreprise.

> N'appuyez qu'une fois sur **« Lancer »** et attendez la réponse : le débit n'a pas de protection contre le double-clic (voir 1.7). Validez toujours le résultat avec un technicien qualifié.

### 3.12 Comprendre un refus (403 / mode consultation / 402)

- **Toutes** les écritures échouent pour **tout le monde**, et l'assistant IA renvoie 403 : le tenant est en **mode consultation** (abonnement Stripe inactif). Régularisez l'abonnement pour rétablir l'écriture (module Configuration / Abonnement).
- **401** partout : l'entreprise est **désactivée**.
- **400 « Contexte tenant manquant »** : vous êtes connecté comme super-administrateur de la plateforme (sans tenant) ; le CRUD de maintenance n'est pas accessible dans ce contexte.
- **402 « Crédits IA insuffisants »** dans l'assistant : rechargez le solde de crédits IA de l'entreprise. Le registre de maintenance, lui, reste gratuit.

---

## 4. Référence

### 4.1 Statuts et énumérations

Toutes ces valeurs sont validées côté serveur (Pydantic, en **MAJUSCULES**, sensibles à la casse) : une valeur hors liste renvoie **422** avant même d'atteindre la base.

| Ensemble | Valeurs | Défaut |
|----------|---------|--------|
| **Statut de demande** (7) | `DEMANDE`, `APPROUVE`, `PLANIFIE`, `EN_COURS`, `EN_ATTENTE_PIECES`, `TERMINE`, `ANNULE` | `DEMANDE` |
| **Statut d'intervention** (3) | `EN_COURS`, `TERMINE`, `REPORTE` | `EN_COURS` |
| **Priorité** (4) | `BASSE`, `NORMALE`, `HAUTE`, `CRITIQUE` | `NORMALE` |
| **Type de maintenance (demande)** (3) | `PREVENTIVE`, `CORRECTIVE`, `URGENTE` | `CORRECTIVE` |
| **Catégorie (type / catalogue)** (3) | `PREVENTIVE`, `CORRECTIVE`, **`PREDICTIVE`** | `PREVENTIVE` |
| **Type d'équipement** (3) | `INVENTORY`, `LOCATION`, `VEHICULE` | `INVENTORY` |
| **Type de fréquence** (5) | `JOURS`, `SEMAINES`, `MOIS`, `HEURES_UTILISATION`, `KILOMETRES` | `JOURS` |
| **Type d'événement (historique)** (6) | `MAINTENANCE`, `PANNE`, `INSPECTION`, `REMPLACEMENT`, `MISE_EN_SERVICE`, `MISE_HORS_SERVICE` | `MAINTENANCE` |
| **Type de compteur** (3) | `HEURES`, `KILOMETRES`, `CYCLES` | `HEURES` |
| **Type d'alerte** (5) | `MAINTENANCE_DUE`, `MAINTENANCE_RETARD`, `PANNE`, `INSPECTION_REQUISE`, `GARANTIE_EXPIRATION` | — |

> **« PREDICTIVE » est le seul endroit où « prédictive » existe** : c'est une catégorie de catalogue. Aucune logique prédictive ne s'y rattache (voir 1.1).

**Couleurs des badges de statut de demande** (`STATUT_COLORS`, `:40-43`) : `DEMANDE` jaune, `APPROUVE` bleu, `PLANIFIE` sarcelle, `EN_COURS` vert, `EN_ATTENTE_PIECES` ambre, `TERMINE` vert, `ANNULE` gris. **Couleurs de priorité** (`:113-118`) : `CRITIQUE` rouge, `HAUTE` jaune, `NORMALE` bleu, `BASSE` gris.

> **Aucune règle de transition.** Hormis les gardes de suppression (demande non supprimable en `EN_COURS`/`TERMINE`) et les auto-clôtures, n'importe quel statut de la liste peut être posé. La séquence recommandée reste `DEMANDE → APPROUVE → PLANIFIE → EN_COURS → TERMINE`.

### 4.2 Cycle de vie d'une demande (automatismes)

| Déclencheur | Effet automatique |
|-------------|-------------------|
| Création d'une demande | Statut `DEMANDE`, numéro `MR-#####` |
| **Création d'une intervention** sur la demande | Si la demande est `DEMANDE` / `APPROUVE` / `PLANIFIE` → passe à **`EN_COURS`** + date de début |
| **Intervention passée à `TERMINE`** | Demande → **`TERMINE`** + date de fin + **ligne d'historique** (au mieux) |
| Demande passée manuellement à `TERMINE` | **Ligne d'historique** (copie équipement / description / coût réel / temps réel) |
| Suppression d'une demande | **Refusée (400)** si `EN_COURS` / `TERMINE` ; sinon cascade pièces → interventions → demande |

> **Statuts non régressifs.** Les auto-clôtures ne rétrogradent jamais un état terminal : une intervention en retard ou rejouée ne « réveille » pas une demande déjà `TERMINE` ou `ANNULE`. L'insertion d'historique est protégée contre les doublons (elle ne se fait qu'au **vrai** passage vers `TERMINE`).

### 4.3 Calcul de la prochaine échéance

`_compute_next_maintenance` (`secondary.py:6180`) :

| Fréquence | Calcul de la prochaine échéance |
|-----------|--------------------------------|
| `JOURS` | date de début + `valeur` jours |
| `SEMAINES` | date de début + `valeur × 7` jours |
| `MOIS` | date de début + `valeur × 30` jours (**approximation**, pas un mois calendaire) |
| `HEURES_UTILISATION` | **aucune date** (pilotée par les compteurs) |
| `KILOMETRES` | **aucune date** (pilotée par les compteurs) |

### 4.4 Génération des alertes — les deux voies

| Voie | Déclenchement | Logique | Type d'alerte |
|------|---------------|---------|---------------|
| **Par échéance** (`generate_maintenance_alertes`, `:7787`) | Bouton **« Générer les alertes »** | Planifications actives dont `prochaine_maintenance ≤ aujourd'hui + seuil (défaut 7 j)` **et** sans alerte non traitée. Insertion en lot, `ON CONFLICT DO NOTHING`, plafond 500. | `MAINTENANCE_RETARD` si dépassée, sinon `MAINTENANCE_DUE` |
| **Par usage** (`_maybe_generate_usage_alerts`, `:7517`) | Saisie d'un **relevé de compteur** | 1er relevé = référence (rien) ; ensuite si `valeur − référence ≥ intervalle` et pas d'alerte `MAINTENANCE_DUE` non traitée → alerte + réavance de la référence. Sérialisé (`FOR UPDATE`) contre les doublons concurrents. | `MAINTENANCE_DUE` |

**Correspondance compteur → fréquence** (`_COMPTEUR_TO_FREQUENCE`, `:7514`) : `HEURES → HEURES_UTILISATION`, `KILOMETRES → KILOMETRES`. **`CYCLES` n'est pas mappé** : aucune alerte d'usage pour ce type de compteur.

> **Anti-doublon à l'épreuve des accès concurrents.** Un index unique partiel `idx_maint_alertes_dedup` sur `(planification_id, type_alerte) WHERE traitee = FALSE AND planification_id IS NOT NULL` (`:1943`) empêche qu'une même planification ait deux alertes non traitées du même type, quel que soit le nombre d'exécutions concurrentes.

### 4.5 Statistiques (`GET /maintenance/statistics`, `:7909`)

L'agrégat renvoie : `total`, `par_statut`, `par_priorite`, `cout_reel` et `cout_estime` (sommes), `en_cours` (`EN_COURS`), `en_attente` (`DEMANDE` / `APPROUVE` / `PLANIFIE` / `EN_ATTENTE_PIECES`), `terminees_mois` (date de fin ≥ début du mois), `alertes_non_lues` (ni lues ni traitées), `planifications_actives`, `planifications_retard` (prochaine échéance < aujourd'hui) et `interventions_mois`.

### 4.6 Numéro de demande

`MR-#####` généré par `_gen_unique_numero` (`:962`, colonne `numero_demande` **UNIQUE**). Le serveur **sonde plusieurs candidats** contre la contrainte d'unicité et élargit l'entropie en secours : il n'y a **jamais** de `COUNT(*)+1` ni de `MAX(id)+1`, et le numéro obtenu est **garanti unique** (correction de la v2.0, qui décrivait à tort un tirage aléatoire sujet aux collisions).

### 4.7 Points d'accès — gestion et statistiques (31)

Préfixe réel : `/api/erp/v1`. Tous en `Depends(get_current_user)`, **sans garde de rôle**. Colonne « UI » = déclenchable depuis l'écran.

**Types (4)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/types` | Oui | Filtres `actif_only`, `categorie`. |
| POST | `/maintenance/types` | Oui | Crée un type ; seul `nom` est obligatoire. |
| PUT | `/maintenance/types/{id}` | Oui | Mise à jour. |
| DELETE | `/maintenance/types/{id}` | Oui | **Suppression douce** (`actif = FALSE`) ; 404 si absent. |

**Planification (4 + 1 alias)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/planification` | Oui | Jointure sur les types, tri par prochaine échéance. |
| POST | `/maintenance/planification` | Oui | Refuse `frequence_valeur ≤ 0` (**400**) ; calcule l'échéance si absente. |
| PUT | `/maintenance/planification/{id}` | Oui | Mise à jour. |
| DELETE | `/maintenance/planification/{id}` | Oui | Suppression douce. |
| GET | `/maintenance/preventive` | Non | **Alias hérité** qui délègue à la liste des planifications actives. |

**Demandes (5)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/requests` | Oui | Filtres `statut` / `equipement_type` / `equipement_id` ; `limit` borné 1-500. |
| GET | `/maintenance/requests/{id}` | Oui | Renvoie `{demande, pieces, interventions}`. |
| POST | `/maintenance/requests` | Oui | Numéro `MR-#####`, statut `DEMANDE`. |
| PUT | `/maintenance/requests/{id}` | Oui | Transaction atomique ; auto-historique au passage à `TERMINE`. |
| DELETE | `/maintenance/requests/{id}` | Oui | **400** si `EN_COURS` / `TERMINE` ; sinon cascade pièces + interventions. |

**Interventions (5)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/interventions` | Oui | Jointure sur la demande, `LIMIT 100`. |
| GET | `/maintenance/interventions/{id}` | Non | Renvoie l'intervention et ses pièces. |
| POST | `/maintenance/interventions` | Oui | 404 si demande parente absente ; passe la demande à `EN_COURS`. |
| PUT | `/maintenance/interventions/{id}` | Oui | `TERMINE` → clôture la demande + historique (SAVEPOINT). |
| DELETE | `/maintenance/interventions/{id}` | Oui | Cascade pièces. |

**Pièces (3)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/pieces` | Oui | Filtres `demande_id` / `intervention_id`, `LIMIT 200`. |
| POST | `/maintenance/pieces` | Oui | Calcule le coût total ; **décrémente l'inventaire** si `inventory_item_id`. |
| DELETE | `/maintenance/pieces/{id}` | Oui | **Pas de re-crédit** du stock. |

**Historique (2)** · **Compteurs (2)** · **Alertes (4)** · **Statistiques (1)**

| Méthode | Chemin | UI | Notes |
|---------|--------|----|-------|
| GET | `/maintenance/historique` | Oui | Filtres équipement, `limit` 1-500. |
| POST | `/maintenance/historique` | Oui | Saisie manuelle. |
| GET | `/maintenance/compteurs` | Oui | `LIMIT 100`. |
| POST | `/maintenance/compteurs` | Oui | **Déclenche les alertes d'usage** ; renvoie `alertes_generees`. |
| GET | `/maintenance/alertes` | Oui | Filtres `non_lues_only` / `priorite`, tri par priorité puis date, `LIMIT 100`. |
| POST | `/maintenance/alertes` | Non | Création manuelle d'une alerte. |
| PUT | `/maintenance/alertes/{id}` | Oui | Marquer lue / traitée (`traitee` → date de traitement). |
| POST | `/maintenance/alertes/generate` | Oui | Génération par échéance (voir 4.4). |
| GET | `/maintenance/statistics` | Oui | Agrégat (voir 4.5). |

### 4.8 Points d'accès — Assistant IA (6, dont 5 câblés)

Tous en **POST**, protégés par `get_current_user` + crédits IA, plafonnés à **32000 jetons** en sortie. En mode consultation, ils sont bloqués (403).

| Chemin | Modèle | Température | Sortie | UI |
|--------|--------|------------|--------|----|
| `/maintenance/ia/chat` | `claude-sonnet-4-6` | 0.4 | texte | Oui |
| `/maintenance/ia/diagnose` | `claude-opus-4-8` | 0.3 | JSON | Oui |
| `/maintenance/ia/preventive` | `claude-opus-4-8` | 0.3 | JSON | Oui |
| `/maintenance/ia/analyze-intervention` | `claude-opus-4-8` | 0.3 | JSON | **Non (dormant)** |
| `/maintenance/ia/checklist` | `claude-sonnet-4-6` | 0.3 | texte | Oui |
| `/maintenance/ia/estimate-cost` | `claude-opus-4-8` | 0.3 | JSON | Oui |

**Chaîne de facturation (identique aux six)** :

1. `check_ai_guard(user)` → **neutre en pratique** : renvoie toujours « autorisé » pour un utilisateur authentifié (`ai.py:824-825`). Ne bloque jamais.
2. `_check_credits(user)` → **le vrai gardien** : super-administrateur illimité ; instance interne (`BILLING_ENABLED=false`) illimitée ; sinon lit les crédits prépayés (`FOR UPDATE`), **recharge automatiquement** au besoin (Stripe), et renvoie **402 « Crédits IA insuffisants »** si le solde est épuisé.
3. Traçage de l'usage (`track_ai_usage`, au mieux).
4. Appel à Claude, **déporté hors de la boucle d'événements** (`asyncio.to_thread`).
5. **Débit des crédits** — **sans clé d'idempotence** et dans un `try/except` silencieux (voir l'avertissement en 1.7).

**Coût** : tarif du modèle **majoré de 30 %** — Sonnet `(entrée × 0,003 + sortie × 0,015) ÷ 1000 × 1,30` ; Opus `(entrée × 0,005 + sortie × 0,025) ÷ 1000 × 1,30` (plus la mise en cache éventuelle).

> **Anti-injection.** Les champs libres envoyés à l'IA (symptômes, description d'équipement…) sont **neutralisés** avant l'appel (`_prompt_safe` : espaces compactés, longueur bornée), et l'invite système rappelle la priorité à la sécurité et le recours à des techniciens certifiés. Les réponses JSON sont nettoyées de leurs balises ` ``` ` ; en cas d'échec d'analyse, l'objet `{raw, error}` est renvoyé (HTTP 200).

### 4.9 Tables PostgreSQL (par tenant, créées à la demande)

DDL et assistant : `secondary.py:1736-1948`.

| Table | Rôle |
|-------|------|
| `maintenance_types` | Catalogue des procédures-types (catégorie, fréquence en jours, durée, coût, compétences, `actif`). |
| `maintenance_planification` | Échéances récurrentes par équipement (fréquence, prochaine échéance, seuil, priorité, plus `derniere_maintenance_valeur` = référence d'usage). |
| `maintenance_demandes` | Demandes (`numero_demande` unique `MR-#####`, statut, priorité, coûts, solution…). |
| `maintenance_interventions` | Interventions rattachées à une demande (type, description, durée, statut, observations). |
| `maintenance_pieces` | Pièces consommées (nom, référence, `inventory_item_id`, quantité, coûts). |
| `maintenance_historique` | Événements par équipement (type, date, description, coût, technicien, compteurs). |
| `maintenance_compteurs` | Relevés d'usage (type de compteur, valeur, date). |
| `maintenance_alertes` | Alertes générées (type, priorité, échéance, `lue`, `traitee`). |

> **Créées en tête de chaque point d'accès** (`_ensure_maintenance_tables`), avec rattrapage défensif de colonnes (`ALTER TABLE ... ADD COLUMN IF NOT EXISTS`) pour les installations plus anciennes. Un tenant sans trafic Maintenance n'a tout simplement pas ces tables.

### 4.10 Validations, bornes et erreurs

| Règle | Effet |
|-------|-------|
| Valeur d'énumération hors liste (statut, priorité, type…) | **422** (avant la base) |
| Coût > 99 999 999,99 · heures > 999,99 · valeur de compteur > 9 999 999 999,99 · nombre négatif | **422** |
| Fréquence de planification `≤ 0` | **400** |
| Suppression d'une demande `EN_COURS` / `TERMINE` | **400** |
| Demande / type / planification / intervention / pièce / alerte inexistante (PUT / DELETE) | **404** |
| Intervention sans demande parente | **404** |
| Contexte tenant manquant (super-administrateur plateforme) | **400 « Contexte tenant manquant »** |
| Écriture en mode consultation / entreprise désactivée | **403** / **401** |
| Assistant IA sans crédits | **402 « Crédits IA insuffisants »** |
| Service IA non configuré | **503** |
| Demande IA trop volumineuse / service surchargé | **413** / **503** |
| Génération d'alertes par appel | plafond **500** |
| Listes (demandes / historique 1-500 ; interventions 100 ; pièces 200 ; compteurs 100 ; alertes 100) | plafonds fixes |

> **Défense contre l'injection SQL.** Les mises à jour dynamiques passent par des **listes blanches de colonnes** ; aucune valeur de champ ne construit du SQL. Les écritures multi-étapes (demande, intervention, pièce + stock, compteur + alertes, génération d'alertes) sont **atomiques**, et leurs effets de bord (historique, stock, alertes) sont isolés au mieux sous **SAVEPOINT** — un échec de l'effet de bord n'annule pas l'écriture principale.

### 4.11 Limites de débit et raccourcis

| Élément | Détail |
|---------|--------|
| `POST /maintenance/ia/*` (les 6 outils IA) | **10 requêtes par minute et par adresse IP** (`erp_api.py:362,470,678-681`, clé `{ip}:maintenance_ia`). |
| Autres points d'accès `/maintenance/*` | Borne générale élevée (≈ 1500 par minute et par IP). |
| Raccourci clavier | Dans l'outil Discussion de l'assistant IA, `Entrée` envoie le message. Aucun autre raccourci propre au module. |

---

## 5. Intégrations et FAQ

### 5.1 Magasin / Inventaire (Module 09)

- `maintenance_pieces.inventory_item_id` peut pointer vers un article d'inventaire. À la création d'une pièce reliée, le serveur **décrémente le stock** : `inventory_items.quantite_metric = GREATEST(0, quantite_metric − quantité)` (au mieux, après vérification que la colonne existe).
- **Pas de re-crédit à la suppression** d'une pièce : supprimer une ligne de pièce **ne rend pas** les unités au stock (à corriger par un mouvement d'entrée manuel dans le Magasin).
- **Aucune** liaison d'équipement : le couple Type + ID n'est jamais rapproché des produits du Magasin.

### 5.2 Logistique (Module 18) — deux systèmes de maintenance distincts

> **Ne pas confondre.** Il existe **deux** maintenances d'équipement dans l'ERP, dans le même fichier serveur mais **totalement séparées**.

- **Ce module (Maintenance, `/maintenance/*`)** : GMAO riche — types, planifications, demandes, interventions, pièces, alertes, historique, compteurs, IA — sur **8 tables** `maintenance_*`. **Écriture ouverte à tout employé.**
- **La maintenance de la Logistique (`/logistics/equipment/{id}/maintenance`)** : mini-suivi d'entretien de la **flotte interne** (interventions préventive / inspection / réparation / certification consignées sous un équipement de logistique, avec « prochaine date » qui met à jour l'échéance de l'équipement). Table **`logistics_equipment_maintenance`**, **protégée par rôle** (`require_tenant_admin_or_role`).
- **Aucune synchronisation** entre les deux : un même engin peut avoir des données des deux côtés. En pratique : Logistique pour une vue rapide d'un véhicule de flotte, Maintenance pour un suivi préventif / correctif structuré.

### 5.3 Location (Module 19)

- Le type d'équipement `LOCATION` n'est qu'une **étiquette** : la Maintenance ne lit ni n'écrit dans les tables `location_*`. L'état `REPARATION` d'un équipement de location reste un indicateur manuel, **sans lien automatique** vers une demande de maintenance.

### 5.4 Bons de travail (Module 11) et Pointage (Module 12)

- **Aucune intégration.** Les bons de travail planifient le **travail humain** sur chantier ; les demandes de maintenance planifient les **interventions sur équipements**. Rien ne se crée automatiquement de l'un vers l'autre.
- Les heures (`duree_heures`, temps réel) sont **saisies à la main** dans la Maintenance et **ne remontent pas** au Pointage ni à la Paie. Le technicien est un simple texte libre.

### 5.5 Comptabilité (Module 14)

- **Aucune écriture comptable.** Le coût réel et le coût estimé d'une demande restent dans la table de maintenance ; rien n'est reporté vers la Comptabilité. Le rapprochement (par exemple pour immobiliser une réparation) est manuel.

### 5.6 Crédits IA (Module 24)

- L'assistant de maintenance partage le **même portefeuille de crédits IA** que les autres assistants de l'ERP.
- Chaque appel est tracé sous `maintenance_chat`, `maintenance_diagnose`, `maintenance_preventive`, `maintenance_checklist` ou `maintenance_estimate_cost`, visibles dans le suivi d'usage du super-administrateur.

### 5.7 Documents et photos (Module 06 Dossiers)

- **Aucune** pièce jointe ni photo dans la Maintenance (les colonnes `photos_avant` / `photos_apres` existent en base mais ne sont pas exposées). Pour conserver des photos ou des rapports, passez par le module Dossiers.

### 5.8 Foire aux questions

**Comment savoir quel équipement est « Inventaire #42 » ?**
Il n'y a **aucune** recherche automatique du nom. Le module ne stocke qu'un couple Type + ID saisi à la main. Reportez-vous au module correspondant (Magasin pour `Inventaire`, Location pour `Location`, Logistique pour `Véhicule`) avec cet identifiant.

**Puis-je imprimer un bon d'intervention ou exporter un rapport ?**
Non. Le module n'a **aucune** exportation ni impression : ni PDF, ni CSV, ni bon imprimable, ni pièce jointe.

**Où sont les onglets « Ordres », « Équipements » et « Planning » ?**
Ils n'existent pas. Ces noms figurent dans les fichiers de traduction, mais **le code n'a que 11 onglets** et aucun de ceux-là. La planification est une simple liste, pas un calendrier.

**Pourquoi ne puis-je pas créer une intervention depuis l'onglet Interventions ?**
Par conception : la création se fait dans le **détail d'une demande** (carte Interventions). L'onglet ne sert qu'à éditer le statut, le temps passé et le rapport, ou à supprimer.

**Une entrée d'historique ou un relevé de compteur est-il modifiable ?**
Non. Historique et Compteurs sont en **ajout seul**. Pour corriger, ajoutez une nouvelle entrée.

**Pourquoi mon relevé en « Cycles » ne génère-t-il jamais d'alerte ?**
Parce que seuls les compteurs **Heures** et **Kilomètres** sont reliés à une fréquence de planification. Le type **Cycles n'est associé à aucune fréquence** : il est conservé pour le suivi, mais ne déclenche rien.

**Les alertes d'échéance se génèrent-elles toutes seules ?**
Non. Il n'y a **aucun cron** : cliquez sur **« Générer les alertes »** régulièrement. Les alertes **d'usage**, elles, naissent au moment d'un relevé de compteur.

**La prochaine échéance est-elle calculée pour toutes les fréquences ?**
Non — seulement pour **jours**, **semaines** et **mois** (le mois vaut 30 jours, une approximation). Pour **heures d'utilisation** et **kilomètres**, l'échéance est pilotée par les compteurs, pas par une date.

**Le numéro `MR-#####` est-il garanti unique ?**
Oui. Le serveur le génère par sondage contre une contrainte d'unicité (jamais de `COUNT(*)+1`). C'est une correction par rapport à l'ancien manuel, qui décrivait à tort un tirage aléatoire sujet aux collisions.

**Le coût réel d'une demande inclut-il automatiquement les pièces ?**
Non. Le coût réel est **saisi à la main**. Le total des pièces s'affiche dans la carte Pièces du détail, sans pré-remplir le coût réel.

**Le stock est-il rendu si je supprime une pièce ?**
Non. La décrémentation à l'ajout **n'est pas** compensée à la suppression. Corrigez par un mouvement d'entrée manuel dans le Magasin.

**N'importe quel employé peut-il supprimer une demande ou une planification ?**
Oui, tant que le tenant n'est pas en mode consultation : **aucun** point d'accès de la Maintenance n'a de garde de rôle. Seule la lecture seule globale (abonnement inactif) bloque l'écriture. Organisez-vous en conséquence.

**Le diagnostic ou l'estimation de l'IA sont-ils fiables ?**
À valider systématiquement par un technicien qualifié — c'est d'ailleurs l'avertissement affiché en permanence. Les coûts sont indicatifs, les enjeux de sécurité doivent être vérifiés.

**Pourquoi l'assistant IA renvoie-t-il 403 alors que j'ai des crédits ?**
Probablement le mode consultation : comme ses appels sont des écritures (POST), ils sont bloqués quand l'abonnement Stripe est inactif. Régularisez l'abonnement.

**Ai-je payé deux fois un appel IA ?**
C'est possible en cas de double-clic ou de reprise réseau : le débit n'a **pas** de clé d'idempotence. N'appuyez qu'une fois sur « Lancer » et attendez la réponse.

---

## 6. Récapitulatif

- **Objet** : GMAO légère pour l'entretien des équipements de construction — **préventif** (types + planifications + alertes d'échéance), **correctif** (demandes → interventions → pièces → historique) et **à l'usage** (compteurs → alertes). « Prédictive » n'est qu'une étiquette de catégorie, sans moteur prédictif.
- **Accès** : barre latérale → groupe **TERRAIN** → **Maintenance** (icône `Wrench`), route `/maintenance`, titre « Maintenance ». Onglet par défaut : Tableau de bord. Le sous-titre n'est jamais affiché.
- **11 onglets** : Tableau de bord, Types, Planification, Demandes, Interventions, Pièces, Alertes, Historique, **Compteurs**, Statistiques, **Assistant IA**.
- **Un seul router côté serveur** : tout vit dans `secondary.py`, préfixe **`/maintenance/*`** (**37 points d'accès** : 31 de gestion + statistiques, 6 pour l'IA dont 5 câblés à l'écran). **Aucun** fichier `maintenance.py` ni `maintenance_ai.py` ; `terrain.py` est un module de cadastre sans rapport.
- **8 tables** `maintenance_*` par tenant, **créées à la demande** (un tenant neuf voit des compteurs à zéro).
- **Permissions** : lecture **et écriture ouvertes à tout employé authentifié** (aucune garde de rôle, contrairement à la maintenance de la Logistique) ; seule protection en écriture = le **mode consultation** global (403) si l'abonnement Stripe est inactif ; super-administrateur plateforme = 400 sur le CRUD.
- **Numéro** : `MR-#####`, généré de façon **atomique et unique**.
- **Cycle d'une demande** : `DEMANDE` → (1re intervention) `EN_COURS` → (intervention `TERMINE`) `TERMINE` + historique automatique ; suppression interdite en `EN_COURS` / `TERMINE`.
- **Fréquences** : `JOURS` / `SEMAINES` / `MOIS` calculent une échéance (mois = 30 jours, approximation) ; `HEURES_UTILISATION` / `KILOMETRES` sont pilotées par les compteurs.
- **Alertes** : deux voies — par échéance (bouton « Générer les alertes », **aucun cron**) et par usage (relevé de compteur Heures ou Kilomètres ; **Cycles ne déclenche rien**) ; anti-doublon à l'épreuve des accès concurrents.
- **Assistant IA** : 5 outils câblés (Discussion et Liste de vérification en Sonnet 4-6 ; Diagnostic, Plan préventif et Estimation de coût en Opus 4-8), tarif du modèle × 1,30, vrai gardien = solde de crédits (402), `check_ai_guard` neutre. **Débit sans idempotence** : ne cliquez qu'une fois. Un 6e point d'accès (analyse d'intervention) existe mais reste **dormant** à l'écran.
- **Ce que le module ne fait pas** : aucun équipement relié (couple Type + ID à la main, sans validation) ; aucune impression / PDF / CSV / téléversement / photo ; aucun onglet Ordres / Équipements / Planning ; aucune notification ; aucune écriture comptable ; heures et technicien non structurés ; historique et compteurs en ajout seul.

---

**Documentation générée à partir du code** : `ERP_REACT/backend/routers/secondary.py` (section Maintenance lignes 6180-8646 : 37 points d'accès `/maintenance/*`, DDL des 8 tables 1736-1898, assistant `_ensure_maintenance_tables` 1899-1948, invite système IA 1664-1723, helpers `_compute_next_maintenance` 6180 / `_maybe_generate_usage_alerts` 7517 / `_gen_unique_numero` 962) ; `ERP_REACT/backend/routers/ai.py` (crédits IA, `check_ai_guard` neutre 824-825, `_check_credits` 1192, `_deduct_credits` 1314) ; `ERP_REACT/backend/erp_auth.py` (mode consultation 526) ; `ERP_REACT/backend/erp_api.py` (montage 1025, limite de débit IA 362/470/678-681) ; `ERP_REACT/frontend/src/pages/MaintenancePage.tsx` (1960 lignes, 11 onglets, un seul fichier) ; `ERP_REACT/frontend/src/api/maintenance.ts` (445 lignes, 36 fonctions) ; `store/useMaintenanceStore.ts` ; textes `i18n/locales/fr/terrain.json` (`terrain.maintenance.*`).

**Manuels liés** :
- Module 09 (Magasin / Inventaire — décrément de stock à l'ajout de pièce, pas de re-crédit) — `09-operations-magasin.md`
- Module 11 (Bons de travail — travail humain, aucune intégration) — `11-operations-bons-de-travail.md`
- Module 12 (Pointage — heures de maintenance non propagées) — `12-operations-pointage.md`
- Module 14 (Comptabilité — aucun report automatique des coûts) — `14-operations-comptabilite.md`
- Module 18 (Logistique — maintenance de flotte distincte, protégée par rôle) — `18-terrain-logistique.md`
- Module 19 (Location — état `REPARATION` sans lien maintenance automatique) — `19-terrain-location.md`
- Module 24 (Assistant IA — portefeuille de crédits partagé) — `24-communication-assistant-ia.md`
