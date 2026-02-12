---
title: Exemples pratiques
layout: default
nav_order: 5
has_toc: true
---

# Exemples Pratiques et Cas d'Usage

Ce document présente des exemples concrets d'utilisation de l'API avec des scénarios réels du marché français.

---

## 📑 Table des matières

1. [Cas d'usage : Mairie](#cas-1-mairie)
2. [Cas d'usage : Entreprise SaaS](#cas-2-entreprise-saas)
3. [Cas d'usage : Plateforme de paiement](#cas-3-plateforme-de-paiement)
4. [Recettes prêtes à l'emploi](#recettes)
5. [Gestion d'erreurs](#gestion-derreurs)
6. [Optimisations](#optimisations)

---

## Cas #1 : Mairie - Factures de Régie d'Eau

### Contexte

La Mairie de Lyon souhaite digitaliser la facturation de sa régie d'eau avec :
- Facturation trimestrielle de 50 000 foyers
- Paiement par virement SEPA
- Rappels automatiques pour impayés
- Historique de 7 ans pour audit

### Architecture

```
Mairie de Lyon (Organization)
  └─ Régie de l'Eau (Department)
      ├─ IBAN: FR7612345678901234567890123
      └─ 50 000 Factures trimestrielles
          └─ Liens de paiement avec QR codes
```

### Implémentation

#### 1. Initialisation (une seule fois)

```python
import requests

API_BASE = "https://api.votreplateforme.fr/v1"
TOKEN = "votre_token_partenaire"

# Créer l'organisation Mairie
mairie = requests.post(
    f"{API_BASE}/organizations",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={
        "type": "public_entity",
        "name": "Mairie de Lyon",
        "legal_name": "Ville de Lyon",
        "siret": "21690123400019",
        "vat_number": "FR21216901234",
        "address": {
            "line1": "1 Place de la Comédie",
            "postal_code": "69001",
            "city": "Lyon",
            "country": "FR"
        },
        "contact": {
            "email": "regie.eau@mairie-lyon.fr",
            "phone": "+33472101010"
        }
    }
).json()

mairie_id = mairie['id']

# Créer la régie d'eau
regie_eau = requests.post(
    f"{API_BASE}/organizations/{mairie_id}/departments",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={
        "name": "Régie de l'Eau",
        "code": "REG-EAU-001",
        "description": "Gestion de la distribution d'eau potable"
    }
).json()

regie_id = regie_eau['id']

# Ajouter le compte bancaire
compte = requests.post(
    f"{API_BASE}/departments/{regie_id}/bank_accounts",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={
        "iban": "FR7612345678901234567890123",
        "bic": "BNPAFRPPXXX",
        "account_name": "Régie Eau - Ville de Lyon",
        "bank_name": "BNP Paribas",
        "is_default": True
    }
).json()
```

#### 2. Facturation trimestrielle (automatisée)

```python
import csv
from datetime import date, timedelta

# Lecture du fichier de consommation
with open('consommations_Q1_2024.csv', 'r') as f:
    reader = csv.DictReader(f)
    
    for row in reader:
        # Calcul du montant (tarif progressif)
        consommation = int(row['m3_consommes'])
        
        if consommation <= 50:
            montant_ht = consommation * 320  # 3.20€/m³
        elif consommation <= 100:
            montant_ht = 50 * 320 + (consommation - 50) * 380
        else:
            montant_ht = 50 * 320 + 50 * 380 + (consommation - 100) * 450
        
        # Créer la facture
        facture = requests.post(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": mairie_id
            },
            json={
                "department_id": regie_id,
                "payer": {
                    "name": row['nom_complet'],
                    "email": row['email'],
                    "address": {
                        "line1": row['adresse'],
                        "postal_code": row['code_postal'],
                        "city": row['ville'],
                        "country": "FR"
                    }
                },
                "line_items": [
                    {
                        "description": f"Consommation eau Q1 2024 - {consommation} m³",
                        "quantity": consommation,
                        "unit_price": 320,  # Simplifié
                        "vat_rate": 5.5
                    },
                    {
                        "description": "Abonnement trimestriel",
                        "quantity": 1,
                        "unit_price": 1500,  # 15€
                        "vat_rate": 5.5
                    }
                ],
                "due_date": (date.today() + timedelta(days=30)).isoformat(),
                "payment_terms": "Paiement sous 30 jours"
            }
        ).json()
        
        # Émettre la facture
        requests.post(
            f"{API_BASE}/invoices/{facture['id']}/issue",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": mairie_id
            }
        )
        
        # Créer le lien de paiement
        payment_link = requests.post(
            f"{API_BASE}/payment_links",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": mairie_id
            },
            json={
                "invoice_id": facture['id'],
                "payment_methods": ["sepa_credit"],
                "expires_at": (date.today() + timedelta(days=45)).isoformat() + "T23:59:59Z"
            }
        ).json()
        
        # Envoyer l'email avec le lien de paiement
        send_email(
            to=row['email'],
            subject=f"Facture eau Q1 2024 - {facture['invoice_number']}",
            body=f"""
            Bonjour {row['nom_complet']},
            
            Votre facture d'eau pour le trimestre Q1 2024 est disponible.
            
            Montant : {facture['total_amount'] / 100:.2f} €
            Date limite : {facture['due_date']}
            
            Pour payer en ligne :
            {payment_link['payment_page_url']}
            
            Ou scannez ce QR code :
            [QR CODE]
            
            Cordialement,
            Régie de l'Eau - Ville de Lyon
            """
        )
        
        print(f"✓ Facture {facture['invoice_number']} créée pour {row['nom_complet']}")
```

#### 3. Gestion des rappels automatiques

```python
from datetime import date

# Récupérer les factures en retard
factures_retard = requests.get(
    f"{API_BASE}/invoices",
    headers={
        "Authorization": f"Bearer {TOKEN}",
        "X-Entity-Id": mairie_id
    },
    params={
        "status": "overdue",
        "department_id": regie_id
    }
).json()

for facture in factures_retard['data']:
    jours_retard = (date.today() - date.fromisoformat(facture['due_date'])).days
    
    # Premier rappel à J+7
    if jours_retard == 7:
        send_reminder_email(facture, niveau=1)
    
    # Deuxième rappel à J+14
    elif jours_retard == 14:
        send_reminder_email(facture, niveau=2)
    
    # Mise en demeure à J+30
    elif jours_retard == 30:
        send_formal_notice(facture)

def send_reminder_email(facture, niveau):
    # Récupérer le lien de paiement existant
    payment_links = requests.get(
        f"{API_BASE}/payment_links",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": mairie_id
        },
        params={"invoice_id": facture['id']}
    ).json()
    
    link = payment_links['data'][0]['payment_page_url']
    
    send_email(
        to=facture['payer']['email'],
        subject=f"Rappel {niveau} - Facture {facture['invoice_number']}",
        body=f"""
        Bonjour,
        
        Nous constatons que votre facture n'a pas encore été réglée.
        
        Facture : {facture['invoice_number']}
        Montant : {facture['amount_due'] / 100:.2f} €
        Échéance : {facture['due_date']}
        Retard : {(date.today() - date.fromisoformat(facture['due_date'])).days} jours
        
        Merci de régulariser votre situation :
        {link}
        
        Cordialement
        """
    )
```

#### 4. Webhook pour mise à jour automatique

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "whsec_K8W5m3n2k4J9p7Q1r6T8v5Y2z4"

@app.route('/webhooks/payment-api', methods=['POST'])
def handle_webhook():
    # Vérifier la signature
    signature = request.headers.get('X-Webhook-Signature')
    payload = request.get_data()
    
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(f"sha256={expected}", signature):
        return 'Invalid signature', 401
    
    event = request.get_json()
    
    # Traiter les paiements réussis
    if event['event_type'] == 'payment_intent.status_changed':
        if event['data']['current_status'] == 'succeeded':
            invoice_id = event['data']['payment_intent']['invoice_id']
            
            # Mettre à jour le système de gestion interne
            update_internal_system(invoice_id, status='paid')
            
            # Envoyer un email de confirmation
            send_payment_confirmation(invoice_id)
    
    return 'OK', 200
```

---

## Cas #2 : Entreprise SaaS - Abonnements mensuels

### Contexte

Une plateforme SaaS B2B avec :
- 1 000 clients entreprises
- Facturation mensuelle automatique
- Plusieurs formules (Starter, Pro, Enterprise)
- Paiement par carte ou virement

### Implémentation

#### 1. Configuration des plans

```python
PLANS = {
    'starter': {
        'price': 4900,  # 49€
        'description': 'Plan Starter - 10 utilisateurs',
        'vat_rate': 20.0
    },
    'pro': {
        'price': 14900,  # 149€
        'description': 'Plan Pro - 50 utilisateurs',
        'vat_rate': 20.0
    },
    'enterprise': {
        'price': 49900,  # 499€
        'description': 'Plan Enterprise - Utilisateurs illimités',
        'vat_rate': 20.0
    }
}
```

#### 2. Facturation mensuelle automatisée

```python
import schedule
import time
from datetime import date, timedelta

def facturer_abonnements_mensuels():
    """Exécuté le 1er de chaque mois"""
    
    # Récupérer tous les clients actifs
    clients = get_active_subscriptions_from_db()
    
    for client in clients:
        # Créer la facture
        facture = requests.post(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            },
            json={
                "department_id": dept_facturation_id,
                "payer": {
                    "name": client['company_name'],
                    "email": client['billing_email'],
                    "siret": client['siret']
                },
                "line_items": [
                    {
                        "description": PLANS[client['plan']]['description'],
                        "quantity": 1,
                        "unit_price": PLANS[client['plan']]['price'],
                        "vat_rate": PLANS[client['plan']]['vat_rate']
                    }
                ],
                "due_date": (date.today() + timedelta(days=15)).isoformat(),
                "payment_terms": "Paiement sous 15 jours"
            }
        ).json()
        
        # Émettre
        requests.post(
            f"{API_BASE}/invoices/{facture['id']}/issue",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            }
        )
        
        # Créer le lien de paiement
        payment_link = requests.post(
            f"{API_BASE}/payment_links",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            },
            json={
                "invoice_id": facture['id'],
                "payment_methods": ["card", "sepa_credit"],
                "return_url": f"https://app.votresaas.fr/billing/success"
            }
        ).json()
        
        # Envoyer l'email
        send_invoice_email(client, facture, payment_link)
        
        # Logger dans la base de données
        save_invoice_to_db(client['id'], facture['id'])

# Planifier l'exécution mensuelle
schedule.every().month.at("00:00").do(facturer_abonnements_mensuels)

while True:
    schedule.run_pending()
    time.sleep(3600)  # Vérifier chaque heure
```

#### 3. Changement de plan avec prorata

```python
def upgrade_plan(customer_id: str, new_plan: str):
    """Changement de plan avec calcul prorata"""
    
    customer = get_customer_from_db(customer_id)
    old_plan = customer['current_plan']
    
    # Calculer le prorata
    days_remaining = (
        date.fromisoformat(customer['next_billing_date']) - date.today()
    ).days
    
    days_in_month = 30
    
    old_price = PLANS[old_plan]['price']
    new_price = PLANS[new_plan]['price']
    
    # Crédit pour l'ancien plan
    credit = (old_price * days_remaining) / days_in_month
    
    # Coût du nouveau plan (prorata)
    charge = (new_price * days_remaining) / days_in_month
    
    # Montant à facturer
    amount_to_charge = int(charge - credit)
    
    if amount_to_charge > 0:
        # Créer une facture de régularisation
        facture = requests.post(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            },
            json={
                "department_id": dept_facturation_id,
                "payer": {
                    "name": customer['company_name'],
                    "email": customer['billing_email']
                },
                "line_items": [
                    {
                        "description": f"Upgrade {old_plan} → {new_plan} (prorata)",
                        "quantity": 1,
                        "unit_price": amount_to_charge,
                        "vat_rate": 20.0
                    }
                ],
                "due_date": date.today().isoformat()
            }
        ).json()
        
        # Émettre et créer le lien de paiement
        requests.post(
            f"{API_BASE}/invoices/{facture['id']}/issue",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            }
        )
        
        payment_link = requests.post(
            f"{API_BASE}/payment_links",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": notre_org_id
            },
            json={
                "invoice_id": facture['id'],
                "payment_methods": ["card"]
            }
        ).json()
        
        return payment_link['payment_page_url']
