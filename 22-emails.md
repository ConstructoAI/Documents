# 📧 Emails

## Introduction

Le module **Emails** intègre la gestion de vos courriels directement dans CONSTRUCTO AI. Il permet de consulter, envoyer et organiser vos communications sans quitter l'application, avec une synchronisation automatique avec votre serveur de messagerie, des templates professionnels et une intégration complète avec le CRM.

Ce module inclut 5 templates d'emails professionnels par défaut, la synchronisation IMAP/SMTP, et s'intègre avec les modules Devis, Factures, Projets et Contacts.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"📧 Emails"**
2. Votre boîte de réception s'affiche avec les onglets :
   - **Réception** : Courriels reçus
   - **Composer** : Nouveau courriel
   - **Templates** : Modèles d'emails
   - **Paramètres** : Configuration des comptes
3. Gérez vos courriels et synchronisez vos comptes

---

## Structure des Données

### Table PostgreSQL : `email_accounts`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `user_id` | INTEGER | Utilisateur propriétaire |
| `email` | TEXT | Adresse email |
| `display_name` | TEXT | Nom affiché |
| `imap_server` | TEXT | Serveur IMAP (réception) |
| `imap_port` | INTEGER | Port IMAP (993) |
| `smtp_server` | TEXT | Serveur SMTP (envoi) |
| `smtp_port` | INTEGER | Port SMTP (587) |
| `password_encrypted` | TEXT | Mot de passe chiffré |
| `is_active` | BOOLEAN | Compte actif |
| `last_sync` | TIMESTAMP | Dernière synchronisation |
| `created_at` | TIMESTAMP | Date de création |

### Table : `email_messages`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `account_id` | INTEGER | Compte email |
| `message_id` | TEXT | ID unique du message |
| `folder` | TEXT | Dossier (INBOX, Sent, etc.) |
| `from_address` | TEXT | Expéditeur |
| `to_addresses` | TEXT | Destinataires |
| `cc_addresses` | TEXT | Copies (CC) |
| `subject` | TEXT | Sujet |
| `body_text` | TEXT | Corps texte |
| `body_html` | TEXT | Corps HTML |
| `date_sent` | TIMESTAMP | Date d'envoi |
| `is_read` | BOOLEAN | Lu/Non lu |
| `has_attachments` | BOOLEAN | Pièces jointes |
| `contact_id` | INTEGER | Lien vers contact CRM |
| `company_id` | INTEGER | Lien vers entreprise |
| `project_id` | INTEGER | Lien vers projet |

### Table : `email_templates`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `code` | TEXT | Code unique (ex: devis_envoye) |
| `name` | TEXT | Nom du template |
| `category` | TEXT | Catégorie (devis, facture, projet) |
| `subject_template` | TEXT | Modèle de sujet avec variables |
| `body_html_template` | TEXT | Corps HTML avec variables |
| `body_text_template` | TEXT | Corps texte alternatif |
| `auto_attach_signature` | BOOLEAN | Ajouter signature automatiquement |
| `usage_count` | INTEGER | Nombre d'utilisations |
| `last_used_at` | TIMESTAMP | Dernière utilisation |
| `active` | BOOLEAN | Template actif |

---

## Templates d'Emails par Défaut (5)

| Code | Nom | Catégorie | Variables disponibles |
|------|-----|-----------|----------------------|
| **devis_envoye** | Envoi de devis | Devis | `{{numero_devis}}`, `{{nom_client}}`, `{{montant}}`, `{{date_validite}}` |
| **facture_envoyee** | Envoi de facture | Facture | `{{numero_facture}}`, `{{nom_client}}`, `{{montant}}`, `{{date_echeance}}` |
| **facture_rappel** | Rappel de facture | Facture | `{{numero_facture}}`, `{{montant_du}}`, `{{jours_retard}}` |
| **projet_update** | Mise à jour projet | Projet | `{{nom_projet}}`, `{{avancement}}`, `{{prochaine_etape}}` |
| **b2b_notification** | Notification B2B | Portail | `{{type_notification}}`, `{{message}}`, `{{lien_portail}}` |

