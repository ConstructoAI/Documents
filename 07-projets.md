# 📋 Projets

## Introduction

Le module **Projets** est le centre névralgique de la gestion de vos chantiers de construction. Il vous permet de créer, suivre et gérer tous vos projets de la planification à la livraison finale, avec un suivi détaillé des tâches, des budgets, de l'avancement et des heures travaillées.

Ce module est intégré avec le CRM (conversion des opportunités gagnées), les Devis (création automatique après approbation), le TimeTracker (suivi des heures), et la Production (bons de travail).

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📋 Projets"**
2. La liste de vos projets s'affiche sous forme de tableau
3. Créez un nouveau projet ou consultez les projets existants
4. Naviguez entre les vues : Liste, Kanban, Gantt, Calendrier

---

## Structure des Données

### Table PostgreSQL : `projects`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique auto-généré |
| `nom_projet` | TEXT | Nom du projet |
| `description` | TEXT | Description détaillée |
| `client_company_id` | INTEGER | FK vers `companies.id` |
| `contact_id` | INTEGER | FK vers `contacts.id` |
| `adresse` | TEXT | Adresse du chantier |
| `ville` | TEXT | Ville du chantier |
| `code_postal` | TEXT | Code postal |
| `statut` | TEXT | Statut du projet |
| `priorite` | TEXT | Priorité (BASSE, NORMALE, HAUTE, CRITIQUE) |
| `type_projet` | TEXT | Type de projet |
| `date_debut` | DATE | Date de début réelle |
| `date_fin_prevue` | DATE | Date de fin prévue |
| `date_fin_reel` | DATE | Date de fin réelle |
| `date_soumis` | DATE | Date de soumission |
| `budget_total` | DECIMAL | Budget total alloué |
| `prix_estime` | DECIMAL | Prix estimé |
| `bd_ft_estime` | DECIMAL | Board-foot estimé (bois) |
| `pourcentage_avancement` | INTEGER | Progression (0-100%) |
| `tache` | TEXT | Tâche principale |
| `assigned_to` | INTEGER | FK vers `employees.id` |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

---

## Statuts des Projets

| Statut | Code | Couleur | Description |
|--------|------|---------|-------------|
| **En attente** | `EN_ATTENTE` | 🔵 Bleu `#3b82f6` | Projet planifié, non démarré |
| **À faire** | `A_FAIRE` | 🟠 Orange `#f59e0b` | Prêt à démarrer |
| **En cours** | `EN_COURS` | 🟡 Jaune `#eab308` | Travaux en cours |
| **En pause** | `EN_PAUSE` | 🔵 Bleu `#3b82f6` | Temporairement suspendu |
| **Terminé** | `TERMINÉ` | 🟢 Vert `#10b981` | Travaux complétés |
| **Annulé** | `ANNULÉ` | ⚫ Noir `#1f2937` | Projet abandonné |

---

## Niveaux de Priorité

| Priorité | Couleur | Usage |
|----------|---------|-------|
| **BASSE** | Vert | Projets non urgents |
| **NORMALE** | Bleu | Projets standard |
| **HAUTE** | Orange | Projets prioritaires |
| **CRITIQUE** | Rouge | Projets urgents (seuil > 50 000 $) |

---

## Fonctionnalités Principales

### 1. Gestion des Projets (CRUD)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouveau projet | Créer un projet manuellement |
| **Lire** | 👁️ Voir | Consulter les détails complets |
| **Modifier** | ✏️ Modifier | Éditer les informations |
| **Supprimer** | 🗑️ Supprimer | Archiver ou supprimer |

### 2. Informations du Projet

| Champ | Obligatoire | Description |
|-------|-------------|-------------|
| **Nom du projet** | Oui | Identification claire |
| **Client** | Non | Entreprise ou particulier |
| **Adresse chantier** | Non | Localisation des travaux |
| **Budget** | Non | Montant estimé total |
| **Prix estimé** | Non | Prix de vente estimé |
| **Dates** | Non | Début et fin prévue |
| **Responsable** | Non | Chef de projet assigné |
| **Priorité** | Oui | BASSE/NORMALE/HAUTE/CRITIQUE |

### 3. Les 16 Phases de Production

Les projets peuvent inclure jusqu'à 16 phases de construction standard :

| # | Phase | Description |
|---|-------|-------------|
| 1 | **Planification et permis** | Obtention des autorisations |
| 2 | **Préparation du site** | Nettoyage et préparation du terrain |
| 3 | **Démolition** | Travaux de démolition si requis |
| 4 | **Excavation** | Creusage et terrassement |
| 5 | **Béton et fondations** | Coulage des fondations |
| 6 | **Charpente** | Structure du bâtiment |
| 7 | **Toiture** | Installation du toit |
| 8 | **Isolation et étanchéité** | Performance thermique |
| 9 | **Électricité** | Installation électrique |
| 10 | **Plomberie** | Installation sanitaire |
| 11 | **CVC** | Chauffage, ventilation, climatisation |
| 12 | **Cloisons et finitions** | Gypse et finitions intérieures |
| 13 | **Revêtements de sol** | Planchers et carrelage |
| 14 | **Menuiserie** | Portes, fenêtres, armoires |
| 15 | **Finitions extérieures** | Aménagement paysager |
| 16 | **Nettoyage et livraison** | Remise des clés au client |

