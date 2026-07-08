# Module 18 — Logistique et livraisons

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — corrections majeures : **7 onglets** et non 6 (ajout de l'onglet Assistant IA), l'assistant réellement actif est `/logistique/ai/chat` en **lecture seule** et non les quatre anciens endpoints `/logistics/ia/*` devenus morts, la **détection de chevauchement de réservation existe désormais** (erreur 409), les numéros de référence sont générés contre une contrainte d'unicité et non tirés au hasard, les tables sont créées **à la demande** et non à l'initialisation du tenant, et le contrôle d'accès IA `check_ai_guard` est un **contrôle neutre** — seul le solde de crédits bloque)
> **Libellé dans le menu** : « Logistique » (groupe « TERRAIN » de la barre latérale, icône `Truck`) — route `/logistique`. Réf. `Sidebar.tsx:74`, `nav.json:24`.
> **Code de référence (côté serveur)** : le module est réparti sur **trois** routers, il n'existe **aucun** fichier `routers/logistique.py` ni `routers/logistics.py` dédié.
> &nbsp;&nbsp;• `ERP_REACT/backend/routers/secondary.py` — 37 points d'accès sous `/logistics/*` (33 de gestion + statistiques, plus 4 endpoints IA aujourd'hui **morts**), tables `logistics_*` créées à la demande ;
> &nbsp;&nbsp;• `ERP_REACT/backend/routers/gps.py` (372 lignes) — 7 points d'accès sous `/gps/*` (flotte GPS) ;
> &nbsp;&nbsp;• `ERP_REACT/backend/routers/logistique_ai.py` (331 lignes) — 1 point d'accès `POST /logistique/ai/chat` (assistant IA de consultation, **le seul câblé à l'écran**).
> **Chemins d'API réels** : préfixe `/api/erp/v1` — donc `/api/erp/v1/logistics/*`, `/api/erp/v1/gps/*`, `/api/erp/v1/logistique/ai/chat`.
> **Code de référence (côté client)** : `ERP_REACT/frontend/src/pages/LogistiquePage.tsx` (1417 lignes, 7 onglets) ; `ERP_REACT/frontend/src/components/logistique/LogistiqueAssistantTab.tsx` (151 lignes, chat IA) ; `ERP_REACT/frontend/src/api/logistics.ts`, `api/gps.ts`, `api/logistiqueAi.ts` ; magasin d'état `store/useLogistiqueStore.ts`.
> **Tables PostgreSQL** : une série `logistics_*` **par tenant** (`logistics_deliveries`, `logistics_delivery_items`, `logistics_equipment`, `logistics_equipment_reservations`, `logistics_equipment_maintenance`, `logistics_vehicles`, `logistics_vehicle_trips`, `logistics_site_coordination`, `logistics_alerts`) — **créées à la demande**, pas à l'ouverture du tenant. Une série `gps_*` **par tenant** (`gps_vehicle_tracking`, `gps_locations`, `gps_geofences`, `gps_routes`). L'assistant IA écrit dans les tables partagées `public.ai_usage_tracking` (traçage) et `public.ai_prepaid_credits` (débit des crédits).
> **Cadrage** : registre **opérationnel** léger de la logistique de chantier — livraisons, équipements (avec historique de maintenance), véhicules, activités de coordination de chantier, suivi GPS de flotte (en lecture) — accompagné d'un **assistant IA de consultation** (lecture seule). Ce **n'est pas** un système d'optimisation de tournées, ni un générateur de bons de livraison en PDF, ni un système de notifications poussées. Plusieurs capacités existent côté serveur mais **sans interface** (articles de livraison, réservations d'équipement, trajets de véhicule, création de zones GPS, génération d'alertes) : voir la section 1.3.

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

Offrir à une entreprise de construction québécoise un **registre unique de la logistique de chantier** : savoir ce qui doit être livré et quand, quels équipements et véhicules sont disponibles ou en maintenance, quelles activités de coordination sont planifiées sur le terrain (coulée de béton, arrivée de grue, fermeture de rue), et où se trouve la flotte selon les dernières positions GPS.

Concrètement, le module répond à des questions comme :

- « Quelles livraisons sont planifiées cette semaine et où doivent-elles être entreposées ? »
- « Cette excavatrice est-elle disponible, en utilisation ou en maintenance ? Quand a-t-elle été inspectée pour la dernière fois ? »
- « Combien de camions sont disponibles et quel est leur kilométrage ? »
- « Quelles activités faut-il coordonner sur le chantier demain matin ? »
- « Y a-t-il des alertes logistiques ouvertes ? » (assistant IA)

### 1.2 Ce que le module fait (vérifié contre le code)

- **Livraisons** : créer, lister (paginé, 25 par page), rechercher, filtrer par statut, changer le statut en ligne et supprimer des livraisons entrantes ou sortantes. Chaque livraison reçoit une référence unique `LIV-#####` générée par le serveur.
- **Équipements** : créer, lister, rechercher, filtrer par catégorie et statut, changer le statut en ligne et supprimer des équipements mobiles (grue, excavatrice, bétonnière, génératrice…). Code unique `EQP-#####`.
- **Maintenance des équipements** : consigner des interventions (préventive, inspection, réparation, certification) sous l'équipement sélectionné, avec coût, technicien, conformité et **prochaine date** qui met automatiquement à jour l'échéance de l'équipement.
- **Véhicules** : créer, lister, rechercher, filtrer par statut, changer le statut en ligne, mettre à jour le kilométrage et supprimer les véhicules de la flotte. Immatriculation unique.
- **Coordination de chantier** : planifier des activités datées (livraison de béton, arrivée de grue, coulée, inspection, réunion, fermeture de rue…) avec horaire, zone concernée et responsable. Référence unique `COORD-#####`. **Attention** : cet écran s'affiche sous l'onglet mal nommé « Trajets ».
- **Tableau de bord** : quatre indicateurs clés (livraisons planifiées, équipements disponibles, véhicules disponibles, alertes actives), trois cartes de détail et la liste des alertes actives (en lecture).
- **Suivi GPS de la flotte** : consulter la dernière position de chaque véhicule, les lieux enregistrés, les zones (geofences) et les routes du jour — sous forme de **tableaux de coordonnées**, sans carte.
- **Assistant IA de consultation** : un chat en langage naturel qui **lit** vos données de logistique réelles (livraisons, équipements, véhicules, coordination, alertes) et répond, **sans jamais écrire** ni accéder à la paie ou aux employés.

### 1.3 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes par l'interface, et certaines capacités existent côté serveur sans écran pour les utiliser.

