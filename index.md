---
title: Accueil
layout: home
nav_order: 1
---

# Modupay — Documentation API v1.0
{: .fs-9 }

API de paiement de factures B2C en marque blanche pour le marché français. Gérez vos organisations, factures et paiements via une seule API REST.
{: .fs-6 .fw-300 }

[Vue d'ensemble]({{ site.baseurl }}/docs/overview){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Démarrage rapide]({{ site.baseurl }}/docs/integration-guide){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }

---

## Présentation

**Modupay** est une API REST pensée pour le marché français, permettant aux intégrateurs de proposer en marque blanche une solution complète de facturation et de paiement.

### Fonctionnalités principales

| Fonctionnalité | Description |
|:---|:---|
| 🏢 **Multi-organisations** | Entreprises et collectivités publiques |
| 🧾 **Facturation** | Création, émission et suivi des factures |
| 🔗 **Liens de paiement** | URLs sécurisées + QR codes |
| 💳 **Virements SEPA** | Intégration Aiia (Open Banking) |
| 💰 **Remboursements** | Partiels ou totaux |
| 🔔 **Webhooks** | Événements temps réel (HMAC-SHA256) |
| 📋 **Audit logs** | Traçabilité 7 ans (conformité fiscale) |
| 🔐 **Sécurité** | JWT RS256, RLS PostgreSQL, RGPD |

---

## Stack technique

```
Backend  : Node.js 18 · NestJS · Prisma ORM
Base     : PostgreSQL 15 (RLS multi-tenant) · Redis
Infra    : AWS Fargate · RDS · ElastiCache · S3
Sécurité : JWT RS256 · AES-256 · TLS 1.3
```

---

## Environnements

| Environnement | URL de base |
|:---|:---|
| **Sandbox** | `https://api.sandbox.votreplateforme.fr/v1` |
| **Production** | `https://api.votreplateforme.fr/v1` |

---

## Statut de conformité

{: .note }
> ✅ **RGPD** — Chiffrement des données, audit logs 7 ans, droit à l'oubli  
> ✅ **PSD2** — Strong Customer Authentication via Aiia  
> ✅ **LCB-FT** — Vérification SIRET/SIREN, monitoring des transactions  
> ⏳ **DORA** — Anticipé  
> ⏳ **RGS** — En cours d'évaluation
