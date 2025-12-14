# 💵 Subventions

## Introduction

Le module **Subventions** répertorie plus de 50 programmes d'aide gouvernementale disponibles au Québec en 2025. Il vous permet d'identifier les subventions pertinentes pour vos projets, vérifier l'éligibilité, suivre vos demandes de financement et accéder aux ressources utiles.

Ce module inclut 5 types d'aide, 4 niveaux de gouvernement, 3 niveaux de difficulté, 9 statuts de demande et 19 secteurs d'activité pour une couverture complète des programmes disponibles.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"💵 Subventions"**
2. Le catalogue des programmes s'affiche avec les onglets :
   - **Programmes** : Liste des subventions disponibles
   - **Mes demandes** : Suivi de vos demandes
   - **Ressources** : Contacts et liens utiles
   - **Statistiques** : Tableau de bord
3. Recherchez et gérez vos demandes de subvention

---

## Structure des Données

### Table PostgreSQL : `subventions_categories`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `code` | TEXT | Code de la catégorie |
| `nom` | TEXT | Nom de la catégorie |
| `description` | TEXT | Description |
| `icone` | TEXT | Icône affichée |
| `ordre_affichage` | INTEGER | Ordre de tri |

### Table : `subventions_programmes`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `categorie_id` | INTEGER | Lien vers catégorie |
| `nom` | TEXT | Nom du programme |
| `organisme` | TEXT | Organisme responsable |
| `description` | TEXT | Description détaillée |
| `type_aide` | TEXT | Subvention, Prêt, Crédit d'impôt, etc. |
| `niveau_gouvernement` | TEXT | Fédéral, Provincial, Municipal, Mixte |
| `montant_min` | DECIMAL | Aide minimale |
| `montant_max` | DECIMAL | Aide maximale |
| `pourcentage_max` | INTEGER | % des coûts admissibles |
| `secteurs_eligible` | TEXT | Secteurs éligibles |
| `url_programme` | TEXT | Lien vers le site officiel |
| `telephone` | TEXT | Numéro de contact |
| `difficulte_demande` | TEXT | Facile, Moyen, Complexe |
| `actif` | BOOLEAN | Programme actif |
| `date_expiration` | DATE | Date de fin du programme |

### Table : `subventions_demandes`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `programme_id` | INTEGER | Lien vers programme |
| `project_id` | INTEGER | Projet associé |
| `numero_dossier` | TEXT | Numéro de dossier externe |
| `date_soumission` | DATE | Date de soumission |
| `montant_demande` | DECIMAL | Montant demandé |
| `montant_approuve` | DECIMAL | Montant approuvé |
| `statut` | TEXT | État de la demande |
| `notes` | TEXT | Notes internes |
| `created_at` | TIMESTAMP | Date de création |

---

## Types d'Aide

| Type | Icône | Description |
|------|-------|-------------|
| **Subvention** | 💰 | Aide non remboursable |
| **Prêt** | 🏦 | Prêt à conditions avantageuses |
| **Crédit d'impôt** | 📋 | Réduction d'impôt |
| **Mixte** | 🔄 | Combinaison de plusieurs types |
| **Garantie** | 🛡️ | Garantie de prêt |

---

## Niveaux de Gouvernement

| Niveau | Icône | Exemples d'organismes |
|--------|-------|----------------------|
| **Fédéral** | 🍁 | BDC, NRCan, Innovation Canada |
| **Provincial** | ⚜️ | Investissement Québec, TEQ, MIFI |
| **Municipal** | 🏛️ | Ville de Montréal, CMM |
| **Mixte** | 🤝 | Programmes conjoints |

---

## Niveaux de Difficulté

| Niveau | Icône | Description |
|--------|-------|-------------|
| **Facile** | 🟢 | Formulaire simple, peu de documents |
| **Moyen** | 🟡 | Documentation standard requise |
| **Complexe** | 🔴 | Dossier complet, plan d'affaires requis |

---

## Statuts des Demandes

