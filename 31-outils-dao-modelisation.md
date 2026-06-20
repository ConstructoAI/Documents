# Manuel utilisateur — Module DAO / Modélisation 3D

Version v2.0 vérifiée — Constructo AI ERP React.

Ce manuel décrit le module **DAO** (Dessin Assisté par Ordinateur), aussi appelé **Modélisation**, accessible via la route `/cao2` du frontend React. C'est un atelier de modélisation 3D et de mise en plan 2D de bâtiment : on y trace des murs, des dalles, des toits, des ouvertures (portes/fenêtres), des escaliers et du mobilier, puis on produit une perspective 3D, un plan 2D et une feuille de mise en plan exportable en PDF. Le module embarque aussi un **assistant IA de dessin** (langage naturel) et un **rendu photoréaliste IA**.

Toutes les informations ci-dessous proviennent de la lecture directe du code source : `frontend/src/pages/Cao2Page.tsx`, l'ensemble du dossier `frontend/src/components/cao2/` (barre d'outils, scène 3D, plan 2D, feuille, panneaux, menus), `frontend/src/i18n/locales/fr/cao2.json` (libellés) et `backend/routers/cao.py` + `backend/routers/cao_ai.py` (persistance, rendu IA, assistant).

> **Note de version** : le module affiche une bannière « DAO - Atelier de construction (en développement) ». Les fonctions décrites ici sont en place et fonctionnelles ; quelques limites connues sont listées à la section 5.

---

## 1. Vue d'ensemble et accès

### 1.1 Rôle du module

Le module DAO permet de **dessiner un bâtiment en 3D directement dans le navigateur**, sans logiciel externe. Il s'adresse à l'estimateur, au concepteur et au chargé de projet qui veulent :

- modéliser rapidement une enveloppe (murs, dalles, toit) et son aménagement (portes, fenêtres, escaliers, mobilier) ;
- obtenir automatiquement des **surfaces** (pièces, dalles) et des **longueurs** (murs) pour un métré sommaire ;
- produire une **mise en plan 2D** (plan au sol coté, cartouche, échelle) exportable en **PDF** ;
- générer une **image de présentation photoréaliste** par IA pour un client ;
- exporter le modèle 3D (GLB / STL / OBJ) vers un autre logiciel.

Le modèle reste **éditable en tout temps** : la 3D, le plan 2D et la feuille partagent le même modèle.

### 1.2 Accès

- **Menu** : section **OUTILS** de la barre latérale → **DAO** (icône cube).
- **Route** : `/cao2`. L'ancienne route `/cao` redirige automatiquement vers `/cao2` (`App.tsx:242`).
- **Composant principal** : `Cao2Page` (`frontend/src/pages/Cao2Page.tsx`).
- **Plein écran** : un bouton de la barre d'outils bascule l'atelier en plein écran (`100vh`) ; hauteur normale `82vh`.
- Page protégée : authentification requise (compte ERP du locataire).

### 1.3 Les trois vues

L'atelier propose trois vues du **même modèle**. On bascule depuis le groupe « Vue » de la barre d'outils. La vue par défaut est la **3D**.

| Vue | Libellé | Contenu | Composant |
|-----|---------|---------|-----------|
| **3D** | « 3D » | Perspective 3D temps réel : dessin, sélection, navigation (orbite / view cube / mode marche), rendu réaliste, rendu IA, export 3D. | `Cao2Scene` |
| **2D** | « 2D » | Plan vue de dessus interactif (SVG) : dessin et sélection en 2D pur, esquisse (linework) disponible. | `Cao2Plan2D` |
| **Plan** | « Plan » | Feuille de mise en plan statique (format papier ARCH D) : plan au sol coté, cartouche, notes, export PDF. | `Cao2Sheet` |

### 1.4 Disposition de l'atelier

