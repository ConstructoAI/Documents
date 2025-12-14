# 📦 Inventaire

## Introduction

Le module **Inventaire** assure le suivi en temps réel de vos stocks de matériaux et équipements. Il gère les entrées, sorties, niveaux d'alerte et valorisation pour optimiser votre gestion des approvisionnements.

Ce module inclut 24 catégories de matériaux de construction, 7 types d'unités de mesure, 10 normes québécoises reconnues, et calcule automatiquement les alertes de stock.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📦 Inventaire"**
2. L'état des stocks s'affiche avec les onglets :
   - **Articles** : Liste des produits en stock
   - **Mouvements** : Historique des entrées/sorties
   - **Alertes** : Articles en stock critique
   - **Statistiques** : Valorisation et analyses
3. Gérez les mouvements et consultez les alertes

---

## Structure des Données

### Table PostgreSQL : `inventory_items`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom` | TEXT | Désignation de l'article |
| `code` | TEXT | Code/SKU unique |
| `description` | TEXT | Description détaillée |
| `categorie` | TEXT | Classification (24 catégories) |
| `unite` | TEXT | Unité de mesure |
| `quantite_stock` | REAL | Quantité actuelle en stock |
| `seuil_alerte` | REAL | Seuil déclenchant alerte |
| `prix_unitaire` | REAL | Prix d'achat unitaire |
| `emplacement` | TEXT | Localisation dans l'entrepôt |
| `fournisseur_principal` | TEXT | Fournisseur par défaut |
| `produit_id` | INTEGER | Lien vers catalogue Produits |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Table : `stock_movements`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `product_id` | INTEGER | Lien vers l'article |
| `type_mouvement` | TEXT | ENTREE, SORTIE, AJUSTEMENT |
| `quantite` | REAL | Quantité du mouvement |
| `date_mouvement` | TIMESTAMP | Date et heure |
| `reference` | TEXT | Référence document (BC, BT) |
| `notes` | TEXT | Notes explicatives |
| `employee_id` | INTEGER | Utilisateur responsable |

---

## Les 24 Catégories de Matériaux Construction

| # | Catégorie | Exemples |
|---|-----------|----------|
| 1 | **Béton et ciment** | Ciment Portland, mortier |
| 2 | **Acier de construction** | Poutrelles, colonnes |
| 3 | **Bois de charpente** | Madriers 2x4, 2x6, 2x8 |
| 4 | **Contreplaqué et OSB** | Panneaux structuraux |
| 5 | **Isolants thermiques** | Laine, mousse rigide |
| 6 | **Pare-vapeur et membranes** | Polyéthylène, Tyvek |
| 7 | **Gypse et produits de finition** | Panneaux 1/2", 5/8" |
| 8 | **Bardeaux et toiture** | Bardeaux, membranes |
| 9 | **Revêtements extérieurs** | Vinyle, fibrociment |
| 10 | **Portes et fenêtres** | Portes, fenêtres PVC |
| 11 | **Plomberie - Tuyauterie** | Tuyaux ABS, cuivre, PEX |
| 12 | **Plomberie - Fixtures** | Robinets, toilettes |
| 13 | **Électricité - Câblage** | Fils 14/2, 12/2, 10/3 |
| 14 | **Électricité - Boîtes et panneaux** | Boîtiers, disjoncteurs |
| 15 | **Quincaillerie de construction** | Vis, clous, boulons |
| 16 | **Fixations et ancrages** | Ancrages béton, tire-fond |
| 17 | **Adhésifs et scellants** | PL Premium, silicone |
| 18 | **Peinture et teinture** | Latex, alkyde, teinture |
| 19 | **Agrégats (sable, gravier)** | Sable 0-5mm, gravier |
| 20 | **Pierre concassée** | 0-3/4", MG-20 |
| 21 | **Armature et treillis** | Barres #3, #4, #5, treillis |
| 22 | **Produits de drainage** | Drain français, géotextile |
| 23 | **Géotextiles** | Membranes de protection |
| 24 | **Équipements de sécurité** | Casques, harnais, gants |

