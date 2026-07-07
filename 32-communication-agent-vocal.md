# Module 32 — Agent vocal IA (standardiste virtuelle)

> **Version** : 1.0 (rédaction initiale vérifiée contre le code source, juillet 2026)
> **Route** : `/agent-vocal` — menu latéral « Agent Vocal » (section **Communication** de la barre de navigation, aux côtés de « Emails » et « Messagerie »)
> **Code de référence** : `backend/routers/voice.py` (2941 lignes ; 1 point d'accès webhook public + 15 points d'accès de gestion), `backend/routers/voice_admin.py` (690 lignes ; 5 points d'accès super-admin), `backend/voice_provider.py` (165 lignes ; adaptateur Vapi), `frontend/src/pages/AgentVocalPage.tsx` (1559 lignes), `frontend/src/api/voice.ts` (282 lignes), `frontend/src/components/admin/AdminVoiceCallsTab.tsx`, i18n `frontend/src/i18n/locales/{fr,en}/voice.json`
> **Tables PostgreSQL par entreprise (tenant)** : `voice_agent_config`, `voice_calls`, `voice_qualification_questions`, `voice_knowledge_base`, `voice_lookup_attempts`
> **Tables PostgreSQL partagées (`public`)** : `voice_phone_routing`, `voice_assistant_routing`, `voice_calls_index`, `voice_admin_access_log`
> **Cadrage** : ce module est une **standardiste téléphonique automatisée par intelligence artificielle**, nommée **« Clara »**, qui répond aux appels entrants de votre entreprise, **qualifie les nouveaux prospects** (et crée une opportunité au module Ventes), **renseigne le statut d'un dossier existant** (soumission, facture, projet, bon de travail) après vérification d'identité, et peut **transférer l'appel** à une personne. La voix, la transcription et le modèle Claude sont fournis par la plateforme **Vapi** ; l'ERP orchestre la configuration, la sécurité et le journal des appels. Ce n'est **pas** un composeur d'appels sortants, ni un module de messagerie vocale classique.

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'API (endpoint) ; « tenant » désigne votre entreprise (chaque entreprise a ses propres données isolées) ; « Vapi » est la plateforme d'agents vocaux qui compose la téléphonie, la reconnaissance vocale et la synthèse vocale.

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

L'agent vocal IA répond au téléphone à votre place, jour et nuit, avec une voix naturelle en français (Québec) et en anglais. Concrètement, « Clara » :

- **Accueille l'appelant** au nom de votre entreprise et se présente comme une assistante virtuelle (transparence exigée par la Loi 25).
- **Qualifie un nouveau prospect** : elle pose vos questions (nature du projet, type de travaux, région, échéancier, budget, financement, décideur, etc.), confirme les réponses, puis annonce qu'un représentant rappellera. À la fin de l'appel, l'ERP **crée automatiquement une opportunité** au module Ventes (étape « Qualification »), avec une **pré-qualification B.A.T. préliminaire** calculée à partir de l'appel.
- **Renseigne le statut d'un dossier existant** : soumission, facture, projet ou bon de travail. L'appelant donne le **numéro du document** et son **nom** (ou celui de son entreprise) ; l'ERP vérifie l'identité côté serveur puis annonce **le statut seulement**. Un montant, un solde ou un total n'est **jamais** communiqué par téléphone.
- **Transfère l'appel** vers un numéro humain si vous en configurez un et que l'appelant le demande.
- **Consigne chaque appel** : date, durée, langue, résumé, transcription, enregistrement (si consentement) et résultat. Vous consultez tout cela dans le **journal des appels**.
- **Vous avertit par courriel** (facultatif) après chaque appel pertinent.
- **Suit l'usage et le coût** (nombre d'appels, minutes, coût estimé) dans un onglet **Statistiques**.

### 1.2 Comment y accéder

- Barre de navigation latérale → section **Communication** → **Agent Vocal** (icône téléphone).
- Adresse : `/agent-vocal`.
- La page est accessible à **tout utilisateur connecté de l'entreprise** (voir §1.3 pour une précision importante sur les permissions).

### 1.3 Rôles et permissions

- **L'ouverture de la page** exige uniquement d'être connecté à l'ERP (route protégée standard).
- **Point d'attention (sécurité).** Contrairement à plusieurs autres modules de l'ERP (Comptabilité, Configuration, Magasin…) où la gestion est réservée aux administrateurs, **tous les points d'accès de gestion de l'agent vocal** (`/voice/config`, `/voice/questions`, `/voice/knowledge`, `/voice/config/sync`, `/voice/calls`, `/voice/stats`) sont protégés **uniquement par l'authentification** (`get_current_user`), **sans** contrôle de rôle administrateur. En pratique, **n'importe quel utilisateur connecté de votre entreprise** peut lire et modifier la configuration vocale, la base de connaissances, activer ou désactiver l'agent, consulter le journal des appels et déclencher la synchronisation vers Vapi. Si vous souhaitez restreindre ces actions, tenez-en compte dans l'attribution des comptes ; à ce jour, l'interface ne pose pas de garde-fou administrateur.
- **La vue centralisée inter-entreprises** (tous les tenants) est, elle, strictement réservée au **super-administrateur de la plateforme** (voir §2.6 et §4.3). Un administrateur d'entreprise n'y a **aucun** accès.
- En **mode consultation** (abonnement suspendu / lecture seule), les écritures du module (enregistrer la configuration, activer, modifier les questions, synchroniser) sont bloquées par le contrôle global de l'ERP ; la lecture reste possible.

### 1.4 Les sous-modules (onglets)

La page présente en permanence un **bandeau d'état** (statut, numéro branché, langue, bouton d'activation), puis une barre de **quatre onglets** :

