# 📊 Analytics BI

## Introduction

Le module **Analytics BI** (Business Intelligence) est votre centre d'analyse avancée de type Power BI. Il offre des tableaux de bord interactifs avec plus de 30 graphiques Plotly, organisés en 5 onglets thématiques, pour analyser en profondeur toutes les données de votre entreprise de construction.

Ce module transforme vos données opérationnelles en insights décisionnels avec des visualisations professionnelles : graphiques à barres, camemberts, entonnoirs (funnel), treemaps, heatmaps et bien d'autres.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📊 Analytics BI"**
2. Le tableau de bord BI se charge avec un header bleu stylisé
3. Utilisez les filtres en haut pour personnaliser la période d'analyse
4. Naviguez entre les 5 onglets thématiques

---

## Structure du Module

### En-tête et Filtres Globaux

| Élément | Description |
|---------|-------------|
| **Titre** | "📊 Analytics BI Dashboard - Business Intelligence" |
| **Filtre période** | Sélecteur déroulant (30 jours, 90 jours, etc.) |
| **Bouton Actualiser** | Rafraîchit toutes les données |
| **Bouton Export PDF** | Export du rapport (fonctionnalité à venir) |

### Périodes d'Analyse Disponibles

| Période | Jours | Usage recommandé |
|---------|-------|------------------|
| **30 jours** | 30 | Suivi opérationnel court terme |
| **90 jours** | 90 | Analyse trimestrielle |
| **6 mois** | 180 | Tendances moyen terme |
| **Année fiscale** | 365 | Bilan annuel |
| **Tout** | Illimité | Historique complet |

---

## Section 1 : KPI Cards Globaux

### Première ligne (4 KPIs)

| KPI | Icône | Style | Description |
|-----|-------|-------|-------------|
| **Revenus Totaux** | 💰 | Vert (success) | Somme des projets terminés |
| **Projets Actifs** | 🏗️ | Bleu (info) | Projets EN COURS + À FAIRE |
| **Employés Actifs** | 👥 | Bleu primaire | Employés statut ACTIF |
| **Alertes Stock** | ⚠️ | Rouge/Orange | Items sous seuil critique |

### Seconde ligne (1 KPI)

| KPI | Icône | Style | Description |
|-----|-------|-------|-------------|
| **Opportunités** | 💼 | Bleu (info) | Pipeline commercial + valeur totale |

### Design des Cartes KPI

- **Background** : Couleur selon le type (vert, bleu, rouge)
- **Effet hover** : Élévation légère (translateY -3px)
- **Ombre** : Box-shadow avec couleur assortie
- **Animation** : Transition 0.3s ease

---

## Section 2 : Les 5 Onglets Thématiques

### Onglet 1 : 🏗️ Projets & Production

#### Graphiques disponibles

| Graphique | Type | Description |
|-----------|------|-------------|
| **Rentabilité Projets** | Barres groupées + ligne | Budget vs Coûts vs Marge (Top 10) |
| **Charge Postes de Travail** | Barres avec seuil | Taux de charge par poste (%) |
| **Évolution Mensuelle** | Aire empilée | Projets par statut dans le temps |
| **Performance par Type** | Double barre | Durée moyenne + Respect délais |
| **Opérations par Poste** | Barres empilées + ligne | Terminées/En cours + Taux completion |
| **Avancement Projets** | Tableau interactif | Progression avec barres colorées |

#### Métriques clés affichées

- Marge moyenne (%)
- Nombre de projets analysés
- CA Total
- Postes en surcharge (alertes rouges)

#### Indicateurs de couleur - Charge des postes

| Taux de charge | Couleur | Signification |
|----------------|---------|---------------|
| > 100% | 🔴 Rouge (`#EF4444`) | Surcharge |
| 80-100% | 🟠 Orange (`#F59E0B`) | Attention |
| < 80% | 🟢 Vert (`#22C55E`) | Normal |

---

### Onglet 2 : 💼 Commercial & CRM

#### Graphiques disponibles

