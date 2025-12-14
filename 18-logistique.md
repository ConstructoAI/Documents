# 🚚 Logistique

## Introduction

Le module **Logistique** gère le transport et la livraison de vos matériaux entre entrepôts, fournisseurs et chantiers. Il permet de planifier les livraisons, gérer les équipements et véhicules, coordonner les activités de chantier et suivre les expéditions en temps réel.

Ce module comprend 6 tables PostgreSQL pour la gestion complète : livraisons, lignes de livraison, équipements, réservations, véhicules, trajets, coordination et alertes.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"🚚 Logistique"**
2. Le tableau de bord logistique s'affiche avec les onglets :
   - **Tableau de bord** : Vue d'ensemble et alertes
   - **Livraisons** : Planification et suivi
   - **Équipements** : Gestion du parc matériel
   - **Véhicules** : Flotte de transport
   - **Coordination** : Planning chantier
3. Gérez vos livraisons, équipements et tournées

---

## Structure des Données

### Table : `logistics_deliveries`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `reference` | TEXT | Référence unique (LIV-YYYY-XXXX) |
| `project_id` | INTEGER | Lien vers projet |
| `fournisseur_id` | INTEGER | Lien vers fournisseur |
| `date_prevue` | DATE | Date de livraison prévue |
| `heure_prevue` | TIME | Heure estimée d'arrivée |
| `date_effective` | DATE | Date réelle de livraison |
| `heure_effective` | TIME | Heure réelle |
| `statut` | VARCHAR(30) | État de la livraison |
| `type_livraison` | VARCHAR(50) | Type (entrante, sortante, etc.) |
| `zone_stockage` | TEXT | Zone de destination |
| `notes` | TEXT | Instructions de livraison |
| `created_at` | TIMESTAMP | Date de création |

### Table : `logistics_delivery_lines`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `delivery_id` | INTEGER | Lien vers livraison |
| `product_id` | INTEGER | Lien vers produit |
| `quantite_prevue` | REAL | Quantité commandée |
| `quantite_recue` | REAL | Quantité reçue |
| `notes` | TEXT | Notes sur l'article |

### Table : `logistics_equipment`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `nom` | TEXT | Nom de l'équipement |
| `code` | TEXT | Code unique |
| `categorie` | TEXT | Type d'équipement |
| `description` | TEXT | Description |
| `cout_journalier` | DECIMAL | Coût par jour |
| `cout_mensuel` | DECIMAL | Coût par mois |
| `date_acquisition` | DATE | Date d'achat |
| `valeur_achat` | DECIMAL | Prix d'acquisition |
| `heures_utilisation` | DECIMAL | Total heures utilisées |
| `statut` | VARCHAR(30) | Disponibilité |
| `localisation_actuelle` | TEXT | Emplacement |
| `project_id_actuel` | INTEGER | Projet en cours |
| `notes` | TEXT | Notes techniques |

### Table : `logistics_equipment_reservations`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `equipment_id` | INTEGER | Lien vers équipement |
| `project_id` | INTEGER | Lien vers projet |
| `date_debut` | DATE | Début de réservation |
| `date_fin` | DATE | Fin de réservation |
| `responsable` | TEXT | Personne responsable |
| `statut` | VARCHAR(30) | État de la réservation |

### Table : `logistics_vehicles`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `immatriculation` | TEXT | Plaque d'immatriculation |
| `marque` | TEXT | Marque du véhicule |
| `modele` | TEXT | Modèle |
| `annee` | INTEGER | Année de fabrication |
| `type_vehicule` | VARCHAR(50) | Type (camion, van, etc.) |
| `capacite_charge` | DECIMAL | Capacité de chargement |
| `unite_capacite` | TEXT | Unité (kg, m³) |
| `kilometrage` | INTEGER | Kilométrage actuel |
| `consommation_moyenne` | DECIMAL | L/100km |
| `cout_km` | DECIMAL | Coût par kilomètre |
| `statut` | VARCHAR(30) | Disponibilité |
| `notes` | TEXT | Notes entretien |

### Table : `logistics_trips`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `vehicle_id` | INTEGER | Lien vers véhicule |
| `chauffeur` | TEXT | Nom du chauffeur |
| `date_depart` | DATE | Date du trajet |
| `km_depart` | INTEGER | Kilométrage départ |
| `km_arrivee` | INTEGER | Kilométrage arrivée |
| `destination` | TEXT | Destination |
| `motif` | TEXT | Raison du déplacement |
| `notes` | TEXT | Remarques |