| # | Onglet | Rôle |
|---|--------|------|
| 1 | **Journal des appels** *(par défaut)* | Liste des appels reçus + détail d'un appel (résumé, transcription, enregistrement) |
| 2 | **Configuration** | Téléphonie, voix, messages d'accueil, transfert, notifications, synchronisation **+ questions de qualification** |
| 3 | **Base de connaissances** | Fiches d'information (titre / contenu / langue) fournies à l'agent pour répondre aux appelants |
| 4 | **Statistiques** | Usage et coût : appels, minutes, coût estimé, montant facturable, taux de qualification et de transfert |

### 1.5 Ce que le module fait — et ne fait PAS

Le module **fait** : répondre aux appels **entrants**, se présenter au nom de l'entreprise, qualifier un prospect selon vos questions, créer une opportunité et une interaction au CRM, pré-remplir une qualification B.A.T., renseigner le statut d'un dossier après vérification d'identité, transférer vers un humain, enregistrer et transcrire l'appel (avec consentement pour l'audio), envoyer un résumé par courriel, suivre le coût et l'usage, et offrir au super-administrateur une vue centralisée de tous les appels.

Le module **ne fait PAS** :

- **Aucun appel sortant** : l'agent répond, il ne compose jamais de numéros pour prospecter.
- **Aucune communication de montant** par téléphone (prix, solde, total) — c'est une règle de sécurité stricte, imposée côté serveur.
- **Aucun engagement contractuel ni prix ferme** donné à l'oral.
- **Aucun débit automatique de crédits ni facturation Stripe** dans ce module (voir §1.6 et §4.5). Le coût est **suivi**, pas prélevé.
- **Aucune base de connaissances Google** : les fiches de connaissances sont stockées dans votre base de données d'entreprise, pas dans un service externe (voir §5, FAQ).
- **Aucune prise de rendez-vous automatique dans un calendrier** : l'agent recueille les disponibilités, mais c'est un représentant qui rappelle.

### 1.6 Deux modes de fonctionnement

L'agent couvre deux usages, tous deux **pleinement implémentés** :

- **Mode A — Qualification (nouveau prospect).** Clara pose vos questions, l'appel est résumé, et une **opportunité** est créée à la fin de l'appel si le seuil est atteint (un nom + au moins une information utile).
- **Mode B — Statut d'un dossier existant.** Clara demande le **numéro** du document et le **nom**, appelle l'un des quatre outils de consultation, et annonce **le statut seulement**. Toute la vérification d'identité, l'anti-énumération et le verrouillage anti-abus vivent **côté serveur** (l'agent ne décide jamais). Seule la **facturation à l'usage** est différée ; le mode lui-même est complet.

---

## 2. Interface

### 2.1 En-tête et bandeau d'état

En haut de la page : une icône téléphone, le titre **« Agent Vocal »** et le sous-titre « Assistant téléphonique IA — réception et qualification des appels ».

Juste en dessous, un **bandeau d'état** toujours visible (quel que soit l'onglet) présente :

- **Statut** : une pastille verte (« Actif ») ou rouge (« Inactif »), accompagnée d'un badge.
- **Numéro branché** : le numéro de téléphone associé à l'agent (mis en forme), ou « Aucun numéro ».
- **Langue par défaut** : « Français » ou « Anglais ».
- **Bouton d'activation** à droite : **« Activer »** (bleu) quand l'agent est inactif, **« Désactiver »** (gris) quand il est actif. Le bouton affiche un indicateur d'attente pendant l'enregistrement.

*À noter :* la bascule Activer / Désactiver n'écrit que le champ « actif » ; elle ne touche à aucun autre réglage. Le statut « Actif » signale votre intention côté ERP ; la réception effective des appels dépend aussi du branchement du numéro chez Vapi et de la synchronisation (voir §3.1).

### 2.2 Onglet « Journal des appels »

C'est l'onglet ouvert par défaut. Il affiche la liste des appels reçus, du plus récent au plus ancien.

**Colonnes du tableau :**

| Colonne | Contenu |
|---------|---------|
| **Date / heure** | Début de l'appel (ou date d'enregistrement à défaut) |
| **Numéro appelant** | Numéro de l'appelant, mis en forme, ou « -- » si masqué |
| **Durée** | Format `minutes:secondes` (ex. `4:32`), ou « -- » |
| **Langue** | « Français » ou « Anglais » (langue détectée) |
| **Résultat** | Badge coloré : **Qualifié** (vert), **Transféré** (bleu), **Message** (ambre), **Abandonné** (gris) |
| **Opportunité** | Lien **« Opp. #N »** vers le module Ventes si une opportunité a été créée, sinon « -- » |

**Interactions :**

- **Cliquer sur une ligne** ouvre le **détail de l'appel** (fenêtre modale, voir §2.3).
- **Cliquer sur le lien « Opp. #N »** (colonne Opportunité) ouvre le module **Ventes** sans ouvrir le détail.
- Un bouton **« Charger plus »** apparaît quand il y a plus de 50 appels ; il ajoute la page suivante à la liste.
- **État vide** : « Aucun appel pour le moment. »

*Le journal n'affiche aucune donnée financière* (ni coût, ni montant) : ces informations vivent dans l'onglet Statistiques.

### 2.3 Détail d'un appel (fenêtre modale)

Ouverte au clic sur une ligne du journal, la fenêtre « Détail de l'appel » présente :

