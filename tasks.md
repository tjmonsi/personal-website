# Implementation Tasks

> **Generated**: 2026-02-22
> **Spec Versions**: 01 (v3.7) · 02 (v3.7) · 03 (v4.1) · 04 (v3.5) · 05 (v4.8) · 06 (v4.7) · 07 (v4.1)

Legend: `[ ]` not started · `[-]` in progress · `[x]` completed · `[!]` blocked

---

## Phase 0 — Prerequisites & Manual Bootstrap

> Must be completed before any Terraform or CI/CD work. See [manuals/pre-requisite-manual.md](manuals/pre-requisite-manual.md) for step-by-step instructions.

- [ ] **P0-001**: Create GCP project and link billing account
- [ ] **P0-002**: Create Terraform state bucket (`<project-id>-terraform-state`, `asia-southeast1`, versioning enabled) (INFRA-015)
- [ ] **P0-003**: Create Terraform SA (`terraform-builder@`) and grant 18 IAM roles (SEC-012)
- [ ] **P0-004**: Create WIF pool `terraform-cicd-pool` with `github-actions` OIDC provider (SEC-012)
- [ ] **P0-005**: Bind `terraform-builder@` and `app-deployer@` SAs to WIF pool (SEC-012, SEC-014)
- [ ] **P0-006**: Register MaxMind account and obtain `MAXMIND_LICENSE_KEY`
- [ ] **P0-007**: Configure GitHub Actions secrets (`GCP_PROJECT_ID`, `WIF_PROVIDER`, `TERRAFORM_SA_EMAIL`, `APP_DEPLOYER_SA_EMAIL`, `MAXMIND_LICENSE_KEY`)
- [ ] **P0-008**: Verify Firestore Enterprise (MongoDB compat mode) can be created via Terraform; if not, create manually (INFRA-006)
- [ ] **P0-009**: Register domain `tjmonsi.com` (if not already done) and point nameservers to Cloud DNS
- [ ] **P0-010**: Create `personal-website-infra` repository (or `/terraform/` directory in this repo) with Terraform project structure

---

## Phase 1 — Terraform Foundation

> Provisions all GCP infrastructure. Depends on Phase 0.

### 1A: Terraform Project Scaffold

