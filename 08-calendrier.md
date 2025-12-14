# 📅 Calendrier

## Introduction

Le module **Calendrier** offre une vue mensuelle intuitive de tous vos événements liés aux projets et devis. Il affiche automatiquement les dates de début, dates de fin prévues, et les échéances importantes pour une planification efficace de vos activités de construction.

Ce module est entièrement natif Streamlit et interconnecté avec les modules Projets et Devis pour une synchronisation automatique des événements.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📅 Calendrier"**
2. Le calendrier du mois courant s'affiche
3. Naviguez entre les mois avec les sélecteurs
4. Cliquez sur un jour pour voir les détails des événements

---

## Types d'Événements

### Événements Projets

| Type | Icône | Couleur | Description |
|------|-------|---------|-------------|
| **Début** | 🚀 | Bleu `#3b82f6` | Date de début d'un projet |
| **Fin Prévue** | 🏁 | Vert `#10b981` | Date de fin prévue |
| **En cours** | 📊 | Jaune `#eab308` | Projet actif |

### Événements Devis

| Type | Icône | Couleur | Description |
|------|-------|---------|-------------|
| **Création Devis** | 📝 | Bleu `#3b82f6` | Date de création |
| **Début Devis** | 📋 | Bleu `#3b82f6` | Date de début prévue |
| **Échéance Devis** | ⏰ | Jaune `#eab308` | Date limite de réponse |

---

## Interface Utilisateur

### Vue Calendrier Mensuelle

Le calendrier affiche une grille de 7 colonnes (Lun-Dim) avec :

| Élément | Description |
|---------|-------------|
| **En-têtes** | Lun, Mar, Mer, Jeu, Ven, Sam, Dim |
| **Cellules jour** | Numéro + badges d'événements |
| **Aujourd'hui** | Fond bleu clair `#E5F2FF` |
| **Jours standards** | Fond gris clair `#F9FAFB` |
| **Badge événement** | Nom tronqué à 15 caractères |
| **Indicateur +N** | Si plus de 3 événements par jour |

### Mois en Français

| # | Mois |
|---|------|
| 1 | Janvier |
| 2 | Février |
| 3 | Mars |
| 4 | Avril |
| 5 | Mai |
| 6 | Juin |
| 7 | Juillet |
| 8 | Août |
| 9 | Septembre |
| 10 | Octobre |
| 11 | Novembre |
| 12 | Décembre |

### Jours de la Semaine en Français

| Code | Nom Complet |
|------|-------------|
| Lun | Lundi |
| Mar | Mardi |
| Mer | Mercredi |
| Jeu | Jeudi |
| Ven | Vendredi |
| Sam | Samedi |
| Dim | Dimanche |

---

## Fonctionnalités Principales

### 1. Navigation Temporelle

| Contrôle | Description |
|----------|-------------|
| **Sélecteur Mois** | Liste déroulante des 12 mois |
| **Sélecteur Année** | Champ numérique (2020-2030) |
| **Bouton Aujourd'hui** | Retour au mois courant |

### 2. Affichage des Événements

Pour chaque jour avec des événements :
- Maximum 3 événements visibles directement
- Indicateur "+N autres" si plus de 3 événements
- Badges colorés selon le type d'événement
- Nom du projet/devis tronqué si trop long

### 3. Détails du Jour Sélectionné

Lorsqu'un jour est sélectionné :

**Journée Libre (aucun événement) :**
- Affichage d'un message "🌟 Journée Libre"
- Suggestions d'activités :
  - Maintenance préventive des équipements
  - Révision des indicateurs de performance
  - Formation du personnel
  - Planification des projets futurs
  - Audit qualité des processus
  - Prospection commerciale

**Journée avec Événements :**
- Compteur d'événements planifiés
- Liste détaillée de chaque événement avec :
  - ID et type
  - Nom du projet/devis
  - Statut actuel
  - Informations client
  - Dates associées
  - Boutons d'action

### 4. Actions sur les Événements

| Action | Description |
|--------|-------------|
| **🔍 Détails Complets** | Ouvre la fiche complète du projet |
| **📊 Analyse Projet** | (En développement) Analyse des métriques |