- **Métadonnées** (en deux colonnes) : Numéro appelant, Début, Durée, Langue, Résultat (badge), et **Consentement à l'enregistrement** (« Oui » / « Non » / « -- »).
- **Lien opportunité** : « Opp. #N » vers le module Ventes, s'il y a lieu.
- **Résumé** : le résumé de l'appel généré par l'IA (dans la langue de l'appel), ou « Aucun résumé disponible. ».
- **Enregistrement** : un lecteur audio natif si l'appel a été enregistré **et** que le consentement a été donné ; sinon « Aucun enregistrement disponible. ». L'audio n'est chargé qu'à la lecture.
- **Transcription** : soit une liste de tours de parole (interlocuteur + texte), soit le texte brut, soit un affichage JSON lisible selon la forme reçue ; « Aucune transcription disponible. » à défaut.

### 2.4 Onglet « Configuration »

Cet onglet regroupe tous les réglages de l'agent, en sections, suivis de la gestion des **questions de qualification**. En cas de succès ou d'erreur, une bande d'alerte s'affiche en haut.

#### 2.4.1 Section « Téléphonie »

- **ID de l'assistant Vapi** : l'identifiant de votre assistant chez Vapi (ex. `161ceedc-ce68-4005-ae8d-473a5d22254a`). Requis pour les appels web ou de test (sans numéro appelé) : il relie l'appel à votre entreprise. Visible dans le tableau de bord Vapi.
- **Numéro de téléphone** : le numéro branché, au format international (ex. `+1 514 555 0199`).
- **Langue par défaut** : menu déroulant « Français » / « Anglais ».
- **Agent bilingue** (case à cocher) : indique que l'agent détecte et bascule dans la langue de l'appelant. *(Dans les faits, le script de l'agent l'invite toujours à basculer de langue si l'appelant change ; la reconnaissance vocale est réglée sur la langue par défaut choisie ci-dessus.)*

#### 2.4.2 Section « Voix et accueil »

- **Identifiant de voix** : identifiant de la voix du fournisseur. Laissez vide pour la voix par défaut (Azure Neural, voix québécoise **Sylvie**). Un identifiant du type `fr-CA-…` est interprété comme une voix Azure ; tout autre identifiant est traité comme une voix native de Vapi.
- **Message d'accueil (français)** et **Message d'accueil (anglais)** : le premier message prononcé au décrochage. Si vous les laissez vides, un message par défaut conforme à la Loi 25 est utilisé (voir §3.1 et §4.6).

#### 2.4.3 Section « Transfert d'appel »

- **Numéro de transfert** : les appels qualifiés ou demandant explicitement une personne sont transférés vers ce numéro. Laissez vide pour désactiver le transfert (l'agent prendra alors un message). Le numéro est normalisé au format international ; s'il est invalide, le transfert est **désactivé proprement** (jamais un mauvais numéro composé).

#### 2.4.4 Section « Notification post-appel »

- **Case « M'avertir par courriel après chaque appel pertinent »** : active l'envoi d'un résumé.
- **Courriel de notification** : l'adresse qui recevra le résumé (résultat, appelant, durée, résumé). Le champ n'est actif que si la case ci-dessus est cochée. Le courriel n'est envoyé que pour les appels porteurs de contenu (opportunité créée **ou** résumé présent).

#### 2.4.5 Section « Synchronisation »

- **Case « Agent géré directement dans Vapi (ne pas synchroniser) »** : cochez-la si le profil de cet agent (script, voix, accueil) est configuré directement dans le tableau de bord Vapi. La synchronisation depuis l'ERP est alors **désactivée** pour ne pas écraser cette configuration externe. C'est le cas, par exemple, d'un profil de vente branché sur un numéro de démonstration.

#### 2.4.6 Boutons d'action

- **« Enregistrer »** : sauvegarde tous les réglages ci-dessus. Un champ vidé est bel et bien effacé (mis à vide), pas conservé silencieusement.
- **« Synchroniser l'agent »** : pousse la configuration **et** les questions actives **et** la base de connaissances vers votre assistant Vapi. Le bouton est **grisé** si la case « Agent géré directement dans Vapi » est cochée. Un texte d'aide rappelle l'effet ou l'état verrouillé de la synchronisation.

#### 2.4.7 Section « Questions de qualification »

Sous les réglages, une carte liste les questions que l'agent pose pour qualifier un appel. **L'ordre définit le déroulé** de l'entretien.

- **Bouton « Ajouter une question »** (en haut à droite) : ouvre la fenêtre d'ajout (voir §2.4.8).
- Chaque question affiche :
  - des **flèches monter / descendre** (réordonnancement, enregistré immédiatement) ;
  - le **texte français** de la question ;
  - un **badge « Champ associé »** (ex. « Type de projet », « Région »…) ;
  - un badge **« Inactif »** si la question est désactivée ;
  - une case **« Obligatoire »** (bascule enregistrée immédiatement) ;
  - les boutons **Modifier** (crayon) et **Supprimer** (corbeille, avec confirmation).
- **État vide** : « Aucune question de qualification. Ajoutez-en une pour démarrer. » — mais en pratique, **13 questions par défaut** sont créées automatiquement à la première ouverture (voir §4.7).

#### 2.4.8 Fenêtre d'ajout / modification d'une question

- **Question (français)** : le texte prononcé (obligatoire).
- **Question (anglais)** : la variante anglaise (facultative).
- **Champ associé** : menu déroulant parmi **14 clés** prédéfinies (voir §4.7). C'est la donnée que la question cherche à recueillir ; l'agent l'associe au bon champ de qualification.
- **Case « Obligatoire »**.
- Boutons **« Enregistrer »** / **« Annuler »**.

### 2.5 Onglet « Base de connaissances »

Des fiches d'information que l'agent utilise pour répondre aux questions des appelants (heures d'ouverture, zones desservies, spécialités, politique de garantie, etc.).

**Colonnes du tableau :** Titre, Contenu (aperçu sur deux lignes), Langue, Statut (« Actif » / « Inactif »), et les actions Modifier / Supprimer.

- **Bouton « Ajouter un élément »** : ouvre la fenêtre d'ajout.
- **Fenêtre d'ajout / modification** : **Titre** (obligatoire), **Contenu** (obligatoire), **Langue** (« Français » / « Anglais »).
- **Suppression** avec confirmation.
- **État vide** : « Aucun élément de connaissance. Ajoutez-en un pour enrichir l'agent. »

*Seules les fiches marquées « Actif » sont injectées dans le script de l'agent lors d'un appel.*

### 2.6 Onglet « Statistiques »

Trois cartes de suivi de l'usage (lecture seule ; les montants sont en dollars US, voir §4.5).

**Mois courant** (avec la période affichée) :

- **Appels** (nombre) ;
- **Minutes** (total) ;
- **Coût estimé** (coût fournisseur Vapi) ;
- **Montant facturable** (coût × marge ; l'indice « Coût × 1,3 » est indiqué).

**Totaux** (tout l'historique) :

- **Total des appels**, **Taux de qualification** (avec le nombre de qualifiés), **Taux de transfert**, **Total des minutes**, **Coût total estimé**, **Total facturable**, **Langue (FR / EN)**, **Messages seulement**.

**Évolution mensuelle** : un tableau sur 6 mois (Mois, Appels, Minutes, Coût estimé).

Une note de bas de carte rappelle : « Montants en USD (coût fournisseur Vapi). La facturation automatique des crédits est désactivée. »

### 2.7 Vue super-administrateur (hors page tenant)

Cette vue **n'est pas** dans la page `/agent-vocal`. Elle vit dans le module **Super-Admin** (`/admin`), onglet **« Appels vocaux »** (composant `AdminVoiceCallsTab`), et n'est visible que par le super-administrateur de la plateforme. Elle permet de consulter, de façon **centralisée et inter-entreprises**, tous les appels de tous les tenants : filtres (entreprise, résultat, langue, dates, recherche), agrégats (total, qualifiés, transférés, coût total), détail complet d'un appel (transcription, enregistrement si consenti) et **journal d'audit** des consultations (traçabilité Loi 25). Voir §4.3.

---

## 3. Workflows pas à pas

### 3.1 Mettre l'agent en service pour la première fois

Prérequis : votre assistant et votre numéro doivent exister chez **Vapi** (opération réalisée par l'équipe Constructo AI lors de la mise en place ; les clés `VAPI_API_KEY` et `VAPI_WEBHOOK_SECRET` sont posées côté serveur).

1. Ouvrez **Agent Vocal** → onglet **Configuration**.
2. Section **Téléphonie** : saisissez l'**ID de l'assistant Vapi** et le **Numéro de téléphone** branché. Choisissez la **Langue par défaut** et cochez **Agent bilingue** si vous voulez l'accueil dans les deux langues.
3. Section **Voix et accueil** : laissez l'**Identifiant de voix** vide pour la voix par défaut, ou saisissez une voix précise. Rédigez vos **messages d'accueil** (français et anglais) si vous ne voulez pas le message par défaut.
4. Section **Transfert d'appel** : indiquez le **Numéro de transfert** vers lequel router les appels qui demandent une personne (facultatif).
5. Section **Notification post-appel** : cochez la case et saisissez un **courriel** si vous voulez recevoir un résumé après chaque appel pertinent.
6. Cliquez **« Enregistrer »**.
7. Cliquez **« Synchroniser l'agent »** : l'ERP construit le profil (script de qualification à partir de vos questions + base de connaissances + outils de statut + accueil) et le pousse vers Vapi. *La voix réglée chez Vapi n'est pas écrasée par la synchronisation.*
8. Remontez au **bandeau d'état** et cliquez **« Activer »**.

*Astuce :* refaites **« Synchroniser l'agent »** chaque fois que vous modifiez vos questions, votre base de connaissances, l'accueil ou le numéro de transfert, pour que Vapi reflète vos changements.

### 3.2 Personnaliser les questions de qualification

1. Onglet **Configuration** → section **Questions de qualification**.
2. Pour **ajouter** : bouton **« Ajouter une question »**, saisissez le texte (français, et anglais si souhaité), choisissez le **Champ associé** parmi la liste, cochez **Obligatoire** au besoin, **Enregistrer**.
3. Pour **réordonner** : utilisez les flèches **monter / descendre** ; l'ordre est enregistré aussitôt. C'est l'ordre dans lequel Clara posera les questions.
4. Pour **rendre une question obligatoire** : cochez sa case **« Obligatoire »** directement dans la liste.
5. Pour **modifier** ou **supprimer** : boutons crayon / corbeille.
6. Cliquez **« Synchroniser l'agent »** (section Configuration) pour appliquer vos changements chez Vapi.

*Bon à savoir :* les **13 questions par défaut** couvrent déjà un entretien de qualification complet en construction (nature du projet, travaux, type de bâtiment, région, échéancier, motivation, décideur, disponibilité, plans/budget, financement, soumissions concurrentes, coordonnées de rappel, provenance). Adaptez-les plutôt que de repartir de zéro.

### 3.3 Alimenter la base de connaissances

1. Onglet **Base de connaissances** → **« Ajouter un élément »**.
2. Donnez un **Titre** clair (ex. « Zones desservies »), un **Contenu** concis (ex. « Grand Montréal, Laval, Rive-Nord et Rive-Sud »), et la **Langue**.
3. **Enregistrer**. Répétez pour chaque sujet fréquent (heures d'ouverture, délais, garanties, types de travaux, etc.).
4. Cliquez **« Synchroniser l'agent »** pour que Clara puisse s'en servir.

*Seules les fiches actives sont utilisées.* Gardez les contenus courts et factuels : ils sont injectés dans le script de l'agent.

### 3.4 Ce qui se passe quand un client appelle (nouveau prospect)

Ce déroulé est automatique ; vous n'avez rien à faire pendant l'appel.

1. Vapi reçoit l'appel sur votre numéro et interroge l'ERP pour obtenir le profil de l'agent. L'ERP identifie **votre entreprise par le numéro appelé**, construit Clara (accueil, script, questions actives, base de connaissances, outils) et la renvoie en moins de 7 secondes.
2. Clara accueille l'appelant, se présente comme assistante virtuelle IA (Loi 25), demande le nom, puis pose vos questions **une à la fois**.
3. En fin d'appel, Vapi transmet le rapport à l'ERP. Si le seuil est atteint (**un nom + au moins une information utile**), l'ERP crée :
   - une **opportunité** au module Ventes (numéro `OPP-NNNNN`, étape « Qualification », provenance « agent_vocal », **montant non estimé**) ;
   - une **interaction** de type « Appel » liée à l'opportunité ;
   - une **pré-qualification B.A.T.** préliminaire (Budget / Autorité / Temporalité, statut « En cours ») à réviser et compléter dans la grille CRM.
4. L'appel apparaît dans le **journal** avec le résultat **« Qualifié »** et le lien vers l'opportunité.
5. Si vous avez activé les notifications, un **résumé par courriel** est envoyé à l'adresse configurée.

*Cas particuliers :* si l'appel n'atteint pas le seuil, il est tout de même consigné (résultat « Message »). Si l'appelant demande une personne et qu'un numéro de transfert est configuré, l'appel est transféré (résultat « Transféré »).

### 3.5 Ce qui se passe quand un client demande le statut de son dossier (Mode B)

1. Clara demande le **numéro du document** (soumission, facture, projet ou bon de travail) et le fait **répéter ou épeler** pour confirmer.
2. Elle demande le **nom complet** de la personne **ou** le nom de son entreprise.
3. L'ERP vérifie **côté serveur** que le numéro désigne bien un document de votre entreprise **et** que le nom concorde avec le client au dossier.
4. Si tout concorde, Clara annonce **le statut** (ex. « votre soumission DEV-2026-043 a été envoyée et est en attente de réponse »), puis rappelle qu'elle **ne peut pas** communiquer de montant par téléphone et qu'un représentant peut fournir les détails.
5. Si le dossier est introuvable **ou** que le nom ne concorde pas, Clara donne **exactement la même réponse neutre** (« Je n'ai pas pu confirmer ce dossier… ») et propose un rappel — l'agent ne confirme jamais l'existence d'un dossier à une personne non vérifiée.

*Protections automatiques :* après **3 échecs** sur un même document, **5 échecs** d'un même numéro appelant, ou **30 échecs** au total sur une heure, la vérification est temporairement verrouillée (anti-devinette). Un nom trop générique seul (« Construction », « Inc », le nom d'une ville) ne suffit **jamais** à vérifier une identité ; l'appelant doit prononcer tous les mots distinctifs du nom au dossier.

### 3.6 Consulter un appel et écouter l'enregistrement

1. Onglet **Journal des appels** : cliquez sur la ligne voulue.
2. Lisez les métadonnées, le **résumé** et la **transcription**.
3. Si un **enregistrement** est présent (consentement donné), utilisez le lecteur audio pour l'écouter.
4. Si une **opportunité** a été créée, cliquez le lien pour l'ouvrir dans **Ventes**.

*L'enregistrement audio n'est conservé que si l'appelant a consenti.* Sans consentement, aucun fichier n'est stocké.

### 3.7 Activer les avis par courriel

1. Onglet **Configuration** → section **Notification post-appel**.
2. Cochez **« M'avertir par courriel après chaque appel pertinent »**.
3. Saisissez le **courriel de notification** (ex. `ventes@entreprise.com`).
4. **Enregistrer**.

Vous recevrez, après chaque appel porteur de contenu, un courriel résumant le résultat, l'appelant, la durée et le résumé — avec, s'il y a lieu, le numéro de l'opportunité créée.

### 3.8 Lire les statistiques d'usage

1. Onglet **Statistiques**.
2. Consultez le **mois courant** (appels, minutes, coût, montant facturable), les **totaux** (dont taux de qualification et de transfert, répartition FR/EN) et l'**évolution mensuelle**.
3. Rappelez-vous que les montants sont **en dollars US** et **indicatifs** : la facturation automatique des crédits est désactivée dans ce module (voir §4.5).

### 3.9 (Super-administrateur) Consulter les appels de toutes les entreprises

1. Ouvrez le module **Super-Admin** (`/admin`) → onglet **« Appels vocaux »**.
2. Filtrez par entreprise, résultat, langue, dates ou mot-clé.
3. Ouvrez un appel pour voir le détail complet ; le cas échéant, écoutez l'enregistrement (l'accès est **journalisé**).
4. Consultez le **journal d'audit** des consultations (traçabilité Loi 25).
5. En cas de besoin technique, la **réindexation** reconstruit la vue centralisée à partir des données de chaque entreprise.

---

## 4. Référence

### 4.1 Points d'accès — webhook public (Vapi → ERP)

Ce point d'accès est **public** (hors préfixe ERP) car c'est Vapi qui l'appelle. Sa sécurité repose entièrement sur la **signature** vérifiée avant tout traitement.

| Méthode | Chemin | Description |
|---------|--------|-------------|
| POST | `/api/voice/webhook` | Point d'entrée unique de Vapi. Vérifie la signature (en-tête `x-vapi-secret`) **avant** toute analyse → **401** si invalide ; refuse un corps de plus de **5 Mo** → **413** ; **400** si le JSON est invalide. Route ensuite selon `message.type` (voir §4.4). |

### 4.2 Points d'accès — gestion (tenant, authentification requise)

Tous montés sous `/api/erp/v1/voice/*`, protégés par l'authentification ERP (mais **sans** garde administrateur, voir §1.3).

| Méthode | Chemin | Description |
|---------|--------|-------------|
| GET | `/voice/config` | État + configuration de l'agent (une ligne, créée à la demande) |
| PATCH | `/voice/config` | Mise à jour de la configuration ; gère le routage numéro→entreprise et assistant→entreprise (voir §4.8) |
| GET | `/voice/calls` | Journal des appels (sans transcription ni montant), paginé (défaut 50, max 200) |
| GET | `/voice/calls/{call_id}` | Détail d'un appel (transcription, résumé, enregistrement si consenti) |
| GET | `/voice/questions` | Questions de qualification (crée les 13 questions par défaut si vide) |
| POST | `/voice/questions` | Créer une question (champ associé validé contre la liste d'autorisation) |
| PUT | `/voice/questions/reorder` | Réordonner un lot de questions |
| PUT | `/voice/questions/{id}` | Modifier une question (champs partiels) |
| DELETE | `/voice/questions/{id}` | Supprimer une question |
| GET | `/voice/knowledge` | Lister la base de connaissances |
| POST | `/voice/knowledge` | Créer une fiche |
| PUT | `/voice/knowledge/{id}` | Modifier une fiche |
| DELETE | `/voice/knowledge/{id}` | Supprimer une fiche |
| POST | `/voice/config/sync` | Pousser la configuration vers Vapi ; **409** si « géré directement dans Vapi », **400** sans assistant, **502** si Vapi refuse |
| GET | `/voice/stats` | Statistiques d'usage (lecture seule ; défaut 6 mois, max 24) |

### 4.3 Points d'accès — super-administrateur (inter-entreprises)

Tous montés sous `/api/erp/v1/admin/voice/*`, protégés par `require_super_admin` (garde forte sur le type de compte, **infalsifiable** ; un administrateur d'entreprise n'y accède jamais).

| Méthode | Chemin | Description |
|---------|--------|-------------|
| GET | `/admin/voice/calls` | Liste paginée inter-entreprises (filtres entreprise / résultat / langue / dates / recherche + agrégats). Chaque consultation est journalisée. |
| GET | `/admin/voice/calls/{schema}/{call_id}` | Détail complet d'un appel d'une entreprise (transcription, enregistrement si consenti). Journalisé. |
| POST | `/admin/voice/calls/{schema}/{call_id}/listen` | Journalise l'**écoute** d'un enregistrement (ne renvoie ni URL ni fichier). **404** sans enregistrement. |
| GET | `/admin/voice/access-log` | Journal d'audit des consultations (transparence Loi 25) |
| POST | `/admin/voice/reindex` | Reconstruit la vue centralisée à partir de toutes les entreprises (maintenance) |

### 4.4 Types de messages Vapi et traitement

| `message.type` | Traitement |
|----------------|-----------|
| `assistant-request` | Construit l'assistant « Clara » selon votre entreprise (accueil, script, questions, base de connaissances, outils) et le renvoie (< 7 s) |
| `tool-calls` | **Mode B** : consultation de statut (liste d'autorisation de 4 outils, vérification d'identité côté serveur) |
| `end-of-call-report` | Consigne l'appel, crée l'opportunité + interaction + pré-qualification B.A.T. si le seuil est atteint, envoie le résumé courriel |
| `status-update` et autres | Ignorés (réponse neutre) |

### 4.5 Résultats d'appel et calcul du coût

**Résultats possibles** (colonne « Résultat » du journal) :

| Valeur | Libellé | Badge | Signification |
|--------|---------|-------|---------------|
| `qualified` | Qualifié | Vert | Opportunité créée (seuil atteint) |
| `transferred` | Transféré | Bleu | Appel transféré à un humain |
| `message_only` | Message | Ambre | Coordonnées / message pris, sans opportunité |
| `dropped` | Abandonné | Gris | Appel abandonné |

**Coût et facturation :**

- Le **coût réel** de l'appel (orchestration + reconnaissance vocale + modèle Claude + synthèse vocale + téléphonie) est facturé **par Vapi**, en **dollars US**. L'ERP le récupère et ne fait que le **suivre**.
- Le **montant facturable** affiché = coût × **1,30** (marge d'affichage `_VOICE_BILLING_MARGIN`).
- **Aucun débit automatique** de crédits ni de facturation Stripe n'est effectué par ce module : la facturation automatique est **explicitement différée** (décision devise USD/CAD à trancher). Les chiffres de l'onglet Statistiques sont **indicatifs**.

### 4.6 Message d'accueil par défaut (Loi 25)

Si vous ne saisissez pas de message d'accueil, Clara utilise ce message par défaut, qui divulgue sa nature d'IA et l'enregistrement possible :

> « Bonjour, ici Clara, l'assistante virtuelle de [votre entreprise]. Je suis une voix générée par intelligence artificielle et cet appel peut être enregistré. Comment puis-je vous aider ? »

Une variante anglaise équivalente est utilisée pour un appel anglophone. Le message que **vous** configurez a toujours priorité sur ce message par défaut.

### 4.7 Questions par défaut et champs associés

**Les 13 questions créées automatiquement** (dans l'ordre) portent sur : (1) nature du projet, (2) type de travaux, (3) type de bâtiment, (4) **région** *(obligatoire)*, (5) échéancier, (6) motivation / urgence, (7) décideur, (8) disponibilité pour une rencontre, (9) plans / budget, (10) financement, (11) soumissions concurrentes, (12) **meilleur numéro et moment pour rappeler** *(obligatoire)*, (13) provenance.

**Les 14 champs associés** disponibles (liste d'autorisation stricte ; toute autre clé est refusée) :

| Clé | Libellé |
|-----|---------|
| `project_type` | Type de projet |
| `work_type` | Type de travaux |
| `building_type` | Type de bâtiment |
| `region` | Région |
| `timeline` | Échéancier |
| `decision_maker` | Décideur |
| `has_plans_budget` | Plans / budget |
| `competing_bids` | Soumissions concurrentes |
| `contact_info` | Coordonnées |
| `source` | Provenance |
| `contact_phone` | Téléphone de contact |
| `urgency` | Urgence |
| `visit_availability` | Disponibilité pour visite |
| `financing_status` | Statut de financement |

### 4.8 Résolution de l'entreprise et anti-détournement

- À chaque appel, l'entreprise est résolue **par le numéro appelé** (table partagée `voice_phone_routing`), puis, à défaut, **par l'identifiant de l'assistant** (table `voice_assistant_routing`, utile aux appels web/test sans numéro). Ces deux clés proviennent du message **signé** de Vapi, jamais d'un paramètre fourni par l'appelant.
- Lorsque vous enregistrez un **numéro** ou un **ID d'assistant** dans la configuration, l'ERP ne l'accepte que s'il est **libre** ou **déjà détenu par votre entreprise** ; sinon la sauvegarde échoue avec un **409** (« déjà associé à un autre compte »). Cela empêche de détourner les appels entrants d'un concurrent. Vider le champ retire le routage correspondant.

### 4.9 Limites, seuils et valeurs par défaut

| Élément | Valeur |
|---------|--------|
| Taille maximale du corps webhook | **5 Mo** (au-delà : 413) |
| Durée maximale d'un appel | **3200 s** (~53 min) poussée à Vapi (`maxDurationSeconds`) |
| Pagination du journal (tenant) | 50 par page (max 200) |
| Fenêtre de verrouillage Mode B | **1 heure** glissante |
| Verrouillage — par document | **3** échecs |
| Verrouillage — par numéro appelant | **5** échecs |
| Verrouillage — plafond entreprise | **30** échecs |
| Modèle IA par défaut (chez Vapi) | fournisseur **Anthropic**, modèle **`claude-sonnet-4-6`** |
| Voix par défaut | Azure Neural **`fr-CA-SylvieNeural`** |
| Reconnaissance vocale | Deepgram **nova-2** |
| Langue par défaut | **`fr-CA`** |
| Marge d'affichage du coût | **1,30** |
| Devise du suivi de coût | **USD** |

### 4.10 Modèle de données

**Tables de votre entreprise (créées à la demande) :**

- `voice_agent_config` — configuration de l'agent (fournisseur, ID assistant, numéro, actif, langue, bilingue, voix, accueils FR/EN, règles de transfert, courriel de notification, verrou « géré dans Vapi »…).
- `voice_calls` — un enregistrement par appel (identifiant fournisseur unique, numéro appelant, horodatages, durée, langue, transcription, résumé, URL d'enregistrement, résultat, consentement, opportunité et interaction liées, coût estimé).
- `voice_qualification_questions` — vos questions (ordre, textes FR/EN, champ associé, obligatoire, actif).
- `voice_knowledge_base` — vos fiches (titre, contenu, langue, actif).
- `voice_lookup_attempts` — journal des tentatives de vérification d'identité (Mode B, pour le verrouillage anti-abus).

**Tables partagées (`public`) :**

- `voice_phone_routing` — correspondance numéro → entreprise.
- `voice_assistant_routing` — correspondance assistant Vapi → entreprise.
- `voice_calls_index` — **miroir de métadonnées** inter-entreprises (sans transcription ni audio), pour la vue super-admin à grande échelle.
- `voice_admin_access_log` — journal d'audit des consultations super-admin (Loi 25).

**Tables d'autres modules alimentées :** `opportunities`, `interactions`, `prospect_qualifications` (module Ventes / CRM). **Tables lues en Mode B :** `devis`, `factures`, `projects`, `formulaires`, avec `companies` et `contacts`.

### 4.11 Protection de la vie privée (Loi 25)

- **Transparence** : Clara se présente comme une IA et signale l'enregistrement possible dès l'accueil.
- **Consentement** : l'**enregistrement audio n'est conservé que si l'appelant y consent** ; sinon aucun fichier n'est stocké et l'URL est masquée même au super-administrateur.
- **Minimisation** : le miroir inter-entreprises ne contient ni transcription ni audio.
- **Traçabilité** : chaque consultation super-admin (liste, détail, écoute, réindexation) est **journalisée dans la même opération** que la lecture ; sans trace, pas de divulgation.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

- **Ventes / CRM** — chaque appel qualifié crée une **opportunité** (étape « Qualification », provenance « agent_vocal »), une **interaction** de type « Appel » et une **pré-qualification B.A.T.** préliminaire. Le lien « Opp. #N » du journal ouvre le module **Ventes**.
- **Soumissions, Facturation, Projets, Bons de travail** — le **Mode B** lit le statut de ces documents (jamais un montant) après vérification d'identité.
- **Courriels** — le **résumé post-appel** est envoyé via le relais SMTP interne de l'ERP (le même que le module Courriels).
- **Super-Admin** — l'onglet **« Appels vocaux »** offre la vue centralisée inter-entreprises et le journal d'audit.
- **Configuration / Abonnement** — en **mode consultation** (abonnement suspendu), les écritures du module sont bloquées.

### 5.2 Foire aux questions

**L'agent peut-il appeler mes clients (appels sortants) ?**
Non. L'agent répond uniquement aux appels **entrants**. Il ne compose jamais de numéros.

**Peut-il donner un prix, un solde ou un total au téléphone ?**
Non, jamais. C'est une règle de sécurité imposée côté serveur : même en Mode B, seul le **statut** est annoncé, jamais un montant. L'agent invite l'appelant à demander un rappel pour les détails financiers.

**Qui peut modifier la configuration de l'agent ?**
Actuellement, **tout utilisateur connecté de votre entreprise** (il n'y a pas de garde administrateur sur ce module — voir §1.3). Tenez-en compte dans l'attribution des comptes.

**La base de connaissances est-elle stockée dans Google ou un service externe ?**
Non. Les fiches vivent dans la table `voice_knowledge_base` **de votre entreprise**, dans la base de données de l'ERP. Le module n'a **aucune** intégration Google. *(Si vous avez déjà entendu parler de « bases de connaissances Google » pour un agent vocal, cela concerne un agent de vente de démonstration configuré directement dans Vapi, hors de ce module.)*

**« Clara », est-ce un agent unique partagé entre toutes les entreprises ?**
Non. **Clara est le nom par défaut**, personnalisé au nom de **chaque entreprise** (l'accueil et le script disent « l'assistante virtuelle de [votre entreprise] »). Chaque entreprise a sa propre configuration et son propre assistant Vapi. Il n'existe pas d'objet « clone » distinct : une entreprise = une ligne de configuration + son assistant routé.

**Le numéro de démonstration (par exemple +1 936 587-1141) est-il codé en dur ?**
Non. Le routage numéro → entreprise est **piloté par les données** (table `voice_phone_routing`). Un « numéro de démonstration » est simplement une ligne de cette table, pas une constante de l'application.

**Que se passe-t-il si Vapi rejoue deux fois le même rapport de fin d'appel ?**
Rien de fâcheux : le traitement est **idempotent** (unicité sur l'identifiant d'appel du fournisseur). L'appel n'est pas dupliqué et une opportunité déjà créée n'est jamais recréée ni rétrogradée.

**Pourquoi l'onglet Statistiques affiche des montants en USD ?**
Parce que Vapi facture en dollars US. Les montants sont **indicatifs** ; la facturation automatique des crédits est désactivée dans ce module.

**J'ai coché « Agent géré directement dans Vapi » : pourquoi « Synchroniser » est-il grisé ?**
C'est voulu. Ce verrou empêche l'ERP d'écraser un profil que vous entretenez directement dans le tableau de bord Vapi. Décochez-le pour reprendre la synchronisation depuis l'ERP.

**Certains réglages sont-ils affichés mais sans effet ?**
Oui, quelques champs sont enregistrés mais **pas encore exploités** dans la construction de l'agent : les **heures d'ouverture** (`business_hours`) et les champs **fournisseur / modèle IA** (`llm_provider` / `llm_model`). Le modèle appliqué reste toujours le **`claude-sonnet-4-6` d'Anthropic** par défaut. N'en attendez donc pas d'effet fonctionnel pour l'instant.

**Un appelant peut-il consulter le dossier de quelqu'un d'autre en Mode B ?**
Non. Il doit fournir le **numéro exact** du document **et** un nom qui concorde avec le client au dossier (tous les mots distinctifs). « Introuvable » et « nom non concordant » donnent la **même** réponse neutre, et des verrous anti-devinette se déclenchent après quelques échecs.

### 5.3 Dépannage courant

| Symptôme | Piste |
|----------|-------|
| L'agent ne répond pas aux appels | Vérifiez que le statut est **Actif**, que le **numéro** est bien saisi et branché chez Vapi, et refaites **« Synchroniser l'agent »**. |
| Un appel n'apparaît pas dans le journal | Le rapport de fin d'appel n'a peut-être pas encore été reçu, ou le numéro appelé n'est routé vers aucune entreprise. |
| « Synchroniser » renvoie une erreur | **409** = case « géré dans Vapi » cochée ; **400** = aucun ID d'assistant ; **502** = Vapi a refusé (réessayez, vérifiez l'ID de l'assistant). |
| Enregistrement audio absent | L'appelant n'a pas consenti : dans ce cas, aucun audio n'est conservé (Loi 25). |
| Impossible d'enregistrer un numéro | **409** « déjà associé à un autre compte » : le numéro ou l'ID d'assistant est détenu par une autre entreprise. |
| Une nouvelle question n'est pas posée | Assurez-vous qu'elle est **active**, puis **synchronisez l'agent**. |

---

## 6. Récapitulatif

- **Standardiste virtuelle IA « Clara »** pour les appels **entrants** : accueil au nom de votre entreprise, qualification de prospects, statut de dossiers, transfert vers un humain — en français et en anglais.
- **Accès** : menu **Communication → Agent Vocal** (`/agent-vocal`). Quatre onglets : **Journal des appels**, **Configuration** (+ questions), **Base de connaissances**, **Statistiques**, surmontés d'un **bandeau d'état** avec bascule Activer / Désactiver.
- **Mode A (qualification)** : à la fin d'un appel pertinent, une **opportunité** (Ventes, étape Qualification), une **interaction** et une **pré-qualification B.A.T.** sont créées automatiquement ; montant jamais deviné.
- **Mode B (statut de dossier)** : soumission / facture / projet / bon de travail, avec **vérification d'identité 100 % côté serveur**, anti-énumération et verrouillage anti-abus ; **jamais de montant** communiqué.
- **Sécurité** : signature Vapi vérifiée **avant** tout traitement, entreprise résolue par le **numéro appelé**, protection anti-détournement du routage, injection de script neutralisée.
- **Loi 25** : transparence IA à l'accueil, **audio conservé seulement avec consentement**, miroir inter-entreprises sans transcription ni audio, consultations super-admin **journalisées**.
- **Argent** : le coût Vapi (USD) est **suivi**, pas prélevé ; **aucun débit de crédits ni Stripe** dans ce module ; montant facturable affiché = coût × 1,30 (indicatif).
- **Points d'attention** : la gestion n'est **pas** réservée aux administrateurs (tout utilisateur connecté peut modifier) ; les **heures d'ouverture** et les champs **fournisseur/modèle IA** sont stockés mais **non exploités** (le modèle reste `claude-sonnet-4-6`).
- **Vue super-admin** : onglet « Appels vocaux » du module Super-Admin, inter-entreprises, avec journal d'audit et réindexation.

---

*Fichiers sources vérifiés :* `backend/routers/voice.py` (2941 lignes), `backend/routers/voice_admin.py` (690 lignes), `backend/voice_provider.py` (165 lignes), `frontend/src/pages/AgentVocalPage.tsx` (1559 lignes), `frontend/src/api/voice.ts` (282 lignes), `frontend/src/components/admin/AdminVoiceCallsTab.tsx`, `frontend/src/components/layout/Sidebar.tsx`, `frontend/src/i18n/locales/fr/voice.json`.

*Manuels liés :* `06-gestion-crm-opportunites.md` (opportunités et B.A.T.), `08-ventes-soumissions.md` et `09-ventes-projets.md` (documents lus en Mode B), `12-operations-bons-de-travail.md`, `15-operations-comptabilite.md` (factures), `23-communication-emails.md` (relais courriel), `24-communication-messagerie.md`, `28-configuration.md` (abonnement et mode consultation).
