# Module 37 — SEAOP (appels d'offres publics Québec)

> **Version** : 1.0 (rédaction initiale vérifiée par rapport au code source, juillet 2026)
> **Public visé** : ce manuel s'adresse aux **utilisateurs** de la plateforme SEAOP — les **donneurs d'ouvrage** (particuliers, entreprises, organismes qui font réaliser des travaux) et les **entrepreneurs / travailleurs de la construction** (qui déposent des soumissions et se font connaître). Il couvre aussi, en fin de document, les fonctions d'**Administration** (modération et suivi). Il ne s'adresse pas aux développeurs.
> **Ce qu'est SEAOP** : « Système Électronique d'Appel d'Offres Public ». C'est une **plateforme gratuite de mise en relation** entre donneurs d'ouvrage et entrepreneurs de la construction au Québec. Un donneur d'ouvrage publie un appel d'offres, les entrepreneurs déposent des soumissions, le client compare, échange par messagerie, attribue le contrat, puis évalue. S'y greffent trois **répertoires** publics, un **service payant** de demande d'estimation, un **Chat Room** communautaire, un **Assistant IA** et une **Administration**.
> **Application séparée** : SEAOP n'est **pas** un onglet interne de l'ERP Constructo AI. C'est une **application autonome** (basename `/seaop` — `App.tsx:37`), servie sous `app.constructoai.ca/seaop`, avec sa propre connexion, ses propres comptes et son propre jeton de session (distinct de celui de l'ERP). En production, elle est co-hébergée dans le même service que l'ERP (service Render `constructo-seaop`), mais reste une expérience à part.
> **Préfixe API** : `/api/seaop/v1` (`seaop_config.py:12`). **14 routeurs** montés (`seaop_api.py:351-398`), **77 points d'accès** au total.
> **Code de référence (backend, `SEAOP_REACT/backend/`)** : `seaop_api.py` (429 lignes, montage) · `seaop_config.py` (175) · `seaop_auth.py` (390, gardes JWT/session) · `seaop_database.py` (1 904, accès données) · `seaop_models.py` (389, validations Pydantic) · `seaop_email.py` (408, 8 courriels). Routeurs (`backend/routers/`) : `auth.py` (529, 9 points) · `leads.py` (489, 7) · `soumissions.py` (538, 7) · `messages.py` (255, 4) · `evaluations.py` (162, 3) · `notifications.py` (139, 4) · `chat_room.py` (392, 9) · `uploads.py` (240, 2) · `services.py` (975, 10 — estimation) · `admin.py` (153, 5) · `repertoire.py` (287, 3 — RBQ) · `professionnels.py` (535, 6) · `ouvriers.py` (641, 7) · `ai.py` (663, 1). Schéma des tables : `modules/seaop/seaop_db_postgres.py` (721, `init_seaop_tables`).
> **Code de référence (frontend, `SEAOP_REACT/frontend/src/`)** : `App.tsx` (105, routes) · pages `AccueilPage.tsx` (161), `NouveauProjetPage.tsx` (181), `EspaceEntrepreneurPage.tsx` (425), `LeadDetailPage.tsx` (754), `MesProjetsPage.tsx` (410), `EntrepreneurMessagesPage.tsx` (145), `ServiceEstimationPage.tsx` (1 292), `RepertoirePage.tsx` (292), `ProfessionnelsPage.tsx` (793), `OuvriersPage.tsx` (896), `AdminPage.tsx` (149), `NotificationsPage.tsx` (51), `ChatRoomPage.tsx` (10), `LoginPage.tsx` (102), `RegisterPage.tsx` (16). Disposition : `components/layout/` — `Sidebar.tsx` (210), `TopBar.tsx` (215), `AppLayout.tsx` (42). Composants clés : `auth/LoginForm.tsx` (490), `auth/RegisterForm.tsx` (545), `auth/ProfileChoice.tsx` (120), `leads/LeadForm.tsx` (673), `leads/LeadCard.tsx` (210), `leads/LeadFilters.tsx` (274), `soumissions/SoumissionForm.tsx` (540), `soumissions/SoumissionCard.tsx` (318), `soumissions/SoumissionList.tsx` (148), `aiAssistant/SeaopAiPanel.tsx` (382), `chatRoom/ChatRoomPanel.tsx` (199), `messages/ChatThread.tsx` (237), `messages/ConversationList.tsx` (144), `evaluations/EvaluationForm.tsx` (86). Administration : `admin/DashboardStats.tsx` (227), `admin/EntrepreneurTable.tsx` (469), `admin/SoumissionTable.tsx` (193), `admin/ServiceTabs.tsx` (940), `admin/OuvriersAdminTab.tsx` (274), `admin/ProfessionnelsAdminTab.tsx` (243), `admin/RepertoireAdminCard.tsx` (141). Constantes : `utils/constants.ts` (301). Traductions : `i18n/locales/{fr,en}/*.json`.
> **Tables PostgreSQL (base partagée `public.seaop_*`, PAS de multi-schéma par entreprise)** : `seaop_leads`, `seaop_entrepreneurs`, `seaop_soumissions`, `seaop_messages`, `seaop_evaluations`, `seaop_notifications`, `seaop_addenda`, `seaop_chat_room` (+ `_likes`, `_online`), `seaop_demandes_estimation`, `seaop_rbq_entrepreneurs`, `seaop_professionnels`, `seaop_ouvriers`, `seaop_ai_usage`. Comptes d'administration de plateforme : `public.super_admins`.
> **Cadrage** : la **plateforme d'appels d'offres est entièrement gratuite** — aucun paiement, aucun crédit facturé, aucune passerelle Stripe côté public. Le **service de demande d'estimation** (200 / 275 / 350 $) est un service professionnel **payant**, mais la facturation se fait **hors plateforme** (par courriel, à la livraison de la soumission) : aucun paiement n'est pris en ligne dans le code.

*Note de terminologie employée dans ce manuel :* « donneur d'ouvrage » (ou « client », `client` dans le code) désigne la personne ou l'organisation qui **publie** un appel d'offres et reçoit des soumissions ; « entrepreneur » désigne l'entreprise de construction qui **dépose** une soumission ; « appel d'offres » (ou « projet », ou « lead » dans le code) désigne la demande publiée par un donneur d'ouvrage ; « soumission » désigne l'offre chiffrée d'un entrepreneur en réponse à un appel d'offres ; « répertoire » (ou « annuaire ») désigne l'une des trois listes publiques d'entreprises et de travailleurs ; « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « administration » désigne les fonctions de modération et de suivi réservées à l'équipe de la plateforme.

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

### 1.1 Mission de la plateforme

SEAOP est une place de marché en ligne qui **met en relation** ceux qui ont des travaux à faire réaliser et ceux qui les réalisent. Le parcours principal est simple :

1. Un **donneur d'ouvrage** décrit son projet et le **publie** comme appel d'offres (public, gratuit, sans compte).
2. Les **entrepreneurs** parcourent les appels d'offres, filtrent par région et par type de travaux, puis **déposent une soumission** (prix, délai, inclusions, conditions).
3. Le donneur d'ouvrage **compare** les soumissions reçues, **échange par messagerie** avec les entrepreneurs, puis **attribue** le contrat à l'un d'eux.
4. Une fois les travaux réalisés, le donneur d'ouvrage **évalue** l'entrepreneur (note de 1 à 5 + commentaire), ce qui alimente sa réputation publique.

Autour de ce parcours gravitent des services complémentaires : **trois répertoires publics** (entrepreneurs RBQ, professionnels, ouvriers spécialisés), un **service payant de demande d'estimation**, un **Chat Room** communautaire, un **Assistant IA** (pour les personnes connectées) et une **Administration** (modération, vérification des licences RBQ, suivi des demandes d'estimation).

### 1.2 SEAOP est une application SÉPARÉE (à ne pas confondre avec l'ERP)

C'est le premier point à comprendre. Contrairement aux autres modules de ce manuel, **SEAOP n'est pas une page interne de l'ERP Constructo AI**. C'est une **application distincte**, avec :

- sa propre adresse (`app.constructoai.ca/seaop`) ;
- sa propre page de connexion et ses propres comptes ;
- son propre jeton de session (le jeton SEAOP **ne** vous ouvre **pas** l'ERP, et le jeton ERP **n'**ouvre **pas** SEAOP) ;
- sa propre barre latérale et sa propre barre supérieure.

> **À retenir.** La barre supérieure de SEAOP porte en permanence un lien **« Connexion Constructo AI »** (`TopBar.tsx`) qui **quitte** SEAOP pour revenir au portail de l'ERP. C'est le seul pont entre les deux mondes, et il est purement une redirection de page (pas un partage de session).

### 1.3 Deux modèles économiques cohabitent : gratuit et payant

SEAOP fait cohabiter **deux logiques d'argent bien distinctes**. Ne pas les confondre.