---

## Variables de Templates

Les templates utilisent des variables entre double accolades `{{variable}}` :

| Variable | Description | Exemple |
|----------|-------------|---------|
| `{{nom_client}}` | Nom du client | Jean Tremblay |
| `{{nom_entreprise}}` | Nom de l'entreprise | Construction ABC Inc. |
| `{{numero_devis}}` | Numéro du devis | DEV-2025-0042 |
| `{{numero_facture}}` | Numéro de la facture | FAC-2025-0128 |
| `{{montant}}` | Montant total | 15 450,00 $ |
| `{{date_validite}}` | Date de validité | 15 janvier 2025 |
| `{{date_echeance}}` | Date d'échéance | 30 janvier 2025 |
| `{{nom_projet}}` | Nom du projet | Rénovation cuisine |
| `{{avancement}}` | % d'avancement | 75% |
| `{{lien_portail}}` | Lien vers portail client | https://portail.xxx.com |

---

## Fonctionnalités Principales

### 1. Boîte de Réception Unifiée

| Fonctionnalité | Description |
|----------------|-------------|
| **Synchronisation** | IMAP automatique avec votre serveur |
| **Multi-comptes** | Plusieurs comptes email supportés |
| **Organisation** | Par dossiers (Réception, Envoyés, etc.) |
| **Recherche** | Par expéditeur, sujet, contenu |
| **Marquage** | Lu/Non lu, Important, Archivé |

### 2. Envoi de Courriels

| Fonctionnalité | Description |
|----------------|-------------|
| **Éditeur riche** | Formatage HTML intégré |
| **Templates** | Utilisation de modèles prédéfinis |
| **Variables** | Remplacement automatique des variables |
| **Pièces jointes** | Documents, images, PDF |
| **CC/BCC** | Copie et copie cachée |
| **Signature** | Signature automatique |

### 3. Templates Professionnels

| Fonctionnalité | Description |
|----------------|-------------|
| **5 templates** | Prêts à l'emploi |
| **Personnalisation** | Créez vos propres templates |
| **Variables** | Remplacement automatique |
| **Statistiques** | Suivi d'utilisation |

### 4. Intégration CRM

| Fonctionnalité | Description |
|----------------|-------------|
| **Lier à un contact** | Associer l'email à une fiche contact |
| **Lier à une entreprise** | Associer l'email à un client |
| **Lier à un projet** | Associer l'email à un projet |
| **Historique** | Voir tous les emails dans la fiche |

---

## Guide Pas-à-Pas

### Configurer votre compte email

1. Allez dans **"⚙️ Paramètres"** du module Email
2. Cliquez sur **"➕ Ajouter un compte"**
3. Entrez vos informations :

   **Section Compte :**
   - Adresse email
   - Nom affiché (votre nom)

   **Section Réception (IMAP) :**
   - Serveur IMAP (ex: imap.gmail.com)
   - Port IMAP (993 avec SSL)

   **Section Envoi (SMTP) :**
   - Serveur SMTP (ex: smtp.gmail.com)
   - Port SMTP (587 avec TLS)

   **Section Authentification :**
   - Mot de passe ou mot de passe d'application

4. Cliquez sur **"🔌 Tester la connexion"**
5. Si le test réussit, cliquez sur **"💾 Enregistrer"**
6. La synchronisation démarre automatiquement

### Paramètres courants par fournisseur