- **Aucune exportation, aucune impression, aucun téléversement de fichier, aucune action groupée** dans tout le module. Pas de bon de livraison en PDF, pas de CSV, pas de pièce jointe.
- **Pas d'édition complète des fiches.** Sur une livraison, un équipement, un véhicule ou une activité déjà créés, l'interface ne permet que **deux** gestes : changer le **statut** (menu déroulant dans le tableau) et **supprimer**. Il n'existe aucune fenêtre de modification des autres champs (type, zone, coûts, notes…). Corriger une adresse ou un coût oblige à supprimer et recréer.
- **Articles de livraison sans interface.** Le serveur sait rattacher des lignes détaillées (description, quantités prévue et reçue, unité, conformité) à une livraison, mais **aucun écran** ne permet de les saisir ni de les voir. Le détail d'une livraison ne s'ouvre pas.
- **Réservations d'équipement sans interface.** Le serveur gère des réservations d'équipement par période — avec **détection de chevauchement** (deux réservations qui se recoupent sont refusées, voir 4.11) — mais **aucun écran** ne permet de créer ou consulter ces réservations.
- **Trajets de véhicule sans interface.** Le serveur enregistre des déplacements (départ, retour, kilométrage, carburant, coût), mais **aucun écran** ne les affiche ni ne les crée. Malgré les libellés « Trajets » et « GPS / Trajets », le vrai concept de trajet de véhicule n'a **aucun onglet**.
- **Alertes en lecture seule.** Le tableau de bord affiche les alertes actives, mais il n'y a **aucun bouton** pour les générer ni pour les marquer traitées. La génération automatique (maintenance, inspection, expiration d'assurance) et le traitement existent côté serveur, sans écran.
- **Pas de carte cartographique.** L'onglet GPS montre des **latitudes et longitudes en texte**, jamais une carte interactive. Les zones (geofences) et les routes sont en lecture seule ; seuls les **lieux** peuvent être ajoutés.
- **Aucune notification poussée** (courriel, notification navigateur, message texte). Les alertes vivent en base et s'affichent seulement dans le tableau de bord.
- **Aucune synchronisation comptable ni de stock.** Les coûts (journalier, mensuel, carburant, maintenance) sont **affichés** mais jamais convertis en écritures comptables. Une livraison passée à « livrée » **ne met pas** le stock à jour dans le Magasin.
- **Assistant IA en lecture seule.** L'assistant **ne crée, ne modifie et ne supprime rien** : il répond à partir de vos données, sans proposer d'action ni écrire en base.
- **L'onglet Véhicules n'a pas de pagination** (la liste complète est chargée d'un coup), contrairement aux onglets Livraisons, Équipements et Coordination (25 par page).

### 1.4 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **Logistique** (icône `Truck`). Réf. `Sidebar.tsx:74`.
- URL directe : `/logistique`.
- Fil d'Ariane / titre de la page : « Logistique » (le sous-titre « Gestion de la flotte, des livraisons et trajets » existe dans les libellés mais **n'est pas affiché**).
- **Onglet par défaut** : Tableau de bord.

### 1.5 Permissions et rôles

Le module distingue nettement la **lecture** (ouverte) de l'**écriture** (réservée à certains rôles).

| Action | Qui peut la faire |
|--------|-------------------|
| **Consulter** (tous les onglets, tous les GET, y compris GPS et statistiques) | Tout utilisateur authentifié du tenant (`get_current_user`). |
| **Créer / modifier le statut / supprimer** une livraison, un équipement, une maintenance, un véhicule, une activité de coordination | Rôle d'écriture logistique : **administrateur**, **super-administrateur**, **gestionnaire**, **contremaître** ou **magasinier** (`require_tenant_admin_or_role`, `erp_auth.py:720`). Un propriétaire dont le compte porte `is_admin` est toujours autorisé (droit relu au serveur, infalsifiable). Sinon : **403**. |
| **Ajouter un lieu ou une zone GPS** | **Tout** utilisateur authentifié — les écritures GPS n'ont **aucune** garde de rôle (incohérence connue, voir 4.10). |
| **Utiliser l'assistant IA** | Tout utilisateur authentifié **ayant des crédits IA** (voir 1.7). |

> **Mode consultation (lecture seule) à l'échelle du tenant.** Si l'entreprise n'a pas d'abonnement Stripe actif (abonnement annulé ou absent), tout le tenant passe en **mode consultation** : les lectures restent permises, mais **toute** création, modification ou suppression renvoie **403** (`erp_auth.py:526`). Cela s'applique aussi aux écritures GPS. Voir le module Configuration / Abonnement pour rétablir l'accès.

### 1.6 Les 7 onglets

Source : `LogistiquePage.tsx:117` et `:153-193`. Les compteurs entre parenthèses proviennent des totaux du magasin d'état.

| # | Clé interne | Libellé affiché | Icône | Contenu réel |
|---|-------------|-----------------|-------|--------------|
| 1 | `dashboard` | Tableau de bord | `BarChart3` | Indicateurs, détails et alertes actives (lecture) |
| 2 | `deliveries` | Livraisons (N) | `Package` | Gestion des livraisons |
| 3 | `equipment` | Équipements (N) | `Wrench` | Gestion des équipements + maintenance |
| 4 | `vehicles` | Véhicules (N) | `Truck` | Gestion de la flotte |
| 5 | `coordination` | **Trajets** (N) | `ClipboardList` | **Coordination de chantier** (libellé trompeur, voir 2.6) |
| 6 | `gps` | **GPS / Trajets** | `MapPin` | Flotte GPS en lecture (+ ajout de lieux) |
| 7 | `assistant` | Assistant IA | `Sparkles` | Chat de consultation en lecture seule |

> **Deux onglets portent le mot « Trajets », aucun ne gère les trajets de véhicule.** L'onglet 5 (« Trajets ») gère la **coordination de chantier** ; l'onglet 6 (« GPS / Trajets ») gère la **flotte GPS**. Les déplacements de véhicule (départ, retour, carburant) n'ont pas d'écran.

### 1.7 Coûts et facturation (assistant IA seulement)

- **Toute la partie registre est gratuite** : créer et consulter des livraisons, équipements, véhicules, activités et lieux GPS ne consomme aucun crédit.
- **Seul l'assistant IA est payant.** Chaque message consomme des **crédits IA prépayés** de l'entreprise. Coût = tarif du modèle `claude-sonnet-4-6` (3 $ US par million de jetons en entrée, 15 $ US par million en sortie) **majoré de 30 %**. Le débit est **ferme** (non best-effort) et tracé sous la fonctionnalité `logistique_chat` dans `public.ai_usage_tracking`. Un compte sans crédits reçoit une erreur **402** et ne peut pas envoyer de message ; la consultation logistique, elle, reste gratuite. Détail en section 4.8.

### 1.8 Architecture technique

```
Frontend LogistiquePage.tsx (7 onglets)
    │
    ├── onglets Livraisons/Équip./Véhic./Coordination/Tableau de bord
    │        └─ api/logistics.ts ──> secondary.py  /api/erp/v1/logistics/*   (33 endpoints + stats)
    │                                 tables logistics_* (créées à la demande)
    │
    ├── onglet GPS / Trajets
    │        └─ api/gps.ts ─────────> gps.py        /api/erp/v1/gps/*         (7 endpoints, lecture)
    │                                 tables gps_* (PAS créées à la demande)
    │
    └── onglet Assistant IA
             └─ api/logistiqueAi.ts ─> logistique_ai.py /api/erp/v1/logistique/ai/chat
                                        Claude sonnet-4-6, outil recherche_bd (lecture seule)
                                        débit crédits IA + traçage usage
```

> **Point d'attention pour un tenant neuf.** Les tables `logistics_*` sont créées **à la première utilisation** (à la volée, `_ensure_table`) : un tenant qui n'a encore rien saisi voit des compteurs à **0** et des listes vides — c'est normal. En revanche, `gps.py` **ne crée pas** ses tables à la demande : sur un tenant qui n'a jamais reçu de données GPS, l'onglet GPS peut renvoyer une erreur serveur (voir la FAQ 5.7).

---

## 2. Interface

Source : `LogistiquePage.tsx` (1417 lignes) et `LogistiqueAssistantTab.tsx` (151 lignes).

### 2.1 Disposition générale

```
+-----------------------------------------------------------------------------+
| Logistique                                                                  |
+-----------------------------------------------------------------------------+
| [bandeau rouge d'erreur, fermable]   [bandeau vert de succès, fermable]      |
+-----------------------------------------------------------------------------+
| Tableau de bord | Livraisons (12) | Équipements (8) | Véhicules (5) |        |
| Trajets (3) | GPS / Trajets | Assistant IA          <- barre d'onglets       |
+-----------------------------------------------------------------------------+
|                                                                             |
|                     contenu de l'onglet sélectionné                          |
|                                                                             |
+-----------------------------------------------------------------------------+
```