```

---

## Cas #3 : Plateforme de paiement multi-marchands

### Contexte

Une marketplace qui :
- Héberge 500 marchands
- Gère les paiements pour eux
- Prend une commission de 2%
- Reverse aux marchands

### Architecture

```python
# Organisation = Marchand
# Department = Catégorie de produits (optionnel)
# Invoice = Commande client
# Payment Intent = Paiement client
# Notre commission = Calculée et facturée séparément
```

### Implémentation

#### 1. Onboarding marchand

```python
def onboard_merchant(merchant_data: dict):
    """Créer une organisation pour un nouveau marchand"""
    
    # Créer l'organisation
    org = requests.post(
        f"{API_BASE}/organizations",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={
            "type": "company",
            "name": merchant_data['company_name'],
            "siret": merchant_data['siret'],
            "vat_number": merchant_data['vat_number'],
            "address": merchant_data['address'],
            "contact": {
                "email": merchant_data['email'],
                "phone": merchant_data['phone']
            }
        }
    ).json()
    
    # Créer un département par défaut
    dept = requests.post(
        f"{API_BASE}/organizations/{org['id']}/departments",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={
            "name": "Ventes",
            "code": "SALES-001"
        }
    ).json()
    
    # Ajouter l'IBAN du marchand
    bank_account = requests.post(
        f"{API_BASE}/departments/{dept['id']}/bank_accounts",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={
            "iban": merchant_data['iban'],
            "bic": merchant_data['bic'],
            "account_name": merchant_data['company_name'],
            "is_default": True
        }
    ).json()
    
    # Sauvegarder dans notre BDD
    save_merchant_to_db({
        'merchant_id': org['id'],
        'department_id': dept['id'],
        'bank_account_id': bank_account['id'],
        'commission_rate': 0.02  # 2%
    })
    
    return org['id']
