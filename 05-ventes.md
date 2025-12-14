# 🤝 Ventes (Opportunités)

## Introduction

Le module **Ventes** (CRM Pipeline) est votre outil de gestion commerciale complet. Il vous permet de suivre votre pipeline de ventes avec un tableau Kanban visuel, de qualifier vos prospects avec la méthode B.A.T., de gérer vos opportunités d'affaires et d'automatiser les tâches de suivi du premier contact jusqu'à la signature du contrat.

Ce module fait partie du système CRM de CONSTRUCTO AI et inclut des workflows automatiques de type Insightly pour créer des tâches lors des changements d'étape.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🤝 Ventes"**
2. Le pipeline commercial s'affiche en vue Kanban
3. Naviguez entre les vues : Pipeline, Liste, Activités

---

## Structure des Données

### Table PostgreSQL : `opportunities`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom` | TEXT | Nom de l'opportunité |
| `company_id` | INTEGER | FK vers `companies.id` |
| `contact_id` | INTEGER | FK vers `contacts.id` |
| `montant_estime` | DECIMAL | Montant estimé en $ |
| `probabilite` | INTEGER | Probabilité de succès (0-100%) |
| `statut` | TEXT | Étape du pipeline |
| `date_cloture_prevue` | DATE | Date de clôture prévue |
| `source` | TEXT | Source de l'opportunité |
| `notes` | TEXT | Notes internes |
| `assigned_to` | INTEGER | FK vers `employees.id` |
| `projet_id` | INTEGER | FK vers `projects.id` (après conversion) |
| `converted_at` | TIMESTAMP | Date de conversion en projet |
| `date_derniere_activite` | TIMESTAMP | Dernière interaction |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

---

## Pipeline Commercial (Kanban)

### Les 6 Étapes du Pipeline

| Étape | Couleur | Code | Description | Probabilité suggérée |
|-------|---------|------|-------------|---------------------|
| **Prospection** | 🔵 Bleu | `#3B82F6` | Nouveau contact identifié | 10% |
| **Qualification** | 🟠 Orange | `#F59E0B` | Évaluation du potentiel | 25% |
| **Proposition** | 🔵 Bleu clair | `#60A5FA` | Devis/soumission envoyé | 50% |
| **Négociation** | 🟣 Violet | `#8B5CF6` | Discussion en cours | 75% |
| **Gagné** | 🟢 Vert | `#10B981` | Contrat signé | 100% |
| **Perdu** | 🔴 Rouge | `#EF4444` | Opportunité non concrétisée | 0% |

### Vue Kanban

- **Colonnes** : Une par étape du pipeline
- **Cartes** : Une par opportunité
- **Drag & Drop** : Glissez une carte pour changer d'étape
- **Montant total** : Affiché en haut de chaque colonne

### Informations sur les Cartes

| Élément | Description |
|---------|-------------|
| **Titre** | Nom de l'opportunité |
| **Entreprise** | Nom du client |
| **Montant** | Montant estimé formaté |
| **Probabilité** | Pourcentage de succès |
| **Date** | Date de clôture prévue |
| **Assigné** | Commercial responsable |

---

## Système de Qualification B.A.T.

### Grille de Qualification (4 critères)

| Critère | Points max | Description |
|---------|------------|-------------|
| **Budget** | 25 | Le prospect a-t-il le budget ? |
| **Autorité** | 25 | Parlez-vous au décideur ? |
| **Timing** | 25 | Le projet est-il imminent ? |
| **Compatibilité** | 25 | Correspond à vos compétences ? |

### Échelle de Scoring

| Score | Catégorie | Badge | Action recommandée |
|-------|-----------|-------|-------------------|
| 80-100 | Excellent | 🟢 Vert | Priorité maximale |
| 60-79 | Bon | 🟡 Jaune | Suivi actif |
| 40-59 | Moyen | 🟠 Orange | À développer |
| 0-39 | Faible | 🔴 Rouge | Requalifier ou abandonner |

### Processus de Qualification

1. Ouvrez l'opportunité
2. Cliquez sur **"🎯 Qualifier"**
3. Répondez à chaque question (échelle 0-25)
4. Le score total est calculé automatiquement
5. La catégorie et recommandation s'affichent

---

## Workflows Automatiques

### Tâches Automatiques par Étape

Lors du changement d'étape, des tâches sont créées automatiquement :

#### Étape "Qualification"
| Tâche | Délai |
|-------|-------|
| Vérifier les besoins du client | 2 jours |
| Préparer une présentation de découverte | 5 jours |

#### Étape "Proposition"
| Tâche | Délai |
|-------|-------|
| Rédiger la proposition commerciale | 3 jours |
| Faire valider la proposition en interne | 1 jour |
| Envoyer la proposition au client | 5 jours |

