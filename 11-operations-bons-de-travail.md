# Module 11 — Bons de travail

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/BonsTravailPage.tsx` (≈ 3 519 lignes, page unique à 4 onglets + vue détail), `frontend/src/components/bt/BtAssistantTab.tsx` (onglet Assistant IA), API `frontend/src/api/production.ts` (≈ 521 lignes), état `frontend/src/store/useProductionStore.ts` (≈ 464 lignes)
> - Backend : `backend/routers/production.py` (≈ 7 209 lignes — fichier **partagé** avec le module Suivi : Kanban, Gantt, Calendrier), `backend/routers/bt_ai.py` (≈ 471 lignes, assistant IA), `backend/routers/operation_templates.py` (≈ 497 lignes, générateur de cédules)
> - Préfixe API : `/api/erp/v1/production` pour le cœur du module ; l'assistant IA vit sous `/api/erp/v1/bons-travail/ai`
> **Tables PostgreSQL (par tenant)** : `formulaires` (un bon de travail = `type_formulaire = 'BON_TRAVAIL'`), `formulaire_lignes` (matériaux), `operations` (tâches), `bt_assignations` (employés assignés), `bt_comments` (commentaires), `operation_types` (catalogue de tâches) ; tables touchées indirectement : `produits` et `mouvements_stock` (mouvements de stock), `time_entries` (pointage, en lecture seule), `dossier_formulaires` (rattachement au dossier d'opportunité) ; table partagée entre tenants : `public.operation_templates` (modèles de cédule).
> **Cadrage** : un **bon de travail** (BT) est l'unité de travail opérationnelle du chantier. Ce module sert à **créer** un BT, à le faire **progresser par statut** (brouillon → en cours → en pause → terminé, ou annulé), à le décomposer en **opérations** (tâches regroupées par phase, avec heures prévues et réelles, employé responsable et sous-traitant), à lister les **matériaux** consommés (chaque ligne liée à l'inventaire **déclenche un mouvement de stock**), à **assigner** des employés, à échanger des **commentaires**, à générer un **document HTML/PDF** imprimable, et à surveiller la **capacité hebdomadaire** de la main-d'œuvre. Un **catalogue** de types d'opérations et un **assistant IA** complètent l'ensemble. Le module **ne génère pas** de cédule d'opérations depuis le formulaire de création (cela passe par le module Projets ou la conversion d'un devis en projet), **ne produit aucune facture ni taxe** (le montant total est informatif), et **ne saisit pas les heures de pointage** (elles se saisissent au Module 12 Pointage). Les mêmes bons de travail apparaissent aussi dans le **Kanban, le Gantt et le Calendrier** du module Suivi, mais ces vues n'appartiennent pas à cette page.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Processus pas à pas](#3-processus-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module Bons de travail est le **poste de pilotage des interventions** de l'entreprise. Il permet de :

- créer un **bon de travail** numéroté (`BT-00001`), optionnellement rattaché à un projet ;
- suivre son **avancement** par un cycle de statuts encadré par une machine à états (démarrer, mettre en pause, reprendre, terminer, annuler) ;
- décomposer le travail en **opérations** (tâches) regroupées par **phase** (fondation, charpente, plomberie…), chacune avec ses heures prévues, ses heures réelles, son employé, son sous-traitant, ses dates et son statut ;
- lister les **matériaux** consommés : chaque ligne liée à un produit du catalogue **débite automatiquement le stock** et laisse une trace dans les mouvements d'inventaire ;
- **assigner** des employés au bon de travail (avec un rôle libre) ;
- tenir un **fil de commentaires** horodaté ;
- générer un **document HTML** professionnel, thématisé aux couleurs de l'entreprise, prêt à imprimer ou à enregistrer en PDF ;
- consulter une **vue globale de toutes les opérations** en cours et un tableau de **capacité hebdomadaire** (charge prévue par rapport à la charge réelle) ;
- gérer un **catalogue** de types d'opérations réutilisables ;
- interroger un **assistant IA** qui lit les bons de travail et peut en **proposer un nouveau** (créé seulement après confirmation).

### 1.2 Ce que le module ne fait PAS

- **Pas de génération de cédule depuis le formulaire de création.** Le bouton « Nouveau bon de travail » crée un BT **vide** (aucune opération pré-remplie). Les cédules par type de projet (résidentiel, rénovation, public) s'appliquent **ailleurs** : par le bouton « Générer la cédule » du module Projets, ou automatiquement à la conversion d'un devis en projet (voir §3.16).
- **Pas de facture, pas de taxes, pas d'export comptable.** Le champ « Montant total » est la somme des lignes de matériaux ; il est **informatif** et n'émet aucune facture. Aucun calcul de TPS/TVQ, aucune marge.
- **Pas de saisie des heures de pointage ici.** Les heures réelles d'une opération se saisissent **à la main**. Le pointage des employés (table `time_entries`) se fait au **Module 12 Pointage** ; aucun mécanisme ne reporte automatiquement les pointages dans les heures réelles des opérations.
- **Le montant total n'est pas modifiable.** Il se recalcule toujours à partir des lignes ; il n'existe aucun champ montant saisissable.
- **L'assistant IA ne crée qu'un bon de travail**, et seulement sur confirmation. Il ne crée ni opération, ni ligne de matériau, ni assignation. En lecture, il n'a pas accès aux données de paie, de ressources humaines ni de sécurité.
- **Pas de suppression définitive directe.** Un bon de travail actif ne peut pas être effacé d'un coup : il faut d'abord l'**annuler** (soft-delete), puis, seulement s'il est annulé, le **supprimer définitivement**.
- **Le Kanban, le Gantt et le Calendrier ne sont pas dans cette page.** Ils affichent bien les bons de travail, mais appartiennent au **Module 02 Suivi**.

### 1.3 Accès

- **Menu latéral** → section **Opérations** → **Bons de Travail** (icône presse-papiers).
- **Adresse** : `/bons-travail`.
- Page protégée : il faut être authentifié dans un tenant.
- **Titre affiché** : « Bons de Travail ».
- **Ouverture directe d'un bon de travail** : un lien du type `/bons-travail?open=<id>` (par exemple depuis le Calendrier) ouvre automatiquement la vue détail du BT concerné.

### 1.4 Permissions et rôles

La **consultation** de la page est ouverte à tout utilisateur authentifié du tenant. L'**écriture** dépend du rôle. Deux gardes distinctes coexistent dans le module :

| Garde | Ce qu'elle protège | Rôles autorisés |
|-------|--------------------|-----------------|
| **Écriture bon de travail** (`require_tenant_admin_or_role`) | Créer / modifier / supprimer / restaurer un BT, changer son statut, gérer ses opérations, ses lignes de matériaux, ses assignations et ses commentaires ; générer une cédule | administrateur (`is_admin`), rôle **admin**, **super-admin**, **gestionnaire** ou **contremaître** |
| **Écriture du catalogue** (`_require_admin`) | Créer / modifier / supprimer un **type d'opération** du catalogue | administrateur (`is_admin`), rôle **admin** ou **super-admin** **uniquement** |

> **Nuance à retenir.** Un **gestionnaire** ou un **contremaître** peut piloter entièrement les bons de travail (opérations, matériaux, assignations, statuts), mais **ne peut pas** modifier le catalogue de types d'opérations : cette dernière action est réservée aux **administrateurs et super-administrateurs**. La garde `require_tenant_admin_or_role` tient toujours compte du statut administrateur du propriétaire (`is_admin`, relu au serveur) afin de ne jamais exclure un patron dont le rôle serait « utilisateur ».
>
> **Mode consultation.** Un tenant en **mode consultation** (abonnement suspendu) est bloqué en amont : toutes les écritures de bons de travail (création, modification, suppression, changement de statut) sont refusées, la lecture reste possible.

### 1.5 Cartes de statistiques (KPI)

Quatre cartes chiffrées coiffent la page en permanence, au-dessus des onglets (source : `GET /production/statistics`) :

| Carte | Contenu | Couleur |
|-------|---------|---------|
| **Total** | Nombre total de bons de travail | neutre |
| **En cours** | Bons au statut **EN_COURS** | bleu |
| **Terminés** | Bons au statut **TERMINE** | vert |
| **Montant total** | Somme des montants (matériaux) de tous les bons, en dollars | primaire |

> Le point d'accès renvoie aussi une répartition par statut et le nombre d'assignations, mais ces valeurs ne sont pas affichées sur les cartes.

### 1.6 Concepts clés

- **Bon de travail (BT)** : l'unité de travail. Techniquement, une ligne de la table `formulaires` avec `type_formulaire = 'BON_TRAVAIL'`. Numéroté `BT-00001` (5 chiffres). Porte un statut, une priorité, un projet, des dates (début, fin, échéance), un montant et des notes.
- **Opération (tâche)** : une étape du bon de travail, rattachée à une **phase**. Elle porte des heures prévues, des heures réelles, un employé, un fournisseur (sous-traitant), un statut et des dates. Les opérations sont **indépendantes** du statut du BT (aucune synchronisation automatique).
- **Phase** (`poste_travail`) : le regroupement d'opérations (par exemple « Fondation », « Charpente »). C'est un simple texte ; le tableau des opérations affiche un bandeau de section par phase.
- **Ligne de matériau** : un article consommé par le bon de travail. Si la ligne est liée à un produit de l'inventaire, elle **débite le stock** (mouvement de sortie) ; sa suppression **recrédite** le stock (mouvement d'entrée).
- **Assignation** : un employé rattaché au bon de travail avec un rôle libre (« Chef d'équipe », « Aide »…). Un employé ne peut être assigné qu'une seule fois par BT.
- **Type d'opération (catalogue)** : un gabarit de tâche réutilisable (nom, catégorie, code, heures standard) qui alimente le menu déroulant « Poste/Opération » des formulaires. Le catalogue est indépendant : renommer ou supprimer un type ne modifie **pas** les opérations déjà créées (elles conservent leur nom en instantané).
- **Cédule automatique** : un bon de travail « Cédule - {projet} » généré à partir d'un modèle selon le type de projet. Elle se déclenche **hors de cette page** (voir §3.16). Un seul BT de cédule automatique par projet.
- **Machine à états** : l'ensemble des transitions de statut autorisées. Elle empêche, par exemple, de passer directement de « Brouillon » à « Terminé », ou de rouvrir un BT terminé (sauf super-administrateur).
- **Invariant de stock** : « bon de travail annulé ⇔ stock restauré ». Annuler un BT recrédite tout le stock consommé ; le restaurer le reconsomme (voir §4.8).

---

## 2. Interface

### 2.1 Disposition générale

```
+------------------------------------------------------------------+
|  Presse-papiers  « Bons de Travail »                             |
+------------------------------------------------------------------+
|  [ Total ]   [ En cours ]   [ Terminés ]   [ Montant total ]     |  <- 4 cartes KPI
+------------------------------------------------------------------+
|  Bons de Travail | Opérations | Catalogue | Assistant IA         |  <- barre d'onglets
+------------------------------------------------------------------+
|  Contenu de l'onglet actif                                       |
+------------------------------------------------------------------+
```

La page comporte **quatre onglets** et une **vue détail** contextuelle :

| Onglet | Clé | Rôle |
|--------|-----|------|
| **Bons de Travail** | `liste` | Liste paginée, filtrable et triable des bons de travail |
| **Opérations** | `operations` | Vue globale de toutes les opérations + capacité hebdomadaire |
| **Catalogue** | `catalogue` | Gestion des types d'opérations réutilisables |
| **Assistant IA** | `assistant` | Clavardage qui interroge les bons de travail et en propose de nouveaux |

> **La vue détail n'est pas un onglet.** Elle s'ouvre lorsqu'on clique un bon de travail ou le bouton « Nouveau bon de travail ». Elle remplace alors la barre d'onglets par un fil d'Ariane : « ← Bons de Travail / {numéro} — {nom} ». Le bouton de retour ramène à la liste.

### 2.2 Onglet « Bons de Travail » (liste)

#### 2.2.1 Barre de commandes

- **Nouveau bon de travail** (bouton principal, icône « + ») : ouvre la vue détail en mode création (pleine page).
- **Recherche** (« Rechercher... ») : filtre en temps réel sur le **nom** et le **numéro** (délai de saisie de 300 ms).
- **Filtre de statut** (menu déroulant) : « Tous les statuts », ou l'un des cinq statuts (Brouillon, En cours, En pause, Terminé, Annulé). Sélection unique.
- **Filtre de priorité** (menu déroulant) : « Toutes les priorités », ou l'une des quatre (Basse, Normale, Haute, Urgente). Sélection unique.

#### 2.2.2 Tableau des bons de travail

Colonnes (triables et redimensionnables) :

| Colonne | Contenu |
|---------|---------|
| **Numéro** | `BT-00001` (police à chasse fixe) |
| **Nom** | Nom du bon (ou nom du projet, ou numéro si laissé vide) |
| **Statut** | Badge de couleur |
| **Priorité** | Badge de couleur |
| **Projet** | Projet associé (si renseigné) |
| **Début** | Date de début prévu — **modifiable directement au clic** (champ date en ligne) |
| **Fin** | Date de fin prévu — **modifiable directement au clic** |
| **Échéance** | Date d'échéance |
| **Montant** | Total des matériaux (aligné à droite) |
| **Actions** | Modifier ; Supprimer (conditionnel) |

**Actions par ligne** :

- **Modifier** (crayon) : ouvre le bon de travail directement en mode édition.
- **Supprimer** (poubelle rouge) : n'apparaît **que si le statut est « Annulé »**. Il déclenche une **suppression définitive** (avec confirmation du navigateur et verrou anti double-clic). Un bon de travail non annulé n'affiche jamais ce bouton dans la liste.

Un clic sur une ligne (hors boutons) ouvre la **vue détail** en lecture. Sous le tableau : un compteur « {n} bon(s) de travail » et la pagination (25 par page). Si la liste est vide : « Aucun bon de travail trouvé ». Sur téléphone, le tableau se replie en cartes équivalentes.

> **Seules les colonnes Début et Fin sont modifiables en ligne** dans la liste. Les autres champs (nom, statut, priorité, projet, échéance, notes) se modifient depuis la vue détail.

### 2.3 Vue détail — mode Création (pleine page)

Ouvert par « Nouveau bon de travail ». En-tête « Nouveau bon de travail » avec les boutons **Annuler** et **Créer** (le bouton de création affiche le décompte des éléments pré-saisis, par exemple « Créer + 3 op. 2 prod. »).

**Champs de l'en-tête** :

| Champ | Notes |
|-------|-------|
| **Nom** | Facultatif — « le nom du projet sera utilisé si vide » (et à défaut, le numéro) |
| **Priorité** | Basse / Normale / Haute / Urgente (défaut : Normale) |
| **Projet** | « Aucun projet » ou un projet de la liste |
| **Date début prévu** | Facultative |
| **Date fin prévu** | Facultative (doit être postérieure au début) |
| **Date d'échéance** | Facultative |
| **Montant total** | **Lecture seule** — indice « -- (calculé depuis les lignes) » |
| **Notes** | Texte libre |

**Carte « Opérations »** : le bouton **Ajouter une tâche** ouvre un sous-formulaire d'opération (mêmes champs qu'en vue détail, voir §2.6). Les opérations saisies ici sont **empilées localement** (en attente) et un tableau récapitulatif affiche Opération / Quantité / Fournisseur / Heures prévues / Statut, avec le total des heures.

**Carte « Produits / Matériaux »** : le bouton **Ajouter un produit** ajoute des lignes éditables (produit d'inventaire avec son stock, quantité, unité en lecture, prix unitaire, montant), avec un « Total matériaux ».

> **Comment fonctionne réellement la création.** Le serveur crée d'abord un **bon de travail vide** (statut Brouillon, numéro `BT-00001`) sans aucune opération. Il crée **ensuite**, une par une, les opérations et les lignes que vous avez pré-saisies, via les points d'accès enfants. Si l'une de ces créations échoue, le bon de travail existe quand même et un message signale le nombre d'éléments non créés. **La création ne déroule aucun modèle de cédule** : les gabarits par type de projet passent par le module Projets (voir §3.16).

### 2.4 Vue détail — mode Lecture

Affichée en cliquant un bon de travail existant.

**En-tête** : numéro (police à chasse fixe), badge de statut, badge de priorité (avec une icône triangle d'alerte si « Urgente »), puis le nom.

**Boutons d'action** :

- **Modifier** (crayon) : passe en mode édition.
- **HTML** et **Aperçu** : génèrent le document HTML du bon de travail et l'affichent dans une **fenêtre intégrée** (voir §2.10).
- **PDF** (imprimante) : ouvre le document dans un **nouvel onglet** du navigateur, prêt à imprimer ou à enregistrer en PDF.

**Boutons de transition de statut** (selon le statut courant) :

| Statut courant | Boutons de transition |
|----------------|-----------------------|
| **Brouillon** | **Démarrer** (→ En cours) |
| **En cours** | **Pause** (→ En pause) · **Terminer** (→ Terminé) |
| **En pause** | **Reprendre** (→ En cours) |
| **Terminé** | (aucune transition) |
| **Annulé** | **Restaurer** (→ Brouillon) · **Supprimer définitivement** |

**Boutons de cycle de vie** (en complément) :

- Si le bon est **annulé** : **Restaurer** (flèche de retour, → Brouillon, reconsomme le stock) et **Supprimer définitivement** (poubelle, action irréversible).
- Sinon, si le bon **n'est pas terminé** : **Annuler** (icône d'interdiction, → Annulé, **soft-delete qui restaure le stock**).

Toutes ces actions demandent une confirmation et sont protégées par un verrou anti double-clic.

**Grille d'informations** : Projet, Début prévu, Fin prévu, Échéance, Montant total, Créé le, plus un bloc Notes.

### 2.5 Vue détail — mode Édition

Titre « Modifier le bon de travail », avec **Annuler** et **Enregistrer**. Champs modifiables : Nom, **Statut** (menu complet, y compris Annulé), Priorité, Projet, Date début prévu, Date fin prévu, Date d'échéance, Notes. Le **Montant total** reste en lecture seule (« calculé depuis les lignes »). L'enregistrement n'envoie que les **champs réellement modifiés** et respecte la machine à états (voir §4.2).

### 2.6 Section Opérations (dans la vue détail)

En-tête « Opérations ({n}) » et bouton **Ajouter une tâche**.

**Formulaire d'ajout d'une opération** :

| Champ | Notes |
|-------|-------|
| **Poste/Opération** | Menu déroulant alimenté par le catalogue actif ; une option « hors catalogue » apparaît si le nom saisi ne s'y trouve pas |
| **Quantité** | Numérique (0 à 1 000 000) |
| **Assigné à** | Employé (menu déroulant) |
| **Fournisseur/Sous-traitant** | « -- Interne -- » (par défaut) ou un fournisseur du Magasin ; une valeur libre héritée est préservée |
| **Heures prévues** | Numérique (0 à 100 000) |
| **Statut** | En attente / En cours / Terminé / Annulé |
| **Date début** | Facultative |
| **Date fin** | Facultative |
| **Phase** | Texte (`poste_travail`), par exemple « Fondation », « Charpente » |
| **Description** | Texte libre |

**Tableau des opérations** (regroupé par **Phase**, avec un bandeau de section) :

Colonnes : **Opération**, **Qté**, **Assigné à**, **Fournisseur**, **Début**, **Fin**, **H. Prévues**, **H. Réelles**, **Statut** (menu déroulant modifiable directement en ligne), et les actions (crayon pour l'édition en ligne, « X » pour supprimer). L'édition en ligne ajoute le champ **Heures réelles**. Le pied affiche les **Totaux** (somme des heures prévues et des heures réelles). Cartes équivalentes sur téléphone.

### 2.7 Section Lignes (matériaux)

En-tête « Lignes ({n}) » et bouton **Ajouter**.

Colonnes : **Description** (avec un badge « Inventaire » si la ligne est liée à un produit), **Qté**, **Unité**, **P.U.** (prix unitaire), **Montant**, actions (crayon / « X »). Pied : **Total**. Si vide : « Aucune ligne ».

> **Effet sur le stock.** Ajouter une ligne liée à un produit **débite le stock** (mouvement de sortie). La modifier ajuste le stock du **delta**. La supprimer **recrédite** le stock (mouvement d'entrée). Ces mouvements sont **ignorés** si le bon de travail est annulé (afin de ne pas comptabiliser deux fois avec la restauration). Voir §4.8.

### 2.8 Section Assignations

En-tête « Assignations ({n}) » et bouton **Assigner**. Chaque ligne montre les initiales de l'employé, son nom, son rôle, la date d'assignation et un bouton « X » pour le retirer. Si vide : « Aucun employé assigné ».

### 2.9 Section Commentaires

En-tête « Commentaires ({n}) » (icône bulle). Fil chronologique : avatar, nom de l'auteur, temps relatif, texte. En bas, une zone de saisie « Ajouter un commentaire... » et le bouton **Envoyer**.

> Un commentaire envoyé **ne se modifie ni ne se supprime**.

### 2.10 Modales

- **Aperçu du bon de travail** : une fenêtre intégrée (cadre `iframe` en bac à sable) affiche le document HTML généré. Titre « Aperçu du bon de travail {numéro} », avec « Ouvrir dans un nouvel onglet » et « Fermer ».
- **Ajouter une ligne** : menu « Produit de l'inventaire (optionnel) » (avec l'option « Saisie libre (sans produit) » et les produits avec leur stock), « Description * », « Quantité », « Unité » (« m, kg, unité... ») et « Prix unitaire ». Boutons Annuler / Ajouter.
- **Assigner un employé** : menu « Employé * » et champ « Rôle » (« Ex : Soudeur, Chef d'équipe... »). Boutons Annuler / Assigner.

### 2.11 Onglet « Opérations » (vue globale)

Cet onglet affiche **toutes les opérations en cours** sur l'ensemble des bons de travail, sous le texte « Vue d'ensemble de toutes les opérations en cours sur les bons de travail. »

**Tableau** (colonnes) : **BT** (numéro + nom), **Opération**, **Qté**, **Assigné à**, **Fournisseur**, **H. Prévues**, **H. Réelles**, **Statut** (badge), **Actions** (modification complète en ligne / suppression). La validation en ligne exige un nom d'opération et des valeurs positives ou nulles pour la quantité et les heures. Le pied affiche « Totaux ({n} opérations) » et la somme des heures. Cartes avec formulaire d'édition sur téléphone.

> **Plafond de la vue globale.** Ce tableau charge jusqu'à **200 opérations** (au-delà, un bandeau signale que la liste est tronquée). Ce plafond ne concerne **que** cette vue globale : à l'intérieur d'un bon de travail, toutes ses opérations sont toujours affichées, sans limite. C'est aussi ce point d'accès qui, historiquement, apparaissait « vide » sur d'anciens tenants — un correctif a ajouté la colonne technique manquante et relevé le plafond ; l'onglet est désormais pleinement fonctionnel.

**Carte « Capacité hebdomadaire par opération »** : sous le tableau, un outil de charge de travail. Une navigation par semaine (‹ › avec la plage du lundi au dimanche) et une légende : **vert** sous 80 %, **jaune** de 80 à 100 %, **rouge** au-delà de 100 % (dépassement du budget d'heures). Le tableau liste : Opération, BT, Heures prévues, Heures réelles, Avancement (%) et une barre de progression colorée (occupation = heures réelles / heures prévues).

> La capacité hebdomadaire ne couvre que les opérations dont la **date de début** tombe dans la semaine sélectionnée.

### 2.12 Onglet « Catalogue »

Gestion des **types d'opérations** réutilisables (table `operation_types`). Ces types alimentent le menu « Poste/Opération » des formulaires d'opération.

**Barre** : bouton **Nouvelle opération** ; recherche (insensible aux accents et à la casse) ; case **Actifs uniquement**.

**Tableau** : **Nom**, **Catégorie**, **Code** (police à chasse fixe), **Heures std** (heures standard), **Actif** (badge Actif / Inactif), **Actions** (Modifier / Supprimer). Pied : « {filtrés} / {total} opérations ».

**Fenêtre de création / modification** : « Nom * », « Catégorie » (« Ex : Menuiserie »), « Code » (« Ex : POSE-FEN »), « Heures standard » (« Ex : 4.0 »), et la case « Actif (visible dans le menu déroulant des BT) ». Renommer ou supprimer un type **avertit** que les bons de travail existants conservent l'ancien nom (instantané historique — il n'y a pas de lien fort entre l'opération et le catalogue).

> **Dix-huit types sont fournis par défaut** (voir §4.5) si le catalogue est vide. **Rappel de permission** : seuls les **administrateurs** et **super-administrateurs** peuvent modifier le catalogue ; un gestionnaire ou un contremaître ne le peut pas.

### 2.13 Onglet « Assistant IA »

Un clavardage « Assistant IA — Bons de travail » sous le sous-titre « Interroge tes bons de travail et crées-en sur confirmation. » Exemples de questions proposées : « Quels bons de travail sont en cours ? », « Crée un bon de travail pour le projet 15, priorité HAUTE, échéance vendredi. », « Quelles opérations sont en attente par statut ? ».

**Comportement** :

- L'assistant **lit** les données en s'appuyant sur une liste blanche stricte de tables (`formulaires`, `formulaire_lignes`, `operations`, `projects`, `companies`). Les tables de **paie, de ressources humaines, d'employés, de sécurité** (NAS, crédits IA, Stripe…) sont **refusées**.
- Pour créer un bon de travail, l'assistant affiche une **carte de proposition** (Nom, Priorité, Projet, Début, Fin, Échéance) avec les boutons **Annuler** et **Confirmer**. **Rien n'est écrit tant que l'on n'a pas confirmé.** À la confirmation, le serveur **revérifie le droit d'écriture** puis crée le bon de travail.
- L'assistant **ne crée que des bons de travail** : ni opérations, ni lignes, ni assignations.

> Chaque échange consomme des **crédits IA** prépayés (voir §4.11). Si le solde est épuisé, l'appel est refusé.

---

## 3. Processus pas à pas

### 3.1 Créer un bon de travail (vide)

1. Onglet **Bons de Travail** → **Nouveau bon de travail**.
2. Renseigner au besoin le **Nom** (facultatif), la **Priorité**, le **Projet**, les **dates** et les **Notes**.
3. **Créer**. Le bon apparaît en statut **Brouillon**, numéroté `BT-00001`.

> Si vous laissez le nom vide en choisissant un projet, le bon prend le **nom du projet**. Sans nom ni projet, il prend son **numéro**. Si le projet est lié à une opportunité disposant d'un dossier, le bon est **rattaché au dossier** automatiquement.

### 3.2 Créer un bon de travail avec opérations et matériaux pré-saisis

1. Ouvrir **Nouveau bon de travail**.
2. Dans la carte **Opérations**, cliquer **Ajouter une tâche** et remplir le sous-formulaire ; répéter pour chaque tâche.
3. Dans la carte **Produits / Matériaux**, cliquer **Ajouter un produit** et remplir les lignes.
4. **Créer + {n} op. {m} prod.**. Le serveur crée le bon, puis chaque opération et chaque ligne.

> Si une opération ou une ligne échoue à la création, le bon existe quand même ; un message indique le nombre d'éléments non créés (à ajouter ensuite depuis la vue détail).

### 3.3 Démarrer, mettre en pause, reprendre, terminer

1. Ouvrir le bon de travail (clic sur la ligne).
2. Selon le statut, cliquer la transition voulue :
   - **Brouillon** → **Démarrer** (passe En cours) ;
   - **En cours** → **Pause** (passe En pause) ou **Terminer** (passe Terminé) ;
   - **En pause** → **Reprendre** (passe En cours).
3. Le badge de statut change de couleur.

> **La machine à états protège les transitions.** On ne peut pas sauter de « Brouillon » à « Terminé » directement, ni passer de « En pause » à « Terminé » sans repasser par « En cours ». Un **bon terminé ou annulé est terminal** : seul un super-administrateur peut l'en sortir.
>
> **Garde de fin de travail.** Le serveur **refuse de passer un bon à « Terminé »** s'il reste des opérations non terminales (ni terminées, ni annulées). Terminez ou annulez d'abord ces opérations. (Un super-administrateur peut passer outre.)

### 3.4 Annuler un bon de travail (soft-delete, stock restauré)

1. Ouvrir un bon **non terminé** → **Annuler** (confirmation).
2. Le statut passe à **Annulé**. **Le stock consommé par les lignes est intégralement restauré** (mouvements d'entrée). L'historique du bon (opérations, lignes, commentaires) est **conservé**.

> C'est la façon normale de « supprimer » un bon actif : il quitte le flux de travail mais reste consultable (filtre « Annulé ») et son stock est rendu.

### 3.5 Restaurer un bon de travail annulé

1. Ouvrir un bon **annulé** → **Restaurer** (confirmation).
2. Le statut revient à **Brouillon** et **le stock est de nouveau consommé** (les lignes redébitent l'inventaire).

> La restauration est l'inverse exact de l'annulation. Le stock reste cohérent dans les deux sens.

### 3.6 Supprimer définitivement un bon de travail

1. Le bon doit **déjà être annulé** (sinon annulez-le d'abord, §3.4).
2. Depuis la liste (bouton poubelle sur une ligne annulée) **ou** depuis la vue détail (**Supprimer définitivement**) → confirmation.
3. Le bon et tous ses enfants (opérations, lignes, assignations, commentaires) sont **effacés physiquement**. Action **irréversible**.

> Le bouton de suppression **n'apparaît que sur les bons annulés**. Un bon actif n'affiche jamais ce bouton dans la liste.

### 3.7 Ajouter une opération (tâche)

1. Vue détail → section **Opérations** → **Ajouter une tâche**.
2. Choisir le **Poste/Opération** (catalogue ou nom libre), la **Quantité**, l'**employé**, le **Fournisseur/Sous-traitant**, les **Heures prévues**, le **Statut**, les **dates**, la **Phase** et une **Description**.
3. **Enregistrer**. L'opération apparaît sous le bandeau de sa phase.

> Le serveur refuse d'ajouter une opération à un bon **terminé ou annulé** (sauf super-administrateur). Il vérifie aussi que l'employé existe et que le statut est valide.

### 3.8 Modifier le statut ou les heures réelles d'une opération

- **Statut** : dans le tableau des opérations, changer le menu déroulant **Statut** de la ligne (En attente / En cours / Terminé / Annulé). Enregistré immédiatement.
- **Heures réelles** : cliquer le crayon de la ligne pour l'édition en ligne, saisir les **Heures réelles**, puis enregistrer.

> **Les heures réelles se saisissent à la main.** Aucun report automatique depuis le pointage : la table `time_entries` n'alimente pas les heures réelles des opérations. Le statut d'une opération est **indépendant** du statut du bon de travail.

### 3.9 Ajouter une ligne de matériau (mouvement de stock)

1. Vue détail → section **Lignes** → **Ajouter**.
2. Choisir un **produit de l'inventaire** (ou « Saisie libre »), la **Description**, la **Quantité**, l'**Unité** et le **Prix unitaire**.
3. **Ajouter**. Le montant de la ligne (quantité × prix) et le total du bon se recalculent.

> Si la ligne est liée à un produit, le stock est **débité** (mouvement de sortie) et tracé dans les mouvements d'inventaire. Le serveur refuse d'ajouter une ligne à un bon **terminé ou annulé** (sauf super-administrateur). Le suivi des niveaux de stock et les alertes de seuil vivent dans le **Module 09 Magasin**.

### 3.10 Modifier ou supprimer une ligne

- **Modifier** (crayon, édition en ligne) : ajuste le montant et le total ; si un produit est lié, le stock est ajusté du **delta** de quantité (sortie supplémentaire si la quantité augmente, retour si elle diminue).
- **Supprimer** (« X ») : retire la ligne, recalcule le total et **recrédite** le stock du produit lié (mouvement d'entrée).

> Ces ajustements de stock sont **suspendus** si le bon est annulé (le stock a déjà été rendu par l'annulation).

### 3.11 Assigner ou désassigner un employé

1. Vue détail → section **Assignations** → **Assigner**.
2. Choisir l'**Employé** et saisir un **Rôle** libre → **Assigner**.
3. Pour retirer un employé : cliquer le « X » de sa ligne.

> Un même employé ne peut être assigné **qu'une seule fois** par bon de travail (une seconde tentative est refusée). Retirer une assignation n'affecte pas les opérations où l'employé est déjà désigné.

### 3.12 Ajouter un commentaire

1. Vue détail → section **Commentaires** → zone de saisie → **Envoyer**.
2. Le commentaire s'ajoute au fil avec l'auteur et l'horodatage.

### 3.13 Générer et imprimer le document HTML / PDF

1. Vue détail → **HTML** ou **Aperçu** (fenêtre intégrée), ou **PDF** (nouvel onglet).
2. Le document reprend l'en-tête de l'entreprise (thème et couleurs configurés), le numéro, le projet, les dates, le statut, la priorité, les lignes de matériaux, les opérations avec leurs heures, et les assignations.
3. Depuis le nouvel onglet (PDF), utiliser l'impression du navigateur (Ctrl+P) pour imprimer ou **Enregistrer en PDF**.

> Le document est **bilingue** (selon la langue configurée du tenant) et **échappé** contre l'injection HTML. Il est généré côté serveur, sans dépendance externe.

### 3.14 Gérer le catalogue de types d'opérations

1. Onglet **Catalogue** → **Nouvelle opération**.
2. Renseigner **Nom**, **Catégorie**, **Code**, **Heures standard**, et la case **Actif**.
3. **Enregistrer**. Le type apparaît dans le menu « Poste/Opération » des formulaires.
4. Pour retirer un type du menu sans l'effacer : le modifier et **décocher « Actif »**.

> Réservé aux **administrateurs / super-administrateurs**. Supprimer ou renommer un type **ne change pas** les opérations déjà créées.

### 3.15 Utiliser l'assistant IA

1. Onglet **Assistant IA**.
2. Poser une question (« Quelles opérations sont en attente par statut ? ») ou demander de créer un bon (« Crée un bon de travail pour le projet 15, priorité HAUTE »).
3. Pour une création, vérifier la **carte de proposition** puis **Confirmer**. Le bon est créé (après revérification du droit d'écriture).

> L'assistant ne crée qu'un **bon de travail** ; ajoutez ensuite ses opérations et ses lignes depuis la vue détail.

### 3.16 Générer une cédule automatique (depuis le module Projets)

La cédule d'opérations par type de projet **ne se déclenche pas depuis cette page** :

1. Aller au **Module Projets**, ouvrir le projet, cliquer **Générer la cédule**.
2. Le serveur crée un bon de travail « Cédule - {projet} » (statut Brouillon) et y déroule les opérations d'un **modèle** choisi selon la catégorie du projet : **résidentiel** (28 opérations), **rénovation** (18 opérations) ou **public** (24 opérations). Les dates sont calées sur le début du projet.
3. L'opération est **idempotente** : un seul bon de cédule automatique par projet (relancer renvoie la cédule existante).

> La cédule se génère aussi **automatiquement à la conversion d'un devis en projet**. Une fois créée, elle se pilote comme n'importe quel bon de travail dans cette page.

### 3.17 Suivre la capacité hebdomadaire

1. Onglet **Opérations** → carte **Capacité hebdomadaire par opération**.
2. Naviguer d'une semaine à l'autre avec ‹ ›.
3. Lire l'occupation : **vert** (marge), **jaune** (proche du budget), **rouge** (dépassement d'heures).

---

## 4. Référence

### 4.1 Points d'accès (API)

Tous préfixés par `/api/erp/v1/production` (sauf l'assistant IA, sous `/api/erp/v1/bons-travail/ai`). « Écriture » = garde `require_tenant_admin_or_role(admin, super_admin, gestionnaire, contremaitre)`.

**Bon de travail (cœur du module)**

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/statistics` | Cartes KPI | lecture |
| GET `/work-orders` | Liste paginée (`per_page` 1 à 100), filtres statut / priorité / recherche | lecture |
| POST `/work-orders` | Créer un bon (vide) | écriture |
| GET `/work-orders/{id}` | En-tête seul (+ nom du projet) | lecture |
| GET `/work-orders/{id}/detail` | Bon + lignes + assignations + commentaires + opérations | lecture |
| PUT `/work-orders/{id}` | Modifier (machine à états, transition de stock à l'annulation) | écriture |
| DELETE `/work-orders/{id}` | Supprimer (dual : soft si actif, hard si annulé) | écriture |
| POST `/work-orders/{id}/restore` | Restaurer un bon annulé (→ Brouillon) | écriture |
| POST `/work-orders/{id}/generate-html` | Document HTML thématisé | lecture |
| GET `/work-orders/{id}/time-entries` | Pointages liés (lecture seule, **non affiché dans la page**) | lecture |

**Lignes de matériaux**

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/work-orders/{id}/lines` | Lister | lecture |
| POST `/work-orders/{id}/lines` | Ajouter (sortie de stock ; refus si terminé/annulé) | écriture |
| PUT `/work-orders/{id}/lines/{lid}` | Modifier (ajustement de stock du delta) | écriture |
| DELETE `/work-orders/{id}/lines/{lid}` | Supprimer (entrée de stock) | écriture |

**Opérations**

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/operations` | **Vue globale** paginée (`per_page` 1 à 200, défaut 50) | lecture |
| GET `/work-orders/{id}/operations` | Toutes les opérations du bon (sans limite) | lecture |
| POST `/work-orders/{id}/operations` | Ajouter (refus si terminé/annulé) | écriture |
| PUT `/work-orders/{id}/operations/{oid}` | Modifier | écriture |
| DELETE `/work-orders/{id}/operations/{oid}` | Supprimer | écriture |

**Assignations et commentaires**

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET / POST `/work-orders/{id}/assignations` | Lister / assigner (unicité par employé) | lecture / écriture |
| DELETE `/work-orders/{id}/assignations/{aid}` | Retirer | écriture |
| GET / POST `/work-orders/{id}/comments` | Lister / ajouter | lecture / écriture |

**Catalogue et cédule**

| Méthode + chemin | Rôle | Droit |
|---|---|---|
| GET `/operation-types` | Lister les types (option « actifs seulement ») | lecture |
| POST / PUT / DELETE `/operation-types[/{id}]` | Gérer le catalogue | **administrateur** (`_require_admin`) |
| POST `/projects/{id}/generate-cedule` | Générer la cédule d'un projet (idempotent) | écriture |

**Assistant IA** (`/api/erp/v1/bons-travail/ai`)

| Méthode + chemin | Rôle | Limite |
|---|---|---|
| POST `/chat` | Interroger + proposer un bon (aucune écriture) | 20 requêtes / min |
| POST `/confirm-action` | Créer le bon après confirmation (revérifie le droit) | 30 requêtes / min |

**Points d'accès partagés avec le module Suivi** (même routeur, mais rattachés au Suivi) : `GET /gantt/bons-travail` (les bons et leurs opérations en sous-tâches), `PUT /kanban/update-status` (change le statut d'un bon depuis le Kanban — **bloque la transition « Annulé »**, à faire depuis cette page pour restaurer le stock), `GET /calendar-events` (échéances des bons dans le Calendrier).

