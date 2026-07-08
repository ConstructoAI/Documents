# Module 09 — Magasin (produits, inventaire, achats)

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/MagasinPage.tsx` (≈ 3 425 lignes, page unique à 7 onglets), `frontend/src/components/rfq/RfqTab.tsx` (≈ 1 412 lignes), `frontend/src/components/magasin/MaterialPricingTab.tsx` (≈ 502 lignes), `frontend/src/components/magasin/MaterialWebSearch.tsx` (≈ 150 lignes), `frontend/src/components/magasin/MagasinAssistantTab.tsx` (≈ 232 lignes)
> - API client : `frontend/src/api/inventory.ts`, `suppliers.ts`, `rfq.ts`, `magasinAi.ts`, `materialsPricing.ts`
> - Backend : `backend/routers/inventory.py` (produits + mouvements + nomenclature + statistiques), `backend/routers/suppliers.py` (fournisseurs + bons de commande), `backend/routers/rfq.py` (demandes de prix, ≈ 2 435 lignes), `backend/routers/materials_pricing.py` (prix matériaux), `backend/routers/magasin_ai.py` (assistant IA)
> - Préfixe API : `/api/erp/v1`
> **Tables PostgreSQL (par tenant)** : `produits`, `mouvements_stock`, `produit_composants`, `fournisseurs`, `bons_commande`, `bon_commande_lignes`, `rfq_demandes`, `rfq_reponses`, `produit_fournisseurs`, `produit_historique_prix` ; tables partagées : `public.bc_public_tokens`
> **Cadrage** : ce module réunit sur **une seule page** tout le cycle d'approvisionnement et de gestion du stock d'une entreprise de construction : le **catalogue de produits** (matériaux, équipements, EPI), le **suivi des niveaux de stock** avec traçabilité complète des mouvements, les **bons de commande** aux fournisseurs (avec envoi par courriel et réception qui alimente le stock), les **demandes de prix** (appels d'offres à plusieurs fournisseurs, avec portail public), un **comparateur de prix de détaillants** (Canac, BMR, Home Depot) et un **assistant IA**. Il **ne gère pas** les entrepôts multiples ni les transferts inter-sites, il **ne calcule pas** de coût moyen pondéré glissant sur la valeur d'inventaire, et le stock **ne se modifie jamais à la main** (uniquement par mouvement).

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

Le Magasin est le poste de commande de l'approvisionnement. Il permet de :

- tenir un **catalogue** de produits (nom, code, prix de vente, coût de revient, unité, fournisseur, emplacement) ;
- suivre le **stock disponible** de chaque produit et être alerté quand un produit passe sous son seuil minimal ;
- tracer **tous les mouvements** de stock (entrée, sortie, ajustement) avec l'employé, le projet, le bon de commande et le coût associés ;
- émettre des **bons de commande** (BC) numérotés aux fournisseurs, les **envoyer par courriel** (avec un lien public), et **recevoir** la marchandise, ce qui **augmente automatiquement le stock** ;
- lancer des **demandes de prix** (RFQ) à plusieurs fournisseurs, **comparer** leurs offres et **octroyer** la commande (ce qui génère les BC) ;
- **comparer les prix** d'une liste de matériaux chez les grands détaillants ;
- déclarer la **nomenclature** (BOM) d'un produit assemblé ;
- s'appuyer sur un **assistant IA** pour interroger les données et créer des produits.

### 1.2 Accès

- **Menu latéral** → section **Opérations** → **Magasin** (icône panier).
- **Adresse** : `/magasin`.
- Page protégée : il faut être authentifié dans un tenant.
- **Onglet ouvert par défaut** : **Commandes** (bons de commande).
- **Ouverture directe d'un bon de commande** : un lien du type `/magasin?open=<id>` (par exemple depuis la fiche 360 d'un dossier) ouvre automatiquement le BC concerné.

### 1.3 Les 7 onglets (sous-modules)

La page se compose de sept onglets, dans cet ordre :

| # | Onglet | Rôle | Icône |
|---|--------|------|-------|
| 1 | **Commandes** | Bons de commande aux fournisseurs (création, envoi, réception, facturation) | document |
| 2 | **Demandes de prix** | Appels d'offres à plusieurs fournisseurs, comparatif, octroi | balance |
| 3 | **Prix Matériaux** | Comparateur de prix Canac / BMR / Home Depot + recherche web | dollar |
| 4 | **Mouvements** | Historique et création des mouvements de stock | flèches |
| 5 | **Produits** | Catalogue + nomenclature (BOM) + étiquettes code-barres | boîte |
| 6 | **Fournisseur** | Fiches fournisseurs | camion |
| 7 | **Assistant IA** | Clavardage qui lit les données et propose de créer des produits | étoiles |

> Le champ de **recherche** et la **pagination** (20 lignes par page) sont partagés en haut de la page.

### 1.4 Permissions et rôles

La **consultation** de tous les onglets est ouverte à tout utilisateur authentifié du tenant. L'**écriture** dépend du rôle. Trois familles de droits coexistent (toutes tiennent compte du statut administrateur du propriétaire, pour ne jamais exclure un patron dont le rôle serait « utilisateur ») :

| Droit | Ce qu'il autorise | Rôles autorisés |
|-------|-------------------|-----------------|
| **Écriture inventaire** (`require_inventory_write`) | Créer/modifier un produit, créer/annuler un mouvement, gérer la nomenclature (BOM), imprimer une étiquette | admin, super-admin, **gestionnaire**, **magasinier**, ou tout administrateur (`is_admin`) |
| **Écriture achats** (`require_purchase_write`) | Créer/modifier un fournisseur, créer/modifier/envoyer/recevoir/supprimer un bon de commande, scanner un BC | admin, super-admin, gestionnaire, magasinier, **utilisateur**, ou administrateur |
| **Écriture demandes de prix** (`require_rfq_write`) | Créer/modifier une RFQ, inviter des fournisseurs, saisir des prix, octroyer | admin, super-admin, gestionnaire, magasinier, ou administrateur |

> **Nuance à retenir** : un simple **utilisateur** (rôle « user » non administrateur) peut créer et gérer des **bons de commande**, mais **pas** modifier le catalogue de produits, ni créer de mouvement, ni gérer une demande de prix. Ces dernières actions exigent un rôle **gestionnaire** ou **magasinier** (ou administrateur).

**Masquage du coût de revient** : le coût d'achat (`cout_revient`, `cout_unitaire`) est **caché** aux rôles de terrain. Seuls les rôles de gestion le voient (admin, super-admin, gestionnaire, magasinier, utilisateur, contremaître, comptable). Un employé de rôle « employé » voit les produits et les stocks, **mais pas les coûts**.

### 1.5 Bandeau de statistiques (4 cartes)

En permanence, quatre cartes chiffrées coiffent la page (source : `GET /inventory/stats`) :

| Carte | Contenu | Couleur |
|-------|---------|---------|
| **Produits** | Nombre de produits actifs | bleu |
| **Alerte seuil minimal** | Nombre de produits sous leur seuil | rouge si > 0, sinon vert |
| **Valeur du stock** | Valeur totale de l'inventaire en dollars | vert |
| **Catégories** | Nombre de catégories distinctes | violet |

### 1.6 Concepts clés

- **Produit** : un article du catalogue. Deux prix y sont **saisis à la main** : le **prix de vente** et le **coût de revient**. Il n'y a **pas** de calcul automatique de coût moyen.
- **Stock disponible** : la quantité en main. Elle ne se corrige **jamais** directement sur la fiche : uniquement par un **mouvement**.
- **Mouvement de stock** : une opération datée et tracée qui fait varier le stock. Trois types seulement : **Entrée**, **Sortie**, **Ajustement**. Chaque mouvement conserve le stock avant/après, l'employé, le projet, le bon de commande, le coût, le motif.
- **Bon de commande (BC)** : un engagement d'achat auprès d'un fournisseur, numéroté `BC-00001`. Quand on le passe à **Reçu**, le stock des produits liés **augmente automatiquement** et un mouvement d'entrée est créé.
- **Demande de prix (RFQ)** : un appel d'offres. On décrit les articles **sans prix**, on invite des fournisseurs, chacun **chiffre** (par un lien public ou par saisie interne), on **compare**, puis on **octroie** — ce qui génère un bon de commande par fournisseur retenu.
- **Nomenclature (BOM)** : la composition d'un produit assemblé (ses composants enfants). Déclaratif : créer un assemblage ne consomme pas les composants automatiquement.

---

## 2. Interface

### 2.1 Disposition générale

```
+------------------------------------------------------------------+
|  Titre « Magasin »                                               |
+------------------------------------------------------------------+
|  [Produits]  [Alerte seuil]  [Valeur du stock]  [Catégories]     |  <- 4 cartes KPI
+------------------------------------------------------------------+
|  Commandes | Demandes de prix | Prix Matériaux | Mouvements |    |  <- barre d'onglets
|  Produits | Fournisseur | Assistant IA                          |
+------------------------------------------------------------------+
|  Barre d'actions (boutons + recherche)                          |
+----------------------------+-------------------------------------+
|  Liste (tableau paginé)    |  Panneau de détail (à droite)       |
+----------------------------+-------------------------------------+
```

Chaque onglet propose sa propre **barre d'actions** (boutons de création, filtres, recherche) et, pour les onglets Commandes / Demandes de prix / Produits / Fournisseur, une vue **maître-détail** : la liste à gauche, le panneau de détail à droite quand on sélectionne un élément. Sur téléphone, les tableaux se replient en cartes simplifiées.

### 2.2 Onglet « Commandes » (bons de commande)

#### 2.2.1 Barre d'actions

- **Commande fournisseur** (bouton principal) : ouvre la fenêtre de création d'un bon de commande.
- **Scanner un BC (IA)** : téléverser une image ou un PDF d'un bon de commande fournisseur ; l'IA en extrait les lignes. Pendant le traitement : « Analyse en cours... ».
- **Recherche** (numéro, fournisseur, projet).

#### 2.2.2 Tableau des bons de commande

Colonnes (triables et redimensionnables) :

| Colonne | Contenu |
|---------|---------|
| **Numéro** | `BC-00001` |
| **Fournisseur** | Nom du fournisseur |
| **Projet** | Projet associé (si renseigné) |
| **Montant** | Total du BC |
| **Date Commande** | Modifiable directement au clic (champ date en ligne) |
| **Livraison Prévue** | Modifiable directement au clic |
| **Statut** | Badge de couleur |

Une icône **poubelle** permet de supprimer un BC (avec confirmation). Si aucun bon n'existe : « Aucun bon de commande ».

#### 2.2.3 Les 7 statuts de bon de commande

| Statut | Couleur | Signification |
|--------|---------|---------------|
| **Brouillon** | gris | Créé, pas encore transmis |
| **Envoyé** | indigo | Transmis au fournisseur |
| **Confirmé** | bleu | Le fournisseur a confirmé |
| **En cours** | violet | Livraison en cours |
| **Reçu** | vert | Marchandise reçue → **le stock a été alimenté** |
| **Facturé** | sarcelle | Facture fournisseur enregistrée |
| **Annulé** | rouge | Commande annulée |

#### 2.2.4 Panneau de détail d'un bon de commande

En sélectionnant un BC, un panneau s'ouvre à droite. Ses sections :

- **En-tête** : numéro + nom du fournisseur + **menu déroulant de statut** coloré (si le BC porte un statut hors liste, il est quand même affiché).
- **Informations** : Date de commande, Date de livraison prévue, Employé responsable (parmi les employés actifs), Projet associé.
- **Livraison** : Adresse de livraison, Conditions de livraison.
- **Articles** : un menu **« Sélectionner article d'inventaire »** (qui remplit automatiquement la description, l'unité et le prix), puis Description, Quantité, Prix unitaire, et le bouton **« Ajouter ligne »**. Chaque ligne affiche son montant (quantité × prix) et peut être supprimée.
- **Totaux** : Sous-total (hors taxes), **TPS/TVQ** calculées selon la configuration de taxes propre au BC (Canada ou États-Unis ; par défaut au Québec, TPS 5 %, TVQ 9,975 %), puis Total (taxes incluses).
- **Notes internes**.
- **Actions (première rangée)** :
  - **Générer HTML** : télécharge le bon de commande en fichier `.html` imprimable.
  - **Aperçu** : ouvre le document dans une fenêtre intégrée.
  - **Envoyer au fournisseur** : ouvre la fenêtre d'envoi par courriel.
  - **CSV QuickBooks** : télécharge un fichier CSV compatible QuickBooks.
  - **Copier CSV** : copie le CSV dans le presse-papiers.
  - **Excel (.xlsx)** : télécharge le BC en tableur.
  - **Créer facture** : crée une facture fournisseur (brouillon) à partir du BC. Désactivé si le BC est déjà **Facturé**.
- **Actions (deuxième rangée)** : **Supprimer** et **Sauvegarder** (ce dernier reste désactivé tant qu'aucun champ n'a été modifié).

> **Sauvegarde ciblée** : le panneau n'envoie au serveur **que les champs réellement modifiés**. Cela évite d'écraser par mégarde le statut d'un BC passé à « Reçu » entre-temps par un collègue.

#### 2.2.5 Fenêtre « Créer un bon de commande »

Cinq sections : **Fournisseur** (obligatoire) · **Assignation** (Projet, Employé, Date de livraison) · **Articles** (une ligne par article : choisir un produit du catalogue **ou** saisir une description et une unité à la main ; récapitulatif des taxes en temps réel) · **Livraison** (adresse, conditions) · **Notes internes**. Boutons **Annuler** / **Créer le bon de commande**.

#### 2.2.6 Fenêtre « Scanner un BC (IA) »

Après l'analyse d'une image ou d'un PDF, un bandeau apparaît dans la fenêtre de création : « {n} ligne(s) extraite(s) par l'IA », un indice de **confiance** (haute/moyenne/basse), le **total** extrait, et une alerte si le fournisseur n'a pas été reconnu. L'utilisateur valide ou corrige, puis crée le BC. Le scan est **en lecture seule** (il n'écrit rien tant que l'on n'a pas cliqué sur « Créer »).

#### 2.2.7 Fenêtre « Envoyer le bon de commande »

Saisir l'adresse courriel du fournisseur, puis envoyer. Le système :

- génère un **lien public partageable** valable **90 jours** (copiable), qui affiche le BC sans authentification ;
- passe le BC au statut **Envoyé** ;
- envoie un courriel thématisé aux couleurs de l'entreprise.

### 2.3 Onglet « Demandes de prix » (RFQ)

Cet onglet gère le cycle complet des appels d'offres fournisseurs. Barre d'actions : **Nouvelle demande** et **Actualiser**. Vue maître-détail.

#### 2.3.1 Liste des demandes

Colonnes : **Numéro** (`RFQ-00001`) · **Titre** · **Projet** · **Statut** · **Date limite** · **Réponses** (nombre de fournisseurs ayant répondu sur nombre d'invités).

**Cinq statuts de demande** : Brouillon · Envoyée · Clôturée · Octroyée · Annulée.

#### 2.3.2 Panneau de détail d'une demande

- **En-tête** : numéro + titre + badge de statut + boutons **Modifier** (crayon), **Supprimer** (impossible si déjà **Octroyée**) et fermer.
- **Informations** : Projet, Employé responsable, Date limite.
- **Livraison** : adresse et conditions (si renseignées).
- **Articles à chiffrer** : la liste des articles **sans prix** (ce sont les fournisseurs qui chiffreront), avec un formulaire d'ajout (Description, Quantité, Unité).
- **Fournisseurs invités** : chaque fournisseur porte un statut (**Invité**, **Répondu**, **Retenu**, **Non retenu**). Par ligne : **Saisir prix** (entrer un devis reçu hors ligne), **Envoyer** (transmettre le lien public) et **Retirer**. Un menu « Choisir un fournisseur » + **Inviter** ajoute des fournisseurs (ceux déjà invités sont filtrés).
- **Comparatif** : le bouton **« Voir le comparatif »** (désactivé tant qu'aucune réponse) affiche un tableau des prix par fournisseur, le **meilleur prix par ligne en vert**, les sous-totaux, la **couverture** (nombre de lignes chiffrées sur le total), le délai et la qualité, des cases **« Retenir »**, et le bouton **« Octroyer → générer BC »** qui crée un bon de commande par fournisseur retenu.
- **Actions** : Envoyer (à plusieurs fournisseurs), Générer HTML, Aperçu, Excel, Copier CSV du comparatif.

#### 2.3.3 Fenêtres de la demande de prix

- **Créer** (cinq sections : Informations, Articles, Fournisseurs à inviter, Livraison, Notes).
- **Aperçu HTML** de la demande.
- **Envoi groupé** : sélection de plusieurs fournisseurs, avec un rapport d'envoi (envoyés / échecs courriel / échecs).
- **Modifier l'en-tête**.
- **Saisir les prix — {fournisseur}** : saisie interne du prix de chaque ligne, plus délai, conditions et note (pour un devis reçu par téléphone ou en personne).

#### 2.3.4 Portail public du fournisseur

Un fournisseur invité reçoit un **lien sécurisé** (jeton signé). En l'ouvrant, il voit la demande, saisit ses prix, son délai et ses conditions, puis soumet. Le portail est **public** (aucun compte requis) mais protégé : jeton signé à durée limitée, limite du nombre de requêtes par heure, refus si la demande est clôturée ou échue, un jeton = un fournisseur.

### 2.4 Onglet « Prix Matériaux »

Assistant qui compare le prix d'une liste de matériaux chez les grands détaillants québécois. Une bascule sépare deux modes : **Comparateur** et **Recherche web**.

#### 2.4.1 Mode Comparateur

On colle une liste de matériaux dans la fenêtre de clavardage. Le système interroge en parallèle **Canac**, **BMR** et **Home Depot** et renvoie :

- un **tableau comparatif** par article (meilleur prix coché en vert, lien vers la fiche du détaillant) ;
- le **total par fournisseur** ;
- le **« Plan le moins cher »** (tout acheter au moins cher, article par article) ;
- un **plan d'achat optimisé par magasin** (regrouper les achats par enseigne), avec l'économie réalisée en dollars et en pourcentage.

Un bouton **« Créer un bon de commande (brouillon) »** permet de transformer un plan en BC : on choisit le détaillant et le fournisseur correspondant, et un BC brouillon est généré.

> **Dépendance externe** : ce comparateur interroge des services web **non officiels** des détaillants. Home Depot est particulièrement fragile (souvent bloqué). Des résultats partiels sont donc normaux. Cet onglet peut même être **absent** si le service n'a pas pu démarrer sur le serveur.

#### 2.4.2 Mode Recherche web

Une requête libre lance une **recherche web** (Claude avec recherche web) et renvoie un texte de synthèse, les **sources citées**, ainsi que le nombre de recherches, la durée et le coût.

### 2.5 Onglet « Mouvements »

Traçabilité complète du stock.

#### 2.5.1 Barre d'actions et filtres

- **Nouveau mouvement**.
- **Filtres** : Type (Tous / Entrée / Sortie / Ajustement), Produit, Projet, Employé, Date depuis, Date jusqu'à, **Recherche** (produit, référence, motif) et **Réinitialiser** (visible quand des filtres sont actifs).

#### 2.5.2 Tableau des mouvements

Colonnes : **Produit** (avec le badge « Annulé » le cas échéant) · **Type** (badge vert pour Entrée, rouge pour Sortie, bleu pour Ajustement) · **Quantité** (avec l'unité) · **Stock avant → après** · **Employé** · **Projet / BC** · **Coût** · **Référence** · **Date**. Un clic sur une ligne ouvre le détail.

#### 2.5.3 Fenêtre « Nouveau mouvement »

Cinq sections :

1. **Type** :
   - **Entrée** (réception, achat, retour) → augmente le stock ;
   - **Sortie** (envoi au chantier, utilisation) → diminue le stock ;
   - **Ajustement** (correction d'inventaire) → fixe le stock à la valeur comptée.
2. **Article** : menu de sélection avec le stock affiché.
3. **Stock et quantité** : aperçu en direct du **stock actuel → après**, avec une alerte rouge **« Stock insuffisant pour cette sortie ! »** si la sortie dépasse le stock.
4. **Détails** : Référence, Date du mouvement (par défaut aujourd'hui, **saisie rétroactive possible**), Motif.
5. **Avancé** : Employé responsable, Projet associé, BC associé, Coût unitaire (avec calcul du total en direct).

Le bouton final change selon le type : « Enregistrer entrée », « Enregistrer sortie » ou « Ajuster le stock ».

#### 2.5.4 Fenêtre de détail d'un mouvement

Affiche les badges (type, annulé, contre-mouvement), l'article, les quantités avant/après, le coût, le contexte (référence, motif, employé, projet, BC), les dates, et le bouton **« Annuler ce mouvement »**. L'annulation crée un **contre-mouvement inverse** (une entrée annule une sortie et vice-versa). Le bouton est désactivé si le mouvement est déjà annulé ou s'il est lui-même un contre-mouvement.

### 2.6 Onglet « Produits »

Catalogue des articles et nomenclatures.

#### 2.6.1 Barre d'actions

- **Ajouter un Nouvel Article**.
- **Stock bas** (bascule) : n'affiche que les produits sous leur seuil.
- **Filtre par catégorie** (catégories réellement présentes en base).
- **Recherche** (nom, code).

#### 2.6.2 Tableau des produits

Colonnes (triables, redimensionnables) : **Produit** (nom + code) · **Catégorie** · **Stock** (avec l'unité) · **Seuil** · **Prix vente** · **Statut** (badge « Bas » rouge si le stock est au seuil ou en dessous et que le seuil est supérieur à 0, sinon « OK » vert) · **Actions** (**Étiquette** code-barres et **Modifier**). Un clic sur une ligne ouvre le panneau de nomenclature. Si aucun produit : « Aucun produit ».

#### 2.6.3 Panneau de nomenclature (BOM)

- **Composants** : la liste des produits enfants (produit, quantité, unité, prix unitaire, stock, notes) avec suppression, plus un formulaire d'ajout (choisir un produit composant, Quantité, Unité, Notes).
- **Utilisé dans** : la liste des produits parents qui utilisent ce produit comme composant (dépendances inverses).

Le serveur refuse une auto-référence, un doublon et **toute référence circulaire** (A qui contiendrait B qui contiendrait A).

#### 2.6.4 Étiquette code-barres

Le bouton **Étiquette** télécharge une étiquette **PDF Code128**. Si le produit n'a pas encore de code, le serveur en génère un. L'étiquette s'ouvre dans un nouvel onglet (avec repli en téléchargement).

#### 2.6.5 Fenêtre « Créer / Modifier un article »

| Champ | Notes |
|-------|-------|
| **Nom** | Obligatoire |
| **Quantité initiale** (création) / **Quantité actuelle** (édition) | En édition, ce champ est **en lecture seule** — un indice renvoie vers la création d'un mouvement d'**ajustement** |
| **Code interne** | Généré automatiquement (`ART-00001`) si laissé vide |
| **Limite minimale** | Seuil d'alerte |
| **Code-barres** | EAN / UPC |
| **Type de produit** | 13 choix (voir §4.7) |
| **Unité de vente** | `un`, `m²`, `pi²`, `kg`, etc. |
| **Fournisseur principal** | Texte libre |
| **Emplacement stock** | Texte libre |
| **Prix de vente ($)** | Saisi à la main |
| **Coût de revient ($)** | Saisi à la main |
| **Norme applicable** | 7 choix (voir §4.7) |
| **Description** | Texte libre |
| **Notes** | Texte libre |

> **Le stock n'est pas modifiable ici en édition.** Pour corriger une quantité, il faut passer par un **mouvement d'ajustement** (onglet Mouvements). C'est le seul chemin qui laisse une trace d'audit.

### 2.7 Onglet « Fournisseur »

#### 2.7.1 Barre d'actions

- **Nouveau Fournisseur**.
- **Commande fournisseur** (raccourci vers la création d'un BC).
- **Recherche**.

#### 2.7.2 Tableau des fournisseurs

Colonnes : **Fournisseur** (nom + courriel) · **Catégorie** · **Contact** · **Ville** · **Éval.** (note sur 5) · **Statut** (Actif / Inactif) · **Actions** (Modifier). Un double-clic ouvre l'édition. Si aucun fournisseur : « Aucun fournisseur ».

#### 2.7.3 Fenêtre « Créer un fournisseur »

Champs : **Entreprise** (obligatoire — choisie parmi les entreprises du carnet) · Code Fournisseur · Conditions de Paiement (par défaut « 30 jours net ») · Catégorie de Produits (11 choix) · Contact Commercial · Délai de Livraison (jours) · Contact Technique · **Évaluation Qualité /10** (curseur) · **Certifications Construction** (10 cases : RBQ, CCQ, CNESST, ISO 9001, BNQ, CSA, LEED, Garantie GCR, ACQ, APCHQ) · Notes d'Évaluation.

#### 2.7.4 Fenêtre « Modifier un fournisseur »

Nom, Catégorie, Conditions de paiement, Contacts, Délai, Évaluation /10, Notes, Notes d'évaluation, et la case **« Fournisseur actif »** (décocher pour retirer le fournisseur des menus tout en le conservant).

### 2.8 Onglet « Assistant IA »

Une fenêtre de clavardage « Assistant IA — Magasin » qui :

- **lit** les données réelles du Magasin (nombre de produits, ruptures, catégories, valeur du stock, etc.) ;
- peut **proposer de créer un produit** : l'IA affiche une **carte de proposition** que l'utilisateur doit **Confirmer** (ou Annuler) — l'écriture n'a lieu qu'à la confirmation.

Chaque message affiche le nombre de jetons, le coût et la durée. Exemples de questions suggérées : « quels produits sont sous leur seuil ? », « crée un produit… », « quelle est la valeur du stock par catégorie ? ».

> **Périmètre volontairement restreint** : l'assistant du Magasin ne crée **que des produits**. Il ne touche **ni au stock, ni aux bons de commande** (opérations plus sensibles, réservées aux écrans dédiés). La confirmation revérifie que l'utilisateur a bien le droit d'écriture inventaire.

---

## 3. Processus pas à pas

### 3.1 Créer un produit

1. Onglet **Produits** → **Ajouter un Nouvel Article**.
2. Remplir au minimum le **Nom**. Optionnel : quantité initiale, seuil, code (généré si vide), unité, prix de vente, coût de revient, catégorie, norme.
3. **Enregistrer**. Le produit apparaît dans le tableau. Son badge est « OK » (ou « Bas » si le stock initial est déjà au seuil).

### 3.2 Corriger le stock d'un produit (inventaire physique)

Le stock ne se modifie jamais sur la fiche produit. Pour l'ajuster :

1. Onglet **Mouvements** → **Nouveau mouvement**.
2. Type **Ajustement**, choisir l'article.
3. Saisir la **quantité réelle comptée** (valeur absolue, pas un écart).
4. Optionnel : motif (« Inventaire 2026-T1 »), employé, date.
5. **Ajuster le stock**. Le mouvement enregistre le stock avant, le stock après et l'écart, pour l'audit.

### 3.3 Créer un fournisseur

1. Onglet **Fournisseur** → **Nouveau Fournisseur**.
2. Choisir l'**Entreprise** (obligatoire). Renseigner les conditions de paiement, la catégorie, les contacts, le délai, l'évaluation, les certifications.
3. **Enregistrer**. Le fournisseur est **Actif** par défaut.

### 3.4 Créer un bon de commande

1. Onglet **Commandes** → **Commande fournisseur** (ou onglet **Fournisseur** → **Commande fournisseur**).
2. Choisir le **fournisseur** (obligatoire), le projet, l'employé, la date de livraison.
3. Ajouter les **articles** : choisir un produit du catalogue (description, unité et prix se remplissent) ou saisir une ligne à la main. Les taxes se calculent en direct.
4. Renseigner l'adresse et les conditions de livraison, et des notes internes.
5. **Créer le bon de commande**. Il apparaît en statut **Brouillon**, numéroté `BC-00001`.

### 3.5 Scanner un bon de commande fournisseur (IA)

1. Onglet **Commandes** → **Scanner un BC (IA)**.
2. Choisir une **image** ou un **PDF** du bon de commande (jusqu'à 20 Mo).
3. Attendre l'analyse. Un bandeau indique le nombre de lignes extraites, la confiance et le total.
4. Vérifier / corriger les lignes proposées, choisir le fournisseur si non reconnu.
5. **Créer le bon de commande**.

> Le scan consomme des **crédits IA** (voir §4.6). Il ne crée rien tant que vous n'avez pas confirmé.

### 3.6 Envoyer un bon de commande au fournisseur

1. Ouvrir le BC → **Envoyer au fournisseur**.
2. Saisir l'adresse courriel du fournisseur, puis envoyer.
3. Le système crée un **lien public** (valable 90 jours), passe le BC à **Envoyé** et expédie le courriel. Vous pouvez copier le lien pour le partager autrement.

### 3.7 Recevoir un bon de commande (alimenter le stock)

1. Ouvrir le BC → dans l'en-tête, changer le statut à **Reçu**.
2. Le système, **automatiquement et de façon atomique** :
   - pour chaque ligne liée à un produit du catalogue, **augmente le stock** de la quantité reçue ;
   - crée un **mouvement d'entrée** par produit (référence « Réception BC-xxxxx »), avec le coût unitaire.
3. Le BC affiche désormais **Reçu**, et les mouvements apparaissent dans l'onglet Mouvements.

> **Réception définitive.** Une fois **Reçu**, le BC ne peut plus être **dévalidé** vers un autre statut (le système renvoie une erreur pour protéger l'inventaire). Si vous vous êtes trompé, **annulez les mouvements de stock** générés (onglet Mouvements, bouton « Annuler ce mouvement »).

### 3.8 Créer une facture fournisseur à partir d'un bon de commande

1. Ouvrir le BC → **Créer facture**.
2. Une **facture fournisseur en brouillon** est créée dans le module Comptabilité, reprenant les lignes du BC.
3. Le bouton est désactivé si le BC est déjà **Facturé**.

### 3.9 Lancer une demande de prix (appel d'offres)

1. Onglet **Demandes de prix** → **Nouvelle demande**.
2. Renseigner le titre, le projet, l'employé, la date limite.
3. Ajouter les **articles à chiffrer** (description, quantité, unité — **sans prix**).
4. Choisir les **fournisseurs à inviter**.
5. **Créer**. La demande est numérotée `RFQ-00001`.
6. **Inviter / Envoyer** : par fournisseur, cliquer **Envoyer** pour transmettre le lien public, ou utiliser l'envoi groupé.

### 3.10 Recueillir les prix et octroyer

1. Quand des fournisseurs répondent (par le lien public ou par **Saisir prix** à l'interne), leur statut passe à **Répondu**.
2. Cliquer **Voir le comparatif** : le meilleur prix par ligne ressort en vert, avec les sous-totaux et la couverture de chiffrage.
3. Cocher **Retenir** le(s) fournisseur(s) gagnant(s).
4. Cliquer **Octroyer → générer BC**. Le système :
   - ne retient que les **lignes chiffrées** (refuse un BC à 0 $) ;
   - crée **un bon de commande par fournisseur retenu** ;
   - marque les gagnants **Retenu**, les autres **Non retenu**, et la demande **Octroyée** ;
   - met à jour le prix d'achat au catalogue du fournisseur (historique de prix).

> Une demande **Octroyée** ne peut plus être supprimée.

### 3.11 Comparer les prix de détaillants et générer un BC

1. Onglet **Prix Matériaux** → mode **Comparateur**.
2. Coller la liste des matériaux dans la fenêtre de clavardage.
3. Lire le comparatif : meilleur prix par article, total par fournisseur, plan le moins cher, plan optimisé par magasin.
4. Pour commander : **Créer un bon de commande (brouillon)**, choisir le détaillant et le fournisseur correspondant. Un BC brouillon est généré dans l'onglet Commandes.

### 3.12 Créer un mouvement de stock manuel

- **Entrée** (réception hors BC, retour de chantier) : Type Entrée, article, quantité, référence, coût. Le stock augmente.
- **Sortie** (envoi au chantier, perte) : Type Sortie, article, quantité. Le système refuse si le stock est insuffisant.
- **Ajustement** (inventaire) : voir §3.2.

### 3.13 Annuler un mouvement de stock

1. Onglet **Mouvements** → cliquer le mouvement → **Annuler ce mouvement**.
2. Le système crée un **contre-mouvement inverse** (référence préfixée « ANNUL- ») et marque l'original comme annulé.
3. Impossible d'annuler un mouvement déjà annulé, ou un contre-mouvement.

### 3.14 Déclarer une nomenclature (BOM)

1. Onglet **Produits** → cliquer le **produit parent** (l'assemblage).
2. Dans **Composants**, choisir un produit enfant, sa quantité, son unité, une note, puis ajouter.
3. Répéter pour chaque composant. La section **Utilisé dans** de l'enfant affichera le parent.

> La nomenclature est **déclarative** : assembler un produit ne consomme pas les composants en stock. Pour cela, faites une sortie de chaque composant et une entrée de l'assemblage (ou passez par un bon de travail).

### 3.15 Imprimer une étiquette code-barres

1. Onglet **Produits** → sur la ligne du produit, cliquer **Étiquette**.
2. Une étiquette **PDF Code128** s'ouvre dans un nouvel onglet, prête à imprimer.

### 3.16 Utiliser l'assistant IA

1. Onglet **Assistant IA**.
2. Poser une question (« quels produits sont sous leur seuil ? ») ou demander de créer un produit.
3. Pour une création, l'IA affiche une **carte de proposition** : vérifier, puis **Confirmer**. Le produit est créé.

---

## 4. Référence

### 4.1 Principaux points d'accès (API)

Tous préfixés par `/api/erp/v1`.

**Produits et statistiques** (`inventory.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/products` | Liste (filtres : recherche, catégorie, stock bas) | lecture |
| GET `/products/categories` | Catégories distinctes | lecture |
| GET `/products/scan` | Résolution par code-barres puis code produit | lecture |
| GET `/products/{id}` | Détail | lecture |
| POST `/products` | Créer | écriture inventaire |
| PUT `/products/{id}` | Modifier (le stock est exclu) | écriture inventaire |
| GET `/products/{id}/label` | Étiquette PDF Code128 | écriture inventaire |
| GET `/inventory/stats` | 4 cartes KPI | lecture |

**Mouvements** (`inventory.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| POST `/stock-movements` | Créer un mouvement | écriture inventaire |
| GET `/stock-movements` | Liste + filtres | lecture |
| GET `/stock-movements/{id}` | Détail | lecture |
| POST `/stock-movements/{id}/cancel` | Annuler (contre-mouvement) | écriture inventaire |

