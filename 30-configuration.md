# Module 30 — Configuration (entreprise, utilisateurs, abonnement)

> **Version** : 3.0 (refonte vérifiée contre le code source, juillet 2026)
> **Code de référence** :
> - Frontend : `frontend/src/pages/ConfigurationPage.tsx` (4 108 lignes — page unique à **11 onglets**), client API `frontend/src/api/config.ts` (661 lignes) et `frontend/src/api/stripe.ts`, magasins d'état `frontend/src/store/useConfigStore.ts` (209 lignes), `frontend/src/store/useStripeStore.ts` (145 lignes) et `frontend/src/store/useUiThemeStore.ts` (236 lignes), traductions `frontend/src/i18n/locales/{fr,en}/config.json` (441 lignes chacune)
> - Backend : `backend/routers/config.py` (3 247 lignes — **38 points d'accès**, préfixe réel `/api/erp/v1/config`), `backend/routers/stripe_routes.py` (379 lignes — **6 points d'accès**, préfixe `/api/erp/v1/stripe`), `backend/routers/html_utils.py` (thème des documents), gardes d'accès `backend/erp_auth.py`
> - Onglet **Intégrations** : module séparé `frontend/src/pages/IntegrationPage.tsx` (1 608 lignes — **7 sous-onglets**) + router `backend/routers/integration.py` (hors du présent module ; voir le manuel dédié)
> **Tables PostgreSQL** : `public.entreprises` (1 ligne par tenant — abonnement, taxes, devise, pays, fuseau, langue, retenue, exercice fiscal), `{tenant}.entreprise_config` (configuration en JSON : logo, coordonnées, thème des documents, catégories de fournisseurs, clés libres), `{tenant}.users` (comptes et rôles), `{tenant}.payroll_config` (renseignements employeur), `{tenant}.webhooks` / `{tenant}.webhook_deliveries` (points d'ancrage sortants, sans interface). Facturation via les tables partagées Stripe et le registre de crédits IA prépayés.
> **Cadrage** : la Configuration est le **centre de paramétrage** de votre entreprise dans l'ERP. Une seule page à onglets réunit votre **profil** personnel, la **gestion des utilisateurs** (comptes et droits), l'**identité de l'entreprise** (logo, coordonnées, numéros RBQ/NEQ/TPS/TVQ), l'**apparence** de vos documents et de votre interface, la **fiscalité** (pays, devise, taxes, retenue de garantie, juridiction, exercice fiscal), la **langue** des documents, le **fuseau horaire**, votre **abonnement** et vos **crédits IA** (Stripe), et l'accès aux **intégrations comptables**. C'est un module d'administration : la plupart de ses onglets sont réservés à l'**administrateur** de l'entreprise.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Procédures pas à pas](#3-procédures-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

La page **Configuration** permet à l'entrepreneur de régler, en un seul endroit, tout ce qui gouverne le comportement de l'ERP pour son entreprise :

- **Son compte** — modifier son nom, son courriel et son mot de passe.
- **Les utilisateurs** — créer des comptes pour ses employés, leur attribuer un rôle, réinitialiser un mot de passe, désactiver un compte.
- **L'identité de l'entreprise** — téléverser un logo, saisir l'adresse, les numéros RBQ, NEQ, TPS et TVQ. Ces renseignements apparaissent ensuite sur toutes les soumissions, factures et bons.
- **L'apparence** — deux systèmes distincts : les **couleurs des documents** générés (soumissions, factures, bons, courriels) et les **couleurs de l'interface** de l'ERP.
- **La fiscalité et la juridiction** — pays, devise, taxes de vente (TPS/TVQ ou autres), taux de retenue de garantie par défaut, état ou province, exercice fiscal, plus les renseignements employeur pour la paie et les catégories de fournisseurs.
- **La langue des documents** — français ou anglais.
- **Le fuseau horaire** — pour l'horodatage du pointage, des dates de création et des échéances.
- **L'abonnement** — souscrire, gérer ou annuler l'abonnement, consulter et recharger les crédits IA (via Stripe).
- **Les intégrations comptables** — brancher QuickBooks Online ou Sage (module séparé embarqué).

### 1.2 Ce que le module fait — et ne fait pas

| Le module **fait** | Le module **ne fait pas** |
|---|---|
| Créer, modifier et **désactiver** des utilisateurs | **Supprimer définitivement** un utilisateur (désactivation seulement) |
| Attribuer des droits d'administrateur | Modifier le **nom d'utilisateur** après la création |
| Personnaliser les couleurs des documents (en base) | Modifier les documents déjà envoyés (ils gardent leur rendu au moment de l'envoi) |
| Personnaliser les couleurs de l'interface (par navigateur) | Synchroniser les couleurs d'interface entre vos appareils |
| Régler taxes, devise, pays, retenue, exercice fiscal | Changer le **pays** une fois que vous avez des factures ou des écritures comptables |
| Souscrire, gérer et annuler l'abonnement Stripe | Facturer votre carte sans passer par Stripe |
| Recharger les crédits IA (paiement réel) | Rembourser des crédits IA depuis l'ERP |
| Brancher QuickBooks / Sage (onglet Intégrations) | Gérer des **points d'ancrage (webhooks)** depuis la Configuration (aucun onglet actif) |

### 1.3 Accès par le menu latéral

- **Menu latéral** → section **Système** → **Configuration** (icône `Settings`).
- **Adresse** : `/configuration` (`App.tsx:257`).
- **Onglet ouvert par défaut** : **Profil**.
- **Ouverture directe de l'abonnement** : l'adresse `/configuration?tab=abonnement` ouvre directement l'onglet **Abonnement** (utilisé au retour d'un paiement Stripe).
- Page protégée : il faut être authentifié dans un tenant.

### 1.4 Qui voit quoi : administrateur ou non

Le module distingue deux profils d'utilisateur. Le signal d'administrateur est calculé ainsi (`ConfigurationPage.tsx:95`) :

> Vous êtes considéré comme **administrateur** si votre compte porte le drapeau `is_admin` (relu sur le serveur, donc infalsifiable), **ou** si votre rôle est `admin`, **ou** si vous êtes super-administrateur de la plateforme.

Six onglets sont **réservés aux administrateurs** (`adminOnly`) : **Utilisateurs**, **Apparence**, **Juridiction & Devise**, **Taxes**, **Préférences**, **Fuseau horaire**. Ils sont **masqués** aux utilisateurs ordinaires.

| Profil | Onglets visibles | Nombre |
|---|---|---|
| **Administrateur** | Les 11 onglets | **11** |
| **Utilisateur ordinaire** | Profil, Entreprise (en lecture seule), Interface, Abonnement, Intégrations | **5** |

> **Important** : masquer un onglet est un confort d'affichage, pas une sécurité. La vraie protection est **côté serveur** : chaque écriture (création d'utilisateur, changement de taxe, etc.) exige le droit d'administrateur, vérifié par la garde `require_tenant_admin_or_role` sur le serveur. Un utilisateur ordinaire qui atteindrait un de ces points d'accès se verrait refuser l'écriture.

### 1.5 Les rôles utilisateurs

