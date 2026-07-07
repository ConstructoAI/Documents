# Module 25 — Assistant IA (Claude, profils experts)

> **Version** : 3.0 (refonte vérifiée ligne par ligne par rapport au code source, juillet 2026)
> **Accès** : **barre supérieure** de l'ERP → bouton-icône « étincelle » (`Sparkles`) intitulé **« Assistant IA »**. **Ce module n'est PAS dans la barre latérale de gauche** (voir §1.2). Adresse directe : `/assistant-ia`.
> **Code de référence (backend)** : `backend/routers/ai.py` (3541 lignes, 12 points d'accès ; moteur IA central de l'ERP) ; `backend/routers/ai_profiles.py` (804 lignes, 8 points d'accès ; sert **Estimation IA** et **Métré PDF**, pas ce module — voir §1.7) ; `backend/routers/data_import.py` (684 lignes ; import de clients)
> **Code de référence (frontend)** : `frontend/src/pages/AssistantIAPage.tsx` (716 lignes) ; `frontend/src/components/ai/ClientImportModal.tsx` (243 lignes) ; `frontend/src/api/ai.ts` (175 lignes) + `frontend/src/api/dataImport.ts` (76 lignes)
> **Libellés** : `i18n/locales/fr/ai.json` (126 lignes) + `i18n/locales/en/ai.json`
> **Tables PostgreSQL partagées (`public`)** : `ai_prepaid_credits`, `ai_usage_tracking`, `ai_credit_applied_invoices`, `ai_credit_ledger`
> **Tables PostgreSQL par entreprise (tenant)** : `conversations`, `conversation_messages`, `ai_profiles`, `ai_profile_documents`, `companies` (cible de l'import)
> **Cadrage** : assistant conversationnel propulsé par **Claude (Anthropic)**, **branché sur les données de votre entreprise**. Il **lit** votre base (projets, factures, employés, devis, stock…) et peut aussi **y écrire** (créer, modifier, supprimer des enregistrements) au moyen de deux outils SQL. Il **analyse un document**, **analyse des plans** (lecture d'image / vision) et pilote un **assistant d'import de clients** depuis un fichier CSV ou Excel. **Ce module n'est pas conçu pour produire des estimations de coût** : pour cela, utilisez le module **Soumissions → Estimation IA** (voir §1.5 et §5.1).

*Note de terminologie employée dans ce manuel :* « point d'accès » désigne un point de terminaison de l'interface de programmation (endpoint) ; « tenant » désigne votre entreprise (chaque entreprise a ses propres données isolées) ; « jeton » (token) est l'unité de facturation de l'IA (un mot vaut à peu près 1,3 jeton) ; « crédits prépayés » désigne le solde en dollars que votre entreprise consomme à chaque appel à l'IA.

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

L'Assistant IA est un **agent de dialogue** intégré à l'ERP. Vous lui écrivez une question ou une consigne en français (ou en anglais), et il répond en s'appuyant sur **les données réelles de votre entreprise**. Concrètement, il peut :

- **Répondre à des questions sur vos données** : « Combien de projets sont en cours ? », « Quelles factures dépassent 5 000 $ et sont impayées ? », « Quel employé a le plus d'heures ce mois-ci ? ». Il consulte la base au moyen de l'outil **`recherche_bd`** (lecture seule) et formule une réponse en langage clair.
- **Créer ou modifier des enregistrements** sur demande : « Crée une note de projet… », « Marque ce bon de travail comme terminé… ». Il utilise l'outil **`executer_action`** (écriture directe). *C'est le seul assistant de l'ERP avec un accès en écriture à votre base — lisez attentivement le point d'attention du §1.3 et le §3.2.*
- **Donner un avis d'expert construction** : conseils techniques, rappels de normes, pistes de résolution de problème de chantier. L'assistant adapte automatiquement son ton et son expertise selon le sujet détecté dans votre question (électricité, plomberie, structure, toiture, isolation, sécurité, aspects juridiques, comptabilité…), sans que vous ayez à choisir quoi que ce soit (voir §4.3).
- **Analyser un document** que vous lui fournissez : contrat, devis reçu, fiche technique, chiffrier. Un seul fichier à la fois (voir §2.6).
- **Analyser des plans** (jusqu'à dix fichiers image ou PDF) par lecture d'image : identification des éléments, estimation des surfaces, corps de métier nécessaires (voir §2.7).
- **Importer une liste de clients** depuis un fichier CSV ou Excel exporté d'un autre logiciel, avec correspondance des colonnes proposée par l'IA (voir §2.8 et §3.6).
- **Suivre votre consommation** : solde de crédits, coûts des 30 derniers jours, usage par fonctionnalité (voir §2.9).

L'assistant conserve **l'historique de vos conversations** : vous pouvez les rouvrir, les poursuivre ou les supprimer.

### 1.2 Comment y accéder

- **Depuis la barre supérieure de l'ERP** (celle qui reste affichée en haut, quel que soit le module ouvert) : cliquez sur le bouton **« Assistant IA »**, reconnaissable à son icône d'étincelle violette (`Sparkles`). Ce bouton est défini dans `TopBar.tsx` (lignes 312-320) et vous amène à la page `/assistant-ia`.
- **Par l'adresse directe** : `/assistant-ia`.

> **Point important — ce module n'est PAS dans la barre latérale de gauche.** Contrairement à la plupart des modules (Ventes, Comptabilité, Magasin…), l'Assistant IA ne figure pas dans le menu latéral. On l'ouvre uniquement par le bouton « étincelle » de la barre supérieure (ou par l'adresse). Si vous le cherchez dans la barre de gauche, vous ne l'y trouverez pas : c'est normal.

La page est **protégée** : il faut être connecté à l'ERP.

### 1.3 Rôles et permissions

- **Ouvrir la page et dialoguer** : accessible à **tout utilisateur connecté** de l'entreprise. Il n'y a **aucun contrôle de rôle** (ni administrateur, ni comptable, ni super-administrateur) sur les points d'accès de ce module ; tous s'appuient uniquement sur l'authentification.
- **Point d'attention (sécurité).** Puisqu'il n'y a pas de garde de rôle, **n'importe quel employé connecté** peut demander à l'assistant de **modifier ou supprimer des données** (outil `executer_action`), sans passer par les écrans habituels ni par une validation d'administrateur. Tenez-en compte dans l'attribution des comptes. Les seuls garde-fous techniques sont : le refus des mots-clés destructeurs, l'obligation d'une clause `WHERE` sur toute modification ou suppression (impossible de « tout effacer » d'un coup), un délai maximal de 10 secondes par opération, et un journal d'audit côté serveur (voir §4.3).
- **Crédits.** Le vrai « verrou » du module est **financier**, pas basé sur les rôles : si le solde de crédits prépayés de l'entreprise est épuisé, l'assistant cesse de répondre jusqu'à la recharge (voir §1.6 et §4.5). Le super-administrateur de la plateforme et quelques comptes internes sont exemptés de facturation (usage illimité).
- **Mode consultation (lecture seule).** Lorsque l'abonnement de l'entreprise est suspendu, l'ERP passe en mode consultation et bloque les écritures au niveau global. La lecture reste possible.

### 1.4 Les surfaces de la page

La page se présente en **deux colonnes** : à gauche, la **zone de dialogue** (la plus large) ; à droite, une **barre latérale de statistiques** qui s'affiche à la demande. S'y ajoutent des panneaux repliables et une fenêtre modale :

| Surface | Où | Rôle |
|---|---|---|
| **Zone de dialogue** | Colonne de gauche | Fil de la conversation, saisie des messages |
| **Panneau Conversations** | Se déplie sous l'en-tête | Liste des conversations enregistrées (ouvrir, nouvelle, supprimer) |
| **Panneau « Analyser un document »** | Se déplie au-dessus de la saisie | Téléverser un fichier + instructions facultatives |
| **Panneau « Analyser un plan »** | Se déplie au-dessus de la saisie | Téléverser jusqu'à 10 plans |
| **Modale « Importer des clients »** | Fenêtre superposée | Assistant d'import CSV / Excel en trois étapes |
| **Barre latérale « Stats »** | Colonne de droite (sur demande) | Crédits, coûts quotidiens, usage sur 30 jours |

### 1.5 Ce que le module fait — et ne fait PAS

Le module **fait** : dialoguer en langage naturel, consulter vos données (lecture), écrire dans votre base (création / modification / suppression encadrées), donner un avis d'expert construction, analyser un document, analyser des plans par lecture d'image, importer des clients, conserver l'historique des conversations, suivre la consommation et le coût.

Le module **ne fait PAS** :

- **Il ne produit pas d'estimation de coût de projet.** Un encart d'avertissement le rappelle à l'écran : « Pour une **estimation**, allez plutôt dans le module **Soumissions → Estimation IA**. L'Assistant IA n'est pas conçu pour produire des estimations de coût. » L'Estimation IA utilise un modèle plus puissant, des profils experts spécialisés et un calcul de prix déterministe — ce que l'Assistant IA n'a pas.
- **Il n'offre pas de sélecteur de profil expert.** L'assistant tourne toujours sur un profil unique (voir §1.7). Vous ne choisissez pas « Électricien » ou « Plombier » dans cet écran.
- **Il n'affiche pas la réponse au fil de la frappe** (pas d'effet « machine à écrire ») : la réponse apparaît d'un bloc, une fois complète. Il n'y a **pas de bouton « Arrêter »** en cours de génération.
- **Il ne permet pas de renommer une conversation** (seulement créer, ouvrir, supprimer).
- **Il n'exporte pas** les conversations ni les analyses (pas de PDF, pas d'impression dédiée). Solution de contournement : sélectionner-copier le texte, ou l'impression du navigateur.
- **Il ne cherche pas sur Internet** et n'appelle pas de services externes : il se limite à vos données et à ses connaissances générales.

