# Module 16 — Conformité RBQ / CCQ

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — correction du nombre de catégories RBQ, des libellés d'onglets réels, du modèle IA, des permissions et des barèmes de score)
> **Libellé dans le menu** : « RBQ/CCQ » (clé `nav.rbqCcq`), groupe « TERRAIN » de la barre latérale, icône `Shield` — route `/conformite`
> **Code de référence (backend)** : `ERP_REACT/backend/routers/conformite.py` (2531 lignes, 31 points d'accès dont 7 outils IA) ; `ERP_REACT/backend/routers/conformite_data.py` (398 lignes, **module de données statiques pur — ce n'est PAS un router monté**, aucun point d'accès)
> **Code de référence (frontend)** : `ERP_REACT/frontend/src/pages/ConformitePage.tsx` (3164 lignes, 6 onglets) ; `ERP_REACT/frontend/src/api/conformite.ts` (587 lignes) ; `ERP_REACT/frontend/src/store/useConformiteStore.ts` (740 lignes, magasin Zustand)
> **Chemin d'API réel** : `/api/erp/v1/conformite` (préfixe `/conformite` monté avec `API_PREFIX = /api/erp/v1`)
> **Tables PostgreSQL (une par tenant, créées à la demande)** : `conformite_licences_rbq`, `conformite_cartes_ccq`, `conformite_attestations` (colonne `fichier_data` BYTEA pour les pièces jointes)
> **Modèle IA** : Claude Opus 4.8 (`claude-opus-4-8`), 32 000 jetons maximum par appel, facturé aux crédits IA prépayés du tenant
> **Cadrage** : registre documentaire **manuel** de la conformité réglementaire québécoise en construction — licences RBQ de l'entreprise, cartes de compétence CCQ des employés, attestations fiscales et sectorielles (avec pièce jointe), un tableau de bord (score, indicateurs, alertes d'expiration) et un assistant IA spécialisé. Le module **ne se connecte à aucun registre officiel** : toute la saisie est faite à la main.

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

Centraliser, dans un seul écran, la **gestion documentaire réglementaire** d'une entreprise de construction au Québec, afin d'éviter les ruptures de conformité (licence expirée, carte de compétence échue, attestation fiscale périmée) qui empêchent de facturer, de soumissionner ou de démarrer un chantier.

Le module répond à quatre besoins concrets :

- « Quelles licences RBQ mon entreprise détient-elle, et lesquelles arrivent à échéance ? »
- « Quels employés ont une carte de compétence CCQ valide, et pour quel métier ? »
- « Mes attestations fiscales (Revenu Québec, ARC, CNESST) sont-elles à jour, et où est le PDF ? »
- « Suis-je globalement conforme ? » (score, alertes et diagnostics IA)

### 1.2 Ce que le module gère (contexte légal québécois)

- **Licences RBQ (Régie du bâtiment du Québec)** : la RBQ délivre aux **entreprises** entrepreneures les licences obligatoires prévues par la Loi sur le bâtiment (chapitre B-1.1). Le module enregistre le numéro de licence, les sous-catégories couvertes (**27 codes officiels du 1.1 au 16**), les dates d'émission et d'expiration, le cautionnement, l'assurance responsabilité civile et le statut.
- **Cartes de compétence CCQ (Commission de la construction du Québec)** : la CCQ gère les cartes des **travailleurs individuels** sous le régime de la Loi R-20. Le module associe une carte à un employé, avec le métier principal (**28 métiers**), une qualification (Compagnon, Apprenti par période, ou Classe), les heures accumulées et la formation ASP Construction.
- **Attestations (5 types)** : documents à durée limitée exigés pour soumissionner ou démarrer — Revenu Québec, Agence du revenu du Canada (ARC), CNESST, CCQ (état de situation) et attestation de solvabilité RBQ. Chaque attestation peut recevoir **une** pièce jointe PDF ou image stockée en base.
- **Tableau de bord** : score de conformité de 0 à 100, indicateurs clés, alertes d'expiration et répartitions par catégorie, métier et type.
- **Assistant IA** : sept outils propulsés par Claude Opus 4.8, spécialisés en réglementation RBQ/CCQ du Québec.

### 1.3 Ce que le module fait (vérifié contre le code)

- Tenir un **registre CRUD** (créer, lire, modifier, supprimer) des licences RBQ, des cartes CCQ et des attestations, chacun avec ses filtres (statut, catégorie/métier/type, recherche texte).
- Attacher **un** fichier (PDF, JPG, PNG ou WebP, jusqu'à 10 Mo) à chaque attestation, puis le **télécharger** au besoin.
- Calculer un **score de conformité** et afficher un **badge coloré** en tête de page (vert ≥ 80 %, jaune ≥ 50 %, rouge < 50 %).
- Produire des **alertes d'expiration** consultables dans le tableau de bord (licences et cartes à 60 jours, attestations à 30 jours par leur point d'accès dédié).
- Offrir un **assistant IA** à sept outils : analyse de conformité, chat réglementaire, vérification des exigences d'un projet, recherche de réglementation, prédiction des renouvellements, génération d'un rapport et recommandation de formations.
- Fournir un onglet de **vérification des exigences réglementaires d'un projet** (licences requises, métiers CCQ, permis, attestations, cautionnement et assurance minimums, ratio compagnon/apprenti).

### 1.4 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes. Ce module est un registre de saisie, pas un connecteur officiel.

