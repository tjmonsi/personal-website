---
title: Draft Cost Estimate — GCP Monthly Costs
version: 0.4-draft
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
- [BigQuery Pricing](https://cloud.google.com/bigquery/pricing)
- [Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)
- [Firestore Native Pricing](https://cloud.google.com/firestore/pricing)
---

### Changes from v0.3-draft

- **Cloud Scheduler**: Updated from 2 to 3 jobs (added `trigger-sync-article-embeddings` daily safety net, INFRA-014b). All 3 jobs remain within the free tier of 3.
- **Content CI/CD authentication**: Documented as Workload Identity Federation (no cost impact — WIF is free). See CLR-056, AD-022, SEC-010.

### Changes from v0.2-draft

- **VPC Serverless Connector removed**: Direct VPC Egress is now the decided architecture (Note 3 from CLR-006). No connector instances needed, saving ~$12.26/month.
- **Cloud Functions**: Added `cleanup-rate-limit-offenders` (4th Cloud Function) triggered daily by Cloud Scheduler.
- **Cloud Scheduler**: Corrected to 2 jobs (sitemap generation, offender cleanup). The log processing function is triggered by Pub/Sub via a log sink, not Cloud Scheduler.
- **Cost Optimization Scenarios**: Updated to reflect Direct VPC Egress as the default, not an optimization option.
- **Totals recalculated**: Normal ~$30.32/month, Spike ~$36.16/month.

### Changes from v0.1-draft

- **Cloud Run memory**: Corrected from 256 MB to 1 GB per INFRA-003.
- **Firestore tracking writes**: Removed. DM-007/DM-008 collections were removed — tracking and error data flows to BigQuery via structured logs only.
- **POST /t volume**: Updated to account for new `link_click` and `time_on_page` events (~2× page view count instead of 1×).
- **Added**: BigQuery costs (storage + query), Vertex AI embedding API costs, Firestore Native Mode costs, Cloud Functions costs, and VPC Serverless connector costs.
- **Added**: Ban status check reads (per-request Firestore read for progressive banning, per SEC-002).
- **Cloud Armor rules**: Updated to 7 rules per INFRA-005.
- **Total request volume**: Recalculated based on revised POST /t multiplier.

---

### Scenario 1: Normal Usage (Low-Traffic Personal Website)

**Assumptions**:
- ~1,000 unique visitors/month
- ~3,000 page views/month
- POST /t tracking events breakdown:
  - ~3,000 `page_view` events (1 per page view)
  - ~1,500 `link_click` events (~50% of page views result in a tracked click)
  - ~1,500 `time_on_page` events (~30% reach 1min, ~15% reach 2min, ~5% reach 5min)
  - ~100 `error_report` events
  - **Total POST /t: ~6,100/month**
- GET requests: ~6,000/month (list + detail + categories + sitemap + misc)
- **Total API requests: ~12,000/month**
- Average response time: 200ms
- Average response size: 10 KB
- Total data transfer: ~120 MB/month outbound
- Database size: < 100 MB (articles, metadata, categories, embedding cache)
- Cloud Run configured: 1 vCPU, 1 GB RAM, request-based billing, min instances = 0
- Search queries with embedding: ~100/month

| Service | Component | Estimated Usage | Unit Price (asia-southeast1) | Estimated Monthly Cost |
|---------|-----------|-----------------|--------------------------------------|----------------------|
| **Cloud Run** | CPU | 12,000 req × 0.2s = 2,400 vCPU-seconds | ~$0.0000312/vCPU-s | $0.07 |
| | Memory | 12,000 req × 0.2s × 1 GiB = 2,400 GiB-s | ~$0.00000325/GiB-s | $0.008 |
| | Requests | 12,000 | $0.52/million | $0.006 |
| | **Cloud Run Subtotal** | | | **~$0.08** |
| **Firestore Enterprise** | Document reads (content) | ~30,000 reads/month (list queries return multiple docs, detail reads, sitemap, categories) | $0.036/100K (Singapore) | $0.01 |
| | Document reads (ban checks) | ~12,000 reads/month (1 per API request for `rate_limit_offenders` check, per SEC-002) | $0.036/100K | $0.004 |
| | Document writes | ~200 writes/month (sitemap regeneration, rate limit offender records, embedding cache writes) | $0.108/100K | $0.0002 |
| | Storage | < 0.1 GiB | ~$0.18/GiB/month | $0.02 |
| | **Firestore Enterprise Subtotal** | | | **~$0.03** |
| **Firestore Native** | Document reads (vector search) | ~100 reads/month | $0.036/100K | $0.00 |
| | Document writes (embedding sync) | ~10 writes/month | $0.108/100K | $0.00 |
| | Storage | ~5 MB (3 collections × ~100 docs × ~16 KB per vector) | ~$0.18/GiB/month | $0.00 |
| | **Firestore Native Subtotal** | | | **~$0.00** |
| **Vertex AI** | Gemini embedding API | ~100 queries × ~50 chars + ~10 articles × ~500 chars ≈ 10K chars/month | Free tier: 1,500 req/day | $0.00 |
| | **Vertex AI Subtotal** | | | **~$0.00** |
| **BigQuery** | Streaming inserts (via log sinks) | ~12,000 log entries × ~500 bytes ≈ 6 MB/month | Free tier: 10 GiB/month | $0.00 |
| | Active storage | < 50 MB (accumulated over months) | Free tier: 10 GiB; then $0.02/GB | $0.00 |
| | Queries (Looker Studio) | < 100 MB scanned/month | Free tier: 1 TB/month | $0.00 |
| | **BigQuery Subtotal** | | | **~$0.00** |
| **Cloud Armor** | Security policy | 1 policy | $5/month | $5.00 |
| | Rules | 7 rules (per INFRA-005) | $1/rule/month | $7.00 |
| | Requests | 12,000 | $0.75/million | $0.01 |
| | **Cloud Armor Subtotal** | | | **~$12.01** |
| **Cloud Load Balancer** | Forwarding rules | 1 rule | $0.025/hour | ~$18.00 |
| | Data processing | 120 MB | $0.008/GB | $0.001 |
| | **Load Balancer Subtotal** | | | **~$18.00** |
| **Firebase Hosting** | Storage | < 1 GB (SPA assets) | Free tier: 10 GB | $0.00 |
| | Data transfer | < 360 MB/day | Free tier: 360 MB/day | $0.00 |
| | **Firebase Hosting Subtotal** | | | **$0.00** |
| **Firebase Functions** | Invocations | ~3,000/month (SPA shell serving) | Free tier: 2M/month | $0.00 |
| | Compute | Minimal (serving static HTML) | Free tier covers | $0.00 |
| | **Firebase Functions Subtotal** | | | **$0.00** |
| **Cloud Functions (Gen 2)** | generate-sitemap | ~120 invocations/month (every 6h), 256 MB, <1s each | Free tier: 2M invocations | $0.00 |
| | process-rate-limit-logs | ~10 invocations/month (rare at low traffic) | Free tier covers | $0.00 |
| | sync-article-embeddings | ~40 invocations/month (~10 content CI/CD + ~30 daily Cloud Scheduler) | Free tier covers | $0.00 |
| | cleanup-rate-limit-offenders | ~30 invocations/month (daily via Cloud Scheduler) | Free tier covers | $0.00 |
| | **Cloud Functions (Gen 2) Subtotal** | | | **$0.00** |
| **Cloud Scheduler** | Jobs | 3 (sitemap generation, offender cleanup, embedding sync safety net) | Free tier: 3 jobs | $0.00 |
| **Cloud Logging** | Log ingestion | < 50 GiB/month | Free tier: 50 GiB/month | $0.00 |
| **Cloud Monitoring** | Metrics | Basic metrics | Free tier covers | $0.00 |
| **Cloud DNS** | Hosted zone | 1 zone | $0.20/zone/month | $0.20 |
| | Queries | ~12,000 | $0.40/million | $0.005 |
| | **DNS Subtotal** | | | **~$0.20** |

**Scenario 1 Total: ~$30.32/month**

> **Note**: The dominant costs are Cloud Armor ($12) and the Cloud Load Balancer forwarding rule ($18). These are fixed infrastructure costs regardless of traffic. Compute, database, analytics, and AI costs are negligible at this traffic level. Direct VPC Egress is used instead of a Serverless VPC Access Connector, which avoids the ~$12.26/month connector cost.

> **Cost Optimization Options**:
> - **Defer Cloud Armor** if budget is tight at launch — saves ~$12/month but reduces WAF/DDoS protection. Can be added later.
> - With that optimization, the cost drops to **~$18.32/month** (dominated by the Load Balancer forwarding rule).

---

### Scenario 2: Traffic Spike (Viral Article)

**Assumptions**:
- ~50,000 unique visitors/month
- ~150,000 page views/month
- POST /t tracking events breakdown:
  - ~150,000 `page_view` events
  - ~75,000 `link_click` events (~50%)
  - ~60,000 `time_on_page` events (~30% hit 1min, ~15% hit 2min, ~5% hit 5min)
  - ~5,000 `error_report` events
  - **Total POST /t: ~290,000/month**
- GET requests: ~300,000/month (list + detail queries for the viral article)
- **Total API requests: ~590,000/month**
- Average response time: 200ms
- Average response size: 15 KB (mix of list and detail views)
- Total data transfer: ~9 GB/month outbound
- 2-3 Cloud Run instances active during peak
- Search queries with embedding: ~5,000/month

| Service | Component | Estimated Usage | Unit Price | Estimated Monthly Cost |
|---------|-----------|-----------------|------------|----------------------|
| **Cloud Run** | CPU | 590,000 req × 0.2s = 118,000 vCPU-s | ~$0.0000312/vCPU-s | $3.68 |
| | Memory | 590,000 × 0.2s × 1 GiB = 118,000 GiB-s | ~$0.00000325/GiB-s | $0.38 |
| | Requests | 590,000 | $0.52/million | $0.31 |
| | **Cloud Run Subtotal** | | | **~$4.37** |
| **Firestore Enterprise** | Document reads (content) | ~1,500,000 reads (list queries + detail reads + caching misses) | $0.036/100K | $0.54 |
| | Document reads (ban checks) | ~590,000 reads (1 per API request, per SEC-002) | $0.036/100K | $0.21 |
| | Document writes | ~1,000 writes (sitemap regen, rate limit offenders, embedding cache) | $0.108/100K | $0.001 |
| | Storage | < 0.5 GiB | ~$0.18/GiB/month | $0.09 |
| | **Firestore Enterprise Subtotal** | | | **~$0.84** |
| **Firestore Native** | Document reads (vector search) | ~5,000 reads | $0.036/100K | $0.002 |
| | Document writes (embedding sync) | ~50 writes | $0.108/100K | $0.00 |
| | Storage | ~5 MB | ~$0.18/GiB/month | $0.00 |
| | **Firestore Native Subtotal** | | | **~$0.00** |
| **Vertex AI** | Gemini embedding API | ~5,000 queries × ~50 chars ≈ 250K chars (many cached) | Free tier covers | $0.00 |
| | **Vertex AI Subtotal** | | | **~$0.00** |
| **BigQuery** | Streaming inserts (via log sinks) | ~590,000 entries × ~500 bytes ≈ 295 MB/month | Free tier: 10 GiB/month | $0.00 |
| | Active storage | < 500 MB (accumulated) | Free tier: 10 GiB; then $0.02/GB | $0.00 |
| | Queries (Looker Studio) | < 5 GB scanned/month | Free tier: 1 TB/month | $0.00 |
| | **BigQuery Subtotal** | | | **~$0.00** |
| **Cloud Armor** | Security policy | 1 policy | $5/month | $5.00 |
| | Rules | 7 rules | $1/rule/month | $7.00 |
| | Requests | 590,000 | $0.75/million | $0.44 |
| | **Cloud Armor Subtotal** | | | **~$12.44** |
| **Cloud Load Balancer** | Forwarding rules | 1 | ~$18/month | $18.00 |
| | Data processing | 9 GB | $0.008/GB | $0.07 |
| | **Load Balancer Subtotal** | | | **~$18.07** |
| **Firebase Hosting** | Storage | < 1 GB | Free tier | $0.00 |
| | Data transfer | ~5 GB/month for static assets | Free tier: ~10.8 GB/month | $0.00 |
| | **Firebase Hosting Subtotal** | | | **$0.00** |
| **Firebase Functions** | Invocations | ~50,000 | Free tier: 2M | $0.00 |
| | **Firebase Functions Subtotal** | | | **$0.00** |
| **Cloud Functions (Gen 2)** | generate-sitemap | ~120/month | Free tier covers | $0.00 |
| | process-rate-limit-logs | ~500 invocations (more rate limit events during spike) | Free tier covers | $0.00 |
| | sync-article-embeddings | ~50/month (~20 content CI/CD + ~30 daily Cloud Scheduler) | Free tier covers | $0.00 |
| | cleanup-rate-limit-offenders | ~30/month | Free tier covers | $0.00 |
| | **Cloud Functions (Gen 2) Subtotal** | | | **$0.00** |
| **Cloud Scheduler** | Jobs | 3 | Free tier | $0.00 |
| **Cloud Logging** | Log ingestion | ~5 GiB | Free tier: 50 GiB | $0.00 |
| **Cloud DNS** | Zone + queries | 1 zone, ~600K queries | $0.20 + $0.24 | $0.44 |

**Scenario 2 Total: ~$36.16/month**

> **Note**: Even with a 50x traffic spike, the total cost only increases by ~$6/month over normal usage. Fixed infrastructure costs (Cloud Armor + Load Balancer) dominate at ~$30/month, and variable costs (Cloud Run, Firestore) scale very efficiently. Direct VPC Egress is used at no additional cost.

---

### Cost Breakdown Summary

| Component | Normal Usage | Traffic Spike | Notes |
|-----------|-------------|--------------|-------|
| Cloud Run | $0.08 | $4.37 | Scale-to-zero keeps costs minimal at low traffic |
| Firestore Enterprise | $0.03 | $0.84 | Includes ban status checks on every request |
| Firestore Native | $0.00 | $0.00 | Vector search usage is minimal at both levels |
| Vertex AI | $0.00 | $0.00 | Within free tier for embedding API |
| BigQuery | $0.00 | $0.00 | Within free tier for storage and queries |
| Cloud Armor | $12.01 | $12.44 | Mostly fixed (policy + 7 rules) |
| Cloud Load Balancer | $18.00 | $18.07 | Fixed forwarding rule cost dominates |
| Firebase Hosting + Functions | $0.00 | $0.00 | Within free tier for both scenarios |
| Cloud Functions (Gen 2) | $0.00 | $0.00 | 4 functions, all within free tier |
| Cloud Scheduler | $0.00 | $0.00 | Within free tier (3 jobs, 3 free allowed) |
| Cloud DNS | $0.20 | $0.44 | Minimal |
| Cloud Logging/Monitoring | $0.00 | $0.00 | Within free tier |
| **Total** | **~$30.32** | **~$36.16** | |

### Cost Optimization Scenarios

| Optimization | Savings | Normal Cost | Trade-off |
|-------------|---------|-------------|-----------|
| Defer Cloud Armor | ~$12/month | ~$18.32 | Reduced WAF/DDoS protection |
| Remove LB (API uses Cloud Run default domain) | ~$30/month | ~$0.32 | No WAF, no global LB, no centralized SSL |

### Key Observations

1. **Fixed costs dominate**: Cloud Armor ($12) and Cloud Load Balancer ($18) account for ~99% of costs at normal traffic levels. These are the price of enterprise-grade WAF and global load balancing.

2. **Direct VPC Egress saves ~$12/month**: The architecture uses [Direct VPC Egress](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc) instead of a Serverless VPC Access Connector, which avoids the ~$12.26/month cost of 2 always-on e2-micro instances with no functional trade-off.

3. **Variable costs are negligible**: Cloud Run, Firestore, BigQuery, Vertex AI, and Cloud Functions costs are all under $1/month even during a 50x traffic spike. The pay-per-use model is excellent for personal websites.

4. **Free tiers cover the analytics pipeline**: BigQuery storage, BigQuery queries, Vertex AI embeddings, and Cloud Functions all stay within free tiers for both scenarios. The entire analytics pipeline (Cloud Logging → BigQuery → Looker Studio) effectively costs $0.

5. **POST /t volume**: With `link_click` and `time_on_page` events, POST /t calls are ~2× the page view count. This primarily increases Cloud Run CPU time and BigQuery ingestion volume, but both remain well within free/negligible cost ranges.

6. **Firestore Enterprise caveat**: The estimates above use standard Firestore pricing. Firestore Enterprise (MongoDB compatibility mode) may have different pricing tiers. Check [Firestore Enterprise pricing](https://cloud.google.com/firestore/enterprise/pricing) for exact rates. Actual Firestore costs could be higher.

7. **Network egress**: Data transfer within the same region (Cloud Run ↔ Firestore Enterprise ↔ Firestore Native ↔ Vertex AI) is free. Outbound internet transfer from Cloud Run is included in the load balancer pricing.

8. **Looker Studio query costs**: Looker Studio queries against BigQuery incur costs based on data scanned. At these data volumes (< 1 GB total), all queries fall within BigQuery's 1 TB/month free tier. Use partitioned tables and targeted date ranges to minimize costs as data grows.

### Cost Optimization Recommendations

1. **Defer Cloud Armor** if budget is a concern at launch — can be added later when traffic grows. Without Cloud Armor, rate limiting must be implemented in the Go application.
2. **Use Cloud Run request-based billing** with scale-to-zero to minimize idle costs (already in spec).
3. **Cache aggressively**: Sitemap caching (1 hour), category caching (24h sessionStorage), embedding cache (no expiry), and article caching (Cache API + IndexedDB) all reduce backend requests and Firestore reads.
4. **Monitor free tier usage**: Set up billing alerts at $10, $25, and $50 thresholds.
5. **Consider simplifying the LB**: If Cloud Armor is deferred, the Load Balancer may be avoidable by using Cloud Run's default HTTPS endpoint for the API and Firebase Hosting's built-in CDN for the frontend. This removes the $18/month forwarding rule cost.