| Statut | Icône | Description |
|--------|-------|-------------|
| **Brouillon** | 📝 | Demande en préparation |
| **En préparation** | 🔧 | Documents en cours de collecte |
| **Soumise** | 📤 | Demande envoyée à l'organisme |
| **En évaluation** | 🔍 | Analyse par l'organisme |
| **Info requise** | ❓ | Documents supplémentaires demandés |
| **Approuvée** | ✅ | Subvention accordée |
| **Refusée** | ❌ | Demande refusée |
| **Annulée** | 🚫 | Demande retirée |
| **Versée** | 💵 | Fonds reçus |

---

## Secteurs d'Activité Éligibles

| # | Secteur | Description |
|---|---------|-------------|
| 1 | **PME** | Petites et moyennes entreprises |
| 2 | **Construction** | Entreprises de construction |
| 3 | **Rénovation** | Rénovation résidentielle et commerciale |
| 4 | **Manufacturier** | Secteur manufacturier |
| 5 | **Énergie** | Efficacité énergétique |
| 6 | **Logement** | Logement social et abordable |
| 7 | **Commercial** | Bâtiments commerciaux |
| 8 | **Résidentiel** | Construction résidentielle |
| 9 | **Numérique** | Transformation numérique |
| 10 | **Formation** | Formation et développement |
| 11 | **Employeur** | Aide aux employeurs |
| 12 | **Exportateur** | Développement international |
| 13 | **Startup** | Entreprises en démarrage |
| 14 | **Démarrage** | Création d'entreprise |
| 15 | **Repreneuriat** | Acquisition d'entreprise |
| 16 | **Rural** | Régions rurales |
| 17 | **Faible revenu** | Ménages à faible revenu |
| 18 | **Patrimoine** | Bâtiments patrimoniaux |
| 19 | **Bois** | Industrie du bois |

---

## Catégories de Subventions (8)

| Catégorie | Exemples de programmes |
|-----------|----------------------|
| **PME** | ESSOR, Aide au démarrage, Croissance PME |
| **Construction** | Novoclimat, Rénovation verte, Accessibilité |
| **Énergie** | Rénoclimat, Chauffez vert, Écoperformance |
| **Innovation** | R&D, Technologies vertes, CNRC-PARI |
| **Formation** | PAMT, Compétences, Formation continue |
| **Export** | Développement international, CanExport |
| **Environnement** | Décontamination, Recyclage, Verdissement |
| **Régional** | Fonds locaux, SADC, MRC |

---

## Programmes Populaires 2025

| Programme | Organisme | Type | Aide max | Difficulté |
|-----------|-----------|------|----------|------------|
| **Rénoclimat** | TEQ | Subvention | 12 500$ | 🟢 Facile |
| **Chauffez vert** | TEQ | Subvention | 1 500$ | 🟢 Facile |
| **Novoclimat** | TEQ | Subvention | 8 000$ | 🟡 Moyen |
| **LogiRénov** | Revenu Québec | Crédit d'impôt | 10 000$ | 🟢 Facile |
| **Écoperformance** | TEQ | Mixte | Variable | 🔴 Complexe |
| **ESSOR** | Investissement Québec | Prêt | 500 000$+ | 🔴 Complexe |
| **PEEIC** | NRCan | Subvention | Variable | 🟡 Moyen |
| **Accessibilité** | SHQ | Subvention | 8 000$ | 🟡 Moyen |

---

## Ressources et Contacts Utiles

| Organisme | Rôle | Contact |
|-----------|------|---------|
| **Réseau Accès PME** | 500+ professionnels pour accompagnement | Via votre MRC |
| **Investissement Québec** | Administration programmes ESSOR et autres | 1 844 474-6367 |
| **SADC** | Sociétés d'aide au développement des collectivités | Variable selon région |
| **APCHQ** | Association des professionnels de la construction | apchq.com |
| **MicroEntreprendre** | Microcrédit aux entrepreneurs | microentreprendre.ca |
| **Annuaire des subventions** | 2 696 programmes de soutien financier | subventionsquebec.net |
| **Gouvernement du Canada** | Outil de recherche d'aide aux entreprises | canada.ca |
| **Gouvernement du Québec** | Programmes provinciaux | quebec.ca |

---

## Fonctionnalités Principales

### 1. Catalogue des Programmes

| Fonctionnalité | Description |
|----------------|-------------|
| **Recherche** | Par mot-clé, catégorie, organisme |
| **Filtres** | Type d'aide, niveau gouvernement, montant |
| **Détails** | Fiche complète du programme |
| **Liens** | Accès direct au site officiel |

