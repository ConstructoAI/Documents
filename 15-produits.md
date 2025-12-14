# 🔧 Produits

## Introduction

Le module **Produits** gère votre catalogue complet de matériaux de construction, équipements et fournitures. Il centralise les informations de prix, références fournisseurs, niveaux de stock et permet une intégration directe avec les devis, achats et inventaire.

Ce module inclut 14 catégories de produits construction et s'intègre avec le module Inventaire pour le suivi des stocks en temps réel.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🔧 Produits"**
2. Le catalogue de produits s'affiche
3. Créez, modifiez ou consultez vos produits
4. Gérez les prix et associations fournisseurs

---

## Structure des Données

### Table PostgreSQL : `produits`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `code_produit` | TEXT | Référence unique (SKU) |
| `nom` | TEXT | Désignation du produit |
| `description` | TEXT | Description détaillée |
| `categorie` | TEXT | Classification (14 catégories) |
| `unite_vente` | TEXT | Unité de mesure |
| `prix_unitaire` | REAL | Prix de vente unitaire |
| `stock_disponible` | REAL | Quantité en stock |
| `stock_minimum` | REAL | Seuil d'alerte |
| `stock_reserve` | REAL | Stock réservé pour projets |
| `stock_en_commande` | REAL | Stock commandé non reçu |
| `fournisseur_principal` | TEXT | Fournisseur par défaut |
| `notes_techniques` | TEXT | Spécifications techniques |
| `emplacement_stock` | TEXT | Localisation entrepôt |
| `actif` | BOOLEAN | Produit actif/inactif |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

---

## Les 14 Catégories de Produits Construction

| # | Catégorie | Exemples |
|---|-----------|----------|
| 1 | **Béton** | Ciment, Mortier, Béton prémélangé |
| 2 | **Bois** | Madriers 2x4, 2x6, Contreplaqué, OSB |
| 3 | **Acier structural** | Poutrelles, Armatures, Barres |
| 4 | **Isolation** | Laine minérale, Styromousse, Roxul |
| 5 | **Plâtre** | Gypse 1/2", 5/8", Composé à joints |
| 6 | **Toiture** | Bardeaux, Membranes, Solins |
| 7 | **Plomberie** | Tuyaux ABS, Cuivre, Raccords |
| 8 | **Électricité** | Câbles, Boîtiers, Panneaux |
| 9 | **Quincaillerie** | Vis, Clous, Fixations |
| 10 | **Finition** | Moulures, Cadrages, Plinthes |
| 11 | **Revêtements** | Vinyle, Céramique, Plancher |
| 12 | **Portes et fenêtres** | Portes, Fenêtres PVC/Alu |
| 13 | **Armature** | Treillis soudé, Barres d'armature |
| 14 | **Granulats** | Gravier, Sable, Pierre concassée |

---

## Produits de Démonstration

Le système peut initialiser des produits de démonstration :

| Code | Produit | Catégorie | Prix | Stock | Fournisseur |
|------|---------|-----------|------|-------|-------------|
| BET-CIM-001 | Ciment Portland GU | Béton | 8.95$ | 250 | Lafarge Canada |
| BOS-MAD-001 | Madrier 2x4x8' SPF | Bois | 7.49$ | 500 | Goodfellow |
| ISO-LAN-001 | Laine R-24 16" | Isolation | 52.99$ | 120 | Roxul/Rockwool |
| PLT-GYP-001 | Gypse 1/2" 4x8' | Plâtre | 13.49$ | 300 | CertainTeed |
| ACI-ARM-001 | Barre armature #4 | Acier structural | 18.95$ | 200 | AGF Steel |
| TOI-BAR-001 | Bardeaux architecturaux | Toiture | 89.99$ | 150 | GAF Canada |
| REV-VIN-001 | Revêtement vinyle blanc | Revêtements | 4.25$ | 800 | Kaycan |

---

## Fonctionnalités Principales

### 1. Gestion des Produits (CRUD)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouveau produit | Ajouter au catalogue |
| **Lire** | 👁️ Voir | Consulter la fiche |
| **Modifier** | ✏️ Modifier | Éditer les informations |
| **Supprimer** | 🗑️ Supprimer | Désactiver ou supprimer |

### 2. Fiche Produit Complète

| Section | Informations |
|---------|--------------|
| **Identification** | Code, Nom, Description |
| **Classification** | Catégorie, Unité de mesure |
| **Prix** | Prix unitaire, Historique |
| **Stock** | Disponible, Minimum, Réservé, En commande |
| **Fournisseur** | Fournisseur principal, Alternatifs |
| **Technique** | Notes techniques, Spécifications |

### 3. Gestion des Prix

| Champ | Description |
|-------|-------------|
| **Prix unitaire** | Prix de vente catalogue |
| **Prix d'achat** | Coût fournisseur (via Inventaire) |
| **Marge** | Calculée automatiquement |
| **Historique** | Évolution des prix |

### 4. Gestion des Stocks

| Champ | Description |
|-------|-------------|
| **Stock disponible** | Quantité actuellement en stock |
| **Stock minimum** | Seuil déclenchant une alerte |
| **Stock réservé** | Quantité réservée pour projets |
| **Stock en commande** | Quantité commandée non encore reçue |

---

## Unités de Mesure Supportées

| Type | Unités |
|------|--------|
| **Linéaire** | m, pi (pied), po (pouce) |
| **Surface** | m², pi² |
| **Volume** | m³, pi³, L (litre) |
| **Quantité** | unité, pqt (paquet), bte (boîte) |
| **Poids** | kg, lb |

