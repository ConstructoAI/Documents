# ⏱️ TimeTracker

## Introduction

Le module **TimeTracker** est le système de pointage unifié spécialisé pour la construction au Québec. Il permet de suivre le temps par employé, par projet, par opération et par poste de travail avec une précision adaptée aux exigences des chantiers de construction. Le système gère le punch in/out sur les opérations des bons de travail avec réinitialisation automatique après chaque pointage.

Ce module de plus de 4000 lignes de code (155 KB) est intégré avec les employés, les projets, les opérations et les postes de travail pour un suivi complet de la productivité.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"⏱️ TimeTracker"**
2. Le tableau de bord de pointage s'affiche
3. Sélectionnez un employé et une opération
4. Pointez en temps réel ou consultez l'historique

---

## Structure des Données

### Table PostgreSQL : `timetracker_entries`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `employee_id` | INTEGER | FK vers `employees.id` |
| `project_id` | INTEGER | FK vers `projects.id` |
| `operation_id` | INTEGER | FK vers `operations.id` |
| `formulaire_bt_id` | INTEGER | FK vers `formulaires.id` (BT) |
| `punch_in` | TIMESTAMP | Heure d'entrée |
| `punch_out` | TIMESTAMP | Heure de sortie |
| `total_hours` | DECIMAL | Heures totales calculées |
| `hourly_rate` | DECIMAL | Taux horaire ($/h) |
| `total_cost` | DECIMAL | Coût total ($) |
| `notes` | TEXT | Notes de pointage |
| `work_center_id` | INTEGER | FK vers `work_centers.id` |
| `created_at` | TIMESTAMP | Date de création |

---

## Fonctionnement du Système de Pointage

### Flux de Pointage sur Opération

```
┌─────────────────┐
│ 1. Sélection    │
│    Employé      │
├─────────────────┤
│ 2. Sélection    │
│    Opération    │──► Hiérarchie: Projet > BT > Opération
├─────────────────┤
│ 3. PUNCH IN     │──► Création entrée avec timestamp
├─────────────────┤
│    TRAVAIL      │
│    EN COURS     │
├─────────────────┤
│ 4. PUNCH OUT    │──► Calcul durée et coût
└─────────────────┘
         │
         ▼
┌─────────────────┐
│ Réinitialisation│
│ automatique du  │
│ sélecteur       │
└─────────────────┘
```

### Processus Punch In

1. Vérification qu'aucun pointage actif n'existe
2. Récupération des informations de l'opération
3. Création de l'entrée avec :
   - `punch_in` = timestamp actuel
   - `employee_id`, `project_id`, `operation_id`
   - `work_center_id` associé
4. Mise à jour du statut opération si nécessaire

### Processus Punch Out

1. Récupération du pointage actif
2. Calcul de la durée :
   ```python
   total_seconds = (punch_out_time - punch_in_time).total_seconds()
   total_hours = total_seconds / 3600
   ```
3. Récupération du taux horaire employé
4. Calcul du coût :
   ```python
   total_cost = total_hours * hourly_rate
   ```
5. Mise à jour de la progression de l'opération

---

## Organisation Hiérarchique des Opérations

Le TimeTracker présente les opérations de manière hiérarchique :

### Structure d'Affichage

```
📁 PROJET: Résidence Dupont
├── 📋 BT: BT-001 Fondations
│   ├── 🔧 Coffrage fondations (8h)
│   ├── 🔧 Coulage béton (6h)
│   └── 🔧 Remblayage (4h)
└── 📋 BT: BT-002 Charpente
    ├── 🔧 Charpente murs (16h)
    └── 🔧 Charpente toit (12h)
```

### Requête SQL Hiérarchique

```sql
SELECT o.id, o.code_operation, o.description, o.statut,
       o.temps_estime, o.work_center_id, o.formulaire_bt_id,
       wc.nom as work_center_name,
       f.numero_document as bt_numero,
       f.statut as bt_statut,
       p.nom_projet
FROM operations o
LEFT JOIN work_centers wc ON o.work_center_id = wc.id
LEFT JOIN formulaires f ON o.formulaire_bt_id = f.id
LEFT JOIN projects p ON o.project_id = p.id
WHERE o.statut IN ('À FAIRE', 'EN COURS')
ORDER BY p.nom_projet, f.numero_document, o.sequence_number
```

