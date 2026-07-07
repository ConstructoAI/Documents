# Module 19 — Immobilier (promoteur et vitrine publique)

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — le module est désormais une application distincte à **deux surfaces** : vitrine publique et espace promoteur ; corrections majeures : route réelle `/immo` et non `/immobilier`, **14 onglets** et non 13, ajout de l'onglet Cadastre, du flux de publication vers la vitrine, de l'assistant IA, des modèles IA réels et des permissions réelles)
> **Accès** : page de connexion de l'ERP → tuile « Immobilier » (lien interne), ou directement à l'URL `app.constructoai.ca/immo`. Retour à l'ERP par le lien « Portail Constructo AI » de la barre supérieure.
> **Code de référence (application)** : application React **séparée** `IMMO_REACT/frontend` (base `/immo`, `App.tsx:41`), servie par le service `constructo-erp-react` (modèle B2B/C2B — pas de service d'hébergement dédié)
> **Code de référence (backend)** : cinq routers montés dans `ERP_REACT/backend/erp_api.py` (bloc défensif) — `routers/immobilier.py` (espace promoteur, ≈ 61 points d'accès), `routers/immo_ai.py` (assistant IA, 5 points d'accès, 9 outils), `routers/fonds_prevoyance.py` (Loi 16, ≈ 31 points d'accès), `IMMO_REACT/backend/routers/public.py` (vitrine publique, 4 points d'accès), `IMMO_REACT/backend/routers/publish.py` (publication, 5 points d'accès)
> **Chemins d'API réels** : `/api/erp/v1/immobilier` (promoteur), `/api/erp/v1/immo/ai` (assistant IA), `/api/erp/v1/fonds-prevoyance` (Loi 16), `/api/immo/v1/public` (vitrine, **sans authentification**), `/api/immo/v1/promoteur` (publier / retirer)
> **Tables PostgreSQL** : une série `immo_*` **par tenant** (terrains, projets, financement, unités, inspections, paiements, déblocages, phases, commercialisation, livraisons, documents) ; une série `fp_*` **par tenant** pour la Loi 16 ; et **une table partagée** `public.immo_listings` (les annonces publiées, cloisonnées par colonne `tenant_schema`)
> **Modèle IA** : analyses en profondeur = Claude Opus 4.8 (`claude-opus-4-8`) ; conversations, rapports et suggestions = Claude Sonnet 4.6 (`claude-sonnet-4-6`). Toute consommation IA est facturée aux **crédits IA prépayés** du tenant, avec une majoration de 30 %.
> **Cadrage** : ce module couvre le **cycle de promotion immobilière neuve** (terrains → projets → financement → construction → unités → commercialisation → livraison), le **fonds de prévoyance des copropriétés (Loi 16)**, une **analyse cadastrale**, un **assistant IA**, et — nouveauté majeure — une **vitrine publique d'annonces** sur laquelle le promoteur publie ses unités à vendre. Ce n'est **pas** un gestionnaire locatif complet (pas de baux, pas de portail locataire, pas de génération de loyers) ni un connecteur vers Centris ou un registre officiel.

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

Donner à un promoteur ou entrepreneur général québécois un seul endroit pour **mener un projet immobilier neuf de bout en bout** — depuis le repérage d'un terrain jusqu'à la remise des clés — et pour **exposer publiquement les unités à vendre** sur une vitrine consultable par n'importe quel acheteur, sans qu'il ait à se connecter.

Le module répond à des besoins concrets et distincts selon la personne :

- Côté **acheteur / visiteur** : « Quelles propriétés neuves sont à vendre, dans quelle ville, à quel prix, et comment joindre le promoteur ? »
- Côté **promoteur** : « Comment publier rapidement mes unités avec photos ? Où en sont mes terrains, mes financements, mes phases de chantier ? Quelle contribution au fonds de prévoyance dois-je prévoir pour ma copropriété ? Ce terrain est-il constructible ? »

### 1.2 Deux surfaces dans une même application

Le module est une application React autonome (`IMMO_REACT`) servie sous `app.constructoai.ca/immo`. Elle présente **deux surfaces** partageant la même disposition (barre latérale foncée + barre supérieure) :

| Surface | Authentification | À qui elle s'adresse | Contenu |
|---------|------------------|----------------------|---------|
| **Vitrine publique** | Aucune (ouverte à tous) | Acheteurs, grand public | Accueil, recherche d'annonces, fiche détaillée avec coordonnées du promoteur |
| **Espace promoteur** | Connexion SSO avec les identifiants de l'ERP | Entreprise (tenant) | Publication d'annonces, gestion complète (14 onglets), fonds de prévoyance Loi 16, cadastre, assistant IA |

Le **cœur du module** est le **flux de publication** : une unité créée dans l'espace promoteur devient publiable (ou retirable) de la vitrine depuis l'écran « Mes annonces ». Publier crée ou met à jour une ligne dans la table partagée `public.immo_listings` ; la vitrine ne montre que les annonces actives.

### 1.3 Ce que le module fait (vérifié contre le code)

- **Vitrine publique** : catalogue d'annonces avec recherche avancée (ville, type, prix, superficie, chambres, salles de bains, statut, tri), pagination (24 par page), fiche détaillée avec galerie de photos et bouton de contact par courriel.
- **Publication** : publier ou retirer chaque unité, joindre jusqu'à **12 photos** (téléversement réel, compression automatique, glisser-déposer, réordonnancement, choix de la photo de couverture).
- **Assistant de démarrage** (« Nouveau projet ») : un seul formulaire crée d'un coup un terrain, un projet et plusieurs unités.
- **Gestion immobilière (14 onglets)** : tableau de bord, terrains, cadastre, projets, financement, construction (phases), unités, commercialisation, livraison, inspections, paiements, documents, calculateurs, fonds de prévoyance.
- **Six calculateurs financiers** : mensualité, amortissement, intérêts intercalaires, prime SCHL, ROI, coût total — avec la capitalisation semestrielle propre aux hypothèques canadiennes.
- **Génération automatique des déblocages** d'un financement de construction (7 étapes selon l'avancement).
- **Fonds de prévoyance Loi 16** : copropriétés, inventaire des composantes, études, carnet d'entretien, projections sur 25 ans (3 scénarios), attestations de vente (art. 1069 C.c.Q.) et conseils IA.
- **Analyse cadastrale** : recherche d'un lot par adresse ou numéro, carte OpenStreetMap, faisabilité (zonage), contraintes, avec rattachement à un projet.
- **Assistant IA** : chat qui **propose** une action et l'exécute seulement après **confirmation** de l'utilisateur (créer un terrain, un projet, une unité, un financement, changer un statut, publier une annonce).
- **Bilingue** français / anglais (bascule dans la barre supérieure) et **thème clair / sombre**.

### 1.4 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes.

- **Pas de gestion locative.** Aucun écran de baux, de locataires, de dépenses locatives ni de portail locataire. Des libellés de traduction héritant de l'ERP existent pour ces notions, mais **ils ne sont branchés à aucun écran** de l'application Immobilier : ces modules n'existent pas ici.
- **Onglets Inspections et Paiements en lecture seule.** Vous pouvez consulter les inspections et les paiements d'un projet, mais l'interface **ne permet pas** d'en créer, modifier ou supprimer (aucune barre de commande, aucune fenêtre de saisie). Ces données proviennent d'ailleurs (autres flux ou saisie technique).
- **Onglet Documents sans vrai téléversement.** La fenêtre de saisie ne capture qu'un **« Chemin du fichier » en texte** plus des métadonnées ; elle ne téléverse aucun fichier (contrairement aux photos d'annonces). L'édition d'un document est impossible : on peut seulement créer et supprimer.
- **Un seul vrai téléversement de fichiers** dans toute l'application : les **photos d'annonce**.
- **Une seule exportation** : le **rapport du fonds de prévoyance** (Loi 16) téléchargeable en fichier `.md`. Aucun autre export PDF ou CSV ailleurs.
- **Aucune connexion à Centris, à un registre officiel ou au rôle d'évaluation en temps réel.** Le cadastre s'appuie sur des données ouvertes (OpenStreetMap) à titre indicatif.
- **Assistant IA absent de la vitrine publique** : il n'apparaît que pour un promoteur connecté.
- **Superficies en mètres carrés (m²)** partout dans l'espace promoteur (le calcul de la valeur de reconstruction Loi 16 utilise, lui, le pied carré).
- **Aucun rappel automatique** par courriel, notification ou calendrier sur les échéances (fin de financement, livraison prévue, expiration d'un document).
- **Pas de rôles métier dédiés** (« directeur immobilier », « courtier », « inspecteur ») : voir les permissions ci-dessous.

### 1.5 Accès

- **Depuis l'ERP** : page de connexion → tuile « Immobilier ».
- **Directement** : `app.constructoai.ca/immo`.
- **Écran par défaut (visiteur)** : l'Accueil de la vitrine.
- **Écran par défaut (promoteur connecté)** : « Mes annonces ».
- **Barre latérale** — section publique : **Accueil**, **Annonces** ; section visible selon l'état :
  - Déconnecté : **Devenez promoteur** (mène à la création de compte ERP), **Espace promoteur** (connexion).
  - Connecté : section repliable **Gestion** → **Mes annonces**, **Gestion immobilière**.
- **Barre supérieure** : lien **« Portail Constructo AI »** (retour à l'ERP), bascule **FR / EN**, bascule **thème clair / sombre**, puis le menu de compte (connecté) ou les boutons **Devenez promoteur** + **Connexion** (déconnecté).

### 1.6 Permissions et rôles

> **Point de sécurité important.** Contrairement à la plupart des autres modules de l'ERP, l'espace promoteur **n'impose aucun rôle** : tout utilisateur authentifié du tenant peut créer, modifier, supprimer **et publier sur la vitrine publique**.

| Action | Qui peut la faire | Garde technique |
|--------|-------------------|-----------------|
| Consulter la vitrine, chercher, ouvrir une fiche | **N'importe qui** (public) | Aucune (endpoints publics) |
| Se connecter à l'espace promoteur | Un utilisateur du tenant (mêmes identifiants que l'ERP) | SSO ERP en deux temps |
| Gérer les 14 onglets (terrains, projets, unités, etc.) | Tout utilisateur connecté du tenant | `get_current_user` (aucun `require_role`) |
| Publier / retirer une annonce, gérer les photos | Tout utilisateur connecté du tenant | `get_current_promoteur` + `require_publish_access` |
| Utiliser l'assistant IA (proposer → confirmer) | Tout utilisateur connecté, **si le tenant a des crédits IA** | `get_current_user` + crédits prépayés |
| Utiliser les outils IA du fonds de prévoyance | Tout utilisateur connecté, **si crédits IA** | `get_current_user` + crédits prépayés |

- **Cloisonnement par tenant** : toutes les données `immo_*` et `fp_*` vivent dans le schéma PostgreSQL propre à l'entreprise ; la table partagée `public.immo_listings` est toujours filtrée par `tenant_schema`. Aucun accès entre tenants n'est possible.
- **Mode consultation (abonnement inactif)** : l'accès du promoteur dérive de l'état de l'abonnement (voir §5.2). Un abonnement **annulé ou absent** place le tenant en **mode consultation** : les lectures fonctionnent, mais **publier ou retirer une annonce est refusé** avec le message « Mode consultation : abonnement inactif » (erreur 403). Une entreprise **désactivée** est carrément déconnectée (401).
- **Crédits IA** : chaque appel à l'assistant ou aux conseils Loi 16 vérifie d'abord la disponibilité du service, puis le solde de crédits prépayés ; un solde épuisé renvoie une erreur **402** affichée à l'écran. Le super-administrateur et les sociétés exemptées ne sont pas bloqués.

### 1.7 Les écrans, d'un coup d'œil

| Surface | Écran | Route | Rôle |
|---------|-------|-------|------|
| Vitrine | Accueil | `/` | Héro, recherche, indicateurs, annonces récentes |
| Vitrine | Annonces | `/annonces` | Recherche avancée + grille paginée |
| Vitrine | Fiche | `/annonce/:id` | Galerie, caractéristiques, contact |
| Promoteur | Connexion | `/promoteur/connexion` | SSO ERP |
| Promoteur | Mes annonces | `/promoteur` | Publier / retirer + photos |
| Promoteur | Nouveau projet | `/promoteur/nouveau-projet` | Assistant de démarrage guidé |
| Promoteur | Gestion immobilière | `/promoteur/immobilier` | 14 onglets (dont Cadastre et Fonds de prévoyance) |

---

## 2. Interface

### 2.0 Éléments communs aux deux surfaces

- **Barre latérale** : titre « Immobilier », sous-titre « Constructo AI », pied « Constructo AI © 2026 ». Les entrées changent selon que vous êtes connecté ou non (voir §1.5).
- **Barre supérieure** : bouton hamburger (sur mobile), lien « Portail Constructo AI » (retour à l'ERP), titre de la page courante, bascule FR / EN, bascule thème clair / sombre.
- **Langue** : la préférence est conservée localement (clé `immo_lang`). L'interface, et les réponses de l'assistant IA, s'adaptent au français ou à l'anglais.
- **Session** : le jeton de connexion du promoteur est conservé localement (clé `immo_token`). S'il expire ou est révoqué, une erreur 401 déclenche une **déconnexion automatique** et un retour à la page de connexion — il n'y a pas d'écran d'avertissement de révocation dédié.

---

### 2.1 Vitrine — Accueil

Page d'entrée du grand public.

- **Héro** : titre « Trouvez votre prochaine propriété », un sous-titre, et une **barre de recherche** (indication « Ville, quartier, type de propriété… ») avec le bouton **Rechercher**, qui redirige vers la liste des annonces en appliquant la recherche.
- **Quatre indicateurs clés** (alimentés par `GET /public/stats`) : **Annonces disponibles**, **Villes desservies**, **Promoteurs**, **Prix moyen**. Pendant le chargement, la valeur affiche « … » ; si elle est indisponible, « — ».
- **Annonces récentes** : une grille de six cartes d'annonces, avec un lien « Tout voir ».
- **Appel à l'action promoteur** (bas de page) : « Vous développez des projets immobiliers ? » avec les boutons **Devenez promoteur** (création de compte) et **J'ai déjà un compte** (connexion).
- **États** : un indicateur de chargement au démarrage ; un message « Aucune annonce pour le moment » s'il n'y a rien à afficher.

**Carte d'annonce (élément réutilisé partout)** : photo de couverture (ou une icône si aucune photo), un badge de **statut** dans le coin supérieur gauche, un badge de **type** dans le coin supérieur droit, le titre, la ville, une description tronquée, des icônes chambres / salles de bains / superficie, le prix (ou « Prix sur demande »), et « Voir détails ».

**Badge de statut** : Disponible (vert), Réservé (ambre), Vendu (gris), Annonce (bleu, pour tout autre cas).

---

### 2.2 Vitrine — Annonces (recherche avancée)

Écran de recherche comparable à un portail de type Centris. **Tous les filtres sont pilotés par l'adresse (URL)**, ce qui rend une recherche partageable par simple copier-coller du lien.

**Ligne de base (toujours visible)**

| Champ | Type | Options |
|-------|------|---------|
| Rechercher | Texte | Recherche libre |
| Ville | Menu déroulant | Villes réelles présentes dans les annonces (`GET /public/filters`) |
| Type de propriété | Menu déroulant | Condo, Appartement, Maison, Commerce, Penthouse, Autre |
| Trier par | Menu déroulant | Plus récentes, Prix croissant, Prix décroissant, Grande superficie |

**Recherche avancée (panneau dépliable)**

| Champ | Type |
|-------|------|
| Prix min / Prix max | Nombre ($) |
| Superficie min / Superficie max | Nombre (m²) |
| Chambres | 1+ à 5+ |
| Salles de bains | 1+ à 3+ |
| Statut | Tous, Disponible, Réservé, Vendu |

- Boutons **Appliquer** et **Réinitialiser**.
- **Résultats** : un compteur « N annonces », la grille de cartes (24 par page) et une **pagination**.
- **États** : « Aucun résultat » quand un filtre ne renvoie rien, « Aucune annonce » quand la vitrine est vide.

---

### 2.3 Vitrine — Fiche annonce

- Lien de retour « Retour aux annonces ».
- **Galerie** : une grande photo, plus des vignettes cliquables s'il y a plusieurs photos ; une icône si l'annonce n'a aucune photo.
- **Carte « Description »** : le texte de l'annonce, ou « Aucune description fournie ».
- **Carte latérale** : le titre et son badge de statut, l'adresse, le **prix** (ou « Prix sur demande »), les icônes chambres / salles de bains / superficie, et surtout les moyens de contact :
  - Bouton **« Demander des informations »** : ouvre le logiciel de courriel vers l'adresse du promoteur (repli sur `info@constructoai.ca` si aucune adresse n'est renseignée).
  - Bouton **téléphone** (affiché seulement si un numéro est renseigné).
- **Carte « Caractéristiques »** : Type, Superficie, Chambres, Salles de bain, Code postal, Promoteur, **Publiée le**.
- **État** : « Annonce introuvable » si l'annonce n'existe pas ou n'est plus active.

> **À savoir (vie privée)** : la fiche publique affiche le **courriel** et le **téléphone du promoteur** en clair. C'est un choix assumé pour permettre le contact direct ; ne renseignez que des coordonnées que vous acceptez de rendre publiques.

---

### 2.4 Espace promoteur — Connexion

Titre « Espace promoteur ».

| Champ | Description |
|-------|-------------|
| Courriel de l'entreprise | Le même que pour l'ERP |
| Mot de passe | Le même que pour l'ERP |

- Bouton **Se connecter**. Note affichée : « Même identifiants que votre ERP Constructo AI ».
- Lien « Créer un compte Constructo AI » (mène à l'inscription de l'ERP).
- **Fonctionnement** : la connexion se fait en deux temps contre l'ERP (identification de l'entreprise, puis de l'utilisateur). Le jeton obtenu est réutilisé pour tous les appels de l'espace promoteur — c'est une véritable authentification unique (SSO) avec l'ERP.

---

### 2.5 Espace promoteur — Mes annonces

Écran central du promoteur : c'est ici que l'on **publie** et que l'on **gère les photos**.

- **En-tête** : titre « Mes annonces », l'adresse courriel du promoteur et le compte « N annonce(s) publiée(s) ».
- **Boutons** : **Nouveau projet**, **Gestion Immobilier**, **Voir la vitrine**.
- **Liste** : une carte par unité du tenant, avec :
  - le nom du projet et le numéro de l'unité ;
  - un badge **Publiée** (vert) ou **Non publiée** (gris) ;
  - une ligne type / superficie / prix ;
  - un bouton **Photos** ;
  - un bouton **Publier** (si non publiée) ou **Retirer** (si publiée).
- **État vide** : « Aucune unité à publier » avec un appel à l'action « Créer un projet ».
- **Comportements** :
  - À la publication, le **courriel de contact de l'annonce est pré-rempli** avec celui du promoteur connecté.
  - En mode consultation (abonnement inactif), publier ou retirer renvoie une erreur affichée : « Mode consultation : abonnement inactif ».

**Fenêtre « Photos de l'annonce »** (bouton Photos)

- **Jusqu'à 12 photos** ; la **première est la couverture**.
- **Formats** acceptés : PNG, JPG, WEBP, GIF. **Limites** : 10 Mo par photo, 60 Mo cumulés.
- **Compression automatique côté navigateur** : les grandes images sont réduites (côté le plus long ramené à 1920 px, converties en JPEG) avant l'envoi, pour rester sous les limites.
- **Ajout** : zone de **glisser-déposer** ou bouton **Ajouter des photos**.
- **Par vignette** : **Définir comme couverture** (icône étoile), **Retirer** (X), et **réordonnancement par glisser**. La couverture porte un badge « Couverture ».
- **Boutons** : **Annuler** et, selon l'état, **Enregistrer** (si l'unité est déjà publiée) ou **Enregistrer et publier** (avec la note « Enregistrer publiera aussi cette unité sur la vitrine »).

