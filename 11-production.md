# 🏭 Production

## Introduction

Le module **Production** est le coeur opérationnel de l'ERP CONSTRUCTO AI pour la gestion de chantier. Il permet de créer des bons de travail (BT), d'assigner des ressources aux postes de travail spécialisés construction du Québec, et de suivre l'avancement de chaque opération en temps réel.

Ce module de plus de 5000 lignes de code est intégré avec les 61 postes de travail construction québécois, le TimeTracker pour le pointage, et le Gantt pour la planification visuelle.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🏭 Production"**
2. Le tableau de bord production s'affiche avec les statistiques
3. Gérez vos bons de travail et opérations
4. Consultez la charge des postes de travail

---

## Structure des Données

### Table PostgreSQL : `bons_travail` (BT)

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `numero_bt` | TEXT | Numéro formaté (BT-XXXX) |
| `titre` | TEXT | Description courte |
| `project_id` | INTEGER | FK vers `projects.id` |
| `client_id` | INTEGER | FK vers `companies.id` |
| `statut` | TEXT | Statut du BT |
| `priorite` | TEXT | Niveau de priorité |
| `date_debut` | DATE | Date de début prévue |
| `date_fin` | DATE | Date de fin prévue |
| `montant_total` | DECIMAL | Montant total estimé |
| `notes` | TEXT | Notes et commentaires |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Table PostgreSQL : `work_centers` (Postes de travail)

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom` | TEXT | Nom du poste |
| `departement` | TEXT | Département de rattachement |
| `categorie` | TEXT | Catégorie de poste |
| `capacite` | INTEGER | Capacité heures/jour |
| `cout_heure` | DECIMAL | Coût horaire ($) |
| `description` | TEXT | Description du poste |
| `statut` | TEXT | ACTIF/INACTIF |
| `type_poste` | TEXT | Type de poste |
| `localisation` | TEXT | Localisation chantier |

### Table PostgreSQL : `operations`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `project_id` | INTEGER | FK vers `projects.id` |
| `work_center_id` | INTEGER | FK vers `work_centers.id` |
| `formulaire_bt_id` | INTEGER | FK vers `formulaires.id` |
| `sequence_number` | INTEGER | Ordre d'exécution |
| `description` | TEXT | Description de l'opération |
| `temps_estime` | DECIMAL | Temps estimé (heures) |
| `ressource` | TEXT | Ressource assignée |
| `statut` | TEXT | Statut de l'opération |
| `notes` | TEXT | Notes de travail |

---

## Statuts des Bons de Travail

| Statut | Code | Couleur | Description |
|--------|------|---------|-------------|
| **BROUILLON** | `BROUILLON` | Gris `#4b5563` | BT en préparation |
| **VALIDÉ** | `VALIDÉ` | Bleu `#3b82f6` | BT validé, prêt à exécuter |
| **ENVOYÉ** | `ENVOYÉ` | Bleu `#3b82f6` | BT envoyé aux équipes |
| **APPROUVÉ** | `APPROUVÉ` | Vert `#10b981` | BT approuvé par le client |
| **EN_COURS** | `EN_COURS` | Jaune `#eab308` | Travaux en cours |
| **TERMINÉ** | `TERMINÉ` | Vert `#10b981` | BT complété |
| **ANNULÉ** | `ANNULÉ` | Noir `#1f2937` | BT annulé |

---

## Statuts des Opérations

| Statut | Couleur | Description |
|--------|---------|-------------|
| **À FAIRE** | Bleu `#3b82f6` | Non démarrée |
| **EN_COURS** | Jaune `#eab308` | Travaux en cours |
| **TERMINÉ** | Vert `#10b981` | Travail complété |
| **SUSPENDU** | Bleu `#3b82f6` | En pause temporaire |
| **ANNULÉ** | Noir `#1f2937` | Opération annulée |

### Conversion Automatique des Statuts

| Statut Tâche BT | Statut Opération |
|-----------------|------------------|
| `pending` | À FAIRE |
| `in-progress` | EN COURS |
| `completed` | TERMINÉ |
| `on-hold` | SUSPENDU |
| Autre | À FAIRE |

---

## Les 61 Postes de Travail Construction Québec

### Organisation par Département

