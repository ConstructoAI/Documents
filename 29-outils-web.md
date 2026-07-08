# Module 29 — Web (recherche temps réel)

> **Version** : 3.0 (refonte vérifiée contre le code source, juillet 2026)
> **Code de référence** : `backend/routers/web.py` (572 lignes, **4 points d'entrée**, préfixe réel `/api/erp/v1/web`, tag « Recherche Web »), `frontend/src/pages/WebPage.tsx` (736 lignes, **5 onglets** — page unique, aucun dossier `components/web`), `frontend/src/store/useWebStore.ts`, `frontend/src/api/web.ts`, `frontend/src/i18n/locales/{fr,en}/web.json`
> **Tables PostgreSQL** : `web_search_history` (dans le schéma de chaque tenant, **créée à la demande** au premier usage — aucune migration officielle). Facturation via les tables partagées `public.ai_prepaid_credits` (solde), `public.ai_credit_ledger` (idempotence) et `public.ai_usage_tracking` (traçage).
> **Cadrage** : interface de **recherche web en temps réel** et d'**analyse de page** alimentée exclusivement par les **outils serveur de Claude (Anthropic)** — `web_search_20260209` et `web_fetch_20260209`. Il n'y a **AUCUNE** intégration directe avec Google, Bing, SerpAPI, Brave ou DuckDuckGo : toute la recherche est exécutée côté Anthropic. Le module utilise le modèle **Claude Opus** (`claude-opus-4-8`), le plus puissant et le plus coûteux de l'ERP, et débite les **crédits IA prépayés** du tenant. Pour interroger vos **données internes** (projets, factures, clients), ce n'est pas ce module : utilisez l'**Assistant IA** et son outil `recherche_bd`.

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

Donner à l'entrepreneur et à ses employés un **accès à l'information du web en direct**, sans quitter l'ERP, avec des réponses rédigées et **des sources citées** :

- **Recherche web** — poser une question en langage naturel ; Claude interroge le web et rédige une réponse structurée avec la liste des sources.
- **Analyse de page** — fournir l'adresse (URL) d'une page web ou d'un PDF ; Claude en récupère le contenu, le résume et en dégage les points importants.
- **Recherche + analyse** — Claude cherche d'abord des sources, retient les 1 ou 2 plus pertinentes, puis les analyse en profondeur avant de synthétiser.
- **Historique** — la liste de **vos** recherches récentes, conservée par tenant.
- **Liens utiles** — 8 raccourcis vers les organismes officiels de la construction au Québec (CCQ, RBQ, CNESST, etc.).

C'est un outil de **veille et de recherche documentaire** (nouvelles normes, prix de marché, réglementation, fiches techniques de produits). Ce n'est **pas** un moteur de recherche interne à vos données, ni un extracteur de fichiers que vous téléversez.

### 1.2 Ce que le module fait — et ne fait pas

| Le module **fait** | Le module **ne fait pas** |
|---|---|
| Chercher sur le web public et citer les sources | Chercher dans vos projets/factures/clients (voir l'Assistant IA) |
| Lire une page ou un PDF **accessible en ligne** par son URL | Analyser un fichier que vous **téléversez** depuis votre poste |
| Synthétiser plusieurs sources | Exporter la réponse en PDF ou CSV, ou l'imprimer via un bouton |
| Conserver un historique en lecture seule | Relancer, modifier ou supprimer une entrée d'historique |
| Filtrer par domaines autorisés/bloqués | Garantir l'exactitude des sources (à valider par vous) |

### 1.3 Le fournisseur : Anthropic uniquement (pas de Google ni de Bing)

> **Important.** Ce module **ne s'appuie sur aucun moteur de recherche classique** (Google Custom Search, Bing, SerpAPI, Brave, DuckDuckGo). Il utilise **exclusivement les deux outils web natifs de Claude**, exécutés côté serveur d'Anthropic :
> - `web_search_20260209` — recherche web temps réel et citations (`web.py:236`, `web.py:431`) ;
> - `web_fetch_20260209` — récupération et analyse du contenu d'une URL (`web.py:338`, `web.py:443`).

La clé d'accès utilisée est `ANTHROPIC_API_KEY` (variable d'environnement, partagée avec l'Assistant IA). **Aucune solution de repli** n'existe : si le service Anthropic est indisponible ou si la clé est absente, le module répond **503 « Service IA non disponible »** (`web.py:217`), sans autre moteur pour prendre le relais.

Le serveur de l'ERP **ne visite jamais lui-même** les sites tiers : c'est Anthropic qui exécute la recherche et le chargement des pages. C'est un point utile pour la conformité (aucune requête sortante depuis vos serveurs vers les sites analysés).

### 1.4 Le modèle et son coût

Source : `web.py:32-33`, `web.py:70-88`.

| Paramètre | Valeur réelle |
|---|---|
| Modèle | **`claude-opus-4-8`** (Opus — le modèle premium) |
| `max_tokens` de la réponse | **32 000** |
| Appel | En **transmission continue (streaming)**, déporté hors de la boucle d'événements partagée pour ne pas ralentir l'ERP (`web.py:91-101`, `asyncio.to_thread`) |
| Tarif entrée | 5 $ US / million de jetons (tokens) |
| Tarif sortie | 25 $ US / million de jetons |
| Tarif écriture de cache | 6,25 $ US / million de jetons |
| Tarif lecture de cache | 0,50 $ US / million de jetons |
| Majoration interne | **× 1,30** (marge de 30 %) |

> Le module Web utilise **Opus, pas Sonnet** (le défaut de l'ERP pour l'Assistant IA est `claude-sonnet-4-6`). C'est un choix délibéré pour la qualité de synthèse sur des recherches ouvertes — mais cela en fait **la fonction IA la plus chère** de la plateforme. Une ancienne surfacturation (tarifs Opus « hérités » à 15 $/75 $ le million, soit ×3) a été corrigée ; les tarifs ci-dessus sont ceux, exacts, de `claude-opus-4-8` (`web.py:70-72`).

**Il n'y a aucune température fixée** dans l'appel (contrairement à ce qu'indiquait une ancienne version de ce manuel) : le modèle tourne à sa valeur par défaut. La formule de coût exacte est donnée en [4.4](#44-modèle-tarifs-et-formule-de-coût).

### 1.5 Une géolocalisation figée sur le Québec

Toutes les recherches (onglets **Recherche Web** et **Recherche + Analyse**) transmettent à Claude une localisation d'utilisateur **codée en dur** : Montréal / Québec / CA / fuseau `America/Montreal` (`web.py:239-245` et `web.py:434-440`). Cela oriente les résultats vers des sources locales et francophones. Ce réglage **n'est pas modifiable** depuis l'interface. L'onglet **Analyse de Page** ne l'utilise pas (il cible une URL précise que vous fournissez).

### 1.6 Accès, permissions et mode consultation

- **Où le trouver** : barre latérale, section **Outils** → **Web** (icône `Globe`), adresse `/web` (`Sidebar.tsx:98-103`, `App.tsx:126`). L'onglet ouvert par défaut est **Recherche Web**.
- **Qui peut l'utiliser** : **tout utilisateur authentifié** du tenant. Les quatre points d'entrée sont protégés par la seule vérification d'ouverture de session (`Depends(get_current_user)`) ; **aucun rôle particulier n'est exigé** (ni administrateur, ni comptable). Il n'y a pas de restriction par métier.
- **Le vrai garde-fou, c'est le crédit.** À chaque recherche payante, le serveur vérifie le solde de crédits IA prépayés ; si le solde est épuisé, il répond **402 « Crédits IA insuffisants »** (`web.py:228-230`). (Une seconde vérification interne, `check_ai_guard`, existe mais **ne bloque personne** dans la configuration actuelle — elle laisse passer tout utilisateur authentifié ; le blocage effectif vient uniquement du crédit.)
- **Mode consultation (lecture seule).** Un tenant sans abonnement Stripe actif (abonnement annulé, sans carte) bascule en **mode consultation** sur tout l'ERP. Dans cet état, les **trois fonctions de recherche** (qui sont des requêtes d'écriture) sont **bloquées en 403 « Mode consultation »**, tandis que l'onglet **Historique** (lecture) et l'onglet **Liens utiles** restent accessibles.
- **Instance interne.** Si la facturation est désactivée au niveau du serveur (`ERP_BILLING_ENABLED=false`), il n'y a **ni vérification de crédit ni débit** : la recherche est libre. Les super-administrateurs de la plateforme sont également exemptés de débit.

### 1.7 Les 5 onglets

Source : tableau `TABS` (`WebPage.tsx:261-267`), barre d'onglets ARIA (`WebPage.tsx:374-402`).

| # | Onglet | Icône | Nature | Point d'entrée |
|---|---|---|---|---|
| 1 | **Recherche Web** | `Search` | Fonction IA payante | `POST /web/search` |
| 2 | **Analyse de Page** | `FileText` | Fonction IA payante | `POST /web/fetch` |
| 3 | **Recherche + Analyse** | `Zap` | Fonction IA payante | `POST /web/search-fetch` |
| 4 | **Historique** | `History` | Lecture seule | `GET /web/history` |
| 5 | **Liens utiles** | `ExternalLink` | Statique (aucun serveur) | — |

Trois onglets consomment des crédits, un affiche l'historique, un dernier n'est qu'une liste de raccourcis.

---

## 2. Interface

### 2.1 Disposition générale

En haut de la page : le titre **« Web - Recherche et Analyse »** avec l'icône `Globe` bleue (`WebPage.tsx:360-363`). Juste en dessous, une **bannière d'erreur** apparaît si la dernière action a échoué ; elle est **fermable à la main** et ne disparaît pas toute seule (voir [2.9](#29-messages-derreur-affichés)). Puis la **barre des 5 onglets**. Le contenu change selon l'onglet actif ; l'onglet ouvert par défaut est **Recherche Web**.

### 2.2 Onglet « Recherche Web »

Source : `WebPage.tsx:405-475`.

| Élément | Détail |
|---|---|
| **Encart d'information** (bleu) | « **Recherche web en temps réel** - Claude recherche sur internet et fournit une réponse avec les sources citées. » |
| **Champ de saisie** | Zone de texte de 3 lignes. Exemple affiché : « Ex: Quelles sont les dernières innovations en construction modulaire au Québec? ». |
| **Curseur « Max recherches »** | Plage **1 à 10, défaut 5**. C'est le nombre maximal de recherches web que Claude peut lancer pour répondre. Plus élevé = plus complet, mais plus long et plus cher. La valeur choisie s'affiche à droite. |
| **Filtrage de domaines** | Composant partagé (voir [2.8](#28-le-filtre-de-domaines-composant-partagé)) : trois modes **Aucun / Autoriser / Bloquer** + un champ de domaines. |
| **Bouton « Rechercher »** | Lance la recherche. Pendant le traitement, il devient « Recherche en cours... » avec une roue animée, et un indice s'affiche : « Recherche web en cours, cela peut prendre quelques secondes... ». |
| **Bouton « Effacer »** | Apparaît **seulement** une fois un résultat obtenu ; il vide le résultat affiché. |
| **Résultat** | Carte de résultat commune (voir [2.5](#25-la-carte-de-résultat-commune)), titrée « Résultat de la recherche ». |

Un **verrou anti-double-clic** empêche de facturer deux fois une même recherche si vous cliquez rapidement (`WebPage.tsx:317-328`).

### 2.3 Onglet « Analyse de Page »

Source : `WebPage.tsx:478-563`.

| Élément | Détail |
|---|---|
| **Encart d'information** (bleu) | « **Analyse de page web ou PDF** - Claude récupère et analyse en détail le contenu d'une URL spécifique. » |
| **Champ URL** | Champ de type URL avec icône de lien. Exemple : « https://exemple.com/page-a-analyser ». Si vous saisissez une adresse **sans** `http://` ou `https://` (ex. `rbq.gouv.qc.ca`), le préfixe **`https://` est ajouté automatiquement** (`WebPage.tsx:335`). |
| **Curseur « Tokens max »** | Plage **10 000 à 200 000, par pas de 10 000, défaut 100 000** ; affiché en « K ». C'est le plafond de contenu de la page que Claude ingère. Une page volumineuse au plafond coûte plus cher. |
| **Case « Citations »** | **Cochée par défaut.** Demande à Claude de citer précisément les passages. |
| **Filtrage de domaines** | Identique à l'onglet Recherche (Aucun / Autoriser / Bloquer). |
| **Bouton « Analyser »** | Lance l'analyse ; devient « Analyse en cours... » avec l'indice « Récupération et analyse de la page en cours... ». |
| **Bouton « Effacer »** | Vide le résultat affiché. |
| **Résultat** | Carte commune, titrée « Analyse de la page ». |

Le serveur impose à Claude une **analyse structurée en 4 points** (`web.py:349-359`) : **1. Résumé** — **2. Points clés** — **3. Contexte et implications** — **4. Recommandations ou conclusions**.

> Le nombre maximal de récupérations de cet onglet est **figé à 5** côté application (`WebPage.tsx:338`) et n'est **pas** réglable par l'utilisateur.

### 2.4 Onglet « Recherche + Analyse »

Source : `WebPage.tsx:566-662`.

| Élément | Détail |
|---|---|
| **Encart d'information** (bleu) | « **Recherche web + analyse approfondie** - Claude recherche d'abord les informations pertinentes, puis analyse en détail les meilleures sources trouvées. » |
| **Avertissement** (ambre) | « Cette fonction utilise plus de ressources car elle combine recherche web ET analyse détaillée des sources. Temps de réponse plus long (30-60 secondes). » |
| **Champ de saisie** | Zone de texte de 3 lignes. Exemple : « Ex: Analyse détaillée des normes de construction sismique au Québec en 2025 ». |
| **Curseur « Max recherches »** | Plage **1 à 5, défaut 3**. |
| **Curseur « Max analyses »** | Plage **1 à 5, défaut 2**. |
| **Domaines autorisés** | Un **seul champ** « Domaines autorisés (optionnel) ». **Pas** de bascule Aucun/Autoriser/Bloquer ici : cet onglet ne prend en charge **que** le filtrage « autoriser » (limite du point d'entrée, `web.py:450-452`). |
| **Bouton « Rechercher et Analyser »** | Lance le traitement ; devient « Recherche approfondie en cours... » avec l'indice « Recherche approfondie en cours, cela peut prendre 30-60 secondes... ». |
| **Bouton « Effacer »** | Vide le résultat affiché. |
| **Résultat** | Carte commune, titrée « Analyse approfondie ». |

Le serveur enchaîne les deux outils (recherche **puis** analyse) et donne à Claude un **processus en 4 étapes** (`web.py:454-465`) : rechercher → repérer les 1-2 meilleures sources → en récupérer le contenu complet → synthétiser. Dans ce mode, le contenu chargé par page est **plafonné à 50 000 jetons** (codé en dur, `web.py:447`) — inférieur au plafond de l'onglet Analyse de Page (jusqu'à 200 000), parce qu'on combine ici deux opérations coûteuses.

### 2.5 La carte de résultat commune

Source : `ResultCard` (`WebPage.tsx:102-162`). Les trois onglets payants affichent le même bloc de résultat.

**Barre de statistiques** (petit texte gris), affichée selon ce qui s'est passé :
- **« {n} recherche(s) »** — nombre de recherches web effectuées (si supérieur à 0) ;
- **« {n} analyse(s) »** — nombre de pages analysées (si supérieur à 0) ;
- **« {s}s »** — durée de l'appel, en secondes (icône horloge) ;
- **« {n} tokens »** — total des jetons d'entrée + sortie ;
- **« {x,xxxx} $ USD »** — le **coût réellement facturé** de cet appel, à **4 décimales**.

**Bloc de texte** : la réponse de Claude, affichée telle quelle (les retours à la ligne sont respectés, mais il n'y a **pas** de mise en forme enrichie du Markdown). Si Claude n'a rien renvoyé de textuel : « Aucun résultat textuel. »

**Bloc « Sources »** (encadré bleu) : titre « Sources ({n}) » puis la liste des liens cités, chacun ouvrant la source dans un **nouvel onglet** du navigateur (icône de lien externe). Les doublons d'URL sont éliminés côté serveur (`web.py:133`, `web.py:145`).

> Il n'y a **aucun bouton** « Copier », « Partager », « Enregistrer » ni « Imprimer » sur la carte de résultat. Pour conserver une réponse, sélectionnez le texte et copiez-le (voir la FAQ).

### 2.6 Onglet « Historique »

Source : `WebPage.tsx:665-718`, serveur `web.py:514-571`.

- **Chargement automatique** dès que vous ouvrez l'onglet ; il demande les **50 entrées les plus récentes** (`WebPage.tsx:299-303`).
- **Sous-titre** : « Historique de vos recherches web récentes. » + un bouton **« Rafraîchir »** pour recharger.
- **Si vide** : icône d'historique + « Aucune recherche dans l'historique. » + « Vos recherches apparaîtront ici après utilisation. »
- **Chaque entrée** (carte) affiche :
  - un **badge de type** coloré : **« Recherche »** (bleu), **« Analyse »** (vert) ou **« Recherche + Analyse »** (violet) ;
  - la **date** localisée (format `fr-CA`) ;
  - **« {n} source(s) »** si des sources ont été citées ;
  - la **requête** (ou l'URL) en gras ;
  - un **aperçu** du résultat (2 lignes maximum).

> **L'historique est en lecture seule.** Il n'y a **ni** bouton « Relancer », **ni** « Voir », **ni** suppression, **ni** export. Pour refaire une recherche, recopiez la question dans l'onglet approprié.
>
> **L'historique est personnel.** Chaque utilisateur ne voit **que ses propres** recherches (filtrage par identifiant d'utilisateur, `web.py:539`) — il n'est **pas** partagé entre les collègues d'un même tenant. Il est aussi borné à **50** entrées affichées.

### 2.7 Onglet « Liens utiles »

Source : `LIENS_UTILES` (`WebPage.tsx:38-95`), textes dans `web.json`.

Introduction : « Organismes gouvernementaux, outils en ligne et ressources essentielles pour le secteur de la construction au Québec. » Puis une grille de **8 cartes** (titre + description + icône colorée), chacune ouvrant le site dans un **nouvel onglet** :

| # | Organisme | Adresse |
|---|---|---|
| 1 | Commission de la construction du Québec (CCQ) | `https://www.ccq.org` |
| 2 | Régie du bâtiment du Québec (RBQ) | `https://www.rbq.gouv.qc.ca` |
| 3 | CNESST | `https://www.cnesst.gouv.qc.ca` |
| 4 | Revenu Québec | `https://www.revenuquebec.ca` |
| 5 | Code de construction du Québec | `https://www.rbq.gouv.qc.ca/lois-reglements-et-codes` |
| 6 | Registre des entreprises du Québec (REQ) | `https://www.registreentreprises.gouv.qc.ca` |
| 7 | Vérificateur de licences RBQ | `https://www.rbq.gouv.qc.ca/services-en-ligne/licence/registre-des-detenteurs-de-licence` |
| 8 | Plan Québec | `https://www.quebec.ca/gouvernement/politiques-orientations/plan-quebecois-infrastructures` |

Cette liste est **fixe** ; elle n'est pas configurable depuis l'interface et ne consomme aucun crédit.

### 2.8 Le filtre de domaines (composant partagé)

Source : composant `DomainFilter` (`WebPage.tsx:194-255`).

Présent sur **Recherche Web** et **Analyse de Page**. Titre « **Filtrage de domaines** », trois boutons-pastilles :

- **« Aucun »** — aucune restriction (défaut) ;
- **« Autoriser »** (vert) — Claude **ne consulte que** les domaines listés ;
- **« Bloquer »** (rouge) — Claude **exclut** les domaines listés.

Dès que vous choisissez Autoriser ou Bloquer, un champ apparaît, avec l'exemple « Ex: quebec.ca, canada.ca, rbq.gouv.qc.ca ». Séparez les domaines par des **virgules**.

Règles à connaître :
- **Autoriser et Bloquer sont mutuellement exclusifs** : on ne peut pas combiner les deux (`web.py:248-251`).
- Vous pouvez saisir jusqu'à **20** domaines, mais le serveur n'en retient que les **10 premiers** (`web.py:249`, `web.py:251`).
- L'onglet **Recherche + Analyse** n'offre **que** « Domaines autorisés » (pas de Bloquer, pas de Aucun) — voir [2.4](#24-onglet--recherche--analyse-).

### 2.9 Messages d'erreur affichés

Source : `useWebStore.ts:35-56`, `web.json:134-137`.

En cas d'échec, une bannière rouge s'affiche en haut de la page. Le message provient d'abord du détail renvoyé par le serveur (ex. « Crédits IA insuffisants »). Deux messages génériques existent côté application :

- **Réseau / délai dépassé** (erreurs 502-504, coupure réseau, absence de réponse) : « La recherche a expiré ou la connexion a été interrompue. Réessayez, idéalement avec moins de recherches ou d'analyses. »
- **Erreur inconnue** (repli) : « Une erreur est survenue. Veuillez réessayer. »

La bannière **reste affichée** tant que vous ne la fermez pas (croix) ; lancer une nouvelle recherche l'efface automatiquement (`WebPage.tsx:305-307`).

---

## 3. Workflows pas à pas

### 3.1 Effectuer une recherche web simple

1. Barre latérale → **Outils** → **Web** → onglet **Recherche Web**.
2. Saisir la question (ex. « Nouvelles exigences RBQ 2026 pour les permis de construction résidentielle »).
3. (Optionnel) Ajuster le curseur **Max recherches** (1 à 10). Plus élevé = réponse plus fouillée, mais plus coûteuse.
4. (Optionnel) Activer un **filtre de domaines** : **Autoriser** pour se limiter à certaines sources, **Bloquer** pour en écarter.
5. Cliquer **Rechercher**.
6. Attendre quelques secondes. En coulisse : le serveur vérifie le crédit, prépare l'outil `web_search` avec la localisation Québec, appelle Claude Opus en continu, extrait le texte, les sources et les compteurs, calcule le coût, débite le crédit, journalise l'usage (fonctionnalité `web_search`) et enregistre l'entrée d'historique (type « search »).
7. Lire la réponse, vérifier les **sources** citées (elles s'ouvrent dans un nouvel onglet), noter le **coût en $ USD** affiché.

### 3.2 Analyser une page web ou un PDF

1. Onglet **Analyse de Page**.
2. Coller l'**URL complète** (ex. `https://www.cnesst.gouv.qc.ca/.../guide-EPI.pdf`). Sans schéma, `https://` est ajouté pour vous.
3. (Optionnel) Régler **Tokens max** (10 000 à 200 000), décocher **Citations**, ou filtrer par domaines.
4. Cliquer **Analyser**.
5. En coulisse : le serveur valide l'URL (elle doit commencer par `http://` ou `https://`), prépare l'outil `web_fetch` (citations activées, plafond de contenu), demande l'analyse en 4 points, débite le crédit, enregistre l'historique (type « fetch »).
6. Lire l'analyse structurée (Résumé / Points clés / Contexte / Recommandations).

> Utile pour : dépouiller un devis de fournisseur en ligne, résumer un long guide réglementaire en PDF, extraire l'essentiel d'une fiche technique de produit.

### 3.3 Recherche + analyse combinée

1. Onglet **Recherche + Analyse**.
2. Saisir une question d'investigation (ex. « Analyse détaillée des normes sismiques au Québec en 2026 »).
3. Régler **Max recherches** (1 à 5, défaut 3) et **Max analyses** (1 à 5, défaut 2).
4. (Optionnel) Saisir des **domaines autorisés** (mode « autoriser » uniquement).
5. Cliquer **Rechercher et Analyser**. Compter **30 à 60 secondes**.
6. En coulisse : Claude cherche, retient les meilleures sources, en récupère le contenu (plafonné à 50 000 jetons par page), puis synthétise. Débit du crédit, historique (type « search_fetch »).
7. Lire la synthèse approfondie et ses sources.

> C'est le mode **le plus coûteux** : à réserver aux questions qui exigent vraiment de lire des sources en entier. Pour une réponse rapide, l'onglet **Recherche Web** suffit souvent.

### 3.4 Restreindre ou exclure des domaines

**Se limiter aux sources gouvernementales du Québec** (onglet Recherche Web ou Analyse de Page) : mode **Autoriser** → saisir par exemple `quebec.ca, gouv.qc.ca, ccq.org, rbq.gouv.qc.ca, cnesst.gouv.qc.ca`. Rappel : 10 domaines effectifs au maximum.

**Écarter des sources jugées peu fiables** : mode **Bloquer** → saisir par exemple `reddit.com, quora.com`.

**Sur Recherche + Analyse** : seul le champ « Domaines autorisés » est disponible ; il n'y a pas de mode « bloquer ».

### 3.5 Consulter l'historique

Onglet **Historique** → la liste se charge seule (50 plus récentes). Bouton **Rafraîchir** pour recharger. Pour refaire une recherche, **recopiez** la question dans l'onglet voulu : il n'y a pas de relance en un clic. Rappel : vous ne voyez que **vos** recherches, pas celles de vos collègues.

### 3.6 Accéder à un organisme officiel

Onglet **Liens utiles** → cliquer sur une carte (CCQ, RBQ, CNESST, Revenu Québec, REQ, etc.) → le site s'ouvre dans un nouvel onglet. Astuce : copiez une URL précise depuis le site officiel puis collez-la dans l'onglet **Analyse de Page** pour en faire résumer le contenu.

### 3.7 Suivre et maîtriser les coûts

Le module Web **n'a pas son propre tableau de bord** de coûts. Trois réflexes :

1. **Regardez le coût affiché** sur chaque carte de résultat (« {x,xxxx} $ USD ») — c'est le montant exact débité.
2. **Choisissez le bon onglet** : Recherche Web (le moins cher) pour une réponse rapide ; réservez Recherche + Analyse aux cas qui l'exigent.
3. **Baissez les curseurs** (Max recherches, Max analyses, Tokens max) quand vous n'avez pas besoin d'exhaustivité.

Pour le suivi agrégé de la consommation IA (toutes fonctions confondues), reportez-vous au module **Assistant IA** : les fonctionnalités `web_search`, `web_fetch` et `web_search_fetch` y apparaissent dans le traçage d'usage.

### 3.8 Que faire si « Crédits IA insuffisants » ou « Mode consultation »

- **« Crédits IA insuffisants » (402)** : le solde de crédits prépayés du tenant est épuisé. Une **recharge automatique** de 10 $ US via Stripe se déclenche normalement quand le solde passe sous 0,10 $ US ; si elle n'a pas eu lieu (carte absente ou refusée), l'administrateur du tenant doit régulariser le moyen de paiement. Réessayez ensuite.
- **« Mode consultation » (403)** : le tenant est en lecture seule (abonnement annulé ou sans carte). Les recherches sont bloquées ; l'**Historique** et les **Liens utiles** restent consultables. Il faut réactiver l'abonnement pour retrouver les fonctions payantes.
- **« Service IA non disponible » (503)** : le service Anthropic n'est pas joignable. Réessayez plus tard ; il n'y a pas de moteur de secours.
- **Message de délai / réseau** : réessayez, idéalement avec **moins** de recherches ou d'analyses (curseurs plus bas).

---

## 4. Référence

### 4.1 Points d'entrée du routeur Web

Préfixe complet : `/api/erp/v1/web` (`erp_api.py:1047`, `web.py:20`). Tous protégés par l'ouverture de session (`get_current_user`), sans autre garde de rôle.

| Méthode | Chemin | Fonctionnalité tracée | Rôle |
|---|---|---|---|
| POST | `/web/search` | `web_search` | Recherche web temps réel (`web.py:212`) |
| POST | `/web/fetch` | `web_fetch` | Récupère et analyse une URL (`web.py:312`) |
| POST | `/web/search-fetch` | `web_search_fetch` | Recherche puis analyse des meilleures sources (`web.py:407`) |
| GET | `/web/history` | — (lecture) | Historique **de l'utilisateur** courant (`web.py:514`) |

### 4.2 Corps de requête (validation Pydantic)

Source : `web.py:43-64`.

| Requête | Champs et bornes |
|---|---|
| **WebSearchRequest** | `query` (requis, ≤ 4000 caractères) ; `max_uses` (1-10, défaut 5) ; `allowed_domains` / `blocked_domains` (≤ 20 côté client, **tronqués à 10** côté serveur ; mutuellement exclusifs) |
| **WebFetchRequest** | `url` (requis, ≤ 2048 caractères, doit commencer par `http(s)://`) ; `max_uses` (1-10, mais **figé à 5** par l'interface) ; `enable_citations` (défaut **vrai**) ; `max_content_tokens` (1000-200000, défaut 100000) ; `allowed_domains` / `blocked_domains` (≤ 20 → 10) |
| **WebSearchFetchRequest** | `query` (requis, ≤ 4000) ; `max_search_uses` (1-5, défaut 3) ; `max_fetch_uses` (1-5, défaut 2) ; `allowed_domains` **seulement** (≤ 20 → 10) |
| **GET /web/history** | `limit` (1-100, défaut 20) — l'interface demande 50 |

### 4.3 Format de réponse (les trois POST)

Les clés sont converties en `camelCase` par l'intercepteur (`api/web.ts:15-25`).

```json
{
  "text": "Réponse rédigée par Claude, avec ses sections.",
  "citations": [{ "title": "...", "url": "https://..." }],
  "search_count": 3,
  "fetch_count": 1,
  "input_tokens": 14832,
  "output_tokens": 2571,
  "cost_usd": 0.1783,
  "elapsed_seconds": 22.4,
  "credit_balance": 9.4217
}
```

`credit_balance` est le solde estimé **après** cet appel. `GET /web/history` renvoie `{ "items": [ { id, user_id, search_type, query, result_preview, citations_count, created_at } ] }`.

### 4.4 Modèle, tarifs et formule de coût

Source : `web.py:32-33`, `web.py:80-88`.

| Paramètre | Valeur |
|---|---|
| `WEB_AI_MODEL` | `claude-opus-4-8` |
| `WEB_AI_MAX_TOKENS` | `32000` |
| Température | Aucune (valeur par défaut du modèle) |
| Tarif entrée | 5 $ US / million de jetons |
| Tarif sortie | 25 $ US / million de jetons |
| Écriture de cache | 6,25 $ US / million de jetons |
| Lecture de cache | 0,50 $ US / million de jetons |
| Majoration | × 1,30 |

**Formule** (`_compute_web_cost`, `web.py:80-88`) :

```
coût_USD = ( entrée × 5      / 1 000 000
           + sortie × 25     / 1 000 000
           + écriture_cache × 6,25 / 1 000 000
           + lecture_cache × 0,50  / 1 000 000 ) × 1,30
```

**Ordres de grandeur indicatifs** (le coût réel dépend surtout de la quantité de contenu web ingéré ; fiez-vous au montant affiché sur la carte) :

| Action | Jetons approx. (entrée / sortie) | Coût approx. |
|---|---|---|
| Recherche web simple | ~15 000 / ~2 500 | ~0,18 $ US |
| Analyse d'une grande page (100 K) | ~100 000 / ~3 000 | ~0,75 $ US |
| Recherche + analyse | ~30 000 / ~5 000 | ~0,36 $ US |

### 4.5 Limites et plafonds

| Limite | Valeur |
|---|---|
| `max_uses` — Recherche Web / Analyse de Page | 10 (Analyse de Page figé à 5) |
| `max_search_uses` / `max_fetch_uses` — Recherche + Analyse | 5 chacun |
| `max_content_tokens` — Analyse de Page | 200 000 |
| `max_content_tokens` — Recherche + Analyse | 50 000 (codé en dur) |
| Domaines autorisés / bloqués | 20 saisis, **10 retenus** |
| Longueur de la question | 4 000 caractères |
| Longueur de l'URL | 2 048 caractères |
| Réponse du modèle | 32 000 jetons |
| Historique affiché | 50 entrées (borne serveur 100) |
| Troncature en base (`query`, `result_preview`) | 500 caractères chacun |

### 4.6 Codes d'erreur et statuts

Source : `web.py`, `erp_auth.py`.

| Cas | Statut | Message |
|---|---|---|
| SDK / clé Anthropic absent | 503 | « Service IA non disponible » |
| Question vide | 400 | « La requête de recherche est vide » |
| URL vide | 400 | « L'URL est vide » |
| URL sans `http(s)://` | 400 | « L'URL doit commencer par http:// ou https:// » |
| Garde IA refuse (ne bloque personne aujourd'hui) | 403 | « Accès IA refusé » |
| Solde de crédits épuisé | 402 | « Crédits IA insuffisants » |
| Tenant en lecture seule | 403 | « Mode consultation » |
| Tenant bloqué | 401 | (accès refusé) |
| Trop de requêtes (voir 4.7) | 429 | (avec en-tête `Retry-After: 60`) |
| Erreur interne (recherche) | 500 | « Erreur lors de la recherche web » |
| Erreur interne (analyse) | 500 | « Erreur lors de l'analyse de la page » |
| Erreur interne (recherche + analyse) | 500 | « Erreur lors de la recherche approfondie » |
| Erreur de chargement de l'historique | 500 | « Erreur lors du chargement de l'historique » |

### 4.7 Limitation de débit (rate limiting)

Source : `erp_api.py:365`, `erp_api.py:473`, `erp_api.py:691-693`.

- **10 requêtes par 60 secondes et par adresse IP** sur les **trois** fonctions de recherche (clé partagée `{ip}:web`). Justification : Opus à 32 000 jetons plus des frais serveur par recherche = la classe la plus chère.
- L'onglet **Historique** (`GET /web/history`) n'est **pas** soumis à cette limite dédiée ; il retombe sur le plafond général (1 500 requêtes/minute).
- En cas de dépassement : réponse **429** avec `Retry-After: 60`. Patientez une minute.

### 4.8 Table `web_search_history` (schéma du tenant)

Créée **à la demande** (`CREATE TABLE IF NOT EXISTS`) au premier enregistrement, puis mise en cache par processus pour éviter un verrou à chaque recherche (`web.py:173-186`). Aucune migration officielle.

```sql
CREATE TABLE IF NOT EXISTS web_search_history (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    search_type VARCHAR(30) NOT NULL,   -- 'search' | 'fetch' | 'search_fetch'
    query TEXT NOT NULL,                 -- tronqué à 500 caractères
    result_preview TEXT,                 -- tronqué à 500 caractères
    citations_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- Lecture **filtrée par `user_id`** : historique **personnel**, pas partagé (`web.py:539`).
- Écriture **au mieux** (best-effort) : si l'enregistrement échoue, la réponse de recherche n'est **pas** compromise (`web.py:193-205`).
- Pas d'index dédié, pas de clé étrangère vers `users`, **pas de purge automatique**.

### 4.9 Facturation et idempotence

Séquence identique sur les trois POST : `check_ai_guard` → `_check_credits` (402 si épuisé) → appel Claude → `_compute_web_cost` → `track_ai_usage` → `_deduct_credits` (débit idempotent) → `_save_search_history`.

- **Débit** sur `public.ai_prepaid_credits` (le solde peut devenir négatif ; c'est la prochaine vérification qui déclenchera la recharge Stripe de 10 $ US sous le seuil de 0,10 $ US). Super-administrateurs et instance sans facturation : **pas de débit**.
- **Idempotence** : en-tête `Idempotency-Key` (l'application génère un identifiant unique par requête, `api/web.ts:49-51`). Le serveur le transforme en `web:{clé}:{empreinte de la question ou de l'URL}` et l'enregistre dans `public.ai_credit_ledger` (`INSERT ... ON CONFLICT DO NOTHING`). Résultat : un **double-clic** ou un **réessai** avec la même clé **ne débite pas deux fois**, mais réutiliser une clé sur une requête **différente** ne permet **pas** d'esquiver la facturation.

### 4.10 Réglages non exposés / codés en dur

| Réglage | Valeur | Modifiable par l'utilisateur ? |
|---|---|---|
| Localisation des recherches | Montréal / Québec / CA | Non |
| Modèle | `claude-opus-4-8` | Non |
| Température | Défaut du modèle | Non |
| `max_uses` de l'Analyse de Page | 5 | Non |
| `max_content_tokens` de Recherche + Analyse | 50 000 | Non |
| Majoration de coût | × 1,30 | Non |
| Domaines effectifs | 10 (sur 20 saisis) | Partiellement |

---

## 5. Intégrations et FAQ

### 5.1 Lien avec l'Assistant IA (crédits et traçage)

Le module Web **réutilise** la mécanique de crédits de l'Assistant IA : `check_ai_guard`, `_check_credits`, `_deduct_credits`, `track_ai_usage` (importés depuis `ai.py`). Ses trois fonctionnalités — `web_search`, `web_fetch`, `web_search_fetch` — apparaissent dans le **traçage d'usage IA** partagé (`public.ai_usage_tracking`), aux côtés des autres consommations IA de l'ERP. Voir le manuel **Assistant IA** (`24-communication-assistant-ia.md`).

### 5.2 Crédits prépayés et Stripe

Mêmes crédits que tout l'ERP (`public.ai_prepaid_credits`). **Recharge automatique** de 10 $ US quand le solde passe sous 0,10 $ US. **Instances internes** (`ERP_BILLING_ENABLED=false`) et **super-administrateurs** : sans débit. La gestion du moyen de paiement et de l'abonnement relève du module de **Configuration** (`30-configuration.md`).

### 5.3 À ne pas confondre : recherche web ≠ recherche interne ≠ recherche matériaux

| Besoin | Le bon outil |
|---|---|
| Chercher sur **le web** (normes, prix, réglementation) | **Ce module** (Web) |
| Chercher dans **vos données** (projets, factures, clients) | **Assistant IA** → outil `recherche_bd` (`24-communication-assistant-ia.md`) |
| Chercher un **matériau / fournisseur** en ligne depuis le magasin | Composant **Recherche web de matériaux** du module **Magasin** (`components/magasin/MaterialWebSearch.tsx`) — un outil **distinct**, calqué sur celui-ci mais ciblé matériaux, **hors** de la route `/web` (`09-operations-magasin.md`) |

### 5.4 Ce qui n'est PAS possible

- **Aucun export** (PDF, CSV, Markdown), **aucun bouton d'impression**, **aucun téléversement de fichier** dans ce module.
- **Pas de relance** ni de suppression depuis l'historique ; **pas de partage** entre utilisateurs (historique personnel).
- **Pas de mise en forme enrichie** du résultat (texte brut, retours à la ligne respectés).
- **Pas de changement** de la localisation (Québec), du modèle, ni de la température.
- **Pas de recherche planifiée** (aucune automatisation / minuterie) : tout est à la demande.
- **Pas de moteur de secours** si Anthropic est indisponible.
- Sur **Recherche + Analyse** : pas de mode « Bloquer » ni « Aucun » (seulement « autoriser »).

### 5.5 FAQ

**Quel moteur de recherche est utilisé ?** Les outils natifs de Claude (`web_search_20260209` + `web_fetch_20260209`), exécutés chez Anthropic. Aucune intégration Google, Bing, SerpAPI, Brave ou DuckDuckGo.

**Pourquoi Opus et pas Sonnet ?** Pour la qualité de synthèse sur des recherches ouvertes. En contrepartie, c'est la fonction IA la plus chère de l'ERP.

**Combien coûte une recherche ?** Le montant exact s'affiche sur la carte de résultat. Ordres de grandeur : recherche simple ~0,18 $ US, analyse de grande page ~0,75 $ US, recherche + analyse ~0,36 $ US. Le coût varie surtout avec la quantité de contenu web lu.

**Puis-je faire analyser un fichier de mon ordinateur ?** Non. L'onglet Analyse de Page ne prend qu'une **URL** publiquement accessible. Pour un document local, hébergez-le d'abord en ligne (ou utilisez un autre module).

**La géolocalisation Montréal/Québec est-elle modifiable ?** Non, elle est codée en dur. Claude reste multilingue et peut chercher en anglais, mais les résultats sont orientés Québec/francophone.

**Un collègue peut-il voir mes recherches ?** Non. L'historique est filtré par utilisateur : chacun ne voit que les siennes.

**Puis-je relancer une recherche depuis l'historique ?** Non, il est en lecture seule. Recopiez la question dans l'onglet voulu.

**La transmission continue (streaming) est-elle visible à l'écran ?** Non. Le serveur utilise la transmission continue (requise par Anthropic pour les longues opérations d'outils), mais l'interface attend la réponse complète, puis l'affiche d'un coup.

**Y a-t-il un cache des résultats ?** Non. Chaque recherche est ré-exécutée (et refacturée). Consultez d'abord l'historique pour éviter de payer deux fois la même chose.

**Les sources sont-elles fiables ?** Les URL viennent d'Anthropic ; l'ERP ne les valide pas. **Vérifiez-les** avant de vous y fier ou de cliquer.

**Un double-clic me facture-t-il deux fois ?** Non : un verrou anti-double-clic et une clé d'idempotence empêchent le double débit d'une même requête.

**Que se passe-t-il si je dépasse un curseur ?** Rien : les curseurs sont déjà bornés, et le serveur re-plafonne les valeurs par sécurité.

**Mes recherches servent-elles à entraîner l'IA ?** Elles transitent par l'API Anthropic. Anthropic n'utilise pas les données d'API pour l'entraînement, sauf accord explicite. En présence de renseignements personnels, tenez compte de la Loi 25 et de la PIPEDA.

---

## 6. Récapitulatif

- **Cadrage** : recherche web + analyse de page en direct, via les outils Claude `web_search_20260209` et `web_fetch_20260209`. **Fournisseur unique : Anthropic** — aucun Google/Bing/SerpAPI ; **503** si indisponible, sans repli.
- **Modèle** : `claude-opus-4-8`, réponse jusqu'à 32 000 jetons, **aucune température fixée**. C'est la fonction IA **la plus chère** de l'ERP.
- **5 onglets** : Recherche Web · Analyse de Page · Recherche + Analyse (les trois payants) · Historique (lecture seule, **personnel**) · Liens utiles (8 raccourcis statiques).
- **4 points d'entrée** : `POST /web/search`, `POST /web/fetch`, `POST /web/search-fetch`, `GET /web/history`.
- **Géolocalisation** figée sur Montréal/Québec. **Filtrage de domaines** : Aucun / Autoriser / Bloquer (mutuellement exclusifs, 20 saisis → 10 retenus) ; **Recherche + Analyse** = « autoriser » seulement.
- **Plafonds** : `max_uses` 10 (Analyse de Page figé à 5) ou 5 (mode combiné) ; contenu de page 200 000 jetons (50 000 en mode combiné).
- **Facturation** : crédits IA prépayés, tarifs Opus (5/25/6,25/0,50 $ US le million) **× 1,30**. Recharge Stripe automatique de 10 $ US sous 0,10 $ US. **Idempotence** en place (pas de double débit). Instance interne / super-admin : sans débit.
- **Accès** : tout utilisateur authentifié ; aucune restriction de rôle. Le tenant en **mode consultation** voit ses 3 recherches bloquées en **403**, mais garde l'historique et les liens.
- **Débit d'appel** : 10 requêtes / 60 s par IP sur les recherches ; 429 avec `Retry-After` en cas de dépassement.
- **Historique** : table `web_search_history` par tenant, créée à la demande, filtrée par utilisateur, 50 entrées affichées, **sans** relance/suppression/export.
- **Limites** : pas d'export, pas d'impression, pas de téléversement, pas de mise en forme enrichie, pas de cache, pas d'automatisation. Pour vos **données internes**, utilisez l'**Assistant IA** (`recherche_bd`).

---

**Sources vérifiées** : `backend/routers/web.py` (572 lignes, 4 points d'entrée), `frontend/src/pages/WebPage.tsx` (736 lignes, 5 onglets), `frontend/src/store/useWebStore.ts`, `frontend/src/api/web.ts`, `frontend/src/i18n/locales/fr/web.json` (138 lignes), `backend/erp_api.py` (montage + limitation de débit), `backend/erp_auth.py` (mode consultation), `backend/routers/ai.py` (crédits, traçage, idempotence).

**Manuels liés** :
- Assistant IA (crédits prépayés, traçage, `recherche_bd`) — `24-communication-assistant-ia.md`
- Magasin (recherche web de matériaux, outil distinct) — `09-operations-magasin.md`
- Configuration (abonnement, moyen de paiement, crédits) — `30-configuration.md`
- Ventes — Dossiers (conserver un résultat dans un dossier) — `06-ventes-dossiers.md`
