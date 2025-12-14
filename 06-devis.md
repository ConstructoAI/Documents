# 🧾 Devis et Soumissions

## Introduction

Le module **Devis** est votre outil professionnel de création de soumissions pour le secteur de la construction au Québec. Il offre trois méthodes de création (manuel, IA, import de documents), calcule automatiquement les taxes québécoises (TPS 5% + TVQ 9.975%) et les marges commerciales, et permet l'envoi de liens clients pour approbation en ligne.

Ce module de 500+ KB est le plus complet de CONSTRUCTO AI, avec 64 tâches de production prédéfinies, 18 unités de mesure et 9 statuts de workflow.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🧾 Devis"**
2. La liste de vos devis s'affiche avec statistiques
3. Créez un nouveau devis ou consultez les existants

---

## Structure des Données

### Table PostgreSQL : `devis`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom_projet` | TEXT | Nom du projet/devis |
| `client_nom_cache` | TEXT | Nom du client (cache) |
| `client_company_id` | INTEGER | FK vers `companies.id` |
| `statut` | TEXT | Statut du devis |
| `priorite` | TEXT | Priorité |
| `prix_estime` | DECIMAL | Montant total estimé |
| `date_soumis` | DATE | Date de soumission |
| `date_prevu` | DATE | Date prévue de réalisation |
| `source` | TEXT | Source du devis (manuel, ia) |
| `po_client` | TEXT | Numéro PO du client |
| `tache` | TEXT | Tâche/type de projet |
| `lien_public` | TEXT | URL de consultation client |
| `token_public` | TEXT | Token unique pour lien public |
| `html_content` | TEXT | Contenu HTML du devis |
| `created_at` | TIMESTAMP | Date de création |

### Table : `devis_lignes`

| Champ | Type | Description |
|-------|------|-------------|
| `devis_id` | INTEGER | FK vers `devis.id` |
| `sequence_ligne` | INTEGER | Ordre d'affichage |
| `description` | TEXT | Description de l'item |
| `quantite` | DECIMAL | Quantité |
| `unite` | TEXT | Unité de mesure |
| `prix_unitaire` | DECIMAL | Prix par unité |
| `code_article` | TEXT | Code produit (optionnel) |

---

## Les 9 Statuts des Devis

| Statut | Code couleur | Description | Actions possibles |
|--------|--------------|-------------|-------------------|
| **BROUILLON** | ⚪ Gris | En cours de rédaction | Modifier, Valider |
| **VALIDE** | 🔵 Bleu | Prêt à envoyer | Envoyer, Modifier |
| **ENVOYE** | 🔵 Bleu | Transmis au client | Relancer, Attendre |
| **EN_ATTENTE** | 🔵 Bleu clair | Attente de réponse | Relancer |
| **APPROUVE** | 🟢 Vert | Accepté par le client | Créer projet |
| **REFUSE** | 🔴 Rouge | Décliné par le client | Archiver, Réviser |
| **TERMINE** | 🟢 Vert | Projet créé et terminé | Consulter |
| **ANNULE** | ⚫ Noir | Annulé | Archiver |
| **EXPIRE** | 🔴 Rouge | Date validité dépassée | Réviser |

### Workflow des Statuts

```
BROUILLON → VALIDE → ENVOYE → EN_ATTENTE
                                    ↓
                              APPROUVE → TERMINE
                                    ↓
                               REFUSE/EXPIRE
```

---

## Trois Façons de Créer un Devis

### 1. Création Manuelle

| Étape | Action |
|-------|--------|
| 1 | Cliquez sur **"➕ Nouveau devis"** |
| 2 | Sélectionnez le client (entreprise) |
| 3 | Remplissez les informations projet |
| 4 | Ajoutez les lignes une par une |
| 5 | Vérifiez les totaux |
| 6 | Enregistrez |

**Idéal pour** : Petits projets simples, devis personnalisés

### 2. Estimation par IA

| Étape | Action |
|-------|--------|
| 1 | Cliquez sur **"🤖 Estimation IA"** |
| 2 | Décrivez le projet en langage naturel |
| 3 | L'IA génère automatiquement les lignes |
| 4 | Vérifiez et ajustez si nécessaire |
| 5 | Enregistrez |

**Exemple de description** :
```
Rénovation cuisine 150 pi²,
nouvelles armoires en mélamine,
comptoir granite 25 pi lin,
électricité mise à niveau (5 prises),
plomberie nouveau lavabo et robinetterie
```

