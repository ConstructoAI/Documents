# Module 14 — Bons de commande (achats fournisseurs)

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/MagasinPage.tsx` (≈ 3 425 lignes — page unique du Magasin ; les bons de commande vivent dans l'onglet « Commandes »), `frontend/src/components/rfq/RfqTab.tsx` (≈ 1 412 lignes — pont Demandes de prix → bon de commande), API `frontend/src/api/suppliers.ts`, `frontend/src/api/rfq.ts`, `frontend/src/api/accounting.ts`
> - Backend : `backend/routers/suppliers.py` (≈ 2 935 lignes — fournisseurs + bons de commande, 15 points d'accès dédiés aux BC), `backend/routers/rfq.py` (≈ 2 434 lignes — octroi d'une demande de prix qui génère les BC), `backend/routers/accounting.py` (facturation fournisseur + comptabilisation automatique au grand livre), `backend/routers/inventory.py` (contre-mouvement d'annulation d'une réception)
> - Préfixe API : `/api/erp/v1`
> **Tables PostgreSQL (par tenant)** : `bons_commande` (en-tête du BC), `bon_commande_lignes` (lignes d'articles — table **partagée** avec les demandes de prix), `mouvements_stock` et `produits` (réception valorisée), `dossier_achats` (rattachement au dossier d'opportunité), `fournisseurs` / `companies` (nom du fournisseur) ; table partagée entre tenants : `public.bc_public_tokens` (jetons des liens publics).
> **Cadrage** : un **bon de commande** (BC) est le document d'engagement d'achat que l'entreprise transmet à un fournisseur. Ce module couvre tout le **cycle d'achat** : **créer** un BC (à la main, par **scan IA** d'une facture ou d'une soumission, ou par **octroi** d'une demande de prix), le **compléter** (lignes d'articles, livraison, notes), l'**envoyer** au fournisseur (courriel + lien public), le **suivre** par statut, le **recevoir** — ce qui **alimente automatiquement le stock** et calcule un coût de réception — puis le **facturer** (facture fournisseur en brouillon). **Point de vocabulaire important** : il n'existe **aucune entrée de menu « Bons de commande »**. Les BC sont l'**onglet « Commandes »** du menu **Magasin** (`/magasin`), qui est l'onglet ouvert par défaut. Ce manuel est le complément détaillé du **Module 10 — Magasin** (qui présente les sept onglets d'ensemble) : il se concentre sur les bons de commande et sur le pont depuis les demandes de prix.

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

Le module Bons de commande est le **poste d'achat** de l'entreprise. Il permet de :

- **créer** un bon de commande numéroté (`BC-00012`) au nom d'un fournisseur, rattaché au besoin à un projet et à un employé responsable ;
- **remplir** le bon : lignes d'articles (issues du catalogue de produits ou saisies à la main), adresse et conditions de livraison, notes internes ;
- accélérer la saisie par un **scan IA** : téléverser l'image ou le PDF d'un bon, d'une facture ou d'une soumission de fournisseur, et laisser l'intelligence artificielle en extraire les lignes ;
- **transmettre** le bon au fournisseur par **courriel**, avec un **lien public** consultable sans compte pendant 90 jours ;
- **suivre** l'avancement par un statut coloré (Brouillon, Envoyé, Confirmé, En cours, Reçu, Facturé, Annulé) ;
- **recevoir** la marchandise : passer le bon à **Reçu** **augmente automatiquement le stock** des produits liés et inscrit un mouvement d'entrée valorisé, de façon atomique et anti-double-comptage ;
- **exporter** le bon en HTML imprimable, en Excel (.xlsx) ou en CSV compatible QuickBooks ;
- **facturer** le bon : créer une **facture fournisseur en brouillon** dans la Comptabilité, ce qui fige le bon au statut Facturé ;
- récupérer les bons **générés par l'octroi** d'une demande de prix (appel d'offres à plusieurs fournisseurs).

### 1.2 Ce que le module ne fait PAS

- **Il n'y a pas de menu « Bons de commande ».** Le menu latéral affiche **Magasin** (icône panier, section Opérations). Les bons de commande sont l'**onglet « Commandes »** de cette page. C'est une correction importante par rapport aux anciennes versions de ce manuel.
- **Pas de réception partielle.** Un bon se reçoit **en entier**, d'un seul coup. Il n'existe pas de « quantité reçue » par ligne, ni de réceptions multiples ou échelonnées.
- **La réception est irréversible.** Une fois le bon passé à **Reçu**, on ne peut plus le ramener vers un autre statut (le système renvoie une erreur pour protéger l'inventaire). Une réception erronée se corrige en **annulant les mouvements de stock** générés (onglet Mouvements), pas en dévalidant le bon.
- **Les lignes se verrouillent après réception ou facturation.** On ne peut plus ajouter ni supprimer de ligne sur un bon **Reçu** ou **Facturé**.
- **La suppression est bloquée** pour un bon **Reçu**, **Facturé** ou déjà **comptabilisé**.
- **Pas de boutons de flux de travail** (« Confirmer », « Marquer En cours »…). Le statut se change **à la main** dans un menu déroulant. Seul l'**envoi par courriel** fait avancer automatiquement le statut (et uniquement de Brouillon à Envoyé).
- **Le coût de revient du produit n'est pas recalculé à la réception.** La réception calcule un coût de réception (moyenne pondérée des lignes du bon) et l'inscrit sur le **mouvement de stock** ; elle **n'écrase pas** le coût de revient saisi sur la fiche produit. Il n'y a **pas de coût moyen pondéré glissant**.
- **Deux taxes au maximum** (par exemple TPS et TVQ), **figées** à la création du bon.
- **Le scan IA n'écrit rien.** Il extrait les lignes dans la fenêtre de création ; vous devez vérifier, corriger, puis créer le bon vous-même.

### 1.3 Où trouver les bons de commande (accès)

- **Menu latéral** → section **Opérations** → **Magasin** (icône panier).
- **Adresse** : `/magasin`.
- **Onglet par défaut** : **Commandes** (c'est le premier onglet ; il s'ouvre automatiquement à l'arrivée sur la page).
- **Ouverture directe d'un bon** : un lien du type `/magasin?open=<id>` (par exemple depuis la fiche 360 d'un dossier) ouvre directement le bon de commande concerné.
- Page protégée : il faut être authentifié dans un tenant.

> **Rappel du point de vocabulaire.** Cherchez « Magasin » dans le menu, puis l'onglet « Commandes ». Il n'existe aucune entrée « Bons de commande » ni « Achats » dans la barre latérale. L'onglet des fiches fournisseurs, lui, s'appelle **« Fournisseur »** (au singulier).

### 1.4 Permissions et rôles

La **consultation** des bons de commande est ouverte à tout utilisateur authentifié du tenant. L'**écriture** dépend du rôle, et deux gardes distinctes se croisent dans ce module :

| Garde | Ce qu'elle protège | Rôles autorisés |
|-------|--------------------|-----------------|
| **Écriture achats** (`require_purchase_write`) | Créer / modifier / supprimer un bon de commande, ajouter / supprimer des lignes, changer les dates et le statut, envoyer par courriel, scanner un BC, créer / modifier un fournisseur | administrateur (`is_admin`), super-admin, **admin**, **gestionnaire**, **magasinier** ou **utilisateur** |
| **Écriture demandes de prix** (`require_rfq_write`) | Créer / gérer une demande de prix, inviter des fournisseurs, saisir les prix et **octroyer** (ce qui génère les BC) | administrateur (`is_admin`), super-admin, admin, gestionnaire, **magasinier** |

> **Nuance à retenir.** Le rôle **utilisateur** (« user ») peut créer et recevoir des bons de commande, mais **ne peut pas octroyer une demande de prix** : l'octroi exige un rôle **gestionnaire** ou **magasinier** (ou administrateur). À l'inverse, les rôles **employé** et **comptable** ne peuvent **pas** écrire dans les achats (lecture seule). La garde `require_purchase_write` teste le **rôle réel** de l'utilisateur (et non le type de jeton), afin de refuser effectivement « employé » et « comptable », tout en admettant d'office tout administrateur.
>
> **Facturation.** La création d'une facture fournisseur à partir d'un bon (bouton « Créer facture ») s'appuie sur la Comptabilité et n'exige qu'un utilisateur authentifié ; le mode consultation et les autres protections d'accès s'appliquent en amont.

### 1.5 Cartes de statistiques (KPI)

Quatre cartes chiffrées coiffent en permanence la page Magasin, **au-dessus** de la barre d'onglets. Elles concernent **l'inventaire dans son ensemble**, pas les bons de commande : **Produits** (nombre de produits actifs), **Alerte seuil minimal** (produits sous leur seuil), **Valeur du stock** (en dollars) et **Catégories**. Ces cartes sont décrites en détail au **Module 10 — Magasin**. Il n'y a **pas** de carte KPI propre aux bons de commande.

### 1.6 Concepts clés

- **Bon de commande (BC)** : l'engagement d'achat auprès d'un fournisseur. Techniquement, une ligne de la table `bons_commande`, numérotée `BC-00012` (cinq chiffres). Porte un fournisseur, un projet, un employé responsable, des dates, un statut, une adresse et des conditions de livraison, deux taxes figées, des totaux et des notes.
- **Ligne d'article** : un matériau ou un service commandé. Chaque ligne porte une description, une quantité, une unité, un prix unitaire et un montant (quantité × prix). Une ligne peut être **liée à un produit** du catalogue (ce qui permet la réception en stock) ou rester **libre** (hors inventaire).
- **Statut** : l'état du bon dans son cycle de vie. Sept valeurs (voir §1.7). Le passage à **Reçu** est le seul qui a un effet matériel (le stock).
- **Réception** : l'action de passer le bon à **Reçu**. Elle **augmente le stock** des produits liés et crée un **mouvement d'entrée** valorisé par produit. Elle est **atomique** (tout ou rien) et **définitive**.
- **Coût de réception** : pour un produit reçu, le coût unitaire inscrit sur le mouvement = **montant total des lignes de ce produit ÷ quantité totale** (moyenne pondérée des lignes **de ce bon**). Ce coût sert à l'audit et à la comptabilité ; il **ne met pas à jour** le coût de revient de la fiche produit.
- **Lien public** : un jeton qui permet au fournisseur de consulter le bon **sans compte**, pendant 90 jours. Créé à l'envoi par courriel.
- **Octroi** : l'action, dans une **demande de prix** (RFQ), de retenir des fournisseurs, ce qui **génère un bon de commande par fournisseur retenu** (voir §2.9 et §3.12).
- **Instantané des taxes** : les deux taux de taxe du bon sont **figés à sa création** d'après la configuration du tenant (Québec par défaut : TPS 5 %, TVQ 9,975 %). Modifier plus tard la configuration de l'entreprise ne touche pas les bons déjà créés.

### 1.7 Cycle de vie d'un bon de commande

Sept statuts sont reconnus par le système (`VALID_BC_STATUTS`, `suppliers.py:1206`) :

| Statut | Couleur du badge | Signification |
|--------|------------------|---------------|
| **Brouillon** | gris | Créé, pas encore transmis |
| **Envoyé** | indigo | Transmis au fournisseur (par courriel) |
| **Confirmé** | bleu | Le fournisseur a confirmé la commande |
| **En cours** | violet | Livraison en cours |
| **Reçu** | vert | Marchandise reçue → **le stock a été alimenté** |
| **Facturé** | sarcelle | Une facture fournisseur a été créée depuis le bon |
| **Annulé** | rouge | Commande annulée |

> **Statuts « libres » sauf deux exceptions.** Il n'existe **pas** de machine à états stricte pour les bons de commande : vous choisissez le statut à la main dans le menu déroulant du panneau de détail. Les statuts **Confirmé** et **En cours** sont valides et affichés, mais **aucune action automatique** ne les déclenche — ils se posent manuellement, pour votre suivi. **Deux transitions sont automatiques** : (1) l'**envoi par courriel** fait passer un bon de **Brouillon** à **Envoyé** (et seulement dans ce sens) ; (2) la **création d'une facture** fait passer le bon à **Facturé**. Le passage à **Reçu**, lui, déclenche la réception en stock — et devient définitif.

---

## 2. Interface

### 2.1 Disposition générale

```
+------------------------------------------------------------------+
|  Titre « Magasin »                                               |
+------------------------------------------------------------------+
|  [Produits]  [Alerte seuil]  [Valeur du stock]  [Catégories]     |  <- 4 cartes KPI (inventaire)
+------------------------------------------------------------------+
|  Commandes | Demandes de prix | Prix Matériaux | Mouvements |    |  <- barre d'onglets
|  Produits | Fournisseur | Assistant IA                          |
+------------------------------------------------------------------+
|  Barre d'actions de l'onglet « Commandes »                      |
+----------------------------+-------------------------------------+
|  Liste des bons (tableau)  |  Panneau de détail du bon (droite)  |
+----------------------------+-------------------------------------+
```

L'onglet **Commandes** est une vue **maître-détail** : la liste des bons à gauche, et, quand on sélectionne un bon, le **panneau de détail** à droite. Sur téléphone, le tableau se replie en cartes simplifiées et le détail s'affiche en pleine largeur.

### 2.2 Barre d'actions de l'onglet « Commandes »

Deux boutons et une recherche coiffent la liste :

- **Commande fournisseur** (bouton principal, icône « + ») : ouvre la fenêtre de création d'un bon de commande (voir §2.6). Ce même bouton est aussi présent dans la barre de l'onglet **Fournisseur**.
- **Scanner un BC (IA)** (icône de numérisation) : ouvre un sélecteur de fichier (image ou PDF) pour l'extraction assistée par IA. Pendant le traitement, le bouton affiche **« Analyse en cours... »** (voir §2.7).
- **Recherche** : filtre la liste sur le numéro, le fournisseur ou le projet.

### 2.3 Tableau des bons de commande

Colonnes (triables et redimensionnables) :

| Colonne | Contenu | Particularité |
|---------|---------|---------------|
| **Numéro** | `BC-00012` | Affiché en couleur de lien |
| **Fournisseur** | Nom du fournisseur | « -- » si vide |
| **Projet** | Projet associé | « -- » si vide |
| **Montant** | Montant du bon (hors taxes) | Aligné à droite |
| **Date Commande** | Date de la commande | **Modifiable directement au clic** (champ date en ligne) |
| **Livraison Prévue** | Date de livraison prévue | **Modifiable directement au clic** |
| **Statut** | Badge de couleur | Voir §1.7 |
| (actions) | — | Bouton **Supprimer** (poubelle) |

**Comportements** :

- **Clic sur une ligne** (hors boutons) : ouvre le **panneau de détail** du bon. Accessible au clavier (Entrée / Espace).
- **Édition des dates en ligne** : cliquer une cellule **Date Commande** ou **Livraison Prévue** la transforme en champ date. En quittant le champ, la date est enregistrée (point d'accès dédié). Laisser le champ **vide efface** la date. Une protection anti-course évite qu'une réponse en retard n'écrase une saisie plus récente.
- **Supprimer** (poubelle, par ligne) : demande une confirmation (« Supprimer ce bon de commande ? »). Refusé par le serveur si le bon est Reçu, Facturé ou déjà comptabilisé (voir §3.11).
- **Liste vide** : « Aucun bon de commande ».
- **Pagination** : 20 bons par page par défaut (ajustable, maximum 100). Tri par colonne au clic sur l'en-tête.

> **Le « Montant » de la liste est un montant hors taxes.** La colonne affiche le sous-total (`montant_total`), c'est-à-dire la somme des lignes **avant** taxes. Le détail du bon, lui, affiche le sous-total, les taxes et le total taxes incluses (voir §2.5).

### 2.4 Les sept statuts et leurs couleurs

Voir le tableau du §1.7. Le badge de statut apparaît dans la liste et dans le panneau de détail. Si un bon porte un statut inattendu (par exemple hérité d'une ancienne version), il est affiché en gris et reste modifiable.

### 2.5 Panneau de détail d'un bon de commande

En sélectionnant un bon, un panneau s'ouvre à droite. Ses sections, de haut en bas :

**En-tête**
- Numéro du bon (police à chasse fixe) et nom du fournisseur (par défaut « Fournisseur » si le nom manque).
- **Menu déroulant de statut** coloré : les sept statuts standards. Si le bon porte une valeur hors liste, elle est ajoutée au menu pour ne pas la perdre. Changer le statut marque le bon comme « modifié » (il faudra **Sauvegarder**).
- Bouton **Fermer** (X). Sur téléphone, un bouton « Retour aux bons de commande ».

**Section « Informations »**
- **Date de commande** (champ date).
- **Date de livraison prévue** (champ date).
- **Employé responsable** : menu des employés **actifs** (option « Aucun »).
- **Projet associé** : menu des projets (option « Aucun »).

**Section « Livraison »**
- **Adresse de livraison** (zone de texte ; exemple « 123 rue Exemple, Granby, QC J2G 8R5 »).
- **Conditions de livraison** (zone de texte ; « Délais, accès chantier, contact sur place... »).

**Section « Articles ({nombre}) »**
- Menu **« Sélectionner article d'inventaire »** : choisir un produit du catalogue **remplit automatiquement** la description, l'unité et le prix.
- Champs **Description**, **Qté**, **Prix unit. ($)**.
- Bouton **« Ajouter ligne »** : ajoute l'article. Désactivé tant qu'aucune description ni produit n'est saisi. Le serveur exige une **quantité supérieure à zéro** et un **produit ou une description**.
- Chaque ligne existante affiche « description » puis « quantité unité × prix = **montant** », avec un bouton pour la supprimer.
- Vide : « Aucune ligne. Ajoutez des articles ci-dessus. »

**Section « Totaux »** (si le bon a des lignes)
- **Sous-total (HT)**, **taxe 1** (par défaut TPS 5 %), **taxe 2** (par défaut TVQ 9,975 %), **Total (taxes incluses)**. Les taxes proviennent de la configuration **figée sur le bon** ; une ligne de taxe est masquée si son taux vaut zéro.

**Section « Notes internes »** (zone de texte)

**Actions — première rangée**

| Bouton | Effet |
|--------|-------|
| **Générer HTML** | Télécharge le bon en fichier `.html` imprimable (`{numéro}.html`) |
| **Aperçu** | Ouvre le document dans une fenêtre intégrée (voir §2.8) |
| **Envoyer au fournisseur** | Ouvre la fenêtre d'envoi par courriel (voir §2.8) |
| **CSV QuickBooks** | Télécharge un fichier CSV compatible QuickBooks (`{numéro}_quickbooks.csv`), généré **côté navigateur** |
| **Copier CSV** | Copie ce même CSV dans le presse-papiers |
| **Excel (.xlsx)** | Télécharge le bon en tableur |
| **Créer facture** | Crée une facture fournisseur (brouillon) à partir du bon. **Désactivé** si le bon est déjà **Facturé** (infobulle « Déjà facturé ») |

**Actions — deuxième rangée**
- **Supprimer** (danger) : supprime le bon (mêmes règles qu'au §3.11).
- **Sauvegarder** : enregistre les modifications. **Désactivé** tant qu'aucun champ n'a changé (infobulle « Aucune modification à sauvegarder »).

> **Sauvegarde ciblée (protection anti-écrasement).** Le bouton **Sauvegarder** n'envoie au serveur **que les champs réellement modifiés**. C'est ce qui vous évite, en rouvrant un vieil onglet, d'écraser par mégarde le statut d'un bon qu'un collègue vient de passer à « Reçu » (ce qui, autrement, aurait pu le faire reculer vers « Brouillon » et défaire silencieusement la réception). C'est aussi par ce bouton **Sauvegarder** que passe la **réception** : choisissez « Reçu » dans le menu de statut, puis **Sauvegarder**.

### 2.6 Fenêtre « Créer un nouveau bon de commande »

Ouverte par **Commande fournisseur**. Ses sections :

- **Bandeau d'extraction IA** (seulement après un scan) : « {n} ligne(s) extraite(s) par l'IA », l'indice de **confiance** (haute / moyenne / basse), le total extrait, et un avertissement si le fournisseur n'a pas été reconnu (« Vérifiez et corrigez les données avant de créer le bon de commande. »).
- **Fournisseur** (obligatoire) : menu **« Sélectionner un fournisseur * »**. Sans fournisseur, la création est bloquée.
- **Assignation** : **Projet associé** (« Aucun projet »), **Employé responsable** (« Aucun (par défaut : utilisateur connecté) »), **Date de livraison prévue** (« Optionnel. Visible sur le PDF envoyé au fournisseur. »).
- **Articles** : une ligne par article. Choisir un **produit** du catalogue **ou** « Saisie manuelle (hors inventaire) », puis **Qté**, **Prix unit. $** et le **Montant** (calculé). En saisie manuelle, deux champs de plus : **Description** et **Unité** (« UN, m, pi, h... »). Bouton **« Ajouter un article »**. Un **récapitulatif des taxes en direct** montre le sous-total, la TPS, la TVQ et le total.
- **Livraison (optionnel)** : adresse et conditions.
- **Notes (optionnel)** : notes internes (« Notes visibles à l'interne, non imprimées sur le bon de commande. »).
- Pied : **Annuler** / **Créer le bon de commande** (désactivé sans fournisseur).

> **Comment fonctionne réellement la création.** Le serveur crée d'abord l'**en-tête** du bon avec ses lignes (statut Brouillon, numéro `BC-00012`). Il enregistre **ensuite**, par un second appel, les champs « avancés » (employé responsable, adresse et conditions de livraison). Si ce second appel échoue, le bon existe quand même et un message vous avertit que « certains champs avancés n'ont pas été sauvegardés » (à compléter depuis le panneau de détail). De même, si une ligne n'a pas pu être ajoutée, un message indique « X/Y ligne(s) non ajoutée(s) ».

### 2.7 Scan IA d'un bon de commande (Vision)

Le bouton **Scanner un BC (IA)** (onglet Commandes) permet de partir d'un document existant plutôt que de tout saisir :

1. Choisir une **image** (PNG, JPEG, GIF, WebP, BMP) ou un **PDF**, jusqu'à **20 Mo**.
2. L'intelligence artificielle (modèle Vision) lit le document et **pré-remplit** la fenêtre de création : fournisseur détecté, dates, lignes d'articles, notes.
3. Vous **vérifiez et corrigez**, choisissez le fournisseur si l'IA ne l'a pas reconnu, puis créez le bon par le flux normal.

> **Le scan ne crée rien tout seul.** C'est une opération **en lecture seule** : aucune donnée n'est écrite tant que vous n'avez pas cliqué sur **Créer le bon de commande**. Le scan **consomme des crédits IA** prépayés (voir §4.8). Si les crédits sont épuisés ou si l'IA n'est pas configurée, le scan est refusé.

### 2.8 Envoi au fournisseur et aperçu HTML

**Fenêtre « Envoyer le bon de commande »** (bouton **Envoyer au fournisseur**) :
- Saisir le **courriel du fournisseur**, puis **Envoyer**.
- Le système génère un **lien public partageable valable 90 jours** (copiable via « Copier le lien »), qui affiche le bon en lecture seule sans authentification.
- Le courriel, thématisé aux couleurs de l'entreprise, est envoyé au fournisseur.
- Le statut passe à **Envoyé** **uniquement si le bon était en Brouillon** ; un bon déjà plus avancé conserve son statut.

**Fenêtre d'aperçu HTML** (bouton **Aperçu**) :
- Affiche le document dans une fenêtre intégrée (cadre en bac à sable).
- Bouton **« Ouvrir dans un nouvel onglet »** (protégé contre le détournement d'onglet) et **« Fermer »**.

### 2.9 Le pont « Demandes de prix → bon de commande » (octroi)

Les bons de commande peuvent aussi **naître d'une demande de prix** (RFQ), c'est-à-dire d'un appel d'offres à plusieurs fournisseurs. Ce cycle vit dans l'onglet voisin **« Demandes de prix »** (décrit au **Module 10**). Le point de jonction est l'**octroi** :

- Après avoir comparé les offres, on **coche les fournisseurs retenus**, puis on clique **« Octroyer → générer BC »** (confirmation « Octroyer aux fournisseurs sélectionnés et générer les bons de commande ? »).
- Le système crée **un bon de commande par fournisseur retenu**, au statut **Brouillon**, avec la note « Issu de la demande de prix {numéro} ».
- Seules les **lignes réellement chiffrées** (prix supérieur à zéro) sont reportées ; si une réponse n'a aucune ligne chiffrée, l'octroi de ce fournisseur est refusé (jamais de bon à 0 $).
- L'adresse et les conditions de livraison de la demande sont recopiées sur les bons.
- Un message confirme : « Octroi effectué. Bon(s) de commande : {numéros} ». Les bons apparaissent alors dans l'onglet **Commandes**, prêts à être complétés et envoyés.

> Les bons issus d'un octroi se comportent **exactement** comme des bons créés à la main : mêmes statuts, même réception, même facturation.

---

## 3. Processus pas à pas

### 3.1 Créer un bon de commande à la main

1. Onglet **Commandes** → **Commande fournisseur** (ou onglet **Fournisseur** → **Commande fournisseur**).
2. Choisir le **fournisseur** (obligatoire).
3. Renseigner au besoin le **projet**, l'**employé responsable** et la **date de livraison prévue**.
4. Ajouter les **articles** : choisir un produit du catalogue (description, unité et prix se remplissent) ou saisir une ligne à la main (description + unité). Les taxes se calculent en direct.
5. Renseigner au besoin l'**adresse** et les **conditions de livraison**, et des **notes internes**.
6. **Créer le bon de commande**. Il apparaît en statut **Brouillon**, numéroté `BC-00012`.

> Si un message signale des « champs avancés non sauvegardés » ou des « ligne(s) non ajoutée(s) », ouvrez le bon et complétez-le depuis le panneau de détail.

### 3.2 Créer un bon par scan IA

1. Onglet **Commandes** → **Scanner un BC (IA)**.
2. Choisir une **image** ou un **PDF** (jusqu'à 20 Mo). Le bouton affiche « Analyse en cours... ».
3. À la fin, la fenêtre de création s'ouvre pré-remplie, avec un bandeau (nombre de lignes, confiance, total).
4. **Vérifier et corriger** les lignes ; choisir le fournisseur si non reconnu.
5. **Créer le bon de commande**.

> Le scan consomme des crédits IA. Il ne crée rien tant que vous n'avez pas confirmé.

### 3.3 Ajouter, modifier ou supprimer des lignes

1. Ouvrir le bon (clic sur la ligne) → section **Articles**.
2. **Ajouter** : choisir un article d'inventaire (remplissage automatique) ou saisir la description, la quantité et le prix, puis **Ajouter ligne**.
3. **Supprimer** une ligne : bouton « Supprimer la ligne » de la ligne concernée.
4. Le sous-total, les taxes et le total se recalculent automatiquement.

> **Les lignes se verrouillent après réception ou facturation.** Le serveur **refuse** d'ajouter ou de supprimer une ligne sur un bon **Reçu** ou **Facturé** (« Impossible de modifier les lignes d'un bon reçu ou facturé »). Corrigez les lignes **avant** de recevoir le bon.

### 3.4 Modifier les dates d'un bon

- **Depuis la liste** : cliquer la cellule **Date Commande** ou **Livraison Prévue**, choisir la date, puis quitter le champ. Laisser le champ vide efface la date.
- **Depuis le détail** : modifier les champs date de la section **Informations**, puis **Sauvegarder**.

### 3.5 Changer le statut d'un bon

1. Ouvrir le bon → menu déroulant de **statut** dans l'en-tête du panneau.
2. Choisir le nouveau statut (Brouillon, Envoyé, Confirmé, En cours, Reçu, Facturé, Annulé).
3. **Sauvegarder**.

> **Attention à « Reçu ».** Choisir « Reçu » puis Sauvegarder **déclenche la réception en stock** (voir §3.7) et devient **définitif**. Les statuts **Confirmé** et **En cours** sont purement informatifs (aucun effet automatique). Le statut **Facturé** se pose normalement en créant une facture (§3.10), pas à la main.

### 3.6 Envoyer un bon au fournisseur

1. Ouvrir le bon → **Envoyer au fournisseur**.
2. Saisir l'adresse **courriel** du fournisseur, puis envoyer.
3. Le système crée un **lien public** (90 jours), envoie le courriel et, si le bon était en **Brouillon**, le passe à **Envoyé**. Copiez le lien au besoin pour le partager autrement.

### 3.7 Recevoir un bon (alimenter le stock)

1. Ouvrir le bon → dans l'en-tête, choisir le statut **Reçu** → **Sauvegarder**.
2. Le système, **automatiquement et de façon atomique** :
   - regroupe les lignes **par produit** (pour ne pas compter deux fois) ;
   - **augmente le stock** de chaque produit lié de la quantité reçue ;
   - crée un **mouvement d'entrée** par produit, avec la référence « Réception BC {numéro} » et un **coût unitaire** = moyenne pondérée des lignes de ce produit dans le bon.
3. Le bon affiche désormais **Reçu**, et les mouvements apparaissent dans l'onglet **Mouvements**.

> **Deux garanties.** La réception est **sérialisée** : si deux personnes reçoivent le même bon en même temps, le stock n'est **jamais** doublé. Et elle est **tout ou rien** : si l'inscription du mouvement échoue, le stock n'est pas augmenté (aucune divergence stock / audit possible). Un plafond protège contre un coût aberrant (au-delà d'environ 10 milliards, la réception est refusée avec un message clair plutôt qu'une erreur technique).
>
> **Le coût de revient du produit ne change pas.** La réception inscrit un coût **sur le mouvement** (pour l'audit et la comptabilité) et met à jour **la quantité** en stock, mais elle **n'écrase pas** le coût de revient saisi sur la fiche produit. Pour tenir un coût de revient à jour, corrigez-le à la main sur la fiche (Module 10).

### 3.8 Corriger une réception faite par erreur

Un bon **Reçu** ne peut pas être ramené en arrière : si vous tentez de le repasser à un autre statut, le système renvoie une erreur (« La réception d'un bon de commande est définitive... »). Pour rectifier :

1. Onglet **Mouvements** → repérer les mouvements d'entrée « Réception BC {numéro} ».
2. Ouvrir chaque mouvement → **Annuler ce mouvement**.
3. Le système crée un **contre-mouvement** de sortie qui ramène le stock à son niveau antérieur.

> Le bon **reste** au statut « Reçu » (pour l'audit) ; c'est l'**inventaire** que l'on corrige, par les contre-mouvements. Voir le Module 10 pour l'annulation des mouvements.

### 3.9 Exporter un bon (HTML, Excel, CSV QuickBooks)

Depuis le panneau de détail, première rangée d'actions :

- **Générer HTML** : télécharge un fichier `.html` imprimable (ouvrez-le, puis Ctrl+P pour imprimer ou enregistrer en PDF).
- **Aperçu** : affiche le même document dans une fenêtre intégrée, avec « Ouvrir dans un nouvel onglet ».
- **Excel (.xlsx)** : télécharge le bon en tableur.
- **CSV QuickBooks** : télécharge un CSV compatible QuickBooks (généré dans votre navigateur). **Copier CSV** met ce même contenu dans le presse-papiers.

### 3.10 Créer une facture fournisseur à partir d'un bon

1. Ouvrir le bon → **Créer facture** → confirmation.
2. Une **facture fournisseur en brouillon** est créée dans la Comptabilité, reprenant les lignes et les taxes du bon.
3. Le bon passe au statut **Facturé**, et le bouton « Créer facture » devient inactif (« Déjà facturé »).

> Le serveur refuse de facturer deux fois le même bon (bon déjà facturé ou déjà comptabilisé), et refuse un bon sans ligne ni fournisseur. La facture créée reste **en brouillon** : elle se complète et se comptabilise dans le **Module 15 — Comptabilité**.

### 3.11 Supprimer un bon de commande

1. Depuis la liste (poubelle sur la ligne) **ou** depuis le détail (**Supprimer**) → confirmation « Supprimer ce bon de commande ? ».
2. Le bon et ses lignes sont effacés ; les dépenses et les mouvements de stock qui le référençaient sont **détachés** (ils subsistent, sans le lien).

> **La suppression est refusée** si le bon est **Reçu**, **Facturé**, ou déjà **comptabilisé** (une écriture au grand livre lui est liée). Dans ce dernier cas, il faut d'abord **contre-passer l'écriture comptable** (Module 15). Autrement dit, un bon qui a eu un effet sur le stock ou sur la comptabilité ne se supprime pas d'un clic — c'est voulu, pour préserver la traçabilité.

### 3.12 Octroyer une demande de prix (génère des bons)

1. Onglet **Demandes de prix** → ouvrir la demande → recueillir les offres (voir Module 10).
2. **Voir le comparatif** → cocher **Retenir** le(s) fournisseur(s) gagnant(s).
3. **Octroyer → générer BC** → confirmation.
4. Le système crée **un bon de commande (Brouillon) par fournisseur retenu**, avec les seules lignes chiffrées, et affiche les numéros générés.
5. Retrouvez ces bons dans l'onglet **Commandes** pour les compléter, les envoyer et les recevoir normalement.

---

## 4. Référence

### 4.1 Points d'accès (API)

Tous préfixés par `/api/erp/v1`. « Écriture achats » = garde `require_purchase_write` (admin, super-admin, gestionnaire, magasinier, utilisateur, ou administrateur).

**Bons de commande** (`suppliers.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/suppliers/purchase-orders` | Liste globale (pagination, filtres statut / projet) | lecture |
| GET `/suppliers/{id}/orders` | Bons d'un fournisseur | lecture |
| POST `/suppliers/{id}/orders` | Créer un bon | écriture achats |
| POST `/suppliers/orders/ai/scan` | Scanner un BC (IA Vision, lecture seule) | écriture achats |
| PUT `/suppliers/purchase-orders/{id}` | Modifier (dates / statut / employé / projet / adresse / conditions / notes / taxes) ; **déclenche la réception si le statut passe à Reçu** | écriture achats |
| PUT `/suppliers/purchase-orders/{id}/dates` | Modifier les dates (édition en ligne, glisser dans le Gantt) | écriture achats |
| PUT `/suppliers/purchase-orders/{id}/status` | Modifier le statut (même effet de réception que le PUT unifié) | écriture achats |
| GET `/suppliers/orders/{id}/lines` | Lister les lignes | lecture |
| POST `/suppliers/orders/{id}/lines` | Ajouter une ligne (refus si Reçu / Facturé) | écriture achats |
| DELETE `/suppliers/orders/{id}/lines/{lid}` | Supprimer une ligne (refus si Reçu / Facturé) | écriture achats |
| DELETE `/suppliers/purchase-orders/{id}` | Supprimer le bon (refus si Reçu / Facturé / comptabilisé) | écriture achats |
| POST `/suppliers/orders/{id}/generate-html` | HTML imprimable thématisé | lecture |
| POST `/suppliers/orders/{id}/send` | Envoyer par courriel + lien public (passage Brouillon → Envoyé) | écriture achats |
| GET `/suppliers/orders/public/{token}` | Vue publique en lecture seule (**sans authentification**) | jeton |
| GET `/suppliers/orders/{id}/export-xlsx` | Export Excel | lecture |

> **Note.** Le panneau de détail change le statut (dont la réception) par le bouton **Sauvegarder**, qui appelle le **PUT unifié** `/purchase-orders/{id}`. Un point d'accès distinct `/purchase-orders/{id}/status` existe et produit le même effet de réception ; les deux chemins sont alignés.

**Octroi d'une demande de prix** (`rfq.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| POST `/rfq/demandes/{id}/octroi` | Retient des fournisseurs → **génère un bon de commande par fournisseur retenu** (atomique) | écriture demandes de prix |

