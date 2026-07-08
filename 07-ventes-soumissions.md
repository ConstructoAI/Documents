# Module 07 — Soumissions et devis (manuel, IA, import)

> **Version** : 3.0 (refonte complète vérifiée contre le code source, 2026-07)
> **Route frontend** : `/devis` (menu latéral « Soumissions », groupe « Ventes »). Le titre affiché à l'écran est **« Soumissions »** (`devis.title`), même si l'adresse et le code interne parlent de « devis ». Page publique de validation par le client : `/devis/public/:token` (sans authentification).
> **Préfixe API** : `/api/erp/v1`. Trois routeurs sont montés sous ce préfixe : `/devis/manuel-template` (catalogue du modèle Manuel), `/devis` (le cœur du module) et `/devis/ai` (assistant en lecture seule). L'ordre de montage est important : `manuel-template` est monté **avant** `devis`, sinon la route dynamique `/{devis_id}` capterait le mot « manuel-template » (`erp_api.py:1000-1010`).
> **Code de référence (backend)** : `backend/routers/devis.py` (13 085 lignes — CRUD, lignes, IA, envoi, page publique, conversion projet) · `backend/routers/devis_ai.py` (344 lignes — assistant Soumissions **en lecture seule**) · `backend/routers/devis_manuel_template.py` (663 lignes — sections et lignes personnalisées du modèle Manuel). Total : **66 endpoints** (57 + 1 + 8).
> **Code de référence (frontend)** : `frontend/src/pages/DevisPage.tsx` (3 052 lignes — liste, panneau détail, modales) · `components/devis/EstimationIA.tsx` (1 728 lignes) · `components/devis/ConstructionTemplate.tsx` (1 130 lignes) · `pages/DevisPublicPage.tsx` (538 lignes) · `components/devis/DevisRenderModal.tsx` (534 lignes) · `components/devis/AiProfileManager.tsx` (410 lignes) · `components/devis/DevisAssistantTab.tsx` (152 lignes) · `components/devis/ClientInfoCard.tsx` · `components/devis/DevisConditionsEditor.tsx` · `components/devis/DevisFinancialSummary.tsx`. Clients API : `api/devis.ts`, `api/devisAi.ts`.
> **Tables PostgreSQL (par locataire)** : `devis` (en-tête), `devis_lignes`, `devis_assignations`, `devis_ai_estimations`, `ai_profiles` + `ai_profile_documents`, `conversations` + `conversation_documents`, `manuel_custom_sections` + `manuel_custom_lignes`, et l'écriture vers `companies`, `contacts`, `projects`, `opportunities`, `produits`, `employees`, `factures`. Table **partagée** : `public.devis_public_tokens` (jetons publics, validité 90 jours). Chemin IA : `public.ai_prepaid_credits`, `public.ai_usage_tracking`.
> **Cadrage** : ce module est l'**éditeur central des soumissions** d'une entreprise. Il gère tout le cycle : bâtir une soumission de **trois façons** (à la main, avec l'**IA**, ou en **important** un plan ou un métré), l'éditer en ligne, appliquer une **majoration** (cost-plus) par soumission ou par ligne, régler les **conditions et exclusions**, produire un **document HTML professionnel**, l'**envoyer au client** par un lien signable, **exporter** (Excel, CSV QuickBooks), **convertir en projet** et **facturer**. Il ne remplace pas le module **Métré** (qui mesure des quantités sur plan et les renvoie ici), ni le module **Comptabilité** (qui émet les factures) : il les alimente et les relie.

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

### 1.1 Mission

Le module Soumissions sert à **préparer un prix pour un client, le lui présenter proprement, et le transformer en projet une fois accepté**. Vous partez d'un besoin (une rénovation de cuisine, une construction neuve, un agrandissement), vous bâtissez la liste des travaux avec leurs prix, vous appliquez votre marge, puis vous envoyez au client un document soigné qu'il peut **accepter en le signant en ligne**. Dès qu'il accepte, l'ERP crée automatiquement le **projet** correspondant et vous pouvez le **facturer**.

Une soumission est donc un document vivant : brouillon tant que vous la montez, envoyée quand elle part chez le client, acceptée ou refusée selon sa décision.

### 1.2 Les trois façons de bâtir une soumission : manuel, IA, import

C'est la clé du module. Vous n'êtes jamais obligé de tout taper à la main.

| Façon | Où | Ce que vous faites | Résultat |
|-------|-----|-------------------|----------|
| **Manuel** | Panneau détail (bouton « Ajouter ») **ou** onglet **« Manuel »** (modèle Construction Québec) | Vous saisissez les lignes une à une, ou vous cochez des postes dans un gabarit de 9 sections avec quantités et prix | Des lignes ajoutées directement à la soumission |
| **IA** | Onglet **« Estimation IA »** | Vous discutez avec un expert virtuel (Claude), vous décrivez le projet, puis vous cliquez « Générer la soumission » | L'IA propose une soumission structurée par corps de métier, que vous révisez puis ajoutez au devis ou dont vous créez un nouveau devis |
| **Import** | Onglet **« Estimation IA »** (téléverser un plan ou un document) **ou** onglet **« Métré »** (mesurer sur un PDF) | Vous fournissez un plan PDF, une image, une liste de prix, ou vous mesurez des quantités sur un plan | L'IA **lit** le document et produit une **analyse** (catégorie, superficies, corps de métier) ou un chiffrage ; le Métré renvoie des **quantités** chiffrées |

> **À comprendre — l'« import » ne remplit pas les lignes tout seul.** Quand vous téléversez un plan dans l'Estimation IA, le système en fait une **analyse** (texte, diagnostic, superficies). Il n'existe **aucun bouton qui écrit directement des lignes de devis à partir d'un fichier**. Vous relisez l'analyse, puis vous cliquez « Générer la soumission » (Estimation IA) ou « Appliquer au devis » (Métré) pour créer les lignes. C'est voulu : vous gardez toujours le dernier mot sur ce qui entre dans le prix.

### 1.3 Accès par le menu latéral

Cliquez sur **Soumissions** dans le menu latéral. La page s'ouvre sur la **liste** des soumissions (`/devis`). Un lien direct existe pour ouvrir une soumission précise : `app.constructoai.ca/devis?open=<id>` (utilisé par les boutons « Voir la soumission » ailleurs dans l'ERP).

### 1.4 Les six onglets

En haut de la page, six onglets (`DevisPage.tsx:1358`) :

| Onglet | Rôle | Accès |
|--------|------|-------|
| **Soumissions** | La liste + le panneau détail (le cœur du module) | Tous |
| **Estimation IA** | Converser avec un expert IA et **générer** une soumission | Tous |
| **Métré** | Prendre des quantités sur un plan PDF (**module séparé**, voir le chapitre 30) | Tous |
| **Manuel** | Le modèle « Construction Québec » (9 sections à cocher) | Tous |
| **Conditions** | Éditer les conditions et exclusions **par défaut** de l'entreprise | **Administrateurs seulement** |
| **Assistant IA** | Poser des questions **en lecture seule** sur vos soumissions (icône « étincelles ») | Tous |

> **L'onglet « Conditions » n'apparaît pas** pour un utilisateur qui n'est pas administrateur (`isAdmin`, `DevisPage.tsx:1363`). C'est normal : il touche des réglages qui s'appliquent à toute l'entreprise.
>
> **L'onglet « Métré » est un module à part entière** (chargé à la demande, `components/metre-pdf/MetrePdf`). Son fonctionnement détaillé est décrit au **chapitre 30 — Métré**. Ici, seul le **pont** compte : « Appliquer au devis » / « Créer un devis ».

### 1.5 Permissions et rôles

Le module est **volontairement ouvert à toute l'équipe** : créer, modifier, ajouter des lignes, envoyer, changer un statut, supprimer — tout cela est permis à **tout utilisateur authentifié** du locataire. La seule action réservée est l'édition des **conditions/exclusions par défaut de l'entreprise** (`PUT /devis/defaults` → `require_tenant_admin_or_role()`, `devis.py:2564`). C'est le seul point d'entrée réservé aux administrateurs de tout le module.

