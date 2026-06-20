# Module 30 — Métré (prise de quantités sur PDF)

> **Version** : 1.0 (rédigé contre le code source réel)
> **Code de référence** : `frontend/src/components/metre-pdf/` (≈ 40 000 lignes TS/TSX : `MetrePdf.tsx`, `store.ts`, `components/MeasurementCanvas.tsx`, `components/PDFViewer.tsx`, panneaux, `utils/`), `backend/routers/metre_pdf.py` (≈ 4 500 lignes, persistance), `backend/routers/metre_ai_chat.py` + `metre_ai_tools.py` + `metre_ai_conversations.py` (assistant IA)
> **Tables PostgreSQL (par tenant)** : `metre_projects`, `metre_documents` (PDF en BYTEA), `metre_calibrations`, `metre_layers`, `metre_layer_groups`, `metre_measurements` (JSONB `points` / `metadata_json`), `metre_products`, `metre_product_components`, `metre_ai_conversations`, `metre_ai_messages`, `metre_ai_tool_executions`
> **Cadrage** : ce module est un **outil de prise de quantités (« takeoff ») sur plans PDF**. On charge un plan, on **calibre l'échelle**, on **trace des mesures** (longueurs, surfaces, périmètres, comptages), on les **associe à des produits** (matériaux + main-d'œuvre) ou à des **assemblages paramétriques (BOM)**, puis on **génère une soumission** ou des **bordereaux** que l'on exporte (CSV, PDF, HTML) ou que l'on **renvoie directement dans un devis**. Un **assistant IA** peut lire le métré et proposer des actions. Il **n'est pas** un logiciel de DAO/modélisation 3D (voir le module **Modélisation 3D / DAO**), il **ne fait pas** de reconnaissance automatique complète du plan (l'aide à la détection se limite à l'accrochage et aux points d'accrochage OpenCV), et il **ne remplace pas** le module Devis (il l'alimente).

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas-à-pas](#3-workflows-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récap one-pager](#6-récap-one-pager)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Transformer un **plan PDF** (ou une image) en **quantités chiffrées** : l'utilisateur trace ses mesures par-dessus le plan à l'échelle, le module calcule les longueurs et surfaces réelles, les multiplie par les prix des produits, applique pertes/déductions/main-d'œuvre, et produit une **soumission prête à envoyer** ou un **bordereau d'achat** pour le fournisseur.

### 1.2 Concepts clés

- **Document** : le PDF (ou l'image) chargé. Stocké en base en **BYTEA** (survit aux redéploiements). Multi-pages supporté.
- **Calibration** : le lien entre les pixels du plan et les unités réelles. **Par page** : on trace une ligne sur une cote connue (ex. un mur de 40 pi) et on saisit sa longueur réelle. Le module en déduit le **facteur d'échelle** (unité réelle par pixel).
- **Mesure** : un tracé typé (distance, surface, périmètre, comptage, etc.) avec une **valeur** calculée dans l'**unité de calibration** de sa page.
- **Calque** : regroupe des mesures (couleur, visibilité, verrouillage). Peut être **lié à un assemblage (composite)** pour alimenter un BOM.
- **Groupe de calques** : section pliable regroupant plusieurs calques.
- **Produit** : un article du **catalogue** (prix, unité de prix, % de perte). Peut être **simple** ou **composite** (assemblage de sous-produits).
- **Assemblage / Composite (BOM)** : un produit contenant des **sous-produits** dont la quantité vient d'une **formule paramétrique** évaluée sur des **variables** (alimentées par les mesures).
- **Déduction** : une mesure « en soustraction » (ex. une ouverture) rattachée à une mesure parente ; la quantité **nette** = parente − déductions.
- **Soumission / Bordereau** : la sortie chiffrée (matériaux + main-d'œuvre + taxes), exportable ou convertible en devis.

### 1.3 Types de mesures

| Type | Rôle | Valeur calculée |
|---|---|---|
| **Distance** | Longueur d'un segment (2 points) | longueur (unité linéaire) |
| **Mur** | Segment avec saisie clavier de longueur exacte | longueur |
| **Polyligne** | Suite de segments ouverte | somme des longueurs |
| **Périmètre** | Contour fermé | longueur du contour |
| **Surface** | Aire d'un polygone (formule du lacet) | aire (unité²) |
| **Rectangle** | Aire d'un rectangle (2 coins) | aire |
| **Cercle** | Aire d'un disque (centre + rayon) | aire (π·r²) |
| **Comptage** | Nombre d'éléments (clic par élément) | nombre |
| **Angle** | Angle entre 3 points | degrés |
| **Cotation** | Cote architecturale annotée | longueur |
| **Annotations** | Texte, flèche, nuage, surligneur, note, légende, symbole | — (non chiffrées) |

### 1.4 Unités

- Unités de **calibration / mesure** : `pi` (ft), `po` (in), `m`, `cm`, `mm`.
- **Saisie impériale** : format pieds-pouces-seizièmes (ex. `20-06-08` = 20 pi 6 po 8/16), précision au 1/16".
- Unités de **prix produit** : `pi²`, `pi lin.`, `m²`, `m lin.`, `m³`, `vg²`, `unité`, `feuille`, `sac`, `boîte`, `heure`.
- **Réconciliation automatique** : si le plan est calibré en mètres mais que le produit est tarifé au `pi²`, la quantité est **convertie** avant chiffrage (pi ≡ ft, pi² ≡ ft², m/m², vg² ≡ verge). En pieds (flux QC standard), aucune conversion (identité). Une **alerte « unité incompatible »** s'affiche si une mesure d'aire est liée à un produit tarifé au linéaire (quantité non convertie).

### 1.5 Accès

- Le Métré est un **onglet de la page Devis / Soumissions** :
  - Sidebar → **Devis / Soumissions** (`/devis`) → onglet **« Métré »**.
- Il se charge à la demande (chunk asynchrone) ; un plan ouvert **survit** à la navigation entre onglets.
- Mode **plein écran** disponible (style « atelier ») pour maximiser la surface de tracé.

### 1.6 Permissions

- Tout utilisateur authentifié du tenant peut **créer, mesurer, chiffrer, exporter, sauvegarder**.
- Les **métrés sauvegardés**, les **catalogues** (produits / main-d'œuvre) et les **assemblages** sont partagés au niveau du **tenant** (isolation par schéma PostgreSQL — un tenant ne voit jamais les données d'un autre).
- L'**assistant IA** consomme des **crédits IA** facturés au tenant ; chaque **action d'écriture** exige une **confirmation humaine**.

### 1.7 Modèle IA et tarification

- Assistant Métré : modèle **Claude Opus 4.8** (`claude-opus-4-8`), réponse en **streaming**, boucle d'outils (max 8 itérations), plafond de coût **2,00 $ US par message**, cap de **200 messages par conversation**.
- Détection de points d'accrochage : **OpenCV** côté serveur (traitement déporté hors de la boucle d'événements).
- La clé API reste **côté serveur** ; le client n'envoie que son jeton d'authentification.

---

## 2. Interface

### 2.1 Disposition générale

```
+------------------------------------------------------------------+
|  Barre d'outils (TopToolbar) : Fichier · Sélection · Outils ·    |
|  Calibrer · Calques · Zoom · Unités · Exporter · IA · Plein écran|
+----------+--------------------------------------------+----------+
| Panneau  |                                            | Panneau  |
| GAUCHE   |        CANVAS (plan PDF + mesures)          | DROIT    |
| (calques,|   PDF.js (fond) + Fabric.js (tracés)       | (proprié-|
|  mesures,|   + couche d'interaction (souris)          |  tés,    |
|  produits|                                            |  coûts)  |
|  BOM)    |                                            |          |
+----------+--------------------------------------------+----------+
|  Barre du bas (BottomBar) : page · zoom · position curseur ·     |
|  unité · valeur de la mesure en cours · échelle                  |
+------------------------------------------------------------------+
```

Le rendu superpose **trois couches** : le PDF (rendu par **PDF.js**, mis en cache haute résolution), les mesures (dessinées par **Fabric.js**), et une couche transparente qui capte les événements souris (l'accrochage et le clic sont gérés sur mesure).

### 2.2 Barre d'outils (haut)

Regroupée par intention :

- **Fichier** : charger un PDF / une image, nouveau métré, ouvrir / sauvegarder (bibliothèque).
- **Sélection** : sélectionner / déplacer / éditer les mesures.
- **Outils de tracé** : Distance, Mur, Polyligne, Périmètre, Surface, Rectangle, Cercle, Comptage, Angle, Cotation, + annotations (texte, flèche, nuage, surligneur, note, légende, symbole).
- **Calibrer** : définir l'échelle de la page.
- **Réglages de tracé** : accrochage (snap), ortho/angle, alignement par axe.
- **Vue** : zoom +/−, ajuster, rotation de page, navigation multi-pages.
- **Unités** : bascule pi-po / métrique, affichage des cotations.
- **Exporter** : CSV / PDF / HTML / PNG / DXF, soumission, bordereaux.
- **Assistant IA** : ouvre le panneau de chat.

### 2.3 Panneau gauche

- **Calques** : liste, couleur, visibilité, verrouillage, regroupement en **groupes pliables**, **liaison à un assemblage** (composite) et **paramètres par calque** (ex. « mur 2×4 » vs « mur 2×6 » dans le même bordereau).
- **Mesures** : liste des mesures de la page (par type), valeur, produit associé.
- **Catalogues** : produits, main-d'œuvre (corps de métier CCQ), blocs-symboles.
- **Bordereau / Estimation BOM** : panneau d'assemblages avec sections et cumulé, et les boutons **CSV Fournisseur** / **CSV Estimation**.

### 2.4 Panneau droit (Propriétés)

Pour la **mesure sélectionnée** : type, **valeur**, points, **produit associé**, **quantité** (override possible), **% de perte**, **facteur de pente** (toiture), **déductions**, **main-d'œuvre** (corps de métier, heures, nb de personnes), **coût total** de la ligne. Affiche un récapitulatif global **Matériaux / Main-d'œuvre / Total** et les boutons d'export (PDF, HTML, CSV, DXF).

Avertissements contextuels :
- **« Unité incompatible »** : la mesure ne correspond pas à l'unité de prix du produit (quantité non convertie).
- **« Polygone auto-intersectant »** : un contour qui se croise (nœud papillon) donne une aire/un périmètre potentiellement faux ; le module invite à retracer.

### 2.5 Barre du bas

Numéro de page, niveau de zoom, **position du curseur** (coordonnées réelles), unité active, **valeur en direct** de la mesure en cours de tracé, et l'**échelle** de la page.

### 2.6 Modaux et panneaux

- **Calibration** : saisie de la longueur de référence + unité.
- **Catalogue produits** / **Catalogue main-d'œuvre** / **Catalogue symboles**.
- **Éditeur de composite** (assemblage paramétrique : variables + formules + sous-produits).
- **Estimation BOM** (bordereau live).
- **Soumission** : aperçu des lignes, mode de consolidation, envoi vers devis.
- **Bibliothèque** / **Sauvegarder le métré**.
- **Assistant IA** (chat + carte de confirmation d'action).

---

## 3. Workflows pas-à-pas

### 3.1 Charger un plan

1. Onglet **Métré** → **Fichier** → choisir un **PDF** (ou une image).
2. Le plan s'affiche ; naviguer entre les pages si multi-pages.
3. Le document est mis en cache et **téléversé** (BYTEA) lorsqu'un métré est actif (sauvegarde serveur).

### 3.2 Calibrer l'échelle (indispensable avant de chiffrer)

1. Outil **Calibrer** → tracer une ligne sur une **cote connue** du plan (ex. la longueur d'un mur indiquée à 40 pi).
2. Saisir la **longueur réelle** + l'**unité** (impérial `40-00-00` ou décimal).
3. Le facteur d'échelle est enregistré **pour cette page**. Répéter par page si les échelles diffèrent.

> Astuce : choisir une cote **longue** et bien lisible (une ligne de référence trop courte amplifie l'imprécision).

### 3.3 Tracer des mesures

- **Distance / Mur** : 2 clics. Pour le **Mur**, on peut saisir la **longueur exacte au clavier** (direction par les flèches, longueur en pi-po-fractions, Entrée pose à la longueur saisie).
- **Surface / Périmètre / Polyligne** : clics successifs, double-clic ou Entrée pour fermer.
- **Rectangle** : 2 coins. **Cercle** : centre + rayon.
- **Comptage** : un clic par élément ; le total s'incrémente.
- **Accrochage** : le curseur s'aimante aux **extrémités**, **milieux** et **intersections** des autres mesures ; l'**alignement par axe** (guides) et le **mode ortho** (45°) aident à tracer droit.

### 3.4 Organiser : calques et groupes

1. Panneau gauche → créer des **calques** (ex. « Murs RDC », « Plancher »), choisir couleur / visibilité.
2. Regrouper les calques en **groupes pliables**.
3. Verrouiller un calque pour éviter de le modifier par erreur.

### 3.5 Associer un produit et chiffrer

1. Sélectionner une mesure (panneau droit).
2. Choisir un **produit** du catalogue (ou en créer un).
3. Optionnel : **quantité** manuelle (override), **% de perte**, **facteur de pente** (toiture), **déductions** (ouvertures), **main-d'œuvre** (corps de métier + heures).
4. Le **coût** s'affiche : `quantité nette × (1 + perte%) × prix`, **après conversion d'unité** vers l'unité de prix du produit.

### 3.6 Déductions (ouvertures)

1. Tracer la mesure parente (ex. surface de mur).
2. Tracer l'ouverture comme **déduction** rattachée à la parente.
3. La quantité **nette** = parente − Σ(déductions) (jamais négative).

### 3.7 Assemblages paramétriques (BOM / composites)

Pour des ouvrages répétitifs (mur 2×4, plancher, etc.) :

1. **Éditeur de composite** : déclarer des **variables** (`surface_mur`, `longueur_mur`, `nombre_coins`, etc.) avec leur **unité** (pi, pi², …).
2. Ajouter des **sous-produits** avec une **quantité par unité** ou une **formule** (ex. `surface_mur / 32` pour des feuilles 4×8, `IF(surface_ss > 800, 3, 2)`).
3. **Lier un calque** à l'assemblage : les variables géométriques se remplissent **automatiquement** depuis les mesures du calque (« trace = calcule »). Les valeurs sont **converties dans l'unité déclarée de la variable** (un plan métrique alimentant une formule impériale est converti).
4. Le **panneau Estimation BOM** affiche le bordereau (sections + **cumulé** par produit).

> Sécurité : les formules sont évaluées par un **interpréteur dédié** (pas d'`eval`), avec liste blanche d'opérateurs/fonctions (`+ − × ÷`, `IF/MIN/MAX/ROUND/SUM/ABS/CEIL/FLOOR`) et bornes anti-boucle.

### 3.8 Générer la soumission

1. **Soumission** (depuis le panneau droit ou le bordereau).
2. Choisir le **mode de consolidation** (détaillé / par produit / par produit et calque).
3. Vérifier les lignes (matériaux + main-d'œuvre), taxes **TPS 5 % / TVQ 9,975 %** (ou la configuration du tenant).
4. **Envoyer vers le devis** : ajouter les lignes à un devis existant ou **créer un nouveau devis** depuis le métré.

### 3.9 Exporter

| Sortie | Bouton | Contenu |
|---|---|---|
| **CSV Fournisseur** | Bordereau BOM | Bordereau matériaux (détaillé + cumulé) pour commande |
| **CSV Estimation** | Bordereau BOM | Liste de commande actionnable (lignes à quantité > 0) |
| **CSV Résumé / Estimation** | Panneaux | Tableaux par produit / par mesure |
| **PDF** | Exporter | Soumission mise en page (matériaux, M.O., taxes) |
| **HTML** | Exporter | Soumission autonome éditable (marges admin/contingences/profit) |
| **PNG** | Exporter | Plan annoté + tableau de coûts |
| **DXF** | Exporter | Géométrie des mesures (CAO) |

> **CSV et Excel** : les fichiers sont en **UTF-8 avec BOM** et utilisent le **séparateur de la langue du document** — `point-virgule (;)` en français (Excel QC), `virgule (,)` en anglais. Chaque cellule est encadrée de guillemets (un nom contenant `;` ou `,` ne décale pas les colonnes) et l'injection de formule est neutralisée. En cas de plan calibré en métrique avec un catalogue impérial, une **note d'avertissement** signale les mesures à unité incompatible.

### 3.10 Sauvegarder et rouvrir

- **Sauvegarder le métré** : nom + enregistrement serveur (un métré = projet + document(s) + mesures + calques + calibrations).
- **Autosauvegarde** (~2 s) en local + synchronisation serveur optimiste.
- **Bibliothèque** : rouvrir, renommer, supprimer un métré.

### 3.11 Assistant IA

1. Ouvrir le panneau **Assistant IA**.
2. Décrire en langage naturel (ex. « ajoute un calque Murs RDC », « crée une mesure de surface de 200 pi² »).
3. Pour toute **action d'écriture**, l'IA présente une **carte d'aperçu** ; rien n'est appliqué tant que l'utilisateur ne **confirme** pas (confirmation **côté serveur**, non contournable).
4. Les **conversations** sont persistées ; des **profils experts** orientent le ton/le domaine.

---

## 4. Référence

### 4.1 Endpoints backend (préfixe `/api/metre`)

Tous protégés par JWT ; isolation par schéma tenant.

| Méthode + route | Rôle |
|---|---|
| `POST/GET/GET{id}/PUT/DELETE /projects` | CRUD métrés (projet) |
| `GET /metres-library` | Vue agrégée de la bibliothèque |
| `POST /projects/{id}/documents/upload` | Téléverser un PDF (magic-byte vérifié, BYTEA, max 150 Mo) |
| `GET /projects/{id}/documents`, `GET /documents/{id}` | Lister / métadonnées document |
| `GET /documents/{id}/file` | Télécharger le PDF |
| `GET /documents/{id}/page/{page}` | Rendu PNG d'une page (cache) |
| `DELETE /documents/{id}` | Supprimer un document |
| `POST /documents/{id}/calibrate` | Définir l'échelle d'une page |
| `POST /documents/{id}/recalibrate` | Recalibration atomique (re-projette les mesures) |
| `GET/DELETE /documents/{id}/calibration/{page}` | Lire / supprimer une calibration |
| `GET/POST /documents/{id}/measurements`, `GET/PUT/DELETE /measurements/{id}` | CRUD mesures |
| `GET /documents/{id}/measurements/export` | Export CSV / JSON |
| `... /layers`, `... /layer-groups` | CRUD calques et groupes |
| `... /products`, `/products/bulk-import`, `/products/{id}/components` | CRUD catalogue + assemblages |
| `POST /documents/{id}/snap-points` | Détection de points d'accrochage (OpenCV) |
| `GET /documents/{id}/summary` | Résumé par type / calque |
| `POST /projects/{id}/import-to-devis/{devis_id}` | Importer les mesures → lignes de devis (verrou atomique) |

### 4.2 Modèle de données

- **`metre_projects`** : le métré (lien optionnel `devis_id`).
- **`metre_documents`** : PDF en **BYTEA** + `mime_type` (forcé `application/pdf`), nb de pages.
- **`metre_calibrations`** : `UNIQUE(document_id, page_number)`, facteur d'échelle + unité.
- **`metre_layers`** / **`metre_layer_groups`** : calques, liaison `composite_id`, overrides `composite_inputs` (JSONB).
- **`metre_measurements`** : `points` (JSONB), `metadata_json` (JSONB), type, valeur, unité, produit, déduction, pente, main-d'œuvre.
- **`metre_products`** / **`metre_product_components`** : catalogue (prix, unité, perte, `bom_inputs` JSONB) et sous-produits (quantité ou formule).
- **`metre_ai_conversations` / `_messages` / `_tool_executions`** : assistant IA.

### 4.3 Assistant IA — outils

19 outils : **8 en lecture** (auto-exécutés : lister mesures/calques, calibration, résumé, chercher produits, détails composite, projets passés) et **11 en écriture** (confirmation requise : créer calque/mesure, lier composite, supprimer mesure, créer/modifier/supprimer produit, variables/lignes de composite). Toute écriture suit le **patron aperçu → confirmation serveur → exécution** (re-validation + verrou anti-double-exécution).

### 4.4 Bornes et validations

- Upload : **150 Mo** max, **magic-byte** `%PDF-` vérifié, `mime_type` forcé.
- `metadata_json` ≤ 64 Ko ; `composite_inputs` ≤ 16 Ko ; recalibration ≤ 5 000 mesures ; région d'accrochage ≤ 4 000 px ; formule ≤ 500 caractères.
- Numéros et identifiants via `INSERT … RETURNING` ; CSV protégé contre l'injection de formule (CWE-1236) ; `Content-Disposition` assaini (RFC 5987).

### 4.5 Limitations connues

- **Calibration par page** : un plan multi-échelles doit être calibré page par page.
- **Aire isotrope** : l'aire suppose la même échelle en X et en Y (un plan scanné/étiré peut fausser les surfaces).
- **Polygone simple** : un contour auto-intersectant donne une aire trompeuse ; le module **avertit** mais ne corrige pas.
- **Rotation de page** : tracer après une rotation de page ; la rotation n'est pas re-projetée sur des mesures existantes.
- **Bordereau à l'écran vs soumission exportée** : en calibration **métrique** avec des assemblages dont les **formules** sont en impérial, le bordereau **affiché** peut, pour certaines variables saisies par étiquette, différer de la **soumission exportée** (qui, elle, applique la conversion). En **pieds** (flux QC), aucune différence. Suivi en cours.
- **Tactile** : optimisé souris ; usage tablette limité.

---

## 5. Intégrations et FAQ

### 5.1 Intégration Devis (Module 08 — Soumissions)

Le Métré **alimente le devis** : depuis la soumission, « **Ajouter au devis** » insère les lignes dans un devis existant, ou « **Créer un devis** » génère un nouveau devis (numéro `DEV-AAAA-NNN`) avec les lignes du métré. Le lien est tracé par `metre_projects.devis_id`.

### 5.2 Catalogues partagés

Les **produits** et la **main-d'œuvre** (corps de métier CCQ) sont partagés au niveau tenant. Un produit créé dans le Métré sert aux assemblages et aux soumissions.

### 5.3 Modélisation 3D (DAO / cao2)

Module distinct : le Métré travaille **sur un plan PDF** (2D, prise de quantités) ; la **Modélisation 3D** construit un **modèle** (murs, toits, mobilier) et produit son propre métré 3D. Les deux partagent des utilitaires (saisie impériale, cotation).

### 5.4 Assistant IA (Module 25)

L'assistant Métré est **dédié au métré** (outils typés sur les mesures/catalogues), distinct de l'Assistant IA général. Il facture des **crédits IA** au tenant.

### 5.5 FAQ

- **Mes accents sont incorrects dans le CSV (`boîte` → `boÃ®te`).** Mise à jour appliquée : les CSV sont en UTF-8 + séparateur selon la langue (`;` en FR), sans directive `sep=` (qui empêchait Excel d'honorer l'encodage). Ré-exporter.
- **Mes colonnes CSV sont décalées / `#NAME?`.** Corrigé : chaque cellule est quotée et les lignes-marqueurs ne sont plus interprétées comme des formules.
- **Le CSV du bordereau ne prenait pas tous les calques.** Corrigé : les mesures à **produit direct** (calque sans assemblage) sont désormais incluses dans les CSV Fournisseur/Estimation.
- **Ma quantité semble 10× trop basse.** Vérifier la cohérence **unité de calibration ↔ unité de prix** : un plan en **mètres** avec un produit au **pi²** est désormais converti automatiquement ; une alerte signale les unités réellement incompatibles (aire liée à un prix linéaire).
- **Excel anglais.** Le séparateur suit la **langue du document** (Configuration) : passez la langue à « anglais » pour obtenir un CSV à séparateur virgule.
- **Mon plan a plusieurs échelles.** Calibrer **chaque page** séparément.

---

## 6. Récap one-pager

| Élément | Détail |
|---|---|
| **Accès** | Devis / Soumissions (`/devis`) → onglet **Métré** |
| **But** | Prise de quantités sur PDF → chiffrage → soumission / bordereaux |
| **Étapes** | Charger PDF → **Calibrer** (par page) → Tracer → Associer produits / BOM → Soumission → Exporter / Envoyer au devis |
| **Mesures** | Distance, Mur, Polyligne, Périmètre, Surface, Rectangle, Cercle, Comptage, Angle, Cotation, annotations |
| **Unités** | pi/po/m/cm/mm ; saisie impériale 1/16" ; conversion auto vers l'unité de prix |
| **Chiffrage** | produit × quantité nette × (1+perte) ; déductions ; pente ; main-d'œuvre CCQ ; taxes QC |
| **BOM** | assemblages paramétriques (variables + formules sûres, sans `eval`) ; auto-alimentation par calque |
| **Exports** | CSV Fournisseur / Estimation (UTF-8 + séparateur selon langue), PDF, HTML, PNG, DXF |
| **Intégration** | Ajoute/crée des lignes de **devis** ; catalogues partagés tenant |
| **IA** | Assistant (Opus 4.8) ; écritures **confirmées** côté serveur ; crédits tenant |
| **Persistance** | PDF en BYTEA ; autosave ~2 s + serveur ; bibliothèque de métrés |
| **À retenir** | **Toujours calibrer avant de chiffrer** ; vérifier unité mesure ↔ unité de prix |

---

> Manuel généré à partir du code source (`frontend/src/components/metre-pdf/`, `backend/routers/metre_pdf.py`, `metre_ai_*`). Pour les sorties qui alimentent le devis, voir **08 — Soumissions**. Pour la modélisation 3D, voir le manuel **Modélisation 3D (DAO)**.
