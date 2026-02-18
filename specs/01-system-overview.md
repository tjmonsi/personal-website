---
title: System Overview
version: 3.1
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
| Sitemap Gen    | Node.js 22 LTS (Cloud Functions Gen 2) | Google Cloud Functions (asia-southeast1) |
| Log Processing | Node.js 22 LTS (Cloud Functions Gen 2) | Google Cloud Functions (asia-southeast1) |
| Offender Cleanup | Node.js 22 LTS (Cloud Functions Gen 2) | Google Cloud Functions (asia-southeast1) |
| Embedding Sync | Node.js 22 LTS (Cloud Functions Gen 2) | Google Cloud Functions (asia-southeast1) |
| Database       | Firestore Enterprise (MongoDB compatibility mode) | Google Cloud (asia-southeast1)  |
| Vector Search  | Firestore Native Mode                  | Google Cloud (asia-southeast1)           |
| Embedding      | Vertex AI — Gemini `gemini-embedding-001` | Google Cloud (asia-southeast1)           |
| CDN / WAF      | Google Cloud Load Balancer + Cloud Armor | Google Cloud                           |
| Networking     | VPC with Private Google Access + Direct VPC Egress (Production only) | Google Cloud (asia-southeast1)           |
| Scheduling     | Google Cloud Scheduler                 | Google Cloud (asia-southeast1)           |
| Messaging      | Google Cloud Pub/Sub                   | Google Cloud (asia-southeast1)           |
| DNS            | Google Cloud DNS                       | Google Cloud                             |
| Container Registry | Google Artifact Registry           | Google Cloud (asia-southeast1)           |
| Media Storage    | Google Cloud Storage                 | Google Cloud (asia-southeast1)           |
| GeoIP          | MaxMind GeoLite2-Country (embedded)  | Bundled in Docker image                  |
| Observability  | Google Cloud Logging + Cloud Monitoring + BigQuery | Google Cloud                |
| Analytics      | Looker Studio                          | Google Cloud (owner-operated)            |
| IaC            | Terraform (HashiCorp)                  | This repository (`/terraform/`)          |

### Definitions