| Action | Qui a le droit |
|--------|----------------|
| Consulter, créer, modifier une soumission ; ajouter/éditer/supprimer des lignes ; envoyer ; changer le statut ; convertir en projet ; facturer ; exporter | Tout compte valide du locataire |
| Supprimer une soumission | Tout compte valide — **sauf** si le statut est `Accepté` ou `Terminé` (refus 400) |
| Éditer les **conditions/exclusions par défaut de l'entreprise** (onglet Conditions) | **Administrateurs** (`is_admin`, relu côté serveur) |

**Mode consultation (lecture seule).** Si l'abonnement du locataire est en souffrance, le compte peut passer en **lecture seule** : les lectures fonctionnent, mais toute **écriture** renvoie 403. Ce contrôle est appliqué en amont dans l'authentification et couvre tous les endpoints du module. Cas particulier : la génération HTML **saute l'écriture du cache** en mode lecture seule mais reste consultable (`devis.py:11202`). Les endpoints publics par jeton (consultation, acceptation, refus par le client) ne passent pas par ce contrôle.

**Crédits IA.** Les fonctions d'intelligence artificielle (Estimation IA, analyse de documents, assistant, estimation d'un devis, rendu 3D) **consomment des crédits IA facturés au locataire**. Chaque endpoint IA vérifie d'abord les crédits (402 si le solde est insuffisant) et journalise la dépense. Le solde s'affiche en tout temps dans la barre d'outils de l'Estimation IA (ou « Illimité » pour les comptes exemptés).

**Isolation.** Chaque requête est bornée au schéma PostgreSQL du locataire ; une entreprise ne voit jamais les soumissions d'une autre. Les documents joints à une conversation IA sont en plus isolés **par utilisateur** à l'intérieur du locataire.

### 1.6 Numérotation automatique

À la création, chaque soumission reçoit un numéro **`DEV-AAAA-NNN`** (ex. `DEV-2026-007`) : l'année en cours suivie de l'identifiant. Le numéro est produit de façon **infaillible même en cas de clics simultanés** : la soumission est d'abord insérée avec un numéro provisoire, puis mise à jour avec son numéro définitif dérivé de son identifiant réel (`RETURNING id`, `devis.py:9084`). On n'utilise jamais un `COUNT + 1` qui pourrait produire des doublons.

### 1.7 Statuts et types

**Statuts.** Une soumission naît en **Brouillon**. Elle passe à **Envoyé** quand vous l'expédiez, puis à **Accepté** ou **Refusé** selon la décision du client. D'autres valeurs existent (`Validé`, `En attente`, `Terminé`, `Annulé`, `Expiré`) et sont atteignables par la mise à jour en lot ou par d'autres flux. Le filtre de la liste n'expose que les six statuts courants (voir 2.3).

**Types de soumission.** **Détaillée** (par défaut) ou **Budgétaire**. Le type « Budgétaire » annonce une estimation approximative.

**Type de projet.** Cinq choix à la création (`TYPE_PROJET_OPTIONS`, `DevisPage.tsx:593`) : Résidentiel neuf, Rénovation résidentielle, Commercial neuf, Rénovation commerciale, Institutionnel/Public.

**Priorité.** Normal (défaut), Haute, Urgente.

### 1.8 Le modèle de prix : la majoration est **incluse dans les prix**

C'est le point le plus important à comprendre, et le plus contre-intuitif.

Depuis une décision d'affaires de 2026, la soumission fonctionne en **« majoration incluse dans les prix unitaires »**. Concrètement :

- Vous entrez vos **coûts de base** (quantité × prix unitaire) sur chaque ligne.
- Le système applique par-dessus une **majoration** composée de trois parts : **Administration** (3 % par défaut), **Contingences** (12 % par défaut) et **Profit** (15 % par défaut). Ensemble, cela fait un facteur d'environ **× 1,30**.
- **Le prix affiché au client sur chaque ligne contient déjà cette majoration.** Les trois lignes « Administration / Contingences / Profit » du résumé financier ne sont **pas** des montants ajoutés en plus : ce sont une **ventilation informative** de la majoration déjà comprise dans les prix.

> **Ne vous méprenez pas** : si vous voyez « Administration 3 %, Contingences 12 %, Profit 15 % » sous le sous-total, cela **n'augmente pas** le total. Le total « toutes taxes comprises » que voit le client est le **même** dans l'aperçu, dans l'Excel et dans le CSV. La ventilation existe uniquement pour que vous sachiez comment votre prix est composé. Le résumé le rappelle par la mention « Dont majoration incluse dans les prix unitaires : ».

Deux cas concrets :

- **Majoration à 3/12/15** : vos prix unitaires sont vos coûts, et la marge est ajoutée pour l'affichage. Un encart gris détaille la part de chaque poste.
- **Majoration à 0/0/0** : un encart bleu s'affiche — « Majoration à 0 % — frais généraux, contingences et profit déjà compris dans les prix unitaires (prix tout inclus) ». C'est le mode « prix tout inclus », typique d'une estimation d'entrepreneur général.

**Le profit de 15 % : un défaut, pas un verrou.** Le 15 % est la valeur **par défaut** (outil de calcul déterministe, génération IA, valeur par défaut de la colonne en base). Mais vous pouvez saisir **n'importe quel pourcentage** de profit sur la soumission (0 à 100 %) et même un pourcentage différent **par ligne**. Le document final utilise toujours le pourcentage **réellement enregistré**, pas 15 % en dur.

**La base de prix au pi² vient de l'expert, pas d'une constante.** Il n'existe **aucune valeur « X $/pi² » figée** dans le module. Quand l'IA chiffre une construction, c'est **elle** qui fournit le tarif au pied carré (par étage), à partir de la gamme et de la région. Le serveur, lui, n'impose que la cascade de majoration (× 1,30) et des **fourchettes de validation souples** (voir 4.6) qui signalent un prix aberrant sans jamais le bloquer. La « gamme Économique » utilisée par défaut correspond au **plancher de la fourchette de la catégorie** (par exemple le bas de 200–450 $/pi² pour une construction neuve résidentielle), pas à un chiffre unique.

### 1.9 Ce que le module ne fait pas (vérifié dans le code)

- **Pas d'export PDF direct.** L'export PDF a été retiré (`DevisPage.tsx:19`). Pour obtenir un PDF, utilisez **« Générer HTML »** puis **Imprimer** (le navigateur produit le PDF), ou la page publique du client (bouton « Imprimer » / « Télécharger »). L'export PDF vectoriel existe seulement dans le module **Métré/DAO**, pas ici.
- **L'onglet « Conditions » est masqué aux non-administrateurs.**
- **L'Assistant IA (dernier onglet) est en lecture seule** : il répond à des questions sur vos données, mais **ne crée et ne modifie aucune soumission**. Pour générer, utilisez l'onglet « Estimation IA ».
- **L'import ne remplit pas les lignes automatiquement** (voir 1.2) : il produit une analyse à réviser.
- **Un seul rendu 3D par soumission** (remplaçable ou retirable).
- **La signature du client est obligatoire pour accepter** : le nom seul ne suffit pas.
- **Les lignes exigent une quantité supérieure à 0.** Les postes « administration / contingences / profit » et les lignes à quantité 0 sont exclus des ajouts en lot.
- **Le rendu 3D est payant** : le générer consomme des crédits IA du locataire (× 1,30) ; l'attacher ou le retirer est gratuit.

---

## 2. Interface

### 2.1 Les quatre indicateurs (KPI)

Toujours visibles en haut (`DevisPage.tsx:1334`), alimentés par `getDevisStatistics()` :

| Indicateur | Signification |
|-----------|---------------|
| **Total soumissions** | Nombre total de soumissions du locataire |
| **Brouillons** | Soumissions encore au statut Brouillon |
| **Envoyés** | Soumissions transmises au client |
| **Taux acceptation** | Acceptées ÷ (acceptées + refusées), en pourcentage |

### 2.2 La barre d'onglets

Les six onglets décrits en 1.4. L'onglet actif est surligné. L'onglet « Assistant IA » porte l'icône « étincelles » (Sparkles).

### 2.3 Onglet « Soumissions » — la liste

**La barre de commandes** (`DevisPage.tsx:1669`) contient :

- Le bouton principal **« Nouvelle soumission »**.
- Trois boutons de vue : **Liste**, **Tableau**, **Cartes**.
- Un champ **Rechercher…** (sur le numéro et le titre).
- Un filtre **Statut** : Tous, Brouillon, Envoyé, Accepté, Refusé, Expiré.
- Un filtre **Type** : Tous types, Détaillée, Budgétaire.