| Fournisseur | IMAP | Port | SMTP | Port |
|-------------|------|------|------|------|
| **Gmail** | imap.gmail.com | 993 | smtp.gmail.com | 587 |
| **Outlook/Hotmail** | outlook.office365.com | 993 | smtp.office365.com | 587 |
| **Yahoo** | imap.mail.yahoo.com | 993 | smtp.mail.yahoo.com | 587 |
| **Vidéotron** | imap.videotron.ca | 993 | smtp.videotron.ca | 587 |
| **Bell** | imap.bell.net | 993 | smtp.bell.net | 587 |

### Envoyer un nouveau courriel

1. Onglet **"Composer"**
2. Remplissez les champs :
   - **À** : Adresse(s) du/des destinataire(s)
   - **CC** : Copie (optionnel)
   - **BCC** : Copie cachée (optionnel)
   - **Objet** : Sujet du message
3. Rédigez votre message dans l'éditeur
4. Ajoutez des pièces jointes si nécessaire
5. Cliquez sur **"📤 Envoyer"**
6. Le courriel est envoyé et copié dans "Envoyés"

### Utiliser un template

1. Lors de la composition, cliquez sur **"📋 Utiliser un template"**
2. Sélectionnez un template dans la liste :
   - Envoi de devis
   - Envoi de facture
   - Rappel de facture
   - Mise à jour projet
   - Notification B2B
3. Les champs sont pré-remplis avec le modèle
4. Les variables `{{...}}` sont remplacées automatiquement
5. Personnalisez si nécessaire
6. Envoyez

### Envoyer un devis par email

1. Dans le module **Devis**, ouvrez un devis
2. Cliquez sur **"📧 Envoyer par email"**
3. Le template "devis_envoye" est utilisé automatiquement
4. Les variables sont remplies :
   - `{{numero_devis}}` → numéro du devis
   - `{{nom_client}}` → nom du client
   - `{{montant}}` → montant total TTC
   - `{{date_validite}}` → date de validité
5. Le PDF du devis est attaché automatiquement
6. Vérifiez et personnalisez si besoin
7. Cliquez sur **"📤 Envoyer"**

### Envoyer une facture par email

1. Dans le module **Comptabilité**, ouvrez une facture
2. Cliquez sur **"📧 Envoyer par email"**
3. Le template "facture_envoyee" est utilisé
4. Les variables sont remplies automatiquement
5. Le PDF de la facture est attaché
6. Envoyez

### Envoyer un rappel de facture

1. Sur une facture en retard
2. Cliquez sur **"📧 Envoyer rappel"**
3. Le template "facture_rappel" est utilisé
4. Les variables incluent :
   - `{{jours_retard}}` → nombre de jours en retard
   - `{{montant_du}}` → solde restant dû
5. Le ton est adapté pour une relance professionnelle
6. Envoyez

### Lier un courriel à un contact

1. Ouvrez le courriel concerné
2. Cliquez sur **"🔗 Lier à..."**
3. Sélectionnez le type de lien :
   - **Contact** : Recherchez le contact
   - **Entreprise** : Recherchez l'entreprise
   - **Projet** : Recherchez le projet
4. Confirmez l'association
5. Le courriel apparaît dans l'historique de la fiche

### Créer une tâche depuis un courriel

1. Ouvrez le courriel
2. Cliquez sur **"📋 Créer une tâche"**
3. La tâche est pré-remplie :
   - Titre = Sujet du courriel
   - Description = Contenu du courriel
4. Définissez la date d'échéance
5. Assignez à un utilisateur si nécessaire
6. Enregistrez

---

## Paramètres de Synchronisation

| Paramètre | Options | Description |
|-----------|---------|-------------|
| **Fréquence** | 5 min, 15 min, 30 min, Manuelle | Intervalle de synchronisation |
| **Période** | 30 jours, 90 jours, Tout | Profondeur de synchronisation |
| **Dossiers** | Sélection | Dossiers à synchroniser |
| **Pièces jointes** | Oui/Non | Télécharger les pièces jointes |

---

## Intégration avec les Modules