| Term                            | Definition |
| ------------------------------- | ---------- |
| Firestore Enterprise            | Google Cloud's enterprise-grade document database. See: https://firebase.google.com/docs/firestore/enterprise/overview-enterprise-edition-modes |
| MongoDB Compatibility Mode      | Firestore Enterprise mode that exposes a MongoDB-compatible wire protocol, allowing use of standard MongoDB drivers and tools. See: https://firebase.google.com/docs/firestore/enterprise/mongodb-compatibility-overview |
| SPA                             | Single Page Application — client-side rendered web application |
| Cloud Run                       | Google Cloud's serverless container hosting platform |
| Cloud Armor                     | Google Cloud's WAF and DDoS protection service |
| Firebase Functions              | Cloud Functions integrated with Firebase, used here to serve the Nuxt 4 SPA |
| Cloud Functions Gen 2           | Google Cloud's second-generation serverless functions platform, built on Cloud Run infrastructure. Used here for internal sitemap generation, Cloud Armor log processing, rate limit offender cleanup, and article embedding synchronization. |
| Log Sink                        | A Cloud Logging export mechanism that routes matching log entries to a destination (e.g., Pub/Sub, Cloud Function) for further processing. Used here to route Cloud Armor rate-limit events to a Cloud Function. |
| Cloud Armor Adaptive Protection | A Cloud Armor feature (Standard tier) that uses machine learning to detect anomalous L7 traffic patterns and generate recommended protective rules for the owner to review and apply. Automatic rule deployment requires Enterprise tier, which is not used. |
| VPC                             | Virtual Private Cloud — isolated network environment for Google Cloud resources. Used in Production only; Development does not use a VPC to reduce cost. |
| Cloud Scheduler                 | Google Cloud's managed cron job service for scheduling periodic tasks |
| BigQuery                        | Google Cloud's serverless data warehouse for SQL-based log analytics. Used here to store exported Cloud Logging logs for long-term querying and Looker Studio integration. |
| Looker Studio                   | Google Cloud's business intelligence and data visualization tool. Used here as an owner-operated analytics dashboard connected to BigQuery via a service account. |
| Log Sink (BigQuery)             | A Cloud Logging export mechanism that routes matching log entries to a BigQuery dataset for long-term storage and SQL querying. Five sinks route logs to dedicated tables. |
| Firestore Native Mode           | Google Cloud Firestore in Native mode. A separate Firestore database used exclusively for vector similarity search. Distinct from Firestore Enterprise (MongoDB compat mode). See: https://firebase.google.com/docs/firestore |
| Vector Embedding                | A fixed-dimension numerical representation of text generated by a machine learning model (Gemini `gemini-embedding-001` via Vertex AI), enabling semantic similarity search via cosine distance. Embeddings are generated with `output_dimensionality=2048` and L2-normalized before storage. The maximum embedding dimension supported by Firestore Native is 2048. |
| Cosine Distance                 | A distance metric for comparing vector similarity. Range 0 (identical) to 2 (opposite). Lower values indicate higher semantic similarity. Firestore Native uses cosine distance for vector search. |
| UUID v5                         | A deterministic UUID generated from a namespace UUID and a name string using SHA-1 hashing. The same inputs always produce the same UUID. Used here to derive cache document IDs from search query strings. |
| Vertex AI                       | Google Cloud's machine learning platform. Used here to access the Gemini `gemini-embedding-001` model for generating text embeddings via the Vertex AI Embeddings API. Auth via IAM (`roles/aiplatform.user`). See: https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-text-embeddings |
| Workload Identity Federation    | A Google Cloud IAM feature that allows external workloads (e.g., GitHub Actions) to authenticate to GCP using short-lived OIDC tokens instead of long-lived service account keys. Used here for the content CI/CD pipeline to invoke the embedding sync Cloud Function (INFRA-014). See: https://cloud.google.com/iam/docs/workload-identity-federation |
| Terraform                       | An open-source infrastructure-as-code (IaC) tool by HashiCorp for declaratively provisioning and managing cloud resources. Used here to manage GCP infrastructure. Configuration files stored in this repository under `/terraform/`. See: https://www.terraform.io/ |
| Infrastructure as Code (IaC)    | The practice of managing and provisioning infrastructure through machine-readable definition files rather than manual processes. Enables version control, reproducibility, and auditability of infrastructure changes. |
| Terraform Remote State          | Terraform state stored in a shared backend (here, GCS bucket INFRA-015) instead of locally, enabling team collaboration and state locking to prevent concurrent modifications. |
| Cloud DNS                       | Google Cloud's managed DNS service. Used here to host the `tjmonsi.com` zone with A/AAAA records pointing to the Cloud Load Balancer. Managed by Terraform (INFRA-017). See: https://cloud.google.com/dns |
| Artifact Registry               | Google Cloud's container image registry. Used here to store Docker images for the Go backend API. Replaces the deprecated Container Registry (`gcr.io`). Managed by Terraform (INFRA-018). See: https://cloud.google.com/artifact-registry |
| Pub/Sub                         | Google Cloud's messaging service. Used here to route Cloud Armor rate-limit log entries from a Cloud Logging log sink to the `process-rate-limit-logs` Cloud Function (INFRA-008c). Topic: `rate-limit-events`. See: https://cloud.google.com/pubsub |

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
┌─────────────────────┐  ┌──── VPC (Production only) ──────┐
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
│  /others            │  │  │  GET  /images/{path}        │ │
│  /privacy           │  │  │  GET  /health (internal)    │ │
│  /changelog         │  │  │  POST /t                    │ │
└─────────────────────┘  │  └─────────────┬──────────────┘ │
                         │                │                 │
                         │                ▼                 │
                         │  ┌────────────────────────────┐  │
                         │  │  Firestore Enterprise      │  │
                         │  │  (MongoDB compat mode)     │  │
                         │  │  asia-southeast1           │  │
                         │  └────────────┬───────────────┘  │
                         │               ▲                  │
                         │  ┌────────────────────────────┐  │
                         │  │  Cloud Storage             │  │
                         │  │  <project-id>-media-bucket │  │
                         │  │  (images for articles)     │  │
                         │  └────────────────────────────┘  │
                         │               ▲                  │
                         │  Cloud Run reads via             │
                         │  GET /images/{path}              │
                         │                                   │
                         │  ┌────────────┴───────────────┐  │