| Département | Nb Postes | Exemples |
|-------------|-----------|----------|
| **TERRASSEMENT** | 2 | Excavation, Remblayage |
| **BÉTON** | 2 | Coffrage fondations, Coulage béton |
| **CHARPENTE** | 5 | Murs, Toit, Poutrelles, OSB, Terrasses |
| **TOITURE** | 4 | Pontage, Membrane, Bardeaux, Gouttières |
| **ISOLATION** | 3 | Murs, Pare-vapeur, Calfeutrage |
| **PLOMBERIE** | 3 | Brute, Finition, Test étanchéité |
| **ÉLECTRICITÉ** | 3 | Brute, Panneau, Finition |
| **CVC** | 3 | Ductwork, Chauffage, Ventilation |
| **REVÊTEMENT** | 1 | Vinyle/Aluminium |
| **MAÇONNERIE** | 2 | Brique parement, Pierre naturelle |
| **MENUISERIE** | 7 | Fenêtres, Portes, Armoires, Moulures, Clôtures |
| **GYPSE** | 3 | Murs, Plafonds, Joints |
| **PEINTURE** | 2 | Préparation, Finition |
| **PLANCHER** | 2 | Flottant, Tapis |
| **CÉRAMIQUE** | 1 | Pose carrelage |
| **NETTOYAGE** | 2 | Chantier, Final |
| **QUALITÉ** | 1 | Inspection finale |
| **PAVAGE** | 1 | Entrée asphalte |
| **PAYSAGE** | 1 | Aménagement paysager |

### Liste Complète des Postes

#### Terrassement et Fondations

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Excavation et terrassement | TERRASSEMENT | 8 | 45$ | Excavation, nivellement, préparation terrain |
| Coffrage fondations | BÉTON | 8 | 40$ | Coffrage murs et semelles de fondation |
| Coulage béton fondations | BÉTON | 6 | 42$ | Coulage et finition béton fondations |
| Remblayage | TERRASSEMENT | 8 | 35$ | Remblayage et compactage autour fondations |

#### Structure et Charpente

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Charpente murs | CHARPENTE | 8 | 38$ | Montage ossature bois 2x4, 2x6 |
| Charpente toit | CHARPENTE | 8 | 42$ | Fermes de toit, solives, chevrons |
| Pose poutrelles | CHARPENTE | 8 | 45$ | Installation poutrelles et solives |
| Planchers OSB | CHARPENTE | 8 | 35$ | Pose contreplaqué et OSB planchers |
| Terrasses balcons | CHARPENTE | 8 | 40$ | Construction terrasses et balcons |

#### Toiture

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Pontage toiture | TOITURE | 8 | 36$ | Pontage OSB ou contreplaqué toiture |
| Membrane toiture | TOITURE | 8 | 38$ | Installation membrane d'étanchéité |
| Pose bardeaux | TOITURE | 8 | 40$ | Installation bardeaux d'asphalte |
| Gouttières | TOITURE | 6 | 42$ | Installation système gouttières |

#### Isolation et Étanchéité

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Isolation murs | ISOLATION | 8 | 32$ | Pose isolant laine ou rigide |
| Pare-vapeur | ISOLATION | 6 | 28$ | Installation pare-vapeur polyéthylène |
| Calfeutrage | ISOLATION | 6 | 35$ | Calfeutrant fenêtres et joints |

#### Mécanique du Bâtiment

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Plomberie brute | PLOMBERIE | 8 | 55$ | Installation tuyauterie ABS, cuivre |
| Plomberie finition | PLOMBERIE | 8 | 58$ | Installation fixtures, robinetterie |
| Test étanchéité | PLOMBERIE | 4 | 55$ | Tests pression et étanchéité |
| Électricité brute | ÉLECTRICITÉ | 8 | 52$ | Câblage et boîtes électriques |
| Panneau électrique | ÉLECTRICITÉ | 6 | 65$ | Installation panneau et branchements |
| Électricité finition | ÉLECTRICITÉ | 8 | 50$ | Prises, interrupteurs, luminaires |
| Ductwork CVC | CVC | 8 | 48$ | Installation conduits ventilation |
| Unité chauffage | CVC | 6 | 55$ | Installation thermopompe, fournaise |
| Système ventilation | CVC | 8 | 45$ | VRC, ventilateurs, grilles |