### Table : `logistics_coordination`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `project_id` | INTEGER | Lien vers projet |
| `date_coordination` | DATE | Date de l'activité |
| `type_activite` | TEXT | Type d'activité |
| `heure_debut` | TIME | Heure de début |
| `heure_fin` | TIME | Heure de fin |
| `description` | TEXT | Description |
| `responsable` | TEXT | Personne responsable |
| `sequence_ordre` | INTEGER | Ordre dans la journée |
| `notes` | TEXT | Notes additionnelles |

### Table : `logistics_alerts`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `type_alerte` | TEXT | Type d'alerte |
| `message` | TEXT | Message d'alerte |
| `equipment_id` | INTEGER | Équipement concerné |
| `date_echeance` | DATE | Date limite |
| `statut` | VARCHAR(20) | active/resolved |

---

## Statuts des Livraisons

| Statut | Description | Couleur |
|--------|-------------|---------|
| **planifiee** | Livraison programmée, non encore partie | 🟡 Jaune |
| **en_cours** | En route vers la destination | 🟠 Orange |
| **livree** | Livraison effectuée avec succès | 🟢 Vert |
| **annulee** | Livraison annulée | 🔴 Rouge |

---

## Types de Livraisons

| Type | Description | Direction |
|------|-------------|-----------|
| **Livraison fournisseur** | Réception depuis un fournisseur | Entrante |
| **Livraison chantier** | Envoi vers un projet/chantier | Sortante |
| **Transfert entrepôt** | Entre deux entrepôts | Interne |
| **Retour fournisseur** | Retour de marchandises | Sortante |
| **Collecte chantier** | Récupération de surplus | Entrante |

---

## Catégories d'Équipements

| Catégorie | Exemples |
|-----------|----------|
| **Grue** | Grue à tour, grue mobile |
| **Excavatrice** | Pelle hydraulique, mini-pelle |
| **Chargeuse** | Chargeuse sur roues, bobcat |
| **Échafaudage** | Échafaudage tubulaire, tour roulante |
| **Compacteur** | Rouleau compacteur, plaque vibrante |
| **Bétonnière** | Bétonnière portative, malaxeur |
| **Génératrice** | Groupe électrogène |
| **Autre** | Équipement divers |

---

## Statuts des Équipements

| Statut | Description | Couleur |
|--------|-------------|---------|
| **disponible** | Prêt à être utilisé | 🟢 Vert |
| **en_utilisation** | Actuellement sur un projet | 🟠 Orange |
| **maintenance** | En réparation/entretien | 🔴 Rouge |
| **reserve** | Réservé pour un projet futur | 🟡 Jaune |

---

## Statuts des Véhicules

| Statut | Description | Couleur |
|--------|-------------|---------|
| **disponible** | Prêt à partir | 🟢 Vert |
| **en_deplacement** | En route | 🟠 Orange |
| **maintenance** | En garage | 🔴 Rouge |

---

## Fonctionnalités Principales

### 1. Tableau de Bord Logistique

Vue d'ensemble en temps réel :

| Section | Information |
|---------|-------------|
| **Livraisons** | Planifiées, En cours, Cette semaine |
| **Équipements** | Disponibles, En utilisation, Maintenance |
| **Véhicules** | Disponibles, En déplacement |
| **Alertes** | Alertes actives nécessitant action |
| **Aujourd'hui** | Livraisons du jour avec détails |

### 2. Gestion des Livraisons

| Fonctionnalité | Description |
|----------------|-------------|
| **Planification** | Créer et programmer des livraisons |
| **Suivi** | Suivre l'état en temps réel |
| **Réception** | Confirmer les livraisons reçues |
| **Historique** | Consulter les livraisons passées |

### 3. Gestion des Équipements

| Fonctionnalité | Description |
|----------------|-------------|
| **Inventaire** | Liste complète du parc matériel |
| **Réservation** | Réserver pour un projet et période |
| **Suivi** | Localisation et heures d'utilisation |
| **Alertes** | Maintenance préventive |

### 4. Gestion des Véhicules

| Fonctionnalité | Description |
|----------------|-------------|
| **Flotte** | Inventaire des véhicules |
| **Trajets** | Enregistrer les déplacements |
| **Kilométrage** | Suivi du kilométrage |
| **Coûts** | Calcul des coûts de transport |

### 5. Coordination Chantier

| Fonctionnalité | Description |
|----------------|-------------|
| **Planning** | Activités par projet et date |
| **Séquence** | Ordre des activités dans la journée |
| **Responsables** | Attribution des tâches |
| **Horaires** | Heures de début et fin |

---

## Guide Pas-à-Pas

### Planifier une livraison

