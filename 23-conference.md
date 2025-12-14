# 💬 Conférence

## Introduction

Le module **Conférence** est un système de messagerie et de collaboration intégré, similaire à Slack ou Microsoft Teams. Il permet à votre équipe de communiquer en temps réel via des canaux de discussion, de partager des fichiers, de réagir aux messages et de rester informée grâce aux notifications.

Ce module inclut 4 rôles de membres, des canaux publics et privés, la gestion des réactions, les mentions et les notifications en temps réel.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"💬 Conférence"**
2. L'interface de messagerie s'affiche avec :
   - **Liste des canaux** : À gauche
   - **Messages** : Au centre
   - **Détails/Membres** : À droite (optionnel)
3. Rejoignez ou créez des canaux pour collaborer

---

## Structure des Données

### Table PostgreSQL : `conference_channels`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `name` | TEXT | Nom du canal |
| `description` | TEXT | Description |
| `is_private` | BOOLEAN | Canal privé (sur invitation) |
| `is_active` | BOOLEAN | Canal actif |
| `created_by` | INTEGER | Créateur du canal |
| `project_id` | INTEGER | Lien vers projet (optionnel) |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Table : `conference_members`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `channel_id` | INTEGER | Lien vers canal |
| `user_id` | INTEGER | Lien vers utilisateur |
| `role` | TEXT | Rôle dans le canal |
| `joined_at` | TIMESTAMP | Date d'adhésion |
| `last_read_at` | TIMESTAMP | Dernière lecture |

### Table : `conference_messages`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `channel_id` | INTEGER | Lien vers canal |
| `user_id` | INTEGER | Auteur du message |
| `content` | TEXT | Contenu du message |
| `parent_message_id` | INTEGER | Réponse à (fil) |
| `is_edited` | BOOLEAN | Message modifié |
| `is_deleted` | BOOLEAN | Message supprimé |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Table : `conference_attachments`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `message_id` | INTEGER | Lien vers message |
| `filename` | TEXT | Nom du fichier |
| `file_path` | TEXT | Chemin de stockage |
| `file_size` | INTEGER | Taille en octets |
| `mime_type` | TEXT | Type MIME |
| `created_at` | TIMESTAMP | Date d'ajout |

### Table : `conference_reactions`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `message_id` | INTEGER | Lien vers message |
| `user_id` | INTEGER | Utilisateur |
| `emoji` | TEXT | Emoji de réaction |
| `created_at` | TIMESTAMP | Date d'ajout |

### Table : `conference_notifications`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `user_id` | INTEGER | Destinataire |
| `channel_id` | INTEGER | Canal concerné |
| `message_id` | INTEGER | Message concerné |
| `type` | TEXT | Type de notification |
| `is_read` | BOOLEAN | Notification lue |
| `created_at` | TIMESTAMP | Date de création |

---

## Rôles des Membres

| Rôle | Description | Permissions |
|------|-------------|-------------|
| **owner** | Propriétaire du canal | Toutes les permissions, suppression du canal |
| **admin** | Administrateur | Gestion des membres, modération |
| **moderator** | Modérateur | Suppression de messages, épinglage |
| **member** | Membre | Lecture, écriture, réactions |

---

## Types de Canaux

| Type | Visibilité | Accès |
|------|------------|-------|
| **Public** | Visible par tous | Tout utilisateur peut rejoindre |
| **Privé** | Sur invitation | Membres invités uniquement |
| **Projet** | Lié à un projet | Équipe du projet automatiquement |

---

## Types de Notifications

| Type | Déclencheur | Description |
|------|-------------|-------------|
| **new_message** | Nouveau message | Message dans un canal rejoint |
| **mention** | @utilisateur | Vous êtes mentionné |
| **reply** | Réponse | Réponse à votre message |
| **reaction** | Réaction | Quelqu'un réagit à votre message |

---

## Fonctionnalités Principales

### 1. Canaux de Discussion

| Fonctionnalité | Description |
|----------------|-------------|
| **Création** | Créer des canaux publics ou privés |
| **Organisation** | Par projet, équipe ou sujet |
| **Description** | Définir le but du canal |
| **Membres** | Gérer qui peut accéder |

### 2. Messages

| Fonctionnalité | Description |
|----------------|-------------|
| **Texte** | Messages formatés (gras, italique, etc.) |
| **Fils** | Réponses organisées en fils de discussion |
| **Mentions** | @utilisateur pour notifier |
| **Édition** | Modifier ses propres messages |
| **Suppression** | Supprimer ses messages |

### 3. Pièces Jointes

