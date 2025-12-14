# 🏪 Achats

## Introduction

Le module **Achats** gère l'ensemble de vos approvisionnements : fournisseurs, demandes de prix, bons d'achat, réceptions et suivi des paiements. Il optimise votre chaîne d'approvisionnement pour garantir que vos chantiers disposent des matériaux nécessaires au bon moment.

Ce module inclut 20 catégories de produits construction, 10 certifications reconnues au Québec, et génère automatiquement les documents d'achat (Demandes de prix, Bons d'achat).

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🏪 Achats"**
2. Le tableau de bord achats s'affiche avec les onglets :
   - **Fournisseurs** : Gestion du répertoire
   - **Demandes de Prix** : Comparaison des offres
   - **Bons d'Achat** : Commandes confirmées
   - **Statistiques** : Analyse des achats
3. Gérez vos fournisseurs et commandes

---

## Structure des Données

### Table PostgreSQL : `formulaires`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `type_formulaire` | TEXT | DEMANDE_PRIX ou BON_ACHAT |
| `numero_document` | TEXT | Référence unique (DP-YYYY-XXXX) |
| `company_id` | INTEGER | Lien vers fournisseur |
| `employee_id` | INTEGER | Créateur du document |
| `statut` | TEXT | État du formulaire |
| `priorite` | TEXT | NORMAL, URGENT |
| `date_echeance` | DATE | Date limite |
| `montant_total` | REAL | Total calculé |
| `notes` | TEXT | Notes libres |
| `metadonnees_json` | TEXT | Données additionnelles JSON |
| `created_at` | TIMESTAMP | Date de création |

### Table : `formulaire_lignes`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `formulaire_id` | INTEGER | Lien vers formulaire |
| `sequence_ligne` | INTEGER | Ordre de la ligne |
| `description` | TEXT | Description du produit |
| `code_article` | TEXT | Référence article |
| `quantite` | REAL | Quantité commandée |
| `unite` | TEXT | Unité de mesure |
| `prix_unitaire` | REAL | Prix par unité |
| `montant_ligne` | REAL | Total ligne |

---

## Types de Documents

### Demande de Prix (DP)

| Élément | Description |
|---------|-------------|
| **Objectif** | Solliciter des cotations de fournisseurs |
| **Numérotation** | DP-YYYY-XXXX (ex: DP-2025-0042) |
| **Usage** | Comparaison avant achat |
| **Export** | HTML/PDF pour envoi |

### Bon d'Achat (BA)

| Élément | Description |
|---------|-------------|
| **Objectif** | Commander officiellement les matériaux |
| **Numérotation** | BA-YYYY-XXXX (ex: BA-2025-0128) |
| **Usage** | Engagement d'achat ferme |
| **Export** | HTML/PDF pour fournisseur |

---

## Statuts des Formulaires

| Statut | Description | Couleur |
|--------|-------------|---------|
| **BROUILLON** | En préparation, modifiable | ⚪ Gris |
| **ENVOYÉ** | Transmis au fournisseur | 🔵 Bleu |
| **CONFIRMÉ** | Accusé de réception fournisseur | 🟢 Vert |
| **EN_LIVRAISON** | Expédié par le fournisseur | 🟠 Orange |
| **REÇU** | Livraison complète | ✅ Vert foncé |
| **PARTIEL** | Livraison partielle | 🟡 Jaune |
| **ANNULÉ** | Commande annulée | 🔴 Rouge |

---

## Les 20 Catégories de Fournisseurs Construction