#### Finition Intérieure

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Gypse murs | GYPSE | 8 | 32$ | Pose panneaux gypse murs |
| Gypse plafonds | GYPSE | 8 | 35$ | Pose panneaux gypse plafonds |
| Joints gypse | GYPSE | 8 | 30$ | Tirage joints et sablage |
| Peinture préparation | PEINTURE | 8 | 28$ | Préparation surfaces, primer |
| Peinture finition | PEINTURE | 8 | 32$ | Application peinture finie |
| Plancher flottant | PLANCHER | 8 | 35$ | Installation plancher laminé/vinyle |
| Céramique | CÉRAMIQUE | 8 | 45$ | Pose carrelage céramique |
| Tapis | PLANCHER | 6 | 30$ | Installation tapis et sous-tapis |

#### Menuiserie et Finition

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Fenêtres | MENUISERIE | 8 | 42$ | Installation fenêtres PVC, aluminium |
| Portes extérieures | MENUISERIE | 6 | 40$ | Installation portes d'entrée |
| Portes intérieures | MENUISERIE | 8 | 36$ | Installation portes intérieures |
| Armoires cuisine | MENUISERIE | 8 | 48$ | Installation armoires et comptoirs |
| Moulures finition | MENUISERIE | 8 | 38$ | Plinthes, cadrages, moulures |
| Quincaillerie | MENUISERIE | 4 | 35$ | Poignées, serrures, accessoires |
| Clôtures | MENUISERIE | 8 | 35$ | Installation clôtures bois/vinyle |

#### Extérieur et Finition

| Poste | Département | Cap. h/j | $/h | Description |
|-------|-------------|----------|-----|-------------|
| Revêtement vinyle | REVÊTEMENT | 8 | 38$ | Pose revêtement vinyle ou aluminium |
| Brique parement | MAÇONNERIE | 8 | 50$ | Pose brique de parement |
| Pierre naturelle | MAÇONNERIE | 6 | 55$ | Installation pierre naturelle |
| Entrée asphalte | PAVAGE | 6 | 40$ | Préparation et pavage entrée |
| Aménagement paysager | PAYSAGE | 8 | 32$ | Plantation, ensemencement |
| Nettoyage chantier | NETTOYAGE | 8 | 25$ | Nettoyage progressif chantier |
| Nettoyage final | NETTOYAGE | 8 | 28$ | Nettoyage complet livraison |
| Inspection finale | QUALITÉ | 4 | 65$ | Inspection qualité pré-livraison |

---

## Niveaux de Priorité

| Priorité | Badge CSS | Couleur | Usage |
|----------|-----------|---------|-------|
| **NORMAL** | `priority-normal` | Vert `#065f46` | Travaux standard |
| **URGENT** | `priority-urgent` | Orange `#92400e` | Travaux prioritaires |
| **CRITIQUE** | `priority-critique` | Rouge `#991b1b` | Urgence maximale |

---

## Fonctionnalités Principales

### 1. Gestion des Bons de Travail (BT)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouveau BT | Créer un bon de travail |
| **Modifier** | ✏️ Modifier | Éditer un BT existant |
| **Valider** | ✅ Valider | Passer en statut VALIDÉ |
| **Imprimer** | 🖨️ Imprimer | Générer PDF ou HTML |
| **Supprimer** | 🗑️ Supprimer | Annuler le BT |

### 2. Informations du BT

| Champ | Obligatoire | Description |
|-------|-------------|-------------|
| **Numéro BT** | Auto | Format BT-XXXX généré |
| **Titre** | Oui | Description courte |
| **Projet** | Non | Projet associé |
| **Client** | Non | Client concerné |
| **Priorité** | Oui | NORMAL/URGENT/CRITIQUE |
| **Dates** | Non | Début et fin prévue |
| **Chargé de projet** | Non | Responsable |

### 3. Gestion des Opérations

Chaque BT contient une liste d'opérations :

| Champ | Description |
|-------|-------------|
| **Séquence** | Ordre d'exécution |
| **Description** | Description du travail |
| **Poste** | Work center assigné |
| **Temps estimé** | Heures prévues |
| **Fournisseur** | Interne ou externe |
| **Statut** | Avancement de l'opération |

### 4. Assignation des Employés

Table `bt_assignations` :