```

#### 2. Traiter une commande

```python
def process_order(order_data: dict):
    """Traiter une commande client"""
    
    merchant = get_merchant_from_db(order_data['merchant_id'])
    
    # Montant total de la commande
    total_amount = order_data['total_amount']
    
    # Notre commission (2%)
    commission = int(total_amount * merchant['commission_rate'])
    
    # Montant net pour le marchand
    merchant_amount = total_amount - commission
    
    # Créer la facture pour le client
    invoice = requests.post(
        f"{API_BASE}/invoices",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": merchant['merchant_id']
        },
        json={
            "department_id": merchant['department_id'],
            "payer": order_data['customer'],
            "line_items": order_data['items'],
            "due_date": date.today().isoformat(),
            "metadata": {
                "order_id": order_data['order_id'],
                "commission": commission,
                "merchant_amount": merchant_amount
            }
        }
    ).json()
    
    # Émettre
    requests.post(
        f"{API_BASE}/invoices/{invoice['id']}/issue",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": merchant['merchant_id']
        }
    )
    
    # Créer le lien de paiement
    payment_link = requests.post(
        f"{API_BASE}/payment_links",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": merchant['merchant_id']
        },
        json={
            "invoice_id": invoice['id'],
            "payment_methods": ["card", "sepa_credit"],
            "return_url": f"https://marketplace.fr/orders/{order_data['order_id']}/success"
        }
    ).json()
    
    return {
        "invoice_id": invoice['id'],
        "payment_url": payment_link['payment_page_url'],
        "commission": commission,
        "merchant_amount": merchant_amount
    }
