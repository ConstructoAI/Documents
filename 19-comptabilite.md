# 💰 Comptabilité

## Introduction

Le module **Comptabilité** est une solution complète de gestion financière, comparable à QuickBooks, conçue spécifiquement pour les entreprises de construction au Québec. Il gère la facturation clients, les dépenses fournisseurs, le grand livre avec écritures automatiques, les états financiers et la conformité aux normes fiscales québécoises (TPS/TVQ).

Ce module inclut 18 catégories de dépenses construction, 6 statuts de facture, 6 conditions de paiement, et génère automatiquement les écritures comptables lors de l'envoi de factures.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"💰 Comptabilité"**
2. Le tableau de bord financier s'affiche avec les onglets :
   - **Factures** : Comptes à recevoir (clients)
   - **Dépenses** : Comptes à payer (fournisseurs)
   - **Grand Livre** : Écritures comptables
   - **États Financiers** : Bilan, résultats, trésorerie
   - **Paie** : Gestion des salaires
3. Naviguez entre les onglets selon vos besoins

---

## Structure des Données

### Table PostgreSQL : `factures`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `numero_facture` | TEXT | Référence unique (FAC-YYYY-XXXX) |
| `company_id` | INTEGER | Client facturé |
| `project_id` | INTEGER | Projet associé (optionnel) |
| `date_facture` | DATE | Date d'émission |
| `date_echeance` | DATE | Date limite de paiement |
| `montant_ht` | REAL | Montant hors taxes |
| `tps` | REAL | Taxe fédérale (5%) |
| `tvq` | REAL | Taxe provinciale (9.975%) |
| `montant_ttc` | REAL | Total toutes taxes comprises |
| `montant_paye` | REAL | Montant déjà reçu |
| `statut` | TEXT | État de la facture |
| `conditions_paiement` | TEXT | Net 30, etc. |
| `journal_entry_id` | INTEGER | Lien vers écriture comptable |
| `notes` | TEXT | Notes internes |
| `created_at` | TIMESTAMP | Date de création |

### Table : `depenses`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Identifiant unique |
| `numero_depense` | TEXT | Référence unique |
| `fournisseur_id` | INTEGER | Fournisseur |
| `project_id` | INTEGER | Projet associé |
| `date_depense` | DATE | Date de la facture fournisseur |
| `date_echeance` | DATE | Date limite de paiement |
| `categorie` | TEXT | Classification (18 catégories) |
| `montant_ht` | REAL | Montant hors taxes |
| `tps` | REAL | TPS récupérable |
| `tvq` | REAL | TVQ récupérable |
| `montant_ttc` | REAL | Total à payer |
| `montant_paye` | REAL | Montant déjà payé |
| `statut` | TEXT | État du paiement |
| `journal_entry_id` | INTEGER | Lien vers écriture comptable |
| `piece_jointe` | TEXT | Chemin vers la pièce justificative |
| `notes` | TEXT | Notes internes |

---

## Les 18 Catégories de Dépenses

| # | Catégorie | Exemples |
|---|-----------|----------|
| 1 | **Matériaux de construction** | Bois, béton, acier, isolation |
| 2 | **Matériel et équipement** | Outils, machines, équipement lourd |
| 3 | **Main-d'œuvre** | Sous-traitance de main-d'œuvre |
| 4 | **Salaires et charges** | Paie employés, charges sociales |
| 5 | **Sous-traitance** | Travaux confiés à d'autres entreprises |
| 6 | **Loyer et charges locatives** | Bureau, entrepôt, locaux |
| 7 | **Électricité et chauffage** | Énergie des locaux |
| 8 | **Téléphone et internet** | Communications |
| 9 | **Assurances** | Responsabilité civile, chantier, véhicules |
| 10 | **Entretien et réparations** | Maintenance équipement et locaux |
| 11 | **Carburant et transport** | Essence, diesel, frais de déplacement |
| 12 | **Frais bancaires** | Frais de compte, intérêts |
| 13 | **Frais juridiques et comptables** | Avocat, comptable, notaire |
| 14 | **Publicité et marketing** | Publicité, site web, représentation |
| 15 | **Fournitures de bureau** | Papeterie, consommables |
| 16 | **Formation** | Cours, certifications, perfectionnement |
| 17 | **Taxes et permis** | Permis de construction, taxes municipales |
| 18 | **Autres dépenses** | Dépenses diverses non catégorisées |

---

## Statuts des Factures

