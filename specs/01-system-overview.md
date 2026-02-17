---
title: System Overview
version: 1.4
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [architecture, system-overview, infrastructure]
---

## System Overview

### Purpose

A personal website for TJ Monserrat serving as a professional online presence with a technical blog, opinion blog, social links, and curated external content references.

### Technology Stack

| Layer          | Technology                             | Hosting                                  |
| -------------- | -------------------------------------- | ---------------------------------------- |
| Frontend       | Nuxt 4 / Vue 3 / Vite (SPA mode)      | Firebase Hosting + Firebase Functions    |
| Backend API    | Go                                     | Google Cloud Run (asia-southeast1)       |
| Sitemap Gen    | Go (Cloud Functions Gen 2)             | Google Cloud Functions (asia-southeast1) |
| Log Processing | Go (Cloud Functions Gen 2)             | Google Cloud Functions (asia-southeast1) |
| Database       | Firestore Enterprise (MongoDB compatibility mode) | Google Cloud (asia-southeast1)  |
| CDN / WAF      | Google Cloud Load Balancer + Cloud Armor | Google Cloud                           |
| Networking     | VPC with Private Google Access         | Google Cloud (asia-southeast1)           |
| Scheduling     | Google Cloud Scheduler                 | Google Cloud (asia-southeast1)           |
| Observability  | Google Cloud Logging + Cloud Monitoring | Google Cloud                            |

### Definitions

| Term                            | Definition |
| ------------------------------- | ---------- |
| Firestore Enterprise            | Google Cloud's enterprise-grade document database. See: https://firebase.google.com/docs/firestore/enterprise/overview-enterprise-edition-modes |
| MongoDB Compatibility Mode      | Firestore Enterprise mode that exposes a MongoDB-compatible wire protocol, allowing use of standard MongoDB drivers and tools. See: https://firebase.google.com/docs/firestore/enterprise/mongodb-compatibility-overview |
| SPA                             | Single Page Application — client-side rendered web application |
| Cloud Run                       | Google Cloud's serverless container hosting platform |
| Cloud Armor                     | Google Cloud's WAF and DDoS protection service |
| Firebase Functions              | Cloud Functions integrated with Firebase, used here to serve the Nuxt 4 SPA |
| Cloud Functions Gen 2           | Google Cloud's second-generation serverless functions platform, built on Cloud Run infrastructure. Used here for internal sitemap generation and Cloud Armor log processing. |
| Log Sink                        | A Cloud Logging export mechanism that routes matching log entries to a destination (e.g., Pub/Sub, Cloud Function) for further processing. Used here to route Cloud Armor rate-limit events to a Cloud Function. |
| Cloud Armor Adaptive Protection | A Cloud Armor feature that uses machine learning to detect and mitigate L7 DDoS attacks automatically, providing escalating protection without manual rule configuration. |
| VPC                             | Virtual Private Cloud — isolated network environment for Google Cloud resources |
| Cloud Scheduler                 | Google Cloud's managed cron job service for scheduling periodic tasks |

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      End Users                          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Cloud DNS     │
              │  tjmonsi.com    │
              │ api.tjmonsi.com │
              └────────┬────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│           Google Cloud Load Balancer                    │
│              + Cloud Armor (WAF)                        │
└──────────┬──────────────────────────────┬───────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐  ┌──── VPC (asia-southeast1) ─────┐
│  Firebase Hosting   │  │                                 │
│  + Functions        │  │  ┌────────────────────────────┐ │
│  (Nuxt 4 SPA)       │  │  │   Google Cloud Run          │ │
│                     │  │  │   (Go API)                 │ │
│  tjmonsi.com        │  │  │   api.tjmonsi.com          │ │
│                     │  │  │                            │ │
│  Rewrites:          │  │  │  GET  /                    │ │
│  /sitemap.xml ──────│──│──│▶ GET  /sitemap.xml          │ │
│                     │  │  │  GET  /technical            │ │
│  Pages:             │  │  │  GET  /technical/{slug}.md  │ │
│  /                  │  │  │  GET  /blog                 │ │
│  /technical         │  │  │  GET  /blog/{slug}.md       │ │
│  /technical/:s.md   │  │  │  GET  /socials              │ │
│  /blog              │  │  │  GET  /others               │ │
│  /blog/:slug.md     │  │  │  GET  /categories           │ │
│  /socials           │  │  │  GET  /robots.txt           │ │
│  /others            │  │  │  GET  /health (internal)    │ │
│  /privacy           │  │  │  POST /t                    │ │
└─────────────────────┘  │  └─────────────┬──────────────┘ │
                         │                │                 │
                         │                ▼                 │
                         │  ┌────────────────────────────┐  │
                         │  │  Firestore Enterprise      │  │
                         │  │  (MongoDB compat mode)     │  │
                         │  │  asia-southeast1           │  │
                         │  └────────────┬───────────────┘  │
                         │               ▲                  │
                         │  ┌────────────┴───────────────┐  │
