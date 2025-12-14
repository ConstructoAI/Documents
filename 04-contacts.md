# 👥 Contacts

## Introduction

Le module **Contacts** centralise toutes les personnes avec lesquelles vous interagissez : clients, décideurs, fournisseurs, sous-traitants, chefs de projet, etc. Chaque contact est lié à une entreprise et peut être associé à des projets, opportunités et interactions pour un suivi complet de vos relations d'affaires.

Ce module fait partie intégrante du système CRM de CONSTRUCTO AI et est interconnecté avec les modules Entreprises, Ventes (Opportunités), Projets et Interactions.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"👥 Contacts"**
2. Le répertoire des contacts s'affiche sous forme de tableau
3. Utilisez la recherche ou les filtres pour trouver un contact

---

## Structure des Données

### Table PostgreSQL : `contacts`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique auto-généré |
| `prenom` | TEXT | Prénom du contact |
| `nom_famille` | TEXT | Nom de famille |
| `email` | TEXT | Adresse courriel |
| `telephone` | TEXT | Numéro de téléphone |
| `company_id` | INTEGER | FK vers `companies.id` |
| `role_poste` | TEXT | Poste/fonction dans l'entreprise |
| `notes` | TEXT | Notes internes |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Alias de Compatibilité

Le système utilise des alias pour la compatibilité :
- `entreprise_id` → `company_id`
- `role` → `role_poste`

---

## Fonctionnalités Principales

### 1. Gestion des Contacts (CRUD)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouveau contact | Ajouter un contact |
| **Lire** | 👁️ Voir | Consulter les détails |
| **Modifier** | ✏️ Modifier | Éditer les informations |
| **Supprimer** | 🗑️ Supprimer | Retirer de la base |

### 2. Informations du Contact

Pour chaque contact, enregistrez :

| Champ | Obligatoire | Description |
|-------|-------------|-------------|
| **Prénom** | Oui | Prénom de la personne |
| **Nom de famille** | Oui | Nom de famille |
| **Courriel** | Non | Adresse email professionnelle |
| **Téléphone** | Non | Numéro direct ou mobile |
| **Entreprise** | Non | Lien vers la fiche entreprise |
| **Poste/Titre** | Non | Fonction dans l'entreprise |
| **Notes** | Non | Informations complémentaires |

### 3. Liaison avec les Entreprises

Un contact peut être :
- **Contact principal** d'une entreprise (affiché en premier)
- **Contact secondaire** pour des projets spécifiques
- **Non lié** (contact indépendant)

### 4. Projets Liés

Les projets associés au contact sont automatiquement récupérés :
- Via l'entreprise du contact (`client_company_id`)
- Affichés sous forme de liste dans la fiche

### 5. Historique des Interactions

Suivi de toutes les communications :
- Appels téléphoniques
- Courriels envoyés/reçus
- Réunions planifiées et effectuées
- Notes et mémos internes
- Visites de chantier

---

## Interface Utilisateur

### Liste des Contacts

| Colonne | Contenu |
|---------|---------|
| **Prénom** | Prénom du contact |
| **Nom** | Nom de famille |
| **Email** | Adresse courriel |
| **Téléphone** | Numéro de téléphone |
| **Poste** | Fonction/titre |
| **Entreprise** | Nom de l'entreprise liée |
| **Projets liés** | Liste des projets associés |
| **Actions** | Boutons Voir/Modifier/Supprimer |

### Barre de Recherche

- Recherche instantanée sur prénom, nom, entreprise
- Filtrage en temps réel
- Résultats mis à jour dynamiquement

### Filtres Disponibles

| Filtre | Options |
|--------|---------|
| **Entreprise** | Liste des entreprises |
| **Poste** | Liste des postes |
| **Avec/Sans email** | Filtrer par présence d'email |

---

## Types d'Interactions

| Type | Icône | Description | Couleur |
|------|-------|-------------|---------|
| **Email** | 📧 | Correspondance électronique | `#4B5563` (gris) |
| **Appel** | 📞 | Communication téléphonique | `#10B981` (vert) |
| **Réunion** | 🤝 | Rencontre en personne ou virtuelle | `#3B82F6` (bleu) |
| **Note** | 📝 | Mémo interne | `#F59E0B` (orange) |
| **Visite** | 🏢 | Visite de chantier ou bureau | `#8B5CF6` (violet) |
| **Présentation** | 📊 | Présentation commerciale | `#EF4444` (rouge) |
| **Suivi** | 🔄 | Activité de suivi | `#06B6D4` (cyan) |
| **Tâche** | ✅ | Tâche à réaliser | `#84CC16` (lime) |