**Facturation** (`accounting.py`)

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| POST `/accounting/invoices/from-bon-commande/{id}` | Crée une facture fournisseur brouillon depuis le bon et le passe à **Facturé** | utilisateur authentifié |

### 4.2 Statuts et couleurs

| Statut | Couleur | Posé par |
|--------|---------|----------|
| **Brouillon** | gris | Création (manuelle, scan ou octroi) |
| **Envoyé** | indigo | Envoi par courriel (depuis Brouillon uniquement) — ou à la main |
| **Confirmé** | bleu | À la main (aucune automatisation) |
| **En cours** | violet | À la main (aucune automatisation) |
| **Reçu** | vert | À la main → **déclenche la réception en stock** (définitif) |
| **Facturé** | sarcelle | Création d'une facture fournisseur (bouton « Créer facture ») |
| **Annulé** | rouge | À la main |

### 4.3 Numérotation

Format `BC-00012` (préfixe `BC-` + identifiant sur cinq chiffres). Le numéro est attribué de façon **sûre face à la concurrence** : le bon est inséré avec un numéro temporaire, puis renuméroté d'après son identifiant. Jamais de `MAX + 1`. Pas de préfixe d'année (numérotation en séquence globale).

### 4.4 Calculs

| Élément | Formule | Déclenché par |
|---------|---------|---------------|
| **Montant d'une ligne** | `arrondi(quantité × prix unitaire, 2)` | Ajout d'une ligne |
| **Sous-total (HT)** | `Σ des montants de lignes` | Ajout / suppression d'une ligne, octroi |
| **Taxe 1 (TPS)** | `arrondi(sous-total × taux 1 ÷ 100, 2)` | Recalcul des totaux |
| **Taxe 2 (TVQ)** | `arrondi(sous-total × taux 2 ÷ 100, 2)` | Recalcul des totaux |
| **Total (taxes incluses)** | `sous-total + taxe 1 + taxe 2` | Recalcul des totaux |
| **`montant_total` persisté** | `= sous-total (HT)` | Recalcul des totaux |
| **Coût d'une réception (par produit)** | `montant total des lignes du produit ÷ quantité totale` (moyenne pondérée des lignes **de ce bon**) | Passage à Reçu |

