# 💳 Abonnement

## Introduction

Le module **Abonnement** gère votre forfait CONSTRUCTO AI. Consultez les détails de votre plan, gérez vos informations de paiement via Stripe, suivez votre utilisation et modifiez votre abonnement selon vos besoins.

Ce module s'intègre avec Stripe pour le traitement sécurisé des paiements et la gestion automatisée des abonnements récurrents.

---

## Accès au Module

1. Dans le menu latéral, cliquez sur **"💳 Abonnement"**
2. Les détails de votre abonnement s'affichent :
   - **Statut actuel** : État de l'abonnement
   - **Détails du plan** : Prix et renouvellement
   - **Historique** : Factures passées
3. Gérez votre plan et vos paiements

---

## Structure des Données

### Table PostgreSQL : `subscriptions`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Identifiant unique |
| `company_id` | INTEGER | ID de l'entreprise |
| `stripe_customer_id` | TEXT | ID client Stripe |
| `stripe_subscription_id` | TEXT | ID abonnement Stripe |
| `status` | TEXT | Statut de l'abonnement |
| `plan_name` | TEXT | Nom du plan |
| `price_monthly` | DECIMAL | Prix mensuel |
| `current_period_start` | TIMESTAMP | Début période |
| `current_period_end` | TIMESTAMP | Fin période |
| `cancel_at_period_end` | BOOLEAN | Annulation programmée |
| `created_at` | TIMESTAMP | Date création |

---

## Statuts d'Abonnement

| Statut | Icône | Description |
|--------|-------|-------------|
| **active** | ✅ Actif | Abonnement en cours, paiements à jour |
| **trialing** | 🎁 Essai | Période d'essai gratuit en cours |
| **past_due** | ⚠️ Retard | Paiement en retard, accès maintenu temporairement |
| **canceled** | ❌ Annulé | Abonnement annulé, accès jusqu'à fin période |
| **incomplete** | 🔄 Incomplet | Paiement initial en attente |
| **unpaid** | 🚫 Impayé | Paiements échoués, accès suspendu |

---

## Fonctionnalités Principales

### 1. Détails du Plan

| Information | Description |
|-------------|-------------|
| **Plan actuel** | Standard |
| **Prix mensuel** | 139,99 $/mois (avant taxes) |
| **Date de renouvellement** | Prochaine facturation automatique |
| **Statut** | Actif, Essai, Annulé |
| **Utilisateurs inclus** | Illimité |

### 2. Période d'Essai

| Paramètre | Valeur |
|-----------|--------|
| **Durée** | 7 jours |
| **Carte requise** | Oui (validée, non débitée) |
| **Fonctionnalités** | Accès complet à tous les modules |
| **Conversion** | Automatique à la fin de l'essai |
| **Annulation** | Possible à tout moment sans frais |

### 3. Types d'Abonnement Entreprise

| Type | Description | Source |
|------|-------------|--------|
| **Client** | Abonnement Stripe actif | Paiement mensuel |
| **Testeur** | Période d'essai | Gratuit 7 jours |
| **Demo** | Compte démonstration | Limité dans le temps |

### 4. Méthodes de Paiement (Stripe)

Paiements sécurisés via Stripe :
- Carte de crédit (Visa, Mastercard, Amex, Discover)
- Carte de débit
- Facturation mensuelle automatique
- Renouvellement automatique

### 5. Historique de Facturation

| Information | Description |
|-------------|-------------|
| **Factures** | Liste mensuelle |
| **Téléchargement** | PDF disponible |
| **Détails** | Montant, taxes, date |
| **Statut** | Payé, En attente, Échoué |

---

## Guide Pas-à-Pas

### Consulter votre abonnement

1. Ouvrez le module **Abonnement**
2. Visualisez :
   - Plan actuel et prix
   - Date du prochain paiement
   - Statut de l'abonnement
   - Utilisation (utilisateurs, stockage)

### Mettre à jour les informations de paiement

1. Cliquez sur **"💳 Modifier le paiement"**
2. Vous êtes redirigé vers le portail Stripe sécurisé
3. Mettez à jour :
   - Numéro de carte
   - Date d'expiration
   - CVV
4. Confirmez les modifications
5. Retournez à CONSTRUCTO AI

### Télécharger une facture

1. Section **"Historique de facturation"**
2. Localisez la facture souhaitée
3. Cliquez sur **"📄 Télécharger PDF"**
4. La facture s'ouvre ou se télécharge
5. Utilisez-la pour votre comptabilité

### Annuler l'abonnement

