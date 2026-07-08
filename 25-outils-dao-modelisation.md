# Module 25 — DAO / Modélisation 3D (cao2)

> **Version** : 2.1 (rédigé d'après le code source réel, révision 2026-07)
> **Code de référence** : `frontend/src/pages/Cao2Page.tsx` + tout le dossier `frontend/src/components/cao2/` (barre d'outils `Cao2Toolbar`, scène 3D `Cao2Scene`, plan 2D `Cao2Plan2D`, feuille de mise en plan `Cao2Sheet`, panneaux, menus, catalogue de mobilier `Cao2ItemCatalog`), `frontend/src/i18n/locales/fr/cao2.json` (libellés) ; **backend** = `backend/routers/cao.py` (persistance + rendu, 6 endpoints), `backend/routers/cao_ai.py` (assistant de dessin, 2 endpoints), `backend/gemini_image.py` (moteur d'image). **8 endpoints au total** ; préfixe backend réel `/api/erp/v1/cao` (à ne pas confondre avec la route frontend `/cao2`).
> **Tables PostgreSQL** : `cao_documents` (**par tenant**, colonne JSONB `model_json`, créée à la demande la première fois). La facturation IA s'appuie sur les tables partagées `public.ai_prepaid_credits` (solde de crédits), `public.ai_usage_tracking` (traçage d'usage) et `public.entreprises`.
> **Cadrage** : atelier **web natif de modélisation 3D et de mise en plan 2D** d'un bâtiment résidentiel, directement dans le navigateur, sans logiciel externe. On y dessine des murs, des pièces, des dalles, des toits, des ouvertures (portes / fenêtres), des escaliers et du mobilier ; on cote ; on met en plan (feuille + PDF) ; on produit un **rendu photoréaliste par IA** ; on **dérive un métré vers un devis** ; et un **assistant IA** peut dessiner à partir d'une description en français ou d'un plan importé (PDF / image). Ce module **n'est pas** le module **Métré** (prise de quantités sur PDF, module 32) ni le module **PDF3D / Hover** (voisin dans la même section CONCEPTION 3D). Il porte une **bannière permanente « en développement »** : les fonctions décrites sont en place et fonctionnelles, mais l'outil évolue encore.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas-à-pas](#3-workflows-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module DAO permet de **modéliser un bâtiment en 3D** et d'en tirer, dans le même atelier :

- une **enveloppe** (murs, dalles, toit) et son aménagement (portes, fenêtres, escaliers, mobilier) ;
- des **quantités automatiques** — surfaces (pièces, dalles, emprises de toit) et longueurs (murs) — pour un métré sommaire ;
- une **mise en plan 2D** (plan au sol coté, cartouche, échelle) exportable en **PDF vectoriel** ;
- une **image de présentation photoréaliste** générée par IA pour un client ;
- une **estimation budgétaire** convertie directement en **devis** (module Ventes) ;
- un **export du modèle 3D** (GLB / STL / OBJ) vers une autre application.

Le modèle reste **éditable en tout temps** : la vue 3D, la vue 2D et la feuille de mise en plan partagent le même modèle. Une modification faite dans une vue est visible dans les autres.

### 1.2 Concepts clés