À la création d'un compte, vous choisissez un **rôle** parmi cinq (`ConfigurationPage.tsx:72-78`) :

| Rôle | Libellé | Usage indicatif |
|---|---|---|
| `admin` | **Administrateur** | Accès complet à l'entreprise |
| `user` | **Utilisateur** | Accès métier standard |
| `employee` | **Employé** | Souvent lié à une fiche employé |
| `comptable` | **Comptable** | Orienté comptabilité et paie |
| `gestionnaire` | **Gestionnaire** | Orienté pilotage |

> **À savoir** : ces cinq rôles servent surtout à l'affichage et à quelques gardes ciblées (par exemple, le rôle **comptable** peut lire les renseignements employeur, et il ouvre les fonctions sensibles de la Comptabilité). Le **vrai** contrôle d'accès de la Configuration repose sur le drapeau **Administrateur** (`is_admin`), pas sur l'intitulé du rôle. Autrement dit, cochez « Administrateur » pour accorder les pleins droits, quel que soit le rôle affiché. Le serveur **refuse** en revanche d'attribuer le rôle réservé « super-administrateur » de la plateforme (message d'erreur 422) : ce statut ne se donne pas depuis un tenant.

### 1.6 Les 11 onglets

Source : tableau `TABS` (`ConfigurationPage.tsx:55-69`).

| # | Onglet | Réservé admin ? | Rôle |
|---|---|---|---|
| 1 | **Profil** | Non (chacun le sien) | Nom, courriel, mot de passe de l'utilisateur courant |
| 2 | **Utilisateurs** | **Oui** | Comptes et droits des employés |
| 3 | **Entreprise** | Non (édition réservée admin) | Logo, coordonnées, numéros ; configuration système |
| 4 | **Apparence** | **Oui** | Couleurs des documents générés |
| 5 | **Interface** | Non (par utilisateur) | Couleurs de l'interface de l'ERP |
| 6 | **Juridiction & Devise** | **Oui** | Pays, devise, retenue, état/province, exercice fiscal, employeur, catégories |
| 7 | **Taxes** | **Oui** | Deux taxes de vente configurables |
| 8 | **Préférences** | **Oui** | Langue des documents (FR / EN) |
| 9 | **Fuseau horaire** | **Oui** | Fuseau du tenant |
| 10 | **Abonnement** | Non | Abonnement Stripe et crédits IA |
| 11 | **Intégrations** | Non | QuickBooks / Sage (module séparé) |

### 1.7 Deux systèmes de couleurs à ne pas confondre

Le module comporte **deux** onglets de couleurs, indépendants l'un de l'autre :

| Aspect | Onglet **Apparence** | Onglet **Interface** |
|---|---|---|
| Ce qui change | Les **documents** générés (soumissions, factures, bons, courriels clients) | L'**interface** de l'ERP (barre latérale, barre du haut, boutons) |
| Portée | Toute l'entreprise (tous les utilisateurs) | **Votre compte, sur ce navigateur** seulement |
| Où c'est enregistré | En base de données (`document_theme`) | Dans le navigateur (mémoire locale), **sans** appel au serveur |
| Qui peut modifier | Administrateur seulement | Chaque utilisateur pour lui-même |
| Nombre de couleurs | **8** | **4** |

> Ni l'un ni l'autre ne touche au mode clair/sombre du système, qui est un réglage distinct de votre navigateur.

### 1.8 Mode consultation (lecture seule)

Le paiement de l'abonnement conditionne l'écriture dans tout l'ERP, y compris la Configuration. Un tenant **sans abonnement Stripe actif** (abonnement annulé, sans carte enregistrée) bascule en **mode consultation** : la lecture reste possible, mais **toutes les écritures de `/config/*` sont bloquées** (réponse « 403 — Mode consultation »).

Deux exceptions échappent au blocage, précisément pour vous laisser régulariser : les appels de **réabonnement et de paiement** (`/stripe/*`) et la **déconnexion** (`/auth/*`) restent autorisés. C'est pourquoi, même en mode consultation, l'onglet **Abonnement** demeure pleinement fonctionnel pour souscrire de nouveau ou recharger.

> Ce mécanisme est global à l'ERP (défini dans `erp_auth.py` et `erp_stripe.py`), pas propre à la Configuration. Il connaît trois états : **`full`** (tout autorisé), **`readonly`** (lecture seule + réabonnement) et **`blocked`** (entreprise désactivée → session coupée). Sur une instance interne où la facturation est désactivée (`ERP_BILLING_ENABLED=false`) ou en mode développement, tout est en `full`.

---

## 2. Interface

### 2.1 Disposition générale

En haut, le titre **« Configuration »**. Juste en dessous, une **barre d'onglets** horizontale déroulante ; les onglets réservés aux administrateurs n'y figurent pas pour un utilisateur ordinaire. Le contenu change selon l'onglet actif.

Deux **bannières** peuvent apparaître au-dessus du contenu :
- une bannière **rouge** en cas d'erreur ;
- une bannière **verte** en cas de succès, qui **disparaît d'elle-même après 3 secondes**.

Sur l'onglet **Abonnement**, ces bannières proviennent du sous-système Stripe (elles reflètent les messages de paiement).

### 2.2 Onglet « Profil »

*Accessible à tous.* Deux cartes côte à côte.

**Carte « Informations personnelles »**

| Élément | Détail |
|---|---|
| Badge de rôle + `@nom_utilisateur` | En lecture seule (le nom d'utilisateur ne se modifie pas). |
| **Nom complet** | Champ texte modifiable. |
| **Courriel** | Champ courriel modifiable. |
| Bouton **Enregistrer** | Sauvegarde le nom et le courriel. Protégé contre le double-clic (roue animée pendant l'envoi). |

**Carte « Changer le mot de passe »**

| Élément | Détail |
|---|---|
| **Nouveau mot de passe** | Indication affichée : « Min. 6 caractères ». |
| **Confirmer le mot de passe** | Doit être identique au précédent. |
| Bouton **Modifier le mot de passe** | Désactivé tant que les deux champs sont vides. |

Contrôles à la saisie : moins de 6 caractères → « Le mot de passe doit avoir au moins 6 caractères » ; les deux champs diffèrent → « Les mots de passe ne correspondent pas ». Pendant le chargement initial, la page affiche « Chargement du profil... ».

### 2.3 Onglet « Utilisateurs »

*Réservé aux administrateurs.* Entête de carte : **« Utilisateurs ({{nombre}}) »** avec deux boutons — **Rafraîchir** et **Nouvel utilisateur**.

**Tableau (vue ordinateur)** — colonnes :

| Colonne | Contenu |
|---|---|
| **Utilisateur** | Nom d'utilisateur ; une icône `Shield` violette signale un administrateur. |
| **Nom** | Nom complet. |
| **Courriel** | Adresse. |
| **Rôle** | Badge (Administrateur / Utilisateur / Employé / Comptable / Gestionnaire). |
| **Statut** | Badge **Actif** ou **Inactif**. |
| **Actions** | Icônes (voir ci-dessous). |

**Actions par ligne :**
- **Modifier** (icône `Edit3`) — ouvre la fenêtre d'édition.
- **Changer le mot de passe** (icône `Key`).
- **Désactiver** (icône `XCircle`) — affichée **seulement** si la ligne n'est pas la vôtre et que le compte est actif. Une confirmation s'affiche : « Désactiver l'utilisateur {{nom}} ? ».

En vue mobile, chaque utilisateur est présenté sous forme de carte équivalente. Si la liste est vide : « Aucun utilisateur ».

**Trois fenêtres (modales) :**

**a) Nouvel utilisateur**

| Champ | Obligatoire | Détail |
|---|---|---|
| **Nom d'utilisateur** | Oui | Exemple affiché : « ex: jdupont ». Unique dans l'entreprise. |
| **Mot de passe** | Oui | Minimum 6 caractères. |
| **Nom complet** | Non | |
| **Courriel** | Non | |
| **Rôle** | Non | Menu déroulant à cinq valeurs. |
| **Administrateur** | Non | Case à cocher (accorde les pleins droits). |

Boutons **Annuler** / **Créer** (le bouton Créer reste désactivé tant que le nom d'utilisateur ou le mot de passe sont vides). Le compte est créé **actif**.

**b) Modifier : {{nom}}** — Nom complet, Courriel, Rôle, case **Administrateur**. Boutons **Annuler** / **Enregistrer**. *Le nom d'utilisateur n'y figure pas : il n'est pas modifiable.*

**c) Changer le mot de passe** — Nouveau mot de passe, Confirmer. Mêmes contrôles (6 caractères, correspondance). Boutons **Annuler** / **Modifier**.

**Deux garde-fous du serveur** empêchent de vous verrouiller dehors :
- Impossible de **retirer le statut d'administrateur** au **dernier administrateur** de l'entreprise.
- Impossible de **désactiver le dernier administrateur actif**.

Dans les deux cas, l'opération est refusée avec un message explicite. La désactivation est une **suppression douce** (le compte passe « Inactif », il n'est jamais effacé) ; vous ne pouvez pas non plus vous désactiver vous-même.

