# Module 13 — Pointage et heures

> **Version** : 3.0 (refonte complète vérifiée par rapport au code source réel)
> **Code de référence** :
> - Frontend : `frontend/src/pages/PointagePage.tsx` (≈ 1 907 lignes, page unique à **7 onglets**), `frontend/src/components/pointage/PointageAssistantTab.tsx` (onglet Assistant IA), API `frontend/src/api/employees.ts` (feuilles de temps), `frontend/src/api/payroll.ts` (paie CCQ), `frontend/src/api/feuillets.ts` (T4 / RL-1 / PD7A), `frontend/src/api/pointageAi.ts` (assistant), `frontend/src/api/projects.ts` et `frontend/src/api/production.ts` (menus déroulants Projet / Bon de travail / Opération)
> - Backend : `backend/routers/employees.py` (le cœur du pointage — points d'accès `/time-entries*` et `/payroll-summary`, ≈ lignes 579 à 1615 ; **et non** `production.py`), `backend/routers/pointage_ai.py` (≈ 300 lignes, assistant en lecture seule), `backend/routers/payroll.py` (paie CCQ, module Paie), `backend/routers/feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (feuillets fiscaux), `backend/routers/payroll_rates.py` (taux de cotisation)
> - Préfixe API : `/api/erp/v1` — les feuilles de temps répondent sous `/employees` (anglais), l'assistant sous `/pointage/ai`, la paie sous `/payroll`, les feuillets sous `/feuillets`
> **Tables PostgreSQL (par tenant)** : `time_entries` (table maîtresse des pointages), `employees` (source du taux horaire figé), `projects` (projet rattaché), `formulaires` (le bon de travail) et `companies` (le client, via le bon de travail), `operations` (l'opération rattachée), `payroll_periods` (verrou de période fermée), `payroll_entries` (fiches de paie CCQ générées)
> **Cadrage** : ce module est le **poste de saisie et de gestion des heures travaillées** sur ordinateur de bureau. Il sert à **saisir** une feuille de temps au nom d'un employé (heure d'entrée, heure de sortie, projet, bon de travail, opération), à la **valider**, à la **modifier**, à la **supprimer**, à consulter les heures **par semaine** et **par projet**, à produire un **résumé de paie rapide** (approximatif), à déclencher la **paie CCQ complète** (périodes, fiches détaillées, bulletins PDF), à générer les **feuillets fiscaux** de fin d'année (T4, RL-1, PD7A) et à interroger un **assistant IA** sur le volume d'heures. Ce **n'est pas** un pointeur en temps réel : le **pointage d'arrivée / départ géolocalisé** de l'employé sur le terrain se fait exclusivement dans l'**application mobile** (`/mobile`), pas ici. La saisie de bureau est **manuelle**. Le module **ne calcule pas** de facture (le lien avec la facturation est passif — voir §5.4) et **n'accède à aucune géolocalisation GPS** (le suivi GPS concerne la flotte de véhicules, un tout autre module — voir §5.6).

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Procédures pas à pas](#3-procédures-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module Pointage (page `/pointage`, titre affiché **« Pointage & Paie »**) est le **registre des heures** de l'entreprise et le **point de départ de la paie**. Il permet à une personne autorisée (administrateur, contremaître, responsable de paie) de :

- **saisir** un pointage au nom d'un employé : heure d'entrée, heure de sortie, projet, bon de travail, opération, notes, et l'indicateur « facturable » ;
- **valider** (approuver) chaque pointage avant qu'il n'entre en facturation ou en paie ;
- **modifier** ou **supprimer** un pointage tant qu'il n'est pas facturé ni figé par une période de paie fermée ;
- consulter les heures **par semaine** (feuille de temps du dimanche au samedi, alignée sur le calendrier CCQ) avec un tableau de **capacité par employé** ;
- consulter les heures **par projet** (agrégat avec forage par employé) ;
- produire un **résumé de paie rapide** sur 7 à 90 jours (chiffres **approximatifs**) ;
- déclencher la **paie CCQ complète** : créer une période, calculer les fiches (heures régulières et supplémentaires, retenues, charges de l'employeur, CCQ), consulter chaque bulletin et le **télécharger en PDF**, puis fermer la période (action irréversible) ;
- générer les **feuillets fiscaux** de fin d'année : **T4** (fédéral, ARC), **RL-1** (Québec, Revenu Québec) et le relevé de **remise des retenues à la source PD7A** ;
- **exporter** les pointages en fichier CSV ;
- interroger un **assistant IA** qui répond uniquement sur des **volumes d'heures agrégés** (jamais sur des montants de paie).

> **Le pointage est la source de vérité des heures.** Le coût de la main-d'œuvre, la facturation au client et la paie CCQ s'appuient tous sur la table `time_entries`. Une saisie erronée se propage en aval. C'est pourquoi la **saisie, la validation, la modification et la suppression** sont réservées aux **approbateurs** (voir §1.4), et pourquoi un pointage **facturé** ou pris dans une **période de paie fermée** devient **verrouillé**.

### 1.2 Ce que le module ne fait PAS

- **Aucun pointeur en temps réel sur le web.** Il n'existe **pas** de bouton « Pointer l'arrivée » / « Pointer le départ » ni de minuterie de pause dans la page `/pointage`. La saisie de bureau est **manuelle** : on tape l'heure d'entrée et l'heure de sortie dans deux champs date-heure. Le vrai pointeur (« punch » en direct, avec géolocalisation) est l'**application mobile** (`/mobile`).
- **Aucune saisie en libre-service par l'employé sur le web.** Sur ordinateur de bureau, seul un approbateur peut créer un pointage, en choisissant l'employé dans une liste. Pour que l'employé pointe lui-même, il utilise l'application mobile avec son **NIP à 4 chiffres**.
- **Aucune géolocalisation GPS dans ce module.** La page `/pointage` ne lit **aucune** position GPS. Le suivi GPS de l'entreprise concerne la **flotte de véhicules et la logistique** (zones géographiques, trajets) — c'est un module distinct, sans lien avec les feuilles de temps. La géolocalisation d'un pointage n'existe **que** dans l'application mobile.
- **Le « Résumé paie » n'est pas la vraie paie.** C'est une **estimation** à taux plats, sans plafonds, sans exemptions, sans impôt sur le revenu et sans charges de l'employeur. Les chiffres publiables se produisent dans l'onglet **Paie CCQ** (voir §2.6).
- **Aucune déduction automatique de pause repas.** Le système n'enlève pas « 30 minutes après 6 heures ». Pour une pause non payée, saisir deux pointages distincts ou ajuster la durée.
- **Aucune détection de retard ou d'absence.** Le module enregistre ce qui est saisi, sans le comparer à un horaire prévu.
- **Aucune validation groupée.** On approuve un pointage à la fois ; il n'y a pas de bouton « Tout valider ».
- **Aucun téléversement de fichier.** On ne joint aucune pièce (photo, note signée) à un pointage dans ce module.
- **Aucune impression de la liste de pointages.** L'export de la liste se fait en **CSV**. Les seuls PDF produits sont les **bulletins de paie CCQ** et les **feuillets T4 / RL-1 / PD7A**.
- **Aucune réouverture d'une période de paie fermée.** La fermeture est **irréversible** ; pour corriger, passer par une période d'ajustement (module Paie).
- **Aucun report automatique** des heures pointées vers les heures réelles des opérations d'un bon de travail (cette saisie reste manuelle au Module 12).

### 1.3 Accès

- **Menu latéral** → section **Opérations** → **Pointage** (icône horloge). L'entrée voisine du Magasin, des Employés, des Bons de travail et de la Comptabilité.
- **Adresse** : `/pointage`.
- Page protégée : il faut être authentifié dans un tenant.
- **Onglet par défaut** : « Pointages ».
- La page comporte **7 onglets** (voir §1.5).

### 1.4 Permissions et rôles

La **consultation** des heures est ouverte à tout utilisateur authentifié du tenant. Toute **écriture** (et le résumé de paie) est réservée aux **approbateurs**. Un « approbateur » est, au sens du serveur (`_require_timecard_approver`, `employees.py:310`) :

- un **administrateur** du tenant (`is_admin` — le vrai signal, relu au serveur ; beaucoup de propriétaires se connectent avec le rôle « utilisateur » mais sont administrateurs), **ou**
- un utilisateur de **rôle** `admin` ou `super_admin`, **ou**
- un **super-administrateur** de la plateforme (`user_type = super_admin`).

Un **simple employé** est exclu (erreur 403).

| Palier | Qui | Ce qu'il peut faire |
|--------|-----|---------------------|
| **Consultation** | Tout utilisateur authentifié du tenant | Voir la liste des pointages, la vue par semaine, la vue par projet, l'export CSV et l'assistant IA. **Ne peut pas** créer, valider, modifier ni supprimer un pointage. |
| **Approbateur** | Administrateur (`is_admin`), rôle `admin` / `super_admin`, ou super-administrateur | Tout ce qui précède, **plus** : **créer**, **valider**, **modifier** et **supprimer** un pointage ; ouvrir le **Résumé paie** ; piloter la **Paie CCQ** ; générer les **feuillets fiscaux**. |

> **L'interface reflète le rôle, mais le serveur reste l'autorité.** Pour un utilisateur qui n'est pas approbateur, l'interface **masque** le bouton « Nouvelle saisie », les boutons « Valider », les icônes de modification et de suppression, ainsi que l'onglet **« Feuillets fiscaux »**. Même en contournant l'interface, le serveur refuse (403) toute écriture par un non-approbateur.
>
> **Petite incohérence connue.** Les onglets **« Résumé paie »** et **« Paie CCQ »** restent **visibles** pour un utilisateur non approbateur (seul l'onglet Feuillets est caché). Mais le point d'accès du résumé de paie renvoie 403 aux non-approbateurs : un employé standard qui ouvre l'onglet « Résumé paie » verra donc un message **« Erreur chargement paie »**. Ce n'est pas un bogue de calcul, c'est la protection qui empêche un employé de lire la masse salariale de ses collègues.
>
> **Mode consultation (abonnement en lecture seule).** Si le tenant est en **mode consultation** (abonnement suspendu), **toutes** les écritures sont bloquées (403), y compris la création / validation / modification / suppression de pointages, la génération de paie et l'envoi d'un message à l'assistant IA. La lecture reste possible.

### 1.5 Les 7 onglets

La page est une barre d'onglets. Voici l'ordre exact, leur rôle et le module backend qu'ils sollicitent :

| # | Onglet | Icône | Rôle | Backend | Visible pour |
|---|--------|-------|------|---------|--------------|
| 1 | **Pointages** | horloge | Liste, création, validation, modification, suppression, recherche, filtres | `employees.py` | tous (actions réservées aux approbateurs) |
| 2 | **Vue semaine** | calendrier | Feuille de temps du dimanche au samedi + capacité par employé | `employees.py` | tous |
| 3 | **Par Projet** | mallette | Heures agrégées par projet + forage par employé | `employees.py` | tous |
| 4 | **Résumé paie** | dollar | Masse salariale **approximative** sur 7 à 90 jours | `employees.py` | tous (données réservées aux approbateurs) |
| 5 | **Paie CCQ** | calculatrice | Paie **complète** : périodes, fiches détaillées, bulletins PDF | `payroll.py` | tous (actions réservées aux approbateurs) |
| 6 | **Feuillets fiscaux** | feuille de calcul | T4, RL-1 et PD7A de fin d'année | `feuillets_t4/rl1/pd7a.py` | **approbateurs seulement** |
| 7 | **Assistant IA** | étoiles | Clavardage en lecture seule sur le volume d'heures | `pointage_ai.py` | tous |

> **Un même écran, quatre modules backend.** Cette page traverse quatre routeurs distincts : `employees.py` (le cœur : feuilles de temps + résumé de paie), `payroll.py` (paie CCQ), les trois routeurs de feuillets, et `pointage_ai.py` (assistant). C'est la raison pour laquelle certains sujets renvoient au **Module 15 Comptabilité / Paie** (paie CCQ, feuillets) et au **Module 11 Employés** (taux horaire, NAS).

### 1.6 Concepts clés

- **Pointage (feuille de temps)** : un enregistrement de la table `time_entries` = un employé, une plage horaire (entrée / sortie), et éventuellement un projet, un bon de travail et une opération.
- **Heures calculées** : la durée `(sortie − entrée)` en heures, arrondie à deux décimales. Elle se calcule automatiquement dès que les deux horodatages sont saisis (à l'écran **et** au serveur).
- **Taux figé au pointage (immuable)** : au moment de la création, le système **capture** le taux horaire de l'employé (ou, à défaut, son salaire annuel ÷ 2080) et le **fige** dans le pointage. Un changement futur de taux ne recalcule **jamais** rétroactivement un pointage déjà saisi ou facturé. C'est cette valeur figée qui sert au coût.
- **Facturable** : indique si l'heure peut être refacturée au client. Coché par défaut.
- **Validé (approuvé)** : un pointage marqué « validé » par un approbateur, avec la trace de qui et quand. Prérequis à la facturation et à la paie.
- **Facturé (verrou)** : lorsqu'un pointage a été inclus dans une facture (posé par le module Comptabilité / Dossiers), il devient **immuable** : impossible de le modifier, de le supprimer ou de le dévalider.
- **Période de paie fermée (verrou)** : une période de paie CCQ fermée **fige** tous les pointages dont la date tombe dedans. On ne peut plus y insérer, y déplacer, y valider ni y supprimer un pointage — ce qui protège les cumuls de fin d'année (T4 / RL-1 / PD7A).
- **Semaine CCQ (dimanche → samedi)** : la vue par semaine est **alignée sur le calendrier de paie de la construction** : elle commence le **dimanche** et se termine le **samedi** (et non du lundi au dimanche).
- **Capacité hebdomadaire** : le nombre d'heures visées par semaine pour un employé (par défaut **40 h**, réglé sur sa fiche au Module 11). Sert à colorer l'occupation (sous-charge, normal, dépassement).

### 1.7 Coordination avec l'application mobile

Constructo AI dispose d'une **application mobile** (`/mobile`, PWA) pour le pointage terrain. Les deux outils :

- **partagent la même table `time_entries`** : un pointage saisi sur le mobile apparaît dans l'onglet Pointages de bureau, et inversement ;
- se distinguent par leur usage : le **mobile** est le **vrai pointeur** (l'employé pointe lui-même son arrivée et son départ, avec son NIP et la géolocalisation), tandis que le **bureau** sert à la **saisie administrative** et à la **gestion** (validation, correction, paie).

> La page de bureau **n'affiche pas** de colonne « source » : rien n'indique visuellement qu'un pointage vient du mobile plutôt que d'une saisie de bureau. Pour la documentation de l'application mobile, consulter le manuel séparé MOBILE_REACT.

---

## 2. Interface

### 2.1 Disposition générale

```
+---------------------------------------------------------------------+
|  Pointage & Paie                                       [ Exporter ]  |  <- barre de titre
+---------------------------------------------------------------------+
| [Pointages][Vue semaine][Par Projet][Résumé paie][Paie CCQ]         |  <- 7 onglets
| [Feuillets fiscaux][Assistant IA]                                   |
+---------------------------------------------------------------------+
|                                                                     |
|   (contenu de l'onglet actif)                                       |
|                                                                     |
+---------------------------------------------------------------------+
```

- **Barre de titre** : le titre « Pointage & Paie » et, à droite, un bouton **« Exporter »** (icône de téléchargement) qui produit le fichier CSV de tous les pointages du tenant.
- **Alertes** : une bande d'erreur (rouge) ou de succès (verte) s'affiche en haut au besoin. Les messages de **succès** disparaissent après 3 secondes ; les messages d'**erreur** restent affichés jusqu'à l'action suivante.

### 2.2 Onglet « Pointages » (par défaut)

C'est la vue principale : la liste de toutes les feuilles de temps, avec la création, la validation, la modification et la suppression.

**Barre de commandes**

| Élément | Rôle |
|---------|------|
| **Nouvelle saisie** (bouton bleu, icône +) | Ouvre la modale de création. **Approbateurs seulement.** |
| **Recherche** (champ texte) | Recherche **au serveur** (délai de 350 ms) sur l'employé, le projet, le bon de travail, le client, l'opération et les notes — sur **tout** le jeu de données, pas seulement la page affichée. |
| **Filtre de statut** (menu déroulant) | « Tous », « Valides », « Non valides », « Facturés ». |

**Tableau** (colonnes triables et redimensionnables)

| Colonne | Contenu |
|---------|---------|
| **Employé** | Prénom et nom. |
| **Client** | Nom du client, rattaché via le bon de travail. |
| **Projet** | Nom du projet, ou vide. |
| **BT** | Numéro du bon de travail, ou vide. |
| **Opération** | Nom (ou description) de l'opération, ou vide. |
| **Début** | Heure d'entrée. |
| **Fin** | Heure de sortie. |
| **Heures** | Durée calculée. |
| **Statut** | Voir ci-dessous. |
| **Actions** | Icônes de modification et de suppression (approbateurs). |

**Colonne « Statut »**

- Badge bleu **« Facturé »** si le pointage est facturé (lecture seule).
- Sinon, badge vert **« Validé »** (avec une coche) s'il est validé.
- Sinon, un bouton **« Valider »** (pour un approbateur) ou un badge gris **« À valider »** (pour un non-approbateur).

**Colonne « Actions »** (approbateurs)

- **Crayon** = modifier ; **corbeille** = supprimer.
- Les deux icônes sont **désactivées** si le pointage est facturé (infobulle « Pointage facturé — verrouillé »).
- La suppression demande une confirmation (« Supprimer ce pointage ? »).

**État vide** : « Commencez par enregistrer une saisie de temps. » avec un bouton « Nouvelle saisie ».

**Pagination** : 20 pointages par page.

**Modale « Nouveau pointage »**

| Champ | Détail |
|-------|--------|
| **Employé \*** | Menu déroulant obligatoire. |
| **Projet** | Menu déroulant (option « -- Aucun projet -- »). |
| **Bon de travail** | Menu déroulant (option « -- Aucun BT -- »), format « numéro — projet ». |
| **Entrée (Punch In)** | Champ date-heure. |
| **Sortie (Punch Out)** | Champ date-heure. |
| *Heures calculées* | Encart automatique « Heures calculées : {N} h » dès que la durée est positive. |
| **Notes** | Zone de texte. |
| **Facturable** | Case à cocher (cochée par défaut). |

Boutons **Annuler** / **Créer** (« Créer » désactivé tant qu'aucun employé n'est choisi). Contrôles de saisie : employé obligatoire ; il faut fournir les **deux** horodatages ou **aucun** ; la sortie doit suivre l'entrée ; la durée ne peut pas dépasser **24 h**.

> **La modale de création ne permet pas de choisir l'opération.** Pour rattacher un pointage à une opération précise, le créer d'abord avec son bon de travail, puis passer par **Modifier** (voir §3.2).

**Modale « Modifier le pointage »** (plus riche que la création)

- Grille : **Employé**, **Projet**, **Bon de travail**, **Opération** (menu déroulant **dépendant** du bon de travail : il se remplit une fois le BT choisi ; libellé « Opération (chargement...) » pendant le chargement, « -- Sélectionnez un BT -- » s'il n'y a pas de BT).
- **Entrée** / **Sortie** : champs date-heure à la **seconde** près.
- Encart **« Heures calculées »** + champ **« Type de travail »** (texte libre, ex. « Installation », « Réparation »).
- **Notes** (zone de texte).
- Cases **« Facturable »** et **« Validé »** (on peut donc valider ou dévalider directement ici).
- Si le pointage est facturé : mention **« Déjà facturé — modifications refusées côté serveur »** (icône de cadenas).
- L'enregistrement n'envoie **que les champs modifiés** ; vider Projet / BT / Opération les détache.

### 2.3 Onglet « Vue semaine »

Feuille de temps hebdomadaire, **du dimanche au samedi** (alignée CCQ).

- **Navigation** : boutons **« Semaine précédente »** et **« Semaine suivante »** (± 7 jours), la plage « {début} au {fin} », et un badge **« Total : {N} h »**.
- **Tableau des jours** (7 lignes) : **Jour**, **Date**, **Nb Entrées**, **Total Heures** ; ligne de pied **« Total semaine »**. Les jours à 0 h sont grisés.
- **Carte « Capacité hebdomadaire par employé »** : légende « Vert < 80 % · Jaune 80-100 % · Rouge > 100 % (surchargé) ». Colonnes : **Employé**, **Heures travaillées**, **Capacité**, **Occupation** (pourcentage coloré) et **Charge** (barre de progression). La capacité par défaut est **40 h** (réglable par employé au Module 11) ; un employé à 100 % ou plus est signalé « surchargé ».

### 2.4 Onglet « Par Projet »

Tableau agrégé : **Projet**, **Heures** (somme), **Nb Employés** (employés distincts). Les lignes sont **cliquables** (accessibles au clavier) : un clic **déplie** le détail par employé (nom + heures de cet employé sur ce projet). L'agrégat couvre **tous** les pointages du tenant (pas de filtre de date) ; il présente les 20 premiers projets par heures décroissantes.

> Cette vue **exclut** les pointages sans projet (seuls les pointages rattachés à un projet y figurent).

### 2.5 Onglet « Résumé paie » (approximatif)

Vue de masse salariale **rapide et estimée**.

- **Sélecteur de période** : **7 jours**, **14 jours**, **30 jours** (par défaut) ou **90 jours**.
- **Deux cartes** : **« Masse salariale brute »** ($) et **« Employés »** (nombre).
- **Tableau** : **Employé**, **Dept.**, **Heures**, **Taux** ($/h), **Brut**, **Déductions** (en rouge), **Net** (en vert). Version en cartes sur téléphone.

> **Chiffres approximatifs — à ne pas publier.** Le calcul est volontairement simplifié (`employees.py:897` : « flat employee rates, no caps/exemption »). Les seules retenues appliquées sont **RRQ 6,30 %**, **RQAP 0,43 %** et **AE 1,30 %** sur le brut (≈ 8,03 % au total). **Aucun plafond**, **aucune exemption** (RRQ 3 500 $), **aucun impôt** fédéral ni provincial, **aucune charge de l'employeur** (CNESST, FSS, CCQ). Seuls les employés **actifs** sont comptés. Pour la vraie paie, utiliser l'onglet **Paie CCQ**.
>
> **Réservé aux approbateurs.** Un utilisateur non approbateur voit cet onglet mais reçoit une erreur au chargement (voir §1.4).

### 2.6 Onglet « Paie CCQ » (paie complète)

La paie réelle, avec retenues détaillées, charges de l'employeur et CCQ. Le détail des taux et des paliers d'impôt est documenté au **Module 15 Comptabilité / Paie** ; voici la surface de cet onglet.

- **Sélecteur de période** : la liste des périodes de paie, au format « {début} au {fin} — {type} [FERMÉ] ». Trois **types** existent : **Hebdomadaire** (52 périodes/an), **Bi-hebdomadaire** (26/an) et **Mensuel** (12/an).

**Actions** (sur une période **non fermée**)

| Bouton | Effet |
|--------|-------|
| **Nouvelle période** | Ouvre une modale : **Date début \***, **Date fin \***, **Type de période**. |
| **Calculer la paie** (icône calculatrice) | Génère les fiches de la période. Si un brouillon existe déjà, une confirmation prévient qu'il sera **remplacé**. |
| **Fermer la période** (icône cadenas, rouge) | Confirmation « Fermer cette période de paie ? Cette action est irréversible. » Une fois fermée, un badge **« Période fermée »** s'affiche et les boutons d'action disparaissent. |

**Quatre cartes de totaux** (si des fiches existent) : **Employés**, **Masse brute**, **Masse nette**, **Coût employeur**.

**Tableau** : **Employé**, **Dept.**, **H. Reg**, **H. Supp** (en orange), **Brut**, **Déductions**, **Net**, **Coût Empl.**, **CCQ** (badge Oui/Non) et **Fiche** (icône qui ouvre le bulletin). Version en cartes sur téléphone.

**Modale « Fiche de paie »** (bulletin détaillé)

- **En-tête** : nom, poste — département, période, type.
- **Heures travaillées** : régulières, supplémentaires (en orange), taux horaire.
- **Trois cartes** : Salaire brut, Salaire net (vert), Coût employeur (violet).
- **Revenus** : « Indemnité de vacances payée » (si applicable).
- **Déductions de l'employé** : Impôt fédéral, Impôt provincial QC, **RRQ (6,40 %)**, **RRQ2 (4,00 %)** (si applicable), **RQAP (0,494 %)**, **AE (1,32 %)**, Total des déductions.
- **Charges de l'employeur** : RRQ employeur (6,40 %), RRQ2 employeur (4,00 %) (si applicable), RQAP employeur (0,692 %), AE employeur (1,848 %), **CNESST (1,80 %)**, **FSS (1,65 %)**, **CCQ (12,5 %)** (badge Applicable / N/A), Total des charges.
- « Vacances accumulées » (information, si applicable).
- Boutons **Fermer** et **« Télécharger le bulletin PDF »**.

> Les pourcentages ci-dessus sont les **étiquettes affichées dans la fiche**. La gouvernance des taux (paliers d'impôt, plafonds, exemptions, CCQ par métier) relève du module Paie — voir le **Module 15**.

### 2.7 Onglet « Feuillets fiscaux » (approbateurs seulement)

Production des feuillets de fin d'année. Un champ **« Année fiscale »** (nombre, de 2000 à 2100) pilote les trois cartes, chargées en parallèle.

**Carte « T4 (fédéral — ARC) »**

- Bouton **« Générer »**.
- **Avertissements** (bande ambre) en cas de données manquantes (par exemple un NAS absent).
- **Sommaire T4SUM** : « Feuillets produits », **Case 14 — Revenus**, **Case 22 — Impôt fédéral**, **Case 17 — RRQ**, **Case 18 — AE**, **Case 55 — RQAP**.
- Liste par employé avec un bouton de **téléchargement PDF** individuel.

**Carte « RL-1 (Québec — Revenu Québec) »**

- Badge **« Préliminaire »**.
- Bouton **« Générer »**.
- **Sommaire** : **Case A — Revenus**, **Case E — Impôt du Québec**, **Case B — RRQ**, **Case H — RQAP**.
- Liste par employé avec **PDF** individuel.

**Carte « PD7A — Remise des retenues (DAS) »**

- Menu déroulant **« Mois »** (1 à 12) et bouton **« Télécharger le PDF »**.
- **Trois cartes** : « À remettre — ARC (fédéral) », « À remettre — Revenu Québec », « Total à remettre ».
- Vide si aucune paie n'a été générée pour le mois choisi.

### 2.8 Onglet « Assistant IA »

Clavardage **en lecture seule** et **non monétaire** sur le volume d'heures.

- **En-tête** : « Assistant IA — Suivi des heures », sous-titre « Volume d'heures pointées à partir d'agrégats non monétaires (lecture seule). »
- **État vide** : trois exemples de questions (« Combien d'heures ont été pointées au total ? », « Quelle est la répartition des heures par type de travail ? », « Combien d'heures facturables cette semaine ? ») et une **note de confidentialité** : l'assistant n'accède à aucun montant de paie.
- **Clavardage** : bulles utilisateur / assistant avec des métadonnées (profil « Pointage », jetons, coût, durée), un indicateur « Analyse en cours… », une zone de saisie (Entrée pour envoyer, Maj+Entrée pour un retour à la ligne) et un bouton **« Envoyer »**. Un verrou empêche le double envoi.
- **Bilingue** (français / anglais selon la langue de l'interface).

> **Portée strictement limitée.** L'assistant ne dispose d'**aucun outil de requête SQL**. Le serveur lui fournit un **contexte agrégé fixe** (total d'heures, heures par type de travail, heures par facturabilité, 7 derniers jours) qui ne lit **jamais** le taux horaire, le coût, le salaire ni le NAS. Voir §4.13 pour le coût.

---

## 3. Procédures pas à pas

### 3.1 Créer un pointage manuel

1. Onglet **Pointages** → **Nouvelle saisie**.
2. Choisir l'**Employé** (obligatoire).
3. Au besoin, choisir le **Projet** et le **Bon de travail**.
4. Saisir l'**Entrée** et la **Sortie** (les deux, ou aucune). L'encart « Heures calculées » se met à jour automatiquement.
5. Ajouter des **Notes** ; laisser **Facturable** coché si l'heure est refacturable.
6. Cliquer **Créer**. Le pointage apparaît dans la liste, **à valider** par défaut.

> **Au moment de la création, le serveur fige le taux horaire** de l'employé dans le pointage. Si l'employé n'a pas de taux horaire, le système prend son salaire annuel ÷ 2080. Ce taux figé ne changera plus, même si vous modifiez la fiche de l'employé plus tard.
>
> **Refus possibles** : l'entrée doit précéder la sortie (sinon erreur), la durée ne peut dépasser 24 h, et la date ne doit pas tomber dans une **période de paie fermée** (sinon la création est refusée). Le système empêche aussi un **second pointage ouvert** pour le même employé (un pointage sans heure de sortie alors qu'un autre est déjà ouvert).

### 3.2 Rattacher une opération à un pointage

La modale de **création** ne propose pas l'opération. Pour la rattacher :

1. Créer (ou ouvrir) un pointage qui a déjà un **bon de travail**.
2. Cliquer sur le **crayon** (Modifier).
3. Dans le menu **Opération** (qui s'est rempli à partir du bon de travail), choisir l'opération.
4. **Enregistrer**.

> Le serveur vérifie que l'opération **appartient bien au bon de travail** choisi ; une opération étrangère au BT, ou une opération sans BT, est refusée.

### 3.3 Modifier un pointage

1. Onglet **Pointages** → **crayon** sur la ligne voulue.
2. Ajuster les champs : employé, projet, bon de travail, opération, entrée / sortie, type de travail, notes, facturable, validé.
3. **Enregistrer**. Seuls les champs réellement modifiés sont envoyés ; si rien n'a changé, la modale se ferme sans requête.

> Si vous changez l'entrée ou la sortie sans imposer une durée, le serveur **recalcule** les heures. Le **coût** est recalculé à partir du **taux figé** stocké dans le pointage, jamais du taux actuel de l'employé. Un pointage **facturé** ou pris dans une **période fermée** ne peut pas être modifié (erreur 400).

### 3.4 Valider ou dévalider un pointage

**Deux façons de valider** :

- **Depuis la liste** : cliquer sur le bouton **« Valider »** de la ligne. Le badge passe au vert et le serveur note **qui** a validé et **quand**.
- **Depuis la modale de modification** : cocher **« Validé »** puis enregistrer.

**Dévalider** : ouvrir la modification et décocher **« Validé »**. Le serveur efface la trace de validation.

> Il n'y a **pas** de validation groupée : une approbation par pointage. La validation est **réservée aux approbateurs** (au serveur, pas seulement à l'écran). Elle est refusée sur un pointage **facturé** ou dans une **période fermée**. Revalider un pointage déjà validé n'a aucun effet (opération sans conséquence).

### 3.5 Supprimer un pointage

1. Cliquer sur la **corbeille** de la ligne.
2. Confirmer (« Supprimer ce pointage ? »).

> La suppression est **définitive** (pas de corbeille de récupération). Elle est **refusée** si le pointage est **facturé** ou dans une **période fermée** (erreur 400). Réservée aux approbateurs.

### 3.6 Consulter la semaine et la capacité

1. Onglet **Vue semaine** → par défaut, la semaine courante (dimanche → samedi).
2. Naviguer avec **Semaine précédente** / **Semaine suivante**.
3. Lire le tableau des jours (entrées et total par jour) et le **Total semaine**.
4. Dans la carte **Capacité**, repérer les employés en **dépassement** (occupation ≥ 100 %, en rouge) ou en **sous-charge** (< 80 %, en vert).

### 3.7 Voir les heures par projet

1. Onglet **Par Projet** → les 20 principaux projets par heures.
2. Cliquer sur une ligne pour **déplier** le détail par employé.

> Pas de filtre de date : la vue agrège tout l'historique du tenant. Les pointages sans projet n'y apparaissent pas.

### 3.8 Produire le résumé de paie rapide

1. Onglet **Résumé paie** (approbateur).
2. Choisir la période (7 / 14 / 30 / 90 jours).
3. Lire la masse brute, le nombre d'employés et le tableau (heures, taux, brut, déductions, net).

> Rappel : chiffres **approximatifs** (RRQ 6,30 % + RQAP 0,43 % + AE 1,30 %, sans impôt ni charges de l'employeur). Pour publier, passer à la Paie CCQ.

### 3.9 Cycle complet de paie CCQ

1. Onglet **Paie CCQ** → **Nouvelle période** : saisir la **date début**, la **date fin** et le **type** (hebdomadaire, bi-hebdomadaire ou mensuel), puis créer.
2. Sélectionner la période → **Calculer la paie**. Le système lit les employés actifs, additionne leurs heures de la période, sépare le régulier des heures supplémentaires, calcule les retenues et les charges, et produit les fiches. (Recommençable tant que la période est ouverte ; un nouveau calcul **remplace** le brouillon existant après confirmation.)
3. Consulter une **Fiche de paie** (icône dans la colonne « Fiche ») : heures, brut / net / coût, retenues de l'employé, charges de l'employeur, CCQ.
4. **Télécharger le bulletin PDF** depuis la fiche.
5. Quand tout est vérifié, **Fermer la période** (confirmation ; **irréversible**). La fermeture **fige** les pointages de la période.

> Pour le détail des taux, des paliers d'impôt et de la CCQ, voir le **Module 15**.

### 3.10 Générer et télécharger les feuillets fiscaux

1. Onglet **Feuillets fiscaux** (approbateur) → saisir l'**Année fiscale**.
2. **T4** : cliquer **Générer**. Lire le sommaire T4SUM (cases 14, 22, 17, 18, 55) ; corriger les employés signalés en **avertissement** (par exemple un NAS manquant) ; télécharger chaque **PDF** individuel.
3. **RL-1** : cliquer **Générer** (feuillet marqué **« Préliminaire »**). Lire le sommaire (cases A, E, B, H) ; télécharger chaque **PDF**.
4. **PD7A** : choisir le **Mois** → **Télécharger le PDF**. Le relevé affiche « À remettre — ARC », « À remettre — Revenu Québec » et le total.

> Les feuillets s'appuient sur la paie CCQ **générée** : sans fiches de paie pour la période, les sommaires restent vides. Le NAS et l'adresse de chaque employé (saisis au Module 11) sont nécessaires à un feuillet complet.

### 3.11 Exporter les pointages en CSV

1. Cliquer **Exporter** (barre de titre).
2. Le fichier **`pointages_export.csv`** se télécharge. Colonnes : **ID, Employé, Projet, BT Numéro, Entrée, Sortie, Heures, Type, Notes, Validé** (Oui / Non).

> Le CSV **ne contient aucun montant** (ni taux, ni coût, ni salaire) : il ne peut donc pas divulguer de données de paie. Les champs texte sont **neutralisés** contre l'injection de formules (protection à l'ouverture dans un tableur). Des filtres (`employee_id`, `date_debut`, `date_fin`) existent dans l'API mais ne sont pas exposés dans l'interface de bureau.

### 3.12 Interroger l'assistant IA

1. Onglet **Assistant IA**.
2. Poser une question sur le **volume d'heures** (« Combien d'heures facturables cette semaine ? »).
3. Lire la réponse ; le coût et la durée s'affichent dessous.

> Si vous demandez un montant de paie, un taux ou un salaire, l'assistant **ne peut pas** répondre : ces données ne lui sont jamais fournies. Chaque message consomme des **crédits IA** (voir §4.13).

---

## 4. Référence

### 4.1 Les 7 onglets et leur backend

| Onglet | Point d'accès principal | Routeur |
|--------|-------------------------|---------|
| Pointages | `/employees/time-entries` (+ create / update / delete / validate / export-csv) | `employees.py` |
| Vue semaine | `/employees/time-entries/weekly` | `employees.py` |
| Par Projet | `/employees/time-entries/by-project` | `employees.py` |
| Résumé paie | `/employees/payroll-summary` | `employees.py` |
| Paie CCQ | `/payroll/*` | `payroll.py` |
| Feuillets fiscaux | `/feuillets/t4|rl1|pd7a` | `feuillets_t4/rl1/pd7a.py` |
| Assistant IA | `/pointage/ai/chat` | `pointage_ai.py` |

### 4.2 Colonnes du tableau « Pointages »

| Colonne | Source |
|---------|--------|
| Employé | Prénom + nom de l'employé. |
| Client | Nom du client (via le bon de travail). |
| Projet | Nom du projet. |
| BT | Numéro du bon de travail. |
| Opération | Nom ou description de l'opération. |
| Début / Fin | Heure d'entrée / de sortie. |
| Heures | Durée calculée. |
| Statut | Facturé / Validé / À valider (ou bouton Valider). |
| Actions | Modifier / Supprimer (approbateurs). |

### 4.3 Statuts d'un pointage

| Statut | Condition | Affichage |
|--------|-----------|-----------|
| **Facturé** | Le pointage est inclus dans une facture | Badge bleu « Facturé » ; modification / suppression / validation bloquées |
| **Validé** | Approuvé, non facturé | Badge vert « Validé » (coche) |
| **À valider** | Non approuvé, non facturé | Bouton « Valider » (approbateur) ou badge gris « À valider » |

### 4.4 Filtres et recherche (onglet Pointages)

| Filtre | Comportement |
|--------|--------------|
| **Recherche** | **Au serveur** (délai 350 ms) sur employé, projet, bon de travail, client, opération et notes ; couvre tout le jeu de données. |
| **Valides** | Uniquement les pointages validés. |
| **Non valides** | Uniquement les non validés. |
| **Facturés** | Uniquement les facturés. |
| **Tous** | Aucun filtre (défaut). |

### 4.5 Champs des modales

| Champ | Création | Modification |
|-------|----------|--------------|
| Employé | oui (obligatoire) | oui |
| Projet | oui | oui |
| Bon de travail | oui | oui |
| Opération | — | oui (dépend du BT) |
| Entrée / Sortie | oui | oui (précision à la seconde) |
| Heures calculées | affichage auto | affichage auto |
| Type de travail | — | oui (texte libre) |
| Notes | oui | oui |
| Facturable | oui (coché) | oui |
| Validé | — | oui |

### 4.6 Validations et verrous (au serveur)

| Règle | Effet |
|-------|-------|
| Employé absent (création) | Bouton « Créer » désactivé ; refus au serveur. |
| Un seul des deux horodatages (création) | Refus (fournir les deux ou aucun). |
| Sortie avant l'entrée | Erreur 400. |
| Durée > 24 h (contrôle de saisie) | Refus à l'écran. |
| Durée saisie hors bornes | Bornée à ≥ 0 et ≤ 100 000 (garde anti-débordement). |
| Date dans une **période fermée** (création ou déplacement) | Refus (erreur 400). |
| Second **pointage ouvert** pour le même employé | Refus (erreur 409). |
| Opération étrangère au bon de travail | Erreur 400. |
| Opération sans bon de travail | Erreur 400. |
| Modifier / supprimer / valider un pointage **facturé** | Erreur 400. |
| Modifier / supprimer / valider dans une **période fermée** | Erreur 400. |
| Écriture par un **non-approbateur** | Erreur 403. |
| **Mode consultation** (abonnement suspendu) | Toutes les écritures bloquées (403). |

### 4.7 Calcul des heures

| Endroit | Formule |
|---------|---------|
| À l'écran | `(sortie − entrée)` en heures, arrondi à 2 décimales, affiché en direct. |
| Au serveur (création) | `round((sortie − entrée) / 3600, 2)` si la durée n'est pas fournie. |
| Au serveur (modification) | Recalcul identique si l'entrée ou la sortie change sans durée imposée. |
| Coût | `heures × taux figé` (le taux capté au pointage, jamais le taux courant). |

### 4.8 Résumé paie — taux (approximatifs)

| Retenue | Taux |
|---------|------|
| RRQ | 6,30 % |
| RQAP (employé) | 0,43 % |
| AE (employé) | 1,30 % |
| **Total des retenues** | **≈ 8,03 %** du brut |

**Exclus** : plafonds, exemption RRQ, impôt fédéral, impôt provincial, CNESST, FSS, CCQ, charges de l'employeur. Employés **actifs** seulement.

### 4.9 Paie CCQ — pourcentages affichés dans la fiche

| Retenue de l'employé | Étiquette |
|----------------------|-----------|
| RRQ | 6,40 % |
| RRQ2 | 4,00 % |
| RQAP | 0,494 % |
| AE | 1,32 % |
| Impôt fédéral / provincial | selon les paliers (Module 15) |

| Charge de l'employeur | Étiquette |
|-----------------------|-----------|
| RRQ | 6,40 % |
| RRQ2 | 4,00 % |
| RQAP | 0,692 % |
| AE | 1,848 % |
| CNESST | 1,80 % |
| FSS | 1,65 % |
| CCQ | 12,5 % (badge Applicable / N/A) |

> Étiquettes telles qu'affichées dans le bulletin. Gouvernance des taux, plafonds et paliers : **Module 15**.

### 4.10 Feuillets fiscaux — cases affichées

| Feuillet | Cases du sommaire |
|----------|-------------------|
| **T4 (T4SUM)** | Case 14 (Revenus), Case 22 (Impôt fédéral), Case 17 (RRQ), Case 18 (AE), Case 55 (RQAP) |
| **RL-1** | Case A (Revenus), Case E (Impôt du Québec), Case B (RRQ), Case H (RQAP) |
| **PD7A** | À remettre — ARC, À remettre — Revenu Québec, Total à remettre (par mois) |

### 4.11 Export CSV — colonnes

`ID`, `Employé`, `Projet`, `BT Numéro`, `Entrée`, `Sortie`, `Heures`, `Type`, `Notes`, `Validé` (Oui / Non). Fichier `pointages_export.csv`. **Aucun montant** exporté.

### 4.12 Points d'accès (API)

**Feuilles de temps** (base `/api/erp/v1/employees`) :

| Méthode + chemin | Rôle requis | Rôle |
|------------------|-------------|------|
| `GET /employees/time-entries` | Tout utilisateur du tenant | Liste paginée + recherche + filtres. |
| `POST /employees/time-entries` | **Approbateur** | Créer (taux figé, verrous période / double pointage ouvert). |
| `GET /employees/payroll-summary` | **Approbateur** | Résumé de paie **approximatif**. |
| `PUT /employees/time-entries/{id}/validate` | **Approbateur** | Valider (idempotent, audité). |
| `PUT /employees/time-entries/{id}` | **Approbateur** | Modifier (diff seulement, recalcul sur taux figé). |
| `DELETE /employees/time-entries/{id}` | **Approbateur** | Supprimer (refus si facturé / fermé). |
| `GET /employees/time-entries/weekly` | Tout utilisateur du tenant | Feuille dimanche → samedi + capacité. |
| `GET /employees/time-entries/by-project` | Tout utilisateur du tenant | Heures par projet (top 20). |
| `GET /employees/time-entries/export-csv` | Tout utilisateur du tenant | Export CSV (sans montant). |

**Paie CCQ** (base `/api/erp/v1/payroll`) : `GET/POST /payroll/periods`, `POST /payroll/generate`, `GET /payroll/entries`, `GET /payroll/entries/{id}`, `GET /payroll/entries/{id}/pdf`, `PUT /payroll/periods/{id}/close`.

**Feuillets** (base `/api/erp/v1/feuillets`) : `.../t4/generate|summary|{id}/pdf`, `.../rl1/generate|summary|{id}/pdf`, `.../pd7a` (+ `/pdf`).

**Assistant** (base `/api/erp/v1/pointage/ai`) : `POST /pointage/ai/chat`.

### 4.13 Assistant IA — fonctionnement et coût

| Aspect | Détail |
|--------|--------|
| **Portée** | Volume d'heures seulement : total, par type de travail, par facturabilité, 7 derniers jours. |
| **Accès aux données** | **Aucun outil SQL.** Contexte agrégé fixe, préparé par des requêtes qui ne lisent **jamais** taux, coût, salaire ni NAS. |
| **Écriture** | Aucune. Lecture seule stricte. |
| **Modèle** | `claude-sonnet-4-6`, réponse plafonnée à 4 000 jetons, un seul appel. |
| **Bilingue** | Français / anglais selon l'interface. |
| **Coût** | (jetons d'entrée × 0,003 $ + jetons de sortie × 0,015 $) ÷ 1000, puis **× 1,30** (majoration 30 %). Débité des **crédits IA prépayés** du tenant. |
| **Erreurs** | 403 si la garde IA refuse ; 402 si les crédits sont épuisés ; 503 si le service IA est indisponible ; bloqué en mode consultation. |

---

## 5. Intégrations et FAQ

### 5.1 Module 11 — Employés

- La fiche employé fournit le **taux horaire** (ou le salaire) **figé** dans chaque pointage à sa création, ainsi que la **capacité hebdomadaire** utilisée dans la carte de capacité.
- Seuls les employés **actifs** entrent dans le Résumé paie et la Paie CCQ.
- Le **NAS** et l'**adresse** saisis au Module 11 sont nécessaires aux feuillets T4 / RL-1.
- Cet écran ne montre pas la fiche RH ; il ne fait qu'utiliser ces données.

### 5.2 Module 12 — Bons de travail

- Un pointage peut se rattacher à un **bon de travail** et à une **opération** de ce BT (l'opération se choisit dans la modale de modification).
- Le numéro du bon de travail et le client (via le BT) apparaissent dans le tableau.
- **Aucun report automatique** : les heures pointées ne remplissent pas les « heures réelles » des opérations du bon de travail — cette saisie reste manuelle au Module 12.

### 5.3 Module 09 — Projets

- Un pointage peut se rattacher à un **projet** ; l'onglet **Par Projet** agrège les heures par projet.
- Le coût de la main-d'œuvre (heures × taux figé) alimente la vue financière du projet.
- La vue Par Projet n'a **pas** de filtre de date et exclut les pointages sans projet.

### 5.4 Module 15 — Comptabilité / Paie / Feuillets

- La **Paie CCQ** (onglet 5) et les **Feuillets fiscaux** (onglet 6) sont des surfaces de ce module de bureau, mais leur moteur vit dans le module Comptabilité / Paie (taux, paliers, plafonds, CCQ par métier).
- **Lien passif avec la facturation** : lorsqu'un pointage est facturé ailleurs (Comptabilité / Dossiers), il reçoit un indicateur « facturé » qui le **verrouille** ici (modification / suppression / validation refusées). Le module Pointage ne pose jamais lui-même ce verrou.
- Fermer une période de paie **fige** les pointages de la période, ce qui protège les cumuls des feuillets.

### 5.5 Application mobile (`/mobile`)

- L'application mobile est le **vrai pointeur** : l'employé y pointe son arrivée et son départ, avec son **NIP à 4 chiffres** et la **géolocalisation**.
- Les pointages mobiles et de bureau **partagent la même table** ; ils apparaissent ensemble dans l'onglet Pointages, sans marqueur de source.
- Documentation : manuel séparé MOBILE_REACT.

### 5.6 GPS et logistique (aucun lien avec le pointage)

- La page `/pointage` **n'utilise aucune** fonction GPS. Il n'y a pas de colonne de position dans les feuilles de temps, et aucun pointage automatique par zone géographique.
- Le suivi GPS de l'entreprise concerne la **flotte de véhicules** (véhicules, positions, zones géographiques, trajets) — un module distinct. La géolocalisation d'un **pointage** existe uniquement dans l'application mobile.

### 5.7 Assistant IA (portée)

- L'assistant de ce module est **distinct** de l'Assistant IA général de l'ERP (Module 25) et des assistants comptables. Il est volontairement **cloisonné** : aucune requête libre en base, aucun montant de paie, aucune donnée nominative sensible. Pour toute analyse financière, se tourner vers les modules comptables et leurs contrôles.

### 5.8 FAQ

**Q : Pourquoi n'y a-t-il pas de bouton « Pointer mon arrivée » sur le web ?**
R : La page de bureau est faite pour la **saisie administrative** (un approbateur inscrit les heures d'un employé). Le vrai pointeur en direct, géolocalisé, est l'**application mobile** (`/mobile`), où l'employé pointe avec son NIP.

**Q : Un employé peut-il saisir ses propres heures sur le web ?**
R : Non. Sur le web, seul un **approbateur** (administrateur ou rôle admin / super-admin) crée les pointages. Pour le libre-service, c'est le mobile.

**Q : Le module fait-il de la géolocalisation GPS ?**
R : Non, pas dans `/pointage`. Le GPS de l'ERP suit la **flotte de véhicules**, pas les pointages. La position d'un pointage n'est captée que sur le mobile.

**Q : Pourquoi je vois l'onglet « Résumé paie » mais j'obtiens une erreur ?**
R : Parce que vous n'êtes pas **approbateur**. L'onglet reste affiché, mais le chargement des chiffres est réservé aux administrateurs (protection de la masse salariale). Ce n'est pas un bogue.

**Q : Le « Résumé paie » donne-t-il des chiffres officiels ?**
R : Non. C'est une **estimation** (RRQ 6,30 % + RQAP 0,43 % + AE 1,30 %, sans impôt, sans plafond, sans charges de l'employeur). Les chiffres publiables sont dans la **Paie CCQ**.

**Q : Pourquoi la semaine commence-t-elle le dimanche ?**
R : La feuille de temps est alignée sur le **calendrier de paie de la construction (CCQ)** : dimanche au samedi.

**Q : Que se passe-t-il si je modifie l'entrée sans toucher aux heures ?**
R : Le serveur **recalcule** la durée à partir des nouveaux horodatages. Le coût, lui, se recalcule sur le **taux figé** du pointage, jamais sur le taux actuel de l'employé.

**Q : Pourquoi je ne peux pas modifier ni supprimer un pointage ?**
R : Il est probablement **facturé** ou pris dans une **période de paie fermée**. Dans les deux cas, il est verrouillé. Pour le corriger, il faut d'abord annuler la facture (module Comptabilité) ou passer par une période d'ajustement.

**Q : Puis-je valider tous les pointages d'un coup ?**
R : Non. Il n'y a pas de validation groupée ; on valide un pointage à la fois.

**Q : Comment gérer une pause repas non payée ?**
R : Aucune déduction automatique. Saisir deux pointages (par exemple 8 h–12 h et 13 h–17 h) ou ajuster la durée.

**Q : Puis-je pointer plusieurs employés en une seule saisie ?**
R : Non. Un pointage = un employé. Pour dix employés, dix saisies.

**Q : L'export CSV contient-il les salaires ?**
R : Non. Le CSV n'exporte **aucun montant** (ni taux, ni coût, ni salaire) — seulement les heures et les métadonnées. Il ne peut donc pas divulguer la paie.

**Q : Comment savoir si un pointage vient du mobile ?**
R : Il n'y a **pas** de marqueur de source visible. Les pointages mobiles et de bureau sont mêlés dans la même liste.

**Q : L'assistant IA peut-il me donner la masse salariale ou un taux horaire ?**
R : Non. Il répond uniquement sur des **volumes d'heures** ; il n'a jamais accès aux montants, aux taux ni aux salaires. Chaque question consomme des crédits IA.

**Q : Peut-on rouvrir une période de paie fermée ?**
R : Non, la fermeture est **irréversible**. Il faut passer par une période d'ajustement (module Paie).

**Q : Les heures supplémentaires apparaissent-elles dans l'onglet Pointages ?**
R : Non. Un pointage de 45 h s'affiche simplement « 45 h ». La séparation régulier / supplémentaire se calcule à la **génération de la paie CCQ**.

---

## 6. Récapitulatif

- **Page** `/pointage`, titre **« Pointage & Paie »**, **7 onglets** : Pointages, Vue semaine, Par Projet, Résumé paie, Paie CCQ, Feuillets fiscaux, Assistant IA.
- **Saisie manuelle sur le web** (deux champs date-heure) — **pas** de pointeur en temps réel ni de géolocalisation. Le vrai pointage géolocalisé est sur l'**application mobile**.
- **Écritures réservées aux approbateurs** (administrateur, rôle admin / super-admin) : créer, valider, modifier, supprimer + Résumé paie + Paie CCQ + Feuillets. La consultation est ouverte à tous.
- **Taux figé au pointage** : le coût utilise le taux capté au moment de la saisie, jamais recalculé rétroactivement.
- **Verrous** : un pointage **facturé** ou dans une **période de paie fermée** est immuable (modification / suppression / validation refusées). Anti double pointage ouvert (409), sortie après l'entrée, durée ≤ 24 h.
- **Vue semaine** alignée CCQ (**dimanche → samedi**) avec carte de **capacité** (vert < 80 %, jaune 80-100 %, rouge > 100 %).
- **Par Projet** : top 20 projets, forage par employé, sans filtre de date, exclut les pointages sans projet.
- **Résumé paie = approximatif** (RRQ 6,30 % + RQAP 0,43 % + AE 1,30 %, sans impôt ni charges). La vraie paie est la **Paie CCQ** (retenues, charges de l'employeur, CCQ 12,5 %, bulletins PDF).
- **Feuillets fiscaux** (approbateurs) : **T4** (cases 14/22/17/18/55), **RL-1** (cases A/E/B/H, « Préliminaire »), **PD7A** par mois. PDF individuels.
- **Export CSV** (10 colonnes, **sans montant**) ; les seuls PDF sont les bulletins de paie et les feuillets.
- **Assistant IA** : lecture seule, volumes d'heures seulement, aucun accès aux montants ; débité des crédits IA (majoration 30 %).
- **Aucun lien GPS**, aucune déduction de pause automatique, aucune détection de retard, aucune validation groupée, aucune réouverture de période fermée, aucun téléversement de fichier.

---

**Documentation générée à partir du code** : `backend/routers/employees.py` (feuilles de temps + résumé de paie), `backend/routers/pointage_ai.py` (assistant), `backend/routers/payroll.py` (paie CCQ), `backend/routers/feuillets_t4.py` / `feuillets_rl1.py` / `feuillets_pd7a.py` (feuillets), `backend/routers/payroll_rates.py` (taux), `frontend/src/pages/PointagePage.tsx`, `frontend/src/components/pointage/PointageAssistantTab.tsx`, `frontend/src/api/employees.ts` / `payroll.ts` / `feuillets.ts` / `pointageAi.ts` / `projects.ts` / `production.ts`.

**Manuels liés** :
- Module 11 (Employés — taux horaire, capacité, NAS, NIP mobile) — `11-operations-employes.md`
- Module 12 (Bons de travail — opérations, heures réelles) — `12-operations-bons-de-travail.md`
- Module 09 (Projets — coût de la main-d'œuvre, heures par projet) — `09-ventes-projets.md`
- Module 15 (Comptabilité / Paie — taux, paliers, feuillets T4 / RL-1 / PD7A) — `15-operations-comptabilite.md`
- Module 25 (Assistant IA général de l'ERP) — `25-communication-assistant-ia.md`
- Module 28 (Configuration — utilisateurs et rôles) — `28-configuration.md`
- Manuel séparé : MOBILE_REACT (application mobile de pointage terrain géolocalisé)