- **Hiérarchie du modèle** : `Site` → `Bâtiment` → `Étage` → éléments (`Mur`, `Dalle`, `Toit`, `Escalier`, `Élément` de mobilier). Une `Porte` ou une `Fenêtre` est l'enfant d'un `Mur` (ou d'une face verticale de toit-forme).
- **Étage actif** : tout nouvel élément est créé dans l'**étage actif**. On rend un étage actif en cliquant dessus dans l'arbre de scène ; un badge « actif » l'indique. Les étages sont **empilés** en 3D (hauteur par défaut 2,7 m) ; en vue 2D et sur la feuille, seul l'étage actif est dessiné.
- **Modèle partagé, sans base de données** : le modèle vit dans un magasin d'état côté navigateur (le « cœur Pascal »). **Aucune écriture en base** n'a lieu pendant que vous dessinez ; seules la **sauvegarde d'un document** et les **appels IA facturés** communiquent avec le serveur.
- **Trois vues du même modèle** (groupe « Vue » de la barre d'outils) :

| Vue | Libellé | Contenu | Composant |
|-----|---------|---------|-----------|
| **3D** (défaut) | « Vue 3D » | Perspective temps réel : dessin, sélection, navigation (orbite, view cube, mode marche), rendu réaliste, rendu IA, export 3D. | `Cao2Scene` |
| **2D** | « Vue 2D (plan) » | Plan vu de dessus interactif (SVG) : dessin et sélection en 2D, esquisse (linework). | `Cao2Plan2D` |
| **Plan** | « Plan » | Feuille de mise en plan **statique** (format papier ARCH D) : plan au sol coté, cartouche, notes, export PDF. | `Cao2Sheet` |

### 1.3 Accès

- **Menu latéral** : section **CONCEPTION 3D** → **DAO** (icône cube). La même section contient **Hover (PDF3D)** et **Rendu 3D**, qui sont des modules distincts.
- **Route** : `/cao2` (page protégée, authentification requise). L'ancienne route `/cao` **redirige automatiquement** vers `/cao2`.
- **Page** : `Cao2Page.tsx`. L'atelier réutilise la charte visuelle du module Métré (aucun style propre) ; hauteur normale **82 vh**, **100 vh** en plein écran.
- Une **bannière permanente** « DAO - Atelier de construction (en développement) » est affichée en haut.

### 1.4 Permissions et rôles

- **Tout utilisateur authentifié** du tenant peut lister, créer, modifier et supprimer des documents DAO, produire des rendus et utiliser l'assistant IA. **Aucune restriction par rôle** propre au module (pas de garde « administrateur », « comptable » ni « super-administrateur »).
- Les **documents** sont propres à votre entreprise (isolation par schéma PostgreSQL : un tenant ne voit jamais les documents d'un autre).
- Le **mode consultation** (lecture seule, hérité du compte du tenant) et l'exemption du **super-administrateur** pour les crédits IA sont gérés de façon centralisée par l'authentification, pas redéfinis dans ce module.

### 1.5 Modèles IA et facturation

Trois fonctions du module consomment des **crédits IA** du tenant, refacturés avec une **marge de 30 % (× 1,30)** :

| Fonction | Modèle | Où | Réf |
|---|---|---|---|
| **Assistant Dessin** (langage naturel) | Claude Sonnet 4.6 | Panneau Assistant Dessin | `cao_ai.py:378` |
| **Importer un plan → 3D** (vision) | Claude Opus 4.8 | Bouton « Importer un plan » de l'assistant | `cao_ai.py:646` |
| **Rendu photoréaliste** | Google Gemini (clé serveur) | Modale « Rendu IA » | `cao.py:539` |

L'**estimation vers devis** (« Estimer ce modèle ») utilise, elle, l'IA d'estimation du module Devis (elle aussi facturée en crédits). La clé API reste **côté serveur** dans tous les cas ; le client n'envoie que son jeton d'authentification. En mode auto-hébergé (facturation désactivée) ou pour un super-administrateur, ces appels restent **gratuits mais tracés**.

### 1.6 Sous-modules et panneaux

| Élément | Rôle |
|---|---|
| **Barre d'outils** (`Cao2Toolbar`) | Fichier, outils de dessin, réglages de tracé, historique, niveaux, vue, affichage, navigation, plein écran, réinitialiser. |
| **Arbre de scène** (`Cao2SceneTree`, gauche) | Hiérarchie du modèle, sélection, visibilité, étage actif. |
| **Vue centrale** | 3D (`Cao2Scene`) / 2D (`Cao2Plan2D`) / Feuille (`Cao2Sheet`), sous frontière d'erreur. |
| **Panneau Propriétés** (`Cao2PropertiesPanel`, droite) | Champs de l'élément sélectionné, ou métré si rien n'est sélectionné. |
| **Panneau Quantités** (`Cao2QuantitiesPanel`) | Métré sommaire + bouton « Estimer ce modèle ». |
| **Assistant Dessin** (`Cao2AiPanel`, extrême-droite) | Chat IA de dessin (optionnel). |
| **Modale Estimation** (`Cao2EstimateModal`) | Métré → estimation IA → devis. |
| **Modale Rendu IA** (`Cao2RenderModal`) | Rendu photoréaliste de la vue 3D. |
| **Catalogue de mobilier** (`Cao2ItemCatalog`) | 110 objets 3D par catégorie. |

---

## 2. Interface

### 2.1 Disposition de l'atelier

```
+----------------------------------------------------------------------+
| Bannière « DAO - Atelier de construction (en développement) »        |
+----------------------------------------------------------------------+
| BARRE D'OUTILS (Fichier | Outils | Réglages | Historique | Niveaux   |
|                 ... | Vue | Affichage | Navigation | Plein écran)     |
+------------------+--------------------------------+------------------+
| ARBRE DE SCÈNE   |   VUE CENTRALE                 | PROPRIÉTÉS       |
| (gauche, w-56)   |   3D / 2D / Feuille            | (droite, w-72)   |
|                  |                                |                  |
| Site             |   [badge saisie longueur]      | [si sélection]   |
|  Bâtiment        |   [fil de rétroaction]         | champs éditables |
|   Étage (actif)  |        modèle du bâtiment       | [sinon]          |
|    Murs, Dalles  |                                | Quantités        |
|    Toit, Escalier|                                |                  |
+------------------+--------------------------------+------------------+
      (Panneau « Assistant Dessin » s'ouvre à l'extrême-droite, en option)
```

Deux **surimpressions** apparaissent au besoin sur la vue centrale : un **badge de saisie de longueur** de mur (`Cao2WallEntryBadge`) pendant le tracé au clavier, et un **fil de rétroaction transitoire** (`Cao2Notice`, messages courts). La vue centrale est protégée par une **frontière d'erreur** : si une vue échoue, votre modèle et vos autres modules ne sont pas affectés.

### 2.2 Barre d'outils

Regroupée **par intention**, séparée par de fins traits verticaux. Sur petit écran, elle passe sur plusieurs lignes (pas de défilement horizontal).

```
[Document ▾] [Exporter ▾] [Assistant Dessin] | [Sélectionner][Mur][Rectangle][Dalle]
[Toit ▾][Porte][Fenêtre][Escalier][Mobilier][Cote][Ajuster murs][Esquisse ▾]
| [Angle 15°][Aligner] [Épaisseur: Ext./Cloison] [Hauteur ▾]
| [Annuler][Rétablir] | [Ajouter un étage]
                                  ... (espace) ...
[3D][2D][Plan][Parallèle][Marche] | [Affichage ▾] | [Zoom−][Zoom+][Ajuster] | [Plein écran]   [Réinitialiser]
```

**2.2.1 Groupe Fichier**

- **Menu Document** (`Cao2DocMenu`) : le bouton affiche le nom du document courant (ou « Document sans titre ») et un **badge d'état** (voir 2.11). Éléments : **Nouveau** ; **Enregistrer** (ou **Enregistrer sous...** si le document n'a pas encore de nom) ; **Renommer** (grisé sans document) ; puis la liste **Ouvrir** (chaque ligne : sélectionner, ou supprimer avec confirmation). Les invites passent par des fenêtres du navigateur (« Nom du document : », « Nouveau nom : », confirmation de suppression).
- **Menu Exporter** (`Cao2ExportMenu`, contextuel selon la vue) : section **Modèle 3D** (GLB / STL / OBJ, actifs en vue 3D) ; section **Mise en plan** (PDF, actif en vue Plan) ; section **Image** (Rendu IA, actif en vue 3D). Les éléments hors de leur vue sont **grisés**.
- **Assistant Dessin** (bouton bascule, icône étincelles) : ouvre ou ferme le panneau IA de dessin.

**2.2.2 Groupe Outils**

| Outil | Libellé | Rôle |
|-------|---------|------|
| Sélection | « Sélectionner » | Sélectionner / déplacer un élément (outil par défaut). |
| Mur | « Mur » | Tracer un mur (2 clics + saisie clavier de longueur). |
| Rectangle | « Rectangle » | Tracer une pièce = 4 murs fermés (2 clics). |
| Dalle | « Dalle » | Tracer un plancher polygonal. |
| Toit | « Toit ▾ » | Menu : Toit auto (sur les murs) / Toit (forme). |
| Porte | « Porte » | Percer une porte en cliquant sur un mur. |
| Fenêtre | « Fenêtre » | Percer une fenêtre en cliquant sur un mur. |
| Escalier | « Escalier » | Poser un escalier. |
| Mobilier | « Mobilier » | Ouvrir le catalogue et poser un meuble. |
| Cote | « Cote » | Cotation manuelle (voir limites, section 5). |
| Ajuster | « Ajuster murs » | Étendre ou rogner un mur jusqu'au mur le plus proche. |
| Esquisse | « Esquisse ▾ » | Menu d'esquisse 2D (grisé hors de la vue 2D). |

Le **menu Toit** (`Cao2RoofMenu`) propose deux modes : **Toit auto (sur les murs)** — squelette multi-pente généré automatiquement sur l'emprise des murs de l'étage actif — et **Toit (forme)** — un toit-forme dessiné par rectangle à 2 coins, avec 7 formes paramétriques.

Le **menu Esquisse** (`Cao2SketchMenu`, vue 2D seulement) contient une section **Dessiner** (Ligne, Cercle, Arc, Rectangle, Point) et une section **Modifier** (Déplacer, Copier, Pivoter, Symétrie — grisées tant qu'aucune entité d'esquisse n'est sélectionnée).

**2.2.3 Groupe Réglages de tracé**

- **Angle 15°** (bascule) : contraint l'angle du tracé au multiple de 15°. **Actif par défaut.**
- **Aligner** (bascule) : aimant d'alignement orthogonal sur les murs existants. **Actif par défaut.**
- **Épaisseur** des nouveaux murs : deux préréglages — **« Ext. 9 1/4" »** (mur extérieur, 0,235 m) et **« Cloison 4 1/2" »** (cloison intérieure, 0,114 m).
- **Hauteur** des nouveaux murs : menu déroulant de 5 hauteurs standard du Québec — 8'2", 9'2", 10'2", 11'2", 12'2" (défaut 8'2").

**2.2.4 Groupes Historique et Niveaux**

- **Annuler / Rétablir** : pile d'annulation ; les boutons sont désactivés quand la pile est vide.
- **Ajouter un étage** : crée un nouvel étage qui devient **actif** et sélectionné (hauteur par défaut 2,7 m).

**2.2.5 Groupe Vue**

- **3D / 2D / Plan** : bascule entre les trois vues.
- **Parallèle** (vue 3D seulement) : projection orthographique (sans perspective) ; désactivé en mode marche.
- **Marche** (vue 3D seulement) : navigation à la première personne.

**2.2.6 Groupe Affichage** (`Cao2DisplayMenu`)

Menu de cases à cocher :

- **Cotes des murs** (défaut activé), **Surfaces des dalles** (activé), **Pièces (surfaces)** (activé), **Grille au sol** (activé).
- **Arêtes (3D)** (vue 3D, désactivé par défaut) : surligne les contours des volumes.
- **Rendu réaliste** (vue 3D, activé par défaut) : éclairage réaliste tout en gardant le modèle éditable.
- Section **Unités** : **m** (métrique) ou **pi-po** (impérial pieds-pouces).

**2.2.7 Navigation, plein écran, réinitialiser**

- **Zoom arrière / Zoom avant / Ajuster à l'écran**.
- **Plein écran** (bascule).
- **Réinitialiser** (isolé à droite, en rouge discret) : vide la scène après confirmation « Réinitialiser la scène DAO ? Cette action est irréversible. »

### 2.3 Arbre de scène (panneau gauche)

L'arbre (`Cao2SceneTree`, titre « Arbre de scène » ; « Scène vide » si le modèle est vide) affiche toute la hiérarchie sous forme d'arborescence dépliable. Chaque ligne comporte un chevron déplier / replier, une icône par type, un libellé (nom ou type traduit), un badge « actif » sur l'étage courant, et un bouton œil **Afficher / Masquer**.

- **Sélection** : un clic sélectionne l'élément ; **Ctrl / Cmd + clic** ajoute à une **multi-sélection**.
- **Étage actif** : cliquer un **étage** le rend **actif** (cible des nouveaux éléments) et le sélectionne.
- **Clavier** : `Entrée` / `Espace` sélectionne, les flèches naviguent et déplient / replient.
- **Garde-fou** : les conteneurs racines **Site** et **Bâtiment** ne sont **pas supprimables** au clavier.

Types affichés : Site, Bâtiment, Étage, Mur, Dalle, Toit, Escalier, Porte, Fenêtre, Élément.

### 2.4 Panneau Propriétés (panneau droit)

Le panneau « Propriétés » (`Cao2PropertiesPanel`, repliable en onglet vertical de 32 px) affiche les champs **selon le type** d'élément sélectionné. Les champs de longueur suivent le **système d'unités** actif ; en impérial, la saisie accepte le format pieds-pouces-seizièmes (astuce affichée : « ex. 8-2-7 » = 8 pi 2 po 7/16, ou simplement « 8 »).

**Cas d'affichage particuliers**

- **Aucune sélection** → le panneau bascule sur le **métré (Quantités)** (voir 2.5).
- **Multi-sélection (2 éléments ou plus du même type)** : murs → Hauteur (+ 5 préréglages) et Épaisseur (+ 2 préréglages), avec la mention « valeurs mixtes » si elles diffèrent ; portes / fenêtres → Largeur et Hauteur ; bouton **Supprimer (N)**.
- **Entité d'esquisse** : cercle / arc → Rayon ; ligne → Longueur (lecture seule) ; rectangle → Largeur / Hauteur (lecture seule) ; bouton Supprimer.

**Champs par type d'élément**

| Type | Champs |
|------|--------|
| **Mur** | Poignées 3D (Déplacer / Pivoter, en 3D), Longueur (éditable), Hauteur (+ préréglages), Épaisseur (+ préréglages Ext. / Cloison), **Pente du dessus (x/12)** + côté le plus haut (mur pignon / rampant), section **Composition** (assemblage). |
| **Dalle** | Élévation (signée), Surface (lecture seule). |
| **Toit (forme)** | **Type de toit** (7 formes, voir 4.5), Hauteur des murs, Pente (°), Débord, Largeur, Profondeur, poignées 3D + Position X / Z + Rotation. |
| **Toit auto** | **Type de toit** (3 formes : 2 versants / 4 versants / Plat), **Pignons par rive** (cases, empreinte rectangulaire seulement), Pente, Débord, Fascia. |
| **Escalier** | Type de segment (**Volée** / **Palier**), Largeur, Longueur, Hauteur, **Marches**, poignées 3D, Position X / Z, Rotation. |
| **Mobilier** | Modèle (lecture seule), **Fixation** Sol / Mur / Plafond (lecture seule), poignées 3D, Position X / Z / Y (hauteur), Rotation. |
| **Porte / Fenêtre** | **Type d'ouverture** (préréglage QC), Largeur, Hauteur, « Position depuis la gauche » ; fenêtre → **Aligner sur le linteau** (case) + **Allège**. |
| **Ouverture de toit** (sur face pignon) | Type, Largeur, Hauteur, « Position le long de la face », « Hauteur sur la face ». |
| **Étage** | Niveau (index), Hauteur d'étage. |

Tous les éléments (sauf portes / fenêtres) ont un champ **Nom** commun ; les murs, dalles, toits et escaliers ont un sélecteur de **Matériau** ; tous ont un bouton **Supprimer**.

**Matériau** : un préréglage **Défaut**, plus 9 pastilles nommées (Blanc, Bois, Brique, Béton, Plâtre, Tuile, Marbre, Métal, Verre) et une **Couleur personnalisée** (sélecteur de couleur).

**Composition de mur** (`Cao2WallAssemblySection`) : un menu **Type d'assemblage** (Aucun, ou une bibliothèque : « Mur extérieur à déclin (R-24.5) », « Cloison 2x4 ») affiche la liste des couches (libellé + épaisseur + valeur R), l'**Épaisseur totale** et le **R total** (lecture seule).

### 2.5 Panneau Quantités (métré)

Quand rien n'est sélectionné, le panneau droit affiche le **métré sommaire** (`Cao2QuantitiesPanel`, titre « Quantités »), dérivé du modèle :

- **Pièces** (nombre + surface), **Murs** (nombre + longueur), **Dalles** (nombre + surface), **Toits** (nombre + emprise), **Portes**, **Fenêtres**, **Ouvertures de toit** (si le compte est supérieur à 0), **Escaliers**.
- Une **section « Métré par matériau »** des murs à assemblage s'affiche automatiquement quand elle est pertinente.
- Note : « Métré sommaire dérivé du modèle. Utilise « Estimer ce modèle » pour en faire un devis. »
- Bouton bleu **« Estimer ce modèle »** (grisé si le modèle n'est pas estimable) : ouvre la modale d'estimation (voir 2.6).

### 2.6 Modale « Estimer ce modèle »

La modale `Cao2EstimateModal` transforme le métré en **devis**, en deux temps :

1. **Résumé du métré** (pièces / murs / dalles / toits / ouvertures / escaliers) puis bouton **« Générer l'estimation IA »** (note : « La génération utilise l'IA et déduit un coût de vos crédits »).
2. **Tableau d'aperçu** — colonnes **Description**, **Qté**, **Prix unit.**, **Montant** — suivi des totaux :

| Ligne | Valeur |
|-------|--------|
| Sous-total | somme des lignes |
| Administration | **3 %** (fixe) |
| Contingences | **12 %** (fixe) |
| Profit | **15 %** (fixe) |
| Total estimé (avant taxes) | sous-total + les 3 majorations |

L'estimation couvre la **construction complète au pied carré** (tous les corps de métier : structure, fondation, enveloppe, isolation, plomberie, électricité, CVAC, intérieur, finitions ; le métré 3D affine l'enveloppe). Un **avertissement** s'affiche si la superficie de plancher est indéterminée (aucune pièce fermée ni dalle) — dans ce cas, fermez le contour des murs ou ajoutez une dalle pour une estimation plus fiable.

Boutons **Régénérer** / **Créer le devis**. À la création, un écran de succès affiche « Devis {numéro} créé » et un bouton **« Ouvrir le devis »** qui navigue vers le module Ventes (`/devis?open=id`). Des verrous internes empêchent le double appel et la création de devis en double.

### 2.7 Assistant Dessin (IA)

Le panneau `Cao2AiPanel` (bouton « Assistant Dessin ») est un chat. Il présente une introduction (« Décris ce que tu veux dessiner et je prépare un plan à confirmer. ») et trois exemples :

- « Une pièce de 4 m par 5 m »
- « Ajoute un toit à deux versants sur les murs »
- « Un mur de (0,0) à (6,0) »

Zone de saisie (`Entrée` envoie ; `Maj + Entrée` insère une nouvelle ligne) et un bouton **« Importer un plan (PDF/image) → 3D »**. Un avertissement rappelle : « L'IA peut se tromper ; vérifie l'aperçu avant de confirmer. »

L'IA répond avec un **« Aperçu du plan »** : la **liste des opérations** proposées (par ex. « Pièce 4 x 5 m (4 murs + plancher) », « Mur de (0, 0) à (6, 0) », « Toit sur les murs du niveau actif », « Toit 2 versants … »). Rien n'est dessiné tant que vous ne cliquez pas **« Confirmer le dessin »** ; **« Annuler »** abandonne le plan. L'exécution se fait **côté navigateur**, dans l'étage actif.

**Périmètre v1** : l'assistant dessine des **murs, des pièces et des toits** uniquement — **pas** d'ouvertures, d'escaliers ni de mobilier (ce sera « Dessin v2 »). L'aperçu est **textuel** (pas de fantôme 3D). L'assistant est **mono-conversation** dans la session (pas d'historique persistant dans le panneau).

