# Module 24 — Messagerie interne

> **Version** : 3.0 (refonte vérifiée ligne par ligne par rapport au code source, 2026-07-07)
> **Menu** : section « COMMUNICATION » de la barre latérale → **Messagerie** (icône `MessageSquare`) — voisins : **Emails**, **Agent vocal**
> **Route** : `/messagerie`
> **Code de référence (backend)** : `backend/routers/messaging.py` (1137 lignes, 13 points d'accès — 6 canaux/messages, 3 messages directs inactifs, 4 notifications ; router sans préfixe monté sous `/api/erp/v1`) ; `backend/routers/messagerie_ai.py` (322 lignes, 1 point d'accès `POST /messagerie/ai/chat`, préfixe `/messagerie/ai`)
> **Code de référence (frontend)** : `frontend/src/pages/MessagingPage.tsx` (502 lignes) ; `frontend/src/components/messaging/MessageAttachments.tsx` (263 lignes) ; `frontend/src/components/messagerie/MessagerieAssistantTab.tsx` (122 lignes) ; `frontend/src/api/messaging.ts` (133 lignes) + `frontend/src/api/messagerieAi.ts`
> **Libellés** : `i18n/locales/fr/messaging.json` (43 lignes) + `i18n/locales/fr/messagerieAssistant.json` (15 lignes)
> **Tables PostgreSQL (par tenant)** : `conference_channels`, `conference_messages`, `conference_reactions`, `conference_members`, `conference_attachments`, `notifications`
> **Cadrage** : messagerie d'équipe interne de type Teams / Slack, propre à chaque entreprise (tenant). Des **canaux** (`#`) contenant un **fil de messages** chronologique, avec **réactions émoji** et **pièces jointes**. S'y ajoutent un **Assistant IA en lecture seule** (résumé et recherche des messages) et un système de **notifications** distinct (la cloche du bandeau supérieur, hors de cette page). Côté web, c'est surtout un **client léger de consultation et de participation** : la création de canaux privés, l'envoi de pièces jointes, l'édition et la suppression de messages sont assurés par l'**application mobile**, qui partage les mêmes tables.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas à pas](#3-workflows-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

La Messagerie interne offre à tous les utilisateurs d'une même entreprise un espace d'échange rapide, sur le modèle de Microsoft Teams ou Slack :

- **Canaux thématiques** (`#general`, `#chantier-rive-sud`, `#soumissions`, etc.) contenant des messages texte.
- **Réactions émoji** sur chaque message (six émojis rapides), en bascule (on ajoute ou on retire d'un clic).
- **Pièces jointes** (images et fichiers) affichées dans le fil, avec une visionneuse d'images plein écran et le téléchargement des fichiers.
- **Assistant IA** en lecture seule, ouvert dans une fenêtre modale, qui résume un canal, retrouve un message ou compte l'activité, en s'appuyant sur les messages réels de l'entreprise.
- **Recherche** locale dans le canal courant.

C'est un outil de communication **interne à l'entreprise** : il n'est pas destiné aux clients. Les échanges avec les clients passent par les modules CRM (interactions), Emails et le portail B2B.

### 1.2 Accès et disposition

- Barre latérale → section **COMMUNICATION** → **Messagerie** (icône `MessageSquare`), à côté de **Emails** et de **Agent vocal**.
- URL : `/messagerie`. La page est protégée : il faut être authentifié.
- **Disposition à 2 volets** (aucun onglet) :
  1. À gauche, la barre **« Canaux »** (largeur fixe sur écran large).
  2. À droite, la **zone de messages** (en-tête du canal, fil, zone de saisie).
- Deux fenêtres modales se superposent au besoin : l'**Assistant IA** et **Nouveau canal**.
- Hauteur adaptative : `calc(100vh - 120px)` sur mobile, `calc(100vh - 180px)` sur ordinateur.
- **Adaptatif mobile** : en dessous du seuil `md` (768 px), l'écran affiche soit la liste des canaux, soit le fil du canal choisi, avec un bouton retour.

> **Important — aucun onglet.** La page n'a pas de système d'onglets. Elle a deux volets et deux modales. Les canaux ne sont pas des onglets fixes : ils sont **créés par les utilisateurs** et donc entièrement dynamiques, propres à chaque entreprise.

### 1.3 Permissions

- Tout utilisateur authentifié de l'entreprise peut : lister les canaux visibles, créer un canal, lire les messages, poster un message, réagir avec un émoji, télécharger une pièce jointe et interroger l'Assistant IA.
- **Aucun rôle de modérateur ni d'administrateur de canal** n'est requis : tous les utilisateurs ont les mêmes droits d'accès aux canaux publics. Il n'y a pas de garde de rôle (`is_admin`, `comptable`, `super_admin`) sur les points d'accès de la messagerie.
- **Canaux privés** : leur accès est réservé à leurs membres. L'appartenance à un canal privé est gérée par l'application mobile (voir la section 5.1). Un utilisateur du web qui n'a pas de fiche employé liée ne voit que les canaux **publics** — c'est une dégradation sûre, jamais une fuite.
- L'**Assistant IA** est ouvert à tout utilisateur authentifié, mais son usage consomme des crédits IA prépayés (voir la section 4.9).

### 1.4 Sous-modules (surfaces de la page)

| Surface | Où | Rôle |
|---|---|---|
| Barre « Canaux » | Volet gauche | Liste des canaux, création, ouverture de l'Assistant IA |
| Zone de messages | Volet droit | En-tête, fil chronologique, réactions, pièces jointes, saisie |
| Modale « Nouveau canal » | Superposée | Créer un canal (nom + description) |
| Modale « Assistant IA — Messagerie » | Superposée | Dialogue IA en lecture seule (résumé, recherche) |
| Notifications (cloche) | **Hors de cette page**, dans le bandeau supérieur | Alertes générées par d'autres modules |

---

## 2. Interface

### 2.1 Barre latérale « Canaux » (volet gauche)

**En-tête.** Le titre **« Canaux »**, suivi de deux boutons d'action :

- **Assistant IA** (icône `Sparkles`) : ouvre la modale de l'Assistant IA. L'infobulle affiche **« Assistant IA »**.
- **Nouveau canal** (icône `Plus`) : ouvre la modale de création. L'infobulle affiche **« Nouveau canal »**.

**Liste des canaux.** Chaque ligne affiche :

- L'icône `#` (dièse) suivie du **nom du canal** (tronqué s'il est long).
- À droite, deux compteurs discrets :
  - Le **nombre de membres** (icône `Users` + chiffre), affiché **seulement s'il est supérieur à 0**. Infobulle : « N membre(s) ».
  - Le **nombre de messages** (chiffre seul), affiché seulement s'il est supérieur à 0.
- Le canal actif est surligné dans la couleur d'accent.

> **Pourquoi « 0 membre » sur un canal que je viens de créer ?** Un canal **public** créé depuis le web n'inscrit aucune ligne de membre : son compteur de membres reste donc à 0, même si toute l'entreprise peut y écrire. L'interface masque simplement le badge quand le compteur vaut 0. Les canaux dont le nombre de membres est visible sont en général des canaux **privés** gérés par l'application mobile.

**État vide.** Si aucun canal n'existe encore : **« Aucun canal »**.

**Chargement.** Au premier affichage, une roue d'attente occupe l'écran, puis le **premier canal** de la liste (ordre alphabétique) est **sélectionné automatiquement**.

### 2.2 En-tête du canal actif (volet droit)

- Sur mobile, un bouton retour (`ChevronLeft`, infobulle **« Retour aux canaux »**) ramène à la liste.
- L'icône `#` + le **nom du canal**.
- Un badge gris **« N membres »** si le canal compte au moins un membre.
- Un **champ de recherche** (icône `Search`, texte d'invite **« Rechercher... »**) avec un bouton d'effacement `X` (infobulle **« Effacer la recherche »**).
- La **description** du canal, affichée sous l'en-tête si elle a été renseignée.

### 2.3 Fil de messages

Le fil est **chronologique et à plat** : les messages les plus anciens en haut, les plus récents en bas. Il n'y a pas de vue en fil de discussion (thread) ni de repli des réponses.

**Fenêtre chargée.** Le web charge les **100 messages les plus récents** du canal (constante `MSG_FETCH_LIMIT = 100`). Un nouveau message qui arrive pousse le plus ancien hors de la fenêtre.

**Rafraîchissement automatique.** Tant qu'un canal est ouvert, le fil se rafraîchit **toutes les 30 secondes** en arrière-plan (interrogation périodique silencieuse). Si un collègue écrit pendant que vous lisez, son message apparaît en moins de 30 secondes sans action de votre part.

**Défilement automatique.** À l'ouverture d'un canal, le fil défile instantanément jusqu'au dernier message. À l'arrivée d'un nouveau message en bas, le défilement est fluide. Si vous remontez lire d'anciens messages sans qu'un nouveau n'arrive, la vue ne redescend pas de force.

**Contenu d'un message.**

- Un **avatar** rond (la première lettre du nom de l'auteur).
- Le **nom de l'auteur**. À défaut, le libellé **« Utilisateur »**.
- La **date relative** (« il y a 5 min », « hier »...).
- La mention **« (modifié) »** si le message a été édité (édition faite depuis le mobile).
- Le **texte** du message, qui conserve les retours à la ligne.
- Les **pièces jointes** éventuelles (voir la section 2.5).
- La **ligne de réactions** (voir la section 2.4).

**États particuliers.**

- Canal ouvert mais vide : icône `MessageSquare` + **« Aucun message dans #canal »** + **« Soyez le premier à écrire ! »**.
- Aucun canal sélectionné : icône `Hash` + **« Sélectionnez un canal »**.

### 2.4 Réactions émoji

Sous chaque message figurent des **pastilles** `emoji + nombre`. Un clic bascule votre réaction :

- Si vous n'aviez pas encore réagi avec cet émoji, il est **ajouté** (infobulle **« Ajouter une réaction »**).
- Si vous aviez déjà réagi, il est **retiré** (infobulle **« Retirer ma réaction »**). Vos propres réactions sont surlignées dans la couleur d'accent.

Au survol d'un message, un **sélecteur rapide** affiche les émojis **non encore utilisés** sur ce message, en transparence, pour réagir en un clic.

**Émojis disponibles (six, fixes)** : 👍 ❤️ 😄 🎉 🤔 👀. La même palette sert aux réactions et au sélecteur de la zone de saisie. Il n'y a pas de sélecteur d'émoji libre.

Une protection empêche le double-clic accidentel sur la même réaction (verrou synchrone côté interface), et la base impose l'unicité `(message, utilisateur, emoji)` : on ne peut pas poser deux fois le même émoji sur un message.

### 2.5 Pièces jointes

Les pièces jointes sont **écrites par l'application mobile** ; le web les **affiche et les télécharge** (lecture seule). Le composant récupère chaque fichier par un appel authentifié, sans URL publique.

- **Images** : miniatures de 96 × 96 pixels. Un clic ouvre une **visionneuse plein écran** avec :
  - Navigation **Précédente / Suivante** entre les images du message (flèches à l'écran ou touches ← et →).
  - Fermeture par le bouton `X` ou la touche Échap.
  - Un compteur « i / N » quand il y a plusieurs images.
  - En cas d'échec de chargement : **« Image indisponible »**.
- **Fichiers non-image** (PDF, Word, Excel...) : une carte téléchargeable affichant une icône selon le type (`FileText` pour PDF et Word, `FileSpreadsheet` pour Excel, icône générique sinon), le nom du fichier et sa taille (« o », « Ko », « Mo »), avec une icône `Download`. Un clic télécharge le fichier.

> **Limite de téléchargement : 25 Mo** par pièce jointe. Les images d'un type sûr (JPEG, PNG, GIF, WebP, AVIF) sont servies en affichage direct ; tout le reste est forcé en téléchargement, avec les en-têtes de sécurité appropriés (anti-détournement de type, politique de contenu restrictive).

### 2.6 Zone de saisie

En bas du volet droit :

- Un **bouton émoji** (icône `Smile`, infobulle **« Ajouter un émoji »**) ouvre un petit sélecteur des six émojis. L'émoji choisi est inséré **à la position du curseur** dans le texte. Le sélecteur se ferme par un clic à l'extérieur ou la touche Échap.
- Un **champ texte** (invite **« Message dans #canal... »**). L'envoi se fait par la touche **Entrée** (sans Maj).
- Un **bouton d'envoi** (icône `Send`, infobulle **« Envoyer »**), désactivé si le champ est vide ou si un envoi est déjà en cours. Une protection anti-double-envoi empêche l'envoi en double sur un appui répété.

> **Un seul type d'envoi depuis le web : du texte (et des émojis).** On ne peut pas joindre de fichier depuis le web : les pièces jointes sont ajoutées par l'application mobile. Voir la section 5.1.

### 2.7 Modale « Nouveau canal »

Ouverte par le bouton `Plus`. Deux champs :

- **« Nom du canal * »** (obligatoire, invite « general »). Longueur maximale : 100 caractères.
- **« Description »** (facultatif, invite « Description du canal... »). Longueur maximale : 2000 caractères.

Boutons : **« Annuler »** et **« Créer »**. Une protection anti-double-création empêche de créer deux canaux identiques sur un double-clic.

> **Aucune option « privé » ni « type » dans le formulaire web.** Le canal créé depuis le web est toujours **public** : le formulaire n'envoie que le nom et la description. La création de canaux privés se fait dans l'application mobile.

### 2.8 Modale « Assistant IA — Messagerie »

Ouverte par le bouton `Sparkles` de la barre « Canaux ». C'est une **fenêtre modale** (et non un onglet, malgré son libellé interne).

- **Titre** : **« Assistant IA — Messagerie »**.
- **Sous-titre** : **« Consulte, résume et retrouve tes messages et canaux (lecture seule). »**
- **Écran d'accueil** : un texte d'invite précisant que l'assistant lit vos messages réels, n'envoie aucun message et n'accède ni aux comptes utilisateurs ni aux données de paie ou de ressources humaines. Trois **exemples cliquables** sont proposés :
  1. « Résume les derniers messages du canal Projet. »
  2. « Y a-t-il des messages qui mentionnent une livraison ? »
  3. « Combien de canaux et de messages avons-nous ? »
- **Saisie** : une zone de texte (invite « Pose ta question sur la messagerie… »). L'envoi se fait par **Entrée** ; **Maj + Entrée** insère un retour à la ligne.
- Pendant le traitement : l'indicateur **« Analyse en cours… »**.
- En cas de problème : **« Une erreur est survenue. Réessaie. »**
- **Réponses** : présentées en bulles, avec des métadonnées (profil « Messagerie », nombre de jetons, coût, durée).

L'assistant est en **lecture seule** : il peut lire le contenu des messages et des canaux, mais il n'écrit rien et n'envoie aucun message. Son périmètre de lecture est strictement limité aux canaux, messages, membres et réactions ; il n'a accès ni aux comptes, ni à la paie, ni aux données sensibles (voir la section 4.8).

### 2.9 Notifications (la cloche du bandeau — hors de cette page)

La messagerie **n'affiche pas la cloche elle-même** : celle-ci vit dans le bandeau supérieur (`TopBar`), sur toutes les pages. Le module Messagerie fournit néanmoins les points d'accès qui l'alimentent :

- Une **cloche** (`Bell`) avec un badge rouge indiquant le nombre de notifications non lues (plafonné à l'affichage « 99+ »).
- Un menu déroulant intitulé **« Notifications »**, avec un bouton **« Tout marquer lu »** et la liste des dernières notifications (titre, message, date relative, point pour les non lues).
- Le compteur est réinterrogé **toutes les 60 secondes** ; la liste (20 éléments au maximum) est chargée à l'ouverture du menu. Un clic marque la notification comme lue et navigue vers le lien associé.

> **Attention — les notifications ne concernent PAS les messages de canal.** La cloche est un système générique alimenté par **d'autres modules** (devis, factures, courriels). **Poster un message dans un canal ne génère aucune notification**, et il n'existe aucun système de mention `@utilisateur`. Voir la section 5.2.

---

## 3. Workflows pas à pas

### 3.1 Ouvrir un canal et lire les messages

1. Ouvrir **Messagerie** (menu latéral, section COMMUNICATION).
2. Au chargement, le premier canal est sélectionné automatiquement ; son fil s'affiche.
3. Pour changer de canal, cliquer sur un autre canal dans la barre de gauche. Le champ de recherche se réinitialise et le fil défile jusqu'au dernier message.
4. Le fil se rafraîchit tout seul toutes les 30 secondes.

### 3.2 Envoyer un message

1. Sélectionner le canal voulu.
2. Cliquer dans le champ **« Message dans #canal... »** et saisir le texte.
3. (Facultatif) Cliquer l'icône `Smile` pour insérer un émoji à la position du curseur.
4. Appuyer sur **Entrée** (ou cliquer le bouton `Send`).
5. Le message part vers `POST /channels/{id}/messages` ; le fil se rafraîchit en silence et affiche votre message.

> Limite : un message fait au maximum 10 000 caractères (au-delà, l'envoi est refusé avec une erreur de validation).

### 3.3 Réagir à un message

1. Survoler le message : les émojis non encore utilisés apparaissent en transparence.
2. Cliquer un émoji pour **ajouter** votre réaction, ou cliquer une pastille déjà surlignée pour **retirer** la vôtre.
3. Le compteur se met à jour au rafraîchissement suivant.

### 3.4 Créer un canal

1. Cliquer le bouton `Plus` de la barre « Canaux ».
2. Saisir un **Nom** (par exemple `chantier-rive-sud`).
3. (Facultatif) Saisir une **Description** (par exemple « Suivi quotidien du projet Rive-Sud »).
4. Cliquer **« Créer »**.
5. Le canal, **public**, apparaît dans la liste. Tous les utilisateurs de l'entreprise le voient et peuvent y écrire ; aucune invitation n'est nécessaire.

> Astuce : pour un échange à deux, créez simplement un canal dédié (par exemple `coordo-marie-jean`). Les messages directs privés ne sont pas offerts depuis le web (voir la section 5.4).

### 3.5 Rechercher dans un canal

1. Sélectionner un canal.
2. Saisir un mot-clé dans le champ **« Rechercher... »** de l'en-tête.
3. Le fil se filtre **instantanément**. Un compteur « N résultat(s) pour "..." » s'affiche, sous la mention **« Recherche dans les messages chargés »**.
4. Cliquer le bouton `X` pour revenir au fil complet.

> **La recherche est locale** : elle ne porte que sur les messages déjà chargés (la fenêtre des 100 plus récents). Elle ne remonte pas dans tout l'historique du canal.

### 3.6 Consulter une pièce jointe

1. Dans le fil, repérer les miniatures d'images ou les cartes de fichier sous un message.
2. Cliquer une miniature pour ouvrir la **visionneuse plein écran** ; naviguer avec les flèches ← / → ; fermer avec Échap.
3. Cliquer une carte de fichier pour **télécharger** le document (PDF, Word, Excel...).

### 3.7 Interroger l'Assistant IA

1. Cliquer le bouton `Sparkles` de la barre « Canaux ».
2. Dans la modale, saisir une question (ou cliquer un exemple).
3. Appuyer sur **Entrée**. L'indicateur « Analyse en cours… » apparaît.
4. L'assistant répond en s'appuyant sur les messages réels de votre entreprise. Chaque réponse consomme des crédits IA (voir la section 4.9).

> Exemples utiles : « Résume ce qui s'est dit cette semaine dans #chantier-rive-sud », « Trouve les messages qui parlent de béton », « Combien de messages a-t-on échangés au total ? »

### 3.8 Naviguer sur mobile

1. À l'ouverture, la liste des canaux occupe tout l'écran.
2. Cliquer un canal bascule vers son fil, plein écran.
3. Le bouton retour (`ChevronLeft`, « Retour aux canaux ») ramène à la liste.

### 3.9 Traiter une notification (cloche du bandeau)

1. Cliquer la cloche du bandeau supérieur (présente sur toutes les pages).
2. Le menu « Notifications » liste les dernières alertes.
3. Cliquer une notification la marque comme lue et ouvre l'élément lié (un devis, une facture, un courriel...).
4. Le bouton « Tout marquer lu » vide le badge d'un coup.

---

## 4. Référence

### 4.1 Points d'accès — canaux et messages (`messaging.py`)

Tous exigent un jeton JWT valide et un contexte tenant (`user.schema`) ; sinon **400 « Contexte tenant manquant »**. Aucun n'impose de rôle particulier. Préfixe global : `/api/erp/v1`.

| Méthode | URL | Rôle |
|---|---|---|
| GET | `/channels` | Liste les canaux actifs ; masque les canaux privés dont l'appelant n'est pas membre ; renvoie le nombre réel de membres et de messages |
| POST | `/channels` | Crée un canal (transaction atomique ; persiste `channel_type` et `is_private`) |
| GET | `/channels/{id}/messages` | Liste les messages (pagination `page` ≥ 1, `per_page` défaut 50, **max 200**) ; enrichit chaque message de ses réactions et pièces jointes |
| GET | `/channels/attachments/{id}` | Télécharge le binaire d'une pièce jointe (lecture seule) |
| POST | `/channels/{id}/messages` | Poste un message (valide que `parent_message_id`, s'il est fourni, appartient au même canal) |
| POST | `/channels/{id}/messages/{mid}/reactions` | Bascule une réaction émoji (ajoute ou retire) |

> Le web charge 100 messages par appel ; le maximum serveur est de 200 par page. Il n'y a pas de bouton « Charger plus d'anciens messages » dans l'interface web.

### 4.2 Points d'accès — messages directs (inactifs)

| Méthode | URL | Comportement réel |
|---|---|---|
| GET | `/direct-messages` | Renvoie toujours `{ items: [], unread_count: 0 }` (table absente) |
| POST | `/direct-messages` | **503** « Service de messages directs temporairement indisponible. » |
| PUT | `/direct-messages/{id}/read` | **503** (même message) |

> Les messages directs privés ne sont **pas fonctionnels**. Le renvoi d'un code 503 (plutôt qu'un faux 200) évite de laisser croire à une livraison. Il n'existe aucune interface de messages directs dans le web.

### 4.3 Points d'accès — notifications (cloche du bandeau)

| Méthode | URL | Rôle |
|---|---|---|
| GET | `/notifications` | Liste les notifications de l'utilisateur (`unread_only` optionnel ; `limit` défaut 20, **max 50**) ; renvoie une liste vide si la table n'existe pas |
| GET | `/notifications/count` | Compteur des non lues (pour la cloche) |
| PUT | `/notifications/{id}/read` | Marque une notification comme lue |
| PUT | `/notifications/read-all` | Marque toutes les notifications comme lues |

### 4.4 Point d'accès — Assistant IA (`messagerie_ai.py`)

| Méthode | URL | Rôle |
|---|---|---|
| POST | `/messagerie/ai/chat` | Assistant IA en lecture seule : répond à partir des messages réels ; n'écrit rien, n'envoie aucun message |

Paramètres du corps : `message` (1 à 8000 caractères), `history` (40 éléments au maximum, tronqués à 12 tours de conversation côté serveur), `language` (optionnel, FR ou EN). Modèle utilisé : `claude-sonnet-4-6`. Plafond de génération : 8000 jetons. Jusqu'à 5 allers-retours d'outil par réponse.

### 4.5 Limites de saisie (validation → 422)

| Champ | Règle |
|---|---|
| Nom du canal | 1 à 100 caractères (obligatoire) |
| Description du canal | 2000 caractères au maximum |
| Type de canal | 50 caractères au maximum (« general » par défaut) |
| Texte d'un message | 1 à 10 000 caractères |
| Émoji d'une réaction | 10 caractères au maximum (colonne `VARCHAR(10)`) |
| Question à l'Assistant IA | 1 à 8000 caractères |

### 4.6 Codes de statut HTTP

| Code | Signification dans ce module |
|---|---|
| 200 | Succès |
| 400 | Contexte tenant manquant, schéma de tenant invalide, ou nom de canal vide |
| 402 | Crédits IA épuisés (Assistant IA) |
| 403 | Canal privé, appelant non membre (lecture, envoi, réaction) ; ou accès IA refusé |
| 404 | Message ou canal introuvable ; pièce jointe d'un canal privé non accessible (masquée pour ne pas révéler son existence) |
| 413 | Pièce jointe dépassant 25 Mo au téléchargement |
| 422 | Donnée hors bornes (voir la section 4.5) |
| 503 | Messages directs (toujours) ; ou service IA momentanément indisponible |

### 4.7 Émojis, rafraîchissement et constantes

| Élément | Valeur | Source |
|---|---|---|
| Émojis (réactions et saisie) | 👍 ❤️ 😄 🎉 🤔 👀 (six, fixes) | `MessagingPage.tsx:24` |
| Messages chargés par le web | 100 (les plus récents) | `MSG_FETCH_LIMIT`, `MessagingPage.tsx:28` |
| Rafraîchissement du fil | 30 secondes | `MessagingPage.tsx:108` |
| Rafraîchissement du compteur de la cloche | 60 secondes | `TopBar.tsx` |
| Pagination serveur des messages | défaut 50, max 200 | `messaging.py:372` |
| Limite de téléchargement d'une pièce jointe | 25 Mo | `messaging.py:60` |
| Limite débit de l'Assistant IA | 20 requêtes par IP et par fenêtre | `erp_api.py` |

### 4.8 Tables PostgreSQL (par tenant)

| Table | Rôle | Qui écrit |
|---|---|---|
| `conference_channels` | Canaux : nom, description, type, `is_private`, `is_active`, créateur, date | Web et mobile |
| `conference_messages` | Messages : canal, auteur, texte, message parent, indicateurs édité/supprimé, dates | Web (envoi) et mobile |
| `conference_reactions` | Réactions : message, utilisateur, émoji ; unicité `(message, utilisateur, emoji)` | Web et mobile (bascule) |
| `conference_members` | Membres des canaux privés ; l'appartenance est indexée par identifiant d'employé | Mobile |
| `conference_attachments` | Pièces jointes (binaire en base) | Mobile (écriture) ; web (lecture seule) |
| `notifications` | Notifications génériques (cloche) alimentées par d'autres modules | Autres modules (devis, factures, courriels) |

> **Provisionnement.** Les tables `conference_*` ne sont pas créées par l'initialisation de l'ERP : elles proviennent de l'application mobile et de migrations historiques. L'ERP se contente d'auto-réparer `conference_attachments` (création à la volée si absente). La table `notifications` n'est créée par aucun composant de l'ERP : les points d'accès vérifient son existence et renvoient une liste vide si elle manque. La messagerie web dépend donc d'un schéma provisionné ailleurs.

**Whitelist de lecture de l'Assistant IA** : uniquement `conference_channels`, `conference_messages`, `conference_members`, `conference_reactions`. Toute requête nommant `users`, `employees`, la paie, les salaires, le NAS, les courriels, les jetons, Stripe, etc. est **refusée** par un garde-fou. Le moteur de lecture est en transaction en lecture seule, avec délai maximal, limite de résultats automatique et masquage des colonnes sensibles. L'assistant **peut lire le contenu des messages** (décision d'affaires), mais rien d'autre.

### 4.9 Calcul du coût de l'Assistant IA

- L'accès à l'IA est ouvert à tout utilisateur authentifié ; le vrai verrou est le **solde de crédits IA prépayés** : si le solde est vide, la requête est refusée avec **402 « Crédits IA épuisés »**.
- Coût facturé par requête : `(jetons_entrée × 0,003 + jetons_sortie × 0,015) / 1000 × 1,30`, soit le tarif de référence majoré de **30 %**. Le coût est débité du solde après chaque réponse réussie.
- Le débit ne comporte **pas de clé d'idempotence** : en cas de nouvel essai réseau ou de double soumission, la requête peut être **débitée deux fois**. La protection anti-double-envoi existe seulement côté interface. Une limite de débit par IP (20 requêtes par fenêtre) réduit ce risque.
- Si l'abonnement le prévoit, un solde bas déclenche une recharge automatique par Stripe.

### 4.10 Raccourcis clavier

| Contexte | Touche | Effet |
|---|---|---|
| Champ de message | Entrée | Envoyer le message |
| Sélecteur d'émoji | Échap | Fermer le sélecteur |
| Zone de l'Assistant IA | Entrée | Envoyer la question |
| Zone de l'Assistant IA | Maj + Entrée | Nouvelle ligne |
| Visionneuse d'images | ← / → | Image précédente / suivante |
| Visionneuse d'images | Échap | Fermer la visionneuse |

---

## 5. Intégrations et FAQ

### 5.1 Intégration avec l'application mobile

L'application mobile est le **client principal** de la messagerie et partage les mêmes tables. Plusieurs fonctions n'existent que là :

- **Canaux privés** : leur création et la gestion de leurs membres se font sur mobile. L'appartenance est indexée par identifiant d'employé. Un utilisateur du web sans fiche employé liée ne voit que les canaux publics.
- **Pièces jointes** : leur ajout se fait sur mobile ; le web les affiche et les télécharge.
- **Édition et suppression de messages** : disponibles sur mobile ; le web affiche la mention « (modifié) » mais n'offre aucune action d'édition ni de suppression.
- **Fils de discussion (threads)** : le lien de message parent est stocké et validé côté serveur, mais le web affiche un fil à plat et n'exploite pas les réponses en fil.

### 5.2 Intégration avec les notifications (cloche du bandeau)

La table `notifications` est **partagée** et alimentée par d'autres modules : un nouveau devis, une facture, un courriel entrant y déposent une alerte, avec un lien cliquable. Le module Messagerie ne fait que **lire, compter et marquer comme lues** ces notifications. Il n'en **crée aucune** : poster un message de canal ne fait pas sonner la cloche, et il n'y a pas de mention `@utilisateur`.

### 5.3 Intégration avec le module Employés

Le nom affiché comme auteur d'un message provient d'une jointure avec les fiches employé et les comptes utilisateurs de l'entreprise (jointures strictement limitées au schéma du tenant, pour éviter toute fuite de noms entre entreprises). Si aucun nom n'est trouvé, le libellé « Utilisateur » est affiché.

### 5.4 Ce qui n'est pas possible depuis le web

- **Messages directs privés** : non fonctionnels (renvoient une liste vide ou un code 503). Solution de contournement : créer un canal dédié à deux personnes.
- **Envoi de pièces jointes** : réservé au mobile.
- **Édition ou suppression de messages** : réservées au mobile.
- **Création de canaux privés** : réservée au mobile.
- **Mentions `@utilisateur`** : inexistantes.
- **Export (PDF, CSV) ou impression** d'un canal : aucun.
- **Statut de présence** (« en ligne », « en train d'écrire »), **accusés de lecture**, **appels audio/vidéo**, **notifications navigateur ou courriel** sur nouveau message : aucun.
- **Recherche dans tout l'historique** : la recherche web ne porte que sur les 100 messages chargés.

### 5.5 FAQ

**Q : Y a-t-il des onglets dans la Messagerie ?**
R : Non. La page a deux volets (canaux à gauche, messages à droite) et deux fenêtres modales (Nouveau canal, Assistant IA). Les canaux sont créés par les utilisateurs, ce ne sont pas des onglets fixes.

**Q : Qui voit le canal que je crée depuis le web ?**
R : Tout le monde dans l'entreprise. Les canaux créés depuis le web sont toujours publics. Les canaux privés se créent dans l'application mobile.

**Q : Pourquoi certains canaux affichent « 0 membre » ?**
R : Un canal public créé depuis le web n'inscrit aucun membre : son compteur reste à 0 (le badge est alors masqué), même si toute l'entreprise peut y écrire. Le nombre de membres est surtout pertinent pour les canaux privés gérés sur mobile.

**Q : Puis-je modifier ou supprimer un message depuis le web ?**
R : Non. L'édition et la suppression se font depuis l'application mobile. Le web affiche la mention « (modifié) » quand un message a été édité.

**Q : Puis-je joindre un fichier à un message depuis le web ?**
R : Non. On peut seulement lire et télécharger les pièces jointes (ajoutées sur mobile). La zone de saisie web n'accepte que du texte et des émojis.

**Q : La recherche remonte-t-elle tout l'historique ?**
R : Non. Elle filtre uniquement les messages déjà chargés (les 100 plus récents du canal courant).

**Q : Est-ce que poster un message avertit mes collègues ?**
R : Non, pas par la cloche de notifications ni par courriel. Vos collègues verront le message en ouvrant le canal ; le fil se rafraîchit tout seul toutes les 30 secondes. Il n'existe pas de mention `@utilisateur`.

**Q : Puis-je écrire `@Marie` pour l'alerter ?**
R : Vous pouvez l'écrire, mais ce n'est que du texte : aucune notification n'est déclenchée. Le système de mentions n'existe pas.

**Q : Les messages directs (privés, un à un) fonctionnent-ils ?**
R : Non depuis le web. Pour un échange restreint, créez un canal dédié.

**Q : Combien d'émojis de réaction sont disponibles ?**
R : Six, fixes : 👍 ❤️ 😄 🎉 🤔 👀. On ne peut pas poser deux fois le même émoji sur un message.

**Q : L'Assistant IA peut-il lire mes messages ? Et voir les salaires ou les mots de passe ?**
R : Il peut lire le contenu des messages et des canaux de votre entreprise, pour les résumer ou les retrouver. Il **ne peut pas** accéder aux comptes utilisateurs, aux mots de passe, à la paie ni aux ressources humaines : sa lecture est verrouillée aux seules tables de la messagerie.

**Q : L'Assistant IA peut-il écrire ou envoyer un message à ma place ?**
R : Non. Il est strictement en lecture seule : il n'écrit rien et n'envoie aucun message.

**Q : L'Assistant IA coûte-t-il quelque chose ?**
R : Oui. Chaque réponse consomme des crédits IA prépayés (tarif de référence majoré de 30 %). Si le solde est épuisé, l'assistant répond « Crédits IA épuisés ». Évitez de renvoyer rapidement la même question : un nouvel essai peut être débité une seconde fois.

**Q : Les messages sont-ils conservés longtemps ?**
R : Oui, indéfiniment ; il n'y a pas de purge automatique.

**Q : La messagerie fonctionne-t-elle sur téléphone ?**
R : La page web est adaptée au mobile (liste des canaux, bascule vers le fil, bouton retour). Il existe par ailleurs une **application mobile** dédiée, qui est le client principal et offre les fonctions avancées (canaux privés, pièces jointes, édition, fils de discussion).

**Q : Puis-je exporter ou imprimer l'historique d'un canal ?**
R : Non, aucune fonction d'export ni d'impression n'est prévue.

---

## 6. Récapitulatif

- **Messagerie d'équipe interne** de type Teams / Slack, propre à chaque entreprise : des canaux, un fil de messages, des réactions émoji, des pièces jointes, un Assistant IA en lecture seule.
- **Accès** : menu latéral, section COMMUNICATION → Messagerie ; route `/messagerie`.
- **Disposition à 2 volets** (canaux + messages), **aucun onglet**, plus 2 modales (Nouveau canal, Assistant IA). Les canaux sont dynamiques, créés par les utilisateurs.
- **Six émojis fixes** : 👍 ❤️ 😄 🎉 🤔 👀 ; réactions en bascule ; unicité `(message, utilisateur, emoji)`.
- **Le web charge 100 messages** par canal ; rafraîchissement du fil toutes les 30 secondes ; recherche **locale** sur la fenêtre chargée.
- **Pièces jointes en lecture seule** côté web (écrites par le mobile) : visionneuse d'images plein écran, téléchargement des fichiers, limite de 25 Mo.
- **Assistant IA** en modale (bouton `Sparkles`) : lecture seule, lit le contenu des messages mais rien de sensible ; modèle `claude-sonnet-4-6` ; coût = tarif de référence + 30 % ; refus à 402 si crédits épuisés ; pas de clé d'idempotence (risque de double débit sur un nouvel essai).
- **Notifications (cloche)** : système **séparé**, dans le bandeau supérieur, alimenté par d'autres modules (devis, factures, courriels). Poster un message ne crée aucune notification ; pas de mention `@utilisateur`.
- **Réservé au mobile** : canaux privés, envoi de pièces jointes, édition et suppression de messages, fils de discussion.
- **Inactif** : messages directs privés (liste vide ou 503).
- **Permissions** : tout utilisateur authentifié a les mêmes droits ; les canaux privés ne sont visibles que par leurs membres (appartenance gérée sur mobile).

---

**Documentation générée à partir du code vérifié** :
- `backend/routers/messaging.py` (1137 lignes, 13 points d'accès)
- `backend/routers/messagerie_ai.py` (322 lignes, 1 point d'accès)
- `frontend/src/pages/MessagingPage.tsx` (502 lignes)
- `frontend/src/components/messaging/MessageAttachments.tsx` (263 lignes)
- `frontend/src/components/messagerie/MessagerieAssistantTab.tsx` (122 lignes)
- `frontend/src/api/messaging.ts` (133 lignes) + `frontend/src/api/messagerieAi.ts`
- `frontend/src/i18n/locales/fr/messaging.json` (43 lignes) + `messagerieAssistant.json` (15 lignes)

**Manuels liés** :
- Module 9 (Employés — résolution des noms d'auteurs) — `09-employes.md`
- Module 23 (Emails — distinct de la messagerie interne) — `25-emails.md`
- Module 25 (Assistant IA — moteur de crédits partagé) — `12-ia.md`
- Module 28 (Administration — table `notifications` partagée, cloche du bandeau) — `14-administration.md`
