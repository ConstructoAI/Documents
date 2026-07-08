# Module 35 — Intégrations comptables (QuickBooks, Sage)

> **Version** : 1.0 (rédaction initiale vérifiée d'après le code source, juillet 2026)
> **Accès** : le module n'a **pas** d'entrée autonome dans le menu latéral. On l'ouvre soit par **Configuration → onglet « Intégrations »** (icône `Link2`, `ConfigurationPage.tsx:286-289`), soit par l'adresse directe **`/integration`** (`App.tsx:256`, page protégée).
> **Code de référence** :
> - Frontend : page unique `frontend/src/pages/IntegrationPage.tsx` (1 608 lignes, **7 sous-onglets**), client API `frontend/src/api/integration.ts` (269 lignes) et `frontend/src/api/sage.ts` (149 lignes), magasin d'état `frontend/src/store/useIntegrationStore.ts` (271 lignes), traductions `frontend/src/i18n/locales/{fr,en}/pages.json` (section `integration`).
> - Backend : `backend/routers/integration.py` (4 073 lignes — **10 points d'accès** ; QuickBooks Online + Sage 50 ODBC) et `backend/routers/sage.py` (2 818 lignes — **7 points d'accès** ; Sage Business Cloud Accounting). Préfixe réel commun : `/api/erp/v1`. Gardes d'accès dans `backend/erp_auth.py`.
> **Tables PostgreSQL (par entreprise, créées à la demande)** : `integrations` (connexions), `integration_sync_logs` (journal de synchronisation), `integration_entity_map` (correspondances réelles 1 local ↔ 1 externe, remplies uniquement par la synchronisation). Les webhooks de l'onglet du même nom vivent ailleurs, dans les tables `{tenant}.webhooks` / `{tenant}.webhook_deliveries` du module Configuration.
> **Cadrage** : ce module relie l'ERP à un **logiciel comptable externe** afin d'y **exporter** vos clients, fournisseurs, factures et paiements, ou d'en **importer** les mêmes données. Quatre connecteurs cohabitent dans l'interface, mais **un seul est réellement actif à ce jour** : **QuickBooks Online** (protocole OAuth 2.0). **Sage 50** (logiciel de bureau, connexion ODBC) permet seulement de tester la connexion ; **Sage Business Cloud** (connecteur OAuth complet) reste **inerte** tant que le serveur n'est pas configuré ; les onglets **Webhooks** et **Correspondance** sont respectivement une fonction de notification manuelle et un tableau de référence en lecture seule. Le module est réservé aux **administrateurs** de l'entreprise.

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « entreprise » ou « tenant » désigne votre compte (chaque entreprise a ses propres données isolées) ; « connexion » désigne un lien enregistré vers un logiciel comptable ; « synchronisation » (ou « sync ») désigne le transfert de données entre l'ERP et ce logiciel ; « export » = de l'ERP **vers** le logiciel comptable ; « import » = du logiciel comptable **vers** l'ERP ; « connecteur » désigne l'un des quatre systèmes (QuickBooks, Sage 50, Sage Business Cloud, Webhooks).

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

Le module **Intégrations comptables** évite la double saisie entre l'ERP Constructo AI et votre logiciel de comptabilité. Concrètement, il vous permet de :

- **Brancher** votre compte comptable externe (aujourd'hui **QuickBooks Online**) de façon sécurisée, sans jamais confier votre mot de passe à l'ERP (le branchement passe par une page d'autorisation officielle du fournisseur) ;
- **Exporter** vers ce logiciel vos **clients**, **fournisseurs**, **factures**, **factures d'achat** et **paiements** créés dans l'ERP ;
- **Importer** depuis ce logiciel votre **plan comptable**, vos **clients**, vos **fournisseurs** et vos **factures** ;
- **Suivre** chaque transfert dans un **historique** détaillé (date, sens, entité, statut, message) ;
- **Consulter** la **correspondance des champs** (quel champ de l'ERP alimente quel champ du logiciel comptable) ;
- **Créer et tester** des **webhooks** (notifications HTTP sortantes vers Zapier, n8n ou votre propre serveur).

La synchronisation est **entièrement manuelle** : elle ne se déclenche que lorsque vous cliquez sur un bouton « Sync export » ou « Sync import ». Il n'existe **aucune planification automatique** dans l'interface.

### 1.2 Comment y accéder

Le module **n'apparaît pas** comme une entrée distincte du menu latéral (la barre latérale ne contient que **« Configuration »**, `Sidebar.tsx:109`). Deux chemins mènent au module :

| Chemin | Détail |
|--------|--------|
| **Configuration → onglet « Intégrations »** | Menu latéral → **Configuration** → onglet **Intégrations** (icône `Link2`). L'onglet embarque la page complète du module (`ConfigurationPage.tsx:286-289`). |
| **Adresse directe** `/integration` | Ouvre la même page en plein écran (`App.tsx:256`). Page protégée : authentification requise. |

> **Important — accès réservé aux administrateurs.** La page elle-même s'ouvre pour tout utilisateur connecté (c'est une route protégée générique), **mais tous les points d'accès du serveur exigent le droit d'administrateur** (`require_tenant_admin_or_role`). Un utilisateur ordinaire qui ouvre la page voit l'interface, mais les appels renvoient une **erreur d'autorisation** (403) : les listes restent vides et les actions échouent. Voir §1.3.

### 1.3 Rôles et permissions

| Action | Qui peut la faire |
|--------|-------------------|
| **Ouvrir la page** | Tout utilisateur connecté (route protégée) |
| **Lister / connecter / synchroniser / tester / supprimer** une connexion | **Administrateur** de l'entreprise uniquement |

Le contrôle d'accès est identique sur **tous** les points d'accès des deux routers (`integration.py` et `sage.py`) : la garde `require_tenant_admin_or_role()` autorise si votre compte porte le drapeau **administrateur** (`is_admin`, relu sur le serveur, donc infalsifiable), **ou** si votre rôle est `admin`, **ou** si vous êtes super-administrateur de la plateforme. **Aucun rôle métier supplémentaire** n'ouvre le module (le rôle « comptable » ne suffit pas ici).

> **Mode consultation (lecture seule).** Si l'abonnement de votre entreprise est suspendu, l'ERP passe en **mode consultation** : toutes les **écritures** du module (créer, modifier, supprimer, tester, connecter, synchroniser, choisir une entreprise Sage) sont **bloquées** (403). Seules les **lectures** (liste des connexions, historique, statistiques) restent possibles.

### 1.4 Les quatre connecteurs et leur état réel

C'est le point le plus important à comprendre avant d'utiliser le module. Quatre connecteurs partagent l'interface, mais ils ne sont **pas** au même niveau de maturité.

| Connecteur | Onglet | Technologie | État réel |
|------------|--------|-------------|-----------|
| **QuickBooks Online** | QuickBooks | OAuth 2.0 (API Intuit v3) | **Pleinement actif** : connexion, test, export **et** import. Seul connecteur qui synchronise réellement. |
| **Sage 50 (Simply Accounting)** | Sage 50 | ODBC (pilote Pervasive/Actian, logiciel de bureau) | **Connexion + test seulement.** La synchronisation en ligne **n'existe pas** (le serveur la refuse). Un bandeau ambre le signale. |
| **Sage Business Cloud** | Sage Business Cloud | OAuth 2.0 (API Sage Accounting v3.1) | **Connecteur complet mais inerte** tant que le serveur n'a pas ses clés d'activation. État probable en production : **« non configuré »**. |
| **Webhooks** | Webhooks | HTTP POST sortant (signé HMAC-SHA256) | **Création et test manuel** seulement. Le déclenchement **automatique** sur événement métier **n'est pas encore actif**. |

> **En clair :** pour synchroniser vraiment votre comptabilité aujourd'hui, utilisez **QuickBooks Online**. Les trois autres connecteurs sont soit préparatoires, soit en attente d'une configuration serveur.

### 1.5 Ce que le module fait — et ne fait PAS

Le module **fait** : brancher QuickBooks (OAuth), exporter/importer les données comptables, tester une connexion, journaliser chaque transfert, afficher la correspondance des champs, créer et tester des webhooks, et suivre l'historique.

Le module **ne fait PAS** :

- **Aucune planification / fréquence de synchronisation.** La sync est **100 % manuelle** (boutons « Sync export » / « Sync import »). Un réglage de fréquence existe côté données, mais **aucun contrôle ne l'expose** dans l'interface.
- **Aucun export PDF ou CSV, aucune impression, aucun téléversement de fichier.** La seule fonction s'apparentant à un export est de **copier** le modèle de charge utile (payload) JSON d'exemple dans l'onglet Webhooks.
- **Aucun assistant IA.** Ce module ne fait appel à aucune intelligence artificielle et n'entame **aucun crédit IA**.
- **Aucune synchronisation Sage 50.** La connexion et le test ODBC existent, mais tout transfert est **refusé** par le serveur.
- **Aucune activité Sage Business Cloud** tant que le serveur n'est pas configuré (état « non configuré »).
- **Aucun déclenchement automatique des webhooks.** Ils se créent et se testent, mais les événements métier ne les appellent pas encore.
- **Aucune édition de la correspondance des champs.** C'est un tableau de **référence figé** (non modifiable).
- **Aucun import QuickBooks des factures d'achat ni des paiements.** À l'import, seuls le plan comptable, les clients, les fournisseurs et les factures de vente sont pris en charge (voir §4.4).

### 1.6 Les 7 sous-onglets

Source : `TAB_KEYS` (`IntegrationPage.tsx:44-52`). La barre d'onglets se pilote à la souris ou au clavier (flèches, Début, Fin).

| # | Onglet | Icône | Rôle |
|---|--------|-------|------|
| 1 | **Vue d'ensemble** | `Database` | Synthèse : indicateurs, cartes fournisseur, méthodes d'intégration, tableau des taxes du Québec. |
| 2 | **QuickBooks** | `BookOpen` | Connecteur **actif** : connexion OAuth, synchronisation export/import, test, suppression. |
| 3 | **Sage 50** | `BookOpen` | Connexion ODBC (DSN) + test **seulement**. |
| 4 | **Sage Business Cloud** | `Link2` | Connecteur OAuth séparé, inerte sans configuration serveur. |
| 5 | **Webhooks** | `Globe` | Création et test de notifications HTTP sortantes. |
| 6 | **Correspondance** | `ArrowRightLeft` | Tableau de référence des champs (lecture seule). |
| 7 | **Historique** | `Clock` | Journal paginé de toutes les synchronisations. |

---

## 2. Interface

### 2.1 En-tête, barre d'onglets et bandeaux globaux

En haut de la page : le titre **« Intégrations comptables »** et le sous-titre **« QuickBooks Online & Sage 50 — Synchronisation des données »** (`pages.json`, section `integration`).

Sous le titre, la **barre des 7 onglets** (voir §1.6). Deux **bandeaux transitoires** peuvent apparaître au-dessus du contenu :

- **Bandeau d'erreur** (rouge) : message d'erreur renvoyé par le serveur.
- **Bandeau OAuth** (vert en cas de succès, rouge en cas d'échec) : confirme ou signale l'échec d'un branchement (par exemple « QuickBooks connecté avec succès ! »). Un bouton **Fermer** l'efface.

Un **indicateur d'attente** global s'affiche pendant les chargements.

**Badges de statut.** Partout dans le module, l'état d'une connexion ou d'un transfert est rendu par un **badge coloré** à six valeurs (couleurs conformes aux normes d'accessibilité, en mode clair comme sombre) :

| Badge | Signification |
|-------|---------------|
| **connecté** | La connexion est établie et valide. |
| **succès** | Le transfert s'est bien déroulé. |
| **déconnecté** | La connexion n'est pas établie. |
| **erreur** | Une erreur est survenue. |
| **en attente** | Connexion amorcée mais non finalisée, ou transfert en file. |
| **ignoré** | Élément volontairement sauté (par exemple une facture taxée dont le code de taxe n'était pas encore disponible ; elle repart au prochain transfert). |

### 2.2 Onglet « Vue d'ensemble »

Écran de synthèse, en quatre blocs.

**a) Quatre indicateurs (cartes KPI) :**

| Indicateur | Contenu |
|------------|---------|
| **Connexions** | Nombre total de connexions ; sous-texte « {n} actives ». |
| **Syncs totales** | Nombre total de synchronisations ; sous-texte « {n} erreurs ». |
| **Webhooks actifs** | Nombre de webhooks actifs ; sous-texte « {n} configurés ». |
| **Dernière sync** | Date de la dernière synchronisation, ou **« Aucune »**. |

**b) Deux cartes fournisseur :**

- **QuickBooks Online** — « Intégration avec Intuit QuickBooks pour la comptabilité et la facturation. » + badge de statut + date de dernière sync.
- **Sage 50 (Simply Accounting)** — « Connexion ODBC via Pervasive/Actian pour Sage 50 Canada. » + badge de statut.

> **À noter :** la Vue d'ensemble ne présente **que** QuickBooks et Sage 50. **Sage Business Cloud n'y figure pas** ; il ne se pilote que depuis son propre onglet.

**c) Carte « Méthodes d'intégration supportées »** — trois tuiles informatives, sans action :

| Méthode | Description | Étiquette |
|---------|-------------|-----------|
| **Zapier** | Connecteur sans code, simple. Idéal pour démarrer. | **Recommandé** |
| **n8n** | Flux de travail libre, auto-hébergé, gratuit. | **Gratuit** |
| **API directe** | Intégration Python/REST personnalisée. | **Avancé** |

**d) Carte « Configuration taxes Québec (TPS/TVQ) »** — tableau de référence :

