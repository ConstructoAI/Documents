# Module 23 — Courriels (IMAP/SMTP)

> **Version** : 3.0 (refonte complète vérifiée contre le code source, juillet 2026)
> **Route** : `/emails` — menu latéral « Emails » (section Communication de la barre de navigation)
> **Code de référence** : `backend/routers/emails.py` (6993 lignes, 29 points d'accès), `backend/routers/emails_ai.py` (332 lignes, 1 point d'accès), `frontend/src/pages/EmailsPage.tsx`, `frontend/src/components/emails/*` (`EmailAccountsPanel`, `EmailSyncPanel`, `EmailAIPanel`, `EmailAIComposeButton`, `EmailsAssistantTab`), `frontend/src/api/emails.ts`, `frontend/src/store/useEmailsStore.ts`
> **Tables PostgreSQL (une copie par entreprise / tenant)** : `email_accounts`, `emails`, `email_attachments`, `email_templates`, `email_threads`, `email_sync_log`
> **Cadrage** : ce module est un **client de messagerie IMAP/SMTP multi-comptes** intégré à l'ERP, façon Outlook. Il gère les **courriels externes** (Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, ou tout serveur IMAP/SMTP) reçus et envoyés depuis les serveurs de messagerie de l'utilisateur, plus une **adresse interne** propre à chaque entreprise. Il n'est **pas** la messagerie instantanée entre utilisateurs (voir le module Messagerie) ni le système de notifications de l'ERP (voir le module Administration).

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « tenant » désigne votre entreprise (chaque entreprise a ses propres données isolées).

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas à pas](#3-workflows-pas-a-pas)
4. [Référence](#4-reference)
5. [Intégrations et FAQ](#5-integrations-et-faq)
6. [Récapitulatif](#6-recapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Gérer vos courriels professionnels sans quitter l'ERP :

- **Plusieurs comptes par entreprise** : Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365 ou tout autre serveur IMAP/SMTP, utilisés simultanément.
- **Deux méthodes de connexion** : mot de passe applicatif (chiffré au repos par Fernet) ou **OAuth2** (Google et Microsoft 365, sans mot de passe applicatif).
- **Réception IMAP** à la demande (trois modes) et **synchronisation automatique** en arrière-plan.
- **Envoi SMTP réel** avec pièces jointes, modèles pré-remplis et signature.
- **Adresse interne** propre à l'entreprise : les courriels envoyés à cette adresse arrivent dans votre boîte de réception grâce à un webhook (intégration n8n et Mailgun).
- **Lecture, recherche, favoris (étoile) et corbeille**.
- **Deux fonctions d'intelligence artificielle Claude** : une pour **rédiger et répondre** (assistée), une pour **consulter et résumer** votre boîte (lecture seule).
- **Liaison automatique au CRM** : à la synchronisation, un courriel entrant est rattaché au contact ou à l'entreprise correspondant (par adresse exacte, sinon par domaine).

### 1.2 Comment y accéder

- Barre de navigation latérale → section **Communication** → **Emails** (icône enveloppe). Les voisins immédiats sont **Messagerie** et **Agent vocal**.
- Adresse : `/emails`.
- La page est accessible à tout utilisateur connecté de l'entreprise.

### 1.3 Rôles et permissions

- **Chaque compte courriel appartient à l'utilisateur qui l'a créé** (`user_id`). Un compte marqué comme « partagé » (sans propriétaire) reste visible par tous les utilisateurs de l'entreprise ; c'est le cas de l'**adresse interne**, partagée par nature.
- **Aucun utilisateur ne voit les comptes personnels d'un autre utilisateur** de la même entreprise, même un administrateur. Il n'y a pas de contournement administrateur sur les comptes personnels.
- **Une seule action est réservée aux administrateurs** : le bouton **« Restaurer »** (réactivation de comptes désactivés). Toutes les autres actions sont ouvertes au propriétaire du compte.
- En **mode consultation** (abonnement suspendu / lecture seule), les opérations d'écriture — envoyer, créer ou modifier un compte, déplacer, synchroniser, répondre automatiquement — sont bloquées par le contrôle global de l'ERP ; la lecture reste possible.

### 1.4 Les sous-modules (onglets)

Le module compte **8 onglets** et **1 dossier supplémentaire** (la Corbeille, accessible par la liste des dossiers, sans onglet dédié) :

| # | Onglet | Rôle |
|---|--------|------|
| 1 | **Boîte de réception** | Lire, rechercher, étoiler et supprimer les courriels reçus ; répondre |
| 2 | **Nouveau message** | Ouvre la fenêtre de composition (ne change pas d'onglet) |
| 3 | **Envoyés** | Courriels expédiés (ou en échec) |
| 4 | **Brouillons** | Voir §2.1 et §5 — non alimentable manuellement depuis l'interface |
| 5 | **Modèles** | Consulter les gabarits de courriels pré-installés (lecture seule) |
| 6 | **Configuration** | Gérer les comptes IMAP/SMTP et les connexions OAuth |
| 7 | **Synchronisation** | Recevoir les courriels IMAP et voir l'historique des synchronisations |
| 8 | **Assistant IA** | Poser des questions sur votre boîte (lecture seule, aucun envoi) |

*À noter :* les commentaires internes du code se contredisent (l'en-tête du fichier annonce « 8 onglets », un commentaire plus bas dit « 7 onglets ») ; le décompte réel affiché est bien **8 onglets**. La **Corbeille** est un quatrième **dossier** dans la liste de gauche, pas un onglet.

### 1.5 Ce que le module fait — et ne fait PAS

Le module **fait** : configurer plusieurs comptes, se connecter par mot de passe applicatif ou par OAuth, synchroniser IMAP (à la demande et automatiquement), envoyer réellement via SMTP avec pièces jointes, recevoir sur une adresse interne via webhook, assainir le HTML reçu, lier les entrants au CRM, rédiger et répondre avec l'IA, consulter la boîte avec un assistant IA en lecture seule.

Le module **ne fait PAS** (limites détaillées en §5) :

- Pas de bouton **« Enregistrer comme brouillon »** dans la fenêtre de composition.
- **Modèles en lecture seule** : impossible de créer, modifier ou supprimer un modèle depuis l'interface.
- Pas de **« Répondre à tous »** ni de **« Transférer »** ; le bouton **« Archiver »** a été retiré volontairement.
- Pas de bouton manuel **« Marquer comme lu / non lu »** : la lecture marque automatiquement le courriel comme lu.
- Pas de **vue de fil de discussion** (conversation regroupée).
- Pas d'**export PDF ou CSV**, ni d'**impression** dans ce module.
- Les états locaux (lu, étoile, corbeille) **ne sont pas répercutés** sur le serveur IMAP d'origine.

---

## 2. Interface

### 2.1 Barre d'onglets

En haut de la page, une barre présente les 8 onglets décrits en §1.4. Éléments notables :

- Un **badge bleu** de courriels non lus s'affiche sur l'onglet **Boîte de réception** lorsqu'il y a au moins un message non lu.
- Le bouton **Nouveau message** ouvre une fenêtre modale et **ne change pas d'onglet** (vous restez sur l'onglet courant).
- Les onglets **Boîte de réception**, **Envoyés** et **Brouillons** partagent la même **disposition à trois panneaux** (dossiers / liste / lecture). La **Corbeille** utilise la même disposition, mais s'atteint par la liste des dossiers du panneau de gauche.

### 2.2 Disposition à trois panneaux (Boîte de réception)

#### 2.2.1 Panneau de gauche — Dossiers

- Bouton **« Nouveau message »** pleine largeur.
- Bandeau **« Adresse interne »** (en lecture seule) affichant l'adresse interne de votre entreprise. Les courriels envoyés à cette adresse arriveront dans la boîte de réception (l'interface se rafraîchit toutes les 60 secondes).
- **Quatre dossiers**, chacun avec un compteur : **Boîte de réception**, **Envoyés**, **Brouillons**, **Corbeille**. Le compteur affiche le nombre de non-lus (badge bleu) ou le total.
- Sur mobile, un en-tête affiche le titre du dossier courant et un bouton « Nouveau ».

#### 2.2.2 Panneau du milieu — Liste des messages

- Champ de **recherche** (« Rechercher... ») avec un délai de 400 ms avant déclenchement. La recherche porte sur l'objet, l'expéditeur, le destinataire et le corps du texte.
- Chaque ligne affiche : une **pastille de non-lu** (bleu clair), l'**expéditeur** (ou le destinataire dans Envoyés et Brouillons), l'**heure relative**, l'**objet** (ou « (sans objet) »), et les icônes **étoile** (favori) et **trombone** (pièces jointes).
- **États vides** distincts selon le dossier : « Aucun courriel reçu / envoyé », « Aucun brouillon », « Aucun courriel », avec un rappel sur l'adresse interne.
- **Pagination** de 50 messages par page dès que le total dépasse 50 : boutons **« Préc. »**, indicateur **« Page X / Y »**, bouton **« Suiv. »**.

#### 2.2.3 Panneau de droite — Lecture d'un message

- **En-tête** : objet, expéditeur (nom et adresse entre chevrons), destinataires (« À : », « Cc : »), date relative.
- **Actions** : bouton **étoile** (ajouter / retirer le favori), bouton **corbeille** (avec confirmation « Voulez-vous mettre à la corbeille ce message ? », ou « Voulez-vous définitivement supprimer ce message ? » si le courriel est déjà dans la Corbeille), bouton **« Répondre »**, bouton fermer.
- **Corps** : le HTML est **assaini** avant affichage (voir §4.10) ou, à défaut, le texte brut (« (aucun contenu) » si vide).
- **Panneau IA** : affiché **uniquement pour un courriel de la boîte de réception**. Il donne accès aux fonctions Analyser, Suggérer une réponse et Répondre auto (voir §2.6).
- **Pièces jointes** : section « Pièces jointes (N) » avec un bouton de téléchargement par fichier (nom et taille en Ko).
- **État vide** (aucun message sélectionné) : « Sélectionnez un courriel à lire » et un rappel de votre adresse.

### 2.3 Fenêtre « Nouveau message » (composition)

La fenêtre de composition présente, dans l'ordre :

1. **Compte expéditeur** (« De ») : menu déroulant avec l'option **« Compte par défaut »** et vos comptes actifs (hors adresse interne). Un badge « (défaut) » et le nom du fournisseur accompagnent chaque compte.
2. **Modèle** : menu déroulant (« Sélectionner un modèle... ») qui remplit automatiquement l'objet et le corps à partir du gabarit choisi (le HTML est converti en texte).
3. **À** (obligatoire) : destinataire(s), séparés par des virgules.
4. **Cc** (copie conforme).
5. **Cci** (copie conforme invisible).
6. Bouton **« Rédiger avec IA »** (voir §2.7).
7. **Objet**.
8. **Message** (zone de texte de 8 lignes).
9. **Joindre un fichier** : sélection de fichiers multiples, **5 fichiers au maximum** côté interface ; chaque pièce jointe apparaît en pastille (nom, taille, bouton retirer).
10. **Actions** : **« Annuler »** et **« Envoyer »** (le bouton Envoyer est désactivé tant que le champ « À » est vide ou pendant l'envoi).

Comportements importants :

- Le corps texte est converti en HTML (échappement et sauts de ligne) avant l'envoi.
- Si l'envoi SMTP échoue, la fenêtre **reste ouverte** et conserve votre saisie, afin que vous puissiez réessayer.
- Il n'y a **pas** de bouton « Enregistrer comme brouillon » (voir §5).

### 2.4 Onglet « Configuration » (comptes)

Titre : **« Comptes courriel »**, sous-titre « Gérer vos comptes IMAP/SMTP, OAuth Gmail et Microsoft 365 ».

- **Boutons d'en-tête** :
  - **« Restaurer »** — réactive les comptes désactivés qui possèdent encore des identifiants (réservé aux administrateurs).
  - **« Nouveau compte »** — ouvre la fenêtre de création.
- **Bloc OAuth dépliable** (« Connexion rapide avec OAuth (recommandé) ») : boutons **« Connecter Gmail (OAuth) »** et **« Connecter Microsoft 365 (OAuth) »**. Chaque bouton est **grisé** si les identifiants du serveur ne sont pas configurés (l'infobulle indique alors « GOOGLE_CLIENT_ID/SECRET non configurés sur Render » ou « MS_CLIENT_ID/SECRET non configurés sur Render »). Un clic redirige vers la page d'autorisation du fournisseur.
- **Liste des comptes** (une carte par compte) : nom, adresse, ligne IMAP/SMTP (serveur et port), dernière synchronisation avec son statut et son éventuelle erreur. **Badges** possibles : **Défaut** (étoile), fournisseur, **OAuth** (vert), **Mot de passe** (cadenas), **Sync auto** (éclair vert) ou **Sync off** (gris).
- **Actions par compte** : **Tester** (vérifie IMAP et SMTP séparément et affiche le résultat), **Modifier** (crayon), **Désactiver** (corbeille).
- **État vide** : « Aucun compte configuré » et « Cliquez sur "Nouveau compte" pour commencer. »

#### 2.4.1 Fenêtre de création / modification d'un compte

Titre « Nouveau compte courriel » ou « Modifier le compte ». Champs :

- **Nom du compte** (obligatoire, ex. « Gmail principal »).
- **Adresse courriel** (obligatoire ; non modifiable en édition). En quittant ce champ, le **fournisseur est détecté automatiquement** à partir du domaine.
- **Fournisseur** : menu déroulant (Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, Autre). Affiche « (détection en cours...) » pendant la détection, plus des instructions et un lien « Guide de configuration ».
- **Bloc IMAP (réception)** : Serveur (obligatoire), Port, Utilisateur (par défaut = l'adresse courriel), case **SSL** (par défaut : port 993, activé).
- **Bloc SMTP (envoi)** : Serveur (obligatoire), Port, Utilisateur, case **STARTTLS** (par défaut : port 587, activé).
- **Mot de passe applicatif** : le libellé s'adapte au contexte —
  - « Mot de passe applicatif * » (création),
  - « Nouveau mot de passe (laisser vide pour conserver) » (modification d'un compte déjà authentifié),
  - « Mot de passe applicatif * (requis pour activer le compte) » (modification d'un compte sans authentification).
  Un encart d'alerte apparaît si le compte n'a aucune authentification enregistrée.
- **Signature HTML** et **Signature texte** (zones de texte).
- Cases **Synchronisation auto** et **Compte par défaut**.
- Champ **Dossiers à sync** (par défaut « INBOX »).
- Un **avertissement** s'affiche si vous créez un compte sans mot de passe (« Utilisez OAuth ou saisissez un mot de passe applicatif »).
- Boutons **Annuler** et **Créer** (ou **Enregistrer** en modification).

#### 2.4.2 Confirmation de désactivation

Titre « Désactiver ce compte ? », message : « Le compte sera désactivé (soft-delete). Les courriels restent accessibles en lecture mais le compte ne pourra plus envoyer ni synchroniser. » Boutons **Annuler** et **Désactiver**.

*La suppression est toujours une désactivation (soft-delete) :* aucune suppression définitive de compte. Les courriels déjà reçus demeurent lisibles.

### 2.5 Onglet « Synchronisation »

Titre « Synchronisation », sous-titre « Recevoir les courriels IMAP depuis vos comptes externes ».

- **Synchroniser tous les comptes** — trois boutons :
  - **« Nouveaux uniquement »** (mode `new`) : les non-lus depuis la dernière synchronisation.
  - **« 50 derniers courriels »** (mode `recent`) : rattrapage initial (lus et non lus).
  - **« Tous (max 200) »** (mode `all`) : jusqu'à 200 courriels par compte ; ce bouton demande une confirmation avant de lancer.
  Des notes explicatives accompagnent chaque bouton. Un avertissement s'affiche s'il n'y a aucun compte externe (la synchronisation ne s'applique pas à l'adresse interne).
- **Synchroniser un compte spécifique** — par compte : boutons **« Nouveaux »** et **« 50 derniers »**.
- **Historique des synchronisations** — bouton **« Actualiser »**, puis une ligne par exécution : badge de statut (**OK** en vert, **Erreur** en rouge, **En cours** en bleu), nom du compte, « +N courriel(s) », « N erreur(s) », heures de démarrage et de fin, message d'erreur éventuel. État vide : « Aucune synchronisation enregistrée ».

### 2.6 Panneau IA sous un courriel reçu (rédaction et réponse)

Visible uniquement pour un courriel de la **boîte de réception**. Titre « Assistant IA Construction », badge « Claude ». Options : **Ton** (Professionnel / Cordial / Formel), **Compte expéditeur** (pour la réponse automatique) et **Contexte additionnel** (facultatif). Trois boutons :

- **« Analyser »** : retourne l'**urgence** (haute / moyenne / basse), le type, le sentiment, un résumé, la liste des **actions requises** (action et échéance) et des **alertes**.
- **« Suggérer une réponse »** : propose 1 ou 2 brouillons de réponse (titre, longueur, corps), avec un badge « Données CRM utilisées », le contexte client, un bouton **« Utiliser »** (qui remplit la fenêtre de composition) et des listes « À inclure » / « À éviter ».
- **« Répondre auto »** (bouton d'alerte) : ouvre une confirmation (« L'IA va **générer ET envoyer** une réponse... » avec l'avertissement « Cette action est irréversible »), puis **génère et envoie la réponse sans validation manuelle**. Le résultat indique « Réponse envoyée automatiquement » ou « Échec d'envoi », un indice de confiance, l'objet, le corps, une note de l'IA et l'erreur SMTP éventuelle.

*Garde-fous de la réponse automatique :* elle ne fonctionne que sur un courriel **entrant** ; elle **refuse** d'agir (erreur 409) si une réponse automatique a déjà été envoyée dans le même fil ou au même expéditeur dans les 24 dernières heures (anti-boucle).

### 2.7 Bouton « Rédiger avec IA » (dans la composition)

Dans la fenêtre de composition, le bouton **« Rédiger avec IA »** ouvre une fenêtre « Rédiger avec IA Construction » : champ **Instructions** (au moins 5 caractères), **Destinataire** (facultatif, pour personnaliser via le CRM) et **Ton**. Le bouton **« Générer »** produit un objet et un corps, plus un conseil sur le meilleur moment d'envoi. Deux boutons appliquent le résultat à la composition : **« Utiliser cette version »** et **« Utiliser la version courte »**.

### 2.8 Onglet « Assistant IA » (consultation, lecture seule)

Titre « Assistant IA — Courriels », sous-titre « Consulte, résume et retrouve tes courriels (lecture seule, aucun envoi). » Une zone de clavardage avec trois exemples de départ :

- « Combien de courriels non lus ai-je ? »
- « Résume les derniers courriels reçus de ce client. »
- « Retrouve les courriels qui parlent de soumission. »

Saisie multiligne (Entrée pour envoyer, Maj+Entrée pour un saut de ligne), bouton **Envoyer**, indicateur « Analyse en cours… ».

*Portée stricte :* cet assistant **lit seulement** vos courriels ; il n'écrit rien et n'envoie aucun courriel. Il n'a **pas** accès aux comptes (identifiants IMAP/SMTP, jetons OAuth) ni à aucune donnée RH, salariale ou de sécurité — sa lecture est limitée aux tables `emails`, `email_threads` et `email_templates`. À ne pas confondre avec la rédaction assistée des §2.6 et §2.7, qui sont des fonctions distinctes.

### 2.9 Onglet « Modèles »

Titre « Modèles de courriels ». **Liste en lecture seule** : pour chaque modèle, le nom, un badge de catégorie, « code: ... », l'objet (en italique) et les **variables disponibles** au format `{{variable}}`. État vide : « Aucun modèle ». Ces modèles alimentent le menu déroulant de la composition (§2.3). Aucune création, modification ou suppression de modèle n'est possible depuis l'interface (voir §5).

---

## 3. Workflows pas à pas

### 3.1 Configurer un compte avec un mot de passe applicatif (exemple Gmail)

1. Onglet **Configuration** → **« Nouveau compte »**.
2. Saisir un **nom** (ex. « Gmail principal ») puis l'**adresse courriel**. Le fournisseur se détecte automatiquement quand vous quittez le champ adresse (les serveurs IMAP et SMTP se pré-remplissent).
3. Générer un **mot de passe applicatif** chez votre fournisseur (pour Gmail avec la validation en deux étapes : `myaccount.google.com/apppasswords`) et le coller dans **Mot de passe applicatif**. N'utilisez **pas** votre mot de passe de session habituel.
4. Vérifier les cases **SSL** (IMAP, port 993) et **STARTTLS** (SMTP, port 587), au besoin ajuster serveurs et ports.
5. Cliquer **Créer**. Le mot de passe est chiffré au repos (voir §4.10).
6. Sur la carte du compte, cliquer **Tester** pour valider IMAP et SMTP, puis aller à l'onglet **Synchronisation** pour recevoir les courriels.

### 3.2 Configurer un compte par OAuth (Gmail ou Microsoft 365)

1. Onglet **Configuration** → déplier le bloc **« Connexion rapide avec OAuth (recommandé) »**.
2. Cliquer **« Connecter Gmail (OAuth) »** ou **« Connecter Microsoft 365 (OAuth) »**. Si le bouton est grisé, les identifiants du serveur ne sont pas configurés — contactez votre administrateur.
3. Autoriser l'accès chez le fournisseur (Google demande l'accès à la messagerie ; Microsoft demande IMAP et SMTP).
4. Au retour, vous êtes redirigé vers l'onglet Courriels avec un message de succès. Le compte apparaît avec le badge **OAuth** (vert). Aucun mot de passe applicatif n'est requis ; le jeton est rafraîchi automatiquement.

*Avantage de l'OAuth :* pas de mot de passe applicatif à gérer, et le jeton d'accès se renouvelle tout seul (le jeton de rafraîchissement est également chiffré au repos).

### 3.3 Configurer un compte manuellement (domaine d'entreprise ou fournisseur « Autre »)

1. Onglet **Configuration** → **« Nouveau compte »** → saisir l'adresse. Un domaine d'entreprise (par exemple hébergé chez GoDaddy ou Microsoft 365) peut être détecté comme **Autre**.
2. Choisir manuellement le bon **fournisseur** dans le menu déroulant, ou saisir directement les serveurs et ports IMAP et SMTP fournis par votre hébergeur.
3. Renseigner l'**Utilisateur** si différent de l'adresse, le **mot de passe**, puis **Créer**.
4. **Tester**, puis **Synchroniser**.

### 3.4 Tester une connexion

Sur la carte d'un compte, le bouton **Tester** vérifie **IMAP puis SMTP séparément** et affiche pour chacun « OK » ou l'erreur rencontrée. Aucun courriel n'est modifié par le test (le jeton OAuth peut être rafraîchi au passage). Utilisez ce bouton chaque fois qu'une synchronisation ou un envoi échoue, pour isoler le problème.

### 3.5 Recevoir ses courriels (synchronisation)

1. Onglet **Synchronisation**.
2. Pour un premier chargement, cliquer **« 50 derniers courriels »** (rattrapage). Ensuite, **« Nouveaux uniquement »** suffit au quotidien. Pour tout importer, **« Tous (max 200) »** (avec confirmation).
3. Les courriels apparaissent dans la **Boîte de réception**. Chaque exécution est tracée dans l'**historique** (statut, nombre de nouveaux courriels, erreurs).

*Synchronisation automatique :* en plus des boutons manuels, un processus de fond synchronise périodiquement les comptes dont la **Synchronisation auto** est activée (mode « Nouveaux uniquement »). L'intervalle par défaut est de 15 minutes (paramétrable côté serveur). L'adresse interne n'est pas concernée par la synchronisation IMAP.

### 3.6 Comprendre l'adresse interne

Chaque entreprise dispose d'une **adresse interne** créée automatiquement, affichée en lecture seule au-dessus de la liste des dossiers. Les courriels envoyés à cette adresse arrivent dans la boîte de réception via un **webhook** (intégration n8n et Mailgun) ; l'interface se rafraîchit toutes les 60 secondes. Cette adresse sert notamment de destination de repli quand vous envoyez sans avoir configuré de compte externe (voir §3.8).

### 3.7 Lire un courriel

Cliquer sur une ligne de la liste ouvre le message dans le panneau de droite. Le courriel est **automatiquement marqué comme lu** (la liste et les badges se mettent à jour aussitôt). Le corps HTML est assaini avant affichage. Pour un courriel de la boîte de réception, le **panneau IA** (Analyser / Suggérer une réponse / Répondre auto) apparaît sous le message.

### 3.8 Composer et envoyer un courriel

1. **Nouveau message**.
2. Choisir le **compte expéditeur** (« De »). Si vous laissez **« Compte par défaut »** et que vous n'avez aucun compte externe, l'envoi passe par le **relais de la plateforme** : le courriel part alors de l'adresse de service `info@constructoai.ca` **au nom de votre entreprise**, et l'adresse de réponse (Reply-To) est votre adresse interne. Pour envoyer depuis votre propre adresse, sélectionnez un de vos comptes configurés.
3. (Facultatif) Choisir un **modèle**, ou cliquer **« Rédiger avec IA »** (§2.7).
4. Renseigner **À**, éventuellement **Cc** et **Cci**, l'**Objet** et le **Message**.
5. (Facultatif) **Joindre un fichier** : jusqu'à 5 fichiers (limites de taille en §4.3).
6. **Envoyer**. Le courriel est enregistré dans **Envoyés** en cas de succès. En cas d'échec SMTP, la fenêtre reste ouverte et conserve votre saisie.

### 3.9 Répondre à un courriel

Dans le panneau de lecture, **« Répondre »** ouvre la composition pré-remplie : **À** = l'expéditeur, **Objet** = « Re: ... », **corps** = citation du message original. Le compte récepteur est présélectionné comme expéditeur (hors adresse interne). Il n'existe pas de « Répondre à tous » ni de « Transférer » (voir §5).

### 3.10 Rédiger un courriel avec l'IA

1. Dans **Nouveau message**, cliquer **« Rédiger avec IA »**.
2. Décrire ce que vous voulez communiquer (au moins 5 caractères). Ajouter un **destinataire** permet à l'IA de personnaliser via le CRM. Choisir un **ton**.
3. **Générer**, puis appliquer avec **« Utiliser cette version »** ou **« Utiliser la version courte »**.
4. Relire et ajuster avant d'envoyer.

### 3.11 Analyser, suggérer une réponse ou répondre automatiquement

Sous un courriel reçu (panneau IA, §2.6) :

- **Analyser** pour obtenir urgence, type, sentiment, résumé, actions requises et alertes.
- **Suggérer une réponse** pour obtenir des brouillons ; **« Utiliser »** remplit la composition afin que vous validiez et envoyiez vous-même.
- **Répondre auto** pour que l'IA **rédige et envoie** immédiatement (après confirmation). À réserver aux cas simples : l'action est irréversible et l'anti-boucle empêche seulement les envois répétés dans un même fil.

### 3.12 Interroger sa boîte avec l'Assistant IA

Onglet **Assistant IA** → poser une question en langage naturel (« Combien de courriels non lus ? », « Résume les derniers courriels de ce client », « Retrouve les courriels qui parlent de soumission »). L'assistant lit vos courriels et répond ; il **n'envoie rien** et **n'accède pas** aux identifiants de compte.

### 3.13 Étoiler, supprimer et vider la Corbeille

- **Étoile** : bouton étoile dans le panneau de lecture pour marquer / retirer un favori.
- **Supprimer** : bouton corbeille → le message part à la **Corbeille** (avec confirmation). Un message **déjà dans la Corbeille** est **supprimé définitivement** au second passage (avec purge des pièces jointes).
- La **Corbeille** s'ouvre depuis la liste des dossiers du panneau de gauche.

*Rappel :* ces actions sont locales à l'ERP ; elles ne se répercutent pas sur le serveur IMAP d'origine.

### 3.14 Télécharger une pièce jointe

Dans le panneau de lecture, section **« Pièces jointes (N) »**, cliquer sur un fichier pour le télécharger (il est servi depuis la base de données ; taille maximale de téléchargement : 25 Mo).

### 3.15 Consulter l'historique de synchronisation

Onglet **Synchronisation** → section **« Historique des synchronisations »** → **« Actualiser »**. Chaque ligne indique le statut, le compte, le nombre de nouveaux courriels, les erreurs, et les heures de début et de fin.

---

## 4. Référence

### 4.1 Points d'accès (30 au total)

Deux routeurs, montés sous le préfixe `/api/erp/v1` : `emails.py` (29 points d'accès, base `/emails`) et `emails_ai.py` (1 point d'accès, base `/emails/ai`).

**Comptes (8)**

| Méthode et chemin | Rôle |
|---|---|
| GET `/accounts` | Liste des comptes actifs ; crée l'adresse interne si aucun compte |
| GET `/providers` | Réglages pré-remplis par fournisseur + disponibilité OAuth |
| GET `/providers/detect?email=` | Détecte le fournisseur d'après le domaine |
| POST `/accounts` | Crée un compte IMAP/SMTP (mot de passe chiffré) |
| PUT `/accounts/{id}` | Modifie un compte (mot de passe vide = conservé) |
| DELETE `/accounts/{id}` | Désactive un compte (soft-delete) |
| POST `/accounts/restore-legacy` | Réactive les comptes désactivés (administrateur) |
| POST `/accounts/{id}/test` | Teste IMAP et SMTP séparément |

**Messages (7)**

| Méthode et chemin | Rôle |
|---|---|
| GET `/messages` | Liste paginée (dossier, recherche, filtres, ≤ 200/page) |
| GET `/messages/{id}` | Détail d'un courriel ; le marque comme lu |
| PUT `/messages/{id}/read` | Marque comme lu |
| PUT `/messages/{id}/star` | Bascule l'étoile |
| PUT `/messages/{id}/move` | Déplace vers un dossier autorisé |
| DELETE `/messages/{id}` | Corbeille, ou suppression définitive si déjà en Corbeille |
| POST `/messages/send` | Envoi (multipart, pièces jointes, modèles) |

**Autres (14)**

| Méthode et chemin | Rôle |
|---|---|
| GET `/templates` | Liste des modèles (lecture seule) |
| GET `/attachments/{id}/download` | Télécharge une pièce jointe (≤ 25 Mo) |
| GET `/threads/{thread_id}` | Messages d'un fil (non exposé dans l'interface) |
| GET `/stats` | Compteurs de non-lus / totaux par dossier |
| POST `/webhook/inbound` | Réception (public ; Bearer n8n ou HMAC Mailgun) |
| GET `/oauth/{provider}/auth-url` | Démarre la connexion OAuth |
| GET `/oauth/{provider}/callback` | Retour OAuth (public ; état signé HMAC) |
| POST `/accounts/{id}/sync` | Synchronise un compte |
| POST `/sync/all` | Synchronise tous les comptes |
| GET `/sync-history` | Historique des synchronisations |
| POST `/ai/suggest-reply` | IA : suggère une réponse |
| POST `/ai/analyze` | IA : analyse un courriel |
| POST `/ai/draft` | IA : rédige un brouillon |
| POST `/ai/auto-reply` | IA : **génère et envoie** une réponse |
| POST `/emails/ai/chat` | Assistant IA de consultation (lecture seule) |

### 4.2 Dossiers et onglets

- **Onglets (8)** : Boîte de réception, Nouveau message, Envoyés, Brouillons, Modèles, Configuration, Synchronisation, Assistant IA.
- **Dossiers de la barre latérale (4)** : Boîte de réception, Envoyés, Brouillons, Corbeille.
- **Dossiers reconnus côté serveur (6, liste fixe)** : `inbox`, `sent`, `drafts`, `trash`, `archive`, `spam`. Seuls les quatre premiers sont affichés ; `archive` et `spam` existent mais n'ont pas de bouton de navigation. Il n'y a pas de dossiers personnalisés.

### 4.3 Limites et quotas

| Élément | Limite |
|---|---|
| Pièces jointes à l'envoi | **5 fichiers**, **10 Mo par fichier**, **20 Mo au total** |
| Objet d'un courriel envoyé | 998 caractères |
| Corps d'un courriel envoyé | 5 000 000 caractères |
| Téléchargement d'une pièce jointe | 25 Mo |
| Pièce jointe reçue (conservée à la synchronisation) | 25 Mo |
| Corps HTML reçu (assaini) | 30 Mo |
| Pagination de la liste | 50 par page (jusqu'à 200 par requête API) |

### 4.4 Modes de synchronisation

| Mode | Bouton | Ce qui est récupéré |
|---|---|---|
| `new` | « Nouveaux uniquement » | Nouveaux depuis la dernière synchronisation (au tout premier passage : jusqu'à 100) |
| `recent` | « 50 derniers courriels » | Les 50 plus récents (lus et non lus) |
| `all` | « Tous (max 200) » | Jusqu'à 200 courriels par compte |

Anti-doublon : un même courriel n'est jamais importé deux fois (identification par le compte et l'identifiant du message).

### 4.5 Fournisseurs pris en charge

**Réglages pré-remplis** : Gmail, Outlook, Yahoo, iCloud, GoDaddy, Microsoft 365, et **Autre** (configuration manuelle). Le fournisseur est détecté à partir du domaine de l'adresse ; un domaine d'entreprise personnalisé peut être classé « Autre » et doit alors être choisi manuellement.

**OAuth disponible** pour **Google (Gmail)** et **Microsoft 365**. Les boutons OAuth restent grisés tant que les identifiants ne sont pas configurés côté serveur (`GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`, `MS_CLIENT_ID` / `MS_CLIENT_SECRET`, plus l'adresse de retour). Les autres fournisseurs utilisent un mot de passe applicatif.

### 4.6 Modèles pré-installés

Six modèles sont installés d'office (courriels de système, non modifiables depuis l'interface) ; chacun contient des variables `{{...}}` remplacées à l'envoi :

| Code | Usage |
|---|---|
| `devis_envoye` | Envoi d'une soumission / d'un devis |
| `facture_envoyee` | Envoi d'une facture |
| `facture_rappel` | Rappel / relance de paiement |
| `projet_update` | Mise à jour de projet |
| `demande_prix` | Demande de prix (fournisseur) |
| `inscription_bienvenue` | Courriel de bienvenue |

Les variables exactes de chaque modèle sont visibles dans l'onglet **Modèles** (§2.9). À l'envoi, les variables sont substituées et les éventuels marqueurs `{{...}}` non résolus sont retirés.

### 4.7 Fonctions d'intelligence artificielle (facturation)

- **Deux surfaces distinctes** :
  - **Rédaction / réponse** (`/ai/analyze`, `/ai/suggest-reply`, `/ai/draft`, `/ai/auto-reply`) — la dernière **génère et envoie**.
  - **Consultation** (`/emails/ai/chat`) — **lecture seule**, sans envoi.
- **Modèle** : Claude Sonnet. **Coût** facturé = (jetons d'entrée × 0,003 + jetons de sortie × 0,015) / 1000 × **1,30** (majoration de 30 %), débité des **crédits IA prépayés** de l'entreprise. Chaque réponse de l'assistant renvoie son coût et le solde restant.
- **Blocage réel** : l'accès n'est pas bloqué par rôle (le garde `check_ai_guard` laisse passer tout utilisateur authentifié). Le seul vrai frein est le **solde de crédits** : si le solde est insuffisant, l'appel est refusé (erreur 402). Un échec de débit après coup est journalisé mais ne bloque pas la réponse.
- **Sécurité de l'IA** : les fonctions de rédaction fonctionnent en lecture seule sur la base de l'entreprise (aucune action d'écriture n'est autorisée à l'IA, en défense contre l'injection par le contenu d'un courriel externe). L'Assistant IA de consultation est en plus limité à trois tables (`emails`, `email_threads`, `email_templates`) et **exclut** les comptes (identifiants, jetons) et toute donnée RH / sécurité.

### 4.8 Limites de fréquence (par adresse IP)

| Point d'accès | Limite |
|---|---|
| `/emails/ai/chat` (Assistant IA) | 20 par minute |
| `/emails/messages/send` (envoi) | 30 par minute |
| `/emails/ai/analyze`, `/suggest-reply`, `/draft`, `/auto-reply` | Aucune limite dédiée (limite générale de 1500/min) |

*À noter :* les quatre fonctions IA de rédaction n'ont pas de plafond dédié, contrairement aux autres clavardages IA de l'ERP. Utilisez-les avec discernement.

### 4.9 Statuts, badges et indicateurs

| Élément | Signification |
|---|---|
| Pastille bleue (liste) | Courriel non lu |
| Étoile | Favori |
| Trombone | Contient des pièces jointes |
| Badge Défaut (compte) | Compte expéditeur par défaut |
| Badge OAuth (vert) | Compte connecté par OAuth |
| Badge Mot de passe (cadenas) | Compte connecté par mot de passe applicatif |
| Badge Sync auto (éclair vert) / Sync off (gris) | Synchronisation automatique activée / désactivée |
| Historique : OK / Erreur / En cours | Résultat d'une synchronisation |

### 4.10 Chiffrement et sécurité

- **Chiffrement au repos (Fernet)** : les mots de passe applicatifs IMAP/SMTP **et** les jetons OAuth (accès et rafraîchissement) sont chiffrés en base, avec une clé dérivée d'une variable de serveur. L'API n'expose jamais les secrets (elle ne renvoie que des indicateurs « a un mot de passe » / « a un OAuth »).
- **Assainissement du HTML reçu** : le contenu est nettoyé à l'affichage (analyseur HTML réel, liste blanche de balises, d'attributs et de schémas d'URL, forçage de `rel="noopener"` contre le détournement d'onglet) et déjà nettoyé côté serveur à la réception. Le contenu des balises dangereuses (script, style, etc.) est supprimé, texte compris.
- **Réception (webhook)** : point d'accès public mais protégé par cryptographie — jeton Bearer (n8n) ou signature HMAC (Mailgun, avec protection anti-rejeu). Il renvoie toujours un succès pour éviter les tempêtes de réessais.
- **Retour OAuth** : point d'accès public protégé par un état signé HMAC à durée de vie limitée ; le compte est rattaché à l'utilisateur d'origine (anti-vol de compte).
- **Protection SSRF** : la connexion refuse tout serveur IMAP/SMTP qui pointerait vers une adresse réseau interne.
- **Isolation** : chaque requête est cloisonnée à votre entreprise ; les comptes sont filtrés par propriétaire.

---

## 5. Intégrations et FAQ

### 5.1 Intégration avec le CRM

À la synchronisation (et à la réception par webhook), un courriel entrant est **rattaché automatiquement** au CRM : d'abord au **contact** dont l'adresse correspond exactement, sinon à l'**entreprise** dont le domaine correspond. Les domaines publics (gmail, outlook, yahoo, etc.) sont ignorés pour éviter les faux rapprochements. Le contexte client sert aussi à personnaliser les suggestions de l'IA.

### 5.2 Intégration avec les notifications

À la réception d'un courriel sur l'**adresse interne** (webhook), une **notification** est envoyée aux administrateurs de l'entreprise, avec un lien vers le module Courriels.

### 5.3 Distinction avec les autres modules de communication

- **Messagerie** (interne, entre utilisateurs de l'ERP) : différente de ce module, qui gère les courriels externes.
- **Agent vocal** : voisin dans la barre latérale, sans lien fonctionnel avec les courriels.
- Les modules **Devis**, **Factures** et autres possèdent leurs propres envois ; ce module ne s'y substitue pas.

### 5.4 Ce qui n'est pas possible (limites connues)

- **Enregistrer un brouillon depuis la composition** : il n'y a pas de bouton « Enregistrer comme brouillon ». Le dossier Brouillons ne se remplit donc pas à la demande depuis l'interface (il peut recevoir un envoi ayant échoué, ou des brouillons synchronisés depuis le serveur).
- **Gérer les modèles** : les modèles sont en lecture seule ; aucun ajout, modification, duplication ou suppression depuis l'interface.
- **Répondre à tous / Transférer** : non disponibles (seul « Répondre » existe). **Archiver** a été retiré volontairement.
- **Marquer manuellement comme lu / non lu** : la lecture marque automatiquement comme lu ; il n'y a pas de bouton inverse.
- **Fil de discussion** : pas de vue conversation regroupée.
- **Filtres avancés** (par lu / non lu, par favori) : non exposés dans l'interface (seuls le dossier et la recherche texte le sont).
- **Envoyer « au nom de » l'adresse interne** : l'adresse interne n'est pas proposée comme expéditeur ; le repli vers le relais interne se fait via « Compte par défaut ».
- **Export / impression** : aucun export PDF ou CSV, aucune impression dans ce module.
- **Synchronisation bidirectionnelle** : les actions locales (lu, étoile, corbeille) ne se répercutent pas sur le serveur d'origine.

### 5.5 Questions fréquentes

**Q : Pourquoi mes nouveaux courriels n'arrivent-ils pas tout de suite ?**
R : Utilisez l'onglet **Synchronisation** (« Nouveaux uniquement »). Une synchronisation automatique existe pour les comptes dont « Sync auto » est activée (intervalle par défaut de 15 minutes) ; l'adresse interne, elle, reçoit en continu via webhook (rafraîchissement de l'interface toutes les 60 secondes).

**Q : Mon mot de passe Gmail / Outlook / iCloud est refusé.**
R : Avec la validation en deux étapes, il faut un **mot de passe applicatif** (Gmail : `myaccount.google.com/apppasswords` ; Outlook et iCloud : section sécurité du compte). Ou utilisez l'**OAuth**.

**Q : Après une connexion OAuth Microsoft 365 réussie, la synchronisation échoue.**
R : Le locataire Azure doit autoriser IMAP et SMTP pour OAuth (Microsoft a désactivé l'authentification de base par défaut). L'administrateur de votre organisation doit les activer.

**Q : Le bouton OAuth est grisé.**
R : Les identifiants du serveur ne sont pas configurés (voir §4.5). Contactez votre administrateur.

**Q : Si je supprime un courriel dans la Corbeille de l'ERP, est-il supprimé sur Gmail aussi ?**
R : Non. Les actions sont locales à l'ERP ; il n'y a pas de synchronisation bidirectionnelle.

**Q : Comment envoyer depuis ma propre adresse et non depuis `info@constructoai.ca` ?**
R : Configurez un compte externe (mot de passe applicatif ou OAuth) et choisissez-le comme expéditeur. Sans compte externe, l'envoi passe par le relais de la plateforme (expéditeur `info@constructoai.ca` au nom de votre entreprise, réponse dirigée vers votre adresse interne).

**Q : Puis-je joindre des fichiers ?**
R : Oui, jusqu'à **5 fichiers**, **10 Mo chacun** et **20 Mo au total**.

**Q : L'IA peut-elle envoyer un courriel à ma place ?**
R : Oui, avec **« Répondre auto »** (elle génère et envoie après confirmation). Les autres fonctions (Analyser, Suggérer, Rédiger avec IA) préparent seulement du texte que vous validez. L'onglet **Assistant IA** n'envoie jamais rien.

**Q : L'Assistant IA peut-il voir mes mots de passe de compte ?**
R : Non. Il est limité à la lecture des courriels, fils et modèles ; il n'a aucun accès aux comptes, jetons, ni aux données RH ou de sécurité.

**Q : Combien coûtent les fonctions IA ?**
R : Elles consomment des **crédits IA prépayés** (coût réel du modèle majoré de 30 %). Chaque réponse de l'assistant affiche son coût et le solde restant. Si le solde est insuffisant, l'appel est refusé.

**Q : Puis-je voir les courriels d'un autre utilisateur de mon entreprise ?**
R : Non. Chaque compte appartient à son créateur ; seuls les comptes partagés (comme l'adresse interne) sont visibles par tous.

**Q : Un compte supprimé est-il récupérable ?**
R : Oui. La suppression est une désactivation. Un administrateur peut le réactiver avec **« Restaurer »**, et les courriels restent lisibles.

**Q : Les pièces jointes sont-elles stockées longtemps ?**
R : Elles sont conservées en base de données. À la suppression définitive d'un courriel, ses pièces jointes sont purgées.

**Q : La recherche est-elle complète ?**
R : Elle porte sur l'objet, l'expéditeur, le destinataire et le corps texte. Il n'y a pas de filtres avancés dans l'interface.

---

## 6. Récapitulatif

- **Client IMAP/SMTP multi-comptes** intégré à l'ERP, façon Outlook, avec une **adresse interne** par entreprise (réception par webhook n8n / Mailgun).
- **8 onglets** (Boîte de réception, Nouveau message, Envoyés, Brouillons, Modèles, Configuration, Synchronisation, Assistant IA) + la **Corbeille** comme 4e dossier.
- **30 points d'accès** : 29 dans `emails.py`, 1 dans `emails_ai.py`.
- **Deux méthodes de connexion** : mot de passe applicatif (chiffré Fernet) ou **OAuth** (Google et Microsoft 365) ; les **jetons OAuth sont aussi chiffrés au repos**.
- **Trois modes de synchronisation** (Nouveaux / 50 derniers / Tous ≤ 200) + **synchronisation automatique** en arrière-plan (comptes « Sync auto », défaut 15 min).
- **Envoi réel** avec pièces jointes (5 fichiers, 10 Mo/fichier, 20 Mo au total), modèles et signature ; repli par le relais de la plateforme sans compte externe.
- **Deux surfaces d'IA à ne pas confondre** : rédaction / réponse (dont **Répondre auto** qui envoie) dans `emails.py`, et **Assistant IA de consultation en lecture seule** dans `emails_ai.py` (limité à 3 tables, sans accès aux comptes).
- **Facturation IA** par crédits prépayés (coût du modèle × 1,30) ; le vrai frein est le solde, pas le rôle.
- **6 modèles pré-installés** en lecture seule, à variables `{{...}}` substituées à l'envoi.
- **Sécurité** : chiffrement au repos, assainissement HTML (analyseur HTML réel, anti-détournement d'onglet), webhook et retour OAuth protégés par cryptographie, protection SSRF, isolation par entreprise et par propriétaire.
- **Ne fait pas** : brouillon manuel, gestion des modèles, « Répondre à tous » / « Transférer » / « Archiver », marquage manuel lu/non lu, vue de fil, filtres avancés, export / impression, synchronisation bidirectionnelle.

---

**Documentation générée à partir du code source vérifié** : `ERP_REACT/backend/routers/emails.py` (6993 lignes, 29 points d'accès), `ERP_REACT/backend/routers/emails_ai.py` (332 lignes, 1 point d'accès), `ERP_REACT/frontend/src/pages/EmailsPage.tsx`, `ERP_REACT/frontend/src/components/emails/EmailAccountsPanel.tsx`, `EmailSyncPanel.tsx`, `EmailAIPanel.tsx`, `EmailAIComposeButton.tsx`, `EmailsAssistantTab.tsx`, `ERP_REACT/frontend/src/api/emails.ts`, `ERP_REACT/frontend/src/store/useEmailsStore.ts`, fichiers de traduction `i18n/locales/fr/emails.json` et `emailsAssistant.json`.

**Manuels liés** :
- Module CRM (rapprochement automatique des contacts et entreprises)
- Module Devis / Soumissions (envois propres)
- Module Factures (envois propres)
- Module IA / Assistant (crédits IA partagés)
- Module Messagerie (messagerie interne entre utilisateurs — distincte de ce module)
- Module Administration (variables de serveur, OAuth, clé de chiffrement)
