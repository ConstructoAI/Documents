# Manuel d'Utilisation - Module Charge Tributaire

## Calcul Structural pour le Québec

**Version:** 1.0
**Conforme:** CNBC 2020, CSA O86-19, CSA S16-19
**Propulsé par:** Claude Opus 4.5

---

## Table des Matières

1. [Introduction](#1-introduction)
2. [Démarrage Rapide](#2-démarrage-rapide)
3. [Calculateur Principal](#3-calculateur-principal)
4. [Dimensionnement Automatique](#4-dimensionnement-automatique)
5. [Solives et Chevrons](#5-solives-et-chevrons)
6. [Poteaux Muraux](#6-poteaux-muraux)
7. [Glulam et HSS](#7-glulam-et-hss)
8. [Colonnes](#8-colonnes)
9. [Historique](#9-historique)
10. [Références Techniques](#10-références-techniques)
11. [Assistant IA](#11-assistant-ia)
12. [Formules et Calculs](#12-formules-et-calculs)
13. [Glossaire](#13-glossaire)
14. [FAQ](#14-faq)

---

## 1. Introduction

### 1.1 Présentation

Le module **Charge Tributaire** est un outil de calcul structural professionnel conçu spécifiquement pour les ingénieurs, technologues et entrepreneurs en construction au Québec. Il permet de vérifier et dimensionner les éléments structuraux selon les normes canadiennes en vigueur.

### 1.2 Fonctionnalités Principales

| Fonctionnalité | Description |
|----------------|-------------|
| **Vérification de membres** | Vérifie si un membre structural est adéquat pour les charges données |
| **Dimensionnement auto** | Trouve automatiquement le membre optimal |
| **Solives/Chevrons** | Calcul spécialisé pour éléments répétitifs avec espacement |
| **Poteaux muraux** | Vérification des studs porteurs |
| **Glulam/HSS** | Support des matériaux d'ingénierie |
| **Colonnes** | Calcul de flambement selon CSA O86 |
| **Assistant IA** | Expert structural propulsé par Claude Opus 4.5 |
| **Rapports PDF** | Génération de rapports professionnels |

### 1.3 Matériaux Supportés

- **Bois dimensionnel:** 2x4 à 2x12 (simple, double, triple)
- **LVL:** 1¾" à 7" × 9½" à 24"
- **Acier W:** W8 à W18
- **HSS:** 4x4 à 10x6
- **Glulam:** 80mm à 265mm × 190mm à 608mm

### 1.4 Normes Appliquées

- **CNBC 2020** - Code National du Bâtiment du Canada
- **CSA O86-19** - Règles de calcul des charpentes en bois
- **CSA S16-19** - Règles de calcul des charpentes en acier

---

## 2. Démarrage Rapide

### 2.1 Accès au Module

1. Ouvrez l'application Constructo AI
2. Naviguez vers **🏗️ Charge Tributaire** dans le menu
3. Le module s'ouvre avec 9 onglets de fonctionnalités

### 2.2 Premier Calcul en 5 Étapes

```
1. Sélectionnez le type de matériau (ex: Bois dimensionnel)
2. Choisissez la dimension (ex: 2x10)
3. Entrez la portée en mm (ex: 3000 mm = 9'-10")
4. Définissez les charges (morte et vive)
5. Cliquez sur "Vérifier le membre"
```

### 2.3 Comprendre les Résultats

| Indicateur | Signification |
|------------|---------------|
| ✅ ADÉQUAT | Le membre supporte les charges (ratio ≤ 100%) |
| ❌ INADÉQUAT | Le membre est insuffisant (ratio > 100%) |
| **Ratio critique** | Le plus élevé des 3 ratios (moment, cisaillement, flèche) |

---

## 3. Calculateur Principal

### 3.1 Onglet Calculateur

L'onglet **📐 Calculateur** est le coeur du module. Il permet de vérifier si un membre structural spécifique est adéquat pour les charges appliquées.

### 3.2 Paramètres d'Entrée

#### Géométrie

| Paramètre | Description | Exemple |
|-----------|-------------|---------|
| Type d'élément | Linteau, Poutre, ou Solive | Linteau |
| Type de matériau | Bois, LVL, ou Acier | Bois dimensionnel |
| Dimension | Taille du membre | 2x10 |
| Grade | Classe de résistance | SPF No.2 |
| Portée | Distance entre appuis (mm) | 2400 mm (7'-10½") |

#### Charges Tributaires

**Largeur tributaire (m):** Distance perpendiculaire supportée par le membre

**Charges mortes (D) - kPa:**
- Plancher (bois): 0.5 kPa typique
- Plafond suspendu: 0.25 kPa
- Cloisons légères: 0.5 kPa
- Autres: Variable

**Charges vives (L) - kPa:**
- Résidentiel habitation: 1.9 kPa (39.7 PSF)
- Résidentiel balcon: 2.4 kPa (50.1 PSF)
- Commercial bureau: 2.4 kPa
- Commercial corridor: 4.8 kPa

### 3.3 Lecture des Résultats

Le système affiche:

1. **Statut global:** ✅ ADÉQUAT ou ❌ INADÉQUAT
2. **Messages détaillés:** Explication des vérifications
3. **Ratios de vérification:**
   - Ratio moment (demande/capacité)
   - Ratio cisaillement
   - Ratio flèche
   - Ratio critique (le plus élevé)
4. **Graphique:** Visualisation des ratios vs limite 100%
5. **Détails techniques:** Sollicitations et propriétés de section

### 3.4 Exemple Complet

**Projet:** Linteau de fenêtre résidentiel

```
Entrées:
- Matériau: Bois dimensionnel
- Dimension: 2x10 double
- Grade: SPF No.2
- Portée: 2400 mm (7'-10½")
- Largeur tributaire: 2.4 m
- Charge morte: 0.75 kPa (plancher + plafond)
- Charge vive: 1.9 kPa (résidentiel)

Résultats:
- Ratio moment: 65%
- Ratio cisaillement: 12%
- Ratio flèche: 82%
- Ratio critique: 82%
- Verdict: ✅ ADÉQUAT
```

---

## 4. Dimensionnement Automatique

### 4.1 Principe

L'onglet **📊 Dimensionnement auto** analyse tous les membres disponibles d'un type de matériau et identifie ceux qui sont adéquats pour vos charges.

### 4.2 Paramètres

| Paramètre | Description |
|-----------|-------------|
| Type de matériau | Bois, LVL, ou Acier |
| Portée (mm) | Distance entre appuis |
| Type d'utilisation | Plancher, Toit, ou Linteau |
| Charge morte (kN/m) | Charge permanente linéaire |
| Charge vive (kN/m) | Charge d'exploitation linéaire |

### 4.3 Résultats

Le système retourne un tableau trié des membres valides avec:
- Membre et grade
- Ratio critique
- Ratios individuels (moment, cisaillement, flèche)
- Flèche calculée

**Recommandation:** Le premier membre de la liste est le plus économique qui satisfait toutes les vérifications.

---

## 5. Solives et Chevrons

### 5.1 Solives de Plancher

L'onglet **🏠 Solives/Chevrons** permet de calculer les éléments répétitifs avec espacement standard.

#### Paramètres Solives

| Paramètre | Options |
|-----------|---------|
| Dimension | 2x6, 2x8, 2x10, 2x12 |
| Grade | SPF No.1, SPF No.2, D.Fir-L No.2 |
| Portée | 1000 à 8000 mm |
| Espacement | 12", 16", ou 24" (305, 406, 610 mm) |
| Charge morte | kPa (surfacique) |
| Charge vive | kPa (1.9 kPa résidentiel typique) |

#### Calcul Automatique

Le module convertit automatiquement les charges surfaciques (kPa) en charges linéaires (kN/m) selon l'espacement:

```
Charge linéaire = Charge surfacique × (Espacement / 1000)
```

### 5.2 Chevrons de Toit

#### Paramètres Chevrons

| Paramètre | Description |
|-----------|-------------|
| Portée horizontale | Projection horizontale du chevron |
| Pente | 3/12 à 12/12 |
| Région | Montréal, Québec, Saguenay, etc. |
| Charge morte toit | kPa (0.5 kPa typique) |

#### Facteur de Réduction Neige (Cs)

Pour les pentes supérieures à 30°:
```
Cs = 1 - (pente° - 30°) / 40°
```

#### Charges de Neige par Région

| Région | Neige au sol (kPa) |
|--------|-------------------|
| Montréal | 2.4 |
| Québec | 3.1 |
| Trois-Rivières | 2.7 |
| Sherbrooke | 3.0 |
| Saguenay | 3.6 |
| Sept-Îles | 3.5 |

---

## 6. Poteaux Muraux

### 6.1 Description

L'onglet **🧱 Poteaux muraux** vérifie les poteaux (studs) porteurs selon CSA O86, incluant le calcul de flambement.

### 6.2 Paramètres

| Paramètre | Description | Valeurs Typiques |
|-----------|-------------|-----------------|
| Dimension | Taille du poteau | 2x4, 2x6, double |
| Hauteur mur | Hauteur libre | 2440 mm (8') |
| Espacement | Entre poteaux | 12", 16", 24" |
| Nb étages | Étages supportés | 0 à 4 |
| Charge toit | D+S combinée | 3.0 kPa typique |
| Charge plancher | D+L par étage | 2.4 kPa typique |
| Largeur tributaire | Demi-portée chaque côté | 3.0 m |

### 6.3 Calcul de Flambement

Le module calcule:
- **Aire tributaire:** Espacement × Largeur trib.
- **Charge axiale totale:** Toit + (Planchers × Nb étages)
- **Élancement (λ):** Hauteur effective / Rayon de giration min
- **Facteur Kc:** Réduction pour flambement
- **Résistance Pr:** Capacité pondérée

---

## 7. Glulam et HSS

### 7.1 Bois Lamellé-Collé (Glulam)

#### Dimensions Disponibles

| Largeur (mm) | Hauteurs (mm) |
|--------------|---------------|
| 130 | 228 à 456 |
| 175 | 228 à 532 |
| 215 | 266 à 570 |
| 265 | 342 à 608 |

#### Grades

| Grade | fb (MPa) | E (MPa) | Application |
|-------|----------|---------|-------------|
| 20f-E SPF | 20.4 | 10300 | Standard |
| 24f-E SPF | 24.5 | 12400 | Haute résistance |
| 20f-E D.Fir-L | 20.4 | 12400 | Douglas Fir |
| 24f-E D.Fir-L | 24.5 | 14900 | Douglas haute perf. |

### 7.2 Profilés HSS

#### Types Disponibles

**Carrés:**
- HSS 4x4 à HSS 8x8
- Épaisseurs: 1/4", 3/8", 1/2"

**Rectangulaires:**
- HSS 6x4, 8x4, 8x6, 10x6
- Épaisseurs: 1/4", 3/8"

#### Propriétés Acier

| Nuance | Fy (MPa) | Fu (MPa) | Application |
|--------|----------|----------|-------------|
| A992/A572 Gr.50 | 345 | 450 | Standard |
| A36 | 250 | 400 | Économique |

---

## 8. Colonnes

### 8.1 Vérification de Colonnes

L'onglet **🏛️ Colonnes** calcule les colonnes en compression axiale avec effet de flambement.

### 8.2 Conditions d'Appui (Facteur K)

| Condition | K | Description |
|-----------|---|-------------|
| Encastré-Encastré | 0.65 | Les deux extrémités fixes |
| Encastré-Rotule | 0.80 | Une extrémité fixe, une libre en rotation |
| Rotule-Rotule | 1.00 | Deux extrémités libres en rotation |
| Encastré-Libre | 2.10 | Console (une extrémité libre) |

### 8.3 Formules Clés

**Longueur effective:**
```
Le = K × L
```

**Élancement:**
```
λ = Le / r_min
où r_min = dimension_min / √12
```

**Contrainte d'Euler:**
```
Fe = π²E₀₅ / λ²
```

---

## 9. Historique

### 9.1 Fonctionnalité

L'onglet **📋 Historique** affiche les calculs précédents sauvegardés dans la base de données.

### 9.2 Informations Affichées

- Date et heure du calcul
- Type d'élément
- Membre vérifié
- Portée
- Ratio critique
- Statut (✅/❌)

---

## 10. Références Techniques

### 10.1 Tables Disponibles

L'onglet **📖 Références** contient des tables de référence pour:

| Onglet | Contenu |
|--------|---------|
| Bois | Dimensions réelles, propriétés mécaniques |
| LVL | Dimensions, propriétés par grade |
| Acier | Profilés W avec propriétés |
| HSS | Profilés tubulaires |
| Glulam | Dimensions et grades |
| Charges | Charges mortes, vives, neige |
| Facteurs | Facteurs CNBC, limites flèche |

### 10.2 Dimensions Réelles du Bois

| Nominal | Largeur b (mm) | Hauteur d (mm) |
|---------|----------------|----------------|
| 2x4 | 38 | 89 |
| 2x6 | 38 | 140 |
| 2x8 | 38 | 184 |
| 2x10 | 38 | 235 |
| 2x12 | 38 | 286 |

### 10.3 Facteurs de Charge CNBC 2020

| Type | Facteur |
|------|---------|
| Morte (D) | 1.25 |
| Vive (L) | 1.50 |
| Neige (S) | 1.50 |
| Vent (W) | 1.40 |

### 10.4 Limites de Flèche

| Application | Limite |
|-------------|--------|
| Plancher (charge vive) | L/360 |
| Plancher (totale) | L/240 |
| Toit (vive) | L/240 |
| Toit (totale) | L/180 |
| Linteau fenêtre | L/360 |
| Vitrage | L/175 |

---

## 11. Assistant IA

### 11.1 Présentation

L'onglet **🧠 Assistant IA** intègre Claude Opus 4.5, un modèle d'intelligence artificielle expert en calcul structural au Québec.

### 11.2 Configuration

L'assistant nécessite une clé API Anthropic:

```bash
# Variable d'environnement requise
export ANTHROPIC_API_KEY="votre-clé-api"
# ou
export CLAUDE_API_KEY="votre-clé-api"
```

Obtenez une clé sur: https://console.anthropic.com/

### 11.3 Fonctionnalités

#### 💬 Chat Expert

Posez vos questions en langage naturel:
- "Quelle portée max pour un 2x10?"
- "Quand utiliser du LVL?"
- "Comment calculer les charges de neige?"

**Questions suggérées disponibles en un clic.**

#### 🔍 Analyser Calcul

Après avoir effectué un calcul, l'IA fournit:
- Score de sécurité (0-100)
- Explication simple pour non-ingénieurs
- Points positifs et d'attention
- Recommandations
- Alternatives suggérées

#### 🎯 Recommandations

Décrivez votre application et obtenez:
- Recommandation principale avec justification
- Alternatives
- Éléments à éviter
- Conseils d'installation
- Points à faire vérifier par un ingénieur

#### 📚 Expliquer

Obtenez des explications vulgarisées sur:
- Moment de flexion
- Flèche (déformation)
- Cisaillement
- Charge tributaire
- Facteurs de charge CNBC
- Limite de flèche L/360
- Et plus...

### 11.4 Avertissement

> ⚠️ **Important:** Les conseils de l'assistant IA sont éducatifs. Pour tout projet réel, faites valider vos calculs par un ingénieur membre de l'Ordre des ingénieurs du Québec (OIQ).

---

## 12. Formules et Calculs

### 12.1 Poutre Simple - Charge Uniforme

```
Moment maximum:      M = wL²/8
Cisaillement max:    V = wL/2
Flèche maximum:      δ = 5wL⁴/(384EI)
```

Où:
- w = charge linéaire (kN/m ou N/mm)
- L = portée (m ou mm)
- E = module d'élasticité (MPa)
- I = moment d'inertie (mm⁴)

### 12.2 Propriétés de Section (Rectangle)

```
Aire:                A = b × d
Moment d'inertie:    I = bd³/12
Module de section:   S = bd²/6
```

### 12.3 Résistances Pondérées

**Flexion (bois):**
```
Mr = φ × fb × S
où φ = 0.9 (bois/LVL)
```

**Cisaillement (bois):**
```
Vr = φ × fv × (2/3) × A
```

**Flexion (acier):**
```
Mr = φ × Zx × Fy
où φ = 0.9
```

### 12.4 Vérification

```
Ratio = Demande / Capacité
Membre adéquat si: Ratio ≤ 1.0 (100%)
```

### 12.5 Combinaisons de Charges

**Principale (CNBC):**
```
1.25D + 1.5L + 0.5S
```

**Avec neige dominante:**
```
1.25D + 0.5L + 1.5S
```

---

## 13. Glossaire

| Terme | Définition |
|-------|------------|
| **Charge morte (D)** | Poids permanent de la structure et des éléments fixes |
| **Charge vive (L)** | Charges d'exploitation variables (personnes, mobilier) |
| **Charge de neige (S)** | Poids de la neige accumulée sur le toit |
| **Charge tributaire** | Aire de plancher dont les charges sont supportées par un élément |
| **Portée** | Distance entre les appuis d'un élément |
| **Flèche** | Déformation verticale maximale sous charge |
| **Moment** | Force de rotation interne causée par les charges |
| **Cisaillement** | Force de glissement interne entre les fibres |
| **Ratio demande/capacité** | Pourcentage d'utilisation de la résistance |
| **Facteur de résistance (φ)** | Réduction de sécurité appliquée à la résistance |
| **Facteur de charge** | Majoration de sécurité appliquée aux charges |
| **Flambement** | Instabilité latérale d'une colonne sous compression |
| **Élancement (λ)** | Ratio longueur/rayon de giration (sensibilité au flambement) |
| **LVL** | Laminated Veneer Lumber - Bois de placage lamellé |
| **Glulam** | Bois lamellé-collé |
| **HSS** | Hollow Structural Section - Profilé tubulaire acier |
| **SPF** | Spruce-Pine-Fir - Essence épinette-pin-sapin |
| **OIQ** | Ordre des ingénieurs du Québec |

---

## 14. FAQ

### Q1: Comment convertir les unités?

Le module utilise le système SI (mm, kPa, kN) mais affiche également les unités impériales:

| Métrique | Impérial | Conversion |
|----------|----------|------------|
| 1 mm | 0.0394" | ÷ 25.4 |
| 1 m | 3.28' | × 3.281 |
| 1 kPa | 20.9 PSF | × 20.8854 |
| 1 kN/m | 68.5 lb/ft | × 68.5218 |

### Q2: Pourquoi mon ratio est supérieur à 100%?

Un ratio > 100% signifie que la demande dépasse la capacité. Solutions:
1. Choisir une dimension plus grande
2. Utiliser un matériau plus résistant (LVL, acier)
3. Réduire la portée
4. Ajouter un support intermédiaire

### Q3: Quelle est la différence entre ratio moment et ratio flèche?

- **Ratio moment:** Vérification de la résistance (sécurité structurale)
- **Ratio flèche:** Vérification de la déformation (confort, apparence, finitions)

Un membre peut être adéquat en résistance mais échouer en flèche.

### Q4: Quand utiliser le LVL plutôt que le bois dimensionnel?

Utilisez le LVL quand:
- La portée dépasse les capacités du bois (> 4m typiquement)
- L'espace vertical est limité
- Des charges plus élevées sont présentes
- Une uniformité de qualité est requise

### Q5: Comment obtenir un rapport PDF?

1. Effectuez un calcul dans l'onglet Calculateur
2. Descendez jusqu'à la section "Génération PDF"
3. Entrez un nom de projet (optionnel)
4. Cliquez sur "Générer PDF"
5. Téléchargez le rapport

### Q6: Les calculs sont-ils conformes aux normes?

Oui, les calculs suivent:
- **CNBC 2020** pour les charges et combinaisons
- **CSA O86-19** pour le bois et LVL
- **CSA S16-19** pour l'acier

> ⚠️ Ces calculs sont fournis à titre indicatif. Pour tout projet de construction, faites valider par un ingénieur professionnel.

### Q7: Comment fonctionne l'Assistant IA?

L'assistant utilise Claude Opus 4.5 d'Anthropic avec un prompt système spécialisé en calcul structural québécois. Il peut:
- Répondre aux questions techniques
- Analyser vos calculs
- Recommander des membres
- Expliquer les concepts

---

## Annexe: Conversions Rapides

### Portées Courantes

| Pieds-Pouces | Millimètres |
|--------------|-------------|
| 6'-0" | 1829 mm |
| 8'-0" | 2438 mm |
| 10'-0" | 3048 mm |
| 12'-0" | 3658 mm |
| 14'-0" | 4267 mm |
| 16'-0" | 4877 mm |

### Charges Résidentielles Typiques

| Élément | Morte (kPa) | Vive (kPa) |
|---------|-------------|------------|
| Plancher bois | 0.5 | 1.9 |
| Toit léger | 0.5 | Neige région |
| Balcon | 0.5 | 2.4 |

---

## Support

**Développé par:** Constructo AI
**Assistance IA:** Claude Opus 4.5
**Contact:** support@constructo-ai.com

**Dernière mise à jour:** Décembre 2025

---

*Ce manuel est fourni à titre informatif. Pour tout projet de construction, consultez un ingénieur professionnel membre de l'Ordre des ingénieurs du Québec (OIQ).*
