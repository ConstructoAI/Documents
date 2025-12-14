# 👥 Employés

## Introduction

Le module **Employés** centralise la gestion de vos ressources humaines spécifiquement adaptée au contexte de la construction au Québec. Il permet de gérer les dossiers des employés, leurs compétences certifiées CCQ, contrats, assignations aux projets et informations administratives complètes.

Ce module est intégré avec le TimeTracker pour le pointage, la Production pour les assignations aux BT, et le module RBQ/CCQ pour la conformité réglementaire.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"👥 Employés"**
2. La liste de vos employés s'affiche avec filtres
3. Créez ou consultez les fiches employé
4. Gérez les compétences et certifications

---

## Structure des Données

### Table PostgreSQL : `employees`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `prenom` | TEXT | Prénom de l'employé |
| `nom` | TEXT | Nom de famille |
| `email` | TEXT | Courriel professionnel |
| `telephone` | TEXT | Numéro de téléphone |
| `poste` | TEXT | Titre du poste |
| `departement` | TEXT | Département de rattachement |
| `statut` | TEXT | Statut de l'employé |
| `type_contrat` | TEXT | Type de contrat |
| `date_embauche` | DATE | Date d'embauche |
| `salaire` | DECIMAL | Salaire ou taux horaire |
| `manager_id` | INTEGER | FK vers le supérieur |
| `charge_travail` | INTEGER | Charge de travail (%) |
| `notes` | TEXT | Notes diverses |
| `photo_url` | TEXT | URL de la photo |
| `pin_code` | TEXT | Code PIN pour TimeTracker |
| `created_at` | TIMESTAMP | Date de création |
| `updated_at` | TIMESTAMP | Dernière modification |

### Table PostgreSQL : `employee_competences`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `employee_id` | INTEGER | FK vers `employees.id` |
| `nom_competence` | TEXT | Nom de la compétence |
| `niveau` | TEXT | DÉBUTANT/INTERMÉDIAIRE/AVANCÉ/EXPERT |
| `certifie` | BOOLEAN | Certification officielle |
| `date_obtention` | DATE | Date d'obtention |
| `date_expiration` | DATE | Date d'expiration |

### Table PostgreSQL : `project_assignments`

| Champ | Type | Description |
|-------|------|-------------|
| `project_id` | INTEGER | FK vers `projects.id` |
| `employee_id` | INTEGER | FK vers `employees.id` |
| `role_projet` | TEXT | Rôle dans le projet |

---

## Les 11 Départements Construction Québec

| # | Département | Code | Description |
|---|-------------|------|-------------|
| 1 | **CHANTIER** | CHANTIER | Équipes terrain, gros oeuvre |
| 2 | **STRUCTURE_BÉTON** | STRUCTURE_BÉTON | Coffrage, béton, fondations |
| 3 | **CHARPENTE_BOIS** | CHARPENTE_BOIS | Ossature bois, toiture |
| 4 | **FINITION** | FINITION | Plâtrerie, peinture, carrelage |
| 5 | **MÉCANIQUE_BÂTIMENT** | MÉCANIQUE_BÂTIMENT | Plomberie, chauffage, ventilation |
| 6 | **ÉLECTRICITÉ** | ÉLECTRICITÉ | Installation électrique |
| 7 | **INGÉNIERIE** | INGÉNIERIE | Conception, plans, calculs |
| 8 | **QUALITÉ_CONFORMITÉ** | QUALITÉ_CONFORMITÉ | Inspection, conformité RBQ/CCQ |
| 9 | **ADMINISTRATION** | ADMINISTRATION | Bureau, comptabilité, RH |
| 10 | **COMMERCIAL** | COMMERCIAL | Ventes, soumissions, développement |
| 11 | **DIRECTION** | DIRECTION | Supervision, contremaîtrise |

---

## Statuts des Employés

| Statut | Code | Description |
|--------|------|-------------|
| **ACTIF** | `ACTIF` | Employé en service |
| **CONGÉ** | `CONGÉ` | Congé personnel/parental |
| **FORMATION** | `FORMATION` | En formation/perfectionnement |
| **ARRÊT_TRAVAIL** | `ARRÊT_TRAVAIL` | Maladie/accident de travail |
| **INACTIF** | `INACTIF` | Temporairement inactif |

---

## Types de Contrat Québécois