### 4.2 Statuts du bon de travail et cycle de vie

Statuts : **BROUILLON**, **EN_COURS**, **EN_PAUSE**, **TERMINE**, **ANNULE**.

| Statut | Couleur | Transitions autorisées |
|--------|---------|------------------------|
| **BROUILLON** | gris | → EN_COURS, → ANNULE |
| **EN_COURS** | bleu | → EN_PAUSE, → TERMINE, → ANNULE |
| **EN_PAUSE** | ambre | → EN_COURS, → ANNULE |
| **TERMINE** | vert | (terminal) |
| **ANNULE** | rouge | (terminal) |

> **TERMINE et ANNULE sont terminaux** : seul un super-administrateur peut les rouvrir. La transition vers **TERMINE** est en outre bloquée s'il reste des opérations non terminales. Les statuts hérités (accents, espaces, variantes) sont normalisés côté serveur avant toute vérification.

### 4.3 Priorités

**BASSE**, **NORMALE** (défaut), **HAUTE**, **URGENTE**. La priorité « Urgente » ajoute une icône d'alerte dans la vue détail.

### 4.4 Statuts d'opération

**En attente** (défaut), **En cours**, **Terminé**, **Annulé**. Valeurs sensibles à la casse (première lettre en majuscule, avec espaces). Aucun lien automatique avec le statut du bon de travail.