---

## Unités de Mesure Construction

| Type | Unités disponibles |
|------|--------------------|
| **Linéaire** | m (mètre), pi (pied), po (pouce) |
| **Surface** | m² (mètre carré), pi² (pied carré) |
| **Volume** | m³ (mètre cube), pi³ (pied cube), vg³ (verge cube) |
| **Poids** | kg (kilogramme), lb (livre), tonne |
| **Liquide** | L (litre), gal (gallon) |
| **Unité** | unité, boîte, paquet, palette, rouleau, sac, feuille |

---

## Normes de Construction Québécoises

| # | Norme | Description |
|---|-------|-------------|
| 1 | **CSA A23.1** | Béton - Constituants et exécution |
| 2 | **CSA A23.2** | Méthodes d'essai pour le béton |
| 3 | **CSA O86** | Règles de calcul du bois d'ingénierie |
| 4 | **CSA S16** | Règles de calcul des charpentes d'acier |
| 5 | **BNQ 2560-114** | Granulats - Béton et mortier |
| 6 | **BNQ 2621-905** | Béton de ciment hydraulique |
| 7 | **CAN/ULC-S701** | Isolants thermiques - Polystyrène |
| 8 | **CAN/ULC-S703** | Isolants - Mousse de polyuréthane |
| 9 | **ASTM** | Standards internationaux |
| 10 | **Code national du bâtiment** | CNB 2020 |

---

## Fonctionnalités Principales

### 1. État des Stocks

| Information | Description |
|-------------|-------------|
| **Produit** | Référence (code) et nom |
| **Quantité en stock** | Stock actuel disponible |
| **Unité** | Unité de mesure |
| **Seuil d'alerte** | Niveau minimum avant alerte |
| **Emplacement** | Localisation (entrepôt, rangée, tablette) |
| **Valeur** | Stock × Prix d'achat |

### 2. Types de Mouvements

| Type | Direction | Code | Origine |
|------|-----------|------|---------|
| **ENTREE** | (+) | Ajout | Réception commande, retour |
| **SORTIE** | (-) | Retrait | Livraison chantier, consommation |
| **AJUSTEMENT** | (±) | Correction | Inventaire physique, écart |

### 3. Alertes de Stock

| Niveau | Condition | Couleur | Action recommandée |
|--------|-----------|---------|-------------------|
| **Stock normal** | quantité > seuil × 1.5 | 🟢 Vert | RAS |
| **Stock bas** | quantité ≤ seuil × 1.5 | 🟡 Jaune | Planifier commande |
| **Stock critique** | quantité ≤ seuil | 🔴 Rouge | Commander urgent |
| **Rupture** | quantité = 0 | ⚫ Noir | Stock épuisé |

### 4. Valorisation du Stock

```
Valeur article = Quantité en stock × Prix unitaire
Valeur totale = Σ (Valeurs articles)
```

Méthodes supportées :
- **Coût moyen pondéré** : Prix moyen des achats (défaut)
- **FIFO** : Premier entré, premier sorti
- **Dernier prix** : Prix du dernier achat

---

## Guide Pas-à-Pas

### Consulter l'état des stocks

1. Ouvrez le module **Inventaire**
2. Onglet **"Articles"** - La liste des produits s'affiche
3. Utilisez les filtres :
   - Par catégorie (24 catégories disponibles)
   - Par emplacement
   - Par niveau de stock (alerte/critique)
   - Recherche par nom ou code
4. Triez par quantité, valeur ou nom
5. Cochez **"Stock critique seulement"** pour voir les alertes

### Enregistrer une entrée de stock

1. Onglet **"Mouvements"**
2. Section **"Ajouter un mouvement"**
3. Sélectionnez l'article dans la liste
4. Type de mouvement : **"ENTREE"**
5. Entrez la quantité reçue
6. Référence : Numéro du bon d'achat (ex: BA-2025-0042)
7. Notes : Détails additionnels
8. Cliquez sur **"✅ Enregistrer le mouvement"**
9. Le stock est mis à jour automatiquement