#### Étape "Négociation"
| Tâche | Délai | Priorité |
|-------|-------|----------|
| Planifier une réunion de négociation | 2 jours | Haute |
| Préparer les arguments de négociation | 1 jour | Haute |
| Obtenir l'approbation finale | 7 jours | Haute |

#### Étape "Gagné"
| Tâche | Délai |
|-------|-------|
| Envoyer le contrat pour signature | 1 jour |
| Créer le projet dans l'ERP | 2 jours |
| Planifier la réunion de lancement | 5 jours |

### Logging des Workflows

Chaque exécution de workflow est enregistrée :
- Type d'entité (OPPORTUNITY)
- ID de l'entité
- Événement déclencheur (STATUS_CHANGED)
- Description des actions exécutées

---

## Gestion des Activités CRM

### Types d'Activités

| Type | Description |
|------|-------------|
| **Email** | Correspondance électronique |
| **Appel** | Communication téléphonique |
| **Réunion** | Rencontre en personne/virtuelle |
| **Tâche** | Action à réaliser |
| **Note** | Mémo interne |
| **Visite** | Visite de chantier |
| **Présentation** | Présentation commerciale |
| **Suivi** | Activité de suivi |

### Statuts d'Activités

| Statut | Description |
|--------|-------------|
| **Planifié** | À venir |
| **En cours** | En train d'être réalisée |
| **Terminé** | Complétée |
| **Annulé** | Annulée |
| **Reporté** | Replanifiée |

### Priorités

| Priorité | Usage |
|----------|-------|
| **Basse** | Actions non urgentes |
| **Normale** | Actions standard |
| **Haute** | Actions prioritaires |
| **Critique** | Actions urgentes |

---

## Conversion en Projet

### Conditions de Conversion

- L'opportunité doit être au statut **"Gagné"**
- Une entreprise doit être associée

### Processus Automatique

1. Cliquez sur **"🚀 Convertir en projet"**
2. Le système crée un projet avec :
   - Nom = Nom de l'opportunité
   - Client = Entreprise de l'opportunité
   - Contact = Contact de l'opportunité
   - Prix estimé = Montant de l'opportunité
   - Statut = "À FAIRE"
   - Priorité = HAUTE si montant > 50 000 $

3. L'opportunité est marquée comme convertie :
   - `projet_id` = ID du nouveau projet
   - `converted_at` = Date de conversion

4. Tâches initiales créées pour le projet :
   - Réunion de lancement (3 jours)
   - Définir les jalons (5 jours)
   - Constituer l'équipe (2 jours)
   - Plan de projet détaillé (7 jours)

---

## Guide Pas-à-Pas

### Créer une nouvelle opportunité

1. Cliquez sur **"➕ Nouvelle opportunité"**
2. Remplissez le formulaire :

   **Informations principales :**
   - Nom de l'opportunité (obligatoire)
   - Entreprise (sélection)
   - Contact (sélection)

   **Financier :**
   - Montant estimé ($)
   - Probabilité (slider 0-100%)

   **Pipeline :**
   - Statut (étape initiale)
   - Date de clôture prévue

   **Attribution :**
   - Source (d'où vient l'opportunité)
   - Assigné à (commercial responsable)
   - Notes

3. Cliquez sur **"💾 Enregistrer"**

### Faire avancer une opportunité (Kanban)

1. Dans le pipeline Kanban, localisez votre opportunité
2. **Glissez-déposez** la carte vers la colonne suivante
3. Les tâches automatiques sont créées
4. Un log de workflow est enregistré

### Faire avancer une opportunité (Formulaire)

1. Ouvrez l'opportunité (clic sur la carte)
2. Modifiez le champ **"Statut"**
3. Cliquez sur **"💾 Enregistrer"**
4. Les tâches automatiques sont créées

### Enregistrer une interaction depuis une opportunité

1. Ouvrez l'opportunité
2. Cliquez sur **"💬 Nouvelle interaction"**
3. Remplissez :
   - Type (Email, Appel, Réunion, etc.)
   - Date et heure
   - Résumé (100 caractères)
   - Détails
   - Résultat (Positif, Neutre, Négatif, En cours, À suivre)
   - Date de suivi prévue
4. Option : "Créer automatiquement une activité de suivi"
5. Enregistrez

### Planifier une activité depuis une opportunité

1. Ouvrez l'opportunité
2. Cliquez sur **"📅 Nouvelle activité"**
3. Remplissez :
   - Titre
   - Type d'activité
   - Date et heure de début
   - Durée (heures)
   - Priorité
   - Assigné à
   - Description
4. Enregistrez

### Convertir une opportunité gagnée en projet