### 4.5 Catalogue d'opérations par défaut (18 types)

Si le catalogue est vide, ces 18 types sont proposés :

```
Démolition · Décontamination · Excavation · Fondation/Coffrage ·
Structure/Charpente · Plomberie · Électricité · CVAC · Isolation ·
Gypse/Plâtre · Peinture · Toiture · Revêtement extérieur ·
Menuiserie/Finition · Plancher · Céramique · Aménagement paysager ·
Nettoyage final
```

Un nom personnalisé (hors catalogue) reste possible dans le formulaire d'opération.

### 4.6 Modèles de cédule (par type de projet)

Appliqués **hors de cette page** (module Projets ou conversion devis → projet) :

| Catégorie | Nombre d'opérations | Étendue indicative |
|-----------|---------------------|--------------------|
| **Résidentiel** | 28 | ≈ jour 0 à jour 181 |
| **Rénovation** | 18 | ≈ jour 0 à jour 98 |
| **Public** | 24 | ≈ jour 0 à jour 245 |

La catégorie est déduite selon la priorité **rénovation > public > résidentiel**. Chaque modèle place les opérations en « En attente », sans employé, avec des dates calées sur le début du projet. Un seul bon de cédule automatique par projet.

### 4.7 Calculs

| Élément | Formule | Déclenché par |
|---------|---------|---------------|
| **Montant d'une ligne** | `quantité × prix unitaire` | ajout / modification d'une ligne |
| **Montant total du bon** | `Σ des montants de lignes` | ajout / modification / suppression d'une ligne |
| **Séquence d'une ligne / d'une opération** | `MAX(séquence) + 1` (verrou sur le bon parent) | à la création |
| **Progression d'une opération** (Gantt) | `heures réelles / heures prévues × 100` | affichage |
| **Occupation hebdomadaire** | `heures réelles / heures prévues` (seuils 80 % / 100 %) | affichage |