**La barre d'actions en lot** apparaît dès que vous cochez au moins une soumission (`DevisPage.tsx:1649`) : « {n} soumission(s) sélectionnée(s) », un menu déroulant « Changer le statut… » (Brouillon / Envoyé / Accepté / Refusé / Expiré) et un bouton « Désélectionner ».

**La vue Liste** (`DevisPage.tsx:1707`) affiche un tableau sur ordinateur et des cartes sur mobile. Les colonnes sont triables et redimensionnables :

| Colonne | Contenu |
|---------|---------|
| (case) | Sélection pour les actions en lot |
| **Numéro** | `DEV-AAAA-NNN` |
| **Titre** | Nom du projet |
| **Client** | Nom (entreprise, contact ou saisie manuelle) |
| **Montant** | Total toutes taxes comprises |
| **Statut** | Badge coloré + édition en ligne par menu déroulant |
| **Type** | Détaillée / Budgétaire, éditable en ligne |
| **Début Prévu** | Date, éditable en ligne |
| **Date Fin** | Date, éditable en ligne |
| **Créé** | Date de création |
| (corbeille) | Supprimer |

> **Édition en ligne du statut vers « Accepté »** : le système **confirme d'abord** (« Marquer la soumission « … » comme Acceptée créera automatiquement un nouveau projet. Continuer ? »), car l'acceptation déclenche la création du projet côté serveur.

**La vue Tableau** (`DevisPage.tsx:1844`) ajoute les colonnes financières détaillées : **HT / TPS / TVQ / Total**.

**La vue Cartes** (`DevisPage.tsx:1936`) présente une grille de cartes (numéro, badge de statut, nom, client, montant en vert, corbeille).

**Couleurs des statuts** : Brouillon = gris, Envoyé = indigo, Accepté/Terminé = vert, Refusé/Annulé = rouge, Expiré = ambre.

### 2.4 Le panneau détail

Cliquez sur une soumission pour ouvrir son **panneau détail** (colonne de droite sur ordinateur, plein écran sur mobile). En haut : le numéro, le numéro d'opportunité s'il y a un lien (badge bleu), le nom du projet, un crayon **Modifier** et le bouton fermer. Puis le badge de statut, le client et la description.

#### A. Le résumé financier (éditable)

Composant `DevisFinancialSummary` (`DevisPage.tsx:383`). C'est ici que vous réglez la majoration et la présentation.

- **Sous-total HT**, suivi de la note « Dont majoration incluse dans les prix unitaires : ».
- Si Administration = Contingences = Profit = 0, un encart bleu explique le mode « prix tout inclus ».
- **Trois lignes Administration / Contingences / Profit.** Chacune propose :
  - un **œil** pour l'afficher ou la masquer dans le document ;
  - un **libellé modifiable** (ex. renommer « Administration » en « Frais généraux ») ;
  - un champ **pourcentage** (0 à 100, pas de 0,5) ;
  - un champ **montant en dollars** (le saisir recalcule le pourcentage correspondant).
  - Valeurs par défaut : Administration 3 %, Contingences 12 %, Profit 15 %.
- Les **lignes de taxes** (TPS 5 %, TVQ 9,975 % au Québec par défaut) puis le **Total TTC** en gras.
- Une section **« Colonnes soumission »** : cinq bascules qui décident des colonnes visibles dans le document exporté — **Unité**, **Quantité**, **Prix unitaire**, **Montant par ligne**, **MO et MAT**.

#### B. Conditions et exclusions

Composant `DevisConditionsEditor` (`DevisPage.tsx:223`), repliable. Deux zones de texte (Conditions / Exclusions), chacune avec un **œil** pour l'afficher ou la masquer, un bouton **« Réinitialiser »** (revient aux défauts de l'entreprise) et un texte d'exemple. Un badge indique « Défauts » ou « Personnalisé ». Une ligne par condition ou exclusion ; les puces et la numérotation sont ajoutées automatiquement dans le document final.

#### C. Les boutons d'action

Jusqu'à huit boutons (`DevisPage.tsx:2036`) selon l'état de la soumission :

| Bouton | Effet |
|--------|-------|
| **Générer HTML** | Produit le document HTML professionnel |
| **Aperçu** | Ouvre le document dans une fenêtre (iframe) |
| **Ajouter un rendu 3D** | Ouvre la fenêtre de rendu photoréaliste (voir 2.9) |
| **Envoyer au client** (principal) | Ouvre la fenêtre d'envoi par courriel + lien public |
| **Convertir en projet** (vert) | Visible si Accepté/Terminé et pas encore de projet lié |
| **Facturer** | Visible si Accepté/Terminé — crée une facture brouillon |
| **CSV QuickBooks** | Télécharge un CSV compatible QuickBooks |
| **Copier CSV** | Copie le CSV dans le presse-papiers |
| **Excel (.xlsx)** | Télécharge le fichier Excel |

> Le CSV QuickBooks contient les colonnes Item, Description, Category, Quantity, Unit, Unit Price, Amount, Tax Code, MO %, MO $, MAT %, MAT $. Les prix y sont **déjà majorés** (facteur de ligne), pour rester cohérents avec l'aperçu HTML et l'Excel.

#### D. Les lignes

Titre « Lignes ({n}) » et bouton **« Ajouter »**. Chaque ligne se lit et s'édite en place.

- **En lecture** : la description (avec un badge « % » si la ligne a sa propre majoration), quantité × prix (majoré), le montant (majoré), un **œil** de visibilité, un **crayon** pour éditer, une **corbeille**. Si l'affichage MO/MAT est activé, une sous-ligne montre la répartition « MO x % / MAT y % ».
- **En édition** (`startEditLine`, `DevisPage.tsx:1116`) : description, quantité, unité, prix unitaire, un **ratio MO/MAT personnalisé** (deux champs qui se complètent, bouton « Auto » pour revenir à la détection par mots-clés), et une **majoration personnalisée par ligne** (Admin / Conting. / Profit — laisser vide = hériter des pourcentages de la soumission ; un badge « Personnalisée » et un lien « Hériter du devis » apparaissent). Un aperçu en direct affiche « = {montant} (majoration x %) ».

En bas, les **totaux MO/MAT** si l'affichage est activé, et la mention **« Validité : {date} »**.

> **Détection automatique MO/MAT.** À partir de mots-clés dans la description, le système devine la répartition main-d'œuvre / matériaux (20 règles couvrant les corps de métier québécois, `MO_MAT_RULES`, `DevisPage.tsx:52`). Exemples : peinture 70/30, démolition 65/35, gypse 60/40, électricité 55/45, plomberie 50/50, toiture 45/55, béton/fondation 40/60, isolation 35/65, excavation 30/70, armoires 30/70, portes et fenêtres 30/70. Sans mot-clé reconnu : 50/50. Vous pouvez toujours forcer vos propres pourcentages.

### 2.5 Les modales du panneau

- **Nouvelle soumission** (`DevisPage.tsx:2491`) — deux colonnes. À gauche : Nom du projet (obligatoire), No. PO Client, Client (Entreprise), Client (Personne), Saisie manuelle, Statut, Priorité, Type de projet. À droite : Tâche actuelle (parmi 25 tâches de production), Date limite de soumission, Début prévu, Fin prévue, Prix ($). En bas : Description. Une garde empêche le double envoi.
- **Modifier la soumission** (`DevisPage.tsx:2555`) — les mêmes champs, plus le Type de soumission.
- **Ajouter une ligne** (`DevisPage.tsx:2628`) — Description (obligatoire), Quantité, Unité, Prix unitaire, avec un aperçu des taxes (Montant HT, taxes, Total TTC).
- **Aperçu HTML** (`DevisPage.tsx:2668`) — l'iframe du document, avec « Ouvrir dans un nouvel onglet » et « Fermer ».
- **Envoyer au client** (`DevisPage.tsx:2746`) — champ « Adresse courriel du client », message d'introduction expliquant que le statut passera à « Envoyé » et qu'un lien de validation publique sera généré. Après l'envoi : une alerte (succès ou avertissement selon que le courriel est parti), le **lien de validation publique** et un bouton « Copier le lien ».

### 2.6 Onglet « Estimation IA »

Composant `EstimationIA.tsx` (1 728 lignes). C'est la conversation avec un expert qui **génère** une soumission.

**La barre d'outils** (`EstimationIA.tsx:1064`) :

