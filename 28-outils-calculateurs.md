# Module 28 — Calculateurs de construction

> **Version** : 3.0 (refonte vérifiée contre le code source, juillet 2026)
> **Code de référence** : `frontend/src/pages/CalculateursPage.tsx` (2 269 lignes, **5 onglets**, **15 tuiles**), `frontend/src/components/calculateurs/MasterProCalculatorBody.tsx` (903 lignes, calculatrice de chantier), `frontend/src/components/calculateurs/MursParametriquePanel.tsx` (2 676 lignes, charpente murale), `frontend/src/api/calculators.ts` (1 642 lignes), `backend/routers/calculators.py` (3 681 lignes, **59 points d'entrée**, préfixe réel `/api/erp/v1/calculators`), `backend/routers/calculators_data.py` (807 lignes — tables de constantes, normes, prix, invite système de l'IA ; **module de données, PAS un routeur**)
> **Table PostgreSQL (par tenant)** : `calculator_history` (créée à la demande au premier appel — une seule table)
> **Cadrage** : suite de **13 calculateurs de construction calculés côté serveur** (béton, escaliers, analyse structurale, toiture, peinture, électricité, plomberie, CVAC, soudure, pliage, poids métal, taxes, paie) + **2 outils calculés côté navigateur** (la **Calculatrice** de chantier et les **Murs paramétrique**) = **15 tuiles**, plus un **Assistant IA Claude** et un **historique** persistant par tenant. Ce module est une **aide à l'estimation et au pré-dimensionnement** : ce n'est **PAS** un logiciel de calcul de structure officiel, **PAS** un BIM, **PAS** une CAO (voir le module *Modélisation 3D / DAO*), et il ne remplace **ni** le sceau d'un ingénieur, **ni** la paie réelle du module *Pointage et Paie*.

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

Donner aux estimateurs, contremaîtres, entrepreneurs et gens de métier une **boîte à outils de calcul professionnelle** adaptée au Québec, couvrant les principaux corps de métier (structure, enveloppe, mécanique, métal) et les calculs financiers (taxes, charges de paie). Chaque calculateur :

- encode des **formules normées** (CSA, CNB, CCQ, CCE, CNP, ASHRAE, AWS, IIW, Blondel, Hazen-Williams, Magnus) ;
- retourne des **quantités de matériaux**, des **coûts indicatifs en CAD** et des **verdicts de conformité** (badge *Conforme* / *Non conforme*) ;
- peut être complété par l'**Assistant IA** (Claude) pour l'explication, la recommandation et le diagnostic.

### 1.2 Les deux natures de calcul

Le module mélange deux familles d'outils. La distinction est importante pour l'utilisateur, car elle explique le comportement hors ligne, la facturation IA et la sauvegarde.

| Nature | Où le calcul est fait | Tuiles concernées | Connexion requise |
|---|---|---|---|
| **Côté serveur** (backend) | FastAPI Python, sur les serveurs Constructo | Les **13** calculateurs de construction (voir 1.3) | Oui — jeton de session valide |
| **Côté navigateur** (client) | Directement dans votre navigateur, en JavaScript | La **Calculatrice** (`master-pro`) et les **Murs paramétrique** (`murs-parametrique`) | Le calcul lui-même est local ; la sauvegarde de projet de murs passe par le serveur |

> Les deux outils côté navigateur **n'ont aucun point d'entrée côté serveur** et sont **exclus de l'Assistant IA** (l'IA n'a pas de contexte pour eux). C'est voulu : ce sont des calculatrices instantanées.

### 1.3 Inventaire des 15 tuiles (6 catégories)

Source : `CALC_DEFS` (`CalculateursPage.tsx:52-68`) et `CALCULATEURS_LISTE` (`calculators_data.py:694-708`).

| # | Catégorie | Identifiant | Nom affiché | Calcul | Norme principale |
|---|-----------|-------------|-------------|--------|------------------|
| 1 | Polyvalent | `master-pro` | **Calculatrice** | Navigateur | Calcul pieds-pouces (style Construction Master Pro) |
| 2 | Structure | `concrete` | **Béton** | Serveur | CSA A23.1 / ACI 209 / CNESST |
| 3 | Structure | `stairs` | **Escaliers** | Serveur | CCQ 9.8 / 3.4 / Blondel |
| 4 | Structure | `murs-parametrique` | **Murs paramétrique (En construction)** | Navigateur | Charpente murale légère (impérial) |
| 5 | Structure | `charge-tributaire-complete` | **Analyse structurale** | Serveur | CNB / CSA O86 |
| 6 | Enveloppe | `roofing` | **Toiture** | Serveur | CCQ 9.19 / CNB 4.1.6.2 |
| 7 | Enveloppe | `painting` | **Peinture** | Serveur | Couverture / DFT / point de rosée (Magnus) |
| 8 | Mécanique | `electrical` | **Électricité** | Serveur | CCE 4-004 / 8-200 |
| 9 | Mécanique | `plumbing` | **Plomberie** | Serveur | CNP / Hazen-Williams |
| 10 | Mécanique | `hvac` | **CVAC** | Serveur | ASHRAE (U·A·ΔT) / 62.2 |
| 11 | Métal | `welding` | **Soudure** | Serveur | CSA W47.1 / W59 / AWS D1.1 / IIW |
| 12 | Métal | `bending` | **Pliage métal** | Serveur | Facteur K / pliage à l'air |
| 13 | Métal | `metal-weight` | **Poids métal** | Serveur | Densités + profilés AISC W/C |
| 14 | Finances | `taxes` | **Taxes Québec** | Serveur | TPS 5 % + TVQ 9,975 % |
| 15 | Finances | `charge-tributaire` | **Paie employé** | Serveur | RRQ / RQAP / AE / CNESST / FSS / CCQ |

> **Résumé des chiffres** : **15 tuiles** = **13** calculées côté serveur + **2** côté navigateur. Le tableau de bord affiche « 15 » (il compte les tuiles), tandis que la documentation technique interne parle parfois de « 13 calculateurs » (elle compte les familles côté serveur). Les deux sont exacts, ils ne comptent simplement pas la même chose.

### 1.4 Accès