> **À savoir** : les photos et les coordonnées définies manuellement sur une annonce sont **préservées** si vous republiez l'unité depuis l'ERP sans les redéfinir (voir §5, préservation par COALESCE). En revanche, le **prix, le statut, la superficie, le type et l'adresse** sont **rafraîchis** à partir de la fiche de l'unité à chaque publication.

---

### 2.6 Espace promoteur — Nouveau projet (assistant de démarrage)

Un seul formulaire qui, en un envoi, crée un terrain, puis un projet, puis autant d'unités que vous en ajoutez. C'est la voie rapide vers la publication.

**Section 1 — « Votre projet »**

| Champ | Obligatoire | Détails |
|-------|:-----------:|---------|
| Nom du projet | Oui | Exemple d'indication : « Le Griffin - phase 2 » |
| Type de projet | — | Residentiel, Commercial, Mixte |
| Adresse | Oui | Exemple : « 1200 rue Ottawa » |
| Ville | Oui | — |
| Code postal | — | — |
| Description | — | Zone de texte |

> L'adresse et la ville servent à **localiser les annonces** : elles sont portées par le terrain que l'assistant crée.

**Section 2 — « Vos unités »** (lignes répétables)

| Champ | Détails |
|-------|---------|
| Numéro | Identifiant de l'unité |
| Type | Condo, Appartement, Maison, Penthouse |
| Superficie (m²) | — |
| Chambres | — |
| Salles de bain | — |
| Prix de vente | — |

