---
title: Architecture technique
layout: default
nav_order: 4
has_toc: true
---

# Architecture Technique - API de Paiement

## 📐 Vue d'ensemble

Ce document détaille l'architecture technique recommandée pour implémenter l'API de paiement en marque blanche pour le marché français.

---

## Table des matières

1. [Stack technique recommandée](#stack-technique-recommandée)
2. [Architecture système](#architecture-système)
3. [Modèle de données](#modèle-de-données)
4. [Sécurité](#sécurité)
5. [Intégration Aiia](#intégration-aiia)
6. [Gestion des webhooks](#gestion-des-webhooks)
7. [Audit et logging](#audit-et-logging)
8. [Performance et scalabilité](#performance-et-scalabilité)
9. [Disaster Recovery](#disaster-recovery)
10. [Conformité réglementaire](#conformité-réglementaire)

---

## Stack technique recommandée

### Backend
- **Langage** : Python 3.11+ ou Node.js 18+ LTS
- **Framework** : FastAPI (Python) ou NestJS (Node.js)
- **Base de données** :
  - PostgreSQL 15+ (données transactionnelles)
  - Redis (cache et rate limiting)
- **Message Queue** : RabbitMQ ou AWS SQS (webhooks asynchrones)
- **Stockage fichiers** : AWS S3 ou compatible (factures PDF)

### Sécurité
- **JWT** : RS256 (clés asymétriques)
- **Chiffrement** : AES-256-GCM (données au repos)
- **TLS** : 1.3 minimum
- **Secrets** : HashiCorp Vault ou AWS Secrets Manager

### Infrastructure
- **Cloud** : AWS, GCP ou Azure
- **Containers** : Docker + Kubernetes
- **Monitoring** : Prometheus + Grafana
- **Logs** : ELK Stack ou Datadog
- **APM** : New Relic ou Datadog

---

## Architecture système

### Architecture globale

```
                                    ┌──────────────────┐
                                    │   Load Balancer  │
                                    │     (AWS ALB)    │
                                    └────────┬─────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
            ┌───────▼──────┐         ┌──────▼──────┐         ┌──────▼──────┐
            │  API Server  │         │ API Server  │         │ API Server  │
            │   (FastAPI)  │         │  (FastAPI)  │         │  (FastAPI)  │
            └───────┬──────┘         └──────┬──────┘         └──────┬──────┘
                    │                       │                        │
                    └───────────────────────┼────────────────────────┘
                                            │
                    ┌───────────────────────┼────────────────────────┐
                    │                       │                        │
            ┌───────▼──────┐        ┌──────▼──────┐         ┌──────▼──────┐
            │  PostgreSQL  │        │    Redis    │         │  RabbitMQ   │
            │  (Primary)   │        │   (Cache)   │         │ (Webhooks)  │
            └──────┬───────┘        └─────────────┘         └──────┬──────┘
                   │                                                │
            ┌──────▼───────┐                                 ┌──────▼──────┐
            │ PostgreSQL   │                                 │  Webhook    │
            │ (Replicas)   │                                 │  Workers    │
            └──────────────┘                                 └──────┬──────┘
                                                                    │
                                                                    ▼
                                                          ┌──────────────────┐
                                                          │  Client Webhook  │
                                                          │    Endpoints     │
                                                          └──────────────────┘

                    ┌───────────────────────────────────────────────┐
                    │           Services Externes                    │
                    ├───────────────────────────────────────────────┤
                    │  • Aiia API (initiation virements)            │
                    │  • Stripe/Adyen (paiements carte)             │
                    │  • AWS S3 (stockage PDFs)                     │
                    │  • SendGrid/SES (emails)                      │
                    └───────────────────────────────────────────────┘
```

### Composants principaux

#### 1. API Gateway
- Rate limiting par client (1000 req/min)
- Authentification JWT
- Routage des requêtes
- Logging centralisé

#### 2. API Servers (Stateless)
- Logique métier
- Validation des données
- Orchestration des services
- Auto-scaling horizontal

#### 3. Database Layer
- PostgreSQL Primary (Writes)
- PostgreSQL Replicas (Reads)
- Connection pooling (PgBouncer)

#### 4. Cache Layer (Redis)
- Token validation cache (60s TTL)
- Rate limiting counters
- Session management
- Temporary data

#### 5. Async Processing
- Webhook delivery (RabbitMQ)
- PDF generation
- Email sending
- Audit log processing

---

## Modèle de données

### Schéma PostgreSQL

```sql
-- Organizations (Entreprises / Collectivités)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(20) NOT NULL CHECK (type IN ('company', 'public_entity')),
    name VARCHAR(200) NOT NULL,
    legal_name VARCHAR(200),
    siret VARCHAR(14) NOT NULL UNIQUE,
    siren VARCHAR(9) NOT NULL,
    vat_number VARCHAR(13),
    address JSONB NOT NULL,
    contact JSONB,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'closed')),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Indexes
    CONSTRAINT valid_siret CHECK (siret ~ '^\d{14}$'),
    CONSTRAINT valid_siren CHECK (siren ~ '^\d{9}$')
);

CREATE INDEX idx_organizations_siret ON organizations(siret);
CREATE INDEX idx_organizations_status ON organizations(status);
CREATE INDEX idx_organizations_type ON organizations(type);

-- Departments (Régies / Services)
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description TEXT,
    default_bank_account_id UUID,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(organization_id, code)
);

CREATE INDEX idx_departments_org ON departments(organization_id);

-- Bank Accounts (Comptes bancaires IBAN)
CREATE TABLE bank_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id UUID NOT NULL REFERENCES departments(id) ON DELETE CASCADE,
    iban VARCHAR(34) NOT NULL,
    bic VARCHAR(11) NOT NULL,
    account_name VARCHAR(100) NOT NULL,
    bank_name VARCHAR(100),
    is_default BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT valid_iban_fr CHECK (iban ~ '^FR[0-9]{2}[0-9A-Z]{23}$'),
    CONSTRAINT valid_bic CHECK (bic ~ '^[A-Z]{6}[A-Z0-9]{2}([A-Z0-9]{3})?$')
);

CREATE INDEX idx_bank_accounts_dept ON bank_accounts(department_id);
CREATE INDEX idx_bank_accounts_active ON bank_accounts(department_id, is_active) WHERE is_active = TRUE;

-- Invoices (Factures)
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    department_id UUID NOT NULL REFERENCES departments(id),
    invoice_number VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'issued', 'partially_paid', 'paid', 'overdue', 'cancelled')),
    issue_date DATE,
    due_date DATE,
    currency VARCHAR(3) DEFAULT 'EUR',
    
    -- Montants en centimes
    subtotal_amount INTEGER NOT NULL,
    vat_amount INTEGER NOT NULL,
    total_amount INTEGER NOT NULL,
    amount_paid INTEGER DEFAULT 0,
    amount_due INTEGER NOT NULL,
    
    -- Payeur
    payer JSONB NOT NULL,
    
    -- Lignes de facture
    line_items JSONB NOT NULL,
    
    notes TEXT,
    payment_terms TEXT,
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    paid_at TIMESTAMPTZ,
    
    UNIQUE(organization_id, invoice_number),
    CONSTRAINT valid_amounts CHECK (
        total_amount = subtotal_amount + vat_amount AND
        amount_due = total_amount - amount_paid AND
        amount_paid >= 0 AND
        amount_due >= 0
    )
);

CREATE INDEX idx_invoices_org ON invoices(organization_id);
CREATE INDEX idx_invoices_dept ON invoices(department_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date) WHERE status NOT IN ('paid', 'cancelled');
CREATE INDEX idx_invoices_number ON invoices(invoice_number);

-- Payment Links (Liens de paiement)
CREATE TABLE payment_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_intent_id UUID, -- Défini après création du Payment Intent
    
    amount INTEGER NOT NULL,
    currency VARCHAR(3) DEFAULT 'EUR',
    description TEXT,
    
    payment_methods TEXT[] NOT NULL,
    payment_page_url TEXT NOT NULL,
    qr_code_data TEXT,
    
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'succeeded', 'cancelled')),
    
    expires_at TIMESTAMPTZ,
    return_url TEXT,
    
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    
    CONSTRAINT valid_amount CHECK (amount > 0)
);

CREATE INDEX idx_payment_links_invoice ON payment_links(invoice_id);
CREATE INDEX idx_payment_links_status ON payment_links(status);
CREATE INDEX idx_payment_links_expires ON payment_links(expires_at) WHERE status = 'active';

-- Payment Intents (Intentions de paiement)
CREATE TABLE payment_intents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_link_id UUID NOT NULL REFERENCES payment_links(id),
    
    amount INTEGER NOT NULL,
    currency VARCHAR(3) DEFAULT 'EUR',
    
    status VARCHAR(30) DEFAULT 'created' CHECK (status IN (
        'created', 'pending', 'processing', 'succeeded', 
        'failed', 'cancelled', 'refunded', 'partially_refunded'
    )),
    
    selected_payment_method VARCHAR(20),
    
    -- Comptes bancaires
    payer_bank_account JSONB,
    recipient_bank_account JSONB,
    
    -- Aiia integration
    aiia_transaction_id VARCHAR(100),
    aiia_status VARCHAR(20),
    
    provider_reference VARCHAR(100),
    failure_reason TEXT,
    
    refunded_amount INTEGER DEFAULT 0,
    
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    
    CONSTRAINT valid_refund CHECK (refunded_amount >= 0 AND refunded_amount <= amount)
);

CREATE INDEX idx_payment_intents_invoice ON payment_intents(invoice_id);
CREATE INDEX idx_payment_intents_link ON payment_intents(payment_link_id);
CREATE INDEX idx_payment_intents_status ON payment_intents(status);
CREATE INDEX idx_payment_intents_aiia ON payment_intents(aiia_transaction_id) WHERE aiia_transaction_id IS NOT NULL;

-- Refunds (Remboursements)
CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_intent_id UUID NOT NULL REFERENCES payment_intents(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    
    amount INTEGER NOT NULL,
    currency VARCHAR(3) DEFAULT 'EUR',
    reason TEXT,
    
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'succeeded', 'failed')),
    
    aiia_refund_id VARCHAR(100),
    failure_reason TEXT,
    
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    
    CONSTRAINT valid_amount CHECK (amount > 0)
);

CREATE INDEX idx_refunds_payment_intent ON refunds(payment_intent_id);
CREATE INDEX idx_refunds_invoice ON refunds(invoice_id);
CREATE INDEX idx_refunds_status ON refunds(status);

-- Webhook Subscriptions
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    secret VARCHAR(100) NOT NULL, -- HMAC secret
    
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'disabled')),
    description TEXT,
    
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT valid_url CHECK (url ~ '^https?://')
);

CREATE INDEX idx_webhooks_org ON webhook_subscriptions(organization_id);
CREATE INDEX idx_webhooks_active ON webhook_subscriptions(status) WHERE status = 'active';

-- Webhook Deliveries (pour retry & monitoring)
CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id) ON DELETE CASCADE,
    
    event_type VARCHAR(50) NOT NULL,
    event_id UUID NOT NULL,
    payload JSONB NOT NULL,
    
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'delivered', 'failed')),
    attempts INTEGER DEFAULT 0,
    max_attempts INTEGER DEFAULT 5,
    
    last_attempt_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ,
    
    response_status INTEGER,
    response_body TEXT,
    error_message TEXT,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_webhook_deliveries_sub ON webhook_deliveries(subscription_id);
CREATE INDEX idx_webhook_deliveries_retry ON webhook_deliveries(next_retry_at) 
    WHERE status = 'pending' AND attempts < max_attempts;

-- Audit Logs
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    
    organization_id UUID REFERENCES organizations(id),
    
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
    
    user_id VARCHAR(100),
    user_type VARCHAR(20),
    ip_address INET,
    
    changes JSONB, -- {before: {...}, after: {...}}
    metadata JSONB DEFAULT '{}',
    
    -- Partition par mois pour performance
    PRIMARY KEY (timestamp, id)
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_org ON audit_logs(organization_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- Créer les partitions (automatiser en production)
CREATE TABLE audit_logs_2024_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE audit_logs_2024_03 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Triggers pour updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_organizations_updated_at BEFORE UPDATE ON organizations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_departments_updated_at BEFORE UPDATE ON departments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_invoices_updated_at BEFORE UPDATE ON invoices
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_payment_intents_updated_at BEFORE UPDATE ON payment_intents
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### Contraintes de cohérence

#### Trigger : Un seul compte bancaire par défaut
```sql
CREATE OR REPLACE FUNCTION check_single_default_bank_account()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.is_default = TRUE THEN
        UPDATE bank_accounts 
        SET is_default = FALSE 
        WHERE department_id = NEW.department_id 
          AND id != NEW.id 
          AND is_default = TRUE;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ensure_single_default_bank_account
    BEFORE INSERT OR UPDATE ON bank_accounts
    FOR EACH ROW
    WHEN (NEW.is_default = TRUE)
    EXECUTE FUNCTION check_single_default_bank_account();
```

#### Trigger : Mise à jour statut facture après paiement
```sql
CREATE OR REPLACE FUNCTION update_invoice_status_on_payment()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'succeeded' AND OLD.status != 'succeeded' THEN
        UPDATE invoices
        SET 
            amount_paid = amount_paid + NEW.amount,
            amount_due = amount_due - NEW.amount,
            status = CASE 
                WHEN amount_due - NEW.amount = 0 THEN 'paid'
                WHEN amount_due - NEW.amount < total_amount THEN 'partially_paid'
                ELSE status
            END,
            paid_at = CASE 
                WHEN amount_due - NEW.amount = 0 THEN NOW()
                ELSE paid_at
            END
        WHERE id = NEW.invoice_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_invoice_on_payment_success
    AFTER UPDATE ON payment_intents
    FOR EACH ROW
    WHEN (NEW.status = 'succeeded' AND OLD.status != 'succeeded')
    EXECUTE FUNCTION update_invoice_status_on_payment();
```

---

## Sécurité

### 1. Authentification JWT

#### Génération des tokens

```python
# Python avec PyJWT
from datetime import datetime, timedelta
import jwt
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

# Générer les clés RSA (une fois, à stocker dans Vault)
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)

public_key = private_key.public_key()

# Créer un token
def create_access_token(
    data: dict, 
    scope: str, 
    expires_delta: timedelta = timedelta(hours=1)
):
    to_encode = data.copy()
    expire = datetime.utcnow() + expires_delta
    
    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),
        "scope": scope,
        "jti": str(uuid.uuid4())  # Unique ID pour révocation
    })
    
    encoded_jwt = jwt.encode(
        to_encode, 
        private_key, 
        algorithm="RS256"
    )
    
    return encoded_jwt

# Valider un token
def verify_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token, 
            public_key, 
            algorithms=["RS256"]
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")
```

#### Structure du token

```json
{
  "sub": "partner_abc123",
  "scope": "partner",
  "entity_id": null,
  "iat": 1706774400,
  "exp": 1706778000,
  "jti": "550e8400-e29b-41d4-a716-446655440000"
}
```

Pour un token entity :
```json
{
  "sub": "partner_abc123",
  "scope": "entity",
  "entity_id": "org-uuid-here",
  "iat": 1706774400,
  "exp": 1706778000,
  "jti": "550e8400-e29b-41d4-a716-446655440000"
}
```

### 2. Chiffrement des données sensibles

```python
from cryptography.fernet import Fernet
import base64

class DataEncryption:
    def __init__(self, key: bytes):
        """key doit être 32 bytes (256 bits)"""
        self.fernet = Fernet(base64.urlsafe_b64encode(key))
    
    def encrypt(self, data: str) -> str:
        """Chiffre une chaîne"""
        return self.fernet.encrypt(data.encode()).decode()
    
    def decrypt(self, encrypted_data: str) -> str:
        """Déchiffre une chaîne"""
        return self.fernet.decrypt(encrypted_data.encode()).decode()

# Utilisation pour IBAN
encryptor = DataEncryption(key=get_encryption_key_from_vault())

# Stockage
encrypted_iban = encryptor.encrypt("FR7612345678901234567890123")
# Stocké en DB: "gAAAAABl..."

# Récupération
iban = encryptor.decrypt(encrypted_iban)
# "FR7612345678901234567890123"
```

### 3. Rate Limiting

```python
from redis import Redis
from datetime import datetime, timedelta

class RateLimiter:
    def __init__(self, redis_client: Redis):
        self.redis = redis_client
    
    def check_rate_limit(
        self, 
        key: str, 
        limit: int = 1000, 
        window: int = 60
    ) -> tuple[bool, dict]:
        """
        Rate limit avec sliding window
        
        Returns:
            (allowed, info) où info contient remaining, reset_at
        """
        now = datetime.utcnow()
        window_key = f"rate_limit:{key}:{now.minute}"
        
        pipe = self.redis.pipeline()
        pipe.incr(window_key)
        pipe.expire(window_key, window)
        current_count, _ = pipe.execute()
        
        remaining = max(0, limit - current_count)
        allowed = current_count <= limit
        
        reset_at = (now + timedelta(seconds=window)).isoformat()
        
        return allowed, {
            "limit": limit,
            "remaining": remaining,
            "reset": reset_at
        }

# Utilisation dans FastAPI
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_id = get_client_id_from_token(request)
    
    allowed, info = rate_limiter.check_rate_limit(
        key=f"client:{client_id}",
        limit=1000,
        window=60
    )
    
    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "rate_limit_exceeded"},
            headers={
                "X-RateLimit-Limit": str(info["limit"]),
                "X-RateLimit-Remaining": str(info["remaining"]),
                "X-RateLimit-Reset": info["reset"]
            }
        )
    
    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(info["limit"])
    response.headers["X-RateLimit-Remaining"] = str(info["remaining"])
    
    return response
```

---

## Intégration Aiia

### Configuration

```python
from typing import Dict, Any
import httpx

class AiiaClient:
    def __init__(
        self, 
        api_key: str, 
        base_url: str = "https://api.aiia.eu/v1"
    ):
        self.api_key = api_key
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            base_url=base_url,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json"
            },
            timeout=30.0
        )
    
    async def initiate_payment(
        self,
        amount: int,  # en centimes
        currency: str,
        debtor_iban: str,
        creditor_iban: str,
        creditor_name: str,
        payment_reference: str,
        redirect_url: str
    ) -> Dict[str, Any]:
        """
        Initie un virement SEPA via Aiia
        
        Returns:
            {
                "payment_id": "aiia-tx-123",
                "authorization_url": "https://auth.aiia.eu/...",
                "status": "initiated"
            }
        """
        payload = {
            "amount": {
                "value": amount,
                "currency": currency
            },
            "debtor": {
                "iban": debtor_iban
            },
            "creditor": {
                "iban": creditor_iban,
                "name": creditor_name
            },
            "remittance_information": payment_reference,
            "redirect_url": redirect_url
        }
        
        response = await self.client.post(
            "/payments/initiate",
            json=payload
        )
        response.raise_for_status()
        
        return response.json()
    
    async def get_payment_status(self, payment_id: str) -> Dict[str, Any]:
        """
        Récupère le statut d'un paiement
        
        Returns:
            {
                "payment_id": "aiia-tx-123",
                "status": "settled",  # initiated, authorized, settled, rejected
                "settlement_date": "2024-02-03"
            }
        """
        response = await self.client.get(f"/payments/{payment_id}")
        response.raise_for_status()
        
        return response.json()
    
    async def handle_webhook(self, payload: Dict[str, Any]) -> None:
        """
        Traite un webhook Aiia
        
        Webhook events:
        - payment.authorized
        - payment.settled
        - payment.rejected
        """
        event_type = payload.get("event")
        payment_id = payload.get("payment_id")
        
        if event_type == "payment.settled":
            await self._mark_payment_as_settled(payment_id)
        elif event_type == "payment.rejected":
            await self._mark_payment_as_failed(payment_id, payload.get("reason"))
```

### Gestion des états

```python
# Mapping des statuts Aiia → Payment Intent
AIIA_STATUS_MAPPING = {
    "initiated": "processing",
    "authorized": "processing",
    "settled": "succeeded",
    "rejected": "failed"
}

async def update_payment_intent_from_aiia(
    payment_intent_id: str,
    aiia_status: str,
    aiia_data: Dict[str, Any]
):
    """Mise à jour du Payment Intent depuis un webhook Aiia"""
    
    new_status = AIIA_STATUS_MAPPING.get(aiia_status)
    
    async with db.transaction():
        # Mettre à jour le Payment Intent
        await db.execute(
            """
            UPDATE payment_intents
            SET 
                status = $1,
                aiia_status = $2,
                updated_at = NOW(),
                completed_at = CASE WHEN $1 IN ('succeeded', 'failed') THEN NOW() ELSE completed_at END
            WHERE id = $3
            """,
            new_status, aiia_status, payment_intent_id
        )
        
        # Déclencher webhook client
        await trigger_webhook(
            event_type="payment_intent.status_changed",
            data={
                "payment_intent_id": payment_intent_id,
                "previous_status": "processing",
                "current_status": new_status
            }
        )
```

---

## Gestion des webhooks

### Architecture du système de webhooks

```python
import asyncio
from datetime import datetime, timedelta
from typing import Dict, Any
import httpx
import hmac
import hashlib

class WebhookDeliverySystem:
    def __init__(self, db, redis, queue):
        self.db = db
        self.redis = redis
        self.queue = queue  # RabbitMQ
    
    async def trigger_webhook(
        self,
        event_type: str,
        organization_id: str,
        data: Dict[str, Any]
    ):
        """Déclenche un webhook pour tous les abonnés pertinents"""
        
        # Récupérer les abonnements actifs pour cet événement
        subscriptions = await self.db.fetch(
            """
            SELECT id, url, secret, events
            FROM webhook_subscriptions
            WHERE organization_id = $1
              AND status = 'active'
              AND $2 = ANY(events)
            """,
            organization_id, event_type
        )
        
        event_id = str(uuid.uuid4())
        
        for sub in subscriptions:
            # Créer une entrée de delivery
            delivery_id = await self.db.fetchval(
                """
                INSERT INTO webhook_deliveries (
                    subscription_id,
                    event_type,
                    event_id,
                    payload,
                    next_retry_at
                )
                VALUES ($1, $2, $3, $4, NOW())
                RETURNING id
                """,
                sub['id'], event_type, event_id, json.dumps(data)
            )
            
            # Envoyer dans la queue pour traitement asynchrone
            await self.queue.publish(
                'webhook_deliveries',
                {
                    'delivery_id': str(delivery_id),
                    'subscription_id': str(sub['id']),
                    'url': sub['url'],
                    'secret': sub['secret'],
                    'payload': {
                        'event_id': event_id,
                        'event_type': event_type,
                        'timestamp': datetime.utcnow().isoformat(),
                        'organization_id': organization_id,
                        'data': data
                    }
                }
            )

class WebhookWorker:
    """Worker qui traite les webhooks de manière asynchrone"""
    
    MAX_RETRIES = 5
    RETRY_DELAYS = [5, 30, 300, 3600, 86400]  # 5s, 30s, 5m, 1h, 24h
    
    async def deliver_webhook(
        self,
        delivery_id: str,
        url: str,
        secret: str,
        payload: Dict[str, Any]
    ):
        """Envoie un webhook avec retry logic"""
        
        # Récupérer le nombre de tentatives
        delivery = await self.db.fetchrow(
            "SELECT attempts FROM webhook_deliveries WHERE id = $1",
            delivery_id
        )
        
        attempt = delivery['attempts']
        
        # Signer le payload
        payload_bytes = json.dumps(payload).encode()
        signature = hmac.new(
            secret.encode(),
            payload_bytes,
            hashlib.sha256
        ).hexdigest()
        
        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    url,
                    json=payload,
                    headers={
                        'Content-Type': 'application/json',
                        'X-Webhook-Signature': f'sha256={signature}',
                        'X-Webhook-ID': payload['event_id']
                    },
                    timeout=10.0
                )
                
                # Succès si 200-299
                if 200 <= response.status_code < 300:
                    await self._mark_delivered(
                        delivery_id,
                        response.status_code,
                        response.text
                    )
                else:
                    await self._mark_failed(
                        delivery_id,
                        attempt,
                        response.status_code,
                        response.text
                    )
                    
        except Exception as e:
            await self._mark_failed(
                delivery_id,
                attempt,
                error_message=str(e)
            )
    
    async def _mark_delivered(
        self,
        delivery_id: str,
        status_code: int,
        response_body: str
    ):
        await self.db.execute(
            """
            UPDATE webhook_deliveries
            SET 
                status = 'delivered',
                response_status = $2,
                response_body = $3,
                completed_at = NOW()
            WHERE id = $1
            """,
            delivery_id, status_code, response_body
        )
    
    async def _mark_failed(
        self,
        delivery_id: str,
        attempt: int,
        status_code: int = None,
        response_body: str = None,
        error_message: str = None
    ):
        next_attempt = attempt + 1
        
        if next_attempt < self.MAX_RETRIES:
            # Planifier le prochain retry
            delay_seconds = self.RETRY_DELAYS[next_attempt]
            next_retry_at = datetime.utcnow() + timedelta(seconds=delay_seconds)
            
            await self.db.execute(
                """
                UPDATE webhook_deliveries
                SET 
                    attempts = $2,
                    last_attempt_at = NOW(),
                    next_retry_at = $3,
                    response_status = $4,
                    response_body = $5,
                    error_message = $6
                WHERE id = $1
                """,
                delivery_id, next_attempt, next_retry_at,
                status_code, response_body, error_message
            )
        else:
            # Max retries atteint, marquer comme failed
            await self.db.execute(
                """
                UPDATE webhook_deliveries
                SET 
                    status = 'failed',
                    attempts = $2,
                    last_attempt_at = NOW(),
                    response_status = $3,
                    response_body = $4,
                    error_message = $5,
                    completed_at = NOW()
                WHERE id = $1
                """,
                delivery_id, next_attempt,
                status_code, response_body, error_message
            )
```

---

## Audit et logging

### Structure des audit logs

Chaque événement crée une entrée d'audit avec :
- Qui (user_id, user_type)
- Quoi (resource_type, resource_id, action)
- Quand (timestamp)
- Où (ip_address)
- Changements (before/after states)

### Implémentation

```python
class AuditLogger:
    def __init__(self, db):
        self.db = db
    
    async def log(
        self,
        resource_type: str,
        resource_id: str,
        action: str,
        user_id: str,
        user_type: str,
        ip_address: str,
        organization_id: str = None,
        changes: Dict[str, Any] = None,
        metadata: Dict[str, Any] = None
    ):
        """Crée un log d'audit"""
        
        await self.db.execute(
            """
            INSERT INTO audit_logs (
                timestamp,
                organization_id,
                resource_type,
                resource_id,
                action,
                user_id,
                user_type,
                ip_address,
                changes,
                metadata
            )
            VALUES (NOW(), $1, $2, $3, $4, $5, $6, $7, $8, $9)
            """,
            organization_id, resource_type, resource_id, action,
            user_id, user_type, ip_address,
            json.dumps(changes) if changes else None,
            json.dumps(metadata) if metadata else None
        )

# Utilisation avec décorateur FastAPI
def audit_log(resource_type: str, action: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            request = kwargs.get('request')
            
            # Exécuter la fonction
            result = await func(*args, **kwargs)
            
            # Logger après succès
            await audit_logger.log(
                resource_type=resource_type,
                resource_id=result.get('id'),
                action=action,
                user_id=request.state.user_id,
                user_type=request.state.user_type,
                ip_address=request.client.host,
                organization_id=request.headers.get('X-Entity-Id'),
                changes={
                    'before': None,
                    'after': result
                }
            )
            
            return result
        
        return wrapper
    return decorator

# Utilisation
@app.post("/invoices")
@audit_log(resource_type="invoice", action="created")
async def create_invoice(request: Request, data: InvoiceCreate):
    invoice = await invoice_service.create(data)
    return invoice
```

---

## Performance et scalabilité

### Optimisations base de données

1. **Indexes** : Créés sur toutes les colonnes de recherche fréquente
2. **Partitioning** : audit_logs partitionné par mois
3. **Connection pooling** : PgBouncer (200 connexions max)
4. **Read replicas** : Pour requêtes en lecture seule

### Caching strategy

```python
# Cache des tokens valides
cache.set(f"token:{token_hash}", user_data, ttl=60)

# Cache des données organisation
cache.set(f"org:{org_id}", org_data, ttl=300)

# Cache des webhooks actifs
cache.set(f"webhooks:active:{org_id}", webhooks, ttl=60)
```

### Monitoring

```python
from prometheus_client import Counter, Histogram, Gauge

# Métriques API
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

api_request_duration = Histogram(
    'api_request_duration_seconds',
    'API request duration',
    ['method', 'endpoint']
)

# Métriques métier
payments_created = Counter(
    'payments_created_total',
    'Total payment intents created',
    ['status']
)

webhook_deliveries = Counter(
    'webhook_deliveries_total',
    'Total webhook deliveries',
    ['status']
)

active_payment_intents = Gauge(
    'active_payment_intents',
    'Number of active payment intents'
)
```

---

## Disaster Recovery

### Stratégie de backup

1. **Base de données** :
   - Backup automatique quotidien (rétention 30 jours)
   - Point-in-time recovery (PITR) activé
   - Réplication cross-region

2. **Fichiers** :
   - S3 avec versioning activé
   - Réplication cross-region
   - Lifecycle policy (archivage après 90 jours)

3. **Secrets** :
   - Vault backup automatique
   - Clés chiffrées et stockées en coffre-fort physique

### Plan de continuité

- **RTO** (Recovery Time Objective) : 4 heures
- **RPO** (Recovery Point Objective) : 15 minutes

### Tests de recovery

Effectuer trimestriellement :
1. Restauration complète de la BDD
2. Failover sur région secondaire
3. Restauration des secrets

---

## Conformité réglementaire

### RGPD

1. **Données personnelles stockées** :
   - Nom, email, téléphone (payeurs)
   - IBAN (chiffré)
   - Adresses
   - IP addresses (logs)

2. **Droits des utilisateurs** :
   - Accès : API GET /data-subject-access-request
   - Rectification : API PATCH
   - Suppression : API DELETE avec anonymisation
   - Portabilité : Export JSON

3. **Durée de conservation** :
   - Factures : 7 ans (obligation légale)
   - Logs d'audit : 7 ans
   - Logs techniques : 1 an

### PSD2

1. **Strong Customer Authentication** :
   - Délégué à Aiia (certifié PSD2)
   - 2FA via banque de l'utilisateur

2. **Traçabilité** :
   - Tous les virements tracés dans audit_logs
   - Conservation 7 ans

### LCB-FT (Lutte anti-blanchiment)

1. **KYC** :
   - Vérification SIRET via API INSEE
   - Validation identité organisations

2. **Monitoring** :
   - Alertes montants > 10 000€
   - Alertes transactions multiples
   - Rapports mensuels

---

## Checklist de mise en production

- [ ] SSL/TLS 1.3 configuré
- [ ] Secrets dans Vault
- [ ] Rate limiting activé
- [ ] Monitoring Prometheus + Grafana
- [ ] Logs centralisés (ELK)
- [ ] Backups automatiques configurés
- [ ] Tests de charge réussis (1000 req/s)
- [ ] DR testée
- [ ] Webhooks de test validés
- [ ] Intégration Aiia production testée
- [ ] Documentation à jour
- [ ] Runbooks incidents créés
- [ ] Astreinte configurée

---

## Contacts techniques

- **Architecture** : architecture@modupay.fr
- **Sécurité** : security@modupay.fr
- **DevOps** : devops@modupay.fr