### 2.4 Onglet « Entreprise »

*Visible par tous ; modifiable par les administrateurs seulement.* Deux sous-onglets : **Informations entreprise** (par défaut) et **Configuration système**.

Pour un utilisateur non-administrateur, une bande ambre s'affiche : **« Lecture seule : seul un administrateur de l'entreprise peut modifier ces informations. »** ; tous les champs sont alors verrouillés.

**a) Informations entreprise**

**Logo.** Aperçu (image ou vignette vide), boutons **Télécharger** / **Changer** et **Retirer**. Contraintes à la sélection : **taille maximale 1 Mo**, formats **PNG, JPG, GIF, SVG, WEBP**. Note affichée : « PNG, JPG, SVG. Max 1 Mo. Recommandé : 500x200px, fond transparent. » Le logo est converti en base64 et enregistré dans la configuration ; il est ensuite repris sur vos documents.

**Douze champs texte** (`ENTREPRISE_INFO_FIELDS`) :

| Champ | Exemple affiché |
|---|---|
| **Nom de l'entreprise** | Construction ABC Inc. |
| **Adresse** | |
| **Ville** | |
| **Province** | |
| **Code postal** | H1A 2B3 |
| **Téléphone** | |
| **Courriel** | |
| **Site web** | |
| **Numéro RBQ** | 5734-1234-01 |
| **Numéro NEQ** | |
| **Numéro TPS** | 123456789 RT0001 |
| **Numéro TVQ** | 1234567890 TQ0001 |

Le bouton **Sauvegarder** ne s'active que s'il y a des changements ; seuls les champs modifiés sont enregistrés (un par un). Message de succès : « Informations sauvegardées ».

**b) Configuration système**

Éditeur avancé des entrées de configuration en clé/valeur, regroupées par catégorie (**General**, **Facturation**, **IA**, **Notifications**). Chaque ligne montre la clé (en style code), un badge de catégorie, une description facultative, un champ texte et un bouton **Sauver** (qui affiche une coche verte 2 secondes après enregistrement). S'il n'y a rien : « Aucune configuration trouvée. Les entrées seront créées automatiquement par le système. »

> **Deux clés protégées** : `document_theme` et `supplier_categories` **ne peuvent pas** être modifiées ici (le serveur refuse avec un message de redirection). Passez plutôt par l'onglet **Apparence** (thème des documents) et par la carte **Catégories de fournisseurs** (onglet Juridiction & Devise).

### 2.5 Onglet « Apparence » (couleurs des documents)

*Réservé aux administrateurs.* Titre **« Apparence des documents »**. Règle les couleurs de tous les **documents HTML** générés : soumissions, factures, bons de commande, bons de travail et courriels envoyés aux clients.

**Six thèmes prédéfinis**, cliquables (une coche marque le thème actif) :

| Thème | Couleur principale |
|---|---|
| **Constructo Bleu** (défaut) | `#1F4E79` |
| **Vert Forêt** | `#166534` |
| **Rouge Brique** | `#991B1B` |
| **Anthracite** | `#1F2937` |
| **Bourgogne** | `#7F1D1D` |
| **Océan** | `#0C4A6E` |

**Personnalisation avancée — 8 couleurs**, chacune avec un sélecteur de couleur, un champ hexadécimal et une indication d'usage :

| Couleur | Clé technique | Usage |
|---|---|---|
| **Couleur principale** | `primary` | Entêtes, bandeau titre, entêtes de tableaux |
| **Principale — foncée** | `primary_dark` | Variante foncée (survol, bordures importantes) |
| **Accent** | `accent` | Sous-titres, bordure gauche des encadrés |
| **Accent — clair** | `accent_light` | Numéro de document sur l'entête |
| **Texte entête** | `header_text` | Texte sur le fond principal |
| **Lignes alternées** | `table_row_alt` | Fond des lignes paires de tableau |
| **Fond sections info** | `info_bg` | Fond des encadrés et des totaux |
| **Bordures** | `border` | Filets fins (lignes de tableau, séparateurs) |

Chaque champ valide le format hexadécimal (`#RGB` ou `#RRGGBB`) ; une bordure rouge signale une valeur incorrecte. Un **aperçu en direct** montre une maquette de devis (entête, encadrés, tableau, total) qui se met à jour à mesure que vous changez les couleurs.