1. Onglet **"Livraisons"**
2. Cliquez sur **"➕ Nouvelle livraison"**
3. Remplissez les informations :

   **Section Destination :**
   - Sélectionnez le projet (chantier de destination)
   - Sélectionnez le fournisseur

   **Section Planification :**
   - Date prévue de livraison
   - Heure prévue (créneau)

   **Section Détails :**
   - Type de livraison (parmi les 5 types)
   - Zone de stockage sur le chantier
   - Notes / Instructions de livraison

4. Cliquez sur **"✅ Créer la livraison"**
5. La livraison apparaît dans la liste avec statut "planifiee"

### Confirmer une livraison reçue

1. Trouvez la livraison dans la liste
2. Cliquez sur la livraison pour ouvrir les détails
3. Cliquez sur **"✅ Marquer comme livrée"**
4. Le statut passe à "livree"
5. La date effective est enregistrée automatiquement
6. L'inventaire est mis à jour si configuré

### Ajouter un équipement

1. Onglet **"Équipements"**
2. Cliquez sur **"➕ Nouvel équipement"**
3. Remplissez les informations :

   **Section Identification :**
   - Nom de l'équipement
   - Code unique
   - Catégorie (Grue, Excavatrice, etc.)

   **Section Coûts :**
   - Coût journalier de location/utilisation
   - Coût mensuel
   - Valeur d'achat
   - Date d'acquisition

   **Section Localisation :**
   - Localisation actuelle
   - Notes techniques

4. Cliquez sur **"✅ Ajouter"**
5. L'équipement est disponible pour réservation

### Réserver un équipement pour un projet

1. Onglet **"Équipements"**
2. Trouvez l'équipement disponible
3. Cliquez sur **"📅 Réserver"**
4. Sélectionnez :
   - Projet destinataire
   - Date de début
   - Date de fin
   - Responsable sur le chantier
5. Validez la réservation
6. L'équipement passe en statut "en_utilisation" si la réservation commence aujourd'hui

### Ajouter un véhicule

1. Onglet **"Véhicules"**
2. Cliquez sur **"➕ Nouveau véhicule"**
3. Remplissez les informations :

   **Section Identification :**
   - Immatriculation (plaque)
   - Marque et modèle
   - Année de fabrication
   - Type de véhicule

   **Section Capacité :**
   - Capacité de charge
   - Unité (kg, m³)

   **Section Suivi :**
   - Kilométrage actuel
   - Consommation moyenne (L/100km)
   - Coût par kilomètre

4. Cliquez sur **"✅ Ajouter"**

### Enregistrer un trajet

1. Onglet **"Véhicules"**
2. Trouvez le véhicule disponible
3. Cliquez sur **"🚗 Nouveau trajet"**
4. Remplissez :
   - Chauffeur
   - Date du départ
   - Kilométrage au départ
   - Destination
   - Motif du déplacement
5. Au retour, complétez :
   - Kilométrage à l'arrivée
   - Notes (incidents, remarques)
6. Le véhicule redevient "disponible"
7. Le kilométrage est mis à jour automatiquement

### Planifier une activité de coordination

1. Onglet **"Coordination"**
2. Sélectionnez le projet
3. Sélectionnez la date
4. Cliquez sur **"➕ Nouvelle activité"**
5. Remplissez :
   - Type d'activité
   - Heure de début et de fin
   - Description
   - Responsable
   - Ordre de séquence (si plusieurs activités)
6. Validez
7. L'activité apparaît dans le planning du jour

---

## Statistiques et Métriques

Le tableau de bord affiche :

### Livraisons
| Métrique | Description |
|----------|-------------|
| **Planifiées** | Livraisons à venir |
| **En cours** | Livraisons en transit |
| **Cette semaine** | Livraisons effectuées dans les 7 derniers jours |

### Équipements
| Métrique | Description |
|----------|-------------|
| **Total** | Nombre total d'équipements |
| **Disponibles** | Prêts à être utilisés |
| **En utilisation** | Sur des projets |
| **Maintenance** | En réparation |

### Véhicules
| Métrique | Description |
|----------|-------------|
| **Total** | Nombre total de véhicules |
| **Disponibles** | Prêts à partir |
| **En déplacement** | Actuellement en route |

---

## Alertes Équipements

Le système génère des alertes automatiques :

| Type d'alerte | Déclencheur |
|---------------|-------------|
| **Maintenance préventive** | Heures d'utilisation atteintes |
| **Fin de réservation** | Équipement à récupérer |
| **Certification expirée** | Document à renouveler |
| **Inspection requise** | Date d'inspection passée |

---

## Astuces et Bonnes Pratiques