### 2.8 Modale Rendu photoréaliste (Rendu IA)

Ouverte depuis le menu **Exporter → Rendu IA** (vue 3D). La modale `Cao2RenderModal` affiche :

- **Vue captée** : l'aperçu de la vue 3D courante (message « Capture indisponible (passez en vue 3D) » si vous n'êtes pas en 3D).
- **Détails et description** (facultatif, jusqu'à 2000 caractères) : par ex. « revêtement de brique rouge, toiture en bardeaux gris, ciel de fin de journée, aménagement paysager avec arbres, voiture dans l'entrée… ».
- **Qualité** : **Pro** (meilleure qualité), **Standard** (équilibre) ou **Rapide** (économique).
- **Résolution** : **2K** (rapide, net) ou **4K** (ultra-détaillé, réservé à Pro / Standard — désactivé en Rapide).
- Bouton **Générer le rendu** ; puis, sur l'image produite, **Télécharger** ou **Régénérer**.

Un verrou synchrone empêche la double facturation par double-clic. Un solde de crédits **strictement positif** est requis (voir 4.3).

### 2.9 Catalogue de mobilier

La modale `Cao2ItemCatalog` s'ouvre avec l'outil **Mobilier** :

- Onglets de **catégories** : **Mobilier**, **Électroménager**, **Salle de bain**, **Cuisine**, **Extérieur**.
- Champ **Rechercher** (par nom) et grille de vignettes.
- **110 objets** au total (mobilier 49, électroménager 22, extérieur 15, cuisine 14, salle de bain 10), sous forme de modèles 3D (GLB) chargés depuis un **CDN externe** (Supabase public de Pascal).
- Astuce affichée : « Choisissez un meuble, puis cliquez dans la scène 3D pour le poser au sol. »

### 2.10 Vue Plan (mise en plan) et cartouche

La vue **Plan** (`Cao2Sheet` + `SheetTitleBlock`) est une **feuille statique** (ni panoramique ni zoom) au format **ARCH D** (24 × 36 po) paysage. Elle contient un grand **PLAN** (ossature des murs et dalles + **cotation automatique**), une **vue isométrique** et **4 élévations** Nord / Est / Sud / Ouest (contenu partiel, marqué « à venir »).

Le **cartouche** est éditable et **persisté** avec le document : Projet, Client, Titre du dessin, No. feuille, Date, Dessiné par, **Échelle** (calculée, en lecture seule), **Nord (deg)**. S'y ajoutent un tableau **Révisions** (No. / Date / Description, avec ajout et suppression) et un bloc **Notes générales**.

L'**export PDF vectoriel** de la feuille se fait par le menu **Exporter → PDF (mise en plan)** ; le fichier est généré côté navigateur.

### 2.11 États d'enregistrement du document

Le badge à côté du menu Document indique l'état de la **sauvegarde automatique** (voir 3.14) :

| État | Libellé | Indication |
|------|---------|------------|
| Enregistré | « Enregistré » | coche verte |
| En cours | « Enregistrement... » | indicateur animé |
| Modifié | « Modifié » | horloge ambre |
| Échec | « Échec de sauvegarde » | alerte rouge |

---

## 3. Workflows pas-à-pas

### 3.1 Démarrer un nouveau plan

1. Ouvrir **CONCEPTION 3D → DAO**. Une scène de démonstration s'affiche.
2. Pour repartir de zéro : menu **Document → Nouveau** (ou bouton **Réinitialiser** à droite, avec confirmation).
3. Vérifier l'**étage actif** dans l'arbre de scène (badge « actif ») : tout ce que vous dessinez ira dans cet étage.
4. Au besoin, choisir les **unités** (menu Affichage → m / pi-po). Par défaut : **pieds-pouces**.

### 3.2 Tracer un mur

1. Cliquer l'outil **Mur**.
2. **1er clic** dans la vue : point de départ. Le curseur s'accroche aux points remarquables (coins, milieux), aux autres murs (aimant « Aligner ») et à la grille.
3. Déplacer : un aperçu du mur suit le curseur, avec sa longueur affichée.
4. **2e clic** : point d'arrivée. Le mur est créé.

**Saisie clavier de la longueur exacte** : pendant le tracé, **tapez la longueur** au clavier (un badge affiche la valeur). En pieds-pouces : `12-6` = 12 pi 6 po ; `8-2-7` = 8 pi 2 po 7/16. En métrique : `3.5`. Utilisez les **flèches** pour verrouiller la direction, puis **Entrée** pour poser le mur à la longueur exacte (**Échap** annule la saisie).

Réglez au besoin l'**Épaisseur** (Ext. 9 1/4" / Cloison 4 1/2") et la **Hauteur** (8'2" à 12'2") avant de tracer.

