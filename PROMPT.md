# Structure et Organisation des Prompts

La structure des prompts s'organise de manière hiérarchique, comparable à l'organisation des calques dans AutoCAD ou à la superposition de couches de peinture sur une toile. Cette approche structurée permet une interaction optimale avec l'intelligence artificielle.

## Principes fondamentaux

Pour tirer le meilleur parti de la plateforme, il est essentiel de :

- Maintenir une communication claire et structurée
- Organiser les informations de façon logique et progressive
- Utiliser les dialogues de manière pertinente et efficace

La plateforme offre une grande flexibilité, permettant une personnalisation adaptée à vos besoins spécifiques, notamment pour la lecture et l'interprétation de plans. Cette adaptabilité nécessite toutefois une approche méthodique dans la formulation des requêtes.

---

## 1. Prompts Généraux

### 1.1 — Estimation budgétaire de base

```
Tu es un estimateur en construction spécialisé dans l'estimation budgétaire résidentielle au Québec.

**Projet** : Construction d'une maison à deux étages

**Spécifications techniques** :
- Superficie par étage : 707 pi²
- Superficie totale : 1 414 pi²
- Type de toiture : plate
- Catégorie de prix : ÉCONOMIQUE

**Contraintes** :
- Les frais sont déjà inclus dans le prix au pied carré
- Utilise les données de la base de données fournie

**Instructions** :
1. Produis une estimation budgétaire détaillée
2. Vérifie chaque calcul étape par étape
3. Assure-toi que les résultats sont cohérents avec les règles de la base de données
4. Signale toute incohérence détectée
```

### 1.2 — Vérification des calculs

```
Effectue une vérification complète des calculs précédents.

**Méthodologie** :
1. Reprends chaque poste budgétaire individuellement
2. Valide les formules utilisées
3. Vérifie les totaux et sous-totaux
4. Compare avec les ratios standards de l'industrie
5. Liste les écarts ou anomalies détectés

**Format de sortie** : Tableau de vérification avec colonnes [Poste | Calcul initial | Vérification | Statut ✓/✗]
```

### 1.3 — Estimation détaillée avec échéancier

```
Produis une estimation détaillée avec ventilation main-d'œuvre/matériaux.

**Contenu requis** :
- Ratio main-d'œuvre / matériaux par poste
- Quantités détaillées pour chaque item
- Échéancier de réalisation par phase

**Format de présentation** :
| Poste | Main-d'œuvre ($) | Matériaux ($) | Quantité | Durée |
|-------|------------------|---------------|----------|-------|

**Sommaire** : Inclure le prix au pi² par étage

**Validation** : Vérifie étape par étape la cohérence des calculs avant de présenter les résultats.
```

### 1.4 — Devis complet

```
Produis un devis professionnel complet pour ce projet.

**Structure du devis** :
1. En-tête avec informations du projet
2. Tableau détaillé par section de travaux
3. Liste des matériaux avec quantités et prix unitaires
4. Sous-totaux par catégorie
5. Total général

**Colonnes requises** : Description | Quantité | Unité | Prix unitaire | Total

**Instructions** :
- Utilise un format professionnel avec titres en gras
- Vérifie chaque calcul avant de l'inclure
- Assure la cohérence entre les quantités et les totaux
```

### 1.5 — Diagramme de Gantt

```
Crée un diagramme de Gantt pour la planification du projet.

**Organisation** : Par corps de métier

**Inclure** :
- Séquence logique des travaux
- Durée estimée par tâche
- Dépendances entre les tâches
- Chemin critique identifié

**Format** : Représentation visuelle en mode texte (ASCII) ou tableau avec semaines/jours
```

---

## 2. Prompts Structure

### 2.1 — Estimation pour maison unifamiliale

```
Tu es un estimateur spécialisé en structures préfabriquées.

**Projet** : Maison unifamiliale

**Caractéristiques techniques** :
| Élément | Spécification |
|---------|---------------|
| Dimensions | 30 pi × 20 pi |
| Étages | 1 |
| Plancher | Poutrelles avec cage d'escalier |
| Murs | Préfabriqués, 9 pi 2 po, isolés |
| Toiture | Fermes régulières, pente 6/12, corniche 12 po |
| Poutre principale | LVL 3 plis, 10 pi × 9 po |
| Colonnes | 2 × (5 po × 5 po) |
| Livraison | Farnham, 54 km de l'usine |

**Demande** : Estimation budgétaire complète

**Format** : Tableau professionnel en colonnes avec titres en gras, incluant :
- Coût des matériaux
- Coût de fabrication
- Coût de livraison
- Total par section
```

