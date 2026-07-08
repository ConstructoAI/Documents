# Module 37 — Estimation Express (sous-app publique payante)

> **Version** : 1.0 (rédaction initiale vérifiée par rapport au code source, juillet 2026)
> **Public visé** : ce manuel s'adresse aux **utilisateurs** d'Estimation Express — les **entrepreneurs** et les **employés de la construction** qui veulent obtenir rapidement une estimation de coûts, produire une soumission professionnelle et la faire approuver par un client, sans avoir de compte dans l'ERP. Il ne s'adresse pas aux développeurs.
> **Ce qu'est Estimation Express** : une **application Web publique, payée à l'usage**, qui met l'estimation par intelligence artificielle de Constructo AI à la portée de tous, **sans abonnement et sans compte ERP**. Vous décrivez votre projet (et joignez vos plans) dans un **clavardage IA**, l'IA vous répond avec une estimation chiffrée, vous pouvez transformer la conversation en **soumission professionnelle** (document HTML thématisé, signable en ligne), et même ajouter un **rendu 3D photoréaliste**. Tout est débité de **crédits prépayés** que vous achetez par carte (Stripe), au **coût réel** de l'IA majoré d'une marge.
> **Application séparée** : Estimation Express n'est **pas** un onglet interne de l'ERP Constructo AI. C'est une **sous-application autonome** (dossier `ESTIMATION_EXPRESS_REACT/frontend`, basename `/estimation` — `main.tsx:15`), servie sous `app.constructoai.ca/estimation`. Aucune authentification, aucun mot de passe, aucun jeton ERP : votre **seule identité** est un **jeton de session** conservé dans votre navigateur (= votre portefeuille de crédits).
> **Préfixe API** : `/api/estimation/v1` (`api/client.ts:20`). Le module tient dans **un seul fichier backend** : `routers/estimation_express.py` (≈ 4 089 lignes), qui expose **une trentaine de points d'accès** publics et réutilise fortement `devis.py` (rendu du document, outil de soumission, analyse multi-agent de plans), `gemini_image.py` (rendu 3D) et `ai_profiles.py` (experts et extraction de texte).
> **Code de référence (frontend, `ESTIMATION_EXPRESS_REACT/frontend/src/`)** : `main.tsx` (15, basename `/estimation`), `App.tsx` (3 routes), pages `ChatPage.tsx` (2 378 lignes), `RenderPage.tsx` (840), `SoumissionPublicPage.tsx` (382) ; composants `components/soumission/SoumissionRenderModal.tsx` (592), `components/SignatureCanvas.tsx`, `components/render/Rendu3DDropzone.tsx`, `components/render/Rendu3DControls.tsx`, `components/render/Rendu3DViewer.tsx`, `components/PlanCropper.tsx` ; API `api/client.ts`, `api/estimation.ts` ; traductions `i18n.ts`, `locales/{fr,en}/translation.json`.
> **Tables PostgreSQL (base partagée, schéma `public`, préfixe `express_*`)** : `express_sessions` (le portefeuille de crédits), `express_topups` (recharges Stripe), `express_messages` (historique du clavardage), `express_files` (plans joints), `express_holds` (réservations de crédits), `express_soumissions` (documents générés), `express_login_codes` (codes à usage unique), `express_profiles` et `express_profile_documents` (profils IA personnalisés), `express_conversations` (fils d'historique). Le module écrit aussi une trace d'usage dans `public.ai_usage_tracking`.
> **Cadrage argent** : contrairement à SEAOP (gratuit), Estimation Express est **payant à l'usage**. Chaque réponse de l'IA, chaque soumission générée et chaque rendu 3D **débitent des crédits** de votre portefeuille — au **coût réel des jetons IA**, converti en dollars canadiens et **majoré** (voir §4.3). Le portefeuille se recharge par **carte, via Stripe** (25 / 50 / 100 $ de crédits, taxes en sus). Il faut au minimum **2 $** de solde pour agir.

*Note de terminologie employée dans ce manuel :* « portefeuille » (ou « solde », `express_sessions.balance_cad` dans le code) désigne le montant de crédits prépayés rattaché à votre session ; « jeton de session » (ou « token ») désigne l'identifiant unique de votre portefeuille, conservé dans votre navigateur ; « émetteur » désigne l'entreprise qui produit la soumission (vous) ; « destinataire » (ou « client ») désigne la personne à qui vous envoyez la soumission pour approbation ; « expert IA » désigne un profil de spécialiste (électricien, plombier, entrepreneur général, etc.) qui oriente les réponses de l'IA ; « recharge » (ou « recharge de crédits ») désigne l'achat de crédits par carte ; « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « lien public » (ou `public_token`) désigne l'adresse en lecture seule d'une soumission, distincte de votre jeton de session.

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

### 1.1 Mission de l'application

Estimation Express répond à un besoin simple : **obtenir une estimation de construction fiable, tout de suite, sans s'abonner à un ERP**. Le parcours principal tient en quatre temps :

1. Vous arrivez sur la page, une **session** (portefeuille vide) se crée automatiquement, et vous **rechargez** quelques crédits par carte.
2. Vous choisissez un **expert IA** (entrepreneur général, électricien, plombier, toiture, etc.), puis vous **décrivez votre projet** dans le clavardage — en joignant vos **plans** au besoin. L'IA répond avec une estimation chiffrée ; le **coût réel** de sa réponse est déduit de votre solde.
3. Quand l'échange vous convient, vous cliquez **« Générer la soumission »** : l'IA transforme la conversation en un **document de soumission professionnel** (mise en page thématisée, logo, conditions, exclusions), avec un **lien public** que vous envoyez à votre client.
4. Le client ouvre le lien, consulte la soumission, puis l'**approuve** (avec sa **signature dessinée**) ou la **refuse**. Vous êtes averti par courriel de sa décision.

En complément, une **page de rendu 3D** (`/estimation/rendu`) permet de produire des images photoréalistes à partir d'un modèle 3D, d'une image ou d'un plan PDF, payées avec le **même portefeuille** que le clavardage.

> **À retenir.** Estimation Express est un « clone public » de l'**Estimation IA** de l'ERP (module 24 / fonctions d'estimation de `devis.py`). La différence n'est pas la qualité — c'est le **même moteur** (Claude Opus, mêmes profils experts, même génération de document) — mais le **modèle d'accès** : pas de compte, pas d'abonnement, **paiement à l'usage** sur des crédits prépayés.

### 1.2 Une application SÉPARÉE (à ne pas confondre avec l'ERP)

C'est le premier point à comprendre. Comme SEAOP (module 36) et le portail B2B (module 35), **Estimation Express n'est pas une page interne de l'ERP**. C'est une **application distincte**, avec :

- sa propre adresse (`app.constructoai.ca/estimation`) ;
- **aucune page de connexion** au sens classique (ni identifiant, ni mot de passe) ;
- son propre **jeton de session** conservé dans le navigateur — il n'ouvre **pas** l'ERP, et un compte ERP n'ouvre **pas** Estimation Express ;
- sa propre barre d'en-tête (pas de barre latérale de navigation façon ERP).

La barre d'en-tête porte un lien **« Retour à Constructo AI »** (`chat.backToApp`) qui **quitte** Estimation Express pour revenir au portail de connexion de l'ERP (`app.constructoai.ca/login`). C'est une simple redirection de page, pas un partage de session.

### 1.3 Le modèle économique : crédits prépayés, payés à l'usage

Estimation Express fonctionne comme une **carte prépayée**. Vous chargez des crédits, puis chaque action facturable en consomme selon son **coût réel**.

| | **Estimation Express** |
|---|---|
| Compte requis | **Aucun** (session anonyme automatique) |
| Abonnement | **Aucun** |
| Paiement | **Crédits prépayés**, achetés par carte via **Stripe** |
| Recharges | **25 / 50 / 100 $** de crédits (taxes en sus, voir §4.4) |
| Ce qui est facturé | Chaque **réponse IA**, chaque **soumission générée**, chaque **rendu 3D** |
| Comment c'est facturé | Au **coût réel** des jetons IA, converti en CAD (× 1,38) et majoré (× 3) |
| Solde minimal pour agir | **2 $** (en dessous, les actions sont bloquées) |
| Ce qui est gratuit | Consulter le clavardage, **attacher** un rendu à une soumission, l'**approbation** côté client |

La mécanique interne est prudente : à chaque action, un montant est **réservé** sur votre solde (une « réservation »), puis, une fois la réponse produite, le **coût réel** est prélevé et le **reste vous est remboursé** sur-le-champ. Si une opération échoue (IA indisponible, erreur réseau), la réservation est **intégralement remboursée** — vous n'êtes jamais débité pour une réponse que vous n'avez pas reçue. Un filet de sécurité récupère automatiquement les réservations restées bloquées (par exemple à la suite d'un redémarrage du serveur).

> **À retenir.** Vous ne payez **que ce que l'IA consomme réellement**. Une question courte coûte quelques cents ; une analyse de plans volumineux ou une longue soumission coûte davantage. Le coût exact de chaque réponse s'affiche **sous** la réponse (« Coût : X,XX $ ») et votre solde se met à jour en direct.

### 1.4 Pas de compte, pas de mot de passe : la session EST le portefeuille

Estimation Express n'a **ni identifiant, ni mot de passe**. À votre première visite, l'application crée une **session** (un jeton unique) et l'enregistre dans votre navigateur (`localStorage`, clé `estimation_token`). Ce jeton **est** votre portefeuille : tout ce que vous rechargez y est rattaché.

Conséquence importante : si vous changez d'appareil, ou si vous videz les données de votre navigateur, vous **perdez l'accès** à ce portefeuille — **sauf** si vous l'avez **lié à un courriel**. C'est le rôle du bouton **« Mon compte »** :

- vous saisissez votre **courriel** ;
- vous recevez un **code à 6 chiffres** (à usage unique, valable 15 minutes) ;
- vous le saisissez, et votre portefeuille est désormais **retrouvable** depuis n'importe quel appareil (courriel + nouveau code).

> **Aucun mot de passe.** L'accès multi-appareils se fait uniquement par **code à usage unique** envoyé par courriel (`account.secureNote` : « Aucun mot de passe — accès par code à usage unique. »).
>
> **1 courriel = 1 portefeuille.** Les soldes **ne fusionnent jamais**. Si vous liez un courriel qui possède **déjà** un portefeuille, l'application bascule sur **ce** portefeuille et votre solde anonyme courant **n'est plus accessible** sur cet appareil (un avertissement vous le signale avant de continuer). C'est voulu, pour éviter tout double-crédit.

### 1.5 Les trois écrans

Toute l'application tient en **trois écrans** (`App.tsx`) :

| Route (adresse) | Écran | Rôle |
|---|---|---|
| `/estimation` | **Clavardage** (`ChatPage.tsx`) | Écran principal : solde, recharge, expert IA, clavardage d'estimation, génération de soumission |
| `/estimation/rendu` | **Rendu 3D** (`RenderPage.tsx`) | Rendus photoréalistes à l'usage (portefeuille **partagé** avec le clavardage) |
| `/estimation/s/:ptoken` | **Approbation publique** (`SoumissionPublicPage.tsx`) | Page en lecture seule où **le client** consulte et approuve / refuse une soumission |

Toute autre adresse renvoie automatiquement vers le clavardage.

### 1.6 Un portefeuille unique, partagé entre le clavardage et le rendu 3D

Le clavardage (`/estimation`) et le rendu 3D (`/estimation/rendu`) partagent **strictement le même portefeuille** (même clé `estimation_token` dans le navigateur, `RenderPage.tsx:43-44`). Vous rechargez une fois, et vous dépensez indifféremment en estimations, en soumissions ou en rendus. Une recharge lancée depuis la page de rendu **y ramène** après le paiement ; une recharge lancée depuis le clavardage ramène au clavardage.

### 1.7 Comment accéder à Estimation Express et s'y repérer

**Accès.** Ouvrez `app.constructoai.ca/estimation`, ou passez par le lien « Estimation Express » présent sur la page d'accueil publique de Constructo AI. Aucune connexion n'est requise : la session se crée d'elle-même.

**La barre d'en-tête** (commune au clavardage et au rendu) contient, de gauche à droite : le **logo** et le titre **« Estimation Express »** avec le sous-titre **« Estimation de construction par IA, à l'usage »** ; un lien **« Retour à Constructo AI »** ; un lien **« Rendu 3D »** (ou **« Estimation »** sur la page de rendu) ; la **bascule de langue FR / EN** ; l'encart **« Solde »** (coloré selon l'état) ; le bouton **« Mon compte »** ; le bouton **« Recharger »**.

**Sous l'en-tête du clavardage**, une **barre d'actions** regroupe le sélecteur d'**Expert IA**, puis les boutons **« Profils IA »**, **« Apparence »**, **« Nouvelle conversation »**, **« Historique »** (visible seulement une fois lié à un courriel) et **« Générer la soumission »**.

### 1.8 Bilingue français / anglais

La bascule **FR / EN** de l'en-tête change à la fois l'**interface**, la **langue des réponses de l'IA**, la **langue du document de soumission** et les **textes par défaut** (conditions, exclusions). Votre choix est mémorisé dans le navigateur (`localStorage`, clé `estimation_lang`). La **page publique** d'approbation s'affiche dans la langue que vous aviez choisie au moment de générer ce document précis.

---

## 2. Interface

### 2.1 Disposition générale

Le clavardage et le rendu 3D partagent la même **barre d'en-tête** (§1.7). Le reste de la page diffère selon l'écran. Il n'y a **pas** de barre latérale de navigation : Estimation Express est volontairement minimaliste, centré sur une tâche à la fois.

L'encart **« Solde »** de l'en-tête change de couleur selon l'état de votre portefeuille (`balanceColor`) :

- **rouge** : solde à zéro ou négatif ;
- **ambre** : solde faible (sous le minimum de 2 $, ou proche de l'épuisement) ;
- **vert** : solde confortable.

Lorsque le solde est bloquant, le bouton **« Recharger »** s'entoure d'un **anneau ambre clignotant** pour attirer l'œil.

### 2.2 Écran du clavardage — en-tête (`ChatPage.tsx`)

En plus des éléments communs, l'en-tête du clavardage donne accès à :

- **« Retour à Constructo AI »** (icône flèche) — quitte l'application vers `app.constructoai.ca/login` ;
- **« Rendu 3D »** (icône image) — ouvre la page de rendu 3D (§2.13) ;
- **FR / EN** — bascule de langue ;
- **« Solde »** — votre portefeuille, en dollars canadiens ;
- **« Mon compte »** (icône silhouette) — ouvre la modale de liaison par courriel (§2.7) ;
- **« Recharger »** (icône portefeuille) — ouvre la modale de recharge (§2.6).

### 2.3 Écran du clavardage — barre d'actions

Sous l'en-tête, la barre d'actions rassemble :

- **Sélecteur « Expert IA »** — un menu déroulant qui choisit le spécialiste qui répondra. Il propose l'option neutre **« Assistant générique »**, un groupe **« Mes profils »** (vos experts personnalisés, s'il y en a) et un groupe **« Experts »** (les experts fournis par la plateforme, voir §4.7). Par défaut, l'entrepreneur général du Québec est présélectionné s'il est disponible. Tant que la liste se charge, l'étiquette affiche **« Chargement des experts... »**.
- **« Profils IA »** (icône robot) — ouvre la gestion de vos experts personnalisés (§2.9).
- **« Apparence »** (icône palette) — ouvre la personnalisation visuelle du document de soumission (§2.10).
- **« Nouvelle conversation »** (icône bulle +) — repart d'une conversation vierge. Désactivé s'il n'y a aucun message. Un message de confirmation rappelle que **la conversation en cours sera effacée mais que votre solde est conservé**.
- **« Historique »** (icône horloge) — ouvre la liste de vos conversations passées (§2.8). **Visible uniquement si vous avez lié votre portefeuille à un courriel.**
- **« Générer la soumission »** (icône document signé) — ouvre la modale de génération du document (§2.11). Désactivé s'il n'y a aucun message.

### 2.4 Écran du clavardage — zone de conversation

**À l'ouverture (aucun message)**, un écran d'accueil s'affiche : icône de casque de chantier, titre **« Bienvenue dans Estimation Express »**, un texte d'introduction, une note rappelant que « Chaque réponse déduit le coût réel de l'IA de votre solde », et une **carte « Exemple à essayer »** cliquable qui pré-remplit la zone de saisie avec un exemple concret (« Peux-tu me faire une estimation de l'agrandissement SEULEMENT, à prix ÉCONOMIQUE... »).

**Le fil de messages** affiche vos messages (bulles bleues, texte brut) et les réponses de l'IA (mise en forme **Markdown** : titres, listes, tableaux, gras). Sous chaque réponse de l'IA, une mention **« Coût : X,XX $ »** indique ce qui a été débité pour cette réponse précise.

Pendant que l'IA réfléchit, un indicateur **« Réflexion... »** (avec animation) s'affiche.

### 2.5 Écran du clavardage — barre de saisie

En bas de l'écran :

- **Garde de solde bloquant** — si votre solde est épuisé ou insuffisant, un encart ambre remplace la saisie (« Rechargez vos crédits pour commencer » ou « Solde faible — rechargez pour continuer ») avec un bouton **« Recharger »**.
- **Rappel non bloquant** — si le solde devient faible (mais reste suffisant), une petite note invite à recharger bientôt, sans empêcher l'envoi.
- **Pastilles de fichiers joints** — chaque plan ajouté apparaît avec son nom, sa taille en mégaoctets et un bouton pour le retirer.
- **Bouton joindre** (icône trombone) — ajoute des plans. Formats acceptés : **PDF, PNG, JPEG, WEBP**. **Maximum 5 fichiers** par message, **32 Mo par fichier** (limite alignée sur l'API d'Anthropic).
- **Zone de texte** — décrivez votre projet ou posez une question. **Entrée** envoie le message ; **Maj + Entrée** insère un saut de ligne.
- **Bouton « Envoyer »** (icône avion / animation d'envoi).
- **Avertissement** permanent : « Estimations indicatives générées par IA — à faire valider par un professionnel. »

> **Deux façons d'interroger l'IA.** Si vous **joignez des plans**, l'application lance une **analyse multi-agent** : un estimateur lit chaque plan en parallèle, puis un coordonnateur en fait la synthèse. Si vous écrivez **du texte seul**, c'est un **échange conversationnel** avec l'expert choisi, qui garde en mémoire les 16 derniers tours de la conversation. Les deux sont facturés au coût réel des jetons.

### 2.6 Modale « Recharger des crédits »

Ouverte par le bouton **« Recharger »**, elle présente les **montants de recharge** disponibles (par défaut **25 / 50 / 100 $** de crédits). Pour chaque option, l'application affiche le **montant de crédits** ajouté à votre solde et, à côté, le **total à payer taxes comprises** (les taxes TPS 5 % et TVQ 9,975 % s'ajoutent **par-dessus** le montant de crédits — par exemple, **25 $ de crédits = 28,74 $ à payer**, voir §4.4).

Un clic sur une option vous **redirige vers la page de paiement sécurisée Stripe**. Après le paiement, Stripe vous **ramène** sur Estimation Express, qui **confirme automatiquement** la recharge et **crédite** votre solde. Un pied de modale rappelle : **« Paiement sécurisé par Stripe »**.

> **Le crédit apparaît après confirmation du paiement.** Estimation Express **interroge Stripe** au retour (et un filet de sécurité au démarrage du serveur rattrape tout paiement confirmé mais non encore crédité). Si votre solde n'a pas bougé immédiatement, rafraîchissez la page : la vérification est **idempotente** (jamais de double-crédit).

### 2.7 Modale « Mon compte » (courriel + code à usage unique)

C'est ici que vous **liez votre portefeuille à un courriel** pour le retrouver partout.

- **Étape courriel** — saisissez votre adresse, puis **« Envoyer le code »**. Un code à **6 chiffres** vous est envoyé par courriel.
- **Étape code** — saisissez les 6 chiffres, puis **« Vérifier et accéder à mon solde »**. Un lien **« Changer de courriel / renvoyer un code »** permet de recommencer.
- Si votre portefeuille est déjà lié, un bandeau **« Solde lié à {courriel} »** s'affiche.
- **Avertissement de bascule** — si vous liez un courriel qui possède **déjà** un portefeuille alors que votre session anonyme contient un solde, l'application vous prévient que ce solde anonyme **ne sera plus accessible** ici (les soldes ne fusionnent pas).
- **« Déconnexion »** — démarre une **nouvelle session vierge** sur cet appareil. Si votre solde courant n'est lié à aucun courriel, un avertissement signale qu'il deviendra **inaccessible**.
- Pied de modale : **« Aucun mot de passe — accès par code à usage unique. »**

### 2.8 Modale « Historique »

**Visible uniquement une fois lié à un courriel.** Elle liste vos conversations passées : titre (ou **« Conversation sans titre »**), date, nombre de messages, et la mention **« en cours »** pour la conversation active. Pour chaque ligne :

- **Renommer** (icône crayon) — modification sur place ; **Entrée** valide, **Échap** annule ;
- **Supprimer** (icône corbeille) — avec confirmation (action irréversible).

Si aucune conversation n'est enregistrée : « Aucune conversation enregistrée pour le moment. »

> **Anonyme ou connecté.** Tant que vous n'avez **pas** lié de courriel, vous n'avez **qu'une seule** conversation « par défaut » : cliquer **« Nouvelle conversation »** l'efface. Une fois **lié** à un courriel, chaque « Nouvelle conversation » crée un **fil distinct**, conservé et retrouvable dans l'Historique sur tous vos appareils.

### 2.9 Modale « Profils IA » personnalisés

Vous pouvez créer vos **propres experts IA**, réutilisables dans le clavardage et la génération de soumission.

**Liste** — un bouton **« Nouveau profil »**, puis vos profils existants (nom + nombre de documents de référence), chacun avec **Modifier** et **Supprimer** (avec confirmation). Vide : « Aucun profil personnalisé pour le moment. » **Maximum 20 profils** par portefeuille.

**Formulaire** — deux champs :

- **« Nom du profil »** (max 120 caractères) ;
- **« Instructions (rôle et expertise de l'IA) »** (zone de texte, **max 50 000 caractères**, avec compteur) — décrivez le rôle, l'expertise, vos règles de chiffrage, le ton attendu.

**Documents de référence** (disponibles **seulement après avoir enregistré** le profil) — bouton **« Ajouter »**. Formats : **PDF, TXT, CSV, TSV, MD, XLSX, DOCX** (le texte est **extrait** et fourni à l'IA). **Maximum 5 documents**, **20 Mo par document**. Le contenu de ces documents sert de référence à l'IA lorsque ce profil est actif.

> **Une garde de solde** : créer un profil ou téléverser un document exige au moins **2 $** de solde (mesure anti-abus, ces opérations mobilisent des ressources).

### 2.10 Modale « Apparence du document »

Personnalise l'**apparence de vos soumissions** (couleurs et textes par défaut). S'applique à tous vos **nouveaux** documents générés. Réglages conservés localement dans le navigateur.

- **Thèmes prédéfinis** — 6 palettes prêtes à l'emploi : **Constructo Bleu, Vert Forêt, Rouge Brique, Anthracite, Bourgogne, Océan**.
- **Personnalisation avancée** — 8 couleurs ajustables (sélecteur + code hexadécimal) : **Couleur principale, Principale foncée, Accent, Accent clair, Texte entête, Lignes alternées, Fond sections info, Bordures**.
- **Aperçu** — un mini-devis thématisé se met à jour en direct.
- **Conditions par défaut** et **Exclusions par défaut** — deux zones de texte (une ligne par élément ; puces et numérotation ajoutées automatiquement ; max 10 000 caractères chacune). Laissées vides, elles reprennent les textes par défaut de la plateforme, adaptés à la langue.
- Boutons **Réinitialiser**, **Annuler**, **Enregistrer**.

### 2.11 Modale « Générer la soumission »

Transforme votre conversation en **document de soumission professionnel**.

**Formulaire :**

- **« Nom du projet »** ;
- Section **« Votre entreprise (émetteur) »** — **Nom de l'entreprise (obligatoire)**, Courriel, Téléphone, Adresse, Ville, Code postal, RBQ (optionnel), et un **Logo** (PNG / JPG / WEBP, **max 600 Ko**) ;
- Section **« Votre client (destinataire) »** — Nom du client, Courriel du client (avec la note : si vous indiquez le courriel du client, il **reçoit le lien d'approbation par courriel**, avec copie à vous).

Le bouton **« Générer »** produit le document. Cette génération **déduit un montant de votre solde** selon l'usage de l'IA (elle relit toute la conversation et rédige les lignes du devis).

**Écran de résultat :**

- **« Soumission {numéro} créée. »** ;
- le **lien public** de la soumission, avec un bouton **« Copier »** et **« Ouvrir la soumission »** (nouvel onglet) ;
- si vous aviez renseigné le courriel du client : la confirmation qu'**un courriel avec le lien d'approbation lui a été envoyé** (copie à vous) ; sinon, l'invitation à **partager le lien** vous-même ;
- un bouton **« Ajouter un rendu 3D »** / **« Gérer le rendu 3D »** (§2.12) ;
- un bouton **« Terminé »**.

Vos coordonnées d'émetteur et votre logo sont **mémorisés localement** pour la prochaine fois.

> **Les pourcentages financiers ne sont pas modifiables dans cet écran.** La soumission applique automatiquement une majoration standard (administration, contingences, **profit fixe de 15 %**) par-dessus le coût de base, puis les taxes (voir §4.3). Ces pourcentages sont **fixés côté serveur** et **n'apparaissent pas** comme champs ajustables dans l'interface actuelle.

### 2.12 Modale « Rendu 3D de la soumission »

Optionnelle, elle ajoute **une** image de rendu au bas de la soumission. Parcours en **5 étapes** :

1. **Téléverser** — importez un **plan (PDF) ou une image**. Les fichiers 3D **ne sont pas acceptés** dans ce contexte.
2. **Recadrer** — sélectionnez au rectangle la **zone à illustrer** (l'outil de recadrage).
3. **Paramètres** — aperçu de la zone, champ **Détails** (max 2 000 caractères), **Qualité** (Pro / Standard / Rapide) et **Résolution** (2K / 4K ; 4K indisponible en qualité Rapide), puis **« Générer le rendu »** (déduit un montant du solde).
4. **Généré** — l'image + son coût, avec **« Attacher à la soumission »** (gratuit) ou **« Refaire »**.
5. **Attaché** — confirmation, avec **Remplacer** ou **Retirer** (gratuits) et **Terminé**.

> **Un seul rendu par soumission**, remplaçable ou retirable à tout moment. Le rendu attaché apparaît au bas du document, dans l'aperçu **et** sur la page publique consultée par le client.

### 2.13 Écran « Rendu 3D » autonome (`/estimation/rendu`)

Version publique du module « Rendu 3D » de Constructo AI (module 27), avec le **même portefeuille** que le clavardage. En-tête identique (le lien de retour s'appelle ici **« Estimation »**). Corps en **trois colonnes** :

- **À gauche — zone de dépôt.** Glissez-déposez ou parcourez. Trois familles de fichiers :
  - **3D** : GLB, GLTF, OBJ, STL, FBX (**max 50 Mo**) ;
  - **Image** : PNG, JPG, WEBP (redimensionnées si nécessaire) ;
  - **PDF** : une page unique est prise telle quelle ; un PDF **multi-pages** ouvre un **sélecteur de page** (aperçu + Précédent / Suivant + « Utiliser cette page »).
- **Au centre — la visionneuse.** Un modèle 3D s'affiche dans une visionneuse orientable (tournez-le pour cadrer l'angle voulu) ; une image ou un PDF s'affiche en aperçu.
- **À droite — le panneau de commandes :**
  - **Type de rendu** : **Extérieur** (bâtiment vu de l'extérieur), **Intérieur** (pièce ou aménagement), **Objet** (produit isolé). L'application suggère « Objet » pour un fichier 3D, sinon « Extérieur ».
  - **Détails et description (optionnel)** — zone de texte (max 2 000 caractères) pour préciser matériaux, éclairage, ambiance.
  - **Qualité** : Pro / Standard / Rapide.
  - **Résolution** : 2K / 4K (4K désactivé en qualité Rapide).
  - **« Générer le rendu »** — déduit le coût réel du solde partagé. Bloqué si le solde est sous le minimum.
  - **Résultat** — l'image générée, son coût, un rappel éventuel de solde faible, puis **« Télécharger »** (PNG) ou **« Régénérer »**.

Un bandeau apparaît si le solde est épuisé, avec un bouton **« Recharger »** (qui ramène ensuite sur la page de rendu).

### 2.14 Page publique d'approbation (`/estimation/s/:ptoken`)

C'est la page que **votre client** ouvre pour consulter et décider. Elle est en **lecture seule** et **n'ouvre aucun accès** à votre solde ni à votre clavardage : son adresse repose sur un **jeton public distinct** (`ptoken`).

- **En-tête** — numéro de la soumission, nom de l'émetteur, contrôles de **zoom** (réduire / **pourcentage cliquable = ajuster à l'écran** / agrandir, entre 30 % et 200 %) et **« Imprimer / PDF »** (impression à 100 %).
- **Document** — la soumission s'affiche dans un cadre sécurisé, ajustée en largeur sur mobile, avec zoom manuel.
- **Bandeaux d'état** — « Cette soumission a été approuvée » ou « Cette soumission a été refusée », le cas échéant.
- **Actions du client** (tant qu'aucune décision n'a été prise) :
  - boutons **« Refuser »** et **« Approuver la soumission »** ;
  - **panneau d'approbation** : champ **« Votre nom complet »** (minimum 2 caractères) + **« Votre signature »** dessinée à la main dans un **canevas de signature** (avec un bouton **« Effacer »**), puis **« Confirmer l'approbation »** ;
  - **confirmation de refus** : un message de confirmation, puis **« Confirmer le refus »**.
- Pied de page : **« Consultation et approbation sécurisées — Constructo AI »**.

> **La signature dessinée est obligatoire pour approuver** : le nom seul ne suffit pas. Une fois la décision prise, elle est **définitive** (une seconde tentative sur un document déjà décidé est refusée). L'émetteur est **averti par courriel** de l'approbation comme du refus.

---

## 3. Workflows pas à pas

### 3.1 Première visite : créer une session et recharger

1. Ouvrez `app.constructoai.ca/estimation`. Une **session** (portefeuille vide) se crée automatiquement — rien à faire.
2. Cliquez **« Recharger »**.
3. Choisissez un montant de crédits (**25 / 50 / 100 $**). Le total à payer, **taxes comprises**, s'affiche (par exemple 28,74 $ pour 25 $ de crédits).
4. Vous êtes redirigé vers **Stripe** : payez par carte.
5. De retour sur Estimation Express, votre **solde** est crédité automatiquement. Vous êtes prêt à estimer.

> **Conseil.** Avant de recharger, pensez à **lier votre courriel** (§3.2) : votre portefeuille sera alors récupérable si vous changez d'appareil.

### 3.2 Lier son portefeuille à un courriel (accès multi-appareils)

1. Cliquez **« Mon compte »**.
2. Saisissez votre **courriel**, puis **« Envoyer le code »**.
3. Relevez le **code à 6 chiffres** reçu par courriel (valable 15 minutes).
4. Saisissez-le, puis **« Vérifier et accéder à mon solde »**.
5. Votre portefeuille est désormais **retrouvable partout** : sur un autre appareil, refaites « Mon compte » avec le même courriel et un nouveau code.

> Si le courriel possède **déjà** un portefeuille, l'application bascule dessus. Un solde anonyme non lié présent sur l'appareil courant **ne sera plus accessible** (avertissement affiché) — les soldes ne fusionnent pas.

### 3.3 Estimer un projet (clavardage + plans)

1. Dans le sélecteur **« Expert IA »**, choisissez le spécialiste voulu (entrepreneur général, électricien, plombier, toiture, etc.) ou laissez **« Assistant générique »**.
2. **Décrivez votre projet** dans la zone de saisie : nature des travaux, superficie, niveau de finition (économique / standard / haut de gamme), contraintes, échéancier souhaité. Plus vous êtes précis, meilleure est l'estimation.
3. Au besoin, **joignez vos plans** (icône trombone) : jusqu'à **5 fichiers** PDF ou images, **32 Mo** chacun.
4. Appuyez sur **Entrée** (ou **« Envoyer »**). L'IA répond avec une estimation chiffrée (souvent sous forme de tableau : postes, quantités, main-d'œuvre, matériaux). Le **coût** de la réponse s'affiche dessous et votre solde diminue d'autant.
5. **Itérez** : demandez des ajustements (« refais en version économique », « ajoute la démolition », « détaille l'électricité »). Chaque échange raffine l'estimation.

> **Astuce coût.** Une question ciblée coûte moins cher qu'une longue analyse de plans. Gardez un œil sur l'encart **« Solde »** (vert / ambre / rouge). Sous **2 $**, l'envoi est bloqué : rechargez pour continuer.

### 3.4 Créer un expert IA personnalisé

1. Cliquez **« Profils IA »** → **« Nouveau profil »**.
2. Donnez un **nom** (ex. « Entrepreneur général – rénovation résidentielle »).
3. Rédigez les **instructions** : rôle, expertise, règles de chiffrage, ton. Soyez explicite (jusqu'à 50 000 caractères).
4. **Enregistrez**.
5. (Facultatif) Rouvrez le profil et **ajoutez des documents de référence** (PDF, TXT, CSV, TSV, MD, XLSX, DOCX ; 5 max, 20 Mo chacun) : bordereaux de prix, gabarits de devis, notes techniques. L'IA en extrait le texte et s'en sert quand ce profil est actif.
6. Sélectionnez ensuite ce profil dans le menu **« Expert IA »** (groupe **« Mes profils »**) avant d'écrire.

### 3.5 Personnaliser l'apparence du document

1. Cliquez **« Apparence »**.
2. Choisissez un **thème prédéfini** (Constructo Bleu, Vert Forêt, Rouge Brique, Anthracite, Bourgogne, Océan) ou ajustez les **8 couleurs** à la main.
3. Au besoin, rédigez vos **Conditions par défaut** et **Exclusions par défaut** (une ligne par élément).
4. Vérifiez l'**aperçu**, puis **Enregistrez**. Vos prochaines soumissions adopteront ce style.

### 3.6 Générer une soumission

1. Après **au moins un échange** avec l'IA, cliquez **« Générer la soumission »**.
2. Renseignez le **Nom du projet**.
3. Remplissez **« Votre entreprise (émetteur) »** — le **Nom de l'entreprise est obligatoire** ; ajoutez au besoin courriel, téléphone, adresse, RBQ et votre **logo** (max 600 Ko).
4. Remplissez **« Votre client (destinataire) »** — nom, et **courriel** si vous voulez que le lien d'approbation lui soit **envoyé automatiquement**.
5. Cliquez **« Générer »** (un montant est déduit de votre solde selon l'usage IA).
6. À l'écran de résultat : **copiez le lien public**, **ouvrez** la soumission pour la vérifier, ou laissez le courriel partir vers votre client.

> **Comment le prix est bâti.** L'IA chiffre les **coûts de base** (matériaux, main-d'œuvre, équipement), puis le document applique par-dessus une **majoration** (administration, contingences, **profit 15 %**) et les **taxes** (TPS 5 %, TVQ 9,975 %). Ces pourcentages sont **standards et fixés** — ils ne se règlent pas dans l'écran de génération.

### 3.7 Ajouter un rendu 3D à la soumission

1. Depuis l'écran de résultat de la soumission, cliquez **« Ajouter un rendu 3D »**.
2. **Téléversez** un plan (PDF) ou une image (les fichiers 3D ne sont pas acceptés ici).
3. **Recadrez** la zone à illustrer.
4. Réglez **Détails**, **Qualité** et **Résolution**, puis **« Générer le rendu »** (déduit du solde).
5. **« Attacher à la soumission »** (gratuit). Le rendu apparaît au bas du document, y compris sur la page publique du client.

> Un **seul** rendu par soumission. Vous pouvez le **Remplacer** ou le **Retirer** à tout moment (ces deux actions sont gratuites).

### 3.8 Faire approuver la soumission par le client (page publique)

Du côté de **votre client** :

1. Il ouvre le **lien public** (reçu par courriel ou que vous lui avez transmis).
2. Il **consulte** la soumission (zoom, impression / PDF possibles).
3. Pour accepter : il clique **« Approuver la soumission »**, saisit son **nom complet**, **dessine sa signature**, puis **« Confirmer l'approbation »**.
4. Pour refuser : il clique **« Refuser »**, puis **« Confirmer le refus »**.

De **votre** côté : vous recevez un **courriel** vous informant de la décision. La décision est **définitive** (pas de retour en arrière sur le même document).

### 3.9 Générer un rendu 3D autonome

1. Cliquez **« Rendu 3D »** dans l'en-tête (ou ouvrez `app.constructoai.ca/estimation/rendu`).
2. **Importez** un modèle 3D (GLB / GLTF / OBJ / STL / FBX, max 50 Mo), une image (PNG / JPG / WEBP) ou un plan PDF (choisissez la page voulue si le PDF en compte plusieurs).
3. Pour un 3D : **orientez** le modèle pour cadrer l'angle souhaité.
4. Choisissez le **Type** (Extérieur / Intérieur / Objet), écrivez des **Détails** au besoin, réglez **Qualité** et **Résolution**.
5. **« Générer le rendu »** (déduit du portefeuille partagé).
6. **« Télécharger »** l'image (PNG) ou **« Régénérer »**.

### 3.10 Gérer l'historique des conversations

*(Nécessite d'avoir lié un courriel — §3.2.)*

1. Cliquez **« Historique »**.
2. **Rouvrez** une conversation en la sélectionnant, **Renommez-la** (icône crayon) ou **Supprimez-la** (icône corbeille, irréversible).
3. Créez de nouveaux fils avec **« Nouvelle conversation »** : chacun est conservé séparément et retrouvable sur tous vos appareils.

### 3.11 Se déconnecter ou changer de courriel

1. **« Mon compte »** → **« Déconnexion »** démarre une **nouvelle session vierge** sur l'appareil.
2. Si votre solde n'était **pas** lié à un courriel, un avertissement signale qu'il deviendra **inaccessible** — liez-le d'abord si vous voulez le conserver.
3. Pour **changer de courriel**, relancez « Mon compte » et le lien **« Changer de courriel / renvoyer un code »**.

---

## 4. Référence

### 4.1 Écrans et routes

| Écran | Route | Fichier | Accès |
|---|---|---|---|
| Clavardage (principal) | `/estimation` | `ChatPage.tsx` | Public (session auto) |
| Rendu 3D autonome | `/estimation/rendu` | `RenderPage.tsx` | Public (portefeuille partagé) |
| Approbation publique | `/estimation/s/:ptoken` | `SoumissionPublicPage.tsx` | Public (lecture seule, jeton distinct) |
| *(toute autre adresse)* | `*` | — | Redirige vers `/estimation` |

### 4.2 Points d'accès de l'API (≈ 30)

Tous préfixés par `/api/estimation/v1`. **Aucune authentification** ; la propriété est vérifiée par le **jeton de session** (le portefeuille) ou, pour une soumission partagée, par le **jeton public** (lecture + décision seulement).

**Configuration et données statiques**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/config` | Montants de recharge, taxes, solde minimal, nombre de fichiers |
| GET | `/experts` | Liste des experts système |
| GET | `/document-defaults?lang=` | Textes par défaut (conditions / exclusions) |

**Session, compte et crédits**

| Méthode | Chemin | Rôle |
|---|---|---|
| POST | `/session` | Crée une session (portefeuille, solde 0) |
| POST | `/account/request-code` | Envoie un code à 6 chiffres par courriel |
| POST | `/account/verify-code` | Vérifie le code → restaure / rattache / crée le portefeuille |
| POST | `/topup` | Crée la session de paiement Stripe (25 / 50 / 100 $) |
| POST | `/topup/confirm/{token}` | Vérifie le paiement et crédite (idempotent) |
| GET | `/balance/{token}` | Solde, courriel, total dépensé |

**Clavardage et conversations**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/history/{token}` | Messages de la conversation active |
| POST | `/chat/{token}` | Un tour de clavardage (message, expert, langue, fichiers) |
| POST | `/chat/{token}/reset` | Nouvelle conversation (connecté) / vider (anonyme) |
| GET | `/conversations/{token}` | Liste des fils d'historique |
| PATCH | `/conversations/{token}/{id}` | Renommer une conversation |
| DELETE | `/conversations/{token}/{id}` | Supprimer une conversation |

**Profils IA personnalisés** (privés au portefeuille)

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/profiles/{token}` | Lister vos profils |
| POST | `/profiles/{token}` | Créer un profil (solde ≥ 2 $, max 20) |
| PUT | `/profiles/{token}/{id}` | Modifier nom / instructions |
| DELETE | `/profiles/{token}/{id}` | Supprimer (+ documents) |
| GET | `/profiles/{token}/{id}/documents` | Lister les documents |
| POST | `/profiles/{token}/{id}/documents` | Téléverser un document (solde ≥ 2 $, max 5, 20 Mo) |
| DELETE | `/profiles/{token}/{id}/documents/{docId}` | Supprimer un document |

**Soumission**

| Méthode | Chemin | Rôle |
|---|---|---|
| POST | `/soumission/{token}` | Générer le document + le lien public (débite) |
| POST | `/soumission/{token}/{id}/render` | Attacher un rendu 3D (gratuit) |
| DELETE | `/soumission/{token}/{id}/render` | Retirer le rendu 3D |
| GET | `/soumission/public/{ptoken}` | Vue publique (lecture seule) |
| POST | `/soumission/public/{ptoken}/accept` | Le client approuve (nom + signature) |
| POST | `/soumission/public/{ptoken}/refuse` | Le client refuse |
| GET | `/soumissions/{token}` | Liste des soumissions du portefeuille *(API seulement, non affichée, voir §4.9)* |

**Rendu 3D**

| Méthode | Chemin | Rôle |
|---|---|---|
| POST | `/render/{token}` | Génère un rendu (débite le coût réel) |

### 4.3 Modèle de facturation (comment le coût est calculé)

Estimation Express vous facture le **coût réel** de l'IA, jamais un forfait. Deux formules :

| Action | Coût débité |
|---|---|
| **Réponse du clavardage** et **génération de soumission** | Coût réel des jetons Claude Opus (en USD) **× 1,38** (conversion USD → CAD) **× 3,0** (marge) |
| **Rendu 3D** (autonome ou de soumission) | Coût réel de l'image Gemini (en USD) **× 1,38 × 3,0** |

- **Réservation puis règlement.** À l'envoi, un montant est **réservé** ; après la réponse, le **coût réel** est prélevé et le **surplus remboursé**. Vous n'êtes jamais débité au-delà de la réservation, ni pour une opération échouée (remboursement intégral).
- **Le coût s'affiche** sous chaque réponse (« Coût : X,XX $ ») et pour chaque rendu.
- **Solde minimal : 2 $.** En dessous, les envois, générations et rendus sont **bloqués** (bouton grisé). Il n'existe **aucune action gratuite** de production (seules la consultation, l'attache d'un rendu et l'approbation côté client ne coûtent rien).

**Construction du prix d'une soumission.** Le document ne se contente pas de recopier les coûts : sur la **somme des coûts de base** (matériaux + main-d'œuvre + équipement, **sans majoration**), il applique une majoration de **≈ 30 %** — administration (≈ 3 %), contingences (≈ 12 %) et **profit fixe de 15 %** — puis les **taxes** (TPS 5 %, TVQ 9,975 %). Ces pourcentages sont **fixés côté serveur** et **ne sont pas modifiables** dans l'interface actuelle (voir §4.9).

### 4.4 Recharges et taxes

| Crédits ajoutés au solde | Taxes (en sus) | Total à payer |
|---|---|---|
| **25 $** | TPS 5 % + TVQ 9,975 % | **28,74 $** |
| **50 $** | TPS 5 % + TVQ 9,975 % | **≈ 57,49 $** |
| **100 $** | TPS 5 % + TVQ 9,975 % | **≈ 114,98 $** |

- Les **crédits** correspondent au montant nominal (25 / 50 / 100 $) ; les **taxes s'ajoutent par-dessus** au moment de payer.
- Paiement par **carte, via Stripe Checkout** (devise CAD, interface en français canadien).
- Le crédit du solde est **idempotent** : vérifié au retour de Stripe et rattrapé par un filet de sécurité au démarrage — **jamais de double-crédit**.
- Il n'y a **pas** de webhook Stripe pour Express : la confirmation se fait par **interrogation** au retour de paiement (rafraîchissez la page si le solde tarde).

### 4.5 Statuts

| Domaine | Valeurs |
|---|---|
| **Soumission** | envoye · accepte · refuse |
| **Recharge (topup)** | non créditée → créditée (`credited`) |
| **Réservation de crédits (hold)** | held (réservé) · settled (réglé) · refunded (remboursé) |
| **Code de connexion (OTP)** | actif → consommé (usage unique, 15 min, 5 essais max) |

### 4.6 Limites et bornes

| Élément | Limite |
|---|---|
| Solde minimal pour agir | **2 $** |
| Message du clavardage | 1 à 12 000 caractères |
| Plans joints — par message | **5 fichiers**, PDF / PNG / JPEG / WEBP, **32 Mo** par fichier |
| Plans joints — total par message | 40 Mo |
| Plans joints — cumul par portefeuille | 15 fichiers |
| Contexte de conversation retenu par l'IA | 16 derniers tours |
| Profils IA personnalisés | **20** par portefeuille |
| Nom de profil / instructions | 120 / 50 000 caractères |
| Documents de référence par profil | **5**, PDF / TXT / CSV / TSV / MD / XLSX / DOCX, **20 Mo** chacun |
| Conversations conservées | 300 par portefeuille |
| Logo de soumission | **600 Ko**, PNG / JPG / WEBP |
| Signature d'approbation | dessin obligatoire (image), nom ≥ 2 caractères |
| Détails de rendu | 2 000 caractères |
| Rendu 3D autonome — 3D | GLB / GLTF / OBJ / STL / FBX, **50 Mo** |
| Conditions / exclusions par défaut | 10 000 caractères chacune |
| Courriel de compte | 254 caractères |

### 4.7 Les experts IA (une soixantaine)

Le sélecteur **« Expert IA »** propose :

- **« Assistant générique »** — un estimateur généraliste (le repli par défaut) ;
- vos **« Mes profils »** personnalisés (§2.9), s'il y en a ;
- un groupe **« Experts »** — une **soixantaine** de spécialistes fournis par la plateforme.

Ces experts système sont **chargés dynamiquement** depuis les fichiers de profils de la plateforme (`ai_profiles`) ; leur **nombre exact dépend des fichiers déployés** (il n'est pas figé dans le code). On y trouve, entre autres : **Architecte, Électricien, Plombier, CVCA / chauffage-ventilation-climatisation, Toiture, Fondations, Excavation, Maçonnerie, Ingénieur en structure, Comptable de construction, RBQ et CCQ, Charpente / structure de bois**, et bien d'autres corps de métier.

> **À retenir.** Choisir le bon expert **oriente** le vocabulaire, les unités et les règles de chiffrage de l'IA. Pour une estimation multi-métiers, l'**entrepreneur général** ou l'**Assistant générique** conviennent ; pour un lot précis (électricité, toiture), choisissez le spécialiste correspondant.

### 4.8 Limites de fréquence (anti-abus)

Pour protéger le service, chaque type d'action est **plafonné par heure**, à la fois par adresse et par portefeuille :

| Action | Plafond indicatif |
|---|---|
| Clavardage | 60 / h par adresse · 40 / h par portefeuille |
| Rendu 3D | 30 / h · 20 / h |
| Génération de soumission | 20 / h · 15 / h |
| Recharge / confirmation | 30 / h |
| Envoi de code par courriel | 15 / h par adresse · 5 par 30 min par courriel |
| Profils — lecture / écriture | 180 / h · 60 / h |
| Création de session | 30 / h |

Si vous atteignez une limite, patientez un instant avant de réessayer (« Trop de requêtes. Veuillez patienter... »).

### 4.9 Pièges et éléments à connaître

- **Votre portefeuille vit dans le navigateur.** Sans **liaison par courriel** (§3.2), vider les données de navigation ou changer d'appareil vous en fait **perdre l'accès**.
- **1 courriel = 1 portefeuille**, jamais de fusion : lier un courriel déjà utilisé bascule sur son portefeuille et rend le solde anonyme courant **inaccessible** (avertissement affiché).
- **Solde sous 2 $ = tout est bloqué** : il n'existe pas d'action de production gratuite.
- **Les pourcentages Administration / Contingences / Profit ne sont PAS ajustables** dans l'écran de génération de soumission. Bien que des libellés et un support technique existent en coulisse, **aucun champ** ne les rend modifiables dans l'interface actuelle : le serveur applique des valeurs standards (profit **15 %** fixe). Ne comptez pas les régler par soumission.
- **Aucun tableau de suivi des soumissions dans l'application.** Il n'existe **pas d'écran** listant vos soumissions passées et leur statut d'approbation. Vous apprenez la décision du client par **courriel**. Conservez le **lien public** de chaque soumission générée.
- **Rendu de soumission = image / PDF seulement**, **un seul** rendu, remplaçable ou retirable. Les fichiers 3D ne sont acceptés que sur la page de rendu **autonome** (`/estimation/rendu`).
- **4K indisponible en qualité « Rapide »** (la Rapide privilégie la vitesse ; utilisez Standard ou Pro pour le 4K).
- **L'historique n'apparaît qu'une fois lié à un courriel.** En anonyme, vous n'avez qu'une conversation « par défaut », effacée par « Nouvelle conversation ».
- **La signature dessinée est obligatoire** pour approuver côté client ; le nom seul ne suffit pas.
- **Estimations indicatives.** Comme le rappelle l'avertissement, ce sont des estimations **générées par IA, à faire valider par un professionnel** avant tout engagement contractuel.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec le reste de Constructo AI

- **Estimation IA de l'ERP (module 24 / `devis.py`).** Estimation Express est le **clone public** de cette fonction. Même moteur (Claude Opus), mêmes profils experts, même générateur de document — mais **sans compte ERP**, en **paiement à l'usage**. À l'inverse, dans l'ERP, l'estimation IA est incluse dans l'abonnement du locataire.
- **Rendu 3D (module 27).** La page `/estimation/rendu` est la **version publique** du module de rendu, avec le **même portefeuille** que le clavardage. Moteur d'images : **Google Gemini**.
- **SEAOP (module 36).** À ne pas confondre : SEAOP est **gratuit** et sert aux **appels d'offres publics** ; son service d'estimation professionnelle est produit **par l'équipe** et facturé hors ligne. Estimation Express, lui, est **libre-service, payé à l'usage**, et l'estimation est produite **par l'IA** en direct.
- **Stripe.** Sert uniquement aux **recharges** (paiement par carte). La clé Stripe est **partagée avec l'ERP** côté serveur et n'est **jamais** exposée. Aucun abonnement : que des achats de crédits ponctuels.
- **Claude (Anthropic) et Gemini (Google).** Les moteurs d'IA. Vous n'êtes facturé que leur **coût réel** de jetons / d'image, converti et majoré (§4.3).
- **Courriels (SMTP système).** Servent aux **codes de connexion** (OTP), à l'**envoi du lien de soumission** au client, et à la **notification de décision** à l'émetteur. Ces envois sont « au mieux » (ils ne bloquent jamais une opération).

### 5.2 Ce qui n'est PAS possible

- **Pas de compte ni de mot de passe.** Uniquement un jeton de navigateur et, en option, une liaison par **code à usage unique**.
- **Pas de tableau de bord de suivi** des soumissions ni des approbations dans l'application (le suivi passe par le courriel de décision et par vos liens publics conservés).
- **Pas de réglage des pourcentages financiers** (administration, contingences, profit) dans l'écran de soumission.
- **Pas de rendu 3D à partir d'un fichier 3D dans le contexte d'une soumission** (image / PDF uniquement) ; ni plus d'**un** rendu par soumission.
- **Pas de 4K en qualité Rapide.**
- **Pas de fusion de portefeuilles** entre deux courriels.
- **Pas d'abonnement** : uniquement des crédits prépayés.

### 5.3 Foire aux questions

**Dois-je créer un compte pour utiliser Estimation Express ?**
Non. Une session se crée automatiquement à votre arrivée. Vous pouvez, en option, **lier un courriel** (« Mon compte ») pour retrouver votre portefeuille sur d'autres appareils — mais toujours **sans mot de passe**, par code à usage unique.

**Combien ça coûte ?**
Vous achetez des **crédits** (25 / 50 / 100 $, taxes en sus). Chaque réponse de l'IA, chaque soumission et chaque rendu **débite le coût réel** de l'IA, majoré (× 1,38 × 3). Le coût s'affiche sous chaque réponse. Il faut au moins **2 $** de solde pour agir.

**Pourquoi 25 $ de crédits me coûtent 28,74 $ ?**
Les **taxes** (TPS 5 % et TVQ 9,975 %) s'ajoutent **par-dessus** le montant de crédits. Vous obtenez bien 25 $ de crédits, et vous payez 28,74 $ taxes comprises.

**J'ai rechargé mais mon solde n'a pas bougé.**
Le crédit se confirme **au retour de Stripe**. **Rafraîchissez la page** : la vérification est automatique et ne peut pas créditer deux fois. Un filet de sécurité rattrape aussi les paiements confirmés au démarrage du serveur.

**Je change d'ordinateur : est-ce que je retrouve mon solde ?**
Seulement si vous l'avez **lié à un courriel**. Sinon, le portefeuille reste attaché au navigateur d'origine.

**Puis-je fusionner deux soldes ?**
Non. **1 courriel = 1 portefeuille**, sans fusion. Basculer vers un courriel déjà utilisé rend le solde anonyme courant inaccessible (l'application vous prévient).

**L'estimation est-elle contractuelle ?**
Non. Ce sont des **estimations indicatives générées par IA**, à **faire valider par un professionnel** avant tout engagement.

**Comment mon client approuve-t-il ?**
Il ouvre le **lien public** de la soumission, la consulte, puis clique **« Approuver »**, saisit son **nom** et **dessine sa signature**, ou clique **« Refuser »**. Vous êtes averti **par courriel** de sa décision.

**Puis-je changer le profit ou les contingences d'une soumission ?**
Pas dans l'interface actuelle. Le document applique des pourcentages **standards fixés** (profit 15 %) par-dessus les coûts de base.

**Où est la liste de mes soumissions ?**
Il n'y a **pas** d'écran de suivi dans Estimation Express. Conservez le **lien public** de chaque soumission ; la décision du client vous parvient par **courriel**.

**Quelle est la différence avec l'estimation de SEAOP ?**
SEAOP est **gratuit** et met en relation ; sa « demande d'estimation » est produite **par l'équipe** et facturée hors ligne. Estimation Express est **libre-service, payé à l'usage**, et l'estimation est produite **par l'IA** immédiatement.

**Puis-je téléverser un modèle 3D dans une soumission ?**
Non : dans le contexte d'une soumission, seuls **images et PDF** sont acceptés. Pour un rendu à partir d'un fichier **3D**, utilisez la page **Rendu 3D** autonome (`/estimation/rendu`).

**Le service est-il bilingue ?**
Oui. La bascule **FR / EN** change l'interface, la **langue des réponses de l'IA** et la **langue du document**. La page publique s'affiche dans la langue choisie au moment de générer ce document.

### 5.4 Dépannage courant

| Symptôme | Piste |
|---|---|
| Impossible d'envoyer un message | Solde sous **2 $** : rechargez. |
| Solde inchangé après paiement | **Rafraîchissez** la page (confirmation au retour de Stripe, idempotente). |
| Bouton « Historique » absent | Il n'apparaît qu'une fois le portefeuille **lié à un courriel** (« Mon compte »). |
| « Trop de requêtes » | Limite de fréquence atteinte : patientez un instant (§4.8). |
| Solde anonyme « disparu » après liaison courriel | Vous avez basculé sur un **autre** portefeuille (1 courriel = 1 portefeuille, pas de fusion). |
| Fichier refusé au clavardage | Vérifiez le format (PDF / PNG / JPEG / WEBP), la taille (**32 Mo**) et le nombre (**5** max). |
| Fichier 3D refusé dans une soumission | Normal : image / PDF seulement ici. Utilisez `/estimation/rendu` pour un 3D. |
| 4K grisé | Vous êtes en qualité **Rapide** : choisissez Standard ou Pro. |
| Le client ne peut pas approuver | La **signature dessinée** est obligatoire (le nom seul ne suffit pas). |
| « Le service IA est momentanément indisponible » | Réessayez : **aucun crédit n'a été déduit** (remboursement automatique). |
| Je ne retrouve pas mon portefeuille | Sans liaison courriel, il reste attaché au navigateur d'origine. |

---

## 6. Récapitulatif

- **Estimation Express est une application publique, payée à l'usage**, qui met l'estimation IA de Constructo AI à la portée de tous, **sans compte ERP ni abonnement**. C'est le **clone public** de l'Estimation IA de l'ERP (même moteur), servie sous `app.constructoai.ca/estimation`.
- **Le modèle est celui d'une carte prépayée** : on achète des **crédits** (25 / 50 / 100 $, taxes en sus, via **Stripe**), et chaque **réponse IA**, **soumission** ou **rendu 3D** débite le **coût réel** de l'IA, converti (× 1,38) et majoré (× 3). Solde minimal pour agir : **2 $**. Réservation puis règlement, avec **remboursement du surplus** et des opérations échouées.
- **Aucun mot de passe.** L'identité est un **jeton de navigateur** (= le portefeuille) ; on peut le **lier à un courriel** par **code à usage unique** pour le retrouver sur tout appareil. **1 courriel = 1 portefeuille**, sans fusion.
- **Trois écrans** : le **clavardage** d'estimation (avec experts, profils personnalisés, apparence, historique et génération de soumission), le **rendu 3D** autonome (portefeuille partagé), et la **page publique** d'approbation (le client consulte et **signe**).
- **La soumission** est un document professionnel thématisé, avec logo, conditions et exclusions ; son prix applique une majoration standard (**profit 15 % fixe**) et les taxes. Le client l'approuve avec une **signature dessinée** ; l'émetteur est **averti par courriel**.
- **Une soixantaine d'experts IA** (chargés dynamiquement, nombre non figé) orientent le chiffrage ; on peut créer ses **propres profils** (instructions + documents de référence).
- **Limites clés à connaître** : les **pourcentages financiers ne sont pas modifiables** dans l'interface, il n'y a **pas de tableau de suivi** des soumissions (décision par courriel), le rendu de soumission est limité à **une** image / PDF, et le **4K** est indisponible en qualité **Rapide**.
- **Volumétrie technique** : un **seul fichier backend** (`estimation_express.py`, ≈ 4 089 lignes, ≈ 30 points d'accès sous `/api/estimation/v1`), tables `public.express_*`, réutilisant `devis.py`, `gemini_image.py` et `ai_profiles.py`.

---

*Fichiers sources vérifiés :* backend — `ERP_REACT/backend/routers/estimation_express.py` (≈ 4 089 lignes, fichier unique du module ; montage `erp_api.py:1399` sous `/api/estimation/v1`, SPA `erp_api.py:1408`). Frontend (`ESTIMATION_EXPRESS_REACT/frontend/src/`) — `main.tsx` (15, basename `/estimation`), `App.tsx` (routes), `pages/ChatPage.tsx` (2 378), `pages/RenderPage.tsx` (840), `pages/SoumissionPublicPage.tsx` (382), `components/soumission/SoumissionRenderModal.tsx` (592), `components/SignatureCanvas.tsx`, `components/render/Rendu3DDropzone.tsx`, `components/render/Rendu3DControls.tsx`, `components/render/Rendu3DViewer.tsx`, `components/PlanCropper.tsx`, `api/client.ts`, `api/estimation.ts`, `i18n.ts`, `locales/{fr,en}/translation.json`. Constantes de facturation vérifiées : `USD_TO_CAD = 1.38` (`estimation_express.py:91`), `MARKUP = 3.0` (`:92`), `RENDER_MARKUP = 3.0` (`:3878`), `_MIN_BALANCE_CAD = 2.0` (`:101`), `_TOPUP_AMOUNTS = [25, 50, 100]` (`:95`), `TPS_RATE = 0.05` / `TVQ_RATE = 0.09975` (`:96-97`), `_MAX_FILES = 5` / `_MAX_FILE_BYTES = 32 Mo` (`:110-112`).

*Manuels liés :* `24-communication-assistant-ia.md` (l'estimation IA interne de l'ERP, dont Estimation Express est le clone public), `27-conception3d-rendu-3d.md` (le module de rendu 3D dont dérive `/estimation/rendu`), `07-ventes-soumissions.md` (les soumissions internes de l'ERP), `36-programme-seaop.md` et `35-programme-portail-b2b.md` (les autres applications autonomes servies par l'ERP).

*Manuel ERP Constructo AI — Module 37 « Estimation Express (sous-app publique payante) » — v1.0 vérifié — 2026-07.*
