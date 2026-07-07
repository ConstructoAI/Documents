# Module 34 — Rendu 3D photoréaliste

> **Version** : 1.0 (rédaction initiale vérifiée d'après le code source, juillet 2026)
> **Route** : `/rendu-3d` — menu latéral « Rendu 3D » (section **CONCEPTION 3D** de la barre de navigation, aux côtés de « DAO » et « PDF3D »)
> **Code de référence (frontend)** : `frontend/src/pages/Rendu3DPage.tsx` (162 lignes ; assemble trois briques) ; composants `frontend/src/components/rendu3d/Rendu3DDropzone.tsx` (511 lignes), `Rendu3DViewer.tsx` (371 lignes), `Rendu3DControls.tsx` (288 lignes), `rendu3dCapture.ts` (145 lignes) ; `PlanCropper.tsx` (440 lignes — **présent dans le dossier mais NON utilisé** par cette page) ; client `frontend/src/api/rendu3d.ts` (68 lignes) ; libellés `frontend/src/i18n/locales/{fr,en}/rendu3d.json` (66 lignes chacun, parité FR/EN)
> **Code de référence (backend)** : **il n'existe AUCUN router dédié « rendu3d ».** Le module est branché sur le point d'accès **partagé** `POST /api/erp/v1/cao/render`, défini par la fonction `render_realistic` dans `backend/routers/cao.py` (621 lignes ; point d'accès à la ligne 539), qui s'appuie sur le module pur `backend/routers/gemini_image.py` (176 lignes ; clé + appel du service d'images).
> **Tables PostgreSQL partagées (`public`)** : `ai_prepaid_credits` (solde de crédits prépayés du compte), `ai_usage_tracking` (traçage de l'usage ; la colonne `feature` reçoit la valeur `render3d`). Aucune table propre au module, aucune persistance de l'image produite.
> **Cadrage** : ce module prend un fichier **de votre ordinateur** — un **modèle 3D** (GLB, GLTF, OBJ, STL, FBX), une **image** (PNG, JPG, WEBP) ou un **PDF** — et en produit une **image de rendu photoréaliste** générée par intelligence artificielle (service d'images Google Gemini, gamme « Nano Banana »). Le rendu est **facturé à votre compte** au coût réel majoré et se **télécharge** en image PNG. Ce n'est **pas** le module **DAO / Modélisation 3D** (dessin manuel de la maquette, module 31), ni **PDF3D** (reconstruction 3D d'un jeu de plans par un service externe, module 33), ni le module **Métré** (prise de quantités sur PDF, module 30).

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « entreprise » ou « compte » (aussi « tenant ») désigne votre organisation, dont les données sont isolées ; « rendu » désigne l'image photoréaliste produite par l'intelligence artificielle ; « solde de crédits » désigne le portefeuille prépayé partagé qui alimente aussi les autres fonctionnalités d'intelligence artificielle de l'ERP.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Processus pas à pas](#3-processus-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module **Rendu 3D** transforme un fichier ordinaire en une **image de présentation photoréaliste**, prête à montrer à un client. Concrètement, il vous permet de :

- **Importer** depuis votre ordinateur un **modèle 3D** (GLB, GLTF, OBJ, STL, FBX), une **image** (PNG, JPG, WEBP) ou un **PDF** ;
- **Cadrer la vue** : pour un modèle 3D, l'orienter à la souris (orbite, zoom, panoramique) jusqu'à l'angle voulu ; pour une image ou un PDF, choisir la page à utiliser ;
- **Régler le rendu** : type de scène (extérieur / intérieur / objet), description libre des matériaux et de l'ambiance, niveau de qualité, résolution ;
- **Générer** le rendu : l'intelligence artificielle remplace l'aspect « maquette » par des matériaux, un éclairage et une ambiance réalistes, tout en **préservant la géométrie** du modèle et l'angle de caméra ;
- **Télécharger** l'image produite (fichier PNG) ou la **régénérer** avec d'autres réglages.

Le rendu est produit **à la demande, en quelques instants** (traitement synchrone : vous attendez le résultat à l'écran), et **facturé à votre solde de crédits** au coût réel du service majoré. C'est un fonctionnement **à l'usage** (« payez-ce-que-vous-utilisez ») : il n'y a **pas** de forfait inclus.

### 1.2 Comment y accéder

- Barre de navigation latérale → section **CONCEPTION 3D** → **Rendu 3D** (icône image).
- Adresse : `/rendu-3d` (page protégée, authentification requise).
- La même section CONCEPTION 3D contient aussi **DAO** (`/cao2`) et **PDF3D** (`/hover`), qui sont des modules distincts.

Le module est présenté sur **un seul écran**, sans onglets, sans sous-modules et sans liste : tout se fait au même endroit, en trois colonnes.

### 1.3 Rôles et permissions

Le Rendu 3D est volontairement **ouvert à tous les utilisateurs connectés** de votre entreprise. Le seul verrou réel est **financier** (le solde de crédits), et non le rôle.

| Situation | Effet |
|-----------|-------|
| **Tout utilisateur connecté d'une entreprise** | Peut importer un fichier, cadrer, régler et **générer un rendu** (aucune restriction de rôle ou de droit d'administration). |
| **Super-administrateur de la plateforme** | Autorisé également, et **exempté de facturation** (voir §1.4). |
| **Mode consultation** (abonnement suspendu / lecture seule) | Le rendu est **bloqué** : la génération est une écriture non autorisée en lecture seule et l'ERP la refuse (message « Mode consultation… ») **avant** tout appel. |
| **Abonnement complètement bloqué** | Accès refusé (déconnexion / réauthentification requise). |

**Précisions importantes :**

- **Contexte d'entreprise obligatoire.** Le point d'accès exige que votre session soit rattachée à une entreprise. Un compte sans entreprise se voit refuser le rendu (erreur 400 « Contexte tenant manquant »).
- **Aucune distinction administrateur / employé.** Contrairement à d'autres modules, il n'y a **pas** de verrou réservant le rendu aux administrateurs. Si un employé peut ouvrir la page, il peut générer (et donc **dépenser des crédits**). Tenez-en compte dans la gestion de votre solde.
- **Instance sans facturation activée.** Sur une installation interne où la facturation est désactivée, le rendu fonctionne **gratuitement** et l'usage reste tout de même tracé. Sur l'ERP commercial standard, la facturation est active.

### 1.4 Facturation : à l'usage, sur votre solde de crédits

Chaque rendu **débite de l'argent réel** de votre **solde de crédits prépayés** — le **même solde** que les estimations par intelligence artificielle, l'Assistant IA et le rendu du DAO. Points essentiels :

- **Solde strictement positif requis.** Avant de lancer le rendu, l'ERP vérifie votre solde. S'il est **inférieur ou égal à zéro**, la génération est **refusée** (erreur 402 « Crédits IA insuffisants pour le rendu ») **avant** tout appel au service payant. Le rendu étant une action coûteuse et non essentielle, cette règle est **plus stricte** que pour le clavardage (qui, lui, tolère un léger dépassement).
- **Coût = coût réel majoré de 30 %.** Le montant facturé est le coût brut du service d'images (calculé au plus juste, selon le nombre réel de jetons d'image produits) **multiplié par 1,30**. Un rendu en 4K, qui consomme plus de jetons qu'un 2K, coûte donc un peu plus cher — vous payez ce que vous consommez réellement.
- **Coût affiché en dollars US.** Le montant du rendu est indiqué **en dollars US** (le service facture en USD), et non en dollars canadiens.
- **Débit après succès.** Le crédit n'est débité **qu'après** la production réussie de l'image. Si le service échoue, rien n'est facturé.
- **Aucune clé anti-double-facturation côté serveur.** Ce point d'accès **ne dispose pas** de mécanisme serveur d'idempotence : la protection contre le double clic est **entièrement côté interface** (le bouton se verrouille pendant la génération et la zone de dépôt est gelée). Concrètement, **si deux requêtes de rendu partaient réellement en parallèle, chacune serait facturée**. En usage normal, l'interface empêche ce cas ; évitez malgré tout de forcer des envois répétés.
- **Super-administrateur exempté.** Ses rendus ne sont pas débités.

> **Refacturez ce coût.** Comme pour les autres fonctionnalités d'intelligence artificielle, le rendu consomme votre solde prépayé. Prévoyez de le refacturer à votre client dans le cadre du mandat.

### 1.5 Ce que le module fait — et ne fait PAS

Le module **fait** : importer un modèle 3D, une image ou un PDF ; cadrer une vue 3D ou choisir une page de PDF ; régler le type, la description, la qualité et la résolution ; produire une image photoréaliste par intelligence artificielle ; afficher son coût ; **télécharger** l'image (PNG) ; **régénérer** avec d'autres réglages.

Le module **ne fait PAS** :

- **Aucun assistant conversationnel.** Il n'y a pas de clavardage ici : uniquement un formulaire de réglages. (À ne pas confondre avec l'Assistant IA du DAO, module 31, qui **dessine** la maquette à la voix.)
- **Aucune persistance, aucune galerie, aucun historique.** L'image produite **n'est pas sauvegardée** côté serveur ni listée. Si vous cliquez « Régénérer » ou changez de fichier, le rendu précédent est **perdu**. **Téléchargez-le** si vous voulez le garder. (Le traçage d'usage `ai_usage_tracking` sert à la comptabilité, ce n'est pas un album d'images.)
- **Aucun export PDF ni CSV, aucune impression.** La seule sortie est le **téléchargement de l'image PNG**.
- **Aucun découpage (recadrage rectangulaire) sur cette page.** On cadre soit **par la 3D** (orientation de la vue), soit en prenant **l'image entière** ou **une page de PDF**. Il n'y a pas d'outil « capturer une portion de plan » ici. (Cet outil de découpe existe ailleurs — voir §5.1.)
- **Aucune reconstruction 3D.** Le module **ne fabrique pas** de maquette 3D à partir d'un plan : il produit une **image**. Pour reconstruire un modèle 3D depuis un jeu de plans, c'est le module **PDF3D** (module 33).
- **Aucun résultat garanti à l'identique.** Le rendu est **non déterministe** : l'intelligence artificielle préserve la géométrie et la caméra, mais **réinterprète** les matériaux, l'éclairage et le contexte. Deux rendus des mêmes réglages peuvent différer. Le résultat est une image « marketing », à valider.

### 1.6 Les trois colonnes de la page

L'écran se compose d'un **en-tête** suivi de **trois colonnes** (empilées à la verticale sur mobile, côte à côte sur grand écran) :

| Colonne | Rôle | Contenu |
|---------|------|---------|
| **Gauche** (≈ 288 px) | **Dépôt du fichier** | Zone de glisser-déposer / bouton « Parcourir » ; sélecteur de page pour un PDF multipage. |
| **Centre** (hauteur ≈ 70 % de l'écran) | **Visionneuse / Aperçu** | Vue 3D orientable (modèle), ou aperçu de l'image, ou invite si aucune source. |
| **Droite** (≈ 384 px) | **Panneau de commandes** | Réglages du rendu (type, description, qualité, résolution) et bouton de génération ; puis le résultat (image, coût, télécharger, régénérer). |

Le **traitement des fichiers 3D et la capture de la vue se font entièrement dans votre navigateur.** Le serveur, lui, ne reçoit qu'une **image** (jamais le fichier 3D d'origine) : c'est cette image — capture de la vue 3D, image importée ou page de PDF — qui part au service de rendu.

---

## 2. Interface

### 2.1 En-tête

En haut de la page : une icône (cube), le titre **« Rendu 3D »** et le sous-titre **« Importez un modèle 3D ou une image et générez un rendu photoréaliste par IA. »**

### 2.2 Colonne gauche — Zone de dépôt

La zone de dépôt a deux visages selon la situation : la **vue de dépôt** (par défaut) et la **vue de sélection de page** (uniquement pour un PDF de plusieurs pages).

#### 2.2.1 Vue de dépôt (par défaut)

- Un **rectangle en pointillés**, cliquable, qui accepte aussi le **glisser-déposer** d'un fichier. Une icône de nuage de téléversement s'affiche (remplacée par un indicateur animé pendant la lecture du fichier).
- Textes affichés : **« Glissez un fichier 3D ou une image ici »**, puis les formats **« 3D : GLB, GLTF, OBJ, STL, FBX — Image : PNG, JPG, WEBP, PDF »**, un séparateur **« ou »**, et un bouton **« Parcourir »** (icône dossier).
- Le sélecteur de fichiers accepte les extensions : `.glb`, `.gltf`, `.obj`, `.stl`, `.fbx`, `.png`, `.jpg`, `.jpeg`, `.webp`, `.pdf`. La détection se fait **par l'extension** du nom de fichier.

**Ce qui se passe selon le type de fichier déposé :**

| Type | Traitement | Limite / erreur |
|------|-----------|-----------------|
| **Modèle 3D** (`.glb`, `.gltf`, `.obj`, `.stl`, `.fbx`) | Chargé dans la visionneuse 3D de la colonne centre. | **Maximum 50 Mo.** Au-delà : « Fichier trop volumineux (maximum 50 Mo). » |
| **Image** (`.png`, `.jpg`, `.jpeg`, `.webp`) | Redimensionnée (côté le plus long ramené à **2048 px** au besoin) et ré-encodée, puis affichée en aperçu. | — |
| **PDF à une seule page** | La page 1 est convertie en image directement (pas de sélecteur). | En cas de lecture impossible : « Impossible de lire ce PDF. » |
| **PDF de plusieurs pages** | Bascule vers la **vue de sélection de page** (ci-dessous). | Idem. |
| **Autre extension** | Refusé. | « Format de fichier non pris en charge. » |

Sous la zone, un **bandeau rouge** (icône d'alerte) affiche le cas échéant le message d'erreur.

> **Zone gelée pendant un rendu.** Tant qu'un rendu est en cours dans la colonne de droite, la zone de dépôt est **désactivée** : c'est une protection pour ne pas perdre — en changeant de fichier — l'image que vous êtes en train de payer.

#### 2.2.2 Vue de sélection de page (PDF multipage)

Quand un PDF contient plusieurs pages, la colonne gauche propose de **choisir la page à utiliser** :

- Titre : **« Choisissez la page à utiliser »** (icône document).
- **Aperçu** de la page courante (le voile « Rendu de la page… » s'affiche pendant sa préparation).
- **Compteur** : « Page {n} / {total} ».
- Boutons de navigation : **« Précédent »** (flèche gauche) et **« Suivant »** (flèche droite).
- Bouton principal : **« Utiliser cette page »** (coche) — valide la page choisie comme image à rendre.
- Bouton secondaire : **« Choisir un autre fichier »** (icône dossier) — revient à la vue de dépôt.

### 2.3 Colonne centre — Visionneuse / Aperçu

Le centre de l'écran montre l'un de **trois états** :

**A. Source 3D — visionneuse orientable.** Le modèle importé s'affiche dans une **vue 3D interactive** (moteur du navigateur, WebGL) :

- **Orientation à la souris** : orbite (rotation autour du modèle), zoom (molette) et panoramique.
- **Cadrage automatique** : à l'ouverture, la caméra se place en vue isométrique pour montrer tout le modèle ; une grille de sol et une ombre portée aident à situer le volume.
- **Chargement selon le format** : GLB/GLTF, OBJ, STL (les normales sont recalculées si absentes) et FBX sont pris en charge. Un OBJ sans fichier de matériaux (MTL) s'affiche avec un matériau par défaut.
- **Voile de chargement** : « Chargement du modèle… ».
- **Erreur sans plantage** : si le fichier ne peut être chargé, la visionneuse affiche « Impossible de charger ce modèle 3D. » (le reste de la page reste utilisable).
- **Prêt = condition pour générer.** Le bouton de génération n'est actif **que lorsque la visionneuse a fini de charger** (état « prêt »). Tant que le modèle charge, aucune capture n'est possible — on évite ainsi de facturer le rendu d'une scène vide.

**B. Source image — aperçu simple.** Une image ou une page de PDF s'affiche comme **image centrée** (pas de 3D). C'est cette image, telle quelle, qui servira au rendu.

**C. Aucune source — invite.** Tant que rien n'est importé, le centre affiche une icône d'image et le message : **« Orientez le modèle pour cadrer l'angle voulu, puis générez le rendu. »**

> **La capture au moment de « Générer ».** Pour un modèle 3D, l'image envoyée au service est une **capture de la frame courante** (l'angle exact que vous avez cadré), prise au clic sur « Générer le rendu ». Elle est bornée à 2048 px de côté pour rester sous les limites du serveur. Pour une image ou un PDF, c'est directement l'image d'aperçu qui part.

### 2.4 Colonne droite — Panneau de commandes

Le panneau de commandes est **réinitialisé à chaque nouveau fichier** importé : il efface le rendu précédent et ré-applique le type suggéré.

Il a **deux états** : les réglages (avant le rendu) et le résultat (après le rendu).

#### 2.4.1 État 1 — Réglages (avant le rendu)

**1. Type de rendu** (« Type de rendu ») — trois boutons :

| Bouton | Indice | Usage |
|--------|--------|-------|
| **Extérieur** | « Bâtiment vu de l'extérieur » | Façades, vue d'ensemble d'un bâtiment. |
| **Intérieur** | « Pièce ou aménagement intérieur » | Rendu d'une pièce (matériaux + éclairage intérieur). |
| **Objet** | « Produit ou objet isolé » | Rendu produit / catalogue, fond studio ou contextuel. |

*Suggestion automatique* : **Objet** si la source est un modèle 3D, **Extérieur** sinon. Vous pouvez toujours changer.

**2. Détails et description (optionnel)** — une **zone de texte** (jusqu'à **2000 caractères**, 3 lignes) pour préciser les matériaux et l'ambiance souhaités. Exemple proposé : « revêtement de brique rouge, plancher de bois, éclairage de fin de journée, mobilier contemporain, fond studio neutre… ». Plus la description est précise, plus le rendu colle à votre intention.

**3. Qualité** — trois boutons :

| Bouton | Indice |
|--------|--------|
| **Pro** *(par défaut)* | « Meilleure qualité » |
| **Standard** | « Équilibre » |
| **Rapide** | « Économique » |

*Effet secondaire* : choisir **Rapide** ramène automatiquement la résolution à **2K** (le mode Rapide ne fait pas de vrai 4K).

**4. Résolution** — deux boutons :

| Bouton | Indice | Disponibilité |
|--------|--------|---------------|
| **2K** *(par défaut)* | « Rapide, net » | Toujours. |
| **4K** | « Ultra-détaillé (Pro/Standard) » | **Désactivé** si la Qualité est réglée sur **Rapide**. |

**5. Bouton de génération** : **« Générer le rendu »** (icône étincelles). Pendant le traitement, il devient **« Génération du rendu… »** avec un indicateur animé. Il est **désactivé** si un rendu est déjà en cours, si aucune source n'est importée, ou si la visionneuse 3D n'est pas encore prête.

Un **bandeau d'erreur** (icône d'alerte) s'affiche au-dessus si la génération échoue (par exemple « Échec du rendu. Réessayez. » ou le message précis renvoyé par le serveur, comme un solde insuffisant).

#### 2.4.2 État 2 — Résultat (après le rendu)

- **L'image de rendu** s'affiche (texte de remplacement : « Rendu photoréaliste »).
- **Note de coût** (si le montant est supérieur à zéro) : **« Coût de ce rendu : {montant} $ US »**.
- Bouton **« Télécharger »** (icône téléchargement) : enregistre l'image sur votre poste sous le nom `rendu-3d-<horodatage>.png`.
- Bouton **« Régénérer »** (icône rafraîchir) : revient aux réglages (les réglages et la source sont conservés) pour produire un **nouveau** rendu. Attention : le rendu affiché est alors **remplacé** — téléchargez-le d'abord si vous voulez le garder.

> **Verrou anti-double-clic.** Le bouton « Générer » se verrouille dès le premier clic et ne se relâche qu'à la fin (succès ou échec). C'est la protection principale contre une double facturation, puisque le serveur, lui, n'en a pas.

---

## 3. Processus pas à pas

### 3.1 Rendu à partir d'un modèle 3D

**Préalable :** un fichier 3D (GLB, GLTF, OBJ, STL ou FBX) de **50 Mo au maximum** ; un solde de crédits **strictement positif**.

1. Ouvrez **CONCEPTION 3D → Rendu 3D**.
2. Dans la colonne gauche, **glissez-déposez** votre fichier 3D (ou cliquez **« Parcourir »**).
3. Le modèle apparaît dans la visionneuse centrale. **Attendez** que le voile « Chargement du modèle… » disparaisse (état « prêt »).
4. **Orientez la vue** à la souris (orbite, zoom, panoramique) pour cadrer l'angle exact que vous voulez rendre.
5. Dans la colonne droite : choisissez le **Type** (Extérieur / Intérieur / Objet), saisissez une **description** des matériaux et de l'ambiance (facultatif mais recommandé), réglez la **Qualité** et la **Résolution**.
6. Cliquez **« Générer le rendu »**. La capture de l'angle courant part au service ; patientez quelques instants.
7. Le rendu s'affiche avec son **coût en $ US**. Cliquez **« Télécharger »** pour l'enregistrer, ou **« Régénérer »** pour retenter avec d'autres réglages.

### 3.2 Rendu à partir d'une image

**Préalable :** une image PNG, JPG/JPEG ou WEBP ; un solde de crédits strictement positif.

1. Ouvrez **Rendu 3D** et **importez** votre image (glisser-déposer ou « Parcourir »).
2. L'image s'affiche en **aperçu** au centre (elle est automatiquement ramenée à 2048 px de côté au besoin).
3. Réglez **Type**, **description**, **Qualité** et **Résolution** dans la colonne droite.
4. Cliquez **« Générer le rendu »**, puis **« Télécharger »** ou **« Régénérer »**.

*Astuce :* pour rendre plus réaliste une capture d'écran de maquette, une esquisse ou un plan, décrivez précisément les matériaux voulus dans le champ **Détails** — l'intelligence artificielle s'en sert pour transformer l'image plate en scène réaliste.

### 3.3 Rendu à partir d'un PDF

**Préalable :** un fichier PDF ; un solde de crédits strictement positif.

1. Ouvrez **Rendu 3D** et **importez** le PDF.
2. **Si le PDF n'a qu'une page**, elle est utilisée directement (l'aperçu s'affiche au centre) : passez à l'étape 4.
3. **Si le PDF a plusieurs pages**, la colonne gauche affiche « Choisissez la page à utiliser » : naviguez avec **« Précédent »** / **« Suivant »**, puis cliquez **« Utiliser cette page »** (ou **« Choisir un autre fichier »** pour recommencer).
4. Réglez **Type**, **description**, **Qualité** et **Résolution**, puis cliquez **« Générer le rendu »**.
5. **Téléchargez** ou **régénérez** le résultat.

### 3.4 Télécharger, régénérer, recommencer

- **Télécharger** enregistre l'image sous `rendu-3d-<horodatage>.png`. Comme rien n'est conservé côté serveur, **c'est la seule façon de garder votre rendu.**
- **Régénérer** relance un rendu avec la même source ; le rendu affiché est **écrasé** par le nouveau. Téléchargez d'abord si nécessaire.
- **Changer de fichier** (nouvelle importation) réinitialise le panneau et **efface** le rendu courant.

### 3.5 Si le rendu est refusé pour crédits insuffisants

Si un message indique que vos crédits sont insuffisants (le rendu exige un solde **strictement positif**) :

1. Rendez-vous dans **Configuration / Abonnement** (module 28) pour **recharger votre solde de crédits prépayés**.
2. Revenez sur **Rendu 3D** et relancez la génération.

Aucun débit n'a lieu tant que le solde n'est pas suffisant : le refus survient **avant** tout appel payant, donc **sans surprise**.

---

## 4. Référence

### 4.1 Point d'accès de l'API

Le module Rendu 3D **n'a pas de router backend à son nom** : il appelle le point d'accès **partagé** du DAO. Ne cherchez pas de fichier `routers/rendu3d.py` — il n'existe pas.

| Méthode | Chemin | Garde | Rôle |
|---------|--------|-------|------|
| POST | `/api/erp/v1/cao/render` | Authentifié + entreprise requise | Génère le rendu (Gemini image) et facture le coût. Fonction `render_realistic` — `cao.py:539` |

- Le module envoie le champ **`source: "render3d"`**. **Ce champ ne change qu'une seule chose** : l'étiquette de facturation écrite dans le traçage d'usage (`feature` = `render3d` au lieu de `cao_render` pour le DAO). **Le montant, le modèle, le portefeuille débité et le code exécuté sont identiques** dans les deux cas : ce n'est pas un aiguillage de facturation, seulement une étiquette de suivi.
- Le client d'interface (`api/rendu3d.ts`) applique un **délai d'attente de 180 s** ; le service d'images côté serveur a lui-même un délai de **120 s**.

### 4.2 Codes de réponse et messages

| Code | Déclencheur |
|------|-------------|
| **200** | Rendu produit. Réponse : image (donnée intégrée), modèle utilisé, coût facturé en USD. |
| **400** | Contexte d'entreprise manquant ; ou image invalide (base64 illisible). |
| **402** | Crédits insuffisants — soit l'autorisation générale échoue, soit le solde n'est **pas strictement positif** (« Crédits IA insuffisants pour le rendu »). |
| **403** | Fonctionnalité d'intelligence artificielle refusée ; **ou mode consultation** (lecture seule) : le rendu est une écriture bloquée. |
| **413** | Image trop volumineuse (base64 trop long, image décodée > 12 Mo, ou corps de requête > 20 Mo intercepté avant le traitement). |
| **429** | Trop de rendus depuis votre adresse IP (voir §4.4). |
| **502** | Le service d'images a renvoyé une erreur, une réponse illisible, ou **aucune image** (par exemple un blocage de sécurité du service). |
| **503** | Rendu non configuré : **clé du service absente** (« Rendu IA non configuré (clé manquante) »), ou facturation indisponible. |
| **504** | Délai dépassé côté service d'images (120 s). |

Les messages d'erreur restent **génériques** : aucun détail technique interne n'est exposé.

### 4.3 Calcul du coût

Le coût suit toujours la même formule : **coût brut du service × 1,30** (marge de 30 %), facturé en **dollars US**.

| Qualité | Modèle du service (gamme « Nano Banana ») | Coût brut par jeton d'image (USD) | Forfait de repli par image (USD) |
|---------|-------------------------------------------|-----------------------------------|----------------------------------|
| **Pro** *(défaut)* | `gemini-3-pro-image` | 0,00012 | 0,14 |
| **Standard** | `gemini-3.1-flash-image` | 0,00006 | 0,06 |
| **Rapide** | `gemini-2.5-flash-image` | 0,00003 | 0,04 |

- Le coût brut est calculé au plus juste : **nombre réel de jetons d'image produits × prix par jeton** du niveau choisi. Un rendu 4K consomme plus de jetons qu'un 2K, donc coûte un peu plus.
- Si le service ne renvoie pas le décompte de jetons, l'ERP applique le **forfait de repli** par image du niveau choisi.
- Le montant final = `coût brut × 1,30`, arrondi, puis **débité de votre solde**. Le super-administrateur est exempté.

*Exemple indicatif :* un rendu **Pro en 2K** (~1120 jetons) coûte environ `1120 × 0,00012 × 1,30 ≈ 0,17 $ US` ; le même en **4K** (~2000 jetons) environ `2000 × 0,00012 × 1,30 ≈ 0,31 $ US`. Les valeurs exactes varient selon la scène.

### 4.4 Limites, bornes et quotas

| Élément | Valeur |
|---------|--------|
| Formats 3D | **GLB, GLTF, OBJ, STL, FBX** (OBJ sans MTL → matériau par défaut) |
| Formats image | **PNG, JPG, JPEG, WEBP** |
| Format document | **PDF** (une page choisie) |
| Taille maximale d'un modèle 3D | **50 Mo** |
| Redimensionnement image / capture | Côté le plus long ramené à **2048 px** |
| Taille maximale de l'image décodée (serveur) | **12 Mo** (au-delà : 413) |
| Longueur maximale du champ image (base64) | **22 000 000 caractères** |
| Cap du corps de requête (avant traitement) | **20 Mo** (au-delà : 413) |
| Description libre | **2000 caractères** |
| Niveaux de qualité | **Pro / Standard / Rapide** |
| Résolutions | **2K / 4K** (le 4K réel exige Pro ou Standard ; le mode Rapide plafonne autour de 1K) |
| Limite de fréquence par adresse IP | **10 rendus par minute** (au-delà : 429) |
| Concurrence côté serveur | **4 rendus simultanés** au maximum (file d'attente au-delà) |
| Délai d'attente (interface / service) | **180 s / 120 s** |
| Solde requis | **Strictement positif** (> 0) |
| Idempotence côté serveur | **Aucune** (protection anti-double-clic dans l'interface seulement) |

### 4.5 Où l'on retrouve les traces (modèle de données)

- **`public.ai_prepaid_credits`** — votre **solde de crédits prépayés** (le même que l'Assistant IA, les estimations et le rendu du DAO). Le rendu y prélève le coût majoré. Ce solde peut être rechargé (module Configuration / Abonnement).
- **`public.ai_usage_tracking`** — le **journal d'usage** : chaque rendu y inscrit une ligne (utilisateur, modèle, jetons d'image, coût, durée, succès). **La colonne `feature` vaut `render3d`** pour ce module (contre `cao_render` pour le rendu lancé depuis le DAO). C'est un journal comptable, **pas** une galerie d'images.
- **Aucune autre table.** L'image produite n'est stockée nulle part côté serveur.

### 4.6 Deux surfaces de rendu à ne pas confondre

Le même module d'images sert **deux surfaces distinctes**, avec des **points d'accès, des portefeuilles et des marges différents**. Elles ne partagent que le moteur d'images (`gemini_image.py`) et une interface quasi identique — **pas** la facturation.

| Aspect | **Ce module — ERP `/rendu-3d`** | **Public — Estimation Express `/estimation/rendu`** |
|--------|--------------------------------|-----------------------------------------------------|
| Point d'accès | `POST /api/erp/v1/cao/render` | `POST /api/estimation/v1/render/{token}` |
| Authentification | Session ERP (entreprise requise) | Aucune (jeton de session prépayée) |
| Portefeuille débité | **Crédits du compte** (`ai_prepaid_credits`) | **Portefeuille prépayé public** d'Estimation Express (partagé avec le clavardage Express, **pas** avec l'ERP) |
| Marge | **× 1,30**, en USD | **× 3,0**, converti en CAD |
| Anti-double-débit | Interface seulement | Réservation atomique côté serveur (verrou anti-concurrence) |

Autrement dit : **il n'y a pas de portefeuille commun entre l'ERP et Estimation Express.** Le rendu ERP puise dans les crédits de votre entreprise ; le rendu public Express puise dans un portefeuille prépayé distinct.

### 4.7 Dépendance de configuration (administrateur d'infrastructure)

Le module est **inerte tant que la clé du service d'images n'est pas posée** sur le serveur (variable d'environnement `GOOGLE_GEMINI_API_KEY`). Sans elle, toute génération renvoie **503 « Rendu IA non configuré (clé manquante) »**. La pose de cette clé est une tâche d'**administrateur d'infrastructure**, pas une action utilisateur.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

- **DAO / Modélisation 3D (module 31).** Le DAO possède **son propre bouton « Rendu IA »** qui appelle exactement le même point d'accès (`/cao/render`), mais depuis la maquette dessinée dans l'ERP. Le module **Rendu 3D**, lui, part d'un **fichier importé**. Les deux consomment le même solde de crédits ; seule l'étiquette de traçage diffère (`cao_render` pour le DAO, `render3d` ici).
- **PDF3D (module 33).** À ne pas confondre : PDF3D **reconstruit une maquette 3D et des mesures** à partir d'un jeu de plans (service externe, ~24 h, facturé par structure). Le Rendu 3D **ne reconstruit rien** : il produit une **image**. Le fichier CAD que PDF3D vous laisse télécharger peut, à l'inverse, être ouvert dans un logiciel 3D puis réexporté en GLB/OBJ pour être **rendu** ici.
- **Soumissions / Devis (module 08).** Une variante de cet outil de rendu est **intégrée à un devis** (« Rendu depuis un devis ») : cette variante-là propose en plus un **découpage rectangulaire** d'une portion de plan (l'outil de découpe qui n'existe pas sur la page `/rendu-3d`). Utilisez le devis si vous voulez isoler une zone d'un plan.
- **Assistant IA et estimations (module 25).** Le Rendu 3D **partage le solde de crédits prépayés** avec l'Assistant IA et les estimations. Un rendu diminue ce solde commun.
- **Configuration / Abonnement (module 28).** C'est là que se **recharge le solde** et que se gère l'abonnement. En **mode consultation** (abonnement suspendu), le rendu est **bloqué**.

### 5.2 Foire aux questions

**Ai-je besoin d'être administrateur pour générer un rendu ?**
Non. **Tout utilisateur connecté** de l'entreprise peut générer un rendu — la seule condition est un **solde de crédits strictement positif**. Gardez à l'esprit qu'un employé qui génère **dépense les crédits de l'entreprise**.

**Combien coûte un rendu ?**
Le **coût réel du service majoré de 30 %**, en **dollars US**, débité de votre solde. Il est **faible et variable** (quelques cents à quelques dizaines de cents selon la qualité et la résolution), et affiché après chaque rendu (« Coût de ce rendu : … $ US »). Voir §4.3.

**Pourquoi le coût est-il en dollars US et pas en dollars canadiens ?**
Parce que le service d'images facture en dollars US. Le débit se fait sur votre solde de crédits prépayés dans cette devise.

**Le rendu reproduit-il exactement mon modèle ?**
Il **préserve la géométrie et l'angle de caméra**, mais **réinterprète** les matériaux, l'éclairage et le contexte. Le résultat est **non déterministe** : deux rendus des mêmes réglages peuvent différer. Considérez-le comme une image « marketing », à valider — pas comme un relevé technique.

**Où sont sauvegardés mes rendus ?**
Nulle part côté serveur. **Il n'y a ni galerie ni historique.** **Téléchargez** l'image (bouton « Télécharger ») si vous voulez la conserver : un « Régénérer » ou un changement de fichier l'efface.

**Peut-on faire une maquette 3D à partir d'un plan avec ce module ?**
Non. Ce module produit une **image**, pas un modèle 3D. Pour reconstruire une maquette 3D depuis un jeu de plans, utilisez **PDF3D** (module 33).

**Peut-on recadrer une portion d'un plan avant de le rendre ?**
Pas sur la page `/rendu-3d` : on prend l'image entière ou une page complète de PDF. L'outil de **découpe rectangulaire** existe uniquement dans la variante **« Rendu depuis un devis »** (module 08).

**Le 4K est grisé, pourquoi ?**
Parce que la **Qualité est réglée sur Rapide**, qui ne fait pas de vrai 4K. Passez en **Pro** ou **Standard** pour activer le 4K.

**Vais-je être facturé deux fois si je clique plusieurs fois sur « Générer » ?**
En usage normal, non : le bouton **se verrouille** pendant la génération et la zone de dépôt est gelée. Attention toutefois : **le serveur n'a pas de garde-fou d'idempotence** — si deux requêtes partaient réellement en parallèle, chacune serait facturée. Ne forcez pas d'envois répétés.

**Rien ne se passe / le bouton « Générer » reste inactif.**
Vérifiez : (1) qu'un fichier est bien importé ; (2) pour un modèle 3D, que la visionneuse a **fini de charger** (état « prêt ») ; (3) qu'un rendu n'est pas déjà en cours.

**Le service met du temps à répondre.**
Le rendu est généralement rapide, mais peut prendre jusqu'à ~2 minutes selon la charge. Au-delà du délai, une erreur s'affiche : réessayez.

**Puis-je générer un rendu en mode consultation (abonnement suspendu) ?**
Non. En lecture seule, la génération est **bloquée** (le rendu est une action d'écriture). Réactivez l'abonnement pour l'utiliser.

### 5.3 Dépannage courant

| Symptôme | Piste |
|----------|-------|
| « Rendu IA non configuré (clé manquante) » | La clé du service n'est pas posée sur le serveur. Tâche d'**administrateur d'infrastructure**. |
| « Crédits IA insuffisants pour le rendu » | Solde ≤ 0. **Rechargez** le solde de crédits (module Configuration / Abonnement), puis relancez. |
| « Mode consultation… » au clic sur Générer | Abonnement suspendu (lecture seule). Réactivez l'abonnement. |
| « Format de fichier non pris en charge. » | Extension hors liste. Utilisez GLB, GLTF, OBJ, STL, FBX, PNG, JPG, JPEG, WEBP ou PDF. |
| « Fichier trop volumineux (maximum 50 Mo). » | Le modèle 3D dépasse 50 Mo. Allégez-le (réduction du maillage, retrait de textures inutiles). |
| « Impossible de lire ce PDF. » | Le PDF est illisible ou protégé. Exportez-le à nouveau ou convertissez la page en image. |
| « Impossible de charger ce modèle 3D. » | Format 3D non standard ou fichier corrompu. Réexportez depuis votre logiciel 3D (GLB de préférence). |
| Le bouton « Générer » reste grisé | Aucune source, visionneuse 3D pas encore prête, ou rendu déjà en cours. |
| « Échec du rendu. Réessayez. » | Erreur transitoire du service (502/504) ou réseau. Relancez ; si cela persiste, réduisez la qualité/résolution. |
| Erreur 413 (« Image trop volumineuse ») | La capture / l'image dépasse les bornes serveur. Réduisez la résolution de la source ou l'ampleur de la vue. |
| Trop de rendus refusés (429) | Vous avez dépassé **10 rendus par minute**. Patientez une minute. |

---

## 6. Récapitulatif

- **Le module Rendu 3D transforme un fichier importé (modèle 3D, image ou PDF) en une image photoréaliste** produite par intelligence artificielle (service d'images Google Gemini, gamme « Nano Banana »). Accès : menu **CONCEPTION 3D → Rendu 3D** (`/rendu-3d`).
- **Aucun router backend dédié** : le module s'appuie sur le point d'accès **partagé** `POST /api/erp/v1/cao/render` (le même que le « Rendu IA » du DAO). Le champ `source: "render3d"` ne change **que l'étiquette de traçage**.
- **Le traitement 3D et la capture de vue sont 100 % dans le navigateur** ; le serveur ne reçoit qu'une **image** (jamais le fichier 3D), bornée à 2048 px et 12 Mo.
- **Trois colonnes** : dépôt du fichier (glisser-déposer, sélecteur de page PDF), visionneuse / aperçu (vue 3D orientable ou image), panneau de commandes (Type, Détails, Qualité, Résolution → Générer → résultat).
- **Réglages** : Type (Extérieur / Intérieur / Objet), description libre (2000 caractères), Qualité (Pro / Standard / Rapide), Résolution (2K / 4K ; le 4K exige Pro ou Standard).
- **Facturation à l'usage** sur le **solde de crédits prépayés** du compte : **coût réel × 1,30**, en **dollars US**, débité après succès. **Solde strictement positif requis** (sinon 402, avant tout appel). **Aucune idempotence serveur** : la protection anti-double-clic est **dans l'interface**.
- **Ouvert à tout utilisateur connecté** (pas de restriction de rôle) ; **bloqué en mode consultation** ; super-administrateur **exempté** de facturation.
- **Aucune persistance** : pas de galerie, pas d'historique, pas d'export PDF/CSV. La seule sortie est le **téléchargement PNG** (`rendu-3d-<horodatage>.png`). « Régénérer » ou changer de fichier **efface** le rendu courant.
- **Résultat non déterministe** : la géométrie et la caméra sont préservées, mais les matériaux et l'ambiance sont réinterprétés — image « marketing » à valider.
- **Ne pas confondre** : DAO (dessiner la maquette, module 31), PDF3D (reconstruire une maquette 3D depuis des plans, module 33), Rendu depuis un devis (avec découpe, module 08). Le Rendu 3D produit **une image**, à partir d'un fichier déjà existant.
- **Dépendance** : inerte sans la clé du service (`GOOGLE_GEMINI_API_KEY`) → 503.

---

*Fichiers sources vérifiés :* `frontend/src/pages/Rendu3DPage.tsx` (162 lignes) ; `frontend/src/components/rendu3d/Rendu3DDropzone.tsx` (511 lignes), `Rendu3DViewer.tsx` (371 lignes), `Rendu3DControls.tsx` (288 lignes), `rendu3dCapture.ts` (145 lignes), `PlanCropper.tsx` (440 lignes, non importé par la page) ; `frontend/src/api/rendu3d.ts` (68 lignes) ; `frontend/src/i18n/locales/fr/rendu3d.json` et `frontend/src/i18n/locales/en/rendu3d.json` (66 lignes chacun) ; backend partagé `backend/routers/cao.py` (621 lignes ; point d'accès `render_realistic` à la ligne 539), `backend/routers/gemini_image.py` (176 lignes) ; garde-fous anti-abus dans `backend/erp_api.py` (cap du corps 20 Mo, limite de 10 rendus/minute par IP) ; tables partagées `public.ai_prepaid_credits` et `public.ai_usage_tracking` (colonne `feature` = `render3d`).

*Manuels liés :* `31-outils-dao-modelisation.md` (DAO, même section CONCEPTION 3D, bouton « Rendu IA » sur le même point d'accès), `33-conception3d-pdf3d-hover.md` (PDF3D, reconstruction 3D depuis des plans), `08-ventes-soumissions.md` (variante « Rendu depuis un devis », avec découpe), `25-communication-assistant-ia.md` (moteur d'intelligence artificielle et crédits prépayés), `28-configuration.md` (recharge du solde, abonnement et mode consultation).