- **Planifiez à l'avance** : Créez les livraisons dès la commande passée
- **Confirmez la veille** : Vérifiez l'accès au chantier et la disponibilité
- **Documentez** : Ajoutez des photos, signatures et notes à chaque livraison
- **Communiquez** : Prévenez le chef de chantier de l'heure d'arrivée
- **Suivez les heures** : Mettez à jour les heures d'utilisation des équipements
- **Maintenez les véhicules** : Respectez les intervalles d'entretien
- **Optimisez les trajets** : Groupez les livraisons par zone géographique
- **Vérifiez les alertes** : Consultez quotidiennement les alertes actives

---

## Résolution de Problèmes

### La livraison n'apparaît pas dans la liste

- **Cause** : Filtre de statut actif
- **Solution** : Changez le filtre à "Tous"

### L'équipement ne peut pas être réservé

- **Cause** : Déjà réservé pour la période ou en maintenance
- **Solution** : Vérifiez les réservations existantes et le statut

### Le kilométrage ne se met pas à jour

- **Cause** : Trajet non terminé (km arrivée non renseigné)
- **Solution** : Complétez le trajet avec le kilométrage final

### L'alerte reste active après résolution

- **Cause** : Statut non mis à jour
- **Solution** : Marquez l'alerte comme résolue manuellement

---

## Questions Fréquentes (FAQ)

**Q: Puis-je suivre les livraisons en temps réel ?**
R: Si vos véhicules sont équipés de GPS et connectés au système, oui. Sinon, le suivi est basé sur les mises à jour manuelles du statut.

**Q: Comment gérer les livraisons urgentes ?**
R: Créez une livraison normale et ajoutez "URGENT" dans les notes. Contactez directement le fournisseur pour accélérer.

**Q: Les chauffeurs peuvent-ils utiliser une application mobile ?**
R: CONSTRUCTO AI est accessible sur mobile via navigateur. Les chauffeurs peuvent mettre à jour les statuts depuis leur téléphone.

**Q: Comment calculer les coûts de transport ?**
R: Entrez le coût par km dans la fiche véhicule. Le système calcule automatiquement le coût de chaque trajet.

**Q: Puis-je voir l'historique d'un équipement ?**
R: Oui, ouvrez la fiche équipement pour voir toutes les réservations passées et les heures d'utilisation cumulées.

**Q: Comment gérer les équipements loués ?**
R: Ajoutez-les comme équipements avec le coût de location. Supprimez-les à la fin de la période de location.

---

## Données Techniques

### Requête Livraisons avec Filtres

```sql
SELECT d.*, p.nom_projet as project_name, c.nom as fournisseur_nom
FROM logistics_deliveries d
LEFT JOIN projects p ON d.project_id = p.id
LEFT JOIN companies c ON d.fournisseur_id = c.id
WHERE d.date_prevue >= :date_debut
  AND d.date_prevue <= :date_fin
  AND d.statut = :statut
ORDER BY d.date_prevue DESC, d.heure_prevue
```

### Requête Statistiques Livraisons

```sql
SELECT
    COUNT(CASE WHEN statut = 'planifiee' THEN 1 END) as planifiees,
    COUNT(CASE WHEN statut = 'en_cours' THEN 1 END) as en_cours,
    COUNT(CASE WHEN statut = 'livree' AND date_effective >= CURRENT_DATE - INTERVAL '7 days' THEN 1 END) as cette_semaine
FROM logistics_deliveries
```

### Requête Équipements par Statut

```sql
SELECT
    COUNT(*) as total,
    COUNT(CASE WHEN statut = 'disponible' THEN 1 END) as disponibles,
    COUNT(CASE WHEN statut = 'en_utilisation' THEN 1 END) as en_utilisation,
    COUNT(CASE WHEN statut = 'maintenance' THEN 1 END) as en_maintenance
FROM logistics_equipment
```

### Requête Coordination par Projet et Date

```sql
SELECT * FROM logistics_coordination
WHERE project_id = :project_id
  AND date_coordination = :date
ORDER BY sequence_ordre, heure_debut
```

---

## Index de Performance

Le système utilise des index pour optimiser les requêtes :

```sql
CREATE INDEX idx_deliveries_date ON logistics_deliveries(date_prevue);
CREATE INDEX idx_deliveries_statut ON logistics_deliveries(statut);
CREATE INDEX idx_equipment_statut ON logistics_equipment(statut);
CREATE INDEX idx_vehicles_statut ON logistics_vehicles(statut);
CREATE INDEX idx_alerts_statut ON logistics_alerts(statut);
```

---

## Voir Aussi

- [📦 Inventaire](17-inventaire.md) - Gestion des stocks
- [🏪 Achats](16-achats.md) - Bons de commande
- [🏭 Production](11-production.md) - Besoins chantier
- [📋 Projets](07-projets.md) - Adresses de livraison
- [📅 Calendrier](08-calendrier.md) - Planification globale