| Statut | Description | Couleur |
|--------|-------------|---------|
| **BROUILLON** | En préparation, non envoyée | ⚪ Gris |
| **ENVOYEE** | Transmise au client | 🔵 Bleu |
| **PAYEE** | Paiement complet reçu | 🟢 Vert |
| **PARTIELLEMENT_PAYEE** | Paiement partiel reçu | 🟡 Jaune |
| **EN_RETARD** | Date d'échéance dépassée | 🔴 Rouge |
| **ANNULEE** | Facture annulée | ⚫ Noir |

---

## Conditions de Paiement

| Condition | Description |
|-----------|-------------|
| **Net 15** | Paiement dans les 15 jours |
| **Net 30** | Paiement dans les 30 jours (standard) |
| **Net 45** | Paiement dans les 45 jours |
| **Net 60** | Paiement dans les 60 jours |
| **Sur réception** | Paiement à réception de la facture |
| **50% acompte** | Acompte de 50% à la commande |

---

## Modes de Paiement

| Mode | Description |
|------|-------------|
| **Comptant** | Paiement en espèces |
| **Chèque** | Paiement par chèque |
| **Virement bancaire** | Transfert électronique |
| **Carte de crédit** | VISA, MasterCard, etc. |
| **Débit préautorisé** | Prélèvement automatique |
| **Autre** | Autre mode de paiement |

---

## Fonctionnalités Principales

### 1. Facturation Clients (Comptes à Recevoir)

| Fonctionnalité | Description |
|----------------|-------------|
| **Création facture** | Manuelle ou depuis devis accepté |
| **Numérotation** | FAC-YYYY-XXXX automatique |
| **Lignes détaillées** | Produits, services, quantités |
| **Taxes** | TPS (5%) + TVQ (9.975%) calculées automatiquement |
| **Envoi** | Email avec PDF ou téléchargement |
| **Suivi paiements** | Paiements partiels ou totaux |
| **Relances** | Rappels automatiques pour factures en retard |

### 2. Gestion des Dépenses (Comptes à Payer)

| Fonctionnalité | Description |
|----------------|-------------|
| **Saisie** | Factures fournisseurs avec pièces jointes |
| **Catégorisation** | Classification (18 catégories) |
| **Attribution projet** | Ventilation des coûts par projet |
| **Pièces justificatives** | Numérisation et archivage |
| **Planification** | Échéancier des paiements |
| **Taxes récupérables** | Suivi TPS/TVQ remboursables |

### 3. Grand Livre

| Fonctionnalité | Description |
|----------------|-------------|
| **Plan comptable** | 70+ comptes spécialisés construction |
| **Écritures automatiques** | Générées à l'envoi des factures |
| **Consultation** | Par période, compte, type |
| **Balance** | Débits = Crédits |
| **Export** | Excel, PDF pour comptable |

### 4. États Financiers

| Rapport | Description |
|---------|-------------|
| **Bilan** | Actif/Passif/Capitaux à une date |
| **État des résultats** | Produits - Charges = Profit/Perte |
| **Flux de trésorerie** | Entrées et sorties de cash |
| **Balance âgée clients** | Créances par ancienneté (0-30, 31-60, 61-90, 90+) |
| **Balance âgée fournisseurs** | Dettes par ancienneté |

### 5. Intégration Grand Livre Automatique

Le système génère automatiquement les écritures comptables :

```
Lors de l'envoi d'une facture client :
- Débit : Compte client (1200)
- Crédit : Ventes (4100)
- Crédit : TPS à payer (2300)
- Crédit : TVQ à payer (2310)

Lors de l'enregistrement d'une dépense :
- Débit : Compte de charge (5xxx)
- Débit : TPS à recevoir (1300)
- Débit : TVQ à recevoir (1310)
- Crédit : Compte fournisseur (2100)
```

---

## Guide Pas-à-Pas

### Créer une facture client

1. Onglet **"Factures"**
2. Cliquez sur **"➕ Nouvelle facture"**
3. Sélectionnez le client dans la liste
4. Associez au projet (optionnel)
5. Ajoutez les lignes de facturation :
   - Description du service/produit
   - Quantité
   - Prix unitaire
   - Le total ligne se calcule
6. Les taxes TPS (5%) et TVQ (9.975%) sont calculées automatiquement
7. Sélectionnez les conditions de paiement (Net 30, etc.)
8. Ajoutez des notes si nécessaire
9. **"💾 Enregistrer"** (brouillon) ou **"📤 Envoyer"**
10. À l'envoi, l'écriture comptable est générée automatiquement

### Enregistrer un paiement reçu