### 2.2 — Estimation pour garage commercial

```
Tu es un estimateur spécialisé en structures préfabriquées.

**Projet** : Garage commercial

**Caractéristiques techniques** :
| Élément | Spécification |
|---------|---------------|
| Dimensions | 100 pi × 40 pi |
| Étages | 1 |
| Murs | Préfabriqués, 16 pi 2 po, non isolés |
| Toiture | Fermes régulières, 2 versants, pente 4/12, corniche 12 po |
| Poutres | 2 × LVL 2 plis, 13 pi × 9½ po |
| Livraison | Farnham, 54 km de l'usine |

**Demande** : Estimation budgétaire complète

**Format** : Tableau professionnel en colonnes avec titres en gras, incluant :
- Coût des matériaux
- Coût de fabrication
- Coût de livraison
- Total par section
```

---

## 3. Analyse des Plans

### 3.1 — Révision complète des plans

```
Effectue une révision exhaustive des plans fournis.

**Objectif** : Identifier tout élément manquant ou incohérent

**Checklist de vérification** :
- [ ] Dimensions et superficies
- [ ] Éléments structuraux (murs, poutres, colonnes)
- [ ] Ouvertures (portes, fenêtres)
- [ ] Systèmes (électrique, plomberie, CVAC)
- [ ] Finitions intérieures et extérieures
- [ ] Éléments spéciaux ou personnalisés

**Format de sortie** : Liste des éléments vérifiés avec statut et observations
```

### 3.2 — Résumé des travaux

```
Produis un résumé détaillé des travaux à effectuer.

**Structure** :
1. **Vue d'ensemble** : Description générale du projet
2. **Travaux par phase** :
   - Préparation du terrain
   - Fondations
   - Structure
   - Enveloppe
   - Mécanique/Électricité
   - Finitions
3. **Points d'attention** : Éléments critiques ou complexes
4. **Exclusions** : Ce qui n'est pas inclus

**Intégrer** : Les dernières modifications et mises à jour discutées
```

### 3.3 — Estimation économique complète

```
Produis une estimation complète en catégorie prix économique.

**Contenu** :
1. Ventilation main-d'œuvre / matériaux par poste
2. Quantités détaillées
3. Échéancier de réalisation

**Présentation** :
- Format tableau professionnel
- Titres en gras
- Sommaire incluant le prix au pi² par étage

**Rappel** : Intégrer toutes les modifications récentes du projet
```

### 3.4 — Vérification de cohérence

```
Effectue une vérification approfondie de la cohérence du projet.

**Analyse des matériaux** :
- Les quantités correspondent-elles aux dimensions des plans?
- Les spécifications sont-elles appropriées pour le type de construction?
- Y a-t-il des doublons ou des omissions?

**Optimisation des échéanciers** :
- La séquence des travaux est-elle logique?
- Peut-on paralléliser certaines tâches?
- Les durées sont-elles réalistes?

**Format de sortie** :
1. Tableau des incohérences détectées (le cas échéant)
2. Suggestions d'optimisation
3. Recommandations
```

### 3.5 — Devis détaillé

```
Produis un devis professionnel détaillé.

**Structure** :
1. Informations du projet
2. Tableau des travaux par étape
3. Liste complète des matériaux

**Colonnes** : Étape | Description | Matériaux | Quantité | Unité | Prix unitaire | Total

**Exigences** :
- Présentation professionnelle
- Titres en gras
- Totaux par section et total général
```

### 3.6 — Validation finale

```
Effectue une validation finale complète du projet.

**Vérifications** :
1. **Estimation** : Les coûts sont-ils réalistes et complets?
2. **Devis** : Tous les postes sont-ils inclus?
3. **Cohérence** : L'estimation correspond-elle aux plans?
4. **Conformité** : Les spécifications respectent-elles les normes?

**Questions à valider** :
- Le total est-il cohérent avec des projets similaires?
- Y a-t-il des postes sous-estimés ou surestimés?
- Les quantités de matériaux sont-elles suffisantes?

**Format de sortie** : Rapport de validation avec statut global et recommandations
```

---

## Conseils d'utilisation

Pour une efficacité maximale avec Claude :

1. **Fournir le contexte** : Toujours préciser le rôle et le domaine d'expertise attendu
2. **Structurer les données** : Utiliser des tableaux et listes pour les spécifications
3. **Expliciter le format** : Décrire précisément la présentation souhaitée
4. **Demander la vérification** : Inclure des instructions de validation dans chaque prompt
5. **Itérer** : Utiliser les prompts de suivi (1.2, 3.4, 3.6) pour affiner les résultats