| Taxe | Taux | Organisme | Champ Constructo | QuickBooks |
|------|------|-----------|------------------|------------|
| **TPS** | 5 % | Agence du revenu du Canada | `tps` | `TxnTaxDetail.TaxLine[0]` |
| **TVQ** | 9,975 % | Revenu Québec | `tvq` | `TxnTaxDetail.TaxLine[1]` |
| **Combiné** | 14,975 % | — | `montant_ttc` | `TotalAmt` |

Note affichée : « La TVQ est calculée sur le montant HT uniquement, pas sur HT + TPS. »

### 2.3 Onglet « QuickBooks » — connecteur pleinement actif

Titre : **« QuickBooks Online »**.

**Carte de connexion.** Logo QB, titre **« Connectez votre compte QuickBooks »** et un rappel de sécurité (le branchement se fait en OAuth 2.0 ; vos identifiants QuickBooks ne sont jamais partagés avec l'ERP).

- Si aucune connexion n'existe : bouton **« Connecter QuickBooks »** (vert). Un clic crée la connexion, récupère l'adresse d'autorisation et **vous redirige vers le site d'Intuit** pour autoriser l'accès. Une garde interne empêche le double-clic.

**Liste des connexions.** Chaque ligne affiche : logo QB, nom de la connexion, badge de statut, date de dernière sync. Boutons **contextuels selon le statut** :

| Situation | Boutons disponibles |
|-----------|---------------------|
| Statut **≠ connecté** | **« Reconnecter »** (relance l'autorisation OAuth). |
| Statut **= connecté** | **« Sync export »** (transfère l'ERP → QuickBooks) + **« Sync import »** (transfère QuickBooks → ERP). |
| Toujours | Icône **Tester** (vérifie que la connexion répond) + icône **Supprimer** (avec confirmation « Êtes-vous sûr de vouloir supprimer cette connexion ? »). |

**Bandeau de résultat** (vert / rouge) après un test ou une synchronisation. Le succès est **strict** : il faut zéro erreur **et** un statut de succès. Sinon, un **avertissement** détaille le bilan : « Synchronisation terminée avec des erreurs : {n} réussie(s), {n} en erreur. », suivi de quelques détails.

**Carte « Données synchronisables »** (propre à QuickBooks) — quatre tuiles :

| Donnée | Sens |
|--------|------|
| **Clients / Fournisseurs** | Bidirectionnel |
| **Factures** | Export |
| **Paiements** | Export |
| **Projets** | Export (métadonnées) |

> **Environnement.** Par défaut, le serveur pointe vers l'environnement **« bac à sable » (sandbox)** de QuickBooks (`QB_ENVIRONMENT`, `integration.py:125`), c'est-à-dire un environnement de **test**. Le passage en **production** est un réglage serveur (variable `QUICKBOOKS_ENVIRONMENT`). Si vos synchronisations ne se retrouvent pas dans votre vrai compte QuickBooks, c'est la première chose à vérifier auprès de l'administrateur d'infrastructure.

### 2.4 Onglet « Sage 50 » — connexion et test seulement

Titre : **« Sage 50 (Simply Accounting) »**.

**Bandeau ambre** (permanent) : **« La synchronisation Sage 50 en ligne n'est pas encore disponible. Connexion et test seulement. »**

**Carte.** Logo S50, titre **« Connectez Sage 50 »** et l'explication : « Connectez Sage 50 (Simply Accounting) via ODBC. Votre technicien informatique doit d'abord configurer un DSN sur le poste où Sage 50 est installé. »

- Bouton **« Ajouter une connexion Sage 50 »** → affiche un petit formulaire :

| Champ | Détail |
|-------|--------|
| **Nom (optionnel)** | Libellé libre. Exemple : « Sage 50 Bureau ». |
| **DSN (Data Source Name)** | Nom exact de la source de données ODBC configurée dans Windows. Exemple : « Sage50_MonEntreprise ». Aide affichée : « Le nom exact du DSN configuré dans les sources de données ODBC Windows. » |

- Boutons **« Connecter »** (désactivé tant que le DSN est vide) et **« Annuler »**.

**Liste des connexions** : identique à celle de QuickBooks, **mais sans** bouton de synchronisation, **sans** « Reconnecter » et **sans** carte « Données synchronisables ». Seuls **Tester** et **Supprimer** sont présents.

> **Ce qu'est un DSN.** Un DSN (Data Source Name) est un nom de connexion ODBC déclaré dans Windows, sur le **poste où Sage 50 est installé**. C'est votre **technicien informatique** qui le crée. L'ERP se contente d'utiliser ce nom pour joindre la base Sage 50. Le champ DSN est validé pour bloquer les caractères dangereux (`; { } =`) et est limité à 64 caractères.
>
> **Rappel :** même une fois la connexion créée et testée, **aucune donnée ne peut être synchronisée** avec Sage 50 (voir §5.2). La direction retenue par l'éditeur est de faire évoluer les clients vers **Sage Business Cloud**.

### 2.5 Onglet « Sage Business Cloud » — connecteur OAuth séparé

Titre : **« Sage Business Cloud »**. Cet onglet fonctionne de manière **autonome** (il ne partage pas la mécanique des onglets QuickBooks / Sage 50) et affiche l'un de **trois états** :

**a) Chargement** — indicateur d'attente le temps d'interroger le serveur.

**b) Non configuré (état probable en production)** — bandeau ambre : **« Le connecteur Sage Business Cloud n'est pas configuré sur le serveur. Contactez votre administrateur pour activer l'intégration. »** C'est l'état par défaut tant que le serveur n'a pas ses clés d'activation (variables `SAGE_CLIENT_ID` / `SAGE_CLIENT_SECRET`). Aucune action n'est possible.

**c) Configuré** — carte avec logo « SBC », nom du compte, badge de statut et description : « Synchronisez vos données comptables avec Sage Business Cloud Accounting via OAuth 2.0. » Selon la situation :