| Type | Code | Description |
|------|------|-------------|
| **CDI** | `CDI` | Contrat à durée indéterminée |
| **CDD** | `CDD` | Contrat à durée déterminée |
| **TEMPORAIRE** | `TEMPORAIRE` | Travail temporaire/saisonnier |
| **STAGE** | `STAGE` | Stagiaire |
| **APPRENTISSAGE** | `APPRENTISSAGE` | Programme d'apprentissage |

---

## Niveaux de Compétence

| Niveau | Code | Description |
|--------|------|-------------|
| **Débutant** | `DÉBUTANT` | Niveau d'entrée |
| **Intermédiaire** | `INTERMÉDIAIRE` | Expérience de base |
| **Avancé** | `AVANCÉ` | Expérience confirmée |
| **Expert** | `EXPERT` | Maîtrise complète |

---

## Catalogue des Compétences (100+)

### Charpenterie et Menuiserie

| Compétence | Description |
|------------|-------------|
| Charpenterie résidentielle | Construction maisons |
| Charpenterie commerciale | Bâtiments commerciaux |
| Charpenterie institutionnelle | Écoles, hôpitaux |
| Ossature bois plateforme | Technique platform frame |
| Ossature bois ballon | Technique ballon frame |
| Charpente traditionnelle | Techniques ancestrales |
| Installation toiture | Pose bardeaux, membranes |
| Pose de revêtement | Vinyle, aluminium, bois |
| Certification CCQ Charpentier | Carte de compétence |
| Escaliers et rampes | Construction escaliers |
| Finition intérieure | Moulures, cadrages |
| Isolation thermique | Pose isolants |
| Portes et fenêtres | Installation menuiserie |
| Terrasses et balcons | Construction extérieure |

### Maçonnerie et Béton

| Compétence | Description |
|------------|-------------|
| Coffrage de fondations | Murs et semelles |
| Coffrage murs | Murs de fondation |
| Coffrage dalles | Dalles de béton |
| Coulage béton | Mise en place béton |
| Finition béton | Surfaçage, lissage |
| Béton décoratif | Béton estampé, coloré |
| Maçonnerie brique | Pose de briques |
| Maçonnerie bloc | Blocs de béton |
| Maçonnerie pierre | Pierre naturelle |
| Armature béton | Ferraillage |
| Lecture de plans structure | Interprétation plans |

### Qualité et Conformité

| Compétence | Description |
|------------|-------------|
| Inspection structure | Vérification structures |
| Conformité RBQ | Réglementation RBQ |
| Normes CCQ | Règles CCQ |
| Code du bâtiment | Code construction Québec |
| Contrôle qualité | Assurance qualité |
| ISO 9001 | Système qualité |
| Vérification sécurité | Audit sécurité |

### Conception et Ingénierie

| Compétence | Description |
|------------|-------------|
| AutoCAD | Dessin 2D |
| Revit | BIM 3D |
| SketchUp | Modélisation 3D |
| Plans construction | Lecture et création |
| Devis technique | Rédaction devis |
| Calcul structure | Calculs ingénierie |
| Code construction Québec | Réglementation |
| Normes CSA | Standards canadiens |

### Équipements Construction

| Compétence | Description |
|------------|-------------|
| Grue à tour | Opération grue |
| Chargeuse-pelleteuse | Engins de chantier |
| Nacelle élévatrice | Travail en hauteur |
| Pompe à béton | Pompage béton |
| Génératrice chantier | Équipement électrique |

### Sécurité et Réglementation Québec

| Compétence | Description |
|------------|-------------|
| CNESST | Santé-sécurité travail |
| Cadenassage LOTO | Lockout/Tagout |
| Espaces clos | Travail confiné |
| Travail en hauteur | Protection chutes |
| SIMDUT 2015 | Matières dangereuses |
| Premiers soins | Secourisme |
| EPI | Équipements protection |

---

## Fonctionnalités Principales

### 1. Gestion des Employés (CRUD)

| Action | Bouton | Description |
|--------|--------|-------------|
| **Créer** | ➕ Nouvel employé | Ajouter un employé |
| **Lire** | 👁️ Voir | Consulter la fiche |
| **Modifier** | ✏️ Modifier | Éditer les informations |
| **Supprimer** | 🗑️ Supprimer | Retirer l'employé |

