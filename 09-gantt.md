# 📈 Gantt

## Introduction

Le module **Gantt** est un outil avancé de planification visuelle qui affiche chronologiquement vos Devis, Projets, Achats et Bons de Travail. Basé sur Plotly, il offre des fonctionnalités professionnelles : chemin critique, export multi-format, thèmes personnalisables, analyse prédictive et scénarios "What-If".

Ce module de plus de 2500 lignes de code est intégré avec tous les modules de gestion de l'ERP pour une vue consolidée de la planification.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📈 Gantt"**
2. Sélectionnez l'onglet souhaité : Devis, Projets, Achats ou Bons de Travail
3. Appliquez les filtres et options selon vos besoins
4. Interagissez avec le diagramme (zoom, sélection, navigation)

---

## Les 4 Vues Gantt

Le module offre 4 vues distinctes organisées en onglets :

| Onglet | Icône | Description |
|--------|-------|-------------|
| **Devis/Estimations** | 💰 | Timeline des devis et soumissions |
| **Projets** | 🏗️ | Timeline des projets de construction |
| **Achats** | 🛒 | Bons d'achat et commandes |
| **Bons de Travail** | 📋 | BT avec postes de travail |

---

## Palette de Couleurs Standardisée

### Couleurs Bons de Travail (BT)

| Statut | Couleur | Hex |
|--------|---------|-----|
| **BROUILLON** | Gris foncé | `#4b5563` |
| **VALIDÉ** | Bleu | `#3b82f6` |
| **ENVOYÉ** | Bleu | `#3b82f6` |
| **APPROUVÉ** | Vert | `#10b981` |
| **EN_COURS** | Jaune | `#eab308` |
| **TERMINÉ** | Vert | `#10b981` |
| **ANNULÉ** | Noir | `#1f2937` |

### Couleurs Postes/Opérations

| Statut | Couleur | Hex |
|--------|---------|-----|
| **À FAIRE** | Bleu | `#3b82f6` |
| **EN_COURS** | Jaune | `#eab308` |
| **TERMINÉ** | Vert | `#10b981` |
| **SUSPENDU** | Bleu | `#3b82f6` |
| **ANNULÉ** | Noir | `#1f2937` |

---

## Thèmes Visuels

### Thème Clair (défaut)

| Élément | Couleur |
|---------|---------|
| Fond principal | `#FAFBFF` |
| Fond papier | `#F8FAFF` |
| Grille | `#E2E8F0` |
| Grille semaine | `#CBD5E1` |
| Week-ends | `rgba(226, 232, 240, 0.3)` |
| Texte | `#1F2937` |
| Ligne Aujourd'hui | `#EF4444` (rouge) |

### Thème Sombre

| Élément | Couleur |
|---------|---------|
| Fond principal | `#111827` |
| Fond papier | `#1F2937` |
| Grille | `#4B5563` |
| Grille semaine | `#374151` |
| Week-ends | `rgba(75, 85, 99, 0.3)` |
| Texte | `#E5E7EB` |
| Ligne Aujourd'hui | `#F87171` (rouge clair) |

---

## Fonctionnalités Principales

### 1. Diagramme Gantt Interactif

| Fonctionnalité | Description |
|----------------|-------------|
| **Barres colorées** | Une barre par élément selon le statut |
| **Texte sur barre** | Numéro et description affichés |
| **Survol (hover)** | Informations détaillées au survol |
| **Zoom temporel** | Jour, Semaine, Mois, Année, Tout |
| **Range slider** | Navigation rapide sur la timeline |
| **Ligne Aujourd'hui** | Marqueur rouge pointillé |
| **Week-ends grisés** | Distinction visuelle des week-ends |

### 2. Filtres et Options

| Filtre | Description |
|--------|-------------|
| **Statut** | Filtrer par statut (Tous, En cours, etc.) |
| **Priorité** | Filtrer par niveau de priorité |
| **Thème** | Clair ou Sombre |
| **Afficher postes** | Inclure/exclure les opérations |
| **Recherche** | Recherche par numéro de BT |
| **Plage de dates** | Période personnalisée |

### 3. Niveaux de Zoom

| Zoom | Plage |
|------|-------|
| **Jour** | -7 à +7 jours |
| **Semaine** | -4 à +4 semaines |
| **Mois** | -90 à +90 jours |
| **Année** | -180 à +365 jours |
| **Tout** | Toutes les données |

### 4. Presets de Filtres

Sauvegardez et chargez des configurations de filtres :

1. Configurez vos filtres souhaités
2. Entrez un nom pour le preset
3. Cliquez sur **"💾 Sauvegarder filtre"**
4. Rechargez le preset depuis la liste déroulante

### 5. Templates de Projets

Créez des modèles réutilisables :

1. Configurez un projet type
2. Sauvegardez comme template
3. Utilisez le template pour créer de nouveaux projets similaires

---

## Fonctionnalités Avancées

### Chemin Critique

L'algorithme identifie les BT avec le moins de marge :

