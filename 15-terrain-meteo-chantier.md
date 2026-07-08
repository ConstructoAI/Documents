# Module 15 — Météo chantier

> **Version** : 3.0 (refonte vérifiée contre le code source du 7 juillet 2026 — ajout de l'assistant IA, du cache 30 min et correction de la localisation réelle des endpoints)
> **Libellé dans le menu** : « Météo » (groupe « TERRAIN » de la barre latérale, icône `CloudSun`) — route `/meteo`
> **Code de référence (backend)** : `ERP_REACT/backend/routers/secondary.py` lignes 8653-8744 (2 endpoints météo Open-Meteo, ~92 lignes) ; `ERP_REACT/backend/routers/meteo_ai.py` (165 lignes, 1 endpoint `POST /meteo/ai/chat`, assistant IA de planification)
> **Code de référence (frontend)** : `ERP_REACT/frontend/src/pages/MeteoPage.tsx` (281 lignes, page unique) ; `ERP_REACT/frontend/src/components/meteo/MeteoAssistantTab.tsx` (122 lignes, chat IA) ; `ERP_REACT/frontend/src/api/secondary.ts` lignes 47-53 (2 fonctions météo) ; `ERP_REACT/frontend/src/api/meteoAi.ts` lignes 25-37 (fonction du chat)
> **Service externe** : Open-Meteo (`https://api.open-meteo.com/v1/forecast`) — API publique, gratuite, sans clé, sans coût
> **Tables PostgreSQL** : aucune table de tenant. Le module ne stocke aucune prévision (cache mémoire 30 min seulement). Seul l'assistant IA écrit dans les tables partagées `public.ai_usage_tracking` (traçage) et `public.ai_prepaid_credits` (débit des crédits).
> **Cadrage** : outil de **consultation passive** des prévisions sur 7 jours pour 7 stations urbaines du Québec, avec mise en évidence automatique des risques chantier (gel, pluie, vent) et un **assistant IA de planification** (chat de conseils, sans accès aux données). Il **n'est pas** un système d'alertes poussées : il ne génère ni courriel, ni notification, ni évènement de calendrier, et n'a **aucun lien de base de données** vers les projets, les phases ou les bons de travail.

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

Donner aux équipes de terrain et aux planificateurs une **vue rapide des conditions météo prévues sur 7 jours** pour les principales agglomérations québécoises, avec une **lecture automatique des risques chantier** (gel, précipitations, vent) accompagnée de **recommandations opérationnelles** (couler du béton oui ou non, asphaltage, peinture extérieure, travaux en hauteur), et un **assistant IA** qui conseille l'organisation des travaux selon la météo.

Concrètement, le module répond à trois questions :

- « Quel temps fera-t-il à Québec, Montréal ou Saguenay au cours des 7 prochains jours ? »
- « Quelles activités de chantier dois-je adapter, reporter ou suspendre en fonction de cette météo ? »
- « Comment organiser ma semaine de travaux compte tenu des conditions annoncées ? » (assistant IA)

### 1.2 Ce que le module fait (vérifié contre le code)

- Lister **7 stations météo** québécoises précâblées (Montréal, Québec, Gatineau, Trois-Rivières, Sherbrooke, Saguenay, Rimouski) via `GET /weather/stations` (`secondary.py:8653`).
- Récupérer les **prévisions journalières sur 7 jours** depuis Open-Meteo via `GET /weather/forecast?lat=X&lon=Y` (`secondary.py:8692`) : température maximale, température minimale, précipitations totales (mm) et vent maximal (km/h).
- Afficher **7 cartes** (une par jour) avec une **icône dynamique** selon la condition dominante et un **badge** si un seuil est franchi : « Gel », « Pluie » ou « Vent fort ».
- Marquer la carte du jour courant avec un badge « Aujourd'hui » et un liseré bleu (date civile en heure du Québec).
- Générer une **section « Impact chantier »** consolidée qui liste les journées à risque avec une **recommandation textuelle** par évènement (6 niveaux, voir section 4.5). Si aucune journée ne franchit de seuil : message vert « Aucune alerte météo - conditions favorables pour les travaux ».
- Ouvrir un **assistant IA de planification** (chat en langage naturel) qui conseille l'organisation des travaux selon la météo — sans accès à la base de données du tenant.

### 1.3 Ce que le module ne fait pas

> **Important** : ce module est **strictement consultatif**. La partie prévisions est en lecture seule ; seul l'assistant IA consomme des crédits. Le module **n'implémente pas** :

- **Aucun lien avec les projets ni les chantiers.** Malgré le nom « Météo chantier », la météo est fournie par **ville fixe** (7 stations) et jamais par adresse de chantier ni par projet. Il n'y a aucune sélection de projet, aucune géolocalisation, aucune coordonnée tirée d'un dossier. La correspondance « météo-chantier » est **humaine** : vous choisissez vous-même la station la plus proche.
- **Aucune saisie de localisation personnalisée dans l'écran.** Le backend accepte techniquement un couple latitude/longitude arbitraire, mais l'interface n'envoie **que** les coordonnées des 7 stations codées en dur. Pas de champ adresse ni GPS.
- **Aucune persistance ni historique.** Les prévisions viennent d'Open-Meteo en direct ; un cache mémoire de 30 minutes existe côté serveur, mais **aucune table de base de données** ne conserve la météo.
- **Prévision quotidienne uniquement** (7 jours). Pas de prévision horaire, pas de « ressenti », pas d'humidité, d'indice UV, de couverture nuageuse, de description textuelle ni d'accumulation de neige. Seulement 4 mesures par jour.
- **Alertes purement visuelles.** Les seuils de gel, de pluie et de vent sont calculés **côté client** sur les prévisions affichées : **aucune notification poussée** (courriel, notification navigateur, message texte, webhook), **aucun rappel**, **aucun seuil configurable** (les seuils sont codés en dur dans `MeteoPage.tsx`).
- **Aucun blocage automatique.** Même si l'écran affiche « ARRÊTER les travaux en hauteur », aucun bon de travail ni aucune phase n'est suspendu par le système.
- **Aucune exportation** (PDF, CSV, iCal), aucune impression, aucun téléversement.
- **Aucun comparatif multistations** simultané (une seule station affichée à la fois).
- **Aucune alerte d'urgence civile** : le module n'est pas branché sur Environnement Canada, le MSC ni le ministère de la Sécurité publique. Les seuils sont des repères internes, pas des avertissements officiels.

### 1.4 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **Météo** (icône `CloudSun`). Réf. `Sidebar.tsx:71`.
- URL directe : `/meteo`.
- Fil d'Ariane : « Météo ».
- Page unique, sans onglets ni sous-pages. L'assistant IA s'ouvre dans une fenêtre modale par-dessus la page.

### 1.5 Permissions et rôles

- **Authentification requise.** Les 3 endpoints sont gardés par `get_current_user` : tout utilisateur connecté au tenant peut consulter la météo et utiliser l'assistant.
- **Aucun rôle particulier requis.** Ni administrateur, ni super-administrateur, ni rôle métier : la consultation est ouverte à tous les membres de l'entreprise.
- **Aucune restriction par tenant.** Les 7 stations sont identiques pour tous ; Open-Meteo est une API mondiale et publique.
- Pour l'assistant IA, le seul gardien réel est le **solde de crédits IA** de l'entreprise (voir section 4.6). Un compte sans crédits reçoit une erreur 402 et ne peut pas envoyer de message, mais la consultation météo, elle, reste gratuite.

### 1.6 Composants du module

| Composant | Fichier | Rôle |
|-----------|---------|------|
| Page météo | `pages/MeteoPage.tsx` | Écran unique : barre d'en-tête, 7 cartes, section Impact chantier, bouton d'ouverture de l'assistant |
| Assistant IA (chat) | `components/meteo/MeteoAssistantTab.tsx` | Fenêtre modale de conversation avec l'assistant de planification |
| Bulle de message | `components/ai/MessageBubble.tsx` | Rendu des messages du chat (partagé avec les autres assistants IA) |
| API météo | `api/secondary.ts` (47-53) | `listWeatherStations()` et `getWeatherForecast(lat, lon)` |
| API chat | `api/meteoAi.ts` (25-37) | `chatMeteo(message, history, language, weatherContext)` |
| Endpoints météo | `routers/secondary.py` (8653-8744) | Stations fixes + proxy Open-Meteo avec cache |
| Endpoint IA | `routers/meteo_ai.py` (65) | Chat de planification, un seul appel à Claude |

### 1.7 Coûts et limites externes

- **Consultation météo : gratuite.** Open-Meteo ne demande aucune clé et tolère environ 10 000 requêtes par jour et par adresse IP. Un cache serveur de 30 minutes réduit fortement le nombre d'appels réels (voir section 4.8).
- **Assistant IA : payant.** Chaque message envoyé à l'assistant consomme des **crédits IA** de l'entreprise (tarif du modèle `claude-sonnet-4-6` majoré de 30 %). C'est le **seul** effet financier du module. Détail du calcul en section 4.6.
- **Délai serveur Open-Meteo : 10 secondes.** Au-delà, la requête échoue en douceur et l'écran affiche un message de service temporairement indisponible.

### 1.8 Architecture technique

```
Frontend MeteoPage.tsx ──> Backend secondary.py /weather/*  ──> Open-Meteo (public, gratuit)
   listWeatherStations()      GET /weather/stations               api.open-meteo.com/v1/forecast
   getWeatherForecast()       GET /weather/forecast               (cache serveur 30 min)

Frontend MeteoAssistantTab ─> Backend meteo_ai.py /meteo/ai/chat ─> Claude (sonnet-4-6)
   chatMeteo()                POST /meteo/ai/chat                  (sans base de données, sans outil)
                                                                   débit crédits IA + traçage usage
```

Aucune base de données côté prévisions. L'assistant IA n'accède à aucune donnée du tenant : il ne fait qu'un seul appel au modèle et écrit uniquement la trace d'usage et le débit de crédits dans les tables partagées `public.ai_usage_tracking` et `public.ai_prepaid_credits`.

---

## 2. Interface

Source : `MeteoPage.tsx` (281 lignes) et `MeteoAssistantTab.tsx` (122 lignes).

### 2.1 Disposition générale

```
+-----------------------------------------------------------------------+
| Météo          Mis à jour à 14:05   [Sparkles Assistant IA] [⟳] [ v ] |
+-----------------------------------------------------------------------+
|  [J1]   [J2]   [J3]   [J4]   [J5]   [J6]   [J7]   <- 7 cartes du jour  |
|                        [Aujourd'hui]                                  |
|                          [Gel]                                        |
+-----------------------------------------------------------------------+
| ShieldAlert  Impact chantier - Recommandations                        |
|  [Snowflake] ven. 2 mai - Gel prévu (-1 °C)          [Attention]      |
|              Protéger le béton frais avec couvertures isolantes...     |
|  [Droplets]  sam. 3 mai - Pluie importante (14 mm)   [Attention]      |
+-----------------------------------------------------------------------+
```

La grille des cartes est adaptative : 1 colonne sur téléphone, 2 sur tablette, 4 sur écran moyen, 7 sur grand écran (1280 px et plus).

### 2.2 Barre d'en-tête

Située en haut de la page (`MeteoPage.tsx:114-151`), de gauche à droite :

| Élément | Comportement |
|---------|--------------|
| **Titre « Météo »** | Toujours affiché. |
| **« Mis à jour à HH:MM »** | Horodatage du dernier chargement réussi. Apparaît **seulement après un succès**, il est masqué sur les petits écrans. Un échec en douceur ne met pas à jour cet horodatage (pour éviter d'afficher une heure trompeuse). |
| **Bouton « Assistant IA »** (icône `Sparkles`) | Ouvre la fenêtre modale de l'assistant de planification (voir 2.6). |
| **Bouton Actualiser** (icône `RefreshCw`) | Recharge les prévisions de la station courante. L'icône tourne pendant le chargement. Le bouton est désactivé pendant un chargement ou si aucune station n'est sélectionnée. Infobulle « Actualiser ». |
| **Menu déroulant de station** | Liste les 7 stations par leur nom. Changer de station recharge automatiquement les prévisions. |

### 2.3 Les 7 cartes de prévision

Chaque carte (`MeteoPage.tsx:156-208`) présente une journée. Anatomie détaillée :

| Élément | Source ou logique |
|---------|-------------------|
| **Jour abrégé** | `formatWeekdayShort(f.date)` → format « ven. 2 mai » (jour de semaine abrégé, quantième, mois abrégé) en français canadien. |
| **Badge « Aujourd'hui »** | Affiché, avec un liseré bleu autour de la carte, si la date de la carte est celle du jour en heure du Québec (`America/Montreal`). |
| **Icône du jour** (dynamique) | `Snowflake` si froid (min < 0), sinon `CloudRain` si pluvieux (précip > 5), sinon `Wind` si venteux (vent > 40), sinon `CloudSun`. La couleur suit : bleue si froid ou pluvieux, orange si venteux, jaune sinon. |
| **Ligne « Max »** | Température maximale, suivie du symbole « ° » (teinte rose). |
| **Ligne « Min »** | Température minimale, suivie de « ° ». Affichée en bleu si elle est sous 0. |
| **Ligne « Pluie »** | Précipitations en mm. Affichée en bleu si supérieure à 5 mm. |
| **Ligne « Vent »** | Vent maximal en km/h. Affiché en orange si supérieur à 40 km/h. |
| **Liseré de la carte** | Contour jaune si la journée franchit au moins un des trois seuils de carte (froid, pluie ou vent). |
| **Badge de condition** | Affiché si un seuil de carte est franchi. **Un seul badge**, par ordre de priorité : « Gel » (bleu) si froid ; sinon « Vent fort » (rouge) si venteux ; sinon « Pluie » (jaune). |

> **Affichage sûr des valeurs manquantes.** Si Open-Meteo renvoie une valeur nulle, la ligne affiche « -- » au lieu de « null ». Les nombres sont mis en forme selon la langue (fr-CA ou en-CA), avec au plus une décimale.

> **Attention aux deux jeux de seuils.** Les seuils qui déclenchent un **badge sur la carte** sont plus permissifs que ceux de la section « Impact chantier ». Une carte peut donc porter un badge « Pluie » (précip > 5 mm) sans qu'aucune recommandation n'apparaisse plus bas (l'Impact chantier ne réagit qu'au-delà de 10 mm). Voir le tableau comparatif en section 4.5.

### 2.4 Section « Impact chantier »

Bloc ajouté sous la grille, **uniquement si au moins une prévision est disponible** (`MeteoPage.tsx:214-268`). Les seuils sont calculés côté client, sur les prévisions affichées. Deux variantes :

**Variante A — aucune alerte** (aucune journée ne franchit les seuils stricts) :

```
[HardHat vert]  Impact chantier
                Aucune alerte météo - conditions favorables pour les travaux
```

**Variante B — alertes détectées** :

```
[ShieldAlert]  Impact chantier - Recommandations
+---------------------------------------------------------------+
| [Icône]  jour - message court                    [Badge]      |
|          recommandation détaillée                              |
+---------------------------------------------------------------+
| ... une ligne par évènement ...                                |
+---------------------------------------------------------------+
```

Chaque ligne porte une icône selon le type (`Snowflake` pour le gel, `Droplets` pour la pluie, `Wind` pour le vent), le libellé « jour - message », un badge de sévérité (« Critique » en rouge ou « Attention » en jaune) et le texte de recommandation.

> **Cumul possible.** Le gel, la pluie et le vent sont évalués indépendamment : une même journée peut donc générer **jusqu'à 3 lignes** distinctes dans l'Impact chantier (par exemple gel + pluie + vent le même jour). À l'intérieur d'une catégorie, seule la ligne la plus grave s'affiche (critique **ou** attention, jamais les deux).

### 2.5 États de chargement, vide et erreur

- **Chargement** (liste des stations ou prévisions en cours) : squelette de page (`SkeletonPage`) à la place de la grille.
- **Aucune prévision** (tableau vide sans erreur) : message centré « Aucune prévision disponible ». La section Impact chantier ne s'affiche pas.
- **Erreur** : une bande rouge en haut de la page affiche le message. Deux cas : « Erreur lors du chargement de la météo » (échec du chargement des stations) ou « Service météo temporairement indisponible. Réessayez plus tard. » (échec des prévisions).
- **Course entre requêtes** : si vous changez rapidement de station, une réponse tardive d'une station précédente est ignorée (garde interne anti-course), pour ne jamais afficher les prévisions d'une autre ville que celle sélectionnée.
- **Changement de langue** : basculer FR/EN ne redéclenche pas d'appel réseau et ne réinitialise pas la station choisie.

### 2.6 Fenêtre modale de l'assistant IA

Ouverte par le bouton « Assistant IA », titre « **Assistant IA — Météo & planification** », taille grande (`MeteoPage.tsx:271-278`). Elle transmet à l'assistant un contexte du type « Station météo consultée : {nom de la station} ».

Le chat (`MeteoAssistantTab.tsx`) comprend :

- **En-tête** : icône `Sparkles`, titre et sous-titre « Conseils d'organisation de chantier selon la météo. ».
- **État vide** : message d'invitation « Pose une question de planification de chantier selon la météo (coulage de béton, toiture, gel/dégel, vent, précipitations). L'assistant conseille en fonction des conditions ; il ne consulte pas tes données internes. », suivi de **3 exemples cliquables à lire** :
  - « Puis-je couler du béton demain avec ces températures ? »
  - « Quelles précautions pour la toiture s'il vente fort ? »
  - « Comment organiser la semaine si de la pluie est prévue ? »
- **Messages** : les vôtres à droite, ceux de l'assistant à gauche avec une icône de casque. Sous chaque réponse de l'assistant, un pied de bulle affiche un badge « Météo », le nombre de jetons consommés, le **coût en dollars américains** (en orange, à 4 décimales) et la durée de la réponse en secondes.
- **En cours** : indicateur « Analyse en cours… ».
- **Erreur** : bande rouge sous le fil ; le message provient du serveur si disponible, sinon « Une erreur est survenue. Réessaie. ».
- **Zone de saisie** : champ multiligne avec l'invite « Pose ta question de planification météo… ». La touche **Entrée** envoie le message ; **Maj+Entrée** insère un saut de ligne. Bouton « Envoyer » (icône d'avion en papier). Un verrou interne empêche le double envoi.

---

## 3. Workflows pas à pas

### 3.1 Consulter la météo d'un chantier (procédure principale)

1. Barre latérale → groupe **TERRAIN** → **Météo**.
2. La page se charge sur Montréal par défaut (première station de la liste), avec 7 cartes.
3. Dans le menu déroulant en haut à droite, choisissez la **ville la plus proche du chantier**.
4. Parcourez les 7 cartes : repérez les journées avec un **liseré jaune** ou un **badge** (« Gel », « Pluie » ou « Vent fort »).
5. Descendez à la section **Impact chantier** pour lire les recommandations propres aux journées à risque.
6. Décidez manuellement (reporter une coulée, avancer une livraison, réorganiser les équipes, annuler des travaux en hauteur, etc.).
7. Communiquez la décision aux équipes par un autre canal (module Messagerie, courriel, bon de travail).

> **Rappel** : aucune action automatique. La météo n'écrit dans aucune table, aucun bon de travail, aucune phase.

### 3.2 Vérifier la fenêtre météo pour couler du béton

1. Sélectionnez la station de la ville du chantier.
2. Repérez les journées où la **température minimale reste au-dessus de 0** sur les 48 à 72 heures suivant la coulée prévue.
3. Si la nuit suivante annonce du gel (min sous 0), prévoyez des couvertures isolantes et un adjuvant antigel — c'est exactement la recommandation « Gel prévu » de l'Impact chantier.
4. En cas de **« Gel sévère »** (min sous -10, sévérité « Critique »), la recommandation est **« Arrêter le coulage de béton. »** — application stricte conseillée.
5. Surveillez aussi les précipitations (évitez plus de 10 mm dans les 24 heures suivant la coulée).

### 3.3 Planifier l'asphaltage et la peinture extérieure

1. **Asphaltage** : choisissez une fenêtre de 3 à 5 jours consécutifs sans pluie (précipitations sous 5 mm chaque jour), idéalement avec une température minimale d'au moins 5.
2. **Peinture extérieure** : privilégiez des journées sans pluie (précip sous 5 mm), avec une température minimale d'au moins 10 et un vent maximal sous 40 km/h.
3. La recommandation « Reporter peinture extérieure » apparaît automatiquement dans l'Impact chantier dès qu'une journée dépasse 10 mm de pluie.
4. Il n'y a pas de logique dédiée à ces métiers dans le code : la lecture du tableau reste humaine. Au besoin, demandez conseil à l'assistant IA (section 3.5).

### 3.4 Anticiper un vent violent (travaux en hauteur)

1. Repérez les badges « Vent fort » (vent supérieur à 40 km/h) sur les 7 cartes.
2. Dans l'Impact chantier, une ligne **« Vents violents »** (au-delà de 70 km/h, « Critique ») porte la recommandation **« ARRÊTER les travaux en hauteur. Descendre grue. Sécuriser tous les matériaux et équipements légers. »**
3. Une ligne **« Vents forts »** (de 50 à 70 km/h, « Attention ») recommande « Sécuriser échafaudages et bannières. Limiter travaux en hauteur. Attacher matériaux légers. »
4. Coordonnez avec le surintendant et le grutier — la décision d'arrêt reste manuelle.

### 3.5 Utiliser l'assistant IA de planification

1. Cliquez sur **« Assistant IA »** dans la barre d'en-tête. La fenêtre s'ouvre en indiquant la station actuellement consultée comme contexte.
2. Saisissez une question de planification (par exemple : « Avec -8 la nuit et 12 le jour, est-ce sécuritaire de décoffrer une dalle demain ? »).
3. Appuyez sur **Entrée** pour envoyer (ou **Maj+Entrée** pour aller à la ligne). L'assistant réfléchit, puis répond avec des conseils prudents adaptés au climat du Québec.
4. Enchaînez les questions : l'assistant tient compte des 12 derniers échanges de la conversation.
5. Chaque réponse affiche son coût en dollars américains : gardez un œil sur votre solde de crédits IA.

> **Ce que l'assistant sait et ne sait pas.** Il **ne consulte pas** vos données internes (ni projets, ni employés, ni prévisions stockées : rien n'est stocké). Il raisonne à partir de la météo que vous décrivez et de ses connaissances générales. Le nom de la station transmis en contexte est traité comme une **simple indication non vérifiée**, jamais comme une mesure officielle. Pour un conseil précis, indiquez vous-même les températures, précipitations et vents dans votre question.