---

## Modes de Pointage

| Mode | Description | Usage |
|------|-------------|-------|
| **Sur opération** | Pointage sur une opération BT | Suivi production |
| **Sur projet** | Pointage global projet | Réunions, supervision |
| **Manuel** | Saisie après coup | Rattrapage, bureau |
| **Punch In/Out** | Temps réel | Chantier, atelier |

---

## Types d'Heures

| Type | Code | Majoration | Description |
|------|------|------------|-------------|
| **Régulières** | REG | 100% | Heures normales (40h/sem) |
| **Supplémentaires** | SUP | 150% | Au-delà de 40h |
| **Doubles** | DBL | 200% | Jours fériés, urgences |
| **Voyage** | VYG | Variable | Déplacements |

---

## Fonctionnalités Principales

### 1. Pointage en Temps Réel

| Action | Bouton | Description |
|--------|--------|-------------|
| **Punch In** | 🟢 Punch In | Démarrer le pointage |
| **Punch Out** | 🔴 Punch Out | Terminer le pointage |
| **Annuler** | ❌ Annuler | Annuler pointage actif |

### 2. Sélection d'Opération

L'interface affiche les opérations disponibles :
- Groupées par projet et BT
- Avec temps estimé
- Avec poste de travail associé
- Filtrage des statuts : À FAIRE, EN COURS

### 3. Suivi du Pointage Actif

Information affichée pendant le pointage :
- Nom de l'opération
- Projet et BT associés
- Poste de travail
- Heure de début
- Durée en cours (mise à jour dynamique)

### 4. Historique des Pointages

| Colonne | Description |
|---------|-------------|
| **Date** | Date du pointage |
| **Employé** | Nom de l'employé |
| **Projet** | Projet concerné |
| **Opération** | Description de l'opération |
| **Poste** | Work center |
| **Punch In** | Heure d'entrée |
| **Punch Out** | Heure de sortie |
| **Durée** | Heures totales |
| **Coût** | Montant calculé |

---

## Système de Cache

| Donnée | TTL | Fonction |
|--------|-----|----------|
| Employés actifs | 5 min | `_get_all_employees_cached()` |
| Projets actifs | 5 min | `_get_all_projects_cached()` |
| Opérations hiérarchiques | 5 min | `_get_operations_hierarchical_cached()` |
| Historique pointages | 2 min | `_get_punch_history_cached()` |
| Résumé journalier | 2 min | `_get_daily_summary_cached()` |
| Stats postes travail | 5 min | `_get_work_centers_stats_cached()` |
| Stats employé | 2 min | `_get_employee_statistics_cached()` |
| Résumé temps projet | 5 min | `_get_project_time_summary_cached()` |

### Invalidation du Cache

```python
def invalidate_timetracker_cache():
    _get_all_employees_cached.clear()
    _get_all_projects_cached.clear()
    _get_operations_hierarchical_cached.clear()
    _get_punch_history_cached.clear()
    _get_daily_summary_cached.clear()
    _get_work_centers_stats_cached.clear()
    _get_employee_statistics_cached.clear()
    _get_project_time_summary_cached.clear()
```

---

## Guide Pas-à-Pas

### Pointer sur une opération (Punch In)

1. Ouvrez **"⏱️ TimeTracker"**
2. Sélectionnez l'**employé** dans la liste
3. La liste des opérations disponibles s'affiche
4. Sélectionnez l'**opération** :
   - Les opérations sont groupées par projet et BT
   - Le temps estimé et le poste sont affichés
5. Cliquez sur **"🟢 Punch In"**
6. L'heure de début est enregistrée
7. Le pointage actif s'affiche avec chronomètre

### Terminer le pointage (Punch Out)

1. Le pointage actif est affiché
2. Vérifiez les informations (opération, durée)
3. Ajoutez des **notes** si nécessaire
4. Cliquez sur **"🔴 Punch Out"**
5. Le système calcule automatiquement :
   - Durée totale (heures)
   - Taux horaire de l'employé
   - Coût total