| Champ | Description |
|-------|-------------|
| `bt_id` | FK vers le bon de travail |
| `employe_id` | FK vers l'employé |
| `statut` | ASSIGNÉ / LIBÉRÉ |
| `date_assignation` | Date d'assignation |

### 5. Statistiques Dashboard

| Métrique | Description |
|----------|-------------|
| **Total BTs** | Nombre total de bons de travail |
| **Par statut** | Répartition par statut |
| **Par priorité** | Répartition par priorité |
| **En retard** | BTs dépassant l'échéance |

---

## Export HTML Professionnel

Le module génère des exports HTML formatés pour impression :

### En-tête Document

```
┌────────────────────────────────────────────────┐
│  CONSTRUCTO AI INC.                            │
│  Construction résidentiel et commercial        │
├────────────────────────────────────────────────┤
│  📋 BON DE TRAVAIL #BT-XXXX                   │
│  Créé le: XX/XX/XXXX                          │
└────────────────────────────────────────────────┘
```

### Sections du Document

1. **Informations Générales** : Projet, Client, Chargé, Priorité, Dates
2. **Résumé** : Nb tâches, Nb matériaux, Heures prévues, Heures interne/externe
3. **Tâches et Opérations** : Tableau des opérations avec statuts
4. **Matériaux et Outils** : Liste des ressources requises

### Palette CSS Export

```css
--primary-color: #2563eb;
--primary-color-light: #3b82f6;
--background-color: #FAFBFF;
--text-color: #374151;
--border-color: #E5E7EB;
```

---

## Guide Pas-à-Pas

### Créer un bon de travail

1. Cliquez sur **"➕ Nouveau BT"**
2. Remplissez les informations :

   **Section Identification :**
   - Numéro BT (généré automatiquement)
   - Titre descriptif (obligatoire)
   - Projet associé

   **Section Client :**
   - Sélectionnez le client
   - Chargé de projet

   **Section Planification :**
   - Date de début
   - Date de fin prévue
   - Priorité (NORMAL/URGENT/CRITIQUE)

3. Ajoutez les opérations :
   - Cliquez sur **"➕ Ajouter opération"**
   - Description détaillée
   - Sélectionnez le poste de travail
   - Temps estimé (heures)
   - Fournisseur (Interne ou externe)
4. Ajoutez les matériaux requis
5. Cliquez sur **"💾 Enregistrer"**
6. Le BT est créé en statut BROUILLON

### Valider et lancer un BT

1. Ouvrez le BT en statut BROUILLON
2. Vérifiez toutes les informations
3. Assignez les employés aux opérations
4. Cliquez sur **"✅ Valider"**
5. Le statut passe à VALIDÉ
6. Un historique de validation est créé

### Suivre l'avancement d'un BT

1. Ouvrez le bon de travail
2. Pour chaque opération, mettez à jour :
   - Statut : À FAIRE → EN_COURS → TERMINÉ
   - Temps réel passé
   - Notes de progression
3. L'avancement global se calcule automatiquement
4. Les alertes de retard s'affichent si nécessaire

### Exporter un BT en HTML/PDF

1. Ouvrez le bon de travail
2. Cliquez sur **"🖨️ Exporter"**
3. Choisissez le format (HTML ou PDF)
4. Le document s'ouvre dans une nouvelle fenêtre
5. Imprimez ou enregistrez le fichier

### Clôturer un bon de travail

1. Vérifiez que toutes les opérations sont "TERMINÉ"
2. Renseignez les temps réels finaux
3. Ajoutez les notes de clôture
4. Passez le BT en statut **"TERMINÉ"**
5. Le BT est archivé avec l'historique complet

---

## Synchronisation BT-Opérations

### Processus Automatique

Lors de la sauvegarde d'un BT, le système :

1. Supprime les anciennes opérations du BT
2. Crée les nouvelles opérations avec :
   - Recherche du `work_center_id` correspondant
   - Attribution automatique du `project_id`
   - Conversion du statut BT vers statut opération
   - Génération du `sequence_number`

### Requête SQL de Création

```sql
INSERT INTO operations
(project_id, work_center_id, formulaire_bt_id, sequence_number,
 description, temps_estime, ressource, statut, notes)
VALUES (:project_id, :work_center_id, :bt_id, :sequence,
        :description, :temps, :ressource, :statut, :notes)
```