### 2. Vérification d'Éligibilité

| Critère | Description |
|---------|-------------|
| **Type d'entreprise** | Incorporée, enregistrée, OBNL |
| **Secteur d'activité** | Construction, rénovation, etc. |
| **Taille** | Nombre d'employés, chiffre d'affaires |
| **Localisation** | Région, zone désignée |
| **Nature du projet** | Investissement, formation, R&D |

### 3. Suivi des Demandes

| Fonctionnalité | Description |
|----------------|-------------|
| **Création** | Nouvelle demande depuis un programme |
| **Statut** | Suivi de l'avancement (9 statuts) |
| **Documents** | Liste des pièces à fournir |
| **Historique** | Journal des modifications |

### 4. Alertes et Notifications

| Alerte | Description |
|--------|-------------|
| **Nouveaux programmes** | Dans vos secteurs d'intérêt |
| **Dates limites** | Programmes qui expirent bientôt |
| **Changements de statut** | Mise à jour de vos demandes |

---

## Guide Pas-à-Pas

### Rechercher une subvention

1. Ouvrez le module **Subventions**
2. Utilisez la barre de recherche ou les filtres :
   - **Catégorie** : Construction, Énergie, etc.
   - **Type d'aide** : Subvention, Prêt, Crédit d'impôt
   - **Niveau** : Fédéral, Provincial, Municipal
   - **Montant minimum** : Aide minimale requise
3. Parcourez les résultats
4. Cliquez sur un programme pour voir les détails

### Consulter les détails d'un programme

1. Cliquez sur le programme dans la liste
2. La fiche affiche :
   - Description complète
   - Critères d'éligibilité
   - Montant min/max et % couvert
   - Documents requis
   - Date limite (si applicable)
   - Niveau de difficulté
3. Cliquez sur le lien pour accéder au site officiel
4. Notez le numéro de téléphone pour plus d'informations

### Vérifier votre éligibilité

1. Ouvrez la fiche du programme
2. Cliquez sur **"🎯 Vérifier éligibilité"**
3. Répondez au questionnaire :
   - Type d'organisation
   - Nombre d'employés
   - Chiffre d'affaires annuel
   - Localisation (région)
   - Type de projet
4. Le système indique :
   - ✅ **Éligible** : Vous répondez aux critères
   - ⚠️ **Possiblement éligible** : Vérification requise
   - ❌ **Non éligible** : Critères non satisfaits
5. Consultez les recommandations

### Créer une demande de subvention

1. Sur un programme éligible
2. Cliquez sur **"➕ Nouvelle demande"**
3. Associez le projet concerné (optionnel)
4. Remplissez les informations :

   **Section Demande :**
   - Montant demandé
   - Description du projet
   - Numéro de dossier (si déjà attribué)

   **Section Budget :**
   - Coût total du projet
   - Coûts admissibles

5. Ajoutez des notes internes
6. Sauvegardez (statut: Brouillon)

### Soumettre une demande

1. Ouvrez la demande en brouillon
2. Vérifiez que tous les documents sont prêts
3. Cliquez sur le lien du programme officiel
4. Soumettez votre demande sur le portail de l'organisme
5. De retour dans CONSTRUCTO AI :
   - Mettez à jour le statut à "Soumise"
   - Entrez le numéro de dossier reçu
   - Notez la date de soumission
6. Le suivi continue dans "Mes demandes"

### Suivre l'avancement d'une demande

1. Allez dans **"Mes demandes"**
2. Consultez le statut de chaque demande
3. Cliquez pour voir les détails :
   - Date de soumission
   - Numéro de dossier
   - Montant demandé vs approuvé
   - Notes et correspondance
4. Mettez à jour le statut selon les communications reçues
5. Ajoutez des notes à chaque étape

### Configurer les alertes

1. Dans les paramètres du module
2. Activez les alertes pour :
   - Nouveaux programmes dans vos catégories
   - Dates limites approchant (30 jours avant)
   - Changements de statut de vos demandes
3. Choisissez le mode de notification

---

## Système de Cache

| Donnée | TTL | Description |
|--------|-----|-------------|
| Catégories | 1 heure | Liste des catégories |
| Programmes | 1 heure | Catalogue des programmes actifs |
| Statistiques | 5 min | Données calculées |