- **Titre** « Logistique » toujours affiché en haut (`:165`).
- **Bandeau d'erreur** (rouge, fermable) et **bandeau de succès** (vert, fermable) apparaissent au-dessus des onglets après une action. Messages de succès : « Créé avec succès », « Mis à jour avec succès », « Supprimé avec succès ».
- **Barre d'onglets** défilable horizontalement sur petit écran. L'onglet actif est souligné en bleu.
- **Adaptatif** : chaque tableau (affichage bureau) se transforme en **cartes empilées** sur téléphone.

### 2.2 Onglet Tableau de bord

Source : `:200-292`. Charge `GET /logistics/statistics` et `GET /logistics/alerts?statut=active` à l'ouverture.

**Quatre cartes d'indicateurs** (en haut) :

| Carte | Couleur | Valeur | Sous-texte |
|-------|---------|--------|------------|
| Livraisons planifiées | bleu | nombre de livraisons au statut `planifiee` | « Total : {N} » |
| Équipements disponibles | vert | nombre d'équipements `disponible` | « Total : {N} » |
| Véhicules disponibles | sarcelle | nombre de véhicules `disponible` | « Total : {N} » |
| Alertes actives | rouge si > 0, sinon jaune | nombre d'alertes actives | — |

**Trois cartes de détail** (au milieu) :

- **Livraisons** : « Planifiées » (badge jaune), « En cours » (bleu), « Cette semaine » (vert).
- **Équipements** : « Disponibles » (vert), « En utilisation » (bleu), « Maintenance » (violet).
- **Véhicules** : « Disponibles » (vert), « En déplacement » (bleu), « KM total » (somme des kilométrages).