6. Le sélecteur d'employé est réinitialisé

### Saisir des heures manuellement

1. Cliquez sur **"➕ Nouvelle entrée"**
2. Sélectionnez l'**employé**
3. Choisissez la **date**
4. Entrez :
   - Heure de début (HH:MM)
   - Heure de fin (HH:MM)
   - OU durée totale (heures)
5. Sélectionnez le **projet**
6. Sélectionnez l'**opération** (si applicable)
7. Ajoutez des **notes** (optionnel)
8. Cliquez sur **"💾 Enregistrer"**

### Consulter l'historique

1. Section **"📜 Historique des pointages"**
2. Filtrez par :
   - **Employé** (liste déroulante)
   - **Période** (nombre de jours)
   - **Projet** (optionnel)
3. Le tableau affiche les entrées
4. Maximum 500 entrées affichées
5. Exportez en CSV si besoin

### Voir les statistiques par employé

1. Sélectionnez un employé
2. Section **"📊 Statistiques"**
3. Consultez :
   - Total heures (période)
   - Heures par projet
   - Heures par poste de travail
   - Coût total

### Consulter le résumé d'un projet

1. Section **"📋 Résumé Projet"**
2. Sélectionnez le projet
3. Le résumé affiche :
   - Total heures pointées
   - Nombre d'entrées
   - Coût total
   - Répartition par opération
   - Répartition par employé

---

## Statistiques et Rapports

### Résumé Journalier

```sql
SELECT
    SUM(total_hours) as total_heures,
    SUM(total_cost) as total_cout,
    COUNT(DISTINCT employee_id) as nb_employes
FROM timetracker_entries
WHERE DATE(punch_in) = :target_date
  AND punch_out IS NOT NULL
```

### Statistiques par Poste de Travail

```sql
SELECT
    wc.nom as poste,
    COUNT(*) as nb_pointages,
    SUM(te.total_hours) as total_heures,
    AVG(te.total_hours) as moyenne_heures
FROM timetracker_entries te
LEFT JOIN work_centers wc ON te.work_center_id = wc.id
WHERE te.punch_out IS NOT NULL AND wc.nom IS NOT NULL
GROUP BY wc.nom
ORDER BY total_heures DESC
```

### Statistiques Employé

```sql
SELECT
    SUM(total_hours) as total_heures,
    SUM(total_cost) as total_cout,
    COUNT(*) as nb_pointages,
    AVG(total_hours) as moyenne_par_pointage
FROM timetracker_entries
WHERE employee_id = :employee_id
  AND punch_in >= CURRENT_DATE - INTERVAL ':days days'
  AND punch_out IS NOT NULL
```

---

## Calcul des Heures et Coûts

### Formule de Calcul

```python
def calculate_punch_out(punch_in_time, punch_out_time, hourly_rate):
    # Calcul durée
    total_seconds = (punch_out_time - punch_in_time).total_seconds()
    total_hours = total_seconds / 3600

    # Calcul coût
    total_cost = total_hours * hourly_rate

    return {
        'total_hours': round(total_hours, 2),
        'total_cost': round(total_cost, 2)
    }
```

### Récupération du Taux Horaire

```python
def get_employee_hourly_rate(self, employee_id: int) -> float:
    result = self.db.execute_query("""
        SELECT salaire FROM employees WHERE id = %s
    """, (employee_id,))

    if result and result[0].get('salaire'):
        return float(result[0]['salaire'])
    return 35.0  # Taux par défaut
```

---

## Mise à Jour de la Progression

Après un punch out sur une opération, le système met à jour la progression :

```python
def update_operation_progress(self, operation_id: int, hours_worked: float):
    # Récupérer les heures déjà travaillées
    existing = self.db.execute_query("""
        SELECT SUM(total_hours) as total
        FROM timetracker_entries
        WHERE operation_id = %s AND punch_out IS NOT NULL
    """, (operation_id,))

    total_worked = existing[0]['total'] if existing else 0

    # Mettre à jour le statut si nécessaire
    self.db.execute_update("""
        UPDATE operations
        SET heures_reelles = %s,
            statut = CASE
                WHEN heures_reelles >= temps_estime THEN 'TERMINÉ'
                ELSE 'EN COURS'
            END
        WHERE id = %s
    """, (total_worked, operation_id))
```