| | **Plateforme d'appels d'offres** | **Service de demande d'estimation** |
|---|---|---|
| Objet | Publier un projet, recevoir des soumissions, attribuer | Faire produire une **estimation professionnelle** par l'équipe Constructo AI |
| Prix | **Gratuit** (0 $), pour tous | **200 / 275 / 350 $** selon la complexité |
| Compte requis | Non pour publier ; compte entrepreneur pour soumissionner | Aucun compte : formulaire public |
| Paiement en ligne | Aucun | **Aucun non plus** — facturation **hors plateforme**, par courriel, **à la livraison** de la soumission |
| Où | Onglets « Déposer un projet » / « Appels d'offres » | Menu « Services » → « Demande d'estimation » |

> **À retenir.** Même le service **payant** ne prend **aucun paiement en ligne** : il n'y a ni Stripe, ni carte, ni passerelle dans le code de SEAOP. Vous soumettez une demande, l'équipe produit l'estimation et vous la facture ensuite, à la livraison, par courriel. Quant aux « crédits » et « abonnements » qui apparaissent parfois pour les comptes entrepreneurs, ce sont des colonnes **jamais facturées ni décomptées** (voir §4.6) : la plateforme d'appels d'offres reste gratuite de bout en bout.

### 1.4 Les quatre rôles et les trois façons de se connecter

SEAOP distingue **quatre types d'utilisateurs authentifiés** (`seaop_auth.py:264-269`) mais n'affiche que **trois onglets de connexion** (`LoginForm.tsx:30-38`).

| Rôle (code) | Qui | Onglet de connexion | Comment on s'identifie | Où l'on arrive |
|---|---|---|---|---|
| **Donneur d'ouvrage** (`client`) | Celui qui publie des projets | **« Donneur d'ouvrage »** | Deux voies : (A) **lien magique** reçu par courriel, ou (B) **courriel + numéro de référence** de son appel d'offres (`SEAOP-AAAAMMJJ-XXXXXXXX`) | `/mes-projets` |
| **Entrepreneur** (`entrepreneur`) | L'entreprise qui soumissionne | **« Entrepreneur »** | **Courriel + mot de passe** (compte à créer) | `/appels-offres` |
| **Administration** (`super_admin`) | L'équipe de la plateforme | **« Administration »** | **Nom d'utilisateur + mot de passe** (comptes individuels) | `/administration` |
| **Admin** (`admin`) | Accès de secours interne | *(masqué de l'interface)* | Mot de passe partagé, côté serveur uniquement | — |

> **Le donneur d'ouvrage n'a pas de mot de passe.** Il ne « crée pas de compte » au sens classique. Son identité repose sur son **courriel** et le **numéro de référence** de son projet (ou sur un **lien magique** que la plateforme lui envoie par courriel). C'est voulu : publier un projet doit rester sans friction.
>
> **Le rôle `admin` (mot de passe partagé) n'apparaît plus dans l'interface** (`LoginForm.tsx:16-18`). Il subsiste côté serveur comme accès de secours, mais toute l'administration passe désormais par des **comptes individuels** (onglet « Administration »). Ne le documentez pas comme une voie d'accès normale.

### 1.5 Les trois répertoires (distincts des rôles ci-dessus)

Il ne faut pas confondre les **rôles** (qui se connectent) avec les **répertoires** (des annuaires publics que l'on consulte). SEAOP publie **trois répertoires** :

| Répertoire | Menu | Contenu | Provenance | Coordonnées |
|---|---|---|---|---|
| **Entrepreneurs (RBQ)** | Services → Répertoire des entrepreneurs | ~53 000 détenteurs d'une **licence RBQ active** | **Données ouvertes de Données Québec** (importées, pas d'inscription) | Publiques (téléphone / courriel de la licence) |
| **Professionnels** | Services → Répertoire des professionnels | Technologues, architectes, ingénieurs en structure | **Auto-inscription modérée** (validée avant publication) | **Masquées** (révélées après contact, Loi 25) |
| **Ouvriers spécialisés** | Services → Répertoire des ouvriers spécialisés | Travailleurs autonomes par corps de métier | **Auto-inscription modérée** | **Masquées** (révélées après contact, Loi 25) |

Le répertoire des **entrepreneurs RBQ** est donc une **liste officielle importée** : on ne s'y inscrit pas. Les répertoires des **professionnels** et des **ouvriers** sont, eux, alimentés par les intéressés eux-mêmes (auto-inscription), puis **modérés** par l'administration avant d'apparaître au public. Le répertoire des ouvriers est le plus récent des trois.

### 1.6 Comment accéder à SEAOP et s'y repérer

**Accès.** Ouvrez `app.constructoai.ca/seaop`, ou passez par la tuile « Appels d'offres publics / SEAOP » du portail Constructo AI. La page d'accueil, les appels d'offres, le dépôt de projet et les trois répertoires sont **publics** (aucune connexion requise).

**La barre latérale** (`Sidebar.tsx`) présente, sous la marque « SEAOP / Appels d'offres » :

- **Items principaux** : « Accueil », « Déposer un projet », « Appels d'offres », « Mes projets » *(donneur d'ouvrage seulement)*, « Mes messages » *(entrepreneur seulement)*, « Chat Room » ;
- **Section « Services »** (repliable) : « Demande d'estimation », « Répertoire des entrepreneurs », « Répertoire des professionnels », « Répertoire des ouvriers spécialisés » ;
- **« Administration »** *(administration seulement)*.

**La barre supérieure** (`TopBar.tsx`) contient : le bouton hamburger (sur mobile), le lien de retour **« Connexion Constructo AI »**, le titre de la page, la **bascule de langue FR / EN**, la **cloche de notifications** (pastille rouge avec le nombre de non-lus), la **bascule de thème** clair / sombre, et le menu utilisateur (nom, courriel, « Se déconnecter ») ou un bouton « Connexion ».

**L'Assistant IA** (bouton flottant + panneau latéral) est monté **sur toutes les pages** (`AppLayout.tsx:39`), mais il n'est utilisable **qu'une fois connecté**.

### 1.7 Confidentialité, modération et protection des coordonnées (Loi 25)

SEAOP applique une **protection systématique des coordonnées** :

- Sur la **liste publique des appels d'offres**, le **courriel et le téléphone** du donneur d'ouvrage sont **masqués** à tout le monde sauf au propriétaire du projet et à l'administration (`leads.py`, `_strip_lead_contact`).
- Sur les répertoires **professionnels** et **ouvriers**, les coordonnées ne sont **jamais** affichées d'emblée : elles ne se révèlent **qu'après** l'envoi d'un **formulaire de contact** (nom, courriel, message). Ce n'est pas un bogue, c'est une mesure anti-moisson conforme à la **Loi 25** (voir §2.14 et §2.15).
- Le **Chat Room** ne diffuse jamais le courriel des participants ; l'Assistant IA masque aussi les coordonnées des tiers.

Les répertoires **professionnels** et **ouvriers** passent par une **file de modération** : une fiche soumise reste « En attente » jusqu'à ce que l'administration l'**approuve** ou la **rejette**.

### 1.8 Le mode « développement » (DEV_MODE)

Un indicateur `DEV_MODE` (variable `VITE_DEV_MODE` côté interface, `SEAOP_DEV_MODE` côté serveur) permet de **verrouiller la plateforme** pendant une phase de préparation : dans ce mode, **seul l'onglet « Administration » est visible** à la connexion, et un bandeau « Mode développement » s'affiche (`LoginPage.tsx`). En production, `DEV_MODE` doit être à `false` : les trois onglets publics apparaissent alors normalement. Si vous ne voyez que l'onglet « Administration », c'est que la plateforme est en mode développement.

---

## 2. Interface

### 2.1 Disposition générale (barre latérale, barre supérieure, Assistant IA)

Toutes les pages partagent la même disposition (`AppLayout.tsx`) : **barre latérale** à gauche (fixe sur grand écran, en tiroir sur mobile), **barre supérieure** en haut, contenu au centre, et **bouton flottant Assistant IA** en bas à droite.

- **Bascule de langue.** Le bouton FR / EN de la barre supérieure traduit toute l'interface. **Exception connue** : l'écran « Demande d'estimation » est rédigé en français « en dur » (voir §2.12) — sa version anglaise est incomplète.
- **Thème clair / sombre.** La bascule de thème mémorise votre choix.
- **Cloche de notifications.** Elle affiche le nombre de notifications non lues et se met à jour périodiquement (interrogation toutes les 60 secondes). Un clic ouvre la page Notifications (§2.16).
- **Menu utilisateur.** Une fois connecté, votre nom et votre courriel s'affichent, avec l'option « Se déconnecter ».

### 2.2 Accueil (`AccueilPage.tsx`) — public

La page d'accueil présente :

- un **bandeau principal** : titre « SEAOP », sous-titre « Système Électronique d'Appel d'Offres Public », et deux boutons d'action — **« Déposer un projet »** et **« Voir les appels d'offres »** ;
- une **rangée de statistiques** (4 cartes) : « Projets actifs » (nombre d'appels d'offres visibles), « Entrepreneurs », « Soumissions », « Taux conformité » (ces trois dernières affichent la mention « Nouveau ») ;
- une section **« Projets récents »** : jusqu'à 6 cartes d'appels d'offres, avec un lien « Voir tous les appels d'offres ». Si rien n'est publié : « Aucun projet publié pour le moment ».

### 2.3 Déposer un appel d'offres (`NouveauProjetPage.tsx` + `LeadForm.tsx`) — public