> **Précision sur le coût.** La « moyenne pondérée » agrège seulement les lignes d'un **même bon de commande** pour un **même produit**. Elle alimente le **coût du mouvement de réception** (audit et comptabilité) mais **ne recalcule pas** le coût de revient du produit en moyenne mobile. **Il n'y a pas de coût moyen pondéré glissant** dans le système : le coût de revient d'un produit reste la valeur saisie à la main sur sa fiche (Module 10).

### 4.5 Taxes

Les deux taxes d'un bon sont **figées à la création** d'après la configuration du tenant (Canada multi-provinces ou États-Unis). Valeurs par défaut au Québec : **TPS 5 %**, **TVQ 9,975 %**. Les taux sont bornés entre 0 et 100, et les libellés à 50 caractères. Un libellé de taxe **vide et légitime** (exonération) est préservé. Modifier un taux dans le panneau de détail **recalcule** les totaux persistés. Le rendu HTML, l'export Excel et le courriel recalculent les taxes à l'affichage (avec repli sur les valeurs par défaut du Québec pour les bons antérieurs à cette fonction).

### 4.6 Réception valorisée (effet sur le stock)

Le passage à **Reçu** (par le PUT unifié ou par le point d'accès `/status`) exécute, sous verrou et de façon atomique :

| Étape | Détail |
|-------|--------|
| Verrou | `SELECT ... FOR UPDATE` sur le bon → **sérialise** deux réceptions concurrentes (anti-double-stockage) |
| Agrégation | Les lignes sont regroupées **par produit** (`produit_id` non nul) |
| Coût | Coût unitaire = montant total ÷ quantité totale du produit dans ce bon |
| Stock | `stock_disponible += quantité` (sous verrou sur le produit) |
| Mouvement | Un mouvement **ENTREE** par produit : référence « Réception BC {numéro} », type « BON_RECEPTION », coût unitaire et coût total, lien vers le bon |
| Garde-fou | Coût total supérieur à ≈ 10 milliards → refus (message clair, pas d'erreur technique) |
| Atomicité | Stock **et** mouvement, tout ou rien ; si l'un échoue, rien n'est appliqué |
| Idempotence | Sauvegarder de nouveau « Reçu » sur un bon déjà reçu **ne remet pas en stock une seconde fois** |

Seules les lignes **liées à un produit** alimentent le stock ; les lignes libres (hors inventaire) n'ont aucun effet matériel.

### 4.7 Règles de verrouillage

| Règle | Effet |
|-------|-------|
| Dévalider un bon déjà **Reçu** (le repasser à un autre statut) | Refusé (HTTP 409) — annuler les mouvements de stock à la place |
| Ajouter / supprimer une ligne sur un bon **Reçu** ou **Facturé** | Refusé (HTTP 400) |
| Supprimer un bon **Reçu** ou **Facturé** | Refusé (HTTP 400) |
| Supprimer un bon **déjà comptabilisé** (écriture au grand livre liée) | Refusé (HTTP 400) — contre-passer l'écriture d'abord |
| Créer un bon sans fournisseur | Refusé (fournisseur obligatoire) |
| Ligne sans quantité positive, ou sans produit ni description | Refusée |
| Octroyer une demande de prix sans aucune ligne chiffrée | Refusé (pas de bon à 0 $) |
| Fichier de scan supérieur à 20 Mo | Refusé |
| Montants hors bornes (dépassement numérique) | Refusés proprement (message d'erreur) |

### 4.8 Effet sur l'argent

Deux effets, tous deux importants à connaître :

**1. Comptabilisation automatique au grand livre.** Une **synchronisation comptable** (déclenchée depuis la Comptabilité) inscrit automatiquement une **écriture d'achat** pour tout bon dont le statut n'est **ni Annulé ni Brouillon** (donc Envoyé, Confirmé, En cours, Reçu ou Facturé), dont le total est supérieur à zéro et qui n'a pas encore d'écriture. L'écriture débite le compte de charge et les taxes à recevoir (TPS / TVQ), et crédite les comptes fournisseurs pour le total taxes incluses. **Conséquence pratique** : dès qu'un bon est **transmis** (au-delà du brouillon) avec un montant, il peut être **comptabilisé à la prochaine synchronisation** — même si vous n'avez pas encore créé de facture. C'est un effet monétaire qui n'est pas visible depuis le seul onglet Commandes.

**2. Crédits IA prépayés.** Le **scan IA** d'un bon consomme des crédits IA du tenant. Le coût facturé est le **coût réel des jetons** (environ 0,003 $ par millier de jetons en entrée, 0,015 $ par millier en sortie) **majoré de 30 %**. Si les crédits sont épuisés, le scan est refusé. C'est le seul appel externe payant du cycle des bons de commande.

> Le module **ne facture rien** directement via Stripe ou QuickBooks. Le bouton « Créer facture » crée une facture **en brouillon** ; c'est la Comptabilité (Module 15) qui la comptabilise et, le cas échéant, l'exporte vers QuickBooks.

### 4.9 Validations et limites (bornes de saisie)

| Règle | Effet |
|-------|-------|
| Description de ligne au-delà de 2 000 caractères | Refusée |
| Quantité de ligne hors bornes (doit être supérieure à 0, plafond très élevé) | Refusée |
| Prix unitaire négatif | Refusé |
| Montant d'une ligne (quantité × prix) trop élevé | Refusé (protection contre le dépassement numérique) |
| Sous-total ou total agrégé trop élevé | Refusé |
| Notes au-delà de 10 000 caractères ; adresse / conditions au-delà de 2 000 | Refusées |
| Taux de taxe hors [0 ; 100] ; libellé de taxe au-delà de 50 caractères | Refusés |
| Dates vides | Interprétées comme « aucune date » |

### 4.10 Tables PostgreSQL (par tenant)

| Table | Rôle |
|-------|------|
| `bons_commande` | En-tête du bon : numéro, fournisseur, projet, employé, dates, statut, adresse / conditions de livraison, taxes figées, sous-total / TPS / TVQ / total, notes, jeton public, liens facture et écriture comptable |
| `bon_commande_lignes` | Lignes d'articles (produit lié, description, quantité, unité, prix, montant). **Table partagée avec les demandes de prix** : des colonnes de rôle distinguent une ligne de bon, une ligne de demande et une ligne de réponse fournisseur |
| `mouvements_stock` | Mouvements de réception (type ENTREE, référence « BON_RECEPTION », coût, lien vers le bon) |
| `produits` | `stock_disponible` incrémenté à la réception |
| `dossier_achats` | Rattachement du bon au dossier d'opportunité (quand le projet est lié à une opportunité disposant d'un dossier) |
| `fournisseurs` / `companies` | Nom du fournisseur (par jointure) |
| `public.bc_public_tokens` | Jetons des liens publics (90 jours), partagés entre tenants |

> La table `bons_commande` (comme les autres tables « formulaires » du produit) n'est pas définie mot pour mot par le code : elle est provisionnée par copie depuis un tenant de référence, et le code **garantit défensivement** ses colonnes étendues (employé, livraison, taxes, totaux) au moment de l'exécution, pour rester compatible avec d'anciens tenants.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|--------|------|
| **Magasin / Inventaire** (Module 10) | Les bons de commande **sont** l'onglet « Commandes » du Magasin. La réception d'un bon alimente le stock (mouvement d'entrée) ; la correction d'une réception passe par l'annulation des mouvements ; les articles proviennent du catalogue de produits ; les fournisseurs sont ceux de l'onglet « Fournisseur ». |
| **Demandes de prix** (Module 10) | L'**octroi** d'une demande de prix génère un bon de commande par fournisseur retenu ; ces bons alimentent l'onglet Commandes. |
| **Comptabilité** (Module 15) | Le bouton **Créer facture** génère une facture fournisseur brouillon ; par ailleurs, un bon transmis (au-delà du brouillon) avec un montant est **comptabilisé automatiquement** au grand livre à la synchronisation. La réception, via le mouvement de stock, alimente aussi la comptabilité d'inventaire. |
| **Projets** (Module 09) | Un bon peut être associé à un projet (colonne, filtre, tri). |
| **Dossiers / Fiche 360** (Module 07) | Un bon dont le projet est lié à une opportunité avec dossier est **rattaché au dossier** ; un bon s'ouvre directement depuis un dossier via `/magasin?open=<id>`. |
| **Employés** (Module 11) | L'employé responsable d'un bon est choisi parmi les employés actifs. |
| **Configuration** (Module 28) | Les taux de taxes (TPS / TVQ, provinces, États-Unis), le logo et le thème des documents proviennent de la Configuration de l'entreprise. |
| **Crédits IA** (Module 25) | Le scan IA d'un bon consomme les crédits prépayés du tenant. |

### 5.2 FAQ

**Où est le menu « Bons de commande » ?**
Il n'existe pas. Les bons de commande sont l'onglet **« Commandes »** du menu **Magasin** (`/magasin`), qui est l'onglet ouvert par défaut. L'onglet des fiches fournisseurs s'appelle « Fournisseur » (au singulier).

**La réception d'un bon met-elle le stock à jour automatiquement ?**
Oui. Passer un bon à **Reçu** (puis Sauvegarder) augmente le stock des produits liés et crée un mouvement d'entrée par produit. C'est atomique et sérialisé (jamais de double-comptage).

**Puis-je recevoir seulement une partie d'un bon ?**
Non. La réception est **totale** : tout le bon est reçu d'un coup. Il n'y a pas de quantité reçue par ligne ni de réceptions échelonnées. Si vous ne recevez qu'une partie, ajustez les lignes **avant** de recevoir, ou faites la réception à la livraison finale et corrigez le stock par des mouvements manuels.

**J'ai reçu un bon par erreur, comment revenir en arrière ?**
On ne peut pas dévalider un bon « Reçu » (le système renvoie une erreur pour protéger l'inventaire). Corrigez l'**inventaire** en **annulant les mouvements de stock** générés (onglet Mouvements → « Annuler ce mouvement »), ce qui crée des contre-mouvements. Le bon reste « Reçu » pour l'audit.

**La réception met-elle à jour le coût de revient de mes produits ?**
Non. Elle inscrit un coût **sur le mouvement** (audit et comptabilité) et met à jour la **quantité** en stock, mais elle **ne touche pas** au coût de revient de la fiche produit. Il n'y a pas de coût moyen glissant ; le coût de revient reste saisi à la main.

**Pourquoi ne puis-je plus modifier les lignes d'un bon ?**
Parce qu'il est **Reçu** ou **Facturé**. Les lignes se verrouillent alors (modifier le sous-total après une réception fausserait le stock et la comptabilité). Corrigez les lignes avant de recevoir ou de facturer.

**Pourquoi la suppression est-elle refusée ?**
Un bon **Reçu**, **Facturé** ou déjà **comptabilisé** ne se supprime pas (traçabilité). Pour un bon comptabilisé, contre-passez d'abord son écriture au grand livre (Module 15). Un simple brouillon, lui, se supprime sans problème.

**L'envoi par courriel change-t-il le statut ?**
Oui, mais **seulement** de Brouillon à Envoyé. Un bon déjà Confirmé, En cours, Reçu, etc. conserve son statut à l'envoi.

**À quoi servent les statuts « Confirmé » et « En cours » ?**
À votre suivi. Ils sont valides et affichés (bleu et violet), mais **aucune action automatique** ne les déclenche : vous les posez à la main dans le menu de statut.

**Le fournisseur a-t-il besoin d'un compte pour voir le bon ?**
Non. L'envoi crée un **lien public** valable 90 jours, qui affiche le bon en lecture seule (avec ses prix, ses totaux et ses notes) sans authentification. Le lien devient inaccessible si le bon est annulé ou après expiration.

**Le scan IA crée-t-il directement un bon ?**
Non. Il **extrait** les lignes dans la fenêtre de création ; vous vérifiez, corrigez, puis **créez** le bon. Le scan consomme des crédits IA.

**Comment sont générés les bons issus d'une demande de prix ?**
Par l'**octroi** : dans la demande de prix, on retient des fournisseurs, puis « Octroyer → générer BC ». Le système crée un bon (Brouillon) par fournisseur retenu, avec les seules lignes chiffrées. L'octroi exige un rôle gestionnaire ou magasinier (un simple utilisateur ne peut pas octroyer, même s'il peut créer des bons à la main).

**Un bon transmis est-il comptabilisé sans que je crée de facture ?**
Oui, potentiellement. La synchronisation comptable inscrit une écriture d'achat pour tout bon **au-delà du brouillon** avec un montant. Créer une facture n'est donc pas la seule voie vers la comptabilité ; c'est une conséquence à connaître.

**Puis-je changer le fournisseur d'un bon ?**
Oui, tant que le bon n'est pas verrouillé (Reçu / Facturé) : le nom du fournisseur est alors mis à jour. Mais en pratique, mieux vaut créer un nouveau bon pour un autre fournisseur.

### 5.3 Ce qui n'existe pas (limites connues)

- Pas d'entrée de menu « Bons de commande » ni « Achats » (c'est l'onglet « Commandes » du Magasin).
- Pas de réception partielle ni échelonnée (réception totale et unique).
- Pas de « dé-réception » : une réception se corrige en annulant les mouvements de stock.
- Pas de boutons de flux de travail (Confirmer, En cours…) : statut à la main, sauf l'envoi (Brouillon → Envoyé) et la facturation (→ Facturé).
- Pas de coût moyen pondéré glissant : le coût de revient du produit n'est pas recalculé à la réception.
- Pas plus de deux taxes, et elles sont figées à la création du bon.
- Pas de suppression d'un bon reçu, facturé ou comptabilisé.
- Pas de duplication de bon (recréer manuellement).
- Le CSV QuickBooks est un export simplifié, généré côté navigateur.

---

## 6. Récapitulatif

- Les **bons de commande** sont l'onglet **« Commandes »** du menu **Magasin** (`/magasin`, section Opérations), onglet ouvert par défaut. **Il n'y a pas de menu « Bons de commande ».** L'onglet des fournisseurs s'appelle « Fournisseur » (au singulier).
- **Créer un bon** : à la main (bouton « Commande fournisseur »), par **scan IA** (image ou PDF, lecture seule, crédits IA) ou par **octroi** d'une demande de prix (un bon par fournisseur retenu).
- **Numéro** `BC-00012` (cinq chiffres, sûr face à la concurrence). **Fournisseur obligatoire.**
- **Sept statuts** : Brouillon → Envoyé → Confirmé → En cours → **Reçu** → Facturé, plus Annulé. Statuts posés à la main, **sauf** l'envoi (Brouillon → Envoyé) et la facturation (→ Facturé).
- **Panneau de détail** riche : informations, livraison, articles (avec sélection depuis l'inventaire), totaux (sous-total HT + deux taxes figées + total taxes incluses), notes, et actions (HTML, Aperçu, Envoyer, CSV QuickBooks, Excel, Créer facture, Supprimer, Sauvegarder). **Sauvegarde ciblée** : seuls les champs modifiés partent au serveur.
- **Envoi au fournisseur** : courriel thématisé + **lien public 90 jours** (consultable sans compte) ; passe le bon à Envoyé s'il était en Brouillon.
- **Réception** (statut « Reçu » + Sauvegarder) : **augmente le stock** et crée un mouvement d'entrée valorisé par produit, de façon **atomique** et **sérialisée**. **Définitive** : on corrige une erreur en annulant les mouvements, pas en dévalidant le bon. Le **coût de revient du produit n'est pas** recalculé.
- **Verrous** : lignes non modifiables après Reçu / Facturé ; suppression refusée pour un bon Reçu, Facturé ou comptabilisé.
- **Facturation** : « Créer facture » génère une facture fournisseur **brouillon** et passe le bon à Facturé. Par ailleurs, un bon **transmis avec un montant** est **comptabilisé automatiquement** au grand livre à la synchronisation.
- **Permissions** : écriture des bons pour admin / super-admin / gestionnaire / magasinier / **utilisateur** ; **l'octroi** d'une demande de prix exige gestionnaire ou magasinier (pas le simple utilisateur) ; employé et comptable sont en lecture seule.
- **Effet argent** : comptabilisation automatique au grand livre (dès qu'un bon est transmis avec un montant) + crédits IA du scan (coût réel des jetons majoré de 30 %). Aucun paiement Stripe direct.

---

**Documentation générée à partir du code source** : `MagasinPage.tsx` (onglet Commandes), `components/rfq/RfqTab.tsx` (octroi), `api/suppliers.ts`, `api/rfq.ts`, `api/accounting.ts` ; `backend/routers/suppliers.py`, `rfq.py`, `accounting.py`, `inventory.py`.

**Manuels liés** :
- Module 10 — Magasin (produits, inventaire, mouvements, demandes de prix, fournisseurs) — `10-operations-magasin.md`
- Module 12 — Bons de Travail (sorties de matériel vers le chantier) — `12-operations-bons-de-travail.md`
- Module 15 — Comptabilité (factures fournisseurs, écritures au grand livre) — `15-operations-comptabilite.md`
- Module 09 — Projets (association des bons à un projet) — `09-ventes-projets.md`
- Module 07 — Dossiers / Fiche 360 (ouverture directe d'un bon) — `07-ventes-dossiers.md`
- Module 28 — Configuration (taxes, thème des documents) — `28-configuration.md`
- Module 25 — Assistant IA (crédits IA du scan) — `25-communication-assistant-ia.md`