### 4. Conversion Automatique depuis Opportunités

Lorsqu'une opportunité passe au statut "Gagné" :

1. Le système crée automatiquement un projet avec :
   - Nom = Nom de l'opportunité
   - Client = Entreprise de l'opportunité
   - Contact = Contact de l'opportunité
   - Prix estimé = Montant de l'opportunité
   - Statut = "À FAIRE"
   - Priorité = HAUTE si montant > 50 000 $

2. Tâches initiales créées :
   - Réunion de lancement (3 jours)
   - Définir les jalons (5 jours)
   - Constituer l'équipe (2 jours)
   - Plan de projet détaillé (7 jours)

### 5. Conversion depuis Devis Approuvé

Un devis approuvé peut être converti en projet :

1. Ouvrez le devis au statut "APPROUVÉ"
2. Cliquez sur **"🚀 Créer le projet"**
3. Le projet est créé avec :
   - Informations client transférées
   - Montant total = Total TTC du devis
   - Lignes de devis converties en tâches
   - Lien vers le devis source conservé

### 6. Suivi de l'Avancement

| Indicateur | Description |
|------------|-------------|
| **Pourcentage global** | Moyenne pondérée des tâches |
| **Avancement par tâche** | 0-100% par phase |
| **Alertes de retard** | Tâches dépassant la date prévue |
| **Temps pointé** | Heures TimeTracker associées |
| **Écart budget** | Différence prévu vs réel |

---

## Vues Disponibles

| Vue | Description | Accès |
|-----|-------------|-------|
| **Liste** | Tableau avec tous les projets et filtres | Module Projets |
| **Kanban** | Cartes par statut avec glisser-déposer | 🔄 Kanban |
| **Gantt** | Chronologie des tâches et dépendances | 📈 Gantt |
| **Calendrier** | Vue mensuelle des échéances | 📅 Calendrier |

---

## Guide Pas-à-Pas

### Créer un nouveau projet

1. Cliquez sur **"➕ Nouveau projet"**
2. Remplissez le formulaire :

   **Section Identification :**
   - Nom du projet (obligatoire)
   - Description détaillée
   - Type de projet

   **Section Client :**
   - Sélectionnez l'entreprise cliente
   - Sélectionnez le contact principal

   **Section Chantier :**
   - Adresse complète
   - Ville et code postal

   **Section Financier :**
   - Budget total alloué
   - Prix estimé de vente
   - BD-FT estimé (si applicable)

   **Section Planification :**
   - Date de début prévue
   - Date de fin prévue
   - Responsable assigné
   - Priorité

3. Cliquez sur **"💾 Enregistrer"**
4. Le projet est créé avec le statut "EN_ATTENTE"

### Ajouter des tâches au projet

1. Ouvrez le projet concerné
2. Cliquez sur l'onglet **"📋 Tâches"**
3. Cliquez sur **"➕ Ajouter des tâches"**
4. Sélectionnez les phases de production applicables
5. Pour chaque tâche, définissez :
   - Date de début prévue
   - Durée estimée (jours ou heures)
   - Responsable assigné
   - Dépendances (tâches préalables)
6. Les tâches apparaissent dans le Gantt

### Mettre à jour l'avancement

1. Ouvrez le projet
2. Onglet **"📊 Avancement"**
3. Pour chaque tâche, entrez le pourcentage (0-100%)
4. Ajoutez des notes de suivi si nécessaire
5. L'avancement global se recalcule automatiquement
6. Les alertes de retard s'affichent si applicable

### Suivre les heures (TimeTracker)

1. Ouvrez le projet
2. Onglet **"⏱️ Heures"**
3. Consultez les pointages des employés :
   - Heures par poste de travail
   - Coût total des heures
   - Comparaison estimé vs réel
4. Les données proviennent du TimeTracker

### Gérer les documents du projet

1. Dans la fiche projet, onglet **"📁 Documents"**
2. Cliquez sur **"📤 Télécharger un document"**
3. Sélectionnez le fichier (PDF, images, etc.)
4. Catégorisez le document :
   - Plans et dessins
   - Permis et autorisations
   - Photos de chantier
   - Contrats et avenants
   - Rapports et notes
5. Le document est accessible à toute l'équipe

### Clôturer un projet

1. Vérifiez que toutes les tâches sont à 100%
2. Mettez à jour la date de fin réelle
3. Passez le statut à **"TERMINÉ"**
4. Générez le **rapport de clôture** (optionnel)
5. Archivez les documents importants
6. Le projet reste consultable dans l'historique

---

## Système de Cache

| Donnée | TTL | Raison |
|--------|-----|--------|
| Liste projets | 10 minutes | Données relativement stables |
| Détails projet | 5 minutes | Accès fréquent |
| Tâches projet | 5 minutes | Données dynamiques |
| Stats avancement | 2 minutes | Calculs fréquents |

---

## Interconnexions