**Idéal pour** : Projets standards, estimations rapides

### 3. Import de Documents

| Étape | Action |
|-------|--------|
| 1 | Cliquez sur **"📄 Import Document"** |
| 2 | Téléchargez le PDF (plans, devis concurrent) |
| 3 | L'IA analyse et extrait les données |
| 4 | Validez les lignes générées |
| 5 | Enregistrez |

**Formats supportés** : PDF, images (PNG, JPG)

**Idéal pour** : Plans d'architecte, soumissions concurrentes

---

## 18 Unités de Mesure

| Unité | Abréviation | Usage typique |
|-------|-------------|---------------|
| unité | unité | Appareils, portes, fenêtres |
| sac | sac | Ciment, plâtre |
| mètre | m | Longueurs |
| mètre carré | m² | Surfaces métriques |
| mètre cube | m³ | Volumes métriques |
| pièce | pièce | Articles divers |
| paquet | paquet | Bardeaux, vis |
| boîte | boîte | Carrelage, clous |
| gallon | gallon | Peinture |
| lot | lot | Ensembles |
| pied | pi | Longueurs impériales |
| pied carré | pi² | Surfaces impériales |
| pied cube | pi³ | Volumes impériaux |
| verge cube | vg³ | Béton, gravier |
| pied linéaire | pi linéaire | Moulures, gouttières |
| tonne | tonne | Matériaux lourds |
| palette | palette | Gros volumes |
| heure | heure | Main d'œuvre |

---

## Calcul Automatique des Marges

### Taux par Défaut (Configurables)

| Type | Pourcentage | Calcul |
|------|-------------|--------|
| **Administration** | 3% | Total travaux × 0.03 |
| **Contingences** | 12% | Total travaux × 0.12 |
| **Profit** | 15% | Total travaux × 0.15 |

### Taxes Québécoises

| Taxe | Taux | Calcul |
|------|------|--------|
| **TPS** | 5% | Sous-total avant taxes × 0.05 |
| **TVQ** | 9.975% | Sous-total avant taxes × 0.09975 |

### Formule Complète

```
Total Travaux = Σ (Quantité × Prix Unitaire)
Administration = Total Travaux × 3%
Contingences = Total Travaux × 12%
Profit = Total Travaux × 15%
Sous-total avant taxes = Total Travaux + Admin + Contingences + Profit
TPS = Sous-total × 5%
TVQ = Sous-total × 9.975%
TOTAL TTC = Sous-total + TPS + TVQ
```

---

## 64 Tâches de Production Prédéfinies

Le module inclut 64 tâches organisées en **16 phases** de construction :

### Phase 1 : Planification
- 1.1 Définir les besoins et objectifs du projet
- 1.2 Concevoir les plans architecturaux
- 1.3 Établir un budget détaillé
- 1.4 Créer un calendrier prévisionnel
- 1.5 Obtenir les permis de construire

### Phase 2 : Préparation
- 2.1 Installer les clôtures de sécurité
- 2.2 Mettre en place la signalisation
- 2.3 Préparer les équipements de protection
- 2.4 Organiser le stockage des matériaux

### Phase 3 : Démolition
- 3.1 Déconnecter les services publics
- 3.2 Retirer les matériaux dangereux
- 3.3 Démolir la structure existante
- 3.4 Trier et évacuer les débris

### Phase 4 : Excavation
- 4.1 Marquer les limites de l'excavation
- 4.2 Creuser pour les fondations
- 4.3 Préparer le sol pour les semelles
- 4.4 Niveler le terrain

### Phase 5 : Béton
- 5.1 Préparer les coffrages
- 5.2 Installer les armatures
- 5.3 Couler les fondations
- 5.4 Réaliser les murs de fondation
- 5.5 Couler les dalles de béton

### Phase 6-16 : Charpente, Toiture, Isolation, Électricité, Plomberie, CVC, Cloisons, Revêtements sol, Menuiserie, Finitions ext, Nettoyage

---

## Interface Utilisateur

### Trois Modes d'Affichage

| Mode | Icône | Description |
|------|-------|-------------|
| **Liste Détaillée** | 📋 | Vue complète avec toutes les infos |
| **Tableau Compact** | 📊 | Vue en grille triable |
| **Cartes Compactes** | 🃏 | Vue en cartes visuelles |

### Statistiques en Haut de Page