---

## Astuces et Bonnes Pratiques

- **Anticipez** : Certains programmes ont des budgets limités (premier arrivé, premier servi)
- **Documentez tout** : Gardez les preuves d'achat, contrats et factures
- **Respectez les critères** : Un écart peut invalider la demande
- **Combinez** : Certains programmes sont cumulables (vérifiez les conditions)
- **Consultez un expert** : Pour les demandes complexes (ESSOR, Écoperformance)
- **Suivez les échéances** : Dates de fin de programme, dates de soumission
- **Préparez les documents** : Plan d'affaires, états financiers, soumissions
- **Contactez l'organisme** : Posez vos questions avant de soumettre

---

## Résolution de Problèmes

### Le programme n'apparaît pas dans la recherche

- **Cause** : Programme expiré ou filtres actifs
- **Solution** : Vérifiez les filtres et les dates d'expiration

### L'éligibilité indique "Non éligible" alors que je crois l'être

- **Cause** : Critères stricts ou information manquante
- **Solution** : Contactez l'organisme pour confirmation

### Le montant approuvé est inférieur au montant demandé

- **Cause** : Coûts non admissibles ou plafond atteint
- **Solution** : Consultez les détails de l'approbation

---

## Questions Fréquentes (FAQ)

**Q: Les montants sont-ils à jour ?**
R: Le catalogue est mis à jour régulièrement. Vérifiez toujours sur le site officiel de l'organisme avant de soumettre.

**Q: Puis-je cumuler plusieurs subventions ?**
R: Certaines sont cumulables, d'autres non. Le système indique la compatibilité connue, mais vérifiez les règles de chaque programme.

**Q: Les demandes peuvent-elles être faites directement depuis CONSTRUCTO AI ?**
R: Le système prépare et organise votre dossier, mais la soumission officielle se fait généralement sur le portail de l'organisme.

**Q: Comment savoir si je suis éligible au crédit d'impôt RénoVert ?**
R: Utilisez le vérificateur d'éligibilité. Les critères incluent le type de résidence, les travaux admissibles et le montant minimum.

**Q: Combien de temps pour recevoir une réponse ?**
R: Variable selon le programme : de 2 semaines (simples) à 6 mois (complexes).

**Q: Que faire si ma demande est refusée ?**
R: Analysez les motifs de refus, corrigez les lacunes et vérifiez si vous pouvez resoumettre ou postuler à un autre programme.

---

## Données Techniques

### Requête Programmes par Catégorie

```sql
SELECT p.*, c.nom as categorie_nom
FROM subventions_programmes p
LEFT JOIN subventions_categories c ON p.categorie_id = c.id
WHERE p.actif = TRUE
  AND (p.date_expiration IS NULL OR p.date_expiration > CURRENT_DATE)
ORDER BY c.ordre_affichage, p.nom
```

### Requête Statistiques

```sql
SELECT
    COUNT(*) as total_programmes,
    COUNT(CASE WHEN type_aide = 'SUBVENTION' THEN 1 END) as subventions,
    COUNT(CASE WHEN type_aide = 'PRET' THEN 1 END) as prets,
    COUNT(CASE WHEN niveau_gouvernement = 'PROVINCIAL' THEN 1 END) as provinciales
FROM subventions_programmes
WHERE actif = TRUE
```

### Requête Demandes par Statut

```sql
SELECT statut, COUNT(*) as count, COALESCE(SUM(montant_demande), 0) as total_demande
FROM subventions_demandes
GROUP BY statut
ORDER BY count DESC
```

### Requête Programmes Expirant Bientôt

```sql
SELECT p.*, c.nom as categorie_nom
FROM subventions_programmes p
LEFT JOIN subventions_categories c ON p.categorie_id = c.id
WHERE p.actif = TRUE
  AND p.date_expiration BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '30 days'
ORDER BY p.date_expiration
```

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Projets à financer
- [🧾 Devis](06-devis.md) - Budget pour la demande
- [💰 Comptabilité](19-comptabilite.md) - Suivi des versements
- [🏢 Fonds Prévoyance](20-fonds-prevoyance.md) - Travaux copropriété
