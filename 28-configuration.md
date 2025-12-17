# ⚙️ Configuration

## Introduction

Le module **Configuration** centralise tous les paramètres de votre entreprise et de l'application CONSTRUCTO AI. Personnalisez les informations de votre entreprise, configurez les modules, gérez les intégrations API et ajustez les préférences système.

Ce module stocke la configuration dans une table dédiée avec support JSON pour une flexibilité maximale.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"⚙️ Configuration"**
2. Les sections de paramètres s'affichent :
   - **Entreprise** : Informations générales
   - **Commercial** : Marges et conditions
   - **Intégrations** : API et webhooks
   - **Import/Export** : Données
3. Naviguez vers la section souhaitée

> **Note** : Certains paramètres sont réservés aux administrateurs.

---

## Structure des Données

### Table PostgreSQL : `entreprise_config`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Identifiant unique |
| `config_data` | TEXT (JSON) | Configuration complète en JSON |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

---

## Paramètres de Configuration

### 1. Informations de l'Entreprise

| Paramètre | Champ JSON | Exemple |
|-----------|------------|---------|
| **Raison sociale** | `nom` | Constructo AI Inc. |
| **Adresse** | `adresse` | 1760 rue Jacques-Cartier Sud |
| **Ville** | `ville` | Farnham |
| **Province** | `province` | Québec |
| **Code postal** | `code_postal` | J2N 1Y8 |
| **Téléphone bureau** | `telephone_bureau` | (514) 820-1972 |
| **Téléphone cellulaire** | `telephone_cellulaire` | (514) 820-1972 |
| **Courriel** | `email` | info@constructoai.ca |
| **Site web** | `site_web` | www.constructoai.ca |

### 2. Numéros Légaux et Fiscaux

| Paramètre | Champ JSON | Format |
|-----------|------------|--------|
| **Licence RBQ** | `rbq` | 1234-5678-01 |
| **NEQ** | `neq` | 1234567890 |
| **Numéro TPS** | `tps` | 123456789RT0001 |
| **Numéro TVQ** | `tvq` | 1234567890TQ0001 |

### 3. Contact Principal

| Paramètre | Champ JSON | Description |
|-----------|------------|-------------|
| **Nom** | `contact_principal_nom` | Nom du dirigeant |
| **Titre** | `contact_principal_titre` | Directeur Général |
| **Téléphone** | `contact_principal_telephone` | Ligne directe |
| **Courriel** | `contact_principal_email` | Email professionnel |

### 4. Paramètres Commerciaux (Valeurs par Défaut)

| Paramètre | Champ JSON | Valeur | Description |
|-----------|------------|--------|-------------|
| **Administration** | `taux_administration` | 3.0% | Frais de gestion |
| **Contingences** | `taux_contingences` | 12.0% | Imprévus |
| **Profit** | `taux_profit` | 15.0% | Marge bénéficiaire |
| **Validité devis** | `delai_validite_soumission` | 30 jours | Durée de validité |

### 5. Conditions de Paiement par Défaut

```
30% à la signature
35% début des travaux
Paiements progressifs selon avancement
35% retenue finale
```

### 6. Garanties par Défaut

```
1 an main-d'œuvre
5 ans toiture
10 ans structure
Selon normes GCR
```

### 7. Personnalisation Visuelle

| Paramètre | Champ JSON | Valeur par défaut |
|-----------|------------|-------------------|
| **Couleur primaire** | `couleur_primaire` | #374151 (gris foncé) |
| **Couleur secondaire** | `couleur_secondaire` | #4b5563 (gris moyen) |
| **Couleur accent** | `couleur_accent` | #3b82f6 (bleu) |
| **Slogan** | `slogan` | Excellence en construction |
| **Logo** | `logo_base64` | Image en base64 |

---

## Intégrations Externes

### 1. Stripe (Paiements)

| Paramètre | Variable d'environnement | Description |
|-----------|-------------------------|-------------|
| **Clé secrète** | `STRIPE_SECRET_KEY` | Clé API Stripe |
| **Clé publique** | `STRIPE_PUBLISHABLE_KEY` | Clé frontend |
| **ID Prix** | `STRIPE_PRICE_ID` | ID du plan d'abonnement |
| **Secret webhook** | `STRIPE_WEBHOOK_SECRET` | Signature des webhooks |

### 2. API REST CONSTRUCTO AI

