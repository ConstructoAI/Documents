# Module 19 — Location d'équipements et prêts de main-d'œuvre

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — corrections majeures par rapport à la v2.0 : la liste des métiers compte **11 métiers nommés** (plus « Autre » et le filtre « Tous »), et non « 17 » ; il n'existe **aucun** fichier `location_ai.py` — l'assistant IA vit dans `secondary.py`, endpoints `/rental/ia/*` ; le préfixe d'API réel est **`/rental`** et non `/location` ; le **tarif dégressif est bel et bien exposé dans l'écran ET calculé** (ce ne sont plus des colonnes mortes) ; les **frais de retour sont facturables et taxés** (ajoutés au sous-total HT) ; l'écriture est **réservée à certains rôles** et non ouverte à tous ; le contrôle d'accès IA `check_ai_guard` est **neutre** — seul le solde de crédits bloque (erreur 402) ; il y a **7 onglets** (l'Assistant IA est un onglet à part entière) ; les 7 tables sont créées **à la demande** ; aucune facture n'est produite dans la comptabilité.)
> **Libellé dans le menu** : « Location » (groupe « TERRAIN » de la barre latérale, icône `HardHat`) — route `/location`. Réf. `Sidebar.tsx:68,75`, `nav.json:20`.
> **Titre affiché de la page** : « Location d'équipement ».
> **Code de référence (côté serveur)** : tout le module vit dans **`ERP_REACT/backend/routers/secondary.py`** (routeur combiné « Secondary Modules » : Immobilier, Logistique, **Location**, Maintenance, Météo, Conformité, Subventions). La section Location occupe les lignes **3961 → 6180** : **27 points d'accès** sous **`/rental/*`** (22 de gestion et statistiques + 5 pour l'assistant IA), les modèles Pydantic (498-737), la DDL des 7 tables (1274-1435) et l'invite système de l'IA (1608-1651). **Il n'existe aucun fichier `routers/location.py` ni `routers/location_ai.py`.**
> **Chemins d'API réels** : préfixe `/api/erp/v1` — donc `/api/erp/v1/rental/*`. Le nom « `/location` » n'existe **qu'à l'écran** (la route React) ; tous les appels partent vers `/rental/*`.
> **Code de référence (côté client)** : `ERP_REACT/frontend/src/pages/LocationPage.tsx` (2268 lignes, **un seul fichier**, 7 onglets, tous les sous-composants en ligne — le dossier `frontend/src/components/location/` **n'existe pas**) ; `frontend/src/api/location.ts` ; magasin d'état `store/useLocationStore.ts` ; textes sous `i18n/locales/fr/terrain.json` (sous-section `terrain.location.*`).
> **Tables PostgreSQL** (une série **par tenant**, **créées à la demande**) : `location_items`, `location_contrats`, `location_contrat_lignes`, `location_retours`, `employee_location`, `location_contrats_employes`, `location_employes_heures`.
> **Cadrage** : registre **opérationnel interne** en deux volets. (1) Un **parc d'équipements de location** — catalogue, contrats avec lignes détaillées et taxes TPS/TVQ, retours par inspection, statistiques — pour louer du matériel à des clients. (2) Un **prêt / location de main-d'œuvre** — des employés « prêtés » à un autre entrepreneur (configuration, contrats, saisie d'heures). S'y ajoute un **assistant IA** (5 outils). Ce **n'est pas** un module documentaire : il ne produit **aucun** contrat, bon de sortie ni reçu imprimable, et n'émet **aucune** facture dans la comptabilité.

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

Donner à une entreprise de construction québécoise un **registre unique de sa location**, sous deux formes complémentaires :

- **Location d'équipements** : tenir un catalogue de matériel louable (excavatrice, grue, nacelle, échafaudage, génératrice, outillage…), monter des contrats de location pour des clients avec des lignes détaillées et des totaux taxés (TPS 5 %, TVQ 9,975 %), gérer les cautions, puis enregistrer les retours par une inspection comparant l'état avant et après.
- **Prêt de main-d'œuvre** : déclarer quels employés sont « prêtables » à un autre entrepreneur (avec métier, taux horaire et journalier), monter des contrats de prêt et y consigner les heures travaillées.

Le module répond concrètement à des questions comme :

- « Quels équipements sont disponibles à la location aujourd'hui, et à quel tarif ? »
- « Combien vaut ce contrat, taxes comprises, et la caution a-t-elle été reçue ? »
- « Cet équipement est-il rentré, et dans quel état ? Y a-t-il des frais de dommages à facturer ? »
- « Quels employés puis-je prêter la semaine prochaine, et combien d'heures ont-ils faites sur le contrat en cours ? »
- « Location ou achat pour cette excavatrice ? » (assistant IA)

### 1.2 Ce que le module fait (vérifié contre le code)

- **Catalogue d'équipements** : créer, lister (jusqu'à 200 par requête), rechercher, filtrer par catégorie et par état, modifier et supprimer (suppression douce) des fiches de matériel. Chaque fiche porte des tarifs journalier, hebdomadaire et mensuel, une valeur d'achat et de remplacement, une caution, un indicateur d'assurance requise et, en option, un **tarif dégressif** (rabais automatique au-delà d'un seuil de jours).
- **Contrats de location** : créer (numéro `LOC-#####` généré par le serveur), lister, rechercher, filtrer par statut, ouvrir le détail, changer le statut, et supprimer (seulement en `BROUILLON` ou `ANNULE`).
- **Lignes de contrat** : ajouter, modifier et retirer des équipements dans un contrat, avec quantité, tarif, type de tarif, remise et dates de sortie/retour. Le serveur calcule le montant de chaque ligne et recalcule les totaux du contrat (HT, TPS, TVQ, TTC). Une **garde anti-survente** empêche de sortir plus d'unités qu'il n'en reste (erreur 409).
- **Retours par inspection** : pour chaque ligne non rentrée, enregistrer un retour avec l'état après, les dommages constatés et jusqu'à trois frais (réparation, nettoyage, retard). Ces **frais sont facturables** : ils remontent dans le sous-total HT du contrat (donc taxés). Quand toutes les lignes sont rentrées, le contrat passe automatiquement à `RETOURNE`.
- **Prêt de main-d'œuvre** : configurer un employé pour la location (disponibilité, statut, métier, taux), créer des contrats de prêt (numéro `EMP-#####`), en changer le statut (ce qui synchronise automatiquement le statut de l'employé), et saisir des heures travaillées (qui s'additionnent au compteur d'heures réelles du contrat).
- **Tableaux de synthèse** : un tableau de bord (indicateurs, derniers contrats, répartition par catégorie) et un onglet Statistiques (répartition par statut, par état, top 10 des clients par revenu).
- **Assistant IA** : cinq outils (recommandation d'équipements, location contre achat, liste de vérification, analyse de contrat, question libre) qui produisent des conseils rédigés par Claude, facturés aux crédits IA de l'entreprise.

### 1.3 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes, et certaines capacités existent côté serveur sans écran pour les utiliser.

- **Aucune exportation, aucune impression, aucun téléversement de fichier, aucune photo.** Il n'y a **ni PDF, ni CSV, ni impression** dans tout le module : pas de contrat imprimable, pas de bon de sortie, pas de reçu de retour, pas de pièce jointe. Le module ne produit **aucun document remis au client**.
- **Aucune facture dans la comptabilité.** Faire passer un contrat au statut `FACTURE` ne fait que **changer le statut** : les montants restent dans les tables de location, aucune facture n'est créée dans le module Comptabilité. Le report vers la facturation est manuel.
- **Le client d'un contrat est un simple texte libre.** La fenêtre de création ne relie **pas** le contrat à une entreprise, à un contact, à un projet ou à un responsable, même si le serveur sait stocker ces liens (`client_company_id`, `client_contact_id`, `project_id`, `responsable_id`). Seul le **nom saisi à la main** est conservé.
- **Édition partielle des contrats.** Dans le détail d'un contrat, on peut changer le **statut** et l'indicateur **« Caution reçue »**, mais la fenêtre de création (client, dates, durée, lieu, caution, conditions) n'est **pas** rééditable après coup depuis l'écran ; l'équipement d'une ligne existante n'est **pas** modifiable (seuls la quantité, le tarif, le type, la remise et les dates le sont).
- **Aucune détection de conflit de réservation dans le temps.** La disponibilité est une quantité (total moins unités sorties), pas un calendrier : le module ne prévient pas qu'un équipement est déjà réservé sur une période future. Il refuse seulement de **dépasser la quantité physique** au moment de la sortie.
- **Le prêt de main-d'œuvre ne calcule pas de total facturable avec les types de tarif de l'écran.** Le calcul automatique du montant (estimé et facturé à partir des heures) n'existe **que** pour un type de tarif « à l'heure » **que le menu déroulant n'offre pas** (il propose jour, semaine, mois, forfait). Avec ces quatre types, la colonne « Montant » reste « -- » tant qu'on n'a pas saisi un montant par l'API (voir 2.6.3 et 4.9).
- **Les heures de prêt ne vont pas dans la paie.** Les heures saisies s'additionnent au compteur du contrat de prêt, mais **ne sont pas** propagées dans le module Paie / Pointage.
- **Aucune notification automatique** (courriel, message texte, notification navigateur) sur les retards, les échéances ou les cautions.
- **L'assistant IA est bloqué en mode consultation.** Comme ses cinq points d'accès sont des requêtes d'écriture (POST), un tenant en **mode consultation** (abonnement Stripe inactif) ne peut **pas** les utiliser, même si l'IA ne fait conceptuellement que lire (voir 1.5).

### 1.4 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **Location** (icône `HardHat`). Réf. `Sidebar.tsx:68,75`.
- URL directe : `/location`.
- Titre de la page et fil d'Ariane : **« Location d'équipement »**.
- **Onglet par défaut** : Tableau de bord.

> **Deux noms pour la même chose.** Le menu affiche « Location », la page s'intitule « Location d'équipement », et l'onglet qui liste les contrats s'appelle **« Locations »** (sa clé interne est pourtant `contrats`). L'onglet **« Main-d'œuvre »** correspond au **prêt de main-d'œuvre**. Ces écarts d'étiquette sont normaux.

### 1.5 Permissions et rôles

Le module distingue nettement la **lecture** (ouverte) de l'**écriture** (réservée à certains rôles).

| Action | Qui peut la faire |
|--------|-------------------|
| **Consulter** (tous les onglets, tous les GET : catalogue, contrats, détail, retours, main-d'œuvre, statistiques) | Tout utilisateur authentifié du tenant (`get_current_user`). |
| **Créer / modifier / supprimer** un équipement, un contrat, une ligne, un retour, une configuration d'employé, un contrat de prêt ou des heures | Rôle d'écriture location : **administrateur**, **super-administrateur**, **gestionnaire**, **contremaître** ou **magasinier** (`require_tenant_admin_or_role`, `secondary.py:44,49` → `erp_auth.py:720`). Un propriétaire dont le compte porte `is_admin` est toujours autorisé (droit relu au serveur, infalsifiable). Sinon : **403**. |
| **Utiliser l'assistant IA** | Tout utilisateur authentifié **ayant des crédits IA** (voir 1.7). |

> **Rôles d'écriture identiques à la Logistique.** `RENTAL_WRITE_ROLES` vaut exactement `LOGISTICS_WRITE_ROLES` : `admin`, `super_admin`, `gestionnaire`, `contremaitre`, `magasinier`.

> **Mode consultation (lecture seule) à l'échelle du tenant.** Si l'entreprise n'a pas d'abonnement Stripe actif (annulé ou absent), tout le tenant passe en **mode consultation** : les lectures restent permises, mais **toute** requête d'écriture renvoie **403** (`erp_auth.py:526`). Comme les cinq points d'accès de l'assistant IA sont des POST, ils tombent aussi sous cette règle : **l'assistant IA n'est pas utilisable en mode consultation.** Voir le module Configuration / Abonnement pour rétablir l'accès.

> **Un tenant qui n'a jamais ouvert la page a des tables vides.** Les 7 tables `location_*` sont créées **à la première utilisation** (à la volée). Un tenant neuf voit des compteurs à **0** et des listes vides : c'est normal, pas une erreur.

### 1.6 Les 7 onglets

Source : `LocationPage.tsx:200-208`. Les compteurs entre parenthèses sont dynamiques.

| # | Clé interne | Libellé affiché | Icône | Compteur | Contenu réel |
|---|-------------|-----------------|-------|----------|--------------|
| 1 | `dashboard` | Tableau de bord | `BarChart3` | — | Indicateurs, derniers contrats, répartition par catégorie |
| 2 | `catalogue` | Catalogue (N) | `HardHat` | nombre d'équipements | Parc d'équipements louables (créer / modifier / supprimer) |
| 3 | `contrats` | **Locations** (N) | `FileText` | nombre de contrats | Contrats de location + détail (lignes, totaux, statut) |
| 4 | `retours` | Retours | `RotateCcw` | — | Contrats en attente de retour + retours complétés |
| 5 | `employes` | **Main-d'œuvre** | `Users` | — | Prêt de main-d'œuvre (4 sous-onglets) |
| 6 | `statistiques` | Statistiques | `ClipboardList` | — | Indicateurs, répartitions, top 10 des clients |
| 7 | `ia` | Assistant IA | `Sparkles` | — | 5 outils d'aide (payants aux crédits IA) |

### 1.7 Coûts et facturation (assistant IA seulement)

- **Toute la partie registre est gratuite** : gérer le catalogue, les contrats, les lignes, les retours et le prêt de main-d'œuvre ne consomme aucun crédit.
- **Seul l'assistant IA est payant.** Chaque appel d'un des cinq outils consomme des **crédits IA prépayés** de l'entreprise. Le coût est le tarif du modèle utilisé **majoré de 30 %** :
  - Outils **Claude Sonnet 4-6** (question libre, liste de vérification, location contre achat) : 3 $ US par million de jetons en entrée, 15 $ US par million en sortie, × 1,30.
  - Outils **Claude Opus 4-8** (recommandation d'équipements, analyse de contrat) : 5 $ US par million de jetons en entrée, 25 $ US par million en sortie, × 1,30.
- Le coût exact s'affiche sous chaque réponse : « **Coût : {montant} $ US** ». L'usage est tracé sous une fonctionnalité `location_chat`, `location_recommander`, `location_analyser_contrat`, `location_checklist` ou `location_compare_achat` dans le suivi d'usage IA du super-administrateur.
- Un compte **sans crédits** reçoit une erreur **402** « Crédits IA insuffisants » et ne peut pas lancer l'outil ; le registre de location, lui, reste gratuit.

> **Attention — pas d'idempotence sur le débit IA.** Le débit des crédits est effectué **sans clé d'idempotence** : un **double-clic** ou une **reprise réseau** sur un même appel peut **débiter deux fois**. Attendez la réponse (ou le message d'erreur) avant de relancer un outil.

### 1.8 Architecture technique

```
Frontend  LocationPage.tsx (2268 lignes, 7 onglets, un seul fichier)
    │
    ├── Tableau de bord / Catalogue / Locations / Retours / Main-d'œuvre / Statistiques
    │        └─ api/location.ts ──> secondary.py  /api/erp/v1/rental/*   (22 endpoints gestion + statistiques)
    │                                tables location_* (7, créées à la demande, par tenant)
    │
    └── onglet Assistant IA
             └─ api/location.ts ──> secondary.py  /api/erp/v1/rental/ia/*  (5 endpoints POST)
                                     Claude sonnet-4-6 (chat / checklist / location-vs-achat)
                                     Claude opus-4-8  (recommander / analyser-contrat)
                                     débit des crédits IA prépayés + traçage de l'usage
```

> **Point d'attention pour un tenant neuf.** Les tables `location_*` sont créées **à la première utilisation** (`_ensure_table`, avec rattrapage de colonnes `_ensure_columns` pour les installations plus anciennes). Elles ne sont **pas** créées à l'ouverture du tenant ni par la réparation automatique de démarrage. Concrètement : dès la première fiche saisie, tout se met en place ; avant cela, les compteurs restent à zéro.

---

## 2. Interface

Source : `LocationPage.tsx` (2268 lignes, sous-composants en ligne).

### 2.1 Disposition générale

- **Titre** « Location d'équipement » toujours affiché en haut (`:214`).
- **Bandeau d'erreur** (rouge, fermable) et **bandeau de succès** (vert, fermable) apparaissent au-dessus des onglets après une action. Le bandeau de succès **s'efface tout seul après 3,5 secondes** (`:166-170`) pour ne pas s'empiler ni rester derrière une fenêtre.
- **Barre d'onglets** défilable horizontalement sur petit écran ; l'onglet actif est souligné.
- **Adaptatif** : chaque tableau (affichage bureau) se transforme en **cartes empilées** sur téléphone.

> **Messages de succès (toasts).** Selon l'action : « Équipement enregistré / supprimé », « Contrat enregistré / supprimé », « Ligne enregistrée / supprimée », « Retour enregistré », « Configuration enregistrée », « Contrat employé enregistré », « Heures enregistrées ».

### 2.2 Onglet Tableau de bord

Source : `:289-354`. Nourri par `GET /rental/statistics` et l'agrégation locale des listes déjà chargées.

**Quatre cartes d'indicateurs :**

| Carte | Valeur |
|-------|--------|
| Équipements | Nombre total de fiches au catalogue |
| Disponibles | Fiches dont l'état n'est **pas** `REPARATION` et qui ne sont pas marquées indisponibles |
| Contrats actifs | Compteur `actifs` renvoyé par le serveur |
| Montant total | Somme des montants totaux des contrats (`montantTotal` du serveur) |

**Carte « Derniers contrats »** : les 5 contrats les plus récents (nom du client, numéro, date, montant, badge de statut). Vide : « Aucun contrat ».

**Carte « Équipements par catégorie »** : comptage par catégorie (affichée seulement s'il y a au moins un équipement).

### 2.3 Onglet Catalogue

Source : `:358-640`.

**Barre d'outils** : bouton **« Ajouter un item »**, champ de recherche (« Rechercher... », filtre local sur le nom, le numéro de série, la marque, le modèle et la catégorie), filtre **Catégorie** (10 valeurs + « Toutes ») et filtre **État** (6 valeurs + « Tous »).

**Tableau (affichage bureau)**, colonnes triables (clic sur l'en-tête) :

| Colonne | Contenu |
|---------|---------|
| Équipement | Nom de la fiche |
| N/S | Numéro de série |
| Catégorie | Catégorie |
| État | Badge de couleur (voir 4.1) |
| Dispo | Badge « quantité disponible / quantité totale », ou « Oui / Non » |
| Tarif/jour · Tarif/sem · Tarif/mois | Tarifs mis en forme en dollars |
| Actions | Crayon (Modifier) · Poubelle (Supprimer) |

Liste vide : « Aucun équipement ». Sur téléphone : cartes empilées.

> **Suppression.** Confirmation « Supprimer cet équipement ? Cette action est irréversible. » Le serveur fait une **suppression douce** (`actif = FALSE`, `disponible = FALSE`) — la fiche disparaît des listes mais reste en base. La suppression est **refusée (400)** si l'équipement est engagé dans un contrat actif (`BROUILLON`, `EN_COURS`, `ACTIF`, `RESERVE` ou `EN_RETARD`).

**Fenêtre « Créer / Modifier un équipement »** (taille large) :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Nom | texte | **Oui** (seul champ obligatoire) |
| Numéro de série | texte | non |
| Catégorie | menu (10 valeurs) | non |
| État | menu (6 valeurs, défaut « Bon ») | non |
| Marque · Modèle | texte · texte | non |
| Année de fabrication | nombre (1900-2100) | non |
| Quantité totale | nombre (défaut 1) | non |
| Valeur d'achat ($) · Valeur de remplacement ($) | nombre · nombre | non |
| Tarif journalier ($) · hebdomadaire ($) · mensuel ($) | nombre · nombre · nombre | non |
| Caution requise ($) | nombre | non |
| Description · Conditions de location · Notes | zones de texte | non |

**Encadré « Tarif dégressif »** : une case **« Tarif dégressif actif »** qui, une fois cochée, révèle **Seuil (jours)** (exemple : 7) et **Réduction (%)** (exemple : 10), avec la note « Applique une réduction automatique lorsque la durée de location atteint le seuil en jours. » Ce rabais est **réellement appliqué** au calcul du montant des lignes (voir 4.9).

**Case « Assurance requise »** : indicateur informatif.

Le bouton Créer / Enregistrer est **désactivé tant que le nom est vide**.

### 2.4 Onglet Locations

Source : `:644-1097`. C'est l'onglet des **contrats de location d'équipement** (clé interne `contrats`).

**Barre d'outils** : bouton **« Nouveau contrat »**, recherche locale, filtre par statut (7 statuts + « Tous »).

**Tableau**, colonnes triables :

| Colonne | Contenu |
|---------|---------|
| Contrat | Numéro `LOC-#####` |
| Client | Nom du client |
| Début · Fin prévue | Dates |
| Statut | Badge de couleur |
| Montant | Total TTC |
| Actions | Œil (Détails) |

Cliquer sur une ligne ouvre le détail. Liste vide : « Aucun contrat ». Sur téléphone : cartes empilées.

**Fenêtre « Nouveau contrat de location »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Client | texte libre (stocké tel quel) | **Oui** |
| Type de client | menu : Entreprise / Contact (informatif) | non |
| Date début · Date fin prévue | date · date | non |
| Type de durée | menu : Par jour / Par semaine / Par mois / Forfait | non |
| Nombre de périodes | nombre | non |
| Lieu de livraison | texte | non |
| Montant de la caution | nombre | non |
| Conditions particulières · Notes | zones de texte | non |

Le bouton Créer est **désactivé tant que le client est vide**. À la création, le serveur attribue le numéro `LOC-#####` et pose le statut `BROUILLON`.

> **Ce que la fenêtre ne montre pas.** Aucun champ pour rattacher le contrat à une entreprise, un contact, un projet ou un responsable : seul le **nom** du client est saisi. Les liens existent en base mais ne sont pas exposés ici.

**Fenêtre « Contrat {numéro} » (détail, taille très large)** :

- **En-tête** : Client, **Période** (début → fin), **Statut** (menu déroulant des 7 statuts, appliqué immédiatement par `PUT`), Lieu de livraison (si présent), **Caution** avec la case **« Caution reçue »** (si la caution est supérieure à 0), Conditions (si présentes).
- **Tableau « Lignes du contrat »** : colonnes **Item, Qté, Tarif, Type, Remise, Montant**, avec un crayon (modifier) et une poubelle (supprimer) par ligne. Vide : « Aucune ligne ».
- **Formulaire pour ajouter / modifier une ligne** :

| Champ | Type | Remarque |
|-------|------|----------|
| Équipement | menu du catalogue | **verrouillé en modification** (on ne change pas l'équipement d'une ligne existante) |
| Qté | nombre | > 0 |
| Tarif ($) | nombre | > 0 |
| Type | menu : JOUR / SEM / MOIS / FORFAIT | — |
| Remise (%) | nombre | 0 à 100 |
| Date de sortie · Date de retour prévue | date · date | déterminent la durée facturée (et le tarif dégressif s'il est actif) |

  Si le catalogue est vide, une alerte le signale. Le bouton Ajouter / Enregistrer est **désactivé** si aucun équipement n'est choisi, si le tarif est ≤ 0 ou si la quantité est ≤ 0.
- **Totaux** (calculés par le serveur) : **Sous-total HT**, **TPS (5 %)**, **TVQ (9,975 %)**, **Total TTC**.
- **« Supprimer ce contrat »** : bouton visible **uniquement** si le statut est `BROUILLON` ou `ANNULE`. Confirmation « Supprimer le contrat {numéro} ? Cette action est irréversible. » La suppression retire en cascade les lignes et les retours, puis recalcule la disponibilité des équipements concernés.

### 2.5 Onglet Retours

Source : `:1101-1369`. Cet onglet organise les **retours par inspection**.

**Carte « Contrats actifs — en attente de retour ({N}) »** : les contrats dont le statut est `EN_COURS`, `RESERVE`, `ACTIF` ou `EN_RETARD`. Pour chaque contrat : nom du client, numéro et « Depuis {date} », bouton **« Voir les lignes »**. Les lignes **non rentrées** (sans date de retour réelle) sont affichées avec « Sorti : {date} · État : {état} » et un bouton **« Enregistrer le retour »**. Si tout est rentré : « Toutes les lignes ont été retournées. » S'il n'y a aucun contrat actif : « Aucun contrat actif ».

**Carte « Retours complétés ({N}) »** : tableau des retours déjà saisis — **Contrat, Item, Date retour, État avant** (badge), **État après** (badge), **Frais total** (= réparation + nettoyage + retard). Vide : « Aucun retour enregistré ».

**Fenêtre « Inspection de retour »** :

| Champ | Type | Remarque |
|-------|------|----------|
| État à la sortie | badge en lecture seule | rappel de l'état à la sortie |
| État après retour | menu (6 états) | l'état constaté au retour |
| Dommages constatés | zone de texte | « Décrire les dommages observés... » |
| Frais réparation ($) · Frais nettoyage ($) · Frais retard ($) | nombre · nombre · nombre | facturables (voir ci-dessous) |
| Commentaires | zone de texte | — |

> **Effets d'un retour.** Le serveur verrouille la ligne, **refuse un double retour** (409), déduit l'équipement **de la ligne elle-même** (une valeur envoyée par le client serait ignorée), pose la date de retour réelle à maintenant, enregistre l'état après, **propage cet état vers la fiche de l'équipement** (un matériel endommagé ne reste pas « Bon ») et, quand **toutes** les lignes du contrat sont rentrées, **passe le contrat à `RETOURNE`** automatiquement.

> **Frais de retard automatiques.** Si vous **laissez le champ « Frais retard » vide** et que le retour est en retard, le serveur calcule lui-même le retard en ramenant le tarif à un équivalent journalier (diviseur 7 pour un tarif à la semaine, 30 pour un tarif au mois) × quantité × jours de retard. Un **FORFAIT est exclu** de ce calcul automatique (sinon le forfait entier serait refacturé par jour de retard). Saisir **explicitement 0** dans « Frais retard » vaut renonciation (distinct du champ laissé vide).

### 2.6 Onglet Main-d'œuvre

Source : `:1375-1932`. Gère le **prêt de main-d'œuvre** : des employés « prêtés » à un autre entrepreneur. Cet onglet contient **4 sous-onglets** : **Tableau de bord**, **Employés**, **Contrats**, **Heures**.

#### 2.6.1 Sous-onglet Tableau de bord

Six indicateurs (source `GET /rental/employees/stats`) : **Total employés**, **En location**, **Disponibles**, **Contrats actifs**, **Heures totales**, **Montant facturé**.

#### 2.6.2 Sous-onglet Employés

- **Barre d'outils** : filtre **Métier** (11 métiers + « Autre » + « Tous »), bouton **« Ajouter un employé »**, compteur.
- **Tableau** : **Nom, Métier, Statut** (badge : Disponible / En location / Indisponible / En congé), **Taux horaire, Taux journalier, Actions** (bouton **« Configurer »**). Vide : « Aucun employé ».

**Fenêtre « Configuration — {nom} »** :

| Champ | Type |
|-------|------|
| Disponible pour location | case à cocher |
| Statut location | menu (Disponible / En location / Indisponible / En congé) |
| Métier principal | menu (11 métiers + « Autre ») |
| Taux horaire ($) · Taux journalier ($) | nombre · nombre |
| Notes | zone de texte |

**Fenêtre « Ajouter un employé »** : elle liste les employés du **module RH** qui ne sont **pas encore** configurés pour la location ; les sélectionner crée leur fiche de location (opération de type « créer ou mettre à jour »). Si tous sont déjà configurés : « Tous les employés sont déjà configurés pour la location. »

> **Un employé n'apparaît dans la liste que s'il a été configuré.** La liste montre les employés déjà présents dans la table de location. Pour ajouter quelqu'un, passez par **« Ajouter un employé »** (qui puise dans le module RH).

#### 2.6.3 Sous-onglet Contrats

- **Barre d'outils** : bouton **« Nouveau contrat employé »**, compteur.
- **Tableau** : **Numéro** (`EMP-#####`), **Employé**, **Statut** (badge), **Dates**, **Tarif** (montant + type), **Heures P/R** (prévues / réelles), **Montant**, **Action** (menu déroulant de statut : Brouillon / En cours / Terminé / Facturé / Annulé, appliqué par `PUT`). Vide : « Aucun contrat employé ».

**Fenêtre « Nouveau contrat employé »** :

| Champ | Type | Obligatoire |
|-------|------|-------------|
| Employé | menu (uniquement les employés « Disponible pour location ») | **Oui** |
| Date début · Date fin prévue | date · date | **Oui** |
| Type de tarif | menu : Par jour / Par semaine / Par mois / Forfait | non |
| Tarif unitaire ($) | nombre | non |
| Heures prévues | nombre | non |
| Lieu de travail | texte | non |
| Description de la mission · Notes | zones de texte | non |

Si aucun employé n'est disponible, une alerte le signale.

> **Important — la colonne « Montant » reste souvent « -- ».** Le serveur ne calcule un montant estimé et un montant facturé **automatiquement** que pour un type de tarif **« à l'heure »**, or le menu **n'offre pas** ce type (seulement jour, semaine, mois, forfait). Avec ces quatre types, aucun total n'est produit : la colonne « Montant » affiche « -- » jusqu'à ce qu'un montant soit posé par l'API. La **saisie des heures** (2.6.4) alimente le compteur d'heures réelles mais, avec ces types, ne génère **pas** de montant facturable.

> **Synchronisation automatique du statut de l'employé.** Passer un contrat de prêt à `EN_COURS` ou `ACTIF` met l'employé en **« En location »** ; le passer à `TERMINE`, `ANNULE` ou `FACTURE` le remet **« Disponible »**.

#### 2.6.4 Sous-onglet Heures

**Formulaire « Saisie des heures »** :

| Champ | Type | Remarque |
|-------|------|----------|
| Contrat employé | menu | **uniquement** les contrats `EN_COURS` ou `ACTIF` |
| Date de travail | date | obligatoire |
| Heures normales | nombre | exemple : 8 |
| Heures supplémentaires | nombre | exemple : 0 |
| Description des tâches | zone de texte | — |

Bouton **« Enregistrer les heures »**. S'il n'y a aucun contrat actif, une alerte le signale. À l'enregistrement, le serveur **additionne** les heures (normales + supplémentaires) au compteur d'heures réelles du contrat.

> **Le tableau « Saisies récentes » est éphémère.** Il affiche les 20 dernières saisies **de la session en cours seulement** (état local, non rechargé depuis la base). Il **se vide** si vous quittez et revenez sur le sous-onglet. C'est un simple retour visuel, pas un journal persistant.

### 2.7 Onglet Statistiques

Source : `:2151-2267`. Nourri par `GET /rental/statistics` + agrégation locale.

**Cinq indicateurs** : **Total contrats**, **Actifs**, **Terminés** (`TERMINE` + `RETOURNE`), **Éq. loués** (nombre d'équipements sortis, `equipementsLoues` du serveur), **Revenu total**.

**« Contrats par statut »** : une barre de progression par statut (pourcentage).
**« Équipements par état »** : une barre par état.
**« Top clients par revenu »** : tableau (#, Client, Revenu), **top 10**. Vide : « Aucune donnée ».

### 2.8 Onglet Assistant IA

Source : `:1953-2147`. Titre « Assistant IA — Location ». **Cinq outils**, chacun avec un bouton **« Lancer »**, un indicateur « Analyse en cours... », un résultat en texte ou en JSON, et le **coût affiché** sous la réponse (« Coût : {montant} $ US »).

| Outil | Modèle | Ce qu'on saisit |
|-------|--------|-----------------|
| **Recommander** | Claude Opus 4-8 | Description du projet + Budget ($, optionnel) + Durée (jours, optionnel) |
| **Location vs achat** | Claude Sonnet 4-6 | Équipement + Prix d'achat ($) + Tarif de location/jour ($) + Utilisation (jours/an) |
| **Liste de vérification** | Claude Sonnet 4-6 | Type d'équipement + Durée de location |
| **Analyser un contrat** | Claude Opus 4-8 | Un contrat à choisir dans la liste |
| **Question libre** | Claude Sonnet 4-6 | Votre question, en texte libre |

> **Ce que l'assistant est et n'est pas.** C'est un **conseiller** : il rédige des recommandations, des comparaisons et des listes de vérification (sécurité CNESST, certifications d'opérateur, documents requis…) à partir de ce que vous saisissez et, pour l'analyse de contrat, à partir des données du contrat choisi. Il **n'écrit rien** dans vos données : il ne crée pas de fiche, ne modifie pas de contrat, ne pose pas de ligne. Les résultats sont des **brouillons de réflexion**, à valider par un professionnel — surtout les coûts et les enjeux de sécurité.

> **Erreurs possibles.** 402 « Crédits IA insuffisants » (rechargez le solde) ; 503 si le service IA n'est pas configuré ; 413 si la demande est trop volumineuse ; 403 en mode consultation (voir 1.5).

### 2.9 Éléments transverses

- **Recherche** : toujours **locale** (elle filtre la page déjà chargée). Présente au Catalogue et aux Locations.
- **Tri** : clic sur les en-têtes de colonne des tableaux principaux.
- **Changement de statut** : par **menu déroulant** (dans le détail d'un contrat, ou dans la ligne d'un contrat de prêt), appliqué immédiatement.
- **Pas de pagination à l'écran** : le catalogue et les contrats sont chargés jusqu'à **200 éléments** d'un coup (le maximum côté serveur).
- **Aucune exportation, impression, CSV, PDF, téléversement ni action groupée** nulle part.

---

## 3. Workflows pas à pas

### 3.1 Ajouter un équipement au catalogue

1. Onglet **Catalogue** → **« Ajouter un item »**.
2. Saisir le **Nom** (obligatoire) ; au besoin la catégorie, l'état, la marque, le modèle, l'année, la **quantité totale**, les valeurs d'achat et de remplacement.
3. Renseigner les **tarifs** journalier, hebdomadaire et/ou mensuel, la **caution requise** et l'indicateur **« Assurance requise »**.
4. (Optionnel) Cocher **« Tarif dégressif actif »** et régler le **Seuil (jours)** et la **Réduction (%)** pour un rabais automatique sur les longues durées.
5. **Créer**. L'équipement est immédiatement actif et disponible ; sa disponibilité est recalculée à chaque mouvement.

### 3.2 Créer un contrat de location

1. Onglet **Locations** → **« Nouveau contrat »**.
2. Saisir le **Client** (obligatoire, texte libre) ; au besoin le type de client, les dates, le type de durée, le nombre de périodes, le lieu de livraison, la caution, les conditions et des notes.
3. **Créer**. Le serveur attribue le numéro `LOC-#####` et pose le statut `BROUILLON`. Le contrat apparaît dans le tableau.

### 3.3 Ajouter des équipements (lignes) à un contrat

1. Onglet **Locations** → cliquer sur la ligne du contrat pour ouvrir son **détail**.
2. Dans le formulaire de ligne : choisir l'**Équipement**, la **Qté**, le **Tarif ($)**, le **Type** (jour / semaine / mois / forfait), la **Remise (%)**, et les **dates de sortie et de retour prévu**.
3. **Ajouter**. Le serveur calcule le montant de la ligne (durée × tarif × quantité, moins la remise, puis le tarif dégressif s'il s'applique) et **recalcule les totaux** du contrat (HT, TPS, TVQ, TTC).

> **Garde anti-survente.** Si vous demandez plus d'unités qu'il n'en reste de disponibles (compte tenu des autres contrats actifs), le serveur refuse la ligne avec l'erreur **409 « Disponibilité insuffisante »**. Même contrôle si vous **augmentez** la quantité d'une ligne existante.

### 3.4 Faire avancer un contrat (changer le statut)

1. Ouvrir le détail du contrat.
2. Dans l'en-tête, choisir le nouveau **Statut** dans le menu déroulant : `BROUILLON` → `RESERVE` → `EN_COURS` → `RETOURNE` → `FACTURE`, plus `ANNULE` et `EN_RETARD` au besoin.
3. Le changement est appliqué immédiatement.

> **Garde d'activation.** Le serveur **refuse (422)** de passer un contrat à `EN_COURS` ou `FACTURE` s'il ne contient **aucune ligne**. Ajoutez au moins un équipement d'abord.

> **Rappel.** Passer un contrat à `FACTURE` ne crée **aucune facture** dans la comptabilité : cela ne change que le statut.

### 3.5 Enregistrer un retour avec inspection

1. Onglet **Retours** → repérer le contrat dans « Contrats actifs — en attente de retour » → **« Voir les lignes »**.
2. Sur une ligne non rentrée, cliquer **« Enregistrer le retour »**.
3. Dans la fenêtre d'inspection : choisir l'**État après retour**, décrire les **Dommages constatés**, saisir au besoin les **frais** (réparation, nettoyage, retard) et des commentaires.
4. **Enregistrer le retour**. La ligne est marquée rentrée, l'état de l'équipement est mis à jour, et si toutes les lignes du contrat sont rentrées, le contrat passe à `RETOURNE`.

> Laissez « Frais retard » **vide** pour que le système calcule le retard à votre place ; mettez **0** pour renoncer explicitement aux frais de retard.

### 3.6 Supprimer un contrat

1. Ouvrir le détail du contrat.
2. Le bouton **« Supprimer ce contrat »** n'apparaît que si le statut est `BROUILLON` ou `ANNULE`. Sinon, passez d'abord le contrat à `ANNULE`.
3. Confirmer. Le contrat, ses lignes et ses retours sont supprimés ; la disponibilité des équipements est recalculée.

### 3.7 Configurer un employé pour le prêt de main-d'œuvre

1. Onglet **Main-d'œuvre** → sous-onglet **Employés**.
2. Si l'employé n'est pas dans la liste : **« Ajouter un employé »**, le choisir parmi les employés RH non configurés.
3. Cliquer **« Configurer »** sur sa ligne : cocher **« Disponible pour location »**, régler le **statut**, le **métier principal**, les **taux** horaire et journalier, ajouter des notes.
4. **Enregistrer**.

### 3.8 Créer un contrat de prêt de main-d'œuvre

1. Onglet **Main-d'œuvre** → sous-onglet **Contrats** → **« Nouveau contrat employé »**.
2. Choisir l'**Employé** (obligatoire, uniquement les « disponibles »), la **Date début** et la **Date fin prévue** (obligatoires) ; au besoin le type de tarif, le tarif unitaire, les heures prévues, le lieu et la description de la mission.
3. **Créer**. Le serveur attribue le numéro `EMP-#####` et pose le statut `BROUILLON`.
4. Faire avancer le contrat par le menu de statut de sa ligne (`EN_COURS` met l'employé « En location » ; `TERMINE` / `FACTURE` / `ANNULE` le remet « Disponible »).

### 3.9 Saisir des heures sur un contrat de prêt

1. Onglet **Main-d'œuvre** → sous-onglet **Heures**.
2. Choisir le **Contrat employé** (uniquement ceux `EN_COURS` / `ACTIF`), la **Date de travail**, les **heures normales** et **supplémentaires**, la description des tâches.
3. **Enregistrer les heures**. Les heures s'ajoutent au compteur d'heures réelles du contrat.

> Avec les types de tarif offerts (jour, semaine, mois, forfait), la saisie d'heures **n'engendre pas** de montant facturable automatique ; elle sert au suivi. Reportez la facturation dans la comptabilité manuellement.

### 3.10 Utiliser l'assistant IA

1. Onglet **Assistant IA** → choisir l'outil.
2. Renseigner les champs (par exemple, pour « Location vs achat » : l'équipement, le prix d'achat, le tarif journalier de location et l'utilisation annuelle).
3. **Lancer**. La réponse s'affiche (texte ou JSON) ; le **coût** apparaît en dessous et est débité des crédits IA de l'entreprise.

> N'appuyez qu'une fois sur **« Lancer »** et attendez la réponse : le débit n'a pas de protection contre le double-clic (voir 1.7).

### 3.11 Comprendre un refus (403 / mode consultation / 402)

- **403 sur une création ou une suppression** alors que vous êtes connecté : votre rôle n'est probablement pas dans la liste d'écriture (administrateur, gestionnaire, contremaître, magasinier). Demandez à un administrateur.
- **Toutes** les écritures échouent pour **tout le monde**, et l'assistant IA renvoie 403 : le tenant est en **mode consultation** (abonnement Stripe inactif). Régularisez l'abonnement pour rétablir l'écriture.
- **402 « Crédits IA insuffisants »** dans l'assistant : rechargez le solde de crédits IA de l'entreprise (module Configuration / Abonnement). Le registre de location, lui, reste gratuit.

---

## 4. Référence

### 4.1 Statuts par entité

| Entité | Statuts | Défaut |
|--------|---------|--------|
| **Équipement** | `actif` (vrai/faux, suppression douce) + `disponible` (vrai/faux, recalculé) + `etat` : `NEUF`, `EXCELLENT`, `BON`, `ACCEPTABLE`, `USURE`, `REPARATION` | `actif=vrai`, `disponible=vrai`, `etat='BON'` |
| **Contrat de location** | `BROUILLON`, `RESERVE`, `EN_COURS`, `RETOURNE`, `FACTURE`, `ANNULE`, `EN_RETARD` (le serveur reconnaît aussi `ACTIF` et `TERMINE` à l'affichage) | `BROUILLON` |
| **Employé (prêt)** | `DISPONIBLE`, `EN_LOCATION`, `INDISPONIBLE`, `EN_CONGE` | `DISPONIBLE` |
| **Contrat de prêt** | `BROUILLON`, `EN_COURS`, `TERMINE`, `FACTURE`, `ANNULE` (le serveur reconnaît aussi `RESERVE` et `ACTIF`) | `BROUILLON` |

**Couleurs de badge** (`statutColor`, `:112-121`) : vert (actif / disponible / neuf / excellent), bleu (en cours / réservé / bon), jaune (retard / acceptable / usure), rouge (annulé / réparation), sarcelle (terminé / retourné / facturé), gris (brouillon / autre).

> **Le statut n'est pas verrouillé par des transitions.** Hormis la garde d'activation (pas de passage à `EN_COURS`/`FACTURE` sans ligne) et la garde de suppression (`BROUILLON`/`ANNULE` seulement), n'importe quel statut de la liste peut être choisi. La séquence logique recommandée reste `BROUILLON` → `RESERVE` → `EN_COURS` → `RETOURNE` → `FACTURE`.

### 4.2 Catégories, états, types de tarif et métiers

Ces listes vivent **côté écran** (`LocationPage.tsx`). En base, `categorie` et `metier_principal` sont des textes libres : les listes sont des **suggestions**, pas des contraintes.

| Champ | Valeurs |
|-------|---------|
| **Catégorie d'équipement** (10) | Excavatrice, Grue, Chargeuse, Compacteur, Échafaudage, Bétonnière, Génératrice, Nacelle, Outil, Autre |
| **État d'équipement** (6) | Neuf, Excellent, Bon, Acceptable, Usure, Réparation |
| **Type de tarif** (4) | Par jour (`JOUR`), Par semaine (`SEMAINE`), Par mois (`MOIS`), Forfait (`FORFAIT`) |
| **Statut d'employé** (4) | Disponible, En location, Indisponible, En congé |
| **Métier principal** (11 nommés + « Autre ») | Charpentier-menuisier, Électricien, Plombier, Soudeur, Opérateur équipement lourd, Grutier, Briqueteur-maçon, Peintre, Mécanicien de chantier, Manœuvre, Contremaître |

> **Correction de la v2.0.** L'ancien manuel évoquait « 17 » (ou « 12 ») métiers : la seule liste réelle compte **11 métiers nommés**, plus « Autre », plus le filtre « Tous ». Aucun enum de métier n'existe côté serveur.

### 4.3 Points d'accès — Catalogue (`/rental/items`)

Préfixe : `/api/erp/v1`. Colonne « UI » = accessible depuis l'écran.

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/rental/items` | lecture | Oui | Pagination (≤ 200/page), filtres `categorie`, `etat`, `disponible`. |
| POST | `/rental/items` | écriture | Oui | Crée une fiche ; seul `nom` est obligatoire. |
| PUT | `/rental/items/{id}` | écriture | Oui | Mise à jour (colonnes en liste blanche). |
| DELETE | `/rental/items/{id}` | écriture | Oui | Suppression douce ; **400** si l'équipement est dans un contrat actif. |

### 4.4 Points d'accès — Contrats et lignes

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/rental/contracts` | lecture | Oui | Pagination (≤ 200/page), filtre `statut`. |
| POST | `/rental/contracts` | écriture | Oui | Génère `LOC-#####`, statut `BROUILLON`. |
| GET | `/rental/contracts/{id}` | lecture | Oui | Contrat + ses lignes. |
| PUT | `/rental/contracts/{id}` | écriture | Oui | Statut, caution reçue, etc. **422** si passage à `EN_COURS`/`FACTURE` sans ligne. |
| DELETE | `/rental/contracts/{id}` | écriture | Oui | **400** hors `BROUILLON`/`ANNULE` ; sinon cascade lignes + retours. |
| POST | `/rental/contracts/{id}/lignes` | écriture | Oui | Ajout de ligne + recalcul des totaux ; **409** en survente. |
| PUT | `/rental/contracts/{id}/lignes/{ligne_id}` | écriture | Oui | Modifie la ligne (pas l'équipement) ; **409** si la hausse de quantité dépasse le stock. |
| DELETE | `/rental/contracts/{id}/lignes/{ligne_id}` | écriture | Oui | Retrait + recalcul des totaux. |

### 4.5 Points d'accès — Retours et statistiques

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| POST | `/rental/returns` | écriture | Oui | Verrou de ligne, refus du double retour (**409**), propage l'état à l'équipement, auto-`RETOURNE`. |
| GET | `/rental/returns` | lecture | Oui | Liste (filtre optionnel `contrat_id`). |
| GET | `/rental/statistics` | lecture | Oui | `total`, `actifs`, `par_statut`, `montant_ht`, `montant_total`, `equipements_loues`. En cas de panne, renvoie un objet avec `error` (HTTP 200) plutôt qu'un 500. |

### 4.6 Points d'accès — Prêt de main-d'œuvre

| Méthode | Chemin | Rôle | UI | Notes |
|---------|--------|------|----|-------|
| GET | `/rental/employees` | lecture | Oui | Filtres `disponible_only`, `metier`. Jointure avec le module RH pour le nom. |
| PUT | `/rental/employees/{id}/config` | écriture | Oui | Crée ou met à jour la fiche de location de l'employé. |
| GET | `/rental/employees/contracts` | lecture | Oui | Filtres `statut`, `employee_id`. |
| POST | `/rental/employees/contracts` | écriture | Oui | Génère `EMP-#####`. `employee_id`, `date_debut`, `date_fin_prevue` obligatoires. |
| PUT | `/rental/employees/contracts/{id}` | écriture | Oui | Statut + synchronisation du statut de l'employé. |
| POST | `/rental/employees/contracts/{id}/heures` | écriture | Oui | Additionne les heures au compteur du contrat. |
| GET | `/rental/employees/stats` | lecture | Oui | Indicateurs du prêt de main-d'œuvre. En cas de panne, renvoie `error` (HTTP 200). |

### 4.7 Points d'accès — Assistant IA et coûts

Tous en **POST**, tous protégés par `get_current_user` + crédits IA. En mode consultation, ils sont bloqués (403).

| Chemin | Modèle | Sortie |
|--------|--------|--------|
| `/rental/ia/chat` | `claude-sonnet-4-6` | Texte (question libre) |
| `/rental/ia/recommander` | `claude-opus-4-8` | Recommandation d'équipements pour un projet |
| `/rental/ia/analyser-contrat` | `claude-opus-4-8` | Analyse d'un contrat (score, risques, tarification…) |
| `/rental/ia/checklist` | `claude-sonnet-4-6` | Liste de vérification d'inspection |
| `/rental/ia/location-vs-achat` | `claude-sonnet-4-6` | Comparaison location contre achat (seuil de rentabilité…) |

**Chaîne de facturation (identique aux cinq)** :

1. `check_ai_guard(user)` → **neutre en pratique** : renvoie toujours « autorisé » pour un utilisateur authentifié (`ai.py:824-825`). Ne bloque jamais.
2. `_check_credits(user)` → **le vrai gardien** : super-administrateur illimité ; instance interne (`BILLING_ENABLED=false`) illimitée ; sinon lit les crédits prépayés du mois, **recharge automatiquement** au besoin (Stripe), **échoue en bloquant** en cas d'erreur, et renvoie **402 « Crédits IA insuffisants »** si le solde est épuisé.
3. Appel à Claude (déporté hors de la boucle d'événements ; réponse plafonnée à 32000 jetons).
4. Traçage de l'usage + **débit des crédits** — **sans clé d'idempotence** (voir l'avertissement en 1.7).

**Coût** : tarif du modèle **majoré de 30 %** — Sonnet `(entrée × 0,003 + sortie × 0,015) ÷ 1000 × 1,30` ; Opus `(entrée × 0,005 + sortie × 0,025) ÷ 1000 × 1,30`.

> **Anti-injection.** Les champs libres du contrat analysé et le nom d'équipement de « location vs achat » sont **neutralisés** avant d'être envoyés au modèle (les espaces sont compactés, la longueur est bornée à 2000 caractères), et l'invite rappelle de ne jamais interpréter ces données comme des instructions.

### 4.8 Tables PostgreSQL (par tenant, créées à la demande)

| Table | Rôle |
|-------|------|
| `location_items` | Catalogue d'équipements (tarifs, caution, tarif dégressif, disponibilité). |
| `location_contrats` | Contrats de location (`numero_contrat` unique `LOC-#####`, client, statut, totaux HT/TPS/TVQ/TTC, caution). |
| `location_contrat_lignes` | Lignes des contrats (équipement, quantité, tarif, type, remise, dates, états sortie/retour). |
| `location_retours` | Retours par inspection (états avant/après, dommages, frais réparation/nettoyage/retard). |
| `employee_location` | Configuration de location d'un employé (disponibilité, statut, métier — **texte libre**, taux). |
| `location_contrats_employes` | Contrats de prêt (`numero_contrat` unique `EMP-#####`, employé, dates, tarif, heures prévues/réelles, montants). |
| `location_employes_heures` | Saisies quotidiennes d'heures (normales, supplémentaires, validation). |

> **Aucune contrainte de clé étrangère dans la DDL.** Les cascades (supprimer un contrat efface ses lignes et retours ; supprimer une ligne efface ses retours) sont gérées **à la main** dans le code. Les colonnes de disponibilité et de tarif dégressif sont rattrapées automatiquement sur les installations plus anciennes.

### 4.9 Calculs et formules

- **Montant d'une ligne** : `montant = tarif_unitaire × quantité × durée × (1 − remise/100)`. Puis, si l'équipement a un **tarif dégressif actif** et que la durée **en jours** atteint le seuil : `montant × (1 − réduction/100)`.
- **Durée facturée** selon le type de tarif :
  - `JOUR` : nombre de jours entre la sortie et le retour prévu.
  - `SEMAINE` : ce nombre de jours divisé par 7, **arrondi au supérieur**.
  - `MOIS` : divisé par 30, **arrondi au supérieur**.
  - `FORFAIT` : **toujours 1** (aucun multiplicateur — garde-fou pour qu'un forfait ne soit pas multiplié par 30 jours).
  - À défaut de dates, on retombe sur le **nombre de périodes** saisi.
- **Totaux du contrat** : `HT = somme des montants de lignes + somme des frais de retour (réparation + nettoyage + retard)`. Puis `TPS = HT × 0,05`, `TVQ = HT × 0,09975`, `TTC = HT + TPS + TVQ` (arrondis à 2 décimales).

  > **Les frais de retour sont facturables et taxés.** Ils remontent dans le HT, donc dans les taxes et le total — sinon les dommages facturés au client disparaîtraient du montant.

  > **Taux de taxe fixés en dur.** TPS 5 % et TVQ 9,975 % sont codés en dur (la colonne de taux en base est en `numeric(5,2)` en production et arrondirait 9,975 à 9,98). Il n'y a **pas** de taux par contrat ni de gestion d'une juridiction hors Québec (États-Unis, exonéré) : c'est un élément de backlog.

- **Frais de retard automatiques** : si « Frais retard » est laissé vide et que le retour est en retard, le tarif est ramené à un équivalent journalier (diviseur 7 pour la semaine, 30 pour le mois) × quantité × jours de retard ; un `FORFAIT` en est exclu. La valeur **0** saisie explicitement vaut renonciation.
- **Disponibilité** : `quantité disponible = max(0, quantité totale − unités sorties)`, où les **unités sorties** sont la somme des quantités des lignes non rentrées appartenant à des contrats « immobilisants » (`BROUILLON`, `RESERVE`, `EN_COURS`, `ACTIF`, `EN_RETARD`). Un équipement dont la quantité totale est vide n'est pas suivi en quantité (il reste disponible tant qu'il est actif). La disponibilité est **recalculée** à chaque mouvement (ligne, retour, changement de statut, suppression).
- **Montant d'un contrat de prêt** : calculé **seulement** pour un tarif « à l'heure » (`montant_estimé = tarif × heures prévues` ; `montant_facturé = heures réelles × tarif`). Comme le menu de l'écran n'offre pas ce type, ces montants restent vides dans l'usage courant.

### 4.10 Validations, bornes et erreurs

| Règle | Effet |
|-------|-------|
| Nom d'équipement vide | **422** (bouton aussi désactivé à l'écran) |
| Tarif, quantité ou montant négatif | **422** (`ge=0`) |
| Remise ou réduction dégressive hors 0-100 | **422** |
| Année de fabrication hors 1900-2100 | **422** |
| Seuil dégressif hors 1-3650 jours | **422** |
| Date de fin antérieure à la date de début (contrat ou prêt) | **422** |
| Date de retour prévue antérieure à la date de sortie (ligne) | **422** |
| Champ date envoyé vide (`""`) | **422** (converti en absence de valeur) |
| Passage à `EN_COURS`/`FACTURE` sans aucune ligne | **422** |
| Sortie ou hausse de quantité dépassant le stock | **409 « Disponibilité insuffisante »** |
| Double retour sur une même ligne | **409** |
| Suppression d'un équipement engagé dans un contrat actif | **400** |
| Suppression d'un contrat hors `BROUILLON`/`ANNULE` | **400** |
| Assistant IA sans crédits | **402 « Crédits IA insuffisants »** |
| Assistant IA en mode consultation (POST bloqué) | **403** |
| Service IA non configuré | **503** |
| Demande IA trop volumineuse / service surchargé | **413** / **503** |
| Contexte tenant manquant | **400 « Contexte tenant manquant »** |

> **Défense contre l'injection SQL.** Les mises à jour dynamiques (équipement, contrat, ligne, contrat de prêt) passent par des **listes blanches de colonnes** ; aucune valeur de champ ne construit du SQL.

### 4.11 Numéros automatiques

- **Contrats de location** : `LOC-` suivi de 5 chiffres (l'identifiant interne, complété par des zéros) — exemple `LOC-00021`.
- **Contrats de prêt** : `EMP-` suivi de 5 chiffres — exemple `EMP-00007`.
- Génération **atomique** : le serveur insère une valeur temporaire unique puis la remplace par le numéro définitif dans la même transaction (aucune collision, jamais de `COUNT(*)+1`).

### 4.12 Limites de débit et raccourcis

| Élément | Détail |
|---------|--------|
| `POST /rental/ia/*` (les 5 outils IA) | **10 requêtes par minute et par adresse IP** |
| Autres points d'accès `/rental/*` | Borne générale élevée (≈ 1500 par minute) |
| Raccourcis clavier | Aucun raccourci propre au module (pas de zone de conversation multiligne comme dans d'autres modules). Les champs se remplissent et se valident au clic sur les boutons. |

---

## 5. Intégrations et FAQ

### 5.1 CRM, Entreprises et Contacts (Modules 04, 05, 06)

> **Intégration limitée.** Le contrat de location ne conserve qu'un **nom de client en texte libre**. Les colonnes `client_company_id` et `client_contact_id` existent en base mais **ne sont pas saisies** par l'écran. Pour relier un contrat à une entreprise ou à un contact du CRM, il faudrait passer par l'API directement. Conséquence : le top 10 des clients (Statistiques) regroupe par **nom saisi**, sensible aux fautes de frappe.

### 5.2 Projets (Module 08)

Le contrat porte un `project_id` en base, **non saisi** par l'écran. Aucun rattachement automatique à un projet ; supprimer un projet ne touche pas aux contrats de location.

### 5.3 Comptabilité et Factures (Module 14)

> **Aucune facture, aucune écriture comptable.** Le statut `FACTURE` ne fait que **changer le statut** : les montants (HT, TPS, TVQ, TTC) restent dans la table du contrat. Reportez-les manuellement dans le module Comptabilité. La caution n'entre pas dans le total du contrat et ne génère aucune écriture.

### 5.4 Magasin / Inventaire (Module 09)

> **Modules distincts.** Les équipements de location (`location_items`) ne sont **pas** liés aux produits du Magasin. Un même matériel présent dans les deux logiques doit être saisi des deux côtés. Louer un équipement **ne modifie pas** le stock du Magasin.

### 5.5 Employés / RH, Pointage et Paie (Modules 11, 13)

- Le prêt de main-d'œuvre **lit** les employés du module RH (pour le nom et le métier) via la table de configuration de location.
- Les **heures** saisies dans le prêt **ne vont pas** dans la Paie ni le Pointage : c'est une comptabilité parallèle, propre au prêt.
- Pour facturer un prêt, reportez le montant manuellement (le calcul automatique n'existe que pour un tarif « à l'heure » non offert à l'écran, voir 4.9).

### 5.6 Logistique (Module 18) et Maintenance (Module 20) — chevauchement de périmètre

- **Logistique → Équipements** déclare les équipements **possédés ou loués par l'entreprise** pour un usage **interne** (flotte de chantier), avec leur maintenance.
- **Ce module (Location)** gère les **contrats de location proposés à des clients**, avec lignes, taxes, cautions et retours. Tables et logiques **distinctes**.
- **Maintenance (Module 20)** couvre l'entretien des équipements. L'état `REPARATION` d'un équipement de location est un simple indicateur : **aucun lien automatique** vers un bon de maintenance. Suivi manuel (passer l'équipement en `REPARATION`, puis le remettre en état après réparation).

### 5.7 Crédits IA (Module 24)

- L'assistant de location partage le **même portefeuille de crédits IA** que les autres assistants de l'ERP.
- Chaque appel est tracé sous les fonctionnalités `location_chat`, `location_recommander`, `location_analyser_contrat`, `location_checklist` ou `location_compare_achat`, visibles dans le suivi d'usage du super-administrateur.

### 5.8 Foire aux questions

**Puis-je imprimer un contrat de location ou un bon de sortie ?**
Non. Le module n'a **aucune** exportation ni impression : ni PDF, ni CSV, ni bon de retour imprimable. Il ne produit aucun document remis au client.

**Le module facture-t-il automatiquement à la fin d'une location ?**
Non. Le statut `FACTURE` ne change que le statut. Aucune facture n'est créée dans la comptabilité ; le report est manuel.

**Comment savoir si un équipement est libre sur une période future ?**
Le module ne gère pas de calendrier de réservation. Il connaît une **quantité disponible** (total moins unités sorties) et refuse seulement de dépasser le stock physique au moment de la sortie. Pour une période future, vérifiez l'onglet Retours (lignes non rentrées) et gérez le chevauchement manuellement.

**Pourquoi ne puis-je pas changer l'équipement d'une ligne existante ?**
Le champ Équipement est verrouillé en modification. Supprimez la ligne et recréez-la avec le bon équipement.

**Pourquoi la colonne « Montant » d'un contrat de prêt reste-t-elle « -- » ?**
Parce que le calcul automatique n'existe que pour un tarif « à l'heure » que le menu n'offre pas (il propose jour, semaine, mois, forfait). Avec ces types, aucun montant n'est produit ; saisissez-le au besoin par l'API, ou reportez-le en comptabilité.

**Le tarif dégressif fonctionne-t-il vraiment ?**
Oui. C'est une correction par rapport à l'ancien manuel : la case « Tarif dégressif actif » (plus le seuil et la réduction) est exposée à l'écran **et** le rabais est appliqué au calcul dès que la durée en jours atteint le seuil.

**Comment gérer un retard de retour ?**
Passez le contrat à `EN_RETARD` au besoin, puis, au retour, laissez « Frais retard » vide pour un calcul automatique du retard, ou saisissez un montant. Mettez 0 pour renoncer aux frais.

**Que deviennent les frais de dommages saisis à l'inspection ?**
Ils sont **facturables** : ils s'ajoutent au sous-total HT du contrat (donc taxés) et remontent dans le total. Ils ne sont pas perdus.

**Un employé peut-il être sur deux contrats de prêt en même temps ?**
Techniquement oui, mais son statut « En location » est écrasé par le dernier contrat passé à `EN_COURS`. À gérer manuellement.

**Un tenant tout neuf affiche des compteurs à zéro : est-ce normal ?**
Oui. Les 7 tables `location_*` sont créées à la première saisie. Dès votre première fiche ou premier contrat, les listes se remplissent.

**Qui peut créer et supprimer des fiches ?**
Les rôles administrateur, super-administrateur, gestionnaire, contremaître et magasinier (ou tout compte `is_admin`). Les autres rôles obtiennent une erreur 403. La lecture est ouverte à tous.

**L'assistant IA peut-il modifier mes données ?**
Non. Il **conseille** seulement (recommandations, comparaisons, listes de vérification, analyse). Il n'écrit rien. Validez ses résultats — surtout les coûts et la sécurité — avec un professionnel.

**Pourquoi l'assistant IA renvoie-t-il 403 alors que j'ai des crédits ?**
Probablement le mode consultation : comme ses appels sont des écritures (POST), ils sont bloqués quand l'abonnement Stripe est inactif. Régularisez l'abonnement.

**Combien coûte un appel à l'assistant ?**
Le tarif du modèle (Sonnet ou Opus selon l'outil) majoré de 30 %, débité de vos crédits IA. Le coût exact s'affiche sous chaque réponse. Le registre de location, lui, est gratuit.

---

## 6. Récapitulatif

- **Objet** : registre opérationnel interne à deux volets — **location d'équipements** à des clients (catalogue, contrats taxés, retours par inspection) et **prêt de main-d'œuvre** (employés prêtés) — plus un **assistant IA** de conseil.
- **Accès** : barre latérale → groupe **TERRAIN** → **Location** (icône `HardHat`), route `/location`, titre « Location d'équipement ». Onglet par défaut : Tableau de bord.
- **7 onglets** : Tableau de bord, Catalogue, **Locations** (les contrats), Retours, **Main-d'œuvre** (le prêt, 4 sous-onglets), Statistiques, Assistant IA.
- **Un seul router côté serveur** : tout vit dans `secondary.py`, sous le préfixe **`/rental/*`** (27 points d'accès : 22 de gestion + statistiques, 5 pour l'IA). **Aucun** fichier `location.py` ni `location_ai.py`. La route `/location` n'existe qu'à l'écran.
- **7 tables** `location_*` par tenant, **créées à la demande** (un tenant neuf voit des compteurs à zéro).
- **Permissions** : lecture ouverte à tous ; écriture réservée aux rôles administrateur / super-administrateur / gestionnaire / contremaître / magasinier (ou `is_admin`) ; mode consultation (lecture seule, y compris l'assistant IA) si l'abonnement Stripe est inactif.
- **Numéros** : `LOC-#####` (contrats de location), `EMP-#####` (contrats de prêt), générés de façon atomique et uniques.
- **Calculs** : `montant de ligne = tarif × quantité × durée × (1 − remise)`, puis tarif dégressif si actif ; `FORFAIT` = durée 1 ; totaux `HT + TPS 5 % + TVQ 9,975 %` (taux fixés en dur) ; **les frais de retour sont facturables et taxés**.
- **Disponibilité** : quantité totale moins unités sorties, recalculée à chaque mouvement ; **garde anti-survente (409)** à la sortie et à la hausse de quantité.
- **Retours** : par inspection, refus du double retour (409), propagation de l'état à l'équipement, passage automatique à `RETOURNE` quand tout est rentré ; frais de retard calculables automatiquement.
- **Prêt de main-d'œuvre** : configuration depuis le module RH, contrats avec synchronisation du statut de l'employé, saisie d'heures ; **aucun montant automatique** avec les types de tarif offerts, et **heures non propagées** en paie.
- **Assistant IA** : 5 outils, Sonnet (chat / checklist / location-vs-achat) et Opus (recommander / analyser-contrat), tarif du modèle × 1,30, vrai gardien = solde de crédits (402), `check_ai_guard` neutre. **Débit sans idempotence** : ne cliquez qu'une fois.
- **Ce que le module ne fait pas** : aucune impression / PDF / CSV / téléversement / photo ; aucune facture ni écriture comptable ; client en texte libre seulement (pas de lien CRM/projet à l'écran) ; pas de calendrier de réservation ; pas de notifications ; heures de prêt hors paie ; état `REPARATION` sans lien maintenance automatique.

---

**Documentation générée à partir du code** : `ERP_REACT/backend/routers/secondary.py` (section Location lignes 3961-6180 : 27 points d'accès `/rental/*`, modèles Pydantic 498-737, DDL des 7 tables 1274-1435, invite système IA 1608-1651) ; `ERP_REACT/backend/erp_auth.py` (gardes `require_tenant_admin_or_role`, mode consultation) ; `ERP_REACT/backend/routers/ai.py` (crédits IA, `check_ai_guard` neutre) ; `ERP_REACT/frontend/src/pages/LocationPage.tsx` (2268 lignes, 7 onglets, un seul fichier) ; `ERP_REACT/frontend/src/api/location.ts` ; `store/useLocationStore.ts` ; textes `i18n/locales/fr/terrain.json` (`terrain.location.*`).

**Manuels liés** :
- Module 03 (Entreprises / Contacts — clients de contrats, liens non câblés à l'écran) — `03-gestion-entreprises.md`
- Module 05 (CRM — opportunités) — `05-gestion-crm-opportunites.md`
- Module 08 (Projets — référence `project_id` non câblée) — `08-ventes-projets.md`
- Module 09 (Magasin / Inventaire — distinct, pas de lien de stock) — `09-operations-magasin.md`
- Module 10 (Employés / RH — source des employés prêtés) — `10-operations-employes.md`
- Module 12 (Pointage — heures de prêt non propagées) — `12-operations-pointage.md`
- Module 14 (Comptabilité / Factures — report manuel, pas de facture auto) — `14-operations-comptabilite.md`
- Module 18 (Logistique — équipements internes, chevauchement de périmètre) — `18-terrain-logistique.md`
- Module 20 (Maintenance — état `REPARATION` suivi manuel) — `20-terrain-maintenance.md`
- Module 24 (Assistant IA — portefeuille de crédits partagé) — `24-communication-assistant-ia.md`