```

#### 3. Facturer notre commission

```python
import schedule

def facturer_commissions_mensuelles():
    """Facturer nos commissions aux marchands (le 1er du mois)"""
    
    merchants = get_all_merchants_from_db()
    
    for merchant in merchants:
        # Calculer la commission du mois
        invoices_du_mois = requests.get(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": merchant['merchant_id']
            },
            params={
                "status": "paid",
                "paid_at_from": first_day_of_month(),
                "paid_at_to": last_day_of_month()
            }
        ).json()
        
        total_commission = sum(
            inv['metadata'].get('commission', 0)
            for inv in invoices_du_mois['data']
        )
        
        if total_commission > 0:
            # Créer une facture de commission
            # (dans notre propre organisation)
            commission_invoice = requests.post(
                f"{API_BASE}/invoices",
                headers={
                    "Authorization": f"Bearer {TOKEN}",
                    "X-Entity-Id": notre_marketplace_org_id
                },
                json={
                    "department_id": notre_dept_commissions_id,
                    "payer": {
                        "name": merchant['company_name'],
                        "email": merchant['email'],
                        "siret": merchant['siret']
                    },
                    "line_items": [
                        {
                            "description": f"Commission plateforme - {len(invoices_du_mois['data'])} ventes",
                            "quantity": 1,
                            "unit_price": total_commission,
                            "vat_rate": 20.0
                        }
                    ],
                    "due_date": (date.today() + timedelta(days=15)).isoformat()
                }
            ).json()
            
            # Émettre et envoyer
            requests.post(
                f"{API_BASE}/invoices/{commission_invoice['id']}/issue",
                headers={
                    "Authorization": f"Bearer {TOKEN}",
                    "X-Entity-Id": notre_marketplace_org_id
                }
            )
            
            payment_link = requests.post(
                f"{API_BASE}/payment_links",
                headers={
                    "Authorization": f"Bearer {TOKEN}",
                    "X-Entity-Id": notre_marketplace_org_id
                },
                json={
                    "invoice_id": commission_invoice['id'],
                    "payment_methods": ["sepa_credit"]
                }
            ).json()
            
            send_commission_invoice_email(merchant, commission_invoice, payment_link)