Cloud Scheduler ────────▶│  │  Cloud Function (Gen 2)    │  │
                         │  │  Sitemap Generation (Go)   │  │
                         │  └────────────────────────────┘  │
                         │                                   │
                         │  ┌────────────────────────────┐  │
Cloud Armor Log Sink ───▶│  │  Cloud Function (Gen 2)    │  │
                         │  │  Rate Limit Log Processing │  │
                         │  │  (Go) → writes to DM-009   │  │
                         │  └────────────────────────────┘  │
                         └──────────────────────────────────┘
```

### Key Architectural Decisions

| ID     | Decision                                      | Rationale                                                                 |
| ------ | --------------------------------------------- | ------------------------------------------------------------------------- |
| AD-001 | Nuxt 4 in SPA mode (no SSR)                   | Simpler deployment; Firebase Functions serves the SPA shell while Firebase Hosting serves static JS/CSS assets. See: https://nuxt.com/deploy/firebase |
| AD-002 | Go for backend API                             | Performance, low memory footprint on Cloud Run, strong typing             |
| AD-003 | Cloud Run behind Load Balancer + Cloud Armor   | Managed scaling, DDoS protection, WAF rules                              |
| AD-004 | GET-only API with single POST exception        | Read-only public surface reduces attack vectors. `POST /t` is the sole exception for visitor tracking and error reporting. |
| AD-005 | Fixed page size of 10 items                    | Simplifies API, predictable performance, no user-controlled query abuse   |
| AD-006 | Firestore Enterprise with MongoDB compatibility | Uses Google-native infrastructure with MongoDB wire protocol, enabling standard MongoDB Go driver and query language |
| AD-007 | API served on subdomain (`api.tjmonsi.com`) | Clean separation of frontend and backend, independent scaling and caching policies |
| AD-008 | Content managed via separate Git repository    | Articles and content are maintained in a dedicated repo; a CI/CD pipeline pushes content to the database on merge. Keeps the public API read-only. |
| AD-009 | Frontend SPA with offline reading support      | Not installable as PWA, but supports offline reading via smart prefetching and manual article saving in the browser |
| AD-010 | Free-form categories derived from articles     | Categories are stored in a dedicated collection and synced from article metadata. Frontend caches categories in sessionStorage for 24 hours. |
| AD-011 | VPC with Private Google Access                  | Cloud Run and Cloud Functions connect to Firestore via VPC, restricting egress to Google Cloud APIs only. No NAT router needed — minimizes cost and attack surface. |
| AD-012 | Cloud Function for sitemap generation            | Sitemap generation runs as a separate internal Cloud Function (Gen 2, Go), triggered by Cloud Scheduler every 6 hours. Keeps the API backend focused on serving requests. |
| AD-013 | Frontend routes include `.md` extension          | Article URLs use `.md` extension (e.g., `/technical/slug.md`) to present the appearance of accessing a markdown file, while content is dynamically fetched from the backend API. |
| AD-014 | Cloud Function for offense tracking from Cloud Armor logs | A log sink routes Cloud Armor rate-limit (429) events to a Cloud Function, which writes offense records to the `rate_limit_offenders` Firestore collection (DM-009). This bridges the gap between Cloud Armor's request-level rate limiting and the application's progressive banning logic. |
| AD-015 | Cloud Armor Adaptive Protection enabled           | Cloud Armor's adaptive protection provides automatic, ML-based escalating DDoS mitigation. This complements application-level progressive banning with infrastructure-level protection that requires no manual rule updates. |

### Deployment Topology

- **Region (Firestore)**: `asia-southeast1`
- **Region (Cloud Run)**: `asia-southeast1`
- **Domain**: `tjmonsi.com` (frontend), `api.tjmonsi.com` (backend API)
- **VPC**: `personal-website-vpc` in `asia-southeast1` with minimum subnets for Cloud Run and Cloud Functions connectors. Private Google Access enabled; no NAT router.
- **Cloud Run**: Min instances = 0 (scale to zero), Max instances = TBD based on budget
- **Cloud Functions**: Sitemap generation function (Gen 2, Go) running internally in `asia-southeast1`
- **Cloud Functions**: Log processing function (Gen 2, Go) triggered by Cloud Armor log sink in `asia-southeast1`
- **Firebase Hosting**: Global CDN distribution for static assets
- **Firebase Functions**: Serves the Nuxt 4 SPA shell
- **Database**: Same region as Cloud Run (`asia-southeast1`) for low latency

### Content Management

Content (articles, front page, social links, external references) is managed through a **separate Git repository**. A CI/CD pipeline in that repository processes markdown files and pushes structured content to the Firestore Enterprise database on merge. The public-facing API remains read-only (`GET`-only, except `POST /t` for tracking).

### Environments

| Environment | Purpose                         | Database          |
| ----------- | ------------------------------- | ----------------- |
| Development | Local development and testing   | Firestore Emulator / Local MongoDB |
| Staging     | Pre-production validation       | Separate Firestore Enterprise instance |
| Production  | Live website                    | Production Firestore Enterprise |
