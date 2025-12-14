# 🔄 Kanban

## Introduction

Le module **Kanban** offre une gestion visuelle de vos Devis, Projets et Bons de Travail par glisser-déposer. Organisé en colonnes par statut, il permet de suivre l'avancement des éléments et de les déplacer facilement d'une étape à l'autre.

Ce module propose 3 tableaux Kanban distincts, chacun adapté au workflow de son type d'élément.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🔄 Kanban"**
2. Le système d'onglets affiche les 3 tableaux :
   - **Devis** (par défaut)
   - **Projets**
   - **Bons de Travail**
3. Naviguez entre les onglets selon vos besoins

---

## Les 3 Tableaux Kanban

### 1. Kanban Devis

| Colonne | Couleur | Description |
|---------|---------|-------------|
| **BROUILLON** | Gris `#4b5563` | Devis en cours de rédaction |
| **EN_ATTENTE** | Bleu `#3b82f6` | En attente de validation/envoi |
| **APPROUVÉ** | Vert `#10b981` | Devis accepté par le client |
| **REFUSÉ** | Rouge `#ef4444` | Devis refusé |
| **ANNULÉ** | Noir `#1f2937` | Devis annulé |

### 2. Kanban Projets

| Colonne | Couleur | Description |
|---------|---------|-------------|
| **EN_ATTENTE** | Bleu `#3b82f6` | Projet planifié, non démarré |
| **EN_COURS** | Jaune `#eab308` | Travaux en cours |
| **EN_PAUSE** | Bleu `#3b82f6` | Temporairement suspendu |
| **TERMINÉ** | Vert `#10b981` | Projet complété |
| **ANNULÉ** | Noir `#1f2937` | Projet abandonné |

### 3. Kanban Bons de Travail

| Colonne | Couleur | Description |
|---------|---------|-------------|
| **BROUILLON** | Gris `#4b5563` | BT en préparation |
| **VALIDÉ** | Bleu clair `#60a5fa` | BT validé, prêt à exécuter |
| **EN COURS** | Jaune `#eab308` | BT en cours d'exécution |
| **TERMINÉ** | Vert `#10b981` | BT complété |
| **ANNULÉ** | Noir `#1f2937` | BT annulé |

---

## Couleurs des Priorités

| Priorité | Icône | Couleur |
|----------|-------|---------|
| **CRITIQUE** | 🔴 | Rouge `#ef4444` |
| **URGENT** | 🟡 | Orange `#f59e0b` |
| **NORMAL** | 🟢 | Vert `#10b981` |

---

## Interface Utilisateur

### Structure d'une Colonne

```
┌──────────────────────┐
│  STATUT (NB)         │  ← En-tête coloré avec compteur
├──────────────────────┤
│  ┌────────────────┐  │
│  │ CARTE          │  │  ← Carte d'élément
│  │ 📄 #ID  BADGE  │  │
│  │ Nom...         │  │
│  │ 🟢 Priorité    │  │
│  │ 💰 Montant     │  │
│  │ 📅 Date        │  │
│  │ [← Préc] [Suiv→] │  │  ← Boutons de déplacement
│  └────────────────┘  │
│         ...          │
└──────────────────────┘
```

### Éléments d'une Carte

| Élément | Description |
|---------|-------------|
| **Badge** | Type (DEVIS/PROJET/BT) |
| **ID** | Identifiant unique |
| **Nom** | Nom du devis/projet/BT (30 caractères max) |
| **Priorité** | Icône + texte (CRITIQUE/URGENT/NORMAL) |
| **Montant** | Valeur en $ formatée |
| **Date** | Date d'échéance ou fin prévue |
| **Boutons** | ← Précédent / Suivant → |

### Couleurs de Bordure des Cartes

| Type | Couleur Bordure |
|------|-----------------|
| **Devis** | Orange `#f59e0b` |
| **Projet** | Bleu `#3b82f6` |
| **Bon de Travail** | Variable selon priorité |

---

## Fonctionnalités Principales

### 1. Déplacement de Cartes

Chaque carte dispose de boutons pour changer de statut :

| Bouton | Action |
|--------|--------|
| **← Précédent** | Déplace vers la colonne précédente |
| **Suivant →** | Déplace vers la colonne suivante |

**Règles de déplacement :**
- Le premier statut n'a pas de bouton "← Précédent"
- Le dernier statut n'a pas de bouton "Suivant →"
- Le changement est enregistré immédiatement en base
- L'historique des validations est conservé (pour les BT)

### 2. Visualisation du Nombre d'Éléments

Chaque en-tête de colonne affiche le compteur :
- Format : `STATUT (N)` où N = nombre d'éléments
- Ex: `EN COURS (5)`

### 3. Colonnes Vides