### 3.6 Actualiser, changer de station et gérer les erreurs

1. Le bouton **Actualiser** relance l'appel pour la station courante (utile car les prévisions sont mises en cache 30 minutes côté serveur).
2. Changer de station dans le menu déroulant recharge automatiquement la page.
3. Si Open-Meteo est indisponible : après 10 secondes au maximum, une bande rouge affiche « Service météo temporairement indisponible. Réessayez plus tard. » et la grille reste vide.
4. Solution de contournement : réessayez plus tard, ou consultez Environnement Canada ou MétéoMédia en parallèle pour les décisions critiques.

---

## 4. Référence

### 4.1 Les 7 stations (codées en dur)

Source : `secondary.py:8656-8664`.

| Code | Ville | Latitude | Longitude |
|------|-------|----------|-----------|
| YUL | Montréal | 45.5017 | -73.5673 |
| YQB | Québec | 46.8139 | -71.2080 |
| YOW | Gatineau | 45.4765 | -75.7013 |
| YQT | Trois-Rivières | 46.3432 | -72.5419 |
| YSH | Sherbrooke | 45.4010 | -71.8884 |
| YSB | Saguenay | 48.4279 | -71.0685 |
| YRI | Rimouski | 48.4489 | -68.5243 |

> **Remarque.** Les codes ressemblent à des codes d'aéroport, mais ce ne sont que des étiquettes : les coordonnées pointent vers le centre-ville. Seuls les couples latitude/longitude sont transmis à Open-Meteo. Ce ne sont donc **pas** de vraies stations d'Environnement Canada.

