# 🏢 Fonds de Prévoyance

## Introduction

Le module **Fonds de Prévoyance** est conçu pour les syndicats de copropriété conformément à la **Loi 16 du Québec** (2019). Il permet de gérer les études de fonds de prévoyance, planifier les travaux majeurs sur 25 ans, calculer les contributions nécessaires et générer les attestations requises pour maintenir les immeubles en bon état.

Ce module inclut 4 types de bâtiments, 4 types de structures, 3 types d'interventions et 5 statuts d'entretien pour une gestion complète des copropriétés.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🏢 Fonds Prévoyance"**
2. Le tableau de bord des copropriétés s'affiche avec les onglets :
   - **Copropriétés** : Liste des immeubles gérés
   - **Études** : Études de fonds de prévoyance
   - **Composantes** : Inventaire des éléments majeurs
   - **Entretiens** : Planification des travaux
   - **Attestations** : Documents officiels
3. Gérez vos études et projections financières

---

## Structure des Données

### Table PostgreSQL : `coproprietes`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom` | TEXT | Nom du syndicat de copropriété |
| `adresse` | TEXT | Adresse de l'immeuble |
| `ville` | TEXT | Ville |
| `code_postal` | TEXT | Code postal |
| `nombre_unites` | INTEGER | Nombre d'unités (condos) |
| `annee_construction` | INTEGER | Année de construction |
| `type_batiment` | TEXT | Résidentiel, Commercial, etc. |
| `type_structure` | TEXT | Bois, Béton, Acier, Mixte |
| `superficie_totale` | REAL | Superficie en m² |
| `etages` | INTEGER | Nombre d'étages |
| `created_at` | TIMESTAMP | Date de création |

### Table : `etudes_fonds_prevoyance`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `copropriete_id` | INTEGER | Lien vers copropriété |
| `annee_etude` | INTEGER | Année de l'étude |
| `duree_projection` | INTEGER | Durée en années (25 ans minimum) |
| `solde_actuel` | DECIMAL | Solde du fonds actuel |
| `contribution_actuelle` | DECIMAL | Contribution mensuelle actuelle |
| `contribution_recommandee` | DECIMAL | Contribution calculée |
| `contingence_pourcentage` | DECIMAL | Marge de sécurité (%) |
| `date_prochaine_revision` | DATE | Date de révision |
| `statut_conformite` | TEXT | Conforme, Non-conforme |
| `notes` | TEXT | Notes de l'étude |
| `created_at` | TIMESTAMP | Date de création |

### Table : `composantes_majeures`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `copropriete_id` | INTEGER | Lien vers copropriété |
| `categorie` | TEXT | Toiture, Enveloppe, etc. |
| `description` | TEXT | Description détaillée |
| `quantite` | REAL | Quantité (m², unités) |
| `unite` | TEXT | Unité de mesure |
| `annee_installation` | INTEGER | Année d'installation |
| `duree_vie_totale` | INTEGER | Durée de vie estimée |
| `duree_vie_residuelle` | INTEGER | Années restantes |
| `etat_actuel` | TEXT | Bon, Acceptable, À remplacer |
| `cout_remplacement` | DECIMAL | Coût estimé |
| `notes` | TEXT | Notes techniques |

### Table : `entretiens_planifies`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `copropriete_id` | INTEGER | Lien vers copropriété |
| `composante_id` | INTEGER | Lien vers composante |
| `type_intervention` | TEXT | Entretien, Réparation, Remplacement |
| `date_prevue` | DATE | Date planifiée |
| `cout_estime` | DECIMAL | Coût estimé |
| `statut` | TEXT | Planifié, En cours, Complété, etc. |
| `notes` | TEXT | Notes sur l'intervention |

### Table : `attestations_fonds`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `copropriete_id` | INTEGER | Lien vers copropriété |
| `date_demande` | DATE | Date de la demande |
| `date_emission` | DATE | Date d'émission |
| `montant_fonds_prevoyance` | DECIMAL | Solde attesté |
| `destinataire` | TEXT | Notaire, acheteur, etc. |
| `notes` | TEXT | Notes additionnelles |

---

## Types de Bâtiments