### 2. Fiche Employé Complète

| Section | Informations |
|---------|--------------|
| **Identité** | Prénom, nom, photo |
| **Contact** | Téléphone, courriel |
| **Emploi** | Poste, département, manager |
| **Contrat** | Type, date embauche, salaire |
| **Charge** | Taux d'occupation (%) |
| **PIN** | Code pour TimeTracker |

### 3. Gestion des Compétences

Pour chaque employé :
- Liste des compétences acquises
- Niveau de maîtrise
- Certification officielle (oui/non)
- Date d'obtention
- Date d'expiration

### 4. Assignation aux Projets

| Champ | Description |
|-------|-------------|
| `project_id` | Projet assigné |
| `employee_id` | Employé concerné |
| `role_projet` | Rôle dans le projet |

### 5. Hiérarchie Managériale

Chaque employé peut avoir un `manager_id` :
- Organigramme automatique
- Validation des absences
- Approbation des heures

---

## Calcul du Salaire Construction

Le système calcule automatiquement les salaires selon les conventions CCQ :

### Formule de Base

```python
def _calculer_salaire_construction(poste, experience_annees=5):
    # Salaires horaires de base CCQ 2024
    salaires_base = {
        'Charpentier-menuisier': 42.50,
        'Électricien': 45.00,
        'Plombier': 44.00,
        'Maçon': 41.00,
        'Opérateur équipement lourd': 43.00,
        ...
    }

    # Prime d'expérience
    prime = min(experience_annees * 0.5, 5.0)  # Max 5$/h

    return salaires_base[poste] + prime
```

---

## Statistiques Dashboard

| Métrique | Description |
|----------|-------------|
| **Total employés** | Nombre total |
| **Employés actifs** | Statut ACTIF |
| **Par département** | Répartition |
| **Par statut** | Distribution statuts |
| **Charge moyenne** | % occupation moyen |

### Requête Statistiques

```sql
SELECT
    COUNT(*) as total,
    SUM(CASE WHEN statut = 'ACTIF' THEN 1 ELSE 0 END) as actifs,
    AVG(charge_travail) as charge_moyenne
FROM employees
```

---

## Guide Pas-à-Pas

### Ajouter un nouvel employé

1. Cliquez sur **"➕ Nouvel employé"**
2. Remplissez les informations :

   **Section Identité :**
   - Prénom (obligatoire)
   - Nom (obligatoire)
   - Photo (optionnel)

   **Section Contact :**
   - Téléphone
   - Courriel

   **Section Emploi :**
   - Département (sélection parmi 11)
   - Poste
   - Manager direct
   - Charge de travail (%)

   **Section Contrat :**
   - Type de contrat (CDI, CDD, etc.)
   - Date d'embauche
   - Salaire ou taux horaire

   **Section TimeTracker :**
   - Code PIN (pour pointage)

3. Cliquez sur **"💾 Enregistrer"**
4. L'employé est créé avec statut ACTIF

### Ajouter des compétences

1. Ouvrez la fiche de l'employé
2. Section **"🎓 Compétences"**
3. Cliquez sur **"➕ Ajouter"**
4. Sélectionnez la compétence dans le catalogue
5. Définissez :
   - Niveau : DÉBUTANT à EXPERT
   - Certifié : Oui/Non
   - Date d'obtention
   - Date d'expiration (si applicable)
6. Enregistrez

### Gérer les certifications CCQ

1. Ouvrez la fiche de l'employé
2. Ajoutez la compétence "Certification CCQ Charpentier" (ou autre)
3. Cochez **"Certifié"**
4. Entrez la date d'expiration de la carte
5. Le système alertera avant expiration

### Assigner à un projet

1. Ouvrez la fiche de l'employé
2. Section **"📋 Projets assignés"**
3. Cliquez sur **"➕ Assigner"**
4. Sélectionnez le projet
5. Définissez le rôle dans le projet
6. Enregistrez

### Filtrer les employés

1. Utilisez les filtres en haut de la liste
2. Filtrez par :
   - Département (liste déroulante)
   - Statut (ACTIF, CONGÉ, etc.)
   - Manager
   - Compétence spécifique
3. Les résultats se mettent à jour en temps réel

### Consulter l'organigramme

1. Cliquez sur **"📊 Organigramme"**
2. La hiérarchie managériale s'affiche
3. Les liens manager → employé sont visualisés
4. Cliquez sur un noeud pour voir les détails