C'est le formulaire par lequel un **donneur d'ouvrage** publie son projet. Aucune connexion n'est nécessaire. En-tête : « Déposer un appel d'offres ».

**Confort de saisie :**

- **Brouillon auto-sauvegardé.** Le formulaire enregistre votre saisie dans le navigateur. Si vous revenez, un bandeau « Brouillon disponible » propose de **Restaurer** ou de repartir à neuf.
- **Indicateur d'urgence en direct.** Un badge (Faible / Normal / Élevé / Critique) se calcule en temps réel à partir de vos deux dates (voir §4.5).

**Champs du formulaire** (`leadForm.json`) :

| Champ | Obligatoire | Détail |
|---|:---:|---|
| **Nom du projet** | Oui | Titre du projet |
| **Courriel** | Oui | Sert à vous recontacter et à vous connecter plus tard |
| **Téléphone** | Oui | — |
| **Code postal** | Oui | Sert au classement par région |
| **Type de projet** | Oui | 11 choix (voir §4.4) |
| **Budget** | Oui | 6 tranches (voir §4.4) |
| **Délai de réalisation** | Oui | 6 choix (voir §4.4) |
| **Date limite des soumissions** | Non | Par défaut, dans 14 jours. Détermine l'urgence. |
| **Date de début souhaitée** | Non | Par défaut, dans 30 jours. |
| **Description** | Oui | Minimum 20 caractères (compteur affiché) |
| **Documents et plans** | Non | Jusqu'à 5 fichiers, 150 Mo max par fichier. Formats : PDF, JPG, PNG, DOC, DOCX, XLS, XLSX |

**Section repliable « Exigences légales et assurances »** (badge « Recommandé ») — pour exiger des soumissionnaires certaines garanties :

- **Licence RBQ requise** (+ **Catégories RBQ acceptées**) ;
- **Attestation CNESST requise** ;
- **Assurance responsabilité civile requise** (+ **Montant minimum** en $) ;
- **Cautionnement de soumission requis** (+ **Pourcentage** en %).

**Publication.** Le bouton **« Publier l'appel d'offres »** enregistre le projet et affiche un **écran de succès** : le **numéro de référence** (`SEAOP-AAAAMMJJ-XXXXXXXX`, avec copie en un clic), la mention « Un courriel de confirmation a été envoyé », et deux boutons (« Retour à l'accueil » / « Voir les autres appels d'offres »).

> **Conservez votre numéro de référence.** C'est lui qui, avec votre courriel, vous permettra de **revenir consulter vos soumissions** et de vous connecter comme donneur d'ouvrage (§3.2).

### 2.4 Appels d'offres / Espace entrepreneur (`EspaceEntrepreneurPage.tsx`) — public + entrepreneur