---

## Guide Pas-à-Pas

### Consulter le calendrier

1. Accédez au module **"📅 Calendrier"**
2. Par défaut, le mois courant est affiché
3. Repérez les badges colorés sur les jours avec événements
4. La date d'aujourd'hui est surlignée en bleu

### Naviguer entre les mois

1. Utilisez le **sélecteur de mois** (liste déroulante)
2. Ou modifiez l'**année** dans le champ numérique
3. Cliquez sur **"📅 Aujourd'hui"** pour revenir au mois courant
4. Le calendrier se rafraîchit automatiquement

### Voir les événements d'un jour

1. En bas du calendrier, section **"Événements du Jour"**
2. Utilisez le **sélecteur de date** pour choisir un jour
3. Les événements du jour s'affichent en détail
4. Cliquez sur un événement pour développer ses informations

### Accéder aux détails d'un projet

1. Sélectionnez un jour avec des événements
2. Développez l'événement souhaité
3. Cliquez sur **"🔍 Détails Complets"**
4. La fiche projet s'affiche avec toutes les informations
5. Cliquez sur **"✖️ Fermer les Détails"** pour revenir

---

## Informations Affichées par Type

### Événements Projets

| Champ | Description |
|-------|-------------|
| **🆔 ID Projet** | Identifiant unique |
| **🏷️ Type** | Début / Fin Prévue |
| **📝 Projet** | Nom du projet |
| **🎯 Tâche** | Tâche associée (si définie) |
| **🚦 Statut** | Statut actuel |
| **⚡ Priorité** | Niveau de priorité |
| **👤 Client** | Nom du client |
| **💰 Prix estimé** | Montant en $ |
| **📅 Dates** | Début et fin prévue |

### Événements Devis

| Champ | Description |
|-------|-------------|
| **🆔 ID Devis** | Identifiant unique |
| **🏷️ Type** | Création / Échéance |
| **📝 Devis** | Numéro ou nom du devis |
| **🚦 Statut** | Statut actuel |
| **👤 Client** | Nom du client |
| **💰 Montant** | Montant total en $ |
| **📅 Création** | Date de création |
| **⏰ Échéance** | Date limite |

---

## Sources de Données

### Projets

Les événements projets sont extraits de la table `projects` :

```sql
-- Événement Début
SELECT id, nom_projet, statut, priorite,
       date_soumis as date_debut,
       date_prevu as date_fin,
       client_nom_cache
FROM projects
WHERE date_soumis BETWEEN :start_date AND :end_date

-- Événement Fin Prévue
SELECT id, nom_projet, statut, priorite,
       date_prevu as date_fin
FROM projects
WHERE date_prevu BETWEEN :start_date AND :end_date
```

### Devis

Les événements devis sont extraits via le `GestionnaireDevis` :

| Champ Source | Événement |
|--------------|-----------|
| `date_creation` | Création Devis |
| `metadonnees_json.date_debut` | Début Devis |
| `date_echeance` | Échéance Devis |

---

## Algorithme de Récupération

### Fonction `get_events_for_month()`

```python
def get_events_for_month(year, month, gestionnaire, gestionnaire_devis):
    """Récupère tous les événements pour le mois donné."""
    events = {}

    # Plage de dates du mois
    start_date = date(year, month, 1)
    end_date = date(year, month, dernier_jour)

    # PROJETS
    for projet in gestionnaire.projets:
        # Date de début
        if start_date <= date_debut <= end_date:
            events[date_debut].append({
                'type': 'Début',
                'source': 'projet',
                ...
            })

        # Date de fin prévue
        if start_date <= date_fin <= end_date:
            events[date_fin].append({
                'type': 'Fin Prévue',
                'source': 'projet',
                ...
            })

    # DEVIS
    for devis in gestionnaire_devis.get_all_devis():
        # Création, Début, Échéance
        ...

    return events
```

---

## Palette de Couleurs

### Couleurs des Badges (standardisées ERP)

