# 👥 Utilisateurs

## Introduction

Le module **Utilisateurs** permet de gérer les comptes utilisateurs de votre entreprise dans CONSTRUCTO AI. Créez des comptes, assignez des rôles, définissez les permissions et contrôlez l'accès aux différentes fonctionnalités de l'application.

Ce module est compatible avec l'architecture multi-tenant : chaque entreprise gère ses propres utilisateurs de manière isolée.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"👥 Utilisateurs"**
2. L'interface s'affiche avec 3 onglets :
   - **Liste des utilisateurs** : Vue d'ensemble
   - **Créer un utilisateur** : Nouveau compte
   - **Modifier/Supprimer** : Gestion existants
3. Gérez les comptes et les permissions

> **Note** : Ce module est réservé aux administrateurs (👑 Admin).

---

## Structure des Données

### Table PostgreSQL : `users`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `username` | TEXT | Nom d'utilisateur (unique) |
| `full_name` | TEXT | Nom complet |
| `email` | TEXT | Adresse courriel |
| `password_hash` | TEXT | Mot de passe hashé (bcrypt) |
| `is_admin` | BOOLEAN | Administrateur (Oui/Non) |
| `active` | BOOLEAN | Compte actif |
| `created_at` | TIMESTAMP | Date de création |
| `last_login` | TIMESTAMP | Dernière connexion |

---

## Fonctionnalités Principales

### 1. Gestion des Comptes

| Champ | Description | Format |
|-------|-------------|--------|
| **Nom d'utilisateur** | Identifiant de connexion | prenom.nom |
| **Courriel** | Adresse email | email@domaine.com |
| **Nom complet** | Prénom et nom | Jean Tremblay |
| **Rôle** | Niveau d'accès | 👑 Admin / 👷 Employé |
| **Statut** | Actif / Inactif | ✅ Actif / ❌ Inactif |
| **Dernière connexion** | Date et heure | 2025-01-15 14:30 |

### 2. Les 5 Rôles Disponibles

| Rôle | Icône | Niveau | Accès |
|------|-------|--------|-------|
| **Super Admin** | 👑 | Maximum | Tout + Multi-tenant + Configuration système |
| **Admin** | 👑 | Élevé | Configuration entreprise + Gestion utilisateurs |
| **Manager** | 👔 | Moyen | Projets + Équipe + Rapports |
| **Employé** | 👷 | Limité | Ses tâches + Pointage + Consultation |
| **Lecture seule** | 👁️ | Minimal | Consultation uniquement (aucune modification) |

### 3. Permissions Granulaires

Chaque module peut être configuré :
- ✅ Lecture (voir)
- ✅ Écriture (créer/modifier)
- ✅ Suppression
- ✅ Export
- ❌ Non autorisé

### 4. Sécurité

- Mots de passe sécurisés (bcrypt)
- Rate limiting (5 tentatives max)
- Session timeout (30 min)
- Journalisation des accès

---

## Guide Pas-à-Pas

### Créer un nouvel utilisateur

1. Cliquez sur **"➕ Nouvel utilisateur"**
2. Remplissez les informations :
   - Nom d'utilisateur (unique)
   - Courriel
   - Nom complet
   - Mot de passe temporaire
3. Sélectionnez le rôle
4. Cochez **"Forcer changement de mot de passe"**
5. Cliquez sur **"Créer"**
6. L'utilisateur reçoit ses identifiants par courriel

### Modifier les permissions d'un utilisateur

1. Trouvez l'utilisateur dans la liste
2. Cliquez sur **"🔒 Permissions"**
3. Pour chaque module, définissez :
   - Lecture seule
   - Lecture et écriture
   - Accès complet
   - Aucun accès
4. Sauvegardez les modifications
5. Les changements sont effectifs immédiatement

### Désactiver un utilisateur

1. Ouvrez la fiche de l'utilisateur
2. Cliquez sur **"🚫 Désactiver"**
3. Confirmez l'action
4. L'utilisateur ne peut plus se connecter
5. Ses données sont conservées

### Réinitialiser un mot de passe

1. Trouvez l'utilisateur concerné
2. Cliquez sur **"🔑 Réinitialiser mot de passe"**
3. Choisissez :
   - Générer automatiquement
   - Définir manuellement
4. Cochez **"Envoyer par courriel"**
5. L'utilisateur recevra le nouveau mot de passe

### Consulter le journal des connexions

1. Ouvrez la fiche de l'utilisateur
2. Onglet **"📋 Historique"**
3. Consultez :
   - Dates et heures de connexion
   - Adresses IP
   - Appareils utilisés
   - Tentatives échouées

---

## Bonnes Pratiques de Sécurité

### Pour les Administrateurs
- Appliquez le principe du moindre privilège
- Révisez les accès périodiquement
- Désactivez les comptes des anciens employés immédiatement
- Activez l'authentification à deux facteurs si disponible

### Pour les Utilisateurs
- Utilisez des mots de passe forts (12+ caractères)
- Ne partagez jamais vos identifiants
- Déconnectez-vous sur les appareils partagés
- Signalez toute activité suspecte

---

## Astuces et Bonnes Pratiques

- **Nommez clairement** : prenom.nom ou initiales.nom
- **Rôles standards** : Évitez les permissions personnalisées excessives
- **Revue trimestrielle** : Vérifiez que les accès sont toujours pertinents
- **Formation** : Assurez-vous que chacun connaît ses accès
- **Backup** : Documentez qui a accès à quoi

---

## Questions Fréquentes (FAQ)

**Q: Combien d'utilisateurs puis-je créer ?**
R: Le nombre dépend de votre forfait. Consultez les détails dans Abonnement.

**Q: Puis-je créer des groupes d'utilisateurs ?**
R: Actuellement, les rôles prédéfinis servent de groupes. Des groupes personnalisés sont prévus.

**Q: Comment récupérer un compte administrateur bloqué ?**
R: Contactez le support CONSTRUCTO AI avec une preuve d'identité.

**Q: Les sessions expirent-elles automatiquement ?**
R: Oui, après 30 minutes d'inactivité, l'utilisateur est déconnecté automatiquement.

---

## Voir Aussi

- [⚙️ Configuration](27-configuration.md) - Paramètres système
- [💳 Abonnement](28-abonnement.md) - Gestion du forfait
- [👥 Employés](12-employes.md) - Dossiers RH (différent des comptes utilisateurs)
