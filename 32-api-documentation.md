# Constructo AI - Documentation API REST

**Version:** 2.0.0
**URL de base:** `https://constructo-api.onrender.com`
**Documentation interactive:** `https://constructo-api.onrender.com/docs`

---

## Table des matières

1. [Introduction](#introduction)
2. [Authentification](#authentification)
3. [Créer une clé API](#créer-une-clé-api)
4. [Rate Limiting](#rate-limiting)
5. [Endpoints disponibles](#endpoints-disponibles)
   - [Health](#health)
   - [Companies (Clients)](#companies-clients)
   - [Projects (Projets)](#projects-projets)
   - [Invoices (Factures)](#invoices-factures)
   - [Products (Produits)](#products-produits)
   - [Inventory (Inventaire)](#inventory-inventaire)
   - [Quotes (Devis)](#quotes-devis)
   - [Employees (Employés)](#employees-employés)
   - [Webhooks](#webhooks)
6. [Exemples d'intégration](#exemples-dintégration)
7. [Codes d'erreur](#codes-derreur)
8. [Support](#support)

---

## Introduction

L'API REST Constructo AI permet d'intégrer votre ERP de construction avec vos applications tierces:

- **QuickBooks / Sage** : Synchronisation des factures, clients et paiements
- **n8n / Zapier** : Automatisation de workflows
- **Applications mobiles** : Accès aux données terrain
- **Systèmes personnalisés** : Intégration sur mesure

### Caractéristiques

- Architecture **Multi-Tenant** : Chaque entreprise a ses données isolées
- **30+ endpoints** CRUD complets
- **Webhooks** temps réel pour les événements
- **Rate limiting** configurable par clé API
- Documentation **OpenAPI 3.1** (Swagger)

---

## Authentification

Toutes les requêtes (sauf `/` health check) nécessitent une clé API.

### Header requis

```
X-API-Key: cai_live_VOTRE_CLE_ICI
```

### Exemple avec cURL

```bash
curl -X GET "https://constructo-api.onrender.com/api/v1/companies" \
     -H "X-API-Key: cai_live_VOTRE_CLE_ICI"
```

### Exemple avec Python

```python
import requests

headers = {
    "X-API-Key": "cai_live_VOTRE_CLE_ICI"
}

response = requests.get(
    "https://constructo-api.onrender.com/api/v1/companies",
    headers=headers
)

data = response.json()
print(data)
```

### Exemple avec JavaScript

```javascript
const response = await fetch('https://constructo-api.onrender.com/api/v1/companies', {
    headers: {
        'X-API-Key': 'cai_live_VOTRE_CLE_ICI'
    }
});

const data = await response.json();
console.log(data);
```

---

## Créer une clé API

### Étape 1 : Accéder à l'application

1. Connectez-vous à **https://app.constructoai.ca**
2. Assurez-vous d'avoir les droits **administrateur**

### Étape 2 : Générer la clé

1. Allez dans **Configuration** → **Clés API**
2. Cliquez sur l'onglet **"Nouvelle Clé"**
3. Remplissez les informations :
   - **Nom** : Ex: "Intégration QuickBooks"
   - **Description** : Usage prévu
   - **Niveau d'accès** :
     - `Lecture seule` : Pour analytics et rapports
     - `Lecture et écriture` : Pour synchronisation bidirectionnelle
     - `Accès complet` : Pour administration
   - **Limite de requêtes** : 100 à 10,000 req/heure
   - **Expiration** : 30 jours à 1 an (ou jamais)

### Étape 3 : Copier la clé

⚠️ **IMPORTANT** : La clé complète n'est affichée qu'**une seule fois**. Copiez-la immédiatement!

Format de la clé :
- Production : `cai_live_XXXXXXXXXXXXXXXXXXXXXXXX`
- Test : `cai_test_XXXXXXXXXXXXXXXXXXXXXXXX`

---

## Rate Limiting

| Type | Limite par défaut |
|------|-------------------|
| Requêtes par heure | 1000 |
| Requêtes par minute | ~17 |

### Headers de réponse

Chaque réponse inclut :

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1702512000
```

### Dépassement de limite

Si vous dépassez la limite, vous recevrez :

```json
{
    "detail": "Rate limit exceeded. Please retry after X seconds.",
    "retry_after": 120
}
```

**Code HTTP** : `429 Too Many Requests`

---

## Endpoints disponibles

### Health

#### `GET /`
Vérification santé de l'API (public, sans authentification)

```bash
curl https://constructo-api.onrender.com/
```

**Réponse :**
```json
{
    "status": "healthy",
    "service": "Constructo AI API",
    "version": "2.0.0",
    "timestamp": "2025-12-14T00:08:32.146279"
}
```

#### `GET /api/v1/health`
Santé détaillée (requiert authentification)

#### `GET /api/v1/stats/dashboard`
Statistiques tableau de bord

---

### Companies (Clients)

#### `GET /api/v1/companies`
Liste tous les clients/fournisseurs

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `type` | string | Filtrer par type (`client`, `fournisseur`, `both`) |
| `search` | string | Recherche par nom |
| `limit` | int | Nombre de résultats (défaut: 100) |
| `offset` | int | Pagination |

**Exemple :**
```bash
curl -X GET "https://constructo-api.onrender.com/api/v1/companies?type=client&limit=50" \
     -H "X-API-Key: cai_live_VOTRE_CLE"
```

**Réponse :**
```json
{
    "items": [
        {
            "id": 1,
            "nom": "Construction ABC Inc.",
            "type": "client",
            "email": "info@abc.com",
            "telephone": "514-555-1234",
            "adresse": "123 Rue Principale",
            "ville": "Montréal",
            "code_postal": "H2X 1Y6",
            "province": "QC",
            "numero_tps": "123456789RT0001",
            "numero_tvq": "1234567890TQ0001",
            "created_at": "2025-01-15T10:30:00"
        }
    ],
    "total": 1,
    "limit": 50,
    "offset": 0
}
```

#### `POST /api/v1/companies`
Créer un nouveau client/fournisseur

**Body :**
```json
{
    "nom": "Nouveau Client Inc.",
    "type": "client",
    "email": "contact@nouveau.com",
    "telephone": "514-555-9999",
    "adresse": "456 Rue Commerce",
    "ville": "Laval",
    "code_postal": "H7N 2T4",
    "province": "QC"
}
```

#### `GET /api/v1/companies/{company_id}`
Détails d'un client spécifique

#### `PUT /api/v1/companies/{company_id}`
Modifier un client existant

---

### Projects (Projets)

#### `GET /api/v1/projects`
Liste tous les projets

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `status` | string | Filtrer par statut (`actif`, `termine`, `en_attente`) |
| `client_id` | int | Filtrer par client |
| `limit` | int | Nombre de résultats |
| `offset` | int | Pagination |

**Réponse :**
```json
{
    "items": [
        {
            "id": 1,
            "nom": "Rénovation Cuisine - Tremblay",
            "description": "Rénovation complète cuisine et salle à manger",
            "client_id": 5,
            "client_nom": "Jean Tremblay",
            "adresse_chantier": "789 Rue des Érables, Longueuil",
            "date_debut": "2025-01-20",
            "date_fin_prevue": "2025-03-15",
            "budget_total": 45000.00,
            "statut": "actif",
            "progression": 35
        }
    ],
    "total": 1
}
```

#### `POST /api/v1/projects`
Créer un nouveau projet

#### `GET /api/v1/projects/{project_id}`
Détails d'un projet avec toutes ses opérations

#### `PUT /api/v1/projects/{project_id}`
Modifier un projet

---

### Invoices (Factures)

#### `GET /api/v1/invoices`
Liste toutes les factures

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `status` | string | `brouillon`, `envoyee`, `payee`, `en_retard` |
| `client_id` | int | Filtrer par client |
| `date_from` | date | Date minimum (YYYY-MM-DD) |
| `date_to` | date | Date maximum |
| `limit` | int | Nombre de résultats |
| `offset` | int | Pagination |

**Réponse :**
```json
{
    "items": [
        {
            "id": 101,
            "numero": "FAC-2025-0101",
            "client_id": 5,
            "client_nom": "Jean Tremblay",
            "projet_id": 1,
            "projet_nom": "Rénovation Cuisine",
            "date_facture": "2025-02-01",
            "date_echeance": "2025-03-01",
            "sous_total": 15000.00,
            "tps": 750.00,
            "tvq": 1496.25,
            "total": 17246.25,
            "montant_paye": 5000.00,
            "solde": 12246.25,
            "statut": "envoyee",
            "lignes": [
                {
                    "description": "Main d'oeuvre - Démolition",
                    "quantite": 16,
                    "unite": "heures",
                    "prix_unitaire": 65.00,
                    "total": 1040.00
                },
                {
                    "description": "Armoires cuisine - Modèle Shaker",
                    "quantite": 1,
                    "unite": "ensemble",
                    "prix_unitaire": 8500.00,
                    "total": 8500.00
                }
            ]
        }
    ],
    "total": 1
}
```

#### `POST /api/v1/invoices`
Créer une nouvelle facture

**Body :**
```json
{
    "client_id": 5,
    "projet_id": 1,
    "date_facture": "2025-02-01",
    "date_echeance": "2025-03-01",
    "lignes": [
        {
            "description": "Main d'oeuvre - Installation",
            "quantite": 24,
            "unite": "heures",
            "prix_unitaire": 65.00
        },
        {
            "description": "Matériaux divers",
            "quantite": 1,
            "unite": "lot",
            "prix_unitaire": 2500.00
        }
    ],
    "notes": "Paiement net 30 jours"
}
```

#### `GET /api/v1/invoices/{invoice_id}`
Détails complets d'une facture

#### `POST /api/v1/invoices/{invoice_id}/payments`
Enregistrer un paiement

**Body :**
```json
{
    "montant": 5000.00,
    "date_paiement": "2025-02-15",
    "methode": "virement",
    "reference": "VIR-123456"
}
```

#### `GET /api/v1/invoices/{invoice_id}/payments`
Historique des paiements d'une facture

---

### Products (Produits)

#### `GET /api/v1/products`
Liste tous les produits du catalogue

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `category` | string | Filtrer par catégorie |
| `search` | string | Recherche par nom/SKU |
| `in_stock` | bool | Seulement les produits en stock |

#### `GET /api/v1/products/{product_id}`
Détails d'un produit

#### `PUT /api/v1/products/{product_id}/stock`
Ajuster le stock d'un produit

**Body :**
```json
{
    "quantite": 50,
    "type_mouvement": "entree",
    "raison": "Réception commande #PO-2025-042"
}
```

---

### Inventory (Inventaire)

#### `GET /api/v1/inventory`
État actuel de l'inventaire

#### `GET /api/v1/inventory/movements`
Historique des mouvements de stock

#### `GET /api/v1/inventory/alerts`
Alertes de stock bas

---

### Quotes (Devis)

#### `GET /api/v1/quotes`
Liste tous les devis/soumissions

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `status` | string | `brouillon`, `envoye`, `accepte`, `refuse`, `expire` |
| `client_id` | int | Filtrer par client |

#### `GET /api/v1/quotes/{quote_id}`
Détails d'un devis

#### `POST /api/v1/quotes/{quote_id}/convert`
Convertir un devis accepté en facture

---

### Employees (Employés)

#### `GET /api/v1/employees`
Liste des employés

**Paramètres query :**
| Paramètre | Type | Description |
|-----------|------|-------------|
| `actif` | bool | Seulement les employés actifs |
| `poste` | string | Filtrer par poste CCQ |

---

### Webhooks

Les webhooks permettent de recevoir des notifications en temps réel lorsque des événements se produisent dans Constructo AI.

#### `GET /api/v1/webhooks`
Liste vos webhooks configurés

#### `POST /api/v1/webhooks`
Créer un nouveau webhook

**Body :**
```json
{
    "url": "https://votre-serveur.com/webhook",
    "events": [
        "invoice.created",
        "invoice.paid",
        "payment.received",
        "project.status_changed"
    ],
    "secret": "votre_secret_pour_signature"
}
```

#### Événements disponibles

| Catégorie | Événements |
|-----------|------------|
| **Factures** | `invoice.created`, `invoice.updated`, `invoice.sent`, `invoice.paid`, `invoice.overdue`, `invoice.deleted` |
| **Paiements** | `payment.received`, `payment.updated`, `payment.deleted` |
| **Projets** | `project.created`, `project.updated`, `project.status_changed`, `project.completed`, `project.deleted` |
| **Devis** | `quote.created`, `quote.sent`, `quote.accepted`, `quote.rejected`, `quote.expired` |
| **Inventaire** | `inventory.low_stock`, `inventory.out_of_stock`, `inventory.restocked` |
| **Clients** | `company.created`, `company.updated` |
| **Employés** | `employee.created`, `employee.updated` |

#### Signature des webhooks

Chaque webhook inclut une signature HMAC-SHA256 dans le header `X-Webhook-Signature` pour vérifier l'authenticité :

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(f"sha256={expected}", signature)
```

#### Format du payload webhook

```json
{
    "event": "invoice.paid",
    "timestamp": "2025-02-15T14:30:00Z",
    "data": {
        "invoice_id": 101,
        "numero": "FAC-2025-0101",
        "client_id": 5,
        "total": 17246.25,
        "montant_paye": 17246.25
    }
}
```

---

## Exemples d'intégration

### Intégration n8n

1. Ajoutez un noeud **HTTP Request**
2. Configurez :
   - **Method** : GET ou POST selon l'endpoint
   - **URL** : `https://constructo-api.onrender.com/api/v1/invoices`
   - **Authentication** : Header Auth
   - **Header Name** : `X-API-Key`
   - **Header Value** : `cai_live_VOTRE_CLE`

### Intégration Python complète

```python
import requests
from datetime import datetime, timedelta

class ConstructoAPI:
    def __init__(self, api_key):
        self.base_url = "https://constructo-api.onrender.com"
        self.headers = {"X-API-Key": api_key}

    def get_invoices(self, status=None, days=30):
        """Récupère les factures des X derniers jours"""
        params = {
            "date_from": (datetime.now() - timedelta(days=days)).strftime("%Y-%m-%d")
        }
        if status:
            params["status"] = status

        response = requests.get(
            f"{self.base_url}/api/v1/invoices",
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()

    def create_invoice(self, client_id, lignes, projet_id=None):
        """Crée une nouvelle facture"""
        data = {
            "client_id": client_id,
            "date_facture": datetime.now().strftime("%Y-%m-%d"),
            "date_echeance": (datetime.now() + timedelta(days=30)).strftime("%Y-%m-%d"),
            "lignes": lignes
        }
        if projet_id:
            data["projet_id"] = projet_id

        response = requests.post(
            f"{self.base_url}/api/v1/invoices",
            headers=self.headers,
            json=data
        )
        response.raise_for_status()
        return response.json()

    def record_payment(self, invoice_id, montant, methode="virement"):
        """Enregistre un paiement sur une facture"""
        data = {
            "montant": montant,
            "date_paiement": datetime.now().strftime("%Y-%m-%d"),
            "methode": methode
        }

        response = requests.post(
            f"{self.base_url}/api/v1/invoices/{invoice_id}/payments",
            headers=self.headers,
            json=data
        )
        response.raise_for_status()
        return response.json()


# Utilisation
api = ConstructoAPI("cai_live_VOTRE_CLE_ICI")

# Récupérer les factures impayées
factures = api.get_invoices(status="envoyee")
print(f"Factures en attente: {len(factures['items'])}")

# Créer une facture
nouvelle_facture = api.create_invoice(
    client_id=5,
    lignes=[
        {"description": "Consultation", "quantite": 2, "unite": "heures", "prix_unitaire": 150.00}
    ]
)
print(f"Facture créée: {nouvelle_facture['numero']}")
```

### Intégration QuickBooks (via n8n ou Zapier)

**Workflow de synchronisation des factures :**

1. **Trigger** : Webhook Constructo AI (`invoice.created`)
2. **Action** : Créer facture dans QuickBooks
   - Mapper les champs :
     - `client_nom` → Customer
     - `lignes[].description` → Line Item Description
     - `lignes[].total` → Line Item Amount
     - `total` → Total Amount

**Workflow de synchronisation des paiements :**

1. **Trigger** : Nouveau paiement dans QuickBooks
2. **Action** : HTTP Request vers Constructo API
   - `POST /api/v1/invoices/{id}/payments`

---

## Codes d'erreur

| Code | Signification | Action recommandée |
|------|---------------|-------------------|
| `200` | Succès | - |
| `201` | Créé avec succès | - |
| `400` | Requête invalide | Vérifiez les paramètres envoyés |
| `401` | Non authentifié | Vérifiez votre clé API |
| `403` | Accès refusé | Vérifiez les permissions de votre clé |
| `404` | Ressource non trouvée | Vérifiez l'ID de la ressource |
| `422` | Données invalides | Vérifiez le format des données |
| `429` | Rate limit dépassé | Attendez avant de réessayer |
| `500` | Erreur serveur | Contactez le support |

### Format des erreurs

```json
{
    "detail": "Description de l'erreur",
    "error_code": "INVALID_PARAMETER",
    "field": "client_id"
}
```

---

## Support

**Besoin d'aide avec l'API?**

- **Documentation interactive** : https://constructo-api.onrender.com/docs
- **Téléphone** : (514) 820-1972
- **Email** : info@constructoai.ca
- **Site web** : https://constructoai.ca

---

*Documentation générée le 14 décembre 2025*
*Constructo AI - L'ERP intelligent pour la construction au Québec*