| Type | Couleur | Hex |
|------|---------|-----|
| **Projet Début** | Bleu | `#3b82f6` |
| **Projet Fin** | Vert | `#10b981` |
| **Projet En cours** | Jaune | `#eab308` |
| **Devis Création** | Bleu | `#3b82f6` |
| **Devis Échéance** | Rouge | `#dc2626` |
| **Autre** | Gris | `#6b7280` |

### Variables CSS (style.css ERP)

| Variable | Usage |
|----------|-------|
| `--lustrous-primary` | En-tête jour sélectionné |
| `--lustrous-success` | Journée libre |
| `--lustrous-blue` | Journée avec événements |
| `--lustrous-error` | Fin prévue |
| `--lustrous-warning` | En cours |

---

## Astuces et Bonnes Pratiques

- **Vérifiez les échéances** : Les devis avec échéance proche apparaissent en rouge
- **Planifiez à l'avance** : Utilisez le calendrier pour anticiper les périodes chargées
- **Consultez régulièrement** : Vérifiez les événements de la semaine à venir
- **Assignez des dates** : Les projets/devis sans dates n'apparaissent pas au calendrier
- **Utilisez Aujourd'hui** : Retournez rapidement au mois courant
- **Cliquez sur les badges** : Pour accéder aux détails rapidement

---

## Résolution de Problèmes

### Un projet n'apparaît pas au calendrier

- **Cause** : Le projet n'a pas de date de début ou de fin définie
- **Solution** : Modifiez le projet pour ajouter les dates

### Les événements d'un devis ne s'affichent pas

- **Cause** : Les dates ne sont pas renseignées dans les métadonnées
- **Solution** : Éditez le devis et ajoutez les dates requises

### Le calendrier est vide

- **Cause** : Aucun projet/devis avec des dates dans ce mois
- **Solution** : Naviguez vers un mois avec des données ou créez des projets

### Les détails ne s'affichent pas au clic

- **Cause** : Le projet référencé n'existe plus
- **Solution** : Le message d'erreur indique l'ID du projet manquant

---

## Questions Fréquentes (FAQ)

**Q: Puis-je créer des événements directement dans le calendrier ?**
R: Non, les événements sont automatiquement générés à partir des projets et devis. Créez ou modifiez un projet/devis pour ajouter des événements.

**Q: Comment voir les événements d'une semaine entière ?**
R: Actuellement, seule la vue mensuelle est disponible. Utilisez la vue Gantt pour une visualisation par semaine.

**Q: Les week-ends sont-ils différenciés ?**
R: Visuellement non, mais tous les jours sont affichés de la même manière. Les week-ends apparaissent dans la grille.

**Q: Puis-je synchroniser avec mon calendrier externe (Google, Outlook) ?**
R: Cette fonctionnalité n'est pas disponible actuellement. Exportez vos événements depuis le Gantt pour les importer manuellement.

**Q: Comment voir les activités CRM (appels, réunions) ?**
R: Les activités CRM s'affichent dans le module Ventes (Opportunités) et leurs calendriers associés.

**Q: Le calendrier affiche-t-il les congés et jours fériés ?**
R: Non, seuls les événements liés aux projets et devis sont affichés. Les congés se gèrent dans le module Employés.

---

## Données Techniques

### Session State Variables

| Variable | Description |
|----------|-------------|
| `selected_date` | Date actuellement sélectionnée |
| `view_month` | Mois affiché (1-12) |
| `view_year` | Année affichée |
| `show_project_details` | Afficher/masquer les détails |
| `selected_project_id` | ID du projet sélectionné |
| `selected_project` | Données du projet sélectionné |

### Dépendances

| Module | Usage |
|--------|-------|
| `erp_database.ERPDatabase` | Accès PostgreSQL |
| `app.GestionnaireProjetSQL` | Gestion des projets |
| `devis.GestionnaireDevis` | Gestion des devis |
| `calendar` (Python) | Génération du calendrier |
| `datetime` (Python) | Manipulation des dates |

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Gestion des projets
- [🧾 Devis](06-devis.md) - Gestion des devis
- [📈 Gantt](09-gantt.md) - Vue chronologique détaillée
- [🔄 Kanban](10-kanban.md) - Vue par statut
- [🤝 Ventes](05-ventes.md) - Opportunités et activités CRM
