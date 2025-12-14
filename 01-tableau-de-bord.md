# 🏠 Tableau de Bord

## Introduction

Le **Tableau de Bord** est le point d'entrée principal de CONSTRUCTO AI. Il offre une vue d'ensemble complète de votre entreprise de construction avec des indicateurs clés de performance (KPI) en temps réel, des alertes intelligentes, des graphiques interactifs et un accès rapide aux actions les plus fréquentes.

Cette page est conçue pour vous permettre de prendre des décisions éclairées en un coup d'œil, sans avoir à naviguer dans plusieurs modules.

---

## Accès au Module

1. Connectez-vous à CONSTRUCTO AI
2. Vous êtes automatiquement dirigé vers le **Tableau de Bord**
3. Alternativement, cliquez sur **"🏠 Tableau de Bord"** dans le menu latéral

> **Bienvenue !** Un message d'accueil personnalisé s'affiche avec votre nom d'utilisateur en haut de la page.

---

## Structure du Tableau de Bord

Le tableau de bord est organisé en **12 sections principales** :

| Section | Contenu |
|---------|---------|
| 1. Alertes Devis | Devis arrivant à échéance (< 3 jours) |
| 2. KPI Projets | Total, Actifs, Taux de complétion, CA |
| 3. KPI Catalogue | Produits, Catégories, Prix moyen |
| 4. KPI Devis | Total, En attente, Acceptés, Montant |
| 5. KPI Formulaires | Bons de travail, Achats, Demandes prix, Réceptions |
| 6. KPI Fournisseurs | Total, Actifs, Catégories, Commandes |
| 7. KPI Fichiers | Total, Photos, Documents |
| 8. KPI Production | Postes, Robots, CNC, Maintenance |
| 9. KPI TimeTracker | Heures totales, Mois, Moyenne/jour |
| 10. KPI RH | Employés total, Actifs, Temps plein/partiel |
| 11. Alertes & Notifications | Devis expirants, Projets inactifs, Stock critique |
| 12. Vues rapides | Inventaire, CRM, Projets prioritaires |

---

## Section 1 : Alertes Devis Urgents

### Description
Cette section s'affiche **uniquement** lorsque des devis arrivent à échéance dans les 3 prochains jours.

### Informations affichées
Pour chaque devis urgent :
- **Numéro du devis** : Format `DEV-XXXX`
- **Client** : Nom de l'entreprise ou du contact
- **Montant** : Total TTC du devis
- **Date limite** : Date de validité du devis
- **Jours restants** : Compte à rebours

### Code couleur des alertes
| Jours restants | Couleur | Signification |
|----------------|---------|---------------|
| 0 jour | Rouge vif (`#EF4444`) | **CRITIQUE** - Expire aujourd'hui |
| 1 jour | Orange (`#F59E0B`) | **URGENT** - Expire demain |
| 2-3 jours | Jaune | **ATTENTION** - Expire bientôt |

### Actions disponibles
- **Cliquer sur un devis** : Ouvre le devis pour modification
- **Envoyer un rappel** : Relance le client par courriel

---

## Section 2 : KPI Projets

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Calcul |
|----------|-------|-------------|--------|
| **Total Projets** | 📁 | Nombre total de projets | `COUNT(projets)` |
| **Projets Actifs** | 🏃 | Projets en cours | `WHERE statut IN ('EN COURS', 'À FAIRE')` |
| **Taux Complétion** | ✅ | Pourcentage terminés | `(TERMINÉ / Total) × 100%` |
| **CA Total** | 💰 | Chiffre d'affaires | `SUM(prix_estime)` |

### Format d'affichage
- Les montants sont formatés en dollars canadiens (ex: `125 000,00 $`)
- Les pourcentages incluent le symbole `%`
- Les grands nombres utilisent des séparateurs de milliers

---