---

## Guide Pas-à-Pas

### Créer un nouveau contact

1. Cliquez sur **"➕ Nouveau contact"**
2. Remplissez le formulaire :

   **Section Identité :**
   - Prénom (obligatoire)
   - Nom de famille (obligatoire)

   **Section Coordonnées :**
   - Courriel
   - Téléphone

   **Section Entreprise :**
   - Sélectionnez l'entreprise dans la liste déroulante
   - Ou laissez vide si contact indépendant

   **Section Poste :**
   - Poste/Titre dans l'entreprise
   - Notes internes

3. Cliquez sur **"💾 Enregistrer"**
4. Le contact est créé avec un ID unique

### Rechercher un contact

1. Utilisez la **barre de recherche** en haut
2. Tapez le nom, prénom ou nom d'entreprise
3. Les résultats apparaissent instantanément
4. Cliquez sur un contact pour voir sa fiche complète

### Modifier un contact

1. Trouvez le contact dans la liste
2. Cliquez sur **"✏️ Modifier"**
3. Modifiez les champs souhaités
4. Cliquez sur **"💾 Enregistrer les modifications"**
5. Le cache est automatiquement rafraîchi

### Supprimer un contact

1. Cliquez sur **"🗑️ Supprimer"**
2. **Confirmation demandée** : "Êtes-vous sûr ?"
3. Les données associées sont nettoyées :
   - Interactions du contact (supprimées)
   - Référence contact_principal dans companies (mise à NULL)
4. Suppression définitive

### Enregistrer une interaction

1. Ouvrez la fiche du contact
2. Cliquez sur **"➕ Nouvelle interaction"**
3. Remplissez le formulaire :
   - **Type** : Email, Appel, Réunion, Note, etc.
   - **Date et heure** de l'interaction
   - **Résumé** : Description courte (100 caractères max)
   - **Détails** : Notes complètes
   - **Résultat** : Positif, Neutre, Négatif, En cours, À suivre
   - **Suivi prévu** : Date du prochain contact
4. Cliquez sur **"💾 Enregistrer"**
5. L'interaction est liée automatiquement à l'opportunité ouverte (si existe)

### Définir comme contact principal

1. Allez dans le module **Entreprises**
2. Modifiez l'entreprise concernée
3. Dans le champ **"Contact principal"**
4. Sélectionnez le contact souhaité
5. Enregistrez l'entreprise

---

## Automatisations

### Liaison Automatique aux Opportunités

Lors de la création d'une interaction :
1. Le système recherche l'opportunité ouverte pour ce contact/entreprise
2. L'interaction est automatiquement liée à l'opportunité
3. La date de dernière activité de l'opportunité est mise à jour

### Création Automatique d'Activité de Suivi

Si une date de suivi est définie dans l'interaction :
1. Une activité de type "Suivi" est créée automatiquement
2. Planifiée à la date de suivi à 09h00
3. Liée au contact, à l'entreprise et à l'opportunité
4. Rappel activé 15 minutes avant

### Événement Calendrier

Chaque interaction crée un événement calendrier :
- Type : INTERACTION
- Titre : "[Type]: [Résumé]"
- Couleur selon le type d'interaction

---

## Système de Cache

### Performance Optimisée

| Donnée | TTL | Raison |
|--------|-----|--------|
| Liste contacts | 10 minutes | Données stables |
| Contacts par entreprise | 5 minutes | Requête fréquente |

### Invalidation Automatique

Le cache est invalidé lors de :
- Création d'un contact
- Modification d'un contact
- Suppression d'un contact

### Synchronisation Séquences

Lors de l'ajout d'un contact, le système :
1. Vérifie le MAX(id) dans la table
2. Resynchronise la séquence `contacts_id_seq`
3. Garantit l'unicité des IDs

---

## Interconnexions

### Modules Connectés

| Module | Liaison | Champ |
|--------|---------|-------|
| **Entreprises** | N-1 | `company_id` |
| **Opportunités** | 1-N | `contact_id` |
| **Interactions** | 1-N | `contact_id` |
| **Activités CRM** | 1-N | `contact_id` |
| **Projets** | Via entreprise | `client_company_id` |

### Cascade de Suppression