### Modules Connectés

| Module | Type de liaison | Description |
|--------|-----------------|-------------|
| **Entreprises** | N-1 | `client_company_id` |
| **Contacts** | N-1 | `contact_id` |
| **Opportunités** | 1-1 | Conversion depuis CRM |
| **Devis** | 1-N | `project_id` dans devis |
| **Bons de Travail** | 1-N | `project_id` dans formulaires |
| **TimeTracker** | 1-N | Pointages sur projet |
| **Documents** | 1-N | `project_id` dans documents |

### Données Dérivées

```sql
-- Calcul avancement global
SELECT AVG(avancement) as avancement_global
FROM project_tasks
WHERE project_id = ?

-- Heures pointées sur projet
SELECT SUM(duree_heures) as total_heures,
       SUM(cout_total) as cout_total
FROM timetracker_entries
WHERE project_id = ?

-- Bons de travail liés
SELECT COUNT(*) as nb_bts
FROM formulaires
WHERE project_id = ? AND type_formulaire = 'BON_TRAVAIL'
```

---

## Astuces et Bonnes Pratiques

- **Planifiez d'abord** : Un projet bien planifié est à moitié réussi
- **Mettez à jour régulièrement** : L'avancement doit refléter la réalité du chantier
- **Documentez les changements** : Notez toutes les modifications au scope
- **Utilisez le Gantt** : Visualisez les dépendances entre tâches
- **Configurez les alertes** : Activez les notifications de retard
- **Liez tout au projet** : Devis, bons de travail, documents
- **Assignez un responsable** : Chaque projet doit avoir un propriétaire

---

## Résolution de Problèmes

### Le projet n'apparaît pas après création

- **Cause** : Cache non invalidé
- **Solution** : Rafraîchissez la page (F5)

### Impossible de créer un projet depuis une opportunité

- **Cause** : L'opportunité n'est pas au statut "Gagné"
- **Solution** : Changez d'abord le statut en "Gagné"

### L'avancement ne se met pas à jour

- **Cause** : Les tâches n'ont pas de pourcentage défini
- **Solution** : Mettez à jour l'avancement de chaque tâche

### Les heures TimeTracker ne s'affichent pas

- **Cause** : Les pointages ne sont pas liés au projet
- **Solution** : Vérifiez que les employés sélectionnent le bon projet lors du pointage

---

## Questions Fréquentes (FAQ)

**Q: Comment créer un projet depuis un devis approuvé ?**
R: Ouvrez le devis approuvé et cliquez sur "🚀 Créer le projet". Les informations client, montants et lignes de devis sont transférées automatiquement.

**Q: Puis-je avoir des sous-projets ?**
R: Pas directement, mais vous pouvez créer des tâches hiérarchiques avec des sous-tâches pour structurer des projets complexes.

**Q: Comment voir la charge de travail de mon équipe ?**
R: Consultez le module Production ou le Gantt pour voir les assignations et la disponibilité des ressources.

**Q: Les notifications de retard sont-elles automatiques ?**
R: Oui, si une tâche dépasse sa date de fin prévue sans être marquée complète, une alerte apparaît sur le tableau de bord.

**Q: Comment réactiver un projet annulé ?**
R: Modifiez le projet et changez le statut de "ANNULÉ" vers "EN_ATTENTE" ou "EN_COURS".

**Q: Comment calculer la rentabilité d'un projet ?**
R: Consultez le module Analytics BI > onglet "Projets/Production" pour voir les marges et la rentabilité par projet.

---

## Données Techniques

### Requête SQL de Récupération

```sql
SELECT p.*,
       c.nom as client_nom,
       co.prenom || ' ' || co.nom_famille as contact_nom,
       e.prenom || ' ' || e.nom as responsable_nom,
       (SELECT COUNT(*) FROM project_tasks pt WHERE pt.project_id = p.id) as nb_taches,
       (SELECT SUM(duree_heures) FROM timetracker_entries tt WHERE tt.project_id = p.id) as heures_pointees
FROM projects p
LEFT JOIN companies c ON p.client_company_id = c.id
LEFT JOIN contacts co ON p.contact_id = co.id
LEFT JOIN employees e ON p.assigned_to = e.id
ORDER BY p.updated_at DESC
```

### Validation des Données

| Champ | Règle |
|-------|-------|
| nom_projet | Obligatoire, texte non vide |
| statut | Valeurs autorisées uniquement |
| priorite | BASSE, NORMALE, HAUTE, CRITIQUE |
| pourcentage_avancement | 0-100 |
| budget_total | Nombre positif (optionnel) |

---

## Voir Aussi

- [📈 Gantt](09-gantt.md) - Planification visuelle
- [🔄 Kanban](10-kanban.md) - Vue par statut
- [📅 Calendrier](08-calendrier.md) - Vue temporelle
- [🏭 Production](11-production.md) - Bons de travail
- [⏱️ TimeTracker](13-timetracker.md) - Suivi des heures
- [🤝 Ventes](05-ventes.md) - Opportunités (source des projets)
- [🧾 Devis](06-devis.md) - Soumissions (source des projets)