- Boutons **Ajouter une unité** et **Retirer**.
- Bouton **Créer le projet** → écran de succès « Projet créé ! » (« N unité(s) ajoutée(s) ») avec les boutons **Publier mes unités** et **Créer un autre projet**.
- **Validations** : le nom est requis, l'adresse et la ville sont requises, et il faut au moins une unité.

> **Nuance à connaître** : cet assistant utilise des listes de types **différentes** de celles de la gestion détaillée (voir §4.3). Ici « Type de projet » propose Residentiel / Commercial / Mixte ; dans la fenêtre Projets de la gestion détaillée, il propose Condos / Locatif / Mixte / Commercial / Maisons.

---

### 2.7 Espace promoteur — Gestion immobilière (14 onglets)

La barre d'onglets affiche l'icône seule et un libellé court sur mobile. Voici les 14 onglets, dans l'ordre.

| # | Onglet | Type | Résumé |
|---|--------|------|--------|
| 1 | Tableau de bord | Consultation | Indicateurs + calculateur rapide |
| 2 | Terrains | CRUD complet | Repérage → acquisition |
| 3 | Cadastre | Outil | Analyse d'un lot (carte) |
| 4 | Projets | CRUD complet | Projets de développement |
| 5 | Financement | CRUD complet | Prêts bancaires |
| 6 | Construction | CRUD complet | Phases de chantier |
| 7 | Unités | CRUD complet | Logements / locaux |
| 8 | Commercialisation | CRUD complet | Stratégie de vente |
| 9 | Livraison | CRUD complet | Remise aux bénéficiaires |
| 10 | Inspections | **Lecture seule** | Inspections + conformité |
| 11 | Paiements | **Lecture seule** | Mouvements financiers |
| 12 | Documents | Créer + supprimer | Références documentaires |
| 13 | Calculateurs | Outils | 6 calculs financiers |
| 14 | Fonds Prévoyance | CRUD + IA | Loi 16 (copropriétés) |

> **Correction majeure par rapport à l'ancien manuel** : le module compte **14 onglets**, pas 13 — l'onglet **Cadastre** a été ajouté. Le commentaire interne du fichier de page annonce d'ailleurs encore « 13 tabs » à tort.

#### 2.7.1 Onglet Tableau de bord

- **Quatre indicateurs clés** : Terrains total, Projets total, Financement approuvé, Unités vendues.
- **Deux listes** : Terrains par statut, Projets par statut (avec badges colorés).
- **Calculateur rapide** intégré : Capital / Taux annuel / Durée → Mensualité, Coût total, Intérêts totaux.

#### 2.7.2 Onglet Terrains (CRUD)

- **Barre de commande** : bouton **Nouveau terrain**, recherche, filtre **Statut** (Prospection, Offre en cours, Acquis, En développement, Rejeté).
- **Table** : Numéro, Adresse, Ville, Superficie, Zonage, Prix demandé, Statut, Actions (modifier / supprimer).
- **Fenêtre de saisie** : Adresse\*, Ville\*, Code postal, Superficie (m²), **Zonage** (Résidentiel, Commercial, Mixte, Industriel), Propriétaire (nom), Prix demandé ($), Notes.

#### 2.7.3 Onglet Cadastre

Outil d'analyse d'un lot, entièrement côté navigateur (carte OpenStreetMap), sans point d'accès dédié dans le backend Immobilier. Voir le détail en §2.7.15.