| Sous-état | Ce que vous voyez |
|-----------|-------------------|
| **Connecté** | « Connecté à {nom} » + date de dernière sync + boutons **« Sync export »**, **« Sync import »**, **« Tester »** et **« Déconnecter »** (avec confirmation « Êtes-vous sûr de vouloir déconnecter Sage Business Cloud ? »). |
| **En attente, plusieurs entreprises** | « Votre compte Sage donne accès à plusieurs entreprises. Choisissez celle à connecter à Constructo AI. » + un **menu déroulant « Entreprise Sage »** + bouton **« Choisir »**. |
| **Non connecté** | Bouton **« Connecter Sage »** → redirige vers l'autorisation OAuth de Sage. |

**Bandeau de résultat** (vert / rouge) après une action. Le **retour d'autorisation** (après avoir autorisé l'accès sur le site de Sage) est traité automatiquement : vous revenez sur la page, l'ERP finalise la connexion et affiche « Sage Business Cloud connecté avec succès ! » ou vous invite à choisir l'entreprise si votre compte en gère plusieurs.

### 2.6 Onglet « Webhooks » — création et test manuel

En-tête : **« Webhooks »** + bouton **« Nouveau webhook »**.

**Encart bleu (information)** : « Les webhooks envoient une notification HTTP POST à votre URL chaque fois qu'un événement se produit. Utilisez-les avec Zapier, n8n ou votre propre serveur… Chaque charge utile (payload) est signée avec HMAC-SHA256 pour la vérification d'intégrité. »