1. Cliquez sur **"🚫 Annuler l'abonnement"**
2. Lisez les informations importantes :
   - Accès jusqu'à la fin de la période payée
   - Données conservées 30 jours
   - Possibilité de réactiver
3. Confirmez votre choix
4. L'abonnement ne se renouvellera pas

### Réactiver un abonnement annulé

1. Si votre abonnement est annulé mais pas expiré
2. Cliquez sur **"🔄 Réactiver"**
3. Confirmez que votre méthode de paiement est valide
4. L'abonnement reprend automatiquement

---

## Tarification

| Forfait | Prix | Inclus |
|---------|------|--------|
| **Standard** | 139,99 $/mois | Utilisateurs illimités, Tous les modules, Support |
| **Essai gratuit** | 0 $ (7 jours) | Accès complet |

### Ce qui est inclus

✅ Tous les modules ERP
✅ Assistant IA (Claude Opus 4.5)
✅ Multi-tenant (isolation des données)
✅ Mises à jour automatiques
✅ Support par messagerie
✅ Sauvegardes quotidiennes
✅ HTTPS sécurisé

---

## Utilisation et Limites

| Ressource | Limite |
|-----------|--------|
| **Utilisateurs** | Illimité |
| **Projets** | Illimité |
| **Stockage documents** | 10 Go inclus |
| **Tokens IA** | Selon utilisation |

---

## Astuces et Bonnes Pratiques

- **Vérifiez avant l'expiration** : Assurez-vous que votre carte est valide
- **Conservez vos factures** : Pour la comptabilité et les impôts
- **Surveillez l'utilisation** : Évitez les dépassements
- **Essai complet** : Testez toutes les fonctionnalités pendant l'essai
- **Contactez avant d'annuler** : Le support peut vous aider

---

## Questions Fréquentes (FAQ)

**Q: Que se passe-t-il si mon paiement échoue ?**
R: Stripe réessaie automatiquement plusieurs fois. Après échec répété, l'accès est suspendu jusqu'à régularisation.

**Q: Puis-je obtenir un remboursement ?**
R: Contactez le support dans les 7 jours suivant un paiement pour étudier votre demande.

**Q: Les prix incluent-ils les taxes ?**
R: Le prix affiché est avant taxes. TPS et TVQ s'appliquent pour les entreprises québécoises.

**Q: Puis-je changer de forfait ?**
R: Actuellement un seul forfait est disponible. Des options seront ajoutées à l'avenir.

**Q: Mes données sont-elles conservées si j'annule ?**
R: Vos données sont conservées 30 jours après expiration, puis supprimées définitivement.

---

## Fonctions Stripe Disponibles

| Fonction | Description |
|----------|-------------|
| `create_checkout_session` | Créer une session de paiement |
| `get_subscription_status` | Obtenir le statut de l'abonnement |
| `cancel_subscription` | Annuler l'abonnement |
| `reactivate_subscription` | Réactiver un abonnement annulé |
| `create_customer_portal_session` | Accéder au portail Stripe |
| `get_invoices` | Récupérer l'historique des factures |
| `check_subscription_active` | Vérifier si l'abonnement est actif |
| `get_subscription_days_remaining` | Jours restants avant expiration |

---

## Données Techniques

### Requête Statut Abonnement

```sql
SELECT s.*, e.nom as entreprise_nom
FROM subscriptions s
LEFT JOIN entreprises e ON s.company_id = e.id
WHERE s.company_id = :company_id
ORDER BY s.created_at DESC
LIMIT 1
```

### Requête Vérification Accès

```sql
SELECT
    CASE
        WHEN status IN ('active', 'trialing') THEN TRUE
        WHEN status = 'canceled' AND current_period_end > NOW() THEN TRUE
        ELSE FALSE
    END as has_access
FROM subscriptions
WHERE company_id = :company_id
```

### Variables d'Environnement Requises

```bash
STRIPE_SECRET_KEY=sk_live_xxxx...
STRIPE_PUBLISHABLE_KEY=pk_live_xxxx...
STRIPE_PRICE_ID=price_xxxx...
STRIPE_WEBHOOK_SECRET=whsec_xxxx...
```

---

## Support

Pour toute question sur votre abonnement :
- **Email** : support@constructoai.ca
- **Téléphone** : (514) 820-1972
- **Messagerie** : Dans l'application
- **Documentation** : Cette aide en ligne

---

## Voir Aussi

- [⚙️ Configuration](27-configuration.md) - Paramètres système
- [👥 Utilisateurs](26-utilisateurs.md) - Gestion des comptes
- [🏠 Tableau de Bord](01-tableau-de-bord.md) - Accueil