```
+----------------------------------------------------------------------+
| Bannière « DAO - Atelier de construction (en développement) »        |
+----------------------------------------------------------------------+
| BARRE D'OUTILS (Fichier | Outils | Réglages | Historique | Niveaux   |
|                 ... | Vue | Affichage | Navigation | Plein écran)     |
+------------------+--------------------------------+------------------+
| ARBRE DE SCÈNE   |   VUE CENTRALE                 | PROPRIÉTÉS       |
| (gauche)         |   3D / 2D / Feuille            | (droite)         |
|                  |                                |                  |
| Site             |                                | [si sélection]   |
|  Bâtiment        |        modèle du bâtiment       | champs éditables |
|   Étage (actif)  |                                | [sinon]          |
|    Murs, Dalles  |                                | Quantités        |
|    Toit, Escalier|                                |                  |
+------------------+--------------------------------+------------------+
        (Panneau « Assistant Dessin » s'ouvre à droite, en option)
```

- **Arbre de scène** (gauche) : hiérarchie complète du modèle, sélection, visibilité, étage actif.
- **Vue centrale** : 3D, 2D ou Feuille selon l'onglet « Vue ».
- **Panneau de propriétés** (droite) : champs de l'élément sélectionné ; si rien n'est sélectionné, il affiche le **métré sommaire** (Quantités).
- **Panneau Assistant Dessin** (droite, optionnel) : chat IA, ouvert par le bouton « Assistant Dessin ».

### 1.5 Concepts clés

- **Hiérarchie du modèle** : `Site` → `Bâtiment` → `Étage` → éléments (`Mur`, `Dalle`, `Toit`, `Escalier`). Une `Porte` ou une `Fenêtre` est un enfant d'un `Mur` (ou d'une face de toit-segment).
- **Étage actif** : tout nouvel élément est créé dans l'**étage actif**. On rend un étage actif en cliquant dessus dans l'arbre de scène. Un badge « (actif) » l'indique.
- **Modèle partagé** : le modèle vit dans un store central (`useScene`, cœur Pascal). Les trois vues lisent le même modèle ; une modification est visible partout.
- **Interaction** : l'outil actif, la sélection, l'étage actif et la vue vivent dans le store d'atelier (`useCao2Ui`) ; les options d'affichage (cotes, unités, rendu réaliste) dans `useCao2Overlay`.

---

## 2. Interface

### 2.1 La barre d'outils

La barre d'outils (`Cao2Toolbar`) regroupe les actions **par intention**, séparées par de fins traits verticaux. Sur petit écran elle passe sur plusieurs lignes (pas de défilement horizontal). Les groupes, de gauche à droite :

```
[Fichier ▾] [Exporter ▾] [Assistant Dessin] | [Sélectionner][Mur][Rectangle][Dalle]
[Toit ▾][Porte][Fenêtre][Escalier][Mobilier][Cote][Ajuster murs][Esquisse ▾]
| [Angle 15°][Aligner] [Épaisseur: Ext/Cloison] [Hauteur ▾]
| [Annuler][Rétablir] | [Ajouter un étage]
                                  ... (espace) ...
[3D][2D][Plan][Parallèle][Marche] | [Affichage ▾] | [Zoom+][Zoom−][Ajuster] | [Plein écran]   [Réinitialiser]
```