Cette page est **publique** (tout le monde voit la liste des appels d'offres) mais devient l'**espace de travail de l'entrepreneur** une fois connecté.

**En-tête.**

- Visiteur anonyme : titre « Appels d'offres » + bouton « Déposer un projet ».
- Entrepreneur connecté : « Bienvenue, {entreprise} » + trois cartes de statistiques (Projets disponibles, Mes soumissions, Évaluation moyenne).

**Onglets** (visibles **seulement** pour l'entrepreneur connecté) :

- **« Appels d'offres »** — la liste filtrable des projets ouverts ;
- **« Mes soumissions »** — la liste de vos soumissions déposées ;
- **« Mon profil »** — votre fiche en lecture seule (Entreprise, Contact, Courriel, Téléphone, N° RBQ, Zones desservies, Types de projets, Certifications, Abonnement, Statut, Évaluation, Date d'inscription).

**Filtres de la liste** (`LeadFilters.tsx`) : recherche texte, **Type de projet**, **Région** (régions administratives du Québec), **Trier par** (date / nombre de soumissions / urgence), une pastille **« Mes zones desservies »** (pour filtrer sur vos régions déclarées) et un bouton **Réinitialiser**. La liste s'affiche en **cartes d'appels d'offres** (§2.5).

### 2.5 Carte d'appel d'offres (`LeadCard.tsx`)

Chaque projet apparaît sous forme de carte : des **badges** (statut Attribué / Fermé / Annulé, ou niveau d'urgence, + type de projet), le **titre**, une **description tronquée**, une grille (Budget, Délai, Code postal, Date limite), un **indicateur coloré de jours restants**, un **compteur de soumissions**, et deux boutons — **« Voir détails »** (vers la page de détail, §2.6) et **« Soumissionner »**. Le bouton « Soumissionner » est **masqué** si le projet est déjà attribué, fermé ou annulé.

### 2.6 Détail d'un appel d'offres (`LeadDetailPage.tsx`, route `/projet/:id`) — public

La page de détail rassemble tout ce qui concerne un projet :

- barre de retour + badges de statut / urgence ;
- **En-tête** : type de projet, numéro de référence, titre, date de publication ;
- **Description** complète ;
- **Documents joints** : chaque pièce (Document / Plan / Photo) est téléchargeable ;
- **Détails** : Budget, Délai, Date de début, Date limite (+ jours restants), Code postal ; le **téléphone et le courriel ne sont visibles que par le propriétaire ou l'administration** ;
- **Exigences de conformité** : badges RBQ / CNESST / Assurance / Cautionnement avec les montants exigés ;
- **Addenda** : la liste des addenda publiés. Un formulaire **« Ajouter un addendum »** (Titre + Description) est réservé au **propriétaire du projet ou à l'administration** ; publier un addendum **notifie et envoie un courriel à tous les entrepreneurs** ayant déjà soumissionné ;
- **Soumissions reçues** : visible **uniquement une fois connecté** (cartes de soumission, §2.8) ; pour le donneur d'ouvrage, l'action **« Attribuer »** y figure ;
- un **bouton fixe « Soumettre une proposition »** (pour l'entrepreneur) qui ouvre le formulaire de soumission.

### 2.7 Formulaire de soumission (`SoumissionForm.tsx`) — entrepreneur

Ouvert depuis une carte ou la page de détail, ce formulaire sert à l'entrepreneur pour **chiffrer son offre**.

**Calculateur de taxes du Québec intégré.** Vous saisissez un montant, et le formulaire affiche en direct le récapitulatif **HT / TPS 5 % / TVQ 9,975 % / TTC**. Une **bascule HT / TTC** permet d'indiquer si votre montant est avant ou après taxes. Un **brouillon** est auto-sauvegardé par projet.

**Champs** (`soumissionForm.json`) :

| Champ | Obligatoire | Détail |
|---|:---:|---|
| **Montant** | Oui | Bascule HT / TTC ; borné (voir §4.7) |
| **Délai d'exécution** | Oui | Ex. « 6 semaines » |
| **Validité de l'offre** | Oui | Par défaut « 30 jours » |
| **Description des travaux** | Oui | Minimum 50 caractères |
| **Inclusions** | Non | Ce qui est compris |
| **Exclusions** | Non | Ce qui n'est pas compris |
| **Conditions** | Non | Modalités particulières |
| **Documents joints** | Non | Jusqu'à 3 fichiers, 150 Mo max par fichier |

**Cautionnement de soumission** (section optionnelle) : un encart explicatif, puis une case **« J'inclus un cautionnement »** qui déploie un **Type** (Chèque certifié / Lettre de garantie bancaire / Cautionnement d'assurance) et un **Montant** en $.

Le bouton **« Soumettre la proposition »** dépose la soumission. Le donneur d'ouvrage est alors **notifié** (dans l'application + courriel).

> **Une seule soumission par projet.** Un entrepreneur ne peut soumettre **qu'une** proposition par appel d'offres (une seconde tentative est refusée). Pour corriger votre offre, **modifiez** la soumission existante — mais seulement tant qu'elle est au statut « envoyée » et que la date limite n'est pas dépassée (voir §3.4).

### 2.8 Carte de soumission (`SoumissionCard.tsx`)

Chaque soumission s'affiche en carte : bannière du projet lié (vue entrepreneur), nom de l'entreprise et du contact, **badge « Vue »** (si le client l'a consultée), badge de statut, badges de conformité (**RBQ vérifiée**, **RBQ n°…**, **Assuré**, **Cautionné**), **montant** bien visible, description, délai + validité, **étoiles d'évaluation** de l'entrepreneur, et la date.

Pour le **donneur d'ouvrage**, deux actions apparaissent (avec confirmation) : **« Attribuer »** et **« Refuser »**. Les statuts possibles : envoyée → vue → en évaluation → acceptée / refusée.

### 2.9 Mes projets (`MesProjetsPage.tsx`, route `/mes-projets`) — donneur d'ouvrage

C'est le tableau de bord du donneur d'ouvrage. En-tête « Mes appels d'offres » + bouton « Messages ». Trois cartes de statistiques (Total projets, Avec soumissions, En cours).

- **Grille de projets** : chaque carte mène au **détail des soumissions** de ce projet (via `SoumissionList`), avec les actions **Accepter / Refuser / Attribuer / Détails**.
- **Panneau de messagerie coulissant** : liste des conversations (`ConversationList`) + fil de discussion (`ChatThread`).
- **Modale d'évaluation** (`EvaluationForm`) : sur une soumission **acceptée**, vous pouvez noter l'entrepreneur (étoiles 1 à 5 + commentaire).
- État vide : « Vous n'avez pas encore publié de projet ».

### 2.10 Mes messages (`EntrepreneurMessagesPage.tsx`, route `/messages`) — entrepreneur

L'espace de messagerie de l'entrepreneur. En-tête « Mes messages ». Disposition en deux colonnes : la **liste des conversations** (`ConversationList`) à gauche, le **fil de discussion** (`ChatThread`) à droite. État vide : « Aucun message pour l'instant… ».

**Le fil de discussion (`ChatThread.tsx`)** : bulles alignées selon l'émetteur, nom de l'expéditeur (Entrepreneur / Client), horodatage relatif, accusés de lecture (une coche = envoyé, deux coches = lu), zone de saisie (la touche **Entrée** envoie), verrou anti-double-envoi.

> **Qui peut écrire à qui.** Un donneur d'ouvrage ne peut écrire **qu'à un entrepreneur ayant réellement soumissionné sur son propre projet** (règle serveur anti-harcèlement, `messages.py`). Il n'existe pas de messagerie « libre » entre inconnus.

### 2.11 Chat Room (`ChatRoomPanel.tsx`, route `/chat-room`) — public en lecture, écriture si connecté

Un **salon communautaire** ouvert à tous en lecture. Il présente : une barre latérale « Utilisateurs en ligne » (avec des statistiques), un bandeau orange des **messages épinglés**, et le **fil de messages** (chaque message peut être aimé, on peut y répondre, et l'administration peut supprimer). Le salon se rafraîchit périodiquement (toutes les 30 secondes). La zone de saisie (max 5 000 caractères) n'apparaît que si vous êtes connecté ; sinon : « Connectez-vous pour participer à la discussion. » Les courriels ne sont **jamais** diffusés dans le salon.

### 2.12 Demande d'estimation (`ServiceEstimationPage.tsx`, route `/services/estimation`) — service payant, public

C'est le **service professionnel payant**. On y demande une estimation détaillée produite par l'équipe Constructo AI. Aucun compte n'est requis.

> **Particularité technique.** Cet écran est rédigé en **français « en dur »** (il n'utilise pas le système de traduction). Sa version anglaise est donc **incomplète** : le contenu reste en français même en mode EN.

**Bandeau de tarification** — trois paliers :

| Palier | Prix | Note |
|---|---|---|
| **SIMPLE** | **200 $** | — |
| **MOYEN** | **275 $** | Badge « Le plus courant » |
| **COMPLEXE** | **350 $** | — |

Trois mentions accompagnent les prix, dont **« Facturation uniquement à la livraison de la soumission complétée »**. Contacts affichés : `info@constructoai.ca` et 1 (936) 587-1141.

**Assistant en 4 étapes :**

1. **Projet** — **Corps de métier** (21 choix, voir §4.4), **Secteur** (Résidentiel / Commercial / Institutionnel / Industriel), **Type de projet**.
2. **Besoin** — **Description** (10 à 5 000 caractères), **Documents** en glisser-déposer (images ≤ 5 Mo, PDF ≤ 150 Mo, jusqu'à 10 fichiers combinés, avec barre de progression), **Niveau d'urgence** (Normal / Urgent), **Disponibilité** (Dès que possible / Date précise).
3. **Détails** (tous facultatifs) — Code postal, Superficie / dimensions, Localisation / adresse, Budget estimé, Délai souhaité.
4. **Coordonnées** — Prénom, Nom, Courriel, Téléphone, Entreprise (facultatif) + récapitulatif.

**Écran de succès** : « Demande reçue » + **numéro de référence** + la promesse d'une **estimation détaillée par courriel dans un délai de 24 à 48 heures ouvrables**.

### 2.13 Répertoire des entrepreneurs (`RepertoirePage.tsx`, route `/services/repertoire`) — public

Annuaire des **détenteurs d'une licence RBQ active**, alimenté par les **données ouvertes de Données Québec**. On **n'y inscrit personne** : c'est une liste officielle importée.

**Filtres** : Région, Métier (sous-catégorie RBQ), Recherche (nom / ville / n° de licence, avec une courte pause de frappe de 400 ms). **Cartes** : nom, « Licence RBQ n° … », badge « Licence avec restriction » le cas échéant, municipalité / région, sous-catégories (jusqu'à 6), liens téléphone / courriel. **Pagination** : 20 par page.

### 2.14 Répertoire des professionnels (`ProfessionnelsPage.tsx`, route `/services/professionnels`) — public

Annuaire **auto-inscrit et modéré** de technologues professionnels, architectes et ingénieurs en structure. Un bandeau rappelle que les fiches sont « validées par SEAOP avant publication ».

**Filtres** : Type (Technologue professionnel / Architecte / Ingénieur en structure), Région (régions administratives du Québec), Recherche. **Cartes** : nom, entreprise, badge de type, N° de membre, région, spécialité, description, boutons **« Contacter »** et « Site web ».

- **Modale « Contacter » (protégée, Loi 25).** Les coordonnées sont **masquées** par défaut. Elles ne se révèlent **qu'après** l'envoi d'un formulaire : **Votre nom**, **Votre courriel**, **Téléphone** (facultatif), **Message** (≥ 10 caractères). Une fois envoyé, le téléphone et le courriel du professionnel s'affichent, et votre message lui est relayé.
- **Modale « S'inscrire ».** Type, Nom complet, Entreprise, N° de membre, Courriel, Téléphone, Région, Municipalité, Site web, Spécialité, Description. La fiche est publiée **après validation** par l'administration.

### 2.15 Répertoire des ouvriers spécialisés (`OuvriersPage.tsx`, route `/services/ouvriers`) — public

Le troisième et plus récent répertoire : des **travailleurs autonomes par corps de métier**. Même logique d'**auto-inscription modérée** et de **coordonnées protégées** que pour les professionnels.

**Filtres** : Corps de métier (21 choix), Région, Recherche. **Cartes** : nom, entreprise, badge de métier, région + zones desservies, méta (années d'expérience, Carte CCQ, RBQ, Travailleur autonome, tarif $/h, disponibilité), certifications, description, **Réalisations** (vignettes de portfolio), boutons **« Contacter »** (protégé, même mécanisme que les professionnels) et « Site web ».

- **Modale « S'inscrire ».** Nom complet, Corps de métier, Entreprise, Courriel, Téléphone, Site web, Région, Municipalité, Zones desservies, Années d'expérience, Carte de compétence CCQ, Tarif horaire min / max, Disponibilité, Licence RBQ, NEQ, case « Je suis travailleur autonome », Certifications, Description. **Au moins un courriel ou un téléphone est requis.** Le portfolio accepte jusqu'à 4 images.

### 2.16 Notifications (`NotificationsPage.tsx`, route `/notifications`) — connecté

La liste de vos notifications, avec une icône par type (soumission, message, évaluation, statut, alerte, information). Un clic **marque comme lu** ; un bouton **« Tout marquer comme lu »** vide le compteur. Chaque ligne affiche un temps relatif. La **cloche** de la barre supérieure reflète le nombre de non-lus (mise à jour toutes les 60 secondes).

### 2.17 Connexion et inscription

**Écran de connexion (`LoginPage.tsx` + `LoginForm.tsx`).** Il commence par un **choix de profil** (`ProfileChoice.tsx`) : deux grandes cartes — **« Entrepreneur »** (mène à la création de compte) et **« Donneur d'ouvrage »** (mène à la publication d'un projet) — plus un lien discret **« Connexion administration »**. Vient ensuite le formulaire de connexion à **trois onglets** (§1.4). L'écran gère aussi les **liens magiques** : si vous arrivez avec un lien reçu par courriel (`?magic=…`), vous êtes connecté automatiquement comme donneur d'ouvrage.

**Écran d'inscription entrepreneur (`RegisterForm.tsx`).** En **deux étapes** :

- **Étape 1 « Compte »** : Nom de l'entreprise, Nom du contact, Courriel, Téléphone, Mot de passe (**≥ 8 caractères**), Confirmation. Tous obligatoires.
- **Étape 2 « Profil » (facultative)** : N° RBQ (format `XXXX-XXXX-XX`, avec un lien de vérification RBQ), Catégories RBQ, case Assurance + Montant, Zones desservies, Types de projets, Certifications. Un bouton **« Ignorer et créer mon compte »** permet de sauter cette étape.

### 2.18 Administration (`AdminPage.tsx`, route `/administration`)

Réservée à l'administration. En-tête « Administration ». Les **onglets** :

| Onglet | Contenu | Accès |
|---|---|---|
| **Vue d'ensemble** | 4 indicateurs (Projets, Entrepreneurs, Soumissions, CA total), graphe « Évolution mensuelle », table « Top 5 entrepreneurs » (Soumissions / Acceptées / Revenus / Évaluation) | admin + super_admin |
| **Entrepreneurs** | Table filtrable par statut (Actif / Inactif / Suspendu) : Entreprise, Contact, Courriel, RBQ (+ badge vérifiée), Abonnement, Crédits, Statut, Évaluation. Actions : **Suspendre / Activer**, **Vérifier RBQ**, **Crédits** | admin + super_admin |
| **Soumissions** | « Soumissions récentes » (Réf. projet, Entrepreneur, Montant, Statut, Date) | admin + super_admin |
| **Services** | (a) Carte **Répertoire RBQ** (compteur, dernier import, bouton **« Rafraîchir le répertoire »** — super_admin) ; (b) **Diagnostic courriels / SMTP** (état + courriel de test) ; (c) table des **demandes d'estimation** avec modale de détail (infos client, pièces jointes, notes internes, actions **Renvoyer au client / Refuser / Marquer comme envoyée**) | admin + super_admin |
| **Ouvriers** | File de modération (En attente / Approuvées / Rejetées), actions **Approuver / Rejeter** | admin + super_admin |
| **Professionnels** | File de modération identique | **super_admin uniquement** |

> **Asymétrie de permissions à connaître.** L'onglet **« Professionnels » est réservé au super_admin** ; l'onglet **« Ouvriers » est ouvert à l'admin et au super_admin**. De même, le **rafraîchissement du répertoire RBQ** (import des données Québec) est réservé au super_admin.

### 2.19 Assistant IA (`SeaopAiPanel.tsx`)

Un **bouton flottant** ouvre un **panneau latéral** d'assistant conversationnel, disponible **sur toutes les pages mais réservé aux personnes connectées**. Sous-titre : « Appels d'offres & conformité QC ».

- **Ce qu'il sait faire** : **lire** vos données (appels d'offres, vos soumissions, votre profil, vos projets, la réputation d'un entrepreneur), **rédiger** des brouillons (un appel d'offres pour un client, une soumission pour un entrepreneur) et **analyser des images** (Vision : lecture de plans et de photos).
- **Pièces jointes** : jusqu'à 3 fichiers (JPEG, PNG, WEBP, GIF, PDF).
- **Suggestions contextuelles** adaptées à votre rôle (entrepreneur / client / général).
- **Confidentialité** : l'assistant ne peut consulter **que** vos propres données (le périmètre est **imposé par le serveur**, jamais choisi par l'IA) ; les coordonnées des tiers sont masquées sauf pour le propriétaire ou l'administration.

> **L'assistant ne modifie jamais la base de données.** Il **lit** et **rédige des brouillons** ; il ne publie, ni ne soumissionne, ni n'attribue à votre place. C'est à vous de reprendre son texte et de l'utiliser dans le formulaire voulu.

---

## 3. Workflows pas à pas

### 3.1 Publier un appel d'offres (donneur d'ouvrage)

1. Sans vous connecter, cliquez **« Déposer un projet »** (accueil ou barre latérale).
2. Remplissez les champs obligatoires : Nom du projet, Courriel, Téléphone, Code postal, Type de projet, Budget, Délai, Description (≥ 20 caractères).
3. Ajustez au besoin la **Date limite des soumissions** (par défaut +14 jours) et la **Date de début** (+30 jours). La date limite fixe le **niveau d'urgence** affiché aux entrepreneurs.
4. Joignez vos **plans / documents** (jusqu'à 5 fichiers).
5. Si nécessaire, dépliez **« Exigences légales et assurances »** et cochez RBQ / CNESST / Assurance / Cautionnement, avec les montants voulus.
6. Cliquez **« Publier l'appel d'offres »**.
7. **Notez et conservez le numéro de référence** affiché (`SEAOP-AAAAMMJJ-XXXXXXXX`). Un courriel de confirmation vous est envoyé.

### 3.2 Revenir consulter ses soumissions (donneur d'ouvrage)

Le donneur d'ouvrage n'a pas de mot de passe : deux voies s'offrent à lui.

**Voie A — lien magique :**
1. Onglet **« Donneur d'ouvrage »** de la connexion.
2. Demandez un **lien de connexion** à votre courriel.
3. Ouvrez le courriel reçu et cliquez le lien ; vous arrivez directement sur **« Mes projets »**. *(Le lien est valable 30 minutes.)*

**Voie B — courriel + numéro de référence :**
1. Onglet **« Donneur d'ouvrage »**.
2. Saisissez votre **courriel** et le **numéro de référence** de votre projet (`SEAOP-AAAAMMJJ-XXXXXXXX`).
3. Vous accédez à « Mes projets ».

3. Dans « Mes projets », ouvrez un projet pour voir ses **soumissions reçues**, comparer les montants, **échanger par messagerie**, puis **attribuer** (§3.5).

### 3.3 Créer un compte entrepreneur

1. Page de connexion → carte **« Entrepreneur »** (ou lien « Créer un compte »).
2. **Étape 1** : Nom de l'entreprise, Nom du contact, Courriel, Téléphone, Mot de passe (**≥ 8 caractères**), Confirmation. Continuez.
3. **Étape 2 (facultative)** : renseignez votre N° RBQ, vos catégories, votre assurance, vos **zones desservies** et vos **types de projets** (ces deux derniers alimentent le filtre « Mes zones desservies » et la pertinence des appels d'offres). Ou cliquez **« Ignorer et créer mon compte »**.
4. Connectez-vous ensuite par l'onglet **« Entrepreneur »** (courriel + mot de passe). Vous arrivez sur **« Appels d'offres »**.

### 3.4 Trouver un appel d'offres et soumissionner (entrepreneur)

1. Onglet **« Appels d'offres »**. Filtrez par **Type**, **Région**, ou activez **« Mes zones desservies »**. Triez par date, nombre de soumissions ou urgence.
2. Ouvrez un projet (**« Voir détails »**) pour lire la description, télécharger les **plans**, et vérifier les **exigences de conformité** (RBQ / CNESST / assurance / cautionnement).
3. Cliquez **« Soumissionner »** / **« Soumettre une proposition »**.
4. Renseignez **Montant** (bascule HT / TTC — le calculateur affiche TPS / TVQ / TTC), **Délai**, **Validité**, **Description** (≥ 50 caractères), et au besoin Inclusions / Exclusions / Conditions et un **cautionnement**.
5. Joignez jusqu'à 3 documents, puis **« Soumettre la proposition »**. Le client est notifié.
6. **Pour corriger** votre offre : retournez dans **« Mes soumissions »** et modifiez-la — possible **uniquement** tant qu'elle est « envoyée » et que la **date limite** n'est pas dépassée.

> Vous ne pouvez déposer **qu'une** soumission par projet. Après la date limite, le dépôt et la modification sont **fermés**.

### 3.5 Comparer les soumissions et attribuer (donneur d'ouvrage)

1. « Mes projets » → ouvrez le projet concerné pour voir la liste des **soumissions reçues**.
2. Comparez montants, délais, inclusions / exclusions, badges de conformité (RBQ vérifiée, Assuré, Cautionné) et l'**évaluation** de chaque entrepreneur.
3. Au besoin, ouvrez la **messagerie** pour poser des questions.
4. Cliquez **« Attribuer »** sur la soumission retenue, puis confirmez.

> **Ce que l'attribution déclenche automatiquement.** La soumission choisie passe **« acceptée »**, le projet passe **« attribué »**, il **cesse d'accepter de nouvelles soumissions**, et **toutes les autres soumissions sont refusées** — chaque entrepreneur (gagnant comme perdants) est **notifié** (dans l'application + courriel).

### 3.6 Échanger des messages

- **Entrepreneur** : onglet **« Mes messages »** → choisissez une conversation → écrivez au donneur d'ouvrage (Entrée envoie).
- **Donneur d'ouvrage** : « Mes projets » → bouton **« Messages »** (panneau coulissant) → choisissez la conversation avec l'entrepreneur voulu.

Rappel : un client ne peut écrire qu'à un entrepreneur **ayant soumissionné sur son projet**. À l'ouverture d'un fil, les messages reçus sont marqués comme lus (le compteur de la cloche baisse).

### 3.7 Évaluer un entrepreneur (donneur d'ouvrage)

1. « Mes projets » → le projet dont une soumission est **acceptée**.
2. Sur cette soumission, ouvrez la **modale d'évaluation**.
3. Donnez une **note de 1 à 5 étoiles** + un **commentaire**, puis validez.

> **On n'évalue qu'une soumission ACCEPTÉE.** C'est une protection : impossible de saboter la réputation d'un entrepreneur qu'on n'a pas retenu. Une évaluation est **unique** par soumission et par évaluateur (une nouvelle note remplace la précédente). La moyenne de l'entrepreneur est recalculée automatiquement.

### 3.8 Publier un addendum (propriétaire d'un projet)

1. Ouvrez le **détail** de votre projet (`/projet/:id`).
2. Section **Addenda** → **« Ajouter un addendum »** : Titre + Description.
3. Publiez. Tous les entrepreneurs ayant **déjà soumissionné** reçoivent une **notification et un courriel**. Les addenda précédents restent listés et numérotés.

### 3.9 Demander une estimation professionnelle (service payant)

1. Menu **Services → « Demande d'estimation »**.
2. Choisissez le palier qui correspond (SIMPLE 200 $ / MOYEN 275 $ / COMPLEXE 350 $), à titre indicatif.
3. Étape **Projet** : Corps de métier, Secteur, Type de projet.
4. Étape **Besoin** : Description (≥ 10 caractères), **documents** (glisser-déposer), urgence, disponibilité.
5. Étape **Détails** (facultative) : code postal, superficie, adresse, budget, délai.
6. Étape **Coordonnées** : Prénom, Nom, Courriel, Téléphone, Entreprise.
7. Envoyez. Vous recevez un **numéro de référence** (`EST-…`) et la promesse d'une réponse **par courriel sous 24 à 48 heures ouvrables**.

> **Aucun paiement à cette étape.** L'équipe produit l'estimation, puis vous la **facture à la livraison**, hors plateforme.

### 3.10 S'inscrire à un répertoire (professionnel ou ouvrier)

1. Menu **Services** → **« Répertoire des professionnels »** ou **« Répertoire des ouvriers spécialisés »**.
2. Ouvrez la modale **« S'inscrire »** et remplissez votre fiche (pour un ouvrier : au moins un courriel **ou** un téléphone ; un portfolio de 4 images maximum).
3. Envoyez. Votre fiche part **« En attente »** : elle n'apparaît au public **qu'après approbation** par l'administration.

### 3.11 Être mis en relation via un répertoire (protection Loi 25)

1. Ouvrez le répertoire (professionnels ou ouvriers) et repérez une fiche.
2. Cliquez **« Contacter »**. Les coordonnées sont **masquées**.
3. Remplissez le **formulaire de contact** (votre nom, votre courriel, votre message ≥ 10 caractères).
4. Après l'envoi, les **coordonnées s'affichent** et votre message est **relayé** au professionnel / ouvrier.

> Ce détour par un formulaire est **normal et voulu** (anti-moisson de courriels, conformité Loi 25). Ce n'est pas un dysfonctionnement.

### 3.12 (Administration) modérer, vérifier, traiter

1. **Onglet « Ouvriers » / « Professionnels »** : parcourez la file (En attente / Approuvées / Rejetées), ouvrez une fiche (coordonnées visibles ici), puis **Approuver** ou **Rejeter**.
2. **Onglet « Entrepreneurs »** : filtrez par statut, **Suspendez / Activez** un compte, **Vérifiez** sa licence RBQ (pose le badge « RBQ vérifiée » et notifie), ou ajustez ses **Crédits**.
3. **Onglet « Services »** : rafraîchissez le **répertoire RBQ** (super_admin), testez la configuration **courriel / SMTP**, et traitez les **demandes d'estimation** (ouvrez la modale, téléchargez les pièces, changez le statut : Renvoyer au client / Refuser / Marquer comme envoyée).

---

## 4. Référence

### 4.1 Écrans de SEAOP

| Écran | Fichier | Accès | Rôle |
|---|---|---|---|
| Accueil | `AccueilPage.tsx` | Public | Vitrine + projets récents |
| Déposer un appel d'offres | `NouveauProjetPage.tsx` / `LeadForm.tsx` | Public | Créer un projet |
| Appels d'offres / Espace entrepreneur | `EspaceEntrepreneurPage.tsx` | Public + entrepreneur | Liste, Mes soumissions, Mon profil |
| Détail d'un projet | `LeadDetailPage.tsx` | Public | Détail + addenda + soumissions |
| Mes projets | `MesProjetsPage.tsx` | Donneur d'ouvrage | Suivi, messagerie, attribution, évaluation |
| Mes messages | `EntrepreneurMessagesPage.tsx` | Entrepreneur | Messagerie |
| Chat Room | `ChatRoomPanel.tsx` | Public (lecture) | Salon communautaire |
| Demande d'estimation | `ServiceEstimationPage.tsx` | Public | Service payant (4 étapes) |
| Répertoire des entrepreneurs | `RepertoirePage.tsx` | Public | Annuaire RBQ |
| Répertoire des professionnels | `ProfessionnelsPage.tsx` | Public | Annuaire modéré |
| Répertoire des ouvriers | `OuvriersPage.tsx` | Public | Annuaire modéré |
| Notifications | `NotificationsPage.tsx` | Connecté | Alertes |
| Connexion / Inscription | `LoginPage.tsx` / `RegisterForm.tsx` | Public | 3 onglets, inscription entrepreneur |
| Administration | `AdminPage.tsx` | admin / super_admin | 6 onglets |
| Assistant IA | `SeaopAiPanel.tsx` | Connecté | Lecture + rédaction + Vision |

### 4.2 Points d'accès de l'API (77 au total)

Tous préfixés par `/api/seaop/v1`.

**Authentification (`auth.py`, 9) :**

| Méthode | Chemin | Accès |
|---|---|---|
| POST | `/auth/entrepreneur/login` | Public |
| POST | `/auth/entrepreneur/register` | Public |
| POST | `/auth/client/login` | Public (courriel + n° de référence) |
| POST | `/auth/client/request-link` | Public (lien magique) |
| POST | `/auth/client/verify-link` | Public (échange du jeton magique) |
| POST | `/auth/admin/login` | Public (mot de passe partagé, masqué de l'UI) |
| POST | `/auth/super-admin/login` | Public (comptes individuels) |
| POST | `/auth/logout` | Connecté |
| GET | `/auth/me` | Connecté |

**Appels d'offres (`leads.py`, 7) :**

| Méthode | Chemin | Accès |
|---|---|---|
| GET | `/leads` | Public (contacts masqués) |
| GET | `/leads/mes-projets` | Donneur d'ouvrage |
| GET | `/leads/{id}` | Public |
| POST | `/leads` | Public (5 / IP / h) |
| PUT | `/leads/{id}` | Propriétaire ou admin |
| GET | `/leads/{id}/addenda` | Public |
| POST | `/leads/{id}/addenda` | Propriétaire ou admin |

**Soumissions (`soumissions.py`, 7) :**

| Méthode | Chemin | Accès |
|---|---|---|
| POST | `/soumissions` | Entrepreneur |
| GET | `/soumissions/lead/{id}` | Client propriétaire ou admin |
| GET | `/soumissions/mes-soumissions` | Entrepreneur |
| GET | `/soumissions/{id}` | Participant ou admin |
| PUT | `/soumissions/{id}` | Entrepreneur émetteur (si « envoyée ») |
| PUT | `/soumissions/{id}/statut` | Client propriétaire ou admin |
| POST | `/soumissions/{id}/attribuer` | Client propriétaire ou admin |

**Messages (`messages.py`, 4) · Évaluations (`evaluations.py`, 3) · Notifications (`notifications.py`, 4) :**

| Méthode | Chemin | Accès |
|---|---|---|
| POST | `/messages` | Participant |
| GET | `/messages/conversations` | Connecté |
| GET | `/messages/conversation/{lead}/{entrepreneur}` | Participant |
| PUT | `/messages/mark-read/{lead}/{entrepreneur}` | Participant |
| POST | `/evaluations` | Client (soumission acceptée seulement) |
| GET | `/evaluations/entrepreneur/{id}` | Public |
| GET | `/evaluations/soumission/{id}` | Participant / admin |
| GET | `/notifications` | Connecté |
| GET | `/notifications/count` | Connecté |
| PUT | `/notifications/read-all` | Connecté |
| PUT | `/notifications/{id}/read` | Connecté |

**Chat Room (`chat_room.py`, 9) :** `GET /chat-room/messages` (public), `POST /chat-room/messages`, `PUT /chat-room/messages/{id}` (auteur), `DELETE /chat-room/messages/{id}` (auteur ou admin), `POST /chat-room/messages/{id}/like`, `PUT /chat-room/messages/{id}/pin` (admin), `GET /chat-room/online` (public), `POST /chat-room/heartbeat`, `GET /chat-room/stats` (public).

**Téléversements (`uploads.py`, 2) :** `POST /uploads` (public, 150 Mo, 30 / IP / h), `POST /uploads/multi` (public, ≤ 5 fichiers).

**Demande d'estimation (`services.py`, 10) :** `POST /services/estimation` (public, 5 / IP / h), `GET /services/estimation/meta` (public), `POST /services/estimation/plans` (public, PDF 150 Mo), `GET /services/estimation/admin`, `GET /services/estimation/admin/{id}`, `PUT /services/estimation/admin/{id}`, `GET /services/estimation/admin/email-status`, `POST /services/estimation/admin/test-email`, `GET /services/estimation/admin/{id}/plans/{plan_id}`, `POST /services/estimation/admin/{id}/resend-client-email` (les 7 derniers : admin / super_admin).

**Administration (`admin.py`, 5) :** `GET /admin/stats`, `GET /admin/entrepreneurs`, `PUT /admin/entrepreneurs/{id}`, `GET /admin/soumissions`, `PUT /admin/entrepreneurs/{id}/verify-rbq` (admin / super_admin).

**Répertoire RBQ (`repertoire.py`, 3) :** `GET /services/repertoire` (public), `GET /services/repertoire/meta` (public), `POST /services/repertoire/admin/refresh` (**super_admin**).

**Professionnels (`professionnels.py`, 6) :** `POST /professionnels/inscription` (public, 5 / IP / h), `GET /professionnels` (public, sans coordonnées), `GET /professionnels/meta` (public), `POST /professionnels/{id}/contact` (public protégé, 10 / IP / h), `GET /professionnels/admin/queue` (**super_admin**), `POST /professionnels/admin/{id}/moderate` (**super_admin**).

**Ouvriers (`ouvriers.py`, 7) :** `POST /ouvriers/inscription` (public, 5 / IP / h), `GET /ouvriers` (public, sans coordonnées), `GET /ouvriers/meta` (public), `GET /ouvriers/{id}` (public, avec portfolio), `POST /ouvriers/{id}/contact` (public protégé, 10 / IP / h), `GET /ouvriers/admin/queue` (admin / super_admin), `POST /ouvriers/admin/{id}/moderate` (admin / super_admin).

**Assistant IA (`ai.py`, 1) :** `POST /ai/chat` (connecté).

### 4.3 Statuts

| Domaine | Valeurs |
|---|---|
| **Appel d'offres** | nouveau · en_cours · ferme · attribue · annule |
| **Soumission** | envoyee · vue · en_evaluation · acceptee · refusee |
| **Demande d'estimation** | nouvelle · en_analyse · estimation_envoyee · refusee · archivee |
| **Modération (professionnels / ouvriers)** | EN_ATTENTE · APPROUVE · REJETE |
| **Compte entrepreneur** | actif · inactif · suspendu (+ bloqué / banni / rejeté : jeton refusé) |
| **Cautionnement (soumission)** | Chèque certifié · Lettre de garantie bancaire · Cautionnement d'assurance |

### 4.4 Énumérations (listes de choix)

**Types de projet (11)** — `Travaux de construction`, `Rénovation de bâtiments publics`, `Infrastructure routière`, `Aménagement urbain`, `Systèmes informatiques`, `Services professionnels`, `Fournitures et équipements`, `Services d'entretien`, `Travaux d'ingénierie`, `Consultations spécialisées`, `Autre`.

**Tranches de budget (6)** — `Moins de 25 000$`, `25 000$ - 100 000$`, `100 000$ - 500 000$`, `500 000$ - 1 000 000$`, `Plus de 1 000 000$`, `À déterminer selon soumissions`.

**Délais de réalisation (6)** — `Urgent (moins de 1 mois)`, `Court terme (1-3 mois)`, `Moyen terme (3-6 mois)`, `Long terme (6-12 mois)`, `Pluriannuel (plus de 12 mois)`, `Selon calendrier projet`.

**Corps de métier de l'estimation et des ouvriers (21)** — Entrepreneur général · Électricité · Plomberie · CVAC (chauffage / ventilation / climatisation) · Toiture · Charpente / Menuiserie · Finition intérieure · Peinture · Revêtement de sol (céramique, bois, vinyle) · Fenestration / Portes · Isolation · Gypse / Joints · Béton / Fondation · Excavation / Terrassement · Démolition · Maçonnerie / Pierre · Revêtement extérieur · Pavé / Asphalte · Aménagement paysager · Piscine / Spa · Autre.

**Secteurs (4)** — Résidentiel · Commercial · Institutionnel · Industriel.

**Régions** — les filtres proposent les régions administratives du Québec (le filtre des appels d'offres en liste 18 entrées ; les filtres des répertoires professionnels et ouvriers en listent 17). Le classement par région se fait à partir du **code postal**.

### 4.5 Calcul automatique de l'urgence

Le niveau d'urgence d'un appel d'offres est **calculé** à partir du nombre de jours restants avant la **date limite des soumissions** (`leads.py`) :

| Jours restants | Urgence |
|---|---|
| ≤ 3 | **Critique** |
| ≤ 7 | **Élevé** |
| ≤ 14 | **Normal** |
| > 14 | **Faible** |

### 4.6 Argent, taxes et gratuité

- **Plateforme d'appels d'offres = gratuite.** Aucun paiement, aucune passerelle. Les colonnes `abonnement` (défaut « gratuit ») et `credits_restants` (défaut 5) des comptes entrepreneurs existent mais ne sont **jamais décomptées ni exigées** : déposer une soumission ne coûte rien et ne consomme aucun crédit.
- **Service d'estimation = payant, hors ligne.** Paliers **200 / 275 / 350 $**, **facturés à la livraison** de la soumission, par courriel. **Aucun paiement en ligne** dans le code.
- **Assistant IA = gratuit pour l'utilisateur.** Son coût (modèle Claude) est **journalisé pour audit interne** mais **jamais facturé**.
- **Calculateur de taxes (soumission)** : **TPS 5 %** et **TVQ 9,975 %** appliquées au montant, avec récapitulatif HT / TPS / TVQ / TTC. C'est un **outil d'aide à la saisie** pour l'entrepreneur (le montant retenu reste celui qu'il inscrit).

### 4.7 Limites et bornes

| Élément | Limite |
|---|---|
| Mot de passe entrepreneur | **≥ 8 caractères** |
| Description d'appel d'offres | **≥ 20 caractères** |
| Description de soumission | **≥ 50 caractères** |
| Description d'estimation | 10 à 5 000 caractères |
| Message de contact (répertoires) | **≥ 10 caractères** |
| Message du Chat Room | ≤ 5 000 caractères |
| Montant de soumission | > 0 et ≤ 1 000 000 000 $ |
| Documents — appel d'offres | 5 fichiers (PDF / images / Office) |
| Documents — soumission | 3 fichiers |
| Documents — estimation | 10 fichiers combinés (images ≤ 5 Mo, PDF ≤ 150 Mo) |
| Téléversement générique (`/uploads`) | 150 Mo par fichier, magic bytes vérifiés |
| Portfolio d'ouvrier | 4 images |
| Session — entrepreneur / super_admin | 7 jours / 24 heures |
| Session — donneur d'ouvrage | 24 heures ; lien magique valable 30 minutes |
| Session — admin | 2 heures |
| Assistant IA | 20 messages par 10 minutes ; 3 pièces jointes |

### 4.8 Numéros de référence

- **Appel d'offres** : `SEAOP-AAAAMMJJ-XXXXXXXX` (8 caractères hexadécimaux). À conserver : il sert à la connexion du donneur d'ouvrage.
- **Demande d'estimation** : `EST-AAAAMMJJ-XXXXXXXX`.

Ces numéros sont générés de façon **unique et non séquentielle** (aucun doublon possible, aucune fuite du nombre total de projets).

### 4.9 Pièges et éléments à connaître

- **Le donneur d'ouvrage n'a pas de mot de passe** : connexion par lien magique **ou** courriel + numéro de référence. Perdre les deux = ne plus pouvoir se reconnecter (republier au besoin).
- **Une seule soumission par projet**, modifiable seulement si « envoyée » et avant la date limite.
- **On n'évalue qu'une soumission acceptée** ; une évaluation remplace la précédente.
- **Coordonnées masquées** sur les listes publiques et les répertoires professionnels / ouvriers : c'est voulu (Loi 25).
- **L'écran « Demande d'estimation » reste en français** même en anglais (textes en dur).
- **Onglet admin « Professionnels » réservé au super_admin** ; « Ouvriers » ouvert à l'admin aussi.
- **Le montant de soumission est stocké en précision simple (REAL / float32)** : sur de très gros montants, un arrondi minime est théoriquement possible à l'affichage ou au tri. Sans effet pratique aux montants usuels.
- **Tables héritées non utilisées** par l'application (`seaop_attributions`, `seaop_estimations`, anciennes demandes technologue / architecture / ingénieur) : l'attribution passe désormais par les statuts du projet et de la soumission ; un **seul** service d'estimation subsiste.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec le reste de Constructo AI

- **Portail Constructo AI (ERP).** SEAOP est **séparé** de l'ERP. Le seul pont est le lien **« Connexion Constructo AI »** de la barre supérieure, qui **quitte** SEAOP. Les comptes et jetons ne sont **pas** partagés.
- **Données Québec (licences RBQ).** Le répertoire des entrepreneurs est alimenté par un **import** du jeu de données public des licences RBQ (rafraîchi par le super_admin).
- **Courriels (SMTP).** SEAOP envoie 8 types de courriels (confirmation d'appel d'offres, nouvelle soumission, changement de statut, message, addendum, estimation côté admin et côté client, lien magique). L'onglet Administration → Services permet d'en **tester** la configuration.
- **Assistant IA (Claude).** L'assistant de SEAOP est **propre à SEAOP** ; ce n'est pas l'Assistant IA de l'ERP (module 25). Il ne lit que les données SEAOP, avec un périmètre imposé par le serveur.
- **Aucune intégration Stripe, QuickBooks ou comptable.** SEAOP n'encaisse rien et ne synchronise avec aucun logiciel comptable.

### 5.2 Foire aux questions

**SEAOP, est-ce la même chose que l'ERP Constructo AI ?**
Non. C'est une **application séparée** (adresse `/seaop`), avec ses propres comptes et sa propre session. Le lien « Connexion Constructo AI » sert seulement à revenir vers l'ERP.

**Publier un appel d'offres, ça coûte combien ?**
**Rien.** La plateforme d'appels d'offres est entièrement gratuite, pour les donneurs d'ouvrage comme pour les entrepreneurs.

**Alors qu'est-ce qui est payant ?**
Uniquement le **service de demande d'estimation** (200 / 275 / 350 $), et encore : **rien ne se paie en ligne**. On vous facture **à la livraison** de la soumission, par courriel.

**Je suis donneur d'ouvrage : comment je me reconnecte ?**
Par un **lien magique** envoyé à votre courriel, **ou** avec votre **courriel + le numéro de référence** de votre projet (`SEAOP-…`). Il n'y a pas de mot de passe pour ce rôle.

**J'ai perdu mon numéro de référence.**
Demandez un **lien magique** à votre courriel (voie A). Si vous n'avez plus accès au courriel non plus, il faudra republier le projet.

**Un entrepreneur peut-il soumissionner deux fois sur le même projet ?**
Non. Une seule soumission par projet. Vous pouvez la **modifier** tant qu'elle est « envoyée » et que la date limite n'est pas passée.

**Pourquoi je ne vois pas les coordonnées d'un professionnel ou d'un ouvrier ?**
C'est **normal** : elles se révèlent après l'envoi du **formulaire de contact** (protection Loi 25). Votre message lui est relayé et ses coordonnées s'affichent.

**Comment mon entreprise entre-t-elle dans le répertoire des entrepreneurs ?**
Ce répertoire est une **liste officielle importée** des licences RBQ actives : on ne s'y inscrit pas. En revanche, vous pouvez vous **inscrire** au répertoire des **professionnels** ou des **ouvriers** (fiche modérée avant publication).

**Pourquoi ma fiche de répertoire n'apparaît pas tout de suite ?**
Elle part **« En attente »** et n'est publiée qu'après **approbation** par l'administration.

**L'Assistant IA peut-il publier un projet ou soumissionner à ma place ?**
Non. Il **lit** vos données et **rédige des brouillons**, mais ne modifie jamais la base. Vous reprenez son texte dans le bon formulaire.

**Est-ce que je paie pour utiliser l'Assistant IA ?**
Non. Son usage est gratuit (limité à 20 messages par tranche de 10 minutes pour éviter les abus).

**Le service d'estimation prend-il ma carte ?**
Non. Aucun paiement en ligne. Facturation à la livraison, hors plateforme.

**Que se passe-t-il quand j'attribue un contrat ?**
La soumission choisie devient « acceptée », le projet devient « attribué », il **cesse d'accepter des soumissions**, et **toutes les autres sont refusées** — tout le monde est notifié.

**Puis-je évaluer un entrepreneur que je n'ai pas retenu ?**
Non. On n'évalue **que** la soumission **acceptée**, pour protéger les réputations.

**L'écran d'estimation reste en français même quand je passe en anglais. Bogue ?**
Non, limitation connue : cet écran est rédigé en français « en dur ». Sa traduction anglaise est incomplète.

**Je ne vois que l'onglet « Administration » sur la page de connexion.**
La plateforme est en **mode développement** (`DEV_MODE`). En production, les trois onglets publics apparaissent.

### 5.3 Dépannage courant

| Symptôme | Piste |
|---|---|
| Connexion « Donneur d'ouvrage » refusée | Vérifiez le **couple courriel + numéro de référence**, ou passez par le **lien magique**. Le lien magique expire après 30 minutes. |
| Bouton « Soumissionner » absent | Le projet est **attribué, fermé ou annulé** — les dépôts sont clos. |
| Impossible de modifier une soumission | Elle n'est plus « envoyée » (déjà vue / évaluée / attribuée) **ou** la **date limite** est dépassée. |
| « Vous avez déjà soumissionné » | Une seule soumission par projet : **modifiez** l'existante. |
| Coordonnées introuvables dans un répertoire | Normal (Loi 25) : cliquez **« Contacter »** et envoyez le formulaire. |
| Ma fiche de répertoire n'est pas visible | Elle est **« En attente »** de modération. |
| Impossible d'évaluer un entrepreneur | La soumission n'est pas **acceptée**. |
| Aucune notification par courriel | Vérifiez vos indésirables ; côté admin, testez le **SMTP** (Administration → Services). |
| Seul l'onglet « Administration » à la connexion | **Mode développement** actif — attendre la mise en production. |
| Interface d'estimation en français en mode EN | Limitation connue (textes en dur sur cet écran). |

---

## 6. Récapitulatif

- **SEAOP est une plateforme gratuite de mise en relation** entre donneurs d'ouvrage et entrepreneurs de la construction au Québec. Un donneur d'ouvrage publie un appel d'offres, les entrepreneurs soumissionnent, le client compare, échange, **attribue**, puis **évalue**.
- **C'est une application séparée** de l'ERP Constructo AI (adresse `/seaop`, comptes et session propres). Le lien « Connexion Constructo AI » ne fait que **quitter** SEAOP.
- **Quatre rôles** (donneur d'ouvrage, entrepreneur, admin, super_admin) mais **trois onglets de connexion** : le rôle `admin` (mot de passe partagé) est **masqué**. Le **donneur d'ouvrage n'a pas de mot de passe** (lien magique ou courriel + numéro de référence).
- **Trois répertoires distincts des rôles** : entrepreneurs **RBQ** (importés, non inscriptibles), **professionnels** et **ouvriers** (auto-inscription **modérée**). Sur les deux derniers, les **coordonnées sont masquées** et se révèlent après un formulaire de contact (**Loi 25**).
- **Deux modèles économiques** : la plateforme d'appels d'offres est **gratuite** ; le **service d'estimation** est **payant** (200 / 275 / 350 $) mais **sans paiement en ligne** — facturé **à la livraison**, par courriel. Les « crédits » et « abonnements » entrepreneurs ne sont **jamais** facturés.
- **Automatismes clés** : urgence calculée d'après la date limite ; attribution qui **refuse toutes les autres soumissions** et notifie tout le monde ; évaluation **réservée à une soumission acceptée** ; une **seule** soumission par projet.
- **Administration** en 6 onglets (Vue d'ensemble, Entrepreneurs, Soumissions, Services, Ouvriers, Professionnels) ; « Professionnels » et le rafraîchissement RBQ sont **réservés au super_admin**.
- **Assistant IA** (connectés) : **lecture + rédaction de brouillons + Vision**, périmètre imposé par le serveur, **jamais** d'écriture en base, **jamais** facturé.
- **Volumétrie technique** : **14 routeurs**, **77 points d'accès**, tables `public.seaop_*` en base partagée (pas de multi-schéma).

---

*Fichiers sources vérifiés :* backend (`SEAOP_REACT/backend/`) — `seaop_api.py` (429), `seaop_config.py` (175), `seaop_auth.py` (390), `seaop_database.py` (1 904), `seaop_models.py` (389), `seaop_email.py` (408) ; routeurs `routers/` — `auth.py` (529, 9 points), `leads.py` (489, 7), `soumissions.py` (538, 7), `messages.py` (255, 4), `evaluations.py` (162, 3), `notifications.py` (139, 4), `chat_room.py` (392, 9), `uploads.py` (240, 2), `services.py` (975, 10), `admin.py` (153, 5), `repertoire.py` (287, 3), `professionnels.py` (535, 6), `ouvriers.py` (641, 7), `ai.py` (663, 1) ; DDL `modules/seaop/seaop_db_postgres.py` (721). Frontend (`SEAOP_REACT/frontend/src/`) — `App.tsx` (105), pages (`AccueilPage` 161, `NouveauProjetPage` 181, `EspaceEntrepreneurPage` 425, `LeadDetailPage` 754, `MesProjetsPage` 410, `EntrepreneurMessagesPage` 145, `ServiceEstimationPage` 1 292, `RepertoirePage` 292, `ProfessionnelsPage` 793, `OuvriersPage` 896, `AdminPage` 149, `NotificationsPage` 51, `LoginPage` 102), composants (`Sidebar` 210, `TopBar` 215, `LoginForm` 490, `RegisterForm` 545, `ProfileChoice` 120, `LeadForm` 673, `LeadCard` 210, `LeadFilters` 274, `SoumissionForm` 540, `SoumissionCard` 318, `SeaopAiPanel` 382, `ChatRoomPanel` 199, `ChatThread` 237, `EvaluationForm` 86 ; administration `DashboardStats` 227, `EntrepreneurTable` 469, `ServiceTabs` 940, `OuvriersAdminTab` 274, `ProfessionnelsAdminTab` 243, `RepertoireAdminCard` 141, `SoumissionTable` 193), constantes `utils/constants.ts` (301). Montage en production : `ERP_REACT/backend/erp_api.py` (routeurs sous `/api/seaop/v1`, SPA sous `/seaop`).

*Manuels liés :* `25-communication-assistant-ia.md` (l'Assistant IA de l'ERP, à ne pas confondre avec celui de SEAOP), `08-ventes-soumissions.md` (les soumissions internes de l'ERP), `36-programme-portail-b2b.md` (l'autre application autonome servie par l'ERP), `17-terrain-conformite.md` (RBQ, CNESST, cautionnement — contexte de conformité québécois).

*Manuel ERP Constructo AI — Module 37 « SEAOP (appels d'offres publics Québec) » — v1.0 vérifié — 2026-07.*
