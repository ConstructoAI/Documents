# Manuel d'utilisation - Module Immobilier

## Constructo AI - PGI Immobilier v2.5

---

## Table des matieres

1. [Introduction](#1-introduction)
2. [Acces au module](#2-acces-au-module)
3. [Tableau de bord](#3-tableau-de-bord)
4. [Gestion des Terrains](#4-gestion-des-terrains)
5. [Financement et Prets](#5-financement-et-prets)
6. [Projets Immobiliers](#6-projets-immobiliers)
7. [Suivi de Construction](#7-suivi-de-construction)
8. [Gestion des Unites](#8-gestion-des-unites)
9. [Commercialisation](#9-commercialisation)
10. [Livraison et Garanties](#10-livraison-et-garanties)
11. [Calculateurs Financiers](#11-calculateurs-financiers)
12. [Assistant IA - Conseils Expert](#12-assistant-ia---conseils-expert)
13. [Glossaire](#13-glossaire)
14. [FAQ](#14-faq)

---

## 1. Introduction

### 1.1 Presentation du module

Le **Module Immobilier** de Constructo AI est une solution complete pour la gestion de projets de construction multi-logements au Quebec. Il couvre l'integralite du cycle de vie d'un projet immobilier :

- **Prospection** : Recherche et evaluation de terrains
- **Financement** : Montage financier et suivi des deblocages
- **Construction** : Gestion des phases de chantier
- **Commercialisation** : Ventes et locations
- **Livraison** : Remise des cles et garanties

### 1.2 Specificites quebecoises

Le module est adapte au contexte quebecois et canadien :

- Financement progressif (deblocages par etapes)
- Interets intercalaires pendant la construction
- Normes SCHL (Societe canadienne d'hypotheques et de logement)
- Taux Banque du Canada
- Taxes et frais quebecois (notaire, taxe de bienvenue)
- Code National du Batiment (CNB)
- Garantie des batiments residentiels neufs (GCR)

### 1.3 Prerequis

Pour utiliser ce module, vous devez :
- Etre connecte a Constructo AI avec un compte utilisateur actif
- Avoir les permissions d'acces au module Immobilier
- Pour les fonctionnalites IA : une cle API Anthropic Claude configuree

---

## 2. Acces au module

### 2.1 Navigation

1. Connectez-vous a Constructo AI
2. Dans le menu de navigation lateral, cliquez sur **"Immobilier"**
3. Le module s'ouvre avec 10 onglets disponibles

### 2.2 Structure du module

Le module est organise en **10 onglets principaux** :

| Onglet | Icone | Description |
|--------|-------|-------------|
| Tableau de bord | 📊 | Vue d'ensemble et KPIs |
| Terrains | 🏞️ | Prospection et acquisition |
| Financement | 💰 | Prets et deblocages |
| Projets Immobiliers | 🏗️ | Gestion globale des projets |
| Construction | 👷 | Suivi des phases de chantier |
| Unites (Logements) | 🏘️ | Gestion des logements/commerces |
| Commercialisation | 📢 | Marketing et ventes |
| Livraison | 🔑 | Remise des cles et garanties |
| Calculateurs Financiers | 🧮 | Outils de calcul |
| Conseils IA | 🧠 | Assistant expert IA |

### 2.3 Indicateur IA

En haut du module, un indicateur affiche le statut de l'assistant IA :

- **Vert** : "Assistant IA Claude Opus 4.5 actif" - L'IA est disponible
- **Orange** : "Assistant IA non disponible" - Verifiez la cle API

---

## 3. Tableau de bord

### 3.1 Apercu financier

Le tableau de bord presente des **indicateurs cles (KPIs)** :

| Indicateur | Description |
|------------|-------------|
| Financements totaux | Nombre total de demandes de financement |
| Financements approuves | Nombre de financements valides par les banques |
| Total demande | Somme des montants demandes ($) |
| Taux moyen | Taux d'interet moyen de vos financements (%) |

### 3.2 Graphiques

Lorsque vous avez des financements enregistres :

- **Repartition par statut** : Graphique circulaire des statuts (En preparation, Approuve, etc.)
- **Montants par institution** : Histogramme des montants par banque

### 3.3 Calculateur rapide

Si vous n'avez pas encore de financements, un calculateur rapide est disponible :

1. Entrez le **Montant** du pret
2. Saisissez le **Taux d'interet** (%)
3. Indiquez la **Duree** en annees
4. Cliquez sur **"Calculer la mensualite"**

Le systeme affiche :
- La mensualite mensuelle
- Les interets totaux
- Le cout total du credit

---

## 4. Gestion des Terrains

### 4.1 Presentation

L'onglet **Terrains** permet de gerer la prospection et l'acquisition de terrains pour vos projets de construction.

### 4.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Liste | Affiche tous vos terrains enregistres |
| Nouveau terrain | Formulaire d'ajout d'un nouveau terrain |
| Analyse | Outils d'analyse de faisabilite |

### 4.3 Ajouter un nouveau terrain

1. Cliquez sur l'onglet **"Nouveau terrain"**
2. Remplissez les informations :

**Informations obligatoires :**
- **Adresse** : Adresse complete du terrain
- **Ville** : Ville (par defaut : Montreal)

**Informations optionnelles :**
- **Code postal**
- **Superficie (m2)** : Surface du terrain
- **Zonage** : Ex: Residentiel R3
- **Prix demande ($)** : Prix affiche par le vendeur
- **Proprietaire** : Nom du proprietaire actuel
- **Numero cadastre** : Reference cadastrale
- **Potentiel de construction** : Description des possibilites

3. Cliquez sur **"Enregistrer le terrain"**

### 4.4 Informations stockees

Pour chaque terrain, le systeme conserve :

- Identification (adresse, lot, cadastre)
- Caracteristiques (superficie, zonage, potentiel)
- Informations proprietaire
- Donnees financieres (prix demande, offre, evaluation)
- Contraintes (servitudes, environnement, acces services)
- Documents (certificat de localisation, etude de sol, permis)

---

## 5. Financement et Prets

### 5.1 Presentation

L'onglet **Financement** est le coeur du module pour la gestion des prets de construction. Il permet de :

- Creer des demandes de financement
- Suivre l'approbation des prets
- Gerer les deblocages progressifs

### 5.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Liste | Tous vos financements avec filtres |
| Nouveau financement | Formulaire de demande |
| Deblocages | Gestion des deblocages progressifs |

### 5.3 Liste des financements

**Filtres disponibles :**
- **Statut** : Tous, En preparation, En cours, Approuve, Refuse, Complete
- **Banque** : Recherche par nom de banque

**Informations affichees :**
- Numero de financement
- Banque
- Montant demande
- Type de pret
- Taux d'interet
- Statut

**Actions :**
- **Deblocages** : Voir/gerer les deblocages
- **Supprimer** : Supprimer le financement

### 5.4 Creer une demande de financement

#### Etape 1 : Informations du projet

Selectionnez le projet associe (par ID temporairement).

#### Etape 2 : Institution financiere

| Champ | Description |
|-------|-------------|
| Banque | Choisir parmi : Banque Nationale, Desjardins, RBC, TD, Scotia, BMO, CIBC |
| Nom du conseiller | Votre contact a la banque |
| Contact conseiller | Email ou telephone |

#### Etape 3 : Type de pret

- **Pret construction** : Financement pendant la construction
- **Hypothecaire commercial** : Pret permanent apres construction
- **Pret-relais (pont)** : Financement temporaire
- **Ligne de credit** : Credit revolving

#### Etape 4 : Conditions financieres

| Champ | Description | Valeur typique |
|-------|-------------|----------------|
| Montant demande ($) | Capital a emprunter | Variable |
| Taux d'interet annuel (%) | Taux propose | 6.5% |
| Type de taux | Fixe, Variable, Preferentiel | Fixe |
| Duree amortissement (annees) | Periode de remboursement | 25 ans |
| Frequence paiement | Mensuel, Bi-hebdo, Hebdo | Mensuel |
| Mise de fonds (%) | Apport personnel | 25% |

#### Etape 5 : Options

- **Financement progressif** : Deblocages par etapes (recommande)
- **Assurance pret SCHL** : Pour ratios pret/valeur > 80%
- **Duree construction** : Nombre de mois prevus

#### Etape 6 : Frais

| Frais | Description | Montant typique |
|-------|-------------|-----------------|
| Evaluation | Expertise du bien | 2 000 $ |
| Notaire | Frais juridiques | 5 000 $ |
| Ouverture | Frais d'ouverture dossier | 1 000 $ |
| Autres frais | Divers | Variable |

#### Etape 7 : Enregistrement

Cliquez sur **"Enregistrer la demande de financement"**.

Si le financement progressif est active, le systeme propose de generer automatiquement le calendrier de deblocages.

### 5.5 Gestion des deblocages progressifs

#### Qu'est-ce qu'un deblocage progressif ?

Dans un pret de construction, la banque ne verse pas tout le montant d'un coup. Elle **debloque** les fonds par etapes, selon l'avancement du chantier.

#### Calendrier standard (genere automatiquement)

| Etape | Pourcentage | Description |
|-------|-------------|-------------|
| 1 | 10% | Terrain et preparation |
| 2 | 15% | Fondations |
| 3 | 25% | Charpente et structure |
| 4 | 15% | Toiture et enveloppe |
| 5 | 20% | Plomberie, electricite, CVC |
| 6 | 10% | Finitions interieures |
| 7 | 5% | Finition finale et nettoyage |

#### Generer un calendrier automatique

1. Selectionnez un financement
2. Si aucun deblocage n'existe, cliquez sur **"Generer calendrier deblocages automatique"**
3. Indiquez la duree de construction en mois
4. Le systeme cree les 7 deblocages standards

#### Visualisation

- **Tableau** : Liste des deblocages avec statut et montants
- **Graphique** : Comparaison montants prevus vs debloques

#### Statuts de deblocage

| Statut | Description |
|--------|-------------|
| Planifie | Deblocage prevu |
| En cours | Demande envoyee a la banque |
| Approuve | Banque a valide |
| Debloque | Fonds recus |

---

## 6. Projets Immobiliers

### 6.1 Presentation

L'onglet **Projets Immobiliers** permet de creer et gerer vos projets de construction dans leur globalite.

### 6.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Liste | Tous vos projets en cours |
| Nouveau projet | Formulaire de creation |
| Rentabilite | Analyse ROI |

### 6.3 Creer un nouveau projet

#### Informations generales

| Champ | Description | Exemple |
|-------|-------------|---------|
| Nom du projet | Nom commercial | Les Jardins du Mont-Royal |
| Type | Residentiel, Commercial, Mixte | Residentiel |
| Nombre de logements | Unites residentielles | 65 |
| Terrain associe | Lien vers terrain existant | - |

#### Details financiers

| Champ | Description | Exemple |
|-------|-------------|---------|
| Budget total ($) | Cout global du projet | 25 000 000 $ |
| Cout terrain ($) | Prix d'acquisition | 3 000 000 $ |
| Cout construction ($) | Travaux de construction | 20 000 000 $ |
| Amenagements ($) | Paysagement, stationnement | 2 000 000 $ |

#### Planification

| Champ | Description |
|-------|-------------|
| Date debut planifiee | Lancement prevu du chantier |
| Duree construction (mois) | Temps estime |

---

## 7. Suivi de Construction

### 7.1 Presentation

L'onglet **Construction** permet de suivre l'avancement des phases de votre chantier en temps reel.

### 7.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Phases en cours | Liste des phases avec statut |
| Nouvelle phase | Ajouter une phase |
| Avancement global | Statistiques et graphiques |
| Gantt | Diagramme de Gantt (en developpement) |

### 7.3 Phases de construction

#### Phases standards predefinies

Le systeme propose 12 phases types :

1. Excavation et terrassement
2. Fondations
3. Structure (charpente)
4. Toiture
5. Enveloppe exterieure
6. Plomberie et electricite (rough-in)
7. CVC (Chauffage, Ventilation, Climatisation)
8. Isolation
9. Gypse
10. Finitions interieures
11. Amenagement paysager
12. Inspections finales

#### Ajouter une nouvelle phase

1. Selectionnez le **Projet**
2. Choisissez une **Phase standard** ou creez une phase personnalisee
3. Remplissez les informations :

| Champ | Description |
|-------|-------------|
| Numero de phase | Ordre d'execution |
| Nom de la phase | Description |
| Statut | A venir, En cours, En retard, Completee, Suspendue |
| Avancement (%) | Pourcentage de completion |
| Date debut prevue | Planification |
| Date fin prevue | Echeance |
| Budget prevu ($) | Cout estime |

**Details techniques :**
- Conforme CNB (Code National du Batiment)
- Materiaux commandes / recus
- Retard (jours) avec raison

4. Cliquez sur **"Enregistrer la phase"**

### 7.4 Statuts de phase

| Statut | Couleur | Description |
|--------|---------|-------------|
| A venir | Gris | Non commencee |
| En cours | Bleu | Travaux en progression |
| En retard | Rouge | Depassement du delai |
| Completee | Vert | Terminee |
| Suspendue | - | Arret temporaire |

### 7.5 Avancement global

Statistiques affichees :
- Total des phases
- Avancement moyen (%)
- Phases completees
- Phases en retard
- Budget total vs Cout reel
- Variance budgetaire

**Graphique** : Histogramme de l'avancement par phase avec couleurs selon statut.

---

## 8. Gestion des Unites

### 8.1 Presentation

L'onglet **Unites (Logements)** permet de gerer l'inventaire des logements et espaces commerciaux de votre projet.

### 8.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Inventaire | Liste de toutes les unites |
| Nouvelle unite | Ajouter une unite |
| Statistiques | Ventes et occupations |

### 8.3 Types d'unites

| Type | Description |
|------|-------------|
| Logement | Appartement residentiel |
| Commercial | Bureau, boutique |
| Stationnement | Place de parking |

### 8.4 Sous-types residentiels

- 3 1/2 (1 chambre)
- 4 1/2 (2 chambres)
- 5 1/2 (3 chambres)
- 6 1/2 (4 chambres)

### 8.5 Ajouter une unite

1. Selectionnez le **Projet**
2. Remplissez les informations :

| Champ | Description | Exemple |
|-------|-------------|---------|
| Numero d'unite | Identifiant | 101 |
| Type | Logement, Commercial, Stationnement | Logement |
| Sous-type | 3 1/2, 4 1/2, etc. | 4 1/2 |
| Superficie (pi2) | Surface habitable | 850 |
| Etage | Niveau | 1 |
| Nombre de chambres | Pieces | 2 |
| Prix de vente ($) | Prix affiche | 350 000 $ |

3. Cliquez sur **"Enregistrer l'unite"**

### 8.6 Statuts d'unite

| Statut | Description |
|--------|-------------|
| Disponible | A vendre ou a louer |
| Promesse d'achat | Offre acceptee |
| Vendu | Transaction completee |
| Loue | Sous bail |

---

## 9. Commercialisation

### 9.1 Presentation

L'onglet **Commercialisation** gere le marketing, les ventes et les locations de votre projet.

### 9.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Vue d'ensemble | KPIs de commercialisation |
| Nouvelle campagne | Creer une campagne marketing |
| Ventes | Suivi des transactions |
| Locations | Gestion des baux |

### 9.3 Vue d'ensemble

**Indicateurs affiches :**

| KPI | Description |
|-----|-------------|
| Total unites | Nombre d'unites dans le projet |
| Vendues | Unites vendues (avec %) |
| Louees | Unites sous bail (avec %) |
| Disponibles | Unites encore disponibles |
| Prix moyen vente | Moyenne des prix de vente |
| Loyer moyen | Moyenne des loyers mensuels |

**Informations campagne active :**
- Strategie de vente
- Site web
- Budget marketing / Cout reel
- Objectif pre-ventes avec barre de progression

### 9.4 Creer une campagne marketing

| Champ | Description |
|-------|-------------|
| Strategie de vente | Pre-vente, Vente finale, Location, Mixte |
| Objectif pre-ventes (%) | Cible avant construction |
| Site web | URL du site projet |
| Nom du courtier | Agence immobiliere |
| Budget marketing ($) | Investissement publicitaire |
| Commission courtier (%) | Pourcentage sur ventes |
| Date de lancement | Debut de la campagne |
| Journee portes ouvertes | Evenement planifie |

**Documents marketing :**
- Brochure prete
- Plans de vente prets
- Maquette 3D disponible

### 9.5 Gestion des ventes

Affiche toutes les unites avec statut **"Vendu"** ou **"Promesse d'achat"** :

- Type d'unite et superficie
- Prix de vente
- Nom de l'acheteur
- Date de promesse d'achat
- Date de vente finale

### 9.6 Gestion des locations

Affiche toutes les unites avec statut **"Loue"** :

- Type d'unite et superficie
- Loyer mensuel
- Nom du locataire
- Date de debut du bail
- Duree du bail

**Calculs automatiques :**
- Revenus locatifs mensuels totaux
- Revenus locatifs annuels estimes

---

## 10. Livraison et Garanties

### 10.1 Presentation

L'onglet **Livraison** gere la remise des cles, les inspections finales et les garanties.

### 10.2 Sous-onglets

| Sous-onglet | Description |
|-------------|-------------|
| Liste livraisons | Toutes les livraisons |
| Nouvelle livraison | Enregistrer une remise des cles |
| Inspections | Suivi des inspections pre-livraison |
| Satisfaction client | Notes et commentaires |

### 10.3 Statistiques de livraison

| KPI | Description |
|-----|-------------|
| Total livraisons | Nombre de livraisons enregistrees |
| Cles remises | Livraisons completees |
| Satisfaction moyenne | Note sur 5 etoiles |
| Reclamations ouvertes | Problemes a resoudre |

### 10.4 Enregistrer une livraison

#### Etape 1 : Selection

- Choisir le **Projet**
- Selectionner l'**Unite** a livrer

#### Etape 2 : Beneficiaire

| Champ | Description |
|-------|-------------|
| Beneficiaire | Nom de l'acheteur/locataire |
| Type | Acheteur ou Locataire |
| Date livraison prevue | Echeance |
| Date livraison reelle | Date effective |
| Cles remises | Case a cocher |

#### Etape 3 : Inspection pre-livraison

| Champ | Description |
|-------|-------------|
| Inspection effectuee | Case a cocher |
| Date d'inspection | Date de visite |
| Liste des deficiences | Defauts constates |
| Deficiences corrigees | Reparations effectuees |

#### Etape 4 : Documents et garanties

**Documents remis :**
- Acte de vente signe
- Bail signe
- Manuel copropriete
- Plans conformes
- Certificat conformite
- Formulaire satisfaction

**Garanties :**
- Garantie legale vice cache (obligatoire)
- Garantie GCR (Garantie Construction Residentielle)
- Duree de garantie (mois)

#### Etape 5 : Satisfaction

- Note de satisfaction (1 a 5 etoiles)
- Commentaires client
- Notes internes

### 10.5 Suivi des inspections

Liste toutes les inspections pre-livraison avec :
- Unite concernee
- Date d'inspection
- Liste des deficiences
- Statut de correction

### 10.6 Satisfaction client

**Statistiques :**
- Nombre d'evaluations recues
- Satisfaction moyenne
- Pourcentage de clients tres satisfaits (4-5 etoiles)

**Detail par evaluation :**
- Note en etoiles
- Commentaires
- Reclamations ouvertes

---

## 11. Calculateurs Financiers

### 11.1 Presentation

L'onglet **Calculateurs Financiers** offre des outils de simulation pour vos projets.

### 11.2 Calculateurs disponibles

| Calculateur | Description |
|-------------|-------------|
| Mensualite | Calcul du paiement periodique |
| Tableau d'amortissement | Detail de chaque paiement |
| Interets intercalaires | Couts pendant la construction |
| Prime SCHL | Assurance pret hypothecaire |
| ROI Projet | Retour sur investissement |

### 11.3 Calculateur de mensualite

**Parametres d'entree :**
- Montant du pret ($)
- Taux d'interet annuel (%)
- Duree (annees)
- Frequence : Mensuel, Bi-hebdomadaire, Hebdomadaire

**Resultats :**
- Mensualite calculee
- Capital initial
- Interets totaux
- Cout total du credit

### 11.4 Tableau d'amortissement

Genere un tableau detaille avec pour chaque paiement :
- Numero de periode
- Montant du paiement
- Part capital
- Part interet
- Solde restant

**Graphique** : Evolution de la repartition capital/interet dans le temps.

### 11.5 Interets intercalaires

Les **interets intercalaires** sont les interets payes pendant la phase de construction, avant le debut des paiements reguliers.

**Parametres :**
- Montant du pret ($)
- Taux annuel (%)
- Duree construction (mois)

**Resultats :**
- Total des interets intercalaires
- Detail mois par mois
- Graphique d'evolution

### 11.6 Calculateur Prime SCHL

La **SCHL** (Societe canadienne d'hypotheques et de logement) exige une assurance pour les prets representant plus de 80% de la valeur du bien.

**Grille des primes SCHL 2025 :**

| Ratio LTV | Prime |
|-----------|-------|
| > 95% | 4.00% |
| 90-95% | 3.10% |
| 85-90% | 2.80% |
| 80-85% | 2.40% |
| < 80% | Aucune |

**Parametres :**
- Montant du pret ($)
- Valeur de la propriete ($)

**Resultats :**
- Ratio LTV (Loan-to-Value)
- Pourcentage de prime
- Montant de la prime
- Pret total incluant la prime

### 11.7 Calculateur ROI

Calcule le **Retour sur Investissement** de votre projet.

**Parametres :**
- Investissement total ($)
- Revenus annuels ($)
- Depenses annuelles ($)
- Duree d'analyse (annees)

**Resultats :**
- ROI en pourcentage
- Benefice net annuel
- Periode de recuperation (payback)
- Graphique de projection des benefices cumules

---

## 12. Assistant IA - Conseils Expert

### 12.1 Presentation

L'onglet **Conseils IA** donne acces a un assistant expert propulse par **Claude Opus 4.5** d'Anthropic, specialise en financement et projets immobiliers au Quebec.

### 12.2 Prerequis

L'assistant necessite une cle API Anthropic configuree dans les variables d'environnement :
- `ANTHROPIC_API_KEY` ou `CLAUDE_API_KEY`

### 12.3 Sous-onglets IA

| Sous-onglet | Description |
|-------------|-------------|
| Analyse projet | Analyse complete automatisee |
| Chat expert | Questions-reponses interactives |
| Rapport financement | Generation de rapport detaille |
| Optimisation financement | Suggestions de structure optimale |

### 12.4 Analyse projet

1. Selectionnez un projet dans la liste
2. Cliquez sur **"Lancer l'analyse complete"**
3. L'IA analyse les donnees et retourne :

**Indicateurs :**
- Score de viabilite (0-100)
- Niveau de risque (faible/moyen/eleve/critique)
- ROI estime

**Analyses detaillees :**
- Resume de la situation
- Points forts du projet
- Points d'attention
- Recommandations financement
- Recommandations gestion de projet
- Analyse de rentabilite (Cap Rate, Payback, Marge)
- Strategie suggeree
- Mise de fonds recommandee
- Conseil expert personnalise

### 12.5 Chat expert

Interface de conversation pour poser vos questions specifiques.

**Fonctionnalites :**
- Historique de conversation
- Questions contextualisees au projet selectionne
- Bouton pour effacer l'historique

**Questions suggerees :**
- Quelle est la meilleure strategie de financement ?
- Comment optimiser le calendrier de deblocages ?
- Quels sont les principaux risques financiers ?
- Comment ameliorer le ROI ?
- Quelle mise de fonds serait optimale ?

### 12.6 Rapport financement

Genere un rapport professionnel complet en format Markdown :

1. Synthese du financement
2. Structure financiere et ratios
3. Risques identifies
4. Recommandations strategiques
5. Alternatives de financement
6. Plan d'optimisation
7. Checklist avant demande de pret

**Telechargement :** Bouton pour exporter le rapport en fichier .md

### 12.7 Optimisation financement

L'IA calcule une structure de financement optimale basee sur :
- Cout total du projet
- Revenus annuels estimes
- Nombre d'unites
- Type de projet

**Resultats :**
- Mise de fonds recommandee
- Montant a financer
- Mensualite estimee
- Structure recommandee (pret construction, hypothecaire, credit)
- Parametres suggeres (taux, amortissement, LTV)
- Capacite d'emprunt selon revenus
- Recommandations strategiques
- Justification detaillee

---

## 13. Glossaire

| Terme | Definition |
|-------|------------|
| **Amortissement** | Periode totale de remboursement d'un pret |
| **Cap Rate** | Taux de capitalisation = Revenus nets / Valeur du bien |
| **CNB** | Code National du Batiment du Canada |
| **Deblocage** | Versement partiel du pret selon l'avancement |
| **GCR** | Garantie Construction Residentielle |
| **Interets intercalaires** | Interets payes pendant la construction |
| **LTV** | Loan-to-Value = Montant pret / Valeur bien |
| **Mise de fonds** | Apport personnel de l'investisseur |
| **Payback** | Periode de recuperation de l'investissement |
| **ROI** | Return on Investment = Gain / Investissement |
| **Rough-in** | Phase de plomberie/electricite avant les murs |
| **SCHL** | Societe canadienne d'hypotheques et de logement |
| **TDS** | Total Debt Service ratio |

---

## 14. FAQ

### Q1 : Comment acceder au module Immobilier ?

Connectez-vous a Constructo AI et cliquez sur "Immobilier" dans le menu lateral.

### Q2 : L'assistant IA ne fonctionne pas ?

Verifiez que la cle API Anthropic est configuree dans les variables d'environnement (`ANTHROPIC_API_KEY`).

### Q3 : Comment generer un calendrier de deblocages ?

1. Creez un financement avec l'option "Financement progressif"
2. Dans l'onglet Deblocages, cliquez sur "Generer calendrier deblocages automatique"

### Q4 : Quelle mise de fonds est recommandee ?

Pour les projets commerciaux multi-logements au Quebec, une mise de fonds de 25% a 35% est generalement exigee par les banques.

### Q5 : Comment suivre l'avancement de mon chantier ?

Utilisez l'onglet "Construction" pour creer des phases et mettre a jour leur avancement en pourcentage.

### Q6 : Comment calculer mes interets intercalaires ?

Utilisez le calculateur dedie dans l'onglet "Calculateurs Financiers" > "Interets intercalaires".

### Q7 : Comment obtenir un rapport d'analyse IA ?

Dans l'onglet "Conseils IA", selectionnez votre projet puis cliquez sur "Lancer l'analyse complete" ou "Generer le rapport complet".

---

## Support

Pour toute question ou assistance :
- Consultez la documentation API dans `/docs/API_DOCUMENTATION.md`
- Contactez le support Constructo AI

---

*Document genere automatiquement - Constructo AI Module Immobilier v2.5*
*Derniere mise a jour : Decembre 2025*