- Un **sélecteur de profil d'expert**, séparé en « Mes profils » et « Profils système ». Le nombre de profils système est **dynamique** (chargé au démarrage) : il couvre des dizaines de métiers. À côté, un **engrenage** ouvre le gestionnaire de profils personnalisés.
- Un bouton **Document** pour téléverser 1 à 5 fichiers (`.pdf, .png, .jpg, .jpeg, .txt, .csv, .xlsx, .docx`, max 32 Mo).
- Un bouton **Historique**, un bouton **Nouveau**, et un **indicateur de crédits IA** (solde en dollars US ou « Illimité »).

**Détection automatique du profil.** Selon la juridiction (mémorisée par locataire), un profil adapté est présélectionné. Le premier téléversement de document bascule vers le profil **Entrepreneur général** (diagnostic de la catégorie).

**La barre de progression** montre les phases de téléversement puis d'analyse. Avec plusieurs documents, le système emploie « un agent IA par document + un chef coordinateur ».

**Le panneau Historique** liste les conversations sauvegardées (sauvegarde automatique après chaque réponse), avec renommage en ligne et suppression. Restaurer une conversation rétablit la fiche client et la soumission générée.

**La bannière de devis connecté** indique en bleu « Devis connecté : {nom} — items ajoutés à ce devis » ou en ambre « Aucun devis sélectionné… ». Si aucun devis n'est connecté, une **fiche client** (voir 2.8) apparaît pour saisir les informations.

**La bannière de diagnostic** (mode Entrepreneur général) affiche la catégorie et la sous-catégorie détectées, et la ventilation en zones (À estimer / Rénovation / Agrandissement / Existant conservé, en pi²).

**Les vignettes de documents** (persistés) permettent, pour chaque fichier, de le télécharger, de l'activer ou le désactiver du contexte de l'IA, ou de le supprimer.

**La zone de conversation** affiche les messages (rendu Markdown), un indicateur « {profil} réfléchit… », et un état de départ avec une question d'exemple cliquable et un guide en trois étapes (« 1. Choisissez un expert · 2. Décrivez votre projet · 3. (optionnel) Joignez un plan »). Le champ de saisie permet de joindre des fichiers (trombone, max 5), d'écrire, puis d'envoyer.

**Le bouton « Générer la soumission »** (`EstimationIA.tsx:1491`) lance la structuration. Le résultat s'affiche dans une **table de soumission générée** (`EstimationIA.tsx:1502`) :