1. Trouvez la facture dans la liste
2. Cliquez sur la facture pour l'ouvrir
3. Cliquez sur **"💵 Enregistrer paiement"**
4. Entrez :
   - Montant reçu
   - Date du paiement
   - Mode de paiement (Chèque, Virement, etc.)
   - Référence (n° de chèque, confirmation)
5. Si paiement partiel → statut "PARTIELLEMENT_PAYEE"
6. Si paiement complet → statut "PAYEE"
7. L'écriture de paiement est générée

### Saisir une dépense fournisseur

1. Onglet **"Dépenses"**
2. Cliquez sur **"➕ Nouvelle dépense"**
3. Sélectionnez le fournisseur
4. Entrez la date de la facture fournisseur
5. Sélectionnez la catégorie de dépense (parmi les 18)
6. Entrez les montants :
   - Montant HT
   - TPS (calculée ou saisie)
   - TVQ (calculée ou saisie)
7. Associez au projet concerné (optionnel)
8. Téléchargez la pièce justificative (PDF/image)
9. Enregistrez
10. L'écriture comptable est générée automatiquement

### Consulter le Grand Livre

1. Onglet **"Grand Livre"**
2. Sélectionnez la période (dates début/fin)
3. Filtrez par type d'écriture :
   - Toutes
   - Revenus (factures)
   - Dépenses
4. Les écritures s'affichent avec :
   - Date
   - Référence document
   - Libellé
   - Montant TTC
   - Type (Revenu/Dépense)
5. Consultez les totaux :
   - Total revenus
   - Total dépenses
   - Résultat net
6. Exportez en CSV pour votre comptable

### Générer les états financiers

1. Onglet **"États Financiers"**
2. Sélectionnez le rapport désiré :
   - Bilan
   - État des résultats
   - Flux de trésorerie
   - Balance âgée
3. Définissez la période
4. Cliquez sur **"📊 Générer"**
5. Le rapport s'affiche
6. Exportez en PDF pour votre comptable

### Envoyer un rappel de facture

1. Trouvez une facture en retard
2. Cliquez sur **"📧 Envoyer rappel"**
3. Un courriel de relance est généré automatiquement
4. Le système inclut :
   - Numéro de facture
   - Montant dû
   - Date d'échéance dépassée
   - Lien vers le paiement (si configuré)
5. L'historique des relances est conservé

---

## Taxes Québec 2025

| Taxe | Taux | Application |
|------|------|-------------|
| **TPS** | 5.00% | Taxe fédérale sur les ventes |
| **TVQ** | 9.975% | Taxe de vente du Québec |
| **Total** | 14.975% | Taux combiné |

Numéros de taxes requis :
- **Numéro TPS** : 9 chiffres + RT0001
- **Numéro TVQ** : 10 chiffres + TQ0001

---

## Plan Comptable Construction

| Classe | Plage | Description |
|--------|-------|-------------|
| **1xxx** | 1000-1999 | Actifs (Caisse, Clients, Stock, TPS/TVQ à recevoir) |
| **2xxx** | 2000-2999 | Passifs (Fournisseurs, Emprunts, TPS/TVQ à payer) |
| **3xxx** | 3000-3999 | Capitaux propres (Capital, Bénéfices non répartis) |
| **4xxx** | 4000-4999 | Produits (Ventes services, Ventes produits) |
| **5xxx** | 5000-5999 | Charges d'exploitation (Achats, Salaires, etc.) |
| **6xxx** | 6000-6999 | Charges externes (Loyer, Assurances, etc.) |
| **7xxx** | 7000-7999 | Charges financières et exceptionnelles |

---

## Journaux Comptables

| Journal | Code | Usage |
|---------|------|-------|
| **Ventes** | VENTES | Factures clients |
| **Achats** | ACHATS | Factures fournisseurs |
| **Banque** | BANQUE | Mouvements bancaires, encaissements |
| **Paie** | PAIE | Écritures de salaires et charges |
| **Opérations diverses** | OD | Ajustements, régularisations |

---

## Paie (Québec 2025)

Conforme aux retenues à la source québécoises :

| Retenue | Description | Taux approximatif |
|---------|-------------|-------------------|
| **RRQ** | Régime des rentes du Québec | 6.4% (employé + employeur) |
| **RQAP** | Assurance parentale | 0.494% employé / 0.692% employeur |
| **AE** | Assurance-emploi | 1.78% employé / 2.49% employeur |
| **FSS** | Fonds des services de santé | Variable selon masse salariale |
| **Impôt QC** | Impôt provincial | Selon tables |
| **Impôt CA** | Impôt fédéral | Selon tables |