#### 2.7.4 Onglet Projets (CRUD)

- **Barre de commande** : bouton **Nouveau projet**, recherche, filtre **Statut** (Planification, En cours, Construction, Terminé, Annulé).
- **Table** : Numéro, Nom, Type, Logements, Budget, ROI %, Statut, Actions.
- **Fenêtre de saisie** : Nom du projet\*, **Type de projet** (Condos, Locatif, Mixte, Commercial, Maisons), Nombre de logements, Budget total ($), Coût terrain ($), Coût construction ($), Revenus ventes estimés ($), Date début, Date fin, Description, Notes.

#### 2.7.5 Onglet Financement (CRUD)

- **Filtre** : « Filtrer par projet ».
- **Table** : Numéro, Banque, Type prêt, Montant demandé, Montant approuvé, Taux %, Statut, Actions.
- **Fenêtre de saisie** : Projet (menu déroulant), Banque\*, **Type de prêt** (Hypothécaire, Construction, Pont, Marge de crédit), Montant demandé ($), Taux intérêt annuel (%), Durée amortissement (ans), Mise de fonds (%), Notes.

#### 2.7.6 Onglet Construction — phases (CRUD)

- **Prérequis** : un menu **Sélectionner un projet** est obligatoire ; sans projet choisi, un message et une icône invitent à en sélectionner un.
- **Table** : #, Phase, Statut, **Complétion** (barre de progression), Budget prévu, Coût réel, Retard (jours), Actions.
- **Fenêtre de saisie** : Nom de la phase\* (choix parmi des phases standards suggérées, ou texte libre), Numéro de phase, **Statut** (À venir, En cours, En retard, Complétée, Suspendue), Complétion (%), Budget prévu ($), Date début prévue, Date fin prévue, Retard (jours), Raison du retard, plus les cases à cocher **Inspection requise**, **Conforme CNB**, **Matériaux commandés**, **Matériaux reçus**, et Notes.

#### 2.7.7 Onglet Unités (CRUD)

- **Prérequis** : sélection d'un projet obligatoire.
- **Table** : Numéro, Type, Superficie, Chambres, SdB, Étage, Prix vente, Statut, Actions.
- **Fenêtre de saisie** : Numéro d'unité\*, **Type** (Condo, Appartement, Commerce, Maison, Penthouse), Superficie (m²), Chambres, Salles de bain, Étage, Prix vente ($), Loyer mensuel ($), Notes.

> C'est l'unité créée ici (ou par l'assistant Nouveau projet) qui devient **publiable** dans « Mes annonces ».

#### 2.7.8 Onglet Commercialisation (CRUD)

- **Prérequis** : sélection d'un projet.
- **Quatre indicateurs** : Unités vendues, Unités louées, Prix moyen vente, Taux de pré-ventes.
- **Table** : Stratégie, Prix moyen, Loyer moyen, Objectif pré-ventes, Budget marketing, Courtier, Lancement, Actions.
- **Fenêtre de saisie** : **Stratégie de vente** (Pré-vente, Vente directe, Location, Mixte), Date de lancement, Prix moyen vente ($), Loyer moyen ($), Objectif pré-ventes (%), Budget marketing ($), Site web, Courtier (nom), Commission courtier (%), plus les cases **Brochure prête**, **Plans de vente prêts**, **Maquette 3D**, et Notes.

#### 2.7.9 Onglet Livraison (CRUD)

- **Prérequis** : sélection d'un projet.
- **Table** : Unité, Bénéficiaire, Type, Date livraison, Clés (badge Oui / Non), Satisfaction (n/10), Réclamations, Actions.
- **Fenêtre de saisie** : ID Unité, Nom du bénéficiaire, **Type de bénéficiaire** (Acheteur, Locataire), Date de livraison prévue, Durée de garantie (mois), Liste des déficiences ;
  - groupe **Documents remis** (cases) : Clés remises, Acte de vente signé, Bail signé, Manuel de copropriété, Plans conformes, Certificat de conformité ;
  - groupe **Garanties** (cases) : Inspection pré-livraison, Déficiences corrigées, Garantie légale (vice caché), Garantie GCR ;
  - Note de satisfaction (1 à 10), Commentaires du client, Notes.

#### 2.7.10 Onglet Inspections (LECTURE SEULE)

- **Prérequis** : sélection d'un projet.
- **Table** : Type, Catégorie, Inspecteur, **Score** (barre colorée), Déficiences, Statut, **Conformité** (badges CNB / CCE / CSST), Date.
- **Aucun bouton** de création, de modification ou de suppression : cet onglet est **consultatif**.

#### 2.7.11 Onglet Paiements (LECTURE SEULE)

- **Prérequis** : sélection d'un projet.
- **Table** : Type, Catégorie, Montant, Bénéficiaire, Description, Date, Statut.
- **Aucun bouton d'action** : cet onglet est **consultatif**.

#### 2.7.12 Onglet Documents (créer + supprimer)

- **Prérequis / filtres** : sélection d'un projet + filtre **Catégorie**.
- **Table** : Nom, Catégorie, Type de fichier, Date du document, **Confidentiel** (badge rouge), Statut, Actions (supprimer).
- **Fenêtre « Nouveau document »** : Nom du document\*, **Catégorie** (Contrats, Permis, Plans et dessins, Études techniques, Financement, Assurances, Correspondance, Rapports d'inspection, Photos, Autre), **Type de fichier** (PDF, Image, Word, Excel, CAD, Autre), **Chemin du fichier** (texte), Description, Date du document, Date d'expiration, **Confidentiel** (case).

> **Limite importante** : cet onglet **n'héberge pas** de fichiers. Le champ « Chemin du fichier » n'est qu'une référence textuelle (par exemple un lien vers un dossier existant). Pour joindre un vrai fichier, utilisez le module Dossiers de l'ERP, puis collez ici le lien.

#### 2.7.13 Onglet Calculateurs (6 sous-onglets)

Tous les calculs se font côté serveur ; rien n'est enregistré (chaque calcul est ponctuel).

| Sous-onglet | Entrées | Sorties |
|-------------|---------|---------|
| **Mensualité** | Capital ($), Taux annuel (%), Durée (années) | Mensualité, Coût total du crédit, Intérêts totaux |
| **Amortissement** | + **Fréquence** (Mensuel, Bi-hebdomadaire, Hebdomadaire) | Résumé (mensualité, total des intérêts, coût total) + table (Période, Paiement, Capital, Intérêt, Solde) — 24 premières lignes |
| **Intérêts intercalaires** | Montant emprunté, Taux annuel, Durée de construction (mois) | Total des intérêts + table mois par mois (Déblocage, Solde cumulé, Intérêt) |
| **Prime SCHL** | Montant du prêt, Valeur de la propriété | Ratio prêt-valeur, Prime %, Prime en $, Prêt total ; + Taxe sur la prime (Québec 9 %) ; avertissement « non assurable » si le ratio dépasse 95 % |
| **ROI** | Investissement total, Revenus annuels, Dépenses annuelles, Durée | ROI, Bénéfice net annuel, Période de récupération |
| **Coût total** | Capital, Taux annuel, Durée | Mensualité, Coût total, Intérêts totaux, Capital |

Voir §4.4 pour les formules exactes (capitalisation semestrielle, barème SCHL).

#### 2.7.14 Onglet Fonds Prévoyance (Loi 16)

Voir la section détaillée §2.7.16.

#### 2.7.15 Cadastre — analyse de terrain (détail)

- **Recherche** par **adresse civique** ou **numéro de lot** → liste de candidats (numéro de lot, municipalité, superficie).
- **Fiche du lot** :
  - **Identité du lot** : numéro de lot, municipalité, superficie, valeur foncière, code CUBF ;
  - **Carte** interactive (OpenStreetMap, tracé du polygone) ;
  - **Faisabilité (zonage)** : code de zone, usages permis ;
  - **Contraintes** : présentes ou absentes, avec un pourcentage ;
  - **Proximité**, **Accès et logistique**, **Avertissements**.
- Bouton **« Rattacher à un projet »** pour relier l'analyse à un projet existant.

> **À savoir** : le cadastre s'appuie sur des données ouvertes et sert d'aide à la décision. Il **ne remplace pas** un certificat de localisation ni une vérification municipale officielle du zonage.

#### 2.7.16 Fonds de prévoyance — Loi 16

Bandeau de rappel : « Fonds de Prévoyance (Loi 16 du Québec) — Étude obligatoire tous les 5 ans · Carnet d'entretien obligatoire · Période minimale 25 ans ». Un **sélecteur de copropriété** partagé s'applique à tous les sous-onglets. Sept sous-onglets :

**a) Copropriétés** (CRUD)
- Colonnes : Nom, Adresse, Année, Unités, Valeur de reconstruction, Composantes, Études.
- La fenêtre de saisie propose un bouton **« Calculer automatiquement (Québec 2025) »** la valeur de reconstruction (voir §4.5).

**b) Composantes** (inventaire du bâtiment)
- Colonnes : Sous-catégorie, Description, Quantité, État, Vie restante, Coût total.
- Des **alertes** signalent l'urgence : CRITIQUE, URGENT, AVERTISSEMENT, OK.