### Enregistrer une sortie de stock

1. Onglet **"Mouvements"**
2. Sélectionnez l'article
3. Type de mouvement : **"SORTIE"**
4. Entrez la quantité sortie
5. Référence : Numéro du bon de travail ou projet
6. Notes : Destination (ex: "Chantier Laval - Lot 45")
7. Cliquez sur **"✅ Enregistrer le mouvement"**
8. Le stock diminue automatiquement

### Faire un ajustement d'inventaire

1. Après un inventaire physique
2. Onglet **"Mouvements"**
3. Sélectionnez l'article avec écart
4. Type de mouvement : **"AJUSTEMENT"**
5. Entrez la quantité d'ajustement :
   - Positive si stock réel > stock système
   - Négative si stock réel < stock système
6. Notes : Motif de l'ajustement (Erreur, Vol, Dommage, etc.)
7. Validez l'ajustement
8. Un historique est conservé pour audit

### Configurer les seuils d'alerte

1. Onglet **"Articles"**
2. Cliquez sur l'article à configurer
3. Cliquez sur **"✏️ Modifier"**
4. Section **"Stock"** :
   - **Seuil d'alerte** : Niveau déclenchant l'alerte
   - Le système calcule automatiquement les niveaux
5. Sauvegardez
6. Les alertes apparaissent dans l'onglet **"Alertes"**

### Consulter l'historique des mouvements

1. Onglet **"Mouvements"**
2. Filtrez par article ou affichez tous les mouvements
3. L'historique affiche :
   - Date et heure du mouvement
   - Type (ENTREE, SORTIE, AJUSTEMENT)
   - Quantité (avec signe + ou -)
   - Référence document
   - Notes
   - Article concerné

### Exporter les stocks critiques

1. Onglet **"Alertes"** ou **"Statistiques"**
2. Section **"Actions rapides"**
3. Cliquez sur **"📥 Exporter stocks critiques"**
4. Un fichier CSV est téléchargé avec :
   - Code article
   - Nom
   - Quantité actuelle
   - Seuil d'alerte
   - Unité
5. Utilisez ce fichier pour passer vos commandes

---

## Emplacements de Stock

| Type | Description | Exemple |
|------|-------------|---------|
| **Entrepôt principal** | Stock central, siège social | ENT-01 |
| **Entrepôt secondaire** | Dépôt régional | ENT-02 |
| **Chantier** | Stock sur site de projet | CHAN-LOT45 |
| **Véhicule** | Stock mobile (camion) | VEH-123 |
| **Fournisseur** | Stock en consignation | CONS-LAFARGE |

---

## Statistiques Inventaire

Le module calcule automatiquement :

| Statistique | Description |
|-------------|-------------|
| **Valeur totale** | Somme (quantité × prix unitaire) |
| **Nombre d'articles** | Total d'articles en inventaire |
| **Alertes stock bas** | Articles sous le seuil |
| **Valeur par catégorie** | Répartition de la valeur |
| **Mouvements récents** | Activité des 7 derniers jours |

---

## Lien Produits ↔ Inventaire

Le module Inventaire s'intègre avec le module Produits :

```sql
SELECT p.*,
       i.quantite_stock as inventory_stock,
       i.seuil_alerte,
       i.emplacement
FROM produits p
LEFT JOIN inventory_items i ON p.id = i.produit_id
```

| Module | Rôle |
|--------|------|
| **Produits** | Catalogue de référence (prix vente, catégories) |
| **Inventaire** | Suivi opérationnel (quantités, mouvements) |

---

## Système de Cache

| Donnée | TTL | Description |
|--------|-----|-------------|
| Liste articles | 2 min | Cache des articles en stock |
| Stock critique | 1 min | Liste des alertes (mise à jour fréquente) |
| Statistiques | 5 min | Valorisation et totaux |
| Mouvements | 2 min | Historique des mouvements |