| # | Catégorie | Exemples de produits |
|---|-----------|----------------------|
| 1 | **Béton et ciment** | Ciment Portland, béton prémélangé |
| 2 | **Acier et métaux** | Poutrelles, barres d'armature |
| 3 | **Bois et charpente** | Madriers, colombages, poutrelles |
| 4 | **Isolation et étanchéité** | Laine, mousse, pare-vapeur |
| 5 | **Plomberie et tuyauterie** | Tuyaux, raccords, valves |
| 6 | **Électricité et câblage** | Fils, câbles, panneaux |
| 7 | **Toiture et bardeau** | Bardeaux, membranes, solins |
| 8 | **Portes et fenêtres** | Portes, fenêtres, cadres |
| 9 | **Revêtements extérieurs** | Vinyle, brique, stucco |
| 10 | **Gypse et plâtre** | Panneaux, composé à joints |
| 11 | **Peinture et finition** | Peintures, teintures, vernis |
| 12 | **Quincaillerie et fixations** | Vis, clous, boulons |
| 13 | **Équipements de chantier** | Bétonnières, échafaudages |
| 14 | **Sécurité et EPI** | Casques, gants, harnais |
| 15 | **Outillage spécialisé** | Outils de construction |
| 16 | **Location d'équipements** | Grues, excavateurs |
| 17 | **Services de béton** | Pompage, finition |
| 18 | **Transport et livraison** | Camionnage, grutage |
| 19 | **Agrégats et remblai** | Sable, gravier, terre |
| 20 | **Produits d'excavation** | Remblayage, compactage |

---

## Certifications Reconnues au Québec

| # | Certification | Signification |
|---|--------------|---------------|
| 1 | **RBQ** | Régie du bâtiment du Québec |
| 2 | **CCQ** | Commission de la construction du Québec |
| 3 | **CNESST** | Santé et sécurité du travail |
| 4 | **ISO 9001:2015** | Système de gestion qualité |
| 5 | **BNQ** | Bureau de normalisation du Québec |
| 6 | **CSA** | Canadian Standards Association |
| 7 | **LEED** | Leadership in Energy and Environmental Design |
| 8 | **Garantie GCR** | Garantie construction résidentielle |
| 9 | **ACQ** | Association de la construction du Québec |
| 10 | **APCHQ** | Association des professionnels de la construction et de l'habitation |

---

## Fonctionnalités Principales

### 1. Gestion des Fournisseurs

| Information | Description |
|-------------|-------------|
| **Raison sociale** | Nom légal du fournisseur |
| **Coordonnées** | Adresse, téléphone, courriel |
| **Contact** | Représentant commercial |
| **Conditions** | Délais paiement, remises |
| **Catégories** | Types de produits fournis (20 catégories) |
| **Certifications** | RBQ, CCQ, CNESST, etc. |
| **Évaluation** | Note qualité/service (1-5 étoiles) |

### 2. Fiche Fournisseur Complète

| Section | Informations |
|---------|--------------|
| **Identification** | Nom, numéro entreprise Québec (NEQ) |
| **Coordonnées** | Adresse, téléphone, fax, courriel, site web |
| **Contact principal** | Nom, titre, téléphone direct, cellulaire |
| **Conditions commerciales** | Délai paiement, remise volume, minimum commande |
| **Certifications** | Liste des certifications avec dates d'expiration |
| **Statistiques** | Nombre de commandes, montant total, dernière commande |

### 3. Demandes de Prix (DP)

Un formulaire de demande de prix comprend :
- Numéro unique généré automatiquement (DP-YYYY-XXXX)
- Fournisseur destinataire
- Lignes de produits avec quantités souhaitées
- Date limite de réponse
- Notes et spécifications techniques
- Export HTML/PDF pour envoi par courriel

### 4. Bons d'Achat (BA)

Un bon d'achat comprend :
- Numéro unique généré automatiquement (BA-YYYY-XXXX)
- Fournisseur sélectionné
- Lignes de produits avec quantités et prix négociés
- Date de livraison souhaitée
- Projet destinataire (optionnel)
- Conditions de paiement
- Export HTML/PDF officiel

### 5. Réception des Marchandises

- Vérification des quantités reçues vs commandées
- Contrôle qualité à la réception
- Gestion des écarts et litiges
- Mise à jour automatique de l'inventaire
- Historique des réceptions

---

## Guide Pas-à-Pas

### Ajouter un fournisseur

1. Allez dans **Achats** > **"Fournisseurs"**
2. Cliquez sur **"➕ Nouveau fournisseur"**
3. Remplissez les informations de base :

   **Section Identification :**
   - Raison sociale
   - Numéro d'entreprise du Québec (NEQ)

   **Section Coordonnées :**
   - Adresse complète
   - Téléphone, Fax, Courriel
   - Site web

   **Section Contact :**
   - Nom du représentant
   - Titre/Fonction
   - Téléphone direct et cellulaire

   **Section Commerciale :**
   - Catégorie de produits (parmi les 20)
   - Délai de paiement (Net 30, etc.)
   - Remise éventuelle
   - Commande minimum

   **Section Certifications :**
   - Sélectionnez les certifications détenues
   - Entrez les dates d'expiration