**Section « Alertes actives »** (en bas, affichée **seulement s'il y a des alertes**) : chaque alerte se présente avec un liseré gauche coloré selon la priorité (**haute** = rose, **moyenne** = orange, sinon jaune), le message, un badge de priorité et une échéance facultative (« Échéance : {date} »).

> **Lecture seule.** Le tableau de bord **n'offre aucun bouton** pour traiter une alerte ni pour en générer. C'est un affichage de synthèse. La génération et le traitement des alertes existent côté serveur (voir 4.6) mais ne sont pas exposés.

### 2.3 Onglet Livraisons

Source : `:296-451`.

**Barre d'outils** : bouton **« Nouvelle livraison »** (icône `Plus`), champ de recherche (« Rechercher... », filtre **local** sur référence, type, zone et notes) et menu de filtre par statut (Tous / Planifiée / En cours / Livrée / Annulée).

**Tableau (affichage bureau)**, colonnes triables (clic sur l'en-tête) :

| Colonne | Contenu |
|---------|---------|
| Référence | `reference` (format `LIV-#####`) ou, à défaut, `#id` |
| Type | type de livraison |
| Zone | zone de stockage |
| Statut | **menu déroulant en ligne** pour changer le statut directement (planifiee / en_cours / livree / annulee) |
| Date prévue | date planifiée |
| Actions | bouton Supprimer (icône `Trash2`), confirmation « Supprimer la livraison {réf} ? » |

Liste vide : « Aucune livraison ». **Pagination** de 25 par page si plus d'une page. Sur téléphone, chaque livraison devient une carte (référence, type · zone, badge de statut, date, bouton supprimer).

**Fenêtre « Nouvelle livraison »** (taille moyenne) :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Date prévue | date | **Oui** (seul champ obligatoire) |
| Heure prévue | heure | non |
| Type de livraison | menu : Sélectionner… / Fournisseur / Chantier / Transfert / Retour / Collecte | non |
| Zone de stockage | texte | non |
| Notes | zone de texte (2 lignes) | non |

Boutons **« Annuler »** / **« Créer »** (le bouton Créer est désactivé tant que la date est vide ; une garde interne empêche le double envoi). À la création, le serveur génère la référence `LIV-#####` et pose le statut `planifiee`.

> **Ce que la fenêtre ne montre pas.** Aucun champ pour rattacher un projet ou un fournisseur (les colonnes existent en base mais ne sont pas saisissables ici), et **aucun** moyen de saisir les articles détaillés de la livraison.

### 2.4 Onglet Équipements

Source : `:455-806`.

**Barre d'outils** : bouton **« Nouvel équipement »**, recherche locale (code, nom, catégorie, localisation, notes), filtre par **catégorie** (Toutes / Grue / Excavatrice / Chargeuse / Échafaudage / Compacteur / Bétonnière / Génératrice / Autre) et filtre par **statut** (Tous / Disponible / En utilisation / Maintenance / Réservé).

**Tableau**, colonnes triables :

| Colonne | Contenu |
|---------|---------|
| Code | `code` en police à chasse fixe (format `EQP-#####`) |
| Nom | nom de l'équipement |
| Catégorie | badge bleu |
| Statut | **menu déroulant en ligne** (disponible / en_utilisation / maintenance / reserve) |
| Localisation | localisation actuelle (texte) |
| Coût/jour | coût journalier mis en forme en dollars |
| Actions | bouton **Historique de maintenance** (icône `Wrench`) + bouton Supprimer (confirmation « Supprimer l'équipement {nom} ? ») |

**Cliquer sur une ligne** sélectionne l'équipement (ligne surlignée) et **charge sa maintenance** en dessous du tableau. Liste vide : « Aucun équipement ». **Pagination** 25 par page.

**Fenêtre « Nouvel équipement »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Nom | texte | **Oui** |
| Catégorie | menu (8 valeurs) | non |
| Type de possession | menu : Propriété / Location (défaut : Propriété) | non |
| Coût journalier ($) | nombre | non |
| Coût mensuel ($) | nombre | non |
| Localisation actuelle | texte | non |
| Notes | zone de texte | non |

Le serveur génère le code `EQP-#####` et pose le statut `disponible`.

**Section « Historique de maintenance — {nom} »** (visible seulement quand un équipement est sélectionné, `:674-720`) :

- En-tête avec le nom de l'équipement + bouton **« Ajouter intervention »**.
- Tableau : Date / Type (badge violet) / Technicien / Coût / **Conforme** (coche verte `CheckCircle` si conforme, sinon triangle rose `AlertTriangle`) / Actions (supprimer, confirmation « Supprimer cette intervention ? »).
- Liste vide : « Aucune intervention enregistrée ».

**Fenêtre « Nouvelle intervention de maintenance »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Type d'intervention | menu : Maintenance préventive / Inspection / Réparation / Certification (défaut : Maintenance préventive) | non |
| Date intervention | date | **Oui** |
| Technicien | texte | non |
| Coût ($) | nombre | non |
| Prochaine date | date | non |
| Conforme | case à cocher (cochée par défaut) | non |
| Description | zone de texte | non |

> **Effet important de « Prochaine date ».** Si vous renseignez « Prochaine date », le serveur met aussi à jour le champ `prochaine_maintenance` de l'équipement — c'est cette valeur qui alimentera plus tard les alertes automatiques (voir 4.11). Modifier ou supprimer une intervention **repropage** ou **recalcule** cette échéance.

> **Réservations d'équipement.** Aucun écran ici, même si le serveur les gère (avec refus des chevauchements). Le clic sur une ligne n'ouvre **que** la maintenance.

### 2.5 Onglet Véhicules

Source : `:810-981`.

**Barre d'outils** : bouton **« Nouveau véhicule »**, recherche locale (immatriculation, marque, modèle, type), filtre par statut (Tous / Disponible / En déplacement / Maintenance / Hors service).

**Tableau**, colonnes triables :

| Colonne | Contenu |
|---------|---------|
| Véhicule | marque + modèle |
| Immatriculation | immatriculation (unique) |
| Type | type de véhicule |
| Statut | **menu déroulant en ligne** (disponible / en_deplacement / maintenance / hors_service) |
| KM | kilométrage (mis en forme avec séparateurs de milliers) |
| Actions | bouton Supprimer (confirmation « Supprimer le véhicule {immat.} ? ») |

Liste vide : « Aucun véhicule ». **Aucune pagination** : la liste complète est chargée (le filtre véhicule ne porte que sur le statut). Sur téléphone : cartes empilées.

**Fenêtre « Nouveau véhicule »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Immatriculation | texte | **Oui** (unique) |
| Marque / Modèle / Année | texte / texte / nombre | non |
| Type de véhicule | menu : Sélectionner… / Camionnette / Camion léger / Camion lourd / Fourgonnette / Remorque / Voiture / Autre | non |
| Capacité charge / Unité / Kilométrage | nombre / menu (kg, lb, tonnes ; défaut kg) / nombre | non |
| Consommation (L/100km) / Coût par km ($) | nombre / nombre | non |
| Notes | zone de texte | non |

Le serveur pose le statut `disponible`. **Modification après coup** : seuls le **statut** (menu en ligne) et le **kilométrage** (via l'API) sont modifiables ; l'année est bornée de 1900 à 2100.

> **Trajets de véhicule.** Aucun écran. Le serveur enregistre les déplacements, mais ni l'onglet Véhicules ni l'onglet GPS ne les affiche.

### 2.6 Onglet « Trajets » = Coordination de chantier

Source : `:985-1152`.

> **Rappel du libellé trompeur.** L'onglet s'appelle « Trajets » mais gère des **activités de coordination de chantier** (table `logistics_site_coordination`). Il n'a rien à voir avec les déplacements de véhicule.

**Barre d'outils** : bouton **« Nouvelle activité »**, recherche locale (type, zone, responsable, notes), filtre par statut (Tous / Planifié / En cours / Terminé / Annulé).

**Tableau**, colonnes triables :

| Colonne | Contenu |
|---------|---------|
| Date | date de l'activité |
| Type | badge bleu (type d'activité) |
| Horaire | heure de début – heure de fin |
| Zone | zone concernée |
| Responsable | nom du responsable |
| Statut | **menu déroulant en ligne** (planifie / en_cours / termine / annule) |
| Actions | bouton Supprimer (confirmation « Supprimer cette activité ? ») |

Liste vide : « Aucune activité de coordination ». **Pagination** 25 par page.

**Fenêtre « Nouvelle activité de coordination »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Date | date | **Oui** |
| Type d'activité | menu : Sélectionner… / Livraison béton / Livraison matériaux / Arrivée grue / Coulée béton / Installation équipement / Inspection / Réunion de chantier / Fermeture de rue / Autre | **Oui** |
| Heure début / Heure fin | heure / heure | non |
| Zone concernée / Responsable | texte / texte | non |
| Notes | zone de texte | non |

Le bouton Créer est désactivé tant que la **date** ou le **type** est vide. Le serveur génère la référence `COORD-#####` et pose le statut `planifie`.

### 2.7 Onglet GPS / Trajets

Source : `:1156-1416`. Au chargement, l'onglet appelle **quatre** jeux de données en parallèle (`gps.py`) : véhicules, lieux, geofences et routes. Il présente **quatre sous-onglets** (boutons en pilule) :

**Sous-onglet « Véhicules GPS ({N}) »** (icône `Truck`) — **lecture seule**. Tableau : Véhicule / Immatriculation / Statut (badge) / Latitude (6 décimales, chasse fixe) / Longitude / Vitesse (km/h) / Dernière position (date). Vide : « Aucun véhicule avec données GPS ». La position vient de la dernière ligne de `gps_vehicle_tracking` jointe à `logistics_vehicles`.

**Sous-onglet « Lieux ({N}) »** (icône `MapPin`) — **la seule zone modifiable de l'onglet GPS**. Bouton **« Ajouter lieu »**. Tableau : Nom / Type (badge bleu) / Latitude / Longitude / Ville. Vide : « Aucun lieu enregistré ».

- **Fenêtre « Ajouter un lieu GPS »** : Nom * / Latitude * (nombre) / Longitude * (nombre) / Type (texte, défaut `chantier`) / Ville. Le bouton Ajouter est désactivé tant que le nom, la latitude ou la longitude manquent.

**Sous-onglet « Geofences ({N}) »** (icône `Navigation`) — **lecture seule, aucun bouton de création**. Tableau : Nom / Type de zone (badge violet) / Centre (latitude) / Centre (longitude) / Rayon (m) / Alertes (indicateurs « Entrée » / « Sortie »). Vide : « Aucune geofence ». (La création de zone existe côté serveur mais **n'a pas d'écran**.)

**Sous-onglet « Routes ({N}) »** (icône `Navigation`) — **lecture seule**. Tableau : Origine / Statut (badge) / Distance (km) / Destination. Vide : « Aucune route ».

> **Pas de carte, pas d'historique.** Les positions sont des tableaux de coordonnées, jamais une carte. L'historique de déplacement d'un véhicule existe côté serveur (`GET /gps/vehicles/{id}/history`) mais n'est pas affiché.

### 2.8 Onglet Assistant IA

Source : `LogistiqueAssistantTab.tsx` (151 lignes). Chat **en lecture seule** : il lit vos données mais n'écrit rien et ne propose aucune action.

- **En-tête** : icône `Sparkles`, titre « **Assistant IA — Logistique** », sous-titre « Interroge tes livraisons, équipements, véhicules et alertes (lecture seule). ».
- **État vide** : message d'invitation « Pose une question sur ta logistique (livraisons, équipements, véhicules, coordination de chantier, alertes). L'assistant lit tes données réelles. », suivi de **3 exemples cliquables** :
  - « Quelles livraisons sont planifiées cette semaine ? »
  - « Combien d'équipements et de véhicules avons-nous ? »
  - « Y a-t-il des alertes logistiques ouvertes ? »
- **Messages** : les vôtres à droite, ceux de l'assistant à gauche. Sous chaque réponse, un pied de bulle affiche le profil « Logistique », le nombre de jetons, le **coût** et la **durée** de la réponse.
- **En cours** : indicateur « Analyse en cours… ».
- **Erreur** : bande rouge (le message vient du serveur si disponible, sinon « Une erreur est survenue. Réessaie. »).
- **Zone de saisie** : champ multiligne (« Pose ta question sur la logistique… »). **Entrée** envoie le message, **Maj + Entrée** insère un saut de ligne. Bouton « Envoyer ». Un verrou interne empêche le double envoi.

> **Ce que l'assistant peut et ne peut pas faire.** Il **lit** vos données réelles (via une requête de lecture seule) sur les livraisons, articles, équipements, réservations, maintenance, véhicules, trajets, coordination et alertes, plus les projets et entreprises. Il **ne peut pas** consulter la paie, les employés, les salaires, les NAS, les crédits IA ni les jetons — ces tables lui sont interdites. Il **n'écrit jamais** en base.

### 2.9 Éléments transverses

- **Recherche** : toujours **locale** (elle filtre la page déjà chargée, pas toute la base). Présente dans les onglets Livraisons, Équipements, Véhicules et Coordination.
- **Tri** : clic sur les en-têtes de colonne dans les quatre tableaux principaux.
- **Changement de statut** : par **menu déroulant en ligne** dans le tableau (aucune fenêtre), pour les livraisons, équipements, véhicules et activités.
- **Adaptatif** : tableaux sur ordinateur, cartes empilées sur téléphone.
- **Aucune exportation, impression, CSV, PDF, téléversement ni action groupée** nulle part.

---

## 3. Workflows pas à pas

### 3.1 Planifier une livraison

1. Onglet **Livraisons** → **« Nouvelle livraison »**.
2. Saisir la **date prévue** (obligatoire) ; au besoin l'heure, le type (par exemple « Fournisseur » pour des matériaux entrants), la zone de stockage et des notes.
3. **Créer**. Le serveur attribue une référence `LIV-#####` et le statut `planifiee` ; la livraison apparaît dans le tableau.

> Il n'y a pas de champ pour les articles de la livraison ni pour le projet/fournisseur : ces informations doivent être consignées ailleurs (notes, module Magasin, module Projets).

### 3.2 Suivre et faire avancer une livraison

1. Onglet **Livraisons** → repérer la ligne (recherche ou filtre par statut au besoin).
2. Dans la colonne **Statut**, ouvrir le menu déroulant et choisir le nouvel état : `planifiee` → `en_cours` (camion en route) → `livree` (reçue), ou `annulee`.
3. Le changement est enregistré immédiatement. **Aucune date effective** n'est saisie automatiquement au passage à « livrée ».

### 3.3 Supprimer une livraison

1. Onglet **Livraisons** → bouton Supprimer (icône poubelle) sur la ligne.
2. Confirmer « Supprimer la livraison {réf} ? ». La livraison **et ses articles** (s'il y en a) sont supprimés (cascade en base).

### 3.4 Enregistrer un équipement

1. Onglet **Équipements** → **« Nouvel équipement »**.
2. Saisir le **nom** (obligatoire) ; au besoin la catégorie, le type de possession (Propriété ou Location), les coûts journalier et mensuel, la localisation et des notes.
3. **Créer**. Le serveur attribue un code `EQP-#####` et le statut `disponible`.

### 3.5 Consigner une intervention de maintenance

1. Onglet **Équipements** → **cliquer sur la ligne** de l'équipement (elle se surligne, la section maintenance s'ouvre en dessous).
2. **« Ajouter intervention »**.
3. Choisir le type (Maintenance préventive / Inspection / Réparation / Certification), saisir la **date** (obligatoire), le technicien, le coût, cocher ou décocher **Conforme**, et — important — remplir **Prochaine date** si vous voulez déclencher une future alerte.
4. **Créer**. L'intervention s'ajoute à l'historique ; si « Prochaine date » est renseignée, l'échéance `prochaine_maintenance` de l'équipement est mise à jour.

> Pour supprimer une intervention : bouton poubelle dans le tableau de maintenance (« Supprimer cette intervention ? »). Le serveur **recalcule** alors l'échéance de l'équipement à partir de l'intervention la plus récente restante.

### 3.6 Surveiller les échéances de maintenance et les alertes

- Le **tableau de bord** affiche le nombre d'**alertes actives** et, plus bas, la liste des alertes avec leur priorité et leur échéance.
- Pour interroger précisément les échéances, utilisez l'**assistant IA** (« Quels équipements ont une maintenance ou une inspection prévue bientôt ? »).
- **Il n'y a pas de bouton** pour générer ni traiter les alertes depuis l'écran. La génération automatique (maintenance/inspection à 7 jours, assurance à 30 jours) et le marquage « traitée » existent côté serveur (voir 4.6 et 4.11) mais ne sont pas exposés à l'utilisateur.

### 3.7 Ajouter un véhicule à la flotte

1. Onglet **Véhicules** → **« Nouveau véhicule »**.
2. Saisir l'**immatriculation** (obligatoire, unique) ; au besoin marque, modèle, année (1900-2100), type, capacité + unité, kilométrage initial, consommation, coût par km et notes.
3. **Créer**. Statut initial `disponible`.

### 3.8 Mettre à jour le statut ou le kilométrage d'un véhicule

1. Onglet **Véhicules** → colonne **Statut** : menu déroulant en ligne (disponible / en_deplacement / maintenance / hors_service).
2. Le kilométrage se met à jour via l'API (`PUT /logistics/vehicles/{id}`). Seuls le statut, le kilométrage et les notes sont modifiables après création ; les autres champs (marque, type, capacité…) ne sont pas éditables — il faut supprimer et recréer.

### 3.9 Planifier une activité de coordination de chantier

1. Onglet **« Trajets »** (qui contient la coordination) → **« Nouvelle activité »**.
2. Saisir la **date** et le **type d'activité** (tous deux obligatoires) ; au besoin les heures de début et de fin, la zone concernée, le responsable et des notes.
3. **Créer**. Référence `COORD-#####`, statut `planifie`.
4. Faire avancer l'activité par le menu de statut en ligne (planifie → en_cours → termine, ou annule).

### 3.10 Consulter la flotte sur le suivi GPS

1. Onglet **GPS / Trajets** → sous-onglet **« Véhicules GPS »**.
2. Le tableau montre la dernière position connue de chaque véhicule (latitude, longitude, vitesse, horodatage). C'est **en lecture seule** : les positions viennent du suivi GPS, pas de saisie manuelle.
3. Les sous-onglets **Geofences** et **Routes** sont eux aussi en lecture seule.

### 3.11 Ajouter un lieu GPS

1. Onglet **GPS / Trajets** → sous-onglet **« Lieux »** → **« Ajouter lieu »**.
2. Saisir le **nom**, la **latitude** et la **longitude** (obligatoires) ; au besoin le type (défaut `chantier`) et la ville.
3. **Ajouter**. Le lieu apparaît dans le tableau. C'est la **seule** création possible dans l'onglet GPS.

### 3.12 Interroger l'assistant IA

1. Onglet **Assistant IA**.
2. Cliquer un exemple ou écrire votre question (par exemple : « Combien d'équipements sont en maintenance et lesquels ? »).
3. **Entrée** pour envoyer (**Maj + Entrée** pour aller à la ligne). L'assistant lit vos données réelles et répond ; le coût s'affiche sous la réponse.
4. Enchaîner les questions : l'assistant tient compte des 12 derniers échanges.

> Si vous voyez « Crédits IA épuisés » (erreur 402), rechargez le solde de crédits IA de l'entreprise (module Configuration / Abonnement). La consultation logistique, elle, reste gratuite.

### 3.13 Comprendre un refus (403) ou le mode consultation

- Si une création ou une suppression renvoie **403** alors que vous êtes connecté : votre rôle n'est probablement pas dans la liste d'écriture (administrateur, gestionnaire, contremaître, magasinier). Demandez à un administrateur.
- Si **toutes** les écritures échouent pour **tout le monde** : le tenant est en **mode consultation** (abonnement Stripe inactif). Régularisez l'abonnement pour rétablir l'écriture.

---

## 4. Référence

### 4.1 Statuts par entité

| Entité | Statuts |
|--------|---------|
| Livraison | `planifiee`, `en_cours`, `livree`, `annulee` |
| Équipement | `disponible`, `en_utilisation`, `maintenance`, `reserve` |
| Véhicule | `disponible`, `en_deplacement`, `maintenance`, `hors_service` |
| Coordination | `planifie`, `en_cours`, `termine`, `annule` |
| Réservation d'équipement (serveur seulement) | `reservee` (défaut) ; les réservations `annulee` sont ignorées au test de chevauchement |
| Maintenance | `conforme` = vrai/faux (pas de statut) |
| Alerte | `active` (défaut), `traitee` |
| Priorité d'alerte | `haute`, `normale` (le libellé `moyenne` est reconnu à l'affichage mais rarement produit) |

**Couleurs de statut** (`:119-127`, `statutColor`) : vert (livrée / disponible / terminé / complet), bleu (en cours / en déplacement / en utilisation / actif), jaune (planifié / réservé), rouge (annulé / hors service), violet (maintenance), gris (autre).

### 4.2 Types et catégories

| Champ | Valeurs |
|-------|---------|
| Type de livraison | Fournisseur, Chantier, Transfert, Retour, Collecte |
| Catégorie d'équipement | Grue, Excavatrice, Chargeuse, Échafaudage, Compacteur, Bétonnière, Génératrice, Autre |
| Type de possession | Propriété (`propriete`, défaut), Location (`location`) |
| Type de véhicule | Camionnette, Camion léger, Camion lourd, Fourgonnette, Remorque, Voiture, Autre |
| Unité de capacité | kg (défaut), lb, tonnes |
| Type d'activité de coordination | Livraison béton, Livraison matériaux, Arrivée grue, Coulée béton, Installation équipement, Inspection, Réunion de chantier, Fermeture de rue, Autre |
| Type d'intervention de maintenance | Maintenance préventive (défaut), Inspection, Réparation, Certification |
| Type d'alerte | `maintenance_prevue`, `inspection_requise`, `assurance_expiration` |

### 4.3 Endpoints — Livraisons (`secondary.py`)

Préfixe : `/api/erp/v1`. Colonne « UI » = accessible depuis l'écran.

| Méthode | Chemin | Rôle requis | UI | Notes |
|---------|--------|-------------|----|-------|
| GET | `/logistics/deliveries` | lecture | Oui | Pagination (1-100/page), filtre statut. Crée la table à la demande. |
| GET | `/logistics/deliveries/{id}` | lecture | **Non** | Livraison + ses articles. 404 si absente. |
| POST | `/logistics/deliveries` | écriture | Oui | Génère `LIV-#####`. `date_prevue` obligatoire. |
| PUT | `/logistics/deliveries/{id}` | écriture | Oui (statut) | Mise à jour partielle. 400 si aucun champ, 404 si absente. |
| DELETE | `/logistics/deliveries/{id}` | écriture | Oui | 404 si absente. |
| POST | `/logistics/deliveries/{id}/items` | écriture | **Non** | Ajoute un article. |
| DELETE | `/logistics/deliveries/{id}/items/{item_id}` | écriture | **Non** | Retire un article. |

### 4.4 Endpoints — Équipements, maintenance, réservations

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/logistics/equipment` | lecture | Oui | Pagination + filtres catégorie/statut. |
| GET | `/logistics/equipment/{id}` | lecture | **Non** | Fiche complète. |
| POST | `/logistics/equipment` | écriture | Oui | Génère `EQP-#####`. `nom` obligatoire. |
| PUT | `/logistics/equipment/{id}` | écriture | Oui (statut) | Mise à jour partielle. 404. |
| DELETE | `/logistics/equipment/{id}` | écriture | Oui | 404. |
| GET | `/logistics/equipment/{id}/reservations` | lecture | **Non** | Liste des réservations. |
| POST | `/logistics/equipment/{id}/reservations` | écriture | **Non** | Refus de chevauchement (**409**), 404 si équipement absent. |
| GET | `/logistics/maintenance/alertes` | lecture | **Non** | Équipements dont maintenance/inspection ≤ aujourd'hui + 7 jours. |
| GET | `/logistics/equipment/{id}/maintenance` | lecture | Oui | Historique des interventions. |
| POST | `/logistics/equipment/{id}/maintenance` | écriture | Oui | Crée l'intervention ; propage `prochaine_date`. 404. |
| PUT | `/logistics/maintenance/{id}` | écriture | **Non** | Modifie une intervention ; repropage l'échéance. 404. |
| DELETE | `/logistics/maintenance/{id}` | écriture | Oui | Recalcule l'échéance de l'équipement. 404. |

### 4.5 Endpoints — Véhicules et trajets

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/logistics/vehicles` | lecture | Oui | Filtre statut. En cas d'erreur interne, renvoie une **liste vide** (jamais 500). |
| POST | `/logistics/vehicles` | écriture | Oui | `immatriculation` unique, année 1900-2100. |
| PUT | `/logistics/vehicles/{id}` | écriture | Oui (statut) | Modifie statut / kilométrage / notes seulement. |
| DELETE | `/logistics/vehicles/{id}` | écriture | Oui | 404. |
| GET | `/logistics/vehicles/{id}/trips` | lecture | **Non** | Trajets, tri par date de départ. |
| POST | `/logistics/vehicles/{id}/trips` | écriture | **Non** | Départ = maintenant. 404 si véhicule absent. |

### 4.6 Endpoints — Coordination, alertes, statistiques

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/logistics/coordination` | lecture | Oui | Pagination + filtres projet/statut. |
| POST | `/logistics/coordination` | écriture | Oui | Génère `COORD-#####`. Date + type obligatoires. |
| PUT | `/logistics/coordination/{id}` | écriture | Oui (statut) | Modifie statut / notes. 404. |
| DELETE | `/logistics/coordination/{id}` | écriture | Oui | 404. |
| GET | `/logistics/statistics` | lecture | Oui | Compteurs consolidés ; `km_total` = somme des kilométrages. |
| GET | `/logistics/alerts` | lecture | Oui | Filtres statut (défaut `active`) et priorité, 50 au maximum. |
| PUT | `/logistics/alerts/{id}` | écriture | **Non** | Marque traitée ; pose l'horodatage de traitement. 404. |
| POST | `/logistics/alerts/generate` | écriture | **Non** | Génère les alertes (3 sources, sans doublon). |

### 4.7 Endpoints — Flotte GPS (`gps.py`)

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/gps/vehicles` | lecture | Oui | Véhicules + dernière position (jointure sur `gps_vehicle_tracking`). |
| GET | `/gps/vehicles/{id}/history` | lecture | **Non** | Historique, `hours` de 1 à 168 (défaut 24). |
| GET | `/gps/locations` | lecture | Oui | Lieux enregistrés. |
| POST | `/gps/locations` | **aucune garde de rôle** | Oui | Tout utilisateur authentifié peut créer un lieu. |
| GET | `/gps/geofences` | lecture | Oui | Zones. |
| POST | `/gps/geofences` | **aucune garde de rôle** | **Non** | Création possible par l'API, sans écran. |
| GET | `/gps/routes` | lecture | Oui | Routes du jour (ou d'une date). |

> **`gps.py` ne crée pas ses tables à la demande.** Sur un tenant dont les tables `gps_*` n'existent pas encore, ces endpoints peuvent renvoyer une erreur serveur (contrairement aux endpoints `logistics/*`, qui créent leur table au besoin).

### 4.8 Assistant IA — paramètres, gardes et coût (`logistique_ai.py`)

| Aspect | Détail |
|--------|--------|
| Endpoint | `POST /api/erp/v1/logistique/ai/chat` |
| Modèle | `claude-sonnet-4-6` |
| Longueur maximale de réponse | 8000 jetons |
| Outils | **un seul** : `recherche_bd` (requête SQL de **lecture seule**) |
| Boucle d'outils | jusqu'à 5 itérations par message |
| Historique conservé | 12 derniers tours, tronqués à 8000 caractères chacun |
| Contexte injecté | résumé des données logistiques du tenant, plafonné à 8000 caractères |
| Langue | français du Québec par défaut, anglais si la langue de l'interface est l'anglais |

**Tables lisibles par l'assistant** : `logistics_deliveries`, `logistics_delivery_items`, `logistics_equipment`, `logistics_equipment_reservations`, `logistics_equipment_maintenance`, `logistics_vehicles`, `logistics_vehicle_trips`, `logistics_site_coordination`, `logistics_alerts`, plus `projects` et `companies`. **Tables interdites** (refus par filtre) : employés, paie, salaires, NAS, utilisateurs, crédits IA, Stripe, jetons, etc. Les références qualifiées par un schéma sont refusées (protection contre l'accès à un autre tenant).

**Gardes, dans l'ordre** :

1. Service IA non configuré → **503**.
2. Contrôle d'accès IA `check_ai_guard` → **neutre en pratique** : il laisse passer tout utilisateur authentifié (`ai.py:824-825`, « allow all authenticated users »).
3. Crédits épuisés → **402** « Crédits IA épuisés ».

**Coût** : `(jetons_entrée × 0,003 + jetons_sortie × 0,015) ÷ 1000 × 1,30` (tarif du modèle **majoré de 30 %**). Le débit est **ferme** et le message est tracé sous `logistique_chat` dans `public.ai_usage_tracking`.

> **Deux assistants IA existent dans le code, un seul est branché.** L'écran utilise `logistique_ai.py` (`/logistique/ai/chat`, ci-dessus). Les quatre anciens endpoints `POST /logistics/ia/analyser`, `/chat`, `/rapport`, `/optimiser` (dans `secondary.py`) ont bien des fonctions côté client mais **ne sont appelés par aucun écran** — ils sont **morts**. Ne comptez pas dessus.

### 4.9 Tables PostgreSQL (par tenant)

| Table | Rôle | Créée à la demande ? |
|-------|------|----------------------|
| `logistics_deliveries` | Livraisons (`reference` unique) | Oui |
| `logistics_delivery_items` | Articles de livraison (cascade à la suppression de la livraison) | Oui |
| `logistics_equipment` | Équipements (`code` unique) | Oui |
| `logistics_equipment_reservations` | Réservations (cascade) | Oui |
| `logistics_equipment_maintenance` | Interventions de maintenance (cascade) | Oui |
| `logistics_vehicles` | Flotte (`immatriculation` unique) | Oui |
| `logistics_vehicle_trips` | Trajets (cascade) | Oui |
| `logistics_site_coordination` | Activités de coordination (`reference` unique) | Oui |
| `logistics_alerts` | Alertes préventives | Oui |
| `gps_vehicle_tracking` | Positions horodatées des véhicules | **Non** |
| `gps_locations` | Lieux enregistrés | **Non** |
| `gps_geofences` | Zones | **Non** |
| `gps_routes` | Routes | **Non** |

### 4.10 Rôles d'écriture et permissions

- **Rôles d'écriture logistique** (`LOGISTICS_WRITE_ROLES`, `secondary.py:44`) : `admin`, `super_admin`, `gestionnaire`, `contremaitre`, `magasinier`. Un compte marqué `is_admin` (droit relu au serveur) est toujours autorisé. Tout autre rôle en écriture → **403**.
- **Lectures** : ouvertes à tout utilisateur authentifié du tenant.
- **Écritures GPS** (`POST /gps/locations`, `POST /gps/geofences`) : **aucune** garde de rôle — tout utilisateur authentifié peut créer un lieu ou une zone (incohérence connue par rapport aux écritures logistiques).
- **Mode consultation** : un tenant sans abonnement Stripe actif voit **toutes** ses écritures (y compris GPS) refusées en **403**, les lectures restant permises.

### 4.11 Numéros automatiques, règles d'alerte et de réservation

- **Numéros de référence** : `LIV-#####`, `EQP-#####`, `COORD-#####`. Le serveur tire un numéro et le teste contre la contrainte d'**unicité** (jusqu'à 8 essais, puis élargit l'entropie). La contrainte d'unicité reste le garde-fou final : deux références identiques ne peuvent jamais exister.
- **Chevauchement de réservation d'équipement** : à la création d'une réservation, le serveur pose un verrou sur l'équipement (avec délai maximal de 5 secondes) et vérifie qu'aucune réservation non annulée ne recoupe la période demandée ; sinon il renvoie **409**. La période doit avoir une fin postérieure ou égale au début (sinon **422**). Cette protection existe **au niveau serveur** ; il n'y a pas d'écran pour la déclencher.
- **Génération des alertes** (`POST /logistics/alerts/generate`, sans doublon actif) :

| Source | Condition | Type d'alerte | Priorité |
|--------|-----------|---------------|----------|
| Maintenance d'équipement | `prochaine_maintenance` ≤ aujourd'hui + 7 jours | `maintenance_prevue` | **haute** si ≤ +2 jours, sinon `normale` |
| Inspection d'équipement | `prochaine_inspection` ≤ aujourd'hui + 7 jours | `inspection_requise` | **haute** si ≤ +2 jours, sinon `normale` |
| Assurance de véhicule | `assurance_expiration` ≤ aujourd'hui + 30 jours | `assurance_expiration` | **haute** si ≤ +7 jours, sinon `normale` |

- **Propagation de maintenance** : créer, modifier ou supprimer une intervention met à jour ou recalcule automatiquement l'échéance `prochaine_maintenance` de l'équipement.

### 4.12 Limites de débit (par adresse IP)

| Endpoint | Limite |
|----------|--------|
| `POST /logistique/ai/chat` (assistant actif) | 20 par minute |
| `POST /logistics/ia/*` (endpoints IA morts) | 10 par minute |
| Endpoints de gestion, statistiques et GPS | borne générale élevée (≈ 1500 par minute) |

### 4.13 Raccourcis clavier (assistant IA)

| Touche | Effet |
|--------|-------|
| Entrée | Envoyer le message |
| Maj + Entrée | Insérer un saut de ligne |

---

## 5. Intégrations et FAQ

### 5.1 Projets et fournisseurs (références faibles)

- Les livraisons, équipements, réservations, trajets et activités peuvent porter un `project_id` (et une livraison un `fournisseur_id`) **en base**, mais l'interface **ne propose pas** de les saisir. Ce sont des liens logiques vers `projects.id` et `companies.id`, **sans** jonction affichée ni cascade : supprimer un projet ne touche pas aux fiches de logistique.
- L'assistant IA peut, lui, lire les tables `projects` et `companies` pour recouper vos données.

### 5.2 Magasin / Inventaire (Module 09)

- Le champ `inventory_item_id` d'un article de livraison peut pointer vers un produit du Magasin, mais **il n'y a pas d'écran** pour saisir les articles, et **aucune mise à jour de stock** n'est déclenchée au passage d'une livraison à « livrée ».
- **Bonne pratique** : après réception physique, créez manuellement un mouvement d'entrée dans le Magasin (Module 09) en citant la référence `LIV-#####` dans le motif.

### 5.3 Maintenance (Module 20) — chevauchement de périmètre

- **Logistique → Équipements → Maintenance** couvre la maintenance des **équipements de la flotte logistique** (grues, excavatrices, camions), avec propagation automatique de l'échéance et alimentation des alertes.
- **Module 20 (Maintenance)** couvre la maintenance générale de chantier avec des fiches d'intervention plus riches.
- Les deux modules ne partagent pas les mêmes tables. Pour une vue consolidée, consultez chaque module séparément.

### 5.4 Location (Module 19) — chevauchement de périmètre

- **Logistique → Équipements** déclare les équipements **possédés ou loués par l'entreprise** (champ « Type de possession ») pour un usage interne.
- **Module 19 (Location)** gère les **contrats de location** proposés aux clients ou contractés chez des fournisseurs, avec facturation (TPS/TVQ), cautions et retours. Tables distinctes.

### 5.5 Module GPS

- L'onglet « GPS / Trajets » consomme les endpoints `/gps/*` (mêmes tables `gps_*`). Le lien entre un véhicule de la flotte et sa trace GPS se fait par la table `logistics_vehicles` (partagée par les deux listes).
- **Deux listes de véhicules lisent la même table** : l'onglet Véhicules (liste simple) et le sous-onglet Véhicules GPS (avec la dernière position). C'est une redondance normale, pas un doublon de données.

### 5.6 Crédits IA (Module 24)

- L'assistant logistique partage le **même portefeuille de crédits IA** que les autres assistants de l'ERP.
- Chaque message est tracé sous la fonctionnalité `logistique_chat`, visible dans le suivi d'usage IA du super-administrateur.

### 5.7 Foire aux questions

**Puis-je générer un bon de livraison en PDF ?**
Non. Le module n'a aucune exportation ni impression. Aucun PDF, aucun CSV.

**Comment modifier le type ou la zone d'une livraison déjà créée ?**
Ce n'est pas possible depuis l'interface : seuls le **statut** (menu en ligne) et la **suppression** sont offerts. Supprimez et recréez la livraison. Même règle pour les équipements, véhicules et activités.

**Où sont les articles détaillés d'une livraison ?**
Il n'y a pas d'écran pour eux. Le serveur sait les stocker (description, quantités, unité, conformité), mais aucune interface ne permet de les saisir ni de les voir.

**Comment réserver un équipement pour un projet ?**
Pas depuis l'interface. Les réservations (avec refus de chevauchement) existent côté serveur mais n'ont pas d'écran. Contournement : passez l'équipement au statut « Réservé » ou consignez la période dans les notes.

**Le module empêche-t-il deux réservations sur les mêmes dates ?**
Oui, **côté serveur** : une réservation qui recoupe une autre est refusée (erreur 409). Mais comme il n'y a pas d'écran de réservation, cette protection n'est pas visible à l'usage.

**Comment enregistrer un déplacement de véhicule (départ, retour, carburant) ?**
Pas d'écran. Malgré les onglets « Trajets » et « GPS / Trajets », les trajets de véhicule ne sont pas exposés.

**Pourquoi l'onglet « Trajets » montre-t-il de la coordination de chantier et non des trajets ?**
C'est un libellé hérité d'une ancienne conception. L'onglet « Trajets » gère la **coordination de chantier** ; l'onglet « GPS / Trajets » gère la **flotte GPS**. Le vrai concept de trajet de véhicule n'a aucun onglet.

**Y a-t-il une carte des véhicules ?**
Non. L'onglet GPS affiche des latitudes et longitudes en texte, jamais une carte interactive.

**Comment générer ou traiter une alerte ?**
Le tableau de bord affiche les alertes en lecture. Il n'y a **aucun bouton** pour les générer ni les marquer traitées. La génération (maintenance/inspection à 7 jours, assurance à 30 jours) et le traitement existent côté serveur mais ne sont pas exposés.

**Une livraison « livrée » met-elle le stock à jour dans le Magasin ?**
Non. Aucune synchronisation. Créez le mouvement d'entrée manuellement dans le Magasin.

**Les coûts (journalier, carburant, maintenance) alimentent-ils la comptabilité ?**
Non. Ils sont affichés seulement, jamais convertis en écritures comptables.

**Un tenant tout neuf affiche des compteurs à zéro : est-ce normal ?**
Oui. Les tables `logistics_*` sont créées à la première saisie. Dès que vous créez une première livraison ou un premier équipement, les compteurs se remplissent.

**L'onglet GPS renvoie une erreur sur un compte qui n'a jamais utilisé le GPS. Pourquoi ?**
Parce que `gps.py` ne crée pas ses tables à la demande. Il faut que les tables `gps_*` existent (données GPS déjà reçues) pour que l'onglet réponde.

**Qui peut créer et supprimer des fiches ?**
Les rôles administrateur, super-administrateur, gestionnaire, contremaître et magasinier (ou tout compte `is_admin`). Les autres rôles obtiennent une erreur 403. Note : l'ajout de lieux et de zones GPS n'a **aucune** restriction de rôle.

**L'assistant IA peut-il voir la paie ou les employés ?**
Non. Il lit uniquement les tables de logistique, plus les projets et les entreprises. La paie, les employés, les salaires et les NAS lui sont interdits, et il n'écrit jamais.

**Combien coûte une question à l'assistant ?**
Le tarif du modèle `claude-sonnet-4-6` majoré de 30 %, débité de vos crédits IA. Le coût exact s'affiche sous chaque réponse. La consultation logistique, elle, est gratuite.

---

## 6. Récapitulatif

- **Objet** : registre opérationnel de la logistique de chantier — livraisons, équipements (+ maintenance), véhicules, coordination, suivi GPS de flotte (lecture) — et un assistant IA de consultation.
- **Accès** : barre latérale → groupe **TERRAIN** → **Logistique** (icône `Truck`), route `/logistique`. Onglet par défaut : Tableau de bord.
- **7 onglets** : Tableau de bord, Livraisons, Équipements, Véhicules, **« Trajets »** (= coordination de chantier), **« GPS / Trajets »** (= flotte GPS), Assistant IA. Deux onglets portent le mot « Trajets », aucun ne gère les trajets de véhicule.
- **3 routers côté serveur** : `secondary.py` (`/logistics/*`, 33 endpoints de gestion + statistiques, plus 4 endpoints IA **morts**), `gps.py` (`/gps/*`, 7 endpoints), `logistique_ai.py` (`/logistique/ai/chat`, l'assistant actif). Il n'existe **aucun** fichier `routers/logistique.py`.
- **Tables** : `logistics_*` par tenant, **créées à la demande** (un tenant neuf voit des compteurs à zéro) ; `gps_*` par tenant, **non créées à la demande** (erreur possible sur un tenant sans données GPS).
- **Permissions** : lecture ouverte à tous ; écriture réservée aux rôles administrateur / super-administrateur / gestionnaire / contremaître / magasinier (ou `is_admin`) ; les écritures GPS n'ont **aucune** garde de rôle ; mode consultation (lecture seule) si l'abonnement Stripe est inactif.
- **Édition limitée** : sur une fiche existante, on ne peut que **changer le statut** (menu en ligne) et **supprimer**. Aucune fenêtre de modification complète.
- **Numéros** : `LIV-#####`, `EQP-#####`, `COORD-#####`, garantis uniques par contrainte de base.
- **Alertes** : trois sources (maintenance et inspection à 7 jours, assurance à 30 jours), priorité haute selon l'imminence — générées et traitées **côté serveur seulement**, affichées en lecture sur le tableau de bord.
- **Réservations d'équipement** : refus de chevauchement (409) **côté serveur**, sans écran.
- **Assistant IA** : chat de **lecture seule**, modèle `claude-sonnet-4-6`, 8000 jetons, outil unique `recherche_bd`, tables sensibles interdites, débit **ferme** des crédits IA (tarif du modèle majoré de 30 %, fonctionnalité `logistique_chat`). Le vrai gardien est le solde de crédits (402) ; `check_ai_guard` est neutre.
- **Ce que le module ne fait pas** : aucune exportation / impression / CSV / PDF / téléversement / action groupée ; pas d'articles de livraison, de réservations, de trajets de véhicule, de création de zone GPS ni de génération d'alertes **dans l'interface** ; pas de carte ; pas de notifications ; pas de synchronisation stock ou comptable ; pas d'assistant qui écrit.

---

**Documentation générée à partir du code** : `ERP_REACT/backend/routers/secondary.py` (endpoints `/logistics/*`, DDL des tables `logistics_*`, règles d'alerte et de réservation) ; `ERP_REACT/backend/routers/gps.py` (372 lignes, endpoints `/gps/*`) ; `ERP_REACT/backend/routers/logistique_ai.py` (331 lignes, assistant `/logistique/ai/chat`) ; `ERP_REACT/backend/erp_auth.py` (gardes `require_tenant_admin_or_role`, mode consultation) ; `ERP_REACT/frontend/src/pages/LogistiquePage.tsx` (1417 lignes, 7 onglets) ; `ERP_REACT/frontend/src/components/logistique/LogistiqueAssistantTab.tsx` (151 lignes) ; `ERP_REACT/frontend/src/api/logistics.ts`, `api/gps.ts`, `api/logistiqueAi.ts` ; textes sous `i18n/locales/fr/terrain.json` (sous-section `logistique.*`) et `i18n/locales/fr/logistiqueAssistant.json`.

**Manuels liés** :
- Module 08 (Projets — références `project_id` faibles) — `08-ventes-projets.md`
- Module 03 (Entreprises / fournisseurs — références `fournisseur_id`) — `03-gestion-entreprises.md`
- Module 09 (Magasin / Inventaire — mouvements de stock manuels après livraison) — `09-operations-magasin.md`
- Module 19 (Location — contrats clients/fournisseurs distincts) — `19-terrain-location.md`
- Module 20 (Maintenance — fiches générales de chantier) — `20-terrain-maintenance.md`
- Module 24 (Assistant IA — portefeuille de crédits partagé) — `24-communication-assistant-ia.md`
