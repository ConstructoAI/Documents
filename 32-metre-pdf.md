# Module 32 — Métré (prise de quantités sur PDF)

> **Version** : 3.0 (réécrit d'après le code source réel, révision 2026-07 ; ajoute le **comptage par gabarit gratuit**, la parité de coût de la vision IA et la correction des dénombrements de points d'accès).
> **Accès** : le module n'a **pas** d'entrée dans le menu latéral et **pas** d'adresse `/metre`. On l'ouvre par **Ventes → Soumissions → onglet « Métré »** (onglet à chargement différé `metre-pdf`, `DevisPage.tsx:50,766,1363,1474`). Il faut donc d'abord ouvrir la page Soumissions.
> **Code de référence** :
> - Frontend : dossier `frontend/src/components/metre-pdf/` (**90 fichiers, ~42 400 lignes TS/TSX**) — composant principal `MetrePdf.tsx`, gestionnaire d'état `store.ts`, canevas de tracé `components/MeasurementCanvas.tsx` (bibliothèque **Fabric.js**), afficheur de plan `components/PDFViewer.tsx` (**pdf.js**), barres et panneaux (`TopToolbar`, `LeftPanel`, `RightPanel`, `BomEstimationPanel`, `BottomBar`, `MetreSavedBar`, `PageNavigator`, `AiCountPanel`, `MetreAssistantPanel`), état d'auto-comptage `useAiCount.ts`, utilitaires `utils/` (dont `canvasRotation.ts` pour la mesure sur plan pivoté, `costing.ts` pour le facteur facturable, `estimationHtml.ts` pour la soumission HTML), client d'API `components/metre-pdf/api.ts` (préfixe `/api/erp/v1/metre`) et `components/metre-pdf/api/metreAi.ts`.
> - Backend : `backend/routers/metre_pdf.py` (**5 055 lignes, 46 points d'accès** ; persistance + rendu + calibration + mesures + calques + produits + accrochage + **comptage par gabarit** + exports + import), `backend/routers/metre_ai_chat.py` (2 154 lignes, **1 point d'accès** ; chat assistant en diffusion continue), `backend/routers/metre_ai_conversations.py` (1 075 lignes, **4 points d'accès** ; conversations + confirmation d'action), `backend/routers/metre_ai_detect.py` (291 lignes, **1 point d'accès** ; auto-comptage par vision IA), `backend/routers/metre_ai_tools.py` (2 625 lignes ; **19 outils** de l'assistant, pas un routeur). **52 points d'accès au total**, préfixe réel commun `/api/erp/v1/metre`.
> **Tables PostgreSQL (par entreprise, créées à la demande)** : `metre_projects`, `metre_documents` (le PDF est stocké en **BYTEA**), `metre_calibrations`, `metre_layers`, `metre_layer_groups`, `metre_measurements` (colonnes JSONB `points` et `metadata_json`), `metre_products`, `metre_product_components`, `metre_ai_detections`, `metre_ai_conversations`, `metre_ai_messages`, `metre_ai_tool_executions` (12 tables). Elles sont créées **la première fois** que vous utilisez le module (mécanisme paresseux `_ensure_tables`, `metre_pdf.py:1252`), pas au moment de la création du compte.
> **Cadrage** : ce module est un **outil de prise de quantités (« métré ») sur plan PDF ou image**. On charge un plan, on **calibre l'échelle page par page**, on **trace des mesures** (distances, surfaces, périmètres, comptages, angles, cercles, plus des annotations), on les **associe à des produits** du catalogue (matériaux + main-d'œuvre CCQ) ou à des **assemblages paramétriques (BOM)**, puis on **génère une soumission** ou des **bordereaux** exportables (CSV, PDF, HTML, DXF, PNG), que l'on peut **renvoyer directement dans un devis**. Un **assistant IA conversationnel** lit le métré et propose des actions, et l'**auto-comptage** offre **deux méthodes** : le **comptage par gabarit (gratuit, OpenCV)** et la **vision IA (payante, Claude)**. Ce module **n'est pas** un logiciel de modélisation 3D (voir le module **25 — DAO / Modélisation 3D**), il **ne remplace pas** le module Soumissions (il l'alimente), et il **ne fait pas** de lecture automatique complète du plan (seuls l'accrochage OpenCV et l'auto-comptage d'une zone tracée assistent la saisie).

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « entreprise » ou « tenant » désigne votre compte (chaque entreprise a ses données isolées par un schéma PostgreSQL) ; « métré » désigne un projet de prise de quantités (un plan + ses mesures + ses calques + ses calibrations) ; « document » désigne le fichier PDF ou image chargé ; « calibration » désigne le lien entre les pixels du plan et les unités réelles ; « mesure » désigne un tracé chiffré ; « calque » regroupe des mesures ; « composite » ou « assemblage (BOM) » désigne un produit fait de sous-produits calculés par formule ; « soumission » ou « bordereau » désigne la sortie chiffrée ; « gabarit » désigne un motif image encadré servant de modèle au comptage. « Téléverser » = envoyer un fichier vers le serveur ; « télécharger » = récupérer un fichier depuis le serveur.

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

Transformer un **plan PDF** (ou une image) en **quantités chiffrées**. Concrètement, vous :

- **chargez** un plan à l'écran (une ou plusieurs pages) ;
- **calibrez** l'échelle de chaque page (vous tracez une ligne sur une cote connue et vous saisissez sa vraie longueur) ;
- **tracez** vos mesures par-dessus le plan (murs, surfaces, périmètres, comptages, etc.) — le module calcule automatiquement les longueurs et surfaces réelles ;
- **organisez** vos mesures en **calques** (par exemple « Murs sous-sol », « Plancher RDC ») et en **groupes** ;
- **associez** un **produit** (prix, unité, pourcentage de perte) et, au besoin, de la **main-d'œuvre** (corps de métier CCQ) à chaque mesure, ou vous liez un calque à un **assemblage paramétrique (BOM)** qui se remplit tout seul ;
- **produisez** une **soumission** (matériaux + main-d'œuvre + taxes) et des **bordereaux** que vous **exportez** (CSV, PDF, HTML, DXF, PNG) ou que vous **renvoyez dans un devis** existant ou neuf.

Un **assistant IA** peut lire votre métré et exécuter des actions (créer un calque, chercher un produit, etc.) après votre confirmation. Pour les éléments répétitifs (prises, portes, fenêtres, luminaires...), deux **auto-comptages** sont offerts : le **comptage par gabarit** (gratuit, sans IA) et le **comptage par vision IA** (payant).

### 1.2 Comment y accéder

Le Métré **n'est pas** une page autonome. C'est un **onglet** de la page Soumissions/Devis.

| Étape | Détail |
|-------|--------|
| 1 | Menu latéral → **Ventes** → **Soumissions** (page Devis, `DevisPage.tsx`). |
| 2 | Dans la barre d'onglets de la page (Devis · Estimation IA · **Métré** · Manuel · Conditions · Assistant), cliquez sur **« Métré »** (`DevisPage.tsx:1363`, libellé `devis.tabs.metre`). |
| 3 | Le module se charge à la demande (un court message « Chargement Métré... » s'affiche pendant le chargement du module). |

Conséquences pratiques de cette intégration :

- Le module **reçoit le contexte du devis** actuellement sélectionné dans la page. Si un devis est ouvert, un **bandeau bleu « Devis connecté »** l'indique et les items du métré pourront lui être ajoutés. Sinon, une **fiche client** repliable apparaît pour préremplir une future soumission.
- Fermer l'onglet (bouton **X** du module) **replie** le Métré et rebascule sur l'onglet « Devis » (`DevisPage.tsx:1561`) ; votre travail est conservé côté serveur (voir §3.15).

### 1.3 Rôles et permissions

- **Aucune restriction par rôle propre au module.** Tous les points d'accès du Métré exigent seulement que vous soyez un **utilisateur authentifié de votre entreprise** (garde `_require_tenant`, `metre_pdf.py:44` ; `get_current_user` pour le chat, la détection et les conversations). Il n'y a **aucune garde « administrateur », « comptable » ou « super-administrateur »** : tout membre du tenant a le **plein contrôle** (créer, mesurer, chiffrer, gérer les catalogues, exporter, supprimer).
- **Isolation stricte par entreprise.** Vos métrés, vos catalogues (produits, main-d'œuvre) et vos assemblages sont partagés **au sein de votre entreprise**, jamais visibles par une autre (isolation par schéma PostgreSQL). La bibliothèque des métrés est particulièrement protégée : la jointure sur `users` est **rigidement qualifiée au schéma** de l'entreprise pour ne jamais retomber sur la table publique (aucune fuite de noms entre entreprises).
- **Mode consultation (lecture seule).** Si l'abonnement de votre entreprise est suspendu, l'ERP passe en mode consultation au niveau global de l'authentification : les écritures sont alors bloquées. Ce comportement n'est pas redéfini dans le module ; il est hérité du compte.
- **Assistant IA et crédits.** L'assistant conversationnel et l'auto-comptage **par vision** consomment des **crédits IA** facturés à votre entreprise (voir §1.7). Le comptage **par gabarit** est **gratuit**. Le super-administrateur de la plateforme est exempté de la facturation IA.

### 1.4 Concepts clés

- **Métré (projet)** : l'unité de travail. Regroupe un ou plusieurs **documents**, toutes les **mesures**, les **calques**, les **calibrations** et un éventuel **lien vers un devis**. Table `metre_projects`.
- **Document** : le plan chargé (PDF ou image). Stocké en base en **BYTEA** (il survit aux redéploiements du serveur ; repli disque hérité si la colonne est vide). Les documents à plusieurs pages sont pris en charge. Table `metre_documents`.
- **Calibration** : le lien pixels ↔ réalité, défini **par page**. Vous tracez une ligne sur une cote connue et saisissez sa longueur réelle ; le module en déduit le **facteur d'échelle** (unité réelle par pixel). Table `metre_calibrations` (une par page, contrainte d'unicité document + page).
- **Mesure** : un tracé typé (distance, surface, comptage, etc.) avec une **valeur** calculée dans l'**unité de calibration** de sa page. Chaque mesure porte des points (JSONB), une couleur, un calque, un produit éventuel, une pente, des déductions et de la main-d'œuvre. Table `metre_measurements`.
- **Calque** : regroupe des mesures et porte une couleur, une visibilité et un verrou. Un calque peut être **lié à un assemblage (composite)** pour alimenter un BOM. Table `metre_layers`.
- **Groupe de calques** : une section pliable qui regroupe plusieurs calques. Table `metre_layer_groups`.
- **Produit** : un article du **catalogue** (prix, unité de prix, pourcentage de perte). **Simple** ou **composite**. Table `metre_products`.
- **Composite / assemblage (BOM)** : un produit fait de **sous-produits** dont la quantité vient d'une **formule paramétrique** évaluée sur des **variables d'entrée** (alimentées par les mesures). Table `metre_product_components`.
- **Gabarit** : un rectangle que vous tracez autour d'**un** symbole ; le module cherche tous les motifs **identiques** de la page (comptage gratuit, voir §3.11).
- **Déduction** : une mesure « en soustraction » (par exemple une ouverture) rattachée à une mesure parente ; la quantité **nette** = parente − somme des déductions (jamais négative).
- **Soumission / bordereau** : la sortie chiffrée (matériaux + main-d'œuvre + taxes), exportable ou convertible en devis.

### 1.5 Types de mesures

| Outil | Raccourci | Rôle | Valeur calculée |
|---|---|---|---|
| **Distance** | D | Longueur d'un segment (2 points) | longueur (unité linéaire) |
| **Surface** | A | Aire d'un polygone (formule du lacet) | aire (unité²) |
| **Rectangle** | R | Aire d'un rectangle (2 coins) | aire |
| **Périmètre** | P | Contour fermé | longueur du contour |
| **Polyligne** | L | Suite de segments ouverte | somme des longueurs |
| **Angle** | N | Angle entre 3 points | degrés |
| **Comptage** | C | Nombre d'éléments (un clic par élément) | nombre |
| **Compter par IA** | U | Auto-comptage d'une zone encadrée — **par gabarit (gratuit)** ou **par vision IA (payant)** (voir §3.11) | nombre |
| **Cercle** | I | Aire d'un disque (centre + rayon) | aire (π·r²) |
| **Cotation** | X | Cote architecturale annotée | longueur |
| **Mur (saisie clavier)** | J | Segment posé à une longueur exacte au clavier (voir §3.4) | longueur |

**Annotations** (non chiffrées) : Texte (T), Flèche (W), Nuage de révision (Q), Main levée (F), Surligneur (G), Note (E), Bulle de texte (B), plus les **symboles architecturaux** (catalogue dédié). Elles servent à documenter le plan, pas à chiffrer.

Source des outils : `TopToolbar.tsx:63-87`.

### 1.6 Unités et saisie impériale

- **Unités de calibration / mesure** : `m`, `cm`, `mm`, `ft` (pi), `in` (po). Ce sont les cinq unités acceptées à la calibration (`metre_pdf.py`, `CalibrationCreate.unit`) et pour les mesures créées par l'assistant IA (`ALLOWED_MEASUREMENT_UNITS`, `metre_ai_tools.py`).
- **Saisie impériale** : format pieds-pouces-seizièmes. Exemples reconnus : `20-06-08` (20 pi 6 po 8/16), `200608` (même valeur, sans tirets), ou une valeur décimale (`3.048`). La détection impériale est automatique et affiche l'interprétation.
- **Unités de prix des produits** : `pi` / `m` (linéaire), `pi²` / `m²` / `vg²` (aire), `m³` (volume), `unité` / `feuille` / `sac` / `boîte` (comptage), `heure`. Voir le tableau de correspondance en §4.4.
- **Réconciliation automatique mesure ↔ prix** (chemin de l'argent) : si votre plan est calibré en **mètres** mais que le produit est tarifé au **pi²**, la quantité est **convertie** avant le chiffrage. Le facteur suit la dimension : linéaire → linéaire = facteur de conversion ; aire → aire = facteur au carré ; **dimension croisée** (par exemple une aire liée à un prix au linéaire, ou un volume) → le facteur reste **1** et la mesure est marquée **« unité incompatible »** (quantité non convertie, avertissement affiché). Cette conversion corrige la sous-facturation classique de ×10,76 quand un plan métrique alimente un catalogue en pi² (`_billable_factor`, `metre_pdf.py:3900`, en parité stricte avec `utils/costing.ts`).

### 1.7 Modèles IA et facturation

Deux fonctions consomment des **crédits IA** de votre entreprise, refacturés au **coût réel majoré de 30 % (× 1,30)**, avec un **plafond de 2,00 $ US par appel** :

| Fonction | Modèle | Type | Réf. |
|---|---|---|---|
| **Assistant Métré** (chat) | Claude Opus 4.8 (`claude-opus-4-8`) | Réponse en diffusion continue, boucle d'outils (max 8 itérations), plafond de 200 messages / conversation | `metre_ai_chat.py:112` |
| **Compter par IA — mode Vision** | Claude Opus 4.8 (`claude-opus-4-8`) | Détection d'occurrences sur une zone encadrée, sans écriture en base | `metre_ai_detect.py:47` |

Fonctions **gratuites** (aucun crédit) :

- Le **comptage par gabarit** (mode « Gabarit (gratuit) » de l'outil Compter par IA) : traitement **OpenCV** local, **0 $ IA, aucun crédit débité** (`detect_template`, `metre_pdf.py:4943`).
- La **détection de points d'accrochage** (snap) : traitement **OpenCV** côté serveur, **sans coût IA**.
- La **confirmation d'une action d'écriture** de l'assistant porte un **coût symbolique fixe** de 0,001 $ (`_METRE_TOOL_CONFIRM_COST_USD`, `metre_ai_conversations.py`).

**Parité de coût avec l'Estimation IA.** Le coût de la vision IA et du chat est calculé **exactement comme l'Estimation IA** (commit `32807a9b`, `metre_ai_detect.py:49-57`) : **5,00 $/M** de jetons en entrée, **25,00 $/M** en sortie, **10,00 $/M** en écriture de cache, **0,50 $/M** en lecture de cache, le tout **× 1,30**, plafonné à **2,00 $** par appel/message. La clé API reste **côté serveur** ; le client n'envoie que son jeton d'authentification. En mode auto-hébergé (facturation désactivée) ou pour un super-administrateur, ces appels restent **gratuits mais tracés** (`track_ai_usage`, tableau Usage IA).

### 1.8 Ce que le module fait — et ne fait pas

Le module **fait** : charger un plan PDF/image, calibrer par page, tracer 11 types de mesures + 7 annotations + symboles, organiser en calques/groupes, associer produits et main-d'œuvre, construire des assemblages BOM paramétriques, **compter par gabarit (gratuit)** ou **par vision IA (payant)**, dialoguer avec un assistant IA, générer une soumission, exporter (CSV, PDF, HTML, DXF, PNG) et alimenter un devis.

Le module **ne fait pas** :

- **Pas de route `/metre` ni d'entrée de menu.** Accès **uniquement** par l'onglet « Métré » de la page Soumissions.
- **Pas d'assistant IA tant que le plan n'est pas enregistré.** Le bouton de l'assistant reste **grisé** tant que le document n'a pas d'identifiant serveur numérique (voir §3.12). Une seule conversation active par document.
- **Pas de comptage par gabarit sur une image ni sur un plan pivoté.** Le gabarit exige un **PDF enregistré** et un plan **droit (0°)** ; pour une image, utilisez la vision IA.
- **Pas d'auto-comptage par vision sur un plan pivoté.** Le plan doit être **droit (0°)** ; sinon un message vous invite à réinitialiser la rotation.
- **Pas de reconnaissance sémantique dans le gabarit.** Le comptage par gabarit trouve des motifs **identiques au pixel** (mêmes symboles) ; il ne « comprend » pas ce qu'il compte. Pour un élément dessiné de plusieurs façons, utilisez la vision IA.
- **Pas de correction automatique des quantités douteuses.** Les unités incompatibles, les contours qui se croisent (« auto-intersectants ») et les composites à formule liés en direct sont **signalés** mais pas corrigés.
- **Pas d'assemblages/BOM hors mode ERP.** Les composites ne sont disponibles que lorsque le module est connecté à l'ERP (message `productCatalog.add.compositeOnlyInErp`).
- **La « Soumission BOM automatique » n'inclut pas** les valeurs manuelles saisies dans le panneau BOM, et ne génère **pas** de lignes de main-d'œuvre (`MetrePdf.tsx:1180-1184`).
- **Le plein écran est un affichage seulement** (CSS). La touche **Échap** n'en sort pas (elle annule l'outil ou désélectionne).
- **L'import direct « métré → devis » par le serveur est gelé.** Le point d'accès `import-to-devis` renvoie 503 par défaut ; le vrai chemin passe par la **modale Soumission** (voir §3.13 et §4.2).

### 1.9 Sous-modules et panneaux

| Élément | Composant | Rôle |
|---|---|---|
| **Barre « Métré sauvegardé »** | `MetreSavedBar` | Nom du métré actif, état de sauvegarde, ouvrir / nouveau / renommer / fermer. |
| **Barre d'outils** | `TopToolbar` | Fichier, outils de mesure et d'annotation, transformations, zoom, aides au tracé, catalogues, IA, exports, plein écran, annuler/rétablir. |
| **Panneau gauche** | `LeftPanel` | Calques, mesures (par page), consolidé (toutes pages). |
| **Zone centrale** | `PDFViewer` + `MeasurementCanvas` | Plan + couche de tracé (Fabric.js) + zoom + navigation de pages + surimpression d'auto-comptage. |
| **Panneau droit** | `RightPanel` | Propriétés de la mesure, calibration, catalogues, résumé des coûts, document. |
| **Panneau BOM** | `BomEstimationPanel` | Estimation en direct des assemblages, bordereaux, exports CSV. |
| **Barre du bas** | `BottomBar` | Outil actif, position du curseur, valeur en direct, unités, état de calibration. |
| **Assistant IA** | `MetreAssistantPanel` | Chat, conversations, profils, confirmation d'action. |
| **Auto-comptage** | `AiCountPanel` | Comptage d'une zone encadrée (gabarit gratuit ou vision IA). |

---

## 2. Interface

### 2.1 Disposition générale

De haut en bas (`MetrePdf.tsx`) :

```
+---------------------------------------------------------------------------+
| BARRE « MÉTRÉ SAUVEGARDÉ » : nom · état de sauvegarde · Ouvrir · Nouveau   |
+---------------------------------------------------------------------------+
| BANDEAU « Devis connecté : {nom} »   OU   FICHE CLIENT (repliable)         |
+---------------------------------------------------------------------------+
| BARRE D'OUTILS : Fichier | Mesures | Annotations | Transf. | Zoom |        |
|                  Aides au tracé | Catalogues | IA | Exports | Plein écran  |
+----------+------------------------------------------+----------+-----------+
| PANNEAU  |   AFFICHEUR (plan PDF/image)             | PANNEAU  |  PANNEAU  |
| GAUCHE   |   + canevas de mesures (Fabric.js)       | DROIT    |  BOM      |
| Calques  |   + contrôles de zoom / rotation         | Propri-  |  (replié  |
| Mesures  |   + surimpression Auto-comptage           | étés     |  par      |
| Consolidé|                                          | Coûts    |  défaut)  |
+----------+------------------------------------------+----------+-----------+
| NAVIGATION DE PAGES : ‹  page N / total  ›                                 |
+---------------------------------------------------------------------------+
| BARRE DU BAS : outil · curseur · valeur · SNAP · unités · calibration      |
+---------------------------------------------------------------------------+
```

Le rendu superpose **trois couches** : le plan (rendu par **pdf.js**, mis en cache haute résolution), les mesures (dessinées par **Fabric.js**, `MeasurementCanvas.tsx:4`), et une couche d'interaction qui capte la souris (accrochage, clic, glisser). Le module gère la **mesure sur plan pivoté** : si vous tournez le plan, les tracés restent cohérents grâce à une matrice de rotation (`utils/canvasRotation.ts`).

### 2.2 Barre « Métré sauvegardé » (`MetreSavedBar`)

Deux états.

- **Aucun métré ouvert** : une icône d'alerte, le message « **Aucun métré ouvert.** Créez ou ouvrez un métré... », et deux boutons : **« Ouvrir »** et **« Nouveau métré »**.
- **Métré actif** : une pastille pulsante, l'étiquette « MÉTRÉ », le **nom** et la description du métré, puis à droite :
  - un **compteur** « {n} mesures » ;
  - un **indicateur de sauvegarde** rafraîchi toutes les 30 secondes : « Sauvegardé », « Sauvegardé il y a N s / min / h » ou « Sauvegardé le {date} » ; ou un badge rouge **« Sauvegarde échouée »** (cliquable pour l'ignorer) ; ou « PDF non chargé » ;
  - des actions : **crayon** (renommer), **« Ouvrir »** (un autre métré), **« Nouveau »**, **X** (fermer).

### 2.3 Bandeau devis / fiche client

- Si un devis est ouvert dans la page Soumissions : bandeau bleu **« Devis connecté : {nom} — les items du métré seront ajoutés à ce devis »**.
- Sinon : une **fiche client** repliable (`ClientInfoCard`) qui préremplit la future soumission (client entreprise, contact ; listes chargées depuis l'ERP).

### 2.4 Barre d'outils supérieure (`TopToolbar`)

Les groupes sont séparés par de fins traits verticaux. Chaque outil affiche son **raccourci** entre parenthèses dans l'infobulle.

**1. Fichier.**
- **Ouvrir un plan — PDF ou image (Ctrl+O)** : accepte `.pdf, .png, .jpg, .jpeg, .bmp, .tiff, .tif, .webp` (`TopToolbar.tsx:272`).
- **Nouveau plan vierge (ARCH D 36"×24")** : génère un PDF vierge au format ARCH D pour dessiner à main levée ou coller une capture.

**2. Outils de mesure** : Calibrer (K), Sélection (V), Distance (D), Surface (A), Rectangle (R), Périmètre (P), Polyligne (L), Angle (N), Comptage (C), **Compter par IA (U)**, Cercle (I), Cotation (X), Déplacer / main (H). Un bouton séparé **« Mur — mesure clavier (J) »** (`TopToolbar.tsx:441`) ouvre la saisie impériale.

**3. Outils d'annotation** : Texte (T), Flèche (W), Nuage de révision (Q), Main levée (F), Surligneur (G), Note (E), Bulle de texte (B). Plus **Symbole** (catalogue de symboles architecturaux).

**4. Transformation** (actives seulement si une sélection existe) : **Pivoter 45°**, **Copie miroir horizontal (M)**, **Copie miroir vertical (Maj+M)**.

**5. Zoom** : Zoom arrière (−), pourcentage affiché, Zoom avant (+), **Ajuster à la page** (borné entre 0,1× et 10×).

**6. Aides au tracé** (bascules) : **Accrochage / snap (F3)**, **Mode ortho (F8)**, **Grille (F7)**.

**7. Panneaux et catalogues** (bascules) : **Calculatrice Construction**, **Convertisseur de pente**, **Catalogue de produits**, **Corps de métier CCQ (main-d'œuvre)**, **Symboles architecturaux**.

**8. Assistant IA** (`TopToolbar.tsx:462`, icône `MessagesSquare`) : ouvre/ferme le panneau de chat. Il est **désactivé** tant qu'aucun document à identifiant serveur numérique n'est chargé (infobulle « Enregistrez d'abord le métré »).

**9. Résumé et exports** : **Résumé multi-pages**, **Exporter en PNG** (3 fichiers : plan annoté + détail produits + détail BOM ; les fichiers vides sont ignorés, `TopToolbar.tsx:189-227`), **Générer une soumission** (mesures avec produit associé), **Générer une soumission BOM automatique** (calques liés aux composites).

**10. À droite** : **Plein écran (F11)**, **Annuler (Ctrl+Z)**, **Rétablir (Ctrl+Y)**.

### 2.5 Panneau gauche (`LeftPanel`)

Trois sections repliables.

**A. Calques.**
- En-tête : **Créer un groupe**, **Ajouter un calque** (saisie du nom en ligne + OK).
- **Groupes de calques** : en-tête repliable, renommer (double-clic), œil « masquer/afficher tous les calques de la section », supprimer (confirmation), zone « Glisser des calques ici », glisser-déposer d'un calque vers un groupe ou vers « Sans groupe ».
- **Par calque** : pastille de couleur (sélecteur), nom (double-clic pour renommer, max 100 caractères), compteur de mesures, **œil** (afficher/masquer), **cadenas** (verrouiller/déverrouiller), **lien BOM** (icône `Link2` ; ouvre le sous-panneau « Liaison BOM »), monter/descendre, **corbeille** (supprimer le calque et ses mesures, confirmation). Glisser une mesure sur un calque la déplace.
- **Sous-panneau « Liaison BOM » (`CompositeLinkPanel`)** : un menu pour choisir un composite (« -- Aucun -- » + liste) ; ensuite, pour chaque variable de l'assemblage : soit un badge **« auto (dérivé du tracé) »** ou **« auto (mesures du calque) »** (lecture seule), soit un champ manuel, soit un champ impérial pieds-pouces-seizièmes (`ImperialFeetInput`) pour l'unité « pi ».

**B. Mesures (page N).**
- Badge orange **« {n} à corriger »** si des mesures n'ont pas de produit (bascule un filtre « n'afficher que celles-ci : sans produit / produit supprimé »).
- Mesures **groupées par calque** (repliables, état mémorisé localement). Par mesure : pastille de couleur, icône de type, étiquette, **valeur formatée**, flèches **↑ / ↓** (copier vers le calque voisin), **Dupliquer** (Ctrl+D), **Déplacer vers...** (recherche de calque + glisser-déposer), **Supprimer**. Ligne 2 : produit associé, ou **« Sans produit »** (orange), ou **« Produit introuvable »** (produit supprimé du catalogue).
- Sélection : clic ; Ctrl+clic (ajouter/retirer) ; Maj+clic (plage).

**C. Consolidé (toutes pages).**
- Lien **« Détails »** (ouvre le résumé multi-pages).
- Totaux **par type** (icône + nombre + total formaté) et **par produit** (nom, quantité + unité, coût $), plus un **Total** global. Le coût suit la formule `valeur nette × pente × perte × prix`, convertie à l'unité de prix (alignée sur le résumé et l'estimation HTML).

### 2.6 Zone centrale (afficheur)

- **Afficheur (`PDFViewer`)** : rend la page courante. À vide : « Glissez-déposez un PDF ou une image directement ici, ou utilisez le bouton... » + « Nouveau plan vierge (ARCH D 36"×24") ».
- **Canevas de mesures (`MeasurementCanvas`, Fabric.js)** : tracé, sélection, déplacement ; gère la rotation du plan.
- **Contrôles de zoom** : Zoom avant/arrière, pourcentage, Ajuster à la page, **Rotation −90° / +90°**, **Réinitialiser la rotation**.
- **Auto-comptage (`AiCountPanel`)** : petit panneau violet, en haut au centre, visible quand l'outil « Compter par IA » est actif (voir §3.11).
- **Navigation de pages (`PageNavigator`)** : ‹ page précédente, champ « page / total » éditable, page suivante › (masqué s'il n'y a pas de document).
- **Saisie de mur (`MurInput`)** : petit dialogue de saisie clavier d'une dimension (format `PP-II-SS` ou `PPIISS`, par exemple « 20-06-08 » ou « 200608 » ; les flèches donnent la direction, Entrée valide, Échap annule).

### 2.7 Panneau droit (`RightPanel`)

Replié par défaut (bande verticale « Propriétés »). Sept blocs.

**A. Propriétés de la mesure** — trois modes :
- *Aucune sélection* : « Sélectionner une mesure pour voir ses propriétés ».
- *Multi-sélection* : « {n} mesures sélectionnées » + **Total de la sélection** (longueur / surface / comptage) + édition groupée (couleur, épaisseur du trait 1-10, groupe, transparence, facteur de pente, **marquer/retirer déductions**, produit associé, coller les propriétés, dupliquer, supprimer).
- *Sélection unique* : dupliquer ; copier/coller les propriétés ; **ordre d'affichage** (fond / reculer / avancer / dessus) ; **étiquette** (avec suggestions de variables BOM) ; contenu texte (note/bulle) ; groupe ; type ; **valeur** (avec avertissement « auto-intersectant » si le contour se croise) ; **points** (P1, P2...) ; **segments** (dimensions par arête + périmètre/longueur totale, avertissement « plan non calibré ») ; épaisseur du trait ; taille de police ; rotation + échelle (symbole) ; couleur (8 choix) ; transparence ; **facteur de pente (toiture)** (préréglages Plat, 2:12 ... 12:12 + personnalisé + calcul surface horizontale → réelle) ; **déduction** (case + choix de la mesure parente Brut/Net) ; bloc **Brut / − déductions / Net** ; **produit associé** (menu groupé par catégorie ; bloc coût : quantité brute éditable, net, perte %, prix, **coût total $**, avertissement « unité incompatible ») ; **main-d'œuvre** (corps de métier CCQ, heures × personnes → coût de main-d'œuvre) ; **TOTAL (matériaux + main-d'œuvre)**.

**B. Calibration** : Échelle (1 px = X unité), référence, pixels — ou « Non calibré - utilisez l'outil Calibrer ».

**C. Catalogue de produits** : compteur + **« Gérer Catalogue »**.

**D. Corps de métier CCQ** : compteur + **« Gérer Main-d'œuvre »**.

**E. Symboles architecturaux** : compteur + **« Gérer Symboles »**.

**F. Résumé des coûts** (si des mesures ont un produit) : liste par mesure (coût $), sous-totaux **Matériaux / Main-d'œuvre**, **Total $**, avertissement d'unités incompatibles, et **cinq boutons d'export** : Soumission PDF, CSV, Estimation HTML (ouvre un onglet), Télécharger HTML, DXF (AutoCAD).

**G. Document** : fichier + nombre de pages, ou « Aucun document chargé ».

### 2.8 Panneau BOM en direct (`BomEstimationPanel`)

Accosté à droite, **replié par défaut** (fine bande). Il montre l'**estimation en direct des assemblages** :

- « {n} assemblages actifs sur {total} », et les mesures **orphelines** (sans assemblage) signalées ;
- la liste **Assemblages actifs** (case à cocher, badge **auto** ou **manuel**, « Tout activer/désactiver » par section) et un bouton **« Reset auto »** ;
- un bloc **Données du chantier** (champs à saisir) ;
- un **Bordereau détaillé** (Section / Produit / Qté / Unité / Prix / Total) ;
- un **Cumulé tous assemblages** (produits uniques, avec le nombre d'assemblages qui les emploient) ;
- le **Total estimation projet** ;
- deux exports : **CSV Fournisseur** (détail + cumul) et **CSV Estimation** (feuille avec temps et coût).

> À retenir : la **« Soumission BOM automatique »** (bouton de la barre d'outils) et le **panneau BOM** ne partagent pas les valeurs manuelles. Les données saisies à la main dans ce panneau **ne sont pas** reprises par la soumission BOM automatique.

### 2.9 Barre de statut inférieure (`BottomBar`)

Affiche, de gauche à droite : l'**outil actif** ; les **coordonnées** du curseur (en unités réelles si calibré, sinon en pixels) ; la **valeur en direct** de la mesure en cours ; l'indicateur **SNAP** ; un compteur de presse-papiers ; « {n} mesures » ; la **bascule Impérial (ft) / Métrique (m) (Ctrl+U)** ; l'état **« Calibré ({unité}) »** ou **« Non calibré »** ; et, si le plan est calibré et contient des mesures, un lien **« Recalibrer »** (réarme l'outil Calibrer).

### 2.10 Modales et panneaux flottants

| Modale / panneau | Composant | Contenu principal |
|---|---|---|
| **Calibration** | `CalibrationModal` | « Longueur réelle de la référence tracée » (ex. « 3.048 ou 10-0-0 ou 160608 »), unité (m/cm/mm/ft/in), détection impériale automatique, case **« Recalculer les N mesures existantes à la nouvelle échelle »** (`rescaleExisting`, cochée par défaut). Calibration **par page**. |
| **Soumission** | `SoumissionModal` | Bandeau ambre d'avertissements (unités incompatibles / aires auto-intersectantes / composites à formule liés). **Section 1** — fiche client. **Section 2** — bascule de consolidation (Détaillé / Produit + calque / Produit seul) + case **« Inclure la main-d'œuvre »**, tableau par catégorie, sous-totaux, taxes dynamiques (tenant/devis, repli QC), **Total TTC**. **Section 3** — Annuler, Aperçu Soumission HTML, Appliquer au devis, Créer un devis. (Détail complet en §3.13.) |
| **Auto-comptage** | `AiCountPanel` | Panneau violet flottant (haut-centre). Sélecteur **Gabarit (gratuit) / Vision IA**, champ « Nom du comptage » ou « Élément à compter », curseur **Sensibilité** (mode gabarit), phases idle → analyzing → review. (Détail complet en §3.11.) |
| **Bibliothèque des métrés** | `MetreLibraryModal` | Recherche par nom/description/PDF, cartes (mesures, calques, pages, « lié à devis », horodatage), supprimer (confirmation), ouvrir. |
| **Nouveau métré / Renommer** | `SaveMetreModal` | Nom* + description. |
| **Catalogue de produits** | `ProductCatalog` | Onglets Produits / Ajouter / Import-Export ; recherche, catégories, badge « Composite », édition en ligne (Modifier / Supprimer / Composants), champ Perte %, création d'un **composite** ; import/export JSON, vider le catalogue. |
| **Éditeur de composite** | `CompositeEditor` | Mode d'affichage (détaillé/résumé), forçage du prix, **sous-produits** (produit + Qté/unité + **Formule**), **variables d'entrée (BOM inputs)** (nom en minuscules avec tirets bas, unité, valeur par défaut, description), validation. |
| **Corps de métier CCQ 2026** | `LaborCatalogPanel` | Métier, spécialité, secteur, **taux horaire** (convention CCQ 2025-2029, secteur ICI), nombre de personnes, productivité + unité, couleur ; import/export/réinitialiser. |
| **Symboles architecturaux** | `SymbolCatalogPanel` | Catégories, vues (Plan/Élévation/Droite), largeur/hauteur en pouces, ajout, import/export JSON, réinitialiser. |
| **Calculatrice Construction Master Pro** | `CalculatorPanel` | Saisie pieds-pouces-fractions, conversion d'unités, historique. |
| **Convertisseur de pente** | `SlopeConverterPanel` | Pente (x:12) ↔ Degrés ↔ Pourcentage + table de référence. |
| **Résumé multi-pages** | `SummaryPanel` | Sections Client / Projet / Métré (Take-Off), vues Par groupe / Par type / Par produit / Par page ; exports Soumission PDF / CSV / Estimation HTML (ouvrir/télécharger) / DXF. |

---

## 3. Workflows pas à pas

### 3.1 Créer ou ouvrir un métré

1. Onglet **Métré**. Si aucun métré n'est ouvert, la barre du haut propose **« Nouveau métré »** et **« Ouvrir »**.
2. **Nouveau métré** : saisissez un **nom** (obligatoire) et une description, puis **Créer**. Un projet vide est créé côté serveur (`POST /projects`).
3. **Ouvrir** : la **bibliothèque** liste vos métrés (recherche par nom/description/PDF). Cliquez une carte pour l'ouvrir ; vous pouvez aussi supprimer un métré (confirmation).

> Vous pouvez commencer à charger un plan avant même de nommer le métré : le PDF est mis en cache localement, puis **téléversé automatiquement** au premier enregistrement (`uploadCachedPdfForProject`).

### 3.2 Charger un plan

1. **Fichier → Ouvrir un plan (Ctrl+O)**, ou **glissez-déposez** un PDF/image dans la zone centrale.
2. Formats acceptés : PDF, PNG, JPG, JPEG, BMP, TIFF, WEBP. Taille maximale du PDF : **150 Mo** (`MAX_FILE_SIZE_MB`).
3. Le plan s'affiche ; naviguez entre les pages si le document comporte plusieurs pages.
4. Astuce : besoin d'une feuille vierge pour croquis ? **Fichier → Nouveau plan vierge (ARCH D 36"×24")**.

Contrôles serveur au téléversement : extension `.pdf`, taille ≤ 150 Mo (sinon erreur 413), fichier non vide, **signature `%PDF-`** vérifiée (sinon 400 + journal de sécurité), lecture par PyMuPDF (400 si corrompu), et **type MIME forcé à `application/pdf`** (protection contre les fichiers piégés « polyglottes »).

### 3.3 Calibrer l'échelle (indispensable avant de chiffrer)

1. Outil **Calibrer (K)**. Tracez une ligne sur une **cote connue** du plan (par exemple un mur coté à 40 pi, ou une barre d'échelle).
2. Dans la modale, saisissez la **longueur réelle** et l'**unité** (impérial `40-00-00` ou décimal `12.192`). La détection impériale affiche l'interprétation.
3. Laissez cochée **« Recalculer les N mesures existantes à la nouvelle échelle »** si vous corrigez une calibration après avoir déjà tracé (les valeurs sont reprojetées).
4. **Calibrer**. Le facteur d'échelle est enregistré **pour cette page** (`POST /documents/{id}/calibrate`). Répétez sur chaque page si les échelles diffèrent — la calibration de toutes les pages est renvoyée intégralement par le serveur (`GET /documents/{id}/calibrations`), pas seulement celle de la page active.

> Conseil : choisissez une cote **longue** et bien lisible. Une référence trop courte amplifie l'imprécision sur toutes les mesures de la page.

### 3.4 Tracer des mesures

- **Distance (D)** : deux clics.
- **Mur — mesure clavier (J)** : posez le premier point, choisissez la **direction** avec les flèches, saisissez la **longueur exacte** au clavier (pieds-pouces-seizièmes), puis **Entrée** pour poser le segment à la longueur voulue. Idéal quand la cote est écrite mais que le tracé à la souris manque de précision.
- **Surface (A) / Périmètre (P) / Polyligne (L)** : clics successifs ; double-clic ou **Entrée** pour fermer/terminer.
- **Rectangle (R)** : deux coins opposés. **Cercle (I)** : centre puis rayon.
- **Comptage (C)** : un clic par élément ; le total s'incrémente.
- **Angle (N)** : trois points. **Cotation (X)** : cote annotée.
- **Aides au tracé** : l'**accrochage (F3)** aimante le curseur aux **extrémités**, **milieux** et **intersections** des mesures voisines ; le **mode ortho (F8)** et la **grille (F7)** facilitent les tracés droits.

> Le module **avertit** si un contour de surface se croise (« polygone auto-intersectant ») : l'aire et le périmètre deviennent alors douteux. Retracez le contour proprement.

### 3.5 Organiser : calques et groupes

1. Panneau gauche → **Ajouter un calque** (par exemple « Murs sous-sol », « Plancher RDC »), choisissez la couleur.
2. Regroupez plusieurs calques en **groupes pliables** (« Créer un groupe », puis glissez-y les calques).
3. **Verrouillez** un calque (cadenas) pour éviter de le modifier par erreur ; **masquez-le** (œil) pour alléger l'affichage.
4. Glissez une mesure sur un autre calque pour la déplacer, ou utilisez les flèches ↑ / ↓ et « Déplacer vers... » dans la liste des mesures.

### 3.6 Associer un produit et chiffrer

1. Sélectionnez une mesure. Le panneau droit s'ouvre sur ses propriétés.
2. Choisissez un **produit** du catalogue (menu groupé par catégorie), ou créez-en un via **« Gérer Catalogue »**.
3. Au besoin : **quantité manuelle** (remplace la valeur calculée), **% de perte**, **facteur de pente** (toiture), **déductions** (voir §3.7), **main-d'œuvre** (voir §3.9).
4. Le **coût** s'affiche : `quantité nette × pente × (1 + perte %) × prix`, **après conversion vers l'unité de prix** du produit. Une alerte **« unité incompatible »** apparaît si la mesure ne peut pas être convertie (par exemple une aire liée à un prix au linéaire).

### 3.7 Déductions (ouvertures)

1. Tracez la mesure parente (par exemple la surface d'un mur).
2. Tracez l'**ouverture** (fenêtre, porte) comme une mesure distincte, puis cochez **Déduction** dans ses propriétés et choisissez la **mesure parente**.
3. La quantité **nette** de la parente = brut − somme des déductions (jamais négative). Le bloc **Brut / − déductions / Net** le montre.

### 3.8 Facteur de pente (toiture)

1. Sur une mesure de **surface** (ou de cercle), ouvrez **Facteur de pente**.
2. Choisissez un préréglage (**Plat**, **2:12** ... **12:12**) ou une pente personnalisée.
3. Le module convertit la **surface horizontale** (mesurée sur le plan) en **surface réelle de toiture** (inclinée). Le facteur entre dans le calcul du coût.

> Le **Convertisseur de pente** (barre d'outils) fait la correspondance Pente (x:12) ↔ Degrés ↔ Pourcentage et fournit une table de référence.

### 3.9 Main-d'œuvre (corps de métier CCQ)

1. Ouvrez **« Gérer Main-d'œuvre »** pour vérifier/compléter votre catalogue de corps de métier (métier, spécialité, secteur, **taux horaire**, nombre de personnes, productivité). Un bouton réinitialise aux **taux CCQ 2026** (convention CCQ 2025-2029, secteur ICI).
2. Sur une mesure, section **Main-d'œuvre** : choisissez le corps de métier, saisissez **Heures × Personnes**. Le **coût de main-d'œuvre** s'ajoute au coût matériaux pour donner le **TOTAL** de la ligne.

### 3.10 Assemblages paramétriques (BOM / composites)

Pour les ouvrages répétitifs (mur 2×4, plancher, etc.), au lieu d'associer un produit à chaque mesure, liez un **calque** à un **assemblage** qui se calcule tout seul.

1. **Gérer Catalogue → créer un produit composite** (assemblage). *Disponible uniquement en mode ERP connecté.*
2. Dans l'**Éditeur de composite** :
   - déclarez des **variables d'entrée** (par exemple `surface_mur`, `longueur_mur`, `nombre_coins`) avec leur **unité** et une valeur par défaut ; le nom doit être en minuscules avec tirets bas (`^[a-z][a-z0-9_]*$`) ;
   - ajoutez des **sous-produits**, chacun avec une **quantité par unité** ou une **formule** (par exemple `surface_mur / 32` pour des feuilles 4×8, ou `IF(surface_ss > 800, 3, 2)`).
3. **Liez un calque** à l'assemblage (icône lien BOM du calque). Les variables géométriques se remplissent **automatiquement** depuis les mesures du calque (badges « auto »), converties dans l'unité déclarée de la variable. Les autres variables se saisissent à la main.
4. Le **panneau BOM** affiche le bordereau (sections + cumulé par produit) et le **total du projet**. Exportez en **CSV Fournisseur** ou **CSV Estimation**.

> **Sécurité des formules.** Les formules sont évaluées par un **interpréteur dédié** (jamais `eval`), avec une liste blanche de fonctions — `CEIL, IF, MIN, MAX, ROUND, SUM, ABS, FLOOR` — et de caractères ; longueur ≤ 500 caractères ; chaque variable citée dans une formule **doit exister** dans les variables d'entrée, sinon la formule est refusée (évite le piège « la ligne donne toujours 0 »). Références : `_validate_formula`, `_check_formula_vars_or_raise` (`metre_pdf.py` / `metre_ai_tools.py`).

### 3.11 Auto-comptage — deux méthodes

L'outil **Compter par IA (U)** ouvre un petit panneau violet en haut au centre. En haut du panneau, un **sélecteur de mode** propose **« Gabarit (gratuit) »** et **« Vision IA »**. Choisissez selon votre besoin (voir le guide de décision plus bas). Dans les deux cas, les phases sont : **idle → analyzing → review** ; à la confirmation, **une** mesure de type comptage à N points est posée sur la page.

> **Prérequis commun : plan droit (0°).** Les deux méthodes exigent un plan **non pivoté**. Réinitialisez la rotation au besoin (message « Remettez le plan droit (0°) avant de compter »).

#### 3.11.A Comptage par gabarit (GRATUIT, sans IA)

Trouve tous les symboles **identiques** à un exemple que vous encadrez, par comparaison d'images (OpenCV). **Aucun crédit IA, 0 $** — c'est le mode à privilégier pour les symboles répétés d'un plan.

1. Sélectionnez le mode **« Gabarit (gratuit) »**.
2. (Facultatif) Saisissez un **« Nom du comptage »** (par exemple « détecteur de fumée »).
3. Réglez le curseur **Sensibilité** : vers **« large »** = plus permissif (trouve plus, mais risque de faux positifs) ; vers **« strict »** = plus sévère (moins de résultats, plus sûrs). Plage 0,5 à 0,9, **valeur par défaut 0,72**.
4. Consigne affichée : « **Encadrez UN exemple du symbole ; je trouve tous les identiques sur la page (gratuit, PDF).** » **Tracez un rectangle serré** autour d'un seul symbole.
5. Le module cherche les occurrences (« Recherche des occurrences… ») et affiche « {n} occurrence(s) détectée(s) » + une **confiance moyenne**.
6. **Confirmer** pose une mesure de comptage à N points ; **Nouvelle région** pour réessayer avec un autre gabarit ou une autre sensibilité.

**Conditions et limites du gabarit :**

- **PDF enregistré uniquement.** Le gabarit ne fonctionne pas sur une image : message « **Le comptage par gabarit ne fonctionne que sur les plans PDF. Utilisez la Vision IA pour une image.** » (le document doit avoir un identifiant serveur numérique **et** l'extension `.pdf`).
- **Symbole ni trop petit ni trop grand.** Un gabarit trop petit renvoie « **Le symbole est trop petit à cette taille de page pour un comptage fiable par gabarit.** » (zoomez d'abord). Une zone quasi vide/uniforme ou un gabarit trop grand ne renvoie aucun résultat.
- **Motifs identiques seulement.** C'est de la comparaison au pixel (avec 4 rotations : 0°, 90°, 180°, 270°) : les symboles doivent être **visuellement identiques**. Pour un élément dessiné de plusieurs manières, utilisez la vision IA.
- Si aucune occurrence : « Aucune occurrence trouvée. Encadrez le symbole plus serré, zoomez, ou déplacez le curseur vers « large ». »

*Sous le capot* : `POST /documents/{id}/detect-template` (`metre_pdf.py:4943`, cœur `_detect_template_sync:4789`). La page est rendue par **PyMuPDF** (rotation forcée à 0), puis **OpenCV `cv2.matchTemplate` (TM_CCOEFF_NORMED)** compare le gabarit sur 4 rotations, avec suppression des doublons voisins. **Aucun appel IA, aucun crédit, aucune écriture en base** ; la mesure est créée côté client à la confirmation. Deux analyses simultanées maximum sur le serveur, limite de **30 requêtes/minute** par adresse. Si OpenCV est indisponible : erreur 503.

#### 3.11.B Comptage par vision IA (PAYANT)

Compte des éléments **décrits en mots**, même s'ils ne sont pas dessinés de façon strictement identique. Utilise Claude Opus 4.8 en vision ; **consomme des crédits IA**.

1. Sélectionnez le mode **« Vision IA »**.
2. Saisissez l'**« Élément à compter »** (obligatoire — par exemple « prise électrique », « porte », « fenêtre »).
3. **Tracez un rectangle** autour de la zone à analyser.
4. L'IA renvoie « {n} occurrence(s) détectée(s) », une **confiance moyenne** en pourcentage et un **coût** en dollars (« coût {cost} $ »).
5. **Confirmer** crée **une** mesure de type comptage avec N points sur la page ; **Annuler** ou **Nouvelle région** pour recommencer.

**Conditions et limites de la vision :**

- **Crédits requis.** Si votre solde est nul : « Crédits IA épuisés » (l'appel est refusé **avant** tout coût).
- **Région raisonnable.** Une zone trop grande (image > 20 Mo) est refusée (« Région trop grande ») ; le service peut être « occupé » (deux analyses simultanées maximum).
- **Plan droit.** Comme le gabarit, la vision refuse un plan pivoté.

*Sous le capot* : `POST /metre/ai/detect` (`metre_ai_detect.py:157`), modèle Claude Opus 4.8, **sans écriture en base** (la mesure est créée côté client après votre confirmation). Facturation au **coût réel × 1,30** (plafond 2 $/appel), déduite **après succès seulement** ; positions renvoyées normalisées entre 0 et 1, plafonnées à 1 000 détections. Le point d'accès est **monté défensivement** : si le module IA est indisponible, il renvoie 404 sans empêcher le reste du Métré de fonctionner.

#### Guide de décision

| Situation | Méthode conseillée |
|---|---|
| Symboles **identiques** répétés sur un **PDF** (prises, luminaires, détecteurs...) | **Gabarit (gratuit)** — commencez toujours par là |
| Plan chargé en **image** (PNG/JPG), pas en PDF | **Vision IA** |
| Élément **dessiné de plusieurs façons** ou décrit sémantiquement | **Vision IA** |
| Vous voulez **zéro coût** | **Gabarit (gratuit)** |
| Le gabarit ne trouve rien même en position « large » | Réessayez en zoomant / cadrage plus serré, sinon **Vision IA** |

### 3.12 Assistant IA conversationnel

Un panneau de chat (à droite) où vous décrivez ce que vous voulez en français ; l'IA lit votre métré et propose des actions.

1. **Enregistrez d'abord le métré** : le bouton Assistant reste **grisé** tant que le document n'a pas d'identifiant serveur numérique.
2. Ouvrez l'assistant. Choisissez un **profil IA** (vos profils ou les profils système via `MetreProfileSelector` / « Mes profils ») et, au besoin, une **nouvelle conversation** (la liste des conversations est dans la barre latérale du panneau).
3. Écrivez votre demande (par exemple « liste les mesures de la page 2 », « crée un calque Murs RDC », « cherche un produit gypse 5/8 »). La réponse arrive **en continu** (streaming), avec l'affichage des outils exécutés et des coûts.
4. Les **8 outils de lecture** (lister mesures/calques, calibration, résumé, chercher produits, détails composite, composites actifs, projets passés) s'exécutent **automatiquement**.
5. Les **11 outils d'écriture** (créer un calque/une mesure, lier un composite, supprimer une mesure, et 7 actions de catalogue) affichent une **carte de confirmation** : rien n'est appliqué tant que vous ne cliquez pas **Confirmer** (confirmation **côté serveur**, non contournable). Vous pouvez **rejeter** une action.
6. Le pied de panneau affiche le **coût cumulé**, le **coût de la conversation** et un **compteur de messages {n}/200**.

Points d'attention :
- **Verrou de page** : si l'IA répond sur la page que vous aviez envoyée mais que vous avez navigué ailleurs, une bannière « Stopper et recadrer » vous prévient.
- **Une seule conversation active à la fois par document**, et **200 messages maximum** par conversation.
- **Un seul flux à la fois par conversation** : si un flux est déjà en cours, un second renvoie « Conversation déjà en cours de streaming » (verrou serveur).
- **Modification avant confirmation** : vous pouvez ajuster les paramètres des **4 outils de mesure** avant de confirmer, mais **pas ceux des 7 outils de catalogue** (créer/modifier/supprimer un produit, variables et lignes de composite) — pour ces derniers, on confirme tel quel ou on rejette.

*Sous le capot* : `POST /documents/{id}/assistant-chat` (`metre_ai_chat.py:1229`, diffusion continue SSE), modèle Claude Opus 4.8, conversations gérées par `metre_ai_conversations.py`, 19 outils dans `metre_ai_tools.py`. La boucle d'outils est plafonnée à **8 itérations** et le budget par message à **2 $** (au-delà : « budget dépassé »). Si vous fermez le panneau en cours de réponse (déconnexion) ou archivez la conversation, la facturation est **sautée** (pas de fuite de revenu).

### 3.13 Générer la soumission

1. Barre d'outils → **Générer une soumission** (mesures avec produit associé), ou **Générer une soumission BOM automatique** (calques liés aux composites). La modale Soumission s'ouvre.
2. Lisez le **bandeau ambre** d'avertissements (unités incompatibles, aires auto-intersectantes, composites à formule liés en direct) et corrigez au besoin.
3. **Section 1 — Fiche client** : Nom du projet (obligatoire), Client (entreprise / contact, ou saisie manuelle du nom), No de PO client, Date limite de soumission, Début prévu des travaux, Priorité (Normal / Haute / Urgente), Description.
4. **Section 2 — Lignes** : choisissez la **consolidation** (Détaillé = 1 ligne par mesure ; **Produit + calque** = recommandé ; Produit seul) et cochez **« Inclure la main-d'œuvre »** si voulu. Vérifiez le tableau (par catégorie : Description, Unité, Qté, Prix unit., Montant), les sous-totaux Matériaux / Main-d'œuvre, les **taxes** (celles du tenant ou du devis ; repli **TPS 5 % / TVQ 9,975 %**) et le **Total TTC**.
5. **Section 3 — Actions** : trois issues :
   - **Aperçu Soumission HTML** (si un devis est connecté) : prévisualise le rendu ;
   - **Appliquer au devis** : ajoute les lignes au devis connecté ;
   - **Créer un devis** : génère un nouveau devis à partir du métré.

> **Ce qui traverse vers le devis.** Les lignes de catégories **Administration / Contingences / Profit** sont **filtrées** au passage vers le devis (pour éviter un double-comptage : le devis applique ses propres majorations), et seules les lignes à **quantité > 0** sont envoyées (le serveur du devis refuse une quantité nulle, `gt=0`). Le montant est **recalculé côté serveur** (`montant = arrondi(quantité × prix)`), et l'aperçu emploie déjà ces mêmes valeurs arrondies pour éviter tout écart (`MetrePdf.tsx:1037-1057`, `1479-1560`).

> **Note technique.** Le point d'accès serveur `POST /projects/{id}/import-to-devis/{devis_id}` existe mais est **gelé** (il renvoie 503 sauf activation explicite par la variable d'environnement `METRE_IMPORT_TO_DEVIS_ENABLED`), parce qu'il ne reproduit pas fidèlement les composites et les déductions. Le chemin utilisé par l'interface est la **modale Soumission** décrite ci-dessus (`onApplyToDevis` / `onCreateDevis`), pas ce point d'accès.

### 3.14 Exporter

| Sortie | Où | Contenu |
|---|---|---|
| **CSV Fournisseur** | Panneau BOM | Bordereau matériaux (détail + cumulé) pour la commande. |
| **CSV Estimation** | Panneau BOM | Feuille d'estimation (temps + coût). |
| **CSV** (produit / résumé) | Panneau droit, Résumé multi-pages | Tableaux par produit / par mesure (colonnes détaillées, voir §4.6). |
| **Soumission PDF** | Panneau droit, Résumé | Soumission mise en page (matériaux, main-d'œuvre, taxes). |
| **Estimation HTML** | Panneau droit, Résumé | Soumission HTML (ouvrir dans un onglet ou télécharger). Le pied de page porte le téléphone émetteur **« Tél: (514) 820-1972 »** (`estimationHtml.ts:1202`). |
| **PNG** | Barre d'outils | 3 fichiers : plan annoté + détail produits + détail BOM (les fichiers vides sont ignorés). |
| **DXF (AutoCAD)** | Panneau droit, Résumé | Géométrie des mesures pour un logiciel de CAO. |

> **CSV et Excel.** Les fichiers sont en **UTF-8 avec BOM** et utilisent le **séparateur de la langue du document** : **point-virgule (`;`) en français** (Excel Québec), **virgule (`,`) en anglais**. Chaque cellule est encadrée de guillemets et l'injection de formule est neutralisée (un nom commençant par `=`, `+`, `-` ou `@` est préfixé d'une apostrophe). L'**impression** se fait via le PDF de soumission.

### 3.15 Sauvegarder, renommer, rouvrir

- **Enregistrement** : un métré ouvert est sauvegardé automatiquement (autosauvegarde locale ~1 s d'anti-rebond + synchronisation serveur à chaque modification en mode ERP — chaque mesure, calque et calibration est synchronisé). L'indicateur de la barre du haut montre l'état (« Sauvegardé il y a N s », ou « Sauvegarde échouée » en rouge).
- **Premier enregistrement** : le PDF mis en cache est **téléversé** automatiquement (BYTEA côté serveur).
- **Renommer** : crayon de la barre du haut.
- **Rouvrir / supprimer** : bouton **« Ouvrir »** → bibliothèque des métrés.

### 3.16 Recalibrer

1. Barre du bas → **« Recalibrer »** (visible si la page est calibrée et contient des mesures), ou relancez l'outil **Calibrer** sur une page déjà calibrée.
2. Tracez la nouvelle référence, saisissez la vraie longueur, laissez cochée **« Recalculer les mesures existantes »**.
3. Le serveur applique une **recalibration atomique** : la calibration et les **valeurs des mesures de cette page uniquement** sont mises à jour dans une seule transaction (les autres pages ne bougent pas). Plafond : **5 000 mesures** par recalibration (`POST /documents/{id}/recalibrate`).

---

## 4. Référence

### 4.1 Raccourcis clavier

| Touche | Action | | Touche | Action |
|---|---|---|---|---|
| **K** | Calibrer | | **X** | Cotation |
| **V** | Sélection | | **H** | Déplacer / main |
| **D** | Distance | | **J** | Mur (mesure clavier) |
| **A** | Surface | | **T** | Texte |
| **R** | Rectangle *(ou Pivoter 45° si sélection)* | | **W** | Flèche |
| **P** | Périmètre | | **Q** | Nuage de révision |
| **L** | Polyligne | | **F** | Main levée |
| **N** | Angle | | **G** | Surligneur |
| **C** | Comptage | | **E** | Note |
| **U** | Compter par IA (gabarit ou vision) | | **B** | Bulle de texte |
| **I** | Cercle | | **M** / **Maj+M** | Copie miroir horizontal / vertical |
| **Ctrl+O** | Ouvrir un plan | | **Ctrl+D** | Dupliquer la sélection |
| **Ctrl+Z / Ctrl+Y** | Annuler / Rétablir | | **Ctrl+U** | Bascule impérial / métrique |
| **Ctrl+Maj+C / V** | Copier / coller les propriétés | | **F3 / F8 / F7** | Accrochage / Ortho / Grille |
| **F11** | Plein écran (Échap n'en sort pas) | | | |

### 4.2 Points d'accès backend (préfixe `/api/erp/v1/metre`, 52 au total)

Tous protégés par jeton JWT et isolés par schéma d'entreprise ; **aucun rôle particulier requis** au-delà de l'appartenance au tenant.

| Groupe | Méthode + route | Rôle |
|---|---|---|
| **Projets** | `POST /projects` · `GET /projects` · `GET /projects/{id}` · `PUT /projects/{id}` · `DELETE /projects/{id}` | CRUD des métrés |
| | `GET /metres-library` | Vue agrégée de la bibliothèque (mesures, calques, pages, auteur) |
| **Documents** | `POST /projects/{id}/documents/upload` | Téléverser un PDF (signature vérifiée, BYTEA, ≤ 150 Mo) |
| | `GET /projects/{id}/documents` · `GET /documents/{id}` | Lister / métadonnées |
| | `GET /documents/{id}/file` | Télécharger le PDF (Content-Disposition assaini) |
| | `GET /documents/{id}/page/{page}?zoom` | Rendu PNG d'une page (cache LRU, zoom 0,1–10) |
| | `DELETE /documents/{id}` | Supprimer un document |
| **Calibrations** | `POST /documents/{id}/calibrate` | Définir l'échelle d'une page |
| | `POST /documents/{id}/recalibrate` | Recalibration atomique (≤ 5 000 mesures) |
| | `GET /documents/{id}/calibrations` | Toutes les pages calibrées (alimente le contrôle client) |
| | `GET /documents/{id}/calibration/{page}` · `DELETE /documents/{id}/calibration/{page}` | Lire / supprimer une calibration |
| **Mesures** | `GET /documents/{id}/measurements?page&layer_id` · `POST /documents/{id}/measurements` | Lister / créer |
| | `GET /measurements/{id}` · `PUT /measurements/{id}` · `DELETE /measurements/{id}` | Lire / modifier / supprimer |
| | `GET /documents/{id}/measurements/export?format` | Export CSV (BOM) ou JSON |
| **Calques** | `GET/POST /documents/{id}/layers` · `GET/PUT/DELETE /layers/{id}` | CRUD des calques |
| **Groupes** | `GET/POST /documents/{id}/layer-groups` · `PUT/DELETE /layer-groups/{id}` | CRUD des groupes de calques |
| **Produits** | `GET /products?category` · `POST /products` · `POST /products/bulk-import` | Lister / créer / import en lot |
| | `GET/PUT/DELETE /products/{id}` | Lire / modifier / supprimer |
| **Composants** | `GET/POST /products/{id}/components` · `PUT/DELETE /products/{id}/components/{cid}` | Sous-produits d'un assemblage |
| **Accrochage** | `POST /documents/{id}/snap-points` | Détection OpenCV (Hough/Harris, région ≤ 4 000 px) — **gratuit** |
| **Gabarit** | `POST /documents/{id}/detect-template` | **Comptage par gabarit — OpenCV, GRATUIT** (30 req./min) |
| **Résumé** | `GET /documents/{id}/summary` | Totaux par type / calque |
| **Devis (gelé)** | `POST /projects/{id}/import-to-devis/{devis_id}` | Import serveur — **désactivé (503)** par défaut |
| **Assistant** | `POST /documents/{id}/assistant-chat` | Chat IA en diffusion continue (SSE) — **payant** |
| **Conversations** | `GET /documents/{id}/conversations` · `GET/DELETE /conversations/{id}` | Lister / détail / archiver |
| | `POST /conversations/{id}/tool-executions/{eid}/confirm` | Confirmer/rejeter une action d'écriture |
| **Auto-comptage vision** | `POST /metre/ai/detect` | Comptage par vision d'une zone (sans écriture) — **payant** (6 req./min) |

*(46 points d'accès dans `metre_pdf.py` + 1 vision + 1 chat + 4 conversations = 52.)*

### 4.3 Outils de l'assistant IA (19)

**8 outils de lecture** (exécutés automatiquement) : `lister_mesures`, `lister_calques`, `obtenir_calibration`, `obtenir_summary`, `chercher_produits_bom`, `obtenir_composite_details`, `lister_composites_actifs`, `lister_projets_passes` (ce dernier filtré sur vos propres projets, sans exposer `devis_id`/`company_id`).

**11 outils d'écriture** (confirmation obligatoire, `TOOLS_REQUIRING_CONFIRMATION`, `metre_ai_tools.py`) :
- Métré (4) : `creer_calque`, `creer_mesure`, `lier_composite_calque`, `supprimer_mesure`.
- Catalogue (7) : `creer_produit`, `modifier_produit`, `supprimer_produit`, `definir_variables_composite`, `ajouter_ligne_composite`, `modifier_ligne_composite`, `supprimer_ligne_composite`.

Chaque écriture suit le patron **aperçu → confirmation serveur → exécution** : au moment de confirmer, le serveur revérifie les crédits **avant** tout effet, revalide les références (message « Les références ont changé » si elles ont bougé), pose un verrou anti-double-clic (un second clic voit l'action « déjà traitée »), puis exécute et enregistre le résultat.

### 4.4 Calculs importants (chemin de l'argent)

**Coût d'une ligne de mesure** : `quantité nette × facteur de pente × (1 + perte %) × prix`, où la quantité est d'abord **convertie** vers l'unité de prix du produit.

**Conversion d'unité (facteur facturable)** — `_billable_factor` (`metre_pdf.py:3900`, en parité avec `utils/costing.ts`) :

| Unité de mesure | 1 unité en mètres | | Dimension du prix | Comportement |
|---|---|---|---|---|
| mm | 0,001 | | linéaire → linéaire | facteur = conversion |
| cm | 0,01 | | aire → aire | facteur = conversion² |
| m | 1 | | comptage (unité, feuille, sac, boîte) | pas de conversion |
| ft (pi) | 0,3048 | | **croisée** (aire tarifée au linéaire, volume) | facteur = 1 + **« incompatible »** |
| in (po) | 0,0254 | | | |
| yd (vg) | 0,9144 | | | |

Ce mécanisme corrige la sous-facturation de ×10,76 (m² → pi²). En flux 100 % impérial (le cas québécois usuel), aucune conversion n'a lieu (identité).

**Coût IA** (vision + chat) : Opus 4.8 = 5 $/M jetons en entrée, 25 $/M en sortie, 10 $/M en écriture de cache, 0,50 $/M en lecture de cache, le tout **× 1,30** (marge), plafonné à **2 $ par message/appel**. Parité stricte avec l'Estimation IA (commit `32807a9b`).

**Résumé des effets d'argent :**

| Point d'accès | Coût | Débit |
|---|---|---|
| `POST /metre/ai/detect` (vision) | coût réel Claude × 1,30, plafond 2 $ | après succès seul (402 si solde nul) |
| `POST .../assistant-chat` (chat) | coût réel Opus × 1,30, budget 2 $/message | 1 fois après la boucle (sauté si déconnexion/archivage) |
| `POST .../confirm` (action acceptée) | **0,001 $ fixe** | après exécution (meilleur effort, crédit revérifié avant) |
| `POST .../detect-template` (gabarit) | **0 $** (OpenCV) | aucun |
| `POST .../snap-points` (accrochage) | **0 $** (OpenCV) | aucun |
| `POST .../import-to-devis` | s.o. | écrit `devis_lignes` (gelé par défaut, 503) |

### 4.5 Bornes, validations et limites

| Élément | Valeur / règle |
|---|---|
| Taille PDF | ≤ **150 Mo**, signature `%PDF-` obligatoire, MIME forcé `application/pdf` |
| `metadata_json` d'une mesure | ≤ **64 Ko** |
| `composite_inputs` d'un calque | ≤ **16 Ko** |
| Recalibration | ≤ **5 000 mesures** par page |
| Région d'accrochage (snap) | ≤ **4 000 px** |
| Formule de composite | ≤ **500 caractères**, fonctions en liste blanche, variables devant exister |
| **Comptage par gabarit** | **PDF enregistré uniquement**, plan droit (0°), symbole ≥ ~10 px, sensibilité 0,5–0,9 (défaut **0,72**) ; **≤ 30 req./min** ; **gratuit** |
| **Auto-comptage vision** | plan droit (0°) obligatoire ; ≤ 1 000 détections ; requête ≤ 20 Mo ; **≤ 6 req./min** ; **payant** |
| Points d'une mesure IA | ≤ **1 000** |
| Assistant : messages | ≤ **200** par conversation ; une seule conversation active par document |
| Assistant : boucle d'outils | ≤ **8 itérations** ; budget ≤ 2 $/message ; un seul flux à la fois (verrou 409) |
| Assistant : débit de requêtes | **pas de plafond dédié** ; garde-fous réels = budget 2 $/message + verrou de flux |
| Unités calibrables | `m`, `cm`, `mm`, `ft`, `in` |
| Plein écran | affichage seulement (Échap n'en sort pas) |

Limites de fond à connaître :
- **Calibration par page** : un plan à plusieurs échelles doit être calibré page par page.
- **Aire isotrope** : l'aire suppose la même échelle en X et en Y ; un plan scanné et étiré peut fausser les surfaces.
- **Contour simple** : un polygone auto-intersectant donne une aire trompeuse ; le module **avertit** mais ne corrige pas.
- **Rotation** : les deux auto-comptages exigent le plan droit ; la mesure manuelle, elle, gère le plan pivoté.
- **Gabarit vs sémantique** : le gabarit ne reconnaît que des motifs identiques ; la sémantique (« compte les portes ») relève de la vision IA.

### 4.6 Colonnes du CSV détaillé

Le CSV « résumé/produit » porte les colonnes : **Page, Groupe, Produit, Catégorie, Mesure, Type, Déduction, Qté brute, Qté nette, Perte %, Qté avec perte, Unité prix, Prix unitaire $, Coût total $**, plus une ligne **TOTAL**. Séparateur `;` (français) ou `,` (anglais), UTF-8 avec BOM, cellules protégées contre l'injection de formule.

### 4.7 Modèle de données (12 tables par entreprise)

| Table | Contenu |
|---|---|
| `metre_projects` | Le métré (nom, description, `company_id`, `devis_id`, auteur). |
| `metre_documents` | Le plan : **PDF en BYTEA** + `mime_type`, nombre de pages, taille. |
| `metre_calibrations` | Échelle par page — contrainte d'unicité (document, page). |
| `metre_layers` | Calques (couleur, visibilité, verrou, `composite_id`, `composite_inputs`, `group_id`, ordre). |
| `metre_layer_groups` | Groupes de calques (nom, ordre, replié). |
| `metre_measurements` | Mesures (`points` JSONB, `metadata_json` JSONB, type, valeur, unité, **quantité (sans valeur par défaut — voir note)**, produit, déduction, pente, main-d'œuvre ; liens `ai_detection_id`, `devis_ligne_id`). |
| `metre_products` | Catalogue (prix, unité, perte %, composite, `bom_inputs`, champs main-d'œuvre, section). Contrainte d'unicité (nom, catégorie). |
| `metre_product_components` | Sous-produits d'un assemblage (quantité ou formule). Contrainte d'unicité (parent, enfant). |
| `metre_ai_detections` | Traces d'auto-comptage IA (jetons, coût, statut). |
| `metre_ai_conversations` | Conversations de l'assistant (une par document + utilisateur). |
| `metre_ai_messages` | Messages (rôle, blocs de contenu). |
| `metre_ai_tool_executions` | Actions d'écriture de l'assistant (statut : en attente / confirmé / rejeté / exécuté / échoué). |

Ces tables sont créées **à la demande** au premier usage (`_ensure_tables`, `metre_pdf.py:1252`), pas à la création du compte. La source de vérité du schéma au signup est `metre_schema_sql.py` (racine du dépôt).

> **Note sur la quantité.** La colonne `metre_measurements.quantity` n'a **pas de valeur par défaut** : laissée vide (NULL), elle **retombe sur la valeur géométrique** calculée (`value`). Un chiffre saisi (y compris `0`) est un **remplacement explicite** respecté tel quel. Seul l'outil IA `creer_mesure` conserve un défaut interne à 1.

### 4.8 Messages et états courants

| Situation | Message |
|---|---|
| Aucun métré ouvert | « Aucun métré ouvert. Créez ou ouvrez un métré... » |
| Sauvegarde en échec | « Sauvegarde échouée » (badge rouge cliquable) |
| Mesure sans produit | « Sans produit » (orange) / badge « {n} à corriger » |
| Produit supprimé du catalogue | « Produit introuvable » |
| Contour qui se croise | avertissement « auto-intersectant » sur la valeur |
| Plan non calibré | « Non calibré - utilisez l'outil Calibrer » |
| Unité mesure ≠ unité prix | « unité incompatible » (quantité non convertie) |
| Assistant indisponible | bouton grisé tant que le PDF n'est pas enregistré |
| Auto-comptage sur plan pivoté | « Remettez le plan droit (0°) avant de compter » |
| Gabarit sur une image | « Le comptage par gabarit ne fonctionne que sur les plans PDF. Utilisez la Vision IA pour une image. » |
| Gabarit trop petit | « Le symbole est trop petit à cette taille de page pour un comptage fiable par gabarit. » |
| Gabarit sans résultat | « Aucune occurrence trouvée. Encadrez le symbole plus serré, zoomez, ou déplacez le curseur vers « large ». » |
| Crédits IA épuisés | « Crédits IA épuisés » |

---

## 5. Intégrations et FAQ

### 5.1 Module 07 — Soumissions (Devis)

Le Métré **vit dans** la page Soumissions et **l'alimente**. Depuis la modale Soumission : **« Appliquer au devis »** ajoute les lignes au devis connecté, **« Créer un devis »** en génère un nouveau. Le lien est mémorisé (`metre_projects.devis_id`). Les catégories Administration / Contingences / Profit sont filtrées au passage (le devis applique ses propres majorations) et le montant de chaque ligne est recalculé côté serveur.

### 5.2 Catalogues partagés

Les **produits** et la **main-d'œuvre** (corps de métier CCQ) sont partagés au niveau de votre entreprise. Un produit créé dans le Métré sert aux assemblages et aux soumissions ; les taux CCQ 2026 (convention CCQ 2025-2029, secteur ICI) sont fournis par défaut et réinitialisables.

### 5.3 Module 25 — DAO / Modélisation 3D

Module **distinct**. Le Métré travaille **sur un plan PDF (2D)** pour la prise de quantités ; le DAO **construit un modèle 3D** (murs, toits, mobilier) et produit son propre métré 3D. Les deux partagent des utilitaires (saisie impériale, cotation) mais n'ont ni la même page ni les mêmes tables.

### 5.4 Module 24 — Assistant IA (général)

L'assistant **Métré** est **dédié au métré** (outils typés sur les mesures et les catalogues) et distinct de l'Assistant IA général de l'ERP. Il facture des **crédits IA** à votre entreprise et exige une **confirmation** pour toute écriture.

### 5.5 Module 30 — Configuration

Les **taxes** appliquées à la soumission proviennent de la configuration de votre entreprise (ou du devis), avec repli **TPS 5 % / TVQ 9,975 %**. La **langue** du compte détermine le **séparateur CSV** (`;` en français, `,` en anglais).

### 5.6 FAQ

- **Je ne trouve pas « Métré » dans le menu.** C'est normal : ce n'est pas une entrée de menu ni une adresse `/metre`. Allez dans **Ventes → Soumissions**, puis onglet **« Métré »**.
- **Quelle méthode de comptage choisir ?** Commencez **toujours par le gabarit (gratuit)** si votre plan est un PDF et que le symbole se répète à l'identique. Passez à la **vision IA (payante)** pour une image, pour un élément dessiné de plusieurs façons, ou si le gabarit ne trouve rien même en position « large ».
- **Le comptage par gabarit dit « ne fonctionne que sur les plans PDF ».** Vous avez chargé une **image** (PNG/JPG). Le gabarit exige un **PDF enregistré**. Utilisez la **vision IA** pour une image.
- **Le gabarit ne trouve aucune occurrence.** Encadrez le symbole **plus serré** (un seul motif, sans texte autour), **zoomez** d'abord, ou déplacez le curseur **Sensibilité** vers « large ». Vérifiez aussi que le plan est **droit (0°)**.
- **Le bouton Assistant IA est grisé.** Enregistrez d'abord le métré (nommez-le et laissez le PDF se téléverser). L'assistant exige un document déjà persisté côté serveur.
- **Ma quantité semble 10× trop basse.** Vérifiez l'accord **unité de calibration ↔ unité de prix**. Un plan en **mètres** avec un produit au **pi²** est converti automatiquement ; l'alerte « unité incompatible » signale les cas réellement non convertibles (aire liée à un prix linéaire, volume).
- **Ma quantité affiche 0 ou une valeur étrange après saisie manuelle.** Le champ quantité **remplace** la valeur calculée. Videz-le pour **retomber sur la valeur géométrique** du tracé.
- **Mes accents sont incorrects dans le CSV (`boîte` → `boÃ®te`).** Les CSV sont en UTF-8 avec BOM et n'emploient pas de directive `sep=` (qui cassait l'encodage sous Excel). Ré-ouvrez le fichier ré-exporté ; en Excel anglais, passez la langue du compte à l'anglais pour obtenir un séparateur virgule.
- **Mes colonnes CSV sont décalées ou affichent `#NAME?`.** Chaque cellule est encadrée de guillemets et les valeurs commençant par `=`, `+`, `-` ou `@` sont neutralisées : ré-exportez.
- **L'auto-comptage refuse de compter.** Le plan doit être **droit (0°)** : réinitialisez la rotation. Pour la vision, vérifiez vos crédits IA et réduisez la taille de la zone si elle est trop grande.
- **L'IA a proposé une action mais rien ne s'est produit.** Les actions d'écriture exigent votre **confirmation** dans la carte affichée ; tant que vous ne confirmez pas (ou si vous rejetez), rien n'est appliqué.
- **Je veux modifier un paramètre avant de confirmer une création de produit.** Ce n'est pas possible pour les 7 actions de **catalogue** (produit, variables/lignes de composite) : confirmez tel quel ou rejetez, puis redemandez avec les bons paramètres. Les 4 actions de **mesure** (calque, mesure, liaison composite, suppression) acceptent, elles, un ajustement avant confirmation.
- **Le plein écran ne se ferme pas avec Échap.** Le plein écran est un affichage : rappuyez sur **F11** (ou le bouton). Échap sert à annuler l'outil ou à désélectionner.
- **La « Soumission BOM automatique » ignore mes valeurs saisies dans le panneau BOM.** C'est voulu : ce traitement ne reprend pas les valeurs manuelles du panneau et ne génère pas de lignes de main-d'œuvre. Utilisez la soumission standard (« Générer une soumission ») si vous avez ajusté des valeurs à la main.
- **Puis-je importer directement le métré dans un devis sans passer par la modale ?** Non : le point d'accès serveur d'import direct est **gelé** (503). Passez toujours par la modale Soumission.
- **Un collègue non-administrateur peut-il modifier mes métrés ?** Oui : le module ne restreint pas par rôle. Tout membre authentifié de votre entreprise a le plein contrôle (les données restent isolées des autres entreprises).

---

## 6. Récapitulatif

- **Accès** : **Ventes → Soumissions → onglet « Métré »** (pas de route `/metre`, pas d'entrée de menu). Le module se charge à la demande et reçoit le devis courant.
- **But** : prise de quantités sur PDF/image → chiffrage → soumission et bordereaux, réinjectables dans un devis.
- **Étapes clés** : ouvrir/créer un métré → charger le plan → **calibrer (par page)** → tracer → associer produits / assemblages → soumission → exporter ou envoyer au devis. **Toujours calibrer avant de chiffrer.**
- **Mesures** : Distance, Surface, Rectangle, Périmètre, Polyligne, Angle, Comptage, **Compter par IA**, Cercle, Cotation, **Mur (clavier)** + annotations et symboles.
- **Auto-comptage — DEUX méthodes** : **Gabarit (gratuit, OpenCV)** pour les symboles identiques d'un PDF droit ; **Vision IA (payante, Opus 4.8)** pour une image, un élément sémantique ou en repli. Les deux exigent un plan droit (0°).
- **Unités** : m/cm/mm/pi/po ; saisie impériale 1/16" ; **conversion automatique** vers l'unité de prix (corrige le piège ×10,76 m²↔pi²) ; alerte « unité incompatible ».
- **Chiffrage** : `quantité nette × pente × (1 + perte) × prix` ; déductions ; main-d'œuvre CCQ 2026 ; taxes QC (TPS 5 % / TVQ 9,975 % en repli).
- **BOM** : assemblages paramétriques (variables + formules sûres, sans `eval`) alimentés automatiquement par les calques ; disponibles **en mode ERP seulement**.
- **IA** : **assistant conversationnel** (Opus 4.8) + **vision** (Opus 4.8), écritures **confirmées côté serveur**, facturés en crédits (coût réel × 1,30, plafond 2 $, parité Estimation IA). Le **gabarit** et l'**accrochage** sont gratuits (OpenCV).
- **Exports** : CSV Fournisseur / Estimation (UTF-8 + séparateur selon la langue), PDF, HTML (pied « Tél: (514) 820-1972 »), PNG (3 fichiers), DXF.
- **Persistance** : PDF en **BYTEA** ; autosauvegarde ~1 s + synchronisation serveur ; bibliothèque de métrés ; **12 tables `metre_*`** créées à la demande. **52 points d'accès** (46 dans `metre_pdf.py` + 1 vision + 1 chat + 4 conversations).
- **Permissions** : **aucun rôle requis** au-delà de l'appartenance à l'entreprise ; isolation stricte par entreprise ; mode consultation hérité du compte.
- **À éviter** : oublier de calibrer ; lier une aire à un prix linéaire ; compter (gabarit ou vision) sur un plan pivoté ; utiliser le gabarit sur une image ; attendre l'assistant IA avant d'avoir enregistré le métré.

---

> Manuel rédigé d'après le code source réel (révision 2026-07, worktree autoritaire). **Fichiers vérifiés** : `frontend/src/components/metre-pdf/` (90 fichiers ; `MetrePdf.tsx`, `store.ts`, `useAiCount.ts`, `components/MeasurementCanvas.tsx`, `components/PDFViewer.tsx`, `components/TopToolbar.tsx`, `components/LeftPanel.tsx`, `components/RightPanel.tsx`, `components/BomEstimationPanel.tsx`, `components/BottomBar.tsx`, `components/AiCountPanel.tsx`, `components/MetreAssistantPanel.tsx`, `api.ts`, `api/metreAi.ts`, `utils/canvasRotation.ts`, `utils/costing.ts`, `utils/estimationHtml.ts`, `i18n/locales/fr/metrePdf.json`), `frontend/src/pages/DevisPage.tsx` (hôte, onglet `metre-pdf`) ; `backend/routers/metre_pdf.py` (5 055 lignes, 46 points d'accès), `metre_ai_chat.py` (2 154 lignes, 1), `metre_ai_conversations.py` (1 075 lignes, 4), `metre_ai_detect.py` (291 lignes, 1), `metre_ai_tools.py` (2 625 lignes, 19 outils). **Manuels liés** : **07 — Soumissions** (destination des lignes), **25 — DAO / Modélisation 3D** (modélisation, module distinct), **24 — Assistant IA** (assistant général), **30 — Configuration** (taxes, langue).