---

## Système de Cache

| Donnée | TTL | Fonction |
|--------|-----|----------|
| Liste employés | 10 min | Cache local |
| Compétences | 10 min | Chargement au besoin |
| Assignations | 5 min | Données dynamiques |

---

## Intégrations

| Module | Type | Description |
|--------|------|-------------|
| **TimeTracker** | 1-N | Pointages de l'employé |
| **Production** | N-N | Assignations aux BT |
| **Projets** | N-N | Assignations aux projets |
| **RBQ/CCQ** | 1-1 | Conformité certifications |
| **Paie** | 1-N | Calcul des salaires |

---

## Astuces et Bonnes Pratiques

- **Gardez les dossiers à jour** : Surtout les certifications et contacts
- **Planifiez les renouvellements** : 60 jours avant l'expiration des cartes CCQ
- **Documentez les formations** : Chaque formation suivie doit être enregistrée
- **Utilisez les départements** : Pour filtrer et organiser efficacement
- **Définissez les managers** : Pour l'organigramme et les approbations
- **Code PIN unique** : Chaque employé doit avoir un PIN pour le TimeTracker
- **Charge de travail** : Surveillez la surcharge (>100%)

---

## Résolution de Problèmes

### L'employé n'apparaît pas dans la liste

- **Cause** : Statut INACTIF ou filtre actif
- **Solution** : Vérifiez les filtres ou changez le statut

### Le PIN TimeTracker ne fonctionne pas

- **Cause** : PIN non défini ou dupliqué
- **Solution** : Vérifiez que le PIN est unique et bien enregistré

### Les compétences ne s'affichent pas

- **Cause** : Aucune compétence ajoutée
- **Solution** : Ajoutez des compétences via la fiche employé

### L'employé ne peut pas être assigné au projet

- **Cause** : Employé non ACTIF ou projet clos
- **Solution** : Vérifiez les statuts

---

## Questions Fréquentes (FAQ)

**Q: Les informations sensibles sont-elles protégées ?**
R: Oui, les informations comme le NAS et les données bancaires sont chiffrées. Seuls les utilisateurs autorisés avec les permissions appropriées peuvent y accéder.

**Q: Puis-je importer une liste d'employés ?**
R: Oui, utilisez l'import CSV dans Configuration > Import. Le format doit correspondre au modèle fourni avec les colonnes : prenom, nom, email, telephone, poste, departement, statut.

**Q: Comment gérer les employés à temps partiel ?**
R: Créez-les comme employés réguliers avec un type de contrat "CDD" ou "TEMPORAIRE" et ajustez la charge de travail (ex: 50% pour mi-temps).

**Q: Les certifications CCQ sont-elles vérifiées automatiquement ?**
R: Le système alerte sur les expirations mais ne vérifie pas directement auprès de la CCQ. Utilisez le module RBQ/CCQ pour la vérification manuelle.

**Q: Comment calculer la charge de travail d'un département ?**
R: Consultez les statistiques du module Employés qui agrègent la charge par département.

**Q: Puis-je exporter la liste des employés ?**
R: Oui, utilisez le bouton **"📥 Exporter CSV"** pour télécharger la liste complète.

---

## Données Techniques

### Requête Liste Employés

```sql
SELECT e.*,
       m.prenom || ' ' || m.nom as manager_nom
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY e.nom, e.prenom
```

### Requête Employés par Département

```sql
SELECT departement, COUNT(*) as count
FROM employees
WHERE statut = 'ACTIF'
GROUP BY departement
ORDER BY count DESC
```

### Requête Compétences Employé

```sql
SELECT ec.*, e.prenom, e.nom
FROM employee_competences ec
JOIN employees e ON ec.employee_id = e.id
WHERE ec.employee_id = :employee_id
ORDER BY ec.niveau DESC, ec.nom_competence
```

---

## Voir Aussi

- [⏱️ TimeTracker](13-timetracker.md) - Pointage des heures
- [🏗️ RBQ/CCQ](14-rbq-ccq.md) - Conformité construction
- [🏭 Production](11-production.md) - Assignation aux BT
- [📋 Projets](07-projets.md) - Assignation aux projets
- [💰 Comptabilité](19-comptabilite.md) - Paie et salaires
