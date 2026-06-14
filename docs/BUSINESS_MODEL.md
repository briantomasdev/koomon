# Koomon — Business Model & Pricing

This document defines Koomon's commercial open-source (COSS) monetization strategy, the three-tier product structure, pricing matrices, and the MRR projection model.

---

## Table of Contents

1. [COSS Strategy Overview](#1-coss-strategy-overview)
2. [The Three Tiers](#2-the-three-tiers)
   - 2.1 [Community Core (AGPLv3)](#21-community-core-agplv3)
   - 2.2 [Koomon Managed Cloud (SaaS)](#22-koomon-managed-cloud-saas)
   - 2.3 [SwarmState AI Enterprise (Proprietary Add-on)](#23-swarmstate-ai-enterprise-proprietary-add-on)
3. [Feature Matrix](#3-feature-matrix)
4. [Pricing Matrix](#4-pricing-matrix)
5. [MRR Projection Model](#5-mrr-projection-model)
6. [Conversion Funnel Assumptions](#6-conversion-funnel-assumptions)
7. [Unit Economics](#7-unit-economics)
8. [Go-To-Market Sequence](#8-go-to-market-sequence)

---

## 1. COSS Strategy Overview

Koomon follows the **Value-Value-Premium** tiered model: each successive tier delivers compounding value rather than withholding features from lower tiers.

```
                  [SwarmState AI Enterprise]
                  ┌──────────────────────────┐
                  │ On-device vector RAG      │  ← Proprietary closed-source
                  │ Compliance audit logs     │
                  │ SSO / SCIM provisioning   │
                  └──────────────────────────┘
                  [Koomon Managed Cloud]
                  ┌──────────────────────────┐
                  │ Managed infra + backups   │  ← Proprietary SaaS
                  │ 99.9% SLA                 │
                  │ Support SLA               │
                  └──────────────────────────┘
                  [Community Core]
                  ┌──────────────────────────┐
                  │ Full chat + CRDT engine   │  ← AGPLv3 open source
                  │ Self-hosted server        │
                  │ All 5 native platforms    │
                  └──────────────────────────┘
```

**Key principle**: The Community Core is complete and useful on its own. Paid tiers reduce operational burden and add intelligence features — they do not paywalled core functionality.

---

## 2. The Three Tiers

### 2.1 Community Core (AGPLv3)

**Price:** Free forever  
**License:** GNU Affero General Public License v3.0  
**Distribution:** GitHub, self-hosted Docker image

#### Target Segments

| Segment | Use Case |
|---|---|
| Families | Private group chat with full message history, no external server |
| Privacy advocates | Sovereign communication with zero third-party data exposure |
| Solo developers / homelab operators | Running Koomon on a mini-PC, Raspberry Pi, or VPS |
| OSS communities | Team coordination without SaaS subscriptions |

#### What's Included

- Complete multi-platform chat (macOS, Windows, Linux, iOS, Android)
- Y-CRDT engine with offline-first conflict resolution
- WebTransport relay server (self-hosted)
- SQLite local storage with SQLCipher encryption
- APNs and FCM mobile push (bring your own certificates)
- Docker Compose single-command server deployment
- No user limits, no message limits, no storage limits (bound only by your own hardware)

#### AGPLv3 Obligation

Any organization that modifies the Community Core and deploys it as a network service must publish those modifications under AGPLv3. This protects the commons and creates a natural commercial upgrade path for organizations that cannot comply.

---

### 2.2 Koomon Managed Cloud (SaaS)

**Price:** See [Pricing Matrix](#4-pricing-matrix)  
**License:** Proprietary SaaS  
**Delivery:** Hosted by Koomon Inc. on managed infrastructure

#### Target Segments

| Segment | Pain Eliminated |
|---|---|
| SMBs (5–200 employees) | No Docker expertise, no sysadmin overhead |
| Remote-first startups | Instant setup, no firewall/TURN configuration |
| Agencies | Multi-workspace management from a single admin panel |

#### What's Added Over Community Core

| Feature | Community Core | Managed Cloud |
|---|---|---|
| Infrastructure management | You operate it | We operate it |
| TLS certificate renewal | Manual | Automatic |
| Database backups | Manual | Daily automated + point-in-time recovery |
| QUIC/UDP firewall rules | Manual | Pre-configured |
| TURN server for NAT traversal | DIY | Included |
| Uptime SLA | N/A | 99.9% monthly |
| Email support | GitHub issues | 48-hour business response |
| Admin dashboard | CLI only | Web UI |
| Workspace audit log (90-day) | Not included | Included |

---

### 2.3 SwarmState AI Enterprise (Proprietary Add-on)

**Price:** See [Pricing Matrix](#4-pricing-matrix)  
**License:** Proprietary — closed source  
**Delivery:** Bundled with Managed Cloud Enterprise tier or as on-premises binary

#### Target Segments

| Segment | Value Delivered |
|---|---|
| Legal & compliance teams | Air-gapped AI search with immutable audit trails |
| Engineering orgs (50–5000) | Semantic search across years of channel history |
| Finance & healthcare | Zero-cloud RAG: no message data leaves the perimeter |

#### Technical Summary

SwarmState AI indexes the local SQLite message corpus into an on-device vector store using a quantized embedding model (target: `nomic-embed-text` or equivalent). All inference runs locally via the `candle` Rust ML framework — no API calls to OpenAI, Anthropic, or any cloud model provider.

| Feature | Description |
|---|---|
| On-device vector indexing | Embeddings computed and stored locally using `usearch` or `hnswlib` |
| Semantic workspace search | Natural-language queries across all channels the user has access to |
| Privacy-compliant RAG | No message content ever transmitted to external AI services |
| Role-aware retrieval | Search results respect existing channel membership permissions |
| SSO / SCIM provisioning | Okta, Azure AD, Google Workspace identity federation |
| Compliance audit log (1-year) | Immutable append-only log of all workspace events |
| Dedicated SLA | 4-hour critical response, named account manager |

---

## 3. Feature Matrix

| Feature | Community Core | Managed Cloud | Enterprise + SwarmState |
|---|---|---|---|
| Multi-platform chat (all 5 OS) | ✅ | ✅ | ✅ |
| Offline-first CRDT sync | ✅ | ✅ | ✅ |
| End-to-end encryption | ✅ | ✅ | ✅ |
| Mobile push (APNs / FCM) | ✅ (BYO certs) | ✅ | ✅ |
| Self-hosted server | ✅ | ❌ | Optional |
| Managed infra + backups | ❌ | ✅ | ✅ |
| 99.9% uptime SLA | ❌ | ✅ | ✅ |
| Admin web dashboard | ❌ | ✅ | ✅ |
| Audit log (90-day) | ❌ | ✅ | ✅ (1-year) |
| SSO / SCIM | ❌ | ❌ | ✅ |
| On-device vector RAG search | ❌ | ❌ | ✅ |
| Privacy-compliant AI | ❌ | ❌ | ✅ |
| Compliance audit log (1-year) | ❌ | ❌ | ✅ |
| Dedicated account manager | ❌ | ❌ | ✅ |
| User limit | Unlimited | Up to plan | Unlimited |

---

## 4. Pricing Matrix

### Koomon Managed Cloud

| Plan | Users | Price | Billing |
|---|---|---|---|
| **Starter** | Up to 10 | $29 / month flat | Monthly or annual (–20%) |
| **Growth** | Up to 50 | $8 / user / month | Monthly or annual (–20%) |
| **Business** | Up to 250 | $6 / user / month | Monthly or annual (–20%) |
| **Scale** | 250+ | Custom | Annual contract |

> Annual pricing example: 25-person Growth team → $8 × 25 × 12 × 0.80 = **$1,920 / year** vs $2,400 billed monthly.

### SwarmState AI Enterprise Add-on

| Plan | Users | Price | Billing |
|---|---|---|---|
| **Enterprise** | Up to 500 | $12 / user / month | Annual only |
| **Enterprise+** | 500+ | Custom | Annual contract |

> Enterprise add-on requires an active Managed Cloud Business or Scale subscription.

---

## 5. MRR Projection Model

The table below models monthly recurring revenue across realistic user growth scenarios using standard COSS conversion benchmarks.

### Assumptions

| Variable | Value | Source |
|---|---|---|
| OSS → Managed Cloud conversion | 3% | Industry benchmark for developer-led COSS |
| Managed Cloud → Enterprise conversion | 15% | Mid-market upsell benchmark |
| Average Managed Cloud users per account | 22 | Blended Growth/Business plan mix |
| Average Managed Cloud ARPU | $7.00 / user / month | Blended Growth + Business pricing |
| Average Enterprise users per account | 120 | Mid-market org size |
| Enterprise ARPU (base + add-on) | $18.00 / user / month | $6 Business + $12 SwarmState |

### MRR Projections by OSS User Base

| OSS Active Users | Cloud Accounts (3%) | Cloud MRR | Enterprise Accounts (15% of Cloud) | Enterprise MRR | **Total MRR** |
|---|---|---|---|---|---|
| 1,000 | 30 | $4,620 | 5 | $10,800 | **$15,420** |
| 5,000 | 150 | $23,100 | 23 | $49,680 | **$72,780** |
| 10,000 | 300 | $46,200 | 45 | $97,200 | **$143,400** |
| 25,000 | 750 | $115,500 | 113 | $243,360 | **$358,860** |
| 50,000 | 1,500 | $231,000 | 225 | $486,000 | **$717,000** |
| 100,000 | 3,000 | $462,000 | 450 | $972,000 | **$1,434,000** |

> All figures are illustrative models. Actual conversion rates depend on product-market fit, content marketing execution, and sales motion.

---

## 6. Conversion Funnel Assumptions

```
[OSS GitHub Stars / Downloads]
         │
         ▼  (awareness → activation)
[Self-Hosted Community Core Users]
         │
         ▼  ~3% (ops friction conversion)
[Managed Cloud Trial / Paid]
         │
         ▼  ~15% (team-intelligence upsell)
[SwarmState AI Enterprise Accounts]
```

### Primary Conversion Triggers

| Friction Point | Conversion Lever |
|---|---|
| "I don't want to manage Docker" | One-click Managed Cloud onboarding |
| "We need backups and an SLA for our board" | Business plan with 99.9% SLA |
| "We can't use cloud AI — we're regulated" | SwarmState on-device RAG pitch |
| "We're 200 people and need SSO" | Enterprise SSO/SCIM gate |

---

## 7. Unit Economics

| Metric | Target |
|---|---|
| Gross margin (Managed Cloud) | ≥ 75% (infra cost ~25% of ARPU at scale) |
| Gross margin (Enterprise) | ≥ 85% (software distribution, minimal infra) |
| Customer Acquisition Cost (CAC) | < 3× monthly ARPU (developer-led, low-touch) |
| Payback period | < 6 months for Growth, < 3 months for Enterprise |
| Net Revenue Retention (NRR) target | ≥ 110% (seat expansion as teams grow) |

---

## 8. Go-To-Market Sequence

| Phase | Action | Goal |
|---|---|---|
| **Pre-launch** | Build in public on GitHub, post architecture content | OSS developer awareness |
| **Phase 4 (Weeks 21–24)** | Public GitHub repo + Hacker News / Reddit launch | First 1,000 OSS users |
| **Month 3–6** | Launch Managed Cloud waitlist and Starter plan | First 30 paying accounts |
| **Month 6–12** | Content SEO targeting "self-hosted Slack alternative" | 5,000 OSS users, 150 cloud accounts |
| **Year 2** | Enterprise outbound for legal/finance/healthcare verticals | First SwarmState AI contracts |
| **Year 2–3** | Partner with MSPs and system integrators | Channel revenue stream |