### 3.3 Tracer une pièce (rectangle de murs)

1. Cliquer l'outil **Rectangle**.
2. **1er clic** : premier coin. **2e clic** : coin opposé.
3. Quatre murs fermés sont créés d'un coup.

### 3.4 Mur en pente (dessus rampant)

1. Sélectionner le mur.
2. Dans **Propriétés**, régler la **Pente du dessus** (de 1/12 à 12/12) et le **côté le plus haut**.
3. Le mur reste vertical ; seule l'arête supérieure s'incline, en 2D comme en 3D.

### 3.5 Poser une dalle

1. Cliquer l'outil **Dalle**.
2. Cliquer successivement les sommets du polygone (plancher).
3. **Double-clic** ou **Entrée** ferme la dalle (minimum 3 sommets).
4. La surface est calculée automatiquement (affichée au centre si « Surfaces des dalles » est actif).

### 3.6 Poser une porte ou une fenêtre

1. Cliquer l'outil **Porte** ou **Fenêtre**.
2. **Cliquer directement sur un mur** (sa surface, en 3D ou en 2D) : l'ouverture est percée au point cliqué, avec un préréglage par défaut. Cliquer **hors** d'un mur ne fait rien.
3. Ajuster dans **Propriétés** : Type d'ouverture (préréglage), Largeur, Hauteur, « Position depuis la gauche ». Pour une fenêtre : **Allège** et **Aligner sur le linteau**.

