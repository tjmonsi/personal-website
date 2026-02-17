## System Overview

### Purpose

A personal website for TJ Monserrat serving as a professional online presence with a technical blog, opinion blog, social links, and curated external content references.

### Technology Stack

| Layer          | Technology                             | Hosting                                  |
| -------------- | -------------------------------------- | ---------------------------------------- |
| Frontend       | Nuxt 4 / Vue 3 / Vite (Static/SPA)    | Firebase Hosting + Firebase Functions    |
| Backend API    | Go                                     | Google Cloud Run                         |
| Database       | *See Clarification CLR-001*            | Google Cloud (Firestore or MongoDB Atlas)|
| CDN / WAF      | Google Cloud Load Balancer + Cloud Armor | Google Cloud                           |
| Observability  | Google Cloud Logging + Cloud Monitoring | Google Cloud                            |

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      End Users                          │
└──────────────────────┬──────────────────────────────────┘
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
│  (Nuxt 4 SPA)       │     │   (Go API)                 │
│                     │     │                            │
│  Static assets      │     │  GET /                     │
│  Client-side        │     │  GET /technical             │
│  rendering          │     │  GET /technical/{slug}.md   │
│                     │     │  GET /blog                  │
│                     │     │  GET /blog/{slug}.md         │
│                     │     │  GET /socials                │
│                     │     │  GET /others                 │
└─────────────────────┘     └─────────────┬──────────────┘
                                          │
                                          ▼
                            ┌────────────────────────────┐
                            │   Database                 │
                            │   (See CLR-001)            │
                            └────────────────────────────┘
```

### Key Architectural Decisions

| ID     | Decision                                      | Rationale                                                                 |
| ------ | --------------------------------------------- | ------------------------------------------------------------------------- |
| AD-001 | Nuxt 4 in static/SPA mode (no SSR)            | Simpler deployment, no server-side rendering costs, content from API      |
| AD-002 | Go for backend API                             | Performance, low memory footprint on Cloud Run, strong typing             |
| AD-003 | Cloud Run behind Load Balancer + Cloud Armor   | Managed scaling, DDoS protection, WAF rules                              |
| AD-004 | GET-only API                                   | Read-only public surface, reduces attack vectors                          |
| AD-005 | Fixed page size of 10 items                    | Simplifies API, predictable performance, no user-controlled query abuse   |

### Deployment Topology

- **Region**: *To be determined* (single region initially, multi-region if needed)
- **Cloud Run**: Min instances = 0 (scale to zero), Max instances = TBD based on budget
- **Firebase Hosting**: Global CDN distribution
- **Database**: Same region as Cloud Run for low latency

### Environments

| Environment | Purpose                         | Database     |
| ----------- | ------------------------------- | ------------ |
| Development | Local development and testing   | Local/Emulator |
| Staging     | Pre-production validation       | Separate DB  |
| Production  | Live website                    | Production DB |
