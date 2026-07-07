# Module 33 — PDF3D (plan vers 3D)

> **Version** : 1.0 (rédaction initiale vérifiée d'après le code source, juillet 2026)
> **Route** : `/hover` — menu latéral « PDF3D » (section **CONCEPTION 3D** de la barre de navigation, aux côtés de « DAO » et « Rendu 3D »)
> **Code de référence** : `backend/routers/hover.py` (1932 lignes ; 13 points d'accès dont 1 webhook public), monté défensivement dans `erp_api.py` sous le préfixe réel `/api/erp/v1/hover/...` ; `frontend/src/pages/HoverPage.tsx` (541 lignes), `frontend/src/api/hover.ts` (182 lignes ; 12 fonctions), `frontend/src/components/hover/HoverEstimateModal.tsx` (293 lignes) + `frontend/src/components/hover/hoverEstimate.ts` (composition mesures → devis), i18n `frontend/src/i18n/locales/{fr,en}/hover.json` (73 clés, parité FR/EN)
> **Tables PostgreSQL partagées (`public`)** : `hover_oauth` (compte central, singleton `id=1`), `hover_jobs` (envois de plan, cloisonnés par entreprise via la colonne `tenant_schema`), `hover_webhooks`. La facturation s'appuie sur les tables partagées `public.ai_prepaid_credits` (solde de crédits) et `public.ai_usage_tracking` (traçage d'usage).
> **Cadrage** : ce module envoie un **jeu de plans d'architecte** (PDF, DWG ou DXF, avec appoint PNG / JPG) à un service externe de reconstruction 3D qui, sous environ 24 heures, retourne un **modèle 3D et des mesures extérieures** (toit, murs, ouvertures). Pour un modèle terminé, on consulte les **livrables** (rapports PDF, image de présentation, visionneuse 3D externe, fichier CAD XML) et on génère une **estimation budgétaire de construction complète au pied carré**, convertie en **devis** (module Ventes). Le service technique est **Hover** (`hover.to`), mais tout le texte visible le nomme **« PDF3D »** (marque blanche). Ce n'est **pas** le module **DAO / Modélisation 3D** (dessin manuel, module 31) ni le module **Métré** (prise de quantités sur PDF, module 30).

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « entreprise » ou « tenant » désigne votre compte (chaque entreprise a ses propres données isolées) ; « job » ou « envoi » désigne une demande de reconstruction 3D soumise au service ; « marque blanche » signifie que le nom du fournisseur (Hover) est masqué et remplacé partout dans l'interface par « PDF3D ».

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

PDF3D transforme un **jeu de plans papier ou numériques** en un **modèle tridimensionnel exploitable**. Concrètement, le module vous permet de :

- **Envoyer un jeu de plans** (élévations des quatre côtés + plans d'étage cotés) au service de reconstruction, en un seul envoi regroupant jusqu'à 24 fichiers ;
- **Suivre l'avancement** de chaque envoi (envoyé, en attente, terminé, échoué) dans un tableau « Mes envois » ;
- **Consulter les livrables** d'un modèle terminé : un **rapport de toit** (PDF), un **rapport Pro Premium** (PDF), une **image de présentation**, un **lien vers une visionneuse 3D** externe et le **fichier CAD** (XML) du modèle ;
- **Générer une estimation budgétaire** de construction **complète au pied carré** (tous les corps de métier) à partir des mesures du modèle, puis la **convertir en devis** du module Ventes, à réviser avant envoi au client.

Le service reconstruit le bâtiment sous **environ 24 heures** (traitement asynchrone) ; vous êtes notifié à la complétion. Une reconstruction est **facturée par structure**.

### 1.2 Comment y accéder

- Barre de navigation latérale → section **CONCEPTION 3D** → **PDF3D** (icône immeuble).
- Adresse : `/hover` (page protégée, authentification requise).
- La même section CONCEPTION 3D contient aussi **DAO** (`/cao2`) et **Rendu 3D** (`/rendu-3d`), qui sont des modules distincts.

### 1.3 Rôles et permissions

Trois niveaux d'accès cohabitent dans ce module. C'est le point le plus important à comprendre avant de l'utiliser.

| Action | Qui peut la faire |
|--------|-------------------|
| **Ouvrir la page, voir le statut, lister les envois, consulter les livrables, générer une estimation** | Tout utilisateur connecté de l'entreprise |
| **Connecter / déconnecter le compte PDF3D central** | **Super-administrateur de la plateforme uniquement** |
| **Envoyer un plan** (action facturée) | **Administrateur ou propriétaire de votre entreprise** — le super-administrateur en est **exclu** |

**Précisions importantes :**

- **Un seul compte PDF3D central existe pour toute la plateforme.** Il est connecté **une seule fois** par le super-administrateur (voir §1.4). Vous n'avez **pas** à créer ni à connecter de compte PDF3D vous-même : une fois le compte central branché, toutes les entreprises l'utilisent.
- **Le super-administrateur ne peut pas envoyer de plan.** Il sert uniquement à brancher le compte central. Techniquement, il n'a pas de schéma d'entreprise, donc l'envoi lui est refusé côté serveur ; l'interface lui masque donc le formulaire d'envoi (`HoverPage.tsx:52`, `hover.py:878`).
- **Seul un administrateur ou le propriétaire d'une entreprise voit le formulaire d'envoi** et peut engager le coût. Un utilisateur ordinaire (sans droit d'administration) voit le statut et la liste des envois, mais **pas** le formulaire d'envoi.
- En **mode consultation** (abonnement suspendu / lecture seule), **toutes** les actions d'écriture du module (connecter, déconnecter, envoyer un plan, rafraîchir un envoi, générer une estimation) sont **bloquées** par le contrôle global de l'ERP. La lecture des statuts et des livrables reste possible.

### 1.4 Le compte PDF3D central (modèle de connexion)

PDF3D fonctionne avec **un compte de service unique** partagé par toute la plateforme, et non un compte par entreprise :

- Le **super-administrateur** clique une fois sur **« Connecter PDF3D »**, s'authentifie auprès du service (protocole OAuth), et le jeton d'accès est **chiffré** puis stocké dans une seule ligne de base de données (`public.hover_oauth`, `id=1`).
- Ensuite, **chaque envoi de plan** de **chaque entreprise** passe par ce compte central. Le service ne distingue pas les entreprises ; c'est l'ERP qui **rattache chaque envoi à votre entreprise** (colonne `tenant_schema` de `public.hover_jobs`) et qui **vous facture** le coût.
- Si le compte central n'est **pas** connecté, la page affiche un statut « non connecté » et l'envoi est impossible. Si le serveur n'a **pas** été configuré du tout (variables d'environnement absentes), la page affiche « PDF3D n'est pas encore activé sur ce serveur ».

### 1.5 Facturation : deux coûts distincts

Le module engage de l'argent réel sur **votre solde de crédits prépayés** (le même solde que le module Rendu 3D et les estimations IA). Il y a **deux coûts séparés** :

| Coût | Quand | Montant | Idempotent |
|------|-------|---------|------------|
| **A — Envoi d'un plan** | À la confirmation de l'envoi | Fixe : **environ 390 $ US** (coût réel du service × marge) | **Oui** — un même envoi n'est jamais facturé deux fois |
| **B — Estimation** | À la lecture des mesures (dans la modale d'estimation) | Coût réel de l'IA (variable, faible) × marge | **Non** — chaque relecture des mesures est facturée |

- Le **coût A** est de **299 $ US × 1,30 = 388,70 $** (affiché « ≈ 390 $ US » dans l'interface). Il est **configurable** par l'administrateur du serveur ; réglé à 0, la facturation de l'envoi est **désactivée**.
- Le **super-administrateur est exempté** de la facturation A (mais il ne peut de toute façon pas envoyer de plan).
- Avant chaque envoi, l'ERP **vérifie votre solde** : s'il est insuffisant, l'envoi est **refusé** (erreur 402) **avant** que le service externe ne soit sollicité — donc aucune dépense surprise.
- Le **coût B** est celui de la lecture des rapports PDF par l'intelligence artificielle (modèle Claude Opus). Il est faible mais **se répète à chaque fois** que vous cliquez sur « Lire les mesures du modèle ».

> **Refacturez ce coût à votre client.** L'avertissement affiché dans le formulaire d'envoi le rappelle : « Assurez-vous d'avoir le solde nécessaire et de refacturer ce coût à votre client. »

### 1.6 Ce que le module fait — et ne fait PAS

Le module **fait** : envoyer un jeu de plans, suivre l'état des envois, afficher les livrables d'un modèle terminé (rapports PDF, image, lien de visionneuse 3D, fichier CAD XML), lire les mesures par IA, produire une estimation de construction complète au pied carré et créer un devis budgétaire dans le module Ventes.

Le module **ne fait PAS** :

- **Aucune visionneuse 3D intégrée.** « Voir en 3D » ouvre la **visionneuse externe** du service dans un nouvel onglet. Une visionneuse interne avait été construite puis **retirée** (rendu jugé insuffisant). N'attendez pas d'affichage 3D dans la page elle-même.
- **Aucune importation automatique dans le DAO.** Le fichier CAD (XML) se **télécharge** ; il n'est **pas** ré-injecté automatiquement dans le module DAO / cao2. Une importation manuelle ou future reste à faire (« à importer dans le DAO au besoin »).
- **Aucun affichage des mesures brutes (JSON).** Les mesures détaillées existent côté serveur mais ne sont **pas** affichées telles quelles. L'estimation s'appuie sur la **lecture des rapports PDF** par l'IA, pas sur ces données brutes.
- **Aucun résultat instantané.** Tant que le modèle n'est pas **terminé** (~24 h), aucun livrable n'est disponible ; les boutons de rapport renvoient une erreur « pas encore disponible ».
- **Aucune saisie du coût exact par envoi.** Le montant facturé pour l'envoi est **fixe** (configuré par le serveur), car le service ne renvoie pas le coût réel de chaque reconstruction.
- **Aucun appel du service par un utilisateur ordinaire.** Seul un administrateur d'entreprise engage la dépense.

### 1.7 Les trois zones de la page

La page se compose de **trois cartes empilées** et d'**une fenêtre modale** :

| # | Zone | Rôle | Visible pour |
|---|------|------|--------------|
| A | **Connexion** | État du compte central, boutons connecter / déconnecter | Toujours |
| B | **Envoi de plan** | Formulaire : nom, fichiers, avertissement coût, envoi en deux temps | Admin/propriétaire d'entreprise seulement, si le compte est connecté |
| C | **Mes envois** | Tableau des envois + livrables des modèles terminés | Dès que le compte est connecté |
| — | **Modale Estimation** | Mesures → estimation → devis (trois temps) | Ouverte depuis un envoi terminé |

---

## 2. Interface

### 2.1 En-tête et bandeaux

En haut de la page : une icône immeuble, le titre **« PDF3D — Plan vers 3D »** et le sous-titre « Transformez un jeu de plans d'architecte (élévations + plans d'étage cotés) en modèle 3D et mesures. »

Au chargement, un **indicateur d'attente** (« Chargement... ») s'affiche le temps de récupérer l'état de connexion.

Deux **bandeaux transitoires** apparaissent au besoin :

- **Erreur** (rouge, icône d'alerte) : message d'erreur détaillé renvoyé par le serveur, ou à défaut « Une erreur est survenue. »
- **Succès** (bleu, icône coche) : confirmation d'une action (par exemple « Compte PDF3D connecté avec succès. » ou « Plan envoyé. »).

### 2.2 Carte A — Connexion

Cette carte est **toujours visible**. Elle indique l'état du **compte central** et propose les actions de connexion selon votre rôle.

**Indicateur d'état :**

- Coche verte + **« Compte PDF3D connecté »** si le compte central est branché ;
- Icône ambre + **« Compte PDF3D non connecté »** sinon.
- Si le compte est connecté et qu'un nom est disponible, une ligne **« Compte »** affiche le nom du compte de service.

**Actions selon la situation :**

| Situation | Ce que vous voyez |
|-----------|-------------------|
| **Serveur non configuré** | Texte gris : « PDF3D n'est pas encore activé sur ce serveur (variables d'environnement manquantes). Contactez l'administrateur. » Aucun bouton. |
| **Super-administrateur, compte non connecté** | Bouton **« Connecter PDF3D »** (bleu, icône lien). Pendant l'appel, il affiche « Redirection... » puis vous redirige vers la page d'autorisation du service. |
| **Super-administrateur, compte connecté** | Bouton **« Déconnecter »** (icône lien brisé). La déconnexion efface les jetons mais **conserve** l'historique des envois. |
| **Utilisateur non super-administrateur, compte non connecté** | Texte gris : « Un super-administrateur doit connecter le compte PDF3D. » Aucune action possible de votre part. |

> **Retour d'autorisation.** Après avoir autorisé l'accès sur le site du service, vous êtes ramené sur la page `/hover` avec un code de confirmation ; l'ERP finalise automatiquement la connexion et affiche « Compte PDF3D connecté avec succès. » ou, en cas d'échec, « La connexion PDF3D a échoué. Réessayez. »

### 2.3 Carte B — Envoi de plan

Cette carte n'apparaît **que si** le compte central est connecté **et** que vous êtes **administrateur ou propriétaire** de votre entreprise (le super-administrateur ne la voit jamais).

**Titre et aide :** « Envoyer un plan » — « Joignez le jeu de plans (PDF, DWG ou DXF) contenant les élévations des 4 côtés et les plans d'étage cotés. »

**Champs :**

| Champ | Détail |
|-------|--------|
| **Nom du projet / structure** | Champ texte. Exemple proposé : « Ex. Résidence Tremblay - lot 42 ». Obligatoire pour activer le bouton d'envoi. |
| **Fichiers de plan** | Sélecteur de fichiers **multiples**. Formats acceptés : **PDF, DWG, DXF, PNG, JPG**. Aides affichées : « Plusieurs fichiers acceptés (max 24, 80 Mo cumulés). » et « Formats : PDF, DWG, DXF, PNG, JPG. » |

**Avertissement de coût (encadré ambre, icône triangle) :** « Coût : environ 390 $ US sont débités de votre solde de crédits à l'envoi. Assurez-vous d'avoir le solde nécessaire et de refacturer ce coût à votre client. »

**Envoi en deux temps (anti-erreur) :**

1. Le bouton **« Envoyer »** (icône téléversement) est **désactivé** tant que le nom est vide ou qu'aucun fichier n'est joint. Un clic bascule la carte en **mode confirmation**.
2. En confirmation, deux boutons apparaissent : **« Confirmer l'envoi (≈ 390 $ US) »** (ambre) et **« Annuler »**. Ce n'est qu'au clic de confirmation que l'envoi (et la facturation) a lieu. Pendant l'opération, le bouton affiche « Envoi... ».

**Note de bas de carte (icône horloge) :** « Le modèle 3D et les mesures sont générés par PDF3D sous environ 24 h (facturés par structure). Vous serez notifié à la complétion. »

> **Protection contre le double envoi.** L'interface génère une **clé d'envoi unique** au moment où vous préparez le formulaire et la réutilise en cas de nouvel essai (réseau instable, double-clic). Le serveur reconnaît cette clé et **ne facture jamais deux fois** le même envoi. La clé n'est réinitialisée qu'après un envoi réussi.

### 2.4 Carte C — Mes envois

Cette carte apparaît dès que le compte central est connecté. Elle liste **vos** envois (ceux de votre entreprise uniquement).

**Titre :** « Mes envois », avec un lien **« Actualiser »** (icône rafraîchir) qui recharge la liste.

**État vide :** « Aucun plan envoyé pour le moment. »

**Colonnes du tableau :**

| Colonne | Contenu |
|---------|---------|
| **Nom** | Nom du projet. Si l'envoi est **terminé**, une flèche (chevron) permet de **déplier** ses livrables ; sinon, texte simple. |
| **Statut** | Libellé : **Envoyé**, **En attente**, **Terminé**, **Échoué**, ou « — » si inconnu. |
| **Demandé par** | Nom de la personne qui a fait l'envoi, ou « — ». |
| **Date** | Date et heure de l'envoi, ou « — ». |
| *(action)* | Icône **Rafraîchir** par ligne : réinterroge le service pour cet envoi précis. |

**Rafraîchir un envoi** interroge le service afin de mettre à jour le statut. Un statut **ne peut jamais reculer** : un envoi « Terminé » ou « Échoué » ne redevient jamais « En attente », même si un message arrive en retard.

### 2.5 Sous-ligne dépliée d'un envoi terminé

En cliquant sur la flèche d'un envoi **terminé**, une sous-ligne s'ouvre et affiche les **livrables** du modèle.

- **Image de présentation** : chargée à la demande (via un canal sécurisé). Un espace réservé s'affiche le temps du chargement.
- **Aide** : « Livrables du modèle terminé : ».

**Boutons de livrables :**

| Bouton | Effet |
|--------|-------|
| **Rapport de toit (PDF)** | Ouvre le rapport de toit dans un nouvel onglet. |
| **Rapport Pro Premium (PDF)** | Ouvre le rapport complet (façades, ouvertures, matériaux) dans un nouvel onglet. |
| **Voir en 3D** | Ouvre la **visionneuse 3D externe** du service dans un nouvel onglet. *N'apparaît que si un lien de visionneuse est disponible.* |
| **Télécharger le CAD (XML)** | Télécharge le fichier CAD du modèle (nom de fichier assaini `{nom}.xml`). |
| **Obtenir l'estimation** | Ouvre la **modale Estimation** (voir §2.6). |

**Note explicative :** « "Voir en 3D" ouvre la visionneuse PDF3D (rendu réaliste). "Télécharger le CAD" fournit le fichier XML du modèle, à importer dans le DAO au besoin. »

En cas de problème sur un livrable précis, un message d'erreur rouge s'affiche sous les boutons.

### 2.6 Modale Estimation (mesures → devis, en trois temps)

Ouverte par le bouton **« Obtenir l'estimation »** d'un envoi terminé. Titre : **« Estimation — {nom du projet} »**, avec un bouton de fermeture (X). La modale suit une **machine à états** en plusieurs phases, avec des **verrous** qui empêchent tout double-clic.

**Phase 1 — Introduction (`idle`).**
Texte : « Génère une estimation budgétaire de CONSTRUCTION COMPLÈTE (au pi², tous les corps de métier) à partir des mesures du modèle. Vous confirmez la superficie de plancher, puis l'IA produit les lignes de devis. À réviser avant envoi. » Encadré ambre : « Coût IA facturé à votre solde de crédits (lecture des mesures + estimation). » Bouton : **« Lire les mesures du modèle »** (icône règle).

**Phase 2 — Lecture des mesures (`loadingMeasures`).**
Indicateur d'attente : « Lecture des mesures du modèle... ». En arrière-plan, l'IA lit les rapports PDF du modèle et pré-remplit une **description de métré** et une **superficie de plancher** estimée. **Cette lecture est facturée** (coût B).

**Phase 3 — Confirmation de la superficie (`measures`).**
- **Champ nombre** : « Superficie de plancher brute (pi²) » (icône règle, valeur pré-remplie, modifiable ; exemple « 1414 »).
- **Aide** : « Empreinte × nombre d'étages. Corrigez au besoin (ex. 1414 = 707 RDC + 707 étage). Pilote l'estimation au pi² d'une construction COMPLÈTE (structure, intérieur, plomberie/électricité/CVAC, finitions inclus). »
- **Bloc dépliable** « Voir les mesures extraites » : affiche la description brute produite par l'IA.
- **Bouton** : **« Générer l'estimation »** (icône étincelles).

**Phase 4 — Estimation (`estimating`).**
Indicateur : « Lecture des mesures et estimation en cours... ». L'IA d'estimation de l'ERP convertit la description + la superficie confirmée en **lignes de devis**. La cible est d'environ **265 $/pi²** pour une construction résidentielle complète et finie (fourchette APCHQ 250-500 $/pi²).

**Phase 5 — Aperçu (`preview`).**
Un **tableau des lignes** s'affiche :

| Colonne | Contenu |
|---------|---------|
| **Description** | Libellé de la ligne (+ catégorie en gris) |
| **Qté** | Quantité |
| **Unité** | Unité de mesure |
| **Prix unit.** | Prix unitaire |
| **Montant** | Montant de la ligne |

En pied : **« Total estimé »** (formaté en dollars canadiens) et la clause de non-responsabilité : « Estimation budgétaire préliminaire de construction COMPLÈTE (au pi², tous les corps de métier ; mesures extérieures PDF3D en appui). À réviser avant envoi au client. »

**Phase 6 — Création du devis (`creating` → `done`).**
Le bouton **« Créer le devis »** crée un devis **budgétaire** de type projet **Construction**, nommé « Estimation - {nom du projet} », avec les majorations (administration / contingences / profit) issues de l'IA, puis y injecte les lignes. À la fin (`done`), un bandeau vert « Devis créé. » affiche le numéro du devis, et le bouton devient **« Ouvrir le devis »** (qui vous emmène dans le module Ventes, devis ouvert). Le bouton **« Fermer »** reste toujours présent.

> **Création idempotente.** Si l'injection des lignes échoue et que vous réessayez, l'ERP **réutilise le même devis** au lieu d'en créer un doublon. Les lignes sans quantité (titres de section) sont automatiquement écartées, et chaque montant est recalculé et borné pour éviter tout dépassement.

---

## 3. Processus pas à pas

### 3.1 Activer PDF3D (super-administrateur, une seule fois)

**Préalable :** le serveur doit être configuré (variables d'environnement du service posées par l'administrateur d'infrastructure). Sinon la carte Connexion affiche « PDF3D n'est pas encore activé sur ce serveur ».

1. Connectez-vous en tant que **super-administrateur de la plateforme**.
2. Ouvrez **CONCEPTION 3D → PDF3D**.
3. Dans la carte **Connexion**, cliquez **« Connecter PDF3D »**.
4. Vous êtes redirigé vers la page d'autorisation du service ; autorisez l'accès.
5. Vous revenez automatiquement sur `/hover` ; le message « Compte PDF3D connecté avec succès. » confirme le branchement.

À partir de là, **toutes les entreprises** peuvent envoyer des plans. Vous n'avez plus à intervenir, sauf pour déconnecter le compte.

### 3.2 Envoyer un jeu de plans (administrateur d'entreprise)

**Préalable :** compte central connecté ; solde de crédits suffisant (~390 $ US) ; vous êtes administrateur ou propriétaire de votre entreprise.

1. Ouvrez **CONCEPTION 3D → PDF3D**.
2. Préparez votre **jeu de plans** : idéalement les **élévations des quatre côtés** et les **plans d'étage cotés**, en PDF, DWG ou DXF (l'appoint PNG / JPG est accepté). Maximum **24 fichiers**, **80 Mo cumulés**.
3. Dans la carte **Envoi de plan**, saisissez un **Nom du projet / structure** clair (il identifiera l'envoi dans « Mes envois »).
4. Cliquez **Fichiers de plan** et sélectionnez vos fichiers.
5. Lisez l'**avertissement de coût** (~390 $ US débités à l'envoi).
6. Cliquez **« Envoyer »**, puis **« Confirmer l'envoi (≈ 390 $ US) »**.
7. Le message « Plan envoyé. » confirme la soumission ; l'envoi apparaît dans **Mes envois** au statut **Envoyé**.

**Que se passe-t-il ensuite ?** Le service reconstruit le bâtiment sous **environ 24 heures**. Vous n'avez rien d'autre à faire ; revenez plus tard et cliquez **Actualiser** (ou l'icône rafraîchir de la ligne) pour voir passer le statut à **Terminé**.

> **Si le solde est insuffisant**, l'envoi est **refusé avant** toute dépense (message d'erreur indiquant le montant requis et votre solde). Rechargez votre solde de crédits, puis recommencez.

### 3.3 Suivre et récupérer un modèle terminé

1. Dans **Mes envois**, cliquez **Actualiser** (ou l'icône rafraîchir d'une ligne) pour mettre à jour les statuts.
2. Quand un envoi affiche **Terminé**, cliquez sur la **flèche** à gauche de son nom pour déplier ses **livrables**.
3. Consultez :
   - **Rapport de toit (PDF)** et **Rapport Pro Premium (PDF)** — s'ouvrent dans un nouvel onglet ;
   - **Voir en 3D** — ouvre la visionneuse 3D externe (si un lien est disponible) ;
   - **Télécharger le CAD (XML)** — enregistre le fichier du modèle sur votre poste.

### 3.4 Produire une estimation puis un devis

**Préalable :** un envoi au statut **Terminé** ; solde de crédits suffisant pour la lecture IA (coût faible mais réel).

1. Dépliez l'envoi terminé et cliquez **« Obtenir l'estimation »**.
2. Dans la modale, cliquez **« Lire les mesures du modèle »**. L'IA lit les rapports PDF (**facturé**) et pré-remplit la superficie.
3. **Vérifiez et corrigez la superficie de plancher brute** (empreinte × nombre d'étages). Au besoin, dépliez « Voir les mesures extraites » pour contrôler ce que l'IA a lu.
4. Cliquez **« Générer l'estimation »**. L'IA d'estimation produit les lignes (cible ~265 $/pi²).
5. Passez en revue le **tableau des lignes** et le **Total estimé**.
6. Cliquez **« Créer le devis »**. Un devis budgétaire est créé dans le module Ventes.
7. Cliquez **« Ouvrir le devis »** pour le finaliser (ajuster les prix, la structure, les majorations) **avant** de l'envoyer au client.

> **Rappel.** Cette estimation est **préliminaire** et repose sur les mesures extérieures du modèle plus une cible au pied carré. Elle **doit être révisée** par un estimateur avant tout envoi contractuel.

### 3.5 Importer le modèle dans le DAO (manuel)

Le module **ne** ré-importe **pas** automatiquement le modèle dans le DAO. Si vous souhaitez retravailler la géométrie :

1. Dans les livrables, cliquez **« Télécharger le CAD (XML) »**.
2. Conservez ce fichier. Son importation dans le module DAO / cao2 relève d'une manipulation manuelle (ou d'une fonctionnalité future) ; à ce jour, aucun bouton « importer dans le DAO » n'existe.

### 3.6 Déconnecter le compte central (super-administrateur)

1. En tant que super-administrateur, ouvrez **CONCEPTION 3D → PDF3D**.
2. Dans la carte **Connexion**, cliquez **« Déconnecter »**.
3. Les jetons du compte central sont effacés ; **l'historique des envois est conservé**. Plus aucune entreprise ne peut envoyer de plan tant que le compte n'est pas reconnecté.

---

## 4. Référence

### 4.1 Points d'accès de l'API

Tous préfixés `/api/erp/v1`. La colonne « Garde » indique le contrôle d'accès appliqué côté serveur.

| Méthode | Chemin | Garde | Rôle |
|---------|--------|-------|------|
| GET | `/hover/status` | Authentifié | État de l'intégration (configuré / connecté / peut connecter) `hover.py:442` |
| POST | `/hover/oauth/connect` | Super-admin | Génère l'URL d'autorisation + jeton anti-CSRF `hover.py:485` |
| POST | `/hover/oauth/callback` | Super-admin | Échange le code contre les jetons, les stocke chiffrés `hover.py:529` |
| POST | `/hover/disconnect` | Super-admin | Efface les jetons (conserve les envois) `hover.py:613` |
| POST | `/hover/blueprints` | Admin/propriétaire d'entreprise | **Envoi de plan → 3D (coût A)** `hover.py:864` |
| GET | `/hover/jobs` | Authentifié | Liste des envois de l'entreprise `hover.py:1062` |
| GET | `/hover/jobs/{id}/measurements` | Authentifié | Mesures JSON (défini mais **non affiché** dans l'interface) `hover.py:1108` |
| GET | `/hover/jobs/{id}/report/{kind}` | Authentifié | Rapport PDF (`roof` ou `pro_premium`) `hover.py:1151` |
| GET | `/hover/jobs/{id}/image` | Authentifié | Image de présentation `hover.py:1204` |
| GET | `/hover/jobs/{id}/cad` | Authentifié | Fichier CAD (XML) `hover.py:1240` |
| POST | `/hover/jobs/{id}/measures` | Authentifié | **Lecture des mesures par IA (coût B)** `hover.py:1331` |
| POST | `/hover/jobs/{id}/refresh` | Authentifié | Rafraîchit le statut depuis le service `hover.py:1774` |
| POST | `/hover/webhook/{secret}` | **Public** (aucune auth) | Réception des notifications du service `hover.py:1790` |

### 4.2 Statuts d'un envoi

| Statut affiché | Valeur interne | Signification |
|----------------|----------------|---------------|
| **Envoyé** | `submitted` | Le plan a été accepté par le service ; reconstruction en cours. |
| **En attente** | `pending` | Envoi enregistré, en attente de traitement. |
| **Terminé** | `complete` | Le modèle 3D et les mesures sont disponibles ; les livrables s'affichent. |
| **Échoué** | `failed` | Le service n'a pas pu reconstruire le modèle. |
| *(interne)* | `error` | Erreur au moment de l'envoi (permet un nouvel essai légitime). |

Un statut **ne recule jamais** : un envoi terminé ou échoué reste dans son état final même si une notification tardive arrive.

### 4.3 Calcul du coût d'envoi (coût A)

| Élément | Valeur |
|---------|--------|
| Coût de base du service (configurable) | **299 $ US** par défaut |
| Marge appliquée | **× 1,30** |
| **Montant facturé** | **388,70 $** (affiché « ≈ 390 $ US ») |
| Réglage à 0 | Facturation de l'envoi **désactivée** |
| Super-administrateur | **Exempté** (et de toute façon ne peut pas envoyer) |
| Débité de | `public.ai_prepaid_credits` (solde de crédits prépayés, en dollars US) |
| Garde de solde | Vérification **avant** l'envoi → **402** si insuffisant |
| Idempotence | Oui (clé `hover_job:{id}` + en-tête d'envoi) — jamais de double facturation |

### 4.4 Calcul du coût d'estimation (coût B)

| Élément | Valeur |
|---------|--------|
| Modèle IA (lecture des rapports PDF) | **Claude Opus 4.8** |
| Tarif appliqué | Coût réel des jetons (entrée / sortie) **× 1,30** |
| Idempotence | **Non** — chaque « Lire les mesures » est facturé |
| Garde | Autorisation IA + solde de crédits → **402** si épuisé |

L'estimation elle-même (génération des lignes de devis) passe ensuite par l'IA d'estimation du module Ventes, elle aussi facturée en crédits.

### 4.5 Limites, bornes et quotas

| Élément | Valeur |
|---------|--------|
| Fichiers par envoi | **24** maximum |
| Taille cumulée par envoi | **80 Mo** (au-delà : 413) |
| Formats acceptés | **PDF, DWG, DXF, PNG, JPG** |
| Envois par entreprise / 24 h | **10** (au-delà : 429) |
| Envois pour toute la plateforme / 24 h | **50** (au-delà : 429) |
| Rapport / image / CAD (proxy) | **25 Mo** maximum |
| Rapports lus par l'IA (estimation) | **12 Mo** maximum |
| Mesures JSON stockées | **2 Mo** maximum |
| Modèle IA de lecture des mesures | `claude-opus-4-8`, `max_tokens` = 4000 |
| Cible d'estimation (au pied carré) | **~265 $/pi²** (fourchette APCHQ 250-500 $/pi²) |
| Durée de reconstruction | **~24 h** (asynchrone) |
| Validité du jeton anti-CSRF (OAuth) | **10 minutes** |

**Limites de fréquence par adresse IP (par minute) :** envoi = 12, webhook = 120, rafraîchissement = 30, lecture des mesures = 6.

### 4.6 Types de livrables (deliverable_id)

Le service reconstruit selon un **type de livrable**. PDF3D utilise par défaut le type **complet**.

| Identifiant | Type | Contenu |
|-------------|------|---------|
| 2 | Toit (roof) | Mesures de toit uniquement |
| **3** | **Complet (défaut)** | **Modèle 3D + mesures toit ET murs** |
| 8 | Intérieur (interior) | Reconstruction intérieure |

### 4.7 Modèle de données (tables partagées `public`)

- **`hover_oauth`** — le **compte central** unique (`id=1`) : jetons d'accès et de rafraîchissement (**chiffrés**), date d'expiration, nom du compte, jeton anti-CSRF temporaire, statut.
- **`hover_jobs`** — un enregistrement par envoi : entreprise propriétaire (`tenant_schema`), demandeur, identifiants du service, `submission_uuid`, nom, type de livrable, statut, mesures (JSON), lien de visionneuse, image de présentation, montant facturé, clé d'idempotence. Chaque entreprise ne voit que **ses** lignes.
- **`hover_webhooks`** — enregistrement des URL de notification du service et de leur état de vérification.

### 4.8 Sécurité et comportements défensifs

- **Compte central chiffré.** Les jetons du service sont chiffrés (Fernet) avant stockage ; le serveur **refuse** de stocker un jeton s'il ne peut pas le chiffrer.
- **Anti-fuite de jeton.** Les livrables ne sont récupérés que depuis le domaine officiel du service (`https://hover.to/`) ; toute autre adresse est refusée.
- **Webhook fermé par défaut.** La réception des notifications exige un **secret dans l'adresse** ; sans secret configuré, le webhook est **inerte** (aucune action). Une notification falsifiée ne peut **jamais** créer un envoi ni déclencher une facturation ; elle ne peut que mettre à jour une ligne **existante**.
- **Isolation par entreprise.** Chaque envoi est rattaché à votre schéma d'entreprise ; un tenant ne voit jamais les envois d'un autre.
- **Messages d'erreur génériques.** Le détail technique des erreurs n'est jamais exposé à l'utilisateur.
- **Mode consultation.** Aucune écriture du module n'est autorisée en lecture seule (abonnement suspendu).

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

- **Ventes / Soumissions (module 08).** L'estimation crée un **devis budgétaire** (projet « Construction ») que vous ouvrez et finalisez dans le module Ventes. Le bouton « Ouvrir le devis » y mène directement.
- **DAO / Modélisation 3D (module 31).** Le fichier CAD (XML) est prévu pour être **importé au besoin** dans le DAO ; cette importation reste **manuelle** (aucun pont automatique à ce jour).
- **Rendu 3D (module) et estimations IA.** PDF3D **partage le même solde de crédits prépayés** (`public.ai_prepaid_credits`) que le Rendu 3D et les estimations IA. Un envoi ou une lecture de mesures diminue ce solde commun.
- **Configuration / Abonnement (module 28).** En **mode consultation** (abonnement suspendu), toutes les écritures de PDF3D sont bloquées.
- **Super-Admin.** La connexion / déconnexion du compte central est une action **super-administrateur** ; elle conditionne l'usage du module pour **toutes** les entreprises.

### 5.2 Foire aux questions

**Dois-je créer un compte PDF3D pour mon entreprise ?**
Non. Un **compte central unique** est branché une fois par le super-administrateur, et toutes les entreprises l'utilisent. Vous n'avez rien à connecter vous-même.

**Pourquoi ne vois-je pas le formulaire d'envoi ?**
Trois raisons possibles : (1) le compte central n'est pas connecté ; (2) vous n'êtes pas administrateur ou propriétaire de votre entreprise (l'envoi est réservé à ce rôle) ; (3) vous êtes super-administrateur, qui est **exclu** de l'envoi.

**Le super-administrateur peut-il envoyer un plan ?**
Non. Il **connecte** le compte central, mais l'envoi lui est refusé (il n'a pas de schéma d'entreprise). Seul un administrateur d'entreprise peut envoyer et est facturé.

**Combien coûte un envoi ?**
Environ **390 $ US** (précisément 299 $ US × 1,30 = 388,70 $), débités de votre solde de crédits à la confirmation. Ce montant est **configurable** par le serveur et peut être mis à 0 pour désactiver la facturation. **Refacturez ce coût à votre client.**

**Que se passe-t-il si mon solde est insuffisant ?**
L'envoi est **refusé avant** toute dépense, avec un message indiquant le montant requis et votre solde. Rechargez, puis recommencez. Aucune dette ni surprise.

**Vais-je être facturé deux fois si je clique plusieurs fois ?**
Non pour l'**envoi** : une clé unique garantit une seule facturation par envoi. En revanche, chaque **« Lire les mesures du modèle »** (dans la modale d'estimation) est facturé séparément — ne le répétez pas inutilement.

**Combien de temps avant d'avoir mon modèle 3D ?**
**Environ 24 heures.** Le traitement est asynchrone. Revenez plus tard et cliquez « Actualiser » ; le statut passera à « Terminé ».

**Puis-je voir le modèle 3D directement dans l'ERP ?**
Non. « Voir en 3D » ouvre la **visionneuse externe** du service dans un nouvel onglet. Une visionneuse interne avait été essayée puis retirée. Vous pouvez aussi télécharger le fichier CAD (XML) pour un usage externe.

**Le fichier CAD s'importe-t-il automatiquement dans le DAO ?**
Non. Il se **télécharge** ; l'importation dans le module DAO / cao2 reste manuelle (ou à venir).

**Pourquoi le montant facturé est-il en dollars US ?**
Parce que le service facture en dollars US. Le débit se fait sur votre solde de crédits prépayés (en dollars US), avec une marge de 30 %.

**Peut-on ajuster la superficie proposée par l'IA ?**
Oui, et c'est recommandé. Le champ « Superficie de plancher brute (pi²) » est **modifiable**. Corrigez-le (empreinte × nombre d'étages) : il pilote directement l'estimation au pied carré.

**L'estimation est-elle fiable pour un contrat ?**
Non telle quelle. C'est une **estimation budgétaire préliminaire** (cible ~265 $/pi², tous corps de métier). Elle **doit être révisée** par un estimateur avant tout envoi au client.

**Que faire si un envoi reste « En attente » longtemps ?**
Cliquez sur l'icône **Rafraîchir** de la ligne pour réinterroger le service. Le traitement pouvant prendre ~24 h, un délai est normal. Un statut terminé ou échoué ne redeviendra jamais « En attente ».

### 5.3 Dépannage courant

| Symptôme | Piste |
|----------|-------|
| « PDF3D n'est pas encore activé sur ce serveur » | Les variables d'environnement du service manquent. C'est une tâche d'**administrateur d'infrastructure**. |
| « Un super-administrateur doit connecter le compte PDF3D » | Le compte central n'est pas branché : un **super-administrateur** doit cliquer « Connecter PDF3D ». |
| Le formulaire d'envoi n'apparaît pas | Compte non connecté, ou vous n'êtes pas administrateur / propriétaire d'entreprise (ou vous êtes super-administrateur, exclu de l'envoi). |
| Envoi refusé (crédits insuffisants) | Rechargez votre solde de crédits (~390 $ US requis), puis recommencez. |
| Envoi refusé (limite quotidienne) | Plafond de **10 envois / entreprise / 24 h** (ou 50 pour la plateforme) atteint. Réessayez le lendemain. |
| « Plans trop volumineux » | Vous dépassez **80 Mo cumulés** ou **24 fichiers**. Réduisez le jeu de plans. |
| Un rapport renvoie une erreur « pas encore disponible » | Le modèle n'est pas encore **Terminé** (~24 h). Patientez et rafraîchissez. |
| « Voir en 3D » est absent | Le service n'a pas fourni de lien de visionneuse pour cet envoi. |
| L'estimation ne produit aucune ligne | Message « L'IA n'a produit aucune ligne exploitable. Réessayez. » — relancez « Générer l'estimation », au besoin après avoir ajusté la superficie. |

---

## 6. Récapitulatif

- **PDF3D transforme un jeu de plans (PDF / DWG / DXF, + PNG / JPG) en modèle 3D et mesures** via le service externe **Hover**, présenté partout sous la **marque blanche « PDF3D »**. Accès : menu **CONCEPTION 3D → PDF3D** (`/hover`).
- **Un compte central unique** est connecté **une seule fois** par le **super-administrateur** ; toutes les entreprises l'utilisent ensuite. Le super-administrateur **ne peut pas envoyer de plan**.
- **Seul un administrateur / propriétaire d'entreprise** voit le formulaire d'envoi et engage la dépense. Un utilisateur ordinaire consulte les statuts et les livrables.
- **Trois zones** : Connexion (état du compte central), Envoi de plan (nom + fichiers + confirmation en deux temps), Mes envois (tableau + livrables des modèles terminés), plus une **modale Estimation** en trois temps.
- **Deux coûts distincts**, débités du solde de crédits prépayés : **A — l'envoi** (~390 $ US, fixe, **idempotent**, refusé si solde insuffisant) et **B — la lecture des mesures par IA** (variable, **facturée à chaque relecture**). À **refacturer au client**.
- **Reconstruction en ~24 h** (asynchrone) ; aucun livrable avant le statut **Terminé**. Un statut ne recule jamais.
- **Livrables** d'un modèle terminé : rapport de toit (PDF), rapport Pro Premium (PDF), image de présentation, **lien de visionneuse 3D externe** (pas de visionneuse intégrée) et **fichier CAD (XML)** à télécharger.
- **Estimation → devis** : lecture des mesures par IA → confirmation de la **superficie de plancher** → génération des lignes (cible ~265 $/pi², construction complète) → **devis budgétaire** dans le module Ventes, **à réviser avant envoi**.
- **Limites** : 24 fichiers / 80 Mo par envoi ; 10 envois / entreprise / 24 h (50 pour la plateforme). **L'importation du CAD dans le DAO reste manuelle** ; les mesures JSON ne sont pas affichées dans l'interface.
- **Sécurité** : jetons du compte central chiffrés, livrables restreints au domaine officiel, webhook fermé par défaut (secret dans l'adresse), isolation stricte par entreprise, écritures bloquées en mode consultation.

---

*Fichiers sources vérifiés :* `backend/routers/hover.py` (1932 lignes, 13 points d'accès), monté dans `backend/erp_api.py` sous `/api/erp/v1/hover` ; `frontend/src/pages/HoverPage.tsx` (541 lignes), `frontend/src/api/hover.ts` (182 lignes), `frontend/src/components/hover/HoverEstimateModal.tsx` (293 lignes), `frontend/src/components/hover/hoverEstimate.ts` ; `frontend/src/i18n/locales/fr/hover.json` et `frontend/src/i18n/locales/en/hover.json` (73 clés) ; `frontend/src/i18n/locales/fr/nav.json` (section « CONCEPTION 3D », entrée « PDF3D »).

*Manuels liés :* `08-ventes-soumissions.md` (devis budgétaire produit par l'estimation), `31-outils-dao-modelisation.md` (DAO, même section CONCEPTION 3D, cible d'un import CAD futur), `25-communication-assistant-ia.md` (moteur d'estimation IA et crédits prépayés), `28-configuration.md` (abonnement et mode consultation).