Actions : **Enregistrer** (actif s'il y a des changements et que toutes les couleurs sont valides), **Annuler les changements**, et le lien **Réinitialiser aux défauts** (avec confirmation).

### 2.6 Onglet « Interface » (couleurs de l'ERP)

*Accessible à tous, réglage par utilisateur.* Titre **« Couleurs de l'interface »**, sous-titre rappelant que « ce choix s'applique à votre compte sur ce navigateur ». Ce réglage est enregistré **localement dans le navigateur** (aucun appel au serveur) et appliqué **immédiatement**.

Un **avertissement de contraste** (non bloquant) apparaît si vos couleurs rendent le texte peu lisible (vérification d'accessibilité WCAG).

**Six thèmes prédéfinis** portant les mêmes noms que pour les documents (Constructo Bleu, Vert Forêt, Rouge Brique, Anthracite, Bourgogne, Océan).

**Quatre couleurs** :

| Couleur | Défaut (thème D365) | Usage |
|---|---|---|
| **Couleur principale** | `#0078D4` | Boutons, onglet actif, liens |
| **Principale — survol** | `#005EA2` | Survol de la couleur principale |
| **Barre latérale** | `#002050` | Fond de la barre latérale |
| **Barre supérieure** | `#002B6B` | Fond de la barre du haut |

**Aperçu** : une maquette d'interface (barre latérale, barre du haut, onglet actif, bouton, lien). Actions : **Enregistrer**, **Annuler les changements**, **Réinitialiser aux défauts** (avec confirmation).

> Ce réglage n'est **pas** partagé : il ne suit pas votre compte d'un appareil ou d'un navigateur à l'autre. Sur un nouvel appareil, vous repartez des couleurs par défaut.

### 2.7 Onglet « Juridiction & Devise »

*Réservé aux administrateurs.* Cet onglet empile **cinq cartes de juridiction**, puis la carte **Renseignements employeur (paie)** et la carte **Catégories de fournisseurs**. Chaque carte possède ses propres messages d'erreur et de succès et son propre bouton d'enregistrement.

**Carte 1 — Juridiction & Devise**
- **Pays** : Canada ou États-Unis.
- **Devise** : Dollar canadien (CAD) ou Dollar américain (USD).
- Affiche le code ISO enregistré. Bouton **Enregistrer la juridiction**.

> **Attention** : le changement de **pays** est **refusé** si votre entreprise possède déjà des factures ou des écritures comptables (message « 409 »). Motif : le pays détermine les libellés fiscaux (TPS/TVQ au Canada, Sales Tax aux États-Unis), et il ne faut pas rendre incohérent l'historique déjà produit. Fixez le pays **avant** de commencer à facturer.

**Carte 2 — Taux de retenue par défaut**
- Champ numérique **Taux de retenue par défaut (%)** (0 à 100, pas de 0,5). Indication : « Standard CA : 10 % (LCS Québec)... ». Bouton **Enregistrer le taux**. Ce taux alimente par défaut les retenues de garantie de la comptabilité et des décomptes.

**Carte 3 — Juridiction (État US / Province CA)**
- Menu déroulant avec l'option « — Non spécifié — », puis deux groupes : **États américains** (50 + District de Columbia) et **Provinces canadiennes** (13), affichés en toutes lettres. Bouton **Enregistrer la juridiction**.

**Carte 4 — Exercice fiscal**
- **Mois de début** (menu des 12 mois) et **Jour de début** (le maximum dépend du mois ; le 29 février est accepté par convention récurrente). Un aperçu affiche « Exercice : {{début}} au {{fin}} ». Par défaut, l'exercice suit l'année civile (1er janvier). Bouton **Enregistrer l'exercice fiscal**.

**Carte 5 — Note importante** : quatre rappels (le pays touche le plan comptable ; les taxes se règlent à part dans l'onglet Taxes ; la devise ; ces valeurs s'appliquent à toute l'entreprise).

**Carte « Renseignements employeur (paie) »** — prérequis pour produire les feuillets de fin d'année T4 et RL-1. Neuf champs, **tous facultatifs** :

| Champ | Exemple |
|---|---|
| **Numéro d'employeur ARC (paie)** | 123456789RP0001 |
| **Numéro d'identification Revenu Québec** | 1234567890RS0001 |
| **Numéro CNESST** | |
| **Classification CNESST** | |
| **Numéro CCQ** | Si assujetti à la CCQ |
| **Adresse** (légale de l'employeur) | |
| **Ville** | |
| **Code postal** | H2X 1Y4 |
| **Province** | QC (2 lettres) |

Boutons **Enregistrer** / **Annuler les modifications**. *La lecture de cette carte est aussi ouverte au rôle **comptable**.*

**Carte « Catégories de fournisseurs »** — liste des catégories proposées au champ « Secteur d'activité » quand vous créez un fournisseur (pour classer vos dépenses). Un champ + bouton **Ajouter** (les doublons, majuscules ou minuscules confondues, sont ignorés), une liste modifiable en ligne avec suppression (icône `Trash2`), un compteur « {{n}} catégorie(s) », puis **Enregistrer** / **Réinitialiser aux valeurs par défaut**. Une liste de 30 catégories est fournie par défaut.

### 2.8 Onglet « Taxes »

*Réservé aux administrateurs.* Titre **« Configuration des taxes »**. Par défaut au Québec : TPS 5 % et TVQ 9,975 %. **Un libellé vide ou un taux à 0 masque la taxe** ; les documents déjà produits conservent leurs taxes historiques.

Quatre champs :

| Champ | Détail |
|---|---|
| **Taxe 1 — Libellé** | Exemple : « TPS / GST / Sales Tax (vide pour masquer) ». Maximum 50 caractères. |
| **Taxe 1 — Taux (%)** | Nombre de 0 à 100 (pas de 0,001). |
| **Taxe 2 — Libellé** | Exemple : « TVQ / PST... ». |
| **Taxe 2 — Taux (%)** | Nombre de 0 à 100. |

Les taux sont bornés à l'intervalle 0-100 et arrondis à 3 décimales. Bouton **Sauvegarder** (actif s'il y a des changements).

Carte **« Exemples de configurations »** — un tableau de référence :

| Juridiction | Taxe 1 | Taxe 2 |
|---|---|---|
| Québec | TPS 5 % | TVQ 9,975 % |
| Ontario | HST 13 % | — |
| Colombie-Britannique | GST 5 % | PST 7 % |
| Alberta | GST 5 % | — |
| USA (ex. Californie) | Sales Tax 8,25 % | — |
| Exonéré | — | — |

### 2.9 Onglet « Préférences » (langue des documents)

*Réservé aux administrateurs.* Titre **« Langue des documents générés »**. Deux boutons radio présentés en cartes :

- **Français** — « Par défaut pour le Québec... Format devise : 1 234,56 $ ».
- **Anglais** — « Recommandé pour les États-Unis... Format devise : $1,234.56 ».

Bouton **Sauvegarder** (actif au changement). Carte **« Note importante »** avec trois rappels : les documents historiques sont régénérés à la volée dans la nouvelle langue ; les libellés de taxes restent indépendants ; les textes que vous avez personnalisés ne sont pas traduits automatiquement.

### 2.10 Onglet « Fuseau horaire »

*Réservé aux administrateurs.* Titre **« Fuseau horaire du tenant »**. Sert à l'horodatage du **pointage**, des **dates de création**, des **échéances** et des **rapports**.

Menu déroulant de **13 fuseaux** (zones IANA), avec la valeur enregistrée affichée en dessous :

| Groupe | Fuseaux |
|---|---|
| **Canada** (6) | Toronto/Montréal, Halifax, St. John's, Winnipeg, Edmonton, Vancouver |
| **États-Unis** (6) | New York, Chicago, Denver, Phoenix (sans heure d'été), Los Angeles, Anchorage |
| **Pacifique** (1) | Honolulu |

Bouton **Sauvegarder** (actif au changement). Carte **« Note importante »** : les dates sont stockées en temps universel (UTC) en base ; l'effet est immédiat ; rappels particuliers pour Phoenix et la Saskatchewan (pas d'heure avancée).

### 2.11 Onglet « Abonnement »

*Accessible à tous.* Piloté par Stripe. Deux cartes.

**Carte « Abonnement »** (bouton **Rafraîchir**). Si un abonnement existe, trois tuiles :

| Tuile | Contenu |
|---|---|
| **Statut** | Badge coloré ; mention « Sera annulé le {{date}} » si une annulation est programmée. |
| **Plan** | Nom du plan + « {{montant}} $ / mois » ou « / an ». |
| **Renouvellement** | Date ; mention « Fin de l'essai : {{date}} » si vous êtes en période d'essai. |

Boutons : **Gérer mon abonnement** (ouvre le **portail client Stripe** : carte, factures, changement de plan), **Annuler** (si l'abonnement est actif et pas déjà en annulation). S'il n'y a **aucun** abonnement : « Aucun abonnement actif » + « Souscrivez à un plan... » + bouton **Souscrire maintenant** (paiement Stripe du plan `pro`).

**Carte « Crédits IA »** — trois tuiles :

| Tuile | Contenu |
|---|---|
| **Solde actuel** | Montant en $, ou **« Illimité »** (en vert) si votre plan est exempté. |
| **Utilisation ce mois** | Montant consommé. |
| **Type de plan** | Nom du plan ; badge « Crédits illimités » si exempté. |

Si vous n'êtes pas exempté, des avertissements apparaissent : **« Solde bas... »** (sous 2 $) et **« Crédits IA épuisés... »** (à 0 ou moins), avec un bouton **Recharger**.

**Fenêtre d'annulation** — titre « Annuler l'abonnement », encadré « Attention », message « ...restera actif jusqu'à la fin de la période ({{date}})... », boutons **Conserver** / **Confirmer l'annulation**. L'annulation prend effet à la **fin de la période en cours** (vous gardez l'accès jusque-là).

**Fenêtre de recharge** — titre « Recharger les crédits IA », six montants rapides (**10 / 25 / 50 / 100 / 200 / 500 $**), un champ « Montant personnalisé ($) » (indication « Min. 5,00 »), un bouton **Recharger {{montant}} $**. Montant accepté : **entre 5 $ et 500 $**.

> **La recharge est un vrai paiement.** Elle débite immédiatement la carte enregistrée dans votre compte Stripe (devise par défaut : dollar canadien). Le système **facture d'abord, puis crédite** de façon idempotente (jamais de double-crédit). Si la carte est refusée, un message clair s'affiche (erreur « 402 ») et **aucun crédit n'est ajouté**.

**Statuts d'abonnement possibles** : Actif, Essai gratuit, Paiement en retard, Annulé, Annulation en cours, Incomplet, Expiré, Impayé.

### 2.12 Onglet « Intégrations »

*Accessible à tous.* Cet onglet **embarque un module séparé** (chargé à la demande) : la gestion des intégrations comptables. Il possède ses **sept propres sous-onglets** :

| Sous-onglet | Rôle |
|---|---|
| **Vue d'ensemble** | État des connexions. |
| **QuickBooks** | Connexion OAuth à QuickBooks Online + synchronisations. |
| **Sage 50** | Connexion au connecteur Sage 50. |
| **Sage** | Sage Business Cloud Accounting (REST/OAuth). |
| **Webhooks** | Points d'ancrage sortants **du module Intégrations** (c'est **ici** que se trouvent les webhooks visibles). |
| **Correspondances** | Mappage des comptes entre l'ERP et le logiciel comptable. |
| **Historique** | Journaux des synchronisations. |

> Le détail de ces sous-onglets relève du **manuel des Intégrations**. Retenez seulement que la Configuration ne fait que **loger** cette page ; elle n'ajoute aucun réglage propre.

---

## 3. Procédures pas à pas

### 3.1 Modifier son nom ou son courriel

1. **Configuration** → onglet **Profil**.
2. Dans « Informations personnelles », modifiez **Nom complet** et/ou **Courriel**.
3. Cliquez **Enregistrer**. Bannière verte de confirmation.

### 3.2 Changer son propre mot de passe

1. Onglet **Profil** → carte « Changer le mot de passe ».
2. Saisissez le **Nouveau mot de passe** (au moins 6 caractères) puis **Confirmer**.
3. Cliquez **Modifier le mot de passe**.

### 3.3 Créer un compte pour un employé

1. Onglet **Utilisateurs** (administrateur) → bouton **Nouvel utilisateur**.
2. Saisissez le **Nom d'utilisateur** (unique) et un **Mot de passe** (6 caractères minimum) — les deux sont obligatoires.
3. Complétez le **Nom complet** et le **Courriel** (facultatifs).
4. Choisissez un **Rôle** ; cochez **Administrateur** si la personne doit avoir les pleins droits.
5. Cliquez **Créer**. Le compte apparaît dans la liste, à l'état **Actif**.

### 3.4 Modifier le rôle ou les droits d'un utilisateur

1. Onglet **Utilisateurs** → icône **Modifier** sur la ligne voulue.
2. Ajustez le **Rôle** et/ou la case **Administrateur**.
3. Cliquez **Enregistrer**.

> Si vous tentez de retirer le statut d'administrateur au **dernier** administrateur, l'opération est refusée : ajoutez d'abord un autre administrateur.

### 3.5 Réinitialiser le mot de passe d'un employé

1. Onglet **Utilisateurs** → icône **Changer le mot de passe** sur la ligne.
2. Saisissez et confirmez le nouveau mot de passe (6 caractères minimum).
3. Cliquez **Modifier**. Communiquez ensuite le mot de passe à l'employé de façon sécuritaire.

### 3.6 Désactiver un utilisateur qui quitte l'entreprise

1. Onglet **Utilisateurs** → icône **Désactiver** sur la ligne (absente sur votre propre ligne).
2. Confirmez « Désactiver l'utilisateur {{nom}} ? ».
3. Le compte passe **Inactif** ; il ne peut plus se connecter, mais son historique reste intact.

> Le dernier administrateur actif ne peut pas être désactivé. Il n'existe pas de suppression définitive : la désactivation est réversible (réactivez en modifiant le compte).

### 3.7 Téléverser le logo et saisir les coordonnées

1. Onglet **Entreprise** → sous-onglet **Informations entreprise** (administrateur).
2. Cliquez **Télécharger** (ou **Changer**) et choisissez une image **PNG/JPG/GIF/SVG/WEBP** de **1 Mo au maximum** (idéalement 500 x 200 px, fond transparent).
3. Remplissez les 12 champs (nom, adresse, téléphone, RBQ, NEQ, TPS, TVQ, etc.).
4. Cliquez **Sauvegarder**. Ces renseignements apparaîtront sur vos soumissions, factures et bons.

### 3.8 Personnaliser les couleurs des documents

1. Onglet **Apparence** (administrateur).
2. Cliquez un **thème prédéfini**, ou ajustez les **8 couleurs** à la main (sélecteur ou code hexadécimal).
3. Vérifiez le rendu dans l'**aperçu en direct**.
4. Cliquez **Enregistrer**. Pour revenir au thème d'origine, utilisez **Réinitialiser aux défauts**.

### 3.9 Personnaliser les couleurs de l'interface

1. Onglet **Interface** (chacun pour soi).
2. Choisissez un thème ou ajustez les **4 couleurs**. Tenez compte de l'avertissement de contraste s'il apparaît.
3. Cliquez **Enregistrer**. Le changement est immédiat et **propre à ce navigateur**.

### 3.10 Configurer pays, devise et taxes (à faire en premier)

1. Onglet **Juridiction & Devise** → carte 1 : choisissez **Pays** et **Devise**, puis **Enregistrer la juridiction**.
2. Onglet **Taxes** : saisissez le **libellé** et le **taux** de chaque taxe (par exemple TPS 5 % et TVQ 9,975 %), puis **Sauvegarder**. Laissez un libellé vide pour masquer une taxe.
3. Faites-le **avant** d'émettre vos premières factures : le pays ne pourra plus changer par la suite.

### 3.11 Régler le taux de retenue de garantie

1. Onglet **Juridiction & Devise** → carte 2.
2. Saisissez le **Taux de retenue par défaut (%)** (10 % est le standard au Québec).
3. Cliquez **Enregistrer le taux**.

### 3.12 Définir l'exercice fiscal

1. Onglet **Juridiction & Devise** → carte 4.
2. Choisissez le **Mois de début** et le **Jour de début** ; vérifiez l'aperçu « Exercice : ... au ... ».
3. Cliquez **Enregistrer l'exercice fiscal**.

### 3.13 Saisir les renseignements employeur (préparation des feuillets)

1. Onglet **Juridiction & Devise** → carte **Renseignements employeur (paie)**.
2. Renseignez les numéros connus (ARC, Revenu Québec, CNESST, CCQ) et l'adresse légale. Tout est facultatif et peut être complété plus tard.
3. Cliquez **Enregistrer**. Ces données servent à produire les T4 et RL-1 (module Pointage et Paie).

### 3.14 Gérer les catégories de fournisseurs

1. Onglet **Juridiction & Devise** → carte **Catégories de fournisseurs**.
2. Saisissez une catégorie et cliquez **Ajouter** (les doublons sont ignorés) ; supprimez celles dont vous n'avez pas besoin.
3. Cliquez **Enregistrer**. Pour repartir de zéro, **Réinitialiser aux valeurs par défaut**.

### 3.15 Choisir la langue des documents

1. Onglet **Préférences** (administrateur).
2. Sélectionnez **Français** ou **Anglais**.
3. Cliquez **Sauvegarder**. Les documents seront désormais générés dans cette langue.

### 3.16 Régler le fuseau horaire

1. Onglet **Fuseau horaire** (administrateur).
2. Choisissez la zone (par exemple « Toronto/Montréal »).
3. Cliquez **Sauvegarder**. L'effet est immédiat sur les horodatages.

### 3.17 Souscrire à un abonnement

1. Onglet **Abonnement**. S'il n'y a pas d'abonnement actif, cliquez **Souscrire maintenant**.
2. Vous êtes redirigé vers la page de paiement **Stripe** ; réglez le plan.
3. Au retour, l'ERP rouvre l'onglet **Abonnement** ; cliquez **Rafraîchir** au besoin pour voir le statut à jour.

### 3.18 Gérer ou annuler l'abonnement

1. Onglet **Abonnement** → **Gérer mon abonnement** pour ouvrir le **portail Stripe** (changer de carte, télécharger les factures, changer de plan).
2. Pour arrêter : bouton **Annuler** → **Confirmer l'annulation**. L'accès reste actif jusqu'à la **fin de la période** déjà payée, puis l'entreprise passe en **mode consultation**.

### 3.19 Recharger les crédits IA

1. Onglet **Abonnement** → carte **Crédits IA** → **Recharger**.
2. Choisissez un montant rapide (10 à 500 $) ou saisissez un **montant personnalisé** (entre 5 $ et 500 $).
3. Cliquez **Recharger {{montant}} $**. La carte enregistrée est débitée, puis le solde est crédité. En cas de refus de carte, aucun crédit n'est ajouté.

### 3.20 Brancher un logiciel comptable

1. Onglet **Intégrations** → sous-onglet **QuickBooks** (ou **Sage 50** / **Sage**).
2. Lancez la connexion (OAuth pour QuickBooks et Sage Business Cloud) et suivez les correspondances de comptes.
3. Consultez l'**Historique** pour vérifier les synchronisations. (Détails dans le manuel des Intégrations.)

---

## 4. Référence

### 4.1 Points d'accès de la Configuration (`/api/erp/v1/config`)

Sauf mention contraire, la **lecture** (GET) est ouverte à tout utilisateur du tenant et l'**écriture** exige le droit d'administrateur (`require_tenant_admin_or_role`).

| Méthode | URL | Rôle | Objet |
|---|---|---|---|
| GET | `/config/profile` | Soi-même | Lire son profil |
| PUT | `/config/profile` | Soi-même | Modifier nom / courriel |
| GET | `/config/users` | Admin | Liste des utilisateurs |
| POST | `/config/users` | Admin | Créer un utilisateur |
| PUT | `/config/users/{id}` | Admin | Modifier (rôle, droits, courriel) |
| PUT | `/config/users/{id}/password` | Soi-même **ou** admin | Changer un mot de passe |
| DELETE | `/config/users/{id}` | Admin | Désactiver (suppression douce) |
| GET | `/config/entreprise` | Tous | Lire la configuration |
| PUT | `/config/entreprise/{cle}` | Admin | Écrire une clé (logo, coordonnées, clés libres) |
| GET / PUT / DELETE | `/config/document-theme` | GET tous / écriture admin | Thème des documents |
| GET / PUT | `/config/tax-config` | GET tous / PUT admin | Taxes de vente |
| GET / PUT | `/config/document-language` | GET tous / PUT admin | Langue (fr/en) |
| GET / PUT | `/config/timezone` | GET tous / PUT admin | Fuseau horaire |
| GET / PUT | `/config/country` | GET tous / PUT admin | Pays (CA/US) |
| GET / PUT | `/config/currency` | GET tous / PUT admin | Devise (CAD/USD) |
| GET / PUT | `/config/retainage` | GET tous / PUT admin | Taux de retenue |
| GET / PUT | `/config/fiscal-year` | GET tous / PUT admin | Exercice fiscal |
| GET / PUT | `/config/juridiction` | GET tous / PUT admin | État US / province CA |
| GET / PUT | `/config/supplier-categories` | GET tous / PUT admin | Catégories de fournisseurs |
| GET / PUT | `/config/employer-payroll` | GET admin **ou comptable** / PUT admin | Renseignements employeur |
| GET POST PUT DELETE | `/config/webhooks[...]` | Admin | Points d'ancrage sortants — **sans interface** (voir 5.4) |

### 4.2 Points d'accès de l'abonnement (`/api/erp/v1/stripe`)

Tous exigent seulement une session authentifiée (l'entreprise est résolue depuis votre compte).

| Méthode | URL | Objet |
|---|---|---|
| POST | `/stripe/checkout` | Créer une session de paiement d'abonnement |
| GET | `/stripe/subscription` | Détails de l'abonnement (source Stripe) |
| POST | `/stripe/portal` | Ouvrir le portail client Stripe |
| POST | `/stripe/cancel` | Programmer l'annulation en fin de période |
| GET | `/stripe/credits` | Solde de crédits IA + usage du mois |
| POST | `/stripe/credits/recharge` | Recharger (paiement réel, 5-500 $) |

### 4.3 Statuts d'abonnement

| Statut | Signification | Accès à l'ERP |
|---|---|---|
| **Actif** | Abonnement payé et en règle | Complet |
| **Essai gratuit** | Période d'essai en cours | Complet |
| **Paiement en retard** | Échec de prélèvement, période de grâce | Complet (à régulariser) |
| **Annulation en cours** | Annulation programmée en fin de période | Complet jusqu'à la date |
| **Annulé** | Abonnement terminé | **Mode consultation** |
| **Incomplet** | Paiement initial non finalisé | **Mode consultation** |
| **Expiré** / **Impayé** | Fin d'accès | **Mode consultation** |

### 4.4 Limites et validations

| Règle | Valeur / effet |
|---|---|
| Mot de passe | **6 caractères minimum** (création et changement) |
| Nom d'utilisateur | Unique dans l'entreprise (sinon refus) ; **non modifiable** après création |
| Dernier administrateur | Ne peut être **ni** rétrogradé **ni** désactivé |
| Auto-désactivation | Interdite (vous ne pouvez pas désactiver votre propre compte) |
| Rôle « super-administrateur » | Refusé à la création/modification (message 422) |
| Logo | **1 Mo maximum** ; PNG, JPG, GIF, SVG, WEBP |
| Couleur de thème | Format hexadécimal `#RGB` ou `#RRGGBB` |
| Taux de taxe | 0 à 100, arrondi à 3 décimales ; libellé ≤ 50 caractères |
| Taux de retenue | 0 à 100 % |
| Exercice fiscal | Mois 1-12, jour valide selon le mois (29 février toléré) |
| Pays | CA ou US ; **changement refusé** si factures/écritures existent |
| Devise | CAD ou USD |
| Fuseau horaire | Parmi les 13 zones autorisées |
| Catégories de fournisseurs | 80 caractères par entrée, 200 entrées au maximum |
| Recharge de crédits | **5 $ à 500 $** ; paiement réel, idempotent |
| Langue | `fr` ou `en` |

### 4.5 Les 8 couleurs des documents (onglet Apparence)

Valeurs par défaut = thème **Constructo Bleu** (source `html_utils.py`).

| Clé | Défaut | Rôle |
|---|---|---|
| `primary` | `#1F4E79` | Entêtes, bandeau titre, entêtes de tableaux |
| `primary_dark` | `#163A5C` | Variante foncée |
| `accent` | `#2563EB` | Sous-titres, bordure des encadrés |
| `accent_light` | `#93C5FD` | Numéro de document sur l'entête |
| `header_text` | `#FFFFFF` | Texte sur fond principal |
| `table_row_alt` | `#F8F9FA` | Alternance des lignes de tableau |
| `info_bg` | `#F8FAFC` | Fond des encadrés et totaux |
| `border` | `#E9ECEF` | Filets fins |

### 4.6 Les 4 couleurs de l'interface (onglet Interface)

Valeurs par défaut = thème **D365** (source `useUiThemeStore.ts`), enregistrées dans le navigateur.

| Couleur | Défaut | Rôle |
|---|---|---|
| Couleur principale | `#0078D4` | Boutons, onglet actif, liens |
| Principale — survol | `#005EA2` | Survol |
| Barre latérale | `#002050` | Fond de la barre latérale |
| Barre supérieure | `#002B6B` | Fond de la barre du haut |

### 4.7 Comportements défensifs du serveur (bon à savoir)

- **Lecture toujours tolérante** : les GET de configuration ne plantent jamais ; ils renvoient des valeurs par défaut en cas de souci.
- **Colonne non encore migrée** : certains réglages (taxes, langue, fuseau, pays, devise, retenue, juridiction) peuvent répondre « 503 » avec le nom du script de migration si la base n'a pas encore la colonne. L'exercice fiscal et les renseignements employeur, eux, **se réparent tout seuls** (création automatique de la colonne ou de la table).
- **Écritures concurrentes** : les modifications de la configuration en JSON sont sérialisées (verrou), pour qu'un enregistrement n'en écrase pas un autre.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Réglage de la Configuration | Où il agit |
|---|---|
| Logo, coordonnées, RBQ, TPS, TVQ | Entêtes des **soumissions**, **factures**, **bons de commande**, **bons de travail** et **courriels** |
| Thème des documents (Apparence) | Rendu de tous les **documents HTML/PDF** générés |
| Taxes (TPS/TVQ ou autres) | **Ventes**, **Soumissions**, **Comptabilité** (calcul des taxes) |
| Pays / devise / juridiction | **Comptabilité** (plan comptable, déclaration de taxes), documents |
| Taux de retenue | **Comptabilité** (retenues de garantie), **Décomptes** |
| Exercice fiscal | **Comptabilité** (périodes, états financiers) |
| Renseignements employeur | **Pointage et Paie** (feuillets T4, RL-1, remise PD7A) |
| Catégories de fournisseurs | **Fournisseurs / Achats**, ventilation des **dépenses** |
| Fuseau horaire | **Pointage**, dates de création, échéances, rapports |
| Utilisateurs et rôles | Toute la plateforme (accès et permissions) |
| Abonnement et crédits IA | Toutes les **fonctions IA** (Assistant, Estimation, Web, etc.) |

### 5.2 Le cas des webhooks

Vous entendrez peut-être parler de « points d'ancrage » ou « webhooks ». Il faut distinguer deux choses :

- **Dans la Configuration** : il n'y a **aucun onglet Webhooks actif**. Le serveur possède bien les fonctions correspondantes (création, test, historique, avec signature HMAC et protection anti-SSRF durcie), mais **aucune interface n'y est branchée** dans cette page. N'en tenez pas compte comme utilisateur.
- **Dans l'onglet Intégrations** : le sous-onglet **Webhooks** que vous voyez appartient au **module Intégrations** (un autre système, relié à la comptabilité). C'est celui-là qui est fonctionnel.

### 5.3 Sécurité et bonnes pratiques

- **N'accordez le statut Administrateur qu'aux personnes de confiance** : il ouvre tous les réglages sensibles (taxes, utilisateurs, abonnement).
- **Fixez pays, devise et taxes avant la première facture.** Le pays se verrouille dès qu'il existe des factures ou des écritures.
- **Gardez toujours au moins deux administrateurs** : le système vous empêche de retirer le dernier, mais un second administrateur évite un blocage si l'un part.
- **La recharge de crédits est un vrai paiement** : vérifiez le montant avant de confirmer.

### 5.4 Questions fréquentes

**Q : Je ne vois que 5 onglets, pas 11. Pourquoi ?**
R : Vous n'êtes pas administrateur. Les onglets Utilisateurs, Apparence, Juridiction & Devise, Taxes, Préférences et Fuseau horaire sont réservés aux administrateurs. Demandez à un administrateur de cocher « Administrateur » sur votre compte.

**Q : Je n'arrive pas à modifier les informations de l'entreprise.**
R : L'onglet Entreprise est en **lecture seule** pour les non-administrateurs (une bande ambre vous le signale). Seul un administrateur peut y écrire.

**Q : Puis-je changer le nom d'utilisateur d'un employé ?**
R : Non. Le nom d'utilisateur est fixé à la création et n'est pas modifiable. Créez un nouveau compte si nécessaire (et désactivez l'ancien).

**Q : Puis-je supprimer définitivement un compte ?**
R : Non. On **désactive** un compte (il devient Inactif) ; il n'est jamais effacé, pour préserver l'historique. La désactivation est réversible.

**Q : Le système refuse de désactiver un administrateur.**
R : C'est le dernier administrateur actif. Créez ou promouvez un autre administrateur, puis réessayez.

**Q : J'ai changé les couleurs de l'interface sur mon ordinateur, mais pas sur ma tablette.**
R : C'est normal. Les couleurs de l'**Interface** sont enregistrées **dans chaque navigateur** et ne se synchronisent pas. Refaites le réglage sur l'autre appareil. (Les couleurs des **Documents**, elles, sont partagées par toute l'entreprise.)

**Q : Le système m'empêche de changer de pays.**
R : Vous avez déjà des factures ou des écritures comptables. Le pays détermine les libellés fiscaux (TPS/TVQ ou Sales Tax) ; le changer fausserait l'historique. Ce réglage se fait avant de commencer à facturer.

**Q : J'ai réglé une taxe à 0 % et elle a disparu des documents.**
R : C'est voulu. Un **taux à 0** ou un **libellé vide** masque la taxe sur les nouveaux documents. Remettez un libellé et un taux pour la réafficher. Les documents déjà produits gardent leurs taxes d'origine.

**Q : Où sont les feuillets T4 et RL-1 ?**
R : Pas ici. La Configuration ne sert qu'à saisir les **renseignements employeur** (numéros, adresse). La production des feuillets se fait dans le module **Pointage et Paie**.

**Q : Ma recharge de crédits a été débitée mais le solde n'a pas bougé tout de suite.**
R : Le système facture d'abord, puis crédite. En cas d'aléa, un mécanisme de sécurité (le webhook Stripe) recrédite automatiquement. Rafraîchissez la carte Crédits IA. Si le doute persiste, contactez le soutien.

**Q : Mon entreprise est passée en « mode consultation ». Que faire ?**
R : Votre abonnement n'est plus actif (annulé, ou paiement non finalisé). La lecture reste possible, mais les écritures sont bloquées. Allez dans l'onglet **Abonnement** (qui reste accessible) et **souscrivez de nouveau** ou régularisez le paiement.

**Q : Puis-je créer de nouveaux rôles personnalisés ?**
R : Non. Les cinq rôles (Administrateur, Utilisateur, Employé, Comptable, Gestionnaire) sont fixes. Ce qui compte vraiment pour les droits, c'est la case **Administrateur**.

**Q : Y a-t-il un assistant IA, un export PDF ou une impression dans la Configuration ?**
R : Non. La Configuration ne fait que régler des paramètres. Les crédits IA y sont seulement **consultés et rechargés**, pas dépensés.

**Q : Puis-je gérer des webhooks depuis la Configuration ?**
R : Non, il n'y a pas d'onglet Webhooks actif dans la Configuration. Les webhooks disponibles se trouvent dans le sous-onglet **Webhooks** du module **Intégrations**.

---

## 6. Récapitulatif

- **Un centre de paramétrage à 11 onglets** : Profil, Utilisateurs, Entreprise, Apparence, Interface, Juridiction & Devise, Taxes, Préférences, Fuseau horaire, Abonnement, Intégrations.
- **Deux profils** : l'**administrateur** voit les 11 onglets ; l'**utilisateur ordinaire** en voit 5 (Profil, Entreprise en lecture seule, Interface, Abonnement, Intégrations).
- **Le vrai droit, c'est la case « Administrateur »** (`is_admin`), pas l'intitulé du rôle. Les cinq rôles sont surtout des étiquettes ; seul « super-administrateur » est interdit.
- **Deux systèmes de couleurs distincts** : **Apparence** (documents, en base, toute l'entreprise, 8 couleurs) et **Interface** (ERP, dans le navigateur, par utilisateur, 4 couleurs).
- **Identité de l'entreprise** : logo (≤ 1 Mo) + 12 champs (RBQ, NEQ, TPS, TVQ...) repris sur tous les documents.
- **Fiscalité** : pays et devise (le pays se verrouille dès la première facture), 2 taxes configurables (0 = masquée), taux de retenue, état/province, exercice fiscal, renseignements employeur (T4/RL-1), catégories de fournisseurs.
- **Utilisateurs** : création, rôles, réinitialisation de mot de passe (6 caractères minimum), **désactivation** (jamais de suppression) ; le **dernier administrateur** est protégé.
- **Abonnement (Stripe)** : souscrire, gérer via le portail, annuler en fin de période ; **crédits IA** rechargeables de 5 à 500 $ (paiement réel, idempotent).
- **Mode consultation** : sans abonnement actif, toutes les écritures de la Configuration sont bloquées, sauf le réabonnement et la déconnexion.
- **Webhooks** : aucun onglet actif dans la Configuration ; ils vivent dans le module **Intégrations**.

---

**Fichiers sources vérifiés** : `frontend/src/pages/ConfigurationPage.tsx` (4 108 lignes, 11 onglets), `frontend/src/api/config.ts` (661 lignes), `frontend/src/api/stripe.ts`, `frontend/src/store/useConfigStore.ts` (209 lignes), `frontend/src/store/useStripeStore.ts` (145 lignes), `frontend/src/store/useUiThemeStore.ts` (236 lignes), `frontend/src/i18n/locales/{fr,en}/config.json` (441 lignes), `backend/routers/config.py` (3 247 lignes, 38 points d'accès), `backend/routers/stripe_routes.py` (379 lignes, 6 points d'accès), `backend/routers/html_utils.py` (thème des documents), `backend/erp_auth.py` (gardes d'accès), `frontend/src/pages/IntegrationPage.tsx` (1 608 lignes, 7 sous-onglets — module séparé).

**Manuels liés** :
- Module 10 (Employés — fiches liées aux comptes utilisateurs) — `10-operations-employes.md`
- Module 12 (Pointage et Paie — feuillets T4/RL-1/PD7A alimentés par les renseignements employeur) — `12-operations-pointage.md`
- Module 14 (Comptabilité — taxes, retenue, pays/devise, exercice fiscal) — `14-operations-comptabilite.md`
- Module 24 (Assistant IA — crédits IA rechargés ici) — `24-communication-assistant-ia.md`
- Manuel des Intégrations (QuickBooks / Sage — onglet Intégrations) — voir la fiche du module Intégrations
- Vue d'ensemble des manuels — `README.md`