---

## Astuces et Bonnes Pratiques

- **Inventaire régulier** : Comptez physiquement au moins mensuellement pour éviter les écarts
- **Seuils personnalisés** : Adaptez les seuils selon la criticité et le délai de réapprovisionnement
- **Documentez les sorties** : Toujours associer à un projet, BT ou référence
- **Retours systématiques** : Récupérez les surplus de chantier et réintégrez-les
- **Valorisation à jour** : Mettez à jour les prix d'achat après chaque commande
- **Codes cohérents** : Utilisez une codification logique (CAT-TYPE-XXX)
- **Emplacements précis** : Définissez un système de localisation clair
- **Vérifiez les alertes quotidiennement** : Ne laissez pas le stock atteindre zéro

---

## Résolution de Problèmes

### Le stock ne correspond pas à l'inventaire physique

- **Cause** : Mouvements non enregistrés ou erreurs de saisie
- **Solution** : Faites un ajustement avec motif documenté

### L'alerte ne s'affiche pas

- **Cause** : Seuil d'alerte non configuré (= 0)
- **Solution** : Définissez un seuil > 0 pour l'article

### Le mouvement ne s'enregistre pas

- **Cause** : Quantité manquante ou article non sélectionné
- **Solution** : Vérifiez tous les champs obligatoires

### La valeur du stock est incorrecte

- **Cause** : Prix unitaire à zéro ou non défini
- **Solution** : Mettez à jour le prix unitaire de l'article

---

## Questions Fréquentes (FAQ)

**Q: Comment voir l'historique des mouvements d'un produit ?**
R: Onglet "Mouvements", filtrez par l'article concerné. Tous les entrées/sorties sont listées avec dates et références.

**Q: Puis-je réserver du stock pour un projet ?**
R: La réservation se fait via le module Produits (stock_reserve). L'inventaire affiche le stock disponible réel.

**Q: Comment gérer le stock sur plusieurs chantiers ?**
R: Créez un emplacement par chantier. Utilisez les mouvements de type SORTIE avec référence du chantier.

**Q: La valorisation est-elle fiscalement conforme ?**
R: Oui, les méthodes coût moyen et FIFO sont acceptées fiscalement au Québec selon les normes comptables.

**Q: Comment annuler un mouvement erroné ?**
R: Créez un mouvement inverse (ENTREE si SORTIE erronée, et vice versa) avec note explicative.

**Q: Puis-je importer mon inventaire ?**
R: Oui, via Configuration > Import CSV. Format requis : code, nom, categorie, quantite_stock, seuil_alerte, prix_unitaire, emplacement.

---

## Données Techniques

### Requête Articles avec Alertes

```sql
SELECT id, nom, code, quantite_stock, seuil_alerte, unite
FROM inventory_items
WHERE quantite_stock <= seuil_alerte AND seuil_alerte > 0
ORDER BY (quantite_stock::float / NULLIF(seuil_alerte, 0)) ASC
```

### Requête Valeur Totale Stock

```sql
SELECT COALESCE(SUM(quantite_stock * prix_unitaire), 0) as valeur_totale
FROM inventory_items
WHERE quantite_stock > 0
```

### Requête Mouvements Récents

```sql
SELECT m.id, m.product_id, m.type_mouvement, m.quantite,
       m.date_mouvement, m.reference, m.notes, p.nom as produit_nom
FROM stock_movements m
LEFT JOIN inventory_items p ON m.product_id = p.id
ORDER BY m.date_mouvement DESC
LIMIT 100
```

---

## Voir Aussi

- [🔧 Produits](15-produits.md) - Catalogue produits
- [🏪 Achats](16-achats.md) - Approvisionnement
- [🚚 Logistique](18-logistique.md) - Livraisons
- [🏭 Production](11-production.md) - Consommation chantier