- Menu latéral -> **Outils** -> **Calculateurs** (icône calculatrice).
- Adresse : `/calculateurs`.
- Onglet ouvert par défaut : **Tableau de bord**.
- 5 onglets globaux (voir la [section 2](#2-interface)).

### 1.5 Permissions et rôles

- **Aucun rôle particulier n'est exigé** pour lancer un calcul. Tout utilisateur authentifié du tenant (compte actif) peut utiliser n'importe quel calculateur. Il n'y a pas de rôle « ingénieur » ni « estimateur », et pas de restriction de mode consultation sur les calculs (ce sont des calculs purs qui ne touchent pas la base de données).
- L'**historique** (`calculator_history`) est **partagé au niveau du tenant** : tous les utilisateurs d'une même entreprise voient et gèrent les mêmes calculs sauvegardés. Il exige un contexte de tenant valide (sinon *erreur 400 — contexte tenant manquant*).
- L'**Assistant IA** ajoute une garde supplémentaire : il faut que le service IA soit disponible et que le tenant dispose de **crédits IA prépayés** (voir 1.6).

### 1.6 Modèle IA et tarification

L'Assistant IA appelle le grand modèle de langage Claude d'Anthropic. Les paramètres réels du code :

| Élément | Valeur (source `calculators.py:121-132`) |
|---|---|
| Modèle | `claude-opus-4-8` |
| Jetons de réponse maximum | 32 000 |
| Prix d'entrée | 5 $ US / million de jetons |
| Prix de sortie | 25 $ US / million de jetons |
| Écriture de cache | 6,25 $ US / million de jetons |
| Lecture de cache | 0,50 $ US / million de jetons |
| Majoration facturée au tenant | × 1,30 |

Chaque message d'IA est **débité des crédits IA prépayés** du tenant (le même portefeuille que l'Assistant IA général et l'Estimation IA), au **coût réel × 1,30**. Détails de facturation :

- Le débit se fait **après** la réponse. Si vous **quittez l'onglet IA pendant que Claude répond**, la requête est **annulée** et **aucun crédit n'est débité** (le code détecte la déconnexion — `abortAi`, `CalculateursPage.tsx:2039`).
- Un compte **super-administrateur** n'est jamais facturé.
- Si les crédits sont épuisés : *erreur 402*. Si l'IA est désactivée pour le tenant : *erreur 403*. Si le service Claude est indisponible : *erreur 503*.

> **Note de version** : certains commentaires internes du code mentionnent encore « Claude Opus 4.7 » ou « 4.6 ». C'est une trace de documentation périmée : le modèle réellement exécuté est **`claude-opus-4-8`** avec **32 000** jetons de réponse. Le présent manuel décrit le comportement réel.

---

## 2. Interface

### 2.1 Les 5 onglets globaux

Source : `CalculateursPage.tsx:725-736`.

| # | Onglet | Icône | Contenu |
|---|--------|-------|---------|
| 1 | **Tableau de bord** | Graphique | 4 indicateurs (KPI) + grilles de tuiles par catégorie |
| 2 | **Calculateurs** | Calculatrice | Barre latérale des 15 tuiles + panneau du calculateur choisi |
| 3 | **Analyse structurale** | Règle | Vérification de poutre/linteau (CNB/CSA O86) + diagramme |
| 4 | **Assistant IA** | Étincelles | 4 sous-outils IA (chat, recommandations, expliquer une norme, diagnostic) |
| 5 | **Historique** | Horloge | Liste des calculs sauvegardés + statistiques |

En haut de la page : le titre **« Calculateurs »** et le sous-titre **« Outils de calcul pour la construction »**. Deux bandeaux d'état (rouge pour les erreurs, vert pour les succès) apparaissent sous le titre et sont refermables.

> **Il n'y a pas d'onglet « Conversions » distinct.** Une clé de traduction « Conversions » existe dans le code, mais l'onglet n'est **pas monté** dans la barre. Les conversions d'unités se font dans la **Calculatrice** (onglet interne *Conversions* et feuille de conversion — voir 2.4).

### 2.2 Onglet « Tableau de bord »

Quatre cartes d'indicateurs en haut (`CalculateursPage.tsx:756-761`) :

| Indicateur | Valeur affichée |
|---|---|
| **Calculateurs** | **15** (nombre de tuiles) |
| **Calculs sauvés** | Nombre réel de calculs dans votre historique |
| **Normes Québec** | **« 10+ »** (valeur fixe, indicative) |
| **IA Claude** | **« 4 outils »** (reflète les 4 sous-outils de l'onglet IA) |

Sous les indicateurs, **6 sections** (une par catégorie : Polyvalent, Structure, Enveloppe, Mécanique, Métal, Finances) présentent chacune leurs tuiles sous forme de boutons (icône colorée + nom + courte description + chevron). **Cliquer sur une tuile** efface les résultats précédents et bascule sur l'onglet **Calculateurs** avec ce calculateur déjà ouvert.

### 2.3 Onglet « Calculateurs »

Disposition en **barre latérale + panneau**.

- **Sur ordinateur** : une barre latérale collante (à gauche) liste les 15 tuiles ; le panneau de droite affiche le calculateur choisi.
- **Sur mobile** : un bouton **« Choisir un calculateur »** ouvre un tiroir plein écran avec la liste.
- **Sans sélection** : une carte vide invite à « Sélectionnez un calculateur dans la liste à gauche ».

**Patron commun à tous les panneaux côté serveur** : une rangée de **sous-onglets** (pilules) en haut, une **carte de gauche** avec le formulaire (chaque champ = une étiquette + une zone de saisie ou un menu déroulant), un bouton **« Calculer »** (avec indicateur d'attente), et une **carte de droite « Résultats »** qui affiche les valeurs (certaines mises en évidence) et les **badges de conformité** *Conforme* / *Non conforme*. Changer de calculateur **efface** automatiquement les résultats affichés (pour éviter d'afficher un ancien résultat qui ne correspond plus).

Deux tuiles échappent à ce patron : la **Calculatrice** (2.4) et les **Murs paramétrique** (2.14), qui ont leur propre interface complète. La tuile **Analyse structurale** ouvre le même écran que l'onglet du même nom (2.5) — c'est un double accès vers le même outil.

Le détail des sous-onglets et champs de chacun des 13 calculateurs côté serveur est donné à la [section 3](#3-workflows-pas-à-pas).

### 2.4 La Calculatrice (`master-pro`) — côté navigateur

Réplique fidèle d'une calculatrice de chantier physique de type **Construction Master Pro**, partagée avec le module Métré. Calcul **100 % local** (aucun appel serveur, fonctionne même hors connexion une fois la page chargée). Aucune donnée n'est envoyée à Claude.

- **Deux onglets internes** : **Charpente** et **Conversions**.
- **Afficheur** structuré en pieds-pouces-seizièmes (repères FEET / INCH, fractions au 1/16) ou en décimal, avec indicateurs d'état (CONV, mémoire, Rise/Run/Pitch, L/L/H).
- **Grille de fonctions (20 boutons)** avec un libellé principal et un libellé « secondaire » (accessible par la touche *Conv/Shift*) : *Pitch, Rise, Run, Diag, Hip-V ; Miter, Stair, Arc, Circ, Jack ; Length, Width, Height, % ; Yds, Feet, Inch, Clear*.
- **Pavé numérique (20 boutons)** : bascule *Conv*, chiffres avec conversions en fonction secondaire (cm, pied-planche « Bd Ft », mm, lb, montants « Studs », tonnes, kg, acre, tonnes métriques, coût, degrés-minutes-secondes), mémoire (Store / Rcl / M+) et opérateurs ÷ × − = +.
- **Panneau de détail contextuel** : selon la dernière fonction, affiche par exemple les contremarches et le giron d'un escalier (avec vérification Blondel), les cordes d'un cercle, un arc, un polygone, un onglet/biseau, les montants espacés à 40 ou 60 cm.
- **Pied de la calculatrice** (4 boutons) : *FT/IN* (feuille de saisie pieds-pouces avec valeurs prédéfinies), *Conversion* (feuille m / cm / mm / pi / po / vg), *Historique* (avec compteur), *Tout effacer*.
- **Clavier physique pris en charge** : chiffres, opérateurs, *Entrée* (=), *Échap* (tout effacer), *f/i/y* pour les unités, *c* pour la fonction secondaire.
- **Aucune sauvegarde serveur** : l'historique de la calculatrice vit dans la session en cours seulement.

### 2.5 Onglet « Analyse structurale »

Vérification d'un élément en flexion (poutre ou linteau) selon les combinaisons de charges du CNB et les résistances du CSA O86. Accessible **soit** par l'onglet global, **soit** par la tuile « Analyse structurale » (les deux ouvrent le même écran).

**Champs de saisie** :

| Champ | Détail |
|---|---|
| Type d'élément | **Poutre** ou **Linteau** uniquement |
| Type de matériau | **Bois dimensionnel** (SPF) ou **LVL** |
| Section | Chargée depuis le serveur selon le matériau (ex. `2x10`) |
| Nombre de plis (*ply*) | 1 à 6 |
| Portée | en mm |
| Largeur tributaire | en m |
| Charge morte / vive / neige | en kPa |
| Utilisation | Plancher (flèche L/360) / Toit (L/180) / Linteau (L/360) |

Bouton **« Analyser »**. **Résultats** : un verdict (conforme / non conforme), un **diagramme** de la poutre, le moment maximum (M max), l'effort tranchant maximum (V max), la flèche (Δ), les résistances pondérées Mr et Vr (CSA O86), la charge pondérée à l'état limite ultime, et **3 vérifications** en pourcentage avec coche verte ou croix rouge : **Flexion**, **Cisaillement**, **Flèche**.

> **L'option « Colonne » n'est pas offerte.** Le menu ne propose que Poutre et Linteau. Une analyse de colonne (compression axiale, flambement) **n'est pas implémentée** et est refusée côté serveur. Voir les [limites](#44-limites-et-simplifications-connues).

### 2.6 Onglet « Assistant IA »

Quatre sous-outils (pilules). Chaque appel consomme des crédits IA (voir 1.6).

| Sous-outil | Ce que vous fournissez | Ce que l'IA renvoie |
|---|---|---|
| **Chat Expert** | Un calculateur (facultatif) + votre question | Réponse en français, citant les normes ; historique Vous / Assistant ; bouton *Effacer* |
| **Recommandations** | Un calculateur (parmi les 13 éligibles) + un objectif + des contraintes (facultatif) | Approche, étapes, coûts estimés |
| **Expliquer une norme** | Une norme ou un article (ex. « CCQ 9.8 », « CSA A23.1 », « CCE 8-200 ») | Titre officiel, organisme, exigences principales, note |
| **Diagnostic** | Un calculateur + le problème + les symptômes | Diagnostic, urgence, causes probables, recommandation d'intervention professionnelle |

> **Deux fonctions IA existent côté serveur mais n'ont pas de bouton dans l'interface** : *Analyser* (noter un calcul de 0 à 100) et *Optimiser* (suggérer des économies). Elles ne sont donc pas utilisables depuis l'écran actuel — seuls les 4 sous-outils ci-dessus sont exposés. C'est pourquoi l'indicateur du tableau de bord affiche « 4 outils ».

### 2.7 Onglet « Historique »

Trois statistiques en haut (**Total des calculs**, **Calculateurs utilisés**, **30 derniers jours**), un filtre par calculateur, un bouton **« Effacer tout »** (avec confirmation) et la liste des calculs sauvegardés. Chaque ligne montre l'icône, le libellé et la date (format `fr-CA`), un bouton **« Détails »** (déplie les entrées et les résultats en clair) et une corbeille pour supprimer une entrée.

> **Ce que l'historique contient réellement.** Dans l'interface actuelle, **seuls les Murs paramétrique écrivent dans l'historique** (c'est ainsi que leurs projets de mur sont sauvegardés et rechargés). Les 13 calculateurs côté serveur **n'ont pas de bouton « Sauvegarder »** câblé : leurs résultats sont éphémères et **ne remplissent pas** l'historique automatiquement. L'historique reste donc surtout un journal de vos projets de murs (plus, éventuellement, des entrées créées par intégration). La persistance elle-même est bien réelle et par tenant (table `calculator_history`).

### 2.8 Composants présents mais non branchés

Le dossier `components/calculateurs/` contient **7 composants construits mais non reliés** à une page (aucun écran ne les ouvre) : un panneau de toiture avancé, un panneau de plancher, un panneau de revêtement, un éditeur de plan d'étage, une fenêtre de rapports, un sélecteur de niveaux de maison et un gestionnaire de calques. Ils semblent être les restes d'un **concepteur de maison multi-niveaux** partiellement intégré. **Ils ne sont pas accessibles** aujourd'hui et n'ont aucun effet. Le seul calculateur de toiture réellement actif est le panneau **Toiture** décrit en 3.7 (côté serveur).

---

## 3. Workflows pas à pas

Chaque calculateur ci-dessous indique ses **sous-onglets**, ses **entrées** principales, la **formule/norme** appliquée et ses **sorties**. Le patron est toujours le même : remplir la carte de gauche -> **Calculer** -> lire la carte de droite.

### 3.1 Estimer le béton d'une dalle (calculateur Béton)

Le calculateur **Béton** a **7 sous-onglets** : *Volume · Dosage CSA · Armature · Cure ACI 209 · Excavation · Talus CNESST · Escalier béton*.

**Exemple — dalle de garage 6 m × 8 m, 100 mm d'épaisseur :**

1. Ouvrir **Calculateurs** -> **Béton** -> sous-onglet **Volume**.
2. Saisir longueur `6`, largeur `8`, épaisseur `0.1` (en mètres), perte `10` %, et choisir la **classe de béton** (ex. *C-2 (25 MPa extérieur)*).
3. Cliquer sur **Calculer**.
4. Lire à droite : volume total (m³), surface, ciment / sable / gravier / eau, **nombre de sacs de 30 kg**, et le nombre de feuilles de coffrage 4×8.

**Autres sous-onglets :**

- **Dosage CSA A23.1** : volume + résistance visée (15 à 40 MPa) -> quantités de matériaux, ratio de mélange, rapport eau/ciment (E/C), sacs de 30 et 40 kg. Le dosage dépend de la **classe d'exposition** (le code fait le lien classe -> résistance -> dosage).
- **Armature (CSA G30.18)** : dimensions, enrobage, espacement, type de barre (10M à 55M), nombre de lits -> nombre de barres dans chaque direction, longueur totale, découpe en barres standard de 6 m, masse totale (kg et lb). *Garde-fou* : si la dalle est plus mince que deux fois l'enrobage, le calcul est refusé (dimensions effectives négatives).
- **Cure (ACI 209)** : résistance finale, âge, température, type de ciment (GU / HE / MS / HS) -> résistance atteinte à l'âge donné, pourcentage de la résistance finale, facteur de maturité, durée de cure minimale. Formule `f(t) = f28 · t / (a + b·t)` avec un facteur de maturité selon la température.
- **Excavation** : dimensions + type de sol (facteur de foisonnement) -> volume compact, volume foisonné (m³ et vg³), nombre de camions de 12 vg³, poids estimé.
- **Talus CNESST** : profondeur + type de sol -> angle de talus, ratio horizontal/vertical, distance de dégagement horizontale, et la **liste des exigences CNESST** (inspection quotidienne au-delà de 1,2 m, analyse d'ingénieur recommandée au-delà de 3 m, obligatoire au-delà de 6 m).
- **Escalier béton (Blondel)** : hauteur totale, largeur, épaisseur de dalle, giron et hauteur de marche cibles -> badge Blondel *OK / Hors*, nombre de marches, valeur `2R + G`, volume et matériaux.

### 3.2 Dimensionner un escalier (calculateur Escaliers)

**2 sous-onglets** : *Dimensions CCQ · Garde-corps*.

1. **Dimensions (CCQ 9.8 / 3.4)** : saisir la hauteur totale à franchir, le giron et la hauteur de marche visés, la largeur, et l'**usage** (*Résidentiel — CCQ 9.8* ou *Commercial — CCQ 3.4*). **Calculer** -> badge de conformité CCQ (avec l'article de référence), nombre de marches, giron réel, `2R + G`, pente en degrés, ligne de foulée, et une **évaluation du confort**.
2. **Garde-corps (CCQ 9.8.7)** : longueur, hauteur, espacement des barreaux, usage -> deux badges (Hauteur et Barreaux), nombre de barreaux, nombre de poteaux, longueur de la main courante.

### 3.3 Vérifier une poutre ou un linteau (Analyse structurale)

1. Onglet **Analyse structurale** (ou la tuile du même nom).
2. Choisir le **type d'élément** (Poutre ou Linteau) et le **matériau** (Bois dimensionnel SPF ou LVL) ; la liste des **sections** se remplit alors depuis le serveur.
3. Choisir la section, le nombre de plis, saisir la portée (mm) et la largeur tributaire (m).
4. Saisir les charges **morte**, **vive** et **neige** (kPa) et l'**utilisation** (Plancher / Toit / Linteau — qui fixe la limite de flèche).
5. Cliquer sur **Analyser**.
6. Lire le verdict, le diagramme, M max / V max / flèche, Mr / Vr et les 3 vérifications (flexion, cisaillement, flèche).

> Résultat **préliminaire** : les résistances utilisent le coefficient de tenue φ = 0,9 du CSA O86, mais avec les coefficients d'ajustement Kd = Kl = 1,0 et sans effet de taille (Kz) — c'est **volontairement conservateur**. Un ingénieur doit valider et sceller tout calcul officiel.

### 3.4 Calculer une toiture (calculateur Toiture)

**4 sous-onglets** : *Surface + bardeaux · Ventilation · Gouttières · Charge de neige*.

- **Surface + bardeaux** : dimensions, pente (x:12), débord, perte, matériau (bardeaux, membranes, tôle) -> surface réelle (facteur de pente `√(1 + (pente/12)²)`), nombre de « carrés », paquets, sous-couche, membrane de glace, et **coûts matériau et total** en CAD.
- **Ventilation (CCQ 9.19.1)** : surface du comble + présence d'un pare-vapeur -> ratio de ventilation (1:300 avec pare-vapeur, sinon 1:150), surface nette de ventilation, longueur de soffite, nombre d'évents de faîte ou de turbines (répartition 50/50 entrée/sortie).
- **Gouttières (CCQ 9.14.6)** : surface de toit, périmètre, format (4 à 7 po) -> nombre de descentes, longueur, supports, angles, embouts.
- **Charge de neige (CNB 4.1.6.2)** : province (QC / ON / BC / AB), **ville** (champ texte libre), type de couverture, pente, exposition (Normale / Exposée) -> neige au sol (Ss), facteur de pente (Cs), charge de neige sur le toit (kPa et lb/pi²), charge morte, charge de design. La formule complète du CNB est appliquée : `S = Is · [Ss · (Cb·Cw·Cs·Ca) + Sr]`. Si la ville saisie n'est pas répertoriée, une valeur par défaut est utilisée.

### 3.5 Calculer la peinture (calculateur Peinture)

**3 sous-onglets** : *Quantité + coût · DFT · Point de rosée*.

- **Quantité + coût** : dimensions et hauteur de la pièce, nombre de portes et de fenêtres, nombre de couches, type de peinture, type de surface, méthode d'application (rouleau, pinceau, sans air, HVLP) -> surface nette (déduction des ouvertures), litres et gallons, épaisseur de film sec théorique, **coût avant et après taxes** (TPS + TVQ), coût au m², temps de recouvrement. La couverture est ajustée par des facteurs d'absorption (selon la surface) et d'efficacité de transfert (selon la méthode).
- **DFT (film sec)** : volume (mL), pourcentage de solides, surface -> épaisseur de film sec (µm et mils), couverture, évaluation.
- **Point de rosée (Magnus)** : température de l'air, humidité relative, température de surface -> badge *Application OK / Risque de condensation*, point de rosée, marge de sécurité, recommandation. Sert à valider qu'une surface est assez chaude avant de peindre (règle courante : surface ≥ point de rosée + 3 °C).

### 3.6 Électricité (calculateur Électricité)

**4 sous-onglets** : *Câble · Charge résidentielle · Éclairage · Mise à la terre*.

- **Câble (CCE 4-004)** : puissance (W), tension, longueur, facteur de puissance, chute de tension maximale, conducteur (cuivre / aluminium), circuit (mono / triphasé) -> courant (A), calibre AWG, section (mm²), chute réelle, ampacité (75 °C), disjoncteur, conformité. La section retenue est le maximum entre le critère de chute de tension et celui d'ampacité (× 1,25).
- **Charge résidentielle (CCE 8-200)** : surface habitable + charges (chauffage, climatisation, cuisinière, sécheuse, chauffe-eau, autres, en kW) -> demande totale (kW), courant sous 240 V, calibre de service recommandé (A), article de référence. Base 5 kW + 1 kW par tranche de 90 m².
- **Éclairage (lumens)** : surface, type de local, flux du luminaire, facteurs d'utilisation et de maintenance -> nombre de luminaires, éclairement requis (lux), disposition, espacement.
- **Mise à la terre** : résistivité du sol, longueur et diamètre du piquet, nombre de piquets -> badge *< 25 ohms (Hydro-Québec)*, résistance totale et par piquet.

### 3.7 Plomberie (calculateur Plomberie)

**4 sous-onglets** : *DFU + WSFU · Hazen-Williams · Chauffe-eau · Pente de drain*.

- **DFU + WSFU (CNP)** : nombre d'appareils (toilettes, lavabos, douches, baignoires, évier de cuisine, lave-vaisselle, laveuse, drain de plancher) -> total d'unités de vidange (DFU) et d'alimentation (WSFU), débit (GPM et LPM), diamètre de drain recommandé.
- **Hazen-Williams** : débit, longueur (pi), diamètre (po), matériau (coefficient C) -> perte de charge (psi/pi), vitesse, coefficient C.
- **Chauffe-eau** : nombre de chambres, de salles de bain et de personnes -> badge *Adéquat / Sous-dimensionné*, capacité (gal et L), débit de première heure (FHR) minimal.
- **Pente de drain** : diamètre, longueur, pente -> badge *Conforme CNP*, chute (m et po), pente recommandée.

### 3.8 CVAC (calculateur CVAC)

**5 sous-onglets** : *Charge thermique · Conduits · CFM par pièce · HRV/ERV · Climatisation*.

- **Charge thermique (ASHRAE)** : surface, hauteur de plafond, niveau d'isolation, zone climatique (8 villes de référence) -> pertes de design (W), BTU/h, tonnage, débit de ventilation (CFM), BTU/pi². Le calcul repose sur la méthode **enveloppe U·A·ΔT** (surface d'enveloppe estimée + infiltration à 0,5 renouvellement d'air à l'heure), et non sur un simple forfait au m².
- **Conduits** : débit (CFM) + type de circuit -> badge *Vitesse OK*, diamètre standard (po), vitesse réelle et recommandée.
- **CFM par pièce** : volume + type de pièce (renouvellements d'air à l'heure) -> débit requis, ACH.
- **HRV/ERV (ASHRAE 62.2)** : surface, nombre de chambres, nombre d'occupants -> débit recommandé, taille d'appareil, débit minimal 62.2.
- **Climatisation (gains solaires)** : surface vitrée, orientation, coefficient de gain solaire (SHGC), rayonnement, occupants, équipements -> gain total (BTU/h), tonnage.

### 3.9 Soudure (calculateur Soudure)

**3 sous-onglets** : *Soudure d'angle · Apport thermique · Préchauffage CE*.

- **Soudure d'angle (CSA W47.1)** : type de joint, épaisseur, longueur, procédé (SMAW / GMAW / FCAW / GTAW / SAW) -> gorge (`0,707 × jambe`), jambe, volume, poids de métal déposé, consommation d'électrode (avec facteur de perte), taux de dépôt.
- **Apport thermique (kJ/mm)** : tension, ampérage, vitesse, procédé -> apport thermique net (tenant compte du rendement d'arc) et évaluations pour acier au carbone et pour inox/aluminium.
- **Préchauffage (carbone équivalent IIW)** : composition (% C, Mn, Cr, Mo, V, Ni, Cu) + épaisseur -> carbone équivalent, risque de fissuration, température de préchauffage recommandée, et la formule utilisée affichée.

### 3.10 Pliage métal (calculateur Pliage métal)

**3 sous-onglets** : *Développement + tonnage · Retour élastique · Rayon minimum*.

- **Développement + tonnage (pliage à l'air)** : longueur, épaisseur, angle, rayon intérieur, largeur, matériau -> longueur développée, facteur K, marge et retrait de pliage, tonnage requis (kN), ouverture de la matrice en V, rayon minimal, retour élastique. Une **alerte** apparaît si le rayon demandé risque de fissurer le matériau. Le tonnage suit la formule `1,42 · UTS · t² · L / (V · 1000)`.
- **Retour élastique** : angle voulu + matériau -> angle à plier (compensé) et retour élastique.
- **Rayon minimum** : épaisseur + matériau -> rayon minimal (mm et po) et facteur.

### 3.11 Poids métal (calculateur Poids métal)

Formulaire unique. Choisir la **forme** (plaque, tube rond, tube carré, barre ronde, barre carrée, cornière L, poutre en I, profilé W AISC, profilé C UPN), le **matériau** (une vingtaine, chargés du serveur) et les **dimensions** (les champs s'adaptent à la forme). Pour les profilés W et C, un sélecteur de section remplace les dimensions manuelles. **Sorties** : poids (kg et lb), volume, densité, coût estimé (CAD), désignation de section.

### 3.12 Taxes Québec (calculateur Taxes Québec)

Saisir le **montant avant taxes** ($) -> **Calculer** -> montant HT, **TPS (5 %)**, **TVQ (9,975 %)**, total TTC.

### 3.13 Paie employé (calculateur Paie employé)

Saisir le **salaire brut** et le **type d'employé** (*Régulier* ou *Construction CCQ*) -> deux blocs :

- **Déductions de l'employé** : RRQ, RQAP, AE, total ;
- **Charges de l'employeur** : CNESST, FSS, CCQ (si applicable), total ;
- plus le **salaire net** et le **coût total employeur**.

> Les taux proviennent du **module central de paie** (`payroll_rates`), synchronisé avec le module *Pointage et Paie* pour éviter les divergences. Attention aux **simplifications** : l'impôt fédéral et provincial est estimé à 15 % / 15 % (pas les vrais paliers progressifs), la CNESST est un **taux plat par défaut** (le vrai taux dépend de la classe de risque du métier), et la contribution CCQ est un **taux plat indicatif** (la vraie cotisation est en $/heure par métier). Ce calculateur donne un **ordre de grandeur**, pas une paie officielle.

### 3.14 Concevoir un mur et l'envoyer au devis (Murs paramétrique)

Outil de **charpente murale légère** en **impérial**, calculé **côté navigateur** (inspiré des générateurs de murs *Wall / Rake / Tall Wall Builder*). Explicitement marqué **« En construction »** : utile, mais encore en évolution.

- **Barre de projet** : nom du projet (avec identifiant), boutons **Nouveau**, **Charger** (projets sauvegardés), **Sauver**, plus **Envoyer au devis** (crée une ou des lignes de devis) et **Créer BOM Métré** (produit un assemblage vers le module Métré).
- **Modes** : Standard / Pignon / Mur haut.
- **Vues** : Avant / Arrière / **3D**, avec annulation/rétablissement (jusqu'à 50 pas) et zoom.
- **4 onglets** : **Mur** (longueur, taille de montant 2×4/2×6/2×8, hauteurs, espacement, premier montant, direction, doublage, blocage, lisse haute, revêtement + un panneau repliable **Conformité entrepreneur général Québec**) ; **Ouvertures** (fenêtre/porte, forme rectangulaire ou arquée, position, largeur, hauteur, hauteur d'appui) ; **Coupe** (liste de coupe exportable en **CSV** et en **PDF**) ; **Détails** (montants de roi, jambages, montants nains, linteaux, lisses, blocages).

**Exemple :**

1. Ouvrir **Calculateurs** -> **Murs paramétrique (En construction)**.
2. Régler les propriétés du mur (onglet **Mur**) : longueur, taille et hauteur de montant, espacement.
3. Ajouter les ouvertures nécessaires (onglet **Ouvertures**).
4. Vérifier la liste de coupe (onglet **Coupe**) et l'exporter en CSV ou PDF au besoin.
5. Cliquer sur **Sauver** pour conserver le projet (il apparaîtra dans l'onglet Historique).
6. Cliquer sur **Envoyer au devis** pour transférer les quantités dans une soumission, **ou Créer BOM Métré** pour alimenter un assemblage du module Métré.

### 3.15 Utiliser l'Assistant IA

1. Onglet **Assistant IA**.
2. Choisir le sous-outil : **Chat Expert**, **Recommandations**, **Expliquer une norme** ou **Diagnostic**.
3. Pour le chat : choisir un calculateur (facultatif), taper la question, **Envoyer**. Pour les autres : remplir les champs (objectif, norme, problème/symptômes selon le cas).
4. Lire la réponse. Chaque appel débite les crédits IA au coût réel × 1,30. Si vous quittez l'onglet pendant la génération, l'appel est annulé et **rien n'est facturé**.

### 3.16 Consulter et gérer l'historique

1. Onglet **Historique**.
2. Filtrer par calculateur au besoin.
3. Sur une ligne, cliquer sur **Détails** pour voir les entrées et résultats.
4. Supprimer une entrée (corbeille) ou tout effacer (**Effacer tout**, avec confirmation).

> Rappel : aujourd'hui, ce sont surtout les projets de **Murs paramétrique** qui remplissent cette liste (voir 2.7).

---

## 4. Référence

### 4.1 Les 59 points d'entrée (préfixe `/api/erp/v1/calculators`)

Tous exigent une session authentifiée. Les points d'entrée de calcul sont des `POST` ; les données de référence et l'historique combinent `GET`, `POST` et `DELETE`.

| Domaine | Points d'entrée |
|---|---|
| **Béton** (8) | `/concrete`, `/concrete/dosage`, `/concrete/rebar`, `/concrete/cure`, `/concrete/formwork`, `/concrete/excavation`, `/concrete/talus`, `/concrete/stairs` |
| **Escaliers** (3) | `/stairs`, `/stairs/materials`, `/stairs/garde-corps` |
| **Électricité** (4) | `/electrical`, `/electrical/residential`, `/electrical/lighting`, `/electrical/grounding` |
| **Toiture** (4) | `/roofing`, `/roofing/ventilation`, `/roofing/gutters`, `/roofing/snow-load` |
| **Peinture** (3) | `/painting`, `/painting/dft`, `/painting/dew-point` |
| **Plomberie** (4) | `/plumbing`, `/plumbing/hazen-williams`, `/plumbing/water-heater`, `/plumbing/drain-slope` |
| **CVAC** (5) | `/hvac`, `/hvac/duct`, `/hvac/cfm`, `/hvac/hrv`, `/hvac/cooling` |
| **Soudure** (4) | `/welding`, `/welding/heat-input`, `/welding/preheat`, `/welding/consumable` |
| **Pliage** (3) | `/bending`, `/bending/springback`, `/bending/min-radius` |
| **Poids métal** (1) | `/metal-weight` |
| **Taxes** (1) | `/taxes` |
| **Paie** (1) | `/charge-tributaire` |
| **Analyse structurale** (3) | `POST /charge-tributaire-complete`, `GET /charge-tributaire-complete/materials`, `GET /charge-tributaire-complete/snow-loads` |
| **Conversions** (1) | `GET /conversions` (données de référence — pas d'onglet dédié dans l'interface) |
| **Historique** (5) | `GET /history`, `POST /history`, `DELETE /history/{id}`, `DELETE /history`, `GET /history/stats` |
| **Assistant IA** (6) | `/ai/chat`, `/ai/analyze`, `/ai/recommend`, `/ai/explain-norm`, `/ai/diagnose`, `/ai/optimize` |
| **Références** (3) | `GET /constants`, `GET /resources`, `GET /` (liste des calculateurs) |

**Total : 59 points d'entrée.** Répartition : 42 `POST` de calcul + 2 `GET` de données structurales + 1 `GET` de conversions + 5 d'historique + 6 d'IA + 3 de références.

> Deux points d'entrée serveur n'ont **aucun bouton** dans l'interface : `/concrete/formwork` (le coffrage est déjà donné dans le résultat *Volume*) et `/ai/analyze` + `/ai/optimize` (2 des 6 fonctions IA). Ils existent, mais ne sont pas atteignables depuis l'écran actuel.

### 4.2 Normes et formules appliquées (comportement réel du code)

| Norme / méthode | Calculateur | Ce que le code applique |
|---|---|---|
| **CSA A23.1** | Béton | Dosages 15 à 40 MPa selon la classe d'exposition |
| **CSA G30.18** | Armature | Barres 10M à 55M, masses linéiques, découpe 6 m |
| **ACI 209** | Cure du béton | `f(t) = f28 · t / (a + b·t)` + facteur de maturité par température |
| **CNESST** | Talus d'excavation | Pentes H:V par type de sol + exigences aux seuils 1,2 / 3 / 6 m |
| **Blondel** | Escaliers | `2R + G` dans la plage confort |
| **CCQ 9.8 / 3.4** | Escaliers | Contremarche, giron, largeur (résidentiel / commercial) |
| **CCQ 9.8.7** | Garde-corps | Hauteur, espacement des barreaux, main courante |
| **CCQ 9.19.1** | Ventilation de comble | 1:300 avec pare-vapeur, 1:150 sans |
| **CCQ 9.14.6** | Gouttières | Capacité de drainage par format |
| **CNB 4.1.6.2** | Charge de neige | `S = Is · [Ss · (Cb·Cw·Cs·Ca) + Sr]` (formule complète) |
| **CNB (combinaisons)** | Analyse structurale | `1,4D` / `1,25D+1,5L` / `+1,5S` |
| **CSA O86** | Analyse structurale (bois) | `Mr = φ·Fb·S·Kd·Kl` et `Vr`, avec **φ = 0,9**, **Kd = Kl = 1,0**, Kz omis (conservateur) |
| **CCE 4-004** | Câble | Section = max(chute de tension, ampacité × 1,25) |
| **CCE 8-200** | Charge résidentielle | Base 5 kW + 1 kW par 90 m² |
| **CNP** | Plomberie | DFU / WSFU, diamètre de drain par charge |
| **Hazen-Williams** | Plomberie | Perte de charge `hf = 10,44·L·Q^1,852 / (C^1,852·d^4,87)` |
| **ASHRAE (U·A·ΔT)** | CVAC | Enveloppe + infiltration 0,5 ACH |
| **ASHRAE 62.2** | CVAC (HRV/ERV) | Débit minimal de ventilation résidentielle |
| **CSA W47.1 / W59 / AWS D1.1** | Soudure | Gorge `0,707 × jambe`, apport thermique net (rendement d'arc) |
| **IIW** | Soudure | `CE = C + Mn/6 + (Cr+Mo+V)/5 + (Ni+Cu)/15` |
| **Magnus** | Peinture | Point de rosée |
| **AISC / CISC** | Poids métal | Bibliothèque de profilés W / C |
| **TPS / TVQ** | Taxes | 5 % + 9,975 % |
| **RRQ / RQAP / AE / CNESST / FSS / CCQ** | Paie | Taux du module central `payroll_rates` |

### 4.3 Codes de réponse HTTP utiles

| Code | Signification pour l'utilisateur |
|---|---|
| 200 | Calcul réussi |
| 400 | Entrée invalide (ex. dalle trop mince pour l'armature, section de profilé inconnue, **type « Colonne » refusé**, ville de neige inconnue) ou **contexte tenant manquant** pour l'historique |
| 402 | Crédits IA épuisés (Assistant IA) |
| 403 | IA désactivée pour le tenant |
| 404 | Élément d'historique introuvable à la suppression |
| 413 | Requête IA trop volumineuse |
| 502 | Réponse IA inexploitable |
| 503 | Service IA indisponible |

### 4.4 Limites et simplifications connues

Le calculateur **produit toujours un résultat**, mais certains modèles sont volontairement simplifiés. À connaître :

| Sujet | Comportement actuel |
|---|---|
| **Analyse structurale (bois)** | φ = 0,9 mais Kd = Kl = 1,0 et effet de taille (Kz) omis -> pré-dimensionnement **conservateur**, à valider par un ingénieur |
| **Colonne / compression** | Non implémentée : l'option n'est pas offerte et est refusée côté serveur (erreur 400) |
| **Linteau** | Traité comme une poutre (flèche L/360), sans vérification dédiée d'appui/portance |
| **Acier structural (CSA S16)** | Non implémenté : seul le bois (CSA O86) est calculé en analyse structurale |
| **Charges sismiques** | Non incluses (pas de spectre, pas de vérification de ductilité) |
| **Impôt de la paie** | 15 % / 15 % forfaitaire, pas les vrais paliers progressifs |
| **CNESST** | Taux plat par défaut ; le vrai taux dépend de la classe de risque du métier |
| **CCQ (paie)** | Taux plat indicatif ; la vraie cotisation est en $/heure par métier |
| **Charge de neige** | Valeur par défaut si la ville n'est pas répertoriée |
| **Pliage** | Un seul pli par calcul (additionner pour une pièce multi-plis) |
| **Prix en CAD** | Indicatifs (tables de référence internes) ; pour une soumission, utiliser les prix fournisseurs du module *Magasin/Inventaire* |

> **Bon à savoir** : deux modèles autrefois simplifiés sont **maintenant complets** — la **charge de neige** applique la formule intégrale du CNB 4.1.6.2, et la **charge CVAC** utilise la méthode enveloppe U·A·ΔT (avec infiltration). Les notes internes plus anciennes qui les listaient « à retravailler » sont périmées.

### 4.5 Limitation de débit et disponibilité

- Les **6 points d'entrée IA** (`/ai/*`) sont limités à **10 requêtes par minute** (par adresse). Les ~50 autres points d'entrée de calcul retombent sur la limite générale, beaucoup plus élevée (1 500/min).
- Tous les calculs se font **côté serveur** (sauf la Calculatrice et les Murs paramétrique) : une connexion réseau et une session valide sont requises.

### 4.6 Raccourcis clavier (Calculatrice)

Valables dans la tuile **Calculatrice** (`master-pro`) :

| Touche | Action |
|---|---|
| `0`–`9` | Saisie des chiffres |
| `+ − × ÷` | Opérateurs |
| `Entrée` | Égale (=) |
| `Échap` | Tout effacer |
| `f` | Pieds |
| `i` | Pouces |
| `y` | Verges |
| `c` | Fonction secondaire (*Conv / Shift*) |

### 4.7 Persistance de l'historique

Table `calculator_history`, créée **à la demande** dans le schéma du tenant au premier usage (colonnes : `id`, `calculator_id`, `subcalc_id`, `label`, `inputs` JSONB, `results` JSONB, `notes`, `user_id`, `created_at`). Deux index accélèrent la lecture. Aucune purge automatique : les entrées restent jusqu'à suppression manuelle.

---

## 5. Intégrations et FAQ

### 5.1 Intégrations avec les autres modules

| Module | Lien | Détail |
|---|---|---|
| **Devis / Soumissions** | Partiel | Les **Murs paramétrique** ont un bouton **Envoyer au devis** qui crée des lignes. Les 13 calculateurs côté serveur, eux, **ne s'injectent pas** automatiquement : on recopie les résultats à la main. |
| **Métré** | Partiel | Les **Murs paramétrique** peuvent **Créer un BOM Métré** (assemblage). |
| **Magasin / Inventaire** | Aucun | Les calculateurs ne consomment ni ne réservent de stock ; les quantités se transcrivent manuellement. Utiliser les prix fournisseurs réels de ce module plutôt que les prix indicatifs. |
| **Bons de commande** | Aucun | Recopier les quantités de matériaux à la main dans le bon. |
| **Comptabilité / Paie** | Aucun (informatif) | Les calculateurs *Taxes* et *Paie employé* donnent des chiffres indicatifs ; ils ne créent aucune facture ni écriture. Pour la paie réelle, voir le module *Pointage et Paie*. |
| **Assistant IA** | Crédits partagés | L'IA des calculateurs puise dans le **même portefeuille de crédits IA prépayés** que les autres modules IA. |

### 5.2 FAQ

**Les résultats valent-ils un calcul d'ingénieur signé ?**
Non. Ce sont des outils d'**aide à l'estimation et au pré-dimensionnement**. Pour un permis ou un plan scellé, un ingénieur (OIQ) doit valider et signer. L'Analyse structurale est particulièrement conservatrice (Kd = Kl = 1,0).

**Pourquoi le tableau de bord affiche-t-il 15 alors qu'on parle parfois de 13 calculateurs ?**
Les **15** comptent les tuiles visibles (13 côté serveur + la Calculatrice + les Murs paramétrique). Les **13** ne comptent que les calculateurs calculés côté serveur. Les deux chiffres sont exacts.

**La Calculatrice fonctionne-t-elle hors ligne ?**
Oui, une fois la page chargée : elle calcule dans le navigateur, sans appel serveur. Idem pour l'aspect calcul des Murs paramétrique (la sauvegarde de projet, elle, exige le réseau). Les 13 autres calculateurs, eux, ont besoin du serveur.

**Où sont passées les Conversions ?**
Il n'y a plus d'onglet « Conversions » séparé. Les conversions d'unités se font dans la **Calculatrice** : onglet interne *Conversions* et feuille de conversion (m, cm, mm, pi, po, vg).

**Pourquoi mon calcul de béton n'apparaît-il pas dans l'Historique ?**
Parce que les 13 calculateurs côté serveur n'ont pas de bouton « Sauvegarder » câblé dans l'interface actuelle : leurs résultats sont éphémères. Seuls les projets de **Murs paramétrique** se sauvegardent dans l'historique. Pour conserver un calcul, notez-le ou recopiez-le dans un devis.

**Puis-je analyser une colonne (compression, flambement) ?**
Non. L'Analyse structurale ne propose que **Poutre** et **Linteau**. Le mode colonne n'est pas implémenté et est refusé.

**Que se passe-t-il si le service IA est en panne ?**
Les 4 sous-outils IA renvoient une erreur 503, mais **tous les autres calculateurs continuent** de fonctionner (ils sont déterministes, sans IA).

**Peut-on personnaliser les taux CCQ / CNESST par entreprise ?**
Non dans cette version. Les taux de paie viennent du module central `payroll_rates` (année en vigueur). Pour une personnalisation, contactez l'administrateur Constructo.

**Comment exporter un calcul en PDF pour un client ?**
Il n'y a pas d'export PDF officiel pour les 13 calculateurs (utilisez l'impression du navigateur, ou l'onglet Historique -> Détails). En revanche, les **Murs paramétrique** exportent leur liste de coupe en **CSV** et en **PDF**.

**Le calculateur de soudure prescrit-il les essais non destructifs ?**
Non. Il dimensionne le joint (gorge, jambe, consommation, préchauffage), mais ne prescrit pas les contrôles (radiographie, ressuage, magnétoscopie). Suivre CSA W59 / AWS D1.1 selon la classe d'ouvrage.

**Les calculateurs de plancher / revêtement / concepteur de maison sont-ils disponibles ?**
Non. Sept composants de ce genre existent dans le code mais **ne sont reliés à aucun écran** : ils sont inaccessibles aujourd'hui. Le seul panneau de toiture actif est le calculateur **Toiture** décrit en 3.7.

**Les charges de neige sont-elles à jour avec la dernière édition du CNB ?**
Les valeurs de neige au sol par ville sont intégrées au code (article 4.1.6.2). Vérifier l'édition applicable à votre projet et demander une mise à jour si une nouvelle édition modifie les valeurs.

---

## 6. Récapitulatif

- **Module** : suite de **15 tuiles** de calcul de construction (Québec) = **13 calculateurs côté serveur** + **2 outils côté navigateur** (Calculatrice, Murs paramétrique), plus un Assistant IA et un historique par tenant.
- **5 onglets** : Tableau de bord · Calculateurs · Analyse structurale · Assistant IA · Historique. **Pas** d'onglet Conversions (les conversions vivent dans la Calculatrice).
- **6 catégories** : Polyvalent (1) · Structure (4) · Enveloppe (2) · Mécanique (3) · Métal (3) · Finances (2).
- **59 points d'entrée** sous `/api/erp/v1/calculators` : 42 calculs + 2 données structurales + 1 conversions + 5 historique + 6 IA + 3 références.
- **Accès** : menu latéral -> Outils -> Calculateurs. **Aucun rôle particulier requis** ; l'historique est partagé par tenant ; l'IA exige des crédits prépayés.
- **Béton** : volume, dosage CSA A23.1, armature, cure ACI 209, excavation, talus CNESST, escalier Blondel (7 sous-onglets).
- **Escaliers** : dimensions CCQ 9.8/3.4 + garde-corps 9.8.7.
- **Analyse structurale** : Poutre / Linteau, bois SPF ou LVL, combinaisons CNB, Mr/Vr CSA O86, flèche L/360 ou L/180, diagramme. **Conservateur** (Kd = Kl = 1,0), **pas de colonne**.
- **Toiture** : surface/bardeaux, ventilation 1:300/1:150, gouttières, charge de neige **CNB 4.1.6.2 complète**.
- **Peinture** : quantité + coût (TPS/TVQ), épaisseur de film sec, point de rosée Magnus.
- **Électricité** : câble AWG (CCE 4-004), charge résidentielle (CCE 8-200), éclairage, mise à la terre.
- **Plomberie** : DFU/WSFU (CNP), Hazen-Williams, chauffe-eau, pente de drain.
- **CVAC** : charge thermique **U·A·ΔT** (ASHRAE), conduits, CFM/pièce, HRV/ERV (62.2), climatisation.
- **Soudure** : angle (CSA W47.1), apport thermique, préchauffage (carbone équivalent IIW).
- **Pliage** : développement + tonnage (pliage à l'air), retour élastique, rayon minimum.
- **Poids métal** : 9 formes × ~20 matériaux + profilés W/C AISC.
- **Taxes** : TPS 5 % + TVQ 9,975 %. **Paie** : déductions employé + charges employeur (taux du module central), avec impôt et CNESST simplifiés.
- **Calculatrice** : calculatrice de chantier pieds-pouces (style Construction Master Pro), 100 % locale, avec conversions.
- **Murs paramétrique (En construction)** : charpente murale impériale, vue 3D, liste de coupe CSV/PDF, **Envoyer au devis** et **Créer BOM Métré**.
- **Assistant IA** : **4 sous-outils** (Chat, Recommandations, Expliquer une norme, Diagnostic) sur **Claude Opus 4.8** (32 000 jetons), facturés au coût réel × 1,30 sur les crédits prépayés ; annulés sans frais si on quitte l'onglet.
- **Limites** : pré-dimensionnement seulement, pas de sceau d'ingénieur, pas d'acier structural, pas de colonne, paie indicative, prix indicatifs.

---

**Fichiers sources vérifiés** : `frontend/src/pages/CalculateursPage.tsx` (2 269 lignes), `frontend/src/components/calculateurs/MasterProCalculatorBody.tsx` (903 lignes), `frontend/src/components/calculateurs/MursParametriquePanel.tsx` (2 676 lignes), `frontend/src/api/calculators.ts` (1 642 lignes), `backend/routers/calculators.py` (3 681 lignes, 59 points d'entrée), `backend/routers/calculators_data.py` (807 lignes).

**Manuels liés** :
- Module 07 — Soumissions (recopier ou recevoir les quantités) — `07-ventes-soumissions.md`
- Module 09 — Magasin / Inventaire (prix fournisseurs réels) — `09-operations-magasin.md`
- Module 12 — Pointage (paie réelle vs calcul indicatif) — `12-operations-pointage.md`
- Module 13 — Bons de commande (achat des matériaux) — `13-operations-bons-de-commande.md`
- Module 24 — Assistant IA (crédits IA prépayés) — `24-communication-assistant-ia.md`
- Module 32 — Métré (assemblages BOM, prise de quantités) — `32-metre-pdf.md`
- Module 25 — Modélisation 3D / DAO — `25-outils-dao-modelisation.md`