### 1.6 Le moteur IA et la facturation à l'usage (en bref)

- **Modèle utilisé** : `claude-sonnet-4-6` (le « Sonnet » d'Anthropic). C'est un modèle rapide et économique — **pas** le modèle « Opus », lequel est réservé à l'Estimation IA. Réponse limitée à 32 000 jetons (`AI_MAX_TOKENS`, `ai.py:45`).
- **Facturation.** Chaque échange consomme des **crédits prépayés** de l'entreprise, calculés d'après le nombre de jetons lus et produits, majorés de 30 % (voir la formule au §4.4). Le solde s'affiche en haut de la page.
- **Recharge automatique.** Quand le solde passe sous 0,10 $, l'ERP tente une recharge automatique par Stripe (montant par défaut 10,00 $). Si la carte est refusée ou le solde épuisé sans recharge, l'assistant renvoie une erreur « Crédits épuisés » (code 402) et affiche une bannière avec un bouton **Recharger** (voir §3.9).
- **Comptes exemptés** : le super-administrateur et quelques entreprises internes (identifiants 1, 105 et 172) ont un usage **illimité**, sans débit.

### 1.7 Nuance importante : « profils experts », deux systèmes à ne pas confondre

Le nom du module parle de « profils experts ». Dans l'ERP, ce terme recouvre **deux mécanismes distincts** — et **un seul** concerne cette page :

1. **Six profils internes** (`AI_PROFILES`, `ai.py:737-790`) : `general` (« Assistant Constructo AI »), `expert_construction`, `estimateur`, `comptable`, `juridique`, `securite`. **L'Assistant IA utilise en permanence le profil `general`.** Le code fixe ce choix (`AssistantIAPage.tsx:38` : `const selectedProfile = 'general';`) et **aucun sélecteur n'est proposé** à l'écran. Vous ne changez donc jamais de profil ici. Le point d'accès `GET /ai/profiles` renverrait bien les six profils, mais la page ne l'appelle pas.
2. **Environ 66 profils experts sur fichier** (Architecte, Électricien, Plombier, Toiture, RBQ et CCQ, Entrepreneur général QC / CA / US, etc.) plus des **profils personnalisés** stockés par entreprise. Ils sont gérés par `ai_profiles.py` et servis par `GET /ai-profiles/experts`. **Ces profils n'alimentent PAS l'Assistant IA** : ils sont consommés par le module **Estimation IA** (Soumissions) et par le **Métré PDF**. Si vous cherchez à choisir un expert précis (par exemple pour une soumission), c'est dans ces modules-là, pas ici.

En résumé : dans cet écran, il y a **un seul cerveau** (`general`), qui **adapte son expertise automatiquement** au sujet de votre question (voir §4.3), sans menu de sélection.

---

## 2. Interface

### 2.1 En-tête de la page

En haut de la zone de dialogue :

- Une icône d'étincelle violette (`Sparkles`), le titre **« Assistant IA »** et le sous-titre **« Expert construction polyvalent — Connecté à vos données »**.
- Le bouton **« Conversations »** (icône `MessageSquare`) : ouvre ou ferme le panneau des conversations enregistrées. Un compteur `(n)` s'affiche à côté du libellé quand des conversations existent.
- L'**indicateur de solde** (icône `CreditCard`) : affiche le montant restant sous la forme `$X.XX`, ou **« Illimité »** si votre entreprise est exemptée. La couleur reflète l'état du solde : **vert** au-dessus de 5 $, **jaune** entre 0 $ et 5 $, **rouge** à 0 $ ou moins.
- Le bouton **« Stats »** (icône `BarChart3`) : affiche ou masque la barre latérale de statistiques (voir §2.9).

### 2.2 Panneau « Conversations »

Ce panneau se déplie quand vous cliquez sur **« Conversations »**.

- En tête : le titre **« Conversations »** et un bouton **« Nouvelle »** (icône `Plus`) qui vide l'écran pour démarrer un fil vierge.
- **Liste des conversations** (défilement vertical) : chaque ligne affiche une icône, le **nom** de la conversation et un compteur **« {n} messages »**. La conversation active est surlignée en bleu.
  - **Un clic sur une ligne** charge la conversation complète dans la zone de dialogue.
  - **Le bouton corbeille** (icône `Trash2`, infobulle « Supprimer ») supprime définitivement la conversation.
- **État vide** : « Aucune conversation ».

> **Rappel des limites.** On peut **créer**, **ouvrir** et **supprimer** une conversation, mais **pas la renommer** : le nom est attribué automatiquement. La liste affiche les 30 conversations les plus récentes de l'utilisateur.

### 2.3 Bannière « Crédits IA épuisés »

Quand le solde est à 0 $ ou moins (et que l'entreprise n'est pas exemptée), une bannière rouge apparaît en haut de la zone de dialogue :

- Icône `AlertTriangle`, titre **« Crédits IA épuisés »** et message **« Rechargez votre solde pour continuer à utiliser l'assistant IA. »**
- Bouton **« Recharger »** (icône `ExternalLink`) : ouvre, dans un nouvel onglet, le portail de facturation Stripe (`https://billing.stripe.com/p/login/constructoai`). Vous y gérez votre carte et votre solde (voir §3.9).

Tant que les crédits sont épuisés, la saisie de message est désactivée.

### 2.4 Zone de dialogue

**État vide (aucun message).** Au centre : une icône d'étincelle, le titre **« Posez votre question »** et un sous-titre d'invitation. Juste en dessous, un **encart d'avertissement ambré** :

> Pour une **estimation**, allez plutôt dans le module **Soumissions → Estimation IA**. L'Assistant IA n'est pas conçu pour produire des estimations de coût.

Cet encart contient aussi deux boutons de démarrage rapide : **« Analyser un document »** et **« Analyser un plan »**, qui ouvrent les panneaux correspondants (voir §2.6 et §2.7).

**Bulles de message.**

- Votre message apparaît dans une **bulle bleue alignée à droite** (avatar « personne »).
- La réponse de l'assistant apparaît dans une **bulle claire à liseré bleu**, avec l'avatar « casque de chantier » (`HardHat`). Le texte est mis en forme (Markdown) : titres, listes, **gras**, et surtout des **tableaux** stylés.
- Sous chaque réponse de l'assistant, un pied de bulle affiche des indicateurs : un **badge de profil** (« Assistant Constructo AI », ou « Analyse Document », ou « Analyse Plan » selon le type), un **badge de type** (Document en bleu, Plan en vert), le **nombre de jetons**, le **coût** en dollars (affiché à quatre décimales, en orange) et la **durée** de la réponse en secondes.

**Pendant l'attente d'une réponse**, un indicateur animé (avatar d'étincelle + roue de chargement) remplace temporairement la future bulle. La réponse s'affiche **d'un seul bloc** une fois prête (pas d'affichage progressif).

### 2.5 Barre de saisie

En bas de la zone de dialogue :

- **Trois boutons-icônes** à gauche du champ :
  - `FileUp` — **« Analyser un document »** : ouvre le panneau d'analyse de document (§2.6).
  - `Image` — **« Analyser un plan »** : ouvre le panneau d'analyse de plan (§2.7).
  - `Database` — **« Importer des clients (CSV/Excel) »** : ouvre la modale d'import (§2.8).
- **Le champ de saisie**, avec le texte d'invite **« Posez votre question à l'expert IA… »** (ou « Crédits épuisés » si le solde est vide). **La touche Entrée envoie le message** (sans `Maj`). Le champ est désactivé pendant qu'une réponse se génère ou si les crédits sont épuisés.
- **Le bouton « Envoyer »** (icône `Send`) : désactivé si le champ est vide, si une réponse est en cours, ou si les crédits sont épuisés.

> **À noter.** Il n'y a **pas de bouton « Arrêter »** pour interrompre une réponse en cours, et **pas d'affichage au fil de la frappe**. Une réponse longue peut prendre quelques secondes ; patientez jusqu'à son affichage complet.

### 2.6 Panneau « Analyser un document »

Ce panneau se déplie au-dessus de la barre de saisie.

- Titre : **« Analyser un document »**.
- Zone de téléversement : **un seul fichier**, taille maximale **50 Mo**. Formats acceptés : **PDF, Word (.docx), Excel (.xlsx), CSV, TXT, Markdown, JSON, HTML** et **images** (JPG, PNG). Le libellé indique « Document (PDF, DOCX, XLSX, CSV, TXT, Images) ».
- Champ **« Instructions spécifiques (optionnel) »** : précisez ce que vous voulez (par exemple « Extrais tous les montants et échéances », « Résume ce contrat en dix points »).
- Bouton **« Analyser le document »** : lance l'analyse. Le résultat s'insère dans le fil sous la forme d'un message assistant précédé de « **Type de document :** … | **Pages :** … ».

Sous le capot : les documents texte sont lus et **tronqués à 100 000 caractères** ; les images sont redimensionnées à 1568 pixels avant lecture.

### 2.7 Panneau « Analyser un plan »

Également au-dessus de la barre de saisie.

- Titre : **« Analyser un plan »**.
- Zone de téléversement : **jusqu'à 10 fichiers**, taille maximale 50 Mo chacun. Formats : **JPG, PNG, PDF**. Le libellé indique « Plans (JPG, PNG, PDF) — jusqu'à 10 fichiers ».
- Bouton **« Analyser les plans »** : lance la lecture d'image. Le résultat s'insère dans le fil, précédé de « **Type de plan :** … | **Fichiers analysés :** … ».

Sous le capot : pour un PDF, seules **les cinq premières pages** sont converties en images (à double résolution) et lues. L'assistant identifie les éléments du plan, estime des surfaces et suggère les corps de métier.

### 2.8 Modale « Importer des clients »

Ouverte par le bouton `Database` de la barre de saisie, cette fenêtre déroule un assistant en **trois étapes**.

**Étape 1 — Choix du fichier.**
- Zone de dépôt : bouton **« Choisir un fichier (.csv ou .xlsx) »**.
- Aide : « Formats acceptés : CSV, Excel (.xlsx). Maximum 8 Mo / 5000 lignes. »
- Boutons **« Annuler »** et **« Analyser le fichier »**. L'analyse est **en lecture seule** : elle ne modifie rien, elle prépare l'aperçu.

**Étape 2 — Aperçu et correspondance des colonnes.**
- Quatre compteurs : **« n à créer »**, **« n à mettre à jour »**, **« n en erreur »**, **« n ligne(s) au total »**.
- Un tableau de **correspondance** : pour chaque colonne de votre fichier, un **menu déroulant** permet de choisir la colonne-client cible. L'IA propose déjà une correspondance ; vous pouvez la corriger. Cibles possibles : **Nom (obligatoire)**, Type, Secteur d'activité, Courriel, Téléphone, Adresse, Ville, Province, Code postal, Pays, Site web, No TPS, No TVQ, Conditions de paiement, Notes — plus l'option **« — Ignorer — »** pour ne pas importer une colonne.
- Garde-fou : le bouton **« Confirmer l'import »** reste **désactivé** tant que la colonne **Nom** n'est pas associée. Le message « La colonne « Nom » est obligatoire : associez-la à une colonne du fichier. » vous le rappelle.
- Boutons **« Retour »** et **« Confirmer l'import »**. La confirmation **écrit réellement** les clients dans votre base.

**Étape 3 — Rapport.**
- Titre **« Import terminé »**, avec le décompte **« n créé(s) »**, **« n mis à jour »**, **« n erreur(s) »** et le détail des erreurs, ligne par ligne.
- Bouton **« Fermer »**.

> **Portée de l'import.** L'assistant d'import alimente uniquement la table des **clients / entreprises** (`companies`). La colonne **Nom** est obligatoire ; le fichier est plafonné à **8 Mo** et **5000 lignes**.

### 2.9 Barre latérale « Stats »

Affichée par le bouton **« Stats »** de l'en-tête, elle empile trois cartes.

**Carte « Crédits IA ».**
- Si l'entreprise est exemptée : mention **« Illimité »** et badge **« Exempt »**.
- Sinon : le **solde** en dollars US, la ligne **« Utilisé ce mois : $X.XXXX »**, et, si la recharge automatique est active, **« Recharge auto : $X »**. Un bouton **« Recharger les crédits »** (lien vers le portail Stripe) apparaît quand le solde est à zéro ou négatif.

**Carte « Coûts quotidiens (30j) ».**
- Un petit **histogramme** de 30 barres (une par jour), avec une infobulle par jour (date, coût, nombre de requêtes), une légende « 30j / Aujourd'hui » et le **total sur 30 jours**.

**Carte « Usage (30 jours) ».**
- Les totaux **Requêtes / Jetons / Coût** sur 30 jours, puis une **ventilation par fonctionnalité** (par exemple `chat_general`, `analyze_document`, `analyze_plan`) avec le nombre de requêtes de chacune.

Ces cartes sont alimentées par les points d'accès `GET /ai/credits`, `GET /ai/usage` et `GET /ai/usage/daily`.

---

## 3. Workflows pas à pas

### 3.1 Poser une question sur vos données

1. Ouvrez l'Assistant IA (bouton « étincelle » de la barre supérieure).
2. Écrivez votre question, par exemple : « Liste les projets en cours avec leur date de fin prévue » ou « Quelles sont mes trois factures impayées les plus anciennes ? ».
3. Appuyez sur **Entrée** (ou cliquez sur **Envoyer**).
4. L'assistant réfléchit, consulte votre base au besoin (outil `recherche_bd`, lecture seule, 50 lignes maximum par requête), puis répond en langage clair, souvent sous forme de **tableau**.
5. Le coût de l'échange et le nombre de jetons s'affichent sous la réponse ; le solde en haut de la page est mis à jour.

> **Astuce.** Vous pouvez demander un format : « Présente-moi ça en tableau avec colonnes Client, Montant, Jours de retard. » L'assistant sait aussi enchaîner : la conversation garde le contexte des échanges précédents.

### 3.2 Demander à l'IA de créer ou de modifier une donnée

L'assistant peut **écrire** dans votre base (outil `executer_action`). Exemples : « Ajoute une note au projet « Rénovation Rive-Sud » : inspection prévue vendredi », « Passe le bon de travail BT-00042 au statut Terminé ».

1. Formulez clairement la demande, en identifiant sans ambiguïté l'enregistrement visé (numéro, nom exact).
2. Envoyez le message.
3. L'assistant exécute l'opération **immédiatement** et confirme le résultat (par exemple « Enregistrement créé, identifiant 128 » ou « 1 ligne modifiée »).

> **Point d'attention majeur — l'écriture est immédiate et sans étape de confirmation.** Dans ce module, il **n'existe pas de mécanisme « je te propose, tu confirmes »** côté serveur : si l'assistant décide d'agir, l'action est appliquée et validée sur-le-champ. Les garde-fous techniques sont : les mots-clés destructeurs (DROP, TRUNCATE, ALTER…) sont bloqués ; **toute modification ou suppression doit comporter une condition `WHERE`** (impossible d'effacer une table entière d'un coup) ; chaque action est journalisée. Mais **il n'y a pas d'annulation automatique**. Soyez précis, et évitez les demandes vagues du type « supprime les vieux dossiers » : préférez « supprime le dossier numéro 57 ». En cas de doute, demandez d'abord à l'assistant de **vous montrer** les enregistrements concernés (lecture) avant de lui demander d'agir.

### 3.3 Reprendre ou gérer une conversation

1. Cliquez sur **« Conversations »** pour dérouler la liste.
2. **Pour reprendre** : cliquez sur une conversation ; son fil complet se recharge, et vous pouvez poursuivre le dialogue.
3. **Pour repartir à neuf** : cliquez sur **« Nouvelle »** ; l'écran se vide (la conversation précédente reste enregistrée).
4. **Pour supprimer** : cliquez sur la **corbeille** de la ligne concernée.

Les conversations sont **enregistrées automatiquement** après chaque échange ; vous n'avez rien à sauvegarder manuellement.

### 3.4 Analyser un document

1. Cliquez sur le bouton **« Analyser un document »** (icône `FileUp`) ou sur le bouton du même nom dans l'encart de l'état vide.
2. Téléversez **un** fichier (PDF, Word, Excel, CSV, TXT, image…), 50 Mo au maximum.
3. Facultatif : précisez vos **instructions** (« Vérifie les clauses de pénalité », « Sors-moi la liste des matériaux et quantités »).
4. Cliquez sur **« Analyser le document »**.
5. La réponse s'ajoute au fil, précédée du type de document et du nombre de pages.

### 3.5 Analyser des plans

1. Cliquez sur **« Analyser un plan »** (icône `Image`).
2. Téléversez **jusqu'à 10** fichiers (JPG, PNG ou PDF).
3. Cliquez sur **« Analyser les plans »**.
4. La réponse (éléments repérés, surfaces estimées, corps de métier suggérés) s'ajoute au fil.

> **Bon à savoir.** Pour un PDF, l'assistant lit **les cinq premières pages** seulement. Si votre plan compte davantage de feuilles utiles, découpez-le ou téléversez les pages pertinentes en images.

### 3.6 Importer des clients depuis un fichier CSV ou Excel

1. Cliquez sur **« Importer des clients (CSV/Excel) »** (icône `Database`).
2. **Étape 1** : choisissez votre fichier (8 Mo / 5000 lignes au maximum), puis **« Analyser le fichier »**.
3. **Étape 2** : vérifiez la **correspondance des colonnes** proposée par l'IA. Assurez-vous que la colonne **Nom** est bien associée (sinon l'import reste bloqué). Ajustez les autres colonnes ou mettez-les à « — Ignorer — ». Contrôlez les compteurs (à créer / à mettre à jour / en erreur).
4. Cliquez sur **« Confirmer l'import »**.
5. **Étape 3** : lisez le **rapport** (créés / mis à jour / erreurs). Corrigez votre fichier au besoin et recommencez pour les lignes en erreur, puis **« Fermer »**.