| Fonctionnalité | Description |
|----------------|-------------|
| **Clés API** | Génération de clés `cai_live_xxxx...` |
| **Permissions** | Lecture seule ou lecture/écriture |
| **Expiration** | Date d'expiration optionnelle |
| **Rate limiting** | Limites par minute/heure |

### 3. Webhooks

| Événement | Payload |
|-----------|---------|
| `facture.created` | Nouvelle facture créée |
| `facture.paid` | Facture payée |
| `devis.accepted` | Devis accepté |
| `projet.updated` | Projet modifié |

### 4. Email (SMTP)

Configuration via le module Emails pour l'envoi automatique.

---

## Types MIME pour Logo

| Extension | Type MIME |
|-----------|-----------|
| PNG | image/png |
| JPG/JPEG | image/jpeg |
| GIF | image/gif |
| SVG | image/svg+xml |

---

## Guide Pas-à-Pas

### Configurer les informations de l'entreprise

1. Allez dans **Configuration** > **"Entreprise"**
2. Remplissez les champs :
   - Raison sociale
   - Adresse complète
   - Téléphone et courriel
3. Téléchargez votre **logo** (PNG, JPG)
4. Entrez les numéros fiscaux :
   - Numéro TPS
   - Numéro TVQ
   - NEQ (si applicable)
5. Enregistrez les modifications

### Personnaliser les marges commerciales

1. Allez dans **Configuration** > **"Commercial"**
2. Ajustez les taux :
   - Administration : 0-10%
   - Contingences : 5-20%
   - Profit : 10-25%
3. Ces taux seront appliqués par défaut aux nouveaux devis
4. Sauvegardez

### Configurer les notifications

1. Allez dans **Configuration** > **"Notifications"**
2. Pour chaque type d'événement :
   - Activez/Désactivez
   - Choisissez le canal (app, email)
3. Types d'événements :
   - Facture en retard
   - Devis expirant
   - Nouvelle demande
   - Stock bas
4. Enregistrez

### Configurer l'API REST

1. Allez dans **Configuration** > **"Intégrations"**
2. Section **"API REST"**
3. Cliquez sur **"➕ Nouvelle clé API"**
4. Définissez :
   - Nom de la clé
   - Permissions (lecture, écriture)
   - Date d'expiration (optionnel)
5. La clé est générée : `cai_live_xxxx...`
6. Copiez-la immédiatement (non réaffichée)

### Configurer un webhook

1. Allez dans **Configuration** > **"Webhooks"**
2. Cliquez sur **"➕ Nouveau webhook"**
3. Configurez :
   - URL de destination
   - Événements à envoyer (facture.created, etc.)
   - Secret de signature (HMAC)
4. Testez avec **"Envoyer un test"**
5. Activez le webhook

---

## Sauvegarde et Export

### Exporter les données
1. **Configuration** > **"Import/Export"**
2. Sélectionnez les données à exporter :
   - Clients
   - Projets
   - Factures
   - etc.
3. Format : CSV ou JSON
4. Téléchargez l'archive

### Importer des données
1. Préparez votre fichier selon le modèle fourni
2. **Configuration** > **"Import/Export"**
3. Sélectionnez le type de données
4. Téléchargez votre fichier
5. Validez le mapping des colonnes
6. Lancez l'import

---

## Astuces et Bonnes Pratiques

- **Complétez tout** : Les informations entreprise apparaissent sur vos documents
- **Testez les webhooks** : Avant de les utiliser en production
- **Sauvegardez régulièrement** : Exportez vos données périodiquement
- **Révisez les accès API** : Supprimez les clés non utilisées
- **Documentez vos paramètres** : Pour la continuité en cas de changement

---

## Questions Fréquentes (FAQ)

**Q: Les modifications sont-elles effectives immédiatement ?**
R: Oui, sauf les paramètres qui nécessitent un redémarrage de session (rare).

**Q: Puis-je avoir différentes configurations par projet ?**
R: Certains paramètres (marges) peuvent être surchargés au niveau projet.

**Q: Comment réinitialiser les paramètres par défaut ?**
R: Contactez le support. Une réinitialisation complète est possible mais irréversible.

**Q: Les données importées remplacent-elles les existantes ?**
R: Par défaut, l'import ajoute de nouvelles entrées. Les doublons sont signalés.

---

## Voir Aussi

- [👥 Utilisateurs](26-utilisateurs.md) - Gestion des accès
- [💳 Abonnement](28-abonnement.md) - Forfait et facturation
- [🧾 Devis](06-devis.md) - Paramètres commerciaux
- [💰 Comptabilité](19-comptabilite.md) - Paramètres financiers