Lors de la suppression d'un contact :
1. Les interactions sont supprimées
2. Les références `contact_principal_id` dans `companies` sont mises à NULL
3. Le contact est supprimé

---

## Timeline Unifiée

### Vue Chronologique

Le système offre une timeline unifiée regroupant :
- Interactions (emails, appels, réunions)
- Activités CRM (tâches, suivis)
- Opportunités créées

### Requête Unifiée

```sql
SELECT 'interaction' as type, id, date_interaction as date, resume as titre...
UNION ALL
SELECT 'activite' as type, id, date_activite as date, sujet as titre...
UNION ALL
SELECT 'opportunite' as type, id, created_at as date, nom as titre...
ORDER BY date DESC
LIMIT 50
```

---

## Astuces et Bonnes Pratiques

- **Documentez chaque interaction** : L'historique est précieux pour la reprise de dossier
- **Vérifiez les doublons** : Avant de créer, recherchez si le contact existe
- **Mettez à jour les coordonnées** : Les gens changent de poste et de numéro
- **Utilisez les notes** : Notez les préférences, horaires de disponibilité
- **Liez toujours à une entreprise** : Facilite le suivi et les rapports
- **Planifiez les suivis** : Ne laissez pas un contact sans prochaine action
- **Format téléphone** : Utilisez un format cohérent (ex: 514-555-1234)

---

## Résolution de Problèmes

### Le contact n'apparaît pas après création

- **Cause** : Cache non invalidé ou erreur de séquence
- **Solution** : Rafraîchissez la page (F5) ou vérifiez les logs

### Impossible de supprimer un contact

- **Cause** : Le contact est contact principal d'une entreprise
- **Solution** : Modifiez d'abord l'entreprise pour désassigner le contact principal

### L'entreprise ne s'affiche pas dans la liste déroulante

- **Cause** : Cache des entreprises non rafraîchi
- **Solution** : Attendez 10 minutes ou rafraîchissez la page

### Erreur "Duplicate key" lors de la création

- **Cause** : Séquence désynchronisée
- **Solution** : Le système corrige automatiquement, réessayez

---

## Questions Fréquentes (FAQ)

**Q: Comment associer un contact à plusieurs entreprises ?**
R: Actuellement, un contact est lié à une seule entreprise. Pour des consultants multi-entreprises, créez une note avec les autres affiliations.

**Q: Les contacts sont-ils synchronisés avec mon téléphone ?**
R: Pas automatiquement. Exportez vos contacts en CSV depuis Configuration > Import/Export, puis importez dans votre téléphone.

**Q: Comment voir tous les contacts d'une entreprise ?**
R: Utilisez le filtre "Entreprise" dans la liste des contacts, ou ouvrez la fiche entreprise qui affiche les contacts associés.

**Q: Puis-je envoyer un courriel directement depuis la fiche contact ?**
R: Oui, cliquez sur l'adresse courriel pour ouvrir votre client email, ou utilisez le module Emails intégré pour un suivi dans l'application.

**Q: Comment voir l'historique complet d'un contact ?**
R: Ouvrez la fiche contact et consultez l'onglet "Historique" ou "Timeline" qui affiche toutes les interactions chronologiquement.

**Q: Puis-je importer mes contacts depuis Excel ?**
R: Oui, exportez votre fichier Excel en CSV avec les colonnes : prenom, nom_famille, email, telephone, company_id (optionnel), role_poste (optionnel). Importez via Configuration > Import/Export.

---

## Données Techniques

### Requête SQL de Récupération

```sql
SELECT
    c.*,
    co.nom as company_nom,
    (SELECT STRING_AGG(p.nom_projet, '; ')
     FROM projects p
     WHERE p.client_company_id = c.company_id) as projets_lies
FROM contacts c
LEFT JOIN companies co ON c.company_id = co.id
ORDER BY c.nom_famille, c.prenom
```

### Validation des Données

| Champ | Règle |
|-------|-------|
| prenom | Obligatoire, texte non vide |
| nom_famille | Obligatoire, texte non vide |
| email | Format email valide (optionnel) |
| telephone | Format libre (optionnel) |

---

## Voir Aussi

- [🏢 Entreprises](03-entreprises.md) - Gestion des entreprises
- [🤝 Ventes](05-ventes.md) - Pipeline commercial (Opportunités)
- [📧 Emails](22-emails.md) - Communication par courriel
- [💬 Conférence](23-conference.md) - Réunions virtuelles
- [📅 Calendrier](08-calendrier.md) - Planification des activités