**Nomenclature (BOM)** (`inventory.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/products/{id}/composants` | Composants + « utilisé dans » | lecture |
| POST `/products/{id}/composants` | Ajouter | écriture inventaire |
| PUT `/products/{id}/composants/{cid}` | Modifier | écriture inventaire |
| DELETE `/products/{id}/composants/{cid}` | Retirer | écriture inventaire |

**Fournisseurs et bons de commande** (`suppliers.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/suppliers` | Liste | lecture |
| GET `/suppliers/{id}` | Détail (+ 20 derniers BC) | lecture |
| POST `/suppliers` | Créer | écriture achats |
| PUT `/suppliers/{id}` | Modifier | écriture achats |
| GET `/suppliers/purchase-orders` | Tous les BC | lecture |
| POST `/suppliers/{id}/orders` | Créer un BC | écriture achats |
| POST `/suppliers/orders/ai/scan` | Scanner un BC (IA Vision) | écriture achats |
| PUT `/suppliers/purchase-orders/{id}` | Modifier un BC | écriture achats |
| PUT `/suppliers/purchase-orders/{id}/dates` | Modifier les dates | écriture achats |
| PUT `/suppliers/purchase-orders/{id}/status` | Modifier le statut | écriture achats |
| POST `/suppliers/orders/{id}/lines` | Ajouter une ligne | écriture achats |
| DELETE `/suppliers/orders/{id}/lines/{lid}` | Supprimer une ligne | écriture achats |
| DELETE `/suppliers/purchase-orders/{id}` | Supprimer le BC | écriture achats |
| POST `/suppliers/orders/{id}/generate-html` | HTML imprimable | lecture |
| POST `/suppliers/orders/{id}/send` | Envoyer par courriel | écriture achats |
| GET `/suppliers/orders/public/{token}` | Vue publique (sans compte) | jeton |
| GET `/suppliers/orders/{id}/export-xlsx` | Export Excel | lecture |

**Demandes de prix (RFQ)** (`rfq.py`) : `/rfq/demandes` (créer, lister, détail, modifier, supprimer), `/rfq/demandes/{id}/lignes`, `/rfq/demandes/{id}/fournisseurs` (inviter, retirer, envoyer), `/rfq/demandes/{id}/reponses/{rid}/lignes` (saisir/lire les prix), `/rfq/demandes/{id}/comparatif`, `/rfq/demandes/{id}/octroi`, `/rfq/demandes/{id}/generate-html`, `/rfq/demandes/{id}/export-xlsx`. **Portail public** : `/public/rfq/{token}` (charger) et `/public/rfq/{token}/submit` (soumettre) — sans authentification, protégés par jeton signé.

**Prix matériaux** (`materials_pricing.py`) : `/materials/pricing/search`, `/materials/pricing/ai/chat`, `/materials/pricing/web-search`. **Aucune écriture en base.**

**Assistant IA** (`magasin_ai.py`) : `/magasin/ai/chat` (interroger + proposer), `/magasin/ai/confirm-action` (exécuter la création confirmée).

### 4.2 Statuts

| Objet | Statuts |
|---|---|
| **Bon de commande** | Brouillon · Envoyé · Confirmé · En cours · Reçu · Facturé · Annulé |
| **Demande de prix** | Brouillon · Envoyée · Clôturée · Octroyée · Annulée |
| **Réponse fournisseur (RFQ)** | Invité · Répondu · Retenu · Non retenu |

### 4.3 Types de mouvements de stock

| Type | Effet sur le stock | Règle |
|---|---|---|
| **Entrée** | `après = avant + quantité` | quantité > 0 |
| **Sortie** | `après = avant − quantité` | quantité > 0 **et** ne peut pas dépasser le stock (sinon refus) |
| **Ajustement** | `après = quantité` (valeur cible) | quantité ≥ 0 |

> Il n'existe **pas** de type « Transfert » (le module ne gère pas de sites multiples).

### 4.4 Calculs

| Élément | Formule |
|---|---|
| **Statut « Bas »** | `stock_disponible ≤ stock_minimum` **et** `stock_minimum > 0` |
| **Valeur du stock** (KPI) | `Σ [ max(stock, 0) × (coût de revient, sinon prix de vente, sinon 0) ]` |
| **Montant d'une ligne BC** | `arrondi(quantité × prix unitaire, 2)` |
| **Sous-total BC** (hors taxes) | `Σ des montants de lignes` |
| **Taxe 1 / Taxe 2 BC** | `arrondi(sous-total × taux / 100, 2)` (taux propres au BC) |
| **Total BC** (taxes incluses) | `sous-total + taxe 1 + taxe 2` |
| **Coût d'une réception** | Par produit : `montant total des lignes / quantité totale` (moyenne pondérée **des lignes de cette réception**) |

> **Précision importante sur le coût.** La « moyenne pondérée » de la réception agrège seulement les lignes d'un **même bon de commande** pour un même produit. Elle alimente le **coût du mouvement** (audit et comptabilité) mais **ne recalcule pas** le `coût de revient` du produit en moyenne mobile sur la valeur totale du stock. **Il n'y a pas de coût moyen pondéré glissant** : le coût de revient d'un produit reste la valeur **saisie à la main** sur sa fiche.

### 4.5 Taxes

Les taxes d'un bon de commande sont **figées à la création** selon la configuration du tenant (Canada multi-provinces ou États-Unis). Valeurs par défaut au Québec : **TPS 5 %**, **TVQ 9,975 %**. Un libellé de taxe vide et légitime (exonération) est préservé. Les taxes se recalculent à l'affichage à partir du sous-total.

### 4.6 Effet sur l'argent et crédits IA

Le module Magasin **ne facture rien via Stripe ou QuickBooks directement**. Le **seul effet monétaire** est le **débit de crédits IA prépayés** du tenant, pour quatre fonctions :

| Fonction | Modèle IA |
|---|---|
| Assistant IA Magasin (clavardage) | Sonnet |
| Prix Matériaux (clavardage comparateur) | Sonnet |
| Prix Matériaux (recherche web) | Opus (recherche web) |
| Scan de BC (Vision) | Sonnet |

Le coût facturé au tenant est le **coût réel des jetons** (0,003 $/millier en entrée, 0,015 $/millier en sortie) **majoré de 30 %**. Si les crédits sont épuisés, l'appel est refusé. Le service est indisponible si l'IA n'est pas configurée.

### 4.7 Listes de valeurs

**13 types de produit** (champ « Type de produit ») :

```
Béton et ciment · Bois et charpente · Acier et métal · Plomberie ·
Électricité · Isolation · Toiture · Peinture et finition ·
Quincaillerie · Revêtement · Outillage · EPI / Sécurité · Autre
```

**11 catégories de produits** (fiche fournisseur) : les mêmes que ci-dessus, **plus** « Location équipement », **sans** « Acier et métal » (ordre légèrement différent) — soit : Béton et ciment · Bois et charpente · Acier et métal · Plomberie · Électricité · Isolation · Toiture · Peinture et finition · Quincaillerie · Location équipement · Autre.

**7 normes applicables** (champ « Norme applicable ») :

```
CSA · ASTM · BNQ · ULC · ISO · LEED · Autre
```

**10 certifications de fournisseur** (cases à cocher) :

```
RBQ · CCQ · CNESST · ISO 9001 · BNQ · CSA · LEED ·
Garantie GCR · ACQ · APCHQ
```

### 4.8 Limites et garde-fous

| Règle | Effet |
|---|---|
| Sortie supérieure au stock (mouvement manuel) | Refusée |
| Dévalider un BC déjà « Reçu » | Refusée — annuler les mouvements de stock à la place |
| Modifier une ligne d'un BC « Reçu » ou « Facturé » | Refusée |
| Supprimer un BC « Reçu », « Facturé » ou déjà comptabilisé | Refusée |
| Supprimer une demande de prix « Octroyée » | Refusée |
| Octroyer une RFQ sans aucune ligne chiffrée | Refusée (pas de BC à 0 $) |
| Modifier le stock directement sur la fiche produit | Impossible — passer par un mouvement d'ajustement |
| Créer un composant BOM en boucle (référence circulaire) | Refusée |
| Montants hors bornes (dépassement numérique) | Refusés proprement (message d'erreur, pas de plantage) |
| Fichier de scan BC supérieur à 20 Mo | Refusé |
| Portail public RFQ : trop de requêtes / heure / adresse | Limité |

### 4.9 Tables PostgreSQL (par tenant)

| Table | Rôle |
|---|---|
| `produits` | Catalogue (nom, code, prix, coût, stock, seuil, catégorie, norme, emplacement, actif) |
| `mouvements_stock` | Historique des mouvements (type, quantités avant/après, coût, employé, projet, BC, annulation) |
| `produit_composants` | Nomenclature parent-enfant (unique par paire) |
| `fournisseurs` | Fiches fournisseurs (liées à une entreprise du carnet) |
| `bons_commande` | En-têtes de BC (numéro, fournisseur, projet, statut, taxes, totaux, jeton public) |
| `bon_commande_lignes` | Lignes de BC (partagées aussi par les demandes de prix, via des colonnes de rôle) |
| `rfq_demandes` / `rfq_reponses` | En-têtes de demandes de prix et réponses fournisseurs |
| `produit_fournisseurs` / `produit_historique_prix` | Prix d'achat par fournisseur et historique (alimentés à l'octroi d'une RFQ) |
| `public.bc_public_tokens` | Jetons des liens publics de BC (90 jours) |

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|---|---|
| **Dossiers (Fiche 360)** | Un BC peut être ouvert directement depuis un dossier (`/magasin?open=<id>`) ; l'octroi d'une RFQ et la création d'un BC peuvent se rattacher au dossier du projet. |
| **Projets** | Un BC, un mouvement et une demande de prix peuvent être associés à un projet. |
| **Bons de travail (BT)** | Les sorties de matériel vers le chantier passent souvent par les bons de travail ; ce module trace la contrepartie en stock. |
| **Comptabilité** | Le bouton **Créer facture** génère une facture fournisseur brouillon ; la réception d'un BC déclenche, via le mouvement de stock, l'écriture au grand livre (déclenchée par la base). |
| **Configuration** | Les taux de taxes (TPS/TVQ, provinces, États-Unis), le logo et le thème des documents proviennent de la Configuration de l'entreprise. |
| **Crédits IA** | Les fonctions IA du Magasin consomment les crédits prépayés du tenant (voir §4.6). |
| **Employés** | Le responsable d'un BC et l'auteur d'un mouvement sont choisis parmi les employés actifs. |

### 5.2 Cycle d'achat recommandé

1. (Optionnel) **Demande de prix** à plusieurs fournisseurs → comparatif → **octroi** (génère les BC).
2. **Bon de commande** : compléter, **envoyer** au fournisseur.
3. À la livraison : passer le BC à **Reçu** → le **stock est alimenté** automatiquement.
4. (Optionnel) **Créer facture** fournisseur → Comptabilité.

### 5.3 FAQ

**La réception d'un bon de commande met-elle le stock à jour automatiquement ?**
Oui. Passer un BC à **Reçu** augmente le stock des produits liés et crée un mouvement d'entrée par produit. (C'est un changement majeur par rapport aux anciennes versions, où la réception n'avait aucun effet sur le stock.)

**Puis-je annuler une réception faite par erreur ?**
Vous ne pouvez pas ramener le BC en arrière (il reste « Reçu » pour l'audit). La correction se fait en **annulant les mouvements de stock** générés (onglet Mouvements → « Annuler ce mouvement »), ce qui recrée un contre-mouvement inverse.

**Comment corriger une quantité en stock ?**
Uniquement par un **mouvement d'ajustement**. Le champ « Quantité actuelle » de la fiche produit est en lecture seule en édition.

**Y a-t-il un coût moyen pondéré automatique ?**
Non. Le **prix de vente** et le **coût de revient** sont saisis à la main sur la fiche produit. Le coût d'un mouvement de réception (moyenne des lignes de ce BC) sert à l'audit et à la comptabilité, mais ne met pas à jour le coût de revient du produit.

**Peut-on gérer plusieurs entrepôts ou faire des transferts entre sites ?**
Non. Le stock est **global par produit**. Le champ « Emplacement stock » est un simple texte informatif. Il n'y a ni entrepôts multiples, ni transferts, ni type de mouvement « Transfert ».

**Y a-t-il un tableau de bord ou une gestion de catégories dédiée dans le Magasin ?**
Non. Certaines étiquettes internes (entrepôts, tableau de bord, catégories, transferts) subsistent dans le code comme vestiges d'une ancienne version **mais ne sont rendues nulle part**. Les seuls onglets réels sont les 7 décrits ici.

**Existe-t-il un onglet « Bons d'achat » ou « réquisitions » ?**
Non. Il n'y a **pas** de document « bon d'achat » distinct dans le Magasin. La forme la plus proche d'une réquisition est la **Demande de prix** (RFQ). (Des commandes « bon d'achat » existent dans des outils techniques externes, mais ne correspondent à aucun onglet de cette page.)

**Le fournisseur a-t-il besoin d'un compte pour voir un BC ou répondre à une demande de prix ?**
Non. Le BC envoyé génère un **lien public** (90 jours). La demande de prix génère un **lien sécurisé** par fournisseur. Aucun compte n'est requis, mais l'accès est protégé par jeton signé à durée limitée.

**Un employé de terrain voit-il les coûts d'achat ?**
Non. Le coût de revient et le coût des mouvements sont **masqués** aux rôles de terrain (rôle « employé »). Seuls les rôles de gestion les voient.

**Le comparateur de prix (Canac / BMR / Home Depot) est-il fiable à 100 % ?**
Non — il interroge des services web **non officiels** des détaillants. Des résultats partiels sont normaux (Home Depot est souvent bloqué). L'onglet peut même être absent si le service n'a pas démarré. Vérifiez toujours les prix avant de commander.

**Le scan de BC crée-t-il directement une commande ?**
Non. Il **extrait** les lignes dans la fenêtre de création ; vous devez vérifier puis **Créer le bon de commande**. Il consomme des crédits IA.

**Puis-je supprimer un bon de commande ?**
Oui, tant qu'il n'est ni **Reçu**, ni **Facturé**, ni déjà comptabilisé. Sinon la suppression est refusée.

**L'assistant IA peut-il modifier le stock ou créer un bon de commande ?**
Non. Sa seule action d'écriture est la **création de produit**, et seulement après confirmation humaine.

**Peut-on importer un catalogue en masse (CSV/Excel) ?**
Il n'y a pas de bouton d'importation en masse dans cette page. On peut exporter un BC (Excel, CSV QuickBooks) et un comparatif de RFQ, mais l'importation de catalogue passe par d'autres outils.

### 5.4 Ce qui n'existe pas (limites connues)

- Pas d'entrepôts multiples, pas de transferts inter-sites, pas de mouvement « Transfert ».
- Pas de tableau de bord dédié, pas de gestion de catégories dédiée dans le Magasin.
- Pas d'onglet « Bons d'achat / réquisitions » (la RFQ tient ce rôle).
- Pas de coût moyen pondéré glissant sur la valeur d'inventaire.
- Pas de modification directe du stock sur la fiche (uniquement par mouvement).
- Pas de suivi de lots, de dates de péremption, ni d'unités imbriquées (ex. « 1 sac = 25 kg »).
- Pas de bouton d'impression dédié (l'impression passe par l'aperçu HTML « Ouvrir dans un nouvel onglet »).
- Pas d'importation de catalogue en masse depuis cette page.

---

## 6. Récapitulatif

- Le **Magasin** (`/magasin`, section Opérations) réunit **7 onglets** : **Commandes**, **Demandes de prix**, **Prix Matériaux**, **Mouvements**, **Produits**, **Fournisseur**, **Assistant IA**. Onglet par défaut : **Commandes**.
- **4 cartes KPI** en permanence : Produits, Alerte seuil minimal, Valeur du stock, Catégories.
- **Bons de commande** : 7 statuts (Brouillon → Envoyé → Confirmé → En cours → **Reçu** → Facturé, plus Annulé), envoi par courriel avec **lien public 90 jours**, scan de BC par IA, export HTML / Excel / CSV QuickBooks, création de facture fournisseur.
- **La réception d'un BC (statut « Reçu ») alimente automatiquement le stock** et crée un mouvement d'entrée par produit. **Réception définitive** : pour corriger, on annule les mouvements.
- **Mouvements** : 3 types (Entrée, Sortie, Ajustement — **pas** de Transfert), traçabilité complète (stock avant/après, employé, projet, BC, coût), annulation par contre-mouvement, saisie rétroactive possible.
- **Le stock ne se modifie jamais à la main** : uniquement par un mouvement d'**ajustement**.
- **Prix de vente et coût de revient sont saisis manuellement** : **pas de coût moyen pondéré** glissant.
- **Demandes de prix (RFQ)** : appel d'offres à plusieurs fournisseurs, **portail public** par jeton signé, **comparatif** (meilleur prix par ligne, couverture), **octroi** qui génère un BC par fournisseur retenu.
- **Prix Matériaux** : comparateur Canac / BMR / Home Depot (services non officiels, résultats partiels possibles) + recherche web, avec création de BC brouillon.
- **Assistant IA** : lit les données, **crée des produits sur confirmation** (rien d'autre).
- **Permissions** : lecture ouverte ; écriture inventaire (gestionnaire / magasinier / admin) ; écriture achats (inclut le rôle utilisateur) ; écriture RFQ (gestionnaire / magasinier / admin). **Coûts masqués** aux rôles de terrain.
- **Effet argent** limité aux **crédits IA prépayés** (4 fonctions IA, coût réel des jetons majoré de 30 %).
- **N'existe pas** : entrepôts multiples, transferts, tableau de bord dédié, bons d'achat, coût moyen glissant, suivi de lots/péremption, importation de catalogue en masse.

---

**Documentation générée à partir du code source** : `MagasinPage.tsx`, `RfqTab.tsx`, `MaterialPricingTab.tsx`, `MaterialWebSearch.tsx`, `MagasinAssistantTab.tsx`, `api/inventory.ts`, `api/suppliers.ts`, `api/rfq.ts`, `api/magasinAi.ts` ; `backend/routers/inventory.py`, `suppliers.py`, `rfq.py`, `materials_pricing.py`, `magasin_ai.py`.

**Manuels liés** :
- Module 11 — Bons de Travail (sorties de matériel) — `11-operations-bons-de-travail.md`
- Module 13 — Bons de Commande / Achats (vue complémentaire) — `13-operations-bons-de-commande.md`
- Module 14 — Comptabilité (factures fournisseurs, grand livre) — `14-operations-comptabilite.md`
- Module 08 — Projets (association des BC et mouvements) — `08-ventes-projets.md`
- Module 06 — Dossiers / Fiche 360 (ouverture directe d'un BC) — `06-ventes-dossiers.md`
- Module 30 — Configuration (taxes, thème des documents) — `30-configuration.md`
- Module 24 — Assistant IA (crédits IA) — `24-communication-assistant-ia.md`