---

## Astuces et Bonnes Pratiques

- **Facturez rapidement** : Dès la livraison ou l'avancement des travaux
- **Relancez régulièrement** : Factures de plus de 30 jours
- **Catégorisez précisément** : Pour des rapports financiers utiles
- **Réconciliez mensuellement** : Comparez la banque avec le Grand Livre
- **Archivez les pièces** : 7 ans minimum au Québec (exigence fiscale)
- **Numérisez tout** : Gardez des copies numériques des factures
- **Vérifiez les écritures** : Balance débit/crédit chaque mois
- **Séparez par projet** : Pour connaître la rentabilité de chaque chantier

---

## Résolution de Problèmes

### La facture n'apparaît pas dans le Grand Livre

- **Cause** : Facture encore en brouillon
- **Solution** : Les écritures sont générées à l'envoi (statut ENVOYEE)

### Les taxes ne sont pas calculées correctement

- **Cause** : Montant HT à zéro ou erreur de saisie
- **Solution** : Vérifiez le montant hors taxes de chaque ligne

### L'écriture comptable n'est pas générée

- **Cause** : Erreur lors de la sauvegarde
- **Solution** : Rafraîchissez et vérifiez le journal_entry_id

### La balance âgée ne correspond pas

- **Cause** : Paiements non enregistrés
- **Solution** : Mettez à jour les paiements reçus sur chaque facture

---

## Questions Fréquentes (FAQ)

**Q: Le système est-il conforme pour Revenu Québec ?**
R: Oui, les calculs de taxes (TPS/TVQ) et de paie respectent la réglementation québécoise 2025.

**Q: Puis-je importer mes données de QuickBooks ?**
R: Une fonction d'import CSV est disponible pour les données de base (clients, fournisseurs, plan comptable).

**Q: Les relevés T4 et Relevé 1 sont-ils générés ?**
R: Oui, en fin d'année, le module Paie génère les documents fiscaux requis pour les employés.

**Q: Comment gérer plusieurs comptes bancaires ?**
R: Créez un compte comptable distinct (1xxx) par compte bancaire et réconciliez-les séparément.

**Q: Les paiements partiels sont-ils supportés ?**
R: Oui, vous pouvez enregistrer plusieurs paiements sur une même facture jusqu'au solde complet.

**Q: Comment créer une facture depuis un devis ?**
R: Dans le module Devis, utilisez le bouton "Convertir en facture" sur un devis accepté.

---

## Données Techniques

### Requête Factures avec Statistiques

```sql
SELECT f.*,
       c.nom as client_nom,
       p.nom_projet as projet_nom,
       (f.montant_ttc - f.montant_paye) as solde_du
FROM factures f
LEFT JOIN companies c ON f.company_id = c.id
LEFT JOIN projects p ON f.project_id = p.id
WHERE f.statut != 'ANNULEE'
ORDER BY f.date_facture DESC
```

### Requête Grand Livre par Période

```sql
SELECT date_ecriture, reference, libelle, montant_ttc, type
FROM journal_entries
WHERE date_ecriture BETWEEN :date_debut AND :date_fin
ORDER BY date_ecriture DESC
```

### Requête Balance Âgée Clients

```sql
SELECT c.nom,
       SUM(CASE WHEN age <= 30 THEN solde ELSE 0 END) as "0-30",
       SUM(CASE WHEN age BETWEEN 31 AND 60 THEN solde ELSE 0 END) as "31-60",
       SUM(CASE WHEN age BETWEEN 61 AND 90 THEN solde ELSE 0 END) as "61-90",
       SUM(CASE WHEN age > 90 THEN solde ELSE 0 END) as "90+"
FROM (
    SELECT company_id,
           (montant_ttc - montant_paye) as solde,
           CURRENT_DATE - date_echeance as age
    FROM factures
    WHERE statut NOT IN ('PAYEE', 'ANNULEE')
) f
JOIN companies c ON f.company_id = c.id
GROUP BY c.nom
```

---

## Voir Aussi

- [🧾 Devis](06-devis.md) - Conversion devis → facture
- [🏪 Achats](16-achats.md) - Factures fournisseurs
- [👥 Employés](12-employes.md) - Données paie
- [⏱️ TimeTracker](13-timetracker.md) - Heures pour paie
- [📊 Analytics BI](02-analytics-bi.md) - Analyses financières
- [🏢 Fonds Prévoyance](20-fonds-prevoyance.md) - Études financières copropriété