> **Aucun calcul de TPS/TVQ ni de marge.** Le montant total est un indicateur opérationnel, pas un document fiscal.

### 4.8 Invariant de stock

« Bon de travail non annulé ⇔ stock consommé ; bon de travail annulé ⇔ stock restauré. » La bascule ne se produit qu'au **franchissement réel** de la frontière « Annulé » :

| Événement | Effet sur le stock |
|-----------|--------------------|
| Ajout d'une ligne liée à un produit | **Sortie** (débit) |
| Modification de la quantité (delta > 0) | **Sortie** du delta |
| Modification de la quantité (delta < 0) | **Entrée** du delta |
| Suppression d'une ligne liée à un produit | **Entrée** (crédit) |
| Annulation du bon (soft-delete) | **Entrée** de toutes les lignes (stock restauré) |
| Restauration du bon (→ Brouillon) | **Sortie** de toutes les lignes (stock reconsommé) |

> Les mutations de lignes sur un bon **déjà annulé** n'ajustent **pas** le stock (il a déjà été rendu). Le Kanban **interdit** l'annulation d'un bon (il faut passer par cette page pour que le stock soit restauré). Chaque mouvement est tracé dans `mouvements_stock`.

### 4.9 Validations et limites

| Règle | Effet |
|-------|-------|
| Nom vide et projet choisi | Nom = nom du projet |
| Nom vide et aucun projet | Nom = numéro du bon |
| Nom au-delà de 255 caractères / notes au-delà de 5 000 | Refusé |
| Date de fin antérieure à la date de début | Refusé |
| Transition de statut non autorisée | Refusée (sauf super-administrateur) |
| Passer à « Terminé » avec des opérations non terminales | Refusé (sauf super-administrateur) |
| Ajouter une ligne / une opération à un bon terminé ou annulé | Refusé (sauf super-administrateur) |
| Quantité de ligne hors bornes (0 à 1 000 000) / prix (0 à 10 000 000) | Refusé |
| Produit quantité × prix supérieur à 10^12 | Refusé (protection contre le dépassement numérique) |
| Heures d'opération hors bornes (0 à 100 000) | Refusé |
| Employé inexistant (opération ou assignation) | Refusé |
| Employé déjà assigné au bon | Refusé (unicité) |
| Supprimer un type de catalogue par un non-administrateur | Refusé |
| Supprimer définitivement un bon non annulé | Impossible (l'annuler d'abord) |