---

## Guide Pas-à-Pas

### Créer un nouveau produit

1. Cliquez sur **"➕ Nouveau produit"**
2. Entrez les informations de base :

   **Section Identification :**
   - Code produit (ex: MAT-BOS-001)
   - Nom du produit
   - Description détaillée

   **Section Classification :**
   - Catégorie (sélection parmi 14)
   - Unité de mesure

   **Section Prix :**
   - Prix unitaire de vente

   **Section Stock :**
   - Stock disponible initial
   - Stock minimum (seuil d'alerte)

   **Section Fournisseur :**
   - Fournisseur principal
   - Notes techniques

3. Cliquez sur **"💾 Enregistrer"**
4. Le produit apparaît dans le catalogue

### Mettre à jour les prix

1. Ouvrez la fiche produit
2. Cliquez sur **"✏️ Modifier"**
3. Ajustez le prix unitaire
4. L'ancien prix est conservé dans l'historique
5. Sauvegardez les modifications

### Rechercher un produit

1. Utilisez la **barre de recherche**
2. Tapez le nom, code ou catégorie
3. Filtrez par :
   - Catégorie
   - Fournisseur
   - Niveau de stock (alerte/critique)
4. Cliquez sur le produit pour voir les détails

### Utiliser un produit dans un devis

1. Lors de la création d'un devis
2. Cliquez sur **"Ajouter depuis catalogue"**
3. Recherchez le produit
4. Sélectionnez-le
5. Le prix est automatiquement rempli
6. Ajustez la quantité
7. La ligne est ajoutée au devis

### Configurer les seuils d'alerte

1. Ouvrez la fiche produit
2. Section **"Stock"**
3. Définissez le stock minimum
4. Quand stock < minimum → alerte jaune
5. Quand stock = 0 → alerte rouge

---

## Calcul de la Marge

```
Marge (%) = ((Prix vente - Prix achat) / Prix achat) × 100

Exemple :
Prix achat : 10,00 $
Prix vente : 15,00 $
Marge = ((15 - 10) / 10) × 100 = 50%
```

---

## Intégration avec l'Inventaire

Le module Produits s'intègre avec l'Inventaire :

```sql
SELECT p.*,
       i.quantite_metric as inventory_stock,
       i.statut as inventory_statut,
       i.fournisseur_principal as inventory_fournisseur
FROM produits p
LEFT JOIN inventory_items i ON p.id = i.produit_id
```

---

## Système de Cache

| Donnée | TTL | Description |
|--------|-----|-------------|
| Liste produits | 5 min | Cache local |
| Catégories | 10 min | Liste des catégories |
| Stats stock | 2 min | Alertes et niveaux |

---

## Astuces et Bonnes Pratiques

- **Codifiez logiquement** : CAT-TYPE-XXX (ex: BOS-MAD-001)
- **Mettez à jour les prix** : Les prix des matériaux fluctuent fréquemment
- **Documentez les spécifications** : Dimensions, normes, certifications
- **Gardez plusieurs fournisseurs** : Pour éviter les ruptures
- **Configurez les seuils** : Adaptez selon la criticité du produit
- **Vérifiez le stock disponible** : Avant chaque devis
- **Utilisez les catégories** : Pour des recherches rapides

---

## Résolution de Problèmes

### Le produit n'apparaît pas dans la recherche

- **Cause** : Produit inactif ou filtre actif
- **Solution** : Vérifiez le statut "actif" et les filtres

### Le prix n'est pas à jour dans les devis

- **Cause** : Prix du catalogue modifié après création du devis
- **Solution** : Les devis conservent le prix au moment de la création

### Le stock ne correspond pas à l'inventaire

- **Cause** : Données non synchronisées
- **Solution** : Rafraîchissez les deux modules ou vérifiez les mouvements

---

## Questions Fréquentes (FAQ)

**Q: Comment importer un catalogue fournisseur ?**
R: Utilisez l'import CSV dans Configuration > Import. Préparez un fichier avec les colonnes : code_produit, nom, description, categorie, unite_vente, prix_unitaire.

**Q: Les prix sont-ils mis à jour automatiquement ?**
R: Non, vous devez les mettre à jour manuellement ou via import CSV. Une intégration EDI est prévue pour certains distributeurs majeurs.

**Q: Comment gérer les variantes (tailles, couleurs) ?**
R: Créez un produit par variante avec un code différent, ou utilisez le champ description pour les options.

**Q: Puis-je voir l'historique des prix ?**
R: Oui, chaque modification de prix est enregistrée avec date et utilisateur dans l'onglet "Historique".

**Q: Comment lier un produit à plusieurs fournisseurs ?**
R: Définissez le fournisseur principal dans la fiche produit. Les fournisseurs alternatifs se gèrent via le module Achats.

---

## Données Techniques

### Requête Liste Produits

```sql
SELECT p.*,
       i.quantite_metric as inventory_stock,
       i.statut as inventory_statut
FROM produits p
LEFT JOIN inventory_items i ON p.id = i.produit_id
WHERE p.actif = TRUE
ORDER BY p.categorie, p.nom
```

### Requête Produits par Catégorie

```sql
SELECT * FROM produits
WHERE categorie = :categorie AND actif = TRUE
ORDER BY nom
```

---

## Voir Aussi

- [📦 Inventaire](17-inventaire.md) - Gestion des stocks
- [🏪 Achats](16-achats.md) - Commandes fournisseurs
- [🧾 Devis](06-devis.md) - Utilisation dans les devis
- [💰 Comptabilité](19-comptabilite.md) - Suivi des coûts