- **Aucune vérification en temps réel** contre les registres officiels de la RBQ ou de la CCQ. Tout est saisi à la main. Le nom de l'entreprise inscrit sur une licence est un **champ texte libre**, il n'est pas tiré du profil du tenant.
- **Aucune déclaration mensuelle d'heures CCQ.** Le champ « heures accumulées » est un cumul manuel, pas un journal mensuel. Utilisez le portail employeur de la CCQ pour les déclarations officielles.
- **Aucune exportation ni impression** des données de conformité : pas de PDF, pas de CSV, pas de bouton d'impression. Le **seul** téléchargement possible est la pièce jointe d'une attestation. Même le **rapport IA est affiché à l'écran seulement** (aucun export, aucune sauvegarde).
- **Aucun import en masse** : pas de téléversement CSV de licences ou de cartes ; chaque enregistrement se saisit un par un.
- **Résultats IA éphémères.** La vérification de projet, l'analyse, le rapport, la recherche et les prédictions ne sont **pas sauvegardés** : ils disparaissent au rechargement de la page ou au changement d'onglet.
- **Alertes uniquement dans l'application.** Aucun rappel par courriel, par notification ou par calendrier (iCal, Google Agenda). Les alertes vivent dans le tableau de bord.
- **Pièce jointe unique par attestation** : pas de multi-fichiers, pas de gestion de versions.
- **Pas de paie ni de cotisations CCQ** : les heures servent de repère de renouvellement, pas de base de calcul salarial (voir Module 12 Pointage et Module 10 Employés).
- **Pas de signature électronique** ni de flux d'approbation interne.
- **Onglets « fantômes » de l'ancien héritage Streamlit** : certaines clés d'affichage (CSST, formations de saisie, colonnes « responsable » d'audit) existent dans les fichiers de traduction mais **ne sont rendues nulle part** dans l'interface React. La « formation » n'existe ici que comme **recommandation IA** ; « Audits et inspections » est en réalité l'onglet de **vérification IA d'un projet** (voir la mise en garde sur les libellés en 2.1).

### 1.5 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **RBQ/CCQ** (icône `Shield`). Réf. `Sidebar.tsx:72`.
- URL directe : `/conformite`.
- Fil d'Ariane et titre de la barre supérieure : « RBQ/CCQ ».
- Titre affiché en haut de la page : « **Conformité RBQ / CCQ** », sous-titre « **Suivi des licences, formations et obligations légales** ».
- **Onglet ouvert par défaut** : **Licences RBQ** (état initial du composant, `ConformitePage.tsx:158`), même si l'onglet « Tableau de bord » apparaît en premier dans la barre d'onglets.

### 1.6 Permissions et rôles

Les droits ne sont **pas** les mêmes pour tout le monde — c'est une correction importante par rapport aux versions antérieures de ce manuel.

| Action | Qui peut la faire | Garde technique |
|--------|-------------------|-----------------|
| **Consulter** licences, cartes, attestations, statistiques, alertes | Tout utilisateur connecté au tenant | `get_current_user` |
| Télécharger la pièce jointe d'une attestation | Tout utilisateur connecté au tenant | `get_current_user` |
| **Créer / modifier / supprimer une licence RBQ** | Administrateur du tenant | `require_tenant_admin_or_role()` |
| **Créer / modifier / supprimer une carte CCQ** | Administrateur du tenant | `require_tenant_admin_or_role()` |
| **Créer / modifier / supprimer / téléverser une attestation** | Administrateur **ou** rôle « comptable » | `require_tenant_admin_or_role("comptable")` |
| **Utiliser les 7 outils IA** | Tout utilisateur connecté, **si le tenant a des crédits IA** | crédits prépayés (voir ci-dessous) |

- Le statut « administrateur » est **relu côté serveur** (`is_admin`) à chaque requête : il ne peut pas être falsifié depuis le navigateur.
- Les **outils IA consomment des crédits IA prépayés** du tenant. Le serveur vérifie d'abord la disponibilité du service (sinon erreur 503), puis un garde de facturation, puis le solde de crédits : un solde épuisé renvoie une erreur **402** affichée dans la bannière rouge. Le super-administrateur et les sociétés exemptées ne sont pas bloqués par ce contrôle.
- Toutes les données sont **strictement cloisonnées par tenant** (schéma PostgreSQL propre à l'entreprise). Aucun accès entre tenants n'est possible.
- Il n'existe **pas** de rôle dédié « responsable conformité ». Bonne pratique interne : désigner une seule personne responsable de la saisie pour éviter les modifications concurrentes.

### 1.7 Les six onglets du module

| # | Onglet (libellé affiché) | Contenu réel | Icône |
|---|--------------------------|--------------|-------|
| 1 | **Tableau de bord** | Score, indicateurs, alertes, répartitions, ressources | `BarChart3` |
| 2 | **Licences RBQ** (N) | Registre CRUD des licences RBQ | `Shield` |
| 3 | **Cartes CCQ** (N) | Registre CRUD des cartes de compétence par employé | `UserCheck` |
| 4 | **Documents légaux** (N) | **En réalité : les attestations fiscales et sectorielles** | `FileText` |
| 5 | **Audits et inspections** | **En réalité : la vérification IA des exigences d'un projet** | `CheckCircle2` |
| 6 | **Assistant IA** | Six outils IA (analyse, chat, recherche, prédiction, rapport, formations) | `Sparkles` |

Le nombre entre parenthèses est un **compteur en direct** (nombre de licences, de cartes, d'attestations). En affichage mobile, les libellés sont abrégés : Dash / RBQ / CCQ / Attest. / Verif. / IA.

> **Mise en garde sur deux libellés trompeurs.** L'onglet **« Documents légaux »** contient exactement les **attestations** (Revenu Québec, ARC, CNESST, CCQ, RBQ). L'onglet **« Audits et inspections »** ne gère aucun audit : c'est l'outil de **vérification IA d'un projet**. Ces deux libellés sont des héritages de l'ancienne version Streamlit et ne décrivent pas fidèlement le contenu. Le présent manuel utilise les libellés affichés, mais précise chaque fois le contenu réel.

---

## 2. Interface

### 2.0 Éléments communs à tous les onglets

- **Badge de score global** : à droite du titre de page, un badge coloré affiche le pourcentage de conformité (vert ≥ 80 %, jaune ≥ 50 %, rouge < 50 %). Il est calculé par le point d'accès `GET /conformite/statistics`.
- **Bannières** : une bannière rouge affiche les erreurs ; une bannière verte confirme les succès (elle se masque seule après environ 4 secondes).
- **Écran d'erreur de démarrage** : si les données de référence (catégories, métiers, etc.) n'arrivent pas à charger, la page affiche « Impossible de charger le module Conformité » avec un bouton **« Réessayer »**.
- **Barre de commandes** : chaque onglet de registre (Licences, Cartes, Documents légaux) possède une barre avec un bouton primaire de création, un champ de recherche (avec un délai de 400 ms avant l'envoi) et des menus déroulants de filtre.
- **Suppression** : chaque suppression demande une confirmation par une petite fenêtre native du navigateur (« Supprimer cette licence ? », etc.).

### 2.1 Onglet « Tableau de bord »

Écran de synthèse. Il recharge ses statistiques et ses alertes à chaque affichage.

**Quatre indicateurs clés (cartes du haut)**

- **Licences RBQ actives** (avec le total en sous-titre : « N total »)
- **Cartes CCQ actives**
- **Attestations valides**
- **Score de conformité** (pourcentage)

**Trois indicateurs d'alerte**

- **À renouveler (60 jours)** : somme des licences, cartes et attestations qui expirent dans les 60 prochains jours.
- **Expirés** : somme des éléments déjà expirés.
- **Cautionnement total** : somme des cautionnements enregistrés sur les licences, en dollars.

**Liste « Alertes de conformité »**

Jusqu'à 30 alertes, chacune avec un badge de priorité, un message en clair et une date. Si tout est à jour : « Aucune alerte - Tous les documents sont à jour ». Les alertes proviennent du point d'accès `GET /conformite/alertes` (voir la section 4.9 pour les fenêtres exactes).

**Trois répartitions**

- Répartition **par catégorie** (licences RBQ)
- Répartition **par métier** (cartes CCQ)
- Répartition **par type** (attestations)

Chaque répartition affiche un décompte par valeur (jusqu'à 10 lignes pour les licences et les cartes).

**Ressources** (si elles sont chargées)

- **Organismes de référence** : 8 organismes officiels avec leur rôle et un contact (RBQ, CCQ, CNESST, Revenu Québec, ARC, ASP Construction, Ombudsman de la construction, CMEQ).
- **Conseils pratiques** : 6 sections de trucs (surveiller les échéances, maintenir la conformité financière, gérer les cartes, préparer les démarrages, prévenir les sanctions, développer les compétences).

### 2.2 Onglet « Licences RBQ »

**Barre de commandes** : bouton **« Nouvelle licence »**, champ de recherche, filtre **Statut** (« Tous les statuts » + les statuts) et filtre **Catégorie** (« Toutes les catégories »).

**Tableau (ordinateur)** — colonnes :

| Colonne | Contenu |
|---------|---------|
| **Numéro** | Numéro de licence (police à chasse fixe) |
| **Entreprise** | Nom de l'entreprise titulaire (texte libre) |
| **Catégories** | Badges bleus (3 au maximum, puis « +N ») |
| **Cautionnement** | Montant formaté « x xxx $ » ou « -- » |
| **Expiration** | Date d'expiration |
| **Statut** | Badge coloré (voir la règle ci-dessous) |
| **Actions** | Icônes Modifier et Supprimer |

**Cartes (mobile)** : numéro, badge de statut, nom de l'entreprise, badges de catégories (4 au maximum), « Exp: {date} », liens Modifier et Supprimer.

**État vide** : icône bouclier + « Aucune licence RBQ enregistrée » + bouton « Ajouter une licence ». Si des filtres sont actifs et ne renvoient rien : « Aucun résultat pour ces filtres » + « Réinitialiser les filtres ».

**Règle de couleur du badge de statut** : rouge si la licence est expirée (ou statut EXPIREE / REVOQUEE) ; jaune si elle expire dans 60 jours ou moins, ou si elle est SUSPENDUE ou EN_RENOUVELLEMENT ; vert si elle est ACTIVE.

**Modale « Nouvelle licence RBQ » / « Modifier la licence RBQ »** — champs :

- **Numéro de licence** (obligatoire, exemple « 5734-1234-01 », 100 caractères maximum, **unique** dans le tenant)
- **Nom de l'entreprise** (obligatoire, 255 caractères maximum, saisie manuelle)
- **Catégories RBQ** (« {n} sélectionnées ») : liste de cases à cocher défilante affichant, pour chaque catégorie, « code — libellé (sous-groupe) »
- **Date d'émission** / **Date d'expiration** (sélecteurs de date)
- **Statut** (menu déroulant : ACTIVE, SUSPENDUE, EXPIREE, REVOQUEE)
- **Cautionnement ($)** / **Assurance responsabilité ($)** (nombres, pas de 1000)
- **Notes** (zone de texte, 5000 caractères maximum)

**Validations** : le numéro et le nom sont obligatoires ; la date d'émission doit être antérieure ou égale à la date d'expiration ; un numéro déjà utilisé est refusé (erreur 409). Boutons « Annuler » et « Créer » (ou « Enregistrer » en modification).

### 2.3 Onglet « Cartes CCQ »

**Barre de commandes** : bouton **« Nouvelle carte »**, recherche, filtre **Statut** et filtre **Métier** (« Tous les métiers »).

**Tableau (ordinateur)** — colonnes :

| Colonne | Contenu |
|---------|---------|
| **Employé** | Nom de l'employé (ou « #id » si la fiche est absente) |
| **Numéro** | Numéro de carte (police à chasse fixe) |
| **Métier** | Métier principal |
| **Qualification** | Compagnon / Apprenti par période / Classe |
| **Heures** | Heures accumulées (« x xxx h ») |
| **ASP** | Badge vert « ASP » ou « -- » |
| **Renouvellement** | Date de renouvellement |
| **Statut** | Badge coloré |
| **Actions** | Modifier et Supprimer |

**État vide** : « Aucune carte CCQ enregistrée » + « Ajouter une carte ».

**Modale « Nouvelle carte CCQ » / « Modifier la carte CCQ »** — champs :

- **Employé** :
  - En **création** : un menu déroulant liste les employés **actifs** (jusqu'à 100). Si la liste est vide ou indisponible, l'interface bascule sur une **saisie manuelle de l'identifiant** de l'employé.
  - En **modification** : l'employé est **verrouillé** (lecture seule) — on ne peut pas réaffecter une carte à un autre employé.
- **Numéro de carte** (obligatoire, exemple « CCQ-12345 », 100 caractères maximum, **unique**)
- **Métier** (menu déroulant, 28 métiers ; défaut « Apprenti »)
- **Compétence** (menu déroulant dont les options **changent selon le métier** ; au changement de métier, la première qualification est sélectionnée automatiquement)
- **Catégorie de compétence** (« {n} ») : cases à cocher des **métiers additionnels** (grille à deux colonnes ; le métier principal est exclu)
- **Heures accumulées** (nombre, pas de 100)
- **Date d'émission** / **Date d'expiration** (cette dernière sert de date de renouvellement)
- **ASP Construction** (case à cocher)
- **Statut** (menu déroulant : ACTIVE, SUSPENDUE, EXPIREE)
- **Notes** (zone de texte)

**Validations** : le numéro et le métier sont obligatoires ; l'identifiant de l'employé doit être supérieur à 0 ; si la table des employés existe, l'identifiant doit correspondre à un employé réel (sinon erreur 404 « Employé introuvable »).

**Qualifications dynamiques par métier**

| Métier | Qualifications proposées |
|--------|--------------------------|
| Apprenti | 1re période, 2e période, 3e période, 4e période |
| Grutier | Classe 1, Classe 2, Classe 3, Classe 4 |
| Opérateur d'équipement lourd | Classe 1, Classe 2, Classe 3, Classe 4 |
| Soudeur | Classe A, Classe B, Classe C |
| Soudeur en tuyauterie | Classe A, Classe B |
| **Tous les autres métiers (23)** | Compagnon |

### 2.4 Onglet « Documents légaux » (les attestations)

> Rappel : malgré son libellé, cet onglet gère uniquement les **attestations** fiscales et sectorielles.

**Barre de commandes** : bouton **« Nouvelle attestation »**, recherche (sur le type, l'organisme, le numéro et les notes), filtre **Statut** et filtre **Type**.

**Tableau** — colonnes :

| Colonne | Contenu |
|---------|---------|
| **Type** | Type d'attestation |
| **Numéro** | Numéro (police à chasse fixe) |
| **Organisme** | Organisme délivreur |
| **Expiration** | Date d'expiration |
| **Statut** | Badge (VALIDE, EN_RENOUVELLEMENT, EXPIREE) |
| **Fichier** | Si une pièce jointe existe : bouton **Télécharger** (avec la taille en Ko) ; sinon : bouton **Téléverser** |
| **Actions** | Modifier et Supprimer |

**État vide** : « Aucune attestation enregistrée » + « Ajouter une attestation ».

**Cinq types officiels**

| Code | Libellé | Organisme délivreur |
|------|---------|---------------------|
| `REVENU_QUEBEC` | Attestation de Revenu Québec | Revenu Québec |
| `ARC` | Attestation de l'Agence du revenu du Canada | Agence du revenu du Canada |
| `CNESST` | Attestation de conformité CNESST | CNESST |
| `CCQ` | Attestation CCQ - État de situation | Commission de la construction du Québec |
| `RBQ` | Attestation de solvabilité RBQ | Régie du bâtiment du Québec |

**Modale « Nouvelle attestation » / « Modifier l'attestation »** — champs : **Type** (obligatoire, menu déroulant des 5 types) · **Numéro d'attestation** (obligatoire, 100 caractères maximum) · **Date d'émission** / **Date d'expiration** · **Statut** (menu déroulant) · **Notes**. Le couple (type, numéro) est **unique** : un doublon est refusé (erreur 409).

**Modale « Téléverser un document »**

- Note affichée : « **Types acceptés: PDF, JPG, PNG, WebP. Taille maximum: 10 Mo.** »
- Un seul fichier accepté. Le bouton « Téléverser » reste désactivé tant qu'aucun fichier n'est choisi.
- Le fichier est validé côté serveur : type MIME dans la liste autorisée (sinon erreur 415), taille au plus 10 Mo (sinon erreur 413), fichier non vide, et **vérification des octets d'en-tête** (le vrai format doit correspondre à l'extension). Le nom du fichier est assaini.
- Après le téléversement, la ligne affiche désormais le bouton **Télécharger** (avec la taille).

### 2.5 Onglet « Audits et inspections » (vérification IA d'un projet)

> Rappel : cet onglet n'enregistre aucun audit. C'est un **outil IA** qui, à partir des paramètres d'un projet, énumère les exigences réglementaires probables. Le résultat est **éphémère** (il n'est pas sauvegardé).

**Formulaire « Vérification d'exigences réglementaires (IA) »**

- **Type de projet** (menu déroulant, 7 valeurs : Résidentiel unifamilial, Résidentiel multifamilial, Commercial, Industriel, Institutionnel, Rénovation majeure, Agrandissement)
- **Valeur estimée ($)** (nombre, pas de 10 000, défaut 100 000)
- **Région** (menu déroulant, 18 valeurs : 17 régions administratives + « Autre région »)
- **Types de travaux** (« {n} », obligatoire) : cases à cocher parmi 12 options (Fondation, Charpente, Électricité, Plomberie, Chauffage/Ventilation, Toiture, Revêtement extérieur, Finition intérieure, Maçonnerie, Structure métallique, Excavation, Piscine)

Bouton **« Vérifier les exigences »** (désactivé si aucun type de travaux n'est coché ; un indicateur d'activité tourne pendant l'appel).

**Panneau de résultat** — l'IA renvoie :

- **Licences RBQ requises** (badges « Obligatoire » / « Recommandée »)
- **Métiers CCQ requis** (nombre estimé + qualification)
- **Permis requis**
- **Attestations requises**
- **Cautionnement minimum** ($)
- **Assurance responsabilité minimum** ($)
- **Ratio compagnon/apprenti**
- **Délai estimé** de mise en conformité
- **Alertes** (non-conformités potentielles à surveiller)

> Ce diagnostic est **indicatif**. Le prompt système interdit à l'IA d'inventer des numéros de loi (« Préfère indiquer "à vérifier" plutôt que fabriquer une référence »). Validez toujours les cas complexes auprès de la RBQ, de la CCQ ou d'un conseiller spécialisé.

### 2.6 Onglet « Assistant IA »

Sous-navigation en pilules ; chaque outil rappelle : « **Cette action consomme des crédits IA.** » Un verrou interne empêche de lancer deux appels IA en même temps.

Six sous-outils :

1. **« Analyser ma conformité »** — bouton « Analyser » → score, niveau de risque, résumé, points conformes, non-conformités (avec badge de gravité), risques, renouvellements urgents, recommandations et estimation des coûts de mise en conformité. Point d'accès `POST /conformite/ai/analyze`.
2. **« Chat réglementaire »** — fil de conversation, zone de saisie (Entrée pour envoyer, Maj+Entrée pour un saut de ligne, 5000 caractères maximum), case **« Inclure le contexte de mon dossier »**, boutons « Envoyer » et « Effacer ». Point d'accès `POST /conformite/ai/chat`.
3. **« Rechercher une réglementation »** — champ de recherche (500 caractères maximum) → interprétation, réponse directe, résultats (titre, source, référence, résumé, lien officiel), points importants, mises en garde et ressources. Point d'accès `POST /conformite/ai/search-regulations`. Les liens renvoyés sont assainis côté serveur (seuls `http://` et `https://` sont conservés).
4. **« Prédire mes renouvellements »** — bouton → coût annuel estimé, budget mensuel, renouvellements urgents, calendrier sur 12 mois (coût, éléments, actions), risques d'expiration et recommandations. Point d'accès `POST /conformite/ai/predict-renewals`.
5. **« Générer un rapport »** — bouton → rapport professionnel (titre, dates, score global, évaluation, résumé exécutif, blocs Conformité RBQ et CCQ, attestations, risques, plan d'action, conclusion, prochaine révision). Point d'accès `POST /conformite/ai/generate-rapport`. **Le rapport est affiché à l'écran seulement : aucun export ni téléchargement.**
6. **« Recommander des formations »** — saisie facultative de « Projets prévus » sous forme de puces ajoutables → analyse des compétences (forces, lacunes, occasions), formations recommandées (organisme, durée, coût, public, priorité, bénéfices), certifications suggérées, plan de développement, budget annuel et rendement estimé. Point d'accès `POST /conformite/ai/recommend-formations`.

> Le septième outil IA du module, la **vérification de projet** (`POST /conformite/ai/verify-project`), n'est pas dans cet onglet : il est utilisé par l'onglet « Audits et inspections » (section 2.5).

---

## 3. Workflows pas à pas

### 3.1 Enregistrer une licence RBQ existante

1. Onglet **Licences RBQ** → bouton **« Nouvelle licence »**.
2. Saisir le **numéro de licence** officiel (format RBQ, par exemple « 5734-1234-01 »).
3. Saisir le **nom de l'entreprise** titulaire (il peut différer du nom commercial du tenant s'il s'agit d'une filiale).
4. Cocher toutes les **sous-catégories** couvertes par la licence (par exemple 1.1 + 15.5 + 15.6 pour résidentiel + ventilation + climatisation).
5. Renseigner la **date d'émission**, la **date d'expiration**, le **statut** (ACTIVE), le **cautionnement** et l'**assurance responsabilité**.
6. Cliquer **« Créer »**. Un message de succès confirme, et la liste comme les statistiques se rafraîchissent.

*Rappel de permission : réservé à l'administrateur du tenant.*

### 3.2 Renouveler une licence avant l'expiration

1. Dans le **Tableau de bord**, repérer les alertes « expire bientôt » (fenêtre de 60 jours).
2. À la réception du certificat de renouvellement, ouvrir la licence concernée (icône Modifier).
3. Mettre à jour la **date d'expiration** et remettre le **statut** à ACTIVE si l'ancien était EN_RENOUVELLEMENT ou EXPIREE.
4. Ajuster le **cautionnement** si la RBQ a modifié le seuil exigé.
5. **Enregistrer**. Le score de conformité remonte automatiquement.

### 3.3 Suspendre ou révoquer une licence

1. Ouvrir la licence concernée.
2. Changer le **statut** : SUSPENDUE (temporaire, badge jaune) ou REVOQUEE (définitif, badge rouge foncé).
3. Documenter le motif et le numéro de dossier RBQ dans les **Notes**.
4. **Enregistrer**. Attention : une suspension retire 6 points de score, une révocation en retire 15 (voir 4.8).

### 3.4 Créer la carte CCQ d'un nouvel employé

1. Au préalable, créer la fiche de l'employé dans le **Module 10 (Employés)** : la carte se rattache à un employé existant.
2. Onglet **Cartes CCQ** → bouton **« Nouvelle carte »**.
3. Choisir l'**employé** dans le menu déroulant (employés actifs). Si la liste est vide, saisir l'identifiant manuellement.
4. Saisir le **numéro de carte** officiel.
5. Choisir le **métier principal** : la liste de compétences se met à jour automatiquement.
6. Choisir la **compétence** (Compagnon par défaut, sinon une Classe ou une période d'apprenti).
7. Cocher au besoin des **métiers additionnels** (le métier principal est déjà exclu).
8. Renseigner les **heures accumulées**, la **date d'émission**, la **date d'expiration** et cocher **ASP Construction** si la formation est valide.
9. Cliquer **« Créer »**.

> **Rappel** : l'employé n'est **pas modifiable** après la création de la carte. Pour réaffecter une carte, il faut la supprimer et en créer une nouvelle.

### 3.5 Mettre à jour les heures CCQ d'un travailleur

1. Récupérer le cumul d'heures depuis le portail employeur de la CCQ ou depuis la paie.
2. Onglet **Cartes CCQ** → ouvrir la carte (icône Modifier).
3. Mettre à jour les **heures accumulées**.
4. Si le seuil de passage est atteint, changer la **compétence** (par exemple « 4e période » → « Compagnon » en changeant le métier de « Apprenti » au métier réel).
5. **Enregistrer**.

### 3.6 Renouveler une carte CCQ

1. Repérer l'alerte « à renouveler » dans le tableau de bord.
2. Ouvrir la carte concernée.
3. Mettre à jour la **date d'expiration (renouvellement)** et remettre le **statut** à ACTIVE au besoin.
4. **Enregistrer**.

### 3.7 Créer une attestation (sans pièce jointe immédiate)

1. Onglet **Documents légaux** → bouton **« Nouvelle attestation »**.
2. Choisir le **type** (Revenu Québec, ARC, CNESST, CCQ ou RBQ).
3. Saisir le **numéro** d'attestation.
4. Renseigner la **date d'émission** et la **date d'expiration**.
5. Laisser le statut à VALIDE.
6. Cliquer **« Créer »**.

*Rappel de permission : réservé à l'administrateur ou au rôle « comptable ».*

### 3.8 Téléverser le fichier d'une attestation

1. Onglet **Documents légaux** → sur la ligne sans fichier, cliquer **« Téléverser »**.
2. Choisir un fichier **PDF, JPG, PNG ou WebP** de 10 Mo au maximum.
3. Cliquer **« Téléverser »**. Le serveur valide le type, la taille et le format réel, puis stocke le fichier.
4. La ligne affiche désormais **« Télécharger »** avec la taille.

> **Bonne pratique** : téléverser le PDF original reçu par courriel (et non une photo) — il contient la signature numérique exigée pour les appels d'offres publics.

### 3.9 Télécharger une pièce jointe

1. Onglet **Documents légaux** → bouton **« Télécharger »** (icône de téléchargement).
2. Le fichier est servi en téléchargement forcé, avec un nom assaini et un type MIME revalidé (servi en flux binaire générique si le type sort de la liste autorisée).

### 3.10 Vérifier les exigences réglementaires d'un projet (IA)

1. Onglet **Audits et inspections**.
2. Renseigner le **type de projet**, la **valeur estimée**, la **région** et cocher au moins un **type de travaux**.
3. Cliquer **« Vérifier les exigences »** (l'appel consomme des crédits IA).
4. Lire le panneau de résultat : licences requises (obligatoires ou recommandées), métiers CCQ, permis, attestations, cautionnement et assurance minimums, ratio compagnon/apprenti, délai et alertes.
5. Comparer avec vos licences et cartes existantes pour repérer les écarts **avant** de soumissionner.

> Le résultat n'est pas sauvegardé : copiez les éléments importants ailleurs si vous devez les conserver.

### 3.11 Diagnostiquer sa conformité avec l'assistant IA

1. Onglet **Assistant IA** → choisir un des six outils.
2. Pour **« Analyser ma conformité »** : cliquer « Analyser » → l'IA lit vos données et renvoie un score, des non-conformités et des recommandations.
3. Pour **« Chat réglementaire »** : poser une question ; cocher « Inclure le contexte de mon dossier » pour une réponse adaptée à vos enregistrements.
4. Pour **« Prédire mes renouvellements »** : obtenir un calendrier de 12 mois avec le budget estimé.
5. Pour **« Générer un rapport »** : produire un rapport structuré à l'écran (à recopier ailleurs si besoin, aucun export n'existe).
6. Pour **« Recommander des formations »** : ajouter au besoin des projets prévus, puis lancer l'analyse des compétences de l'équipe.

Chaque appel consomme des crédits ; un solde épuisé renvoie une erreur 402 dans la bannière rouge.

### 3.12 Lire le tableau de bord et prioriser

1. Onglet **Tableau de bord**.
2. Vérifier le **score** (badge en tête de page) et les **indicateurs d'alerte** (« À renouveler », « Expirés »).
3. Parcourir la **liste des alertes** : traiter d'abord les alertes de priorité HAUTE (déjà expirées), puis les MOYENNE (à renouveler).
4. Consulter les **répartitions** pour voir la concentration par catégorie, métier ou type.
5. Au besoin, se référer aux **organismes de référence** et aux **conseils pratiques** en bas de page.

---

## 4. Référence

### 4.1 Les six onglets

| Clé interne | Libellé affiché | Contenu réel |
|-------------|-----------------|--------------|
| `dashboard` | Tableau de bord | Synthèse (score, indicateurs, alertes, répartitions, ressources) |
| `rbq` | Licences RBQ (N) | Registre des licences RBQ |
| `ccq` | Cartes CCQ (N) | Registre des cartes CCQ |
| `attestations` | Documents légaux (N) | Registre des attestations |
| `verifications` | Audits et inspections | Vérification IA d'un projet |
| `assistant` | Assistant IA | Six outils IA |

### 4.2 Points d'accès de l'API (31 au total, préfixe `/api/erp/v1/conformite`)

**Métadonnées (2)** — lecture, tout utilisateur du tenant :

| Méthode + chemin | Rôle |
|------------------|------|
| `GET /constants` | Renvoie les enumérations et listes de référence pour l'interface |
| `GET /resources` | Renvoie les 8 organismes et les 6 sections de conseils |

**Licences RBQ (6)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `GET /licences` (filtres statut, catégorie, recherche) | tout utilisateur |
| `GET /licences/expiring?days=60` (bornes 1-365) | tout utilisateur |
| `GET /licences/{id}` | tout utilisateur |
| `POST /licences` | administrateur |
| `PUT /licences/{id}` | administrateur |
| `DELETE /licences/{id}` | administrateur |

**Cartes CCQ (6)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `GET /cartes` (filtres statut, métier, recherche) | tout utilisateur |
| `GET /cartes/expiring?days=60` (bornes 1-365) | tout utilisateur |
| `GET /cartes/{id}` | tout utilisateur |
| `POST /cartes` | administrateur |
| `PUT /cartes/{id}` | administrateur |
| `DELETE /cartes/{id}` | administrateur |

**Attestations (8)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `GET /attestations` (filtres statut, type) | tout utilisateur |
| `GET /attestations/expiring?days=30` (bornes 1-365) | tout utilisateur |
| `GET /attestations/{id}` | tout utilisateur |
| `POST /attestations` | administrateur ou comptable |
| `PUT /attestations/{id}` | administrateur ou comptable |
| `DELETE /attestations/{id}` | administrateur ou comptable |
| `POST /attestations/{id}/upload` (PDF/JPG/PNG/WebP, 10 Mo) | administrateur ou comptable |
| `GET /attestations/{id}/download` | tout utilisateur |

**Tableau de bord et alertes (2)** :

| Méthode + chemin | Rôle |
|------------------|------|
| `GET /statistics` | Indicateurs, score et 3 répartitions |
| `GET /alertes` | 6 familles d'alertes (20 lignes maximum chacune) |

**Assistant IA (7)** — tout utilisateur du tenant, blocage réel par les crédits :

| Méthode + chemin | Rôle |
|------------------|------|
| `POST /ai/analyze` | Analyse complète (score, risques, non-conformités) |
| `POST /ai/chat` | Chat réglementaire (question 1-2000 caractères, contexte optionnel) |
| `POST /ai/verify-project` | Exigences réglementaires d'un projet |
| `POST /ai/search-regulations` | Recherche de réglementation (requête 1-1000 caractères) |
| `POST /ai/predict-renewals` | Calendrier de renouvellements sur 12 mois |
| `POST /ai/generate-rapport` | Rapport de conformité structuré |
| `POST /ai/recommend-formations` | Recommandations de formations (jusqu'à 20 projets prévus) |

> **Note d'architecture** : le module est **monté directement** (sans bloc défensif) dans `erp_api.py`. Les données de référence proviennent de `conformite_data.py`, qui est un simple **module de données** (aucun point d'accès n'y est défini). Aucun point d'accès de conformité ne se trouve dans `secondary.py`.

### 4.3 Statuts par entité

| Entité | Statuts possibles | Couleurs |
|--------|-------------------|----------|
| Licence RBQ | ACTIVE · SUSPENDUE · EXPIREE · REVOQUEE | vert · jaune · rouge · gris foncé |
| Carte CCQ | ACTIVE · SUSPENDUE · EXPIREE | vert · jaune · rouge |
| Attestation | VALIDE · EN_RENOUVELLEMENT · EXPIREE | vert · jaune · rouge |
| Niveau de risque (IA) | FAIBLE · MOYEN · ELEVE · CRITIQUE | vert · jaune · orange · rouge |
| Priorité (alerte) | HAUTE · MOYENNE · BASSE | rouge · jaune · vert |
| Gravité (non-conformité IA) | MINEURE · MAJEURE · CRITIQUE | jaune · orange · rouge |

> Les statuts d'attestation ne sont que **trois** dans le moteur (VALIDE, EN_RENOUVELLEMENT, EXPIREE). Toute autre valeur qui apparaîtrait dans un menu (par exemple « échu » ou « suspendu ») serait un vestige de traduction sans effet réel.

### 4.4 Les 27 catégories RBQ (par sous-groupe)

Source : `CATEGORIES_RBQ`, 27 entrées. (Plusieurs commentaires du code annoncent « 26 » ; la liste réelle en contient bien **27**.)

**Générale (4)**
- `1.1` — Entrepreneur en bâtiments résidentiels neufs classe I
- `1.2` — Entrepreneur en bâtiments résidentiels neufs classe II
- `1.3` — Entrepreneur en petits bâtiments
- `16` — Entrepreneur général

**Mécanique (10)**
- `2` — Entrepreneur en systèmes de chauffage à air chaud
- `3` — Entrepreneur en plomberie
- `15.1` — Systèmes de chauffage à eau chaude
- `15.2` — Systèmes de chauffage à vapeur
- `15.3` — Systèmes de brûleurs au mazout
- `15.4` — Systèmes de brûleurs au gaz
- `15.5` — Ventilation
- `15.6` — Climatisation
- `15.7` — Réfrigération
- `15.8` — Protection-incendie

**Électricité (1)**
- `4` — Entrepreneur en électricité

**Génie civil (2)**
- `5.1` — Excavation et terrassement
- `5.2` — Fondations profondes

**Structure (6)**
- `6` — Charpente et menuiserie
- `11.1` — Structures de béton
- `11.2` — Béton préfabriqué
- `12` — Armature et ferraillage
- `13` — Structures métalliques et éléments préfabriqués
- `14` — Maçonnerie

**Enveloppe (3)**
- `7` — Revêtements extérieurs
- `9` — Toitures
- `10` — Isolation, étanchéité, couvertures et revêtements métalliques

**Finition (1)**
- `8` — Systèmes intérieurs

### 4.5 Les 28 métiers CCQ

Source : `METIERS_CCQ`, 28 métiers.

**Métiers à progression multiple (5)** : Apprenti (4 périodes), Grutier (4 classes), Opérateur d'équipement lourd (4 classes), Soudeur (classes A/B/C), Soudeur en tuyauterie (classes A/B) — voir le tableau en 2.3.

**Métiers à qualification « Compagnon » (23)** : Briqueteur-maçon, Calorifugeur, Carreleur, Charpentier-menuisier, Chaudronnier, Cimentier-applicateur, Couvreur, Électricien, Ferblantier, Ferrailleur, Frigoriste, Mécanicien d'ascenseur, Mécanicien de chantier, Mécanicien en protection-incendie, Monteur-assembleur, Monteur-mécanicien (vitrier), Opérateur de pelles mécaniques, Peintre, Plâtrier, Plombier, Poseur de revêtements souples, Poseur de systèmes intérieurs, Tuyauteur.

### 4.6 Les 5 types d'attestation

| Code | Libellé | Organisme | Objet |
|------|---------|-----------|-------|
| `REVENU_QUEBEC` | Attestation de Revenu Québec | Revenu Québec | Conformité fiscale provinciale |
| `ARC` | Attestation de l'Agence du revenu du Canada | Agence du revenu du Canada | Conformité fiscale fédérale |
| `CNESST` | Attestation de conformité CNESST | CNESST | Santé et sécurité au travail |
| `CCQ` | Attestation CCQ - État de situation | Commission de la construction du Québec | État des cotisations |
| `RBQ` | Attestation de solvabilité RBQ | Régie du bâtiment du Québec | Solvabilité et cautionnement |

### 4.7 Paramètres de la vérification de projet (IA)

- **Types de projet (7)** : Résidentiel unifamilial, Résidentiel multifamilial, Commercial, Industriel, Institutionnel, Rénovation majeure, Agrandissement.
- **Régions (18)** : Bas-Saint-Laurent, Saguenay–Lac-Saint-Jean, Capitale-Nationale, Mauricie, Estrie, Montréal, Outaouais, Abitibi-Témiscamingue, Côte-Nord, Nord-du-Québec, Gaspésie–Îles-de-la-Madeleine, Chaudière-Appalaches, Laval, Lanaudière, Laurentides, Montérégie, Centre-du-Québec, Autre région.
- **Types de travaux (12)** : Fondation, Charpente, Électricité, Plomberie, Chauffage/Ventilation, Toiture, Revêtement extérieur, Finition intérieure, Maçonnerie, Structure métallique, Excavation, Piscine.
- **Types de projet pour la recommandation de formations (5)** : Résidentiel, Commercial, Industriel, Institutionnel, Infrastructure.

### 4.8 Score de conformité (barème complet)

Le score part de **100** puis retranche les pénalités suivantes :

| Situation | Pénalité |
|-----------|----------|
| Licence RBQ **révoquée** | −15 |
| Licence RBQ **expirée** | −10 |
| Licence RBQ **suspendue** | −6 |
| Attestation **expirée** | −8 |
| Carte CCQ **expirée** | −5 |
| Carte CCQ **suspendue** | −3 |

- Le score est borné entre **0 et 100**.
- **Aucune donnée enregistrée → score = 0** (les enregistrements suspendus, révoqués ou en renouvellement comptent tout de même comme « des données », pour éviter qu'un tenant qui n'a que ce type d'enregistrements soit affiché à tort à 0).
- Affichage du badge : **vert ≥ 80 %**, **jaune de 50 à 79 %**, **rouge < 50 %**.

> Correction par rapport aux versions antérieures de ce manuel : les **suspensions** et les **révocations** pénalisent bel et bien le score (elles n'étaient pas prises en compte dans l'ancien barème documenté).

### 4.9 Fenêtres d'alerte

**Liste « Alertes de conformité » du tableau de bord** (`GET /alertes`) — 6 familles, 20 lignes maximum chacune :

| Type d'alerte | Condition | Priorité |
|---------------|-----------|----------|
| LICENCE_EXPIREE | date d'expiration passée | HAUTE |
| LICENCE_EXPIRE_BIENTOT | expire dans les 60 jours | MOYENNE |
| CARTE_EXPIREE | renouvellement passé | HAUTE |
| CARTE_EXPIRE_BIENTOT | expire dans les 60 jours | MOYENNE |
| ATTESTATION_EXPIREE | date d'expiration passée | HAUTE |
| ATTESTATION_EXPIRE_BIENTOT | expire dans les 60 jours | MOYENNE |

**Points d'accès « expiring » autonomes** (paramètre `?days=` borné à 1-365) :

| Point d'accès | Défaut |
|---------------|--------|
| `GET /licences/expiring` | 60 jours |
| `GET /cartes/expiring` | 60 jours |
| `GET /attestations/expiring` | 30 jours |

> Les dates limites sont calculées sur la **date locale du tenant** (fuseau horaire de l'entreprise), pas sur l'heure UTC du serveur — la bascule « expiré / valide » du soir correspond donc au calendrier local.

### 4.10 Tables PostgreSQL (schéma du tenant)

Les trois tables sont créées **à la demande** (à la première requête), pas au moment de la création du tenant.

| Table | Contenu et contraintes |
|-------|------------------------|
| `conformite_licences_rbq` | `numero_licence` **UNIQUE**, `categories` en JSONB, cautionnement et assurance en numérique |
| `conformite_cartes_ccq` | `numero_carte` **UNIQUE**, `employee_id` (lien logique vers `employees.id`, validé si la table existe), `metiers_additionnels` en JSONB, `asp_construction` booléen |
| `conformite_attestations` | **UNIQUE (type, numero)**, pièce jointe en `fichier_data` BYTEA + `fichier_nom`, `mime_type`, `taille` |

**Index créés automatiquement** : `idx_conf_licences_expiration`, `idx_conf_licences_statut`, `idx_conf_cartes_renouvellement`, `idx_conf_cartes_employee`, `idx_conf_attestations_expiration`, `idx_conf_attestations_type`.

### 4.11 Validations et codes d'erreur

| Règle ou limite | Réponse HTTP |
|-----------------|--------------|
| Numéro de licence déjà utilisé | 409 (conflit) |
| Numéro de carte déjà utilisé | 409 |
| Couple (type, numéro) d'attestation déjà utilisé | 409 |
| Date d'émission postérieure à la date d'expiration | 422 |
| Cautionnement ou assurance hors de 0 à 1 000 000 000 | 422 |
| Heures accumulées hors de 0 à 1 000 000 | 422 |
| Notes de plus de 5000 caractères | 422 |
| Plus de 30 catégories, ou un code de plus de 200 caractères | 422 |
| Statut hors de la liste autorisée | 400 |
| Type d'attestation hors de la liste | 400 |
| Catégorie RBQ ou métier CCQ hors de la liste officielle | 400 |
| Employé inexistant (création de carte) | 404 |
| Corps de mise à jour vide | 400 |
| Fichier de plus de 10 Mo | 413 |
| Type MIME hors PDF/JPG/PNG/WebP, ou octets d'en-tête non conformes | 415 |
| Service IA indisponible | 503 |
| Crédits IA épuisés | 402 |
| IA surchargée (« overload ») | 503 |
| Réponse IA vide ou mal formée | 502 |

### 4.12 Coûts de l'IA

- **Modèle** : `claude-opus-4-8`, 32 000 jetons maximum par appel.
- **Tarifs de base** : 5 $ US par million de jetons en entrée, 25 $ par million en sortie, 6,25 $ par million pour l'écriture de cache, 0,50 $ par million pour la lecture de cache.
- **Majoration** : × 1,30 (30 %).
- **Le débit se fait APRÈS validation de la réponse** : un appel qui échoue, revient vide ou mal formé **n'est pas facturé**.
- **Limite de débit dédiée** : 10 appels IA par minute et par adresse IP sur les chemins `/conformite/ai/` (c'est la classe d'endpoint la plus coûteuse de l'application).

### 4.13 Raccourcis et comportements utiles

- **Recherche** : délai de 400 ms avant l'envoi ; les caractères spéciaux (`\`, `%`, `_`) sont échappés côté serveur.
- **Chat réglementaire** : Entrée pour envoyer, Maj+Entrée pour un saut de ligne.
- **Badges de catégories** : 3 affichés au maximum sur ordinateur (puis « +N »), 4 sur mobile.
- **Verrou IA** : un seul appel IA à la fois (les boutons se désactivent pendant le traitement).

---

## 5. Intégrations et FAQ

### 5.1 Intégration avec le Module 10 (Employés)

- La table `employees` est consultée pour **valider l'existence** d'un employé à la création d'une carte CCQ ; la jointure affiche le nom complet dans le tableau.
- Si la fiche employé n'existe pas encore (tenant très récent), la jointure est omise et le tableau affiche « #identifiant ».
- **Aucune synchronisation automatique** : si un employé est supprimé, sa carte CCQ subsiste. Bonne pratique : supprimer la carte en parallèle.

### 5.2 Intégration avec les Projets, l'Immobilier et le Pointage

- **Aucun lien automatique.** Les licences RBQ ne sont pas vérifiées automatiquement à la création d'un projet.
- L'onglet « Audits et inspections » (vérification IA) s'utilise **manuellement** avant de soumissionner.
- Les heures CCQ ne sont pas alimentées par le **Module 12 (Pointage)** : le champ « heures accumulées » est saisi à la main.
- Les indicateurs de conformité de phase du **Module 34 (Immobilier)** sont distincts et sans lien avec les licences enregistrées ici.

### 5.3 Intégration avec la Comptabilité et les Subventions

- **Aucune écriture comptable automatique.** Les cautionnements et les assurances sont informatifs : ce ne sont pas des passifs ou des actifs comptables. À comptabiliser manuellement dans le **Module 14 (Comptabilité)**.
- Les recommandations de formations de l'IA peuvent évoquer des programmes subventionnés, mais **sans lien** automatique vers le **Module 17 (Subventions)**.

### 5.4 Intégration IA et crédits

- Les 7 outils IA passent par le contrôle de **crédits prépayés** du tenant (mêmes crédits que les autres fonctions IA de l'ERP, voir le **Module 24**).
- Le coût est journalisé après chaque appel réussi.
- La saisie de l'utilisateur est encadrée pour prévenir l'injection de consignes ; les liens renvoyés par la recherche sont assainis (seuls `http://` et `https://` sont conservés).

### 5.5 FAQ

**Q : Quelle est la différence entre la RBQ et la CCQ ?**
R : La **RBQ** délivre des licences aux **entreprises** (une par entité morale). La **CCQ** délivre des cartes de compétence aux **travailleurs individuels** (régime R-20). Une entreprise a une licence RBQ ; chaque ouvrier a sa carte CCQ.

**Q : Le module vérifie-t-il mes numéros de licence auprès du registre officiel de la RBQ ?**
R : **Non.** Aucune connexion aux registres RBQ ou CCQ. Toute la saisie est manuelle. Pour une vérification officielle, utilisez `rbq.gouv.qc.ca`.

**Q : Combien de catégories RBQ le module connaît-il ?**
R : **27** sous-catégories, du code 1.1 au code 16, réparties en 7 sous-groupes (voir 4.4). Certains commentaires internes du code disent « 26 », mais la liste réelle en compte 27.

**Q : Le score de conformité tient-il compte des suspensions et des révocations ?**
R : **Oui.** Une licence révoquée retire 15 points, une licence expirée 10, une licence suspendue 6, une attestation expirée 8, une carte expirée 5, une carte suspendue 3. (C'est une correction : l'ancien barème ne pénalisait que les expirations.)

**Q : Puis-je exporter mes données en Excel, en CSV ou en PDF ?**
R : **Non.** Il n'y a aucune exportation ni impression des données de conformité. Le seul téléchargement est la **pièce jointe** d'une attestation. Même le **rapport IA** est affiché à l'écran seulement.

**Q : Le module envoie-t-il des rappels par courriel avant les échéances ?**
R : **Non.** Les alertes sont uniquement visibles dans le tableau de bord. Prenez l'habitude de le consulter (les échéances se surveillent idéalement 60 à 90 jours à l'avance).

**Q : Puis-je téléverser plusieurs fichiers pour une même attestation ?**
R : **Non.** Une seule pièce jointe par attestation (PDF, JPG, PNG ou WebP, 10 Mo maximum), sans gestion de versions.

**Q : Que se passe-t-il si je téléverse un fichier de plus de 10 Mo ?**
R : Le serveur le refuse (erreur 413). Compressez le PDF ou réduisez la résolution de l'image. Il n'y a pas de compression automatique.

**Q : Le module gère-t-il les déclarations mensuelles d'heures à la CCQ ?**
R : **Non.** Le champ « heures accumulées » est un cumul manuel. Pour les déclarations, utilisez le portail employeur de la CCQ.

**Q : Puis-je réaffecter une carte CCQ à un autre employé ?**
R : **Non**, l'employé est verrouillé après la création. Supprimez la carte et créez-en une nouvelle.

**Q : Puis-je enregistrer plusieurs licences RBQ pour la même entreprise (société mère et filiales) ?**
R : **Oui.** Chaque licence est indépendante ; il n'y a pas de limite.

**Q : L'onglet « Audits et inspections » sert-il à consigner mes audits ?**
R : **Non**, malgré son nom. C'est l'outil de **vérification IA** des exigences d'un projet ; il ne sauvegarde rien. De même, l'onglet « Documents légaux » contient exactement les **attestations**.

**Q : Les crédits IA sont-ils consommés si la réponse de l'IA est mauvaise ?**
R : **Non.** Le débit intervient après validation de la réponse : un appel qui échoue, revient vide ou mal formé n'est pas facturé.

**Q : L'IA garantit-elle la conformité légale ?**
R : **Non.** Le diagnostic est indicatif. Le prompt système interdit d'inventer des références de loi. Pour les cas critiques, consultez la RBQ, la CCQ ou un conseiller spécialisé.

**Q : Les pièces jointes sont-elles servies de façon sécuritaire ?**
R : **Oui.** Le téléchargement force l'enregistrement du fichier, le nom est assaini, et le type MIME est revalidé (servi en flux binaire générique s'il sort de la liste autorisée).

**Q : Existe-t-il un journal des modifications (qui a changé quoi) ?**
R : Les tables conservent la date de création et de dernière modification, mais **pas** de journal détaillé par utilisateur. Utilisez le champ « Notes » pour consigner les changements importants.

**Q : Que faire en cas d'audit de la RBQ ou de la CNESST ?**
R : Téléchargez vos pièces jointes une par une, générez au besoin un rapport IA (à recopier, car il n'est pas exportable), et conservez vos documents selon les délais légaux applicables au Québec.

---

## 6. Récapitulatif

- **Rôle** : registre **manuel** de la conformité réglementaire québécoise — licences RBQ, cartes CCQ, attestations — avec tableau de bord et assistant IA. **Aucune** connexion aux registres officiels.
- **Accès** : barre latérale → groupe TERRAIN → **RBQ/CCQ** (icône bouclier), route `/conformite`. Onglet ouvert par défaut : **Licences RBQ**.
- **Six onglets** : Tableau de bord · Licences RBQ · Cartes CCQ · **Documents légaux** (= attestations) · **Audits et inspections** (= vérification IA) · Assistant IA. Attention : les deux libellés en gras sont trompeurs.
- **Trois entités** : licences RBQ (**27** catégories réparties en 7 groupes), cartes CCQ (**28** métiers à qualifications dynamiques), attestations (**5** types, une pièce jointe PDF ou image de 10 Mo maximum).
- **Permissions** : consultation pour tous ; écritures licences et cartes réservées à l'**administrateur** ; écritures attestations à l'**administrateur ou au comptable** ; outils IA soumis aux **crédits prépayés**.
- **Score de conformité** (0-100) : −15 licence révoquée, −10 licence expirée, −6 licence suspendue, −8 attestation expirée, −5 carte expirée, −3 carte suspendue ; 0 si aucune donnée ; badge vert ≥ 80 %, jaune ≥ 50 %, rouge < 50 %.
- **Alertes** : dans le tableau de bord seulement (licences et cartes à 60 jours, attestations à 30 jours par le point d'accès dédié) ; **aucun rappel** par courriel ou par calendrier.
- **Sept outils IA** (Claude Opus 4.8, 32 000 jetons, majoration 30 %, non facturés si la réponse échoue) : analyser · chat · vérifier un projet · rechercher · prédire · rapport · formations.
- **Limites clés** : aucune exportation ni impression (sauf la pièce jointe d'attestation), rapport IA à l'écran seulement, résultats IA éphémères, aucun import en masse, une seule pièce jointe par attestation, employé verrouillé après la création de la carte, pas de paie ni de déclarations CCQ.
- **31 points d'accès** sous `/api/erp/v1/conformite` ; **3 tables** par tenant créées à la demande.

---

**Documentation générée à partir du code (fichiers vérifiés)** :
- `ERP_REACT/backend/routers/conformite.py` (2531 lignes, 31 points d'accès dont 7 outils IA)
- `ERP_REACT/backend/routers/conformite_data.py` (398 lignes, module de données statiques — 27 catégories RBQ, 28 métiers CCQ, 5 types d'attestation, 18 régions, 8 organismes, 6 sections de conseils)
- `ERP_REACT/frontend/src/pages/ConformitePage.tsx` (3164 lignes, 6 onglets)
- `ERP_REACT/frontend/src/api/conformite.ts` (587 lignes)
- `ERP_REACT/frontend/src/store/useConformiteStore.ts` (740 lignes)

**Manuels liés** :
- Module 10 (Employés — créer la fiche employé avant la carte CCQ) — `10-operations-employes.md`
- Module 12 (Pointage — les heures ne sont pas synchronisées automatiquement ici) — `12-operations-pointage.md`
- Module 14 (Comptabilité — comptabilisation manuelle des cautionnements et assurances) — `14-operations-comptabilite.md`
- Module 17 (Subventions — programmes de formation financés) — `17-terrain-subventions.md`
- Module 34 (Immobilier — indicateurs de conformité de phase distincts) — `34-terrain-immobilier.md`
- Module 24 (Assistant IA — crédits IA et fonctionnement général de l'IA) — `24-communication-assistant-ia.md`
- Module 30 (Configuration — gestion du tenant et des accès) — `30-configuration.md`