## Section 3 : KPI Catalogue Produits

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Source |
|----------|-------|-------------|--------|
| **Produits** | 📦 | Nombre total de produits | Table `products` |
| **Catégories** | 🏷️ | Nombre de catégories | Table `product_categories` |
| **Prix Moyen** | 💲 | Prix moyen des produits | `AVG(prix)` |
| **Produits Actifs** | ✅ | Produits avec stock > 0 | `WHERE actif = true` |

---

## Section 4 : KPI Devis

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Calcul |
|----------|-------|-------------|--------|
| **Total Devis** | 📋 | Tous les devis | `COUNT(devis)` |
| **En Attente** | ⏳ | Devis non répondus | `WHERE statut = 'EN ATTENTE'` |
| **Acceptés** | ✅ | Devis signés | `WHERE statut = 'ACCEPTÉ'` |
| **Montant Total** | 💵 | Valeur totale | `SUM(montant_total)` |

### Statuts des devis
| Statut | Signification |
|--------|---------------|
| BROUILLON | En cours de rédaction |
| ENVOYÉ | Transmis au client |
| EN ATTENTE | Attente de réponse |
| ACCEPTÉ | Client a signé |
| REFUSÉ | Client a décliné |
| EXPIRÉ | Date de validité dépassée |

---

## Section 5 : KPI Formulaires

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Table source |
|----------|-------|-------------|--------------|
| **Bons de Travail** | 📝 | BT créés | `bons_travail` |
| **Bons d'Achat** | 🛒 | BA créés | `bons_achat` |
| **Demandes Prix** | 📧 | DP envoyées | `demandes_prix` |
| **Réceptions** | 📥 | Réceptions enregistrées | `receptions` |

### Liens vers les formulaires
Chaque métrique est cliquable et mène vers le module correspondant pour créer ou gérer ces documents.

---

## Section 6 : KPI Fournisseurs

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description |
|----------|-------|-------------|
| **Total Fournisseurs** | 🏭 | Nombre total de fournisseurs |
| **Fournisseurs Actifs** | ✅ | Fournisseurs avec commandes récentes |
| **Catégories** | 📂 | Types de fournisseurs |
| **Commandes Année** | 📊 | Commandes de l'année en cours |

---

## Section 7 : KPI Fichiers et Pièces Jointes

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Extensions |
|----------|-------|-------------|------------|
| **Total Fichiers** | 📎 | Tous les fichiers | Toutes |
| **Photos** | 📷 | Images | `.jpg`, `.png`, `.gif`, `.webp` |
| **Documents** | 📄 | Documents texte | `.pdf`, `.doc`, `.docx`, `.xlsx` |
| **Autres** | 📁 | Fichiers divers | Autres extensions |

---

## Section 8 : KPI Production

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Signification |
|----------|-------|-------------|---------------|
| **Postes de Travail** | 🏭 | Stations de production | Postes configurés |
| **Robots** | 🤖 | Équipements automatisés | Robots de soudage, etc. |
| **Machines CNC** | ⚙️ | Machines à commande numérique | Découpe, usinage |
| **Maintenance Due** | 🔧 | Équipements à entretenir | Maintenance préventive requise |

### Alerte maintenance
Si le nombre de maintenances dues est > 0, l'indicateur s'affiche en **rouge** pour signaler l'urgence.

---

## Section 9 : KPI TimeTracker

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description | Calcul |
|----------|-------|-------------|--------|
| **Heures Totales** | ⏱️ | Cumul historique | `SUM(heures)` |
| **Heures Mois** | 📅 | Heures du mois en cours | Filtre mois courant |
| **Moyenne/jour** | 📊 | Heures moyennes quotidiennes | `Heures Mois / Jours ouvrés` |
| **Sessions Mois** | 🔢 | Nombre d'entrées de temps | `COUNT(sessions)` ce mois |

---

## Section 10 : KPI Ressources Humaines

### Métriques affichées (4 colonnes)