schedule.every().month.at("02:00").do(facturer_commissions_mensuelles)
```

---

## Recettes prêtes à l'emploi

### Recette 1 : Vérifier qu'une facture est payée

```python
def is_invoice_paid(invoice_id: str, entity_id: str) -> bool:
    """Vérifie si une facture est entièrement payée"""
    
    invoice = requests.get(
        f"{API_BASE}/invoices/{invoice_id}",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        }
    ).json()
    
    return invoice['status'] == 'paid'
```

### Recette 2 : Obtenir le statut temps réel d'un paiement

```python
def get_payment_status(payment_intent_id: str, entity_id: str) -> dict:
    """Récupère le statut détaillé d'un paiement"""
    
    payment_intent = requests.get(
        f"{API_BASE}/payment_intents/{payment_intent_id}",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        }
    ).json()
    
    return {
        'status': payment_intent['status'],
        'amount': payment_intent['amount'] / 100,
        'method': payment_intent.get('selected_payment_method'),
        'aiia_status': payment_intent.get('aiia_status'),
        'completed': payment_intent['status'] in ['succeeded', 'failed']
    }
```

### Recette 3 : Créer un avoir (remboursement total)

```python
def create_credit_note(invoice_id: str, entity_id: str, reason: str) -> dict:
    """Crée un avoir en annulant une facture payée"""
    
    # Récupérer la facture
    invoice = requests.get(
        f"{API_BASE}/invoices/{invoice_id}",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        }
    ).json()
    
    if invoice['status'] != 'paid':
        raise ValueError("La facture doit être payée pour créer un avoir")
    
    # Récupérer le payment intent
    payment_intents = requests.get(
        f"{API_BASE}/payment_intents",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        },
        params={"invoice_id": invoice_id}
    ).json()
    
    payment_intent = next(
        pi for pi in payment_intents['data']
        if pi['status'] == 'succeeded'
    )
    
    # Créer le remboursement
    refund = requests.post(
        f"{API_BASE}/refunds",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        },
        json={
            "payment_intent_id": payment_intent['id'],
            "reason": reason
        }
    ).json()
    
    # Annuler la facture
    requests.post(
        f"{API_BASE}/invoices/{invoice_id}/cancel",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "X-Entity-Id": entity_id
        },
        json={"reason": reason}
    )
    
    return {
        'refund_id': refund['id'],
        'amount': refund['amount'] / 100,
        'status': refund['status']
    }
