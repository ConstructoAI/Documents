# Module 18 — Subventions et aides gouvernementales

> **Version** : 3.0 (refonte vérifiée ligne par ligne contre le code source du 7 juillet 2026 — correction du nombre d'onglets (6 et non 5), du modèle IA, des permissions réelles d'écriture, du calcul des indicateurs financiers, du nombre exact de programmes et de la tarification IA)
> **Libellé dans le menu** : « Subventions » (clé `nav.subventions`), groupe « TERRAIN » de la barre latérale, icône `Landmark` — route `/subventions`
> **Code de référence (backend)** : `ERP_REACT/backend/routers/subventions.py` (1687 lignes, 23 points d'accès dont 5 outils IA) ; `ERP_REACT/backend/routers/subventions_data.py` (732 lignes, **module de données statiques pur — ce n'est PAS un router monté**, aucun point d'accès)
> **Code de référence (frontend)** : `ERP_REACT/frontend/src/pages/SubventionsPage.tsx` (1921 lignes, 6 onglets, tous les composants en ligne — le dossier `components/subventions` n'existe pas) ; `ERP_REACT/frontend/src/api/subventions.ts` ; `ERP_REACT/frontend/src/store/useSubventionsStore.ts` (magasin Zustand)
> **Chemin d'API réel** : `/api/erp/v1/subventions` (préfixe `/subventions` monté avec `API_PREFIX = /api/erp/v1`)
> **Tables PostgreSQL (une par tenant, créées à la demande)** : `subventions_categories`, `subventions_programmes`, `subventions_demandes`, `subventions_documents` (colonne `fichier_data` BYTEA pour les pièces jointes)
> **Modèle IA** : Claude Opus 4.8 (`claude-opus-4-8`), 32 000 jetons maximum par appel, facturé aux crédits IA prépayés du tenant
> **Cadrage** : ce module est un **catalogue de programmes de subventions et un registre de suivi des demandes** (découverte → vérification d'éligibilité → demande → soumission → décision → versement). Ce n'est **pas** un module comptable : les versements ne sont **pas** comptabilisés automatiquement (voir Module 15), et ce n'est **pas** un connecteur officiel : le module ne dépose aucune demande auprès des organismes et ne se connecte à aucun registre gouvernemental. Il est spécialisé sur les programmes accessibles aux entreprises québécoises de construction, de rénovation et de services connexes (fédéral, provincial, municipal).

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

Centraliser, dans un seul écran, la **veille et le suivi des programmes de subventions** accessibles aux entreprises québécoises de construction, afin de ne rien laisser sur la table : repérer les programmes pertinents, vérifier son admissibilité, préparer un dossier solide et suivre chaque demande jusqu'au versement des fonds.

Le module répond à cinq besoins concrets :

- « Quels programmes de subventions, de prêts ou de crédits d'impôt sont accessibles à mon entreprise ? »
- « Suis-je admissible, et lesquels valent le plus la peine selon mon secteur et mon budget ? »
- « Où en sont mes demandes en cours (brouillon, soumise, approuvée, versée) ? »
- « Quels documents dois-je fournir, et comment maximiser mes chances d'approbation ? »
- « Combien ai-je demandé, combien m'a-t-on accordé, et quels programmes arrivent à échéance ? »

### 1.2 Ce que le module fait (vérifié contre le code)

- Offrir un **catalogue de 47 programmes** préchargés (fédéral, provincial, municipal), filtrables par catégorie, type d'aide, niveau de gouvernement, difficulté et texte libre.
- Fournir un **vérificateur d'éligibilité algorithmique** (pointage sectoriel et budgétaire) instantané et **gratuit** (aucun crédit IA consommé).
- Tenir un **registre de demandes** avec un cycle de vie complet à **9 statuts** (`BROUILLON` → `EN_PREPARATION` → `SOUMISE` → `EN_EVALUATION` → `INFO_SUPPLEMENTAIRE` → `APPROUVEE` / `REFUSEE` → `VERSEE`, plus `ANNULEE`).
- Permettre le **téléversement de pièces justificatives** (PDF, Word, Excel, images, texte, CSV) jusqu'à 10 Mo par fichier, stockées en base, avec un statut par document et un **téléchargement** au besoin.
- Afficher un **tableau de bord** : quatre indicateurs clés, trois graphiques (par catégorie, par niveau, par statut) et une alerte des programmes qui expirent dans les 30 prochains jours.
- Rassembler des **ressources** : 8 organismes partenaires, le Plan PME 2025-2028 (219 M$) et deux blocs de conseils pratiques.
- Offrir un **assistant IA** à cinq outils (Claude Opus 4.8) : suggérer des programmes, analyser l'éligibilité en profondeur, discuter avec un expert, générer une liste de vérification pour un programme et analyser la préparation d'une demande.

### 1.3 Ce que le module NE fait PAS (limites importantes)

> **À lire avant de vous fier au module.** Plusieurs attentes naturelles ne sont **pas** couvertes. Ce module est un outil de veille et de suivi interne, pas un guichet de dépôt officiel ni un logiciel comptable.

- **Aucun dépôt de demande auprès des organismes.** Le bouton « Soumettre » ne fait que passer le statut **interne** à `SOUMISE` pour votre propre suivi. La présentation réelle du dossier se fait **manuellement** sur les portails officiels (Investissement Québec, SCHL, BDC, votre MRC, etc.).
- **Aucune connexion aux registres gouvernementaux.** Le catalogue est un jeu de données préchargé dans l'ERP ; il n'interroge aucun site officiel en direct. Les montants et conditions sont **indicatifs** et changent fréquemment : validez toujours sur les sites officiels.
- **Aucune écriture comptable automatique.** Un versement (`statut = VERSEE`) n'est **pas** comptabilisé dans le grand livre. Il faut créer l'écriture manuellement dans le Module 15 (Comptabilité).
- **Aucune exportation ni impression** des données : pas de PDF, pas de CSV, pas de bouton d'impression, pas de génération de rapport. Le **seul** téléchargement possible est une **pièce jointe** que vous avez vous-même téléversée.
- **Catalogue en lecture seule.** Vous ne pouvez ni créer, ni modifier, ni supprimer un programme ou une catégorie depuis l'interface. Le catalogue est alimenté uniquement par un jeu de données interne (aucun point d'accès d'écriture n'existe pour les programmes).
- **Aucun rappel par courriel, par notification ou par calendrier** (iCal, Google Agenda). L'alerte des programmes qui expirent vit uniquement dans le tableau de bord.
- **Aucune machine d'états.** Rien n'empêche techniquement de faire passer une demande de `BROUILLON` directement à `VERSEE` : la logique du cycle de vie est laissée à votre discipline.
- **Aucun processus d'approbation interne** ni signature électronique.
- **Aucun versement partiel.** Une demande porte un seul montant accordé et une seule date de versement (créez plusieurs demandes pour suivre des versements échelonnés).
- **Clés d'affichage héritées inutilisées.** Certaines chaînes de traduction (sous-titre, « Consulter », « Appliquer », un ancien bloc de statuts en minuscules) subsistent dans les fichiers de langue mais **ne sont rendues nulle part** dans l'interface React actuelle. Le présent manuel décrit uniquement ce qui est réellement affiché.

### 1.4 Accès par le menu latéral

- Barre latérale gauche → groupe **TERRAIN** (repliable) → **Subventions** (icône `Landmark`).
- URL directe : `/subventions`.
- Titre affiché en haut de la page : « **Subventions** » (à côté de l'icône Landmark). Le sous-titre défini dans les traductions (« Programmes gouvernementaux et aides financières ») n'est **pas** affiché.
- **Onglet ouvert par défaut** : **Catalogue** (état initial du composant), même si l'onglet « Tableau de bord » apparaît en premier dans la barre d'onglets.

### 1.5 Permissions et rôles

Les droits ne sont **pas** les mêmes pour tout le monde — c'est la correction la plus importante par rapport aux versions antérieures de ce manuel, qui affirmaient à tort que « tous les utilisateurs peuvent créer et modifier des demandes ».

| Action | Qui peut la faire | Garde technique |
|--------|-------------------|-----------------|
| **Consulter** le catalogue, l'éligibilité, les demandes, les statistiques, les ressources | Tout utilisateur connecté au tenant | `get_current_user` |
| Télécharger une pièce jointe | Tout utilisateur connecté au tenant | `get_current_user` |
| Lancer le **vérificateur d'éligibilité** (algorithmique, gratuit) | Tout utilisateur connecté au tenant | `get_current_user` |
| Utiliser les **5 outils IA** | Tout utilisateur connecté, **si le tenant a des crédits IA** | crédits prépayés (voir ci-dessous) |
| **Créer / modifier / soumettre / supprimer une demande** | Administrateur **ou** rôle « comptable » | `require_tenant_admin_or_role("comptable")` |
| **Téléverser / changer le statut / supprimer une pièce jointe** | Administrateur **ou** rôle « comptable » | `require_tenant_admin_or_role("comptable")` |

- Le rôle « administrateur ou comptable » est **relu côté serveur** à chaque requête (`is_admin` infalsifiable) : il ne peut pas être contourné depuis le navigateur. Il couvre l'administrateur du tenant, le rôle `admin`, le rôle `comptable` et le super-administrateur.
- **Nuance à connaître** : les cinq outils IA ne sont protégés que par `get_current_user`. Un utilisateur qui **n'est ni administrateur ni comptable** peut donc **lancer l'IA payante** (suggestions, analyses, chat) sans pouvoir **créer** de demande. Encadrez cet usage si le contrôle des coûts est un enjeu.
- Les **outils IA consomment des crédits IA prépayés** du tenant. Le serveur vérifie d'abord la disponibilité du service (sinon erreur 503), puis le solde de crédits : un solde épuisé renvoie une erreur **402** affichée dans la bannière rouge. Le super-administrateur et les sociétés exemptées ne sont pas bloqués par ce contrôle.
- Toutes les données sont **strictement cloisonnées par tenant** (schéma PostgreSQL propre à l'entreprise). Aucun accès entre tenants n'est possible.

### 1.6 Les six onglets du module

| # | Onglet (libellé affiché) | Contenu réel | Icône |
|---|--------------------------|--------------|-------|
| 1 | **Tableau de bord** | Indicateurs, graphiques, alerte des programmes expirants | `BarChart3` |
| 2 | **Catalogue** | 47 programmes filtrables + liste de vérification IA + création de demande | `BookOpen` |
| 3 | **Éligibilité** | Vérificateur algorithmique gratuit selon le profil de l'entreprise | `Target` |
| 4 | **Mes demandes** | Registre des demandes + cycle de vie + documents + analyse IA | `FileText` |
| 5 | **Ressources** | Conseils pratiques, 8 organismes, Plan PME 2025-2028 | `Layers` |
| 6 | **Assistant IA** | Trois sous-outils IA (suggérer, analyser l'éligibilité, discuter) | `Sparkles` |

> **Correction majeure vs l'ancien manuel** : le module compte **6 onglets**, pas 5. L'onglet **« Assistant IA »** a été ajouté et n'existait pas dans la documentation précédente. Le commentaire interne du fichier de page annonce d'ailleurs encore « 5 tabs » à tort.

---

## 2. Interface

### 2.0 Éléments communs à tous les onglets

- **Titre de page** : icône `Landmark` + « Subventions ».
- **Chargement initial** : à l'ouverture, la page lance **sept requêtes en parallèle** (constantes, catégories, programmes, demandes, statistiques, programmes expirant sous 30 jours, ressources). Tant que les constantes ne sont pas arrivées, un squelette pleine page s'affiche.
- **Bannières** : une bannière rouge affiche les erreurs ; une bannière verte confirme les succès (elle se masque seule après environ 4 secondes).
- **Changement d'onglet** : en quittant un onglet, l'erreur affichée et les **résultats IA transitoires** sont purgés ; **l'historique de la conversation** du chat, lui, est conservé.
- **Recherche** : le champ de recherche du catalogue attend **400 ms** après la dernière frappe avant d'envoyer la requête (les caractères spéciaux sont échappés côté serveur).
- **Confirmations** : soumettre ou supprimer une demande, comme supprimer un document, demande une confirmation par une petite fenêtre native du navigateur.

### 2.1 Onglet « Tableau de bord »

Écran de synthèse alimenté par `GET /statistics`.

**Quatre indicateurs clés (cartes du haut)**

| Indicateur | Couleur | Calcul exact |
|------------|---------|--------------|
| **Programmes actifs** | bleu | Nombre de programmes actifs du catalogue |
| **Demandes totales** | violet | Nombre total de demandes, tous statuts confondus |
| **Montant demandé** | jaune | Somme des montants demandés de **toutes** les demandes **sauf** celles au statut `ANNULEE` |
| **Montant accordé** | vert | Somme des montants accordés des demandes au statut `APPROUVEE` ou `VERSEE` |

> **Correction vs l'ancien manuel** : l'indicateur « Montant demandé » additionne **toutes** les demandes non annulées — y compris une demande simplement `SOUMISE` en attente de décision. L'ancienne documentation prétendait à tort qu'il ne comptait que les demandes approuvées ou versées. Le « Montant accordé », lui, ne compte bien que les `APPROUVEE` et `VERSEE`.

**Trois graphiques**

- **Programmes par catégorie** (diagramme à barres) — décompte par catégorie du catalogue.
- **Programmes par niveau** (diagramme circulaire) — répartition fédéral / provincial / municipal / mixte.
- **Demandes par statut** (diagramme à barres) — affiché seulement s'il existe des demandes.

**Carte d'alerte « Programmes expirant dans les 30 prochains jours »**

Liste des programmes dont la date de fin tombe dans les 30 jours (nom, organisme, date de fin), avec l'icône `AlertTriangle`. Si aucun programme n'expire, un message rassurant s'affiche avec l'icône `CheckCircle2`.

> La plupart des programmes préchargés **n'ont pas** de date de fin renseignée : ils n'apparaîtront donc jamais dans cette alerte. Seuls quelques-uns (les volets ESSOR, par exemple) portent une échéance. C'est normal.

### 2.2 Onglet « Catalogue »

Le cœur de la découverte. Deux colonnes de cartes de programme, précédées d'une barre de filtres.

**Barre de filtres (5 champs, combinés)**

| Filtre | Valeurs |
|--------|---------|
| **Catégorie** | « Toutes » + les 8 catégories |
| **Type d'aide** | « Tous » + les 5 types (Subvention, Prêt, Crédit d'impôt, Mixte, Garantie de prêt) |
| **Niveau** | « Tous » + Fédéral / Provincial / Municipal / Mixte |
| **Difficulté** | « Toutes » + Facile / Moyen / Complexe |
| **Recherche** | Champ texte (« Programme, organisme... »), avec un délai de 400 ms |

Sous les filtres, un compteur affiche « **{n} programme(s) trouvé(s)** ».

> **À noter** : l'API sait aussi filtrer par **secteur d'activité**, mais ce filtre **n'est pas exposé** dans l'interface du catalogue (seuls les cinq champs ci-dessus le sont). Pour un tri par secteur, utilisez plutôt l'onglet **Éligibilité**.

**Carte de programme** — chaque carte affiche :

- le **nom** du programme et l'**organisme** ;
- un **badge coloré** de type d'aide ;
- la **description** (limitée à trois lignes) ;
- des **badges** de niveau, de difficulté et de catégorie ;
- la **plage de montants** « montant min – montant max » suivie du **pourcentage d'aide** (%) le cas échéant ;
- un **lien téléphone** cliquable (`tel:`) et un **lien externe** vers le site officiel (nouvel onglet) ;
- la **date limite** (« Date limite: {date} ») si le programme en a une ;
- deux boutons : **« Liste de vérification (IA) »** (icône `ClipboardList`) et **« Créer une demande »** (icône `FilePlus2`, bouton primaire).

**État vide** : « Aucun programme ne correspond aux filtres ».

**Modale « Liste de vérification — {nom} » (IA)**

Un clic sur « Liste de vérification (IA) » ouvre une grande modale qui appelle l'IA (`POST /ai/checklist`). Pendant le calcul : « Génération de la liste de vérification en cours... ». L'IA renvoie une **liste de vérification en Markdown** à cases à cocher (`- [ ]`), organisée en **cinq sections** : documents à rassembler, informations à préparer, éléments de la demande, étapes chronologiques et conseils pour maximiser les chances.

> Cette action **consomme des crédits IA**. Un verrou empêche de lancer deux appels IA en même temps : si une requête IA est déjà en cours, le clic est ignoré.
>
> Les programmes préchargés n'ayant **pas** de critères d'éligibilité ni de liste de documents renseignés dans les données (ces champs sont vides), la liste de vérification s'appuie surtout sur la connaissance générale de l'IA et pourra indiquer « Non spécifiés » pour ces éléments. Utilisez-la comme point de départ, pas comme vérité officielle.

### 2.3 Onglet « Éligibilité »

Vérificateur **algorithmique**, instantané et **gratuit** (aucun crédit IA). À ne pas confondre avec l'« Analyse d'éligibilité » de l'Assistant IA (section 2.6), qui est payante.

Un encadré d'information rappelle l'objet de l'outil et précise : « Le vérificateur rapide se base sur vos secteurs d'activité et votre budget. Pour une analyse complète (taille, région, projets), utilisez l'onglet Assistant IA. »

**Formulaire de profil**

| Champ | Type |
|-------|------|
| **Taille de l'entreprise** | menu déroulant, 5 tailles (Travailleur autonome, Micro, Petite, Moyenne, Grande) |
| **Région** | menu déroulant, 18 régions du Québec (17 régions administratives + « Autre ») |
| **Budget approximatif** | nombre (défaut 50 000, pas de 10 000) |
| **Urgence** | menu déroulant, 4 niveaux (Immédiat < 3 mois, Court terme, Moyen terme, Long terme) |
| **Secteurs d'activité** | 19 boutons-pastilles à sélection multiple |
| **Types de projet** | 13 boutons-pastilles à sélection multiple |

Le bouton **« Vérifier mon éligibilité »** est **désactivé tant qu'aucun secteur n'est coché**. Un bouton **« Effacer »** apparaît une fois un résultat obtenu.

**Panneau de résultats**

Titre « **{n} programme(s) potentiellement éligible(s)** », puis les **10 meilleurs** programmes (les mieux pointés), chacun avec :

- un **badge de score** (vert si le score est **≥ 50**, ambre sinon) ;
- le montant « **Jusqu'à {montant}** » ;
- un lien « **Site officiel** » ;
- un bouton **« Créer une demande »**.

Si rien ne ressort : « Aucun programme ne correspond parfaitement. Essayez d'ajuster votre profil. »

**Comment le score est calculé** (voir aussi la section 4.5)

- **+20 points** par secteur d'activité en commun entre votre profil et le programme ;
- **+15 points** si le plafond du programme couvre au moins **10 % de votre budget** — ou si le programme est **sans plafond** (montant maximal nul, c'est-à-dire un programme exprimé en pourcentage ou illimité) ;
- **+25 points bonus** si vous avez coché **Construction** ou **Rénovation** et que le programme vise le secteur **Construction**.

Seuls les programmes au score strictement positif sont retenus, triés du plus élevé au plus faible ; les **10 premiers** sont affichés.

### 2.4 Onglet « Mes demandes »

Registre des demandes de subvention et de leur cycle de vie.

**Barre de commandes** : bouton **« Nouvelle demande »** (primaire), un champ de recherche et un filtre **Statut** (« Tous les statuts » + les 9 statuts). La recherche est **locale** (côté navigateur) et porte sur le nom du programme, la référence, les notes et le statut.

**Carte de demande** — chaque carte affiche :

- la **référence interne** (format `SUB-...`) et un **badge de statut** ;
- le **nom du programme** et l'**organisme** ;
- le **montant demandé** (à droite) et « Accordé: {montant} » si un montant a été accordé ;
- les **notes** (deux lignes maximum) ;
- les **dates** : Créée, Soumise, Décision.

**Boutons d'action conditionnels**

| Action | Disponible quand… |
|--------|-------------------|
| **Détails** | Toujours |
| **Modifier** | Le statut n'est **pas** `ANNULEE` |
| **Soumettre** | Le statut est `BROUILLON` ou `EN_PREPARATION` (avec confirmation) |
| **Supprimer** | Le statut n'est **ni** `APPROUVEE` **ni** `VERSEE` (avec confirmation) |

**État vide** : « Aucune demande de subvention ».

**Modale « Nouvelle demande de subvention »**

- **Programme** (obligatoire, menu déroulant). Lorsqu'on arrive depuis un bouton « Créer une demande » du catalogue ou de l'éligibilité, le programme est **pré-rempli** (voir 3.3).
- **Montant demandé ($)** (nombre).
- **Notes** (zone de texte).

Si le programme n'est pas choisi : erreur « Programme requis ». Une protection empêche le double envoi. La demande est créée au statut `BROUILLON` avec une référence interne générée automatiquement.

**Modale « Modifier {réf} »**

- **Statut** (menu déroulant des 9 statuts) ;
- **Montant demandé ($)** ;
- **Montant accordé ($)** ;
- **Référence externe** (le numéro attribué par l'organisme) ;
- **Notes** ;
- **Motif de refus** (visible **seulement** si le statut choisi est `REFUSEE`).

> **Garde-fou financier** : le montant accordé ne peut **pas** dépasser le montant demandé (le serveur renvoie une erreur 400 sinon). Lorsque vous passez le statut à `APPROUVEE` ou `REFUSEE`, la **date de décision** est estampillée automatiquement ; lorsque vous passez à `VERSEE`, la **date de versement** l'est aussi. Ces dates sont calculées sur le **calendrier local du tenant** (et non l'heure UTC du serveur) et ne remplacent jamais une date que vous auriez saisie à la main.

**Modale de détail « Demande {réf} »**

- Badges de **statut** et de **niveau de gouvernement** ;
- Bloc programme / organisme ;
- Grille : montant demandé, montant accordé, date de soumission, date de décision ;
- Notes, **motif de refus** (encadré rouge s'il est présent), critères du programme ;
- **Section « Analyse IA »** : bouton **« Analyser (IA) »** (`POST /ai/analyze-demande`). L'IA renvoie un **badge « Préparation: {score}/100 »** (vert ≥ 70, ambre ≥ 40, rouge en dessous), un délai estimé de traitement, puis cinq listes — **Points forts**, **Points à améliorer**, **Documents probablement manquants**, **Conseils de rédaction**, **Risques de refus** — et un **conseil global**.
- **Section « Documents »** : bouton **« Téléverser »**, puis la liste des pièces jointes. Chaque ligne affiche le nom, un badge de statut de document, le type et la taille (en Ko) et la date de téléversement, avec par ligne : un **menu de statut** (À fournir / Fourni / Validé / Rejeté), un bouton **Télécharger** et un bouton **Supprimer** (avec confirmation). Si aucun fichier : « Aucun document téléversé ».

**Types de fichiers acceptés pour les pièces jointes** : PDF, Word (DOC, DOCX), Excel (XLS, XLSX), images (JPG, PNG, WebP), texte (TXT) et CSV — soit **10 types MIME**. Taille maximale : **10 Mo** par fichier.

> À la différence du Module 17 (Conformité), qui n'accepte que le PDF et les images, ce module accepte aussi les documents Word, Excel, texte et CSV — pratique pour joindre un plan d'affaires, des projections financières ou un chiffrier.

### 2.5 Onglet « Ressources »

Trois sections d'information, servies par `GET /resources`.

**Conseils pratiques (2 blocs)**

- **Étapes recommandées** (5 conseils) : commencer par sa MRC (point d'entrée officiel) ; cumuler les programmes (maximum 80 % des dépenses admissibles) ; préparer son dossier (états financiers, plan d'affaires, projections) ; respecter les délais ; consulter un expert (les conseillers de MRC sont gratuits).
- **Points importants** (4 constats) : cumul maximal de 80 % des dépenses admissibles ; en 2024-2025, 95 % de l'aide directe d'Investissement Québec va aux PME ; le Québec compte environ 230 000 PME (99,7 % du tissu industriel) ; les programmes changent fréquemment — vérifier les sites officiels.

**Organismes partenaires (8 cartes)** : chaque carte porte le nom, le rôle (en italique), un contact téléphonique s'il existe et un lien « Site » s'il existe.

| Organisme | Rôle | Contact / site |
|-----------|------|----------------|
| Réseau Accès PME | 500+ professionnels pour l'accompagnement | Via votre MRC |
| Investissement Québec | Administration des programmes ESSOR et autres | 1 844 474-6367 |
| SADC | Sociétés d'aide au développement des collectivités | reseau-sadc.qc.ca |
| APCHQ | Association des professionnels de la construction | apchq.com |
| MicroEntreprendre | Microcrédit aux entrepreneurs | microentreprendre.ca |
| Annuaire des subventions | 2 696 programmes de soutien financier | subventionsquebec.net |
| Gouvernement du Canada | Outil de recherche d'aide aux entreprises | canada.ca |
| Gouvernement du Québec | Aide financière aux entreprises | quebec.ca |

**Plan PME 2025-2028 (219 M$)** : une carte titrée avec le montant total et un **tableau** à trois colonnes (Programme / Enveloppe / Description) :

| Programme | Enveloppe | Description |
|-----------|-----------|-------------|
| ESSOR | 136 M$ | Reconduction du programme |
| Réseau accès PME | 22,6 M$ | 450 conseillers en développement économique |
| MicroEntreprendre | 12,7 M$ | Services de microcrédit |
| Espaces PME innovation | 14,4 M$ | Accompagnement de projets novateurs |
| Groupes sous-représentés | 14,88 M$ | Formation et accompagnement |
| Repreneuriat | 17 M$ | Transfert d'entreprises |

### 2.6 Onglet « Assistant IA »

Sous-navigation en pilules, avec un encadré d'introduction. Chaque appel **consomme des crédits IA** ; l'indicateur « L'IA analyse votre demande... » s'affiche pendant le traitement. Trois sous-outils :

**1. « Suggérer des programmes »** (`POST /ai/suggest`)

- Zone de texte **« Décrivez votre projet »** + champ **« Budget estimé ($) »** + bouton « Suggérer des programmes ».
- Résultat : un **montant total potentiel** (gros chiffre), deux colonnes **« Programmes fédéraux »** et **« Programmes provinciaux »**, deux listes **« Crédits d'impôt »** et **« Autres aides »**, un encadré **« Stratégie de financement »** et un encadré d'avertissement **« Points à surveiller »**.
- L'IA s'appuie sur sa connaissance générale des programmes québécois et canadiens : elle peut donc mentionner des programmes **absents** du catalogue préchargé.

**2. « Analyse d'éligibilité »** (`POST /ai/analyze-eligibility`)

- Menus **Secteur**, **Taille**, **Région**, champs **« Chiffre d'affaires ($) »** et **« Nombre d'employés »**, pastilles **« Projets prévus »**, bouton « Analyser mon éligibilité (IA) » (désactivé tant qu'aucun secteur n'est choisi).
- Résultat : un **montant total potentiel**, une liste **« Programmes recommandés »** (chacun avec un badge de score de compatibilité, la raison, le montant potentiel, la difficulté d'obtention et les actions requises à cocher), une liste **« Programmes à éviter »**, un encadré **« Stratégie recommandée »** et une liste ordonnée **« Prochaines étapes »**.

**3. « Discuter »** (`POST /ai/chat`)

- Un chat avec un **« Expert en subventions »** : bulles de conversation, saisie au clavier (Entrée pour envoyer), bouton « Effacer », champ « Posez votre question sur les subventions... ». État vide : « Posez une question pour démarrer la conversation. »
- L'historique de conversation est conservé même si vous changez d'onglet (contrairement aux autres résultats IA, qui sont purgés).

> **Deux « analyses d'éligibilité » à ne pas confondre** : celle de **l'onglet Éligibilité** (section 2.3) est **algorithmique, gratuite et instantanée** ; celle de **l'Assistant IA** ci-dessus est **propulsée par Claude, payante** et plus riche (elle tient compte de la taille, de la région, du chiffre d'affaires et des projets). Commencez par la version gratuite, puis affinez avec l'IA au besoin.

---

## 3. Workflows pas à pas

### 3.1 Découvrir les programmes accessibles

1. Onglet **Catalogue**.
2. Appliquer les filtres : **Catégorie** (par exemple « Construction & Rénovation » ou « Énergie & Environnement »), **Niveau** (Provincial ou Fédéral), **Difficulté** (commencer par « Facile », puis « Moyen »).
3. Au besoin, taper un mot dans **Recherche** (nom de programme ou organisme).
4. Sur une carte intéressante : cliquer le **lien téléphone** pour appeler l'organisme, ou le **lien externe** pour ouvrir le site officiel dans un nouvel onglet.

### 3.2 Vérifier son éligibilité (gratuit)

1. Onglet **Éligibilité**.
2. Remplir le profil : **Taille**, **Région**, **Budget**, **Urgence**, cocher au moins un **Secteur** (par exemple Construction et Rénovation) et des **Types de projet**.
3. Cliquer **« Vérifier mon éligibilité »** (le bouton reste désactivé tant qu'aucun secteur n'est coché).
4. Lire les **10 meilleurs** programmes, du score le plus élevé au plus faible. Un badge vert (score ≥ 50) signale les meilleures correspondances.
5. Cliquer « Créer une demande » sur un programme retenu, ou « Site officiel » pour en savoir plus.

> **Astuce** : si vous cochez Construction ou Rénovation, tout programme visant le secteur Construction reçoit un bonus de 25 points — il remontera donc en tête de liste.

### 3.3 Créer une demande

1. Depuis le **Catalogue** ou l'**Éligibilité**, cliquer **« Créer une demande »** sur un programme : l'application bascule sur l'onglet **Mes demandes** et ouvre la modale avec le **programme déjà rempli**. (On peut aussi partir de zéro avec le bouton **« Nouvelle demande »** et choisir le programme dans le menu.)
2. Saisir le **Montant demandé** et des **Notes** au besoin.
3. Cliquer **« Créer la demande »**. La demande est créée au statut `BROUILLON` avec une **référence interne** du type `SUB-20260707143052-00031`.

*Rappel de permission : réservé à l'administrateur ou au rôle « comptable ».*

### 3.4 Téléverser des pièces justificatives

1. Onglet **Mes demandes** → **Détails** sur la demande → section **Documents** → **« Téléverser »**.
2. Choisir un fichier : PDF, Word, Excel, image (JPG/PNG/WebP), texte ou CSV, de **10 Mo maximum**.
3. Le serveur valide le **type** (sinon erreur 415) et la **taille** (sinon erreur 413), puis stocke le fichier en base.
4. La pièce apparaît dans la liste, au statut **Fourni** par défaut.

> **Bonne pratique** : joindre le document original (le PDF signé reçu par courriel, le chiffrier d'états financiers, le plan d'affaires) plutôt qu'une photo, surtout pour les programmes exigeants comme la SCHL ou Investissement Québec.

### 3.5 Suivre le statut d'un document

Dans la section Documents d'une demande, le **menu de statut** de chaque ligne offre quatre valeurs : **À fournir** (gris, réservation), **Fourni** (bleu, défaut au téléversement), **Validé** (vert, conforme) ou **Rejeté** (rouge, à refaire). Le bouton **Télécharger** récupère le fichier ; le bouton **Supprimer** l'efface définitivement (avec confirmation).

### 3.6 Marquer une demande comme « soumise »

> **Important** : cette action ne dépose **rien** auprès de l'organisme. Elle ne fait que passer le statut **interne** à `SOUMISE` pour votre suivi.

1. Sur une carte au statut `BROUILLON` ou `EN_PREPARATION`, cliquer **« Soumettre »** et confirmer.
2. Le statut passe à `SOUMISE` et la **date de soumission** est estampillée (calendrier local du tenant).
3. **Action externe réelle** : présenter le dossier sur le portail officiel du programme.
4. De retour dans l'ERP, ouvrir **Modifier** pour renseigner la **Référence externe** (le numéro attribué par l'organisme).

### 3.7 Suivre le cycle de vie d'une demande

Progression typique :

```
BROUILLON → EN_PREPARATION → SOUMISE → EN_EVALUATION
  → INFO_SUPPLEMENTAIRE → APPROUVEE / REFUSEE → VERSEE
```

À chaque étape, ouvrir **Modifier** et mettre à jour le **statut**, le **montant accordé** (si approuvée), le **motif de refus** (si refusée) et la **référence externe**. Les dates de décision et de versement sont posées automatiquement quand le statut change (voir 2.4).

> **Aucune machine d'états** : rien ne vous empêche techniquement de sauter des étapes. Suivez le cycle naturel par discipline, pour que vos indicateurs restent cohérents.

### 3.8 Analyser la préparation d'une demande avec l'IA

1. Onglet **Mes demandes** → **Détails** sur la demande → section **« Analyse IA »** → **« Analyser (IA) »**.
2. Lire le **score de préparation sur 100** (vert ≥ 70, ambre ≥ 40, rouge sinon), le délai estimé, puis les points forts, les points à améliorer, les documents probablement manquants, les conseils de rédaction et les risques de refus.
3. Corriger le dossier en conséquence **avant** de le présenter à l'organisme.

*Cette action consomme des crédits IA.*

### 3.9 Obtenir des suggestions et une stratégie de financement (IA)

1. Onglet **Assistant IA** → **« Suggérer des programmes »**.
2. Décrire le projet et saisir un budget estimé, puis lancer la suggestion.
3. Lire les programmes fédéraux et provinciaux, les crédits d'impôt, les autres aides, le montant total potentiel et la **stratégie de financement** proposée (l'IA tient compte du cumul de programmes).
4. Pour une lecture croisée avec votre profil complet, utiliser **« Analyse d'éligibilité »**.

### 3.10 Comptabiliser un versement (manuel)

> **Le module ne crée aucune écriture comptable.**

Quand une demande passe à `VERSEE` : aller dans le **Module 15 (Comptabilité)** → Journal → **Nouvelle écriture**. Enregistrer la réception des fonds (débit de l'encaisse, crédit d'un compte de revenu de subvention). Reporter la **référence interne** de la demande dans le libellé de l'écriture pour faciliter le rapprochement.

### 3.11 Supprimer une demande

1. Onglet **Mes demandes** → **Supprimer** sur la carte (avec confirmation).
2. Le serveur **refuse** la suppression si le statut est `APPROUVEE` ou `VERSEE` (erreur 400). Pour ces demandes, utilisez plutôt le statut `ANNULEE` via **Modifier** si vous devez les écarter.
3. Sinon, la demande est supprimée définitivement et ses **documents** sont effacés en cascade.

> Il n'y a **pas** de corbeille ni de restauration : la suppression est définitive.

---

## 4. Référence

### 4.1 Les six onglets

| Clé interne | Libellé affiché | Contenu réel |
|-------------|-----------------|--------------|
| `dashboard` | Tableau de bord | Indicateurs, graphiques, alerte d'expiration |
| `catalogue` | Catalogue | 47 programmes + liste de vérification IA + création de demande |
| `eligibilite` | Éligibilité | Vérificateur algorithmique gratuit |
| `demandes` | Mes demandes | Registre des demandes + documents + analyse IA |
| `ressources` | Ressources | Conseils, organismes, Plan PME |
| `assistant` | Assistant IA | Suggérer / Analyser l'éligibilité / Discuter |

### 4.2 Points d'accès de l'API (23 au total, préfixe `/api/erp/v1/subventions`)

**Métadonnées (2)** — lecture, tout utilisateur du tenant :

| Méthode + chemin | Rôle |
|------------------|------|
| `GET /constants` | Énumérations et listes de référence pour l'interface |
| `GET /resources` | 8 organismes + Plan PME 2025-2028 + conseils pratiques |

**Catalogue — lecture seule (4)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `GET /categories` | tout utilisateur |
| `GET /programmes` (filtres catégorie, type, niveau, difficulté, secteur, recherche) | tout utilisateur |
| `GET /programmes/expiring?days=30` (bornes 1-365) | tout utilisateur |
| `GET /programmes/{id}` | tout utilisateur |

> Aucun `POST` / `PUT` / `DELETE` n'existe sur les programmes ou les catégories : **le catalogue n'est pas modifiable via l'interface**.

**Demandes (6)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `GET /demandes` (filtre statut) | tout utilisateur |
| `GET /demandes/{id}` (joint le programme et les documents) | tout utilisateur |
| `POST /demandes` | administrateur ou comptable |
| `PUT /demandes/{id}` | administrateur ou comptable |
| `POST /demandes/{id}/soumettre` | administrateur ou comptable |
| `DELETE /demandes/{id}` | administrateur ou comptable |

**Documents (4)** :

| Méthode + chemin | Garde |
|------------------|-------|
| `POST /demandes/{id}/documents` (multipart, 10 Mo) | administrateur ou comptable |
| `GET /documents/{id}/download` | tout utilisateur |
| `PUT /documents/{id}/status` | administrateur ou comptable |
| `DELETE /documents/{id}` | administrateur ou comptable |

**Statistiques et éligibilité (2)** :

| Méthode + chemin | Rôle |
|------------------|------|
| `GET /statistics` | Indicateurs et 4 répartitions (tout utilisateur) |
| `POST /eligibility-check` | Vérificateur algorithmique — **aucune IA, aucun crédit** (tout utilisateur) |

**Assistant IA (5)** — tout utilisateur du tenant, blocage réel par les crédits :

| Méthode + chemin | Rôle |
|------------------|------|
| `POST /ai/suggest` | Suggérer des programmes selon une description de projet |
| `POST /ai/chat` | Chat avec un expert en subventions |
| `POST /ai/checklist` | Liste de vérification (Markdown) pour un programme du catalogue |
| `POST /ai/analyze-demande` | Analyse de la préparation d'une demande |
| `POST /ai/analyze-eligibility` | Analyse d'éligibilité approfondie selon un profil complet |

> **Note d'architecture** : les données de référence (enums, catalogue, organismes, Plan PME, conseils, consigne système de l'IA) proviennent de `subventions_data.py`, qui est un simple **module de données** — aucun point d'accès n'y est défini. Tous les points d'accès vivent dans `subventions.py`, monté directement sous `/api/erp/v1/subventions`.

### 4.3 Les 9 statuts d'une demande

| Statut | Libellé affiché | Couleur | Posé par |
|--------|-----------------|---------|----------|
| `BROUILLON` | Brouillon | gris | Automatique à la création |
| `EN_PREPARATION` | En préparation | ambre | Manuel |
| `SOUMISE` | Soumise | bleu | Automatique via « Soumettre » |
| `EN_EVALUATION` | En évaluation | violet | Manuel |
| `INFO_SUPPLEMENTAIRE` | Info requise | orange | Manuel |
| `APPROUVEE` | Approuvée | vert | Manuel |
| `REFUSEE` | Refusée | rouge | Manuel (saisir le motif de refus) |
| `ANNULEE` | Annulée | gris foncé | Manuel |
| `VERSEE` | Versée | vert foncé | Manuel |

Règles de cycle de vie :

- **Soumission** autorisée **seulement** depuis `BROUILLON` ou `EN_PREPARATION` (erreur 400 sinon).
- **Suppression** bloquée si le statut est `APPROUVEE` ou `VERSEE` (erreur 400).
- **Modification du statut** validée contre la liste ci-dessus (erreur 400 pour une valeur inconnue).
- **Estampillage automatique** (calendrier local du tenant, jamais l'UTC) : date de décision quand le statut passe à `APPROUVEE` ou `REFUSEE` ; date de versement quand il passe à `VERSEE`. Une date saisie manuellement n'est jamais écrasée.

### 4.4 Les 4 statuts d'un document

| Statut | Libellé | Couleur | Sens |
|--------|---------|---------|------|
| `A_FOURNIR` | À fournir | gris | Emplacement réservé, pièce attendue |
| `FOURNI` | Fourni | bleu | Défaut au téléversement |
| `VALIDE` | Validé | vert | Vérifié et conforme |
| `REJETE` | Rejeté | rouge | À refaire |

### 4.5 Vérificateur d'éligibilité (barème algorithmique)

Pour **chaque** programme actif, le score part de 0 et cumule :

| Règle | Points |
|-------|--------|
| Par secteur d'activité en commun (comparaison insensible à la casse) | +20 chacun |
| Le plafond du programme couvre ≥ 10 % du budget, **ou** le programme est sans plafond (montant maximal nul ou négatif) | +15 |
| Profil Construction ou Rénovation **et** programme visant le secteur Construction | +25 |

- Seuls les programmes au score **strictement positif** sont retenus, triés du plus élevé au plus faible ; les **10 premiers** sont renvoyés.
- **Gratuit et instantané** : aucun crédit IA n'est consommé.
- Le traitement spécial des programmes « sans plafond » (montant maximal à 0) corrige un ancien défaut où ces programmes — pourtant souvent les plus généreux — se voyaient injustement privés du bonus de 15 points.

### 4.6 Indicateurs du tableau de bord (calculs exacts)

| Indicateur | Calcul |
|------------|--------|
| Programmes actifs | Nombre de programmes actifs |
| Demandes totales | Nombre total de demandes |
| Montant demandé | Somme des montants demandés, **toutes** demandes **sauf** `ANNULEE` |
| Montant accordé | Somme des montants accordés, demandes `APPROUVEE` ou `VERSEE` |
| Programmes par catégorie | Décompte par catégorie (jointure catalogue) |
| Programmes par niveau | Décompte fédéral / provincial / municipal / mixte |
| Demandes par statut | Décompte par statut |
| Programmes expirant (30 j) | Programmes dont la date de fin tombe dans les 30 prochains jours |

### 4.7 Les 8 catégories

| Code | Libellé | Objet |
|------|---------|-------|
| `PME_GENERAL` | PME & Entreprises | Programmes généraux pour PME |
| `CONSTRUCTION` | Construction & Rénovation | Programmes du secteur de la construction |
| `ENERGIE` | Énergie & Environnement | Efficacité énergétique et développement durable |
| `FORMATION` | Formation & Emploi | Formation et développement des compétences |
| `INNOVATION` | Innovation & Technologie | R&D et transformation numérique |
| `REGIONAL` | Développement Régional | Programmes régionaux et municipaux |
| `DEMARRAGE` | Démarrage & Repreneuriat | Création et reprise d'entreprises |
| `EXPORT` | Exportation | Programmes pour l'exportation |

### 4.8 Énumérations de référence

- **Types d'aide (5)** : Subvention · Prêt · Crédit d'impôt · Mixte · Garantie de prêt.
- **Niveaux de gouvernement (4)** : Fédéral · Provincial · Municipal · Mixte.
- **Difficultés (3)** : Facile · Moyen · Complexe.
- **Secteurs d'activité (19)** : PME, Construction, Rénovation, Manufacturier, Énergie, Logement, Commercial, Résidentiel, Numérique, Formation, Employeur, Exportateur, Startup, Démarrage, Repreneuriat, Rural, Faible revenu, Patrimoine, Bois.
- **Régions (18)** : Bas-Saint-Laurent, Saguenay–Lac-Saint-Jean, Capitale-Nationale, Mauricie, Estrie, Montréal, Outaouais, Abitibi-Témiscamingue, Côte-Nord, Nord-du-Québec, Gaspésie–Îles-de-la-Madeleine, Chaudière-Appalaches, Laval, Lanaudière, Laurentides, Montérégie, Centre-du-Québec, Autre.
- **Tailles d'entreprise (5)** : Travailleur autonome · Micro (1-4 employés) · Petite (5-49 employés) · Moyenne (50-199 employés) · Grande (200+ employés).
- **Types de projet (13)** : Démarrage, Expansion, Modernisation, Transformation numérique, Efficacité énergétique, Formation, Exportation, Repreneuriat, Rénovation, Équipement, R&D, Embauche, Énergie verte.
- **Niveaux d'urgence (4)** : Immédiat (< 3 mois) · Court terme (3-6 mois) · Moyen terme (6-12 mois) · Long terme (> 12 mois).

### 4.9 Catalogue des 47 programmes préchargés

Répartition : **47 programmes** au total (PME & Entreprises 6, Construction & Rénovation 6, Énergie & Environnement 9, Formation & Emploi 6, Innovation & Technologie 6, Développement Régional 4, Démarrage & Repreneuriat 4, Exportation 6).

> **Correction vs l'ancien manuel** : le compte réel est **47**, et non « 50+ ». Les commentaires internes du code disent même « 40+ » et **sous-estiment** ; le décompte exact des programmes est 47.

**PME & Entreprises (6)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| ESSOR – Volet 1 Études | Investissement Québec | Subvention · Provincial | 100 000 $ · 50 % |
| ESSOR – Volet 2 Productivité | Investissement Québec | Mixte · Provincial | 5 M$ · 50 % |
| ESSOR – Volet 3 Environnement | Investissement Québec | Mixte · Provincial | 2 M$ · 50 % |
| ESSOR – Volet 4 International | Investissement Québec | Mixte · Provincial | 1 M$ · 50 % |
| Fonds locaux d'investissement (FLI) | MRC locales | Prêt · Municipal | 5 000 $ – 500 000 $ |
| Financement PME BDC | Banque de développement du Canada | Prêt · Fédéral | 10 000 $ – 5 M$ |

**Construction & Rénovation (6)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| Fonds logement abordable – Construction | SCHL | Prêt · Fédéral | 50 M$ · jusqu'à 95 % |
| Fonds logement abordable – Rénovation | SCHL | Mixte · Fédéral | 10 M$ |
| Rénovation écoénergétique (immeubles collectifs) | SCHL | Subvention · Fédéral | 170 000 $ / logement |
| Certification Novoclimat | Transition Énergétique Québec | Subvention · Provincial | Sans plafond · prime 25 % |
| Programme Maisons Canada | Gouvernement du Canada | Subvention · Fédéral | 10 M$ |
| Crédit d'impôt RenoVert | Revenu Québec | Crédit d'impôt · Provincial | 10 000 $ · 20 % |

**Énergie & Environnement (9)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| LogisVert | Hydro-Québec | Subvention · Provincial | 22 000 $ |
| Rénoclimat | Transition Énergétique Québec | Subvention · Provincial | 20 000 $ |
| Prêt canadien maisons plus vertes | Gouvernement du Canada | Prêt · Fédéral | 40 000 $ (sans intérêt) |
| Initiative maisons plus vertes | Gouvernement du Canada | Subvention · Fédéral | 5 000 $ |
| Chauffez Vert | Transition Énergétique Québec | Subvention · Provincial | 15 000 $ |
| ÉcoPerformance | Transition Énergétique Québec | Subvention · Provincial | 100 000 $ · 50 % |
| Technoclimat | Transition Énergétique Québec | Subvention · Provincial | 5 M$ · 50 % |
| RénoRégion | SHQ | Subvention · Provincial | 25 000 $ |
| Éconologis | Transition Énergétique Québec | Subvention · Provincial | Services gratuits · 100 % |

**Formation & Emploi (6)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| PACME – Formation PME | Emploi-Québec | Subvention · Provincial | 100 000 $ · 50 % |
| Crédit d'impôt pour apprenti | Gouvernement du Canada | Crédit d'impôt · Fédéral | 2 000 $ / apprenti |
| Crédit d'impôt stage en milieu de travail | Revenu Québec | Crédit d'impôt · Provincial | Sans plafond · 30 % |
| Crédit d'impôt formation PME | Revenu Québec | Crédit d'impôt · Provincial | 5 460 $ / employé |
| Mesure de formation de la main-d'œuvre (MFOR) | Services Québec | Subvention · Provincial | 100 000 $ · 75 % |
| Subvention salariale | Emploi-Québec | Subvention · Provincial | 50 000 $ · 50 % |

**Innovation & Technologie (6)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| Innovation PARI-CNRC | CNRC-PARI | Subvention · Fédéral | 500 000 $ · 80 % |
| RS&DE (recherche et développement) | Agence du revenu du Canada | Crédit d'impôt · Fédéral | 3 M$ · 35 % (PME) |
| PCAN – Croître en ligne | Gouvernement du Canada | Subvention · Fédéral | 2 400 $ |
| PCAN – Adoption technologique | Gouvernement du Canada | Mixte · Fédéral | 15 000 $ + prêt 0 % |
| ESSOR – Volet numérique | Investissement Québec | Subvention · Provincial | 50 000 $ · 50 % |
| Offensive transformation numérique (OTN) | MEI | Subvention · Provincial | 100 000 $ · 50 % |

**Développement Régional (4)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| SADC – Développement économique régional | DEC Canada | Mixte · Fédéral | 250 000 $ |
| Rénovation de façades commerciales | Municipalités | Subvention · Municipal | 66 000 $ · 50 % |
| Restauration de bâtiments patrimoniaux | Municipalités | Subvention · Municipal | 100 000 $ · 50 % |
| Dispositifs antirefoulement | Ville de Québec | Subvention · Municipal | 5 000 $ · 50 % |

**Démarrage & Repreneuriat (4)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| MicroEntreprendre | MicroEntreprendre | Prêt · Provincial | 500 $ – 20 000 $ |
| Programme Relève entreprise | MEI | Subvention · Provincial | 100 000 $ |
| Repreneuriat Québec | MEI | Subvention · Provincial | 50 000 $ |
| Campus du repreneuriat | MEI | Subvention · Provincial | 25 000 $ |

**Exportation (6)**

| Programme | Organisme | Type · Niveau | Montant max · Aide |
|-----------|-----------|---------------|--------------------|
| CanExport PME | Gouvernement du Canada | Subvention · Fédéral | 75 000 $ · 50 % |
| Export Québec | Investissement Québec | Mixte · Provincial | 100 000 $ · 50 % |
| Programme Frontière (tarifs douaniers) | Investissement Québec | Prêt · Provincial | 50 M$ |
| Chantier productivité | Investissement Québec | Mixte · Provincial | 5 M$ |
| Programme IRRT | MEI | Subvention · Provincial | 500 000 $ · 50 % |
| BDC – Bois d'œuvre | BDC | Prêt · Fédéral | 10 M$ |

> Les noms, montants, taux, téléphones et liens exacts vivent dans le jeu de données interne. Plusieurs programmes affichent un **montant maximal nul** : ce sont des programmes **sans plafond** (aide en pourcentage ou services), traités comme tels par le vérificateur d'éligibilité. Les montants sont **indicatifs** et changent souvent — validez sur les sites officiels.

### 4.10 Format de la référence interne

**`SUB-AAAAMMJJHHMMSS-NNNNN`** — par exemple `SUB-20260707143052-00031`.

- La partie horodatage est bâtie sur la date de création (horodatage universel), la partie `NNNNN` est l'identifiant de la demande complété à 5 chiffres.
- La référence est générée de façon **atomique** (insertion avec renvoi de l'identifiant, puis mise à jour), donc sans collision possible.
- La **référence externe** est un champ libre distinct, réservé au numéro attribué par l'organisme (par exemple `ESSOR-2026-12345`).

### 4.11 Validations et codes d'erreur

| Règle ou limite | Réponse HTTP |
|-----------------|--------------|
| Programme / projet / entreprise / demande / document introuvable | 404 |
| Corps de mise à jour vide ou sans champ valide | 400 |
| Statut hors de la liste des 9 statuts | 400 |
| Soumission depuis un statut autre que `BROUILLON` / `EN_PREPARATION` | 400 |
| Suppression d'une demande `APPROUVEE` ou `VERSEE` | 400 |
| Montant accordé supérieur au montant demandé | 400 |
| Montants hors de 0 à 1 000 000 000 | 422 |
| Notes de plus de 5000 caractères ; motif de refus de plus de 2000 ; référence externe de plus de 255 | 422 |
| Date mal formée dans une mise à jour | 422 |
| Fichier vide | 400 |
| Fichier de plus de 10 Mo | 413 |
| Type de fichier hors de la liste autorisée | 415 |
| Crédits IA épuisés | 402 |
| Accès IA refusé (garde de facturation) | 403 |
| Service IA indisponible (SDK absent) | 503 |
| IA surchargée (« overload ») | 503 |
| Requête IA trop volumineuse | 413 |
| Réponse IA vide, non JSON ou de format inattendu | 502 |

- Bornes de saisie de l'IA : description de projet 1 à 5000 caractères ; question de chat 1 à 2000 ; contexte jusqu'à 10 000 ; listes de secteurs et de types jusqu'à 50 entrées ; nombre d'employés et chiffre d'affaires bornés.
- Le paramètre `days` de `GET /programmes/expiring` est borné de 1 à 365.

### 4.12 Coûts de l'IA

- **Modèle** : `claude-opus-4-8`, 32 000 jetons maximum par appel. (Un ancien commentaire du code mentionne encore « Opus 4.6 » ; le modèle réellement configuré est bien Opus 4.8.)
- **Tarifs de base** : 5 $ US par million de jetons en entrée, 25 $ par million en sortie, 6,25 $ par million pour l'écriture de cache, 0,50 $ par million pour la lecture de cache.
- **Majoration** : × 1,30 (30 %).
- **Le débit se fait APRÈS validation de la réponse** : un appel qui échoue, revient vide ou mal formé **n'est pas facturé** (la réponse JSON est analysée avant toute facturation).
- **Limite de débit dédiée** : 10 appels IA par minute et par adresse IP sur les chemins `/subventions/ai/` — c'est la classe de points d'accès la plus coûteuse.
- **Le vrai contrôle d'accès à l'IA, ce sont les crédits.** La garde de facturation interne (`check_ai_guard`) laisse en pratique passer tout utilisateur authentifié ; c'est la **vérification du solde de crédits prépayés** qui bloque (erreur 402 « Crédits IA épuisés »). Le seul lien indirect avec Stripe est une recharge automatique de crédits lorsque le solde tombe sous un seuil.

### 4.13 Tables PostgreSQL (schéma du tenant)

Les quatre tables sont créées **à la demande** (à la première requête), pas au moment de la création du tenant. Le catalogue par défaut (8 catégories, 47 programmes) est **semé une seule fois**, de façon idempotente, au premier accès.

| Table | Contenu et particularités |
|-------|---------------------------|
| `subventions_categories` | 8 catégories semées ; `code` unique, ordre d'affichage, actif |
| `subventions_programmes` | 47 programmes semés ; type d'aide, niveau, montants, pourcentage, `secteurs_admissibles` en JSONB, dates, difficulté ; index unique partiel sur le code |
| `subventions_demandes` | Demandes ; `reference_interne` (format `SUB-...`), statut, montants, dates de soumission / décision / versement, notes, motif de refus. Les liens vers le programme, le projet et l'entreprise sont validés **applicativement** (pas de contrainte de clé étrangère en base) |
| `subventions_documents` | Pièces jointes ; `fichier_data` en BYTEA, type MIME, taille, statut ; suppression **en cascade** avec la demande |

> **Semis partiel** : les champs « critères d'éligibilité », « documents requis », « courriel » et « notes » des programmes ne sont **pas** remplis par le semis (ils sont vides). C'est pourquoi la liste de vérification IA et l'analyse de demande peuvent afficher « Non spécifiés » pour ces éléments.

---

## 5. Intégrations et FAQ

### 5.1 Intégration avec la Comptabilité (Module 15)

- **Aucune écriture comptable automatique.** Un versement (`statut = VERSEE`, montant accordé, date de versement) n'est pas porté au grand livre.
- À comptabiliser manuellement : réception des fonds (débit de l'encaisse, crédit d'un compte de revenu de subvention). Reportez la **référence interne** de la demande dans le libellé de l'écriture.
- Les crédits d'impôt (RS&DE, RenoVert, crédit formation, crédit apprenti) se suivent ici pour mémoire, mais leur traitement fiscal se fait avec votre comptable et dans votre déclaration.

### 5.2 Intégration avec la Conformité (Module 17)

- **Aucune jointure** entre les demandes et les licences RBQ, cartes CCQ ou attestations.
- Plusieurs programmes (SCHL, Maisons Canada, Investissement Québec) **exigent** une licence RBQ valide et des attestations fiscales à jour. Vérifiez-les dans le Module 17 **avant** de soumissionner, et joignez-en une copie dans la section Documents de la demande.

### 5.3 Intégration avec les Projets et le CRM

- Une demande peut porter un identifiant de **projet** et un identifiant d'**entreprise** (client). Ces liens sont validés à la création et à la modification si les tables correspondantes existent, mais **il n'y a pas d'écran de rattachement** dans l'interface actuelle : ils ne sont pas exposés comme champs éditables. Il n'y a pas non plus de filtre par projet dans la liste des demandes.
- Les liens ne portent **aucune contrainte de clé étrangère** en base : l'intégrité repose sur la validation applicative.

### 5.4 Intégration avec les Documents (Module 7)

- Les pièces jointes des demandes sont stockées **dans les tables du module Subventions** (en BYTEA), séparément du module Dossiers. Il n'y a pas de partage de fichiers entre les deux.

### 5.5 Intégration IA et crédits (Module 25)

- Les 5 outils IA passent par le **même mécanisme de crédits prépayés** que les autres fonctions IA de l'ERP (voir le Module 25).
- Le coût est journalisé après chaque appel réussi ; un appel qui échoue n'est pas facturé.
- Aucune intégration avec QuickBooks, l'agent vocal Vapi ou le module SEAOP dans ce module.

### 5.6 Ce que le module ne fait pas (rappel)

Pas de dépôt officiel aux organismes, pas de connexion aux registres, pas d'écriture comptable automatique, pas d'exportation ni d'impression ni de rapport, pas de création de programme par l'utilisateur, pas de rappel par courriel ou par calendrier, pas de versement partiel, pas de machine d'états, pas de processus d'approbation ni de signature électronique.

### 5.7 FAQ

**Q : Le module dépose-t-il ma demande auprès de l'organisme quand je clique « Soumettre » ?**
R : **Non.** « Soumettre » ne fait que passer le statut interne à `SOUMISE` pour votre suivi. Vous devez présenter le dossier vous-même sur le portail officiel du programme.

**Q : Qui peut créer et modifier des demandes ?**
R : L'**administrateur** du tenant ou une personne au rôle **comptable** (le super-administrateur aussi). Un utilisateur ordinaire peut **consulter** les demandes, lancer le vérificateur d'éligibilité gratuit et **utiliser l'IA payante**, mais il ne peut **pas** créer, modifier, soumettre ni supprimer une demande. (C'est une correction : les versions antérieures affirmaient à tort que tout le monde pouvait tout faire.)

**Q : Combien de programmes sont préchargés ?**
R : **47**, répartis en 8 catégories (voir 4.9). L'ancienne documentation disait « 50+ » et les commentaires internes du code disent « 40+ » — les deux sont inexacts.

**Q : Combien y a-t-il d'onglets ?**
R : **Six** : Tableau de bord, Catalogue, Éligibilité, Mes demandes, Ressources et Assistant IA. L'ancien manuel n'en décrivait que cinq (l'onglet Assistant IA a été ajouté).

**Q : Quelle est la différence entre l'onglet Éligibilité et l'« Analyse d'éligibilité » de l'Assistant IA ?**
R : L'onglet **Éligibilité** est un vérificateur **algorithmique, gratuit et instantané** (pointage par secteur et budget, top 10). L'« Analyse d'éligibilité » de l'**Assistant IA** est **propulsée par Claude, payante** et plus riche (elle intègre la taille, la région, le chiffre d'affaires et les projets). Commencez par la gratuite.

**Q : Puis-je exporter mes demandes en Excel, en CSV ou en PDF, ou imprimer un rapport ?**
R : **Non.** Il n'existe aucune exportation, aucune impression et aucune génération de rapport. Le seul téléchargement possible est une **pièce jointe** que vous avez vous-même téléversée.

**Q : Puis-je ajouter ou modifier un programme dans le catalogue ?**
R : **Non.** Le catalogue est en lecture seule et alimenté par un jeu de données interne. Pour suivre un programme absent, créez une demande sur un programme approchant et documentez la différence dans les Notes, ou faites ajouter le programme par l'équipe qui maintient l'ERP.

**Q : Pourquoi la liste de vérification IA affiche-t-elle « Non spécifiés » pour les critères et les documents ?**
R : Parce que ces champs ne sont pas remplis dans les données préchargées des programmes. L'IA s'appuie alors sur sa connaissance générale. Consultez le site officiel du programme pour la liste exacte.

**Q : Que se passe-t-il si je saisis un montant accordé supérieur au montant demandé ?**
R : Le serveur refuse (erreur 400). Le montant accordé ne peut pas dépasser le montant demandé, pour garder les indicateurs financiers cohérents.

**Q : Puis-je supprimer une demande approuvée ou versée ?**
R : **Non** (erreur 400). Utilisez plutôt le statut `ANNULEE` via Modifier si vous devez l'écarter. Les demandes supprimables le sont **définitivement**, avec leurs documents (aucune corbeille).

**Q : Le module m'enverra-t-il un rappel avant l'échéance d'un programme ?**
R : **Non.** L'alerte des programmes qui expirent vit uniquement dans le tableau de bord (et beaucoup de programmes n'ont pas de date de fin renseignée). Prenez l'habitude de le consulter et notez les échéances importantes dans votre agenda.

**Q : Quels fichiers puis-je téléverser, et jusqu'à quelle taille ?**
R : PDF, Word (DOC/DOCX), Excel (XLS/XLSX), images (JPG, PNG, WebP), texte (TXT) et CSV — 10 types au total, jusqu'à **10 Mo** par fichier. Au-delà, compressez le fichier : il n'y a pas de compression automatique.

**Q : L'IA peut-elle me suggérer des programmes qui ne sont pas dans le catalogue ?**
R : **Oui.** Les outils « Suggérer des programmes » et « Analyse d'éligibilité » s'appuient sur la connaissance générale de Claude et peuvent mentionner des programmes absents du catalogue préchargé. Pour un pointage exact sur tout le catalogue, utilisez l'onglet Éligibilité.

**Q : Suis-je facturé si la réponse de l'IA est mauvaise ?**
R : **Non.** Le débit intervient après validation de la réponse : un appel qui échoue, revient vide ou de format inattendu n'est pas facturé.

**Q : Un utilisateur non-administrateur peut-il faire grimper la facture IA ?**
R : **Oui, en théorie** : les cinq outils IA sont accessibles à tout utilisateur connecté (dès qu'il reste des crédits), même sans droit d'écriture sur les demandes. Une limite de 10 appels par minute et par adresse IP borne les abus, et le solde de crédits reste le garde-fou ultime.

**Q : Puis-je enregistrer plusieurs demandes pour un même programme ?**
R : **Oui.** Chaque demande est indépendante ; utilisez plusieurs demandes pour suivre, par exemple, des versements échelonnés ou des exercices différents.

**Q : Le cumul de programmes est-il vérifié par le module ?**
R : **Non.** La règle générale au Québec plafonne le cumul à environ 80 % des dépenses admissibles, mais le module ne la contrôle pas. Vérifiez le cumul avec les organismes concernés (voir les conseils de l'onglet Ressources).

---

## 6. Récapitulatif

- **Rôle** : catalogue de subventions et registre de suivi des demandes pour les entreprises de construction du Québec. **Pas** un module comptable, **pas** un guichet de dépôt officiel.
- **Accès** : barre latérale → groupe TERRAIN → **Subventions** (icône Landmark), route `/subventions`. Onglet ouvert par défaut : **Catalogue**.
- **Six onglets** : Tableau de bord · Catalogue · Éligibilité · Mes demandes · Ressources · Assistant IA. (Correction : l'ancien manuel n'en décrivait que cinq.)
- **47 programmes** préchargés dans **8 catégories** (PME 6, Construction 6, Énergie 9, Formation 6, Innovation 6, Régional 4, Démarrage 4, Export 6), en **lecture seule**.
- **9 statuts de demande** : `BROUILLON` → `EN_PREPARATION` → `SOUMISE` → `EN_EVALUATION` → `INFO_SUPPLEMENTAIRE` → `APPROUVEE` / `REFUSEE` → `VERSEE`, plus `ANNULEE`. Soumission depuis brouillon/préparation seulement ; suppression bloquée si approuvée ou versée ; dates de décision et de versement estampillées automatiquement en heure locale.
- **Permissions** : consultation, vérificateur d'éligibilité gratuit et outils IA pour tous ; **création et modification des demandes et des documents réservées à l'administrateur ou au comptable**.
- **Éligibilité algorithmique** (gratuite) : +20 par secteur en commun, +15 si le plafond couvre ≥ 10 % du budget ou si le programme est sans plafond, +25 bonus construction ; top 10.
- **Cinq outils IA** (Claude Opus 4.8, 32 000 jetons, majoration 30 %, non facturés si la réponse échoue, 10 appels/min par IP) : suggérer des programmes, analyser l'éligibilité, discuter, générer une liste de vérification, analyser une demande. Le vrai blocage, ce sont les **crédits prépayés** (erreur 402).
- **Documents** : téléversement en base jusqu'à 10 Mo, 10 types de fichiers (PDF, Word, Excel, images, texte, CSV), 4 statuts (À fournir / Fourni / Validé / Rejeté).
- **Indicateurs** : Montant demandé = somme de toutes les demandes sauf annulées ; Montant accordé = somme des approuvées et versées.
- **Référence interne** : `SUB-AAAAMMJJHHMMSS-NNNNN`, générée sans collision. **Référence externe** : champ libre pour le numéro de l'organisme.
- **Limites clés** : aucun dépôt officiel, aucune connexion aux registres, aucune écriture comptable automatique, aucune exportation ni impression, catalogue non modifiable, aucun rappel par courriel ou calendrier, aucun versement partiel, aucune machine d'états.
- **23 points d'accès** sous `/api/erp/v1/subventions` ; **4 tables** par tenant créées à la demande.

---

**Documentation générée à partir du code (fichiers vérifiés)** :
- `ERP_REACT/backend/routers/subventions.py` (1687 lignes, 23 points d'accès dont 5 outils IA)
- `ERP_REACT/backend/routers/subventions_data.py` (732 lignes, module de données statiques — 8 catégories, 47 programmes, 8 organismes, Plan PME 2025-2028, 2 blocs de conseils, consigne système de l'IA)
- `ERP_REACT/frontend/src/pages/SubventionsPage.tsx` (1921 lignes, 6 onglets)
- `ERP_REACT/frontend/src/api/subventions.ts`
- `ERP_REACT/frontend/src/store/useSubventionsStore.ts`
- `ERP_REACT/frontend/src/i18n/locales/fr/terrain.json` (libellés de l'interface, section `subventions`)

**Manuels liés** :
- Module 7 (Dossiers — gestion documentaire distincte) — `07-ventes-dossiers.md`
- Module 9 (Projets — lien facultatif projet ↔ demande) — `09-ventes-projets.md`
- Module 15 (Comptabilité — comptabilisation manuelle des versements) — `15-operations-comptabilite.md`
- Module 17 (Conformité RBQ/CCQ — licences et attestations exigées par certains programmes) — `17-terrain-conformite.md`
- Module 19 (Immobilier — programmes SCHL de logement abordable) — `19-terrain-immobilier.md`
- Module 25 (Assistant IA — crédits IA et fonctionnement général de l'IA) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — gestion du tenant, des rôles et des accès) — `28-configuration.md`