- [ ] **T1-001**: Initialize Terraform project (`main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `backend.tf`, `versions.tf`)
- [ ] **T1-002**: Configure GCS backend for remote state (INFRA-015)
- [ ] **T1-003**: Configure `google` and `google-beta` providers (`asia-southeast1`)
- [ ] **T1-004**: Define input variables: `project_id`, `region`, `domain`, `alert_email`, environment flag (dev/prod)
- [ ] **T1-005**: Enable all 15 GCP APIs via `google_project_service` (INFRA-016)

### 1B: Networking & Security (Production Only)

- [ ] **T1-006**: Create VPC `personal-website-vpc` with `/27` subnet in `asia-southeast1` (INFRA-009)
- [ ] **T1-007**: Enable Private Google Access on subnet (INFRA-009)
- [ ] **T1-008**: Configure firewall rules (deny all egress, allow Google API ranges) (INFRA-009)
- [ ] **T1-009**: Create Cloud Armor security policy with rate limiting rules (INFRA-005)
  - Rule 1: Throttle rule — 60 requests/min per IP, 429 response
  - Rule 2: Ban rule — 5 bans in 60s → 10-min Cloud Armor ban
  - Adaptive Protection enabled (AD-015)
- [ ] **T1-010**: Create Regional External Application Load Balancer (INFRA-004)
  - Backend service → Cloud Run NEG
  - URL map: `api.tjmonsi.com/*` → Cloud Run, `tjmonsi.com/sitemap.xml` → Cloud Run
  - Google-managed SSL certificates for both domains
- [ ] **T1-011**: Create Cloud DNS zone and records for `tjmonsi.com` and `api.tjmonsi.com` (INFRA-017)

### 1C: Data Layer

- [ ] **T1-012**: Create Firestore Enterprise database (MongoDB compat mode, `asia-southeast1`) (INFRA-006)
- [ ] **T1-013**: Create Firestore Native database `vector-search` (`asia-southeast1`) (INFRA-012)
- [ ] **T1-014**: Create Cloud Storage media bucket `<project-id>-media-bucket` (INFRA-019)
  - Standard class, `asia-southeast1`, uniform bucket-level access
- [ ] **T1-015**: Create Artifact Registry Docker repository `website-images` (INFRA-018)
- [ ] **T1-016**: Create BigQuery dataset `website_logs` (`asia-southeast1`, 2-year retention) (INFRA-010)

### 1D: Service Accounts (Terraform-managed)

- [ ] **T1-017**: Create SA #1 `cloud-run-api@` with roles (SEC-013)
  - `roles/datastore.viewer` (Firestore Native), `roles/aiplatform.user`, `roles/storage.objectViewer` (media bucket), `roles/datastore.user` (Firestore Enterprise), `roles/monitoring.metricWriter`
- [ ] **T1-018**: Create SA #2 `cf-sitemap-gen@` with `roles/datastore.user` (SEC-013)
- [ ] **T1-019**: Create SA #3 `cf-rate-limit-proc@` with `roles/datastore.user` (SEC-013)
- [ ] **T1-020**: Create SA #4 `cf-offender-cleanup@` with `roles/datastore.user` (SEC-013)
- [ ] **T1-021**: Create SA #5 `cf-embed-sync@` with roles (SEC-013)
  - `roles/datastore.viewer` (Firestore Enterprise), `roles/datastore.user` (Firestore Native), `roles/aiplatform.user`
- [ ] **T1-022**: Create SA #6 `content-cicd@` with `roles/cloudfunctions.invoker` + `roles/run.invoker` on embedding sync function (SEC-010)
- [ ] **T1-023**: Create SA #8 `looker-studio-reader@` with `roles/bigquery.dataViewer` + `roles/bigquery.jobUser` (SEC-009)
- [ ] **T1-024**: Create SA #9 `cloud-scheduler-invoker@` with `roles/cloudfunctions.invoker` + `roles/run.invoker` on target functions (SEC-013)
- [ ] **T1-025**: Create SA #10 `app-deployer@` with 5 roles + WIF binding to `terraform-cicd-pool` (SEC-014)

### 1E: Pub/Sub, Log Sinks, Logging

- [ ] **T1-026**: Create Pub/Sub topic `rate-limit-events` (INFRA-008c)
- [ ] **T1-027**: Create Cloud Logging Pub/Sub sink: filter `resource.type="http_load_balancer" AND httpRequest.status=429` → `rate-limit-events` topic (INFRA-008c)
- [ ] **T1-028**: Create 6 BigQuery log sinks (INFRA-010a–010f)
  - 010a: `frontend_tracking_logs` — `jsonPayload.log_type="frontend_tracking"`
  - 010b: `backend_request_logs` — backend request logs
  - 010c: `frontend_error_logs` — `jsonPayload.log_type="frontend_error"`
  - 010d: `backend_error_logs` — ERROR severity from Cloud Run `website-api`
  - 010e: `cloud_armor_logs` — Cloud Armor decisions
  - 010f: `cloud_function_logs` — logs from all Cloud Functions
- [ ] **T1-029**: Configure Cloud Logging default bucket retention to 30 days (INFRA-007)

### 1F: Compute Resources (shell definitions — CI/CD deploys code)

- [ ] **T1-030**: Create Cloud Run service `website-api` (INFRA-003)
  - 1 vCPU, 1 GB memory, min 0 / max 5 instances, concurrency 160, 30s timeout
  - Startup CPU boost, ingress internal + Cloud Load Balancing
  - SA: `cloud-run-api@`, Direct VPC Egress (prod only)
  - `lifecycle { ignore_changes = [template[0].containers[0].image] }`
- [ ] **T1-031**: Create Cloud Functions placeholder definitions (INFRA-008a, 008c, 008d, INFRA-014)
  - Each: Node.js 22, 256 MB, `asia-southeast1`, respective SA, Direct VPC Egress (prod only)
  - `lifecycle { ignore_changes = [build_config[0].source] }`
- [ ] **T1-032**: Create 3 Cloud Scheduler jobs (INFRA-008b, 008e, 014b)
  - `trigger-sitemap-generation`: every 6 hours (`0 */6 * * *`)
  - `trigger-cleanup-rate-limit-offenders`: daily (`0 0 * * *`)
  - `trigger-sync-article-embeddings`: daily 04:00 UTC (`0 4 * * *`)
  - All use SA #9 OIDC token

### 1G: Firebase & WIF (content pipeline)

- [ ] **T1-033**: Create Firebase Hosting site resource (`google_firebase_hosting_site`) (INFRA-001)
- [ ] **T1-034**: Create WIF pool `content-cicd-pool` with `github-actions` OIDC provider (SEC-010)
- [ ] **T1-035**: Bind `content-cicd@` SA to `content-cicd-pool`

### 1H: Observability (Terraform-managed)

- [ ] **T1-036**: Create 17 Cloud Monitoring alert policies (OBS-005)
- [ ] **T1-037**: Configure notification channel (email) for alerts
- [ ] **T1-038**: Create SLO on Cloud Run service (99.9% availability, 28-day rolling window) (OBS-010)

### 1I: Terraform CI/CD Pipeline

- [ ] **T1-039**: Create `.github/workflows/terraform.yml` (INFRA-019)
  - Stages: fmt → validate → plan → apply (manual approval)
  - Trigger: push to `main` (terraform paths), PR for plan-only
  - Auth: WIF → `terraform-builder@`

---

## Phase 2 — Go Backend API

> Core backend implementation. Depends on Phase 1 (Firestore, Cloud Run provisioned).

### 2A: Project Scaffold

- [ ] **T2-001**: Initialize Go module (`go mod init`), add dependencies: Fiber v2, MongoDB Go driver, Firestore SDK, Vertex AI SDK, MaxMind GeoIP reader, JWT library
- [ ] **T2-002**: Create project structure per CODE_STYLE_GUIDE:
  ```
  cmd/api/main.go
  internal/handler/
  internal/service/
  internal/repository/
  internal/model/
  internal/middleware/
  internal/router/
  ```
- [ ] **T2-003**: Create Dockerfile (multi-stage: `golang:1.24-alpine` builder → `distroless/static-debian12`) (INFRA-003)
- [ ] **T2-004**: Create `.golangci.yml` linter configuration
- [ ] **T2-005**: Create `docker-compose.yml` for local dev (MongoDB + Firestore emulator)

### 2B: Middleware Chain

> Order matters — spec 03 defines exact middleware execution order.

- [ ] **T2-006**: Implement security headers middleware (SEC-005) — all 6 headers on every response
- [ ] **T2-007**: Implement request ID middleware (UUID per request)
- [ ] **T2-008**: Implement request logging middleware (structured JSON to stdout) (OBS-001)
- [ ] **T2-009**: Implement GeoIP middleware — resolve `geo_country` from client IP using embedded MaxMind DB before IP truncation (BE-API-009)
- [ ] **T2-010**: Implement IP truncation middleware — IPv4 last octet → 0, IPv6 last 80 bits → 0 (SEC-007)
- [ ] **T2-011**: Implement rate limit ban check middleware — query `rate_limit_offenders` collection (DM-009), LRU cache (60s TTL), return 403 if active ban (SEC-002)
- [ ] **T2-012**: Implement CORS middleware — allow `tjmonsi.com` origin, GET/POST/OPTIONS, specific headers (SEC-006)
- [ ] **T2-013**: Implement unknown query parameter rejection middleware — 400 for unrecognized params (SEC-004)
- [ ] **T2-014**: Implement custom metrics middleware — `api_requests_total`, `api_request_duration_ms` (OBS-004)

### 2C: Data Layer

- [ ] **T2-015**: Implement MongoDB client connection with connection pooling (Firestore Enterprise) (INFRA-006)
- [ ] **T2-016**: Implement Firestore Native client for vector search DB (INFRA-012)
- [ ] **T2-017**: Implement repository layer for articles (DM-002, DM-003): CRUD, pagination, filtering, sorting
- [ ] **T2-018**: Implement repository layer for `frontpage` (DM-001), `socials` (DM-004), `others` (DM-005), `categories` (DM-006)
- [ ] **T2-019**: Implement repository layer for `sitemap` (DM-010), `rate_limit_offenders` (DM-009), `embedding_cache` (DM-011)
- [ ] **T2-020**: Implement repository layer for Firestore Native vector collections (DM-012a/b/c)
- [ ] **T2-021**: Create all MongoDB indexes per spec 04 (9 Firestore Enterprise collections)
- [ ] **T2-022**: Create Firestore Native vector indexes (2048 dimensions, cosine distance) (INFRA-013)

### 2D: API Endpoints

- [ ] **T2-023**: `GET /health` (BE-API-010) — Firestore ping, return `{"status":"ok","timestamp":"..."}`
- [ ] **T2-024**: `GET /` (BE-API-001) — frontpage markdown, ETag/Last-Modified, conditional requests (304)
- [ ] **T2-025**: `GET /technical` (BE-API-002) — paginated list, `page`/`category`/`tags`/`q` params, fixed page size 10 (AD-005)
- [ ] **T2-026**: `GET /technical/{slug}.md` (BE-API-003) — full article as text/markdown with YAML front matter, dynamic citations (APA/MLA/Chicago/BibTeX/IEEE), ETag, conditional requests
- [ ] **T2-027**: `GET /blog` (BE-API-004) — same behavior as BE-API-002 for blog articles
- [ ] **T2-028**: `GET /blog/{slug}.md` (BE-API-005) — same behavior as BE-API-003 for blog articles
- [ ] **T2-029**: `GET /others` (BE-API-006) — paginated list with `page`/`category`/`tags`/`q` params
- [ ] **T2-030**: `GET /socials` (BE-API-007) — all active social links sorted by `sort_order`
- [ ] **T2-031**: `GET /categories` (BE-API-008) — all categories, filterable by `type` param
- [ ] **T2-032**: `POST /t` (BE-API-009) — JWT validation (HMAC-SHA256, 5-min exp, 60s skew), structured logging by action type, visitor_id computation (SHA-256 of session_id:truncated_ip:UA)
- [ ] **T2-033**: `GET /sitemap.xml` (BE-API-011) — return cached sitemap from DM-010
- [ ] **T2-034**: `GET /robots.txt` (BE-API-012) — static response referencing sitemap URL
- [ ] **T2-035**: `GET /images/{path}` (BE-API-013) — proxy from Cloud Storage media bucket, path traversal protection, `Cache-Control: public, max-age=86400`

### 2E: Vector Search

- [ ] **T2-036**: Implement Vertex AI Gemini embedding client (`gemini-embedding-001`, `output_dimensionality=2048`) (INFRA-013)
- [ ] **T2-037**: Implement embedding cache layer with UUID v5 keying (DM-011, AD-019)
- [ ] **T2-038**: Implement vector search query: get embedding → query Firestore Native (500 candidates, cosine ≤ 0.35) → fetch full articles from Firestore Enterprise (BE-API-002)
- [ ] **T2-039**: Integrate `q` parameter into `GET /technical`, `GET /blog`, `GET /others` endpoints

### 2F: Error Handling & Response Format

- [ ] **T2-040**: Implement standard error response format: `{"error":{"code":"...","message":"..."}}` (spec 03)
- [ ] **T2-041**: Implement 500 masking as 404 (SEC-004) — log true error at ERROR level with `masked_500` label
- [ ] **T2-042**: Implement allowed method enforcement (405 for non-GET/POST) (spec 03)
- [ ] **T2-043**: Implement payload size limit (413 for bodies > configured max) (spec 03)

### 2G: Testing

- [ ] **T2-044**: Unit tests for all middleware (table-driven, per CODE_STYLE_GUIDE)
- [ ] **T2-045**: Unit tests for all handlers/services
- [ ] **T2-046**: Integration tests against local MongoDB + Firestore emulator
- [ ] **T2-047**: Test progressive banning algorithm (3 offenses → 30d, 4 → 90d, 5+ → indefinite)
- [ ] **T2-048**: Test ETag/conditional request flow (304 responses)
- [ ] **T2-049**: Test vector search flow (embedding → cache → Firestore Native → article fetch)

---

## Phase 3 — Cloud Functions (Gen 2, Node.js 22)

> Depends on Phase 1 (Pub/Sub, Scheduler, SAs provisioned). Partially parallel with Phase 2.

- [ ] **T3-001**: Create Cloud Functions project structure (shared package.json or per-function)
- [ ] **T3-002**: Implement `generate-sitemap` (INFRA-008a)
  - Read `technical_articles` + `blog_articles` from Firestore Enterprise
  - Include hardcoded static page URLs (`/`, `/technical`, `/blog`, `/others`, `/socials`, `/privacy`, `/changelog`)
  - Generate valid sitemap XML
  - Write to `sitemap` collection (DM-010)
  - SA: `cf-sitemap-gen@`
- [ ] **T3-003**: Implement `process-rate-limit-logs` (INFRA-008c)
  - Triggered by Pub/Sub (`rate-limit-events` topic)
  - Parse Cloud Armor 429 log entry, extract full IP
  - `findOneAndUpdate` with `$inc` + `$push` + `$setOnInsert` + `upsert: true` (DM-009)
  - Evaluate progressive banning: 3 offenses → 30d, 4 → 90d, 5+ → indefinite
  - SA: `cf-rate-limit-proc@`
- [ ] **T3-004**: Implement `cleanup-rate-limit-offenders` (INFRA-008d)
  - HTTP-triggered by Cloud Scheduler
  - Delete records: no active ban AND no offenses in last 90 days
  - SA: `cf-offender-cleanup@`
- [ ] **T3-005**: Implement `sync-article-embeddings` (INFRA-014)
  - Read all articles from Firestore Enterprise (technical, blog, others)
  - Compute `embedding_text_hash` per article
  - Compare against existing embeddings in Firestore Native (DM-012)
  - Call Vertex AI for new/changed articles only
  - Write/update embedding documents in Firestore Native
  - Update `embedding_cache` (DM-011) with UUID v5 keys
  - SA: `cf-embed-sync@`
- [ ] **T3-006**: Unit tests for all Cloud Functions
- [ ] **T3-007**: Integration tests against Firestore emulator

---

## Phase 4 — Nuxt 4 Frontend (SPA)

> Depends on Phase 2 API being available (can use mocks initially). Parallel with Phase 3.

### 4A: Project Scaffold

- [ ] **T4-001**: Initialize Nuxt 4 project with Firebase preset (`nuxt build --preset=firebase`)
- [ ] **T4-002**: Configure `nuxt.config.ts`: SPA mode, runtime config (`apiBaseUrl`), modules
- [ ] **T4-003**: Configure ESLint 9 flat config: `withNuxt()` + `neostandard({ noStyle: true })` + `@stylistic` (CODE_STYLE_GUIDE)
- [ ] **T4-004**: Create `firebase.json` with rewrites (sitemap → Cloud Run, `**` → `server` function) + security headers (SEC-005)
- [ ] **T4-005**: Set up Vitest for frontend testing
- [ ] **T4-006**: Create API client composable (`useApi`) with base URL, error handling, ETag support

### 4B: Layout & Global Components

- [ ] **T4-007**: Create default layout with header + footer + main content area
- [ ] **T4-008**: Implement FE-COMP-012 (Site Header Navigation) — horizontal desktop nav, hamburger + drawer on mobile, dark mode toggle (light → dark → auto)
- [ ] **T4-009**: Implement FE-COMP-010 (Site Footer) — navigation links, copyright
- [ ] **T4-010**: Implement FE-COMP-002 (Loading Bar) — top-of-page progress on API requests
- [ ] **T4-011**: Implement FE-COMP-003 (Error Snackbar) — status-code-mapped messages, 5s auto-dismiss

### 4C: Pages

- [ ] **T4-012**: Implement FE-PAGE-001 (Front Page `/`) — fetch `GET /`, render markdown
- [ ] **T4-013**: Implement FE-PAGE-002 (Technical Blog List `/technical`) — paginated list, desktop pagination + mobile infinite scroll
- [ ] **T4-014**: Implement FE-PAGE-003 (Technical Article `/technical/:slug.md`) — markdown render, TOC, metadata, changelog, citation selector, heading anchors
- [ ] **T4-015**: Implement FE-PAGE-004 (Blog List `/blog`) — same behavior as technical list
- [ ] **T4-016**: Implement FE-PAGE-005 (Blog Article `/blog/:slug.md`) — same as technical article
- [ ] **T4-017**: Implement FE-PAGE-006 (Socials `/socials`) — social links with icons
- [ ] **T4-018**: Implement FE-PAGE-007 (Others `/others`) — paginated external content list
- [ ] **T4-019**: Implement FE-PAGE-008 (404) — catch-all, 30 randomized humorous anecdotes
- [ ] **T4-020**: Implement FE-PAGE-009 (Privacy `/privacy`) — static markdown, no API call
- [ ] **T4-021**: Implement FE-PAGE-010 (Changelog `/changelog`) — static markdown, no API call

### 4D: Shared Components

- [ ] **T4-022**: Implement FE-COMP-001 (Search Bar) — 300-char limit, debounced input
- [ ] **T4-023**: Implement FE-COMP-014 (Tag Chip Input) — comma/Enter-delimited, lowercase, max 10, removable chips
- [ ] **T4-024**: Implement FE-COMP-011 (URL State Sync) — query params ↔ `history.pushState()` + `popstate`, hash fragment for headings
- [ ] **T4-025**: Implement FE-COMP-007 (Category Caching) — per-type localStorage with 24h TTL
- [ ] **T4-026**: Implement FE-COMP-009 (Empty Search Results) — randomized messages, escalating humor on repeated similar queries (Levenshtein ≤ 2)

### 4E: Tracking & Error Reporting

- [ ] **T4-027**: Implement FE-COMP-013 (Activity Breadcrumb Trail) — in-memory ring buffer (50 entries)
- [ ] **T4-028**: Implement FE-COMP-004 (Visitor Tracking) — JWT client-side signing, `sendBeacon()`, page_view/link_click/time_on_page
- [ ] **T4-029**: Implement FE-COMP-005 (Frontend Error Reporting) — `POST /t` with `error_report` action, retry (5 attempts, exponential backoff), modal fallback
- [ ] **T4-030**: Create `robots.txt` static file in `public/` directory (FE-COMP-006)

### 4F: Offline & Service Worker

- [ ] **T4-031**: Implement FE-COMP-008 (Offline Reading) — service worker, cache-first strategy
  - Smart download prefetching (3 articles at 30% scroll, 4g only, 50MB cap)
  - Manual "Save for offline" button
  - Reading progress tracking (IndexedDB)
  - Stale-while-revalidate lifecycle, toast on new version, `SKIP_WAITING` on user action

### 4G: Frontend Testing

- [ ] **T4-032**: Unit tests for all composables and utility functions
- [ ] **T4-033**: Component tests for core components (search bar, tag input, error snackbar)
- [ ] **T4-034**: Page-level tests with mocked API responses

---

## Phase 5 — CI/CD Pipelines

> Depends on Phases 1-4 having buildable code. App CI/CD deploys backend + frontend + Cloud Functions.

### 5A: Application CI/CD

- [ ] **T5-001**: Create `.github/workflows/deploy.yml` (INFRA-020)
  - Trigger: push to `main`, PR to `main` (no deploy on PR)
  - Stage 1 — Lint: `golangci-lint`, `eslint` (frontend), `eslint` (Cloud Functions)
  - Stage 2 — Test: `go test ./...`, `vitest`, Node.js tests
  - Stage 3 — Build: Docker image (`backend:<git-sha-short>`), `nuxt build`
  - Stage 4 — Scan: `govulncheck`, `npm audit`, `trivy` (Docker image)
  - Stage 5 — Deploy (main only):
    - Push Docker image to Artifact Registry
    - `gcloud run deploy` with SA #1
    - `firebase deploy --only hosting,functions`
    - `gcloud functions deploy` × 4 Gen 2 functions with respective SAs
  - Auth: WIF → `app-deployer@`
- [ ] **T5-002**: Create `.github/workflows/geoip-refresh.yml`
  - Trigger: weekly schedule (`0 0 * * 0`)
  - Download latest GeoLite2-Country DB
  - Rebuild and redeploy Docker image

### 5B: Initial Deployment & Verification

- [ ] **T5-003**: Execute first Terraform apply (all infrastructure)
- [ ] **T5-004**: Execute first application deployment (backend + frontend + Cloud Functions)
- [ ] **T5-005**: Verify Cloud Run health check (`GET /health`)
- [ ] **T5-006**: Verify Firebase Hosting serves SPA shell
- [ ] **T5-007**: Verify Cloud Scheduler triggers all 3 functions
- [ ] **T5-008**: Verify Cloud Armor rate limiting (429 responses)
- [ ] **T5-009**: Verify BigQuery log sinks are receiving data
- [ ] **T5-010**: Verify DNS resolution for `tjmonsi.com` and `api.tjmonsi.com`
- [ ] **T5-011**: Verify SSL certificates are active

---

## Phase 6 — Observability & Dashboards

> Depends on Phase 5 (live system needed for meaningful data).

- [ ] **T6-001**: Verify all 17 alert policies fire correctly (test each condition)
- [ ] **T6-002**: Verify SLO monitoring and burn rate alerting (OBS-010)
- [ ] **T6-003**: Create Cloud Monitoring operational dashboard (OBS-006)
  - Request rate, latency percentiles, error rate, instance count, DB query latency, ban count, cache ratio, embedding latency, vector search metrics
- [ ] **T6-004**: Create Looker Studio analytics dashboard (INFRA-011)
  - Connect via `looker-studio-reader@` SA
  - Sections: unique visitors, page views, top pages, referrers, browsers, geo, connection speed, link clicks, time-on-page, content helpfulness, bot detection, error trends, Cloud Armor activity
- [ ] **T6-005**: Generate Looker Studio SA key and configure 90-day rotation reminder

---

## Phase 7 — Content Pipeline Integration (Separate Repo)

> Out of scope for this project, but documents the interface. Depends on all prior phases.

- [ ] **T7-001**: Document content pipeline setup in content repo README
- [ ] **T7-002**: Verify WIF authentication from content repo GitHub Actions → `content-cicd@`
- [ ] **T7-003**: Verify content pipeline writes to all 6 Firestore Enterprise collections (frontpage, technical_articles, blog_articles, socials, others, categories) per Integration Contract
- [ ] **T7-004**: Verify `sync-article-embeddings` invocation from content pipeline
- [ ] **T7-005**: End-to-end validation: content push → Firestore → embeddings sync → search works

---

## Dependency Graph

```
Phase 0 (Manual Bootstrap)
  └──▶ Phase 1 (Terraform)
         ├──▶ Phase 2 (Go Backend)  ──────────┐
         ├──▶ Phase 3 (Cloud Functions)        ├──▶ Phase 5 (CI/CD & Deploy)
         └──▶ Phase 4 (Nuxt Frontend) ────────┘         │
                                                         ▼
                                                  Phase 6 (Observability)
                                                         │
                                                         ▼
                                                  Phase 7 (Content Pipeline)
```

**Parallelization**: Phases 2, 3, and 4 can run in parallel once Phase 1 is complete. Phase 4 can start with mocked APIs before Phase 2 is complete.

---

## Task Summary

| Phase | Tasks | Description |
|-------|-------|-------------|
| 0 | 10 | Manual bootstrap prerequisites |
| 1 | 39 | Terraform infrastructure |
| 2 | 49 | Go backend API |
| 3 | 7 | Cloud Functions (Gen 2) |
| 4 | 34 | Nuxt 4 frontend |
| 5 | 11 | CI/CD pipelines & deployment |
| 6 | 5 | Observability dashboards |
| 7 | 5 | Content pipeline integration |
| **Total** | **160** | |