### 4.2 Endpoints (3 au total)

Préfixe de montage : `/api/erp/v1`.

| Méthode | Chemin complet | Fichier:ligne | Authentification | Réponse |
|---------|----------------|---------------|------------------|---------|
| GET | `/api/erp/v1/weather/stations` | `secondary.py:8653` | `get_current_user` | `{ stations: [ {code, name, lat, lon} × 7 ] }` |
| GET | `/api/erp/v1/weather/forecast?lat=X&lon=Y` | `secondary.py:8692` | `get_current_user` | `{ forecasts: [...7], latitude, longitude }` ou `{ forecasts: [], error }` |
| POST | `/api/erp/v1/meteo/ai/chat` | `meteo_ai.py:65` | `get_current_user` + crédits IA | `{ response, input_tokens, output_tokens, cost_usd, credit_balance, elapsed_seconds }` |

Valeurs par défaut de `/weather/forecast` si `lat`/`lon` sont absents : Montréal (`lat=45.5017`, `lon=-73.5673`). Bornes acceptées : latitude de -90 à 90, longitude de -180 à 180.

### 4.3 Format d'une prévision

Réponse du backend (Python, minuscules avec tiret bas) :

```json
{
  "date": "2026-05-02",
  "temp_max": 18.4,
  "temp_min": 5.1,
  "precipitation": 2.3,
  "wind_max": 22.0
}
```