### 4.10 Numérotation

Format `BT-00001` (5 chiffres, zéros de tête). Le numéro est attribué de façon **sûre face à la concurrence** : le bon est inséré avec un numéro temporaire, puis renuméroté d'après son identifiant. Jamais de `MAX + 1`.

### 4.11 Effet sur l'argent et crédits IA

Le module **ne facture rien** via Stripe ou QuickBooks. Le champ « Montant total » n'émet aucune facture. Le **seul effet monétaire** est le **débit de crédits IA prépayés** lors de l'usage de l'assistant IA : le coût réel des jetons (0,003 $/millier en entrée, 0,015 $/millier en sortie) est **majoré de 30 %**. Si les crédits sont épuisés, l'assistant est refusé. Le **seul effet matériel** est le mouvement de stock d'inventaire déclenché par les lignes de matériaux (voir §4.8).

### 4.12 Tables PostgreSQL (par tenant)

| Table | Rôle |
|-------|------|
| `formulaires` | En-tête du bon (`type_formulaire = 'BON_TRAVAIL'`) : numéro, nom, statut, priorité, projet, dates, montant, notes |
| `formulaire_lignes` | Lignes de matériaux (description, quantité, unité, prix, montant, produit lié) |
| `operations` | Opérations / tâches (nom, phase, employé, fournisseur, heures prévues et réelles, statut, dates) |
| `bt_assignations` | Employés assignés au bon (rôle, date) — sans clé étrangère forte |
| `bt_comments` | Commentaires (auteur, texte, date) — sans clé étrangère forte |
| `operation_types` | Catalogue de types d'opérations (nom, catégorie, code, heures standard, actif) |
| `produits` / `mouvements_stock` | Inventaire touché par les lignes de matériaux |
| `time_entries` | Pointages liés au bon (lecture seule) |
| `dossier_formulaires` | Rattachement du bon au dossier d'opportunité |
| `public.operation_templates` | Modèles de cédule partagés entre tenants |