Une colonne sans élément affiche :
- Message "Aucun BT" / "Aucun projet" / "Aucun devis"
- Style italique grisé

### 4. Statistiques (BT uniquement)

Section de métriques en bas du Kanban BT :

| Métrique | Description |
|----------|-------------|
| **📋 Total BTs** | Nombre total de bons de travail |
| **🔄 En Cours** | Nombre au statut EN COURS |
| **✅ Terminés** | Nombre terminés + pourcentage |
| **🔴 Critiques** | Nombre à priorité CRITIQUE |

### 5. Actions Rapides (BT uniquement)

| Action | Description |
|--------|-------------|
| **🔧 Créer BTs de test** | Génère des BT de démonstration |
| **🔄 Actualiser** | Recharge les données |
| **📊 Total BTs** | Affiche le compteur total |

---

## Guide Pas-à-Pas

### Consulter le Kanban des Devis

1. Accédez au module **"🔄 Kanban"**
2. Par défaut, l'onglet **"Devis"** est actif
3. Les colonnes affichent vos devis par statut :
   - BROUILLON → EN_ATTENTE → APPROUVÉ → REFUSÉ → ANNULÉ
4. Identifiez rapidement les devis en attente d'action

### Déplacer un Devis

1. Localisez le devis dans sa colonne
2. Pour l'avancer : cliquez sur **"Suivant →"**
3. Pour le reculer : cliquez sur **"← Précédent"**
4. Le changement est confirmé par un message de succès
5. La page se rafraîchit automatiquement

### Consulter le Kanban des Projets

1. Cliquez sur l'onglet **"Projets"**
2. Les colonnes s'organisent :
   - EN_ATTENTE → EN_COURS → EN_PAUSE → TERMINÉ → ANNULÉ
3. Visualisez l'état de vos chantiers
4. Déplacez les projets selon leur avancement

### Gérer les Bons de Travail

1. Cliquez sur l'onglet **"Bons de Travail"**
2. Organisation des colonnes :
   - BROUILLON → VALIDÉ → EN COURS → TERMINÉ → ANNULÉ
3. Les cartes affichent plus d'informations :
   - Numéro de document
   - Projet lié
   - Priorité avec icône
   - Date d'échéance
   - Nombre d'employés assignés
4. Surveillez les BT en retard (⚠️)

### Voir les détails d'un BT

1. Dans le Kanban BT, localisez le bon de travail
2. Cliquez sur **"👁️ Voir détails"**
3. Un panneau s'ouvre avec :
   - ID et numéro
   - Statut et priorité
   - Date création et échéance
   - Projet lié
   - Client
   - Responsable
   - Notes
4. Cliquez sur **"❌ Fermer"** pour revenir

### Créer des BT de test

1. Si aucun BT n'existe, un message s'affiche
2. Cliquez sur **"🔧 Créer des BTs de test"**
3. 4 BT de démonstration sont créés :
   - BT-TEST-001 (BROUILLON, URGENT)
   - BT-TEST-002 (VALIDÉ, NORMAL)
   - BT-TEST-003 (EN COURS, CRITIQUE)
   - BT-TEST-004 (TERMINÉ, NORMAL)
4. Le Kanban se rafraîchit avec les nouveaux BT

---

## Mapping des Statuts

### Projets : Normalisation

Le système normalise automatiquement les variantes de statuts :

| Variantes Acceptées | Statut Normalisé |
|--------------------|------------------|
| EN ATTENTE, ATTENTE, NOUVEAU | EN_ATTENTE |
| EN COURS, ACTIF, OUVERT | EN_COURS |
| EN PAUSE, PAUSE, SUSPENDU | EN_PAUSE |
| TERMINÉ, TERMINE, COMPLET, FERMÉ | TERMINÉ |
| ANNULÉ, ANNULE, CANCELLED | ANNULÉ |

### BT : Mapping

| Statut Source | Colonne Kanban |
|---------------|----------------|
| APPROUVÉ | EN COURS |
| ENVOYÉ | VALIDÉ |
| Autre | BROUILLON |

### Devis : Mapping

| Statut Source | Statut Normalisé |
|---------------|------------------|
| APPROUVE | APPROUVÉ |
| REFUSE | REFUSÉ |
| ANNULE | ANNULÉ |

---

## Historique des Changements (BT)

Chaque changement de statut d'un BT est enregistré :

```sql
INSERT INTO formulaire_validations
(formulaire_id, type_validation, ancien_statut, nouveau_statut, commentaires)
VALUES (:bt_id, 'CHANGEMENT_STATUT', :ancien, :nouveau,
        'Déplacé via Kanban: :ancien → :nouveau')
```

---

## CSS et Style

### Classes CSS Principales

