# Module 36 — Portail B2B / B2C (clients et fournisseurs)

> **Version** : 1.0 (rédaction initiale vérifiée par rapport au code source, juillet 2026)
> **Public visé** : ce manuel s'adresse au **client** d'un fournisseur (l'entrepreneur ou l'entreprise qui achète, demande des soumissions et suit ses dossiers), et non au fournisseur qui exploite l'ERP. Le **back-office** du fournisseur (approbation des accès, gestion du catalogue, chiffrage des soumissions, changement de statut des commandes) vit ailleurs, dans le module **06 — Ventes (onglet B2B/B2C)**, réservé aux administrateurs.
> **Route frontend** : `/b2b-portal` (application autonome servie **par** l'ERP). Écrans publics `/b2b-portal/login` et `/b2b-portal/register` ; écrans protégés `/b2b-portal/dashboard`, `/catalogue`, `/panier`, `/commandes`, `/suivi`, `/contrats`, `/demandes`, `/messages` (`App.tsx:173-184`). La tuile d'accès, sur la page de connexion de l'ERP, s'appelle **« B2B / C2B — Portail client »** (`LoginPage.tsx:119-128`).
> **Préfixe API** : `/api/erp/v1`. Trois routeurs servent le monde B2B, mais **un seul concerne ce manuel** : le portail client `/b2b-portal` (les deux autres, `/b2b` et `/b2b/ai`, sont côté fournisseur — voir §1.2).
> **Code de référence (backend)** : `backend/routers/b2b_portal.py` (1 321 lignes — **22 points d'accès** côté client) · `backend/routers/auth.py` (4 points d'accès B2B : recherche de fournisseur, connexion, inscription, profil) · gardes d'accès dans `backend/erp_auth.py` (`get_current_b2b_client`, `erp_auth.py:581`). Les tables `b2b_*` sont créées et réparées à la demande par `_ensure_b2b_tables` (`backend/routers/b2b.py:227`).
> **Code de référence (frontend)** : `frontend/src/pages/b2b-portal/` — `B2bLoginPage.tsx` (208 lignes), `B2bRegisterPage.tsx` (391 lignes), `B2bDashboardPage.tsx` (97 lignes), `B2bCataloguePage.tsx` (134 lignes), `B2bPanierPage.tsx` (154 lignes), `B2bCommandesPage.tsx` (119 lignes), `B2bSuiviPage.tsx` (230 lignes), `B2bContratsPage.tsx` (99 lignes), `B2bDemandesPage.tsx` (172 lignes), `B2bMessagesPage.tsx` (152 lignes). Disposition : `components/layout/B2bPortalLayout.tsx` (183 lignes) + `B2bProtectedRoute.tsx` (25 lignes). Clients API : `api/b2b-portal.ts` (277 lignes), `api/b2b-portal-auth.ts` (175 lignes). Magasins d'état : `store/useB2bPortalStore.ts` (252 lignes), `store/useB2bAuthStore.ts` (98 lignes). Traductions : `i18n/locales/{fr,en}/b2b.json` (section `portal`) + `layout.json` (section `b2bPortalLayout`) + `auth.json` (section `b2b`).
> **Tables PostgreSQL (par entreprise fournisseur)** : `b2b_clients`, `b2b_client_users` (comptes de connexion), `b2b_paniers`, `b2b_panier_lignes`, `b2b_commandes`, `b2b_commande_lignes`, `b2b_favoris`, `b2b_demandes`, `b2b_soumissions`, `b2b_contrats`, `b2b_messages`, `b2b_notifications`. Tables partagées lues par le portail : `produits` et `mouvements_stock` (module Magasin), `devis` et `projects` (Suivi), `companies` (fiche client CRM).
> **Cadrage** : le portail est un **espace en ligne que le fournisseur ouvre à ses clients**. Une fois son accès approuvé, le client peut y **parcourir le catalogue** du fournisseur, remplir un **panier**, passer une **commande** (avec TPS et TVQ), **suivre ses commandes**, **suivre en lecture seule ses devis et projets** réels dans l'ERP du fournisseur, consulter ses **contrats**, soumettre des **demandes de soumission** et **échanger des messages** avec le fournisseur. Le portail **ne remplace pas** l'ERP : il en est la vitrine côté client. Il **n'encaisse aucun paiement** en ligne, ne contient **aucun assistant IA**, et ne laisse le client modifier **que** son panier, ses commandes (à la création), ses demandes et ses messages.

*Note de terminologie employée dans ce manuel :* « fournisseur » (ou « entreprise ») désigne l'entreprise qui exploite l'ERP et vous a ouvert un accès ; « client » désigne vous, l'utilisateur du portail ; « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « soumission » (ou « devis ») désigne un prix chiffré par le fournisseur ; « demande » désigne une **demande de soumission** que vous adressez au fournisseur ; « commande » désigne un achat de produits du catalogue.

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

### 1.1 Mission du portail

Le **Portail B2B / B2C** est l'espace en ligne qu'un fournisseur (une entreprise cliente de Constructo AI) met à la disposition de **ses propres clients**. Concrètement, une fois votre accès approuvé, il vous permet de :

- **Parcourir le catalogue** de produits du fournisseur (nom, code, description, catégorie, prix, unité) et marquer vos **favoris** ;
- **Remplir un panier** et **passer une commande** en ligne, avec calcul automatique de la **TPS (5 %)** et de la **TVQ (9,975 %)** ;
- **Suivre vos commandes** (numéro, date, statut, détail des lignes) ;
- **Suivre, en lecture seule, vos devis et vos projets réels** tels qu'ils existent dans l'ERP du fournisseur (à condition que le fournisseur ait relié votre compte à votre fiche — voir §1.5) ;
- **Consulter vos contrats** (montant, montant payé, avancement) ;
- **Soumettre des demandes de soumission** (décrire un projet et recevoir des offres du fournisseur) ;
- **Échanger des messages** avec le fournisseur, un fil de discussion par demande.

Le portail est une **application distincte** de l'ERP. Vous n'avez pas besoin d'un compte ERP : vous vous connectez avec un **compte client** que le fournisseur approuve. Votre session, vos données et votre navigation sont **entièrement séparées** de celles de l'ERP.

### 1.2 Deux mondes séparés : portail client ↔ back-office fournisseur

C'est le point le plus important à comprendre. Autour du même sigle « B2B » cohabitent **deux applications complètement distinctes** :

| | **Portail client** (ce manuel) | **Back-office fournisseur** (module 06) |
|---|---|---|
| Qui l'utilise | **Vous**, le client | Le fournisseur (administrateur ERP) |
| Adresse | `/b2b-portal` | `/ventes?tab=b2b` (onglet B2B/B2C) |
| Type de compte | Compte client (`b2b_client`) | Compte ERP administrateur |
| Ce qu'on y fait | Commander, demander, suivre, discuter | Approuver les accès, gérer le catalogue, **chiffrer** les soumissions, créer les contrats, **faire avancer** les commandes, répondre aux messages |
| Assistant IA | **Aucun** | Oui (« Assistant IA — Gestion B2B », lecture seule, réservé à l'administrateur) |

> **À retenir :** il n'existe **aucun assistant IA dans le portail client**. L'assistant « Gestion B2B » que le fournisseur peut utiliser vit dans le back-office ERP (module 06) et n'est **jamais** accessible depuis votre portail. Aucun bouton, aucune fonction du portail ne consomme de crédits d'intelligence artificielle.

De nombreuses actions du portail sont **volontairement en lecture seule** de votre côté, parce que la décision revient au fournisseur : c'est lui qui **accepte ou refuse** une soumission, qui **fait avancer ou annule** une commande, qui **rédige** un contrat. Vous initiez (une demande, une commande, un message), le fournisseur dispose.

### 1.3 Comment y accéder

Trois chemins mènent au portail :

| Chemin | Détail |
|--------|--------|
| **Tuile sur la page de connexion de l'ERP** | Sur `app.constructoai.ca`, la page de connexion affiche une tuile **« B2B / C2B — Portail client »** (`LoginPage.tsx:119-128`) qui ouvre `/b2b-portal/login`. |
| **Adresse directe** | Tapez directement `app.constructoai.ca/b2b-portal`. La racine `/b2b-portal` vous redirige vers le tableau de bord si vous êtes connecté, sinon vers la page de connexion. |
| **Lien reçu du fournisseur** | Le fournisseur peut simplement vous transmettre l'adresse du portail et le **courriel de contact** à utiliser pour l'identifier (voir §3.1). |

Depuis le portail, la barre supérieure porte en tout temps un lien **« Connexion Constructo AI »** (flèche de retour) qui **quitte** le portail et ramène à la racine de l'ERP (`B2bPortalLayout.tsx:87-94`). C'est le chemin inverse : du portail vers l'ERP.

### 1.4 Rôles, permissions et sécurité

**Un seul type de compte, un seul niveau d'accès.** Contrairement à l'ERP (qui distingue administrateurs, comptables, employés), le portail ne connaît **aucun rôle interne**. Tout compte client actif et approuvé a exactement le même accès. Il n'y a pas d'« administrateur du portail » côté client.

**Chaque compte est cloisonné.** Vous ne voyez **que vos propres** commandes, demandes, contrats, messages et dossiers. Le portail borne systématiquement chaque requête à votre identifiant client et au schéma de votre fournisseur (`get_current_b2b_client`, `erp_auth.py:581` ; filtre `client_company_id` / `client_id` sur toutes les requêtes). Vous ne pouvez jamais consulter les données d'un autre client du même fournisseur, ni celles d'un autre fournisseur.

**Deux mondes étanches.** Un jeton de portail client ne peut **pas** ouvrir l'ERP, et un jeton ERP ne peut **pas** ouvrir le portail : chaque monde refuse le jeton de l'autre (403). Votre session portail utilise son propre jeton (stocké sous `b2b_token`), sa propre instance réseau et sa propre page de connexion.

**Révocation immédiate.** Même si votre session dure plusieurs jours (le jeton est valable 7 jours), le serveur revérifie à **chaque requête** que votre compte et votre fiche client sont toujours actifs. Si le fournisseur désactive votre accès, vous êtes déconnecté dès l'action suivante (401).

**Le portail n'est PAS soumis au « mode consultation ».** Lorsque l'abonnement d'un fournisseur ERP est en souffrance, son ERP peut passer en lecture seule ; **cette restriction ne s'applique pas au portail client** (`erp_auth.py:581` n'applique pas la logique de mode consultation). Votre portail continue de fonctionner normalement (lecture **et** écriture du panier / des commandes).

### 1.5 Le « suivi » de vos dossiers dépend d'un lien posé par le fournisseur

L'onglet **Suivi** (vos devis et projets réels) n'affiche des données **que si** le fournisseur a **relié** votre compte de portail à votre **fiche d'entreprise** dans son CRM. Ce lien est **explicite** et **posé uniquement par le fournisseur** (dans le module 06, onglet Clients). Il n'y a **jamais** d'association automatique par courriel : c'est une décision de sécurité délibérée (le courriel d'inscription n'étant pas vérifié, l'utiliser permettrait d'usurper les devis et projets d'une autre entreprise, `b2b_portal.py:54-88`).

Tant que ce lien n'est pas posé, l'onglet Suivi affiche un bandeau jaune **« Compte non encore relié »** et reste vide. Les autres onglets (catalogue, panier, commandes, demandes, contrats, messages) fonctionnent **sans** ce lien.

### 1.6 Ce que le portail fait — et ne fait PAS

Le portail **fait** : s'inscrire et se connecter par fournisseur, parcourir le catalogue et gérer des favoris, remplir un panier, passer une commande taxée, suivre ses commandes, suivre ses devis et projets réels (si relié), consulter ses contrats, soumettre des demandes de soumission, échanger des messages liés à une demande.

Le portail **ne fait PAS** :

- **Aucun paiement en ligne.** Passer une commande **ne débite aucune carte**. La commande naît « non payée » ; le règlement se fait **hors portail**, selon l'entente avec votre fournisseur (voir §4.4). Il n'y a **ni** Stripe, **ni** passerelle de paiement dans le portail.
- **Aucun assistant IA**, aucune consommation de crédits (voir §1.2).
- **Aucune annulation de commande** de votre côté. Une fois la commande passée, seul le fournisseur peut la faire avancer ou l'annuler. Votre vue des commandes est en **lecture seule**.
- **Aucune acceptation ni refus de soumission** dans le portail. Quand le fournisseur chiffre votre demande, vous **voyez** les soumissions reçues, mais la décision d'accepter/refuser se règle avec lui (côté back-office).
- **Aucune page « Profil » ni « Mon compte ».** Le menu utilisateur ne propose que **« Se déconnecter »**. Vous ne pouvez pas modifier votre profil ni votre mot de passe depuis le portail (des libellés « Profil » / « Paramètres » existent dans les traductions mais ne sont reliés à **aucun** écran — voir §4.6).
- **Aucun export PDF, CSV ni impression** propre au portail. Le seul document consultable est votre **devis officiel**, via un lien externe vers sa page publique (onglet Suivi → Devis, voir §2.8).
- **Aucune pièce jointe** dans la messagerie, **aucun affichage du stock** au catalogue, **aucune pagination** visible du catalogue (voir les limites au §4.5).

### 1.7 Les huit onglets (plus deux écrans publics)

Après connexion, la barre supérieure présente **huit onglets** (`B2bPortalLayout.tsx:21-30`). Deux écrans publics (Connexion, Inscription) précèdent la connexion, hors barre de navigation. Total : **dix écrans**.

| # | Onglet | Route | Icône | Rôle |
|---|--------|-------|-------|------|
| 1 | **Accueil** | `/dashboard` | `LayoutDashboard` | Tableau de bord : 4 indicateurs + 3 actions rapides |
| 2 | **Catalogue** | `/catalogue` | `ShoppingBag` | Produits du fournisseur, recherche, favoris, ajout au panier |
| 3 | **Panier** | `/panier` | `ShoppingCart` | Articles, quantités, taxes, passage de commande (badge du nombre d'articles) |
| 4 | **Commandes** | `/commandes` | `Package` | Historique des commandes + détail |
| 5 | **Suivi** | `/suivi` | `ClipboardList` | Vos **devis** et **projets** réels (lecture seule) |
| 6 | **Contrats** | `/contrats` | `Handshake` | Vos contrats (lecture seule) |
| 7 | **Demandes** | `/demandes` | `FileText` | Vos demandes de soumission + soumissions reçues |
| 8 | **Messages** | `/messages` | `MessageSquare` | Fils de discussion, un par demande |

---

## 2. Interface

### 2.1 Disposition générale (barre supérieure, navigation, pied de page)

Toutes les pages protégées partagent la même **disposition** (`B2bPortalLayout.tsx`). De gauche à droite, la **barre supérieure** (fixée en haut) contient :

- **Bouton hamburger** (sur petit écran, < lg) : déroule les huit onglets ;
- **Lien « Connexion Constructo AI »** (flèche de retour) : quitte le portail vers la racine de l'ERP. Sur écran étroit, seule l'icône reste visible ;
- **Marque « Portail B2B »** accompagnée du **nom de votre entreprise** (sous la marque) ;
- **Navigation** (les huit onglets, sur grand écran) : chaque onglet s'illumine en bleu lorsqu'il est actif. L'onglet **Panier** affiche un **badge** avec le nombre d'articles dès qu'il y en a au moins un ;
- **Menu utilisateur** (à droite) : votre nom et votre courriel, puis une seule option, **« Se déconnecter »**.

En bas de page, un **pied de page** discret affiche « Constructo AI · Portail Client B2B · 2026 ».

> **Thème.** Le portail s'affiche toujours avec le **thème par défaut** de Constructo AI (bleu D365). Il **n'hérite pas** du thème personnalisé que le fournisseur a pu configurer pour son ERP (`B2bPortalLayout.tsx:43`). C'est voulu : le portail est un espace neutre, commun à tous les clients.

### 2.2 Écran public — Connexion (deux étapes)

La connexion (`B2bLoginPage.tsx`) se fait en **deux étapes**, car un même courriel pourrait exister chez plusieurs fournisseurs : il faut d'abord **identifier le fournisseur**.

**Étape 1 — « Connexion Client B2B ».** Un seul champ : **« Courriel du fournisseur »** (exemple d'aide : `info@fournisseur.ca`). Vous saisissez le courriel **de votre fournisseur** (pas le vôtre), puis **« Continuer »** (le bouton affiche « Recherche... » pendant la vérification). Deux liens secondaires : **« Pas de compte? Créer un compte »** (vers l'inscription) et **« Connexion ERP »** (vers la connexion de l'ERP, si vous êtes en réalité un utilisateur ERP).

- Si le fournisseur est introuvable ou inactif, un message le signale (« Fournisseur introuvable. Vérifiez le courriel. »).

**Étape 2 — « Connexion Client ».** Le nom du fournisseur identifié s'affiche en tête (avec une flèche pour revenir en arrière). Deux champs : **« Votre courriel »** (le vôtre, cette fois) et **« Mot de passe »**. Le bouton **« Se connecter »** (« Connexion... » pendant l'envoi) vous mène au tableau de bord en cas de succès.

Sur grand écran, un panneau de gauche présente le portail : titre **« Portail Client »**, texte « Parcourez le catalogue, passez des commandes, soumettez des demandes de soumission et suivez vos projets. », et la mention de version « Constructo AI B2B v1.0 ». Un bandeau d'erreur rouge (avec un bouton de fermeture) apparaît en cas de problème.

> **Sécurité — message d'erreur volontairement vague.** Que le compte n'existe pas ou que le mot de passe soit erroné, le message est le **même** (401 générique). C'est une protection contre l'énumération des comptes : personne ne peut deviner, par la page de connexion, quels courriels sont enregistrés.

### 2.3 Écran public — Inscription (identifier le fournisseur, puis vos informations)

L'inscription (`B2bRegisterPage.tsx`) permet de **demander** un accès. Elle se déroule en **deux étapes de saisie**, suivies d'un **écran de confirmation**.

**Étape 1 — « Créer un compte client » (Identifier votre fournisseur).** Champ **« Courriel du fournisseur »**, avec l'aide « Demandez ce courriel à votre contact chez le fournisseur. ». Bouton **« Continuer »** (désactivé tant que le champ est vide). Lien « Déjà un compte? Se connecter ». (L'écran est libellé « Étape 1 sur 2 ».)

**Étape 2 — « Vos informations ».** Un formulaire :

| Champ | Obligatoire | Détail |
|-------|:-----------:|--------|
| **Nom de votre entreprise** | Oui | Ex. « Acme Construction Inc. » |
| **Votre nom complet** | Oui | Ex. « Jean Dupont » |
| **Votre courriel** | Oui | Votre adresse de connexion |
| **Téléphone** | Non | Ex. « (514) 555-0100 » |
| **Ville** | Non | Ex. « Montréal » |
| **Code postal** | Non | Ex. « H1A 1A1 » |
| **Mot de passe** | Oui | **Minimum 6 caractères** |
| **Confirmer le mot de passe** | Oui | Doit correspondre au précédent |

Bouton **« Créer mon compte »** (« Création en cours... » pendant l'envoi). Le portail vérifie côté client que le mot de passe fait au moins 6 caractères et que la confirmation correspond, avant même d'envoyer.

> **La province est fixée à « Québec ».** Le formulaire ne propose pas de choisir la province : elle est enregistrée à « Quebec » (`B2bRegisterPage.tsx:47`). Si vous êtes hors Québec, signalez-le à votre fournisseur, qui ajustera votre fiche de son côté.

**Écran de confirmation — « Demande envoyée! ».** Une icône de succès, puis « Votre demande d'accès a été envoyée à **{fournisseur}**. Vous recevrez un accès dès que le fournisseur aura approuvé votre demande. » et « Vous pouvez fermer cette page. Essayez de vous connecter plus tard. ». Bouton **« Retour à la connexion »**.

> **Votre compte est créé INACTIF.** À l'inscription, le compte est enregistré **désactivé** : vous **ne recevez pas** de session, et vous **ne pouvez pas** vous connecter tant que le fournisseur n'a pas **approuvé** votre demande (dans son back-office, module 06 → B2B → Demandes d'accès). Le fournisseur reçoit une notification. Il n'y a **pas** de courriel de confirmation à cliquer : l'activation est **manuelle**, faite par une personne chez le fournisseur.

### 2.4 Accueil / Tableau de bord

La page d'accueil (`B2bDashboardPage.tsx`) vous accueille par **« Bienvenue, {votre nom} »** suivi du nom de votre entreprise, puis présente **quatre indicateurs** cliquables et **trois actions rapides**.

**Les quatre indicateurs** (chacun mène à l'onglet correspondant) :

| Carte | Ce qu'elle compte réellement | Mène à |
|-------|------------------------------|--------|
| **Commandes en cours** | Vos commandes dont le statut n'est **ni Livrée ni Annulée** | Commandes |
| **Demandes en attente** | Vos demandes au statut **Nouvelle** ou **En cours** | Demandes |
| **Contrats actifs** | Vos contrats au statut **Actif** ou **En cours** | Contrats |
| **Messages non lus** | Les messages **écrits par le fournisseur** que vous n'avez pas encore lus | Messages |

Tant que les données ne sont pas chargées, chaque carte affiche `--`.

> **Nuance de libellé.** La carte **« Demandes en attente »** compte en réalité les demandes **« Nouvelle » ou « En cours »** (le compteur `demandes_en_cours` du serveur, `b2b_portal.py:180`). Le libellé parle d'« attente », la donnée parle d'« en cours » : c'est le **même** ensemble de demandes ouvertes (celles qui ne sont ni acceptées, ni refusées, ni annulées). Ne cherchez pas une distinction entre les deux mots.

**Les trois actions rapides** : **« Catalogue »** (parcourir les produits), **« Nouvelle demande »** (soumettre une demande de soumission), **« Nouveau message »** (aller à la messagerie).

### 2.5 Catalogue

Le catalogue (`B2bCataloguePage.tsx`) présente les **produits du fournisseur** sous forme de grille de cartes.

**Filtres, en haut :**

- **Champ de recherche** (icône loupe, « Rechercher un produit... ») : filtre sur le **nom**, le **code produit** ou la **description**. La recherche se déclenche **après une courte pause** de frappe (400 ms) pour ne pas surcharger le serveur ;
- **Menu déroulant de catégorie** : « Toutes les catégories » par défaut, ou l'une des catégories réellement présentes chez le fournisseur.

**Chaque carte produit** affiche :

- le **nom** du produit ;
- son **code produit** (s'il existe) ;
- un **bouton cœur** (favori) : cliquez pour l'ajouter à vos favoris (le cœur devient rouge) ou le retirer (il redevient gris) ;
- une **description** (deux lignes au maximum) ;
- un **badge de catégorie** ;
- le **prix unitaire** (ex. « 12,50 $ », ou « -- » si aucun prix n'est fixé) et l'**unité** ;
- un bouton **« Ajouter au panier »** : au clic, il devient vert pendant environ une seconde et demie pour confirmer l'ajout. Une protection empêche l'empilement d'ajouts en cas de double-clic.

Un indicateur d'attente s'affiche pendant le chargement ; « Aucun produit » si le catalogue est vide ou si aucun produit ne correspond au filtre ; un bandeau rouge en cas d'erreur.

> **Ce que le catalogue n'affiche PAS.** Le **stock disponible** n'est **pas montré** au client (l'information circule côté serveur mais reste masquée dans la carte). Il n'y a **pas** de pagination visible : la page présente la première série de produits (jusqu'à 20). Si le catalogue est très volumineux, affinez votre recherche ou votre catégorie pour retrouver un produit précis. Enfin, les **favoris** ne disposent **pas** d'une page dédiée : le cœur ne sert qu'à colorer le produit dans le catalogue (voir §4.6).

### 2.6 Panier et commande

Le panier (`B2bPanierPage.tsx`) rassemble les produits choisis et sert à **passer la commande**.

**Panier vide :** message « Votre panier est vide » et bouton « Parcourir le catalogue ».

**Colonne de gauche — les articles.** Pour chaque ligne :

- le **nom** du produit (ou « Produit #{identifiant} » si le nom n'est plus disponible) et son **code** ;
- le **prix unitaire** et l'**unité** ;
- des contrôles de **quantité** **−** / **+** (le « − » est désactivé quand la quantité atteint 1 ; une protection évite l'empilement d'actions rapides sur un même article) ;
- le **sous-total de la ligne** ;
- une **corbeille** pour retirer l'article.

**Colonne de droite — le résumé :**

| Ligne | Calcul |
|-------|--------|
| **Sous-total** | Somme des (prix × quantité) |
| **TPS (5%)** | 5 % du sous-total |
| **TVQ (9.975%)** | 9,975 % du sous-total |
| **Total TTC** | Sous-total + TPS + TVQ |

Un bouton **« Commander »** déplie le **formulaire de commande** : **Adresse de livraison**, **Ville**, **Code postal**, **Notes (optionnel)**, puis **« Confirmer la commande »** (vert, « Traitement... » pendant l'envoi) et **« Annuler »**.

**Après confirmation**, un **écran plein écran** confirme : icône verte, « Commande confirmée! », le **numéro** de commande et le **total**, puis un bouton « Retour au panier ».

> **Ce qui se passe à la confirmation (côté serveur).** Le portail vérifie que **le stock est suffisant** pour chaque produit avant de créer la commande ; si un produit manque, la commande est **refusée** avec un message « Stock insuffisant pour : ... » et **rien** n'est prélevé. Si le stock est suffisant, la commande est créée, le **stock est décrémenté** et un mouvement de stock est enregistré pour la traçabilité. Deux confirmations simultanées (deux onglets, double-clic) sont **sérialisées** : la seconde échoue proprement, sans créer de commande en double (`b2b_portal.py:474-694`). **Aucun montant n'est débité** : la commande est créée « non payée » (voir §4.4).

### 2.7 Commandes

L'onglet Commandes (`B2bCommandesPage.tsx`) liste vos commandes, de la plus récente à la plus ancienne. Vide : « Aucune commande ».

**La liste** présente, par ligne : le **numéro**, la **date**, un **badge de statut** coloré et le **total TTC** (ce dernier à partir des écrans moyens).

**Les statuts d'une commande :**

| Statut | Signification |
|--------|---------------|
| **En attente** | Commande reçue, pas encore confirmée par le fournisseur |
| **Confirmée** | Le fournisseur a confirmé la commande |
| **En préparation** | La commande est en cours de préparation |
| **Expédiée** | La commande a quitté le fournisseur |
| **Livrée** | La commande vous a été livrée |
| **Annulée** | La commande a été annulée |

**Le détail** (au clic sur une commande) affiche : un bouton « Retour aux commandes », l'en-tête (numéro + date + badge de statut), un **tableau des lignes** (colonnes **Produit / Qté / Prix unit. / Total**), puis **Sous-total / TPS (5%) / TVQ (9.975%) / Total TTC**.

> **La vue est en lecture seule.** Vous ne pouvez **pas** annuler ni modifier une commande depuis le portail. La progression (Confirmée → En préparation → Expédiée → Livrée) et l'annulation éventuelle sont pilotées par le fournisseur dans son back-office. Pour tout changement, contactez-le (au besoin par la messagerie, §2.11).

### 2.8 Suivi (Devis / Projets)

L'onglet Suivi (`B2bSuiviPage.tsx`) affiche **vos dossiers réels dans l'ERP du fournisseur** : « Suivi de mes dossiers » / « Suivez l'état de vos soumissions et l'avancement de vos projets. ». **Tout y est en lecture seule.**

> **Bandeau « Compte non encore relié ».** Si le fournisseur n'a pas encore relié votre compte à votre fiche d'entreprise (voir §1.5), un bandeau jaune s'affiche : « Vos dossiers ne sont pas encore reliés à votre fiche dans l'entreprise. Communiquez avec l'entreprise pour activer le suivi. » Les deux onglets ci-dessous restent alors vides.

**Deux sous-onglets :**

**« Devis ».** Vos **soumissions réelles** (celles que le fournisseur vous a préparées dans son module Soumissions). Chaque carte montre : le **nom du projet** ou le **numéro de devis**, la **date**, un **badge de statut** (Envoyée, En attente, En révision, Acceptée, Refusée, Expirée, Terminée…) et le **montant**. Si un devis dispose d'un **lien public**, la carte devient un **lien externe** (icône de lien) vers le **document officiel** (page publique `/devis/public/{jeton}`) : c'est le seul document consultable/imprimable depuis le portail. Les **brouillons** du fournisseur ne sont **jamais** affichés (prix encore confidentiels).

**« Projets ».** Vos **projets réels**. Chaque carte montre : le **nom** ou **numéro** du projet, la **ville** du chantier, un **badge de statut**, une **barre d'avancement** en pourcentage et, si disponibles, les dates **Début** / **Fin prévue** et l'adresse du **Chantier**.

> **Le budget interne n'est JAMAIS montré.** La vue Projets expose l'avancement, les dates et le chantier, mais **pas** le budget interne du projet (`b2b_portal.py:838`). C'est une donnée de gestion du fournisseur, hors de votre portail.

### 2.9 Contrats

L'onglet Contrats (`B2bContratsPage.tsx`) liste vos contrats. Vide : « Aucun contrat ». **Lecture seule.**

**La liste** : titre / numéro, date, **badge de statut** et montant. **Statuts** possibles : **Brouillon**, **Actif**, **En cours**, **Terminé**, **Annulé**, **Suspendu**.

**Le détail** (au clic) : titre + numéro de contrat, badge de statut, **Montant**, **Montant payé**, **Date de début**, **Date de fin prévue** et une **barre d'avancement** en pourcentage. Aucune action n'est possible : un contrat est **créé par le fournisseur** (généralement lorsqu'il **accepte** l'une de vos soumissions).

### 2.10 Demandes

L'onglet Demandes (`B2bDemandesPage.tsx`) est l'endroit où vous **demandez une soumission** au fournisseur. Titre « Mes demandes », avec un bouton **« Nouvelle demande »**.

**Le formulaire de nouvelle demande** (« Nouvelle demande de soumission ») :

| Champ | Obligatoire | Détail |
|-------|:-----------:|--------|
| **Titre du projet** | Oui | Résumé court du besoin |
| **Description détaillée** | Non | Le détail de ce que vous voulez faire chiffrer |
| **Catégorie (ex: rénovation)** | Non | Type de travaux |
| **Budget estimé ($)** | Non | Montant indicatif |
| **Date limite** | Non | Échéance souhaitée |
| **Priorité** | Non | **Basse**, **Normale**, **Haute** ou **Urgente** (Normale par défaut) |

Boutons **« Soumettre »** (« Envoi... » pendant l'envoi) et **« Annuler »**. Un message de succès confirme : « Demande soumise avec succès ».

**La liste des demandes** : titre, **nombre de soumissions reçues** et date, **badge de statut** (Nouvelle, En cours, Soumise, Acceptée, Refusée, Annulée…). Vide : « Aucune demande » et « Soumettez votre première demande de soumission ».

**Le détail** (au clic) : titre + badge de statut, description, puis **Catégorie / Budget / Priorité / Date limite**, et enfin la section **« Soumissions reçues (N) »**. Chaque soumission reçue montre : son **montant**, un **badge de statut** (Brouillon, Soumise, En évaluation, Acceptée, Refusée, Expirée), sa **description** et le **délai en jours**. S'il n'y en a pas encore : « Aucune soumission reçue pour le moment. »

> **Vous ne décidez pas dans le portail.** Vous **voyez** les soumissions que le fournisseur vous adresse, mais vous ne pouvez **pas** les accepter ni les refuser depuis le portail : la décision se prend avec le fournisseur (côté back-office). De plus, les **brouillons** de soumission du fournisseur ne vous sont **pas** montrés (prix en cours de préparation). Enfin, le champ **adresse/ville du chantier** n'est **pas** proposé dans ce formulaire (voir §4.6) : précisez le lieu dans la **description** si nécessaire.

### 2.11 Messages

L'onglet Messages (`B2bMessagesPage.tsx`) est votre **fil de discussion** avec le fournisseur. Particularité importante : **chaque message est rattaché à une demande**. Il n'existe pas de message « libre » sans demande.

**Colonne de gauche — « Mes demandes ».** La liste de vos demandes, qui sert de **liste de conversations**. Si vous n'en avez aucune : « Aucune demande. Créez-en une pour discuter avec votre fournisseur. »

**Colonne de droite — la conversation.** Tant qu'aucune demande n'est choisie : « Sélectionnez une demande pour voir la conversation. » Une fois une demande choisie, les messages s'affichent en bulles : **« Vous »** (à droite) et **« Fournisseur »** (à gauche), avec l'horodatage. Si le fil est vide : « Aucun message. Démarrez la conversation. »

**Zone de saisie.** Un champ « Votre message... » et un bouton **« Envoyer »** (désactivé si le champ est vide). À l'ouverture d'un fil, les messages du fournisseur sont automatiquement **marqués comme lus** (ce qui fait baisser le compteur « Messages non lus » du tableau de bord).

> **Limites de la messagerie.** Pas de **pièce jointe**, pas de **sujet** modifiable, et la conversation est toujours **rattachée à une demande** (les messages liés à un contrat sont gérés par le serveur mais aucun écran du portail n'en crée). Pour transmettre un document, utilisez un autre canal convenu avec votre fournisseur.

---

## 3. Workflows pas à pas

### 3.1 Obtenir un accès au portail (première fois)

1. Demandez à votre contact chez le fournisseur **le courriel** qui identifie son entreprise dans Constructo AI (ex. `info@fournisseur.ca`) et l'adresse du portail (`app.constructoai.ca/b2b-portal`).
2. Ouvrez `/b2b-portal/register`. **Étape 1 :** saisissez le **courriel du fournisseur**, puis « Continuer ».
3. **Étape 2 :** remplissez vos informations (nom d'entreprise, votre nom, votre courriel, mot de passe d'au moins 6 caractères, et éventuellement téléphone/ville/code postal), puis **« Créer mon compte »**.
4. L'écran « Demande envoyée! » confirme l'envoi. **Votre compte est inactif** : attendez que le fournisseur **approuve** votre demande.
5. Une fois averti de l'approbation, revenez sur `/b2b-portal/login`, identifiez le fournisseur (Étape 1), puis connectez-vous avec **votre** courriel et votre mot de passe (Étape 2).

> Si la connexion échoue juste après l'inscription, c'est probablement que l'approbation n'est pas encore faite : réessayez plus tard ou relancez votre contact.

### 3.2 Passer une première commande

1. Onglet **Catalogue**. Cherchez un produit (barre de recherche) ou filtrez par catégorie.
2. Sur chaque produit voulu, cliquez **« Ajouter au panier »** (le bouton devient vert un instant). Répétez pour les autres produits.
3. Onglet **Panier** (le badge indique le nombre d'articles). Ajustez les **quantités** avec **−** / **+**, retirez au besoin un article (corbeille).
4. Vérifiez le **résumé** : Sous-total, TPS (5 %), TVQ (9,975 %), Total TTC.
5. Cliquez **« Commander »**, remplissez **Adresse de livraison**, **Ville**, **Code postal** et, au besoin, des **Notes**.
6. Cliquez **« Confirmer la commande »**. L'écran « Commande confirmée! » affiche le **numéro** et le **total**.
7. Retrouvez la commande dans l'onglet **Commandes**. Le fournisseur la fera ensuite avancer (Confirmée → … → Livrée).

> **Aucun paiement n'est demandé à cette étape.** Le règlement se fait selon votre entente avec le fournisseur, en dehors du portail.

### 3.3 Suivre une commande

1. Onglet **Commandes**. Repérez la commande par son **numéro**, sa **date** ou son **badge de statut**.
2. Cliquez la ligne pour ouvrir le **détail** : lignes de produits (Qté, Prix unit., Total) et récapitulatif taxé.
3. Le **statut** vous renseigne sur l'avancement (En attente, Confirmée, En préparation, Expédiée, Livrée). Vous ne pouvez pas l'annuler vous-même : contactez le fournisseur au besoin.

### 3.4 Demander une soumission et échanger des messages

1. Onglet **Demandes** → **« Nouvelle demande »**.
2. Renseignez au minimum le **Titre du projet** ; ajoutez description, catégorie, budget, date limite et priorité selon vos besoins. Précisez le **lieu du chantier dans la description** si nécessaire (le champ dédié n'est pas dans ce formulaire).
3. **« Soumettre »**. La demande apparaît dans la liste (statut « Nouvelle »).
4. Onglet **Messages** : sélectionnez votre demande à gauche, écrivez au fournisseur pour préciser le besoin, puis **« Envoyer »**.
5. Quand le fournisseur chiffre, revenez au **détail de la demande** : la section **« Soumissions reçues »** affiche montant, statut, description et délai. Discutez de l'acceptation directement avec le fournisseur.

### 3.5 Suivre vos devis et projets réels

1. Onglet **Suivi**. Si un bandeau jaune **« Compte non encore relié »** s'affiche, demandez au fournisseur de **relier** votre compte à votre fiche d'entreprise (voir §3.7). Sans ce lien, rien ne s'affiche.
2. Une fois relié, sous-onglet **« Devis »** : consultez vos soumissions réelles (statut, montant). Si une carte est un **lien** (icône), cliquez pour ouvrir le **document officiel** (page publique) — vous pourrez le consulter, l'imprimer ou le télécharger depuis cette page externe.
3. Sous-onglet **« Projets »** : suivez l'**avancement** (barre de pourcentage), les **dates** et le **chantier** de vos projets.

### 3.6 Consulter un contrat

1. Onglet **Contrats**. Repérez le contrat par son titre/numéro et son **badge de statut**.
2. Cliquez pour voir le **détail** : Montant, Montant payé, Dates de début et de fin prévue, avancement. Aucune action : le contrat est géré par le fournisseur.

### 3.7 (Pour le fournisseur) Rendre le suivi visible à un client

Ce workflow s'exécute **côté fournisseur** (module 06 — Ventes → onglet B2B/B2C), pas dans le portail client. Il est décrit ici pour que le client sache quoi demander :

1. Dans le back-office B2B, ouvrir la fiche du **client** (`b2b_clients`).
2. **Relier** ce client à l'**entreprise CRM** correspondante (`companies`) : c'est ce lien explicite (`company_id`) qui active le Suivi côté portail.
3. Ne **jamais** compter sur une association automatique par courriel : elle n'existe pas (décision de sécurité). Le lien doit être **posé manuellement** par le fournisseur.

Dès ce lien posé, les devis (hors brouillon) et les projets de cette entreprise apparaissent dans l'onglet Suivi du client.

---

## 4. Référence

### 4.1 Écrans du portail

| Écran | Fichier | Accès | Lecture/Écriture |
|-------|---------|-------|------------------|
| Connexion (2 étapes) | `B2bLoginPage.tsx` | Public | — |
| Inscription (2 étapes + confirmation) | `B2bRegisterPage.tsx` | Public | Crée un compte **inactif** |
| Accueil / Tableau de bord | `B2bDashboardPage.tsx` | Protégé | Lecture |
| Catalogue | `B2bCataloguePage.tsx` | Protégé | Lecture + favoris + ajout au panier |
| Panier & commande | `B2bPanierPage.tsx` | Protégé | Écriture (panier, création de commande) |
| Commandes | `B2bCommandesPage.tsx` | Protégé | Lecture seule |
| Suivi (Devis / Projets) | `B2bSuiviPage.tsx` | Protégé | Lecture seule |
| Contrats | `B2bContratsPage.tsx` | Protégé | Lecture seule |
| Demandes | `B2bDemandesPage.tsx` | Protégé | Lecture + création de demande |
| Messages | `B2bMessagesPage.tsx` | Protégé | Lecture + envoi de message |

Les écrans protégés passent par `B2bProtectedRoute.tsx` : sans session valide, redirection vers `/b2b-portal/login`.

### 4.2 Points d'accès de l'API (26 au total)

Tous préfixés par `/api/erp/v1`. Les 22 points du portail exigent une session client (`get_current_b2b_client`) ; les 4 points d'authentification sont publics (sauf le profil).

**Authentification (`routers/auth.py`) :**

| Méthode | Chemin | Accès | Rôle |
|---------|--------|-------|------|
| POST | `/auth/b2b-tenant-lookup` | Public | Étape 1 : courriel du fournisseur → nom + identifiant de schéma. Refuse une entreprise inactive. |
| POST | `/auth/b2b-client-login` | Public | Étape 2 : courriel + mot de passe + fournisseur → session (jeton 7 jours). |
| POST | `/auth/b2b-client-register` | Public | Inscription : crée un compte **inactif** en attente d'approbation. |
| GET | `/auth/b2b-me` | Client | Profil du client connecté. |

**Portail (`routers/b2b_portal.py`, 22 points) :**

| # | Méthode | Chemin | Rôle |
|---|---------|--------|------|
| 1 | GET | `/b2b-portal/dashboard` | Les 4 indicateurs du tableau de bord |
| 2 | GET | `/b2b-portal/catalogue` | Produits (recherche, catégorie, pagination serveur 20/page) |
| 3 | GET | `/b2b-portal/panier` | Le panier actif (avec taxes et nombre d'articles) |
| 4 | POST | `/b2b-portal/panier/items` | Ajouter un produit au panier |
| 5 | PUT | `/b2b-portal/panier/items/{id}` | Changer la quantité (0 = retirer) |
| 6 | DELETE | `/b2b-portal/panier/items/{id}` | Retirer un article |
| 7 | POST | `/b2b-portal/panier/commander` | **Passer la commande** (vérif stock, décrément, taxes) |
| 8 | GET | `/b2b-portal/commandes` | Liste des commandes |
| 9 | GET | `/b2b-portal/commandes/{id}` | Détail d'une commande |
| 10 | GET | `/b2b-portal/suivi/soumissions` | Vos devis réels (hors brouillon) |
| 11 | GET | `/b2b-portal/suivi/projets` | Vos projets réels (sans le budget interne) |
| 12 | GET | `/b2b-portal/demandes` | Liste des demandes + nombre de soumissions |
| 13 | POST | `/b2b-portal/demandes` | Créer une demande de soumission |
| 14 | GET | `/b2b-portal/demandes/{id}` | Détail + soumissions reçues (hors brouillon) |
| 15 | GET | `/b2b-portal/contrats` | Liste des contrats |
| 16 | GET | `/b2b-portal/contrats/{id}` | Détail d'un contrat |
| 17 | GET | `/b2b-portal/messages` | Fils de discussion (par demande ou contrat) |
| 18 | POST | `/b2b-portal/messages` | Envoyer un message (rattaché à une demande/contrat) |
| 19 | PUT | `/b2b-portal/messages/read` | Marquer les messages du fournisseur comme lus |
| 20 | GET | `/b2b-portal/favoris` | Liste des favoris |
| 21 | POST | `/b2b-portal/favoris/{produit_id}` | Ajouter un favori (idempotent) |
| 22 | DELETE | `/b2b-portal/favoris/{produit_id}` | Retirer un favori |

### 4.3 Statuts

| Domaine | Valeurs (affichage) |
|---------|---------------------|
| **Commande** | En attente · Confirmée · En préparation · Expédiée · Livrée · Annulée |
| **Paiement de commande** | Non payé · Payé · Remboursé · En attente (une commande naît **Non payé**) |
| **Demande** | Nouvelle · En cours · Soumise · Acceptée · Refusée · Annulée |
| **Soumission reçue** | Brouillon *(jamais montré)* · Soumise · En évaluation · Acceptée · Refusée · Expirée |
| **Contrat** | Brouillon · Actif · En cours · Terminé · Annulé · Suspendu |
| **Devis (Suivi)** | Envoyée · En attente · En révision · Acceptée · Approuvée · Gagnée · Refusée · Expirée · Terminée *(brouillon exclu)* |
| **Projet (Suivi)** | En attente · À faire · Planifié · En cours · Terminé · Complété · Annulé · Suspendu · En pause |

### 4.4 Calculs et argent

- **Taxes.** TPS = **5 %** et TVQ = **9,975 %**, appliquées au **sous-total** du panier et de la commande (`b2b_portal.py:23-24`). À l'écran, les libellés exacts sont « TPS (5%) » et « TVQ (9.975%) ».
- **Sous-total** = somme des (prix unitaire × quantité) de chaque ligne. **Total TTC** = Sous-total + TPS + TVQ.
- **Prix figé à l'ajout.** Le prix d'un article est celui du produit **au moment où vous l'ajoutez au panier**. Si le fournisseur change son prix ensuite, votre panier conserve le prix initial.
- **Aucun règlement en ligne.** Passer une commande **ne débite aucune carte** et ne fait appel à **aucune** passerelle de paiement. La commande est créée avec le statut de paiement **« Non payé »**. Le seul effet concret côté fournisseur est le **décrément du stock** (`produits.stock_disponible`) et l'écriture d'un **mouvement de stock** de type Sortie, pour la traçabilité. Le paiement se règle hors portail, selon votre entente.
- **Vérification de stock au moment de commander.** Si un produit du panier n'a plus assez de stock, la commande entière est **refusée** (« Stock insuffisant pour : ... ») et rien n'est prélevé. Ajustez les quantités ou retirez l'article, puis réessayez.
- **Numéro de commande** : de la forme **`CMD-AAAAMMJJ-NNNN`** (dérivé de l'identifiant réel de la commande, jamais d'un simple compteur — pas de doublon possible).

### 4.5 Limites et bornes

| Élément | Limite |
|---------|--------|
| Longueur du mot de passe (inscription) | **6 caractères minimum** |
| Quantité par article (ajout / modification) | de **1** à **1 000 000** |
| Catalogue — résultats par page (serveur) | 20 (la pagination n'est **pas** exposée dans l'interface) |
| Messages retournés par fil | 100 au maximum |
| Adresse de livraison / Notes de commande | bornées (respectivement 500 et 5 000 caractères) |
| Durée de la session (jeton) | 7 jours (révocable immédiatement par le fournisseur) |

### 4.6 Pièges et éléments non exposés (à connaître)

- **Aucune page « Profil / Mon compte ».** Des libellés « Profil », « Paramètres » et « Mon compte » existent dans les traductions mais ne sont reliés à **aucun** écran. Le menu utilisateur ne propose que **« Se déconnecter »**. On ne peut pas changer son profil ni son mot de passe depuis le portail.
- **Favoris sans page dédiée.** Les favoris se posent au catalogue (bouton cœur) mais il n'existe **pas** de vue « Mes favoris » : le favori ne sert qu'à colorer le produit.
- **Champs présents dans l'API mais absents de l'écran.** La création de demande accepte techniquement une **adresse/ville de chantier**, mais le formulaire ne les propose pas (mettez le lieu dans la description). L'envoi de message accepte techniquement un **sujet** et un rattachement à un **contrat**, non exposés dans l'écran (qui ne gère que le fil par demande). Le catalogue reçoit techniquement le **stock disponible**, non affiché.
- **Ambiguïté technique du nom `client_company_id`** *(pour administrateurs avertis)*. Dans les tables de **panier, commandes, contrats et favoris**, la colonne `client_company_id` contient en réalité l'identifiant du **client** (`b2b_clients.id`), **pas** celui de l'entreprise CRM. En revanche, dans les tables **devis** et **projects** (Suivi), le champ `client_company_id` contient bien l'identifiant de l'**entreprise CRM** (`companies.id`), résolu à partir du lien `b2b_clients.company_id`. Deux sens pour un même nom : à ne pas confondre lors d'une lecture directe de la base.
- **Tables partagées avec l'ancienne application.** Les tables `b2b_*` sont communes à l'ERP React et à l'ancienne application Streamlit du fournisseur. Le serveur répare et harmonise leurs contraintes à la demande (`_ensure_b2b_tables`, `b2b.py:227`) pour que les deux mondes coexistent (casse des statuts, clés primaires héritées). C'est transparent pour vous.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

- **Ventes / Back-office B2B (module 06).** C'est le **pendant fournisseur** du portail. Le fournisseur y **approuve** votre accès, **relie** votre compte à votre fiche d'entreprise (active le Suivi), **chiffre** vos demandes en soumissions, **crée** vos contrats, **fait avancer** vos commandes et **répond** à vos messages. L'« Assistant IA — Gestion B2B » y vit aussi, du côté du fournisseur uniquement.
- **Entreprises (module 04).** Votre **fiche d'entreprise** dans le CRM du fournisseur (`companies`) est ce à quoi votre compte de portail doit être **relié** pour que le Suivi montre vos devis et projets.
- **Soumissions / Devis (module 08).** Les **devis réels** visibles dans Suivi → Devis proviennent de ce module. Le **lien externe** d'une carte ouvre la **page publique** du devis (`/devis/public/{jeton}`), seul document consultable et imprimable depuis le portail.
- **Projets (module 09).** Les **projets réels** visibles dans Suivi → Projets proviennent de ce module (avancement, dates, chantier — sans le budget).
- **Magasin / Inventaire (module 10).** Le **catalogue** du portail lit les **produits** de ce module (nom, code, prix, catégorie). Passer une commande **décrémente le stock** et écrit un **mouvement de stock** ici.
- **Comptabilité (module 15).** Le portail **ne génère aucune facture** : les commandes naissent « non payées ». La facturation et l'encaissement se gèrent côté fournisseur.
- **Configuration (module 28).** L'abonnement du fournisseur y est géré. Bon à savoir : **le portail n'est pas affecté** par le mode consultation (lecture seule) que la Configuration peut imposer à l'ERP du fournisseur.

### 5.2 Foire aux questions

**Faut-il un compte ERP pour utiliser le portail ?**
Non. Vous utilisez un **compte client** distinct. Vous vous inscrivez, puis le fournisseur **approuve** votre accès.

**Pourquoi dois-je entrer le courriel du fournisseur avant le mien ?**
Parce qu'un même courriel pourrait exister chez plusieurs fournisseurs. La première étape **identifie l'entreprise** ; la seconde vous connecte.

**Je viens de m'inscrire et je ne peux pas me connecter. Pourquoi ?**
Votre compte est créé **inactif**. Vous ne pourrez vous connecter qu'après **approbation** par le fournisseur (action manuelle de son côté). Il n'y a pas de courriel de confirmation à cliquer.

**Passer une commande me fait-il payer en ligne ?**
Non. **Aucun paiement** n'est pris dans le portail. La commande est créée « non payée » ; vous réglez selon votre entente avec le fournisseur. Le seul effet immédiat est la **réservation du stock** (décrément).

**Puis-je annuler ou modifier une commande déjà passée ?**
Non, pas depuis le portail. Contactez le fournisseur : lui seul peut faire avancer ou annuler une commande.

**Puis-je accepter ou refuser une soumission dans le portail ?**
Non. Vous **voyez** les soumissions reçues, mais la décision se prend avec le fournisseur (côté back-office).

**Mon onglet Suivi affiche « Compte non encore relié ». Que faire ?**
Demandez au fournisseur de **relier** votre compte à votre fiche d'entreprise dans son CRM. Ce lien est **manuel** et **obligatoire** ; sans lui, aucun devis ni projet ne s'affiche.

**Où sont mes favoris ?**
Il n'y a **pas** de page « Favoris ». Le bouton cœur colore simplement le produit dans le catalogue.

**Pourquoi ne vois-je pas le stock disponible d'un produit ?**
Le stock n'est **pas** affiché aux clients. Le contrôle se fait au moment de commander : si le stock manque, la commande est refusée avec un message clair.

**Le catalogue est immense : où est la pagination ?**
L'interface montre la première série de produits (20). Il n'y a pas de pagination visible. Utilisez la **recherche** ou le **filtre de catégorie** pour retrouver un produit précis.

**Puis-je joindre un fichier à un message ?**
Non. La messagerie ne gère pas de pièce jointe et se limite à un **fil par demande**. Passez par un autre canal convenu pour transmettre un document.

**Puis-je modifier mon mot de passe ou mon profil dans le portail ?**
Non. Il n'existe **aucune page** de profil/paramètres côté client. Demandez au fournisseur si un changement est nécessaire.

**Le portail utilise-t-il de l'intelligence artificielle ? Est-ce facturé ?**
Non. Le portail client **ne contient aucun assistant IA** et **ne consomme aucun crédit**. L'assistant « Gestion B2B » est réservé au fournisseur (module 06).

**Puis-je exporter mes commandes en PDF ou en CSV ?**
Non. Le portail n'a **ni** export **ni** impression propres. Seul votre **devis officiel** est consultable/imprimable, via son lien externe (Suivi → Devis).

**Ma session expire-t-elle ?**
Elle dure 7 jours, mais le fournisseur peut **révoquer** votre accès immédiatement (vous êtes alors déconnecté dès l'action suivante).

**Le portail reste-t-il ouvert si l'abonnement du fournisseur est suspendu ?**
Oui, le portail client **n'est pas** soumis au mode consultation. Il continue de fonctionner normalement (contrairement à l'ERP du fournisseur, qui peut passer en lecture seule).

### 5.3 Dépannage courant

| Symptôme | Piste |
|----------|-------|
| « Fournisseur introuvable » à la connexion | Le courriel du fournisseur est erroné, ou son entreprise est inactive. Confirmez l'adresse auprès de votre contact. |
| Connexion refusée juste après l'inscription | Votre compte n'est pas encore **approuvé**. Réessayez plus tard ou relancez le fournisseur. |
| Message d'erreur identique quel que soit le champ | Normal : le message de connexion est **volontairement générique** (protection anti-énumération). Vérifiez fournisseur, courriel et mot de passe. |
| Onglet Suivi vide + bandeau jaune | **Compte non relié** : demandez au fournisseur de poser le lien vers votre fiche d'entreprise. |
| « Stock insuffisant pour : ... » à la commande | Un produit n'a plus assez de stock. Réduisez la quantité ou retirez l'article, puis recommandez. |
| Le prix d'un article du panier ne correspond plus au catalogue | Le prix est **figé à l'ajout**. Retirez puis rajoutez l'article pour prendre le nouveau prix. |
| Impossible de trouver un produit dans un grand catalogue | Pas de pagination visible : utilisez la **recherche** ou le **filtre de catégorie**. |
| Déconnexion soudaine | Session expirée (7 jours) **ou** accès révoqué par le fournisseur. Reconnectez-vous ; si l'accès est révoqué, contactez le fournisseur. |
| Pas de bouton pour accepter une soumission / annuler une commande | Normal : ces décisions sont côté fournisseur. Passez par la messagerie ou un appel. |

---

## 6. Récapitulatif

- **Le Portail B2B / B2C est l'espace en ligne qu'un fournisseur ouvre à ses clients.** Adresse : `/b2b-portal` (tuile « B2B / C2B — Portail client » sur la page de connexion de l'ERP). Application **autonome**, séparée de l'ERP.
- **Deux mondes distincts** : le **portail client** (ce manuel) et le **back-office fournisseur** (module 06, `/ventes?tab=b2b`, administrateurs). C'est le fournisseur qui **approuve** les accès, **chiffre** les soumissions, **crée** les contrats et **fait avancer** les commandes.
- **Huit onglets** : Accueil, Catalogue, Panier, Commandes, Suivi, Contrats, Demandes, Messages ; plus deux écrans publics (Connexion en 2 étapes, Inscription).
- **Inscription = demande d'accès.** Le compte naît **inactif** et n'est utilisable qu'après **approbation manuelle** du fournisseur (aucun courriel de confirmation). Province figée à « Québec ».
- **Commander ≠ payer.** Aucun paiement en ligne : la commande naît **« Non payé »**. Seuls effets concrets : **décrément du stock** + mouvement de stock. Taxes **TPS 5 %** et **TVQ 9,975 %** sur le sous-total. Vérification de stock au moment de commander (refus si insuffisant), sans double-commande possible.
- **Suivi conditionnel.** Les onglets **Devis** et **Projets** ne montrent vos dossiers réels **que si** le fournisseur a **relié** votre compte à votre fiche d'entreprise (lien explicite, jamais automatique). Le **budget** des projets et les **brouillons** de devis restent masqués.
- **Beaucoup de lectures seules côté client** : commandes (après création), suivi, contrats, soumissions reçues. Vous **initiez** (demande, commande, message) ; le fournisseur **décide**.
- **Ce que le portail n'a PAS** : aucun assistant IA, aucun crédit IA, aucun paiement en ligne, aucune page profil/mot de passe, aucune page favoris, aucun export/impression propre, aucune pièce jointe, aucune pagination visible du catalogue, aucun affichage du stock, aucune annulation de commande, aucune acceptation de soumission.
- **Sécurité** : mondes ERP et portail étanches (chacun refuse le jeton de l'autre) ; cloisonnement strict par client ; message de connexion générique (anti-énumération) ; révocation d'accès immédiate ; **le portail n'est pas** soumis au mode consultation Stripe du fournisseur.
- **26 points d'accès client** : 4 d'authentification (`auth.py`) + 22 du portail (`b2b_portal.py`).

---

*Fichiers sources vérifiés :* `backend/routers/b2b_portal.py` (1 321 lignes, 22 points d'accès) ; `backend/routers/auth.py` (4 points d'accès B2B : `b2b-tenant-lookup`, `b2b-client-login`, `b2b-client-register`, `b2b-me`) ; gardes `backend/erp_auth.py` (`get_current_b2b_client`, `create_b2b_client_jwt`) ; DDL et réparations `backend/routers/b2b.py` (`_ensure_b2b_tables`, tables `b2b_*` partagées). Frontend : `frontend/src/pages/b2b-portal/` (`B2bLoginPage.tsx` 208, `B2bRegisterPage.tsx` 391, `B2bDashboardPage.tsx` 97, `B2bCataloguePage.tsx` 134, `B2bPanierPage.tsx` 154, `B2bCommandesPage.tsx` 119, `B2bSuiviPage.tsx` 230, `B2bContratsPage.tsx` 99, `B2bDemandesPage.tsx` 172, `B2bMessagesPage.tsx` 152) ; `components/layout/B2bPortalLayout.tsx` (183), `B2bProtectedRoute.tsx` (25) ; `api/b2b-portal.ts` (277), `api/b2b-portal-auth.ts` (175) ; `store/useB2bPortalStore.ts` (252), `store/useB2bAuthStore.ts` (98) ; traductions `i18n/locales/fr/b2b.json` (section `portal`), `layout.json` (`b2bPortalLayout`), `auth.json` (`b2b`). Montage : `backend/erp_api.py` (`/api/erp/v1/b2b-portal`).

*Manuels liés :* `06-gestion-crm-opportunites.md` (back-office B2B fournisseur : approbation des accès, liaison des fiches, chiffrage, contrats, statuts de commande, assistant IA B2B), `04-gestion-entreprises.md` (fiche d'entreprise CRM reliée au Suivi), `08-ventes-soumissions.md` (devis réels et page publique du document), `09-ventes-projets.md` (projets réels du Suivi), `10-operations-magasin.md` (produits du catalogue et stock), `15-operations-comptabilite.md` (facturation, hors portail), `28-configuration.md` (abonnement du fournisseur ; le portail n'est pas soumis au mode consultation).

*Manuel ERP Constructo AI — Module 36 « Portail B2B / B2C (clients et fournisseurs) » — v1.0 vérifié — 2026-07.*