Vue par l'interface (TypeScript, casse chameau après transformation par le client API) : `date`, `tempMax`, `tempMin`, `precipitation`, `windMax`. Chaque valeur numérique peut être nulle ; l'interface affiche alors « -- ». Le backend tronque tous les tableaux au plus court pour ne jamais se bloquer sur des données partielles (`secondary.py:8718`).

### 4.4 Seuils des badges de carte

Source : `MeteoPage.tsx:158-160`. Ces seuils déclenchent le liseré jaune et le badge de la carte.

| Condition | Seuil | Badge | Couleur |
|-----------|-------|-------|---------|
| Froid | température minimale sous 0 | « Gel » | bleu |
| Pluie | précipitations supérieures à 5 mm | « Pluie » | jaune |
| Vent | vent maximal supérieur à 40 km/h | « Vent fort » | rouge |

Un seul badge est affiché, par priorité **Gel > Vent fort > Pluie**.

### 4.5 Seuils et recommandations de l'Impact chantier

Source : `MeteoPage.tsx:218-232` et `terrain.json:42-53`. Tous les seuils sont codés en dur.

| Type | Seuil déclencheur | Sévérité | Message | Recommandation |
|------|-------------------|----------|---------|----------------|
| Gel | min sous -10 | **Critique** | Gel sévère (X °C) | Arrêter le coulage de béton. Protéger les canalisations. Prévoir chauffage des zones de travail. |
| Gel | min sous 0 (et -10 ou plus) | Attention | Gel prévu (X °C) | Protéger le béton frais avec couvertures isolantes. Utiliser additifs antigel. Vérifier protection des tuyaux. |
| Pluie | précip supérieures à 20 mm | **Critique** | Fortes précipitations (X mm) | Reporter les travaux extérieurs. Sécuriser les excavations contre inondation. Vérifier les pompes de drainage. |
| Pluie | précip supérieures à 10 mm (et 20 ou moins) | Attention | Pluie importante (X mm) | Protéger les matériaux sensibles à l'humidité. Prévoir bâches pour zones de travail. Reporter peinture extérieure. |
| Vent | vent supérieur à 70 km/h | **Critique** | Vents violents (X km/h) | ARRÊTER les travaux en hauteur. Descendre grue. Sécuriser tous les matériaux et équipements légers. |
| Vent | vent supérieur à 50 km/h (et 70 ou moins) | Attention | Vents forts (X km/h) | Sécuriser échafaudages et bannières. Limiter travaux en hauteur. Attacher matériaux légers. |