| Fonctionnalité | Description |
|----------------|-------------|
| **Documents** | PDF, Word, Excel |
| **Images** | PNG, JPG, GIF |
| **Prévisualisation** | Aperçu des images inline |
| **Téléchargement** | Télécharger les fichiers |

### 4. Réactions

| Fonctionnalité | Description |
|----------------|-------------|
| **Emojis** | Réagir avec des emojis |
| **Multiple** | Plusieurs réactions par message |
| **Compteur** | Voir qui a réagi |
| **Retrait** | Retirer sa réaction |

### 5. Notifications

| Fonctionnalité | Description |
|----------------|-------------|
| **Badge** | Compteur de non lus |
| **Mentions** | Notification immédiate |
| **Paramètres** | Configurer par canal |

---

## Guide Pas-à-Pas

### Créer un canal

1. Cliquez sur **"➕"** à côté de "Canaux"
2. Remplissez les informations :

   **Section Informations :**
   - Nom du canal (ex: "equipe-chantier-laval")
   - Description (but du canal)

   **Section Paramètres :**
   - Type : Public ou Privé
   - Projet associé (optionnel)

3. Cliquez sur **"✅ Créer le canal"**
4. Le canal est créé et vous en êtes le propriétaire

### Rejoindre un canal public

1. Dans la liste des canaux, cliquez sur **"Explorer"**
2. Parcourez les canaux publics disponibles
3. Cliquez sur le canal qui vous intéresse
4. Cliquez sur **"Rejoindre"**
5. Vous êtes ajouté comme membre

### Inviter des membres à un canal privé

1. Ouvrez le canal privé
2. Cliquez sur **"👥 Membres"** en haut à droite
3. Cliquez sur **"➕ Inviter"**
4. Recherchez l'utilisateur par nom
5. Sélectionnez-le et confirmez
6. L'utilisateur est ajouté et notifié

### Envoyer un message

1. Ouvrez le canal souhaité
2. Cliquez dans la zone de saisie en bas
3. Tapez votre message
4. Utilisez le formatage si besoin :
   - `**gras**` pour le gras
   - `*italique*` pour l'italique
   - `\`code\`` pour le code inline
5. Appuyez sur **Entrée** ou cliquez sur **"📤"**
6. Le message est envoyé

### Mentionner quelqu'un

1. Dans votre message, tapez **@**
2. Commencez à taper le nom de la personne
3. Sélectionnez dans la liste qui apparaît
4. La personne sera notifiée du message
5. Exemple : "@jean.tremblay peux-tu vérifier ceci?"

### Répondre dans un fil

1. Survolez le message auquel vous voulez répondre
2. Cliquez sur **"💬 Répondre"**
3. Le fil de discussion s'ouvre
4. Tapez votre réponse
5. Envoyez
6. La réponse apparaît groupée avec le message original

### Joindre un fichier

1. Dans la zone de message, cliquez sur **"📎"**
2. Sélectionnez le fichier à joindre
3. Le fichier s'ajoute au message
4. Ajoutez un commentaire si souhaité
5. Envoyez
6. Le fichier est disponible au téléchargement

### Ajouter une réaction

1. Survolez le message
2. Cliquez sur **"😊"** (icône emoji)
3. Sélectionnez l'emoji de réaction
4. La réaction s'ajoute au message
5. Pour retirer : cliquez à nouveau sur votre réaction

### Modifier un message

1. Survolez votre message
2. Cliquez sur **"✏️ Modifier"**
3. Éditez le texte
4. Confirmez
5. Le message affiche "(modifié)"

### Supprimer un message

1. Survolez votre message
2. Cliquez sur **"🗑️ Supprimer"**
3. Confirmez la suppression
4. Le message est marqué comme supprimé

### Gérer les notifications d'un canal

1. Ouvrez le canal
2. Cliquez sur l'icône **"🔔"** en haut
3. Choisissez votre préférence :
   - **Tous** : Notifié pour tous les messages
   - **Mentions** : Seulement si vous êtes mentionné
   - **Aucune** : Notifications désactivées
4. La préférence est enregistrée

---

## Canaux de Projet

Les canaux peuvent être liés à un projet :

| Avantage | Description |
|----------|-------------|
| **Équipe auto** | Les membres du projet sont ajoutés automatiquement |
| **Contexte** | Discussion centrée sur le projet |
| **Archivage** | Le canal peut être archivé avec le projet |
| **Recherche** | Retrouver les discussions du projet |

### Créer un canal pour un projet

1. Créez un nouveau canal
2. Dans "Projet associé", sélectionnez le projet
3. Le canal est automatiquement lié
4. Les membres du projet peuvent rejoindre facilement

---

## Raccourcis Clavier

