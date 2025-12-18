# Constructo AI - Documentation API REST

## Table des matières

1. [Introduction](#introduction)
2. [Démarrage rapide](#démarrage-rapide)
3. [Authentification](#authentification)
4. [Rate Limiting](#rate-limiting)
5. [Référence des Endpoints](#référence-des-endpoints)
6. [Webhooks](#webhooks)
7. [Intégration QuickBooks](#intégration-quickbooks)
8. [Intégration Sage](#intégration-sage)
9. [Intégration Acomba](#intégration-acomba)
10. [Autres Intégrations Comptables](#autres-intégrations-comptables)
11. [Exemples de Code](#exemples-de-code)
12. [Erreurs et Dépannage](#erreurs-et-dépannage)

---

## Introduction

L'API REST Constructo AI permet d'intégrer votre ERP de construction avec des applications tierces telles que:

- **Logiciels comptables**: QuickBooks, Sage, Acomba
- **Outils d'automatisation**: n8n, Zapier, Make (Integromat)
- **Applications mobiles**: Accès aux données terrain
- **Systèmes de gestion de projet**: Procore, Buildertrend, CoConstruct

### Caractéristiques principales

| Caractéristique | Description |
|-----------------|-------------|
| **Version API** | 2.0.0 |
| **Format** | JSON (UTF-8) |
| **Authentification** | Clé API (X-API-Key) |
| **Rate Limiting** | 1000 requêtes/heure par défaut |
| **Protocole** | HTTPS uniquement |
| **Base URL** | `https://votre-instance.constructo.ai/api/v1` |

### Taxes québécoises

L'API calcule automatiquement les taxes québécoises:
- **TPS**: 5%
- **TVQ**: 9.975%

---

## Démarrage rapide

### 1. Obtenir une clé API

Connectez-vous à votre compte Constructo AI et accédez à:
**Paramètres → Intégrations → Clés API → Créer une nouvelle clé**

### 2. Tester la connexion

```bash
curl -X GET "https://votre-instance.constructo.ai/api/v1/health" \
  -H "X-API-Key: cai_live_votre_cle_api_ici"
```

### 3. Récupérer vos premières données

```bash
# Liste des clients
curl -X GET "https://votre-instance.constructo.ai/api/v1/companies?type_company=CLIENT" \
  -H "X-API-Key: cai_live_votre_cle_api_ici"

# Liste des factures
curl -X GET "https://votre-instance.constructo.ai/api/v1/invoices" \
  -H "X-API-Key: cai_live_votre_cle_api_ici"
```

---

## Authentification

### Format de la clé API

Les clés API Constructo AI suivent le format:

```
cai_[environnement]_[secret]
```

| Préfixe | Description |
|---------|-------------|
| `cai_live_` | Clé de production |
| `cai_test_` | Clé de test (sandbox) |

### Utilisation du header

Toutes les requêtes authentifiées doivent inclure le header `X-API-Key`:

```http
GET /api/v1/companies HTTP/1.1
Host: votre-instance.constructo.ai
X-API-Key: cai_live_abc123xyz789...
Content-Type: application/json
```

### Présets de permissions

| Préset | Permissions |
|--------|-------------|
| **read_only** | Lecture seule sur toutes les ressources |
| **read_write** | Lecture et écriture sur les données métier |
| **full_access** | Accès complet incluant gestion webhooks et clés API |

### Permissions disponibles

**Lecture (read)**:
- `companies:read` - Clients et fournisseurs
- `projects:read` - Projets
- `invoices:read` - Factures
- `products:read` - Produits
- `inventory:read` - Inventaire
- `quotes:read` - Devis
- `employees:read` - Employés
- `timetracking:read` - Feuilles de temps
- `payments:read` - Paiements

**Écriture (write)**:
- `companies:write` - Création/modification clients
- `projects:write` - Création/modification projets
- `invoices:write` - Création/modification factures
- `products:write` - Création/modification produits
- `inventory:write` - Ajustements inventaire
- `quotes:write` - Création/modification devis
- `payments:write` - Enregistrement paiements

**Administration**:
- `webhooks:manage` - Gestion des webhooks
- `api_keys:manage` - Gestion des clés API

### Sécurité

- Les clés sont hashées avec **bcrypt** (12 rounds)
- Seul le préfixe est stocké en clair pour recherche
- La clé complète n'est affichée qu'**une seule fois** à la création
- Expiration configurable (optionnel)

---

## Rate Limiting

### Limites par défaut

| Niveau | Limite | Période |
|--------|--------|---------|
| Standard | 1000 requêtes | Par heure |
| Premium | 5000 requêtes | Par heure |
| Enterprise | Illimité | - |

### Headers de réponse

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1640000000
```

### Réponse en cas de dépassement

```json
{
  "error": "rate_limit_exceeded",
  "message": "Limite de requêtes dépassée. Réessayez dans 342 secondes.",
  "retry_after": 342
}
```

**Code HTTP**: `429 Too Many Requests`

### Bonnes pratiques

1. **Cachez les données** qui changent peu (clients, produits)
2. **Utilisez les webhooks** au lieu de polling
3. **Implémentez un backoff exponentiel** en cas d'erreur 429
4. **Regroupez les requêtes** quand possible

---

## Référence des Endpoints

### Base URL

```
https://votre-instance.constructo.ai/api/v1
```

### Format des réponses

Toutes les réponses suivent ce format:

```json
{
  "success": true,
  "timestamp": "2024-01-15T10:30:00.000Z",
  "data": { ... }
}
```

---

### Health Check

#### GET /
Vérifie que l'API est opérationnelle (public, sans authentification).

```bash
curl https://votre-instance.constructo.ai/
```

**Réponse**:
```json
{
  "status": "healthy",
  "service": "Constructo AI API",
  "version": "2.0.0",
  "database": {
    "status": "connected",
    "latency_ms": 2.5
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

#### GET /api/v1/health
Santé détaillée (authentifié).

**Permission requise**: Aucune (authentification seulement)

---

### Companies (Clients/Fournisseurs)

#### GET /api/v1/companies
Liste des entreprises.

**Permission requise**: `companies:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `type_company` | string | `CLIENT` ou `FOURNISSEUR` |
| `active` | boolean | Filtrer par statut actif (défaut: true) |
| `search` | string | Recherche par nom ou email |
| `limit` | integer | Nombre de résultats (1-1000, défaut: 100) |
| `offset` | integer | Décalage pour pagination |

**Exemple**:
```bash
curl -X GET "https://api.constructo.ai/api/v1/companies?type_company=CLIENT&limit=50" \
  -H "X-API-Key: cai_live_xxx"
```

**Réponse**:
```json
{
  "success": true,
  "count": 50,
  "total": 127,
  "limit": 50,
  "offset": 0,
  "companies": [
    {
      "id": 1,
      "nom": "Construction ABC Inc.",
      "email": "info@constructionabc.ca",
      "telephone": "514-555-1234",
      "adresse": "123 Rue Principale",
      "ville": "Montréal",
      "province": "QC",
      "code_postal": "H2X 1Y4",
      "type_company": "CLIENT",
      "active": true,
      "created_at": "2024-01-10T08:00:00.000Z"
    }
  ]
}
```

#### GET /api/v1/companies/{id}
Détails d'une entreprise avec ses contacts.

**Permission requise**: `companies:read`

#### POST /api/v1/companies
Créer une entreprise.

**Permission requise**: `companies:write`

**Corps de la requête**:
```json
{
  "nom": "Nouveau Client Inc.",
  "email": "contact@nouveauclient.ca",
  "telephone": "450-555-9876",
  "adresse": "456 Avenue des Érables",
  "ville": "Laval",
  "province": "QC",
  "code_postal": "H7N 2T3",
  "type_company": "CLIENT",
  "notes": "Client référé par Construction XYZ"
}
```

#### PUT /api/v1/companies/{id}
Modifier une entreprise.

**Permission requise**: `companies:write`

---

### Invoices (Factures)

#### GET /api/v1/invoices
Liste des factures.

**Permission requise**: `invoices:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `statut` | string | `BROUILLON`, `ENVOYEE`, `PAYEE`, `PARTIELLEMENT_PAYEE`, `ANNULEE` |
| `client_id` | integer | Filtrer par client |
| `date_from` | date | Date de début (YYYY-MM-DD) |
| `date_to` | date | Date de fin (YYYY-MM-DD) |
| `limit` | integer | Nombre de résultats |
| `offset` | integer | Décalage pour pagination |

**Exemple**:
```bash
curl -X GET "https://api.constructo.ai/api/v1/invoices?statut=ENVOYEE&date_from=2024-01-01" \
  -H "X-API-Key: cai_live_xxx"
```

#### GET /api/v1/invoices/{id}
Détails d'une facture avec lignes et paiements.

**Réponse**:
```json
{
  "success": true,
  "invoice": {
    "id": 123,
    "numero": "FAC-202401-0001",
    "client_company_id": 45,
    "project_id": 12,
    "date_facture": "2024-01-15",
    "date_echeance": "2024-02-14",
    "conditions_paiement": "Net 30",
    "montant_ht": 10000.00,
    "tps": 500.00,
    "tvq": 997.50,
    "montant_ttc": 11497.50,
    "montant_paye": 5000.00,
    "solde_du": 6497.50,
    "statut": "PARTIELLEMENT_PAYEE",
    "lignes": [
      {
        "id": 1,
        "sequence_ligne": 1,
        "description": "Main d'oeuvre - Installation électrique",
        "quantite": 40.0,
        "unite": "heure",
        "prix_unitaire": 85.00,
        "montant_ligne": 3400.00,
        "produit_id": null
      },
      {
        "id": 2,
        "sequence_ligne": 2,
        "description": "Matériaux électriques",
        "quantite": 1.0,
        "unite": "lot",
        "prix_unitaire": 6600.00,
        "montant_ligne": 6600.00,
        "produit_id": 78
      }
    ],
    "paiements": [
      {
        "id": 1,
        "montant": 5000.00,
        "date_paiement": "2024-01-20",
        "methode_paiement": "Virement",
        "reference": "VIR-2024-001"
      }
    ]
  }
}
```

#### POST /api/v1/invoices
Créer une facture.

**Permission requise**: `invoices:write`

**Corps de la requête**:
```json
{
  "client_company_id": 45,
  "project_id": 12,
  "date_facture": "2024-01-15",
  "date_echeance": "2024-02-14",
  "conditions_paiement": "Net 30",
  "notes": "Travaux phase 1 complétés",
  "lignes": [
    {
      "description": "Main d'oeuvre - Plomberie",
      "quantite": 24,
      "unite": "heure",
      "prix_unitaire": 75.00
    },
    {
      "description": "Tuyauterie cuivre 3/4\"",
      "quantite": 50,
      "unite": "pied",
      "prix_unitaire": 12.50,
      "produit_id": 156
    }
  ]
}
```

**Réponse** (201 Created):
```json
{
  "success": true,
  "invoice": {
    "id": 124,
    "numero": "FAC-202401-0002",
    "montant_ht": 2425.00,
    "montant_ttc": 2788.96,
    "statut": "BROUILLON"
  }
}
```

#### POST /api/v1/invoices/{id}/payments
Enregistrer un paiement.

**Permission requise**: `payments:write`

**Corps de la requête**:
```json
{
  "montant": 2788.96,
  "date_paiement": "2024-01-25",
  "methode_paiement": "Chèque",
  "reference": "CHQ-45678",
  "notes": "Paiement complet reçu"
}
```

#### GET /api/v1/invoices/{id}/payments
Historique des paiements d'une facture.

**Permission requise**: `payments:read`

---

### Projects (Projets)

#### GET /api/v1/projects
Liste des projets.

**Permission requise**: `projects:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `statut` | string | `À FAIRE`, `EN COURS`, `TERMINÉ`, `EN ATTENTE`, `ANNULÉ` |
| `client_id` | integer | Filtrer par client |
| `limit` | integer | Nombre de résultats |
| `offset` | integer | Décalage |

#### GET /api/v1/projects/{id}
Détails d'un projet avec ses opérations.

#### POST /api/v1/projects
Créer un projet.

**Permission requise**: `projects:write`

```json
{
  "nom_projet": "Rénovation cuisine - 123 rue des Pins",
  "client_company_id": 45,
  "statut": "À FAIRE",
  "priorite": "HAUTE",
  "date_soumis": "2024-01-10",
  "date_prevu": "2024-03-15",
  "prix_estime": 35000.00,
  "description": "Rénovation complète incluant armoires, comptoirs et électroménagers"
}
```

#### PUT /api/v1/projects/{id}
Modifier un projet.

---

### Products (Produits)

#### GET /api/v1/products
Liste des produits.

**Permission requise**: `products:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `categorie` | string | Filtrer par catégorie |
| `search` | string | Recherche par nom ou code produit |
| `limit` | integer | Nombre de résultats |
| `offset` | integer | Décalage |

#### GET /api/v1/products/{id}
Détails d'un produit.

#### PUT /api/v1/products/{id}/stock
Ajuster le stock d'un produit.

**Permission requise**: `inventory:write`

```json
{
  "quantite": -5,
  "type_mouvement": "SORTIE",
  "notes": "Utilisé sur projet #123"
}
```

---

### Inventory (Inventaire)

#### GET /api/v1/inventory
État de l'inventaire.

**Permission requise**: `inventory:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `statut` | string | `DISPONIBLE`, `FAIBLE`, `CRITIQUE`, `ÉPUISÉ` |
| `limit` | integer | Nombre de résultats |

#### GET /api/v1/inventory/stats
Statistiques globales de l'inventaire.

**Réponse**:
```json
{
  "success": true,
  "stats": {
    "total_items": 450,
    "disponible": 380,
    "faible": 45,
    "critique": 25
  }
}
```

---

### Quotes (Devis)

#### GET /api/v1/quotes
Liste des devis.

**Permission requise**: `quotes:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `statut` | string | `BROUILLON`, `ENVOYÉ`, `APPROUVÉ`, `REFUSÉ`, `EXPIRÉ` |
| `limit` | integer | Nombre de résultats |
| `offset` | integer | Décalage |

#### GET /api/v1/quotes/{id}
Détails d'un devis.

---

### Employees (Employés)

#### GET /api/v1/employees
Liste des employés actifs.

**Permission requise**: `employees:read`

**Paramètres query**:
| Paramètre | Type | Description |
|-----------|------|-------------|
| `departement` | string | Filtrer par département |
| `limit` | integer | Nombre de résultats |

**Réponse**:
```json
{
  "success": true,
  "count": 25,
  "employees": [
    {
      "id": 1,
      "prenom": "Jean",
      "nom": "Tremblay",
      "email": "jean.tremblay@constructo.ca",
      "telephone": "514-555-1111",
      "poste": "Chef de chantier",
      "departement": "Construction"
    }
  ]
}
```

---

### Dashboard Stats

#### GET /api/v1/stats/dashboard
Statistiques pour tableau de bord.

**Réponse**:
```json
{
  "success": true,
  "stats": {
    "timestamp": "2024-01-15T10:30:00.000Z",
    "projects_total": 45,
    "projects_en_cours": 12,
    "factures_en_attente": 8,
    "solde_total_du": 125750.50,
    "items_critiques": 5
  }
}
```

---

### Webhooks

#### GET /api/v1/webhooks
Liste des webhooks configurés.

**Permission requise**: `webhooks:manage`

#### POST /api/v1/webhooks
Créer un webhook.

**Permission requise**: `webhooks:manage`

```json
{
  "url": "https://votre-app.com/webhook/constructo",
  "events": ["invoice.created", "invoice.paid", "payment.received"]
}
```

**Réponse**:
```json
{
  "success": true,
  "webhook": {
    "id": 5,
    "url": "https://votre-app.com/webhook/constructo",
    "events": ["invoice.created", "invoice.paid", "payment.received"],
    "secret": "whsec_abc123xyz..."
  }
}
```

> ⚠️ **Important**: Le `secret` n'est affiché qu'une seule fois. Conservez-le précieusement pour valider les signatures.

---

## Webhooks

### Événements disponibles

#### Factures
| Événement | Description |
|-----------|-------------|
| `invoice.created` | Nouvelle facture créée |
| `invoice.updated` | Facture modifiée |
| `invoice.sent` | Facture envoyée au client |
| `invoice.paid` | Facture entièrement payée |
| `invoice.overdue` | Facture en retard de paiement |
| `invoice.cancelled` | Facture annulée |

#### Paiements
| Événement | Description |
|-----------|-------------|
| `payment.received` | Paiement reçu |
| `payment.refunded` | Remboursement effectué |

#### Projets
| Événement | Description |
|-----------|-------------|
| `project.created` | Nouveau projet créé |
| `project.updated` | Projet modifié |
| `project.status_changed` | Changement de statut |
| `project.completed` | Projet terminé |

#### Devis
| Événement | Description |
|-----------|-------------|
| `quote.created` | Nouveau devis créé |
| `quote.sent` | Devis envoyé |
| `quote.approved` | Devis approuvé par le client |
| `quote.rejected` | Devis refusé |
| `quote.expired` | Devis expiré |

#### Inventaire
| Événement | Description |
|-----------|-------------|
| `inventory.low_stock` | Stock bas (alerte) |
| `inventory.out_of_stock` | Rupture de stock |
| `inventory.adjusted` | Ajustement de stock |

#### Entreprises
| Événement | Description |
|-----------|-------------|
| `company.created` | Nouveau client/fournisseur |
| `company.updated` | Client/fournisseur modifié |

#### Employés
| Événement | Description |
|-----------|-------------|
| `employee.created` | Nouvel employé |
| `time_entry.created` | Entrée de temps créée |

### Format des webhooks

```json
{
  "event": "invoice.paid",
  "created_at": "2024-01-15T10:30:00.000Z",
  "entreprise_id": 1,
  "api_version": "2.0",
  "data": {
    "id": 123,
    "numero": "FAC-202401-0001",
    "montant_ttc": 11497.50,
    "montant_paye": 11497.50,
    "statut": "PAYEE"
  }
}
```

### Headers des webhooks

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `User-Agent` | `ConstructoAI-Webhook/2.0` |
| `X-Webhook-Signature` | Signature HMAC-SHA256 |
| `X-Webhook-Event` | Type d'événement |
| `X-Webhook-Delivery` | ID unique de livraison |
| `X-Webhook-Attempt` | Numéro de tentative |

### Validation de signature

```python
import hmac
import hashlib

def verify_webhook_signature(payload: str, signature: str, secret: str) -> bool:
    """Vérifie la signature HMAC-SHA256 d'un webhook."""
    expected = 'sha256=' + hmac.new(
        secret.encode('utf-8'),
        payload.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)

# Utilisation
payload = request.body.decode('utf-8')
signature = request.headers.get('X-Webhook-Signature')
is_valid = verify_webhook_signature(payload, signature, 'whsec_votre_secret')
```

### Retry automatique

En cas d'échec (timeout ou code HTTP >= 400):
- **Tentative 1**: Immédiate
- **Tentative 2**: Après 1 minute
- **Tentative 3**: Après 5 minutes
- **Tentative 4**: Après 15 minutes

Après 10 échecs consécutifs, le webhook est automatiquement désactivé.

---

## Intégration QuickBooks

### QuickBooks Online

#### Prérequis
1. Compte QuickBooks Online (Essentials, Plus ou Advanced)
2. Application créée sur [developer.intuit.com](https://developer.intuit.com)
3. Clé API Constructo AI avec permissions `invoices:read`, `companies:read`, `payments:read`

#### Architecture de synchronisation

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Constructo AI  │ ──────► │   n8n / Zapier  │ ──────► │ QuickBooks      │
│     (Source)    │ Webhook │   (Middleware)  │  OAuth  │    Online       │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

#### Mapping des données

| Constructo AI | QuickBooks Online |
|---------------|-------------------|
| Company (CLIENT) | Customer |
| Company (FOURNISSEUR) | Vendor |
| Invoice | Invoice |
| Invoice.lignes | Invoice.Line |
| Payment | Payment |
| Product | Item (Service/NonInventory) |

#### Configuration avec n8n

**Workflow: Synchronisation des factures**

```json
{
  "name": "Constructo → QuickBooks - Factures",
  "nodes": [
    {
      "name": "Webhook Constructo",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "constructo-invoice",
        "responseMode": "onReceived"
      }
    },
    {
      "name": "Vérifier signature",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "const crypto = require('crypto');\nconst payload = JSON.stringify($input.all()[0].json);\nconst signature = $input.all()[0].headers['x-webhook-signature'];\nconst secret = 'whsec_votre_secret';\n\nconst expected = 'sha256=' + crypto.createHmac('sha256', secret).update(payload).digest('hex');\n\nif (signature !== expected) {\n  throw new Error('Signature invalide');\n}\n\nreturn $input.all();"
      }
    },
    {
      "name": "Créer facture QuickBooks",
      "type": "n8n-nodes-base.quickbooks",
      "parameters": {
        "operation": "create",
        "resource": "invoice",
        "additionalFields": {
          "CustomerRef": "={{ $json.data.client_company_id }}",
          "TxnDate": "={{ $json.data.date_facture }}",
          "DueDate": "={{ $json.data.date_echeance }}"
        }
      }
    }
  ]
}
```

#### Création de facture via API QuickBooks

```python
import requests
from datetime import datetime

def sync_invoice_to_quickbooks(constructo_invoice, qb_access_token, qb_company_id):
    """Synchronise une facture Constructo vers QuickBooks Online."""

    # Mapper les lignes de facture
    lines = []
    for idx, ligne in enumerate(constructo_invoice['lignes'], 1):
        lines.append({
            "DetailType": "SalesItemLineDetail",
            "Amount": ligne['montant_ligne'],
            "Description": ligne['description'],
            "SalesItemLineDetail": {
                "Qty": ligne['quantite'],
                "UnitPrice": ligne['prix_unitaire']
            }
        })

    # Ajouter les taxes (TPS + TVQ)
    lines.append({
        "DetailType": "SubTotalLineDetail",
        "Amount": constructo_invoice['montant_ht'],
        "SubTotalLineDetail": {}
    })

    qb_invoice = {
        "CustomerRef": {
            "value": str(constructo_invoice['qb_customer_id'])  # ID QuickBooks du client
        },
        "TxnDate": constructo_invoice['date_facture'],
        "DueDate": constructo_invoice['date_echeance'],
        "DocNumber": constructo_invoice['numero'],
        "Line": lines,
        "TxnTaxDetail": {
            "TotalTax": constructo_invoice['tps'] + constructo_invoice['tvq']
        }
    }

    response = requests.post(
        f"https://quickbooks.api.intuit.com/v3/company/{qb_company_id}/invoice",
        headers={
            "Authorization": f"Bearer {qb_access_token}",
            "Content-Type": "application/json",
            "Accept": "application/json"
        },
        json=qb_invoice
    )

    return response.json()
```

### QuickBooks Desktop

Pour QuickBooks Desktop (Pro, Premier, Enterprise), utilisez le **Web Connector** avec QBXML:

```xml
<?xml version="1.0" encoding="utf-8"?>
<?qbxml version="13.0"?>
<QBXML>
  <QBXMLMsgsRq onError="stopOnError">
    <InvoiceAddRq>
      <InvoiceAdd>
        <CustomerRef>
          <FullName>Construction ABC Inc.</FullName>
        </CustomerRef>
        <TxnDate>2024-01-15</TxnDate>
        <RefNumber>FAC-202401-0001</RefNumber>
        <InvoiceLineAdd>
          <ItemRef>
            <FullName>Services:Main d'oeuvre</FullName>
          </ItemRef>
          <Desc>Installation électrique</Desc>
          <Quantity>40</Quantity>
          <Rate>85.00</Rate>
        </InvoiceLineAdd>
      </InvoiceAdd>
    </InvoiceAddRq>
  </QBXMLMsgsRq>
</QBXML>
```

---

## Intégration Sage

### Sage 50 (Simply Accounting)

#### Configuration ODBC

Sage 50 utilise une base Pervasive/Actian. Configurez un DSN ODBC:

```
[Sage50DSN]
Driver=Pervasive ODBC Engine Interface
ServerName=localhost
DBQ=C:\Sage\Company.btr
```

#### Synchronisation des factures

```python
import pyodbc
import requests

# Connexion Sage 50
sage_conn = pyodbc.connect('DSN=Sage50DSN')
sage_cursor = sage_conn.cursor()

# Récupérer les factures de Constructo
response = requests.get(
    'https://api.constructo.ai/api/v1/invoices',
    headers={'X-API-Key': 'cai_live_xxx'},
    params={'statut': 'ENVOYEE', 'limit': 100}
)

for invoice in response.json()['invoices']:
    # Créer la facture dans Sage
    sage_cursor.execute('''
        INSERT INTO tSalesInvoice
        (nIDCustomer, szInvoiceNumber, dtInvoiceDate, dtDueDate,
         dSubtotal, dTax1Amount, dTax2Amount, dInvoiceTotal)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    ''', (
        get_sage_customer_id(invoice['client_company_id']),
        invoice['numero'],
        invoice['date_facture'],
        invoice['date_echeance'],
        invoice['montant_ht'],
        invoice['tps'],
        invoice['tvq'],
        invoice['montant_ttc']
    ))

sage_conn.commit()
```

### Sage 100 / 300

Pour Sage 100/300, utilisez l'**API REST Sage Business Cloud**:

```python
import requests

SAGE_BASE_URL = "https://api.accounting.sage.com/v3.1"

def create_sage_sales_invoice(constructo_invoice, sage_token):
    """Crée une facture dans Sage 100/300."""

    # Mapper le client
    contact_id = get_or_create_sage_contact(
        constructo_invoice['client_company_id'],
        sage_token
    )

    # Créer la facture
    sage_invoice = {
        "contact_id": contact_id,
        "date": constructo_invoice['date_facture'],
        "due_date": constructo_invoice['date_echeance'],
        "reference": constructo_invoice['numero'],
        "invoice_lines": [
            {
                "description": ligne['description'],
                "quantity": ligne['quantite'],
                "unit_price": ligne['prix_unitaire'],
                "tax_rate_id": get_sage_tax_rate_id()  # TPS+TVQ combiné
            }
            for ligne in constructo_invoice['lignes']
        ]
    }

    response = requests.post(
        f"{SAGE_BASE_URL}/sales_invoices",
        headers={
            "Authorization": f"Bearer {sage_token}",
            "Content-Type": "application/json"
        },
        json={"sales_invoice": sage_invoice}
    )

    return response.json()
```

### Configuration des taxes québécoises dans Sage

```python
# Configuration des taux de taxe pour le Québec
SAGE_TAX_RATES = {
    "TPS": {
        "name": "TPS - Taxe fédérale",
        "rate": 5.0,
        "tax_number": "123456789RT0001"  # Votre numéro TPS
    },
    "TVQ": {
        "name": "TVQ - Taxe provinciale",
        "rate": 9.975,
        "tax_number": "1234567890TQ0001"  # Votre numéro TVQ
    }
}
```

---

## Intégration Acomba

### À propos d'Acomba

Acomba est un logiciel comptable québécois très répandu dans l'industrie de la construction. L'intégration se fait via:
- **Acomba X**: API REST moderne
- **Acomba classique**: Import/Export fichiers ou ODBC

### Acomba X (API REST)

#### Configuration

```python
ACOMBA_CONFIG = {
    "base_url": "https://api.acomba.com/v1",
    "client_id": "votre_client_id",
    "client_secret": "votre_client_secret",
    "company_id": "votre_company_id"
}
```

#### Synchronisation bidirectionnelle

```python
import requests
from datetime import datetime

class AcombaIntegration:
    def __init__(self, config):
        self.config = config
        self.token = None
        self.token_expires = None

    def authenticate(self):
        """Obtient un token d'accès Acomba."""
        response = requests.post(
            f"{self.config['base_url']}/oauth/token",
            data={
                "grant_type": "client_credentials",
                "client_id": self.config['client_id'],
                "client_secret": self.config['client_secret']
            }
        )
        data = response.json()
        self.token = data['access_token']
        self.token_expires = datetime.now() + timedelta(seconds=data['expires_in'])
        return self.token

    def sync_customer(self, constructo_company):
        """Synchronise un client Constructo vers Acomba."""
        acomba_customer = {
            "code": f"CON-{constructo_company['id']:05d}",
            "name": constructo_company['nom'],
            "address": {
                "line1": constructo_company['adresse'],
                "city": constructo_company['ville'],
                "province": constructo_company['province'],
                "postalCode": constructo_company['code_postal']
            },
            "contact": {
                "email": constructo_company['email'],
                "phone": constructo_company['telephone']
            }
        }

        response = requests.post(
            f"{self.config['base_url']}/companies/{self.config['company_id']}/customers",
            headers={"Authorization": f"Bearer {self.token}"},
            json=acomba_customer
        )

        return response.json()

    def sync_invoice(self, constructo_invoice):
        """Synchronise une facture Constructo vers Acomba."""
        acomba_invoice = {
            "customerCode": f"CON-{constructo_invoice['client_company_id']:05d}",
            "invoiceNumber": constructo_invoice['numero'],
            "invoiceDate": constructo_invoice['date_facture'],
            "dueDate": constructo_invoice['date_echeance'],
            "lines": [
                {
                    "description": ligne['description'],
                    "quantity": ligne['quantite'],
                    "unitPrice": ligne['prix_unitaire'],
                    "amount": ligne['montant_ligne'],
                    "taxCodes": ["TPS", "TVQ"]  # Codes taxes Acomba
                }
                for ligne in constructo_invoice['lignes']
            ],
            "subtotal": constructo_invoice['montant_ht'],
            "taxes": [
                {"code": "TPS", "amount": constructo_invoice['tps']},
                {"code": "TVQ", "amount": constructo_invoice['tvq']}
            ],
            "total": constructo_invoice['montant_ttc']
        }

        response = requests.post(
            f"{self.config['base_url']}/companies/{self.config['company_id']}/invoices",
            headers={"Authorization": f"Bearer {self.token}"},
            json=acomba_invoice
        )

        return response.json()

    def sync_payment(self, constructo_payment, invoice_number):
        """Synchronise un paiement vers Acomba."""
        acomba_payment = {
            "invoiceNumber": invoice_number,
            "paymentDate": constructo_payment['date_paiement'],
            "amount": constructo_payment['montant'],
            "paymentMethod": self._map_payment_method(constructo_payment['methode_paiement']),
            "reference": constructo_payment['reference']
        }

        response = requests.post(
            f"{self.config['base_url']}/companies/{self.config['company_id']}/payments",
            headers={"Authorization": f"Bearer {self.token}"},
            json=acomba_payment
        )

        return response.json()

    def _map_payment_method(self, constructo_method):
        """Mappe les méthodes de paiement Constructo vers Acomba."""
        mapping = {
            "Virement": "EFT",
            "Chèque": "CHK",
            "Carte de crédit": "CC",
            "Espèces": "CSH",
            "Interac": "DBT"
        }
        return mapping.get(constructo_method, "OTH")
```

### Acomba Classique (Import fichier)

Pour les versions classiques d'Acomba, utilisez l'import par fichier CSV:

```python
import csv
from io import StringIO

def generate_acomba_import_file(invoices):
    """Génère un fichier d'import pour Acomba classique."""

    output = StringIO()
    writer = csv.writer(output, delimiter=';')

    # En-tête Acomba
    writer.writerow([
        'Type', 'No_Client', 'No_Facture', 'Date', 'Echeance',
        'Description', 'Montant_HT', 'TPS', 'TVQ', 'Total'
    ])

    for invoice in invoices:
        writer.writerow([
            'FAC',
            f"CON{invoice['client_company_id']:05d}",
            invoice['numero'],
            invoice['date_facture'],
            invoice['date_echeance'],
            f"Facture Constructo - Projet {invoice.get('project_id', 'N/A')}",
            invoice['montant_ht'],
            invoice['tps'],
            invoice['tvq'],
            invoice['montant_ttc']
        ])

    return output.getvalue()

# Utilisation
invoices = get_constructo_invoices()
csv_content = generate_acomba_import_file(invoices)

# Sauvegarder pour import
with open('/chemin/vers/import_acomba.csv', 'w', encoding='cp1252') as f:
    f.write(csv_content)
```

---

## Autres Intégrations Comptables

### Procore (Gestion de projet construction)

```python
import requests

PROCORE_CONFIG = {
    "base_url": "https://api.procore.com/rest/v1.0",
    "company_id": "votre_company_id"
}

def sync_project_to_procore(constructo_project, procore_token):
    """Synchronise un projet Constructo vers Procore."""

    procore_project = {
        "project": {
            "name": constructo_project['nom_projet'],
            "project_number": f"CON-{constructo_project['id']}",
            "address": constructo_project.get('adresse'),
            "city": constructo_project.get('ville'),
            "state_code": "QC",
            "zip": constructo_project.get('code_postal'),
            "country_code": "CA",
            "estimated_value": constructo_project['prix_estime'],
            "start_date": constructo_project.get('date_soumis'),
            "projected_finish_date": constructo_project.get('date_prevu'),
            "stage": map_status_to_procore(constructo_project['statut'])
        }
    }

    response = requests.post(
        f"{PROCORE_CONFIG['base_url']}/companies/{PROCORE_CONFIG['company_id']}/projects",
        headers={
            "Authorization": f"Bearer {procore_token}",
            "Content-Type": "application/json"
        },
        json=procore_project
    )

    return response.json()

def map_status_to_procore(constructo_status):
    """Mappe les statuts Constructo vers Procore."""
    mapping = {
        "À FAIRE": "Pre-Construction",
        "EN COURS": "Active",
        "TERMINÉ": "Complete",
        "EN ATTENTE": "On Hold",
        "ANNULÉ": "Abandoned"
    }
    return mapping.get(constructo_status, "Active")
```

### Buildertrend

```python
import requests

def sync_to_buildertrend(constructo_data, bt_api_key):
    """Synchronise les données vers Buildertrend."""

    headers = {
        "Authorization": f"Bearer {bt_api_key}",
        "Content-Type": "application/json"
    }

    # Sync clients
    for company in constructo_data['companies']:
        bt_customer = {
            "firstName": company['nom'].split()[0] if ' ' in company['nom'] else company['nom'],
            "lastName": ' '.join(company['nom'].split()[1:]) if ' ' in company['nom'] else '',
            "companyName": company['nom'],
            "email": company['email'],
            "phone": company['telephone'],
            "address": company['adresse'],
            "city": company['ville'],
            "state": company['province'],
            "zip": company['code_postal']
        }

        requests.post(
            "https://api.buildertrend.com/v1/customers",
            headers=headers,
            json=bt_customer
        )
```

### CoConstruct

```python
import requests

def sync_to_coconstruct(constructo_invoice, cc_api_key, cc_company_id):
    """Synchronise une facture vers CoConstruct."""

    cc_invoice = {
        "projectId": constructo_invoice.get('coconstruct_project_id'),
        "invoiceNumber": constructo_invoice['numero'],
        "date": constructo_invoice['date_facture'],
        "dueDate": constructo_invoice['date_echeance'],
        "subtotal": constructo_invoice['montant_ht'],
        "tax": constructo_invoice['tps'] + constructo_invoice['tvq'],
        "total": constructo_invoice['montant_ttc'],
        "lineItems": [
            {
                "description": ligne['description'],
                "quantity": ligne['quantite'],
                "rate": ligne['prix_unitaire'],
                "amount": ligne['montant_ligne']
            }
            for ligne in constructo_invoice['lignes']
        ]
    }

    response = requests.post(
        f"https://api.coconstruct.com/v1/companies/{cc_company_id}/invoices",
        headers={
            "Authorization": f"Bearer {cc_api_key}",
            "Content-Type": "application/json"
        },
        json=cc_invoice
    )

    return response.json()
```

### Xero

```python
from xero_python.api_client import ApiClient
from xero_python.api_client.configuration import Configuration
from xero_python.accounting import AccountingApi, Invoice, LineItem, Contact

def sync_invoice_to_xero(constructo_invoice, xero_token, xero_tenant_id):
    """Synchronise une facture vers Xero."""

    api_client = ApiClient(
        configuration=Configuration(
            oauth2_token=xero_token
        )
    )

    accounting_api = AccountingApi(api_client)

    # Créer les lignes de facture
    line_items = []
    for ligne in constructo_invoice['lignes']:
        line_items.append(LineItem(
            description=ligne['description'],
            quantity=ligne['quantite'],
            unit_amount=ligne['prix_unitaire'],
            tax_type="OUTPUT2"  # TPS + TVQ pour le Québec
        ))

    # Créer la facture
    invoice = Invoice(
        type="ACCREC",
        contact=Contact(contact_id=constructo_invoice['xero_contact_id']),
        line_items=line_items,
        date=constructo_invoice['date_facture'],
        due_date=constructo_invoice['date_echeance'],
        invoice_number=constructo_invoice['numero'],
        reference=f"Constructo-{constructo_invoice['id']}"
    )

    result = accounting_api.create_invoices(
        xero_tenant_id,
        invoices={"invoices": [invoice]}
    )

    return result
```

---

## Exemples de Code

### Python - Client API complet

```python
"""
Client Python pour l'API Constructo AI
"""

import requests
from typing import Optional, Dict, List, Any
from datetime import date

class ConstructoAPI:
    def __init__(self, api_key: str, base_url: str = "https://api.constructo.ai/api/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "X-API-Key": api_key,
            "Content-Type": "application/json"
        })

    def _request(self, method: str, endpoint: str, **kwargs) -> Dict[str, Any]:
        """Effectue une requête API."""
        url = f"{self.base_url}{endpoint}"
        response = self.session.request(method, url, **kwargs)
        response.raise_for_status()
        return response.json()

    # === Companies ===

    def get_companies(
        self,
        type_company: Optional[str] = None,
        search: Optional[str] = None,
        limit: int = 100,
        offset: int = 0
    ) -> Dict[str, Any]:
        """Liste les entreprises."""
        params = {"limit": limit, "offset": offset}
        if type_company:
            params["type_company"] = type_company
        if search:
            params["search"] = search
        return self._request("GET", "/companies", params=params)

    def get_company(self, company_id: int) -> Dict[str, Any]:
        """Détails d'une entreprise."""
        return self._request("GET", f"/companies/{company_id}")

    def create_company(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Crée une entreprise."""
        return self._request("POST", "/companies", json=data)

    # === Invoices ===

    def get_invoices(
        self,
        statut: Optional[str] = None,
        client_id: Optional[int] = None,
        date_from: Optional[date] = None,
        date_to: Optional[date] = None,
        limit: int = 100,
        offset: int = 0
    ) -> Dict[str, Any]:
        """Liste les factures."""
        params = {"limit": limit, "offset": offset}
        if statut:
            params["statut"] = statut
        if client_id:
            params["client_id"] = client_id
        if date_from:
            params["date_from"] = date_from.isoformat()
        if date_to:
            params["date_to"] = date_to.isoformat()
        return self._request("GET", "/invoices", params=params)

    def get_invoice(self, invoice_id: int) -> Dict[str, Any]:
        """Détails d'une facture."""
        return self._request("GET", f"/invoices/{invoice_id}")

    def create_invoice(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Crée une facture."""
        return self._request("POST", "/invoices", json=data)

    def add_payment(self, invoice_id: int, payment_data: Dict[str, Any]) -> Dict[str, Any]:
        """Ajoute un paiement à une facture."""
        return self._request("POST", f"/invoices/{invoice_id}/payments", json=payment_data)

    # === Projects ===

    def get_projects(
        self,
        statut: Optional[str] = None,
        client_id: Optional[int] = None,
        limit: int = 100,
        offset: int = 0
    ) -> Dict[str, Any]:
        """Liste les projets."""
        params = {"limit": limit, "offset": offset}
        if statut:
            params["statut"] = statut
        if client_id:
            params["client_id"] = client_id
        return self._request("GET", "/projects", params=params)

    def create_project(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Crée un projet."""
        return self._request("POST", "/projects", json=data)

    # === Webhooks ===

    def create_webhook(self, url: str, events: List[str]) -> Dict[str, Any]:
        """Crée un webhook."""
        return self._request("POST", "/webhooks", json={"url": url, "events": events})

    def get_webhooks(self) -> Dict[str, Any]:
        """Liste les webhooks."""
        return self._request("GET", "/webhooks")


# Utilisation
if __name__ == "__main__":
    api = ConstructoAPI("cai_live_votre_cle_api")

    # Lister les clients
    clients = api.get_companies(type_company="CLIENT")
    print(f"Nombre de clients: {clients['count']}")

    # Créer une facture
    nouvelle_facture = api.create_invoice({
        "client_company_id": 1,
        "lignes": [
            {
                "description": "Consultation",
                "quantite": 2,
                "unite": "heure",
                "prix_unitaire": 150.00
            }
        ]
    })
    print(f"Facture créée: {nouvelle_facture['invoice']['numero']}")
```

### JavaScript/Node.js

```javascript
/**
 * Client JavaScript pour l'API Constructo AI
 */

const axios = require('axios');

class ConstructoAPI {
    constructor(apiKey, baseUrl = 'https://api.constructo.ai/api/v1') {
        this.client = axios.create({
            baseURL: baseUrl,
            headers: {
                'X-API-Key': apiKey,
                'Content-Type': 'application/json'
            }
        });
    }

    // === Companies ===

    async getCompanies(params = {}) {
        const response = await this.client.get('/companies', { params });
        return response.data;
    }

    async getCompany(id) {
        const response = await this.client.get(`/companies/${id}`);
        return response.data;
    }

    async createCompany(data) {
        const response = await this.client.post('/companies', data);
        return response.data;
    }

    // === Invoices ===

    async getInvoices(params = {}) {
        const response = await this.client.get('/invoices', { params });
        return response.data;
    }

    async getInvoice(id) {
        const response = await this.client.get(`/invoices/${id}`);
        return response.data;
    }

    async createInvoice(data) {
        const response = await this.client.post('/invoices', data);
        return response.data;
    }

    async addPayment(invoiceId, paymentData) {
        const response = await this.client.post(`/invoices/${invoiceId}/payments`, paymentData);
        return response.data;
    }

    // === Webhooks ===

    async createWebhook(url, events) {
        const response = await this.client.post('/webhooks', { url, events });
        return response.data;
    }
}

// Serveur webhook Express
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

const WEBHOOK_SECRET = 'whsec_votre_secret';

function verifySignature(payload, signature) {
    const expected = 'sha256=' + crypto
        .createHmac('sha256', WEBHOOK_SECRET)
        .update(JSON.stringify(payload))
        .digest('hex');
    return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

app.post('/webhook/constructo', (req, res) => {
    const signature = req.headers['x-webhook-signature'];

    if (!verifySignature(req.body, signature)) {
        return res.status(401).json({ error: 'Invalid signature' });
    }

    const { event, data } = req.body;

    switch (event) {
        case 'invoice.created':
            console.log('Nouvelle facture:', data.numero);
            // Synchroniser avec votre système comptable
            break;
        case 'invoice.paid':
            console.log('Facture payée:', data.numero);
            break;
        case 'payment.received':
            console.log('Paiement reçu:', data.montant);
            break;
    }

    res.status(200).json({ received: true });
});

app.listen(3000, () => console.log('Webhook server running on port 3000'));

module.exports = ConstructoAPI;
```

### PHP

```php
<?php
/**
 * Client PHP pour l'API Constructo AI
 */

class ConstructoAPI {
    private $apiKey;
    private $baseUrl;

    public function __construct($apiKey, $baseUrl = 'https://api.constructo.ai/api/v1') {
        $this->apiKey = $apiKey;
        $this->baseUrl = $baseUrl;
    }

    private function request($method, $endpoint, $data = null, $params = []) {
        $url = $this->baseUrl . $endpoint;

        if (!empty($params)) {
            $url .= '?' . http_build_query($params);
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'X-API-Key: ' . $this->apiKey,
                'Content-Type: application/json'
            ]
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new Exception("API Error: HTTP $httpCode - $response");
        }

        return json_decode($response, true);
    }

    // Companies
    public function getCompanies($params = []) {
        return $this->request('GET', '/companies', null, $params);
    }

    public function createCompany($data) {
        return $this->request('POST', '/companies', $data);
    }

    // Invoices
    public function getInvoices($params = []) {
        return $this->request('GET', '/invoices', null, $params);
    }

    public function createInvoice($data) {
        return $this->request('POST', '/invoices', $data);
    }

    public function addPayment($invoiceId, $data) {
        return $this->request('POST', "/invoices/$invoiceId/payments", $data);
    }
}

// Vérification signature webhook
function verifyWebhookSignature($payload, $signature, $secret) {
    $expected = 'sha256=' . hash_hmac('sha256', $payload, $secret);
    return hash_equals($expected, $signature);
}

// Handler webhook
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';

if (verifyWebhookSignature($payload, $signature, 'whsec_votre_secret')) {
    $data = json_decode($payload, true);

    switch ($data['event']) {
        case 'invoice.created':
            // Traiter nouvelle facture
            break;
        case 'invoice.paid':
            // Traiter facture payée
            break;
    }

    http_response_code(200);
    echo json_encode(['received' => true]);
} else {
    http_response_code(401);
    echo json_encode(['error' => 'Invalid signature']);
}
```

### C# (.NET)

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Security.Cryptography;
using System.Threading.Tasks;

public class ConstructoAPI
{
    private readonly HttpClient _client;
    private readonly string _baseUrl;

    public ConstructoAPI(string apiKey, string baseUrl = "https://api.constructo.ai/api/v1")
    {
        _baseUrl = baseUrl;
        _client = new HttpClient();
        _client.DefaultRequestHeaders.Add("X-API-Key", apiKey);
    }

    public async Task<JsonDocument> GetCompaniesAsync(string typeCompany = null, int limit = 100)
    {
        var url = $"{_baseUrl}/companies?limit={limit}";
        if (!string.IsNullOrEmpty(typeCompany))
            url += $"&type_company={typeCompany}";

        var response = await _client.GetAsync(url);
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        return JsonDocument.Parse(content);
    }

    public async Task<JsonDocument> CreateInvoiceAsync(object invoiceData)
    {
        var json = JsonSerializer.Serialize(invoiceData);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _client.PostAsync($"{_baseUrl}/invoices", content);
        response.EnsureSuccessStatusCode();

        var responseContent = await response.Content.ReadAsStringAsync();
        return JsonDocument.Parse(responseContent);
    }

    // Vérification signature webhook
    public static bool VerifyWebhookSignature(string payload, string signature, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
        var expected = "sha256=" + BitConverter.ToString(hash).Replace("-", "").ToLower();
        return signature == expected;
    }
}

// Exemple ASP.NET Core Controller pour webhooks
[ApiController]
[Route("webhook")]
public class WebhookController : ControllerBase
{
    private const string WebhookSecret = "whsec_votre_secret";

    [HttpPost("constructo")]
    public IActionResult HandleWebhook()
    {
        using var reader = new StreamReader(Request.Body);
        var payload = reader.ReadToEndAsync().Result;
        var signature = Request.Headers["X-Webhook-Signature"].ToString();

        if (!ConstructoAPI.VerifyWebhookSignature(payload, signature, WebhookSecret))
        {
            return Unauthorized(new { error = "Invalid signature" });
        }

        var data = JsonDocument.Parse(payload);
        var eventType = data.RootElement.GetProperty("event").GetString();

        switch (eventType)
        {
            case "invoice.created":
                // Traiter nouvelle facture
                break;
            case "invoice.paid":
                // Traiter facture payée
                break;
        }

        return Ok(new { received = true });
    }
}
```

---

## Erreurs et Dépannage

### Codes d'erreur HTTP

| Code | Description | Action recommandée |
|------|-------------|-------------------|
| 400 | Bad Request | Vérifiez le format de votre requête |
| 401 | Unauthorized | Vérifiez votre clé API |
| 403 | Forbidden | Permission insuffisante |
| 404 | Not Found | Ressource inexistante |
| 422 | Unprocessable Entity | Données invalides |
| 429 | Too Many Requests | Attendez avant de réessayer |
| 500 | Internal Server Error | Contactez le support |

### Format des erreurs

```json
{
  "success": false,
  "error": {
    "code": "INVALID_API_KEY",
    "message": "La clé API fournie est invalide ou expirée",
    "details": {
      "key_prefix": "cai_live_abc..."
    }
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Erreurs courantes

#### Clé API invalide
```json
{
  "error": {
    "code": "INVALID_API_KEY",
    "message": "La clé API fournie est invalide"
  }
}
```
**Solution**: Vérifiez que vous utilisez la clé complète et qu'elle n'a pas été révoquée.

#### Permission refusée
```json
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "Permission 'invoices:write' requise"
  }
}
```
**Solution**: Mettez à jour les permissions de votre clé API ou utilisez une clé avec les permissions appropriées.

#### Rate limit dépassé
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Limite de 1000 requêtes/heure dépassée",
    "retry_after": 1847
  }
}
```
**Solution**: Attendez le délai indiqué ou implémentez un cache.

#### Données invalides
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Erreur de validation",
    "details": {
      "client_company_id": "Ce champ est requis",
      "lignes": "Au moins une ligne est requise"
    }
  }
}
```
**Solution**: Corrigez les champs indiqués dans les détails.

### Logs et monitoring

Pour déboguer vos intégrations:

1. **Activez le mode verbose** dans votre client HTTP
2. **Consultez les logs** dans Constructo AI: Paramètres → Intégrations → Logs API
3. **Vérifiez les livraisons webhook**: Paramètres → Webhooks → Historique

### Support

- **Documentation**: https://docs.constructo.ai
- **Email**: support@constructo.ai
- **Statut API**: https://status.constructo.ai

---

## Annexe: Mapping des champs

### Constructo AI → QuickBooks

| Constructo AI | QuickBooks Online | QuickBooks Desktop |
|---------------|-------------------|-------------------|
| `companies.nom` | `Customer.DisplayName` | `Customer.Name` |
| `companies.email` | `Customer.PrimaryEmailAddr` | `Customer.Email` |
| `factures.numero` | `Invoice.DocNumber` | `Invoice.RefNumber` |
| `factures.date_facture` | `Invoice.TxnDate` | `Invoice.TxnDate` |
| `factures.montant_ttc` | `Invoice.TotalAmt` | `Invoice.Amount` |
| `facture_lignes.description` | `Line.Description` | `InvoiceLine.Desc` |
| `facture_lignes.quantite` | `Line.Qty` | `InvoiceLine.Quantity` |
| `facture_lignes.prix_unitaire` | `Line.UnitPrice` | `InvoiceLine.Rate` |

### Constructo AI → Sage

| Constructo AI | Sage 50 | Sage 100/300 |
|---------------|---------|--------------|
| `companies.nom` | `tCustomer.szCustomerName` | `contact.name` |
| `factures.numero` | `tSalesInvoice.szInvoiceNumber` | `sales_invoice.reference` |
| `factures.tps` | `tSalesInvoice.dTax1Amount` | `sales_invoice.tax_amount` |
| `factures.tvq` | `tSalesInvoice.dTax2Amount` | (inclus dans tax_amount) |

### Constructo AI → Acomba

| Constructo AI | Acomba X | Acomba Classique |
|---------------|----------|------------------|
| `companies.nom` | `customer.name` | `Client.Nom` |
| `factures.numero` | `invoice.invoiceNumber` | `Facture.NoFacture` |
| `factures.tps` | `taxes[TPS].amount` | `Facture.TPS` |
| `factures.tvq` | `taxes[TVQ].amount` | `Facture.TVQ` |

---

*Documentation générée pour Constructo AI API v2.0.0*
*Dernière mise à jour: Décembre 2024*