> **Comparaison des deux jeux de seuils.** Les cartes (badges) réagissent plus tôt que l'Impact chantier :
> - Carte : gel sous 0, pluie au-delà de 5 mm, vent au-delà de 40 km/h.
> - Impact chantier : gel sous 0 ou sous -10, pluie au-delà de 10 mm ou 20 mm, vent au-delà de 50 km/h ou 70 km/h.
>
> Une carte peut donc porter un badge « Pluie » (par exemple 6 mm) sans générer de ligne dans l'Impact chantier.

### 4.6 Assistant IA — paramètres, gardes et coût

Source : `meteo_ai.py`.

| Aspect | Détail |
|--------|--------|
| Modèle | `claude-sonnet-4-6` |
| Longueur maximale de réponse | 4000 jetons |
| Base de données / outils | **aucun** : un seul appel au modèle, sans accès aux données du tenant |
| Longueur du message | de 1 à 8000 caractères |
| Historique conservé | les 12 derniers tours (rôles « utilisateur » et « assistant »), tronqués à 8000 caractères chacun |
| Contexte météo | au plus 4000 caractères, traité comme une **indication non fiable** (protection contre l'injection) |
| Langue | français du Québec par défaut, anglais si la langue commence par « en » (directive répétée en tête et en fin d'invite) |

**Gardes, dans l'ordre** (`meteo_ai.py:69-80`) :

1. Service IA non configuré → **503** « Service IA non disponible ».
2. Aucun contexte de tenant → **400** « Contexte tenant manquant ».
3. Contrôle d'accès IA (`check_ai_guard`) → **403** si refusé. En pratique, ce contrôle laisse passer tout utilisateur authentifié après quelques exemptions : le vrai gardien est le solde de crédits.
4. Crédits épuisés → **402** « Credits IA epuises. Veuillez recharger votre solde pour continuer. ».

**Coût et facturation** : `coût = (jetons_entrée × 0,003 + jetons_sortie × 0,015) ÷ 1000 × 1,30`. Autrement dit, le tarif du modèle (3 $ US par million de jetons en entrée, 15 $ US par million en sortie) **majoré de 30 %**. Le montant est débité des crédits prépayés (`public.ai_prepaid_credits`, produit « ERP ») et tracé sous la fonctionnalité `meteo_chat` dans `public.ai_usage_tracking`. À titre indicatif, une réponse typique de 500 jetons en entrée et 600 en sortie coûte environ 0,0137 $ US.

**Erreurs** (`meteo_ai.py:157-164`) : une surcharge ou un délai côté modèle renvoie **503** « Service IA temporairement indisponible. » ; toute autre erreur renvoie **500** « Erreur interne de l'assistant météo. ».

### 4.7 Limites de débit (par adresse IP)

Réf. `erp_api.py`.

| Endpoint | Limite | Portée |
|----------|--------|--------|
| `POST /meteo/ai/chat` | 20 par minute | ERP |
| `GET /weather/forecast` | 40 par minute | **partagée** entre l'ERP et l'application mobile (même sortie vers Open-Meteo, quota commun) |
| `GET /weather/stations` | aucune limite dédiée | liste codée en dur, aucun appel externe |

### 4.8 Service externe Open-Meteo

- **URL** : `https://api.open-meteo.com/v1/forecast`
- **Paramètres** : `latitude`, `longitude`, `daily=temperature_2m_max,temperature_2m_min,precipitation_sum,wind_speed_10m_max`, `timezone=America/Montreal`, `forecast_days=7`.
- **Authentification** : aucune (API publique gratuite).
- **Cache serveur** : 30 minutes, jusqu'à 128 entrées, clé = latitude/longitude arrondies à 2 décimales (les 7 stations occupent donc 7 entrées). Seuls les résultats réussis et non vides sont mis en cache. L'appel bloquant est exécuté hors de la boucle d'évènements pour ne pas geler le serveur partagé (`secondary.py:8708`).
- **Comportement en cas d'échec** (réponse HTTP 200 malgré tout) : erreur HTTP, réseau ou délai dépassé → `{ forecasts: [], error: "Service meteo temporairement indisponible" }` ; erreur imprévue → `{ forecasts: [], error: "Erreur interne" }`.
- **Documentation officielle** : `https://open-meteo.com/en/docs`.

### 4.9 Textes de l'interface (internationalisation)

- L'ensemble des fichiers de langue est fusionné dans un **espace de noms unique**, chaque fichier étant indexé par son nom.
- Les libellés de l'écran météo vivent donc sous **`terrain.meteo.*`** (sous-section `meteo` du fichier `terrain.json`, 31 clés) — et **non** dans un fichier `meteo.json` dédié. Le fichier `terrain.json` est en réalité l'ensemble **partagé** des modules secondaires (Météo, Conformité, Subventions, Immobilier, Location, Maintenance, Logistique).
- Les libellés de l'assistant vivent sous **`meteoAssistant.*`** (fichier `meteoAssistant.json`, 13 clés). L'interface est bilingue (français/anglais).
- L'entrée de menu et le fil d'Ariane utilisent `nav.meteo` et `breadcrumb.meteo` = « Météo ».

### 4.10 Raccourcis clavier (assistant IA)

| Touche | Effet |
|--------|-------|
| Entrée | Envoyer le message |
| Maj + Entrée | Insérer un saut de ligne |

---

## 5. Intégrations et FAQ

### 5.1 Projets, phases, bons de travail, suivi Gantt

> **Aucune intégration directe.** Le module Météo est un **silo indépendant** :

- Aucune clé étrangère `projet_id`, `phase_id` ou `bt_id` n'existe dans ce module.
- Aucun bouton « Voir la météo de ce projet » dans les modules Projets ou Suivi.
- Un bon de travail planifié une journée de gel sévère **n'est pas** marqué automatiquement : le surintendant reporte ou suspend manuellement.
- Pour relier mentalement une météo à un chantier, vous retenez vous-même la ville et sélectionnez la station correspondante.

### 5.2 Application mobile (météo au pointage)

La seule météo réellement **liée à un lieu de chantier** existe dans l'application mobile de pointage, pas dans cet écran ERP :

- Au pointage d'entrée et de sortie, l'application mobile capture un instantané météo à partir de l'adresse du chantier et le stocke sur l'enregistrement de temps (`time_entries`).
- L'application mobile expose aussi ses propres endpoints `weather/stations` et `weather/forecast` (mêmes 7 stations, même API Open-Meteo).
- Cette météo mobile est rattachée aux **pointages**, jamais à des projets. Elle est indépendante de la page ERP `/meteo`.

### 5.3 Le module Terrain/Cadastre est sans rapport

Le fichier de code `terrain.py` porte le même préfixe que le nom du regroupement de langue, mais il **n'a aucun lien avec la météo** : c'est un module de **cadastre** (fiche de terrain par numéro de lot, zonage, contraintes environnementales, proximité du patrimoine). Il est actuellement **dormant** (aucun écran ne l'utilise). Les endpoints météo, eux, vivent dans `secondary.py` et `meteo_ai.py`. De même, le module GPS concerne le suivi des **véhicules**, pas la météo.

### 5.4 Crédits IA et facturation

- L'assistant météo partage le même **portefeuille de crédits IA** que les autres fonctionnalités d'intelligence artificielle de l'ERP (produit « ERP »).
- Chaque message est tracé sous la fonctionnalité `meteo_chat`, visible dans le suivi d'usage IA (onglet Usage IA du super-administrateur).
- La consultation météo (cartes et Impact chantier) reste **entièrement gratuite** : seuls les messages du chat sont facturés.

### 5.5 CNESST et sécurité

Bien que les seuils de vent (au-delà de 50 puis de 70 km/h) recoupent les bonnes pratiques en matière de travaux en hauteur, le module **n'intègre pas** les obligations légales et ne produit **aucun** arrêt de travaux conforme. Les recommandations sont indicatives. Toute décision d'arrêt de chantier doit être validée par le responsable, conformément aux exigences de la CNESST et aux lois et règlements applicables.

### 5.6 Foire aux questions

**Puis-je ajouter ma propre ville (Drummondville, Joliette, etc.) ?**
Non, pas depuis l'interface. Les 7 stations sont codées en dur (`secondary.py:8656-8664`). Contournement : choisissez la station la plus proche (par exemple Trois-Rivières pour Drummondville).

**Puis-je voir plus de 7 jours ou une prévision horaire ?**
Non. L'horizon est fixé à 7 jours et seules les valeurs journalières sont demandées. Pour de l'horaire, consultez Environnement Canada ou MétéoMédia directement.

**Le module garde-t-il un historique des prévisions consultées ?**
Non. Aucune persistance : chaque ouverture rappelle Open-Meteo en direct (avec un cache serveur de 30 minutes). Pour archiver, faites une capture d'écran.

**Que se passe-t-il si Open-Meteo est en panne ?**
Après 10 secondes au maximum, la page affiche « Service météo temporairement indisponible. Réessayez plus tard. » et la grille reste vide. Réessayez plus tard.

**Les seuils de gel, de pluie et de vent sont-ils configurables ?**
Non. Ils sont codés en dur dans l'interface (`MeteoPage.tsx`) et identiques pour tous les tenants.

**Le module envoie-t-il des courriels ou bloque-t-il des bons de travail en cas d'alerte critique ?**
Non. Aucun système de notification poussée, aucune tâche planifiée, aucun webhook, aucune écriture vers d'autres tables. Les recommandations sont textuelles ; le surintendant agit manuellement.

**Puis-je comparer deux villes côte à côte ?**
Non. Une seule station est affichée à la fois. Contournement : ouvrez deux onglets de navigateur.

**Y a-t-il un risque d'épuiser le quota Open-Meteo ?**
Faible. Le cache serveur de 30 minutes réduit fortement les appels, et Open-Meteo tolère environ 10 000 requêtes par jour et par adresse IP. La limite de débit de `/weather/forecast` est toutefois **partagée** avec l'application mobile.

**L'assistant IA connaît-il ma météo affichée et mes projets ?**
Il reçoit uniquement le **nom de la station** consultée, comme simple indication non vérifiée. Il n'a accès à **aucune** donnée du tenant (ni projets, ni prévisions stockées). Pour un conseil précis, indiquez vous-même les conditions (températures, précipitations, vent) dans votre question.

**Combien coûte une question à l'assistant ?**
Le tarif du modèle `claude-sonnet-4-6` majoré de 30 %, débité de vos crédits IA. Une réponse typique coûte quelques centièmes de dollar américain. Le coût exact est affiché sous chaque réponse.

**Open-Meteo est-il assez fiable pour décider en construction ?**
Open-Meteo agrège plusieurs modèles. La précision est raisonnable à 3-5 jours et plus incertaine à 7 jours. Pour une décision critique (coulée majeure, levage de grue), croisez toujours avec Environnement Canada et un responsable de terrain.

**Les unités sont-elles métriques ?**
Oui : degrés Celsius, millimètres, kilomètres par heure.

---

## 6. Récapitulatif

- **Objet** : consultation des prévisions sur 7 jours pour 7 villes du Québec, avec recommandations chantier automatiques et un assistant IA de planification.
- **Accès** : barre latérale → groupe **TERRAIN** → **Météo** (icône `CloudSun`), route `/meteo`. Authentification requise, aucun rôle particulier.
- **3 endpoints** : `GET /weather/stations`, `GET /weather/forecast` (dans `secondary.py`), `POST /meteo/ai/chat` (dans `meteo_ai.py`). **Aucune table de tenant.**
- **Source des données** : Open-Meteo (gratuit, sans clé), cache serveur de 30 minutes. Ce **n'est pas** Environnement Canada.
- **7 stations** : Montréal, Québec, Gatineau, Trois-Rivières, Sherbrooke, Saguenay, Rimouski (coordonnées de centre-ville codées en dur).
- **4 mesures par jour** : température maximale, température minimale, précipitations, vent maximal. Prévision journalière seulement.
- **3 badges de carte** : « Gel » (min sous 0), « Pluie » (précip au-delà de 5 mm), « Vent fort » (vent au-delà de 40 km/h).
- **6 recommandations Impact chantier** : Gel sévère, Gel prévu, Fortes précipitations, Pluie importante, Vents violents, Vents forts — calculées côté client, cumulables (jusqu'à 3 lignes par jour).
- **Assistant IA** : chat de planification `claude-sonnet-4-6`, **sans base de données ni outil**, 4000 jetons au maximum, facturé aux crédits IA (tarif du modèle majoré de 30 %, fonctionnalité `meteo_chat`).
- **Ce qu'il ne fait pas** : aucun lien projet/chantier/BT, aucune géolocalisation, aucun ajout de ville, aucune notification poussée, aucun historique, aucune exportation, aucun seuil configurable.
- **En cas d'échec** : message « Service météo temporairement indisponible. Réessayez plus tard. », grille vide.

---

**Documentation générée à partir du code** : `ERP_REACT/backend/routers/secondary.py` (8653-8744, endpoints météo + cache) ; `ERP_REACT/backend/routers/meteo_ai.py` (165 lignes, assistant IA) ; `ERP_REACT/frontend/src/pages/MeteoPage.tsx` (281 lignes) ; `ERP_REACT/frontend/src/components/meteo/MeteoAssistantTab.tsx` (122 lignes) ; `ERP_REACT/frontend/src/components/ai/MessageBubble.tsx` ; `ERP_REACT/frontend/src/api/secondary.ts` (47-53) ; `ERP_REACT/frontend/src/api/meteoAi.ts` (25-37) ; textes sous `i18n/locales/fr/terrain.json` (`meteo.*`) et `i18n/locales/fr/meteoAssistant.json`.

**Manuels liés** :
- Module 08 — Projets (référence géographique du chantier, sans lien technique) — `08-ventes-projets.md`
- Module Suivi / Gantt (phases de construction, sans intégration météo) — `02-suivi-gantt.md`
- Module 11 — Bons de travail (à suspendre manuellement en cas d'alerte critique) — `11-operations-bons-de-travail.md`
- Module 24 — Assistant IA (portefeuille de crédits partagé avec l'assistant météo) — `24-communication-assistant-ia.md`