| Raccourci | Action |
|-----------|--------|
| **Entrée** | Envoyer le message |
| **Shift + Entrée** | Nouvelle ligne |
| **Ctrl + B** | Gras |
| **Ctrl + I** | Italique |
| **Esc** | Annuler l'édition |

---

## Statistiques du Module

Le module affiche des statistiques :

| Métrique | Description |
|----------|-------------|
| **Messages total** | Nombre total de messages (non supprimés) |
| **Canaux actifs** | Canaux avec activité récente |
| **Membres actifs** | Utilisateurs ayant participé |
| **Fichiers partagés** | Nombre de pièces jointes |

---

## Astuces et Bonnes Pratiques

- **Nommez clairement** : Utilisez des noms de canaux descriptifs (ex: "projet-laval-2025")
- **Utilisez les fils** : Pour garder les conversations organisées
- **Mentionnez avec parcimonie** : @mention seulement quand nécessaire
- **Archivez les canaux inactifs** : Gardez la liste propre
- **Partagez des fichiers** : Centralisez les documents dans les canaux
- **Réagissez** : Les réactions permettent de valider sans encombrer
- **Configurez les notifications** : Évitez la surcharge d'alertes
- **Utilisez les canaux par projet** : Contexte clair pour l'équipe

---

## Résolution de Problèmes

### Je ne vois pas un canal

- **Cause** : Canal privé et vous n'êtes pas membre
- **Solution** : Demandez une invitation au propriétaire

### Les notifications ne fonctionnent pas

- **Cause** : Notifications désactivées pour le canal
- **Solution** : Vérifiez les paramètres de notification du canal

### Je ne peux pas supprimer un message

- **Cause** : Le message n'est pas le vôtre ou vous n'avez pas les droits
- **Solution** : Seul l'auteur ou un modérateur peut supprimer

### Le fichier joint est trop gros

- **Cause** : Limite de taille dépassée
- **Solution** : Compressez le fichier ou utilisez un lien externe

---

## Questions Fréquentes (FAQ)

**Q: Combien de canaux puis-je créer ?**
R: Il n'y a pas de limite. Créez autant de canaux que nécessaire pour organiser vos discussions.

**Q: Les messages sont-ils archivés ?**
R: Oui, tous les messages sont conservés. Les canaux peuvent être archivés mais pas supprimés.

**Q: Puis-je quitter un canal ?**
R: Oui, cliquez sur "Quitter le canal" dans les options. Le propriétaire ne peut pas quitter sans transférer la propriété.

**Q: Les messages sont-ils sécurisés ?**
R: Oui, les messages sont stockés de manière sécurisée et accessibles uniquement aux membres du canal.

**Q: Puis-je rechercher dans les messages ?**
R: Oui, utilisez la barre de recherche pour trouver des messages par mot-clé.

**Q: Puis-je intégrer avec Slack ou Teams ?**
R: Actuellement, le module utilise son propre système. Des intégrations sont prévues dans les versions futures.

---

## Données Techniques

### Requête Canaux avec Statistiques

```sql
SELECT c.*,
       (SELECT COUNT(*) FROM conference_members cm WHERE cm.channel_id = c.id) as member_count,
       (SELECT COUNT(*) FROM conference_messages msg WHERE msg.channel_id = c.id AND msg.is_deleted = FALSE) as message_count
FROM conference_channels c
WHERE c.is_active = TRUE
ORDER BY c.name
```

### Requête Messages d'un Canal

```sql
SELECT m.*, u.nom as user_name,
       (SELECT COUNT(*) FROM conference_reactions r WHERE r.message_id = m.id) as reaction_count,
       (SELECT COUNT(*) FROM conference_messages replies WHERE replies.parent_message_id = m.id AND replies.is_deleted = FALSE) as reply_count
FROM conference_messages m
LEFT JOIN users u ON m.user_id = u.id
WHERE m.channel_id = :channel_id AND m.is_deleted = FALSE
ORDER BY m.created_at DESC
```

### Requête Notifications Non Lues

```sql
SELECT COUNT(*) as unread_count
FROM conference_notifications
WHERE user_id = :user_id AND is_read = FALSE
```

### Requête Réactions par Message

```sql
SELECT r.emoji, COUNT(*) as count,
       STRING_AGG(u.nom, ', ') as users
FROM conference_reactions r
LEFT JOIN users u ON r.user_id = u.id
WHERE r.message_id = :message_id
GROUP BY r.emoji
```

---

## Voir Aussi

- [📋 Projets](07-projets.md) - Canaux de projet
- [👥 Contacts](04-contacts.md) - Équipe
- [📅 Calendrier](08-calendrier.md) - Événements d'équipe
- [📧 Emails](22-emails.md) - Communications externes