- En-tête « Soumission générée — {n} items » avec les boutons **HTML** (export local, document complet aux couleurs de l'entreprise, avec échéancier et zone de signature), **Ajouter au devis** (si un devis est connecté) ou **Créer un nouveau devis**.
- La table est **groupée par corps de métier** (sections colorées, total par section), avec des éléments **éditables en ligne** (crayon, corbeille, bornes sur quantité et prix).
- Les totaux : Sous-total, Administration (x %), Contingences (x %), Profit (x %), Sous-total HT, taxes, **Total TTC**.
- Un **diagramme de Gantt** « Échéancier estimatif » apparaît s'il y a plus d'une section.

**Le gestionnaire de profils** (`AiProfileManager.tsx`) est une fenêtre pour créer et éditer vos **profils IA personnalisés** : un **Nom**, des **Instructions** (personnalité et expertise), et une **base de connaissance** (téléverser des documents `.pdf, .txt, .csv, .xlsx, .docx, .md, .tsv`, max 20 Mo — le texte est extrait et injecté dans le contexte de l'IA).

### 2.7 Onglet « Manuel » — le modèle Construction Québec

Composant `ConstructionTemplate.tsx` (1 130 lignes). Une bannière indique si un devis est connecté. Si aucun, une **fiche client** s'affiche. Puis le gabarit lui-même, avec trois sous-onglets (`ConstructionTemplate.tsx:583`) :

- **Travaux** — **9 sections fixes** numérotées 0.0 à 8.0 : Travaux préparatoires et démolition, Fondation, Structure et charpente, Enveloppe extérieure, Systèmes mécaniques et électriques, Isolation et étanchéité, Finitions intérieures, Aménagement extérieur et garage, Machinerie. Chaque poste est une **case à cocher** qui ouvre les champs Quantité / Unité / Prix unitaire / Montant. Neuf unités sont proposées (forfait, pi², pi. lin., unité, heure, jour, m², m. lin., verge cube). Vous pouvez ajouter des **lignes personnalisées** par section et des **sections personnalisées** (numérotées 9.0 et plus, renommables et supprimables). Un avertissement s'affiche si vous employez un nom réservé (administration, contingences, profit).
- **Recap** — les éléments regroupés par section, avec le résumé financier (Total travaux, Administration x %, Contingences x %, Profit x %, Sous-total avant taxes, taxes, TOTAL TTC).
- **Config** — trois curseurs Administration / Contingences / Profit et l'affichage de la « majoration totale ».

En bas, le bouton **« Appliquer au devis « {nom} » ({n} items — {total}) »** (si un devis est connecté) ou **« Créer un nouveau devis »**. Le système exclut les catégories administration/contingences/profit et n'ajoute que les lignes à quantité supérieure à 0. Un bouton **« Aperçu Soumission HTML »** permet de voir le rendu **sans rien enregistrer**.

### 2.8 La fiche client (partagée)

Composant `ClientInfoCard.tsx`, réutilisé par l'Estimation IA, le Métré et le Manuel quand aucun devis n'est connecté. Carte repliable « Fiche client / Informations du devis » avec cinq sections : **Projet** (Nom), **Client** (Entreprise / Personne / Saisie manuelle), **Échéancier** (Date limite de soumission / Début prévu), **Références** (No. PO / Priorité : Normal, Haute, Urgente), **Notes** (Description).

### 2.9 Le rendu 3D (optionnel, payant)

Composant `DevisRenderModal.tsx` (534 lignes). Ajoute une **image photoréaliste** au bas de la soumission. Le parcours a cinq étapes :

1. **Téléverser** une image ou un PDF (pas de fichier 3D).
2. **Recadrer** la zone à rendre.
3. **Régler** les paramètres (Détails, Qualité : pro / standard / rapide, Résolution : 2K / 4K).
4. **Rendu** : aperçu + coût affiché en dollars US, avec « Attacher » ou « Refaire ».
5. **Attaché** : « Remplacer », « Retirer » ou « Terminé ».

> **La génération est facturée** (crédits du locataire × 1,30, via le module de rendu `/cao/render`). **Attacher ou retirer est gratuit.** Des verrous empêchent le double clic. Le rendu apparaît ensuite dans l'aperçu HTML **et** sur la page publique du client. **Un seul rendu par soumission.**

### 2.10 Onglet « Conditions » (administrateurs)

Composant `DevisDefaultsTab` (`DevisPage.tsx:2826`). Réservé aux administrateurs. Il édite les **Conditions et Exclusions par défaut de l'entreprise** (deux zones de texte, Sauvegarder, Réinitialiser aux valeurs système). Ces textes s'appliquent aux **nouvelles** soumissions ; les soumissions existantes ne sont pas touchées, et chaque soumission peut ensuite être personnalisée individuellement.

### 2.11 Onglet « Assistant IA » (lecture seule)

Composant `DevisAssistantTab.tsx` (152 lignes). Un assistant conversationnel qui interroge vos **données réelles** de soumissions (montants, statuts, taxes, clients) et répond en langage naturel. Titre « Assistant IA — Soumissions », sous-titre rappelant qu'il est en **lecture seule**. Il propose trois questions d'exemple. Chaque réponse affiche des métadonnées (jetons, coût, durée).

> **Cet assistant ne crée et ne modifie rien.** Il est distinct de l'Estimation IA (qui, elle, génère). Il consulte, résume, compare. Pour agir, retournez à l'Estimation IA ou à l'éditeur.

### 2.12 Onglet « Métré » (le pont)

L'onglet « Métré » charge le module de prise de quantités sur plan (chapitre 30). Depuis ce module, deux boutons ramènent le résultat ici : **« Appliquer au devis »** (ajoute les lignes chiffrées à la soumission connectée) et **« Créer un devis »** (crée une nouvelle soumission à partir du métré). C'est la seconde grande voie d'« import » : des quantités mesurées sur un vrai plan deviennent des lignes de prix.

### 2.13 La page publique (côté client)

Composant `DevisPublicPage.tsx` (538 lignes), à l'adresse `/devis/public/:token`, **sans authentification**. C'est ce que voit votre client quand il clique sur le lien reçu par courriel.

- **En-tête** : les coordonnées de l'entreprise, le numéro et le titre de la soumission, puis le document complet dans une iframe.
- **Barre d'outils** : un **zoom** (50 à 200 %, la version mobile démarre à 60 %), un bouton **Imprimer** et un bouton **Télécharger** (fichier HTML). Un pied de page « Propulsé par Constructo AI ».
- **Deux boutons de décision** : **Refuser** et **Accepter la soumission**.
  - **Accepter** ouvre un formulaire : « Votre nom complet » **et** une **signature dessinée** (dans un canevas). Le bouton de confirmation reste désactivé tant que le nom **et** la signature ne sont pas fournis. L'écran de confirmation affiche ensuite « Signé par : {nom} » avec l'image de la signature.
  - **Refuser** permet d'indiquer une raison (facultative).
- Les états possibles : chargement, prête, acceptée, refusée, erreur, ou « déjà décidée » (si le client revient après coup).

---

## 3. Workflows pas à pas

### 3.1 Créer une soumission manuellement, ligne à ligne

1. Onglet **Soumissions** → bouton **« Nouvelle soumission »**.
2. Saisissez au minimum le **Nom du projet** (obligatoire).
3. Choisissez le client : une **entreprise** du CRM, une **personne**, ou une **saisie manuelle** si le client n'est pas encore enregistré.
4. Réglez si besoin le statut, la priorité, le type de projet, les dates et le No. PO.
5. Cliquez **Créer**. Le numéro `DEV-AAAA-NNN` est attribué automatiquement.
6. Ouvrez la soumission, puis, dans la section **Lignes**, cliquez **« Ajouter »**.
7. Pour chaque ligne : description, quantité, unité, prix unitaire. Répétez.
8. Réglez la **majoration** et les **conditions** dans le résumé financier (voir 3.7 et 3.8).

### 3.2 Bâtir avec le modèle Construction Québec (onglet Manuel)

1. Connectez d'abord une soumission (sélectionnez-la dans l'onglet Soumissions) — ou laissez le système en créer une.
2. Onglet **Manuel** → sous-onglet **Travaux**.
3. Parcourez les **9 sections**, **cochez** les postes qui s'appliquent, saisissez Quantité / Unité / Prix unitaire.
4. Ajoutez au besoin des **lignes** ou des **sections personnalisées**.
5. Sous-onglet **Config** : réglez Administration / Contingences / Profit.
6. Sous-onglet **Recap** : vérifiez le total.
7. Cliquez **« Aperçu Soumission HTML »** pour voir le rendu sans rien enregistrer.
8. Cliquez **« Appliquer au devis »** (ou **« Créer un nouveau devis »**). Seules les lignes à quantité supérieure à 0 sont transférées.

### 3.3 Estimer avec l'IA (onglet Estimation IA)

1. Onglet **Estimation IA**.
2. Choisissez un **profil d'expert** adapté (ou laissez la présélection).
3. Renseignez la **fiche client** (si aucun devis n'est connecté).
4. Dans la conversation, **décrivez le projet** (type, superficie, gamme, contraintes). Vous pouvez joindre un plan (trombone).
5. Échangez avec l'expert jusqu'à obtenir une bonne compréhension du projet.
6. Cliquez **« Générer la soumission »**. L'IA structure les lignes **par corps de métier** et propose la majoration (le profit est ramené à 15 % par défaut).
7. **Révisez** la table générée (éditez, supprimez des éléments au besoin).
8. Cliquez **« Ajouter au devis »** (devis connecté) ou **« Créer un nouveau devis »**.

> Le prix d'une construction est calculé de façon **déterministe** : l'IA fournit les étages et leurs superficies, et le serveur applique la formule « base × 1,30 × taxes » (voir 4.5). Cela évite les oublis (garage, aire manquante) et rend le chiffrage reproductible.

### 3.4 Importer un plan ou un document pour l'analyse IA

1. Onglet **Estimation IA** → bouton **Document** (ou le trombone de la conversation).
2. Sélectionnez 1 à 5 fichiers (PDF, image, `.txt`, `.csv`, `.xlsx`, `.docx`), 32 Mo maximum, PDF jusqu'à 100 pages.
3. Le système téléverse puis **analyse** (un agent par document + un coordinateur pour plusieurs fichiers).
4. Lisez le **diagnostic** : catégorie, gamme, superficies par zone, corps de métier.
5. Poursuivez la conversation pour préciser, puis **générez la soumission** comme en 3.3.

> **Rappel** : l'analyse ne crée pas de lignes toute seule. C'est vous qui déclenchez la génération une fois l'analyse validée.

### 3.5 Estimer un devis existant (texte ou plan)

Deux endpoints spécialisés travaillent sur une soumission **déjà ouverte** :

- **À partir du texte** (`POST /devis/{id}/ai-estimate`) : l'IA analyse le contenu de la soumission. En **mode précision** (par défaut), elle « réfléchit » davantage ; elle peut consulter le **catalogue de produits** du locataire pour ancrer les prix.
- **À partir d'un plan** (`POST /devis/{id}/ai-estimate-with-plan`) : vous téléversez un plan, l'IA le lit (vision) et estime. Cap 32 Mo, PDF jusqu'à 100 pages, compression automatique des grandes images, contexte additionnel jusqu'à 20 000 caractères.

Chaque estimation est **archivée** (`devis_ai_estimations`) et consultable / supprimable.

### 3.6 Créer et gérer un profil d'expert IA

1. Onglet **Estimation IA** → **engrenage** à côté du sélecteur de profil.
2. Cliquez **« Créer un profil »**.
3. Donnez un **Nom** et des **Instructions** (personnalité, expertise, règles de prix, normes à citer).
4. Dans **Base de connaissance**, ajoutez vos documents (listes de prix, spécifications, catalogues), 20 Mo maximum chacun. Le texte est extrait et injecté dans le contexte.
5. Sauvegardez. Le profil apparaît désormais sous « Mes profils » dans le sélecteur.

### 3.7 Ajuster la majoration (par soumission et par ligne)

**Au niveau de la soumission** (résumé financier) :

1. Réglez le **pourcentage** d'Administration, de Contingences et de Profit, ou saisissez directement un **montant** (le pourcentage se recalcule).
2. Renommez les libellés si vous le souhaitez (ex. « Frais généraux »).
3. Masquez une part avec son **œil** si vous ne voulez pas la montrer au client.

**Au niveau d'une ligne** (édition de ligne) : renseignez Admin / Conting. / Profit **propres à cette ligne**. Laissez vide pour hériter des pourcentages de la soumission.

> **Attention** : modifier un pourcentage **au niveau de la soumission efface** l'éventuelle dérogation de la même part **sur les lignes** (`devis.py:9250`). Réglez donc d'abord le niveau soumission, puis les exceptions par ligne.

### 3.8 Personnaliser les conditions et les exclusions

1. Dans le panneau détail, dépliez **Conditions & Exclusions**.
2. Écrivez **une ligne par condition** et **une ligne par exclusion** (les puces et la numérotation sont automatiques).
3. Utilisez l'**œil** pour masquer une section, ou **« Réinitialiser »** pour revenir aux défauts de l'entreprise.
4. Ces textes remplacent les défauts pour **cette** soumission seulement.

### 3.9 Générer et vérifier le rendu HTML

1. Panneau détail → **« Générer HTML »**, puis **« Aperçu »**.
2. Vérifiez la mise en page, les prix, l'échéancier et les conditions dans l'iframe.
3. Au besoin, **« Ouvrir dans un nouvel onglet »** pour une vue pleine page.

> Le document est un rendu **haut de gamme sur trois pages** au format Lettre (8,5 × 11 po), aux couleurs de l'entreprise (logo compris), avec échéancier/Gantt et bandeaux de prix (superficie, $/pi², taxes, validité).

### 3.10 Ajouter un rendu 3D

1. Panneau détail → **« Ajouter un rendu 3D »**.
2. Téléversez une image ou un PDF, **recadrez**, réglez la qualité et la résolution.
3. Lancez le rendu (il **consomme des crédits**), vérifiez l'aperçu, puis **« Attacher »**.
4. Le rendu apparaît dans l'aperçu HTML et sur la page publique. Vous pouvez le **Remplacer** ou le **Retirer** (gratuit).

### 3.11 Envoyer la soumission au client

1. Panneau détail → **« Envoyer au client »**.
2. Saisissez l'**adresse courriel** du client.
3. Cliquez envoyer. Le système :
   - passe le statut à **Envoyé** ;
   - génère un **jeton public** (s'il n'existe pas) valable **90 jours** ;
   - envoie un **courriel** aux couleurs de l'entreprise avec le lien `/devis/public/{token}` ;
   - enregistre le destinataire et la date d'envoi.
4. Copiez le **lien de validation publique** affiché si vous voulez aussi le transmettre autrement.

### 3.12 Le client accepte (avec signature) ou refuse

Côté client, sur la page publique :

- **Accepter** : il saisit son **nom complet**, **dessine sa signature**, puis confirme. Le serveur, par une mise à jour **atomique**, passe le statut à **Accepté** (un seul « gagnant » en cas de double clic), enregistre le nom, la signature et la date. Ensuite, en tâche de fond et sans jamais annuler l'acceptation même en cas de pépin : **création du projet**, copie des pièces jointes, création des bons de travail, passage de l'opportunité liée à « Gagné ».
- **Refuser** : il peut indiquer une **raison** (facultative). Le statut passe à **Refusé** et la raison est consignée.

Dans les deux cas, l'entrepreneur est notifié.

### 3.13 Convertir en projet (manuellement)

Si une soumission est **Acceptée** ou **Terminée** mais n'a pas encore de projet (par exemple, acceptée hors ligne) :

1. Panneau détail → **« Convertir en projet »** (bouton vert).
2. L'opération est **idempotente** : si un projet existe déjà, elle renvoie l'identifiant existant sans en créer un second.

### 3.14 Facturer

1. Panneau détail → **« Facturer »** (visible si Accepté/Terminé).
2. Confirmez. Une **facture** est créée depuis la soumission (module Comptabilité).

### 3.15 Exporter (Excel, CSV QuickBooks)

- **Excel (.xlsx)** : bouton dédié, téléchargement direct (en-tête client, lignes, sous-totaux, taxes, TTC ; protection contre l'injection de formules).
- **CSV QuickBooks** : bouton « CSV QuickBooks » (télécharger) ou « Copier CSV » (presse-papiers).
- **PDF** : il n'y a **pas** d'export PDF direct. Faites **Générer HTML → Aperçu → Imprimer** (choisissez « Enregistrer en PDF » dans la fenêtre d'impression), ou laissez le client imprimer/télécharger depuis la page publique.

### 3.16 Changer le statut de plusieurs soumissions à la fois

1. Dans la liste, **cochez** plusieurs soumissions.
2. Dans la barre d'actions en lot, choisissez le nouveau statut dans « Changer le statut… ».

### 3.17 Assigner des employés à une soumission

Trois endpoints permettent de lier des employés à une soumission (`GET/POST/DELETE /devis/{id}/assignments`), avec un rôle. Un doublon est refusé (409).

### 3.18 Calculer une cotisation CCQ ou CNESST

Deux petits calculateurs sont exposés (endpoints de **pur calcul**, sans authentification) :

- **CCQ** (`POST /devis/calculate-ccq`) : à partir d'un montant de main-d'œuvre et d'une liste de métiers ; taux par métier codés (environ 11,8 à 12,5 %, défaut 12,5 %). Ce n'est pas un calcul « heures × taux horaire », mais « montant × taux du métier ».
- **CNESST** (`POST /devis/calculate-cnesst`) : cotisation = main-d'œuvre × taux (paramètre `taux_unite`, défaut 1,80 %).

### 3.19 Modifier les conditions par défaut de l'entreprise (administrateurs)

1. Onglet **Conditions** (visible seulement pour les administrateurs).
2. Éditez les Conditions et les Exclusions par défaut.
3. **Sauvegarder** (ou **Réinitialiser** aux valeurs système). Les nouvelles soumissions en héritent ; les anciennes ne changent pas.

### 3.20 Supprimer une soumission

1. Corbeille dans la liste ou le panneau détail, puis confirmez.
2. **Refus (400)** si le statut est **Accepté** ou **Terminé**. Pour supprimer, changez d'abord le statut.
3. La suppression retire les lignes, assignations et autres éléments liés, et **détache** (met à NULL) les factures, projets et opportunités rattachés.

---

## 4. Référence

### 4.1 Statuts

| Statut (affiché) | Couleur du badge | Sens |
|------------------|------------------|------|
| Brouillon | Gris | En préparation |
| Validé | (interne) | Vérifié à l'interne |
| Envoyé | Indigo | Transmis au client |
| En attente | (interne) | Reçu, décision en attente |
| Accepté | Vert | Signé par le client → projet créé |
| Refusé | Rouge | Refusé par le client |
| Terminé | Vert | Cycle terminé |
| Annulé | Rouge | Annulé à l'interne |
| Expiré | Ambre | Validité dépassée |

Le filtre de la liste expose six statuts (Tous, Brouillon, Envoyé, Accepté, Refusé, Expiré). Les transitions clés : envoi → **Envoyé** ; acceptation publique → **Accepté** ; refus public → **Refusé**. La mise à jour en lot autorise n'importe quel statut. Côté serveur, l'accès public est **fermé par défaut** : un statut n'est consultable ou « décidable » par le client que s'il figure sur une liste blanche (`devis.py:12421` / `12427`).

### 4.2 Types et priorités

- **Type de soumission** : Détaillée (défaut), Budgétaire.
- **Type de projet** : Résidentiel neuf, Rénovation résidentielle, Commercial neuf, Rénovation commerciale, Institutionnel/Public.
- **Priorité** : Normal (défaut), Haute, Urgente.

### 4.3 Le modèle de prix cost-plus (cascade)

Les valeurs par défaut sont Administration **3 %**, Contingences **12 %**, Profit **15 %** (`devis.py:1289-1291`), soit une majoration totale de **× 1,30**.

```
base                 = somme des (quantité × prix unitaire) des lignes
administration       = base × adm%
contingences         = base × con%
profit               = base × pro%
sous-total avant taxes = base + administration + contingences + profit
TPS (5 %)            = sous-total avant taxes × 0,05
TVQ (9,975 %)        = sous-total avant taxes × 0,09975
TOTAL TTC            = sous-total avant taxes + TPS + TVQ
```

**Deux représentations de la majoration** :

- **En base de données**, `devis_lignes.montant_ligne` = coût de base pur (`quantité × prix`, sans majoration). Les agrégats `administration / contingences / profit` de la soumission sont recalculés à chaque écriture.
- **Dans le document rendu** (modèle « majoration incluse »), chaque ligne affiche `montant_ligne × facteur_de_ligne`, où `facteur_de_ligne = 1 + adm + con + pro` (avec les éventuelles dérogations de ligne). Le sommaire **n'ajoute pas** la majoration : il montre le « Sous-total HT » (majoration déjà comprise) puis la **ventile** en gris.

### 4.4 Dérogations de majoration par ligne

Une ligne peut porter ses propres pourcentages : `admin_pct_ligne` et `contingence_pct_ligne` (0 à 100), `profit_pct_ligne` (−100 à 999, pour permettre des ristournes ou des cas particuliers). Une valeur vide (NULL) = la ligne **hérite** des pourcentages de la soumission. Ces dérogations sont un outil **interne** : elles ne sont **jamais** montrées au client (la page publique retire ces champs).

### 4.5 L'outil déterministe `calculer_prix_construction`

Pour chiffrer une construction, l'IA n'invente pas le total : elle appelle un **outil de calcul** côté serveur (`devis.py:736`). Elle fournit la liste des **étages** (superficie brute au pi², avec un poids par étage : rez-de-chaussée 1,0, dernier 0,85, intermédiaires 0,80) et les **zones réduites** (garage froid, sous-sol), ainsi qu'un **tarif au pi²** (coût de base). Le serveur applique alors la cascade « base × 1,30 × taxes » — **exactement** la même formule que le document HTML. Résultat : un chiffrage reproductible, sans oubli de surface.

> C'est ici que se situe le tarif au pi² : il est **fourni par l'IA/le profil expert**, pas fixé par le serveur. Aucune constante « X $/pi² » n'existe dans le module.

### 4.6 Fourchettes de validation APCHQ (souples)

Après une génération, le serveur vérifie que le prix au pi² reste dans une **fourchette raisonnable** selon la catégorie, et signale (sans bloquer) les valeurs aberrantes (`_validate_estimation_items`, `devis.py:6413`). Exemples :

| Catégorie | Fourchette $/pi² |
|-----------|------------------|
| Construction neuve résidentielle | 200 – 450 |
| Agrandissement résidentiel | 250 – 500 |
| Rénovation majeure | 150 – 400 |
| Sous-sol | 80 – 200 |
| Rénovation de cuisine | 200 – 1000 |
| (défaut) | 50 – 2000 |

Ces contrôles sont **non bloquants** : ils produisent des avertissements ([SOFT] ou [CRITIQUE]) mais laissent passer la soumission. D'autres signaux : prix hors 0,01 $–1 M$, quantité hors 0,001–100 000, total supérieur à 10 M$, concentration d'une catégorie au-delà de 40 %, doublons. La **gamme par défaut est « Économique »** = plancher de la fourchette de la catégorie.

### 4.7 Champs d'une ligne

| Champ | Obligatoire | Notes |
|-------|-------------|-------|
| `description` | Oui | — |
| `quantite` | Oui | Doit être supérieure à 0 pour les ajouts en lot |
| `unite` | Non | Texte libre (défaut « unité ») |
| `prix_unitaire` | Oui | ≥ 0 |
| `montant_ligne` | Auto | `round(quantité × prix, 2)` = **coût de base, sans majoration** |
| `categorie` | Non | Corps de métier (regroupement) |
| `notes_ligne` | Non | Détail du poste (puces) |
| `visible` | Non | Défaut vrai ; faux = exclu du document |
| `mo_pct` / `mat_pct` | Non | Répartition main-d'œuvre / matériaux |
| `admin_pct_ligne` / `contingence_pct_ligne` / `profit_pct_ligne` | Non | Dérogations de majoration (voir 4.4) |
| `sequence_ligne` | Auto | Ordre |

### 4.8 Unités et catégories

- **Unités du modèle Manuel** (9) : forfait, pi², pi. lin., unité, heure, jour, m², m. lin., verge cube.
- **Catégories de soumission** : 21 corps de métier de référence (`_SOUMISSION_CATEGORIES`), avec un équivalent anglais. Le schéma d'outil de l'IA bascule automatiquement selon la langue du document du locataire.
- **Conditions et exclusions par défaut** : 5 conditions (validité 30 jours, modalités 30/40/30, garantie 1 an, mention RBQ) et 15 exclusions codées, surchargeables par soumission puis par entreprise.

### 4.9 Modèles IA et tarification

| Fonction | Modèle | Jetons max | Coût (par million, avant marge) |
|----------|--------|-----------|--------------------------------|
| Estimation IA (conversation, génération, analyse, plan) | `claude-opus-4-8` | 32 000 | entrée 5 $, sortie 25 $, écriture cache 10 $, lecture cache 0,50 $ |
| Assistant IA **lecture seule** (`/devis/ai/chat`) | Sonnet (`AI_MODEL`) | 8 000 | ≈ 0,003 $/1k entrée, 0,015 $/1k sortie |

Dans les deux cas, le coût réel est **majoré de 30 %** et déduit des crédits prépayés du locataire, puis journalisé (`track_ai_usage`). Les appels à Claude sont déportés hors de la boucle d'événements pour ne pas geler l'ERP partagé. Le cache de prompt (1 h) et l'API de fichiers réduisent les coûts sur les longues conversations.

### 4.10 Limites et plafonds

| Élément | Limite |
|---------|--------|
| Validité du jeton public | 90 jours |
| Signature (image data URL) | ≤ 500 000 caractères |
| Analyse de document / plan | ≤ 32 Mo, PDF ≤ 100 pages |
| Conversation avec fichiers | ≤ 5 fichiers, ≤ 10 Mo chacun |
| Analyse multi-documents | ≤ 5 fichiers |
| Base de connaissance d'un profil | ≤ 20 Mo par document |
| Instructions d'un profil | ≤ 200 000 caractères |
| Conditions / exclusions | ≤ 10 000 caractères, ≤ 200 lignes |
| Messages IA (anti-abus) | ≤ 400 messages, ≤ 1 500 000 caractères |
| Débit `/devis/ai/chat` | 20 requêtes/min par IP |
| Débit `/devis/public/` | 60 requêtes/min par IP |
| Suppression | Interdite si Accepté/Terminé |

### 4.11 Table des endpoints

**Routeur principal `/api/erp/v1/devis`** (57 endpoints — extrait des plus utilisés) :

| Méthode | Chemin | Rôle |
|---------|--------|------|
| GET | `/devis` | Liste paginée (filtres statut, type, client) |
| GET | `/devis/statistics` | Indicateurs (KPI) |
| POST | `/devis` | Créer (statut Brouillon, numéro auto) |
| GET | `/devis/{id}` | Détail + lignes |
| PUT | `/devis/{id}` | Modifier (recalcul cascade, création projet auto si Accepté) |
| DELETE | `/devis/{id}` | Supprimer (refus si Accepté/Terminé) |
| POST | `/devis/batch-update` | Changer le statut en lot |
| POST | `/devis/{id}/lignes` · `/lignes/batch` | Ajouter une / plusieurs lignes |
| PUT · PATCH · DELETE | `/devis/{id}/lignes/{lid}` [`/visibility`] | Éditer / masquer / supprimer une ligne |
| POST | `/devis/{id}/preview-html-with-items` | Aperçu avec des lignes en mémoire (0 écriture) |
| POST | `/devis/{id}/generate-html` | Générer et mettre en cache le document |
| POST · DELETE | `/devis/{id}/render` | Attacher / retirer un rendu 3D |
| GET | `/devis/{id}/export-xlsx` | Export Excel |
| POST | `/devis/{id}/send` | Envoyer (statut → Envoyé, courriel + jeton) |
| POST | `/devis/{id}/convert-to-project` | Convertir en projet (idempotent) |
| POST | `/devis/{id}/ai-estimate` · `/ai-estimate-with-plan` | Estimer un devis existant (texte / plan) |
| GET · DELETE | `/devis/{id}/ai-estimations[/{eid}]` | Historique des estimations |
| GET · POST · DELETE | `/devis/{id}/assignments[/{aid}]` | Assignations d'employés |
| POST | `/devis/ai-chat` · `/ai-chat-with-files` | Conversation experte (avec fichiers) |
| POST | `/devis/ai-generate-soumission` | Structurer une soumission |
| POST | `/devis/ai-analyze-document` · `/ai-analyze-documents` | Analyser 1 / jusqu'à 5 documents |
| GET · POST · PUT · DELETE | `/devis/ai-profiles[/{id}][/documents]` | Profils IA personnalisés |
| GET | `/devis/expert-profiles` | Liste des profils système + personnalisés |
| GET · POST · PUT · PATCH · DELETE | `/devis/conversations[/{id}][/documents]` | Historique des conversations IA |
| GET · PUT | `/devis/defaults` | Conditions/exclusions par défaut (**admin** en écriture) |
| POST | `/devis/calculate-ccq` · `/calculate-cnesst` | Calculateurs (sans authentification) |
| GET | `/devis/public/{token}` | Vue publique (sans authentification) |
| POST | `/devis/public/{token}/accept` · `/refuse` | Acceptation (signature) / refus par le client |

**Routeur assistant `/api/erp/v1/devis/ai`** (1 endpoint) : `POST /chat` — assistant lecture seule.

**Routeur catalogue Manuel `/api/erp/v1/devis/manuel-template`** (8 endpoints) : `GET/POST /sections`, `PUT/DELETE /sections/{id}`, `GET/POST /lignes`, `PUT/DELETE /lignes/{id}`.

### 4.12 Tables PostgreSQL

`devis`, `devis_lignes`, `devis_assignations`, `devis_ai_estimations`, `ai_profiles`, `ai_profile_documents`, `conversations`, `conversation_documents`, `manuel_custom_sections`, `manuel_custom_lignes` (par locataire) ; `public.devis_public_tokens` (partagée). Le module écrit aussi vers `companies`, `contacts`, `projects`, `opportunities`, `produits`, `employees`, `factures`. Plusieurs colonnes sont créées « à la demande » (migrations paresseuses idempotentes), avec un DDL qualifié par schéma pour éviter d'écrire par erreur dans le schéma partagé.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|--------|------|
| **CRM / Opportunités** (ch. 06) | Une opportunité peut engendrer une soumission ; à l'acceptation, l'opportunité liée passe à « Gagné ». Le badge du numéro d'opportunité s'affiche en tête du panneau détail. |
| **Projets** (ch. 09) | À l'acceptation (ou par conversion manuelle), un **projet** est créé et lié (opération à l'épreuve des accès simultanés et idempotente). Les bons de travail de départ sont générés. |
| **Entreprises / Contacts** (ch. 04-05) | Le client d'une soumission référence une entreprise, un contact, ou un nom saisi manuellement ; le nom est mis en cache. |
| **Dossiers** (ch. 07) | À l'acceptation, les pièces jointes sont copiées vers le projet ; la soumission apparaît dans la Fiche 360. |
| **Comptabilité** (ch. 15) | Le bouton **Facturer** crée une facture (brouillon) depuis la soumission. |
| **Métré** (ch. 30) | Le métré mesure des quantités sur plan et les renvoie ici (« Appliquer au devis » / « Créer un devis »). |
| **DAO / Rendu 3D** (ch. 31) | Le rendu 3D photoréaliste attaché à la soumission provient du moteur de rendu (facturé aux crédits). |
| **Magasin / Produits** (ch. 10) | L'estimation d'un devis existant peut consulter le **catalogue de produits** pour ancrer les prix. |
| **Employés** (ch. 11) | Assignation d'employés à une soumission. |

### 5.2 FAQ

**Comment obtenir un PDF de ma soumission ?**
Il n'y a pas d'export PDF direct. Faites **Générer HTML → Aperçu → Imprimer** et choisissez « Enregistrer en PDF », ou laissez le client télécharger/imprimer depuis la page publique.

**La ligne « Profit 15 % » s'ajoute-t-elle à mon total ?**
Non. Avec le modèle « majoration incluse », les prix des lignes **contiennent déjà** l'administration, les contingences et le profit. Les trois lignes du résumé ne font que **ventiler** cette majoration. Le total ne change pas selon qu'elles sont affichées ou masquées.

**Puis-je changer le profit de 15 % ?**
Oui. Le 15 % est un **défaut**. Vous pouvez saisir n'importe quel pourcentage de profit (0 à 100 %) sur la soumission, et même un pourcentage différent par ligne. Le document utilise toujours la valeur enregistrée.

**Quel est le prix au pi² utilisé par l'IA ?**
Il n'y a pas de valeur fixe dans le système. C'est l'**expert IA** (et son profil) qui fournit le tarif au pi², par étage, selon la gamme et la région. Le serveur ne fait qu'appliquer la majoration × 1,30 et vérifier que le résultat reste dans une fourchette raisonnable (validation souple, non bloquante).

**L'« import » d'un plan crée-t-il les lignes automatiquement ?**
Non. L'import **analyse** le document (catégorie, superficies, corps de métier). Vous révisez, puis vous cliquez « Générer la soumission » pour créer les lignes. Le Métré, lui, produit des quantités que vous transférez avec « Appliquer au devis ».

**Pourquoi l'onglet « Conditions » n'apparaît-il pas chez moi ?**
Il est réservé aux **administrateurs** : il règle les conditions/exclusions par défaut de toute l'entreprise.

**Le rendu 3D est-il gratuit ?**
Non pour la **génération** (crédits IA du locataire × 1,30). Oui pour **attacher** ou **retirer** un rendu déjà généré.

**Quelle est la différence entre « Estimation IA » et « Assistant IA » ?**
L'**Estimation IA** (onglet 2) **génère** des soumissions. L'**Assistant IA** (dernier onglet) est en **lecture seule** : il répond à des questions sur vos données, mais ne crée ni ne modifie rien.

**Combien de temps le lien public reste-t-il valide ?**
90 jours. Passé ce délai, renvoyez la soumission pour régénérer un lien.

**Le client peut-il accepter sans signer ?**
Non. Le nom **et** la signature dessinée sont requis pour accepter.

**Puis-je supprimer une soumission acceptée ?**
Non (refus 400). Changez d'abord son statut, puis supprimez.

**Que voit exactement le client sur la page publique ?**
Le document complet, avec le zoom, l'impression et le téléchargement — mais **sans** les informations sensibles (majorations par ligne, répartition MO/MAT, jeton, notes internes, métadonnées) qui sont retirées avant l'envoi.

**Les calculateurs CCQ et CNESST tiennent-ils compte des heures ?**
Non. Ils travaillent sur un **montant** de main-d'œuvre × un **taux** (par métier pour la CCQ, un taux d'unité pour la CNESST).

**Y a-t-il une fonction « Dupliquer » ?**
Il n'y a pas de duplication d'une soumission complète. Pour repartir d'une base, utilisez l'onglet **Manuel** (modèle) ou l'**Estimation IA**, ou recréez la soumission et transférez des lignes.

**Le système gère-t-il plusieurs devises ?**
Les taxes et les libellés sont ceux du locataire (par défaut TPS 5 % / TVQ 9,975 % au Québec). Il n'y a pas de conversion multidevise dans ce module.

---

## 6. Récapitulatif

- **Titre à l'écran : « Soumissions »** (route `/devis`). Six onglets : Soumissions, Estimation IA, Métré, Manuel, Conditions (administrateurs), Assistant IA (lecture seule).
- **Trois façons de bâtir** : **manuel** (ligne à ligne ou modèle Construction Québec 9 sections), **IA** (conversation experte → « Générer la soumission »), **import** (plan/document analysé par l'IA, ou quantités du Métré). L'import **n'écrit jamais** les lignes tout seul : vous révisez, puis vous générez.
- **Modèle de prix « majoration incluse »** : les prix des lignes contiennent déjà Administration (3 %) + Contingences (12 %) + Profit (15 %) ≈ × 1,30. Les trois lignes du résumé sont une **ventilation informative**, pas des montants ajoutés. Le **profit 15 %** est un **défaut modifiable**, pas un verrou. Le **tarif au pi²** vient de l'IA, pas d'une constante.
- **Numérotation** `DEV-AAAA-NNN`, infaillible même en clics simultanés.
- **Envoi** : statut → Envoyé + lien public 90 jours + courriel aux couleurs de l'entreprise. **Acceptation** par le client avec **signature dessinée obligatoire** → création automatique du **projet** et passage de l'opportunité à « Gagné ».
- **Exports** : Excel (.xlsx), CSV QuickBooks. **Pas de PDF direct** — utilisez Générer HTML + Imprimer, ou la page publique.
- **Rendu 3D** optionnel et **payant** (génération facturée aux crédits ; attache gratuite ; un seul par soumission).
- **IA** : Opus 4.8 pour l'estimation (32 000 jetons), Sonnet pour l'assistant lecture seule ; coût réel × 1,30 déduit des crédits, débit possible avant l'action (402 si solde insuffisant).
- **Permissions** : tout est ouvert à l'équipe, sauf l'édition des **conditions par défaut de l'entreprise** (administrateurs). **Suppression interdite** si Accepté/Terminé. Mode **lecture seule** si l'abonnement est en souffrance.

---

*Fichiers sources vérifiés : `backend/routers/devis.py` (13 085 lignes, 57 endpoints) · `backend/routers/devis_ai.py` (344 lignes) · `backend/routers/devis_manuel_template.py` (663 lignes) · `frontend/src/pages/DevisPage.tsx` (3 052 lignes) · `pages/DevisPublicPage.tsx` (538 lignes) · `components/devis/EstimationIA.tsx` (1 728 lignes) · `ConstructionTemplate.tsx` (1 130 lignes) · `DevisRenderModal.tsx` (534 lignes) · `AiProfileManager.tsx` (410 lignes) · `DevisAssistantTab.tsx` (152 lignes) · `api/devis.ts` · `api/devisAi.ts` · i18n `fr/{devis,devisAssistant,devisRender}.json`.*
*Manuels liés : 06 — CRM/Opportunités · 07 — Dossiers · 09 — Projets · 15 — Comptabilité · 30 — Métré · 31 — DAO/Modélisation 3D.*
*Manuel ERP Constructo AI — Module 07 « Soumissions et devis (manuel, IA, import) » — v3.0 vérifié — 2026-07.*