```python
def identify_critical_path(bts_list):
    """Identifie les BT critiques (marge <= 2 jours)."""
    for bt in bts_list:
        # Calculer durée prévue
        duree_prevue = (date_echeance - date_debut).days

        # Calculer temps nécessaire (8h/jour)
        jours_necessaires = temps_total_operations / 8

        # Calculer marge
        marge = duree_prevue - jours_necessaires

        # BT critique si marge <= 2 jours
        if marge <= 2:
            risque = 'ÉLEVÉ' if marge < 0 else 'MOYEN'
```

| Marge | Niveau de Risque |
|-------|------------------|
| < 0 jours | 🔴 ÉLEVÉ |
| 0-1 jours | 🟠 MOYEN |
| 1-2 jours | 🟡 FAIBLE |
| > 2 jours | 🟢 Normal |

### Calcul de Progression

Progression calculée sur deux facteurs :

```python
def calculate_bt_progression(bt_data, erp_db):
    # Méthode 1: Heures pointées vs estimées (60%)
    progression_heures = (heures_pointees / heures_estimees) * 100

    # Méthode 2: Statut des opérations (40%)
    # Terminées = 100%, En cours = 50%
    progression_operations = ((terminees + en_cours * 0.5) / total) * 100

    # Moyenne pondérée
    return (progression_heures * 0.6) + (progression_operations * 0.4)
```

### Analyse Prédictive IA

Estimation du risque de retard basée sur :

| Facteur | Points |
|---------|--------|
| Progression < attendue (-20%) | +40 |
| Priorité HAUTE | +20 |
| > 10 opérations | +15 |
| Statut BROUILLON/VALIDÉ | +10 |

| Score | Niveau | Action |
|-------|--------|--------|
| ≥ 60 | 🔴 ÉLEVÉ | Action immédiate requise |
| 30-59 | 🟠 MOYEN | Surveillance recommandée |
| < 30 | 🟢 FAIBLE | Bon avancement |

### Dépendances entre BT

Gérez les relations de dépendance :

| Type | Description |
|------|-------------|
| **Fin-Début (FS)** | BT B commence après fin de BT A |
| **Début-Début (SS)** | BT B commence avec BT A |
| **Fin-Fin (FF)** | BT B finit avec BT A |

### Scénarios What-If

Simulez des modifications sans affecter les données réelles :

1. Créez un scénario nommé
2. Appliquez des modifications hypothétiques
3. Visualisez l'impact sur le planning
4. Comparez avec la situation actuelle

### Baselines (Lignes de Base)

Sauvegardez des instantanés pour comparaison future :

1. À une date clé, sauvegardez une baseline
2. Plus tard, comparez l'état actuel avec la baseline
3. Identifiez les dérives de planning

---

## Exports Disponibles

### Export CSV

| Colonne | Description |
|---------|-------------|
| Type | Bon de Travail / Opération |
| Numéro | Numéro du BT |
| Description | Client - Projet |
| Statut | Statut actuel |
| Priorité | Niveau de priorité |
| Date Début | Date de début |
| Date Fin | Date de fin |
| Progression (%) | Avancement |
| Jours Restants | Délai restant |
| Montant | Montant total |

### Export Excel

Fichier avec 2 feuilles :

1. **Vue Ensemble** : Tous les BT avec métriques
2. **Opérations** : Détail des opérations par BT

### Export PDF

Rapport formaté avec :
- Titre et date du rapport
- Tableau des BT avec colonnes clés
- Mise en forme professionnelle (ReportLab)

### Export MS Project XML

Format compatible Microsoft Project :
- Structure XML standard MS Project
- Tâches avec UID, ID, Nom
- Dates de début et fin
- Pourcentage de complétion

---

## KPIs Dashboard

Métriques calculées automatiquement :

| KPI | Description |
|-----|-------------|
| **Total BTs** | Nombre total de bons de travail |
| **En cours** | BTs au statut EN_COURS |
| **Terminés** | BTs complétés |
| **En retard** | BTs dépassant l'échéance |
| **Progression moyenne** | Moyenne des avancements |
| **BTs critiques** | Nombre sur chemin critique |
| **Taux de complétion** | % de BTs terminés |
| **Taux de retard** | % de BTs en retard |

---

## Guide Pas-à-Pas

### Consulter le Gantt des Bons de Travail

1. Accédez au module **"📈 Gantt"**
2. Cliquez sur l'onglet **"📋 Bons de Travail"**
3. Le diagramme affiche tous les BTs avec leurs opérations
4. Survolez une barre pour voir les détails
5. Utilisez le range slider pour naviguer

### Filtrer les données

1. Développez le panneau **"🔍 Filtres et Options"**
2. Sélectionnez le statut souhaité
3. Choisissez un niveau de priorité
4. Activez/désactivez l'affichage des postes
5. Le Gantt se met à jour automatiquement

### Exporter les données

1. Dans le panneau des options avancées
2. Cliquez sur **"📥 Export CSV"**, **"📊 Export Excel"** ou **"📄 Export PDF"**
3. Le fichier est téléchargé automatiquement
4. Ouvrez avec l'application appropriée

### Identifier le chemin critique