| Métrique | Description |
|----------|-------------|
| **X devis trouvés** | Nombre total selon filtres |
| **X brouillons** | Devis en cours de rédaction |
| **X envoyés** | Devis transmis aux clients |
| **CA total** | Somme des montants |

### Filtres Disponibles

| Filtre | Options |
|--------|---------|
| **Statut** | Tous, BROUILLON, ENVOYE, etc. |
| **Client** | Liste des entreprises |
| **Période** | Date début / Date fin |

---

## Guide Pas-à-Pas

### Créer un devis manuellement

1. Cliquez sur **"➕ Nouveau devis"**
2. **Section Client** :
   - Sélectionnez l'entreprise cliente
   - Le contact principal s'affiche automatiquement
3. **Section Projet** :
   - Nom du projet
   - Description détaillée
   - Adresse du chantier
   - Date de début prévue
   - Numéro PO client (optionnel)
4. **Section Lignes** :
   - Cliquez sur **"➕ Ajouter une ligne"**
   - Description de l'item
   - Quantité
   - Unité (sélection parmi 18)
   - Prix unitaire
   - Le montant se calcule automatiquement
5. **Vérification** :
   - Total travaux
   - Marges (admin, contingences, profit)
   - Taxes (TPS, TVQ)
   - Total TTC
6. Cliquez sur **"💾 Enregistrer"**

### Créer un devis avec l'IA

1. Cliquez sur **"🤖 Estimation IA"**
2. Dans le champ de description, écrivez :
   ```
   Construction garage double 24x24 pi,
   fondation béton 6 pouces,
   murs 2x4 isolés R-20,
   toiture bardeaux asphalte,
   2 portes de garage 9x7,
   électricité 200A avec 4 prises
   ```
3. Cliquez sur **"Générer le devis"**
4. L'IA crée les lignes avec quantités et prix estimés
5. Vérifiez chaque ligne et ajustez si nécessaire
6. Enregistrez

### Envoyer un devis au client

1. Ouvrez le devis (statut VALIDE ou BROUILLON)
2. Cliquez sur **"📤 Envoyer au client"**
3. Deux options :
   - **Par email** : Saisir l'adresse, PDF en pièce jointe
   - **Lien public** : Générer une URL de consultation

4. **Lien public** :
   - URL unique sécurisée par token
   - Le client peut consulter sans se connecter
   - Boutons "Approuver" et "Refuser" en ligne
   - Mise à jour automatique du statut

### Consulter un devis (mode Voir)

1. Cliquez sur **"👁️ Voir"** dans la liste
2. Informations affichées :
   - En-tête avec numéro et statut
   - Informations client
   - Description du projet
   - Tableau des lignes avec totaux
   - Récapitulatif financier
   - Lien public (si généré)

### Dupliquer un devis

1. Cliquez sur **"📄 Dupliquer"**
2. Une copie est créée avec :
   - Nouveau numéro de devis
   - Statut BROUILLON
   - Toutes les lignes copiées
3. Modifiez selon vos besoins

### Convertir en projet (après approbation)

1. Quand un devis passe à **APPROUVE**
2. Cliquez sur **"🚀 Créer le projet"**
3. Le système crée automatiquement :
   - Un projet avec les infos du devis
   - Les tâches basées sur les lignes du devis
   - La liaison client
4. Le devis passe en statut **TERMINE**
5. Vous êtes redirigé vers le module Projets

---

## Système de Cache

### Performance Optimisée

| Donnée | TTL | Raison |
|--------|-----|--------|
| Liste devis | 2 minutes | Données dynamiques |
| Statistiques | 5 minutes | Calculs agrégés |
| Liste clients | 10 minutes | Données stables |

### Invalidation Automatique

Le cache est invalidé lors de :
- Création d'un devis
- Modification d'un devis
- Changement de statut
- Suppression

---

## Lien Public Client

### Fonctionnement

1. Un token unique est généré pour chaque devis
2. URL format : `https://app.constructoai.ca/devis/view/{token}`
3. Le client accède sans compte CONSTRUCTO AI
4. Il peut consulter tous les détails
5. Boutons "Approuver" ou "Refuser" disponibles
6. Le statut se met à jour automatiquement

### Sécurité

- Token UUID unique par devis
- Pas d'accès aux autres données
- Expiration optionnelle du lien
- Logs de consultation

---

## Numérotation des Devis