```

### Recette 4 : Exporter les factures pour comptabilité

```python
import csv
from datetime import datetime

def export_invoices_for_accounting(entity_id: str, month: str):
    """Exporte les factures au format CSV pour import comptable"""
    
    # Format: month = "2024-02"
    year, month_num = month.split('-')
    
    from_date = f"{year}-{month_num}-01"
    
    # Dernier jour du mois
    if month_num == '12':
        to_date = f"{int(year)+1}-01-01"
    else:
        to_date = f"{year}-{int(month_num)+1:02d}-01"
    
    # Récupérer toutes les factures du mois
    invoices = []
    page = 1
    
    while True:
        response = requests.get(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": entity_id
            },
            params={
                "issue_date_from": from_date,
                "issue_date_to": to_date,
                "status": "paid",
                "page": page,
                "page_size": 100
            }
        ).json()
        
        invoices.extend(response['data'])
        
        if page >= response['pagination']['total_pages']:
            break
        
        page += 1
    
    # Écrire le CSV
    filename = f"export_factures_{month}.csv"
    
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=[
            'numero', 'date_emission', 'date_paiement', 'client',
            'montant_ht', 'montant_tva', 'montant_ttc', 'statut'
        ])
        
        writer.writeheader()
        
        for inv in invoices:
            writer.writerow({
                'numero': inv['invoice_number'],
                'date_emission': inv['issue_date'],
                'date_paiement': inv['paid_at'][:10] if inv.get('paid_at') else '',
                'client': inv['payer']['name'],
                'montant_ht': inv['subtotal_amount'] / 100,
                'montant_tva': inv['vat_amount'] / 100,
                'montant_ttc': inv['total_amount'] / 100,
                'statut': inv['status']
            })
    
    return filename
```

### Recette 5 : Calculer les statistiques de paiement

```python
def get_payment_statistics(entity_id: str, month: str) -> dict:
    """Calcule les statistiques de paiement pour un mois"""
    
    invoices = get_invoices_for_month(entity_id, month)
    
    total = len(invoices)
    paid = sum(1 for inv in invoices if inv['status'] == 'paid')
    overdue = sum(1 for inv in invoices if inv['status'] == 'overdue')
    pending = sum(1 for inv in invoices if inv['status'] in ['issued', 'partially_paid'])
    
    total_amount = sum(inv['total_amount'] for inv in invoices)
    paid_amount = sum(inv['amount_paid'] for inv in invoices)
    due_amount = sum(inv['amount_due'] for inv in invoices)
    
    # Délai moyen de paiement
    payment_delays = []
    for inv in invoices:
        if inv['status'] == 'paid' and inv.get('paid_at'):
            issue = datetime.fromisoformat(inv['issue_date'])
            paid = datetime.fromisoformat(inv['paid_at'][:10])
            delay = (paid - issue).days
            payment_delays.append(delay)
    
    avg_delay = sum(payment_delays) / len(payment_delays) if payment_delays else 0
    
    return {
        'total_invoices': total,
        'paid_count': paid,
        'overdue_count': overdue,
        'pending_count': pending,
        'payment_rate': (paid / total * 100) if total > 0 else 0,
        'total_amount': total_amount / 100,
        'paid_amount': paid_amount / 100,
        'due_amount': due_amount / 100,
        'avg_payment_delay_days': round(avg_delay, 1)
    }
```

---

## Gestion d'erreurs

### Erreurs courantes et solutions

```python
import requests
from requests.exceptions import HTTPError