---

## Historique des Validations

Chaque changement de statut est enregistré dans `formulaire_validations` :

```sql
INSERT INTO formulaire_validations
(formulaire_id, type_validation, ancien_statut, nouveau_statut, commentaires)
VALUES (:bt_id, 'CHANGEMENT_STATUT', :ancien, :nouveau,
        'Déplacé via Production: :ancien → :nouveau')
```

---

## Système de Cache

| Donnée | TTL | Fonction |
|--------|-----|----------|
| Bons de travail | 5 min | `_get_all_bons_travail_cached()` |
| Statistiques BT | 5 min | `_get_bt_statistics_cached()` |
| Postes de travail | 10 min | `_get_postes_travail_cached()` |

### Invalidation

```python
def invalidate_production_cache():
    _get_all_bons_travail_cached.clear()
    _get_bt_statistics_cached.clear()
    _get_postes_travail_cached.clear()
```

---

## Astuces et Bonnes Pratiques

- **Découpez finement** : Mieux vaut plusieurs petites opérations qu'une seule grosse
- **Estimez réalistement** : Ajoutez 20% de marge aux temps estimés
- **Utilisez les bons postes** : Choisissez le poste correspondant au travail réel
- **Documentez les fournisseurs** : Distinguez interne vs externe pour l'analyse des coûts
- **Mettez à jour en temps réel** : Utilisez le TimeTracker pour le suivi des heures
- **Utilisez les priorités** : Traitez les CRITIQUE en premier
- **Documentez les blocages** : Notez les raisons des suspensions

---

## Résolution de Problèmes

### Le BT ne s'enregistre pas

- **Cause** : Champ obligatoire manquant
- **Solution** : Vérifiez que le titre est renseigné

### Les opérations ne s'affichent pas

- **Cause** : Aucune opération ajoutée au BT
- **Solution** : Ajoutez des opérations via le formulaire

### Le poste de travail n'existe pas

- **Cause** : Table `work_centers` non initialisée
- **Solution** : Le système initialise automatiquement les 61 postes au premier accès

### L'export HTML est vide

- **Cause** : Données du BT incomplètes
- **Solution** : Complétez toutes les informations avant export

---

## Questions Fréquentes (FAQ)

**Q: Comment créer un BT depuis un devis ?**
R: Quand vous convertissez un devis approuvé en projet, les lignes de devis peuvent être transformées en opérations de BT via le module Production.

**Q: Puis-je assigner plusieurs employés à une opération ?**
R: Oui, utilisez la table `bt_assignations` pour assigner plusieurs employés. Le temps peut être réparti.

**Q: Comment voir qui travaille sur quoi aujourd'hui ?**
R: Consultez le TimeTracker ou le Gantt vue "Bons de Travail" avec filtre sur la date du jour.

**Q: Les temps réels alimentent-ils la paie ?**
R: Oui, les heures pointées via TimeTracker sont associées aux opérations et disponibles pour la paie.

**Q: Comment dupliquer un BT ?**
R: Créez un nouveau BT et copiez manuellement les opérations, ou utilisez les templates si disponibles.

**Q: Quelle est la différence entre VALIDÉ et APPROUVÉ ?**
R: VALIDÉ = validation interne (chef de projet). APPROUVÉ = validation client (signature).

---

## Données Techniques

### Requête Récupération BT

```sql
SELECT bt.id, bt.numero_bt, bt.titre, bt.statut, bt.priorite,
       bt.date_debut, bt.date_fin, bt.project_id,
       p.nom_projet, c.nom as client_nom
FROM bons_travail bt
LEFT JOIN projects p ON bt.project_id = p.id
LEFT JOIN companies c ON bt.client_id = c.id
ORDER BY bt.updated_at DESC
```

### Requête Statistiques

```sql
SELECT statut, COUNT(*) as count
FROM bons_travail
GROUP BY statut
```

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Projets associés aux BT
- [📈 Gantt](09-gantt.md) - Planification visuelle des BT
- [🔄 Kanban](10-kanban.md) - Vue Kanban des BT
- [👥 Employés](12-employes.md) - Gestion des ressources
- [⏱️ TimeTracker](13-timetracker.md) - Pointage sur les opérations
- [🧾 Devis](06-devis.md) - Source des bons de travail