Cloud Scheduler ────────▶│  │  Cloud Function (Gen 2)    │  │
                         │  │  Sitemap Generation       │  │
                         │  └────────────────────────────┘  │
                         │                                   │
                         │  ┌────────────────────────────┐  │
Cloud Armor Log Sink ───▶│  │  Pub/Sub                   │  │
                         │  │  (rate-limit-events)       │  │
                         │  └────────────┬───────────────┘  │
                         │               │                  │
                         │               ▼                  │
                         │  ┌────────────────────────────┐  │
                         │  │  Cloud Function (Gen 2)    │  │
                         │  │  Rate Limit Log Processing │  │
                         │  │  (Node.js) → writes to DM-009 │
                         │  └────────────────────────────┘  │
                         │                                   │
                         │  ┌────────────────────────────┐  │
Cloud Scheduler ────────▶│  │  Cloud Function (Gen 2)    │  │
                         │  │  Offender Cleanup (daily)  │  │
                         │  │  (Node.js) → cleans DM-009 │  │
                         │  └────────────────────────────┘  │
                         │                                   │
                         │  ┌────────────────────────────┐  │
Content CI/CD (WIF) ────▶│  │  Cloud Function (Gen 2)    │  │
Cloud Scheduler ────────▶│  │  Embedding Sync            │  │
                         │  │  (Node.js) → syncs vectors │  │
                         │  │  (daily safety net)        │  │
                         │  └────────────────────────────┘  │
                         └──────────────────────────────────┘

              ┌──────────────────────────────────┐
              │     Firestore Native Mode        │
              │     (Vector Search)              │
              │     asia-southeast1              │
              └──────────────────────────────────┘
                         ▲                ▲
              Cloud Run queries     Embedding Sync
              for nearest vectors   Cloud Function writes
                                        │
              ┌──────────────────────────────────┐
              │     Vertex AI (Gemini)           │
              │     gemini-embedding-001          │
              └──────────────────────────────────┘
                         ▲
              Cloud Run + Embedding Sync
              call for embeddings

              Cloud Logging (receives all service logs)
                           │
                           ▼ (5 Log Sinks → INFRA-010)
              ┌──────────────────────────────────┐
              │      BigQuery (website_logs)     │
              └────────────────┬─────────────────┘
                           │
                           ▼
              ┌──────────────────────────────────┐
              │   Looker Studio (Analytics)      │
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
| AD-011 | VPC with Private Google Access and Direct VPC Egress (Production only) | In Production, Cloud Run and Cloud Functions use Direct VPC Egress to route traffic through the VPC, restricting egress to Google Cloud APIs only. Direct VPC Egress requires no separate connector instances, reducing cost versus Serverless VPC Access Connectors. No NAT router needed — minimizes cost and attack surface. The Development environment does NOT use a VPC to reduce cost; services connect to Google Cloud APIs directly. Reference: https://cloud.google.com/run/docs/configuring/vpc-direct-vpc |
| AD-012 | Cloud Function for sitemap generation            | Sitemap generation runs as a separate internal Cloud Function (Gen 2, Node.js), triggered by Cloud Scheduler every 6 hours. Keeps the API backend focused on serving requests. |
| AD-013 | Frontend routes include `.md` extension          | Article URLs use `.md` extension (e.g., `/technical/slug.md`) to present the appearance of accessing a markdown file, while content is dynamically fetched from the backend API. |
| AD-014 | Cloud Function for offense tracking from Cloud Armor logs | A log sink routes Cloud Armor rate-limit (429) events to a Cloud Function, which writes offense records to the `rate_limit_offenders` Firestore collection (DM-009). This bridges the gap between Cloud Armor's request-level rate limiting and the application's progressive banning logic. |
| AD-015 | Cloud Armor Adaptive Protection enabled           | Cloud Armor's adaptive protection (Standard tier) provides ML-based DDoS detection with recommended rules that the owner can review and apply. This complements application-level progressive banning with infrastructure-level anomaly detection. Automatic rule enforcement is not available in Standard tier — the owner is alerted and applies suggested rules manually. |
| AD-016 | BigQuery log sinks for analytics                  | All Cloud Logging logs are exported to a BigQuery dataset (`website_logs`) via 5 dedicated log sinks — enabling SQL-based analytics, up to 2 years of retention, and Looker Studio dashboards without impacting operational logging. IP addresses are truncated before logging to ensure anonymization in BigQuery. |
| AD-017 | Looker Studio for analytics dashboards            | Owner-operated Looker Studio dashboards connect to BigQuery via a service account with read-only access, providing visitor analytics (unique visitors, page views, referrer sources, etc.) without third-party analytics tools. |
| AD-018 | Dual-database: Firestore Enterprise + Firestore Native | Firestore Enterprise (MongoDB compat) stores all application data. Firestore Native stores vector embeddings for semantic article search. Two databases in the same project, each optimized for its purpose. The Go API uses the MongoDB Go driver for Enterprise and the Firestore Go SDK for Native. |
| AD-019 | Deterministic embedding cache with UUID v5         | Search query embeddings are cached in Firestore Enterprise (`embedding_cache` collection, DM-011) using UUID v5 of the lowercased query as the document ID. No search strings are stored — only the deterministic UUID and the embedding vector. No cache expiration. The UUID v5 namespace is the standard URL namespace (`6ba7b811-9dad-11d1-80b4-00c04fd430c8`) with name = lowercased search text. |
| AD-020 | Cosine distance threshold for vector search        | Search results with cosine distance > 0.35 (cosine similarity < 0.65) are excluded. This threshold balances recall and precision for semantic article search. Configurable per deployment. Reference: https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-text-embeddings |
| AD-021 | Vector search replaces text search for `q` parameter | When the `q` query parameter is present, the system uses Gemini embedding + Firestore Native vector similarity search instead of MongoDB text indexes. Without `q`, queries use Firestore Enterprise directly. Filtering (category, tags, date range) is always applied in Firestore Enterprise. |
| AD-022 | Workload Identity Federation for content CI/CD | The content repository's GitHub Actions pipeline authenticates to GCP via Workload Identity Federation (WIF) — no long-lived service account keys. WIF provides OIDC-based authentication to invoke the `sync-article-embeddings` Cloud Function (INFRA-014) after pushing content. Reference: https://cloud.google.com/iam/docs/workload-identity-federation |
| AD-023 | Terraform for infrastructure management | GCP infrastructure is managed declaratively via Terraform. Configuration files are stored in this repository under `/terraform/`. State is stored remotely in a GCS bucket (INFRA-015) with versioning and locking. The state bucket and Terraform service account are manually created bootstrap resources. Project ID is obfuscated in documentation for security. Reference: https://www.terraform.io/ |