4. Cliquez sur **"💾 Enregistrer"**
5. Le fournisseur apparaît dans le répertoire

### Créer une demande de prix

1. Allez dans **Achats** > **"Demandes de Prix"**
2. Cliquez sur **"➕ Nouvelle demande"**
3. Sélectionnez le fournisseur
4. Le numéro DP-YYYY-XXXX est généré automatiquement
5. Ajoutez les lignes de produits :
   - Description du produit
   - Code article (optionnel)
   - Quantité souhaitée
   - Unité de mesure
6. Définissez la date limite de réponse
7. Ajoutez des notes techniques si nécessaire
8. Cliquez sur **"💾 Enregistrer"** (brouillon)
9. Pour envoyer : **"📤 Exporter HTML"** et envoyez par courriel

### Créer un bon d'achat

1. Allez dans **Achats** > **"Bons d'Achat"**
2. Cliquez sur **"➕ Nouveau bon d'achat"**
3. Sélectionnez le fournisseur
4. Le numéro BA-YYYY-XXXX est généré automatiquement
5. Associez au projet (optionnel)
6. Ajoutez les lignes de commande :
   - Sélectionnez un produit du catalogue ou entrez manuellement
   - Quantité commandée
   - Prix unitaire négocié
   - Le total ligne se calcule automatiquement
7. Définissez la date de livraison souhaitée
8. Sélectionnez la priorité (Normal/Urgent)
9. Ajoutez des notes si nécessaire
10. Cliquez sur **"💾 Enregistrer"**
11. Pour envoyer : **"📤 Exporter HTML"** et envoyez au fournisseur

### Envoyer un document au fournisseur

1. Ouvrez le document (DP ou BA)
2. Vérifiez les informations
3. Cliquez sur **"📤 Exporter HTML"**
4. Le document HTML s'ouvre ou se télécharge
5. Envoyez par courriel au fournisseur
6. Changez le statut à "ENVOYÉ"

### Réceptionner une livraison

1. Allez dans **Achats** > **"Bons d'Achat"**
2. Trouvez le BA concerné
3. Cliquez sur **"📦 Réceptionner"**
4. Pour chaque ligne :
   - Entrez la quantité reçue
   - Vérifiez la qualité
   - Notez les écarts
5. Si livraison partielle → statut "PARTIEL"
6. Si complète → statut "REÇU"
7. L'inventaire est mis à jour automatiquement

### Gérer un litige fournisseur

1. Lors de la réception, notez le problème
2. Cliquez sur **"⚠️ Signaler un écart"**
3. Décrivez le problème :
   - Quantité manquante
   - Produit endommagé
   - Produit incorrect
   - Délai non respecté
4. Joignez des photos si possible
5. Le litige est enregistré dans l'historique
6. Contactez le fournisseur pour résolution

---

## Numérotation Automatique

Le système génère automatiquement les numéros de documents :

```
Format : PRÉFIXE-ANNÉE-SÉQUENCE

Demande de Prix : DP-2025-0001, DP-2025-0002, ...
Bon d'Achat     : BA-2025-0001, BA-2025-0002, ...

La séquence repart à 0001 chaque année.
```

---

## Calcul des Totaux

```
Montant ligne = Quantité × Prix unitaire
Sous-total = Σ Montants lignes
TPS (5%) = Sous-total × 0.05
TVQ (9.975%) = Sous-total × 0.09975
Total = Sous-total + TPS + TVQ
```

---

## Rapports Disponibles

| Rapport | Description |
|---------|-------------|
| **Achats par période** | Volume d'achats mensuel/annuel |
| **Par fournisseur** | Dépenses par fournisseur avec classement |
| **Par projet** | Coût matériaux par projet |
| **Par catégorie** | Répartition par type de produit |
| **Délais livraison** | Performance fournisseurs |
| **Évaluation fournisseurs** | Classement qualité/service |

---

## Système de Cache

