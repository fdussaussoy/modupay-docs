---
title: Vue d'ensemble
layout: default
nav_order: 2
has_children: false
---

# Vue d'ensemble
{: .no_toc }

## Sommaire
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Architecture Modupay V1

Modupay est un **monolithe modulaire** conçu pour évoluer vers microservices sans dette technique initiale.

| Axe | Décision |
|:---|:---|
| **Type d'architecture** | Monolithe modulaire → microservices |
| **Backend** | Node.js / NestJS + Prisma, Clean Architecture |
| **Frontend** | Next.js 14 (SSR/SPA), personnalisation par tenant |
| **Base de données** | AWS RDS PostgreSQL avec RLS (multi-tenant) + Redis |
| **Infrastructure** | AWS Fargate, ALB, CDN, S3 |
| **Sécurité** | RGPD ✅ · DORA ⏳ · RGS ⏳ |
| **Évolutivité** | API REST code-first, Swagger, agnostique PSP |

> **Objectif** : Architecture robuste, souveraine, modulaire, prête à évoluer sans dette technique initiale.

---

## Hiérarchie des entités

```
Partner (Intégrateur)
└── Organization (Entreprise / Collectivité)
    └── Department (Régie / Service)
        ├── BankAccount (IBAN dédié)
        └── Invoice (Facture)
            ├── PaymentLink (Lien + QR Code)
            ├── PaymentIntent (Transaction Aiia)
            └── Refund (Remboursement)
```

## Cycle de vie d'une facture

```
DRAFT → ISSUED → PARTIALLY_PAID → PAID
                               ↘ OVERDUE
              ↘ CANCELLED
```

## Cycle de vie d'un paiement

```
CREATED → PENDING → PROCESSING → SUCCEEDED → REFUNDED
                              ↘ FAILED
```

---

## Sécurité

- **JWT RS256** — clés asymétriques, expiration 1h
- **RLS PostgreSQL** — isolation totale des données par tenant
- **AES-256-GCM** — chiffrement des IBANs en base
- **TLS 1.3** — transport sécurisé
- **HMAC-SHA256** — signatures des webhooks

---

## Performances cibles

| Métrique | Objectif |
|:---|:---|
| Temps de réponse API | < 200ms (P95) |
| Throughput | 1 000 req/s |
| Disponibilité | 99.9% |
| Requêtes DB | < 50ms (P95) |

---

## Roadmap

### v1.0 — Q1 2026 ✅
- API REST complète (60+ endpoints)
- Multi-tenant RLS
- Intégration Aiia (SEPA Open Banking)

### v1.1 — Q2 2026 🔜
- Envoi ticket de paiement
- Portail utilisateur Next.js
- Gestion des remboursements

### v1.2 — Q3 2026 📅
- Intégration Payfip
- Connecteurs outil facture
- plugins CMS

### v2.0 — 2027 📅
- IA
- Open Banking étendu