> La table `formulaires` (partagée avec les devis, bons de commande et factures selon leur `type_formulaire`) n'est **pas** définie par le code source : elle est provisionnée par copie depuis un tenant de référence, et le code garantit défensivement ses colonnes au moment de l'exécution.

---

## 5. Intégrations et FAQ

### 5.1 Liens avec les autres modules

| Module | Lien |
|--------|------|
| **Projets** (Module 08) | Un bon peut être rattaché à un projet ; le bouton « Générer la cédule » (page Projets) crée un bon de cédule ; la conversion d'un devis en projet génère aussi la cédule automatiquement. |
| **Magasin / Inventaire** (Module 09) | Les lignes de matériaux liées à un produit débitent le stock (sortie) et le recréditent à la suppression ou à l'annulation (entrée). Les mouvements et les alertes de seuil se consultent au Magasin. |
| **Suivi (Kanban / Gantt / Calendrier)** (Module 02) | Les mêmes bons de travail s'affichent dans ces vues. Le Kanban change leur statut (mais bloque l'annulation, à faire ici) ; le Gantt affiche les opérations en sous-tâches ; le Calendrier montre les échéances. |
| **Pointage** (Module 12) | Les employés pointent leurs heures ; ces pointages sont **liés** au bon mais **ne remontent pas** dans les heures réelles des opérations (saisie manuelle). |
| **Employés** (Module 10) | Les assignations et l'employé d'une opération sont choisis parmi les employés du tenant. |
| **Dossiers / Fiche 360** (Module 06) | Un bon rattaché à un projet lié à une opportunité est **auto-rattaché** au dossier correspondant. |
| **Configuration** (Module 30) | Le logo, les couleurs du document et la langue proviennent de la configuration de l'entreprise. |
| **Assistant IA** (Module 24) | L'assistant du module consomme les crédits IA prépayés du tenant. |