### Format Standard

```
DEVIS-{ANNÉE}-{NUMÉRO_SÉQUENTIEL}
```

Exemples :
- DEVIS-2025-001
- DEVIS-2025-042
- DEVIS-2025-156

### Révisions

Pour les devis modifiés après envoi :
```
DEVIS-2025-001-R1  (Révision 1)
DEVIS-2025-001-R2  (Révision 2)
```

---

## Astuces et Bonnes Pratiques

- **Détaillez les postes** : Plus c'est clair, moins il y a de litiges
- **Incluez les exclusions** : Précisez ce qui n'est PAS inclus
- **Fixez une date de validité** : 30 jours est standard au Québec
- **Utilisez l'IA** : Gagnez du temps sur les estimations de prix
- **Dupliquez** : Réutilisez vos devis pour projets similaires
- **Gardez les brouillons** : Ne supprimez jamais, archivez
- **Vérifiez les taxes** : TPS 5% + TVQ 9.975% sont automatiques

---

## Résolution de Problèmes

### Le devis ne s'affiche pas dans la liste

- **Cause** : Cache non rafraîchi
- **Solution** : Rafraîchissez la page (F5)

### Les taxes sont incorrectes

- **Cause** : Taux configurés manuellement
- **Solution** : Vérifiez Configuration > Paramètres commerciaux

### Impossible de supprimer un devis

- **Cause** : Statut APPROUVE ou TERMINE
- **Solution** : Les devis approuvés ne peuvent être supprimés (intégrité comptable)

### L'IA génère des prix incorrects

- **Cause** : Les prix sont des estimations
- **Solution** : Vérifiez et ajustez manuellement. Consultez vos fournisseurs pour prix réels.

### Le lien client ne fonctionne pas

- **Cause** : Token expiré ou invalide
- **Solution** : Régénérez un nouveau lien public

---

## Questions Fréquentes (FAQ)

**Q: Comment modifier un devis déjà envoyé ?**
R: Créez une nouvelle révision. L'original est conservé. Le nouveau devis portera un numéro de révision (ex: DEVIS-2025-001-R1).

**Q: Les prix des matériaux sont-ils mis à jour automatiquement ?**
R: Non, les prix sont saisis manuellement ou estimés par l'IA. Consultez vos fournisseurs régulièrement.

**Q: Comment ajouter ma signature au devis ?**
R: Dans Configuration > Entreprise, téléchargez votre signature numérique. Elle apparaîtra automatiquement.

**Q: Puis-je personnaliser le modèle de devis ?**
R: Le modèle est standardisé. Vous pouvez personnaliser logo, couleurs et mentions légales dans Configuration.

**Q: Comment gérer les devis en USD ?**
R: Actuellement, seul le CAD est supporté. Les taxes sont calculées selon les taux québécois.

**Q: Le client peut-il modifier le devis via le lien public ?**
R: Non, le lien est en lecture seule. Le client peut uniquement approuver ou refuser.

**Q: Comment suivre si le client a consulté le devis ?**
R: Les consultations sont enregistrées. Vous verrez "Consulté le [date]" dans les détails.

---

## Données Techniques

### Requête SQL Liste Devis

```sql
SELECT id, nom_projet, client_nom_cache, statut, priorite,
       date_soumis, prix_estime, source, created_at,
       'DEVIS-' || EXTRACT(YEAR FROM created_at)::int || '-' ||
       LPAD(id::text, 3, '0') as numero_devis
FROM devis
ORDER BY id DESC
LIMIT 300
```

### Calcul Récapitulatif

```python
total_travaux = sum(ligne.quantite * ligne.prix_unitaire)
admin = total_travaux * 0.03
contingences = total_travaux * 0.12
profit = total_travaux * 0.15
sous_total = total_travaux + admin + contingences + profit
tps = sous_total * 0.05
tvq = sous_total * 0.09975
total_ttc = sous_total + tps + tvq
```

---

## Voir Aussi

- [🏢 Entreprises](03-entreprises.md) - Sélection du client
- [🤝 Ventes](05-ventes.md) - Pipeline commercial
- [📋 Projets](07-projets.md) - Conversion en projet
- [💰 Comptabilité](19-comptabilite.md) - Facturation
- [🙋‍♂️ Assistant IA](24-assistant-ia.md) - Estimation intelligente
- [⚙️ Configuration](27-configuration.md) - Paramètres commerciaux