### 3.7 Surveiller la consommation et le solde

1. Cliquez sur **« Stats »** dans l'en-tête.
2. Consultez la carte **« Crédits IA »** (solde, utilisé ce mois, recharge auto).
3. Consultez l'**histogramme des coûts quotidiens** (30 jours) et la carte **« Usage »** (requêtes, jetons, coût, ventilation par fonctionnalité).

### 3.8 Recharger les crédits

1. Cliquez sur **« Recharger »** (bannière rouge) ou **« Recharger les crédits »** (carte Stats).
2. Le **portail de facturation Stripe** s'ouvre dans un nouvel onglet (`https://billing.stripe.com/p/login/constructoai`).
3. Gérez-y votre carte, votre solde et votre historique. La recharge se fait **hors de l'ERP**, dans l'interface hébergée par Stripe.

> **À savoir.** Il n'y a pas de champ « montant à recharger » dans l'ERP : la recharge passe par Stripe. Si la **recharge automatique** est active, l'ERP recharge tout seul (montant par défaut 10,00 $) dès que le solde tombe sous 0,10 $.

### 3.9 Que faire quand l'assistant affiche « Crédits épuisés »

1. Le champ de saisie est verrouillé et une bannière rouge s'affiche.
2. Cliquez sur **« Recharger »** et complétez l'opération dans Stripe.
3. Revenez dans l'ERP et rechargez la page (ou attendez la mise à jour du solde). Dès que le solde redevient positif, l'assistant répond de nouveau.