| Module | Intégration |
|--------|-------------|
| **Devis** | Envoi de devis avec template et PDF |
| **Comptabilité** | Envoi de factures et rappels |
| **Projets** | Mises à jour de projet aux clients |
| **CRM** | Historique des échanges dans les fiches |
| **Calendrier** | Invitations à des événements |

---

## Astuces et Bonnes Pratiques

- **Répondez rapidement** : Les clients apprécient la réactivité
- **Utilisez les templates** : Gain de temps et cohérence du message
- **Liez aux fiches** : Historique complet des échanges dans le CRM
- **Archivez régulièrement** : Gardez la boîte de réception propre
- **Vérifiez les spams** : Certains emails importants y atterrissent
- **Personnalisez les templates** : Adaptez-les à votre style
- **Vérifiez avant d'envoyer** : Relisez l'email et vérifiez les pièces jointes
- **Utilisez BCC** : Pour les envois groupés, protégez les adresses

---

## Résolution de Problèmes

### La connexion IMAP échoue

- **Cause** : Mauvais serveur/port ou mot de passe incorrect
- **Solution** : Vérifiez les paramètres. Pour Gmail, utilisez un mot de passe d'application.

### Les emails ne s'envoient pas

- **Cause** : Configuration SMTP incorrecte
- **Solution** : Vérifiez le serveur, port et authentification

### Les variables ne sont pas remplacées

- **Cause** : Variable mal orthographiée ou données manquantes
- **Solution** : Vérifiez l'orthographe exacte des variables `{{nom_variable}}`

### Les pièces jointes sont trop volumineuses

- **Cause** : Limite du serveur de messagerie (généralement 25 Mo)
- **Solution** : Compressez les fichiers ou utilisez un lien de partage

---

## Questions Fréquentes (FAQ)

**Q: Puis-je connecter plusieurs comptes email ?**
R: Oui, vous pouvez configurer plusieurs comptes et basculer entre eux. Chaque utilisateur peut avoir ses propres comptes.

**Q: Les courriels sont-ils stockés dans CONSTRUCTO AI ?**
R: Les courriels restent sur votre serveur de messagerie. CONSTRUCTO AI les synchronise en lecture et conserve les métadonnées pour le CRM.

**Q: Puis-je utiliser Gmail ou Outlook ?**
R: Oui, avec les paramètres IMAP/SMTP appropriés. Pour Gmail, activez l'accès aux applications moins sécurisées ou créez un mot de passe d'application.

**Q: Les pièces jointes volumineuses sont-elles supportées ?**
R: Oui, jusqu'à la limite de votre serveur de messagerie (généralement 25 Mo).

**Q: Puis-je créer mes propres templates ?**
R: Oui, dans l'onglet Templates, cliquez sur "Nouveau template" et utilisez les variables `{{...}}` pour personnaliser.

**Q: L'historique des emails est-il conservé si je supprime le compte ?**
R: Les liens CRM sont conservés, mais les contenus des emails sont supprimés du cache local.

---

## Données Techniques

### Requête Emails par Contact

```sql
SELECT em.*, c.nom as contact_nom
FROM email_messages em
LEFT JOIN contacts c ON em.contact_id = c.id
WHERE em.contact_id = :contact_id
ORDER BY em.date_sent DESC
```

### Requête Templates Actifs

```sql
SELECT * FROM email_templates
WHERE active = TRUE
ORDER BY category, name
```

### Requête Utilisation Templates

```sql
SELECT code, name, usage_count, last_used_at
FROM email_templates
WHERE active = TRUE
ORDER BY usage_count DESC
```

---

## Voir Aussi

- [👥 Contacts](04-contacts.md) - Historique des échanges
- [🏢 Entreprises](03-entreprises.md) - Communication clients
- [🧾 Devis](06-devis.md) - Envoi de devis par email
- [💰 Comptabilité](19-comptabilite.md) - Envoi de factures
- [💬 Conférence](23-conference.md) - Invitations par email