| Métrique | Icône | Description |
|----------|-------|-------------|
| **Total Employés** | 👥 | Nombre total d'employés |
| **Actifs** | ✅ | Employés en poste |
| **Temps Plein** | 👤 | Employés 40h/semaine |
| **Temps Partiel** | 👤 | Employés < 40h/semaine |

---

## Section 11 : Alertes et Notifications

### Panneau d'alertes intelligent
Ce panneau regroupe toutes les alertes nécessitant votre attention.

### Types d'alertes

#### 1. Devis expirant bientôt
- **Seuil** : 3 jours avant expiration
- **Icône** : ⚠️
- **Couleur** : Jaune/Orange
- **Action** : Relancer le client ou prolonger la validité

#### 2. Projets sans activité récente
- **Seuil** : 7 jours sans modification
- **Icône** : 📋
- **Couleur** : Bleu
- **Action** : Vérifier l'état du projet

#### 3. Stock critique
- **Seuil** : Quantité ≤ seuil critique configuré
- **Icône** : 📦
- **Couleur** : Rouge (`#EF4444`)
- **Action** : Commander du stock

### Codes couleur des alertes
| Type | Couleur fond | Couleur bordure | Signification |
|------|--------------|-----------------|---------------|
| Danger | `#FEE2E2` | `#EF4444` | Action immédiate requise |
| Avertissement | `#FEF3C7` | `#F59E0B` | Attention recommandée |
| Info | `#DBEAFE` | `#3B82F6` | Information à consulter |
| Succès | `#D1FAE5` | `#10B981` | Tout va bien |

---

## Section 12 : Vues Rapides

### 12.1 Aperçu Inventaire

Affiche l'état du stock en 3 catégories :

| État | Icône | Couleur | Description |
|------|-------|---------|-------------|
| **Critique** | 🔴 | Rouge | Stock ≤ seuil critique |
| **Faible** | 🟡 | Jaune | Stock entre critique et optimal |
| **Suffisant** | 🟢 | Vert | Stock ≥ seuil optimal |

### 12.2 Aperçu CRM

Affiche les statistiques CRM en un coup d'œil :

| Métrique | Description |
|----------|-------------|
| **Entreprises** | Nombre total de clients/prospects |
| **Contacts** | Personnes dans votre réseau |
| **Opportunités** | Ventes en cours |
| **Pipeline** | Montant total des opportunités ouvertes |

### 12.3 Top 5 Projets Prioritaires

Liste les 5 projets les plus urgents, triés par :
1. **Priorité** (ÉLEVÉ → MOYEN → BAS)
2. **Date de début** (plus ancien en premier)

#### Informations affichées par projet
- **Numéro et nom** : `#123 - Construction Résidence ABC`
- **Client** : Nom de l'entreprise cliente
- **Montant estimé** : Budget du projet
- **Priorité** : Badge coloré
- **Jours restants** : Avant la date cible

#### Codes couleur des priorités
| Priorité | Couleur bordure | Couleur fond |
|----------|-----------------|--------------|
| ÉLEVÉ | `#EF4444` (rouge) | `#FEE2E2` |
| MOYEN | `#F59E0B` (orange) | `#FEF3C7` |
| BAS | `#10B981` (vert) | `#D1FAE5` |

---

## Section 13 : Graphiques Interactifs

### 13.1 Graphique circulaire - Projets par Statut

Visualise la répartition de vos projets selon leur statut.

| Statut | Couleur | Code hexadécimal |
|--------|---------|------------------|
| À FAIRE | Bleu | `#3B82F6` |
| EN COURS | Jaune | `#EAB308` |
| EN ATTENTE | Bleu | `#3B82F6` |
| TERMINÉ | Vert | `#10B981` |
| ANNULÉ | Gris foncé | `#1F2937` |
| LIVRAISON | Violet | `#A78BFA` |

**Interactivité** : Survolez une section pour voir le détail (nombre et pourcentage).

### 13.2 Graphique à barres - Postes par Département