| Type | Description |
|------|-------------|
| **Résidentiel** | Copropriété d'habitation (condos) |
| **Commercial** | Immeuble commercial |
| **Mixte** | Combinaison résidentiel/commercial |
| **Industriel** | Bâtiment industriel |

---

## Types de Structures

| Type | Description | Durée de vie typique |
|------|-------------|----------------------|
| **Bois** | Charpente de bois | 75-100 ans |
| **Béton** | Structure de béton armé | 100+ ans |
| **Acier** | Charpente d'acier | 75-100 ans |
| **Mixte** | Combinaison de matériaux | Variable |

---

## Types d'Interventions

| Type | Description |
|------|-------------|
| **Entretien** | Maintenance préventive régulière |
| **Réparation** | Correction d'un problème |
| **Remplacement** | Remplacement complet de la composante |

---

## Statuts d'Entretien

| Statut | Description | Couleur |
|--------|-------------|---------|
| **Planifié** | Intervention programmée | 🔵 Bleu |
| **En cours** | Travaux en exécution | 🟠 Orange |
| **Complété** | Intervention terminée | 🟢 Vert |
| **Reporté** | Remis à plus tard | 🟡 Jaune |
| **Annulé** | Intervention annulée | 🔴 Rouge |

---

## Contexte Loi 16 (Québec)

La Loi 16 (2019) exige pour les copropriétés divises au Québec :

| Exigence | Détail |
|----------|--------|
| **Étude du fonds de prévoyance** | Obligatoire tous les 5 ans |
| **Plan de 25 ans minimum** | Projection des travaux sur 25 ans |
| **Inspection des parties communes** | Par un professionnel (ingénieur, architecte) |
| **Contribution adéquate** | Basée sur l'étude |
| **Carnet d'entretien** | Registre des interventions |
| **Attestation** | Pour les transactions immobilières |

### Échéances Loi 16

| Taille de la copropriété | Date limite |
|--------------------------|-------------|
| 5+ étages ou 50+ unités | 2024 (déjà passée) |
| Construites avant 2000 | 2025 |
| Autres copropriétés | 2026 |

---

## Fonctionnalités Principales

### 1. Gestion des Copropriétés

| Fonctionnalité | Description |
|----------------|-------------|
| **Fiche complète** | Informations de l'immeuble |
| **Caractéristiques** | Type, structure, superficie, étages |
| **Historique** | Études et interventions passées |
| **Documents** | Attachement de fichiers |

### 2. Études de Fonds de Prévoyance

| Fonctionnalité | Description |
|----------------|-------------|
| **Nouvelle étude** | Création d'une étude conforme Loi 16 |
| **Projection 25 ans** | Calcul des besoins futurs |
| **Contribution recommandée** | Calcul mensuel par unité |
| **Contingence** | Marge de sécurité (10-15% recommandé) |

### 3. Inventaire des Composantes

| Fonctionnalité | Description |
|----------------|-------------|
| **Catalogage** | Liste de toutes les composantes majeures |
| **État actuel** | Évaluation de l'état |
| **Durée de vie** | Vie totale et résiduelle |
| **Coût de remplacement** | Estimation actualisée |

### 4. Projections Financières

| Calcul | Description |
|--------|-------------|
| **Coût total** | Somme des remplacements prévus |
| **Contribution nécessaire** | Mensuelle par unité |
| **Évolution du fonds** | Solde projeté année par année |
| **Graphique 25 ans** | Visualisation des flux |

### 5. Attestations

| Fonctionnalité | Description |
|----------------|-------------|
| **Génération** | Création d'attestation officielle |
| **Solde certifié** | Montant du fonds à la date |
| **Destinataire** | Notaire, acheteur potentiel |
| **Historique** | Archivage des attestations émises |

---

## Guide Pas-à-Pas

### Créer une nouvelle copropriété

1. Onglet **"Copropriétés"**
2. Cliquez sur **"➕ Nouvelle copropriété"**
3. Entrez les informations générales :

   **Section Identification :**
   - Nom du syndicat
   - Adresse complète
   - Ville et code postal

   **Section Caractéristiques :**
   - Nombre d'unités
   - Année de construction
   - Type de bâtiment (Résidentiel, Commercial, Mixte, Industriel)
   - Type de structure (Bois, Béton, Acier, Mixte)
   - Superficie totale (m²)
   - Nombre d'étages

