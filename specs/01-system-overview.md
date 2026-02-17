---
title: System Overview
version: 1.2
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
| Database       | Firestore Enterprise (MongoDB compatibility mode) | Google Cloud (asia-southeast1)  |
| CDN / WAF      | Google Cloud Load Balancer + Cloud Armor | Google Cloud                           |
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

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      End Users                          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Cloud DNS     │
              │ tjmonserrat.com │
              │ api.tjmonserrat │
              │     .com        │
              └────────┬────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│           Google Cloud Load Balancer                    │
│              + Cloud Armor (WAF)                        │
└──────────┬──────────────────────────────┬───────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐     ┌────────────────────────────┐
│  Firebase Hosting   │     │   Google Cloud Run          │
│  + Functions        │     │   (Go API)                 │
│  (Nuxt 4 SPA)       │     │   api.tjmonserrat.com      │
│                     │     │                            │
│  tjmonserrat.com    │     │  GET  /                    │
│                     │     │  GET  /technical            │
│  Static assets +    │     │  GET  /technical/{slug}.md  │
│  SPA serving via    │     │  GET  /blog                 │
│  Firebase Functions │     │  GET  /blog/{slug}.md       │
│                     │     │  GET  /socials              │
│                     │     │  GET  /others               │
│                     │     │  GET  /categories           │
│                     │     │  GET  /sitemap.xml          │
│                     │     │  POST /t                    │
└─────────────────────┘     └─────────────┬──────────────┘
                                          │
                                          ▼
                            ┌────────────────────────────┐
                            │  Firestore Enterprise      │
                            │  (MongoDB compat mode)     │
                            │  asia-southeast1           │
                            └────────────────────────────┘
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
| AD-007 | API served on subdomain (`api.tjmonserrat.com`) | Clean separation of frontend and backend, independent scaling and caching policies |
| AD-008 | Content managed via separate Git repository    | Articles and content are maintained in a dedicated repo; a CI/CD pipeline pushes content to the database on merge. Keeps the public API read-only. |
| AD-009 | Frontend SPA with offline reading support      | Not installable as PWA, but supports offline reading via smart prefetching and manual article saving in the browser |
| AD-010 | Free-form categories derived from articles     | Categories are stored in a dedicated collection and synced from article metadata. Frontend caches categories in sessionStorage for 24 hours. |

### Deployment Topology

- **Region (Firestore)**: `asia-southeast1`
- **Region (Cloud Run)**: `asia-southeast1`
- **Domain**: `tjmonserrat.com` (frontend), `api.tjmonserrat.com` (backend API)
- **Cloud Run**: Min instances = 0 (scale to zero), Max instances = TBD based on budget
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