---

## Astuces et Bonnes Pratiques

- **Pointez en temps réel** : Plus précis que la saisie a posteriori
- **Sélectionnez toujours une opération** : Permet le suivi de production
- **Vérifiez le poste de travail** : Essentiel pour les statistiques
- **Utilisez les notes** : Documentez les particularités
- **Validez chaque semaine** : Ne laissez pas s'accumuler
- **Surveillez les oublis** : Pointages actifs depuis longtemps

---

## Résolution de Problèmes

### L'employé ne peut pas pointer

- **Cause** : Pointage actif non clôturé
- **Solution** : Terminez le pointage actif avec Punch Out

### L'opération n'apparaît pas

- **Cause** : Statut TERMINÉ ou ANNULÉ
- **Solution** : Seules les opérations À FAIRE ou EN COURS sont listées

### Le calcul des heures semble incorrect

- **Cause** : Décalage horaire ou fuseau
- **Solution** : Vérifiez la configuration du serveur

### Les statistiques ne se mettent pas à jour

- **Cause** : Cache non invalidé
- **Solution** : Rafraîchissez la page ou attendez 2-5 minutes

### L'historique est incomplet

- **Cause** : Limite de 500 entrées atteinte
- **Solution** : Réduisez la période ou filtrez par employé

---

## Questions Fréquentes (FAQ)

**Q: Puis-je pointer depuis mon téléphone ?**
R: Oui, CONSTRUCTO AI est accessible sur mobile. Ouvrez l'application dans votre navigateur et pointez normalement. L'interface s'adapte aux petits écrans.

**Q: Comment corriger une erreur de pointage ?**
R: Les employés peuvent demander une modification via les notes. Un superviseur peut éditer l'entrée dans la section historique.

**Q: Les heures sont-elles automatiquement envoyées à la paie ?**
R: Les heures sont disponibles pour le module Comptabilité/Paie. L'intégration permet d'extraire les données validées.

**Q: Comment gérer les pauses ?**
R: Faites un Punch Out avant la pause et un Punch In après. Ou configurez une déduction automatique dans les paramètres.

**Q: Puis-je voir qui est actuellement au travail ?**
R: Oui, la section "Pointages actifs" affiche tous les employés actuellement punchés in avec leur opération en cours.

**Q: Comment attribuer un pointage à plusieurs projets ?**
R: Chaque pointage est associé à une seule opération/projet. Faites des pointages séparés pour différents projets.

---

## Données Techniques

### Requête Pointages Actifs

```sql
SELECT te.*, e.prenom, e.nom, p.nom_projet,
       o.description as operation_desc
FROM timetracker_entries te
JOIN employees e ON te.employee_id = e.id
LEFT JOIN projects p ON te.project_id = p.id
LEFT JOIN operations o ON te.operation_id = o.id
WHERE te.punch_out IS NULL
ORDER BY te.punch_in DESC
```

### Requête Historique

```sql
SELECT te.*, e.prenom || ' ' || e.nom as employe,
       p.nom_projet, o.description as operation,
       wc.nom as poste_travail
FROM timetracker_entries te
JOIN employees e ON te.employee_id = e.id
LEFT JOIN projects p ON te.project_id = p.id
LEFT JOIN operations o ON te.operation_id = o.id
LEFT JOIN work_centers wc ON te.work_center_id = wc.id
WHERE te.punch_in >= CURRENT_DATE - INTERVAL ':days days'
ORDER BY te.punch_in DESC
LIMIT 500
```

---

## Voir Aussi

- [👥 Employés](12-employes.md) - Gestion des employés et taux horaires
- [🏭 Production](11-production.md) - Bons de travail et opérations
- [📋 Projets](07-projets.md) - Suivi des projets
- [📈 Gantt](09-gantt.md) - Vue chronologique des opérations
- [💰 Comptabilité](19-comptabilite.md) - Intégration paie