| Donnée | TTL | Description |
|--------|-----|-------------|
| Liste fournisseurs | 5 min | Cache local des fournisseurs |
| Formulaires | 5 min | Demandes de prix et bons d'achat |
| Statistiques | 2 min | Données calculées |

---

## Astuces et Bonnes Pratiques

- **Comparez les prix** : Envoyez des demandes de prix à plusieurs fournisseurs
- **Groupez les commandes** : Réduisez les frais de livraison et obtenez des remises volume
- **Vérifiez à la réception** : Contrôlez immédiatement les livraisons et signalez les écarts
- **Évaluez vos fournisseurs** : Notez la qualité et le service après chaque commande
- **Vérifiez les certifications** : Assurez-vous que les fournisseurs ont les licences requises (RBQ, etc.)
- **Planifiez en avance** : Évitez les commandes urgentes (plus chères et plus risquées)
- **Documentez les conditions** : Conservez les ententes de prix et délais par écrit
- **Diversifiez** : Gardez plusieurs fournisseurs par catégorie pour éviter les ruptures

---

## Résolution de Problèmes

### Le fournisseur n'apparaît pas dans la liste

- **Cause** : Fournisseur inactif ou filtre actif
- **Solution** : Vérifiez le statut "actif" et désactivez les filtres

### Le numéro de document ne se génère pas

- **Cause** : Problème de connexion base de données
- **Solution** : Rafraîchissez la page et réessayez

### Les totaux ne sont pas corrects

- **Cause** : Prix unitaire ou quantité manquants
- **Solution** : Vérifiez que toutes les lignes ont des valeurs numériques

### L'export HTML n'affiche pas les données

- **Cause** : Données incomplètes dans le formulaire
- **Solution** : Enregistrez le formulaire avant d'exporter

---

## Questions Fréquentes (FAQ)

**Q: Puis-je créer un bon d'achat depuis une demande de prix ?**
R: Oui, après réception des cotations, vous pouvez convertir la meilleure offre en bon d'achat en un clic.

**Q: Comment gérer les retours fournisseurs ?**
R: Créez un avoir ou une note de crédit. Le fournisseur doit confirmer et ajuster la facturation.

**Q: Les prix fournisseurs sont-ils mis à jour automatiquement ?**
R: Non, vous devez mettre à jour manuellement ou lors de chaque négociation. Le système conserve l'historique des prix.

**Q: Puis-je commander pour plusieurs projets sur un seul BA ?**
R: Oui, utilisez des lignes séparées et attribuez chaque ligne à son projet via les notes.

**Q: Comment vérifier les certifications RBQ d'un fournisseur ?**
R: Consultez le registre RBQ en ligne ou demandez une copie de la licence au fournisseur. Entrez la date d'expiration dans sa fiche.

**Q: Puis-je importer une liste de fournisseurs ?**
R: Oui, via Configuration > Import CSV. Préparez un fichier avec les colonnes : nom, adresse, telephone, courriel, categorie.

---

## Données Techniques

### Requête Liste Fournisseurs avec Statistiques

```sql
SELECT c.*,
       (SELECT COUNT(*) FROM formulaires form
        WHERE form.company_id = c.id
        AND form.type_formulaire IN ('BON_ACHAT', 'BON_COMMANDE')) as nombre_commandes,
       (SELECT COALESCE(SUM(form.montant_total), 0) FROM formulaires form
        WHERE form.company_id = c.id
        AND form.type_formulaire IN ('BON_ACHAT', 'BON_COMMANDE')) as montant_total_commandes
FROM companies c
WHERE c.company_type = 'fournisseur'
ORDER BY c.nom
```

### Requête Formulaires par Type

```sql
SELECT f.*, c.nom as fournisseur_nom
FROM formulaires f
LEFT JOIN companies c ON f.company_id = c.id
WHERE f.type_formulaire = :type_formulaire
ORDER BY f.date_creation DESC
```

---

## Voir Aussi

- [🔧 Produits](15-produits.md) - Catalogue produits
- [📦 Inventaire](17-inventaire.md) - Gestion des stocks
- [🚚 Logistique](18-logistique.md) - Suivi des livraisons
- [💰 Comptabilité](19-comptabilite.md) - Paiements fournisseurs
