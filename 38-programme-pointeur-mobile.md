# Module 38 — Pointeur mobile (PWA de pointage employés)

> **Version** : 1.0 (rédaction initiale vérifiée par rapport au code source, juillet 2026)
> **Public visé** : ce manuel s'adresse aux **utilisateurs** du Pointeur mobile — les **entrepreneurs**, **contremaîtres** et **employés de la construction** qui pointent leurs heures, échangent sur le chantier, consultent leurs dossiers et gèrent l'inventaire depuis leur téléphone. Il ne s'adresse pas aux développeurs.
> **Ce qu'est le « Pointeur mobile »** : malgré son nom, ce n'est pas une simple pointeuse. C'est une **application mobile complète** (une « PWA », application Web installable sur le téléphone) qui apporte une bonne partie de l'ERP Constructo AI directement sur le terrain : **pointer ses heures** avec géolocalisation et météo automatiques, **messagerie** de chantier (canaux et messages directs), **dossiers 360** (notes, photos, étapes, extras), **documents commerciaux** (devis, factures, bons de travail, bons de commande), **scanner de code-barres** pour l'inventaire, **assistant IA**, **météo de chantier**, **calculatrice de construction** et plus encore. Le pointage n'est qu'une des six grandes familles de fonctions.
> **Application SÉPARÉE (pas un onglet de l'ERP Web)** : le Pointeur mobile est une **application autonome** (dossier `MOBILE_REACT/frontend`, React 18 + Vite + Zustand + Tailwind), **co-hébergée** par l'ERP et servie sous **`app.constructoai.ca/mobile`**. Elle possède son **propre système de connexion** (jeton distinct de l'ERP) : on se connecte avec le **courriel de l'entreprise**, puis on choisit **son nom** dans la liste des employés, puis on saisit son **code NIP à 4 chiffres**. Un compte ERP n'ouvre pas l'application mobile, et l'inverse est vrai aussi.
> **Préfixe API** : `/api/mobile/v1` (plus `/api/mobile/v1/attachments` pour les pièces jointes).
> **Code de référence (backend, `MOBILE_REACT/backend/`)** : `mobile_api.py` (5 278 lignes, **118 points d'accès**), `attachments_api.py` (361 lignes, **7 points d'accès**), `mobile_database.py` (12 822 lignes — logique métier et base de données), `mobile_auth.py` (266 lignes — connexion, jetons, URL signées), `mobile_models.py` (1 390 lignes), `mobile_pdf_service.py` (635 lignes — PDF), `mobile_attachments_service.py` (434 lignes — pièces jointes), `mobile_idempotency.py` (264 lignes — **présent mais non activé**, voir §4.10).
> **Code de référence (frontend, `MOBILE_REACT/frontend/src/`)** : `App.tsx` (**24 routes**), `components/layout/MobileLayout.tsx` (256 lignes — la coquille), pages `LoginPage.tsx`, `DashboardPage.tsx`, `PunchPage.tsx` (504 lignes), `CrewPage.tsx`, `MessagesPage.tsx` / `ChannelChatPage.tsx` / `DmChatPage.tsx`, `DossiersPage.tsx` / `DossierDetailPage.tsx` (1 471 lignes), `DocumentsHubPage.tsx` / `DocumentListPage.tsx` / `DocumentDetailPage.tsx` / `DocumentFormPage.tsx`, `AiAssistantPage.tsx`, `MeteoPage.tsx`, `ScannerPage.tsx` (1 123 lignes), `ReceptionPage.tsx`, `CountingPage.tsx`, `CalculatricePage.tsx`, `RemindersPage.tsx`, `AuditLogPage.tsx`, `AppearancePage.tsx`, `PhotoUploadPage.tsx`.
> **Tables PostgreSQL (base partagée avec l'ERP)** : les données de pointage, dossiers, documents et inventaire vivent dans le **schéma de votre entreprise** (le même que l'ERP Web) ; certaines fonctions transversales utilisent des tables du schéma `public` préfixées `mobile_*` (`mobile_punch_photos`, `mobile_push_subscriptions`, `mobile_auth_rate_limit`, `mobile_audit_events`, `mobile_idempotency`, `mobile_recurrent_invoices_config`, etc.). Le pointage écrit dans `time_entries`, les mouvements de stock dans `mouvements_stock` (**les mêmes tables que l'ERP**) : ce que vous saisissez au chantier est immédiatement visible au bureau, et inversement.
> **Cadrage important** : l'application mobile **partage la base de données de l'ERP**. Elle n'est donc pas un système parallèle : c'est une **fenêtre terrain sur les mêmes données**. Les permissions y sont contrôlées par **quatre rôles** (ADMIN, MANAGER, EMPLOYE, APPRENTI) qui décident de ce que chacun voit et peut faire.

*Note de terminologie employée dans ce manuel :* « PWA » (Progressive Web App) désigne une application Web qui s'installe et se comporte comme une application native sur le téléphone ; « locataire » ou « entreprise » (tenant) désigne votre organisation dans Constructo AI ; « pointer l'entrée / la sortie » (punch in / punch out) désigne le fait de démarrer ou d'arrêter le chronomètre de vos heures ; « BT » désigne un **bon de travail** ; « BC » un **bon de commande** ; « Fiche 360 » désigne la vue détaillée d'un dossier réunissant toutes ses informations ; « rôle » désigne votre niveau d'accès (ADMIN, MANAGER, EMPLOYE, APPRENTI) ; « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « hors-ligne » (offline) désigne le fonctionnement sans connexion Internet ; « pièce jointe » désigne un fichier attaché (photo, PDF, document) ; « NIP » désigne votre code personnel à 4 chiffres.

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

Le Pointeur mobile répond à un besoin simple : **donner aux gens du chantier l'accès à l'essentiel de l'ERP, sur leur téléphone, même sans réseau**. Un employé arrive sur un site, ouvre l'application, choisit son bon de travail et **pointe son entrée** ; sa position et la météo sont enregistrées **automatiquement**. En fin de journée, il **pointe sa sortie** et peut, au besoin, faire **signer un superviseur** présent sur place. Entre-temps, il consulte ses **messages** de chantier, ouvre le **dossier** du projet, prend une **photo de note**, pose une question à l'**assistant IA** ou vérifie la **météo** des prochains jours.

Le parcours quotidien type tient en quelques gestes :

1. **Se connecter** une fois (courriel d'entreprise, choix de son nom, code NIP) — la session reste ouverte 24 heures.
2. **Pointer l'entrée** sur le bon de travail du jour ; **pointer la sortie** en fin de quart.
3. **Communiquer** avec l'équipe (canaux, messages directs) et **consulter les dossiers**.
4. Selon son rôle, **gérer l'inventaire** (scanner, réception, comptage), **produire des documents** (devis, factures, bons) ou **suivre les impayés**.

> **À retenir.** Le pointage est le cœur historique du module, mais l'application est en réalité un **mini-ERP de chantier**. Ce que vous y faites (pointer, bouger du stock, créer un document, ajouter une note) est écrit dans **la même base de données** que l'ERP Web : aucune ressaisie au bureau.

### 1.2 Une application SÉPARÉE (à ne pas confondre avec l'ERP Web)

C'est le premier point à comprendre. Le Pointeur mobile **n'est pas une page interne de l'ERP**. C'est une **application distincte**, avec :

- sa propre adresse : **`app.constructoai.ca/mobile`** ;
- sa **propre connexion** — un jeton d'authentification **distinct** de celui de l'ERP (le fait d'être connecté à l'un n'ouvre pas l'autre) ;
- son **propre design** épuré, pensé pour le pouce et le plein soleil : une barre en haut, une barre de navigation en bas, pas de barre latérale façon bureau.

L'ancienne adresse dédiée `mobile.constructoai.ca` **redirige** désormais automatiquement vers `app.constructoai.ca/mobile`.

> **Conséquence pratique du déménagement sous `/mobile`.** Si vous aviez installé l'application ou activé les notifications à l'ancienne adresse, il faut **réinstaller** l'application et **réactiver les notifications** (voir §2.17). Les anciens abonnements ne sont plus valides.

### 1.3 Comment y accéder et installer l'application

**Ouvrir l'application.** Dans le navigateur du téléphone, allez à **`app.constructoai.ca/mobile`**. La page de connexion s'affiche.

**Installer sur l'écran d'accueil (recommandé).** Comme toute PWA, l'application peut s'ajouter à l'écran d'accueil pour s'ouvrir en plein écran, comme une vraie application :

- **iPhone / iPad (Safari)** : bouton **Partager** → **« Sur l'écran d'accueil »**.
- **Android (Chrome)** : menu **⋮** → **« Installer l'application »** (ou la bannière proposée).

Une fois installée, l'application fonctionne **hors-ligne** pour l'essentiel (voir §1.7) et peut recevoir des **notifications push** (§2.17).

### 1.4 La connexion en trois étapes

L'application ne demande **pas** un identifiant et un mot de passe personnels comme l'ERP. La connexion se fait en **trois temps** (voir le détail à la §2.2) :

1. **L'entreprise** — vous saisissez le **courriel de l'entreprise** et son **mot de passe**. C'est la même entreprise (le même « locataire ») que dans l'ERP.
2. **Qui êtes-vous ?** — l'application affiche la **liste des employés** de l'entreprise ; vous **touchez votre nom**.
3. **Votre code NIP** — vous saisissez votre **code personnel à 4 chiffres**. Au quatrième chiffre, la connexion se fait **automatiquement**.

Une fois connecté, votre session reste valide **24 heures**. Votre rôle, le fuseau horaire de l'entreprise et vos permissions sont mémorisés sur l'appareil.

> **Appareil partagé, en toute sécurité.** Si plusieurs employés utilisent le même téléphone (cas fréquent au chantier), l'application **efface** au moment de la connexion les données en attente et les pièces jointes en cache d'un **autre** employé. Chacun ne voit que ses propres données.

### 1.5 Les quatre rôles (ce que chacun peut faire)

Tout ce que vous voyez et pouvez faire dépend de votre **rôle**, défini par votre entreprise. Il y en a **quatre** :

| Rôle | Pour qui | Accès type |
|---|---|---|
| **ADMIN** | Propriétaire, administrateur | **Tout** : pointage, messagerie, dossiers, **tous** les documents (devis / factures / bons), inventaire, assistant IA avec outils ERP, **relances d'impayés**, **journal d'audit** |
| **MANAGER** | Contremaître, chargé de projet | Comme ADMIN **sauf le journal d'audit** ; approuve les heures de l'équipe, gère l'inventaire et les documents financiers |
| **EMPLOYE** | Employé de chantier (défaut) | Pointage, messagerie, dossiers, **bons de travail seulement** (pas les devis / factures / bons de commande), assistant IA **général** (sans outils ERP), calculatrice, météo |
| **APPRENTI** | Apprenti | Comme EMPLOYE |

En plus du rôle, une **permission de gestion de stock** (`peut gérer le stock`) peut être accordée individuellement : elle débloque le **scanner de mouvements**, la **réception** et le **comptage** même pour un employé qui n'est pas ADMIN/MANAGER (les rôles ADMIN et MANAGER l'ont d'office).

> **À retenir.** Le rôle est **vérifié côté serveur** : l'interface cache les fonctions non autorisées, mais même en contournant l'affichage, le serveur refuse (erreur 403) toute action interdite. Un employé ne peut donc **jamais** voir un devis ou une facture, ni approuver ses propres heures.

### 1.6 Les six familles de fonctions

L'application réunit une vingtaine d'écrans, qu'on peut regrouper en **six familles** :

| Famille | Écrans | Pour qui |
|---|---|---|
| **1. Pointage & temps** | Accueil, Pointage (entrée / sortie, historique, résumé hebdomadaire), Équipe sur chantier | Tous ; l'approbation des heures = contremaîtres |
| **2. Communication** | Messages (canaux + messages directs), Assistant IA, Notifications | Tous |
| **3. Dossiers de projet** | Dossiers, Fiche 360 (14 onglets : notes, étapes, extras, documents, etc.) | Tous ; volet financier réservé ADMIN/MANAGER |
| **4. Documents commerciaux** | Documents (devis, factures, bons de travail, bons de commande), Relances d'impayés | Bons de travail = tous ; le reste = ADMIN/MANAGER |
| **5. Inventaire / magasin** | Scanner (code-barres), Réception, Comptage | Gestion de stock (ADMIN/MANAGER ou permission) |
| **6. Outils terrain** | Météo Chantier, Calculatrice de construction, Apparence, Journal d'audit | Tous ; journal d'audit = ADMIN |

### 1.7 Le mode hors-ligne (chantier sans réseau)

Le chantier est souvent hors de portée du réseau. L'application est conçue pour **continuer à fonctionner sans connexion** :

- vos données essentielles (pointage en cours, bons de travail, historique, dossiers) sont **préchargées** pendant que vous êtes en ligne et restent **consultables hors-ligne** ;
- quand vous **agissez** hors-ligne (pointer, ajouter une note, créer un document, valider un panier de stock…), l'action est **mise en file d'attente** dans le téléphone ;
- dès que le réseau revient, tout se **synchronise automatiquement** ; un indicateur « Hors-ligne / N actions en attente » et un bouton **« Synchroniser maintenant »** vous tiennent informé.

**18 types d'actions** peuvent ainsi être différés puis rejoués (voir la liste complète à la §4.10). Deux exceptions notables : la **réception** et le **comptage** d'inventaire exigent une connexion, et l'**édition d'un document existant** est désactivée hors-ligne (pour éviter d'écraser des données modifiées au bureau entre-temps).

> **À retenir.** Vous pouvez pointer votre entrée et votre sortie **sans réseau** : l'heure exacte de votre geste est enregistrée localement, et le pointage remonte tel quel dès la reconnexion. Vous ne perdez pas vos heures parce que le chantier n'a pas de Wi-Fi.

### 1.8 Bilingue, thème clair/sombre et apparence

- **Français / anglais** — une bascule **FR / EN** dans la barre du haut change toute l'interface. Votre choix est mémorisé sur l'appareil.
- **Thème clair / sombre** — une bascule soleil / lune adapte l'affichage aux conditions (plein soleil, travail de nuit).
- **Apparence personnalisée** — l'écran **Apparence** (§2.16) permet de choisir les **couleurs** de l'interface (thèmes prédéfinis ou couleurs sur mesure). Ce réglage est **local à l'appareil**.

---

## 2. Interface

### 2.1 La coquille : barres de navigation (`MobileLayout.tsx`)

Une fois connecté, tous les écrans (sauf la connexion) s'affichent dans une **coquille** commune composée de trois zones.

**Barre supérieure (en haut).** De gauche à droite :

- une **pastille « C »** et votre **Prénom Nom**, avec le **nom de l'entreprise** en dessous ;
- la **bascule de langue FR / EN** ;
- la **cloche de notifications** (activer / désactiver les notifications push, §2.17) ;
- la **bascule de thème** (soleil = passer en clair, lune = passer en sombre).

**Barre de navigation inférieure (en bas) — 5 onglets.** C'est le cœur de la navigation :

| Onglet | Écran | Note |
|---|---|---|
| **Accueil** | Tableau de bord | Vue d'ensemble de la journée |
| **Pointage** | Entrée / sortie et historique | Le cœur du module |
| **Messages** | Messagerie | **Pastille rouge** = nombre de messages non lus (rafraîchie toutes les 30 secondes) |
| **Dossiers** | Liste des dossiers | Fiches 360 des projets |
| **Plus** | Menu glissant | Ouvre le menu des autres fonctions |

**Menu « Plus » (feuille glissante).** Un appui sur **Plus** fait remonter un menu depuis le bas :

- **Équipe** (vue contremaître) ;
- **Assistant IA** ;
- **Météo Chantier** ;
- **Calculatrice** ;
- **Apparence** ;
- **Relances factures** — *visible seulement pour ADMIN et MANAGER* ;
- **Journal d'audit** — *visible seulement pour ADMIN* ;
- **Déconnexion**.

> **Où sont les autres écrans ?** Certains écrans (Documents, Scanner, Réception, Comptage) ne sont pas dans la barre du bas ni dans le menu Plus : on les atteint depuis la **grille d'actions rapides** de l'Accueil (§2.3), selon votre rôle et vos permissions.

### 2.2 Écran de connexion (`LoginPage.tsx`)

L'écran de connexion, hors de la coquille, affiche un bandeau bleu marine, le logo, le titre **« Constructo AI »** et le sous-titre **« Application mobile »**. La connexion se déroule en **trois étapes**, avec possibilité de revenir en arrière à chaque fois.

**Étape 1 — Entreprise.** Titre **« Connexion entreprise »**. Deux champs :

- **Courriel de l'entreprise** (exemple affiché : `info@monentreprise.ca`) ;
- **Mot de passe**.

Bouton **« Continuer »**. C'est le **compte de l'entreprise** (le locataire), pas un compte personnel.

**Étape 2 — « Qui êtes-vous ? »** L'application affiche la **liste des employés actifs** de l'entreprise : pour chacun, un **avatar**, le **Prénom Nom** et le **poste**. Vous **touchez votre nom**. Si la liste est vide : « Aucun employé trouvé. »

**Étape 3 — Code NIP.** Titre **« Entrez votre code NIP à 4 chiffres »**. Quatre cases numériques avec **avance automatique** ; dès le **quatrième chiffre saisi**, la connexion se fait toute seule (pas besoin de bouton). Une flèche permet de revenir choisir un autre employé.

Le pied de page indique la version : **« Constructo AI Mobile v1.0 »**.

> **Sécurité du NIP.** Le code NIP est **vérifié sur le serveur** (jamais dans le téléphone). Il est protégé contre les tentatives répétées : au-delà de **6 essais erronés** pour un même employé (sur 5 minutes), les tentatives sont **temporairement bloquées**. Un code oublié se réinitialise par votre administrateur.

### 2.3 Accueil / Tableau de bord (`DashboardPage.tsx`)

L'écran d'Accueil résume la journée :

- **Salutation** — « Bonjour, {prénom} ! » et la **date longue** du jour.
- **Carte de statut du pointage** (cliquable, mène au Pointage) — indique **« En service »** ou **« Hors service »**, la **durée écoulée** (« Depuis {durée} ») ou « Aucun pointage actif », une **pastille verte animée** quand vous êtes pointé, et, le cas échéant, le **projet** et le **numéro de bon de travail** en cours.
- **Deux tuiles de statistiques** — **« Heures cette semaine »** (total en heures) et **« Messages non lus »** (compteur).
- **Grille d'actions rapides** — des raccourcis vers : **Pointage, Équipe, Messages** (avec pastille), **Dossiers, Documents, Météo, Assistant IA, Calculatrice, Scanner**, et — **uniquement si vous gérez le stock** — **Réception** et **Comptage**.

> **Note.** Les écrans **Téléversement photo** (`/photo`), **Relances**, **Journal d'audit**, **Apparence** et **Comptage** ne figurent pas dans la grille d'Accueil : les uns passent par le menu Plus (selon le rôle), et `/photo` n'est relié à aucune navigation (voir §2.18).

### 2.4 Pointage (`PunchPage.tsx`) — le cœur du module

L'écran de Pointage affiche en haut une **horloge** et l'état **« En service » / « Hors service »**. Son contenu change selon que vous êtes pointé ou non.

**Quand vous n'êtes PAS pointé :**

- **Sélecteur « Bon de travail »** — la liste de vos bons de travail disponibles, chacun affiché sous la forme `numéro - description | projet`. Si la liste est vide : « Aucun bon de travail disponible. »
- **Détails du bon de travail sélectionné** — numéro, projet, et **adresse + ville du chantier** (avec une icône de repère).
- **Sélecteur « Opération (optionnel) »** — s'affiche si le bon de travail comporte des opérations, pour préciser la tâche.
- **Zone « Notes (optionnel) »** — pour noter les tâches prévues, les conditions, etc. (exemple proposé : « Tâches prévues, conditions météo... »).
- **Bouton vert « Pointer l'entrée »** — désactivé tant qu'aucun bon de travail n'est choisi.

**Quand vous ÊTES pointé :**

- une **carte verte** rappelle le bon de travail, l'opération et le projet en cours ;
- l'**heure d'entrée** (« Entrée à {heure} », dans le fuseau de l'entreprise) ;
- un **chronomètre en direct** (mis à jour chaque seconde) ;
- un **badge météo** capturé au moment de l'entrée, s'il est disponible ;
- une **zone de notes de sortie** ;
- deux boutons : **« Pointer la sortie »** (rouge) et **« Pointer la sortie + signature client »**.

**Géolocalisation (GPS) — automatique et invisible.** À l'entrée **comme** à la sortie, l'application capte votre position **en arrière-plan, silencieusement** (avec un délai maximal de 6 secondes). **Il n'y a aucun bouton GPS ni aucune carte** : vous ne voyez jamais vos coordonnées. Si le GPS est indisponible, le serveur utilise l'**adresse du chantier** du bon de travail comme position de repli.

**Météo — capturée par le serveur.** Au moment du pointage, le serveur enregistre un **instantané météo** (via Open-Meteo) rattaché à votre pointage — d'abord d'après votre position GPS, sinon d'après l'adresse du chantier. Cette météo s'affiche ensuite en **badge, en lecture seule** (vous ne la saisissez jamais). Cet instantané est **distinct** de l'écran Météo Chantier (§2.12), qui donne les **prévisions** à venir.

**Signature externe (superviseur sur place).** Le bouton **« Pointer la sortie + signature client »** ouvre une modale **« Signature du superviseur »** destinée à un **superviseur présent sur le chantier** (qui n'a pas de compte) :

- champ **« Nom du signataire »** (au moins 2 caractères) ;
- **canevas tactile** où le superviseur signe **au doigt ou au stylet** ;
- bouton **« Effacer »** pour recommencer ;
- un rappel du bon de travail et de la **durée pointée** ;
- boutons **« Annuler »** et **« Valider »**.

> **La signature de pointage est celle d'un superviseur externe, pas la vôtre.** Elle sert de preuve de présence validée sur place. Elle est distincte de l'**approbation par NIP** faite par un contremaître (§2.5).

**Historique intégré (« Mes heures »).** En bas de l'écran de Pointage, une section à **deux onglets** :

- **Onglet « Historique »** — vos pointages **groupés par date** (fuseau de l'entreprise). Chaque carte montre le bon de travail, le projet, l'opération, les heures d'entrée / sortie (ou « En cours ») et le total. Des **badges** signalent l'état : **« Facturé »** (verrouillé) ou **« Validé »** (avec « Validé par {nom} le {date} »). Tant qu'un pointage n'est **ni facturé ni validé**, vous pouvez le **Modifier** (les notes seulement) ou le **Supprimer** (avec confirmation).
- **Onglet « Résumé hebdo »** — navigation d'une **semaine à l'autre**, **total de la semaine** (carte orange s'il y a des heures supplémentaires), badge **« + {h} supp. (> 40 h) »**, et un **graphique à barres par jour** avec un seuil de 8 h, distinguant heures régulières et supplémentaires.

> **Règle des heures supplémentaires.** Le résumé considère comme supplémentaires les heures **au-delà de 8 h dans une journée** et **au-delà de 40 h dans une semaine** (constantes `OVERTIME_DAILY = 8` et `OVERTIME_WEEKLY = 40`).

### 2.5 Équipe sur chantier (`CrewPage.tsx`) — vue contremaître

Accessible par le menu **Plus → Équipe**, cet écran donne au contremaître la vue de **qui est présent**. Titre **« Équipe sur chantier »**, bouton **« Rafraîchir »**.

- La liste est **regroupée par projet** (accordéon). Chaque projet affiche un compteur **présents / assignés** et, le cas échéant, un badge **« Approbateur »**.
- En dépliant un projet, on voit ses membres : **pastille verte** (pointé) ou **grise** (absent), le poste, le **numéro de bon de travail** et un **chronomètre en direct** pour ceux qui sont pointés.
- Si vous êtes **approbateur**, un bouton **« Approuver »** valide les heures d'un employé ; sinon, un badge **« Validé »** indique l'état.

> **Approbation par NIP vs signature externe.** L'approbation faite ici par un contremaître (avec son compte) est **enregistrée à son nom**. C'est différent de la signature d'un superviseur externe prise au moment du pointage (§2.4), qui, elle, n'a pas de compte.

### 2.6 Messagerie (`MessagesPage.tsx`, `ChannelChatPage.tsx`, `DmChatPage.tsx`)

**Centre des messages.** Titre **« Messages »**, deux onglets : **Canaux** et **Messages directs**.

- **Onglet Canaux** — la liste des canaux (icône **cadenas** si privé, **#** sinon), avec nom, **badge de non-lus**, description et nombre de membres. Un bouton flottant **+** ouvre la modale **« Nouveau canal »** (champs **Nom du canal** obligatoire et **Description**). Vide : « Aucun canal disponible. »
- **Onglet Messages directs** — vos conversations privées (avatar, nom, aperçu du dernier message, heure relative, badge de non-lus). Le bouton **+** démarre une nouvelle conversation.

**Dans un canal (`ChannelChatPage.tsx`).** Le fil de discussion affiche chaque message (avatar coloré, nom, heure, mention **« (modifié) »** ou **« Message supprimé »**). Vous pouvez :

- **réagir** avec un émoji (👍 ❤️ ✅ 😀 🎉 🙏) ;
- **joindre des photos** (grille d'images) ;
- écrire et **envoyer** un message.

La barre de saisie offre des boutons **joindre une photo** / **prendre une photo**, un champ de texte et l'envoi. Les photos sont **compressées et débarrassées de leurs données EXIF** (dont la position GPS) **avant l'envoi**, conformément à la Loi 25. Les messages en cours d'envoi affichent un statut **« En attente »** ou **« Échec »**.

**En message direct (`DmChatPage.tsx`).** Un mode composition permet de **rechercher un employé** (« Rechercher un employé... »). Les bulles s'alignent (les vôtres à droite), avec **accusés de lecture** (✓ / ✓✓) et pièces jointes photo.

> **Ce que l'application mobile expose (et pas tout à fait).** Le serveur gère des **canaux privés**, l'**adhésion par membre** et des **fils de discussion** (réponses en fil). Mais l'interface mobile actuelle **ne propose pas**, à la création d'un canal, de choisir privé/public ni de sélectionner les membres (elle n'envoie que le nom et la description), et il **n'existe pas d'écran pour répondre en fil** dans un canal (seulement réactions et photos). Ces capacités existent en coulisse mais ne sont pas toutes exposées ici.

### 2.7 Dossiers et Fiche 360 (`DossiersPage.tsx`, `DossierDetailPage.tsx`)

**Liste des dossiers.** Titre **« Dossiers »**, une **recherche** (titre / numéro / projet) et des **puces de filtre par statut** (Tous + les statuts). Chaque carte montre le numéro, le titre, des badges **statut** et **priorité**, le projet, l'**échéance** et une **barre de progression des étapes** (« {faites}/{total} ({pourcentage} %) »).

**Fiche 360 (détail d'un dossier).** En-tête avec titre, numéro et badge de statut, puis **14 onglets** défilables :

| Onglet | Contenu |
|---|---|
| **Résumé** | Infos clés (projet, client, responsable, dates), progression, 4 indicateurs (Budget / Facturé / Payé / Solde dû), compteurs d'activité |
| **Devis** | Devis liés, avec statuts et montants |
| **Projets** | Projets rattachés |
| **BT** | Bons de travail |
| **Achats** | Bons d'achat |
| **Demandes** | Demandes de prix |
| **Factures** | Factures liées |
| **Pointage** | Entrées de temps du dossier (entrée / sortie / heures, badge « Validé ») |
| **Compta** | Revenus / facturation, coûts et marge (%) |
| **Étapes** | Étapes triées ; on change le statut (À faire / En cours / Terminé) par menu déroulant |
| **Notes** | Notes avec photos ; ajout de note, enrichissement et analyse par IA, résumé IA |
| **Docs** | Pièces jointes (téléverser, aperçu, télécharger, renommer, supprimer) |
| **Liens** | Liens Web (URL + description) |
| **Extras** | Avenants (extras) ; création et assistant IA |

**Volet financier réservé.** Pour un **EMPLOYE**, les données financières du dossier (devis, factures, bons de commande, marge) sont **masquées** : il voit l'opérationnel (étapes, notes, pointage) mais pas les chiffres.

**Onglet Notes en détail.** Un bouton **« Ajouter une note »** ouvre une modale où l'on saisit le texte et où l'on peut **« Enrichir avec IA »**, **« Analyser photo IA »** et **ajouter des photos** (compressées, EXIF retiré). Un bouton **« Résumé IA »** produit un résumé du dossier (points ouverts et actions en attente).

**Onglet Extras (avenants).** Un bouton **« Assistant IA »** permet de décrire un extra en langage naturel : l'IA **propose**, vous **confirmez**. Un formulaire **« Ajouter un extra »** demande une **Description** et un **Montant avant taxes**. Chaque extra porte un badge (Proposé / Approuvé / Refusé / Facturé). Une note rappelle : **« La facturation se fait au bureau (web). »**

> **Les extras se créent au chantier, mais se facturent au bureau.** L'application mobile permet de **saisir** et de faire **approuver** un extra, mais **la transformation en facture se fait dans l'ERP Web** (ou par un ADMIN/MANAGER via l'action de facturation dédiée). C'est voulu : la facturation reste centralisée.

### 2.8 Documents commerciaux (`DocumentsHubPage.tsx` et suivants)

L'application embarque un véritable **gestionnaire de documents commerciaux**, avec quatre types : **Devis, Factures, Bons de travail, Bons de commande**.

**Centre des documents.** Pour chaque type, une carte de statistiques (Total / Brouillon / En cours / Terminé). **Restriction par rôle importante :**

- un **EMPLOYE** ne voit que **« Bons de travail »** ;
- les **Devis, Factures et Bons de commande** sont réservés aux **ADMIN et MANAGER**.

Un bouton flottant **« Scanner un reçu »** (managers seulement) ouvre le scanner de reçu : vous photographiez un reçu, l'**IA en extrait** le fournisseur, les lignes, les taxes et le total, et pré-remplit un **bon de commande** à valider.

**Liste d'un type (`DocumentListPage.tsx`).** Recherche, bouton **« Ajouter un {type} »**, **« Exporter en CSV »**, cartes avec statut et client, et un menu contextuel **Modifier / Dupliquer / Supprimer**. Des indicateurs signalent les éléments créés hors-ligne.

**Détail d'un document (`DocumentDetailPage.tsx`).** Une barre d'actions réunit :

- **« Télécharger PDF »** ;
- **« Envoyer par courriel »** (destinataire, copie conforme, sujet, message) ;
- **« Lien de paiement Stripe »** (factures) — génère un lien de paiement par carte ;
- **« Dupliquer »** ;
- **« Rendre récurrent »** (factures) — fréquence hebdomadaire, mensuelle, trimestrielle ou annuelle ;
- **« Faire signer »** (devis et factures) — signature électronique du client.

Les **lignes** du document s'ajoutent, se modifient et se suppriment (Description / Quantité / Unité / Prix), avec sous-total, **TPS 5 %**, **TVQ 9,975 %** et total.

**Formulaire (`DocumentFormPage.tsx`).** À la création / modification : Nom du projet, Nom (obligatoire), Client ou Fournisseur, Projet, Statut, Priorité, dates d'échéance / livraison, Description, Notes. L'**édition hors-ligne est désactivée** (un message le signale).

> **Numérotation professionnelle.** Chaque document reçoit un numéro séquentiel de la forme `PRÉFIXE-AAAA-NNN` (par exemple `DEV-2026-001`), généré de façon sûre côté serveur (pas de doublon même en cas d'usage simultané).

### 2.9 Scanner de code-barres / POS (`ScannerPage.tsx`)

Le Scanner transforme le téléphone en **terminal d'inventaire**. Il utilise la **caméra** (formats EAN, UPC, Code128, QR), avec repli en **saisie manuelle** et **lampe torche**.

Un **sélecteur de mode** (visible si vous gérez le stock) propose :

- **Consultation** — scannez un produit pour voir sa **fiche** (nom, codes, **stock**, emplacement, catégorie, marque, prix). Si plusieurs produits correspondent, une désambiguïsation s'affiche. Un ADMIN/MANAGER peut **« Associer un code-barres »** à un produit non reconnu. Un panneau **« Mouvement de stock »** permet un ajustement unitaire (Entrée / Sortie / Ajustement + quantité + note).
- **Entrée / Sortie / Ajustement** — mode **panier** : scannez en continu, les articles s'empilent, les quantités s'ajustent (±), on ajoute une note, puis on **« Valide »** le tout d'un coup.

Le panier est **résistant au hors-ligne** : les validations sont mises en file avec une **clé d'unicité** qui garantit qu'une même validation n'est **jamais comptée deux fois**, même en cas de nouvelle tentative (voir §4.10). Le panier survit à un rafraîchissement de la page.

> **Les trois types de mouvement.** **Entrée** ajoute au stock, **Sortie** retire (refusée si le stock est insuffisant), **Ajustement** fixe la quantité à une **valeur absolue** (inventaire physique). Ces mouvements s'écrivent dans **la même table que l'ERP** (`mouvements_stock`).

### 2.10 Réception et Comptage (`ReceptionPage.tsx`, `CountingPage.tsx`)

Deux écrans d'inventaire, réservés à ceux qui **gèrent le stock**, et qui **exigent une connexion** (pas de mode hors-ligne).

**Réception.** La liste des **bons de commande** à réceptionner (Envoyé / Confirmé / En cours). En ouvrant un bon, on voit ses lignes ; **scanner un article coche automatiquement** la ligne correspondante (une aide visuelle, pas un verrou). Le bouton **« Confirmer la réception »** enregistre la **réception complète** et met à jour le stock.

**Comptage.** Des **sessions de comptage cyclique** : on scanne les articles, on saisit la **quantité comptée** face à la quantité théorique, l'application calcule les **écarts**, puis on **« Clôture »** (les ajustements sont appliqués au stock) ou on **« Annule »** la session.

### 2.11 Assistant IA (`AiAssistantPage.tsx`)

Un assistant conversationnel accessible par le menu **Plus → Assistant IA**. À gauche, la liste de vos **conversations** (créer, supprimer) ; à droite, le fil de discussion.

- **Saisie** — texte, **pièces jointes** (caméra ou fichier, jusqu'à 5, compressées), **dictée vocale** (reconnaissance en français canadien) et **lecture vocale** des réponses.
- **Actions en attente** — lorsque l'IA propose une **écriture** dans la base (créer, modifier, supprimer une donnée), elle présente une carte avec **« Confirmer »** ou **« Annuler »**, et un statut (Exécuté / Annulé / Expiré / Rejeté / Échec). **Rien n'est écrit sans votre confirmation.**

**Différence majeure selon le rôle.** Pour un **ADMIN ou MANAGER**, l'assistant dispose d'**outils ERP** : il peut lister et créer des factures, devis et bons, enregistrer un paiement, faire une recherche dans la base, calculer des taxes, proposer une écriture, etc. Pour un **EMPLOYE**, l'assistant reste **général** : il répond aux questions mais **n'a aucun accès aux données financières ni aux écritures ERP**.

> **Facturation de l'assistant.** L'assistant IA puise dans les **crédits IA prépayés de votre entreprise** (les mêmes que l'ERP). Chaque échange consomme des crédits selon le **coût réel** des jetons du modèle (Claude Sonnet). Si le solde de crédits de l'entreprise est épuisé, l'assistant est temporairement indisponible (voir §4.5).

### 2.12 Météo Chantier (`MeteoPage.tsx`)

Accessible par **Plus → Météo Chantier**. Un **sélecteur de station** (7 stations du Québec), les **prévisions sur 7 jours** (une carte « Aujourd'hui » plus 6 jours : Max / Min / Pluie / Vent) et des **alertes colorées** (Gel, Vent fort, Pluie).

Une section **« Impact chantier »** traduit la météo en **recommandations concrètes** — par exemple, en cas de grand vent : **« ARRÊTER les travaux en hauteur »**. Les seuils d'alerte : gel sous 0 / sous −10 °C, pluie au-delà de 10 / 20 mm, vent au-delà de 50 / 70 km/h.

> **Deux « météo » à ne pas confondre.** Cet écran donne les **prévisions** (jours à venir). Le **badge météo du pointage** (§2.4), lui, est un **instantané passé** enregistré au moment précis où vous avez pointé.

### 2.13 Calculatrice de construction (`CalculatricePage.tsx`)

Une **calculatrice de chantier** inspirée de la Construction Master Pro, avec un écran type afficheur, deux onglets **Charpente** et **Conversions**, et le support du **clavier physique**.

- **Saisie en pieds-pouces-fractions** (avec préréglages) et **conversions d'unités** (m, cm, mm, pi, po, verges, kg, lb…).
- **Fonctions charpente** : pente, montée, portée, diagonale, arêtier ; onglet, escalier, arc, cercle ; longueur, largeur, hauteur ; pieds-planches, montants, etc.
- **Panneau de résultats détaillés** (escalier : contremarches, marches, limon, formule de Blondel ; cercle ; arc ; polygone…).
- **Historique** des calculs.

### 2.14 Relances d'impayés (`RemindersPage.tsx`) — ADMIN / MANAGER

Accessible par **Plus → Relances factures** (visible seulement pour ADMIN et MANAGER). L'écran classe les **factures en retard** par **tranches d'ancienneté** (J+30, J+60, J+90, plus de 90 jours).

Un bouton **« Envoyer relances »** ouvre une modale : sélection des tranches, **mode simulation** (essai à blanc, sans envoyer), **courriel de test**. À la confirmation, des **courriels de relance** (adaptés à l'ancienneté, du plus courtois au dernier avis) sont envoyés, PDF joint.

### 2.15 Journal d'audit (`AuditLogPage.tsx`) — ADMIN seulement

Accessible par **Plus → Journal d'audit** (ADMIN uniquement, conformité Loi 25). Il liste les **événements** enregistrés, avec des **filtres** :

- **Entité** : Factures, Devis, Bons de travail, Bons de commande, Authentification, Pièces jointes ;
- **Action** : Création, Modification, Suppression, Connexion, Signature, Envoi de courriel, Paiement reçu ;
- **Dates**.

Chaque événement conserve un **contexte** (adresse IP, appareil, valeurs « Avant / Après »), avec pagination.

### 2.16 Apparence (`AppearancePage.tsx`)

Accessible par **Plus → Apparence**. Personnalise les **couleurs de l'interface** de l'application (réglage **local à l'appareil**) :

- **thèmes prédéfinis** : Constructo, Forêt, Brique, Anthracite, Bourgogne, Océan ;
- **couleurs sur mesure** : couleur principale, survol, barre supérieure ;
- **aperçu** en direct et **avertissement de contraste** ;
- boutons **Enregistrer / Annuler / Réinitialiser**.

### 2.17 Notifications push (la cloche)

La **cloche** de la barre supérieure active ou désactive les **notifications push** (nouveau message, etc.). L'activation est **volontaire** : un appui sur la cloche demande l'autorisation du navigateur, puis abonne l'appareil. L'icône reflète l'état (cloche pleine = activée, barrée = désactivée).

> **Limites à connaître.** Les notifications sont **masquées** sur les navigateurs qui ne les prennent pas en charge (notamment **iOS avant la version 16.4**). Et, en raison du déménagement de l'application sous `/mobile` (§1.2), les employés ayant activé les notifications à l'ancienne adresse doivent les **réactiver** ici.

### 2.18 Écran « Téléversement photo » (`/photo`) — non relié

L'application contient un écran **« Téléversement photo »** (prendre ou choisir une photo, puis la téléverser) et son **stockage serveur** fonctionne. **Mais aucun bouton ni menu ne mène à cet écran** dans l'interface actuelle : il est absent de la grille d'Accueil, du menu Plus et de la barre du bas.

**En pratique**, pour joindre une photo au chantier, on passe donc par les **notes de dossier** (§2.7) ou par la **messagerie** (§2.6), qui, elles, sont pleinement accessibles. Cet écran de téléversement autonome est à considérer comme **non disponible** tant qu'aucun lien n'y mène.

---

## 3. Workflows pas à pas

### 3.1 Se connecter la première fois

1. Ouvrez **`app.constructoai.ca/mobile`** dans le navigateur du téléphone.
2. *(Recommandé)* Ajoutez l'application à l'écran d'accueil (§1.3).
3. **Étape entreprise** : saisissez le **courriel** et le **mot de passe** de l'entreprise, puis **« Continuer »**.
4. **Étape « Qui êtes-vous ? »** : touchez **votre nom** dans la liste.
5. **Étape NIP** : saisissez votre **code à 4 chiffres** — la connexion se fait au quatrième chiffre.
6. Vous arrivez sur l'**Accueil**. Votre session dure 24 heures.

> **NIP oublié ?** Demandez à votre administrateur de le réinitialiser. Après plusieurs essais erronés, patientez quelques minutes avant de réessayer.

### 3.2 Pointer son entrée et sa sortie

1. Onglet **Pointage**.
2. **Choisissez le bon de travail** du jour (et l'**opération** si demandée).
3. *(Facultatif)* Ajoutez une **note**.
4. Touchez **« Pointer l'entrée »**. Votre position et la météo sont enregistrées **automatiquement** ; le **chronomètre** démarre.
5. En fin de quart, revenez au Pointage, ajoutez au besoin une **note de sortie**, puis **« Pointer la sortie »**.

> **Sans réseau ?** Pointez quand même : l'heure de votre geste est enregistrée dans le téléphone et remontera dès la reconnexion. Le pointage est **plafonné à 16 heures** : si vous oubliez de pointer la sortie, le système borne automatiquement la durée (il ne comptera pas 24 h ou plus).

### 3.3 Pointer avec la signature d'un superviseur

1. Étant pointé, touchez **« Pointer la sortie + signature client »**.
2. Passez le téléphone au **superviseur présent sur place**.
3. Il saisit son **nom** (au moins 2 caractères) et **signe au doigt** dans le canevas (bouton **« Effacer »** pour recommencer).
4. Il touche **« Valider »**. La sortie est enregistrée **avec la signature** comme preuve de présence.

### 3.4 Corriger ou supprimer un pointage

1. Onglet **Pointage** → section **« Mes heures »** → onglet **Historique**.
2. Repérez le pointage. S'il n'est **ni facturé ni validé**, deux actions sont possibles :
   - **Modifier** — ajuste **les notes** (les heures ne sont pas modifiables ici) ;
   - **Supprimer** — avec confirmation.
3. Un pointage **« Facturé »** ou **« Validé »** est **verrouillé** : adressez-vous à un responsable au bureau.

### 3.5 Consulter ses heures de la semaine

1. Onglet **Pointage** → **« Mes heures »** → onglet **Résumé hebdo**.
2. Naviguez d'une **semaine** à l'autre.
3. Lisez le **total**, le badge d'heures **supplémentaires** (au-delà de 40 h) et le **graphique par jour** (seuil de 8 h).

### 3.6 Approuver les heures de son équipe (contremaître)

1. Menu **Plus → Équipe**.
2. Dépliez le **projet** voulu ; repérez les membres **pointés** (pastille verte).
3. Si vous êtes **approbateur**, touchez **« Approuver »** pour valider les heures d'un employé. Le pointage passe à **« Validé »** (à votre nom).

### 3.7 Écrire dans un canal ou en message direct

1. Onglet **Messages**.
2. **Canal** : onglet **Canaux** → ouvrez un canal → écrivez, **réagissez** avec un émoji ou **joignez une photo** → envoyez.
3. **Message direct** : onglet **Messages directs** → **+** → **cherchez l'employé** → écrivez et envoyez.
4. Pour **créer un canal** : onglet Canaux → **+** → saisissez un **Nom** (et une **Description**) → **Créer**.

> **Photos et vie privée.** Les photos envoyées sont **compressées** et **débarrassées de leurs données EXIF** (dont la localisation) avant l'envoi.

### 3.8 Consulter un dossier et ajouter une note (avec IA)

1. Onglet **Dossiers** → recherchez / filtrez → ouvrez un dossier (**Fiche 360**).
2. Parcourez les **14 onglets** (Résumé, Étapes, Pointage, etc.).
3. Onglet **Notes** → **« Ajouter une note »** → saisissez le texte.
4. *(Facultatif)* **« Enrichir avec IA »**, **« Analyser photo IA »**, ou ajoutez des **photos**.
5. Enregistrez. Le bouton **« Résumé IA »** génère à tout moment un **résumé** du dossier.

### 3.9 Ajouter un extra (avenant) à un dossier

1. Fiche 360 → onglet **Extras**.
2. **« Ajouter un extra »** → **Description** + **Montant avant taxes**, ou utilisez **« Assistant IA »** (décrivez, l'IA propose, vous confirmez).
3. L'extra apparaît avec un badge (Proposé / Approuvé…).
4. **La facturation se fait au bureau (web)** — l'application mobile crée l'extra mais ne le facture pas.

### 3.10 Créer un bon de travail (tous) ou un devis / facture (ADMIN / MANAGER)

1. Accueil → tuile **Documents** (ou menu contextuel selon l'écran).
2. Choisissez le **type** — un **EMPLOYE** ne voit que **Bons de travail** ; Devis / Factures / Bons de commande demandent le rôle ADMIN ou MANAGER.
3. **« Ajouter un {type} »** → remplissez le formulaire (Nom obligatoire, client, projet, statut…).
4. Ouvrez le document → **ajoutez des lignes** (Description / Quantité / Unité / Prix) ; les taxes (TPS 5 %, TVQ 9,975 %) se calculent automatiquement.

### 3.11 Envoyer un document par courriel ou générer un lien de paiement

1. Ouvrez le **document** (§3.10).
2. **« Télécharger PDF »** pour l'obtenir, ou **« Envoyer par courriel »** (destinataire, copie, sujet, message — le PDF est joint).
3. Pour une **facture** : **« Lien de paiement Stripe »** crée un lien de paiement par carte à transmettre au client. Une fois payé, la facture passe automatiquement à **« Payée »**.

### 3.12 Scanner un produit et enregistrer un mouvement de stock

1. Accueil → tuile **Scanner** (si vous gérez le stock).
2. **Mode Consultation** : scannez pour voir la **fiche** du produit ; au besoin, faites un **mouvement unitaire** (Entrée / Sortie / Ajustement + quantité + note).
3. **Mode Entrée / Sortie / Ajustement** (panier) : scannez plusieurs articles, ajustez les quantités, puis **« Valider »** le panier.

> **Sans réseau ?** Le panier est mis en file avec une **clé d'unicité** : à la reconnexion, il se valide **une seule fois**, jamais en double.

### 3.13 Réceptionner un bon de commande

1. Accueil → tuile **Réception** (connexion requise).
2. Ouvrez le **bon de commande** à recevoir.
3. **Scannez** les articles reçus (les lignes se **cochent**).
4. **« Confirmer la réception »** — le stock est mis à jour.

### 3.14 Faire un comptage d'inventaire

1. Accueil → tuile **Comptage** (connexion requise).
2. Démarrez une **session** ; scannez les articles et saisissez la **quantité comptée**.
3. Vérifiez les **écarts**, puis **« Clôturer »** (applique les ajustements) ou **« Annuler »**.

### 3.15 Poser une question à l'assistant IA

1. Menu **Plus → Assistant IA**.
2. Écrivez (ou **dictez**) votre question ; joignez une photo au besoin.
3. Si l'IA propose une **écriture** (ADMIN / MANAGER), une carte apparaît : **« Confirmer »** ou **« Annuler »**. Rien n'est écrit sans votre accord.

### 3.16 Activer les notifications et travailler hors-ligne

1. **Notifications** : touchez la **cloche** (barre du haut) et acceptez la demande du navigateur. *(Indisponible sur iOS avant 16.4.)*
2. **Hors-ligne** : agissez normalement ; les actions se mettent en file (indicateur « N actions en attente »).
3. À la reconnexion, tout se **synchronise**. Vous pouvez forcer avec **« Synchroniser maintenant »**.

---

## 4. Référence

### 4.1 Écrans et routes (24)

Toutes les routes sont servies sous **`app.constructoai.ca/mobile`**.

| Écran | Route | Fichier | Accès |
|---|---|---|---|
| Connexion | `/login` | `LoginPage.tsx` | Public |
| Accueil / Tableau de bord | `/` | `DashboardPage.tsx` | Connecté |
| Pointage | `/pointage` | `PunchPage.tsx` | Connecté |
| Équipe sur chantier | `/equipe` | `CrewPage.tsx` | Connecté |
| Messages (centre) | `/messages` | `MessagesPage.tsx` | Connecté |
| Canal | `/messages/channel/:id` | `ChannelChatPage.tsx` | Connecté |
| Message direct | `/messages/dm/:id` | `DmChatPage.tsx` | Connecté |
| Dossiers (liste) | `/dossiers` | `DossiersPage.tsx` | Connecté |
| Fiche 360 | `/dossiers/:id` | `DossierDetailPage.tsx` | Connecté |
| Documents (centre) | `/documents` | `DocumentsHubPage.tsx` | Bons de travail = tous ; reste = ADMIN/MANAGER |
| Documents (liste) | `/documents/:docType` | `DocumentListPage.tsx` | Idem |
| Nouveau document | `/documents/:docType/nouveau` | `DocumentFormPage.tsx` | Idem |
| Détail document | `/documents/:docType/:docId` | `DocumentDetailPage.tsx` | Idem |
| Modifier document | `/documents/:docType/:docId/modifier` | `DocumentFormPage.tsx` | Idem |
| Assistant IA | `/assistant` | `AiAssistantPage.tsx` | Connecté (outils ERP = ADMIN/MANAGER) |
| Météo Chantier | `/meteo` | `MeteoPage.tsx` | Connecté |
| Téléversement photo | `/photo` | `PhotoUploadPage.tsx` | **Non relié** (voir §2.18) |
| Scanner | `/scanner` | `ScannerPage.tsx` | Gestion de stock |
| Réception | `/reception` | `ReceptionPage.tsx` | Gestion de stock (en ligne) |
| Comptage | `/comptage` | `CountingPage.tsx` | Gestion de stock (en ligne) |
| Calculatrice | `/calculatrice` | `CalculatricePage.tsx` | Connecté |
| Relances factures | `/reminders` | `RemindersPage.tsx` | ADMIN / MANAGER |
| Journal d'audit | `/audit` | `AuditLogPage.tsx` | ADMIN |
| Apparence | `/apparence` | `AppearancePage.tsx` | Connecté |

*(Toute autre adresse renvoie à l'Accueil.)*

### 4.2 Rôles et permissions

| Fonction | ADMIN | MANAGER | EMPLOYE | APPRENTI |
|---|:---:|:---:|:---:|:---:|
| Pointer, consulter ses heures | ✅ | ✅ | ✅ | ✅ |
| Approuver les heures de l'équipe | ✅ | ✅ | selon approbateur | selon approbateur |
| Messagerie (canaux, messages directs) | ✅ | ✅ | ✅ | ✅ |
| Dossiers — volet opérationnel | ✅ | ✅ | ✅ | ✅ |
| Dossiers — volet financier | ✅ | ✅ | ❌ | ❌ |
| Documents — **Bons de travail** | ✅ | ✅ | ✅ | ✅ |
| Documents — **Devis / Factures / Bons de commande** | ✅ | ✅ | ❌ | ❌ |
| Scanner un reçu (→ bon de commande) | ✅ | ✅ | ❌ | ❌ |
| Inventaire (scanner, réception, comptage) | ✅ | ✅ | selon permission stock | selon permission stock |
| Assistant IA — **général** | ✅ | ✅ | ✅ | ✅ |
| Assistant IA — **outils ERP (argent)** | ✅ | ✅ | ❌ | ❌ |
| Relances d'impayés | ✅ | ✅ | ❌ | ❌ |
| Factures récurrentes — déclencher | ✅ | ❌ | ❌ | ❌ |
| Journal d'audit | ✅ | ❌ | ❌ | ❌ |

> La permission **« peut gérer le stock »** peut être accordée individuellement à un EMPLOYE / APPRENTI pour débloquer l'inventaire, sans lui donner le rôle MANAGER.

### 4.3 Points d'accès de l'API (principaux)

Tous préfixés par **`/api/mobile/v1`** (et `/api/mobile/v1/attachments` pour les pièces jointes). L'application compte **118 points d'accès** dans `mobile_api.py` et **7** dans `attachments_api.py`. Voici les principaux, par domaine.

**Connexion et session**

| Méthode | Chemin | Rôle |
|---|---|---|
| POST | `/auth/tenant` | Étape 1 : courriel + mot de passe d'entreprise → liste des employés |
| POST | `/auth/pin` | Étape 2 : employé + NIP → jeton de session (24 h) |
| GET | `/me` | Rafraîchit rôle, fuseau et permission de stock |
| POST | `/auth/signed-url` | Génère une URL signée temporaire (images, pièces jointes) |

**Pointage**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/punch/status` | Pointage actif et durée écoulée |
| POST | `/punch/in` | Pointer l'entrée (bon de travail, opération, notes, GPS) |
| POST | `/punch/out` | Pointer la sortie |
| GET | `/history` | Historique des pointages |
| GET | `/weekly-summary` | Résumé hebdomadaire (heures supp.) |
| PUT / DELETE | `/time-entries/{id}` | Modifier les notes / supprimer (si non validé) |
| GET | `/crew` | Vue équipe sur chantier |
| POST | `/punch/{id}/approve` | Approuver un pointage |
| POST | `/punch/{id}/signature-externe` | Signature d'un superviseur externe |
| GET | `/work-orders` | Bons de travail disponibles |

**Messagerie**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET / POST | `/channels` | Lister / créer un canal |
| GET / POST | `/channels/{id}/messages` | Lire / écrire dans un canal |
| POST | `/messages/{id}/reactions` | Réagir (émoji) |
| GET / POST | `/dm/...` | Messages directs (conversations, envoi) |
| GET | `/messaging/unread` | Compteur de non-lus |

**Dossiers et extras**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/dossiers`, `/dossiers/{id}` | Liste / Fiche 360 |
| POST | `/dossiers/{id}/notes` | Ajouter une note (texte + photos) |
| PATCH | `/dossiers/{id}/etapes/{id}` | Changer le statut d'une étape |
| GET/POST/PUT/DELETE | `/dossiers/{id}/liens` | Liens Web |
| GET/POST/PUT/DELETE | `/dossiers/{id}/extras` | Extras (avenants) |
| POST | `/dossiers/{id}/extras/{id}/facturer` | Facturer un extra (ADMIN/MANAGER) |

**Documents**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/documents/{type}`, `/documents/{type}/{id}` | Liste / détail |
| POST | `/documents/{type}` | Créer |
| POST | `/documents/{type}/{id}/pdf` | Générer le PDF |
| POST | `/documents/{type}/{id}/email` | Envoyer par courriel |
| GET | `/documents/{type}/export.csv` | Export CSV |
| GET/POST | `/documents/{type}/{id}/signature` | Signature électronique (devis / factures) |
| POST | `/documents/factures/{id}/payment-link` | Lien de paiement Stripe |
| GET | `/factures/overdue` | Factures en retard (ADMIN/MANAGER) |
| POST | `/factures/send-reminders` | Envoyer des relances |
| POST | `/factures/{id}/recurrent` | Rendre une facture récurrente |

**Inventaire**

| Méthode | Chemin | Rôle |
|---|---|---|
| GET | `/products/scan`, `/products/search` | Chercher un produit |
| POST | `/stock-movements`, `/stock-movements/bulk` | Mouvement unitaire / panier (idempotent) |
| GET / POST | `/purchase-orders`, `/purchase-orders/{id}/receive` | Réception |
| POST | `/inventory-counts` (+ `/lines`, `/close`, `/cancel`) | Comptage |

**Assistant IA, météo, notifications, audit**

| Méthode | Chemin | Rôle |
|---|---|---|
| POST | `/ai/chat` | Un échange avec l'assistant |
| POST | `/ai/pending-actions/{id}/confirm` / `/cancel` | Confirmer / annuler une écriture proposée |
| GET | `/ai/quota` | Solde de crédits IA |
| GET | `/weather/stations`, `/weather/forecast` | Météo |
| GET/POST | `/push/vapid-public-key`, `/push/subscribe`, `/push/unsubscribe` | Notifications |
| GET | `/audit/events` | Journal d'audit (ADMIN) |
| GET | `/health` | État du service |

### 4.4 Pointage — calculs et bornes

| Élément | Règle |
|---|---|
| Un seul pointage ouvert à la fois | Un verrou serveur interdit deux entrées simultanées (protège la paie) |
| Heure enregistrée | L'heure du **geste** (clic), même hors-ligne, dans le fuseau de l'entreprise |
| Tolérance d'horloge (hors-ligne) | Jusqu'à **15 min dans le futur** / **7 jours dans le passé** ; au-delà, l'heure du serveur est utilisée |
| Durée maximale d'un pointage | **16 heures** (plafond automatique en cas d'oubli de sortie) |
| Taux horaire | Salaire annuel de l'employé ÷ 2080 heures |
| Heures supplémentaires — jour | Au-delà de **8 h** |
| Heures supplémentaires — semaine | Au-delà de **40 h** |
| GPS | Capté automatiquement à l'entrée **et** à la sortie (délai max 6 s) ; **invisible** ; repli = adresse du chantier |
| Météo | Instantané enregistré par le serveur au pointage (lecture seule) |

### 4.5 Facturation de l'assistant IA

| Élément | Valeur |
|---|---|
| Source des crédits | Crédits IA **prépayés de l'entreprise** (`ai_prepaid_credits`), partagés avec l'ERP |
| Modèle IA (texte + outils) | Claude Sonnet — **3 $ / million** de jetons en entrée, **15 $ / million** en sortie |
| Modèle IA (analyse d'image) | Claude Opus (vision) |
| Ce qui est débité | Le **coût réel** des jetons de chaque échange |
| Majoration | **Aucune** (le coût brut est débité, sans marge supplémentaire) |
| Solde épuisé | L'assistant devient indisponible (erreur de quota) jusqu'à recharge |
| Confirmation des écritures | Toute action d'écriture (créer / modifier / supprimer) exige une **confirmation** ; elle **expire** après 30 minutes |

> **Note de politique de prix.** Contrairement à d'autres modules IA de l'écosystème (qui appliquent une majoration), l'assistant mobile débite le **coût brut** des jetons. C'est un point de configuration susceptible d'évoluer ; référez-vous au solde affiché par **`/ai/quota`**.

### 4.6 Taxes et numérotation des documents

| Élément | Valeur |
|---|---|
| TPS | **5 %** |
| TVQ | **9,975 %** |
| Numérotation | `PRÉFIXE-AAAA-NNN` (ex. `DEV-2026-001`), séquentielle et sûre |
| Types de document | devis · factures · bons-travail · bons-commande |
| Export CSV | UTF-8 avec séparateur `;` (compatible Excel Québec), jusqu'à 5000 lignes |
| PDF | Format Lettre, avec l'image de marque de l'entreprise |
| Lien de paiement | Stripe, en **CAD**, montant toutes taxes comprises |

### 4.7 Limites et bornes (fichiers, tailles)

| Élément | Limite |
|---|---|
| Durée de session | **24 heures** |
| NIP | **4 chiffres** ; 6 essais erronés max par employé (5 min) |
| Photo de pointage (`/photo`) | **5 Mo**, JPEG / PNG / GIF / WebP |
| Photos par note de dossier | **10 photos**, 5 Mo chacune |
| Photos par message | Compression + retrait EXIF avant envoi |
| Pièces jointes de documents | **10 Mo**, JPEG / PNG / WebP / HEIC / PDF / DOCX / XLSX |
| Pièces jointes IA | 5 max ; ~5 Mo par image, ~32 Mo par PDF |
| OCR de reçu | **10 Mo** (HEIC accepté) |
| Assistant IA — jetons max | 32 000 par réponse |
| Fichiers 3D dans un rendu | *(non concerné : voir module 27)* |

### 4.8 Statuts

| Domaine | Valeurs |
|---|---|
| **Pointage** | actif (en cours) · terminé · **validé** (approuvé) · **facturé** (verrouillé) |
| **Étape de dossier** | À faire · En cours · Terminé |
| **Extra (avenant)** | Proposé · Approuvé · Refusé · Facturé |
| **Document** | Brouillon · En cours · Terminé (+ Payée pour une facture) |
| **Bon de commande (réception)** | Envoyé · Confirmé · En cours · Reçu |
| **Session de comptage** | En cours · Clôturée · Annulée |
| **Action IA en attente** | En attente · Exécuté · Annulé · Expiré · Rejeté · Échec |

### 4.9 Sécurité et confidentialité

- **Connexion propre** — jeton distinct de l'ERP, valable 24 heures ; le NIP est vérifié côté serveur, protégé contre les tentatives répétées.
- **Permissions vérifiées côté serveur** — l'interface cache, mais le serveur **refuse** toute action non autorisée (erreur 403). Un employé ne peut jamais voir un devis / une facture ni approuver ses propres heures.
- **Isolation par entreprise** — chaque entreprise ne voit que ses propres données (les photos et pièces jointes d'une autre entreprise sont inaccessibles).
- **Loi 25** — les photos sont **débarrassées de leurs données EXIF** (dont la position) avant envoi ; un **journal d'audit** (ADMIN) trace les actions sensibles (création, modification, suppression, connexion, signature, envoi de courriel, paiement reçu).
- **Limites anti-abus** — les appels à l'IA et à l'OCR sont plafonnés (par exemple 20 échanges d'assistant IA par minute).

### 4.10 Mode hors-ligne — les 18 actions différables

Les actions suivantes peuvent être **mises en file** hors-ligne, puis **rejouées** à la reconnexion :

1. Pointer l'entrée · 2. Pointer la sortie · 3. Approuver un pointage · 4. Modifier un pointage · 5. Supprimer un pointage · 6. Créer un document · 7. Modifier un document · 8. Supprimer un document · 9. Créer une ligne de document · 10. Modifier une ligne · 11. Supprimer une ligne · 12. Ajouter une note de projet · 13. Téléverser une photo · 14. Téléverser une pièce jointe · 15. Envoyer un message · 16. Marquer un message lu · 17. Mettre à jour une étape de dossier · 18. Valider un panier de stock.

**Idempotence (pas de doublon).**

- Le **panier de stock** dispose d'une **clé d'unicité** réelle : même rejoué plusieurs fois, il n'est compté **qu'une fois**.
- Un mécanisme d'idempotence **général** (en-tête `Idempotency-Key`) existe dans le code (`mobile_idempotency.py`) **mais n'est pas activé** en production à ce jour. Autrement dit, seule la validation de panier de stock bénéficie aujourd'hui d'une protection anti-doublon garantie côté serveur.

> **Ce qui n'est PAS disponible hors-ligne** : la **réception** et le **comptage** d'inventaire (connexion requise), et l'**édition d'un document existant** (désactivée pour éviter d'écraser des données modifiées au bureau).

### 4.11 Pièges et éléments à connaître

- **L'écran « Téléversement photo » (`/photo`) n'est relié à rien** : passez par les **notes de dossier** ou la **messagerie** pour joindre des photos (§2.18).
- **Aucun bouton GPS, aucune carte** : la position est captée **silencieusement** au pointage ; vous ne la voyez jamais.
- **La météo du pointage est un instantané passé** (lecture seule), à ne pas confondre avec les **prévisions** de l'écran Météo Chantier.
- **La signature de pointage est celle d'un superviseur externe**, pas la vôtre.
- **Un EMPLOYE ne voit que les Bons de travail** dans Documents ; Devis / Factures / Bons de commande + scanner de reçu = ADMIN/MANAGER.
- **Les extras se créent au chantier mais se facturent au bureau (web).**
- **Création de canal simplifiée** : pas de choix privé/public ni de sélection de membres à la création ; **pas de réponse en fil** (threads) dans l'interface (réactions et photos seulement).
- **Réception / Comptage exigent une connexion** ; pas de mode hors-ligne.
- **Notifications** indisponibles sur iOS avant 16.4, et à **réactiver** après le déménagement sous `/mobile`.
- **Factures récurrentes** : il **n'y a pas de planification automatique** ; leur génération est **déclenchée manuellement** (par un ADMIN).
- **Idempotence générale non active** : hors le panier de stock, évitez de renvoyer sciemment deux fois la même action.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec le reste de Constructo AI

- **ERP Web (base partagée).** L'application mobile **partage la base de données** de l'ERP. Le pointage écrit dans **`time_entries`**, les mouvements de stock dans **`mouvements_stock`** — **les mêmes tables** que l'ERP. Ce que vous saisissez au chantier apparaît au bureau, et inversement, en temps réel.
- **Module Pointage (13) et Bons de Travail (12).** Le pointage mobile alimente le module **Pointage** de l'ERP ; les bons de travail que vous pointez proviennent du module **Bons de Travail**.
- **Module Dossiers (07).** Les Fiches 360 mobiles sont la version terrain des **dossiers** de l'ERP (notes, étapes, extras, documents).
- **Module Magasin (10) et Bons de Commande (14).** Le scanner, la réception et le comptage agissent sur le **même inventaire** que le magasin ; la réception clôt des **bons de commande**.
- **Module Comptabilité (15).** Les factures créées, envoyées et payées au mobile sont les **mêmes** que celles de la comptabilité ; les paiements Stripe s'y reflètent.
- **Assistant IA (25) et crédits IA.** L'assistant mobile puise dans les **crédits IA de l'entreprise**, comme l'assistant de l'ERP.
- **Météo Chantier (16).** L'écran météo mobile reprend les **prévisions** Open-Meteo du module Météo.
- **Stripe.** Sert aux **liens de paiement** de factures (paiement client par carte). La clé Stripe est côté serveur, jamais exposée.
- **Courriels (SMTP).** Servent à l'**envoi des documents** (PDF joint) et aux **relances d'impayés**.

### 5.2 Ce qui n'est PAS possible

- **Pas de connexion par courriel / mot de passe personnels** : la connexion se fait par entreprise + choix de l'employé + **NIP**.
- **Pas d'affichage de la géolocalisation** : aucun écran ne montre vos coordonnées ni une carte.
- **Pas de saisie de la météo** : elle est capturée automatiquement.
- **Pas de facturation des extras au mobile** : la facturation se fait au bureau.
- **Pas de réponse en fil ni de gestion fine des membres de canal** dans l'interface mobile.
- **Pas de réception / comptage hors-ligne**, ni d'**édition de document** hors-ligne.
- **Pas de planification automatique** des factures récurrentes (déclenchement manuel).
- **Écran de téléversement photo autonome non accessible** (route `/photo` non reliée).
- **Devis / Factures / Bons de commande invisibles pour un EMPLOYE.**

### 5.3 Foire aux questions

**Est-ce la même application que l'ERP ?**
Non. C'est une **application mobile séparée**, servie sous `app.constructoai.ca/mobile`, avec sa propre connexion. Elle partage toutefois la **base de données** de l'ERP : les données sont communes.

**Comment se connecter ?**
En trois étapes : **courriel + mot de passe de l'entreprise**, puis **votre nom** dans la liste, puis votre **NIP à 4 chiffres**. Pas de compte personnel à créer.

**J'ai oublié mon NIP.**
Demandez à votre administrateur de le réinitialiser. Après plusieurs essais erronés, attendez quelques minutes.

**Puis-je pointer sans réseau ?**
Oui. L'heure de votre geste est enregistrée localement et remonte à la reconnexion. Un pointage sans sortie est **plafonné à 16 heures**.

**Mon patron voit-il où je suis ?**
La position est captée au pointage pour rattacher vos heures au bon chantier, mais **elle ne s'affiche nulle part** dans l'application. Les photos sont, elles, **débarrassées de leur localisation** avant envoi.

**Pourquoi je ne vois pas les devis ni les factures ?**
Parce que vous avez le rôle **EMPLOYE** : seuls les **bons de travail** vous sont visibles. Les documents financiers sont réservés aux ADMIN et MANAGER.

**Comment joindre une photo à un projet ?**
Par une **note de dossier** (Fiche 360 → Notes → ajouter des photos) ou par la **messagerie**. L'écran de téléversement photo autonome n'est pas accessible actuellement.

**Je suis contremaître : comment approuver les heures ?**
Menu **Plus → Équipe**, dépliez le projet, puis **« Approuver »** pour chaque employé (si vous êtes approbateur).

**L'assistant IA coûte-t-il quelque chose ?**
Il consomme les **crédits IA de l'entreprise**, au **coût réel** des jetons. Si le solde est épuisé, l'assistant devient indisponible jusqu'à recharge.

**Les notifications ne s'activent pas.**
Sur iPhone, il faut **iOS 16.4 ou plus récent**. Si vous aviez activé les notifications à l'ancienne adresse, **réactivez-les** ici (touchez la cloche).

**Comment un client paie-t-il une facture ?**
Ouvrez la facture → **« Lien de paiement Stripe »** → transmettez le lien. Une fois payée, la facture passe à **« Payée »** automatiquement.

**Puis-je facturer un extra depuis mon téléphone ?**
Vous pouvez le **créer** et le faire **approuver**, mais la **facturation se fait au bureau (web)**.

**L'application est-elle bilingue ?**
Oui. La bascule **FR / EN** (barre du haut) change toute l'interface. Votre choix est mémorisé.

### 5.4 Dépannage courant

| Symptôme | Piste |
|---|---|
| Impossible de pointer l'entrée | Choisissez d'abord un **bon de travail** ; le bouton est grisé sinon |
| « Déjà pointé » au moment d'entrer | Un pointage est déjà ouvert : pointez d'abord la **sortie** |
| Le NIP est refusé | Vérifiez le code ; après plusieurs essais, **patientez** quelques minutes |
| Mes heures ne remontent pas | Vous étiez **hors-ligne** : reconnectez-vous, l'action se synchronise (bouton **« Synchroniser maintenant »**) |
| Je ne peux pas modifier un pointage | Il est **validé** ou **facturé** (verrouillé) ; voyez avec le bureau |
| Le bouton « Relances » / « Journal d'audit » est absent | Réservés respectivement aux **ADMIN/MANAGER** et **ADMIN** |
| Les tuiles Réception / Comptage n'apparaissent pas | Vous n'avez pas la **permission de stock** |
| Réception / Comptage refusés hors-ligne | Ces écrans **exigent une connexion** |
| Impossible d'éditer un document | L'édition **hors-ligne est désactivée** ; reconnectez-vous |
| Les notifications ne fonctionnent pas | iOS < 16.4 non pris en charge ; sinon **réactivez** la cloche |
| L'assistant IA répond « indisponible » | Le **solde de crédits IA** de l'entreprise est peut-être épuisé |
| Je ne trouve pas l'écran « photo » | Normal : passez par les **notes de dossier** ou la **messagerie** |

---

## 6. Récapitulatif

- **Le Pointeur mobile est une application autonome** (une PWA installable), servie sous **`app.constructoai.ca/mobile`**, avec sa **propre connexion** (courriel d'entreprise → choix de l'employé → **NIP à 4 chiffres**, session de 24 h). Ce n'est pas un onglet de l'ERP Web, mais elle **partage la même base de données**.
- **C'est un mini-ERP de chantier**, pas seulement une pointeuse : **pointage** (GPS et météo automatiques, signature de superviseur, historique et résumé hebdomadaire), **messagerie** (canaux et messages directs), **dossiers 360** (notes, étapes, extras), **documents** (devis, factures, bons de travail, bons de commande, PDF, courriel, paiement Stripe, relances), **inventaire** (scanner, réception, comptage) et **outils** (assistant IA, météo, calculatrice).
- **Quatre rôles** (ADMIN, MANAGER, EMPLOYE, APPRENTI) décident de ce que chacun voit et fait, **vérifiés côté serveur**. Un EMPLOYE ne voit que les **bons de travail** côté documents et n'a **pas** d'outils ERP dans l'assistant ; les fonctions sensibles (relances, journal d'audit, factures récurrentes) sont réservées aux responsables.
- **Le pointage** enregistre l'heure de votre geste (même hors-ligne), capte la **position** et la **météo** en arrière-plan, se plafonne à **16 h**, et calcule les heures supplémentaires **au-delà de 8 h/jour et 40 h/semaine**.
- **Le mode hors-ligne** met en file **18 types d'actions** puis les synchronise ; seuls la **réception**, le **comptage** et l'**édition de document** exigent une connexion. Seul le **panier de stock** est protégé contre les doublons (l'idempotence générale est présente dans le code mais **non activée**).
- **Points de vigilance** : l'écran de **téléversement photo `/photo` n'est relié à aucune navigation** (utilisez les notes ou la messagerie) ; la **création de canal** et les **fils de discussion** ne sont que partiellement exposés ; les **extras se facturent au bureau** ; les **notifications** exigent iOS ≥ 16.4 et une **réactivation** après le déménagement sous `/mobile`.
- **Volumétrie technique** : backend `mobile_api.py` (**5 278 lignes, 118 points d'accès**) + `attachments_api.py` (**7 points d'accès**) + `mobile_database.py` (12 822 lignes), sous **`/api/mobile/v1`** ; frontend `MOBILE_REACT/frontend` (**24 routes**), co-hébergé par l'ERP.

---

*Fichiers sources vérifiés :* backend (`MOBILE_REACT/backend/`) — `mobile_api.py` (5 278 lignes ; 118 décorateurs de route ; pointage `punch_in`/`punch_out`, seuils `OVERTIME_DAILY = 8` / `OVERTIME_WEEKLY = 40` à `mobile_api.py:1295-1296` ; types de document `VALID_DOC_TYPES` à `:3281`), `attachments_api.py` (361 lignes, 7 points d'accès), `mobile_database.py` (12 822 lignes ; rôles `VALID_ROLES_MOBILE` à `:229` ; plafond 16 h du pointage à `:3018` ; tarifs IA Sonnet 3 $/15 $ et débit `balance_usd` sans majoration à `:8213-8214` et `:8318`), `mobile_auth.py` (266 lignes ; JWT 24 h, URL signées), `mobile_models.py` (1 390 lignes), `mobile_pdf_service.py` (635 lignes), `mobile_attachments_service.py` (434 lignes), `mobile_idempotency.py` (264 lignes ; `install(app)` défini à `:260` mais **jamais appelé** = middleware dormant). Frontend (`MOBILE_REACT/frontend/src/`) — `App.tsx` (24 routes), `components/layout/MobileLayout.tsx` (256 lignes ; 5 onglets + menu Plus, garde `canAccessReminders`/`canAccessAudit` à `:47-49`), `pages/PunchPage.tsx` (504 lignes), `pages/DossierDetailPage.tsx` (1 471 lignes), `pages/ScannerPage.tsx` (1 123 lignes), et les pages Login / Dashboard / Crew / Messages / Dossiers / Documents / AiAssistant / Meteo / Reception / Counting / Calculatrice / Reminders / AuditLog / Appearance / PhotoUpload. Montage co-hôte par l'ERP : `ERP_REACT/backend/erp_api.py:1339-1379` (routers montés + SPA sous `/mobile`). Route `/photo` orpheline confirmée par recherche (aucun `navigate('/photo')` ni `to="/photo"` hors `App.tsx:110`).

*Manuels liés :* `12-operations-pointage.md` (le pointage côté ERP, qui partage `time_entries`), `11-operations-bons-de-travail.md` (les bons de travail pointés), `06-ventes-dossiers.md` (les dossiers / Fiche 360), `09-operations-magasin.md` et `13-operations-bons-de-commande.md` (l'inventaire, la réception et les bons de commande), `14-operations-comptabilite.md` (les factures et paiements), `24-communication-assistant-ia.md` (l'assistant IA), `22-communication-messagerie.md` (la messagerie), `15-terrain-meteo-chantier.md` (la météo), `28-outils-calculateurs.md` (la calculatrice).

*Manuel ERP Constructo AI — Module 38 « Pointeur mobile (PWA de pointage employés) » — v1.0 vérifié — 2026-07.*
