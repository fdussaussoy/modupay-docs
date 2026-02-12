---
title: Référence API
layout: default
nav_order: 6
---

# Référence API
{: .no_toc }

## Sommaire
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Endpoints principaux

### Authentification

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/auth/token` | Obtenir un token JWT |

### Organisations

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/organizations` | Lister les organisations |
| `POST` | `/organizations` | Créer une organisation |
| `GET` | `/organizations/:id` | Récupérer une organisation |
| `PATCH` | `/organizations/:id` | Modifier une organisation |

### Régies & Services

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/organizations/:id/departments` | Lister les régies |
| `POST` | `/organizations/:id/departments` | Créer une régie |
| `GET` | `/organizations/:id/departments/:id` | Récupérer une régie |

### Factures

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/invoices` | Lister les factures |
| `POST` | `/invoices` | Créer une facture |
| `GET` | `/invoices/:id` | Récupérer une facture |
| `POST` | `/invoices/:id/issue` | Émettre une facture |
| `POST` | `/invoices/:id/cancel` | Annuler une facture |

### Liens de paiement

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/payment_links` | Créer un lien de paiement |
| `GET` | `/payment_links/:id` | Récupérer un lien |
| `DELETE` | `/payment_links/:id` | Révoquer un lien |

### Intentions de paiement

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/payment_intents` | Lister les intentions |
| `GET` | `/payment_intents/:id` | Récupérer une intention |

### Remboursements

| Méthode | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/refunds` | Créer un remboursement |
| `GET` | `/refunds/:id` | Récupérer un remboursement |

---

## Exemples de requêtes

### Créer une facture

```bash
curl -X POST https://api.votreplateforme.fr/v1/invoices \
  -H "Authorization: Bearer {token}" \
  -H "X-Entity-Id: {organization_id}" \
  -H "Content-Type: application/json" \
  -d '{
    "department_id": "uuid-régie",
    "payer": {
      "name": "Jean Dupont",
      "email": "jean.dupont@email.fr"
    },
    "due_date": "2026-03-15",
    "line_items": [
      {
        "description": "Consommation eau T1 2026 — 125 m³",
        "quantity": 1,
        "unit_price": 3800,
        "vat_rate": 5.5
      }
    ]
  }'
```

### Générer un lien de paiement

```bash
curl -X POST https://api.votreplateforme.fr/v1/payment_links \
  -H "Authorization: Bearer {token}" \
  -H "X-Entity-Id: {organization_id}" \
  -H "Content-Type: application/json" \
  -d '{
    "invoice_id": "uuid-facture",
    "payment_methods": ["sepa_credit", "card"],
    "expires_at": "2026-03-14T23:59:59Z",
    "return_url": "https://votre-site.fr/paiement/ok"
  }'
```

### Créer un remboursement

```bash
curl -X POST https://api.votreplateforme.fr/v1/refunds \
  -H "Authorization: Bearer {token}" \
  -H "X-Entity-Id: {organization_id}" \
  -H "Content-Type: application/json" \
  -d '{
    "payment_intent_id": "uuid-intention",
    "amount": 4220,
    "reason": "Erreur de facturation"
  }'
```

---

## Codes d'erreur

| Code | Signification |
|:---|:---|
| `400` | Requête invalide (validation) |
| `401` | Token manquant ou expiré |
| `403` | Accès non autorisé |
| `404` | Ressource introuvable |
| `409` | Conflit (ex: facture déjà émise) |
| `422` | Erreur métier (ex: montant > disponible) |
| `429` | Rate limit dépassé (1 000 req/min) |
| `500` | Erreur serveur |

---

## Webhooks

### Événements disponibles

| Événement | Déclencheur |
|:---|:---|
| `invoice.created` | Facture créée |
| `invoice.status_changed` | Statut facture modifié |
| `payment_link.created` | Lien de paiement créé |
| `payment_intent.status_changed` | Statut paiement modifié |
| `refund.created` | Remboursement initié |
| `refund.status_changed` | Statut remboursement modifié |

### Vérification de signature

```python
import hmac, hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

{: .note }
> 📋 Consultez la [spécification OpenAPI complète](../docs/payment-api-spec.yaml) pour le détail de tous les schémas et paramètres.