| Groupe | Contenu |
|--------|---------|
| **Fichier** | Menu **Document** (nouveau / ouvrir / enregistrer / renommer / supprimer + état d'enregistrement) ; menu **Exporter** ; bouton **Assistant Dessin** (IA). |
| **Outils** | Sélectionner, Mur, Rectangle, Dalle, menu **Toit**, Porte, Fenêtre, Escalier, Mobilier, Cote, Ajuster murs, menu **Esquisse**. |
| **Réglages de tracé** | Angle 15°, Aligner, Épaisseur (préréglages mur), Hauteur (préréglages mur). |
| **Historique** | Annuler, Rétablir. |
| **Niveaux** | Ajouter un étage. |
| **Vue** | 3D, 2D, Plan, Parallèle (projection), Marche (1re personne). |
| **Affichage** | Menu des bascules d'affichage + unités. |
| **Navigation** | Zoom avant, Zoom arrière, Ajuster à l'écran. |
| **Plein écran** / **Réinitialiser** | Bascule plein écran ; réinitialiser la scène (à droite, isolé). |

Le détail de chaque outil et de chaque menu est en **section 4 (Référence)**.

### 2.2 Arbre de scène (panneau gauche)

L'arbre (`Cao2SceneTree`, titre « Arbre de scène ») affiche tout le modèle sous forme d'arborescence dépliable.

- **Sélection** : clic = sélectionne l'élément ; `Ctrl`/`Cmd` + clic = multi-sélection ; clic dans le vide = désélectionne.
- **Clavier** (ligne ciblée) : `Entrée`/`Espace` = sélectionner ; flèches haut/bas = naviguer ; flèche droite/gauche = déplier/replier.
- **Visibilité** : un bouton œil (icône `Eye` / `EyeOff`) masque ou affiche un élément.
- **Étage actif** : cliquer un étage le rend **actif** (cible des nouveaux éléments) et le sélectionne ; un badge « (actif) » le distingue.
- **Garde-fou** : les conteneurs racines `Site` et `Bâtiment` ne sont pas supprimables au clavier (la suppression est en cascade).

### 2.3 Panneau de propriétés (panneau droit)

Le panneau « Propriétés » (`Cao2PropertiesPanel`) affiche les champs **selon le type** d'élément sélectionné (voir section 4 pour le détail par type). Tous les éléments ont un champ **Nom** et un bouton **Supprimer**.

Si **aucun** élément n'est sélectionné, le panneau bascule sur le **métré sommaire** (`Cao2QuantitiesPanel`) : nombre et total pour Pièces, Murs, Dalles, Toits, Portes, Fenêtres, Escaliers.

> Le métré indique : « Métré sommaire dérivé du modèle. L'intégration au devis viendra dans une prochaine étape. »

### 2.4 Panneau Assistant Dessin (optionnel)

Ouvert par le bouton « Assistant Dessin » de la barre d'outils. C'est un chat : on décrit en français ce qu'on veut dessiner, l'IA propose un **plan d'opérations** (aperçu), et le dessin n'est appliqué qu'après **confirmation**. Détail en section 3.15.

---

## 3. Workflows pas-à-pas

### 3.1 Démarrer un nouveau plan

1. Ouvrir **OUTILS → DAO**. Une scène de démonstration s'affiche.
2. Pour repartir de zéro : menu **Fichier → Nouveau** (ou bouton **Réinitialiser** à droite, qui demande confirmation).
3. Vérifier l'**étage actif** dans l'arbre de scène (badge « (actif) »). Tout ce que vous dessinez ira dans cet étage.
4. Au besoin, choisir les **unités** (menu Affichage → m / pi-po). Par défaut : **pieds-pouces**.

### 3.2 Tracer un mur

1. Cliquer l'outil **Mur**.
2. **1er clic** dans la vue : point de départ. Le curseur s'accroche aux points remarquables (coins, milieux), aux autres murs (aimant « Aligner ») et à la grille.
3. Déplacer : un aperçu du mur suit le curseur, avec sa longueur affichée.
4. **2e clic** : point d'arrivée → le mur est créé.

**Saisie clavier de la longueur exacte** (très pratique) :

- Pendant le tracé, **tapez la longueur** au clavier. Un badge affiche la valeur saisie.
  - En pieds-pouces : format `pi-po` ou `pi-po-seizièmes` (ex. `12-6` = 12 pi 6 po ; `8-2-7` = 8 pi 2 po 7/16).
  - En métrique : ex. `3.5`.
- Utilisez les **flèches** du clavier pour **verrouiller la direction** (parallèle à un axe).
- **Entrée** : pose le mur à la longueur exacte. **Échap** : annule la saisie.

**Aides au tracé** (groupe Réglages, activées par défaut) :

- **Angle 15°** : contraint l'angle du tracé par pas de 15°.
- **Aligner** : aimant d'alignement orthogonal sur les autres murs.

**Épaisseur et hauteur** du nouveau mur (groupe Réglages) :

- **Épaisseur** : préréglages « Ext. 9 1/4" » (mur extérieur, défaut) et « Cloison 4 1/2" » (mur intérieur).
- **Hauteur** : liste déroulante des hauteurs standard QC (8'2" à 12'2"; défaut 8'2").

### 3.3 Tracer un rectangle de murs

1. Cliquer l'outil **Rectangle**.
2. **1er clic** : premier coin. **2e clic** : coin opposé.
3. Quatre murs fermés sont créés d'un coup (idéal pour démarrer une pièce).

### 3.4 Mur en pente (dessus rampant)

Pour un mur dont le dessus suit une pente (ex. sous un toit) :

1. Sélectionner le mur.
2. Dans **Propriétés**, régler la **Pente du dessus** (de 1/12 à 12/12) et le **côté le plus haut**.
3. Le mur reste vertical ; seule l'arête supérieure s'incline, en 2D comme en 3D.

### 3.5 Poser une dalle

1. Cliquer l'outil **Dalle**.
2. Cliquer successivement les sommets du polygone (plancher).
3. **Double-clic** ou **Entrée** ferme la dalle (minimum 3 sommets).
4. La surface est calculée automatiquement (affichée au centre si « Surfaces des dalles » est actif).

### 3.6 Afficher les pièces et leurs surfaces

Le module **détecte automatiquement** les pièces (polygones fermés délimités par les murs) et affiche leur **surface** sur le plan.

- Bascule « Pièces (surfaces) » dans le menu **Affichage** (active par défaut).
- Les surfaces s'affichent surtout en vue 2D / Feuille.

### 3.7 Poser une porte ou une fenêtre

1. Cliquer l'outil **Porte** ou **Fenêtre**.
2. **Cliquer directement sur le mur** (sur sa surface, en 3D ou en 2D) à l'endroit voulu : l'ouverture est percée à ce point avec un préréglage par défaut.
   - Cliquer **hors** d'un mur ne fait rien.
3. Ajuster ensuite dans **Propriétés** :
   - **Type d'ouverture** (préréglage) — voir la liste en section 4.4 ;
   - **Largeur**, **Hauteur** ;
   - **Position le long** du mur ;
   - **Allège** / **hauteur sur la face** (fenêtres).

> Astuce : si vous deviez auparavant cliquer plusieurs fois pour percer un trou, ce n'est plus le cas — la pose se fait au point cliqué sur le mur lui-même.

### 3.8 Ajouter un toit

Deux approches, via le menu **Toit** de la barre d'outils :

**A. Toit auto (sur les murs)** — `Toit auto (sur les murs)`

1. Choisir **Toit → Toit auto**.
2. Cliquer dans la vue : un toit à versants est généré automatiquement sur **l'emprise des murs de l'étage actif** (squelette multi-pente, avec noues sur les plans en L).

**B. Toit (forme)** — `Toit (forme)`, 7 formes paramétriques

1. Choisir **Toit → Toit (forme)**.
2. **1er clic** puis **2e clic** : tracer le rectangle d'emprise du toit.
3. Choisir la forme dans **Propriétés** (voir section 4.5) : Plat, 2 versants, 4 versants (croupe), Appentis, Gambrel, Mansarde, Demi-croupe.
4. Régler **pente**, **débord**, **hauteur de mur**, **fascia** dans Propriétés.
5. Un **toit-forme** se déplace / pivote avec une poignée 3D (bascule Déplacer / Pivoter dans Propriétés).

### 3.9 Ouvertures sur un pignon (toit-forme)

On peut poser des portes/fenêtres sur les **faces verticales** (murs de pignon) d'un toit-forme :

1. Outil **Porte** ou **Fenêtre** actif.
2. Cliquer sur une **face verticale** (pignon) du toit-forme.
   - Cliquer sur la **pente** du toit est refusé (message « Posez l'ouverture sur une face verticale (mur de pignon) »).
3. Éditer l'ouverture dans Propriétés (largeur, hauteur, position le long de la face, hauteur sur la face).

### 3.10 Ajouter un escalier

1. Cliquer l'outil **Escalier**.
2. Cliquer dans la vue : un escalier droit est créé au point cliqué.
3. Dans Propriétés : nombre de **marches** et **type de segment** (Volée droite / Palier).

### 3.11 Placer du mobilier

1. Cliquer l'outil **Mobilier** : le **catalogue** s'ouvre.
2. Choisir une **catégorie** (Mobilier, Électroménager, Salle de bain, Cuisine, Extérieur) ou **rechercher** par nom.
3. Cliquer un meuble : il devient l'objet à poser et le catalogue se ferme.
4. **Cliquer dans la scène 3D** : le meuble est posé au sol au point cliqué.
5. Sélectionner le meuble pour afficher une **poignée 3D** : bascule **Déplacer** / **Pivoter** dans Propriétés.

> Astuce : `Maj` + action sur un meuble agit sur tous les meubles du même type (en 3D).

### 3.12 Gérer les étages

1. Bouton **Ajouter un étage** (groupe Niveaux) : crée un nouvel étage, le rend **actif** et le sélectionne.
2. Les étages sont **empilés** en 3D (hauteur d'étage par défaut 2,7 m, réglable). En 2D / plan, seul l'**étage actif** est affiché.
3. Cliquer un étage dans l'arbre de scène le rend actif.

### 3.13 Naviguer en 3D

- **Orbite** (défaut) : glisser avec le bouton droit de la souris = tourner autour du modèle ; `Maj` = déplacement latéral (pan).
- **Molette** : zoom ; boutons **Zoom avant / arrière / Ajuster à l'écran** (groupe Navigation).
- **View cube** (coin de la vue, façon Inventor) : cliquer une face = vue orthogonale ; coin = vue isométrique.
- **Compas N/E/S/O** : au pied du view cube, tourne avec la vue.
- **Projection parallèle** : bouton **Parallèle** = bascule perspective ↔ orthographique (sans distorsion). Vue 3D seulement.
- **Mode Marche (1re personne)** : bouton **Marche**.
  - Souris = regarder ; **flèches** ou **WASD/ZQSD** = avancer / reculer / pas latéraux ; hauteur d'œil ~1,7 m.
  - **Cliquer la vue** pour verrouiller la souris ; **Échap** pour sortir.

### 3.14 Matériaux et apparence

- **Matériau d'un élément** : dans Propriétés, choisir un **préréglage** (Blanc, Bois, Brique, Béton, Plâtre, Tuile, Marbre, Métal, Verre) ou une **couleur personnalisée**.
- **Rendu réaliste** (menu Affichage, vue 3D, **activé par défaut**) : éclairage réaliste (tone mapping ACES, éclairage d'ambiance et ciel procéduraux, ombre portée). Le modèle **reste éditable**. Désactiver pour un rendu « à plat » plus rapide.
- **Arêtes (3D)** (menu Affichage) : surligne les contours des volumes ; désactivé par défaut.

### 3.15 Utiliser l'Assistant Dessin (IA)

1. Bouton **Assistant Dessin** : le panneau de chat s'ouvre à droite.
2. Décrire ce que vous voulez, par ex. :
   - « Une pièce de 4 m par 5 m »
   - « Ajoute un toit à deux versants sur les murs »
   - « Un mur de (0,0) à (6,0) »
3. L'IA répond avec un **Aperçu du plan** (liste des opérations).
4. **Confirmer le dessin** applique le plan dans l'étage actif ; **Annuler** l'abandonne.

> **Avertissement affiché** : « L'IA peut se tromper ; vérifie l'aperçu avant de confirmer. »
> **Périmètre v1** : l'assistant dessine des **murs, des pièces et des toits** (pas encore les ouvertures, escaliers, mobilier). L'aperçu est **textuel** (pas de fantôme 3D).

### 3.16 Générer un rendu photoréaliste (Rendu IA)

Transforme la vue 3D courante en image photoréaliste, par IA (utile pour présenter un projet à un client).

1. Passer en **vue 3D** et cadrer la caméra.
2. Menu **Exporter → Rendu IA** : une fenêtre s'ouvre avec l'aperçu de la vue capturée.
3. (Optionnel) Ajouter une **description** (revêtement, toiture, ambiance, paysage…).
4. Choisir la **qualité** (Pro / Standard / Rapide) et la **résolution** (2K / 4K ; 4K réservé à Pro/Standard).
5. **Générer le rendu**. À la fin : **Télécharger** l'image ou **Régénérer**.

> **Facturation** : le rendu IA consomme des **crédits IA** du locataire (un solde de crédit **strictement positif** est requis ; sinon le rendu est refusé). Le coût est refacturé avec une marge. Voir section 4.10.

### 3.17 Mettre en plan (vue Feuille) et exporter en PDF

1. Onglet **Vue → Plan**. La feuille (format **ARCH D**, 24×36 po, paysage) s'affiche.
2. Le **plan au sol** est tracé automatiquement : murs et dalles en poché, **cotations automatiques**, **symboles d'ouvertures** (porte = trouée + battant + arc ; fenêtre = vitrage), arêtes de toit, flèche du nord.
3. Remplir le **cartouche** (titre, numéro, échelle, date, dessinateur) et les **notes** : les champs sont éditables et **persistés** avec le document.
4. Menu **Exporter → PDF (mise en plan)** : génère un **PDF vectoriel** de la feuille.

> L'échelle est choisie automatiquement pour cadrer le plan ; le numéro/échelle restent éditables dans le cartouche.

### 3.18 Exporter le modèle 3D

En **vue 3D**, menu **Exporter** :

- **GLB (.glb)** — modèle 3D complet (géométrie + matériaux), pour visionneuses et autres logiciels.
- **STL (.stl)** — maillage, pour impression 3D.
- **OBJ (.obj)** — maillage + matériaux, interopérable.

### 3.19 Enregistrer, ouvrir, renommer, supprimer un document

Tout passe par le menu **Fichier (Document)** :

- **Nouveau** : scène vierge.
- **Enregistrer** : si le document n'a pas de nom, une invite le demande ; sinon sauvegarde directe.
- **Renommer** / **Supprimer** : sur le document courant (confirmation pour la suppression).
- **Ouvrir** : liste des documents du locataire.
- **Autosave** : enregistrement automatique en arrière-plan après ~2 s d'inactivité.

L'**état d'enregistrement** est affiché à côté du menu :

| État | Libellé | Indication |
|------|---------|------------|
| Enregistré | « Enregistré » | coche verte |
| En cours | « Enregistrement... » | indicateur animé |
| Modifié | « Modifié » | horloge ambre |
| Échec | « Échec de sauvegarde » | alerte rouge |

Les documents sont **propres à votre entreprise** (locataire) ; stockés côté serveur (`backend/routers/cao.py`, table `cao_documents`).

---

## 4. Référence

### 4.1 Outils de dessin (barre d'outils)

| Outil | Libellé | Rôle |
|-------|---------|------|
| Sélection | « Sélectionner » | Sélectionner / déplacer un élément (outil par défaut). |
| Mur | « Mur » | Tracer un mur (2 clics + saisie clavier de longueur). |
| Rectangle | « Rectangle » | Tracer 4 murs fermés (2 clics). |
| Dalle | « Dalle » | Tracer un plancher polygonal (N clics + double-clic). |
| Toit | « Toit ▾ » | Menu : Toit auto / Toit (forme). |
| Porte | « Porte » | Percer une porte en cliquant sur un mur. |
| Fenêtre | « Fenêtre » | Percer une fenêtre en cliquant sur un mur. |
| Escalier | « Escalier » | Poser un escalier droit. |
| Mobilier | « Mobilier » | Ouvrir le catalogue d'objets et poser un meuble. |
| Cote | « Cote » | Cotation manuelle (voir limites, section 5.1). |
| Ajuster | « Ajuster murs » | Étendre / rogner un mur jusqu'au mur le plus proche. |
| Esquisse | « Esquisse ▾ » | Menu d'esquisse 2D (ligne, cercle, arc, rectangle, point + modifier). |

### 4.2 Menus déroulants

**Toit** : « Toit auto (sur les murs) » (squelette multi-pente sur l'emprise des murs) ; « Toit (forme) » (toit-forme par rectangle 2 coins, 7 formes).

**Esquisse** (vue 2D uniquement) :
- *Dessiner* : Ligne, Cercle, Arc, Rectangle, Point.
- *Modifier* (après sélection d'une entité d'esquisse) : Déplacer, Copier, Pivoter, Symétrie.

**Affichage** :
- *Bascules* : « Cotes des murs » (défaut ON), « Surfaces des dalles » (ON), « Pièces (surfaces) » (ON), « Grille au sol » (ON), « Arêtes (3D) » (OFF), « Rendu réaliste » (ON, 3D).
- *Unités* : « m » (métrique) / « pi-po » (impérial). **Défaut : pi-po.**

**Exporter** (contextuel selon la vue) :
- *Modèle 3D* (vue 3D) : GLB, STL, OBJ.
- *Mise en plan* (vue Plan) : PDF.
- *Image* (vue 3D) : Rendu IA.

**Document** : Nouveau, Enregistrer / Enregistrer sous, Renommer, Supprimer, et la liste « Ouvrir ».

### 4.3 Préréglages de mur (valeurs Québec)

**Épaisseur** :

| Préréglage | Libellé | Valeur |
|------------|---------|--------|
| Extérieur (défaut) | « Ext. 9 1/4" » | 0,235 m (9 1/4 po) |
| Intérieur | « Cloison 4 1/2" » | 0,114 m (4 1/2 po) |

**Hauteur** (liste déroulante ; défaut 8'2") :

| Libellé | Valeur |
|---------|--------|
| 8'2" | 2,489 m |
| 9'2" | 2,794 m |
| 10'2" | 3,099 m |
| 11'2" | 3,404 m |
| 12'2" | 3,708 m |

### 4.4 Préréglages d'ouvertures

**Portes** : Porte intérieure ; Porte extérieure ; Porte extérieure 8 pi ; Porte de garage simple ; Porte de garage double ; Porte de garage haute ; Porte-patio coulissante.

**Fenêtres** : Fenêtre de chambre ; Fenêtre de salle de bain ; Fenêtre de salle de bain haute ; Fenêtre de sous-sol ; Fenêtre panoramique ; Fenêtre plancher-plafond.

Chaque préréglage fixe une largeur et une hauteur de départ, ajustables ensuite dans Propriétés.

### 4.5 Types de toit (toit-forme)

| Type | Libellé | Description |
|------|---------|-------------|
| flat | Plat | Toit plat. |
| gable | 2 versants | Deux pentes, pignons aux extrémités. |
| hip | 4 versants (croupe) | Quatre pentes. |
| shed | Appentis | Une seule pente. |
| gambrel | Gambrel | Toit « à la Mansart » à 2 pentes par versant (grange). |
| mansard | Mansarde | Quatre versants à double pente. |
| dutch | Demi-croupe | Croupe surmontée d'un petit pignon. |

**Pignons (par rive)** : sur un toit-forme à empreinte **rectangulaire**, on peut désigner quelles rives sont des pignons (cases « Rive 1 », « Rive 2 »…). Sur une empreinte non rectangulaire (L, T, U), l'option n'est pas disponible (voir limites, 5.1).

### 4.6 Matériaux (préréglages)

Blanc, Bois, Brique, Béton, Plâtre, Tuile, Marbre, Métal, Verre — plus une **couleur personnalisée** (sélecteur de couleur).

### 4.7 Mobilier (catalogue)

Catégories : **Mobilier**, **Électroménager**, **Salle de bain**, **Cuisine**, **Extérieur**. Les objets sont des modèles 3D (GLB) chargés depuis un CDN. Recherche par nom disponible.

### 4.8 Unités et formats

- **Impérial (pi-po)** — **défaut** : pieds-pouces (et seizièmes). Saisie de longueur acceptée sous les formes `8-2-7`, `8-2`, `8`.
- **Métrique (m)** : mètres, mètres carrés.
- Bascule dans le menu **Affichage → Unités**.

### 4.9 Raccourcis clavier

| Touche | Effet |
|--------|-------|
| `Échap` | Revenir à l'outil **Sélectionner** ; annule un tracé en cours ; ferme un menu/une fenêtre ; sort du mode Marche. |
| `Suppr` / `Retour arrière` | Supprimer le(s) élément(s) sélectionné(s) (en cascade). Protégé pour Site / Bâtiment. |
| `Entrée` | Poser le mur à la longueur saisie ; fermer une dalle ; envoyer un message dans le chat IA (`Maj`+`Entrée` = nouvelle ligne). |
| Chiffres | Saisir la longueur exacte d'un mur (outil Mur). |
| Flèches | Verrouiller la direction d'un mur (tracé) ; se déplacer en mode Marche. |
| `WASD` / `ZQSD` | Se déplacer en mode Marche. |
| Molette | Zoom. |
| Bouton droit + glisser | Orbite (vue 3D) ; `Maj` = déplacement latéral. |

### 4.10 Rendu IA — paramètres et facturation

- **Moteur** : modèle d'image de Google Gemini, appelé côté serveur (clé serveur ; `backend/routers/cao.py`).
- **Paramètres** : qualité (Pro / Standard / Rapide), résolution (2K / 4K), description libre optionnelle.
- **Pré-requis** : solde de **crédits IA** strictement positif sur le compte du locataire.
- **Coût** : refacturé au locataire avec marge (facturation à l'usage, suivie comme les autres appels IA).

---

## 5. Cas particuliers, limites et FAQ

### 5.1 Limites connues (vérifiées dans le code)

- **Bannière « en développement »** : le module est livré et fonctionnel, mais évolue encore.
- **Outil « Cote » (cotation manuelle)** : la cotation est **automatique** sur le plan / la feuille ; l'outil de cote manuelle n'est pas encore complet.
- **Vue Feuille** : seul le **plan au sol** est rendu. Les cellules d'**élévations (N/E/S)** et la **vue isométrique** sont réservées (vides). **Marqueurs de coupe/détail** : non implémentés (nécessitent un outil de coupe). **Multi-feuilles** : à venir.
- **Pignons de toit-forme** : fiables sur empreinte **rectangulaire** seulement. Sur un plan en L / T / U, le module produit une **croupe** automatique (pas de pignon par rive).
- **Assistant IA** : dessine **murs, pièces, toits** uniquement ; aperçu **textuel** (pas de fantôme 3D) ; l'annulation se fait pas-à-pas.
- **Métré → devis** : le métré est **sommaire** (affichage seulement) ; l'intégration au devis viendra dans une prochaine étape.
- **Ouvertures sur pignon (toit-forme)** : visibles en **3D** et sur la **feuille / PDF**, mais pas encore dans la **vue 2D interactive**.

### 5.2 FAQ

**Q. Où est passé l'ancien module CAO (`/cao`) ?**
Il a été remplacé par le module DAO (`/cao2`). L'ancienne adresse redirige automatiquement vers le nouveau module.

**Q. Quelle est l'unité par défaut ?**
Le **pied-pouce** (impérial). On bascule en mètres via le menu Affichage → Unités.

**Q. Je clique pour percer une fenêtre et rien ne se passe.**
Cliquez **directement sur la surface d'un mur** (pas dans le vide). Sur un toit-forme, cliquez une **face verticale** (pignon) — la pente est refusée.

**Q. Mon dessin est-il enregistré automatiquement ?**
Oui, l'autosave enregistre ~2 s après votre dernière action (voir l'état à côté du menu Document). Vous pouvez aussi enregistrer manuellement.

**Q. Comment obtenir une image réaliste pour un client ?**
Vue 3D → Exporter → Rendu IA → choisir qualité/résolution → Générer → Télécharger. Un solde de crédits IA est requis.

**Q. Pourquoi mon toit devient-il plat quand je coche une rive en pignon sur un plan en L ?**
Les pignons par rive ne sont fiables que sur une empreinte **rectangulaire**. Sur un L/T/U, utilisez plutôt l'outil **Toit (forme)** par aile, ou laissez la **croupe** automatique.

**Q. Le nouvel étage ne reçoit pas mes murs.**
Vérifiez l'**étage actif** dans l'arbre de scène (badge « (actif) ») : les nouveaux éléments vont dans l'étage actif. Cliquez l'étage voulu pour l'activer.

**Q. Comment naviguer comme dans une visite ?**
Bouton **Marche** (vue 3D) : cliquez la vue pour verrouiller la souris, déplacez-vous avec les flèches ou WASD, `Échap` pour sortir.

**Q. Puis-je récupérer mon modèle dans un autre logiciel ?**
Oui : vue 3D → Exporter → GLB / STL / OBJ.

---

*Manuel ERP Constructo — Module DAO / Modélisation 3D — v2.0 vérifié — 2026-06-20*