**c) Études**
- Colonnes : Date, Professionnel, Ordre, Fonds actuel, Recommandé, Contribution annuelle, Conforme.
- Champs d'hypothèses : taux d'inflation, taux de rendement, contingence.

**d) Carnet d'entretien**
- Colonnes : Description, Type, Date prévue, Date réalisée, Coût prévu, Coût réel, Entrepreneur.

**e) Projections**
- Bouton **« Générer les 3 scénarios »** (Uniforme, Progressif, Variable).
- **Graphique** d'évolution du solde du fonds.
- Bouton **« Enregistrer le scénario sélectionné »**.
- Table année par année, avec un **avertissement de découvert** si le fonds devient négatif une année donnée.

**f) Attestations** (art. 1069 C.c.Q. — vente d'une unité)
- Colonnes : Unité, Vendeur, Acheteur, Date de la demande, Date d'émission, Fonds, Arriérés, Statut.

**g) Conseils IA** (4 sous-onglets)
- **Analyse complète** : score de santé, niveau de risque, points d'attention, recommandations, conformité Loi 16, conseil d'expert.
- **Chat expert** : posez une question, avec une case « Inclure le contexte de la copropriété ».
- **Rapport complet** : bouton **Générer**, puis **Télécharger (.md)** — c'est la **seule exportation** de fichier de tout le module.
- **Suggestion de contribution** : à partir du coût de remplacement, du nombre d'unités, de l'horizon et du solde actuel → propose une contribution uniforme, une contribution par unité et par mois, et une contribution progressive par phase.

---

### 2.8 Assistant IA Immobilier

- **Bouton flottant « Assistant IA »**, en bas à droite, **visible uniquement pour le promoteur connecté** — jamais sur la vitrine publique. Il ouvre un panneau latéral coulissant.
- **Titre** : « Assistant Immobilier — Expert IA : terrains, projets, unités, financement ».
- **En-tête du panneau** : **Nouvelle conversation**, **Historique**, Fermer.
- **Chat** : bulles utilisateur / assistant (texte enrichi Markdown), 3 exemples de départ pour amorcer.
- **Principe « proposer → confirmer »** : l'IA ne modifie jamais vos données directement. Elle affiche une **carte de proposition** (aperçu champ par champ) que vous validez avec **Confirmer** ou rejetez avec **Annuler**.
- **Six types d'actions confirmables** : créer un **terrain**, un **projet**, une **unité**, un **financement** ; effectuer un **changement de statut** ; **publier une annonce**. Ce dernier cas est traité à part : il affiche un avertissement (« publier rend cette annonce visible au public… immédiatement ») et un bouton **Publier** de couleur ambre.
- **Historique** : la liste des conversations est **conservée côté serveur** ; vous pouvez reprendre ou supprimer une conversation.

> **À savoir** : l'assistant ne s'exécute qu'après votre confirmation, et il n'agit **que sur votre tenant** — il ne peut pas cibler une autre entreprise. La recherche interne qu'il utilise est cloisonnée et protégée contre l'accès aux tables sensibles (employés, paie, utilisateurs, etc.).

---

## 3. Workflows pas à pas

### 3.1 Visiteur : de la recherche au contact

1. Ouvrez `app.constructoai.ca/immo` (Accueil).
2. Saisissez une ville ou un type dans la barre de recherche, puis **Rechercher** (ou allez dans **Annonces**).
3. Affinez avec la **recherche avancée** (prix, superficie, chambres, statut) et un **tri**.
4. Cliquez une carte pour ouvrir la **fiche**.
5. Parcourez la galerie et les caractéristiques.
6. Cliquez **« Demander des informations »** (ouvre un courriel vers le promoteur) ou utilisez le **bouton téléphone**.

### 3.2 Promoteur : publier rapidement (voie express)

1. **Connexion** à l'espace promoteur (identifiants ERP).
2. **Nouveau projet** : remplissez la section « Votre projet » (nom, adresse, ville) et ajoutez une ou plusieurs unités (numéro, type, superficie, chambres, salles de bain, prix).
3. **Créer le projet** → écran de succès.
4. Cliquez **Publier mes unités** (ou allez dans **Mes annonces**).
5. Pour chaque unité, cliquez **Photos**, ajoutez jusqu'à 12 images (glisser-déposer), choisissez la **couverture**, réordonnez au besoin, puis **Enregistrer et publier**.
6. Vérifiez le résultat avec **Voir la vitrine**.

### 3.3 Publier, retirer et gérer les photos d'une unité existante

1. **Mes annonces** : repérez l'unité (badge Publiée / Non publiée).
2. **Publier** : rend l'annonce visible (le courriel de contact est pré-rempli avec le vôtre).
3. **Photos** : gérez la galerie à tout moment ; « Définir comme couverture » (étoile), « Retirer » (X), glisser pour réordonner.
4. **Retirer** : masque l'annonce de la vitrine **sans supprimer** ses données ni ses photos (vous pourrez republier plus tard avec la même galerie).
5. Si un message « Mode consultation : abonnement inactif » apparaît, réactivez l'abonnement de l'entreprise (voir §5.2).

### 3.4 Promoteur : gestion fine d'un projet (cycle complet)

1. **Terrains** → Nouveau terrain (adresse, ville, superficie, zonage). Faites évoluer le **statut** : Prospection → Offre en cours → Acquis → En développement.
2. (Optionnel) **Cadastre** → analysez le lot, puis **Rattacher à un projet**.
3. **Projets** → Nouveau projet (type, logements, budgets, revenus estimés, dates).
4. **Financement** → Nouveau financement (banque, type de prêt, montant demandé, taux, durée) ; passez le montant approuvé une fois l'accord obtenu.
5. **Déblocages** → générez-les automatiquement (voir §3.5).
6. **Construction** → sélectionnez le projet, créez les **phases** (statut, complétion, budget prévu, cases de conformité et de matériaux).
7. **Unités** → sélectionnez le projet, créez chaque unité (type, superficie, chambres, prix).
8. **Commercialisation** → définissez la stratégie, l'objectif de pré-ventes, le courtier, le budget marketing.
9. **Livraison** → à la remise, saisissez le bénéficiaire, cochez les documents remis et les garanties, notez la satisfaction.
10. **Inspections** et **Paiements** → consultez l'avancement (lecture seule).

### 3.5 Générer automatiquement les déblocages d'un financement