| Graphique | Type | Description |
|-----------|------|-------------|
| **Pipeline Commercial** | Entonnoir (Funnel) | Opportunités par étape |
| **Top Clients** | Tableau + Camembert | Classement par CA |
| **Évolution Opportunités** | Barres + ligne | Mensuel + Taux de succès |
| **Performance Commerciaux** | Barres + ligne | CA réalisé + Taux conversion |
| **Devis par Catégorie** | Barres + ligne | Montant + Taux approbation |

#### Métriques Taux de Conversion Devis

| Métrique | Calcul |
|----------|--------|
| **Total Devis** | COUNT(devis) |
| **Taux d'Envoi** | Envoyés / Total × 100 |
| **Taux de Conversion** | Approuvés / Envoyés × 100 |
| **Valeur Moyenne** | AVG(montant_total) |

#### Étapes du Pipeline Commercial

| Étape | Position | Couleur |
|-------|----------|---------|
| Prospection | 1 | Cyan |
| Qualification | 2 | Bleu primaire |
| Proposition | 3 | Orange |
| Négociation | 4 | Vert |
| Gagné | 5 | Vert foncé |

---

### Onglet 3 : 💰 Finances & Budget

#### Graphiques disponibles

| Graphique | Type | Description |
|-----------|------|-------------|
| **Revenus vs Dépenses** | Barres + ligne | Mensuel + Marge (%) |
| **Marges par Type** | Tableau + Barres | Par priorité de projet |
| **Rentabilité par Client** | Treemap | Surface selon CA |
| **Répartition des Coûts** | Donut | Main-d'œuvre, Matériaux, Sous-traitance |
| **Flux de Trésorerie** | Barres + ligne | Hebdomadaire |
| **Budget vs Réel** | Double barres + ligne | Top 10 écarts |
| **CA Mensuel 12 mois** | Barres + ligne | Évolution annuelle |

#### Métriques financières calculées

```
Marge Totale = Revenus - Dépenses
Marge % = (Marge / Revenus) × 100
Écart % = ((Budget - Coût Réel) / Budget) × 100
```

#### Catégories de coûts analysées

| Catégorie | Source | Calcul |
|-----------|--------|--------|
| **Main-d'œuvre** | time_entries + employees | heures × salaire horaire |
| **Matériaux** | materials | quantité × prix unitaire |
| **Sous-traitance** | formulaires (BON_COMMANDE) | montant_total |

---

### Onglet 4 : 👥 Ressources Humaines

#### Graphiques disponibles

| Graphique | Type | Description |
|-----------|------|-------------|
| **Productivité Employés** | Barres colorées | Top 15 par heures (Viridis) |
| **Répartition Temps** | Donut | Par département |
| **Évolution Effectifs** | Lignes | Embauches + Cumulatif |
| **Coûts Salariaux** | Barres horizontales | Par département (Reds) |

#### Métriques RH calculées

| Métrique | Formule |
|----------|---------|
| **Heures par jour** | Heures totales / Jours travaillés |
| **Coût employé** | Heures × Salaire horaire |
| **Productivité moyenne** | Moyenne des heures par employé |

#### Échelle de couleurs - Productivité

Le graphique utilise l'échelle **Viridis** :
- Bleu foncé → Heures faibles
- Vert → Heures moyennes
- Jaune → Heures élevées

---

### Onglet 5 : 📦 Inventaire & Achats

#### Graphiques disponibles

| Graphique | Type | Description |
|-----------|------|-------------|
| **Alertes Stock Critiques** | Tableau avec progress bar | Items sous seuil |
| **Top Fournisseurs** | Tableau | Par nombre de commandes |
| **Mouvements Stock** | Barres groupées | Entrées vs Sorties hebdo |
| **Valeur Stock** | Sunburst | Par catégorie |
| **Rotation Stock** | Scatter | Mouvements vs Taux rotation |

#### Statistiques Stock

| Métrique | Description |
|----------|-------------|
| **Items en Alerte** | stock_actuel ≤ seuil_alerte |
| **Niveau Critique** | taux_stock < 50% |
| **Taux de rotation** | quantité_sortie / stock_actuel |

#### Niveaux d'alerte stock

