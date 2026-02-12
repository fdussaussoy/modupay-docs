---
title: Guide d'intégration
layout: default
nav_order: 3
has_toc: true
---

# Guide d'intégration - API de Paiement en Marque Blanche

## 📋 Table des matières

1. [Introduction](#introduction)
2. [Concepts clés](#concepts-clés)
3. [Démarrage rapide](#démarrage-rapide)
4. [Flux de paiement complet](#flux-de-paiement-complet)
5. [Intégration Aiia](#intégration-aiia)
6. [Gestion des webhooks](#gestion-des-webhooks)
7. [Remboursements](#remboursements)
8. [Audit et conformité](#audit-et-conformité)
9. [Environnements](#environnements)
10. [Exemples de code](#exemples-de-code)

---

## Introduction

Cette API permet aux intégrateurs de déployer en marque blanche une solution complète de gestion de paiements pour le marché français, incluant :

- 🏢 Gestion multi-organisations (entreprises et collectivités)
- 📊 Régies et services avec comptes bancaires dédiés
- 🧾 Facturation avec liens de paiement
- 💳 Initiation de virements via Aiia de Mastercard
- 🔔 Webhooks temps réel
- 💰 Remboursements partiels ou totaux
- 📝 Traçabilité complète pour audit

---

## Concepts clés

### Architecture hiérarchique

```
Partner (Intégrateur)
  │
  ├─ Organization A (Entreprise)
  │    ├─ Department 1 (Régie Eau)
  │    │    ├─ Bank Account: FR76...
  │    │    └─ Invoices
  │    └─ Department 2 (Régie Électricité)
  │         ├─ Bank Account: FR76...
  │         └─ Invoices
  │
  └─ Organization B (Collectivité)
       ├─ Department 1 (Service Urbanisme)
       │    └─ Bank Account: FR76...
       └─ Department 2 (Service Cantines)
            └─ Bank Account: FR76...
```

### Flux de vie d'une facture

```
draft → issued → partially_paid → paid
   ↓       ↓
cancelled  overdue
```

### Flux d'un paiement

```
PaymentLink (créé) 
    ↓
PaymentIntent (created)
    ↓
Utilisateur sélectionne méthode → pending
    ↓
Initiation virement Aiia → processing
    ↓
Virement confirmé → succeeded
    ↓
Facture mise à jour → paid
```

---

## Démarrage rapide

### 1. Obtenir vos credentials

Contactez notre équipe pour obtenir :
- `client_id`
- `client_secret`

### 2. Authentification

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "votre_client_id",
    "client_secret": "votre_client_secret",
    "scope": "partner"
  }'
```

Réponse :
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "partner"
}
```

### 3. Créer votre première organisation

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/organizations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "public_entity",
    "name": "Mairie de Lyon",
    "siret": "21690123400019",
    "vat_number": "FR21216901234",
    "address": {
      "line1": "1 Place de la Comédie",
      "postal_code": "69001",
      "city": "Lyon",
      "country": "FR"
    }
  }'
```

### 4. Créer une régie/service

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/organizations/{org_id}/departments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Régie de l'\''Eau",
    "code": "REG-EAU-001",
    "description": "Gestion des factures d'\''eau potable"
  }'
```

### 5. Ajouter un compte bancaire

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/departments/{dept_id}/bank_accounts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "iban": "FR7612345678901234567890123",
    "bic": "BNPAFRPPXXX",
    "account_name": "Compte Régie Eau",
    "bank_name": "BNP Paribas",
    "is_default": true
  }'
```

### 6. Valider le KYC de l'organisation

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/departments/{dept_id}/kyc \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "iban": "FR7612345678901234567890123",
    "bic": "BNPAFRPPXXX",
    "account_name": "Compte Régie Eau",
    "bank_name": "BNP Paribas",
    "is_default": true
  }'
```

---

## Flux de paiement complet

### Étape 1 : Créer une facture

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/invoices \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "department_id": "dept-uuid",
    "payer": {
      "name": "Jean Dupont",
      "email": "jean.dupont@example.com",
      "address": {
        "line1": "15 rue de la Paix",
        "postal_code": "69002",
        "city": "Lyon",
        "country": "FR"
      }
    },
    "line_items": [
      {
        "description": "Consommation eau T1 2024",
        "quantity": 125,
        "unit_price": 320,
        "vat_rate": 5.5
      }
    ],
    "due_date": "2024-03-31",
    "payment_terms": "Paiement sous 30 jours"
  }'
```

Réponse :
```json
{
  "id": "inv-123e4567-e89b-12d3-a456-426614174000",
  "invoice_number": "FAC-2024-00001",
  "status": "draft",
  "total_amount": 42220,
  "amount_due": 42220,
  ...
}
```

### Étape 2 : Émettre la facture

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/invoices/{invoice_id}/issue \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID"
```

La facture passe en statut `issued`.

### Étape 3 : Créer un lien de paiement

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/payment_links \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "invoice_id": "inv-123e4567-e89b-12d3-a456-426614174000",
    "payment_methods": ["sepa_credit", "card"],
    "return_url": "https://votreapp.fr/payment/success",
    "expires_at": "2024-03-31T23:59:59Z"
  }'
```

Réponse :
```json
{
  "id": "pl-987fcdeb-51a2-43f1-b789-123456789abc",
  "invoice_id": "inv-123e4567-e89b-12d3-a456-426614174000",
  "payment_intent_id": "pi-456def78-90ab-cdef-1234-567890abcdef",
  "amount": 42220,
  "currency": "EUR",
  "status": "active",
  "payment_page_url": "https://pay.modupay.fr/p/abc123xyz456",
  "qr_code_data": "https://pay.modupay.fr/p/abc123xyz456",
  "expires_at": "2024-03-31T23:59:59Z",
  "created_at": "2024-02-01T10:30:00Z"
}
```

### Étape 4 : Partager le lien avec le payeur

Envoyez `payment_page_url` au payeur par :
- Email
- SMS
- QR Code (utilisez `qr_code_data`)

### Étape 5 : Le payeur effectue le paiement

1. Le payeur accède à la page de paiement
2. Il sélectionne "Virement SEPA"
3. Il autorise le virement via son application bancaire (Aiia)
4. Le virement est initié

### Étape 6 : Suivi via webhooks

Vous recevez des webhooks à chaque étape :

```json
// Webhook 1 : Méthode sélectionnée
{
  "event_id": "evt-abc123",
  "event_type": "payment_intent.status_changed",
  "timestamp": "2024-02-01T10:35:00Z",
  "organization_id": "org-uuid",
  "data": {
    "payment_intent_id": "pi-456def78-90ab-cdef-1234-567890abcdef",
    "previous_status": "created",
    "current_status": "pending",
    "payment_intent": {
      "selected_payment_method": "sepa_credit",
      ...
    }
  }
}

// Webhook 2 : Virement en cours
{
  "event_type": "payment_intent.status_changed",
  "data": {
    "previous_status": "pending",
    "current_status": "processing",
    "payment_intent": {
      "aiia_transaction_id": "aiia-tx-789",
      "aiia_status": "initiated"
    }
  }
}

// Webhook 3 : Virement réussi
{
  "event_type": "payment_intent.status_changed",
  "data": {
    "previous_status": "processing",
    "current_status": "succeeded",
    "payment_intent": {
      "aiia_status": "settled",
      "completed_at": "2024-02-01T10:37:00Z"
    }
  }
}

// Webhook 4 : Facture payée
{
  "event_type": "invoice.status_changed",
  "data": {
    "invoice_id": "inv-123e4567-e89b-12d3-a456-426614174000",
    "previous_status": "issued",
    "current_status": "paid"
  }
}
```

---

## Intégration Aiia

### Configuration

Aiia de Mastercard est intégré nativement pour les virements SEPA. Aucune configuration additionnelle n'est requise de votre côté.

### Flux technique

```
Utilisateur sur page de paiement
    ↓
Sélectionne "Virement SEPA"
    ↓
Redirection vers Aiia
    ↓
Authentification bancaire (Strong Customer Auth)
    ↓
Autorisation du virement
    ↓
Retour sur page de paiement
    ↓
API initie le virement via Aiia
    ↓
Virement SEPA traité (1-2 jours ouvrés)
```

### Suivi du statut Aiia

Le champ `aiia_status` du PaymentIntent reflète l'état côté Aiia :

- `initiated` : Virement initié
- `authorized` : Virement autorisé par la banque
- `settled` : Virement exécuté (fonds reçus)
- `rejected` : Virement rejeté

### Délais de traitement

- **Initiation** : Immédiate
- **Autorisation** : Quelques secondes
- **Règlement** : 1-2 jours ouvrés (SEPA standard)

---

## Gestion des webhooks

### Configuration

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/webhook_subscriptions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://votreapp.fr/webhooks/payment-api",
    "events": [
      "invoice.status_changed",
      "payment_intent.status_changed",
      "refund.created"
    ],
    "description": "Production webhook"
  }'
```

Réponse :
```json
{
  "id": "wh-sub-123",
  "url": "https://votreapp.fr/webhooks/payment-api",
  "events": ["invoice.status_changed", "payment_intent.status_changed"],
  "secret": "whsec_K8W5m3n2k4J9p7Q1r6T8v5Y2z4",
  "status": "active",
  "created_at": "2024-02-01T09:00:00Z"
}
```

⚠️ **Conservez le `secret` précieusement** - il sert à vérifier l'authenticité des webhooks.

### Vérification de signature

#### Python
```python
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Vérifie la signature HMAC-SHA256 d'un webhook"""
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    expected = f"sha256={expected_signature}"
    
    return hmac.compare_digest(expected, signature)

# Utilisation dans Flask
@app.route('/webhooks/payment-api', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Webhook-Signature')
    payload = request.get_data()
    
    if not verify_webhook_signature(payload, signature, WEBHOOK_SECRET):
        return 'Invalid signature', 401
    
    event = request.get_json()
    
    if event['event_type'] == 'payment_intent.status_changed':
        handle_payment_status_change(event['data'])
    
    return 'OK', 200
```

#### Node.js
```javascript
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  const expected = `sha256=${expectedSignature}`;
  
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}

// Utilisation avec Express
app.post('/webhooks/payment-api', express.raw({type: 'application/json'}), (req, res) => {
  const signature = req.headers['x-webhook-signature'];
  
  if (!verifyWebhookSignature(req.body, signature, WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }
  
  const event = JSON.parse(req.body);
  
  if (event.event_type === 'payment_intent.status_changed') {
    handlePaymentStatusChange(event.data);
  }
  
  res.status(200).send('OK');
});
```

### Bonnes pratiques

1. **Idempotence** : Traitez les webhooks de manière idempotente (même événement reçu plusieurs fois = même résultat)
2. **Réponse rapide** : Répondez 200 OK rapidement, puis traitez en asynchrone
3. **Retry** : Si votre endpoint répond 5xx, nous réessayons avec backoff exponentiel (5 tentatives max)
4. **Logs** : Loggez tous les événements reçus pour debug
5. **Monitoring** : Surveillez les webhooks non reçus dans le dashboard

---

## Remboursements

### Remboursement total

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/refunds \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "payment_intent_id": "pi-456def78-90ab-cdef-1234-567890abcdef",
    "reason": "Facture annulée suite à erreur de facturation"
  }'
```

Si `amount` n'est pas spécifié, c'est un remboursement total.

### Remboursement partiel

```bash
curl -X POST https://api.sandbox.modupay.fr/v1/refunds \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "payment_intent_id": "pi-456def78-90ab-cdef-1234-567890abcdef",
    "amount": 10000,
    "reason": "Remise commerciale de 100€"
  }'
```

### Remboursements multiples

Vous pouvez effectuer plusieurs remboursements partiels tant que :
```
somme(remboursements) <= montant_paiement_initial
```

### Suivi des remboursements

```bash
# Lister tous les remboursements d'un paiement
curl -X GET https://api.sandbox.modupay.fr/v1/payment_intents/{payment_intent_id}/refunds \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID"
```

Réponse :
```json
{
  "data": [
    {
      "id": "ref-abc123",
      "amount": 10000,
      "status": "succeeded",
      "reason": "Remise commerciale",
      "created_at": "2024-02-05T14:30:00Z",
      "completed_at": "2024-02-05T14:31:00Z"
    },
    {
      "id": "ref-def456",
      "amount": 5000,
      "status": "succeeded",
      "reason": "Geste commercial",
      "created_at": "2024-02-10T09:15:00Z",
      "completed_at": "2024-02-10T09:16:00Z"
    }
  ]
}
```

### Impact sur la facture

Après remboursement :
- `payment_intent.status` → `refunded` ou `partially_refunded`
- `payment_intent.refunded_amount` est mis à jour
- Si remboursement total : `invoice.status` → revient à `issued`
- Si remboursement partiel : `invoice.amount_paid` est recalculé

---

## Audit et conformité

### Consultation des logs

```bash
curl -X GET "https://api.sandbox.modupay.fr/v1/audit_logs?resource_type=invoice&from_date=2024-02-01T00:00:00Z" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Entity-Id: $ORG_ID"
```

Réponse :
```json
{
  "data": [
    {
      "id": "log-123",
      "timestamp": "2024-02-01T10:30:15Z",
      "resource_type": "invoice",
      "resource_id": "inv-123e4567-e89b-12d3-a456-426614174000",
      "action": "created",
      "user_id": "partner-admin",
      "user_type": "partner",
      "ip_address": "203.0.113.42",
      "changes": {
        "before": null,
        "after": {
          "status": "draft",
          "total_amount": 42220,
          ...
        }
      }
    },
    {
      "id": "log-124",
      "timestamp": "2024-02-01T10:31:00Z",
      "resource_type": "invoice",
      "resource_id": "inv-123e4567-e89b-12d3-a456-426614174000",
      "action": "status_changed",
      "user_id": "partner-admin",
      "user_type": "partner",
      "changes": {
        "before": {"status": "draft"},
        "after": {"status": "issued"}
      }
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total_items": 2,
    "total_pages": 1
  }
}
```

### Données tracées

Tous les événements suivants sont loggés :

| Événement | Durée de rétention |
|-----------|-------------------|
| Création d'organisation | 7 ans |
| Création/modification de facture | 7 ans |
| Création de lien de paiement | 7 ans |
| Changement de statut de paiement | 7 ans |
| Remboursement | 7 ans |
| Appel API (tous) | 1 an |

### Conformité RGPD

- Les données personnelles sont chiffrées au repos et en transit
- Droit d'accès, rectification et suppression disponibles sur demande
- Sous-traitants certifiés (Aiia : certifié PSD2)
- DPA disponible sur demande

### Conformité PSD2

- Authentification forte (SCA) via Aiia
- Traçabilité complète des virements
- Conformité aux directives européennes

---

## Environnements

### Sandbox

**Base URL:** `https://api.sandbox.modupay.fr/v1`

- Données de test uniquement
- Paiements simulés (aucun argent réel)
- Aiia en mode test
- Webhooks fonctionnels

**IBANs de test:**
```
FR7612345678901234567890123  (succès)
FR7612345678901234567890124  (échec - fonds insuffisants)
FR7612345678901234567890125  (échec - compte bloqué)
```

### Production

**Base URL:** `https://api.modupay.fr/v1`

- Données et paiements réels
- Aiia en production
- Conformité bancaire complète

---

## Exemples de code

### SDK Python (exemple non officiel)

```python
import requests
from typing import Optional, Dict, Any

class PaymentAPIClient:
    def __init__(self, client_id: str, client_secret: str, base_url: str):
        self.base_url = base_url
        self.client_id = client_id
        self.client_secret = client_secret
        self.token = None
    
    def authenticate(self):
        """Obtenir un token d'accès"""
        response = requests.post(
            f"{self.base_url}/auth/token",
            json={
                "grant_type": "client_credentials",
                "client_id": self.client_id,
                "client_secret": self.client_secret,
                "scope": "partner"
            }
        )
        response.raise_for_status()
        self.token = response.json()["access_token"]
    
    def _headers(self, entity_id: Optional[str] = None) -> Dict[str, str]:
        headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }
        if entity_id:
            headers["X-Entity-Id"] = entity_id
        return headers
    
    def create_organization(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Créer une organisation"""
        response = requests.post(
            f"{self.base_url}/organizations",
            json=data,
            headers=self._headers()
        )
        response.raise_for_status()
        return response.json()
    
    def create_invoice(self, entity_id: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Créer une facture"""
        response = requests.post(
            f"{self.base_url}/invoices",
            json=data,
            headers=self._headers(entity_id)
        )
        response.raise_for_status()
        return response.json()
    
    def create_payment_link(self, entity_id: str, invoice_id: str, 
                           payment_methods: list) -> Dict[str, Any]:
        """Créer un lien de paiement"""
        response = requests.post(
            f"{self.base_url}/payment_links",
            json={
                "invoice_id": invoice_id,
                "payment_methods": payment_methods
            },
            headers=self._headers(entity_id)
        )
        response.raise_for_status()
        return response.json()
    
    def get_payment_intent(self, entity_id: str, 
                          payment_intent_id: str) -> Dict[str, Any]:
        """Récupérer une intention de paiement"""
        response = requests.get(
            f"{self.base_url}/payment_intents/{payment_intent_id}",
            headers=self._headers(entity_id)
        )
        response.raise_for_status()
        return response.json()

# Utilisation
client = PaymentAPIClient(
    client_id="votre_client_id",
    client_secret="votre_client_secret",
    base_url="https://api.sandbox.modupay.fr/v1"
)

client.authenticate()

# Créer une organisation
org = client.create_organization({
    "type": "public_entity",
    "name": "Mairie de Paris",
    "siret": "21750001600019"
})

# Créer une facture
invoice = client.create_invoice(
    entity_id=org["id"],
    data={
        "department_id": "dept-uuid",
        "payer": {
            "name": "Jean Dupont",
            "email": "jean@example.com"
        },
        "line_items": [{
            "description": "Service",
            "quantity": 1,
            "unit_price": 10000,
            "vat_rate": 20
        }]
    }
)

# Créer un lien de paiement
payment_link = client.create_payment_link(
    entity_id=org["id"],
    invoice_id=invoice["id"],
    payment_methods=["sepa_credit", "card"]
)

print(f"URL de paiement : {payment_link['payment_page_url']}")
```

### Exemple TypeScript/Node.js

```typescript
import axios, { AxiosInstance } from 'axios';

interface AuthResponse {
  access_token: string;
  token_type: string;
  expires_in: number;
}

interface Organization {
  id: string;
  name: string;
  siret: string;
  // ... autres champs
}

class PaymentAPIClient {
  private client: AxiosInstance;
  private token?: string;

  constructor(
    private clientId: string,
    private clientSecret: string,
    baseUrl: string
  ) {
    this.client = axios.create({
      baseURL: baseUrl,
      headers: {
        'Content-Type': 'application/json',
      },
    });
  }

  async authenticate(): Promise<void> {
    const response = await this.client.post<AuthResponse>('/auth/token', {
      grant_type: 'client_credentials',
      client_id: this.clientId,
      client_secret: this.clientSecret,
      scope: 'partner',
    });

    this.token = response.data.access_token;
  }

  private getHeaders(entityId?: string): Record<string, string> {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${this.token}`,
    };

    if (entityId) {
      headers['X-Entity-Id'] = entityId;
    }

    return headers;
  }

  async createOrganization(data: any): Promise<Organization> {
    const response = await this.client.post<Organization>(
      '/organizations',
      data,
      { headers: this.getHeaders() }
    );

    return response.data;
  }

  async createPaymentLink(
    entityId: string,
    invoiceId: string,
    paymentMethods: string[]
  ): Promise<any> {
    const response = await this.client.post(
      '/payment_links',
      {
        invoice_id: invoiceId,
        payment_methods: paymentMethods,
      },
      { headers: this.getHeaders(entityId) }
    );

    return response.data;
  }

  async getAuditLogs(
    entityId: string,
    params: {
      resource_type?: string;
      from_date?: string;
      to_date?: string;
    }
  ): Promise<any> {
    const response = await this.client.get('/audit_logs', {
      params,
      headers: this.getHeaders(entityId),
    });

    return response.data;
  }
}

// Utilisation
const client = new PaymentAPIClient(
  'votre_client_id',
  'votre_client_secret',
  'https://api.sandbox.modupay.fr/v1'
);

await client.authenticate();

const org = await client.createOrganization({
  type: 'company',
  name: 'Mon Entreprise',
  siret: '12345678901234',
});

console.log('Organisation créée:', org.id);
```

---

## Support

### Documentation
- OpenAPI Spec : `/api-docs`
- Guide intégration : Ce document
- Changelog : `/changelog`

### Assistance
- Email : support-api@modupay.fr
- Slack : #api-support (accès partenaires)
- Status page : https://status.modupay.fr

### SLA Production
- Disponibilité : 99.9%
- Temps de réponse : < 200ms (P95)
- Support : 24/7 pour incidents critiques

---

## Changelog

### v1.0.0 (Février 2024)
- ✨ Version initiale
- 🏢 Gestion organisations et régies
- 🧾 Facturation complète
- 💳 Paiements via Aiia
- 🔔 Webhooks
- 💰 Remboursements
- 📝 Audit logs