| Classe | Description |
|--------|-------------|
| `.kanban-container` | Conteneur principal |
| `.kanban-column` | Colonne de statut |
| `.kanban-header` | En-tête de colonne |
| `.kanban-cards` | Zone des cartes |
| `.kanban-card` | Carte individuelle |
| `.card-title` | Titre de carte |
| `.card-info` | Ligne d'information |
| `.drop-zone` | Zone de dépôt |
| `.empty-column` | Colonne vide |

### Dimensions

| Élément | Valeur |
|---------|--------|
| Hauteur min cartes | 80px |
| Hauteur max cartes | 500px (scroll) |
| Bordure carte | 1px solid #e5e7eb |
| Border-left carte | 4px solid (couleur priorité) |
| Border-radius | 8px |
| Padding carte | 12px |

### Animation

```css
.kanban-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}
```

---

## Requêtes SQL

### Récupération des BT

```sql
SELECT f.id, f.numero_document, f.type_formulaire, f.statut, f.priorite,
       f.date_creation, f.date_echeance, f.montant_total, f.notes,
       p.nom_projet,
       c.nom as company_nom,
       e.prenom || ' ' || e.nom as employee_nom,
       COUNT(DISTINCT bta.employe_id) as nb_employes_assignes
FROM formulaires f
LEFT JOIN projects p ON f.project_id = p.id
LEFT JOIN companies c ON f.company_id = c.id
LEFT JOIN employees e ON f.employee_id = e.id
LEFT JOIN bt_assignations bta ON f.id = bta.bt_id AND bta.statut = 'ASSIGNÉ'
WHERE f.type_formulaire = 'BON_TRAVAIL'
GROUP BY f.id, ...
ORDER BY
    CASE f.priorite
        WHEN 'CRITIQUE' THEN 1
        WHEN 'URGENT' THEN 2
        WHEN 'NORMAL' THEN 3
        ELSE 4
    END,
    f.date_echeance ASC NULLS LAST,
    f.date_creation DESC
```

### Mise à jour Statut

```sql
UPDATE formulaires
SET statut = :nouveau_statut,
    updated_at = CURRENT_TIMESTAMP
WHERE id = :bt_id
```

---

## Astuces et Bonnes Pratiques

- **Utilisez les priorités** : Les cartes sont triées par priorité (CRITIQUE en premier)
- **Surveillez les dates** : Les dates dépassées sont marquées ⚠️
- **Flux logique** : Déplacez les éléments de gauche à droite selon l'avancement
- **Colonnes vides** : Une colonne vide signifie aucun blocage à cette étape
- **Actualisation** : Utilisez le bouton 🔄 Actualiser si les données semblent obsolètes
- **Vue d'ensemble** : Le compteur dans chaque en-tête donne une vue rapide

---

## Résolution de Problèmes

### Les cartes ne se déplacent pas

- **Cause** : Erreur de mise à jour en base
- **Solution** : Vérifiez les logs et la connexion à la base de données

### Un statut n'est pas reconnu

- **Cause** : Statut non standard dans les données
- **Solution** : Le système mappe vers BROUILLON/EN_ATTENTE par défaut

### Le compteur est incorrect

- **Cause** : Cache non actualisé
- **Solution** : Cliquez sur **🔄 Actualiser**

### Les BT de test ne se créent pas

- **Cause** : Erreur de création en base
- **Solution** : Vérifiez qu'au moins un projet existe ou sera créé

---

## Questions Fréquentes (FAQ)

**Q: Puis-je glisser-déposer les cartes ?**
R: Non, utilisez les boutons ← Précédent / Suivant → pour déplacer les éléments. Le drag & drop natif n'est pas supporté par Streamlit.

**Q: Comment créer un nouveau devis depuis le Kanban ?**
R: Le Kanban est une vue de gestion, pas de création. Utilisez le module Devis pour créer.

**Q: Les changements sont-ils enregistrés immédiatement ?**
R: Oui, chaque clic sur les boutons de déplacement enregistre le changement en base.

**Q: Puis-je filtrer par projet ou client ?**
R: Pas directement dans le Kanban. Utilisez le Gantt ou les listes avec filtres pour cela.

**Q: Comment voir l'historique des déplacements ?**
R: Pour les BT, consultez la table `formulaire_validations` dans l'administration.

**Q: Le Kanban fonctionne-t-il sur mobile ?**
R: Oui, mais l'interface est optimisée pour desktop. Les colonnes s'empilent verticalement sur mobile.

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Gestion complète des projets
- [🧾 Devis](06-devis.md) - Création et gestion des devis
- [🏭 Production](11-production.md) - Bons de travail détaillés
- [📈 Gantt](09-gantt.md) - Vue chronologique
- [📅 Calendrier](08-calendrier.md) - Vue mensuelle
- [🤝 Ventes](05-ventes.md) - Pipeline commercial (Kanban CRM)