**Bandeau ambre (important)** : **« Note : la livraison automatique des événements métier n'est pas encore active. Les webhooks peuvent être créés et testés (bouton "Tester"), mais les événements ne se déclenchent pas encore automatiquement. »**

**Formulaire de création** (bouton « Nouveau webhook ») :

| Champ | Détail |
|-------|--------|
| **URL de destination** | Adresse qui recevra la notification. Exemple : « https://hooks.zapier.com/hooks/catch/… ». |
| **Description** | Libellé libre. Exemple : « Sync factures vers QuickBooks ». |
| **Événements** | Cases à cocher regroupées par catégorie : **4 catégories, 13 événements** (voir §4.7). |
| Boutons | **« Créer »** (désactivé si l'URL est vide) et **« Annuler »**. |

**État vide** : « Aucun webhook configuré » + « Créez un webhook pour déclencher les synchronisations ».

**Liste des webhooks.** Chaque ligne : chevron d'expansion, description (ou URL), badge **« Actif » / « Inactif »**, URL, puces d'événements. Boutons **Tester** et **Supprimer** (confirmation « Êtes-vous sûr de vouloir supprimer ce webhook ? »).

- **Zone dépliée « Livraisons récentes »** : « Chargement… », « Aucune livraison », ou la liste des 10 dernières livraisons (icône succès/échec, type d'événement, code de réponse HTTP, date).

**Carte « Exemple de payload webhook »** : un bloc JSON formaté (exemple d'un événement `invoice.created` avec numéro, montant HT, TPS, TVQ, montant TTC, statut…) et un **bouton copier** (une coche verte confirme la copie).

> **Note technique.** L'onglet Webhooks ne passe **pas** par les routers de ce module. Il s'appuie sur les **webhooks sortants génériques de l'ERP** (points d'accès `/config/webhooks`, gérés par le module Configuration). Il ne s'agit donc **pas** de webhooks entrants « QuickBooks » ou « Sage » : ce sont des notifications que **votre** ERP envoie à une URL de **votre** choix.

### 2.7 Onglet « Correspondance » — tableau de référence en lecture seule

En-tête : **« Correspondance des champs »**.

**Deux filtres** : un menu déroulant **fournisseur** (**QuickBooks** / **Sage 50**) et un menu **type d'entité** (**Toutes les entités**, ou Entreprise, Facture, Ligne facture, Paiement).

Si **Sage 50** est sélectionné, un bandeau ambre rappelle : « Informatif / à venir : la correspondance Sage 50 est fournie à titre de référence. La synchronisation en ligne n'est pas encore disponible. »

**Tableau**, colonnes : **Entité / Champ Constructo AI / Champ {fournisseur} / Direction** (Bidirectionnel, Export ou Import, avec icône). Les données sont **figées dans le logiciel** (non modifiables) : **16 lignes** pour QuickBooks, **14 lignes** pour Sage 50. Exemple QuickBooks : le nom d'entreprise (`nom`) correspond à `DisplayName / CompanyName` (bidirectionnel), le montant TTC (`montant_ttc`) correspond à `TotalAmt` (export), la quantité de ligne (`quantite`) correspond à `SalesItemLineDetail.Qty`, etc.

> **À quoi sert ce tableau ?** Il documente **quel champ de l'ERP alimente quel champ du logiciel comptable**, et dans quel sens. C'est une aide à la compréhension : vous ne le modifiez pas. Les correspondances **réelles** entre un enregistrement de l'ERP et son jumeau dans QuickBooks/Sage sont créées automatiquement par la synchronisation (elles ne sont pas éditables non plus).

### 2.8 Onglet « Historique »

En-tête : **« Historique de synchronisation »** + compteur « {n} entrée(s) ».

**Trois filtres** :

| Filtre | Valeurs |
|--------|---------|
| **Fournisseur** | Tous les fournisseurs / QuickBooks / Sage 50 |
| **Statut** | Tous les statuts / Succès / Erreur / En attente / Ignoré |
| **Type d'entité** | Toutes les entités / Factures / Entreprises / Paiements / Projets |

**État vide** : « Aucun historique de synchronisation » + « Les synchronisations apparaîtront ici une fois configurées ».

**Tableau**, colonnes : **Date / Fournisseur / Direction / Entité / Statut / Détails** (la direction porte une icône export ou import ; le statut est un badge ; les détails montrent le message d'erreur ou une note, tronqué au besoin).

**Pagination** : « {de}–{à} sur {total} », boutons **« Précédent »** / **« Suivant »**, « Page {n} / {total} ». Par défaut, **25 entrées par page**.

> **À noter :** l'historique **regroupe les deux fournisseurs**. Les synchronisations Sage Business Cloud y apparaissent aussi (fournisseur « sage »), même si elles n'ont pas de bouton dédié dans l'onglet.

---

## 3. Procédures pas à pas

### 3.1 Connecter QuickBooks Online

**Préalable :** être **administrateur** de l'entreprise ; le serveur doit avoir été configuré pour QuickBooks (clés Intuit).

1. Ouvrez **Configuration → Intégrations** (ou l'adresse `/integration`).
2. Allez à l'onglet **QuickBooks**.
3. Cliquez **« Connecter QuickBooks »**.
4. Vous êtes redirigé vers le site d'**Intuit** : connectez-vous à votre compte QuickBooks et **autorisez** l'accès.
5. Vous revenez automatiquement sur la page ; le bandeau vert **« QuickBooks connecté avec succès ! »** confirme le branchement. La connexion apparaît dans la liste, au statut **connecté**.

> **En cas d'échec :** le message « Impossible de lancer la connexion. Vérifiez que QuickBooks est configuré sur le serveur. » signale que les clés d'activation manquent côté serveur (tâche d'administrateur d'infrastructure). Le message « Échec de la connexion QuickBooks. Réessayez. » invite simplement à recommencer l'autorisation.

### 3.2 Synchroniser avec QuickBooks (export ou import)

**Préalable :** une connexion QuickBooks au statut **connecté**.

1. Onglet **QuickBooks** → repérez la connexion connectée.
2. Pour **envoyer vos données de l'ERP vers QuickBooks**, cliquez **« Sync export »**. Pour **rapporter les données de QuickBooks dans l'ERP**, cliquez **« Sync import »**.
3. Patientez : le transfert traite les entités **dans l'ordre de leurs dépendances** (d'abord le plan comptable, puis les clients et fournisseurs, puis les factures, factures d'achat et paiements).
4. Lisez le **bandeau de résultat** : un succès strict (« … avec succès »), ou un bilan « {n} réussie(s), {n} en erreur ».
5. Ouvrez l'onglet **Historique** pour le détail ligne par ligne (chaque entité, son sens, son statut, son message).

> **Ce qui est transféré.** À l'**export** : clients, fournisseurs, factures de vente, factures d'achat, paiements. À l'**import** : plan comptable, clients, fournisseurs et factures de vente. **Les factures d'achat et les paiements ne s'importent pas** (ils sont comptés en erreur et l'ensemble se termine au statut « partiel » — voir §4.4).
>
> **Pas de doublon.** Un enregistrement déjà synchronisé n'est **jamais recréé** : l'ERP mémorise la correspondance entre chaque enregistrement local et son jumeau externe, et il met à jour l'existant plutôt que d'en créer un second.

### 3.3 Connecter Sage 50 (ODBC) et tester

**Préalable :** Sage 50 installé sur un poste, et un **DSN ODBC** déjà créé par votre technicien informatique sur ce poste.

1. Onglet **Sage 50** → **« Ajouter une connexion Sage 50 »**.
2. Saisissez un **Nom** (facultatif) et surtout le **DSN** exact.
3. Cliquez **« Connecter »**.
4. Cliquez l'icône **Tester** pour vérifier que l'ERP joint bien la base Sage 50.

> **Limite importante.** Vous ne pourrez **pas** synchroniser de données avec Sage 50 : il n'y a **ni** « Sync export » **ni** « Sync import » sur cet onglet, et le serveur refuse tout transfert. De plus, le pilote ODBC nécessaire n'est **vraisemblablement pas** présent sur l'hébergement en nuage (Linux) : le test peut renvoyer « Driver ODBC pyodbc non installé ». Sage 50 reste donc, en pratique, un connecteur **de démonstration**.

### 3.4 Connecter Sage Business Cloud

**Préalable :** le serveur doit être configuré pour Sage (clés d'activation). Sinon, l'onglet affiche « non configuré » et rien n'est possible.

1. Onglet **Sage Business Cloud**.
2. Si l'écran indique « non configuré », **arrêtez-vous** : demandez à votre administrateur d'activer l'intégration côté serveur.
3. Sinon, cliquez **« Connecter Sage »** et **autorisez** l'accès sur le site de Sage.
4. Au retour :
   - Si votre compte Sage ne gère **qu'une** entreprise, la connexion se finalise automatiquement (statut **connecté**).
   - Si votre compte gère **plusieurs** entreprises, choisissez la bonne dans le menu **« Entreprise Sage »**, puis cliquez **« Choisir »**.
5. Cliquez **« Tester »** pour valider, puis utilisez **« Sync export »** / **« Sync import »** au besoin.

> **Changer d'entreprise Sage** efface les correspondances précédentes (pour repartir proprement sur la nouvelle entreprise). C'est volontaire : un même ERP ne doit pas mélanger deux comptabilités Sage.

### 3.5 Créer et tester un webhook

1. Onglet **Webhooks** → **« Nouveau webhook »**.
2. Saisissez l'**URL de destination** (par exemple votre point d'entrée Zapier ou n8n) et une **Description**.
3. Cochez les **événements** qui vous intéressent (Factures, Paiements, Projets, Entreprises).
4. Cliquez **« Créer »**.
5. Sur la ligne créée, cliquez **Tester** : l'ERP envoie une notification d'essai et affiche le résultat dans **« Livraisons récentes »**.

> **Rappel :** seuls le **test manuel** fonctionne. Les événements métier (une facture réellement créée, un paiement réellement reçu, etc.) **ne déclenchent pas encore** ces webhooks automatiquement. Créez-les et testez-les dès maintenant, mais n'attendez pas d'envois automatiques.

### 3.6 Consulter l'historique d'un transfert

1. Onglet **Historique**.
2. Filtrez au besoin par **fournisseur**, **statut** ou **type d'entité**.
3. Parcourez le tableau (25 lignes par page ; boutons **Précédent** / **Suivant**).
4. Pour une ligne en **erreur**, lisez la colonne **Détails** : elle contient le message renvoyé (par exemple un écart de total détecté par la garde de rapprochement).

---

## 4. Référence

### 4.1 Points d'accès — QuickBooks & Sage 50 (`integration.py`)

Tous préfixés `/api/erp/v1`. Tous protégés par la garde **administrateur** (`require_tenant_admin_or_role`).

| Méthode | Chemin | Rôle | Ligne |
|---------|--------|------|-------|
| GET | `/integrations` | Liste des connexions (jetons masqués) | `integration.py:804` |
| POST | `/integrations` | Créer une connexion (**quickbooks** ou **sage50** seulement) | `:833` |
| PUT | `/integrations/{id}` | Modifier (nom, statut, fréquence, config) | `:871` |
| DELETE | `/integrations/{id}` | Supprimer (révoque d'abord le jeton chez Intuit) | `:928` |
| GET | `/integrations/quickbooks/auth-url` | Générer l'adresse d'autorisation OAuth | `:997` |
| POST | `/integrations/quickbooks/callback` | Finaliser l'autorisation (échange du code contre les jetons) | `:1045` |
| POST | `/integrations/{id}/test` | Tester la connexion (QuickBooks ou Sage 50) | `:1228` |
| POST | `/integrations/{id}/sync` | Déclencher la synchronisation (**QuickBooks uniquement**) | `:3685` |
| GET | `/integrations/sync-history` | Historique paginé | `:3943` |
| GET | `/integrations/sync-stats` | Compteurs (total / succès / erreurs, par fournisseur, par entité) | `:4013` |

> Une connexion **Sage Business Cloud** ne se crée **pas** ici : `POST /integrations` n'accepte que `quickbooks` et `sage50`. La connexion Sage Cloud naît de son propre parcours OAuth (§4.2).

### 4.2 Points d'accès — Sage Business Cloud (`sage.py`)

Tous préfixés `/api/erp/v1`, protégés par la garde **administrateur**, et **bloqués (503)** tant que le serveur n'est pas configuré.

| Méthode | Chemin | Rôle | Ligne |
|---------|--------|------|-------|
| GET | `/sage/connections` | Liste des connexions Sage (+ indicateur « configuré ») | `sage.py:292` |
| GET | `/sage/auth-url` | Préparer l'autorisation OAuth (une seule connexion Sage par entreprise) | `:336` |
| POST | `/sage/callback` | Finaliser l'autorisation, récupérer les entreprises Sage | `:408` |
| POST | `/sage/connections/{id}/business` | Choisir l'entreprise Sage | `:551` |
| POST | `/sage/connections/{id}/test` | Tester la connexion | `:649` |
| DELETE | `/sage/connections/{id}` | Supprimer la connexion | `:709` |
| POST | `/sage/connections/{id}/sync` | Déclencher la synchronisation | `:2537` |

### 4.3 Statuts et badges

| Badge | Valeur interne | Où |
|-------|----------------|----|
| **connecté** | `connected` | Connexion établie |
| **succès** | `success` | Transfert réussi (historique) |
| **déconnecté** | `disconnected` | Connexion non établie |
| **erreur** | `error` | Erreur de connexion ou de transfert |
| **en attente** | `pending` | Connexion amorcée / transfert en file |
| **ignoré** | `skipped` | Élément volontairement sauté (repris au prochain transfert) |

### 4.4 Entités synchronisées et directions

**QuickBooks** — ordre de traitement (dépendances respectées) : plan comptable → clients → fournisseurs → factures de vente → factures d'achat → paiements clients → paiements fournisseurs.

| Entité | Export (ERP → QB) | Import (QB → ERP) |
|--------|:---:|:---:|
| Plan comptable (comptes) | Import implicite (sens unique) | Oui |
| Clients | Oui | Oui |
| Fournisseurs | Oui | Oui |
| Factures de vente | Oui | Oui |
| Factures d'achat | Oui | **Non** |
| Paiements clients | Oui | **Non** |
| Paiements fournisseurs | Oui | **Non** |

> Les factures d'achat et les paiements **ne s'importent pas** : à l'import, ces entités sont comptées en erreur et l'opération se termine au statut **« partiel »**. Les factures de type **avoir** et les factures **fournisseur** sont **exclues** de l'export des factures de vente.

**Sage Business Cloud** — ordre : plan comptable → contacts (clients/fournisseurs) → factures de vente → factures d'achat. La synchronisation Sage Cloud est **bidirectionnelle** sur ces quatre entités (import : plan comptable, contacts, factures de vente et d'achat ; export : contacts, factures de vente et d'achat ; le plan comptable est en import implicite).

### 4.5 Le « chemin de l'argent » : taxes et rapprochement

Le module manipule des montants réels ; plusieurs garde-fous évitent de créer un total silencieusement faux dans votre comptabilité.

- **Taxes du Québec.** À l'export, l'ERP retrouve les **codes de taxe** correspondants dans le logiciel comptable (TPS ≈ 5 %, TVQ ≈ 9,975 %, combiné ≈ 14,975 %). Une **tolérance de 0,02** distingue le combiné québécois (14,975 %) de la TVH (HST 15 %). Si un code correspond, la facture part **hors taxes** et le logiciel recalcule la taxe ; sinon, l'ERP fournit le montant de taxe total.
- **Facture taxée sans code disponible.** Si les codes de taxe ne sont pas (encore) lisibles, la facture est **différée** (statut **ignoré**) et repart au prochain transfert, plutôt que d'être exportée avec une taxe erronée.
- **Garde de rapprochement (post-envoi).** Après avoir créé une facture ou une facture d'achat dans le logiciel comptable, l'ERP **compare le total renvoyé au total de l'ERP**, avec une tolérance de **max(0,05 $ ; 1 % du total)**. En cas d'écart sur une facture taxée, l'événement est journalisé **en erreur** — **sans** renvoyer la facture (la correspondance est conservée, donc **aucun doublon**).
- **À l'import**, l'ERP ne reconstitue la ventilation TPS/TVQ que si le taux combiné avoisine 14,975 % ; sinon la taxe reste « non ventilée ». **L'import n'écrase jamais** le nom local d'un client ou fournisseur (un renommage fait dans l'ERP est prioritaire).
- **Anti-doublon.** La correspondance (local ↔ externe) est écrite **avant** toute décision d'envoi ; combinée à un index d'unicité, elle garantit qu'un même enregistrement n'est jamais créé deux fois. Sage Cloud renforce cela avec une **clé d'idempotence déterministe** par enregistrement ; QuickBooks s'appuie sur la correspondance et l'unicité.

### 4.6 Fournisseurs américains (contexte US)

Pour un tenant configuré aux États-Unis, l'export d'un fournisseur peut joindre son **numéro fiscal W-9** (TIN). Ce numéro est **chiffré** et n'est déchiffré qu'après **journalisation d'audit obligatoire** (conformité) : si l'audit échoue, le numéro **n'est pas** transmis. Le journal masque toujours les chiffres du TIN.

### 4.7 Événements de webhook (13, en 4 catégories)

| Catégorie | Événements |
|-----------|------------|
| **Factures** | Facture créée, Facture modifiée, Facture envoyée, Facture payée, Facture en retard, Facture annulée |
| **Paiements** | Paiement reçu, Remboursement |
| **Projets** | Projet créé, Projet modifié, Statut projet changé |
| **Entreprises** | Entreprise créée, Entreprise modifiée |

> Ces événements sont **définis** et cochables, mais **ne se déclenchent pas encore automatiquement**. Seul le bouton **Tester** provoque une livraison.

### 4.8 Limites, quotas et sécurité

| Élément | Valeur |
|---------|--------|
| **Fréquence de synchronisation** | **5 requêtes / 60 secondes** par adresse IP (s'applique aux boutons **Sync** et **Tester**, QuickBooks comme Sage Cloud). Au-delà : 429. |
| **Traitements simultanés** | 2 synchronisations en parallèle maximum (QuickBooks) ; idem pour Sage Cloud. |
| **Verrou anti-double-synchronisation** | Une seule synchronisation à la fois **par connexion**. Une seconde tentative renvoie **409 « Une synchronisation est déjà en cours pour cette connexion. Réessayez dans quelques minutes. »**. Un verrou « resté coincé » plus de **30 minutes** est levé automatiquement. |
| **Historique** | 25 entrées par page (réglable de 1 à 100 côté serveur). |
| **Validité du jeton anti-CSRF (OAuth)** | 10 minutes. |
| **Longueur du DSN Sage 50** | 64 caractères ; caractères `; { } =` interdits. |

**Sécurité :**

- **Jetons chiffrés.** Les jetons d'accès OAuth (QuickBooks et Sage Cloud) sont chiffrés (Fernet) avant stockage ; le serveur **refuse** d'enregistrer un jeton s'il ne peut pas le chiffrer.
- **Protection anti-CSRF.** Le retour d'autorisation OAuth est validé par un jeton `state` à durée de vie limitée (10 min) ; un horodatage manquant ou une horloge incohérente fait échouer la finalisation (fermé par défaut).
- **Anti-substitution.** QuickBooks : le compte (realm) déjà lié ne peut pas être remplacé par un autre au retour. Sage : l'entreprise choisie doit figurer dans la liste autorisée renvoyée au branchement.
- **Anti-SSRF (Sage).** Les appels à l'API Sage n'utilisent que des chemins relatifs sous le domaine officiel ; le jeton n'est jamais envoyé à une adresse arbitraire.
- **Charges utiles signées.** Chaque webhook est signé en HMAC-SHA256 pour que le destinataire vérifie son intégrité.
- **Isolation par entreprise.** Chaque connexion, journal et correspondance est cloisonné à votre entreprise.
- **Messages génériques.** Le détail technique des erreurs n'est jamais exposé dans l'interface.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

- **Comptabilité (module 15).** L'import remplit votre **plan comptable** et crée des **factures** dans l'ERP ; l'export envoie vos factures et paiements vers le logiciel comptable. C'est le module dont les données circulent le plus par ce connecteur.
- **Ventes / Soumissions et Projets (modules 08 et 09).** Les **factures de vente** issues des devis et des projets sont ce que l'export pousse vers QuickBooks/Sage ; les **projets** sont exportés en métadonnées.
- **Gestion des entreprises (module 04).** Vos **clients** et **fournisseurs** sont les entités « bidirectionnelles » : ils partent vers le logiciel comptable et en reviennent, sans jamais écraser un nom modifié localement.
- **Configuration (module 28).** Ce module **est** un onglet de la Configuration. C'est aussi la Configuration qui héberge les **webhooks** utilisés par l'onglet du même nom, et qui gère l'**abonnement** : un abonnement suspendu fait passer le module en **mode consultation** (écritures bloquées). Les **taxes** (TPS/TVQ) réglées dans la Configuration déterminent la ventilation exportée.
- **Magasin / Bons de commande (modules 10 et 14).** Les **fournisseurs** exportés vers le logiciel comptable proviennent aussi de ces modules.

### 5.2 Foire aux questions

**Où se trouve le module dans le menu ?**
Il n'a pas d'entrée propre. Ouvrez **Configuration** (menu latéral), puis l'onglet **Intégrations**. L'adresse directe `/integration` mène au même endroit.

**Pourquoi la page est-elle vide ou affiche-t-elle des erreurs alors que je suis connecté ?**
Parce que le module est **réservé aux administrateurs**. Un utilisateur ordinaire voit la page, mais le serveur refuse les données (403). Demandez le droit d'administrateur, ou faites configurer l'intégration par un administrateur.

**Quel connecteur fonctionne vraiment ?**
**QuickBooks Online**, aujourd'hui, est le seul qui synchronise réellement (export **et** import). Sage 50 se limite à la connexion et au test ; Sage Business Cloud est prêt mais **inerte** tant que le serveur n'est pas configuré ; les webhooks ne se déclenchent pas encore automatiquement.

**Puis-je programmer une synchronisation automatique (toutes les nuits, par exemple) ?**
Non. La synchronisation est **100 % manuelle** : elle ne part qu'au clic sur « Sync export » ou « Sync import ». Aucun réglage de fréquence n'est offert dans l'interface.

**Mes données QuickBooks n'apparaissent pas dans mon vrai compte. Pourquoi ?**
Le serveur pointe par défaut vers l'environnement de **test (sandbox)** de QuickBooks. Le passage en **production** est un réglage d'administrateur d'infrastructure (variable `QUICKBOOKS_ENVIRONMENT`).

**Vais-je créer des doublons si je synchronise plusieurs fois ?**
Non. L'ERP mémorise la correspondance entre chaque enregistrement local et son jumeau externe : un élément déjà transféré est **mis à jour**, jamais recréé. Un même transfert ne peut pas non plus tourner deux fois en parallèle sur une connexion (verrou, message 409).

**Puis-je synchroniser Sage 50 ?**
Non. La connexion et le test ODBC existent, mais **tout transfert est refusé**. De plus, le pilote ODBC nécessaire n'est vraisemblablement pas installé sur l'hébergement en nuage. La voie recommandée est **Sage Business Cloud**.

**L'onglet Sage Business Cloud affiche « non configuré ». Que faire ?**
Le serveur n'a pas ses clés d'activation Sage. C'est une tâche d'**administrateur d'infrastructure** ; vous ne pouvez rien faire depuis l'interface tant que ce n'est pas activé.

**Les webhooks se déclenchent-ils quand je crée une facture ?**
Pas encore. Vous pouvez **créer** et **tester** un webhook, mais les événements métier ne l'appellent **pas** automatiquement pour l'instant.

**Puis-je modifier la correspondance des champs ?**
Non. C'est un **tableau de référence figé** (16 lignes QuickBooks, 14 lignes Sage 50). Les correspondances réelles sont créées par la synchronisation et ne sont pas éditables non plus.

**Puis-je exporter l'historique en PDF ou en CSV, ou l'imprimer ?**
Non. Le module ne propose ni export de fichier, ni impression, ni téléversement. La seule fonction s'apparentant à un export est de **copier** le modèle JSON d'exemple dans l'onglet Webhooks.

**Le module consomme-t-il des crédits IA ?**
Non. Ce module ne fait appel à aucune intelligence artificielle et n'entame aucun crédit.

**Pourquoi une facture apparaît-elle « ignorée » dans l'historique ?**
Le plus souvent, parce que le **code de taxe** correspondant n'était pas encore disponible dans le logiciel comptable au moment du transfert. La facture n'est pas envoyée avec une taxe erronée ; elle repart automatiquement au prochain transfert.

**Que signifie un statut « partiel » ?**
Qu'une partie des entités demandées n'a pas pu être traitée — typiquement un **import** de factures d'achat ou de paiements QuickBooks, qui **n'est pas pris en charge**. Consultez l'historique pour le détail.

### 5.3 Dépannage courant

| Symptôme | Piste |
|----------|-------|
| Page vide / erreurs 403 | Vous n'êtes pas **administrateur**. Le module est réservé aux administrateurs. |
| « Impossible de lancer la connexion. Vérifiez que QuickBooks est configuré sur le serveur. » | Les clés Intuit manquent côté serveur (administrateur d'infrastructure). |
| Les données ne vont pas dans le vrai compte QuickBooks | Environnement **sandbox** actif ; à basculer en production côté serveur. |
| « Une synchronisation est déjà en cours pour cette connexion. » | Une sync tourne déjà. Patientez quelques minutes (verrou levé au bout de 30 min si coincé). |
| Erreur 429 après plusieurs clics | Limite de **5 requêtes / 60 s** atteinte. Attendez une minute. |
| Test Sage 50 : « Driver ODBC pyodbc non installé » | Le pilote ODBC n'est pas présent sur l'hôte (attendu en nuage). Sage 50 reste un connecteur de démonstration. |
| Onglet Sage Business Cloud « non configuré » | Clés Sage absentes côté serveur (administrateur d'infrastructure). |
| Webhook créé mais jamais appelé | Normal : la livraison automatique n'est pas active. Utilisez **Tester**. |
| Écritures refusées (mode consultation) | Abonnement suspendu → lecture seule. Régularisez l'abonnement (module Configuration). |
| Historique en erreur avec « écart de total » | La garde de rapprochement a détecté un écart sur une facture taxée ; vérifiez les taxes de la facture. La facture n'est **pas** dupliquée. |

---

## 6. Récapitulatif

- **Le module relie l'ERP à un logiciel comptable externe** pour éviter la double saisie (export/import des clients, fournisseurs, factures et paiements). Accès : **Configuration → onglet « Intégrations »** ou adresse `/integration`. **Pas** d'entrée propre dans le menu latéral.
- **Réservé aux administrateurs.** Un utilisateur ordinaire voit la page mais reçoit des erreurs 403. En **mode consultation** (abonnement suspendu), toutes les écritures sont bloquées.
- **Sept onglets** : Vue d'ensemble, QuickBooks, Sage 50, Sage Business Cloud, Webhooks, **Correspondance** (au singulier), Historique.
- **Un seul connecteur pleinement actif : QuickBooks Online** (OAuth 2.0), avec export **et** import. Par défaut, il pointe vers l'environnement de **test (sandbox)** — à basculer en production côté serveur.
- **Sage 50** = connexion ODBC (DSN) + **test seulement** ; aucune synchronisation. **Sage Business Cloud** = connecteur OAuth complet mais **inerte** sans configuration serveur (état « non configuré »).
- **Synchronisation entièrement manuelle** (boutons « Sync export » / « Sync import ») ; **aucune planification**. Ordre respectant les dépendances : plan comptable → clients/fournisseurs → factures → factures d'achat → paiements.
- **À l'import QuickBooks**, les **factures d'achat et les paiements ne sont pas pris en charge** (statut « partiel »).
- **Garde-fous d'argent** : différer les factures taxées sans code de taxe (« ignoré »), rapprochement du total après envoi (écart → « erreur » sans renvoyer), correspondance écrite avant tout envoi (**aucun doublon**), verrou anti-double-synchronisation (409), limite de 5 requêtes / 60 s.
- **Webhooks** : création et **test manuel** seulement (livraison automatique non active) ; ils s'appuient sur les webhooks génériques du module Configuration (`/config/webhooks`), signés HMAC-SHA256.
- **Correspondance des champs** : tableau de **référence figé** (16 lignes QuickBooks, 14 lignes Sage 50), non modifiable.
- **Aucun IA, aucun export PDF/CSV, aucune impression, aucun téléversement.** Sécurité : jetons OAuth chiffrés (Fernet), anti-CSRF (`state`), anti-SSRF (Sage), isolation stricte par entreprise.

---

*Fichiers sources vérifiés :* `backend/routers/integration.py` (4 073 lignes, 10 points d'accès ; QuickBooks Online + Sage 50 ODBC), `backend/routers/sage.py` (2 818 lignes, 7 points d'accès ; Sage Business Cloud), montés dans `backend/erp_api.py` sous `/api/erp/v1` ; `frontend/src/pages/IntegrationPage.tsx` (1 608 lignes, 7 sous-onglets), `frontend/src/api/integration.ts` (269 lignes), `frontend/src/api/sage.ts` (149 lignes), `frontend/src/store/useIntegrationStore.ts` (271 lignes) ; `frontend/src/i18n/locales/fr/pages.json` (section `integration`) ; onglet Webhooks servi par `backend/routers/config.py` (`/config/webhooks`). Gardes d'accès : `backend/erp_auth.py` (`require_tenant_admin_or_role`).

*Manuels liés :* `28-configuration.md` (module parent : onglet Intégrations, webhooks, abonnement et mode consultation), `15-operations-comptabilite.md` (plan comptable et factures échangés), `08-ventes-soumissions.md` et `09-ventes-projets.md` (factures de vente exportées), `04-gestion-entreprises.md` (clients et fournisseurs bidirectionnels).
