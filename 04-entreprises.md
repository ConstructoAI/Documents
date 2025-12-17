# 🏢 Entreprises

## Introduction

Le module **Entreprises** est le cœur de votre gestion commerciale CRM. Il vous permet de gérer l'ensemble de vos clients, prospects, fournisseurs et partenaires d'affaires dans un répertoire centralisé, spécialement conçu pour le secteur de la construction au Québec.

Ce module fait partie du système CRM (Customer Relationship Management) de CONSTRUCTO AI et est interconnecté avec les modules Contacts, Ventes (Opportunités), Projets et Devis.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🏢 Entreprises"**
2. La liste de vos entreprises s'affiche sous forme de tableau
3. Utilisez les filtres et la recherche pour naviguer rapidement

---

## Structure des Données

### Table PostgreSQL : `companies`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique auto-généré |
| `nom` | TEXT | Nom légal de l'entreprise |
| `type_company` | TEXT | Type d'entreprise |
| `secteur_activite` | TEXT | Secteur d'activité |
| `adresse` | TEXT | Numéro et rue |
| `ville` | TEXT | Ville |
| `province` | TEXT | Province (défaut: Québec) |
| `code_postal` | TEXT | Code postal format A1A 1A1 |
| `pays` | TEXT | Pays (défaut: Canada) |
| `telephone` | TEXT | Téléphone principal |
| `email` | TEXT | Courriel principal |
| `site_web` | TEXT | URL du site web |
| `contact_principal_id` | INTEGER | FK vers `contacts.id` |
| `statut_relation` | TEXT | Statut de la relation |
| `notes` | TEXT | Notes internes |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

---

## Types d'Entreprises Construction

Le module propose 14 types d'entreprises spécifiques au secteur de la construction au Québec :

| Type | Description | Usage typique |
|------|-------------|---------------|
| **Entrepreneur général** | Entreprise principale de construction | Clients principaux |
| **Sous-traitant spécialisé** | Électricien, plombier, couvreur, etc. | Partenaires de projet |
| **Promoteur immobilier** | Développeur de projets | Clients institutionnels |
| **Fournisseur matériaux** | Quincaillerie, distributeur | Achats et approvisionnement |
| **Consultant/Ingénieur** | Services professionnels | Expertise externe |
| **Architecte** | Conception de plans | Partenaires conception |
| **Arpenteur-géomètre** | Services d'arpentage | Projets terrain |
| **Organisme de contrôle** | Inspection, certification | Conformité |
| **Institution financière** | Banques, caisses | Financement projets |
| **Assureur** | Compagnies d'assurance | Couverture chantier |
| **Client résidentiel** | Particulier propriétaire | Rénovations maison |
| **Client commercial** | Entreprise cliente | Projets commerciaux |
| **Client industriel** | Usine, manufacture | Projets industriels |
| **Municipalité** | Organisme public local | Projets publics |

---

## Secteurs d'Activité Construction

18 secteurs d'activité spécialisés :

| Secteur | Description |
|---------|-------------|
| Construction résidentielle | Maisons neuves |
| Construction commerciale | Bureaux, commerces |
| Construction industrielle | Usines, entrepôts |
| Rénovation résidentielle | Travaux maison existante |
| Rénovation commerciale | Travaux bâtiments commerciaux |
| Excavation et terrassement | Préparation terrain |
| Fondations spécialisées | Béton, pilotis |
| Charpenterie générale | Structure bois |
| Couverture et toiture | Toits, bardeaux |
| Plomberie et chauffage | Systèmes sanitaires |
| Électricité du bâtiment | Installations électriques |
| Isolation et étanchéité | Performance thermique |
| Revêtements extérieurs | Façades, parement |
| Finition intérieure | Gypse, peinture |
| Aménagement paysager | Extérieur, verdure |
| Démolition | Déconstruction |
| Location d'équipements | Machinerie |
| Transport construction | Livraison matériaux |

---

## Statuts de Relation

| Statut | Couleur | Description |
|--------|---------|-------------|
| **PROSPECT** | Bleu | Contact initial, potentiel client |
| **ACTIF** | Vert | Relation d'affaires en cours |
| **CLIENT** | Vert foncé | Client établi avec historique |
| **FOURNISSEUR** | Orange | Fournisseur approuvé |
| **PARTENAIRE** | Violet | Partenaire stratégique |
| **INACTIF** | Gris | Relation en pause |
| **ARCHIVE** | Gris foncé | Entreprise archivée |

---

## Fonctionnalités Principales

### 1. Gestion des Entreprises (CRUD)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouvelle entreprise | Ajouter une entreprise |
| **Lire** | 👁️ Voir | Consulter les détails |
| **Modifier** | ✏️ Modifier | Éditer les informations |
| **Supprimer** | 🗑️ Supprimer | Retirer de la base |

### 2. Adresses Structurées

Le système utilise des adresses structurées pour une meilleure intégration :