1. Le système identifie automatiquement les BTs critiques
2. Les BTs avec marge ≤ 2 jours sont signalés
3. Consultez le niveau de risque (ÉLEVÉ/MOYEN/FAIBLE)
4. Priorisez les actions sur les BTs critiques

### Créer une baseline

1. Section **"📊 Baselines"**
2. Entrez un nom descriptif (ex: "Planning initial v1")
3. Cliquez sur **"💾 Sauvegarder baseline"**
4. La baseline est enregistrée pour comparaison future

### Voir les statistiques de productivité

1. Section **"📊 Productivité"**
2. Consultez les stats par employé :
   - Nombre d'opérations
   - Temps estimé vs réel
   - Écart
3. Consultez les stats par département

---

## Système de Cache

| Donnée | TTL | Description |
|--------|-----|-------------|
| Bons de travail | 2 minutes | Données dynamiques |
| Projets | 2 minutes | Données dynamiques |
| Opérations | 5 minutes | Moins fréquemment modifiées |

### Invalidation

```python
def invalidate_gantt_cache():
    """Invalide tout le cache Gantt."""
    _get_bons_travail_gantt_cached.clear()
    _get_projets_gantt_cached.clear()
    _get_operations_gantt_cached.clear()
```

---

## Tables Supplémentaires

Le module Gantt utilise des tables spécifiques :

| Table | Description |
|-------|-------------|
| `gantt_templates` | Templates de projets |
| `gantt_filter_presets` | Presets de filtres |
| `gantt_baselines` | Lignes de base |
| `gantt_audit_log` | Historique des modifications |
| `bt_dependencies` | Dépendances entre BT |
| `bt_comments` | Commentaires sur BT |
| `whatif_scenarios` | Scénarios What-If |
| `user_color_schemes` | Thèmes personnalisés |

---

## Astuces et Bonnes Pratiques

- **Utilisez les filtres** : Affichez uniquement les BTs pertinents
- **Surveillez le chemin critique** : Priorisez les éléments à risque
- **Sauvegardez des baselines** : Comparez l'évolution du planning
- **Exportez régulièrement** : Gardez un historique des plannings
- **Thème sombre** : Réduit la fatigue oculaire pour les longues sessions
- **Zoom adapté** : Utilisez le zoom Jour pour le court terme, Année pour la vue d'ensemble

---

## Résolution de Problèmes

### Le Gantt est vide

- **Cause** : Aucun BT avec des dates définies
- **Solution** : Créez des BTs avec date_creation et date_echeance

### Les opérations ne s'affichent pas

- **Cause** : Option "Afficher postes" désactivée ou pas d'opérations
- **Solution** : Activez l'option et vérifiez que le BT a des opérations

### Les couleurs ne correspondent pas

- **Cause** : Statut non reconnu
- **Solution** : Vérifiez que les statuts sont dans la liste autorisée

### L'export PDF échoue

- **Cause** : Bibliothèque ReportLab non installée
- **Solution** : Contactez l'administrateur système

---

## Questions Fréquentes (FAQ)

**Q: Puis-je modifier les dates directement dans le Gantt ?**
R: Non, les modifications se font dans les modules respectifs (BT, Projets, Devis). Le Gantt est une vue de lecture.

**Q: Comment créer des dépendances entre tâches ?**
R: Utilisez la fonction de dépendances pour lier des BTs (Fin-Début, etc.).

**Q: Le Gantt montre-t-il les employés assignés ?**
R: Oui, dans les détails du BT et les statistiques de productivité.

**Q: Puis-je imprimer le Gantt ?**
R: Utilisez l'export PDF ou faites une capture d'écran. Le range slider permet de cadrer la période souhaitée.

**Q: Les baselines sont-elles limitées en nombre ?**
R: Non, vous pouvez créer autant de baselines que nécessaire.

**Q: Comment voir uniquement mes BTs ?**
R: Utilisez le filtre de recherche avec votre identifiant ou demandez un filtre par responsable.

---

## Données Techniques

### Requête BTs avec Opérations

```sql
SELECT f.*,
       c.nom as company_nom,
       p.nom_projet,
       e.prenom || ' ' || e.nom as employee_nom
FROM formulaires f
LEFT JOIN companies c ON f.company_id = c.id
LEFT JOIN projects p ON f.project_id = p.id
LEFT JOIN employees e ON f.employee_id = e.id
WHERE f.type_formulaire = 'BON_TRAVAIL'
ORDER BY f.id DESC
```

### Requête Opérations

```sql
SELECT o.*,
       wc.nom as work_center_name,
       wc.departement as work_center_departement,
       wc.capacite_theorique,
       wc.cout_horaire
FROM operations o
LEFT JOIN work_centers wc ON o.work_center_id = wc.id
WHERE o.formulaire_bt_id = :bt_id
ORDER BY o.sequence_number, o.id
```

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Gestion des projets
- [📅 Calendrier](08-calendrier.md) - Vue mensuelle
- [🔄 Kanban](10-kanban.md) - Vue par statut
- [🏭 Production](11-production.md) - Bons de travail
- [⏱️ TimeTracker](13-timetracker.md) - Suivi des heures
- [🧾 Devis](06-devis.md) - Soumissions