1. Passez l'opportunité au statut **"Gagné"**
2. Cliquez sur **"🚀 Convertir en projet"**
3. Le projet est créé automatiquement
4. Les tâches initiales sont générées
5. Vous êtes redirigé vers le module Projets

---

## Tableau de Bord Commercial

### Métriques Affichées

| Métrique | Description |
|----------|-------------|
| **Pipeline valorisé** | Montant total par étape |
| **Taux de conversion** | % de prospects devenus clients |
| **Cycle de vente moyen** | Durée moyenne de conversion |
| **Top opportunités** | Les plus gros projets potentiels |
| **Valeur pondérée** | Montant × Probabilité |

### Calculs

```
Valeur pondérée = Σ (Montant × Probabilité / 100)

Taux de conversion = (Opportunités Gagnées / Total Opportunités) × 100

Cycle de vente = AVG(date_gagné - date_création)
```

---

## Système de Cache

### Performance Optimisée

| Donnée | TTL | Raison |
|--------|-----|--------|
| Pipeline opportunités | 5 minutes | Données dynamiques |
| Activités récentes | 2 minutes | Très dynamiques |
| Stats pipeline | 5 minutes | Calculs complexes |

---

## Astuces et Bonnes Pratiques

- **Qualifiez tôt** : Ne perdez pas de temps sur des prospects non qualifiés
- **Documentez chaque interaction** : L'historique aide à la reprise du dossier
- **Mettez à jour les montants** : Ajustez les estimations au fil des discussions
- **Utilisez les probabilités réalistes** : Ne surestimez pas
- **Planifiez toujours le prochain suivi** : Jamais sans prochaine action
- **Gardez le pipeline propre** : Fermez les opportunités mortes
- **Assignez un responsable** : Chaque opportunité doit avoir un propriétaire

---

## Résolution de Problèmes

### L'opportunité ne change pas d'étape

- **Cause** : Erreur de mise à jour
- **Solution** : Ouvrez l'opportunité en mode édition et changez manuellement le statut

### Les tâches automatiques ne se créent pas

- **Cause** : Workflow non déclenché
- **Solution** : Vérifiez que le changement de statut a bien été enregistré

### Impossible de convertir en projet

- **Cause** : L'opportunité n'est pas au statut "Gagné"
- **Solution** : Changez d'abord le statut en "Gagné"

### Le montant pondéré est incorrect

- **Cause** : Probabilité non mise à jour
- **Solution** : Ajustez la probabilité selon l'étape du pipeline

---

## Questions Fréquentes (FAQ)

**Q: Comment calculer la probabilité de succès ?**
R: Basez-vous sur l'étape : Prospection (10%), Qualification (25%), Proposition (50%), Négociation (75%). Ajustez selon votre expérience et la qualification B.A.T.

**Q: Puis-je avoir plusieurs opportunités pour le même client ?**
R: Oui, un client peut avoir plusieurs projets potentiels en parallèle. Chaque projet est une opportunité distincte.

**Q: Comment voir mes performances de vente ?**
R: Consultez le module Analytics BI > onglet "Commercial & CRM" pour des rapports détaillés.

**Q: Que faire si un prospect ne répond plus ?**
R: Après 3 relances sans réponse, marquez l'opportunité comme "Perdu - Sans réponse" pour garder votre pipeline propre.

**Q: Les tâches automatiques sont-elles obligatoires ?**
R: Non, elles sont créées automatiquement mais vous pouvez les modifier ou les supprimer selon vos besoins.

**Q: Comment réouvrir une opportunité perdue ?**
R: Modifiez l'opportunité et changez le statut de "Perdu" vers une autre étape.

---

## Données Techniques

### Requête SQL Pipeline

```sql
SELECT o.id, o.nom, o.montant_estime, o.statut, o.probabilite,
       o.date_cloture_prevue, o.company_id, c.nom as company_nom
FROM opportunities o
LEFT JOIN companies c ON o.company_id = c.id
ORDER BY o.date_derniere_activite DESC
```

### Calcul Valeur Pipeline

```sql
SELECT statut,
       COUNT(*) as nb_opportunites,
       SUM(montant_estime) as montant_total,
       SUM(montant_estime * probabilite / 100) as valeur_ponderee
FROM opportunities
WHERE statut NOT IN ('Gagné', 'Perdu')
GROUP BY statut
```

---

## Voir Aussi

- [🏢 Entreprises](03-entreprises.md) - Base clients
- [👥 Contacts](04-contacts.md) - Gestion des contacts
- [🧾 Devis](06-devis.md) - Créer des soumissions
- [📋 Projets](07-projets.md) - Gestion de projets
- [📊 Analytics BI](02-analytics-bi.md) - Analyses commerciales
- [📅 Calendrier](08-calendrier.md) - Planification des activités