```
Format d'adresse complète :
1550 Boul. Saint-Martin Ouest
Laval, Québec H7S 1N4
Canada
```

| Champ | Exemple |
|-------|---------|
| Adresse | 1550 Boul. Saint-Martin Ouest |
| Ville | Laval |
| Province | Québec |
| Code postal | H7S 1N4 |
| Pays | Canada |

### 3. Contact Principal

Chaque entreprise peut avoir un **contact principal** désigné :
- Liaison vers le module Contacts
- Affiché directement dans la liste des entreprises
- Utilisé par défaut pour la communication

### 4. Projets Liés

Les projets associés à l'entreprise sont automatiquement affichés :
- Via le champ `client_company_id` dans la table `projects`
- Historique complet des collaborations
- Accès rapide aux détails du projet

### 5. Interactions et Timeline

Toutes les interactions avec l'entreprise sont tracées :
- Appels, emails, réunions
- Notes et suivis
- Historique chronologique

---

## Interface Utilisateur

### Liste des Entreprises

| Colonne | Contenu |
|---------|---------|
| **Nom** | Nom de l'entreprise |
| **Téléphone** | Numéro principal |
| **Email** | Courriel principal |
| **Ville** | Localisation |
| **Statut** | Badge coloré du statut |
| **Type** | Type d'entreprise |
| **Secteur** | Secteur d'activité |
| **Contact Principal** | Nom du contact lié |
| **Actions** | Boutons Voir/Modifier/Supprimer |

### Barre de Recherche

- Recherche instantanée sur tous les champs
- Filtrage par nom, ville, secteur, type
- Résultats en temps réel

### Filtres Disponibles

| Filtre | Options |
|--------|---------|
| **Type d'entreprise** | Liste des 14 types |
| **Secteur d'activité** | Liste des 18 secteurs |
| **Statut relation** | Prospect, Actif, Client, etc. |
| **Ville** | Liste des villes présentes |

---

## Guide Pas-à-Pas

### Créer une nouvelle entreprise

1. Cliquez sur **"➕ Nouvelle entreprise"**
2. Remplissez le formulaire :

   **Section Identification :**
   - Nom de l'entreprise (obligatoire)
   - Type d'entreprise (sélection)
   - Secteur d'activité (sélection)

   **Section Adresse :**
   - Adresse (numéro et rue)
   - Ville
   - Province (Québec par défaut)
   - Code postal (format A1A 1A1)
   - Pays (Canada par défaut)

   **Section Contact :**
   - Téléphone principal
   - Courriel
   - Site web (optionnel)

   **Section Relation :**
   - Statut de relation
   - Contact principal (sélection)
   - Notes internes

3. Cliquez sur **"💾 Enregistrer"**
4. Un message de confirmation s'affiche

### Modifier une entreprise

1. Trouvez l'entreprise dans la liste
2. Cliquez sur **"✏️ Modifier"** dans la colonne Actions
3. Le formulaire de modification s'ouvre avec les valeurs actuelles
4. Modifiez les champs souhaités
5. Cliquez sur **"💾 Enregistrer les modifications"**
6. Le cache est automatiquement invalidé

### Supprimer une entreprise

**Attention** : La suppression vérifie d'abord les dépendances.

1. Cliquez sur **"🗑️ Supprimer"**
2. Le système vérifie si l'entreprise est utilisée dans :
   - Des projets (bloque la suppression)
   - Des devis/soumissions (bloque la suppression)
3. Si aucune dépendance bloquante :
   - Confirmation demandée
   - Les données secondaires sont nettoyées automatiquement
4. Si des dépendances existent :
   - Message d'erreur avec la liste des utilisations
   - Action requise : modifier ou supprimer les éléments liés d'abord

### Associer un contact principal

1. Ouvrez le formulaire de modification de l'entreprise
2. Dans le champ **"Contact principal"**
3. Sélectionnez un contact existant dans la liste déroulante
4. Ou créez d'abord un nouveau contact dans le module Contacts
5. Enregistrez les modifications

### Consulter les projets associés

1. Ouvrez la fiche de l'entreprise (mode Voir)
2. La section **"Projets liés"** affiche automatiquement :
   - Liste des noms de projets
   - Séparés par des points-virgules
3. Cliquez sur un projet pour accéder aux détails

---

## Système de Cache

### Performance Optimisée

Le module utilise un cache pour améliorer les performances :

| Donnée | TTL (durée) | Raison |
|--------|-------------|--------|
| Liste entreprises | 10 minutes | Données relativement stables |
| Détails entreprise | 5 minutes | Accès fréquent |

### Invalidation Automatique

Le cache est invalidé automatiquement lors de :
- Création d'une entreprise
- Modification d'une entreprise
- Suppression d'une entreprise

---

## Interconnexions

### Modules Connectés