Affiche la répartition des postes de travail par département.

| Département | Couleur | Code hexadécimal |
|-------------|---------|------------------|
| PRODUCTION | Vert | `#10B981` |
| STRUCTURE | Bleu | `#3B82F6` |
| QUALITÉ | Orange | `#F59E0B` |
| INGÉNIERIE | Violet | `#8B5CF6` |
| COMMERCIAL | Rouge | `#EF4444` |
| ADMINISTRATION | Indigo | `#6366F1` |
| DIRECTION | Rose | `#F43F5E` |

---

## Section 14 : Projets Récents

### Description
Liste les 5 derniers projets créés ou modifiés.

### Informations affichées par projet

| Colonne | Contenu |
|---------|---------|
| **Identité** | `#ID - Nom du projet` + Description (100 premiers caractères) |
| **Client** | Nom de l'entreprise + Montant estimé |
| **Statut** | Badge avec icône + Priorité |
| **Actions** | Boutons d'action rapide |

### Icônes de statut
| Statut | Icône |
|--------|-------|
| À FAIRE | 🟡 |
| EN COURS | 🔵 |
| EN ATTENTE | 🔴 |
| TERMINÉ | 🟢 |
| ANNULÉ | ⚫ |
| LIVRAISON | 🟣 |

### Icônes de priorité
| Priorité | Icône |
|----------|-------|
| ÉLEVÉ | 🔴 |
| MOYEN | 🟡 |
| BAS | 🟢 |

### Boutons d'action rapide

| Bouton | Icône | Action |
|--------|-------|--------|
| **Voir** | 👁️ | Ouvre les détails du projet |
| **Bon de Travail** | 🔧 | Crée un BT pour ce projet |
| **Bon d'Achat** | 🛒 | Crée un BA pour ce projet |

---

## Guide Pas-à-Pas

### Consulter l'état global de l'entreprise

1. Connectez-vous à CONSTRUCTO AI
2. Le tableau de bord s'affiche automatiquement
3. Parcourez les sections de KPI pour une vue d'ensemble
4. Notez les alertes en rouge ou orange (actions urgentes)
5. Consultez les graphiques pour les tendances

### Réagir à une alerte de devis expirant

1. Localisez la section **Alertes Devis** en haut de page
2. Identifiez les devis avec moins de 3 jours restants
3. Cliquez sur le devis concerné
4. Options :
   - **Relancer le client** par courriel
   - **Prolonger la validité** de 30 jours
   - **Clôturer le devis** comme refusé

### Créer un bon de travail depuis le tableau de bord

1. Repérez le projet concerné dans **Projets Récents**
2. Cliquez sur le bouton **🔧** (Créer Bon de Travail)
3. Le formulaire de création s'ouvre avec le projet pré-sélectionné
4. Complétez les informations du bon de travail
5. Enregistrez

### Vérifier le stock critique

1. Consultez la section **Aperçu Inventaire**
2. Si le compteur **Critique** est > 0, cliquez dessus
3. Vous êtes redirigé vers le module Inventaire
4. Filtrez par "Stock critique"
5. Passez les commandes nécessaires

### Analyser la répartition des projets

1. Faites défiler jusqu'au graphique **Projets par Statut**
2. Survolez chaque section pour voir les détails
3. Identifiez les goulots d'étranglement (trop de projets "EN ATTENTE" par exemple)
4. Prenez les actions correctives nécessaires

---

## Personnalisation

### Données affichées
Les KPI sont calculés en temps réel à partir de votre base de données PostgreSQL. Aucune donnée n'est en cache.

### Rafraîchissement
- **Automatique** : À chaque chargement de la page
- **Manuel** : Utilisez `Ctrl+R` ou `F5` pour rafraîchir

### Période d'analyse
- **Projets** : Tous les projets de la base
- **TimeTracker** : Mois en cours pour les métriques mensuelles
- **Commandes** : Année civile en cours