### 5.2 FAQ

**Comment créer un bon avec une cédule d'opérations toute faite ?**
Pas depuis cette page. Le formulaire « Nouveau bon de travail » crée un bon **vide**. Utilisez le bouton « Générer la cédule » du module Projets (ou convertissez un devis en projet). La cédule choisit un modèle selon le type de projet (résidentiel 28, rénovation 18, public 24 opérations).

**Y a-t-il vraiment quatre onglets ?**
Oui : **Bons de Travail**, **Opérations**, **Catalogue**, **Assistant IA**. La vue détail d'un bon n'est pas un onglet (elle s'ouvre par-dessus). Le Kanban, le Gantt et le Calendrier, eux, sont dans le module Suivi.

**L'annulation d'un bon restaure-t-elle le stock ?**
Oui. Annuler un bon (soft-delete) recrédite **automatiquement** tout le stock consommé par ses lignes. Le restaurer le reconsomme. C'est un changement important par rapport aux anciennes versions, où l'annulation ne touchait pas le stock.

**Comment supprimer définitivement un bon de travail ?**
Il faut d'abord l'**annuler** (soft-delete). Une fois annulé, le bouton « Supprimer définitivement » apparaît (dans la liste et dans la vue détail) et efface physiquement le bon et ses enfants. Un bon actif ne peut pas être supprimé d'un coup.

**Les heures réelles des opérations se remplissent-elles depuis le pointage ?**
Non. Elles se saisissent **à la main** (édition de l'opération). Aucun report automatique depuis la table des pointages.

**Pourquoi ne puis-je pas passer mon bon à « Terminé » ?**
Deux raisons possibles : la transition n'est pas permise depuis le statut courant (par exemple depuis « Brouillon » ou « En pause » directement), ou il reste des **opérations non terminales**. Terminez d'abord (ou annulez) ces opérations, en passant si besoin par « En cours ».

**Qui peut modifier les bons de travail ?**
Les administrateurs, super-administrateurs, gestionnaires et contremaîtres. Le **catalogue** de types d'opérations est plus restreint : administrateurs et super-administrateurs seulement.

**Le montant total est-il modifiable ?**
Non. Il se calcule toujours à partir des lignes de matériaux. Il n'existe aucun champ montant saisissable, et le bon de travail n'émet ni facture ni taxes.

**L'assistant IA peut-il tout faire ?**
Non. Il **lit** les bons de travail (jamais la paie, les employés ni les données de sécurité) et peut **proposer** un bon, créé seulement après confirmation. Il ne crée ni opération, ni ligne, ni assignation.

**Pourquoi l'onglet « Opérations » semblait-il vide auparavant ?**
C'était un défaut corrigé : une colonne technique manquait sur d'anciens tenants et le plafond d'affichage était bas. La vue globale charge désormais jusqu'à 200 opérations (avec un avertissement au-delà) ; à l'intérieur d'un bon, toutes ses opérations sont toujours affichées.

**Peut-on éditer plusieurs champs directement dans la liste ?**
Seulement les dates **Début** et **Fin** (édition en ligne). Les autres champs se modifient depuis la vue détail.

**Le stock peut-il tomber sous zéro ?**
Le suivi des niveaux et les alertes de seuil se gèrent au module Magasin. Le bon de travail se contente d'y inscrire les mouvements ; consultez le Magasin pour l'état réel de l'inventaire.

### 5.3 Ce qui n'existe pas (limites connues)

- Pas de génération de cédule depuis le formulaire de création (elle passe par le module Projets).
- Pas de facture, de taxes ni d'export comptable ; le montant total est informatif.
- Pas de report automatique du pointage dans les heures réelles des opérations.
- Pas de section « Pointage » affichée dans la page (le point d'accès existe, mais n'est pas rendu).
- Pas de champ montant saisissable.
- Pas de suppression définitive directe d'un bon actif (l'annuler d'abord).
- Vue globale des opérations plafonnée à 200 lignes ; la capacité hebdomadaire ne couvre que les opérations dont le début tombe dans la semaine choisie.
- Assistant IA limité à la création d'un bon de travail, sur confirmation.
- Le Kanban, le Gantt et le Calendrier appartiennent au module Suivi, pas à cette page.

---

## 6. Récapitulatif

- Le module **Bons de travail** (`/bons-travail`, section Opérations) réunit **4 onglets** : **Bons de Travail**, **Opérations**, **Catalogue**, **Assistant IA**, plus une **vue détail** contextuelle.
- **4 cartes KPI** en permanence : Total, En cours, Terminés, Montant total.
- **Cycle de vie encadré** : Brouillon → En cours → (En pause) → Terminé, ou Annulé. Machine à états stricte (TERMINE et ANNULE terminaux, sauf super-administrateur), avec garde « pas de fin tant qu'il reste des opérations non terminales ».
- **Création = bon vide** : aucun modèle d'opérations n'est déroulé ici ; les cédules (résidentiel 28 / rénovation 18 / public 24) passent par le module Projets ou la conversion d'un devis.
- **Suppression duale** : un bon actif s'**annule** (soft-delete, **stock restauré**) ; un bon annulé se **supprime définitivement** (hard-delete). Restaurer un bon annulé le reconsomme.
- **Invariant de stock** : les lignes de matériaux liées à un produit débitent le stock (sortie) et le recréditent à la suppression ou à l'annulation (entrée) ; les mutations sur un bon annulé n'ajustent pas le stock.
- **Opérations** regroupées par **phase**, avec heures prévues et réelles (saisie manuelle, pas de report du pointage), statut modifiable en ligne, employé et sous-traitant. Vue globale (max 200) + **capacité hebdomadaire** (vert / jaune / rouge).
- **Catalogue** de types d'opérations (18 par défaut), réservé aux administrateurs ; renommer un type ne change pas les opérations existantes.
- **Assistant IA** : lit les bons de travail (jamais la paie ni les employés), **propose** un bon créé sur confirmation (revérification du droit), et rien d'autre.
- **Document HTML/PDF** thématisé et bilingue, généré côté serveur.
- **Permissions** : écriture des bons pour admin / super-admin / gestionnaire / contremaître ; écriture du catalogue pour admin / super-admin seulement ; lecture ouverte à tout utilisateur du tenant.
- **Effet argent** limité aux **crédits IA** (assistant) ; le **montant total n'émet aucune facture** et il n'y a **pas de taxes**.
- **N'appartiennent pas à cette page** : le Kanban, le Gantt et le Calendrier (module Suivi), et la saisie du pointage (module Pointage).

---

**Documentation générée à partir du code source** : `BonsTravailPage.tsx`, `components/bt/BtAssistantTab.tsx`, `api/production.ts`, `store/useProductionStore.ts` ; `backend/routers/production.py`, `bt_ai.py`, `operation_templates.py`.

**Manuels liés** :
- Module 02 — Suivi & Gantt (Kanban, Gantt, Calendrier des bons de travail) — `02-suivi-gantt.md`
- Module 08 — Projets (génération de la cédule) — `08-ventes-projets.md`
- Module 09 — Magasin (mouvements de stock des matériaux) — `09-operations-magasin.md`
- Module 10 — Employés (assignations, employé d'opération) — `10-operations-employes.md`
- Module 12 — Pointage (heures des employés) — `12-operations-pointage.md`
- Module 24 — Assistant IA (crédits IA) — `24-communication-assistant-ia.md`
- Module 30 — Configuration (thème du document, langue) — `30-configuration.md`