### 3.7 Ouverture sur un pignon (toit-forme)

On peut poser des portes / fenêtres sur les **faces verticales** (murs de pignon) d'un toit-forme :

1. Outil **Porte** ou **Fenêtre** actif.
2. Cliquer une **face verticale** (pignon) du toit-forme. Cliquer sur la **pente** est refusé (message « Posez l'ouverture sur une face verticale (mur de pignon). »).
3. Éditer dans Propriétés : Type, Largeur, Hauteur, « Position le long de la face », « Hauteur sur la face ».

### 3.8 Ajouter un toit

**A. Toit auto (sur les murs)** — via le menu **Toit → Toit auto** :

1. Choisir **Toit → Toit auto**.
2. Cliquer dans la vue : un toit à versants est généré automatiquement sur **l'emprise des murs de l'étage actif** (squelette multi-pente, avec noues sur les plans en L).
3. Dans Propriétés : Type (2 versants / 4 versants / Plat), Pente, Débord, Fascia, et **Pignons par rive** (cases) — uniquement si l'empreinte est **rectangulaire**.

**B. Toit (forme)** — via le menu **Toit → Toit (forme)** :

1. Choisir **Toit → Toit (forme)**.
2. **1er clic** puis **2e clic** : tracer le rectangle d'emprise.
3. Dans Propriétés : choisir la **forme** parmi 7 (voir 4.5), régler Pente, Débord, Hauteur des murs, Largeur, Profondeur.
4. Le toit-forme se **déplace / pivote** avec une poignée 3D (bascule Déplacer / Pivoter, ou champs Position X / Z et Rotation).

### 3.9 Ajouter un escalier

1. Cliquer l'outil **Escalier**.
2. Cliquer dans la vue : un escalier est créé au point cliqué.
3. Dans Propriétés : Type de segment (**Volée** / **Palier**), Largeur, Longueur, Hauteur, **Marches**, Position et Rotation.

### 3.10 Placer du mobilier

1. Cliquer l'outil **Mobilier** : le **catalogue** s'ouvre.
2. Choisir une **catégorie** ou **rechercher** par nom.
3. Cliquer un meuble : il devient l'objet à poser et le catalogue se ferme.
4. **Cliquer dans la scène 3D** : le meuble est posé au sol au point cliqué.
5. Sélectionner le meuble pour afficher une **poignée 3D** (Déplacer / Pivoter) et régler Position X / Z / Y et Rotation. La **Fixation** (Sol / Mur / Plafond) est déterminée à la pose.

### 3.11 Gérer les étages

1. Bouton **Ajouter un étage** : crée un étage, le rend **actif** et le sélectionne.
2. Les étages sont **empilés** en 3D (hauteur d'étage par défaut 2,7 m, réglable dans Propriétés). En 2D / plan, seul l'**étage actif** est affiché.
3. Cliquer un étage dans l'arbre de scène le rend actif.

### 3.12 Naviguer en 3D

- **Orbite** (défaut) : glisser avec le bouton droit = tourner autour du modèle ; `Maj` = déplacement latéral.
- **Molette** : zoom ; ou les boutons **Zoom avant / arrière / Ajuster à l'écran**.
- **View cube** (coin de la vue) : cliquer une face = vue orthogonale ; un coin = vue isométrique. Un **compas N/E/S/O** au pied du cube tourne avec la vue.
- **Parallèle** : bascule perspective ↔ orthographique (vue 3D seulement).
- **Marche (1re personne)** : bouton **Marche** ; **cliquer la vue** verrouille la souris ; **flèches** ou **WASD / ZQSD** = avancer / reculer / pas latéraux (hauteur d'œil ~1,7 m) ; **Échap** pour sortir.

### 3.13 Matériaux et rendu réaliste

- **Matériau** : dans Propriétés, choisir un préréglage (Blanc, Bois, Brique, Béton, Plâtre, Tuile, Marbre, Métal, Verre) ou une **couleur personnalisée**.
- **Rendu réaliste** (menu Affichage, vue 3D, activé par défaut) : éclairage réaliste (ambiance et ciel procéduraux, ombre portée), le modèle **restant éditable**. Désactiver pour un rendu « à plat » plus rapide.
- **Arêtes (3D)** : surligne les contours des volumes (désactivé par défaut).

### 3.14 Enregistrer, ouvrir, renommer, supprimer un document

Tout passe par le menu **Document** :

- **Nouveau** : scène vierge.
- **Enregistrer** / **Enregistrer sous...** : si le document n'a pas de nom, une invite le demande.
- **Renommer** / **Supprimer** : sur le document courant (confirmation pour la suppression).
- **Ouvrir** : liste des documents de votre entreprise.
- **Sauvegarde automatique** : le document est enregistré en arrière-plan après ~2 s d'inactivité (l'état est affiché à côté du menu). Les documents sont stockés côté serveur, isolés par tenant.

### 3.15 Utiliser l'Assistant Dessin (IA)

1. Bouton **Assistant Dessin** : le panneau de chat s'ouvre.
2. Décrire ce que vous voulez (« Une pièce de 4 m par 5 m », « Ajoute un toit à deux versants sur les murs »…).
3. L'IA répond avec un **Aperçu du plan** (liste des opérations).
4. **Confirmer le dessin** applique le plan dans l'étage actif ; **Annuler** l'abandonne.

> Rappel : l'assistant v1 dessine des murs, des pièces et des toits ; il **consomme des crédits IA**.

### 3.16 Importer un plan (PDF / image) → 3D

1. Dans le panneau Assistant Dessin, bouton **« Importer un plan (PDF/image) → 3D »**.
2. Choisir un fichier (PDF, PNG, JPEG, GIF ou WEBP). L'IA de vision (Claude Opus) lit le plan d'étage, calcule une échelle et extrait la géométrie.
3. L'IA renvoie un **plan d'opérations** (murs / pièces / toit) à **confirmer** comme un dessin classique.

> Fonction **facturée** ; le résultat est un **brouillon à réviser** (un plan vectoriel donne de bien meilleurs résultats qu'un plan scanné). Distincte du module **PDF3D / Hover**.

### 3.17 Générer un rendu photoréaliste (Rendu IA)

1. Passer en **vue 3D** et cadrer la caméra.
2. Menu **Exporter → Rendu IA** : la modale s'ouvre avec l'aperçu de la vue.
3. (Facultatif) Ajouter une **description** (revêtement, toiture, ambiance, paysage…).
4. Choisir **Qualité** (Pro / Standard / Rapide) et **Résolution** (2K / 4K).
5. **Générer le rendu**, puis **Télécharger** ou **Régénérer**.

> **Facturation** : consomme des crédits IA ; un solde **strictement positif** est requis (sinon le rendu est refusé, code 402).

### 3.18 Mettre en plan et exporter en PDF

1. Onglet **Vue → Plan** : la feuille (ARCH D, 24 × 36 po, paysage) s'affiche.
2. Le plan au sol est tracé automatiquement (murs et dalles, cotations automatiques, symboles d'ouvertures, arêtes de toit, flèche du nord).
3. Remplir le **cartouche** (titre, numéro, date, dessinateur), les **Révisions** et les **Notes** : ces champs sont **persistés** avec le document.
4. Menu **Exporter → PDF (mise en plan)** : génère un PDF vectoriel de la feuille.

### 3.19 Exporter le modèle 3D

En **vue 3D**, menu **Exporter** :

- **GLB (.glb)** — modèle 3D complet (géométrie + matériaux).
- **STL (.stl)** — maillage, pour l'impression 3D.
- **OBJ (.obj)** — maillage + matériaux, interopérable.

### 3.20 Estimer le modèle et créer un devis

1. Se placer sans sélection (le panneau **Quantités** s'affiche).
2. Cliquer **« Estimer ce modèle »** : la modale d'estimation s'ouvre sur le résumé du métré.
3. **Générer l'estimation IA** : l'IA produit un tableau de lignes de devis (par corps de métier), majoré Administration 3 % / Contingences 12 % / Profit 15 %.
4. **Créer le devis**, puis **Ouvrir le devis** dans le module Ventes.

---

## 4. Référence

### 4.1 Endpoints backend (préfixe `/api/erp/v1/cao`)

Tous les endpoints exigent un compte du tenant authentifié ; aucun n'a de garde de rôle spécifique.

| Méthode + route | Rôle | Réf |
|---|---|---|
| `GET /cao/documents` | Liste **légère** des documents (id, nom, dates ; sans le modèle), triée par date de mise à jour | `cao.py:455` |
| `POST /cao/documents` | Créer un document (scène sérialisée) | `cao.py:461` |
| `GET /cao/documents/{id}` | Charger un document complet (404 si absent) | `cao.py:471` |
| `PUT /cao/documents/{id}` | Mise à jour partielle (nom et/ou modèle) — sert la **sauvegarde automatique** | `cao.py:483` |
| `DELETE /cao/documents/{id}` | Supprimer (404 si absent) | `cao.py:500` |
| `POST /cao/render` | **Rendu photoréaliste** de la capture 3D (Google Gemini). **Facturé.** | `cao.py:539` |
| `POST /cao/ai/chat` | Chat langage naturel → **plan d'opérations** (murs / pièces / toit). N'écrit jamais en base. **Facturé.** | `cao_ai.py:378` |
| `POST /cao/ai/plan-to-3d` | Import multipart d'un plan (PDF / image) → **vision** → plan d'opérations. **Facturé.** | `cao_ai.py:646` |

> Le préfixe backend est **`/cao`** (pas `/cao2`). La route `/cao2` et la redirection `/cao → /cao2` sont purement frontend.

### 4.2 Limites d'entrée (validations serveur)

| Domaine | Limite |
|---|---|
| Modèle (`model_json`) | **8 Mo** max, profondeur ≤ 64, ≤ 200 000 éléments (413 sinon) |
| Nom de document | 1 à 255 caractères |
| Rendu — image | 32 à 22 000 000 caractères en base64 ; ≤ **12 Mo** décodés (413) |
| Rendu — description | ≤ 2000 caractères |
| Rendu — qualité / résolution | qualité ∈ {pro, standard, fast} (défaut pro) ; résolution ∈ {1K, 2K, 4K} (défaut 2K) |
| Chat — message | 1 à 8000 caractères ; historique borné à 12 tours ; contexte de scène ≤ 6000 |
| Chat — plan | max 5 itérations d'outils, ≤ **50 opérations** |
| Import plan — fichier | requête ≤ **35 Mo** ; document revalidé à 32 Mo / 100 pages ; type ∈ {PDF, PNG, JPEG, GIF, WEBP} (422 sinon) |
| Import plan — plan | ≤ **120 opérations** |
| Bornes géométriques | dimension 0,1 à 200 m ; coordonnée ≤ 1000 m ; hauteur ≤ 20 m ; épaisseur ≤ 2 m |

### 4.3 Facturation IA (calculs)

Marge tenant appliquée à tous les appels : **× 1,30**.

| Appel | Coût (avant marge) | Particularité |
|---|---|---|
| **Rendu IA** (`/cao/render`) | jetons image × prix/jeton (réel), sinon forfait par palier (Pro / Standard / Rapide) | Solde **strictement positif** requis (402) ; endpoint **partagé** avec le module Rendu 3D |
| **Chat de dessin** (`/cao/ai/chat`) | (entrée × 0,003 + sortie × 0,015) / 1000 | Modèle Sonnet ; tolère un solde négatif (post-payé) |
| **Import plan** (`/cao/ai/plan-to-3d`) | (entrée × 0,005 + écriture cache × 0,00625 + lecture cache × 0,0005 + sortie × 0,025) / 1000 | Modèle Opus, tarif qui tient compte du cache |
| **Estimation → devis** | facturée par l'IA d'estimation du module Devis | Majorations fixes Administration 3 % / Contingences 12 % / Profit 15 % |

> L'endpoint `/cao/render` est **le même code** que celui du module **Rendu 3D** (`/rendu-3d`) : le champ `source` ne change que le libellé de facturation (`cao_render` ou `render3d`). Il n'y a **pas de clé d'idempotence serveur** sur les endpoints payants ; le verrou anti-double-clic du rendu est **côté navigateur**.

### 4.4 Limites de débit (par adresse IP)

| Endpoint | Limite |
|---|---|
| `POST /cao/render` | 10 par fenêtre (+ garde de taille du corps avant traitement) |
| `POST /cao/ai/chat` | 20 par fenêtre |
| `POST /cao/ai/plan-to-3d` | **6 par fenêtre** (le plus bas : appel de vision le plus coûteux) |
| `GET/POST/PUT/DELETE /cao/documents*` | tranche générale (1500 par fenêtre) |

### 4.5 Types de toit

**Toit (forme)** — 7 formes (libellés exacts de la liste déroulante `Cao2PropertiesPanel.tsx:1107-1113`) :

| Valeur | Libellé affiché | Description |
|--------|-----------------|-------------|
| flat | Plat | Toit plat. |
| hip | 4 versants (croupe) | Quatre pentes. |
| gable | 2 versants | Deux pentes, pignons aux extrémités. |
| shed | Appentis (1 versant) | Une seule pente. |
| gambrel | Mansardé (gambrel) | Toit de grange, 2 pentes par versant. |
| dutch | Croupe hollandaise | Croupe surmontée d'un petit pignon. |
| mansard | Mansardé (4 versants) | Quatre versants à double pente. |

**Toit auto (sur les murs)** — 3 formes seulement : **2 versants** (gable), **4 versants (croupe)** (hip), **Plat** (flat).

**Pignons par rive** : sur un **Toit auto** à empreinte **rectangulaire** (4 arêtes) et non plat, des cases « Rive 1 », « Rive 2 »… désignent les rives en pignon (par paire opposée). Sur une empreinte en L / T / U, l'option n'est pas disponible : la note « Pignon disponible sur empreinte rectangulaire. Pour une forme en L, posez un toit-segment (« Toit (forme) ») par aile. » vous guide. Voir 5.2.

### 4.6 Préréglages de mur (valeurs Québec)

**Épaisseur** :

| Préréglage | Libellé | Valeur |
|------------|---------|--------|
| Extérieur (défaut) | « Ext. 9 1/4" » | 0,235 m (9 1/4 po) |
| Intérieur | « Cloison 4 1/2" » | 0,114 m (4 1/2 po) |

**Hauteur** (défaut 8'2") :

| Libellé | Valeur |
|---------|--------|
| 8'2" | 2,489 m |
| 9'2" | 2,794 m |
| 10'2" | 3,099 m |
| 11'2" | 3,404 m |
| 12'2" | 3,708 m |

### 4.7 Préréglages d'ouvertures

**Portes** : Porte intérieure ; Porte extérieure ; Porte extérieure 8 pi ; Porte de garage simple ; Porte de garage double ; Porte de garage haute ; Porte-patio coulissante.

**Fenêtres** : Fenêtre de chambre ; Fenêtre de salle de bain ; Fenêtre de salle de bain haute ; Fenêtre de sous-sol ; Fenêtre panoramique ; Fenêtre plancher-plafond.

Chaque préréglage fixe une largeur et une hauteur de départ, ajustables ensuite dans Propriétés.

### 4.8 Matériaux et mobilier

- **Matériaux** : Blanc, Bois, Brique, Béton, Plâtre, Tuile, Marbre, Métal, Verre, plus une **couleur personnalisée**.
- **Mobilier** : 110 objets 3D (GLB), catégories Mobilier, Électroménager, Salle de bain, Cuisine, Extérieur ; chargés depuis un **CDN externe** (dépendance réseau).
- **Assemblages de mur** : « Mur extérieur à déclin (R-24.5) », « Cloison 2x4 » (avec couches, épaisseur totale et R total).

### 4.9 Unités et formats

- **Impérial (pi-po)** — **défaut** : pieds-pouces (et seizièmes). Saisie acceptée sous les formes `8-2-7`, `8-2`, `8`.
- **Métrique (m)** : mètres et mètres carrés.
- Bascule dans le menu **Affichage → Unités**.
- Feuille de mise en plan : format papier **ARCH D** (24 × 36 po), paysage.

### 4.10 Raccourcis clavier

| Touche | Effet |
|--------|-------|
| `Échap` | Revenir à l'outil **Sélectionner** ; annuler un tracé en cours ; fermer un menu ou une fenêtre ; sortir du mode marche. |
| `Suppr` / `Retour arrière` | Supprimer le(s) élément(s) sélectionné(s) (protégé pour Site et Bâtiment ; ignoré si le curseur est dans un champ). |
| `Entrée` | Poser le mur à la longueur saisie ; fermer une dalle ; envoyer un message dans le chat IA. |
| `Maj + Entrée` | Nouvelle ligne dans le chat IA. |
| Chiffres | Saisir la longueur exacte d'un mur (outil Mur). |
| Flèches | Verrouiller la direction d'un mur (tracé) ; se déplacer en mode marche. |
| `WASD` / `ZQSD` | Se déplacer en mode marche. |
| Molette | Zoom. |
| Bouton droit + glisser | Orbite (vue 3D) ; `Maj` = déplacement latéral. |

---

## 5. Intégrations et FAQ

### 5.1 Intégrations avec les autres modules

- **Ventes / Devis (module 07)** : le bouton **« Estimer ce modèle »** dérive un devis à partir du métré 3D (numéro `DEV-AAAA-NNN`), puis « Ouvrir le devis » ouvre le module Ventes. Le pont est assuré par la composition d'endpoints existants du module Devis (aucun endpoint dédié au DAO).
- **Métré (module 32)** : outil **distinct**. Le Métré fait de la **prise de quantités sur un plan PDF (2D)** ; le DAO **construit un modèle 3D** et en dérive son propre métré. Les deux partagent des utilitaires (saisie impériale, cotation).
- **PDF3D / Hover** : module **séparé** (voisin dans la section CONCEPTION 3D) qui convertit un plan en 3D via le service externe Hover. À ne pas confondre avec la fonction « Importer un plan → 3D » du DAO (qui, elle, utilise l'IA de vision Claude).
- **Rendu 3D (`/rendu-3d`)** : module séparé qui réutilise **le même endpoint** de rendu que le DAO. Le résultat et la facturation sont de même nature.

### 5.2 Limites connues (vérifiées dans le code)

- **Bannière « en développement »** : le module évolue encore.
- **Assistant IA (v1)** : dessine **murs, pièces, toits** uniquement (pas d'ouvertures, escaliers ni mobilier) ; aperçu **textuel** ; mono-conversation dans la session.
- **Outil « Cote »** : la cotation est **automatique** sur le plan / la feuille ; l'outil de cote manuelle n'est pas encore complet.
- **Vue Plan (feuille)** : **statique** (aucun panoramique ni zoom). Seul le **plan au sol** est pleinement rendu ; les **élévations** (N / E / S / O) et la **vue isométrique** sont en partie « à venir ». Marqueurs de coupe / détail et multi-feuilles : non implémentés.
- **Pignons de toit** : fiables sur empreinte **rectangulaire** seulement. Sur un plan en L / T / U, le module produit une **croupe** automatique ; pour de vrais pignons, posez un **Toit (forme)** par aile.
- **Ouvertures de toit** : posables uniquement sur les **faces verticales (pignon)** d'un toit-forme ; un clic sur la pente est refusé. Elles sont visibles en 3D et sur la feuille / PDF, mais pas encore dans la vue 2D interactive.
- **Mobilier** : dépend d'un **CDN externe** (Supabase public) ; sans accès réseau à ce CDN, les modèles ne se chargent pas.
- **Idempotence** : pas de clé d'idempotence serveur sur les endpoints payants ; le verrou anti-double-facturation est côté navigateur.

### 5.3 FAQ

**Q. Où est passé l'ancien module CAO (`/cao`) ?**
Il a été remplacé par le DAO (`/cao2`) ; l'ancienne adresse redirige automatiquement.

**Q. Quelle est l'unité par défaut ?**
Le **pied-pouce** (impérial). On bascule en mètres via le menu Affichage → Unités.

**Q. Je clique pour percer une fenêtre et rien ne se passe.**
Cliquez **directement sur la surface d'un mur** (pas dans le vide). Sur un toit-forme, cliquez une **face verticale** (pignon) — la pente est refusée.

**Q. Mon dessin est-il enregistré automatiquement ?**
Oui, ~2 s après votre dernière action (voir l'état à côté du menu Document). Vous pouvez aussi enregistrer manuellement.

**Q. Le rendu IA ou l'estimation sont-ils gratuits ?**
Non : le **Rendu IA**, l'**Importer un plan → 3D**, l'**Assistant Dessin** et l'**Estimation → devis** consomment des **crédits IA** de votre entreprise (refacturés avec une marge de 30 %). Le rendu exige en plus un solde **strictement positif**.

**Q. Puis-je faire dessiner tout mon bâtiment par l'IA ?**
Partiellement : l'assistant v1 dessine murs, pièces et toits. Les ouvertures, escaliers et meubles se posent à la main (outils dédiés).

**Q. Pourquoi mon toit devient-il plat quand je coche une rive en pignon sur un plan en L ?**
Les pignons par rive ne sont fiables que sur une empreinte **rectangulaire**. Sur un L / T / U, utilisez l'outil **Toit (forme)** par aile, ou laissez la **croupe** automatique.

**Q. Le nouvel étage ne reçoit pas mes murs.**
Vérifiez l'**étage actif** dans l'arbre de scène (badge « actif ») : les nouveaux éléments vont dans l'étage actif. Cliquez l'étage voulu pour l'activer.

**Q. Comment obtenir une image réaliste pour un client ?**
Vue 3D → Exporter → Rendu IA → choisir qualité / résolution → Générer → Télécharger. Un solde de crédits IA est requis.

**Q. Comment récupérer mon modèle dans un autre logiciel ?**
Vue 3D → Exporter → GLB / STL / OBJ.

**Q. La mise en plan comporte-t-elle les élévations ?**
Le plan au sol est complet et coté ; les élévations et la vue isométrique sont en partie réservées (« à venir »).

---

## 6. Récapitulatif

| Élément | Détail |
|---|---|
| **Accès** | Menu latéral → CONCEPTION 3D → **DAO** (`/cao2`) |
| **But** | Modéliser un bâtiment en 3D → métré, mise en plan PDF, rendu IA, devis, export 3D |
| **Vues** | 3D (`Cao2Scene`), 2D (`Cao2Plan2D`), Plan / feuille **statique** (`Cao2Sheet`) |
| **Dessin** | Murs, Rectangle (pièce), Dalle, Toit (auto / forme), Porte, Fenêtre, Escalier, Mobilier (110 objets), Cote, Ajuster murs, Esquisse (2D) |
| **Toit** | 7 formes en « Toit (forme) » ; 3 formes en « Toit auto » ; pignons par rive sur rectangle seulement |
| **Unités** | Impérial pi-po (défaut) ou métrique ; saisie clavier de longueur exacte |
| **Métré → devis** | « Estimer ce modèle » → IA → majorations fixes 3 % / 12 % / 15 % → devis (module Ventes) |
| **Rendu IA** | Modale Rendu (Gemini) ; qualité Pro / Standard / Rapide ; 2K / 4K ; solde positif requis |
| **Assistant IA** | Chat de dessin (Sonnet) v1 = murs / pièces / toit ; Import plan → 3D (Opus, vision) |
| **Persistance** | `cao_documents` **par tenant** ; sauvegarde automatique ~2 s ; export GLB / STL / OBJ, PDF vectoriel |
| **Facturation** | Rendu, chat, import plan, estimation = **crédits IA × 1,30** |
| **Backend** | 8 endpoints, préfixe `/api/erp/v1/cao` ; aucune garde de rôle propre |

**À retenir :**

- Le module est marqué **« en développement »** : fonctionnel, mais en évolution.
- Vérifiez toujours l'**étage actif** avant de dessiner : les nouveaux éléments y sont créés.
- Quatre fonctions sont **payantes en crédits IA** : Rendu, Import plan → 3D, Assistant Dessin, Estimation → devis.
- L'assistant IA v1 se limite aux **murs, pièces et toits** ; le reste se dessine à la main.
- **Toit auto** (3 formes) ≠ **Toit (forme)** (7 formes) ; les **pignons par rive** ne sont fiables que sur empreinte **rectangulaire**.
- Le préfixe backend réel est **`/cao`** ; `/cao2` est la route frontend.
- Le module partage son endpoint de **rendu** avec le module **Rendu 3D** et se distingue du **Métré** (module 32) et de **PDF3D / Hover**.

---

> Manuel généré à partir du code source vérifié : `frontend/src/pages/Cao2Page.tsx`, `frontend/src/components/cao2/` (barre d'outils, `Cao2Scene`, `Cao2Plan2D`, `Cao2Sheet`, panneaux, menus, catalogue), `frontend/src/i18n/locales/fr/cao2.json`, `backend/routers/cao.py`, `backend/routers/cao_ai.py`, `backend/gemini_image.py`. Manuels liés : **08 — Soumissions** (devis dérivé), **30 — Métré (prise de quantités sur PDF)**, module **PDF3D / Hover** et module **Rendu 3D**.