### Deployment Topology

- **Region (Firestore)**: `asia-southeast1`
- **Region (Cloud Run)**: `asia-southeast1`
- **Domain**: `tjmonsi.com` (frontend), `api.tjmonsi.com` (backend API)
- **VPC** (Production only): `personal-website-vpc` in `asia-southeast1` with a `/28` subnet for Cloud Run and Cloud Functions Direct VPC Egress. Private Google Access enabled; no NAT router, no Serverless VPC Access Connector. Reference: https://cloud.google.com/run/docs/configuring/vpc-direct-vpc. The Development environment does NOT use a VPC to reduce cost.
- **Cloud Run**: Min instances = 0 (scale to zero), Max instances = 5
- **Cloud Functions**: Sitemap generation function (Gen 2, Node.js) running internally in `asia-southeast1`
- **Cloud Functions**: Log processing function (Gen 2, Node.js) triggered by Cloud Armor log sink via Pub/Sub (`rate-limit-events` topic, see INFRA-008c) in `asia-southeast1`
- **Cloud Functions**: Rate limit offender cleanup function (Gen 2, Node.js) triggered by Cloud Scheduler daily in `asia-southeast1`
- **Cloud Functions**: Embedding sync function (Gen 2, Node.js) triggered by Content CI/CD pipeline (via Workload Identity Federation) and Cloud Scheduler (daily safety net, INFRA-014b) to sync content embeddings to Firestore Native in `asia-southeast1`
- **BigQuery**: Dataset `website_logs` in `asia-southeast1` with 5 tables fed by Cloud Logging log sinks (see INFRA-010)
- **Looker Studio**: Owner-operated analytics dashboards connected to BigQuery via service account (see INFRA-011)
- **Firebase Hosting**: Global CDN distribution for static assets
- **Firebase Functions**: Serves the Nuxt 4 SPA shell
- **Database**: Same region as Cloud Run (`asia-southeast1`) for low latency
- **Firestore Native**: Vector search database in `asia-southeast1` with vector indexes on embedding fields (2048 dimensions, cosine distance). Max embedding dimension supported by Firestore Native is 2048. See: https://cloud.google.com/firestore/native/docs/vector-search
- **Vertex AI**: Gemini `gemini-embedding-001` model in `asia-southeast1` for generating text embeddings via IAM-based auth
- **Terraform**: Infrastructure managed via Terraform (INFRA-016) with remote state in GCS (INFRA-015). Terraform service account (SEC-012) authenticates via Workload Identity Federation in CI/CD. Configuration files stored in `/terraform/` directory.
- **Artifact Registry**: Docker repository `website-images` in `asia-southeast1` for Go backend container images (INFRA-018).