Si la recharge automatique échoue (carte refusée), le message d'erreur Stripe (en français) précise la cause ; mettez à jour votre moyen de paiement dans le portail.

---

## 4. Référence

### 4.1 Points d'accès du moteur IA (`ai.py`, préfixe `/api/erp/v1/ai`)

Tous exigent d'être connecté ; aucun ne comporte de garde de rôle supplémentaire.

| # | Méthode + chemin | Rôle | Notes |
|---|---|---|---|
| 1 | `POST /ai/chat` | Dialogue principal (boucle d'outils) | Cœur du module ; utilisé par la page |
| 2 | `GET /ai/conversations` | Liste des conversations (`subject='assistant_ia'`) | 30 au maximum, par utilisateur |
| 3 | `GET /ai/conversations/{id}` | Détail d'une conversation (messages) | |
| 4 | `DELETE /ai/conversations/{id}` | Suppression | 404 si déjà absente |
| 5 | `GET /ai/profiles` | Liste des 6 profils internes | **Non appelé** par cette page |
| 6 | `GET /ai/usage` | Statistiques d'usage (période 1-365 j, défaut 30) | Super-admin = totaux globaux |
| 7 | `GET /ai/credits` | Solde de crédits prépayés | Crée une ligne à 0 si absente |
| 8 | `GET /ai/quota` | Indicateur de quota (`allowed` **toujours vrai**) | **Non appelé** par cette page |
| 9 | `GET /ai/usage/daily` | Ventilation quotidienne (1-90 j, défaut 30) | Alimente l'histogramme |
| 10 | `GET /ai/usage/monthly` | Ventilation mensuelle (1-24 mois, défaut 6) | **Non appelé** par cette page |
| 11 | `POST /ai/analyze-document` | Analyse d'un document | 1 fichier, 50 Mo |
| 12 | `POST /ai/analyze-plan` | Analyse de plans (lecture d'image) | 10 fichiers au maximum |

### 4.2 Points d'accès des profils experts (`ai_profiles.py`, préfixe `/api/erp/v1/ai-profiles`)

> **Rappel** : ces huit points d'accès servent **Estimation IA** et **Métré PDF**, **pas** l'Assistant IA. Ils sont listés ici pour lever la confusion sur les « profils experts ». Aucun n'a de garde d'administrateur : tout utilisateur connecté du tenant peut créer / supprimer un profil personnalisé et y téléverser des documents (jusqu'à 20 Mo).

| # | Méthode + chemin | Rôle |
|---|---|---|
| 1 | `GET /ai-profiles/` | Liste des profils personnalisés du tenant |
| 2 | `POST /ai-profiles/` | Créer un profil personnalisé (nom, instructions) |
| 3 | `GET /ai-profiles/experts` | **≈ 66 profils sur fichier** + profils personnalisés |
| 4 | `GET /ai-profiles/{id}` | Détail d'un profil personnalisé + documents |
| 5 | `PUT /ai-profiles/{id}` | Modifier un profil personnalisé |
| 6 | `DELETE /ai-profiles/{id}` | Supprimer (avec ses documents) |
| 7 | `POST /ai-profiles/{id}/documents` | Ajouter un document de référence (20 Mo max) |
| 8 | `DELETE /ai-profiles/{id}/documents/{doc_id}` | Retirer un document |

### 4.3 Les deux outils SQL de l'assistant

Ces outils ne sont fournis au modèle que si l'utilisateur est rattaché à une entreprise. La boucle d'outils tourne au maximum **cinq fois** par question.

**`recherche_bd` — lecture seule.**
- Uniquement des requêtes `SELECT` / `WITH`, **50 lignes au maximum**, en transaction **lecture seule** avec délai limité à 10 secondes.
- Mots-clés bloqués : `DROP, TRUNCATE, ALTER, CREATE, GRANT, REVOKE, SET ROLE, SET SESSION, COPY, LOCK, VACUUM, TABLE`. Point-virgule interdit (pas d'enchaînement de requêtes). Fonctions de lecture de fichier / d'accès inter-bases bloquées.
- **Données sensibles protégées** : la table des utilisateurs et les colonnes sensibles (mot de passe, NAS, jeton, secret, limite de crédit…) sont refusées dans la requête **et** masquées dans le résultat, même sur un `SELECT *`.

**`executer_action` — écriture.**
- `INSERT` / `UPDATE` / `DELETE` sur **n'importe quelle table de votre entreprise** (pas de liste blanche de tables).
- Garde-fous : mêmes mots-clés bloqués ; **`UPDATE` et `DELETE` sans `WHERE` refusés** ; délai limité à 10 secondes ; validation et journalisation d'audit.
- **Pas d'étape de confirmation côté serveur** : l'exécution est immédiate (voir le point d'attention du §3.2).

**Adaptation automatique de l'expertise.** À chaque question, l'assistant **détecte le sujet** (parmi 18 intentions : conseil, finances, équipe, matériel, échéancier, électricité, plomberie, structure, toiture, isolation, sécurité, juridique, comptabilité, création, modification…) et ajuste son cadrage. Ce réglage est **invisible** et **automatique** ; il n'y a rien à sélectionner.

### 4.4 Modèle IA et calcul du coût

| Élément | Valeur |
|---|---|
| Modèle | `claude-sonnet-4-6` (Sonnet) |
| Jetons maximum par réponse | 32 000 |
| Tarif d'entrée | 0,003 $ / 1000 jetons lus |
| Tarif de sortie | 0,015 $ / 1000 jetons produits |
| Majoration appliquée | × 1,30 (30 %) |

**Formule** (identique au chat, à l'analyse de document et à l'analyse de plan) :

```
coût_$ = ( jetons_entrée × 0,003 + jetons_sortie × 0,015 ) / 1000 × 1,30
```

*Exemples.* Une question simple (≈ 500 jetons lus + 300 produits) coûte environ 0,008 $ (moins d'un cent). Une analyse de plan plus lourde (≈ 4000 + 2000 jetons) coûte environ 0,055 $. Avec un solde de 10 $, cela représente des centaines à plus d'un millier d'échanges selon leur complexité.

### 4.5 Crédits, recharge et seuils

| Paramètre | Valeur | Effet |
|---|---|---|
| Facturation active | oui (par défaut) | Sinon, usage interne illimité sur la clé du client |
| Comptes exemptés | super-admin + entreprises 1, 105, 172 | Usage illimité, aucun débit |
| Seuil de recharge auto | solde < 0,10 $ | Déclenche une recharge Stripe |
| Montant de recharge par défaut | 10,00 $ | Configurable côté plateforme |
| Plafond mensuel de dépense | **aucun** | Décision produit : jamais de blocage dur par l'usage |
| Solde négatif | autorisé | L'usage réel est toujours consommé et tracé |
| En cas d'erreur de vérification | blocage (fail-closed) | L'IA cesse de répondre par prudence |

> **Point d'attention (facturation).** Les points d'accès `/ai/chat`, `/ai/analyze-document` et `/ai/analyze-plan` **débitent sans clé d'idempotence**. En clair : si le réseau relance la requête (double-soumission, reconnexion), le débit peut se **répéter**. Évitez de cliquer plusieurs fois sur « Envoyer » ou de recharger la page pendant une réponse.

### 4.6 Limites et plafonds

| Fonction | Limite |
|---|---|
| Boucle d'outils par question | 5 itérations |
| `recherche_bd` | 50 lignes par requête, lecture seule, délai 10 s |
| `executer_action` | `WHERE` obligatoire sur modification/suppression, délai 10 s |
| Analyse de document | 1 fichier, 50 Mo, texte tronqué à 100 000 caractères |
| Analyse de plan | 10 fichiers, PDF lu sur ses 5 premières pages, images réduites à 1568 px |
| Import de clients | table clients uniquement, colonne Nom obligatoire, 8 Mo / 5000 lignes |
| Historique des conversations | 30 conversations récentes affichées |
| Débit de vitesse (par IP) | 1500 requêtes / 60 s (limite générale de l'ERP) |

> **Note technique.** Contrairement aux mini-assistants des autres modules (plafonnés à 20 requêtes/minute), l'Assistant IA **n'a pas de débit de vitesse dédié** : il retombe sur la limite générale de 1500 requêtes par minute et par adresse IP. Le véritable frein reste le solde de crédits.

### 4.7 Codes de réponse et messages d'erreur

| Code | Signification | Ce que vous voyez |
|---|---|---|
| 200 | Succès | La réponse s'affiche |
| 402 | Crédits épuisés / carte refusée | Bannière « Crédits IA épuisés » + bouton Recharger |
| 403 | Accès au service IA refusé | Message « Accès au service IA refusé. » |
| 413 | Fichier ou conversation trop volumineux | Message dédié |
| 429 | Trop de requêtes (débit dépassé) | Message dédié |
| 503 | Service IA absent ou surchargé | Message dédié ; réessayez plus tard |
| 500 | Erreur interne | Message générique |

En cas d'erreur, la page affiche un encart d'alerte et une bulle « Erreur : {message} ». Le message précis provient du serveur (par exemple la cause du refus de carte).

### 4.8 Tables PostgreSQL

| Table | Emplacement | Rôle |
|---|---|---|
| `ai_prepaid_credits` | `public` (partagée) | Solde de crédits par entreprise et par mois, info Stripe, recharge auto |
| `ai_usage_tracking` | `public` (partagée) | Journal de chaque appel : utilisateur, fonctionnalité, modèle, jetons, coût, durée |
| `ai_credit_applied_invoices` | `public` (partagée) | Anti-doublon des recharges Stripe |
| `ai_credit_ledger` | `public` (partagée) | Registre d'idempotence des débits (non utilisé par les 3 points d'accès du chat — voir §4.5) |
| `conversations` | par tenant | Conversations de l'Assistant IA (`subject='assistant_ia'`) |
| `conversation_messages` | par tenant | Messages des conversations |
| `ai_profiles` | par tenant | Profils experts **personnalisés** (module Estimation IA / Métré) |
| `ai_profile_documents` | par tenant | Documents de référence des profils personnalisés |
| `companies` | par tenant | Clients / entreprises — **cible de l'import** |

> Les tables `ai_*` du schéma `public` sont **partagées** entre toutes les entreprises pour centraliser la facturation IA ; les conversations, elles, restent **isolées** par entreprise et par utilisateur.

### 4.9 Ce qui n'existe pas dans ce module (récapitulatif des limites)

- Pas de **sélecteur de profil expert** (profil `general` fixe).
- Pas d'**affichage au fil de la frappe** ni de bouton **« Arrêter »**.
- Pas de **renommage** de conversation.
- Pas d'**export** ni d'impression dédiée.
- Pas de **recherche sur Internet** ni d'appel à des services externes.
- Pas d'**estimation de coût de projet** (renvoi vers Soumissions → Estimation IA).

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien avec l'Assistant IA |
|---|---|
| **Soumissions → Estimation IA** | La **vraie** estimation de coût. Modèle plus puissant, profils experts sélectionnables, calcul de prix déterministe. L'Assistant IA y renvoie explicitement. |
| **Métré PDF** | Consomme aussi les 66 profils experts sur fichier (via `ai_profiles.py`). |
| **Ventes / Contacts** | L'assistant d'import alimente la table des clients (`companies`), visible ensuite dans Contacts / Ventes. |
| **Configuration / Administration** | Le solde de crédits, la recharge automatique et la facturation Stripe se gèrent au niveau de l'entreprise ; le super-administrateur voit les totaux d'usage de toutes les entreprises. |
| **Autres mini-assistants IA** (Comptabilité, Immobilier, DAO, Messagerie, Métré, Extras…) | Ils réutilisent le **même moteur** (`ai.py`) pour la facturation et l'appel à Claude, mais chacun a son propre périmètre et ses propres garde-fous. Contrairement à l'Assistant IA, plusieurs d'entre eux appliquent un patron « je propose, tu confirmes » avant d'écrire. |

### 5.2 Isolation des données et confidentialité

- Chaque requête de l'assistant s'exécute **dans le schéma de votre entreprise** : les outils `recherche_bd` et `executer_action` ne voient **que vos données**. Une fuite de données d'une entreprise vers une autre par ces outils est bloquée par le cloisonnement PostgreSQL par schéma.
- Les données envoyées à l'API Claude transitent par les serveurs d'Anthropic. Anthropic s'engage à **ne pas utiliser** les données d'API pour l'entraînement (sauf accord explicite). Si vous manipulez des renseignements personnels, tenez compte de vos obligations (Loi 25 / PIPEDA).

### 5.3 FAQ

**Où se trouve l'Assistant IA ? Je ne le vois pas dans le menu de gauche.**
Il n'y est pas. On l'ouvre par le bouton **« Assistant IA »** (icône d'étincelle) de la **barre supérieure**, présent partout dans l'ERP, ou par l'adresse `/assistant-ia`.

**Puis-je choisir un profil « Électricien » ou « Plombier » ici ?**
Non. Cet écran tourne sur un profil unique (`general`) qui **adapte automatiquement** son expertise au sujet de votre question. Les profils experts nommés (Électricien, Plombier, Architecte, RBQ et CCQ…) servent le module **Estimation IA** et le **Métré PDF**, pas l'Assistant IA.

**L'IA peut-elle vraiment modifier ou supprimer mes données ?**
Oui, par l'outil `executer_action` (création / modification / suppression). **L'action est immédiate, sans étape de confirmation** dans ce module. Les protections sont : mots-clés destructeurs bloqués, condition `WHERE` obligatoire, journal d'audit — mais **pas d'annulation automatique**. Soyez précis dans vos demandes (voir §3.2).

**Combien coûte une conversation ?**
Quelques fractions de cent pour une question simple ; environ 0,05 $ pour une analyse de plan complexe. Le coût exact s'affiche sous chaque réponse et se cumule dans l'onglet Stats.

**Pourquoi l'assistant me dit-il d'aller dans « Estimation IA » ?**
Parce que l'Assistant IA **n'est pas conçu pour chiffrer un projet**. L'Estimation IA (module Soumissions) utilise un modèle plus puissant, des profils experts et un calcul de prix contrôlé, adaptés au marché québécois.

**Puis-je exporter une conversation en PDF ?**
Pas de bouton dédié. Solution de contournement : sélectionner-copier le texte, ou utiliser l'impression du navigateur.

**Le chat s'affiche-t-il mot par mot ?**
Non. La réponse apparaît d'un bloc, une fois complète. Il n'y a pas non plus de bouton « Arrêter ».

**Que se passe-t-il si mon solde est épuisé ?**
L'assistant renvoie une erreur « Crédits épuisés » (402) et la saisie se verrouille. Rechargez via le portail Stripe (bouton **Recharger**). Si la recharge automatique est active, l'ERP recharge tout seul dès que le solde passe sous 0,10 $.

**Puis-je téléverser plusieurs documents à analyser ?**
Pour l'**analyse de document**, un seul fichier à la fois (50 Mo). Pour l'**analyse de plan**, jusqu'à dix fichiers. Un PDF de plan n'est lu que sur ses **cinq premières pages**.

**L'import de clients peut-il modifier des clients existants ?**
Oui : l'aperçu distingue « à créer » et « à mettre à jour ». La confirmation applique les deux. La colonne **Nom** est obligatoire ; fichier plafonné à 8 Mo / 5000 lignes.

**Un employé sans droits d'administrateur peut-il utiliser l'assistant ?**
Oui. Tout utilisateur connecté de l'entreprise y a accès, **y compris** la capacité de faire écrire l'IA dans la base. Il n'y a pas de contrôle de rôle sur ce module ; le seul frein est le solde de crédits.

**Mes conversations sont-elles privées à mon compte ?**
Oui, elles sont rattachées à votre utilisateur et à votre entreprise ; les 30 plus récentes s'affichent dans le panneau Conversations.

**L'IA peut-elle chercher sur Internet ?**
Non. Elle se limite à vos données et à ses connaissances générales. Pour la recherche web, voyez le module **Web** de l'ERP.

---

## 6. Récapitulatif

- **Accès par la barre supérieure** (bouton « étincelle » **Assistant IA**), **pas** par le menu latéral ; adresse `/assistant-ia`.
- **Un seul profil actif** (`general`), **sans sélecteur** ; l'expertise s'adapte automatiquement au sujet. Les « 66 profils experts » appartiennent à **Estimation IA** et au **Métré PDF**, pas à ce module.
- **Modèle** `claude-sonnet-4-6` (Sonnet), 32 000 jetons maximum — pas Opus.
- **Deux outils SQL** : `recherche_bd` (lecture, 50 lignes) et `executer_action` (**écriture immédiate, sans confirmation serveur** ; `WHERE` obligatoire). C'est le **seul** assistant de l'ERP avec accès en écriture — à utiliser avec prudence.
- **Analyse de document** (1 fichier, 50 Mo) et **analyse de plans** (10 fichiers, PDF sur 5 pages) par lecture d'image.
- **Assistant d'import de clients** CSV / Excel en trois étapes (aperçu, correspondance, rapport), colonne **Nom obligatoire**, 8 Mo / 5000 lignes.
- **Conversations** enregistrées automatiquement : créer, ouvrir, supprimer — **pas de renommage, pas d'export**.
- **Pas d'affichage au fil de la frappe, pas de bouton « Arrêter »**.
- **Facturation à l'usage** : coût = (entrée × 0,003 + sortie × 0,015) / 1000 × 1,30 ; recharge auto Stripe sous 0,10 $ (défaut 10,00 $) ; pas de plafond mensuel dur ; super-admin et entreprises 1/105/172 exemptés.
- **N'est pas un outil d'estimation** : l'écran renvoie vers **Soumissions → Estimation IA**.
- **Aucun contrôle de rôle** sur le module : tout utilisateur connecté peut dialoguer et déclencher des écritures ; le frein est le solde de crédits.

---

**Documentation générée à partir du code source vérifié** : `backend/routers/ai.py` (3541 lignes, 12 points d'accès), `backend/routers/ai_profiles.py` (804 lignes, 8 points d'accès), `backend/routers/data_import.py` (684 lignes), `frontend/src/pages/AssistantIAPage.tsx` (716 lignes), `frontend/src/components/ai/ClientImportModal.tsx` (243 lignes), `frontend/src/api/ai.ts`, `frontend/src/api/dataImport.ts`, `frontend/src/components/layout/TopBar.tsx` (bouton d'accès, lignes 312-320), `i18n/locales/fr/ai.json`.

**Manuels liés** :
- Module 08 (Ventes — Soumissions / **Estimation IA**) — `08-ventes-soumissions.md`
- Module 05 (Gestion des contacts — clients importés) — `05-gestion-contacts.md`
- Module 19 (Terrain — Immobilier, mini-assistant IA) — `19-terrain-immobilier.md`
- Module 24 (Communication — Messagerie, mini-assistant IA) — `24-communication-messagerie.md`
- Module 28 (Configuration — crédits IA, Stripe) — `28-configuration.md`
- Module 30 (Métré PDF — profils experts) — `30-metre-pdf.md`