| Taux stock | Statut | Action |
|------------|--------|--------|
| < 50% | 🔴 CRITIQUE | Commander immédiatement |
| 50-75% | 🟠 FAIBLE | Planifier commande |
| > 75% | 🟢 SUFFISANT | Aucune action |

---

## Types de Graphiques Utilisés

### Graphiques Plotly

| Type | Usage | Interactivité |
|------|-------|---------------|
| **Bar Chart** | Comparaisons | Survol, zoom |
| **Pie/Donut** | Répartitions | Rotation, labels |
| **Line Chart** | Tendances | Points, annotations |
| **Funnel** | Pipelines | Pourcentages |
| **Treemap** | Hiérarchies | Drill-down |
| **Sunburst** | Catégories | Expansion |
| **Scatter** | Corrélations | Taille variable |
| **Heatmap** | Matrices | Échelle couleur |
| **Stacked Area** | Cumuls | Empilage |

### Palette de Couleurs

| Nom | Code | Usage |
|-----|------|-------|
| Primary | `#3B82F6` | Éléments principaux |
| Success | `#22C55E` | Positif, revenus |
| Warning | `#F59E0B` | Attention, alertes |
| Danger | `#EF4444` | Critique, dépenses |
| Info | `#06B6D4` | Information neutre |

---

## Système de Cache

### Performance optimisée

Le module utilise un système de cache intelligent :

| Donnée | TTL (durée) | Raison |
|--------|-------------|--------|
| **KPIs globaux** | 1 minute | Données critiques temps réel |
| **Avancement projets** | 2 minutes | Mise à jour fréquente |
| **Pipeline commercial** | 2 minutes | Opportunités actives |
| **Alertes stock** | 2 minutes | Urgence opérationnelle |
| **Productivité** | 5 minutes | Données RH stables |
| **Rentabilité projets** | 5 minutes | Calculs complexes |
| **Top clients** | 10 minutes | Données historiques |
| **Top fournisseurs** | 10 minutes | Données stables |
| **Revenus mensuels** | 10 minutes | Historique comptable |

### Invalidation du cache

Le bouton **🔄 Actualiser** invalide tout le cache et recharge les données fraîches.

---

## Guide Pas-à-Pas

### Analyser la rentabilité d'un projet

1. Accédez à **Analytics BI**
2. Cliquez sur l'onglet **"🏗️ Projets & Production"**
3. Dans le graphique "Rentabilité Projets (Top 10)" :
   - Barres bleues = Budget
   - Barres rouges = Coûts réels
   - Ligne verte = Marge en %
4. Survolez un projet pour voir les détails
5. Identifiez les projets avec marge < 0 (déficit)

### Identifier les goulots d'étranglement

1. Allez dans **"🏗️ Projets & Production"**
2. Consultez le graphique **"Charge des Postes de Travail"**
3. Repérez les barres rouges (> 100%)
4. Dans le panneau "Postes Critiques", lisez les alertes
5. Réaffectez les ressources ou ajoutez du personnel

### Évaluer l'efficacité commerciale

1. Cliquez sur l'onglet **"💼 Commercial & CRM"**
2. Le **Pipeline Commercial** (entonnoir) montre les étapes
3. Analysez où les opportunités "bloquent"
4. Vérifiez le **Taux de Conversion** (objectif > 30%)
5. Consultez la **Performance des Commerciaux**

### Suivre la trésorerie

1. Allez dans **"💰 Finances & Budget"**
2. Graphique **"Revenus vs Dépenses"** :
   - Barres vertes = Revenus
   - Barres rouges = Dépenses
   - Ligne = Marge %
3. Consultez **"Flux de Trésorerie"** pour le détail hebdo
4. Surveillez le solde net (ligne bleue)

### Optimiser les stocks

1. Cliquez sur **"📦 Inventaire & Achats"**
2. Le tableau **"Alertes Stock Critiques"** liste les urgences
3. La colonne "Niveau" avec barre de progression montre la criticité
4. Passez commande pour les items < 50%
5. Analysez la **Rotation Stock** pour éviter le surstockage

---