1. Assurez-vous que le financement porte un **montant approuvé (engagement)**.
2. Lancez la génération automatique (bouton dédié ou l'assistant IA).
3. Le système crée **7 étapes** de déblocage, réparties selon l'avancement typique d'un chantier : **10 %, 15 %, 25 %, 15 %, 20 %, 10 %, 5 %** (total 100 %).
4. **Garde-fous** : l'opération est **atomique** et **idempotente** — si des déblocages existent déjà, elle est refusée (409) pour éviter les doublons ; si le total dépassait l'engagement, elle est refusée (400) ; la dernière étape absorbe l'arrondi pour que la somme tombe juste.
5. Suivez ensuite chaque déblocage individuellement.

### 3.6 Fonds de prévoyance Loi 16 : de la copropriété à la projection

1. Onglet **Fonds Prévoyance** → sous-onglet **Copropriétés** → créez la copropriété (nom, adresse, année, unités). Utilisez **« Calculer automatiquement (Québec 2025) »** pour la valeur de reconstruction.
2. Sous-onglet **Composantes** → inventoriez toits, ascenseurs, fenêtres, etc. (quantité, coût unitaire, état, vie restante). Le coût total et la vie restante peuvent être calculés automatiquement.
3. Sous-onglet **Études** → créez une étude du fonds (professionnel, ordre, hypothèses d'inflation, de rendement et de contingence).
4. Sous-onglet **Projections** → **Générer les 3 scénarios** (Uniforme, Progressif, Variable), examinez le graphique et le tableau année par année, surveillez tout **avertissement de découvert**, puis **Enregistrer le scénario sélectionné**.
5. Sous-onglet **Carnet d'entretien** → planifiez et suivez les travaux (dates prévues / réalisées, coûts, entrepreneur).
6. Sous-onglet **Attestations** → à la vente d'une unité, émettez l'attestation (art. 1069 C.c.Q.) avec l'état du fonds et les arriérés.
7. Sous-onglet **Conseils IA** → lancez une **Analyse complète**, discutez avec l'**expert**, ou générez un **Rapport complet** (téléchargeable en `.md`).

### 3.7 Utiliser les calculateurs financiers

1. Onglet **Calculateurs** → choisissez le sous-onglet voulu.
2. **Mensualité / Coût total** : Capital, Taux annuel, Durée → mensualité et intérêts. La formule utilise la **capitalisation semestrielle** canadienne (voir §4.4).
3. **Amortissement** : ajoutez la **Fréquence** (mensuel, bi-hebdomadaire, hebdomadaire) pour obtenir le tableau détaillé.
4. **Intérêts intercalaires** : Montant, Taux, Durée de construction → intérêts à capitaliser pendant les travaux.
5. **Prime SCHL** : Montant du prêt, Valeur de la propriété → ratio, prime, taxe québécoise de 9 % ; au-delà de 95 %, le prêt est **non assurable**.
6. **ROI** : Investissement, Revenus, Dépenses, Durée → rendement et période de récupération.

> Les calculateurs **n'enregistrent rien**. Pour conserver un résultat, recopiez-le dans les notes d'un financement ou dans un document du projet.

### 3.8 Analyser un lot au cadastre et le rattacher

1. Onglet **Cadastre** → saisissez une adresse civique ou un numéro de lot → **Rechercher**.
2. Choisissez un candidat dans la liste.
3. Examinez l'**identité du lot**, la **carte**, la **faisabilité (zonage)**, les **contraintes**, la **proximité** et les **avertissements**.
4. Cliquez **« Rattacher à un projet »** pour relier l'analyse.

### 3.9 Piloter par l'assistant IA (proposer → confirmer)

1. Ouvrez l'**Assistant IA** (bouton flottant, promoteur connecté).
2. Formulez votre demande en langage naturel — par exemple « combien d'unités invendues ? » ou « crée un terrain à Granby ».
3. Pour une **lecture** (tableau de bord, recherche), la réponse s'affiche directement.
4. Pour une **action** (créer, changer un statut, publier), l'IA présente une **carte de proposition** : vérifiez l'aperçu, puis **Confirmer** ou **Annuler**.
5. Pour une **publication**, lisez l'avertissement (visibilité publique immédiate) avant de cliquer **Publier**.
6. Reprenez une discussion antérieure via **Historique**, ou repartez à neuf avec **Nouvelle conversation**.

---

## 4. Référence

### 4.1 Points d'accès par backend

**a) Espace promoteur — `routers/immobilier.py` (`/api/erp/v1/immobilier`, ≈ 61 points d'accès, authentification ERP)**

| Entité | Points d'accès |
|--------|----------------|
| Tableau de bord | `GET /dashboard` |
| Terrains | `GET /terrains`, `GET /terrains/{id}`, `POST`, `PUT`, `DELETE` |
| Projets | `GET /projets`, `GET /projets/{id}`, `POST`, `PUT`, `DELETE` |
| Financements | `GET /financements`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Unités | `GET /unites` (**`projet_id` obligatoire**), `POST`, `PUT`, `DELETE` |
| Inspections | `GET /inspections`, `POST`, `PUT` (pas de suppression) |
| Paiements | `GET /paiements`, `POST` (pas de modification ni suppression) |
| Déblocages | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE`, `POST /deblocages/generer-auto` |
| Phases | `GET`, `GET /phases/types`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Commercialisation | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Livraisons | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE` |
| Documents | `GET`, `GET /{id}`, `POST`, `DELETE` |
| Calculateurs | `POST /calculer-mensualite`, `/calculer-amortissement`, `/calculer-interets-intercalaires`, `/calculer-prime-schl`, `/calculer-roi`, `/calculer-cout-total` |
| IA | `POST /ia/analyser-projet` (Opus), `/ia/chat`, `/ia/rapport-financement`, `/ia/optimiser-financement` (Sonnet) |

**b) Assistant IA — `routers/immo_ai.py` (`/api/erp/v1/immo/ai`, 5 points d'accès, 9 outils)**

| Point d'accès | Rôle |
|---------------|------|
| `POST /chat` | Conversation (boucle d'outils, historique persisté) |
| `POST /confirm-action` | Exécute une proposition **après confirmation** (revalidée entièrement côté serveur) |
| `GET /conversations` | Liste des conversations |
| `GET /conversations/{id}` | Détail (404 si elle n'est pas à l'utilisateur) |
| `DELETE /conversations/{id}` | Suppression |

Les 9 outils : `recherche_bd` (lecture SQL restreinte), `tableau_de_bord_immo`, `calculer_financier_immo`, `proposer_terrain`, `proposer_projet`, `proposer_unite`, `proposer_financement`, `proposer_changement_statut`, `proposer_publication_annonce`.

> **Nuance** : la mémoire produit parlait de « 9 outils » ; côté interface, seuls **6 types d'actions** sont *confirmables* (terrain, projet, unité, financement, changement de statut, publication). Les trois autres outils sont des lectures qui ne demandent pas de confirmation.

**c) Fonds de prévoyance — `routers/fonds_prevoyance.py` (`/api/erp/v1/fonds-prevoyance`, ≈ 31 points d'accès, authentification ERP)**

| Entité | Points d'accès |
|--------|----------------|
| Référence | `GET /reference` |
| Copropriétés | `GET`, `GET /{id}`, `POST`, `PUT`, `DELETE`, `GET /{id}/statistiques` |
| Composantes | `GET /coproprietes/{id}/composantes`, `POST`, `PUT`, `DELETE` |
| Études | `GET /coproprietes/{id}/etudes`, `GET /etudes/{id}`, `POST`, `PUT`, `DELETE` |
| Projections | `POST /etudes/{id}/generer-projections`, `GET /etudes/{id}/projections` |
| Carnet d'entretien | `GET /coproprietes/{id}/entretiens`, `POST`, `PUT`, `DELETE` |
| Attestations | `GET /coproprietes/{id}/attestations`, `POST`, `PUT`, `DELETE` |
| IA | `POST /ia/analyze-copropriete` (Opus), `/ia/chat`, `/ia/suggest-contribution`, `/ia/rapport-recommandations` (Sonnet) |
| Calcul | `POST /calculer-valeur-reconstruction` |

**d) Vitrine publique — `IMMO_REACT/backend/routers/public.py` (`/api/immo/v1/public`, 4 points d'accès, SANS authentification)**

| Point d'accès | Rôle |
|---------------|------|
| `GET /listings` | Recherche avancée (ville, type, prix, superficie, chambres, salles de bains, statut, tri, texte ; 100 max par page) |
| `GET /filters` | Villes et valeurs disponibles pour les menus |
| `GET /stats` | Indicateurs de l'Accueil |
| `GET /listings/{id}` | Fiche (404 si l'annonce est inactive) |

**e) Publication — `IMMO_REACT/backend/routers/publish.py` (`/api/immo/v1/promoteur`, 5 points d'accès, authentification promoteur)**

| Point d'accès | Rôle |
|---------------|------|
| `GET /me` | Profil promoteur |
| `GET /units` | Unités du tenant + état de publication |
| `GET /units/{id}/photos` | Lecture des photos |
| `POST /units/{id}/publish` | Publier (valide les photos, refusé 403 en mode consultation) |
| `POST /units/{id}/unpublish` | Retirer de la vitrine |

### 4.2 Statuts par entité

| Entité | Statuts |
|--------|---------|
| Terrain | Prospection, Offre en cours, Acquis, En développement, Rejeté |
| Projet | Planification, En cours, Construction, Terminé, Annulé |
| Phase de construction | À venir, En cours, En retard, Complétée, Suspendue |
| Unité / Annonce | Disponible, Réservé, Vendu |
| Bénéficiaire (livraison) | Acheteur, Locataire |

> Les statuts d'annonce sont normalisés côté serveur en `disponible` / `reserve` / `vendu` (jamais d'autre valeur acceptée à la publication).

### 4.3 Taxonomies (types)

| Champ | Valeurs |
|-------|---------|
| Zonage (terrain) | Résidentiel, Commercial, Mixte, Industriel |
| Type de projet (gestion détaillée) | Condos, Locatif, Mixte, Commercial, Maisons |
| Type de projet (assistant Nouveau projet) | Residentiel, Commercial, Mixte |
| Type de prêt | Hypothécaire, Construction, Pont, Marge de crédit |
| Type d'unité (gestion détaillée) | Condo, Appartement, Commerce, Maison, Penthouse |
| Type d'unité (assistant Nouveau projet) | Condo, Appartement, Maison, Penthouse |
| Type de propriété (recherche vitrine) | Condo, Appartement, Maison, Commerce, Penthouse, Autre |
| Catégorie de document | Contrats, Permis, Plans et dessins, Études techniques, Financement, Assurances, Correspondance, Rapports d'inspection, Photos, Autre |
| Stratégie de vente | Pré-vente, Vente directe, Location, Mixte |

> **Deux listes de types coexistent** (assistant Nouveau projet par rapport à la gestion détaillée) : c'est une réalité du code, pas une erreur de saisie. Choisissez la valeur qui a du sens ; elle reste modifiable ensuite dans l'onglet correspondant.

### 4.4 Calculs financiers (formules réelles)

- **Taux périodique (hypothèque canadienne)** : capitalisation **semestrielle**, donc le taux par période vaut `(1 + i/2)^(2/p) − 1` (et non `i/p`), où `i` est le taux annuel et `p` le nombre de périodes par an. C'est la convention légale au Canada.
- **Mensualité** : `M = P · [ r (1+r)^n ] / [ (1+r)^n − 1 ]`, avec `r` le taux périodique et `n` le nombre de versements. La dernière période absorbe le résidu d'arrondi.
- **Fréquences d'amortissement** : Mensuel = 12 versements/an, Bi-hebdomadaire = 26, Hebdomadaire = 52.
- **Intérêts intercalaires** : déblocage supposé **linéaire** (`montant / durée en mois`), intérêt mensuel `(taux/100)/12` appliqué sur le solde cumulé.
- **ROI** : `(bénéfice net annuel × durée / investissement) × 100`. Période de récupération = `investissement / bénéfice net annuel` (non définie si le bénéfice est nul ou négatif).
- **Prime SCHL** (selon le ratio prêt-valeur, RPV) :

| RPV | Prime | Remarque |
|-----|-------|----------|
| > 95 % | — | **Non assurable** |
| 90,01 % à 95 % | 4,00 % | — |
| 85,01 % à 90 % | 3,10 % | — |
| 80,01 % à 85 % | 2,80 % | — |
| ≤ 80 % | 0 % | Prêt conventionnel (pas de prime) |

  La **taxe québécoise de 9 %** sur la prime est payable **comptant** (elle ne s'ajoute pas au prêt).

- **Déblocages automatiques** : 7 étapes de **10 / 15 / 25 / 15 / 20 / 10 / 5 %** (= 100 %).

### 4.5 Fonds de prévoyance — barèmes Loi 16

- **Valeur de reconstruction (Québec 2025)** : taux au pied carré × facteur de type × facteur d'âge.

| Qualité | $/pi² | | Type | Facteur | | Âge du bâtiment | Facteur |
|---------|------:|---|------|--------:|---|-----------------|--------:|
| Économique | 250 | | Résidentiel | 1,00 | | > 50 ans | 0,85 |
| Base | 325 | | Commercial | 1,15 | | … | … |
| Moyenne | 387 | | Mixte | 1,08 | | ≤ 15 ans | 1,00 |
| Haut de gamme | 487 | | Industriel | 0,95 | | | |

- **Facteurs d'état d'une composante** (influencent la vie restante) : Excellent 1,10 · Bon 1,00 · Moyen 0,85 · Mauvais 0,70 · **Critique 0,00** (vie restante nulle → remplacement immédiat).
- **Période de projection** : 25 ans par défaut (minimum légal), configurable de 1 à 100 ans.
- **Trois scénarios de contribution** :
  - **Uniforme** : cotisation annuelle constante calculée par annuité (valeur actuelle nette des dépenses futures).
  - **Progressif** : cotisation qui augmente de 3 %/an, ajustée par un solveur pour équilibrer le fonds.
  - **Variable** : cotisation qui vise, chaque année, une réserve d'au moins la prochaine grosse dépense majorée de 15 % (avec un plancher de 50 000 $).
- **Réinflation cyclique** : chaque composante est remplacée à la fin de sa vie restante, puis à chaque cycle de vie théorique ; son coût est **ré-inflaté** à chaque remplacement.
- **Alerte de découvert intermédiaire** : même si le solde final est positif, le système signale l'année et le montant du **découvert maximal** en cours de route (le solde peut passer sous zéro une année donnée).

### 4.6 Numéros de référence générés

Les entités reçoivent un numéro unique à la création (par exemple `TER-#####` pour un terrain, `IMMO-#####` pour une unité, `DEB-#####` pour un déblocage, `FIN-…` pour un financement). Ces numéros sont générés de façon fiable (jamais par un simple compteur), avec réessai en cas de collision.

### 4.7 Photos d'annonce — limites

| Règle | Valeur |
|-------|--------|
| Nombre maximal | 12 photos (la 1re = couverture) |
| Formats acceptés | PNG, JPG, WEBP, GIF (et liens `https://`) |
| Format refusé | SVG (protection contre les scripts malveillants) |
| Taille par photo | ~ 10 Mo |
| Taille cumulée | 60 Mo |
| Compression | Automatique côté navigateur (1920 px max, JPEG) |

### 4.8 Bornes et validations utiles

- **Unités** : la liste exige un `projet_id` (on ne liste jamais toutes les unités du tenant d'un coup) ; créer une unité valide l'existence du projet parent (404 sinon).
- **Calculateurs** : taux plafonné à 100 %, durée à 100 ans, durée de construction à 600 mois (protections anti-débordement).
- **Champs de date vides** : convertis automatiquement en « vide » pour éviter les erreurs de base de données.
- **Suppressions en cascade** : supprimer un **projet** retire ses entités liées (financements, unités, phases, etc.) **et désactive ses annonces publiques** ; supprimer une **unité** désactive aussi son annonce (sinon elle resterait affichée à vie) ; supprimer un **terrain** délie les projets qui le référençaient.

---

## 5. Intégrations et FAQ

### 5.1 Authentification unique (SSO) avec l'ERP

L'espace promoteur partage l'authentification de l'ERP : mêmes identifiants, même jeton. Se connecter à l'Immobilier, c'est se connecter à l'entreprise. La déconnexion, ou l'expiration du jeton, ramène à la page de connexion.

### 5.2 Stripe et le mode consultation

L'accès du promoteur dépend de l'état de l'abonnement de l'entreprise (interrogé via l'ERP) :

| Niveau | Condition | Effet |
|--------|-----------|-------|
| **Complet** | Abonnement actif, en essai, en annulation, en retard ou impayé | Lecture et écriture |
| **Consultation** | Sans abonnement Stripe ou abonnement annulé | Lecture seule ; **publier / retirer refusé (403)** |
| **Bloqué** | Entreprise désactivée | Déconnexion (401) |

En pratique, si vous voyez « Mode consultation : abonnement inactif » en tentant de publier, c'est que l'abonnement de l'entreprise doit être régularisé (module Configuration / abonnement de l'ERP).

### 5.3 Crédits IA

L'assistant Immobilier et les conseils IA du fonds de prévoyance **consomment les crédits IA prépayés** du tenant (les mêmes que le reste de l'ERP), avec une majoration de 30 % sur le coût réel. Un solde épuisé renvoie une erreur 402. Les analyses en profondeur utilisent Claude Opus 4.8 ; les conversations, rapports et suggestions utilisent Claude Sonnet 4.6.

### 5.4 Vitrine publique et vie privée

La vitrine est **ouverte à tous**, sans connexion. Elle n'expose jamais l'identifiant interne du tenant, mais elle affiche **volontairement** le courriel et le téléphone du promoteur sur chaque fiche, pour permettre le contact. Un plafond de requêtes par adresse IP limite la moisson automatisée.

### 5.5 Lien avec les autres modules

- **Module Dossiers de l'ERP** : à utiliser pour héberger de vrais fichiers, puisque l'onglet Documents d'Immobilier ne stocke qu'un chemin texte.
- **Module Comptabilité** : les paiements de l'onglet Paiements ne sont **pas** comptabilisés automatiquement dans le grand livre. À saisir manuellement au besoin.
- **Module Projets de l'ERP** : les projets immobiliers (`immo_projets`) sont **distincts** des projets de chantier de l'ERP ; il n'y a pas de lien automatique entre les deux.
- **Loi 16** : le fonds de prévoyance est un sous-module à part entière, avec ses propres tables et son propre assistant IA.

### 5.6 FAQ

**Q : Où sont passés les onglets Propriétés, Locataires, Baux et Dépenses de l'ancien manuel ?**
R : Ils n'existent pas ici. L'application Immobilier est un outil de **promotion** et de **vitrine**, pas un gestionnaire locatif. Des libellés de traduction traînent dans les fichiers de langue, mais aucun écran ne les affiche.

**Q : Puis-je créer une inspection ou un paiement depuis l'interface ?**
R : Non. Les onglets Inspections et Paiements sont en **lecture seule** : ni bouton de création, ni fenêtre de saisie. On peut seulement consulter.

**Q : Comment joindre un vrai PDF à un document de projet ?**
R : L'onglet Documents ne téléverse pas de fichier ; il ne garde qu'un « Chemin du fichier » (texte). Hébergez le fichier dans le module Dossiers de l'ERP, puis collez son lien dans ce champ. Le **seul** vrai téléversement de l'application est celui des **photos d'annonce**.

**Q : Si je republie une unité depuis l'ERP, est-ce que je perds mes photos et le texte que j'ai peaufinés sur l'annonce ?**
R : Non pour les photos, le titre, la description et les coordonnées : ils sont **préservés** si vous ne les redéfinissez pas. **Oui** en revanche pour le **prix, le statut, la superficie, le type et l'adresse**, qui sont **rafraîchis** à partir de la fiche de l'unité à chaque publication. Si vous aviez ajusté le prix directement sur l'annonce, il sera écrasé par le prix de vente de l'unité.

**Q : Retirer une annonce supprime-t-il ses photos ?**
R : Non. « Retirer » masque l'annonce (elle devient inactive) mais conserve toutes ses données et sa galerie. Vous pouvez republier plus tard sans tout refaire.

**Q : L'assistant IA peut-il modifier mes données sans mon accord ?**
R : Non. Il **propose** une action et attend votre **confirmation**. Il agit uniquement sur votre entreprise, et il est protégé contre l'accès aux données sensibles (paie, employés, utilisateurs).

**Q : Y a-t-il un export PDF ou CSV des annonces, projets ou financements ?**
R : Non. La **seule** exportation de fichier de tout le module est le **rapport du fonds de prévoyance** en `.md` (onglet Fonds Prévoyance → Conseils IA → Rapport complet).

**Q : Le cadastre remplace-t-il un certificat de localisation ?**
R : Non. Il donne une lecture indicative (données ouvertes, carte OpenStreetMap) pour aider à évaluer un lot. Validez toujours le zonage et les contraintes auprès de la municipalité.

**Q : Les calculateurs enregistrent-ils leurs résultats ?**
R : Non. Chaque calcul est ponctuel. Recopiez les résultats dans les notes d'un financement ou dans un document si vous voulez les conserver.

**Q : Tout le monde dans l'entreprise peut-il publier sur la vitrine publique ?**
R : Oui. L'espace promoteur n'impose aucun rôle : tout utilisateur connecté du tenant peut créer, modifier, supprimer et **publier**. Si le contrôle est un enjeu, encadrez cet usage à l'interne.

**Q : Pourquoi ne puis-je pas publier alors que je suis bien connecté ?**
R : Probablement le **mode consultation** : l'abonnement de l'entreprise est inactif. Le message « Mode consultation : abonnement inactif » l'indique. Régularisez l'abonnement pour réactiver la publication.

**Q : Combien d'unités puis-je créer par projet ?**
R : Pas de limite fixe. En pratique, la liste des unités exige de choisir un projet, ce qui garde l'affichage gérable même avec beaucoup d'unités.

**Q : Le fonds de prévoyance garantit-il que le fonds ne sera jamais à découvert ?**
R : Non, et le système est transparent là-dessus : même quand le solde final est positif, il vous signale l'**année et le montant du découvert maximal** en cours de route, pour trois scénarios de cotisation.

---

## 6. Récapitulatif

- **Deux surfaces** : une **vitrine publique** sans connexion (Accueil, Annonces, Fiche) et un **espace promoteur** en SSO ERP.
- **Route réelle** : `/immo` (application `IMMO_REACT` séparée), pas `/immobilier`.
- **Cœur du module** : le **flux de publication** — une unité de l'espace promoteur devient une annonce publique (table partagée `public.immo_listings`) depuis « Mes annonces ».
- **Voie express** : l'assistant « Nouveau projet » crée terrain + projet + unités en un envoi, puis « Publier mes unités ».
- **Photos d'annonce** : jusqu'à 12, glisser-déposer, réordonnancement, couverture, compression automatique — le **seul** vrai téléversement de fichiers.
- **Gestion détaillée : 14 onglets** (dont **Cadastre**, ajouté) — Tableau de bord, Terrains, Cadastre, Projets, Financement, Construction, Unités, Commercialisation, Livraison, Inspections, Paiements, Documents, Calculateurs, Fonds Prévoyance.
- **Inspections et Paiements = lecture seule** ; **Documents = chemin texte** (pas de vrai fichier) ; création et suppression seulement.
- **Six calculateurs** avec la **capitalisation semestrielle** canadienne et le barème **SCHL** (non assurable > 95 %, taxe québécoise de 9 %).
- **Déblocages automatiques** : 7 étapes (10/15/25/15/20/10/5 %), atomiques et sans doublon.
- **Fonds de prévoyance Loi 16** : copropriétés, composantes, études, carnet, **projections 25 ans à 3 scénarios**, attestations (art. 1069 C.c.Q.), conseils IA + **rapport `.md`** (seule exportation).
- **Cadastre** : recherche d'un lot, carte OpenStreetMap, faisabilité, contraintes, rattachement à un projet (données indicatives).
- **Assistant IA** (promoteur seulement) : **propose → confirme**, six actions confirmables dont la publication, historique conservé.
- **Modèles IA** : Opus 4.8 (analyses), Sonnet 4.6 (chat, rapports, suggestions), facturés aux crédits prépayés (+30 %).
- **Permissions** : aucun rôle dédié — tout utilisateur du tenant peut tout faire, y compris publier ; **mode consultation** (abonnement inactif) bloque la publication (403).
- **Ce que le module ne fait pas** : pas de gestion locative, pas de Centris, pas d'exports (sauf le `.md` Loi 16), pas de comptabilisation automatique, superficies en m².

---

**Documentation générée à partir du code** (7 juillet 2026) :
- `IMMO_REACT/frontend` (application React, base `/immo`) — pages Accueil, Annonces, Fiche, Connexion, Mes annonces, Nouveau projet, Gestion immobilière (14 onglets), Cadastre, panneau Assistant IA.
- `ERP_REACT/backend/routers/immobilier.py` (espace promoteur, ≈ 61 points d'accès), `routers/immo_ai.py` (assistant IA, 5 points d'accès, 9 outils), `routers/fonds_prevoyance.py` (Loi 16, ≈ 31 points d'accès).
- `IMMO_REACT/backend/routers/public.py` (vitrine, 4 points d'accès), `routers/publish.py` (publication, 5 points d'accès), `immo_database.py` (table partagée `public.immo_listings`), `immo_auth.py` (authentification et mode consultation).

**Manuels liés** :
- Module 07 (Dossiers — pour héberger de vrais fichiers) — `07-ventes-dossiers.md`
- Module 09 (Projets — distincts des projets immobiliers) — `09-ventes-projets.md`
- Module 15 (Comptabilité — écritures manuelles) — `15-operations-comptabilite.md`
- Module 25 (Assistant IA — crédits IA partagés) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — abonnement et mode consultation) — `28-configuration.md`