4. Cliquez sur **"💾 Enregistrer"**
5. La copropriété apparaît dans la liste

### Créer une étude de fonds de prévoyance

1. Ouvrez la fiche de la copropriété
2. Onglet **"Études"**
3. Cliquez sur **"➕ Nouvelle étude"**
4. Remplissez les paramètres :

   **Section Période :**
   - Année de l'étude (ex: 2025)
   - Durée de projection (minimum 25 ans)

   **Section Financière :**
   - Solde actuel du fonds
   - Contribution mensuelle actuelle
   - Contingence (%) (recommandé: 10-15%)

5. Le système calcule la contribution recommandée
6. Enregistrez l'étude

### Inventorier les composantes majeures

1. Dans la copropriété, onglet **"Composantes"**
2. Cliquez sur **"➕ Nouvelle composante"**
3. Pour chaque composante :

   **Section Identification :**
   - Catégorie (Toiture, Enveloppe, Mécanique, etc.)
   - Description détaillée
   - Quantité et unité (ex: 500 m²)

   **Section État :**
   - Année d'installation
   - Durée de vie totale (années)
   - Durée de vie résiduelle (années)
   - État actuel (Bon, Acceptable, À remplacer)

   **Section Coûts :**
   - Coût de remplacement estimé

4. Répétez pour toutes les composantes majeures
5. Le total des coûts est calculé automatiquement

### Générer les projections sur 25 ans

1. Avec les composantes saisies
2. Cliquez sur **"📊 Générer projections"**
3. Le système calcule :
   - Année prévue de chaque remplacement
   - Coût par année
   - Contribution annuelle nécessaire
   - Évolution du solde du fonds
4. Un graphique visualise les 25 prochaines années
5. Exportez le rapport en PDF

### Planifier un entretien

1. Onglet **"Entretiens"**
2. Cliquez sur **"➕ Nouvel entretien"**
3. Sélectionnez :
   - Composante concernée
   - Type d'intervention (Entretien, Réparation, Remplacement)
   - Date prévue
   - Coût estimé
   - Statut (Planifié)
4. Ajoutez des notes si nécessaire
5. Enregistrez
6. L'entretien apparaît dans le calendrier

### Émettre une attestation

1. Onglet **"Attestations"**
2. Cliquez sur **"➕ Nouvelle attestation"**
3. Remplissez :
   - Date de la demande
   - Destinataire (nom du notaire ou acheteur)
   - Le montant du fonds est calculé automatiquement
4. Cliquez sur **"📄 Générer l'attestation"**
5. Un document PDF est créé
6. Enregistrez dans l'historique

---

## Composantes Standard (selon RGCQ)

| Catégorie | Composantes | Durée de vie typique |
|-----------|-------------|----------------------|
| **Toiture** | Membrane élastomère, Bardeaux, Solins | 20-30 ans |
| **Fenêtres** | Cadres PVC/Alu, Vitrages, Moustiquaires | 25-30 ans |
| **Balcons** | Dalles béton, Garde-corps, Membrane | 30-40 ans |
| **Revêtement** | Maçonnerie, Bardage, Crépi | 30-50 ans |
| **Ascenseur** | Cabine, Mécanique, Contrôles | 25-30 ans |
| **Stationnement** | Membrane, Asphalte, Marquage | 20-25 ans |
| **Plomberie** | Colonnes, Valves, Chauffe-eau | 25-40 ans |
| **Électricité** | Panneaux, Câblage, Éclairage commun | 30-40 ans |
| **CVC** | Chaudière, Ventilation, Climatisation | 15-25 ans |

---

## Calcul de la Contribution Recommandée

```
Coût total des travaux sur 25 ans = Σ (Coûts de remplacement × facteur inflation)

Contribution annuelle = (Coût total - Solde actuel) / Durée projection
                        + (Contingence × Coût total / Durée)

Contribution mensuelle par unité = Contribution annuelle / 12 / Nombre d'unités
```

### Exemple de calcul