def create_invoice_with_error_handling(entity_id: str, invoice_data: dict):
    """Création de facture avec gestion d'erreurs robuste"""
    
    try:
        response = requests.post(
            f"{API_BASE}/invoices",
            headers={
                "Authorization": f"Bearer {TOKEN}",
                "X-Entity-Id": entity_id
            },
            json=invoice_data,
            timeout=10
        )
        response.raise_for_status()
        return response.json()
        
    except HTTPError as e:
        if e.response.status_code == 400:
            # Erreur de validation
            error = e.response.json()
            print(f"❌ Données invalides : {error['message']}")
            
            if 'details' in error:
                for field, messages in error['details'].items():
                    print(f"  - {field}: {messages}")
            
            return None
            
        elif e.response.status_code == 401:
            # Token expiré
            print("🔑 Token expiré, renouvellement...")
            refresh_token()
            return create_invoice_with_error_handling(entity_id, invoice_data)
            
        elif e.response.status_code == 404:
            # Ressource introuvable
            print(f"❌ Organisation {entity_id} introuvable")
            return None
            
        elif e.response.status_code == 429:
            # Rate limit
            reset_time = e.response.headers.get('X-RateLimit-Reset')
            print(f"⏱️ Rate limit atteint, retry après {reset_time}")
            time.sleep(60)
            return create_invoice_with_error_handling(entity_id, invoice_data)
            
        elif e.response.status_code >= 500:
            # Erreur serveur
            print(f"🚨 Erreur serveur {e.response.status_code}, retry...")
            time.sleep(5)
            return create_invoice_with_error_handling(entity_id, invoice_data)
            
        else:
            print(f"❌ Erreur inattendue : {e}")
            raise
            
    except requests.exceptions.Timeout:
        print("⏱️ Timeout, nouvelle tentative...")
        return create_invoice_with_error_handling(entity_id, invoice_data)
        
    except requests.exceptions.ConnectionError:
        print("🌐 Erreur de connexion, nouvelle tentative...")
        time.sleep(2)
        return create_invoice_with_error_handling(entity_id, invoice_data)
```

---

## Optimisations

### 1. Batch processing pour volumes importants

```python
import concurrent.futures
from typing import List

def create_invoices_batch(entity_id: str, invoices_data: List[dict]):
    """Création de factures en parallèle pour optimiser les performances"""
    
    def create_single_invoice(invoice_data):
        try:
            return create_invoice_with_error_handling(entity_id, invoice_data)
        except Exception as e:
            return {"error": str(e), "data": invoice_data}
    
    # Traiter en parallèle (max 10 threads)
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(create_single_invoice, invoices_data))
    
    # Séparer succès et échecs
    successes = [r for r in results if r and 'error' not in r]
    failures = [r for r in results if r and 'error' in r]
    
    return {
        'success_count': len(successes),
        'failure_count': len(failures),
        'successes': successes,
        'failures': failures
    }

# Utilisation
invoices_to_create = [...]  # 1000 factures
results = create_invoices_batch(entity_id, invoices_to_create)
print(f"✅ {results['success_count']} factures créées")
print(f"❌ {results['failure_count']} échecs")
```

### 2. Cache pour réduire les appels API

```python
from functools import lru_cache
from datetime import datetime, timedelta

class CachedAPIClient:
    def __init__(self):
        self.cache_ttl = 300  # 5 minutes
        self._cache = {}
    
    def _get_cache_key(self, *args):
        return ':'.join(str(arg) for arg in args)
    
    def _is_cache_valid(self, timestamp):
        return datetime.now() - timestamp < timedelta(seconds=self.cache_ttl)
    
    @lru_cache(maxsize=100)
    def get_organization(self, org_id: str):
        """Récupère une organisation avec cache"""
        
        cache_key = self._get_cache_key('org', org_id)
        
        if cache_key in self._cache:
            cached_data, timestamp = self._cache[cache_key]
            if self._is_cache_valid(timestamp):
                return cached_data
        
        # Cache miss ou expiré
        org = requests.get(
            f"{API_BASE}/organizations/{org_id}",
            headers={"Authorization": f"Bearer {TOKEN}"}
        ).json()
        
        self._cache[cache_key] = (org, datetime.now())
        return org

# Utilisation
client = CachedAPIClient()

# Premier appel : requête API
org1 = client.get_organization("org-123")

# Deuxième appel : depuis le cache
org2 = client.get_organization("org-123")  # Pas d'appel API
```

---

Ces exemples couvrent les cas d'usage les plus courants de l'API. N'hésitez pas à les adapter à vos besoins spécifiques !