| Module | Type de liaison | Champ de liaison |
|--------|-----------------|------------------|
| **Contacts** | 1-N | `company_id` |
| **Projets** | 1-N | `client_company_id` |
| **Devis** | 1-N | `client_id` |
| **Opportunités** | 1-N | `company_id` |
| **Interactions** | 1-N | `company_id` |
| **Factures** | 1-N | `company_id` |
| **Fournisseurs** | 1-1 | `company_id` |

### Cascade de Suppression

Lors de la suppression d'une entreprise, le système :

1. **Bloque** si utilisée dans :
   - Projets
   - Devis/Soumissions

2. **Nettoie automatiquement** :
   - Contacts (company_id → NULL)
   - Interactions (supprimées)
   - Documents (company_id → NULL)
   - Formulaires (company_id → NULL)
   - Opportunités (supprimées)
   - Activités CRM (supprimées)
   - Sous-traitants (supprimés)
   - Fournisseurs (supprimés)

---

## Import/Export

### Import CSV

1. Allez dans **Configuration > Import/Export**
2. Sélectionnez **"Entreprises"**
3. Téléchargez le modèle CSV
4. Remplissez avec vos données
5. Colonnes requises : nom, type_company, secteur
6. Colonnes optionnelles : adresse, ville, province, code_postal, telephone, email
7. Importez le fichier
8. Validez le mapping
9. Confirmez l'import

### Export

1. Dans la liste des entreprises
2. Cliquez sur **"📥 Exporter"**
3. Choisissez le format : CSV ou JSON
4. Le fichier est téléchargé

---

## Astuces et Bonnes Pratiques

- **Complétez toujours l'adresse** : Indispensable pour les devis et la facturation
- **Définissez le type correctement** : Facilite le filtrage et les rapports
- **Ajoutez un contact principal** : Communication plus efficace
- **Mettez à jour les statuts** : Gardez votre base de données à jour
- **Utilisez les notes** : Documentez les particularités de chaque relation
- **Vérifiez les doublons** : Avant de créer, recherchez si l'entreprise existe
- **Format code postal** : Utilisez le format A1A 1A1 pour le Québec

---

## Résolution de Problèmes

### L'entreprise n'apparaît pas dans la liste

- **Cause** : Cache non invalidé
- **Solution** : Rafraîchissez la page (F5) ou attendez 10 minutes

### Impossible de supprimer une entreprise

- **Cause** : L'entreprise est liée à des projets ou devis
- **Solution** : Supprimez ou réassignez d'abord les projets/devis liés

### Le contact principal ne s'affiche pas

- **Cause** : Le contact n'existe plus ou liaison incorrecte
- **Solution** : Modifiez l'entreprise et resélectionnez un contact

### Erreur lors de la création

- **Cause** : Champ obligatoire manquant
- **Solution** : Vérifiez que le nom est rempli

---

## Questions Fréquentes (FAQ)

**Q: Comment importer une liste d'entreprises existante ?**
R: Utilisez la fonction d'import CSV dans Configuration > Import/Export. Téléchargez le modèle, remplissez-le et importez.

**Q: Puis-je avoir plusieurs contacts par entreprise ?**
R: Oui, une entreprise peut avoir plusieurs contacts (tous accessibles dans le module Contacts). Un seul est désigné comme "contact principal".

**Q: Comment fusionner deux fiches entreprise en double ?**
R: Actuellement, la fusion n'est pas automatique. Transférez manuellement les données vers une fiche, puis supprimez le doublon.

**Q: Les entreprises supprimées sont-elles récupérables ?**
R: Non, la suppression est définitive. Les données secondaires sont nettoyées automatiquement.

**Q: Comment voir l'historique des modifications ?**
R: Les champs `created_at` et `updated_at` indiquent les dates. Pour un historique détaillé, consultez les logs système.

**Q: Puis-je créer des champs personnalisés ?**
R: Actuellement, les champs sont fixes. Utilisez le champ "Notes" pour des informations supplémentaires.

---

## Données Techniques

### Requête SQL de Récupération

```sql
SELECT c.*,
       co.prenom || ' ' || co.nom_famille as contact_principal_nom,
       (SELECT STRING_AGG(p.nom_projet, '; ')
        FROM projects p
        WHERE p.client_company_id = c.id) as projets_lies
FROM companies c
LEFT JOIN contacts co ON c.contact_principal_id = co.id
ORDER BY c.nom
```

### Validation des Données

| Champ | Règle |
|-------|-------|
| nom | Obligatoire, texte non vide |
| email | Format email valide (optionnel) |
| telephone | Format téléphone (optionnel) |
| code_postal | Format A1A 1A1 recommandé |

---

## Voir Aussi

- [👥 Contacts](04-contacts.md) - Gestion des personnes
- [🤝 Ventes](05-ventes.md) - Pipeline commercial (Opportunités)
- [🧾 Devis](06-devis.md) - Créer des soumissions
- [📋 Projets](07-projets.md) - Gestion des projets
- [💰 Comptabilité](19-comptabilite.md) - Facturation
- [🏪 Achats](16-achats.md) - Gestion fournisseurs