```
Copropriété : 20 unités
Coût total travaux 25 ans : 500 000$
Solde actuel du fonds : 50 000$
Contingence : 10%

Besoin net = 500 000$ - 50 000$ = 450 000$
Contingence = 500 000$ × 10% = 50 000$
Total à accumuler = 500 000$

Contribution annuelle = 500 000$ / 25 ans = 20 000$/an
Contribution mensuelle = 20 000$ / 12 = 1 667$/mois
Par unité = 1 667$ / 20 = 83$/mois/unité
```

---

## Astuces et Bonnes Pratiques

- **Faites inspecter régulièrement** : L'état réel peut différer des estimations
- **Mettez à jour les coûts** : L'inflation affecte les projections
- **Prévoyez une contingence** : 10-15% pour les imprévus
- **Communiquez aux copropriétaires** : Transparence sur les contributions
- **Archivez les études** : Historique pour les prochaines études
- **Documentez les travaux** : Photos avant/après, factures
- **Respectez les échéances Loi 16** : Évitez les pénalités
- **Consultez un professionnel** : Ingénieur ou architecte pour l'étude officielle

---

## Résolution de Problèmes

### La contribution calculée semble trop élevée

- **Cause** : Composantes à remplacer à court terme
- **Solution** : Répartissez sur une période plus longue ou revoyez les priorités

### L'étude n'est pas conforme Loi 16

- **Cause** : Durée de projection < 25 ans ou composantes manquantes
- **Solution** : Ajoutez les composantes requises et ajustez la durée

### Le solde projeté devient négatif

- **Cause** : Contribution insuffisante pour les travaux prévus
- **Solution** : Augmentez la contribution mensuelle ou reportez certains travaux

---

## Questions Fréquentes (FAQ)

**Q: Une étude est-elle obligatoire pour toutes les copropriétés ?**
R: Oui, la Loi 16 l'exige pour toutes les copropriétés divises au Québec, avec des échéances progressives selon la taille et l'âge.

**Q: Qui peut réaliser l'étude officielle ?**
R: Un professionnel (ingénieur ou architecte) doit inspecter et signer l'étude. Ce module vous aide à la préparer et à suivre les recommandations.

**Q: Comment gérer plusieurs immeubles pour un même syndicat ?**
R: Créez chaque bâtiment comme une copropriété distincte ou utilisez des sections dans une même fiche.

**Q: Les coûts incluent-ils l'inflation ?**
R: Oui, vous pouvez définir un taux d'inflation annuel pour ajuster les coûts futurs automatiquement.

**Q: À quoi sert l'attestation ?**
R: Lors d'une vente, le notaire demande une attestation du solde du fonds de prévoyance pour informer l'acheteur.

**Q: Que se passe-t-il si on ne respecte pas la Loi 16 ?**
R: Des pénalités peuvent s'appliquer et les transactions immobilières peuvent être retardées.

---

## Données Techniques

### Requête Copropriétés avec Études

```sql
SELECT c.*,
       e.annee_etude as derniere_etude,
       e.contribution_recommandee,
       e.statut_conformite
FROM coproprietes c
LEFT JOIN etudes_fonds_prevoyance e ON c.id = e.copropriete_id
ORDER BY c.nom
```

### Requête Composantes par Copropriété

```sql
SELECT cm.*,
       (cm.annee_installation + cm.duree_vie_totale) as annee_remplacement_prevue
FROM composantes_majeures cm
WHERE cm.copropriete_id = :copropriete_id
ORDER BY (cm.annee_installation + cm.duree_vie_totale)
```

### Requête Entretiens à Venir

```sql
SELECT e.*, c.description as composante_nom
FROM entretiens_planifies e
LEFT JOIN composantes_majeures c ON e.composante_id = c.id
WHERE e.statut IN ('Planifié', 'En cours')
  AND e.date_prevue >= CURRENT_DATE
ORDER BY e.date_prevue
```

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Projets de rénovation
- [🧾 Devis](06-devis.md) - Soumissions pour travaux
- [💰 Comptabilité](19-comptabilite.md) - Suivi du fonds
- [💵 Subventions](21-subventions.md) - Aides disponibles pour rénovation
