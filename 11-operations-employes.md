# Module 11 — Employés (ressources humaines)

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/EmployeesPage.tsx` (≈ 1 088 lignes, page unique maître-détail), `frontend/src/components/employes/EmployesAssistantTab.tsx` (assistant d'effectif), API `frontend/src/api/employees.ts`
> - Backend : `backend/routers/employees.py` (≈ 2 162 lignes ; fiche employé + révélation du NAS + statistiques ; ce même fichier héberge aussi les points d'accès de pointage `/time-entries*`, mais ceux-ci sont pilotés par le **Module 13 Pointage**, pas par cet écran), `backend/routers/employes_ai.py` (≈ 257 lignes, assistant d'effectif en lecture seule)
> - Préfixe API : `/api/erp/v1` — le CRUD répond sous `/employees` (anglais), l'assistant sous `/employes/ai` (français)
> **Tables PostgreSQL (par tenant)** : `employees` (fiche), `employee_competences` (compétences, **lecture seule** ici), `time_entries` (pointages, **lecture seule** ici — 5 derniers), `nas_decrypt_audit` (journal d'accès au NAS, Loi 25) ; table lue en référence : `payroll_periods`
> **Cadrage** : ce module est le **répertoire des employés** — la fiche des ressources humaines. Il sert à **créer, rechercher, filtrer, modifier et désactiver** les employés, à **exporter la liste en CSV** et à interroger un **assistant IA d'effectif**. La fiche capte aussi les données de **sécurité mobile** (NIP, rôle mobile, droit de gérer le stock) et les **données fiscales** (numéro d'assurance sociale chiffré, adresse) « pour les feuillets T4 / RL-1 ». Il **ne saisit pas** les heures (cela se fait au **Module 13 Pointage**), il **ne calcule pas** la paie ni les feuillets fiscaux (cela se fait au **Module 15 Comptabilité / Paie**), et il **ne supprime jamais** un employé (seule la désactivation existe). Les compétences y sont **affichées** mais **ne s'éditent pas** ici.

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

Le module Employés est le **carnet du personnel** de l'entreprise. Il permet de :

- tenir une **fiche** par employé (identité, coordonnées, poste, département, type de contrat, statut, date d'embauche, taux horaire, salaire, capacité hebdomadaire, photo, notes) ;
- gérer les **données de sécurité du pointage mobile** de chaque employé : son **NIP à 4 chiffres** (pour pointer sur l'application `/mobile`), son **rôle mobile** (droits dans l'application) et son **droit de gérer le stock** (lecture de code-barres et mouvements sur mobile) ;
- capter les **informations fiscales** requises pour les feuillets de fin d'année : le **numéro d'assurance sociale (NAS)**, chiffré au repos, et l'**adresse** postale complète ;
- **rechercher** et **filtrer** les employés (par nom ou courriel, par département, par statut) ;
- **désactiver** un employé qui a quitté (son statut passe à Inactif — il n'est jamais effacé) ;
- **exporter** la liste filtrée en fichier CSV ;
- consulter, en **lecture seule**, les **compétences / certifications** et les **5 derniers pointages** de chaque employé ;
- poser des questions à un **assistant IA d'effectif** qui répond uniquement sur des **totaux non nominatifs** (combien d'employés par département, par poste, par statut).

> **Frontière importante.** Ce module est la **gestion** des employés. Le **Pointage** des heures (saisie, validation, feuille de temps, export) est un module distinct (`/pointage`, Module 13). La **Paie** et les **feuillets T4 / RL-1 / PD7A** vivent dans la **Comptabilité** (Module 15). Le NAS et l'adresse sont **saisis ici**, mais ils ne servent qu'aux traitements de paie situés ailleurs.

### 1.2 Ce que le module ne fait PAS

- **Aucune suppression** d'employé. Il n'existe pas de bouton « Supprimer ». Seule la **désactivation** (statut → Inactif) est possible. Cela préserve l'historique de paie, de pointage et de facturation.
- **Aucun bouton « Réactiver ».** Pour remettre un employé en service, il faut passer par **Modifier → Statut → Actif**.
- **Aucun éditeur de compétences.** Les compétences et certifications sont **affichées** (badges) mais **ne se créent, ni ne se modifient, ni ne se suppriment** depuis cet écran. Le module backend ne fournit d'ailleurs aucune écriture de compétence.
- **Aucune saisie ni validation d'heures.** Les pointages n'apparaissent qu'en **lecture seule** (les 5 plus récents). Toute la gestion des heures (création, validation, feuille hebdomadaire, exportation, coût par projet) se fait au **Module 13 Pointage**.
- **Aucune génération de feuillets fiscaux.** Le NAS et l'adresse sont **captés** ici ; les feuillets T4 / RL-1 / PD7A se **produisent** dans la Comptabilité / Paie.
- **Aucune action groupée.** Pas de multisélection, pas de traitement en lot dans la liste.
- **Assistant IA en lecture seule stricte.** Il n'écrit rien, ne propose aucune action à confirmer, et n'accède à **aucune donnée individuelle** (ni NAS, ni salaire, ni taux horaire, ni liste nominative).
- **Aucun pointage terrain ici.** L'employé pointe ses heures depuis l'application mobile (`/mobile`) avec son NIP ; la saisie sur cet écran de bureau est réservée à la consultation.

### 1.3 Accès

- **Menu latéral** → groupe repliable **Opérations** → **Employés** (icône silhouette cochée). L'entrée se situe entre **Magasin** et **Bons de travail / Pointage**.
- **Adresse** : `/employes`.
- Page protégée : il faut être authentifié dans un tenant.
- **Titre affiché** en haut de la page : « Employés ».

### 1.4 Permissions et rôles

La **consultation** de base est ouverte à tout utilisateur authentifié du tenant. Les données sensibles et les actions d'écriture sont réservées à des rôles précis. Trois paliers coexistent :

| Palier | Qui | Ce qu'il peut faire de plus |
|--------|-----|------------------------------|
| **Consultation** | Tout utilisateur authentifié du tenant | Voir la liste et le détail (nom, poste, département, courriel, téléphone, statut), les cartes de statistiques, les compétences et les 5 derniers pointages, et utiliser l'assistant d'effectif. **Ne voit ni les salaires, ni les taux horaires, ni les adresses, ni le NAS masqué.** |
| **Responsable paie / administrateur** | Administrateur (`is_admin`), ou rôle **admin** / **super-admin** | Tout ce qui précède, **plus** : voir le **salaire**, le **taux horaire**, l'**adresse** et le **NAS masqué** ; **créer**, **modifier** et **désactiver** un employé ; octroyer le **rôle mobile** et le **droit de gérer le stock**. |
| **Administrateur du tenant** | Administrateur du tenant uniquement (`is_admin`, le vrai signal, relu au serveur à chaque requête) | Tout ce qui précède, **plus** : **révéler le NAS en clair** (avec justification et journalisation Loi 25). |

> **Filtre de confidentialité de la paie (règle appliquée au serveur, 2026-07-06).** Pour un utilisateur qui **n'est pas** responsable paie (ni administrateur, ni rôle admin / super-admin), le serveur **retire** de la liste **et** du détail : le salaire, le taux horaire, l'adresse, la ville, le code postal et le NAS masqué. Conséquence concrète : la colonne « Taux h. » et les lignes Salaire / Taux / Adresse / NAS **sont vides** pour un employé standard. Ce n'est pas un bogue — c'est la protection qui empêche un employé de lire la paie et l'adresse de ses collègues via le répertoire.
>
> **Octroi des droits mobiles réservé à l'administrateur.** Un utilisateur ordinaire ne peut pas s'auto-attribuer le droit de gérer le stock ni changer un rôle mobile : ces deux champs sont **désactivés** dans les modales pour un non-administrateur, et le serveur **ignore** silencieusement ces champs si la requête vient d'un non-administrateur.

### 1.5 Cartes de statistiques (KPI)

Trois cartes chiffrées coiffent la page dès que les statistiques sont chargées (source : `GET /employees/statistics`) :

| Carte | Contenu | Couleur |
|-------|---------|---------|
| **Total** | Nombre total d'employés (tous statuts) | neutre |
| **Actifs** | Nombre d'employés au statut **Actif** | vert |
| **Départements** | Nombre de départements distincts **parmi les employés actifs** | mauve |

> La carte **Départements** ne compte que les départements des employés **actifs**. Les valeurs vides ou nulles sont regroupées sous « Non défini » pour éviter le surcomptage.

### 1.6 Concepts clés

- **Fiche employé** : l'enregistrement central. Deux valeurs de rémunération peuvent y figurer : le **taux horaire** (dollars par heure) et le **salaire** annuel. Elles sont facultatives et visibles seulement par les responsables paie.
- **Statut** : l'état d'un employé (Actif, Congé, Formation, Arrêt de travail, Inactif). Seuls les employés **Actifs** entrent dans les traitements de paie du Module 15.
- **NIP mobile** : un code à **4 chiffres** que l'employé saisit pour pointer sur l'application mobile. Il est **haché** (bcrypt) au serveur — jamais stocké en clair. Sur la fiche, on ne voit qu'un badge « Configuré » ou « Non configuré ».
- **Rôle mobile** : le niveau de droits de l'employé dans l'application `/mobile` (Employé, Apprenti, Gestionnaire, Admin).
- **Droit de gérer le stock** : autorise l'employé à lire des codes-barres et à faire des mouvements de stock depuis le mobile.
- **Capacité hebdomadaire** : le nombre d'heures visées par semaine (par défaut 40). Le Module 13 s'en sert pour signaler une semaine en sous-charge ou en dépassement.
- **NAS chiffré (Loi 25)** : le numéro d'assurance sociale est **chiffré au repos**. On n'affiche qu'un **masque** (par exemple `XXX-XXX-1234`). Seul un administrateur du tenant peut le révéler en clair, et chaque révélation est **journalisée**.
- **Informations fiscales** : l'adresse postale (adresse, ville, code postal, province) requise pour produire les feuillets T4 / RL-1.
- **Compétences** : les certifications de l'employé (carte CCQ, RBQ, soudure, etc.), affichées en **lecture seule** sur cet écran.
- **Assistant d'effectif** : un clavardage IA qui répond uniquement à partir de **totaux agrégés non nominatifs** (comptages par département, poste, statut).

---

## 2. Interface

### 2.1 Disposition générale

```
+------------------------------------------------------------------+
|  Titre « Employés »                                              |
+------------------------------------------------------------------+
|  [Nouvel employé] [Assistant IA] [Exporter CSV]   Recherche  ▾Dept ▾Statut |  <- barre de commandes
+------------------------------------------------------------------+
|  [ Total ]        [ Actifs ]        [ Départements ]             |  <- 3 cartes KPI
+----------------------------+-------------------------------------+
|  Liste des employés        |  Panneau de détail (à droite)       |
|  (tableau paginé, triable) |  (fiche + compétences + pointages)  |
+----------------------------+-------------------------------------+
```

La page est une vue **maître-détail** : la liste à gauche, le panneau de détail à droite dès qu'on sélectionne un employé. Sur téléphone, la liste se replie en cartes, et le détail occupe tout l'écran (avec un bouton retour). Les trois modales (Nouvel employé, Modifier l'employé, Assistant IA) s'ouvrent par-dessus la page.

### 2.2 Barre de commandes

**Actions (à gauche)**

| Bouton | Icône | Effet |
|--------|-------|-------|
| **Nouvel employé** | plus | Ouvre la modale de création. |
| **Assistant IA** | étoiles | Ouvre la modale de l'assistant d'effectif. |
| **Exporter CSV** | téléchargement | Génère et télécharge un fichier CSV de **tous** les employés correspondant aux filtres courants. Le bouton est désactivé pendant l'exportation. |

**Filtres (à droite)**

| Filtre | Comportement |
|--------|--------------|
| **Recherche** | Champ texte (« Rechercher… »). Filtre par **nom complet** ou **courriel**. Remet la pagination à la première page. |
| **Département** | Menu déroulant : « Tous » + les 11 départements. |
| **Statut** | Menu déroulant : « Tous les statuts » + les 5 statuts. |

### 2.3 Liste des employés

**Tableau (bureau)** — colonnes triables et redimensionnables :

| Colonne | Contenu |
|---------|---------|
| **Nom** | Avatar (photo ou initiales) + « prénom nom », avec le courriel en dessous. |
| **Poste** | Le poste, ou `--` si vide. |
| **Dept.** | Le libellé traduit du département. |
| **Statut** | Un badge coloré (vert Actif, ambre Congé, bleu Formation, rouge Arrêt de travail, gris Inactif). |
| **Taux h.** | « {taux} $/h », ou `--`. **Vide pour un utilisateur qui n'est pas responsable paie** (filtre de confidentialité). |

- Un **clic** sur une ligne charge le détail à droite ; la ligne active est surlignée.
- **Liste vide** : « Aucun employé ».
- **Pagination** : 20 employés par page ; les contrôles n'apparaissent qu'au-delà d'une page.
- **Tri** : cliquer un en-tête trie la colonne ; les largeurs se règlent à la souris (poignée de redimensionnement) ou par ajustement automatique.

**Cartes (téléphone)** : avatar, nom, « poste — département », badge de statut, et le taux horaire (ou, à défaut, le salaire).

### 2.4 Panneau de détail

**En-tête** : « prénom nom », « poste — département », un bouton **Modifier** (crayon), un bouton **fermer** (X, ou une flèche « retour à la liste » sur mobile), et le **badge de statut**.

**Informations affichées** (chacune n'apparaît que si elle est renseignée) :

- **Courriel** (icône enveloppe) et **Téléphone** (icône combiné).
- **Type de contrat** (libellé traduit).
- **Taux horaire** « {x} $/h » — *réservé aux responsables paie*.
- **Salaire** (formaté en dollars) — *réservé aux responsables paie*.
- **Embauche** : la date d'embauche.

**Bloc de badges fonctionnels** :

- **NIP** : « Configuré » ou « Non configuré ».
- **Approbateur** : présent si l'employé a le droit d'approuver les heures.
- **Gestion stock** : présent si l'employé peut gérer le stock au mobile.
- **Rôle mobile** : présent si le rôle mobile n'est pas « Employé », avec le libellé du rôle.

**NAS et sa révélation** :

- Le **NAS** s'affiche **masqué** (par exemple `XXX-XXX-1234`) — *réservé aux responsables paie*.
- Le bouton **Révéler le NAS** (icône œil) n'apparaît **que pour l'administrateur du tenant**. Le clic ouvre un champ **Raison** (« …audit Loi 25, minimum 10 caractères ») accompagné de deux boutons **Révéler** / **Annuler** et d'un avertissement : « Accès audité (Loi 25). Toute consultation est tracée. » Le bouton **Révéler** reste désactivé tant que la raison compte moins de 10 caractères. Une fois révélé, le NAS s'affiche en clair avec un badge « NAS en clair ».

**Adresse** (icône épingle) : la concaténation adresse, ville, code postal, province — *réservée aux responsables paie*.

**Désactiver** : le bouton **Désactiver** (icône silhouette barrée) n'apparaît que si le statut n'est pas déjà Inactif. Le clic affiche une **confirmation en ligne** (« Désactiver cet employé (statut Inactif) ? ») avec un bouton **Désactiver** (rouge) et **Annuler**. La confirmation fait passer le statut à **Inactif**.

**Sections additionnelles (lecture seule)** :

- **Compétences** (avec le compte) : des badges — **vert** si la compétence est certifiée, gris sinon. Aucun ajout ni édition ici.
- **Pointages récents** : les **5 derniers** pointages (date + heures). Aucune création ni validation ici — cela se passe au Module 13.

### 2.5 Modale « Nouvel employé »

Un bandeau d'erreur dédié s'affiche en haut en cas de problème. Les champs, dans l'ordre :

1. **Photo** : aperçu rond, boutons « Ajouter une photo » / « Changer la photo » et « Retirer ». L'image (`image/*`) est **compressée côté navigateur** avant l'envoi (état « Traitement… » ; message « Image impossible à traiter… » en cas d'échec).
2. **Prénom \*** / **Nom \*** (obligatoires).
3. **Courriel** (type courriel) / **Téléphone**.
4. **Poste** / **Département** (menu déroulant, 11 options).
5. **Type de contrat** (menu déroulant, 7 options) / **Statut** (menu déroulant, 5 options).
6. **Date d'embauche**.
7. **Taux horaire ($/h)** / **Salaire annuel ($)** (nombres, pas de 0,01).
8. **Capacité hebdomadaire (h/sem)** (nombre de 0 à 168, exemple : 40).
9. **Section « Sécurité pointage mobile »** (icône clé) :
   - **Code NIP (4 chiffres)** : champ masqué, chiffres seulement, longueur exacte de 4. Un avertissement « Le NIP doit contenir 4 chiffres » s'affiche si vous saisissez 1 à 3 chiffres.
   - Case **« Peut approuver les heures »**.
   - Case **« Peut gérer le stock (scan mobile) »** — **désactivée pour un non-administrateur**.
10. **Rôle mobile (application /mobile)** (menu déroulant) : « Employé (limité) », « Apprenti (limité) », « Gestionnaire (tous les droits) », « Admin (tous les droits) » — **désactivé pour un non-administrateur**.
11. **Section « Coordonnées et informations fiscales (T4 / RL-1) »** (icône épingle) :
    - **NAS** (« Numéro d'assurance sociale (NAS) », 9 chiffres). Note affichée : « Chiffré au repos (Loi 25). Requis pour les feuillets T4 / RL-1. » La saisie n'accepte que chiffres, espaces et tirets.
    - **Adresse** ; puis **Ville** / **Code postal** / **Province**.
12. **Notes** (zone de texte, 3 lignes).

**Boutons** : **Annuler** / **Créer**. Le bouton **Créer** reste désactivé tant que le prénom ou le nom est vide, ou tant que le NIP est incomplet (1 à 3 chiffres). Fermer la modale vide le formulaire.

### 2.6 Modale « Modifier l'employé »

Structure **identique** à la création (mêmes champs 1 à 12), avec ces différences :

- Le champ **NAS** est **laissé vide** ; s'il existe déjà un NAS, l'aide de saisie affiche « NAS enregistré (masqué) ». **Laisser vide = ne pas modifier** le NAS existant.
- Le champ **NIP** affiche « •••• » si un NIP est déjà configuré ; laisser tel quel conserve le NIP.
- Le bouton d'action s'appelle **Enregistrer** (avec un indicateur de progression).

### 2.7 Modale « Assistant IA — Effectif RH »

- **En-tête** : « Assistant IA — Effectif RH », sous-titre « Vue d'effectif à partir d'agrégats non nominatifs (lecture seule). »
- **État vide** : trois exemples de questions (« Combien d'employés ai-je au total ? », « Quelle est la répartition par département ? », « Combien d'employés actifs par poste ? ») et une **note de confidentialité** : l'assistant « n'accède à aucune donnée individuelle sensible (NAS, salaires, taux horaires) ».
- **Clavardage** : bulles utilisateur / assistant, indicateur « Analyse en cours… », zone de saisie (« Pose ta question sur l'effectif… ») et bouton **Envoyer** (la touche Entrée sans Maj envoie). Un verrou empêche le double envoi.
- **Métadonnées** affichées sous chaque réponse : profil « RH », nombre de jetons, coût, durée.
- **Bilingue** (français / anglais selon la langue de l'interface).

---

## 3. Workflows pas à pas

### 3.1 Créer un employé

1. Cliquer **Nouvel employé** dans la barre de commandes.
2. Renseigner au minimum le **Prénom** et le **Nom** (les seuls champs obligatoires).
3. Compléter au besoin : coordonnées, poste, département, type de contrat, statut (par défaut **Actif**), date d'embauche, taux horaire ou salaire, capacité hebdomadaire.
4. Cliquer **Créer**. L'employé apparaît dans la liste et un bandeau « Employé créé » confirme l'opération.

> Réservé aux **responsables paie / administrateurs**. Un utilisateur ordinaire ne voit pas cette action aboutir (le serveur refuse la création).

### 3.2 Ajouter ou changer la photo

1. Dans la modale de création ou de modification, section **Photo**, cliquer **Ajouter une photo** (ou **Changer la photo**).
2. Choisir un fichier image. L'application le **compresse** automatiquement (état « Traitement… »).
3. L'aperçu rond se met à jour. Pour l'enlever, cliquer **Retirer**.
4. Enregistrer la fiche. L'avatar (photo ou, à défaut, initiales) apparaît ensuite dans la liste et le détail.

### 3.3 Configurer le pointage mobile (NIP, rôle, droit stock)

1. Ouvrir la fiche en **Modification**, aller à la section **Sécurité pointage mobile**.
2. Saisir un **NIP à 4 chiffres** : ce sera le code de l'employé pour pointer sur l'application `/mobile`.
3. Cocher **« Peut approuver les heures »** si l'employé doit pouvoir valider des pointages.
4. Si vous êtes **administrateur** : cocher **« Peut gérer le stock »** et choisir le **Rôle mobile** approprié. (Ces deux contrôles sont désactivés pour un non-administrateur.)
5. Enregistrer. Le NIP est **haché** au serveur ; la fiche n'affiche ensuite qu'un badge « Configuré ».

> **Après un changement de sous-chemin de l'application mobile, les employés peuvent devoir se réabonner aux notifications** (voir Module 13). Le NIP, lui, reste valide.

### 3.4 Saisir les informations fiscales (NAS et adresse)

1. Ouvrir la fiche en **Modification**, aller à la section **Coordonnées et informations fiscales (T4 / RL-1)**.
2. Saisir le **NAS** (9 chiffres). Le système **valide** le numéro (contrôle de Luhn) et le **chiffre** avant de l'enregistrer ; en cas de numéro invalide, la sauvegarde est refusée.
3. Renseigner l'**adresse**, la **ville**, le **code postal** et la **province**.
4. Enregistrer. La fiche n'affichera plus qu'un **NAS masqué** ; l'adresse n'est visible que des responsables paie.

> Ces données ne servent qu'à la **paie** et aux **feuillets** produits dans la Comptabilité (Module 15). Les saisir ici est un prérequis à la génération des T4 / RL-1.

### 3.5 Modifier un employé

1. Sélectionner l'employé, puis cliquer le crayon **Modifier** dans l'en-tête du détail.
2. Ajuster les champs voulus. Pour le **NAS**, laisser le champ **vide** conserve le numéro existant ; pour le **NIP**, laisser « •••• » conserve le code existant.
3. Cliquer **Enregistrer**. Un bandeau « Modifications enregistrées » confirme.

### 3.6 Désactiver un employé (départ)

1. Sélectionner l'employé (son statut doit être différent d'Inactif).
2. Dans le panneau de détail, cliquer **Désactiver**.
3. Confirmer dans l'invite en ligne (« Désactiver cet employé (statut Inactif) ? »).
4. Le statut passe à **Inactif** et un bandeau « Employé désactivé » confirme. L'employé **reste** dans le système (son historique est préservé), mais il sort des listes filtrées sur les actifs et des traitements de paie.

> Il n'y a **pas de suppression**. La désactivation est la façon prévue de retirer un employé du service.

### 3.7 Réactiver un employé

1. Retrouver l'employé (filtrer sur le statut **Inactif** au besoin).
2. Cliquer **Modifier**.
3. Remettre le **Statut** à **Actif** (ou un autre statut actif comme Congé / Formation).
4. **Enregistrer**. Il n'existe pas de bouton « Réactiver » dédié — c'est la voie normale.

### 3.8 Révéler le NAS en clair (administrateur, Loi 25)

1. Être **administrateur du tenant**. Sélectionner l'employé.
2. Dans le panneau de détail, à la ligne NAS, cliquer **Révéler le NAS**.
3. Saisir une **raison** d'au moins 10 caractères (par exemple : « Production du feuillet RL-1 2026 »).
4. Cliquer **Révéler**. Le système **journalise** l'accès (qui, quand, la raison, l'adresse IP) **avant** de déchiffrer, puis affiche le NAS en clair avec le badge « NAS en clair ».

> **Conformité.** Si le journal d'audit ne peut pas être écrit, le déchiffrement est **bloqué** (erreur 503) — sans trace, pas d'accès. La réponse n'est jamais mise en cache par le navigateur. Toute consultation est traçable.

### 3.9 Rechercher et filtrer

1. Taper un nom ou un courriel dans la **Recherche** : la liste se restreint immédiatement (et revient à la page 1).
2. Choisir un **Département** et/ou un **Statut** dans les menus déroulants pour affiner.
3. Combiner les trois filtres au besoin. Les filtres s'appliquent aussi à l'exportation CSV.

### 3.10 Exporter la liste en CSV

1. Régler les filtres (recherche, département, statut) pour cibler les employés voulus.
2. Cliquer **Exporter CSV**. L'application récupère **tous** les employés correspondants (par lots), pas seulement la page affichée.
3. Le fichier **`employes_export.csv`** se télécharge. Colonnes : ID, Prénom, Nom, Courriel, Téléphone, Poste, Département, Statut, Type de contrat, Date d'embauche, Taux horaire.

> Le CSV **ne contient ni le salaire, ni le NAS**. Pour un utilisateur qui n'est pas responsable paie, la colonne **Taux horaire** ressort **vide** (filtre de confidentialité). Le fichier est encodé en UTF-8 (avec BOM) pour s'ouvrir proprement dans Excel au Québec.

### 3.11 Consulter les compétences et les pointages récents

1. Sélectionner l'employé.
2. Dans le détail, la section **Compétences** liste les certifications (badge vert = certifiée).
3. La section **Pointages récents** montre les **5 derniers** pointages (date + heures).
4. Ces deux sections sont en **lecture seule**. Pour saisir des heures, aller au Module 13 Pointage ; les compétences ne s'éditent pas dans l'ERP.

### 3.12 Interroger l'assistant d'effectif

1. Cliquer **Assistant IA** dans la barre de commandes.
2. Poser une question sur les **effectifs** (« Combien d'employés actifs par poste ? »).
3. L'assistant répond à partir des **totaux agrégés** (par département, poste, statut). Le coût et la durée s'affichent sous la réponse.
4. Si vous demandez un NAS, un salaire, un taux horaire ou une liste nominative, l'assistant **refuse poliment** : ces informations lui sont inaccessibles par conception.

---

## 4. Référence

### 4.1 Référentiels

**11 départements** (miroir du backend `DEPARTEMENTS`) :

`CHANTIER`, `STRUCTURE_BETON`, `CHARPENTE_BOIS`, `FINITION`, `MECANIQUE_BATIMENT`, `ELECTRICITE`, `INGENIERIE`, `QUALITE_CONFORMITE`, `ADMINISTRATION`, `COMMERCIAL`, `DIRECTION`.

> Les quatre métiers `STRUCTURE_BETON`, `CHARPENTE_BOIS`, `FINITION`, `MECANIQUE_BATIMENT` (avec `CHANTIER` et `ELECTRICITE`) déterminent, au moment de la paie (Module 15), si l'employé relève de la CCQ. Bien classer le département ici est donc utile en aval.

**5 statuts** (`STATUTS`, défaut **ACTIF**) :

| Statut | Couleur du badge | Entre en paie (Module 15) |
|--------|------------------|----------------------------|
| **Actif** (`ACTIF`) | vert | Oui |
| **Congé** (`CONGE`) | ambre | Non |
| **Formation** (`FORMATION`) | bleu | Non |
| **Arrêt de travail** (`ARRET_TRAVAIL`) | rouge | Non |
| **Inactif** (`INACTIF`) | gris | Non |

**7 types de contrat** (`TYPES_CONTRAT`, défaut **CDI**) :

`CDI`, `CDD`, `TEMPORAIRE`, `SAISONNIER`, `CONSULTANT`, `STAGE`, `APPRENTISSAGE`.

**4 rôles mobiles** (droits dans l'application `/mobile`, défaut **Employé**) :

| Rôle | Libellé dans l'interface |
|------|--------------------------|
| `EMPLOYE` | Employé (limité) |
| `APPRENTI` | Apprenti (limité) |
| `MANAGER` | Gestionnaire (tous les droits) |
| `ADMIN` | Admin (tous les droits) |

### 4.2 Champs de la fiche et limites de saisie

| Champ | Type / bornes |
|-------|----------------|
| Prénom, Nom | Texte (obligatoires). |
| Courriel, Téléphone | Texte. |
| Poste | Texte libre. |
| Département | Une des 11 valeurs (accepté librement au backend). |
| Statut | Une des 5 valeurs (validé ; défaut Actif). |
| Type de contrat | Une des 7 valeurs (validé ; défaut CDI). |
| Date d'embauche | Date. |
| Taux horaire | Nombre ≥ 0, au plus 100 000. |
| Salaire | Nombre ≥ 0, au plus 100 000 000. |
| Capacité hebdomadaire | Nombre de 0 à 168 (heures/semaine). |
| NIP | Exactement 4 chiffres ; **haché** (bcrypt) au serveur. |
| Rôle mobile | Un des 4 rôles ; octroi **réservé à l'administrateur**. |
| Droit de gérer le stock | Oui / Non ; octroi **réservé à l'administrateur**. |
| Photo | Image compressée (donnée intégrée) ; limite technique élevée (anti-abus). |
| NAS | Au plus 20 caractères ; **validé** (Luhn, 9 chiffres) ; **chiffré** au repos. |
| Adresse / Ville / Code postal / Province | Textes (limites respectives 300 / 120 / 12 / 60 caractères). |
| Notes | Texte libre. |

### 4.3 Points d'accès (API)

Points d'accès de **gestion RH** (base `/api/erp/v1/employees`) :

| Méthode + chemin | Rôle requis | Rôle |
|------------------|-------------|------|
| `GET /employees` | Tout utilisateur du tenant | Liste paginée + recherche + filtres (paie masquée si non-responsable). |
| `GET /employees/{id}` | Tout utilisateur du tenant | Détail + compétences + pointages récents (paie masquée si non-responsable). |
| `GET /employees/statistics` | Tout utilisateur du tenant | Totaux pour les cartes KPI. |
| `POST /employees` | Responsable paie / administrateur | Créer un employé. |
| `PUT /employees/{id}` | Responsable paie / administrateur | Modifier (renvoie 404 si l'employé n'existe pas). |
| `POST /employees/{id}/reveal-nas` | **Administrateur du tenant** | Révéler le NAS en clair (audité, fail-closed). |

Assistant d'effectif (base `/api/erp/v1/employes/ai`) :

| Méthode + chemin | Rôle requis | Rôle |
|------------------|-------------|------|
| `POST /employes/ai/chat` | Tout utilisateur du tenant (+ garde IA + crédits) | Répond sur les agrégats d'effectif. Aucun accès individuel. |

> Les points d'accès de **pointage** (`/employees/time-entries*`, `/employees/payroll-summary`) résident dans le même fichier backend mais sont **documentés au Module 13 Pointage**. Cet écran ne les utilise qu'en lecture pour les « 5 derniers pointages ».

### 4.4 Sécurité du NAS (Loi 25)

| Mécanisme | Comportement |
|-----------|--------------|
| **Validation** | Contrôle de Luhn sur 9 chiffres ; numéro invalide → refus (erreur 422). |
| **Chiffrement** | Chiffré au repos (Fernet). Seuls le numéro **chiffré** et les **4 derniers chiffres** sont stockés ; **jamais** le NAS en clair. Si le chiffrement est indisponible, la sauvegarde échoue plutôt que de stocker en clair. |
| **Affichage** | Toujours masqué (`XXX-XXX-1234`) dans la liste et le détail ; retiré pour les non-responsables paie. |
| **Révélation** | Réservée à l'administrateur du tenant, sur **justification** (≥ 10 caractères). |
| **Audit** | Chaque révélation est journalisée (qui, quand, raison, IP) **avant** le déchiffrement. Si le journal échoue, le déchiffrement est **bloqué** (503). |
| **Non-mise en cache** | La réponse contenant le NAS clair porte l'en-tête « ne pas mettre en cache ». |
| **Journaux** | Tout motif à 9 chiffres est expurgé des journaux techniques. |

### 4.5 Assistant d'effectif — fonctionnement et coût

| Aspect | Détail |
|--------|--------|
| **Portée** | Vue d'effectif seulement : totaux par département, poste et statut. |
| **Accès aux données** | **Aucun outil SQL.** Le modèle ne reçoit qu'un contexte agrégé préparé par des requêtes fixes qui ne lisent **jamais** de colonne sensible (NAS, salaire, taux horaire, date de naissance). |
| **Écriture** | Aucune. Lecture seule stricte. |
| **Refus** | Refuse toute demande de NAS, salaire, taux horaire ou liste nominative. |
| **Modèle** | `claude-sonnet-4-6`, réponse plafonnée à 4 000 jetons, un seul appel (pas de boucle d'outils). |
| **Débit** | Coût = (jetons d'entrée × 0,003 $ + jetons de sortie × 0,015 $) ÷ 1000, puis × 1,30 (majoration 30 %). Débité des **crédits IA prépayés** du tenant. Un solde épuisé renvoie une erreur 402 (recharge requise) ; le super-administrateur est exempté. |
| **Cadence** | Limite de 20 requêtes par minute et par adresse IP. |

### 4.6 Règles et validations retenues

| Règle | Effet |
|-------|-------|
| Prénom ou nom vide | Création / modification refusée. |
| Statut hors liste | Refusé (validation). |
| Type de contrat hors liste | Refusé (validation). |
| Taux horaire / salaire négatif | Refusé (bornes de saisie). |
| Capacité hebdomadaire hors 0–168 | Refusée. |
| NIP autre que 4 chiffres | Bloque le bouton **Créer** ; refusé au serveur. |
| NAS invalide (Luhn) | Refusé (422). |
| Raison de révélation < 10 caractères | Bouton **Révéler** désactivé ; refusé au serveur. |
| Modifier un employé inexistant | Erreur 404 (pas de faux succès). |
| Écriture par un non-responsable paie | Refusée (création / modification / révélation NAS). |
| Rôle mobile / droit stock envoyés par un non-administrateur | Silencieusement ignorés. |
| Mode consultation (abonnement en lecture seule) | Toutes les écritures sont bloquées (403). |

---

## 5. Intégrations et FAQ

### 5.1 Module 13 — Pointage

- Le **pointage des heures** (saisie, validation, feuille de temps hebdomadaire, coût par projet, exportation CSV des heures) se fait au Module 13, à l'adresse `/pointage`.
- La **capacité hebdomadaire** saisie ici alimente les indicateurs de sous-charge / dépassement de la feuille de temps du Module 13.
- Le **NIP** et le **rôle mobile** définis ici gouvernent le pointage terrain sur l'application `/mobile`.
- Sur cet écran, seuls les **5 derniers pointages** apparaissent, en lecture seule.

### 5.2 Module 15 — Comptabilité / Paie

- Le **NAS** et l'**adresse** captés ici sont les prérequis des **feuillets T4, RL-1 et PD7A**, produits dans la Comptabilité.
- La génération d'un feuillet **déchiffre** le NAS de façon **audité** (même mécanisme de journalisation que la révélation manuelle).
- **Nuance d'accès** : la révélation manuelle du NAS (cet écran) est réservée à l'**administrateur** du tenant, tandis que la production des feuillets RL-1 est ouverte à l'**administrateur ou au comptable**. Deux portes différentes vers le même numéro, chacune journalisée.
- Le statut **Actif** est ce qui fait entrer un employé dans les traitements de paie ; le **département** détermine l'applicabilité CCQ.

### 5.3 Application mobile (`/mobile`)

- Les employés pointent depuis la PWA mobile avec leur **NIP à 4 chiffres**.
- Le **rôle mobile** (Employé / Apprenti / Gestionnaire / Admin) et le **droit de gérer le stock** contrôlent ce que l'employé peut faire dans l'application (pointer, approuver, lire les codes-barres du stock).

### 5.4 Module 10 — Magasin

- Le **droit de gérer le stock** octroyé ici autorise l'employé à effectuer des mouvements de stock et de la lecture de code-barres depuis le mobile. La gestion du catalogue et des mouvements côté bureau reste dans le Magasin.

### 5.5 Assistant IA (portée)

- L'assistant d'**effectif** de ce module est **distinct** de l'Assistant IA général de l'ERP (Module 25). Il est volontairement **cloisonné** : aucune requête libre en base, aucune donnée individuelle. Pour toute question nominative ou sensible, il renvoie l'utilisateur vers le module Employés / Paie et ses contrôles Loi 25.

### 5.6 FAQ

**Q : Comment supprimer un employé ?**
R : On ne supprime pas un employé. On le **désactive** (bouton **Désactiver**, statut → Inactif), ce qui préserve tout son historique. C'est volontaire.

**Q : J'ai désactivé quelqu'un par erreur, comment le remettre en service ?**
R : **Modifier** sa fiche et remettre le **Statut** à **Actif**. Il n'y a pas de bouton « Réactiver » séparé.

**Q : Pourquoi je ne vois pas les taux horaires, les salaires ni les adresses ?**
R : Parce que vous n'êtes pas **responsable paie**. Par protection de la vie privée (Loi 25), le serveur retire le salaire, le taux horaire, l'adresse et le NAS masqué de la liste **et** du détail pour tout utilisateur qui n'est ni administrateur, ni de rôle admin / super-admin. La colonne « Taux h. » et ces lignes de détail restent alors **vides**. Il ne s'agit pas d'un bogue.

**Q : Pourquoi la colonne « Taux horaire » de mon exportation CSV est-elle vide ?**
R : Même raison : l'exportation applique le même filtre de confidentialité. Un responsable paie qui exporte verra la colonne remplie.

**Q : Comment ajouter une compétence ou une certification à un employé ?**
R : Ce n'est **pas possible** depuis cet écran. Les compétences y sont uniquement **affichées** (badges, vert si certifiée). Aucune saisie de compétence n'est implémentée dans l'ERP pour l'instant.

**Q : Où est-ce que je saisis les heures de travail ?**
R : Au **Module 13 Pointage** (`/pointage`). Cet écran ne montre que les **5 derniers** pointages, en lecture seule.

**Q : Comment produire les T4 / RL-1 ?**
R : Dans la **Comptabilité / Paie** (Module 15). Ici, vous ne faites que **saisir** le NAS et l'adresse nécessaires. Assurez-vous que ces champs sont remplis pour chaque employé concerné avant la période de production.

**Q : Qui peut voir le NAS en clair ?**
R : Uniquement l'**administrateur du tenant**, et seulement en fournissant une **raison** (≥ 10 caractères). Chaque révélation est **journalisée**. Un comptable, lui, y accède indirectement au moment de générer un feuillet RL-1 (accès également audité).

**Q : Le NAS est-il stocké en clair quelque part ?**
R : Non. Il est **chiffré au repos** ; seuls le numéro chiffré et les 4 derniers chiffres sont conservés. Si le chiffrement n'est pas disponible, la sauvegarde **échoue** plutôt que d'enregistrer un numéro en clair.

**Q : Pourquoi ma saisie de NAS est-elle refusée ?**
R : Le numéro doit être un NAS **valide** (9 chiffres, contrôle de Luhn). Vérifiez la saisie.

**Q : Je ne suis pas administrateur ; pourquoi les cases « Peut gérer le stock » et « Rôle mobile » sont-elles grisées ?**
R : L'octroi de ces droits est réservé à l'administrateur du tenant. Même en forçant, le serveur ignorerait ces champs.

**Q : L'assistant IA peut-il me donner le salaire de quelqu'un ou modifier une fiche ?**
R : Non. Il est en **lecture seule stricte** et **refuse** toute donnée individuelle (NAS, salaire, taux horaire, liste nominative). Il ne répond que sur des **totaux** (combien d'employés par département / poste / statut) et n'écrit jamais rien.

**Q : Puis-je traiter plusieurs employés à la fois (désactivation groupée, etc.) ?**
R : Non. Il n'y a pas d'actions groupées ni de multisélection. Chaque opération se fait employé par employé.

**Q : Le NIP sert-il à se connecter à l'ERP ?**
R : Non. Le NIP à 4 chiffres sert au **pointage sur l'application mobile**. La connexion à l'ERP de bureau utilise le compte utilisateur habituel.

**Q : Que se passe-t-il si je change le taux horaire d'un employé ?**
R : Sur cette fiche, le nouveau taux s'applique aux **futurs** pointages. Les pointages **déjà** enregistrés conservent le taux capté au moment du pointage (le Module 13 fige ce taux pour éviter de recalculer rétroactivement la paie).

---

## 6. Récapitulatif

- Le module **Employés** (`/employes`) est le **répertoire du personnel** : créer, chercher, filtrer, modifier, **désactiver** (jamais supprimer), exporter en CSV.
- **3 cartes KPI** : Total, Actifs, Départements (départements comptés parmi les **actifs**).
- **11 départements**, **5 statuts** (seul **Actif** entre en paie), **7 types de contrat**, **4 rôles mobiles**.
- **Trois paliers de droits** : consultation (tout le monde), responsable paie / administrateur (voit la paie, crée / modifie / désactive), administrateur du tenant (révèle le NAS).
- **Filtre de confidentialité Loi 25** : salaire, taux horaire, adresse et NAS masqué **cachés** aux non-responsables paie — d'où des colonnes / lignes vides pour un employé standard.
- **NAS** : validé (Luhn), **chiffré** au repos, **masqué** à l'affichage ; révélation **auditée** et **fail-closed** (bloquée si le journal échoue), réservée à l'administrateur.
- **NIP mobile** (4 chiffres, haché), **rôle mobile** et **droit de gérer le stock** : octroi réservé à l'administrateur.
- **Compétences** et **5 derniers pointages** : **lecture seule** ici (saisie des heures au Module 13, compétences non éditables).
- **Assistant d'effectif** : **lecture seule stricte**, agrégats non nominatifs, aucun accès individuel, aucune écriture ; débité des crédits IA (majoration 30 %).
- **Pas de suppression, pas de réactivation dédiée, pas d'édition de compétences, pas d'actions groupées, pas de feuillets fiscaux ici.**

---

**Documentation générée à partir du code** : `backend/routers/employees.py`, `backend/routers/employes_ai.py`, `frontend/src/pages/EmployeesPage.tsx`, `frontend/src/components/employes/EmployesAssistantTab.tsx`, `frontend/src/api/employees.ts`.

**Manuels liés** :
- Module 10 (Magasin — droit de gérer le stock) — `10-operations-magasin.md`
- Module 13 (Pointage — saisie et validation des heures) — `13-operations-pointage.md`
- Module 15 (Comptabilité / Paie — feuillets T4 / RL-1 / PD7A) — `15-operations-comptabilite.md`
- Module 25 (Assistant IA général de l'ERP) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — utilisateurs et rôles) — `28-configuration.md`