## Exports et Rapports

### Options d'export actuelles

| Format | Disponibilité | Usage |
|--------|---------------|-------|
| **Image PNG** | ✅ Disponible | Clic sur icône 📷 dans chaque graphique |
| **Export PDF** | 🔜 À venir | Rapport complet multi-pages |
| **Export Excel** | 🔜 À venir | Données brutes des tableaux |

### Export d'un graphique

1. Survolez le graphique souhaité
2. Dans le coin supérieur droit, cliquez sur l'icône **📷**
3. L'image PNG est téléchargée
4. Utilisez-la dans vos présentations

---

## Données Techniques

### Tables PostgreSQL utilisées

| Onglet | Tables sources |
|--------|----------------|
| Projets | `projects`, `operations`, `work_centers`, `time_entries` |
| Commercial | `opportunities`, `companies`, `formulaires` (devis) |
| Finances | `projects`, `time_entries`, `materials`, `employees` |
| RH | `employees`, `time_entries` |
| Inventaire | `inventory_items`, `inventory_history`, `fournisseurs` |

### Requêtes SQL optimisées

- Jointures LEFT pour inclure les entrées sans correspondance
- Agrégations avec COALESCE pour gérer les NULL
- Filtres par période avec INTERVAL
- Limites (LIMIT) pour les Top N
- Tri par métriques clés (ORDER BY)

---

## Résolution de Problèmes

### Les graphiques sont vides

- **Cause** : Aucune donnée pour la période sélectionnée
- **Solution** : Étendez la période (ex: "Tout" au lieu de "30 jours")

### Le chargement est lent

- **Cause** : Beaucoup de données à analyser
- **Solution** : Réduisez la période ou attendez la mise en cache

### Les KPIs affichent 0

- **Cause** : Modules correspondants non utilisés
- **Solution** : Commencez par créer des projets, opportunités, etc.

### Message "Aucune opportunité commerciale"

- **Cause** : Module CRM non configuré
- **Solution** : Créez des opportunités dans le module CRM

---

## Astuces et Bonnes Pratiques

- **Consultez le BI chaque semaine** pour détecter les tendances
- **Comparez les périodes** : 30 jours vs 90 jours pour voir l'évolution
- **Surveillez les alertes rouges** en priorité (surcharge, stock critique)
- **Exportez les graphiques** pour vos réunions de direction
- **Analysez par onglet** selon votre rôle (Finance pour CFO, RH pour DRH)
- **Utilisez les tableaux interactifs** : triez, filtrez les colonnes

---

## Questions Fréquentes (FAQ)

**Q: D'où viennent les données analysées ?**
R: Les données proviennent en temps réel de tous les modules de CONSTRUCTO AI via PostgreSQL.

**Q: À quelle fréquence les graphiques sont-ils mis à jour ?**
R: Le cache varie de 1 à 10 minutes selon le type de données. Utilisez le bouton "Actualiser" pour forcer le rafraîchissement.

**Q: Puis-je créer mes propres tableaux de bord ?**
R: Actuellement, les tableaux de bord sont prédéfinis avec 30+ graphiques. Une fonctionnalité de personnalisation est prévue.

**Q: Comment exporter un graphique ?**
R: Survolez le graphique et cliquez sur l'icône 📷 (appareil photo) pour télécharger en PNG.

**Q: Les calculs sont-ils fiables ?**
R: Oui, toutes les formules sont documentées et les données proviennent directement de votre base PostgreSQL.

**Q: Pourquoi certains graphiques utilisent des données simulées ?**
R: Pour les tables optionnelles (comme `inventory_history`), des données de démonstration sont générées si la table est vide.

---

## Voir Aussi

- [🏠 Tableau de Bord](01-tableau-de-bord.md) - Vue d'ensemble rapide
- [📋 Projets](07-projets.md) - Détail des projets
- [💰 Comptabilité](19-comptabilite.md) - Données financières détaillées
- [🙋‍♂️ Assistant IA](24-assistant-ia.md) - Analyses personnalisées
- [📦 Inventaire](17-inventaire.md) - Gestion des stocks