---

## Astuces et Bonnes Pratiques

- **Consultez le tableau de bord chaque matin** pour avoir une vue d'ensemble
- **Traitez les alertes rouges en priorité** pour éviter les retards
- **Réagissez aux alertes rouges immédiatement** : Stock critique = perte de productivité potentielle
- **Suivez le taux de complétion** : Objectif recommandé > 80%
- **Surveillez les projets inactifs** : 7 jours sans activité peut indiquer un problème
- **Utilisez les actions rapides** : Les boutons 🔧 et 🛒 vous font gagner du temps
- **Interprétez les graphiques** : Un excès de projets "EN ATTENTE" indique des blocages

---

## Données Techniques

### Sources de données (Tables PostgreSQL)

| KPI | Table source | Champ principal |
|-----|--------------|-----------------|
| Projets | `projets` | `id`, `statut`, `prix_estime` |
| Produits | `products` | `id`, `prix`, `actif` |
| Devis | `devis` | `id`, `statut`, `montant_total` |
| Formulaires | `bons_travail`, `bons_achat`, etc. | `id` |
| Fournisseurs | `fournisseurs` | `id`, `actif` |
| Fichiers | `attachments` | `id`, `type_fichier` |
| Production | `workstations` | `type` (robot, cnc) |
| TimeTracker | `time_entries` | `heures`, `date` |
| Employés | `employes` | `id`, `type_contrat` |
| Inventaire | `inventory` | `quantite`, `seuil_critique` |
| CRM | `companies`, `contacts`, `opportunities` | `id` |

### Calculs spécifiques

```
Taux Complétion = (Projets TERMINÉ / Total Projets) × 100

CA Total = SUM(prix_estime) de tous les projets

Heures Moyenne/Jour = Heures Mois / Jours ouvrés dans le mois

Pipeline CRM = SUM(montant) des opportunités OUVERTES
```

---

## Résolution de Problèmes

### Le tableau de bord est vide
- **Cause** : Aucune donnée en base
- **Solution** : Créez votre premier projet, client ou produit

### Les graphiques ne s'affichent pas
- **Cause** : Données insuffisantes pour générer un graphique
- **Solution** : Ajoutez au moins 2-3 entrées dans chaque catégorie

### Les alertes ne sont pas à jour
- **Cause** : Page non rafraîchie
- **Solution** : Appuyez sur F5 ou cliquez sur le menu Tableau de Bord

### Certains KPI affichent "0"
- **Cause** : Module correspondant non utilisé
- **Solution** : Commencez à utiliser le module (ex: créer des devis)

---

## Questions Fréquentes (FAQ)

**Q: Les données sont-elles en temps réel ?**
R: Oui, chaque KPI est calculé à chaque chargement de page depuis la base PostgreSQL.

**Q: Puis-je personnaliser les seuils d'alerte ?**
R: Les seuils par défaut sont : 3 jours pour les devis, 7 jours pour les projets inactifs. Contactez le support pour les modifier.

**Q: Comment exporter les données du tableau de bord ?**
R: Utilisez le module **Analytics BI** pour des exports détaillés et des rapports personnalisés.

**Q: Le tableau de bord ralentit avec beaucoup de données ?**
R: Non, les requêtes sont optimisées. Si vous constatez des lenteurs, vérifiez votre connexion Internet.

**Q: Puis-je avoir plusieurs tableaux de bord ?**
R: Le tableau de bord principal est unique. Utilisez **Analytics BI** pour des vues personnalisées.

---

## Voir Aussi

- [📊 Analytics BI](02-analytics-bi.md) - Analyses avancées et rapports
- [📁 Projets](07-projets.md) - Gestion complète des projets
- [🧾 Devis](06-devis.md) - Création et suivi des devis
- [📦 Inventaire](17-inventaire.md) - Gestion des stocks
- [👥 CRM - Entreprises](03-entreprises.md) - Gestion des clients