### Content Management

Content (articles, front page, social links, external references) is managed through a **separate Git repository**. A CI/CD pipeline (GitHub Actions) in that repository processes markdown files and pushes structured content to the Firestore Enterprise database on merge. The pipeline authenticates to GCP via **Workload Identity Federation** (AD-022) — no long-lived service account keys. After pushing content, the pipeline invokes the `sync-article-embeddings` Cloud Function (INFRA-014) to synchronize embedding vectors. The content pipeline's implementation is out of scope for this project; only the integration contract is specified (see INFRA-014 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)). The public-facing API remains read-only (`GET`-only, except `POST /t` for tracking).

### Environments

| Environment | Purpose                                                   | Database          |
| ----------- | --------------------------------------------------------- | ----------------- |
| Development | Local development and testing only (no GCP deployment)    | Firestore Emulator / Local MongoDB (Firestore Enterprise), Local Firestore Emulator (Firestore Native) |
| Production  | Live website                                              | Production Firestore Enterprise + Production Firestore Native |

> **Note**: Only two environments are maintained. The Development environment is **local-only** — it runs entirely on the developer's machine using emulators and local databases. No GCP resources are provisioned or deployed for Development. The `dev` branch is used for local development and testing; code is merged to `main` for Production deployment. The owner can deploy immediately, so no separate staging GCP project is needed.

> **Note on VPC**: The Production environment uses a VPC with Direct VPC Egress (INFRA-009). Since the Development environment is local-only, VPC configuration is not applicable to Development. References to "Development does not use a VPC" in other specs reflect the local-only nature of the Dev environment — there are no GCP services in Dev that would require a VPC.

---

### Acceptance Criteria

- **AC-SYS-001**: Given the production deployment, when the system is fully provisioned, then all components listed in the Technology Stack table are deployed and operational in `asia-southeast1`.
- **AC-SYS-002**: Given the architecture, when a user visits `tjmonsi.com`, then traffic routes through the Load Balancer → Cloud Armor → Firebase Hosting/Functions → Nuxt 4 SPA.
- **AC-SYS-003**: Given the architecture, when the frontend calls `api.tjmonsi.com`, then traffic routes through the Load Balancer → Cloud Armor → Cloud Run (Go backend).
- **AC-SYS-004**: Given the deployment topology, when the system is running, then the frontend and backend operate on separate subdomains (`tjmonsi.com` and `api.tjmonsi.com`).
- **AC-SYS-005**: Given the two-environment strategy, when developing locally, then all services (database, functions) run via emulators without requiring GCP resources.
- **AC-SYS-006**: Given the IaC approach, when infrastructure changes are needed, then all GCP resources (except manually bootstrapped ones) are managed via Terraform.
