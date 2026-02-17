---
title: Draft Cost Estimate — GCP Monthly Costs
version: 0.1-draft
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [cost, infrastructure, gcp, estimate]
---

## Draft Cost Estimate

This document provides a rough cost estimate for running the personal website infrastructure on Google Cloud Platform. Two scenarios are modeled: normal low-traffic usage and a traffic spike from a viral article.

**Disclaimer**: These are estimates based on publicly available GCP pricing as of February 2026. Actual costs may vary. Firestore Enterprise pricing may differ from standard Firestore pricing — check the [Firestore Enterprise pricing page](https://cloud.google.com/firestore/enterprise/pricing) for exact rates. All prices are in USD.

**Pricing References**:
- [Cloud Run Pricing](https://cloud.google.com/run/pricing) | [SKUs](https://cloud.google.com/skus?hl=en)
- [Firestore Standard Pricing](https://cloud.google.com/firestore/pricing) | [Firestore Enterprise Pricing](https://cloud.google.com/firestore/enterprise/pricing)
- [Cloud Armor Pricing](https://cloud.google.com/armor/pricing)
- [Cloud Scheduler Pricing](https://cloud.google.com/scheduler/pricing)
- [Firebase Pricing](https://firebase.google.com/pricing)
- [Cloud Load Balancing Pricing](https://cloud.google.com/vpc/network-pricing)

---

### Scenario 1: Normal Usage (Low-Traffic Personal Website)

**Assumptions**:
- ~1,000 unique visitors/month
- ~3,000 page views/month
- Average 3 API calls per page view (list endpoint + detail endpoint + `POST /t` tracking)
- ~10,000 total API requests/month
- Average response time: 200ms
- Average response size: 10 KB
- Total data transfer: ~100 MB/month outbound
- Database size: < 100 MB (articles, metadata, tracking)
- Cloud Run configured: 1 vCPU, 256 MB RAM, request-based billing, min instances = 0

| Service | Component | Estimated Usage | Unit Price (asia-southeast1, Tier 2) | Estimated Monthly Cost |
|---------|-----------|-----------------|--------------------------------------|----------------------|
| **Cloud Run** | CPU | 10,000 req × 0.2s = 2,000 vCPU-seconds | ~$0.0000312/vCPU-s | $0.06 |
| | Memory | 10,000 req × 0.2s × 0.25 GiB = 500 GiB-s | ~$0.00000325/GiB-s | $0.002 |
| | Requests | 10,000 | $0.52/million | $0.005 |
| | **Cloud Run Subtotal** | | | **~$0.07** |
| **Firestore** | Document reads | ~30,000 reads/month (list queries, detail reads, sitemap) | $0.036/100K (Singapore) | $0.01 |
| | Document writes | ~3,000 writes/month (tracking + error reports) | $0.108/100K | $0.003 |
| | Storage | < 0.1 GiB | ~$0.18/GiB/month | $0.02 |
| | **Firestore Subtotal** | | | **~$0.03** |
| **Cloud Armor** | Security policy | 1 policy | $5/month | $5.00 |
| | Rules | 6 rules | $1/rule/month | $6.00 |
| | Requests | 10,000 | $0.75/million | $0.01 |
| | **Cloud Armor Subtotal** | | | **~$11.01** |
| **Cloud Load Balancer** | Forwarding rules | 1 rule (first 5 free) | $0.025/hour | ~$18.00 |
| | Data processing | 100 MB | $0.008/GB | $0.001 |
| | **Load Balancer Subtotal** | | | **~$18.00** |
| **Firebase Hosting** | Storage | < 1 GB (SPA assets) | Free tier: 10 GB | $0.00 |
| | Data transfer | < 360 MB/day | Free tier: 360 MB/day | $0.00 |
| | **Firebase Hosting Subtotal** | | | **$0.00** |
| **Firebase Functions** | Invocations | ~3,000/month (SPA shell serving) | Free tier: 2M/month | $0.00 |
| | Compute | Minimal (serving static HTML) | Free tier covers | $0.00 |
| | **Firebase Functions Subtotal** | | | **$0.00** |
| **Cloud Scheduler** | Jobs | 1 (sitemap generation) | Free tier: 3 jobs | $0.00 |
| **Cloud Logging** | Log ingestion | < 50 GiB/month | Free tier: 50 GiB/month | $0.00 |
| **Cloud Monitoring** | Metrics | Basic metrics | Free tier covers | $0.00 |
| **Cloud DNS** | Hosted zone | 1 zone | $0.20/zone/month | $0.20 |
| | Queries | ~10,000 | $0.40/million | $0.004 |
| | **DNS Subtotal** | | | **~$0.20** |

**Scenario 1 Total: ~$29.31/month**

> **Note**: The dominant costs are Cloud Armor ($11) and the Cloud Load Balancer forwarding rule ($18). These are fixed infrastructure costs regardless of traffic. The actual compute and database costs are negligible at this traffic level — most fall within free tiers.

> **Cost Optimization Option**: If budget is very tight, consider whether Cloud Armor is needed at launch. Without Cloud Armor and with a simpler load balancing setup, the cost could drop to under $5/month. However, this reduces security posture.

---

### Scenario 2: Traffic Spike (Viral Article)

**Assumptions**:
- ~50,000 unique visitors/month
- ~150,000 page views/month
- ~500,000 total API requests/month
- Average response time: 200ms
- Average response size: 15 KB (mix of list and detail views)
- Total data transfer: ~7.5 GB/month outbound
- 2-3 Cloud Run instances active during peak
- Firestore reads surge for the popular article

| Service | Component | Estimated Usage | Unit Price | Estimated Monthly Cost |
|---------|-----------|-----------------|------------|----------------------|
| **Cloud Run** | CPU | 500,000 req × 0.2s = 100,000 vCPU-s | ~$0.0000312/vCPU-s | $3.12 |
| | Memory | 500,000 × 0.2s × 0.25 GiB = 25,000 GiB-s | ~$0.00000325/GiB-s | $0.08 |
| | Requests | 500,000 | $0.52/million | $0.26 |
| | **Cloud Run Subtotal** | | | **~$3.46** |
| **Firestore** | Document reads | ~1,500,000 reads (list queries return 10 docs each + detail reads + caching misses) | $0.036/100K | $0.54 |
| | Document writes | ~150,000 (tracking writes) | $0.108/100K | $0.16 |
| | Storage | < 0.5 GiB | ~$0.18/GiB/month | $0.09 |
| | **Firestore Subtotal** | | | **~$0.79** |
| **Cloud Armor** | Security policy | 1 policy | $5/month | $5.00 |
| | Rules | 6 rules | $1/rule/month | $6.00 |
| | Requests | 500,000 | $0.75/million | $0.38 |
| | **Cloud Armor Subtotal** | | | **~$11.38** |
| **Cloud Load Balancer** | Forwarding rules | 1 | ~$18/month | $18.00 |
| | Data processing | 7.5 GB | $0.008/GB | $0.06 |
| | **Load Balancer Subtotal** | | | **~$18.06** |
| **Firebase Hosting** | Storage | < 1 GB | Free tier | $0.00 |
| | Data transfer | ~5 GB/month for static assets | Free tier: ~10.8 GB/month | $0.00 |
| | **Firebase Hosting Subtotal** | | | **$0.00** |
| **Firebase Functions** | Invocations | ~50,000 | Free tier: 2M | $0.00 |
| | **Firebase Functions Subtotal** | | | **$0.00** |
| **Cloud Scheduler** | Jobs | 1 | Free tier | $0.00 |
| **Cloud Logging** | Log ingestion | ~5 GiB | Free tier: 50 GiB | $0.00 |
| **Cloud DNS** | Zone + queries | 1 zone, ~500K queries | $0.20 + $0.20 | $0.40 |

**Scenario 2 Total: ~$34.09/month**

> **Note**: Even with a 50x traffic spike, the total cost only increases by ~$5/month. This is because the fixed infrastructure costs (Cloud Armor + Load Balancer) dominate, and the variable costs (Cloud Run, Firestore) scale very efficiently at these volumes.

---

### Cost Breakdown Summary

| Component | Normal Usage | Traffic Spike | Notes |
|-----------|-------------|--------------|-------|
| Cloud Run | $0.07 | $3.46 | Scale-to-zero keeps costs minimal at low traffic |
| Firestore | $0.03 | $0.79 | Document read/write costs scale linearly |
| Cloud Armor | $11.01 | $11.38 | Mostly fixed cost (policy + rules) |
| Cloud Load Balancer | $18.00 | $18.06 | Fixed forwarding rule cost dominates |
| Firebase Hosting + Functions | $0.00 | $0.00 | Within free tier for both scenarios |
| Cloud Scheduler | $0.00 | $0.00 | Within free tier (3 free jobs) |
| Cloud DNS | $0.20 | $0.40 | Minimal |
| Cloud Logging/Monitoring | $0.00 | $0.00 | Within free tier |
| **Total** | **~$29.31** | **~$34.09** | |

### Key Observations

1. **Fixed costs dominate**: Cloud Armor ($11) and Cloud Load Balancer ($18) account for ~99% of costs at normal traffic levels. These are the price of having enterprise-grade WAF and global load balancing.

2. **Variable costs are negligible**: Cloud Run and Firestore costs are under $1/month even during a traffic spike. The pay-per-use model is excellent for low-to-medium traffic personal websites.

3. **Free tiers help significantly**: Firebase Hosting, Firebase Functions, Cloud Scheduler, and Cloud Logging all stay within their free tiers for both scenarios.

4. **Firestore Enterprise caveat**: The estimates above use standard Firestore pricing. Firestore Enterprise (MongoDB compatibility mode) may have different pricing. Check [Firestore Enterprise pricing](https://cloud.google.com/firestore/enterprise/pricing) for exact rates.

5. **Network egress**: Data transfer within the same region (Cloud Run ↔ Firestore) is free. Outbound internet transfer from Cloud Run is included in the load balancer pricing.

### Cost Optimization Recommendations

- **Defer Cloud Armor** if budget is a concern at launch — can be added later when traffic grows.
- **Use Cloud Run request-based billing** with scale-to-zero to minimize idle costs.
- **Cache aggressively**: The sitemap caching (1 hour), category caching (24 hours in sessionStorage), and article caching (Cache API + IndexedDB) all reduce backend requests and Firestore reads.
- **Monitor free tier usage**: Set up billing alerts at $10, $25, and $50 thresholds.
