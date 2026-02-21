# Gap Analysis v9 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07) after all v8 (6 findings) gaps were applied. This analysis verifies v8 fixes, performs IAM role audit, cross-spec reference validation, data flow tracing, algorithm validation, and searches for anything missed in 8 previous rounds.

**Methodology**:
- Verified all 6 v8 fixes were correctly applied across affected specs
- Audited SEC-012 (18 roles) and SEC-013 SA #7 (18 roles) for consistency and coverage
- Audited SEC-014 / SEC-013 SA #10 against GCP deployment permission requirements
- Traced 5 end-to-end data flows
- Validated 4 algorithms (progressive banning, vector search, ETag, ban cache LRU)
- Validated all 6 BigQuery table names, 10 service accounts, 13 API endpoints, and DM field names across all specs
- Searched for stale references introduced by v8 changes

---

## 1. v8 Fix Verification

All 6 v8 fixes verified as correctly applied:

| v8 Finding | Fix Applied | Verified In |
|------------|-------------|-------------|
| FINDING-001: `roles/storage.objectAdmin` → `roles/storage.admin` | SEC-012 table updated, scope changed to Project, purpose updated | Spec 06 SEC-012 table — `roles/storage.admin \| Project \| Create and manage Cloud Storage buckets (INFRA-019) and read/write Terraform state (INFRA-015)` ✅ |
| FINDING-002: Added `roles/serviceusage.serviceUsageAdmin` | SEC-012 table and SEC-013 SA #7 updated | Spec 06 SEC-012 table — `roles/serviceusage.serviceUsageAdmin \| Project \| Enable and disable GCP APIs (INFRA-016 API enablement)` ✅ |
| FINDING-003: Added `roles/iam.serviceAccountUser` | SEC-012 table and SEC-013 SA #7 updated | Spec 06 SEC-012 table — `roles/iam.serviceAccountUser \| Project \| Assign service accounts to Cloud Run and Cloud Functions resources (actAs permission)` ✅ |
| FINDING-004: `roles/datastore.user` → `roles/datastore.owner` | SEC-012 table and SEC-013 SA #7 updated, purpose updated | Spec 06 SEC-012 table — `roles/datastore.owner \| Project \| Create and manage Firestore Enterprise and Native databases, entities, and indexes (INFRA-006, INFRA-012)` ✅ |
| FINDING-005: DM-005 `content_hash` description corrected | Description now says "Not consumed within this system" | Spec 04 DM-005 — `content_hash` description: "Not consumed within this system — the sync-article-embeddings function uses embedding_text_hash (DM-012) for change detection, and no /others detail endpoint exists for ETag handling. Retained for content CI/CD pipeline internal use." ✅ |
| FINDING-006: AC-OBS-010 lists all 10 metrics | `api_bans_active` and `db_connection_pool_active` added | Spec 07 AC-OBS-010 — now lists all 10 metrics: `api_requests_total`, `api_request_duration_ms`, `api_bans_active`, `db_query_duration_ms`, `db_connection_pool_active`, `embedding_cache_hits_total`, `embedding_cache_misses_total`, `embedding_api_duration_ms`, `vector_search_duration_ms`, and `vector_search_candidates` ✅ |

---

## 2. IAM Role Audit

### SEC-012 / SEC-013 SA #7 Role Count

**SEC-012 Terraform SA Granted Roles table**: 18 roles.
**SEC-013 SA #7 Key IAM Roles list**: 18 roles.
**Match**: ✅ Both lists contain the same 18 roles.

> **Note**: The v8 report summary stated "Currently 17 roles; would become 20 with fixes." The correct pre-v8 count was 16 (not 17), and the correct post-v8 count is 18 (not 20). This was an arithmetic error in the v8 report and has no impact on the current specs.

### SEC-012 Coverage vs INFRA-016 Scope

Every item in INFRA-016's Terraform scope has a corresponding role:

| INFRA-016 Scope Item | SEC-012 Role(s) | ✅ |
|----------------------|-----------------|---|
| GCP API enablement | `roles/serviceusage.serviceUsageAdmin` | ✅ |
| Cloud Run services (INFRA-003) | `roles/run.admin` | ✅ |
| Cloud Functions (INFRA-008a, 008c, 008d, INFRA-014) | `roles/cloudfunctions.admin` | ✅ |
| VPC and networking (INFRA-009) | `roles/compute.networkAdmin` | ✅ |
| Cloud Armor (INFRA-005) | `roles/compute.securityAdmin` | ✅ |
| Cloud Load Balancer (INFRA-004) | `roles/compute.networkAdmin` + `roles/compute.securityAdmin` | ✅ |
| Cloud Scheduler jobs (INFRA-008b, 008e, 014b) | `roles/cloudscheduler.admin` | ✅ |
| Firebase Hosting site resource (INFRA-001) | `roles/firebasehosting.admin` | ✅ |
| Cloud DNS (INFRA-017) | `roles/dns.admin` | ✅ |
| Artifact Registry (INFRA-018) | `roles/artifactregistry.admin` | ✅ |
| Cloud Logging log sinks + bucket retention (INFRA-007, 010) | `roles/logging.admin` | ✅ |
| Cloud Monitoring alerting + dashboards (OBS-005, OBS-006) | `roles/monitoring.admin` | ✅ |
| IAM service accounts (SEC-013) | `roles/iam.serviceAccountAdmin` | ✅ |
| WIF pools/providers (SEC-010) | `roles/iam.workloadIdentityPoolAdmin` | ✅ |
| Assign SAs to resources (Cloud Run, CFs) | `roles/iam.serviceAccountUser` | ✅ |
| Pub/Sub topics (INFRA-008c) | `roles/pubsub.admin` | ✅ |
| BigQuery dataset (INFRA-010) | `roles/bigquery.admin` | ✅ |
| Firestore Enterprise + Native (INFRA-006, INFRA-012) | `roles/datastore.owner` | ✅ |
| Cloud Storage media bucket (INFRA-019) + state bucket (INFRA-015) | `roles/storage.admin` | ✅ |

### Excessive Permission Analysis

| Role | Scope | Concern | Disposition |
|------|-------|---------|-------------|
| `roles/storage.admin` | Project | Grants bucket-level + object-level access to ALL buckets | Accepted trade-off (v8 FP #4): personal project with 2 buckets; bucket-level bindings for state bucket would create chicken-and-egg with Terraform |
| `roles/datastore.owner` | Project | Full Firestore access including data operations | Accepted trade-off (CLR-149, CLR-192): required for database creation; inherent data access is documented |
| `roles/bigquery.admin` | Project | Full BigQuery management + data access | Required for dataset creation, table management, and log sink configuration |

No excessive permission findings.

### Service Account Inventory (#1–#10)

All 10 SAs verified. No default service accounts used. Each component has a dedicated SA. No SA has `roles/owner` or `roles/editor`. Exception for Firebase Functions default SA documented and justified.

---

## 3. Cross-Spec Reference Validation

### BigQuery Tables (6 tables)

| Table | INFRA-010 | SEC-007 Retention | OBS-006 Dashboard | AC-INFRA-006 | ✅ |
|-------|-----------|-------------------|-------------------|--------------|---|
| `all_logs` | INFRA-010a | 2 years | Request latency analysis | ✅ | ✅ |
| `cloud_armor_lb_logs` | INFRA-010b | 90 days | Cloud Armor activity | ✅ | ✅ |
| `frontend_error_logs` | INFRA-010c | 2 years | Frontend error trends | ✅ | ✅ |
| `backend_error_logs` | INFRA-010d | 2 years | Backend error trends | ✅ | ✅ |
| `frontend_tracking_logs` | INFRA-010e | 2 years | Visitor analytics | ✅ | ✅ |
| `cloud_functions_error_logs` | INFRA-010f | 2 years | (used for CF error monitoring) | ✅ | ✅ |

All 6 table names consistent across specs 05, 06, and 07. ✅

### API Endpoints (13 endpoints: 12 GET + 1 POST)

| # | Endpoint | BE-API | Spec 02 Page | Spec 06 Refs | ✅ |
|---|----------|--------|-------------|--------------|---|
| 1 | `GET /` | BE-API-001 | FE-PAGE-001 | — | ✅ |
| 2 | `GET /technical` | BE-API-002 | FE-PAGE-002 | SEC-001 (search validation) | ✅ |
| 3 | `GET /technical/{slug}.md` | BE-API-003 | FE-PAGE-003 | SEC-001 (slug validation) | ✅ |
| 4 | `GET /blog` | BE-API-004 | FE-PAGE-004 | — | ✅ |
| 5 | `GET /blog/{slug}.md` | BE-API-005 | FE-PAGE-005 | — | ✅ |
| 6 | `GET /socials` | BE-API-006 | FE-PAGE-006 | — | ✅ |
| 7 | `GET /others` | BE-API-007 | FE-PAGE-007 | — | ✅ |
| 8 | `GET /categories` | BE-API-008 | FE-COMP-007 | — | ✅ |
| 9 | `POST /t` | BE-API-009 | FE-COMP-004/005 | SEC-003A (JWT auth) | ✅ |
| 10 | `GET /health` | BE-API-010 | — | INFRA-005 (rule 500 blocks public) | ✅ |
| 11 | `GET /sitemap.xml` | BE-API-011 | — | OBS-007 | ✅ |
| 12 | `GET /robots.txt` | BE-API-012 | — | OBS-007 | ✅ |
| 13 | `GET /images/{path}` | BE-API-013 | — | INFRA-019 (media bucket) | ✅ |

All consistent. ✅ Note: "No detail endpoint for `/others`" is explicitly documented in spec 03 General API Notes.

### Service Account References (10 SAs)

All 10 SAs cross-referenced between SEC-013 inventory, their defining spec sections (SEC-009/010/011/012/014, INFRA sections), and acceptance criteria. No orphaned or missing references. ✅

### DM Field Names

Key fields verified across specs:

| DM Field | Used In | Consistent | ✅ |
|----------|---------|------------|---|
| DM-002 `content_hash` | BE-API-003 ETag generation | SHA-256 of full response body → ETag `W/"<content_hash>"` | ✅ |
| DM-002 `slug` pattern | SEC-001, BE-API-003 validation | `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}\.md$` (with `.md` in URL) | ✅ |
| DM-009 `identifier` | INFRA-008c step 2, SEC-007 exception | Full (untruncated) client IP — consistent | ✅ |
| DM-009 `current_ban.tier` | SEC-002 progressive banning | `"30d"`, `"90d"`, `"indefinite"` — consistent | ✅ |
| DM-011 `model_version` | BE-API-002 cache invalidation, INFRA-014 sync | Both check model_version for cache/sync validity | ✅ |
| DM-012 `embedding_text_hash` | INFRA-014 step 3-4 | SHA-256 of `title + \n + abstract + \n + category + \n + tags` | ✅ |

### Log Sink Filters

| Sink | Filter | Non-overlapping | ✅ |
|------|--------|-----------------|---|
| INFRA-010d (backend errors) | `service_name="website-api" AND severity>=ERROR AND NOT log_type="frontend_error"` | Excludes CF errors and frontend error logs | ✅ |
| INFRA-010f (CF errors) | `service_name!="website-api" AND severity>=ERROR` | Inverse of 010d — no overlap | ✅ |
| INFRA-010c (frontend errors) | `log_type="frontend_error"` | Matches only frontend error log entries | ✅ |
| INFRA-010e (frontend tracking) | `log_type="frontend_tracking"` | Matches only frontend tracking log entries | ✅ |
| INFRA-008c Pub/Sub sink | `resource.type="http_load_balancer" AND httpRequest.status=429` | Independent of BigQuery sinks | ✅ |

AC-INFRA-021 correctly verifies non-overlap between 010d and 010f. ✅

---

## 4. Data Flow Tracing

### Flow 1: Page View Tracking (end-to-end)

```
User visits /technical
  → FE-COMP-004: Generate visitor_session_id (sessionStorage UUID v4), collect page/referrer/browser/connection_speed
  → FE-COMP-004: Generate JWT (SEC-003A: HMAC-SHA256, 5-min expiry, obfuscated secret)
  → navigator.sendBeacon(POST /t, JSON blob with action:"page_view", token, visitor_session_id)
  → Cloud Armor (60 req/min rate check) → passes through
  → Cloud Run middleware:
    1. Request ID + breadcrumb init
    2. CORS (not OPTIONS, continue)
    3. Ban check (LRU cache 60s/1000 entries → Firestore DM-009 on miss)
    4. HTTP method check (POST on /t → allowed)
    5. Route match (/t → matched)
    6. Body size check (< 100KB → proceed)
    7. Auth: validate JWT token field (signature, iss, exp, iat with 60s clock skew)
    8. Input validation: action, visitor_session_id, page, browser fields
    9. Business logic:
       - GeoIP lookup (MaxMind GeoLite2-Country embedded .mmdb) → geo_country
       - IP truncation (last octet zeroed IPv4 / last 80 bits IPv6)
       - visitor_id = SHA-256(visitor_session_id + ":" + truncated_ip + ":" + user_agent)[:32]
       - Emit structured log: severity=INFO, log_type="frontend_tracking", all fields
  → Cloud Logging ingests from stdout
  → INFRA-010e sink: matches jsonPayload.log_type="frontend_tracking" → BigQuery frontend_tracking_logs
  → Looker Studio queries frontend_tracking_logs for analytics dashboards (INFRA-011)
```

All steps verified. ✅ No broken paths.

### Flow 2: Vector Search (GET /technical?q=...)

```
User searches "cloud run deployment"
  → Frontend sends GET /technical?q=cloud+run+deployment
  → Cloud Armor → Cloud Run → middleware steps 1-5, 8
  → BE-API-002 Vector Search Flow:
    1. Normalize: lowercase("cloud run deployment")
    2. Cache key: UUID v5(URL namespace, "cloud run deployment")
    3. Check embedding_cache (DM-011) by UUID:
       - Hit + model_version matches → use cached vector
       - Hit + model_version mismatch → treat as miss (CLR-136)
       - Miss → Vertex AI gemini-embedding-001 (RETRIEVAL_QUERY, 2048 dims, 5s timeout)
              → L2-normalize → store in DM-011 with model_version
       - Vertex AI failure + no cache → return 404 (masked, SEC-004)
    4. findNearest() on Firestore Native technical_article_vectors (DM-012a)
       → cosine distance, threshold 0.35, limit 500
    5. Get candidate IDs → MongoDB query on Firestore Enterprise technical_articles (DM-002)
       → apply filters (category, tags, date_from, date_to) at database level
    6. Sort by cosine distance ascending
    7. Paginate (page × 10)
    8. Return 200 with items + pagination (or empty 200 if all filtered out)
  → Response: Cache-Control: no-store
```

Cross-spec consistency verified:
- DM-011 UUID v5 namespace `6ba7b811-9dad-11d1-80b4-00c04fd430c8` ✅
- INFRA-013 model `gemini-embedding-001`, 2048 dims, L2-normalize ✅
- INFRA-012 vector index: 2048 dimensions, COSINE ✅
- SEC-011: Cloud Run SA has `roles/datastore.viewer` (Firestore Native read) + `roles/aiplatform.user` (Vertex AI) ✅

No broken paths. ✅

### Flow 3: Progressive Banning

```
Client repeatedly exceeds 60 req/min:
  → Cloud Armor returns 429 (custom JSON error response, INFRA-005)
  → Cloud Armor LB log (httpRequest.status=429, resource.type="http_load_balancer")
  → Log sink filter matches → publishes to Pub/Sub topic "rate-limit-events"
  → Eventarc trigger → INFRA-008c process-rate-limit-logs Cloud Function (SA #3)
  → CF step 2: extract IP from httpRequest.remoteIp
  → CF step 4: findOneAndUpdate(DM-009) with $inc offense_count, $push offense_history (atomic, CLR-197)
  → CF step 5: evaluate rolling 7-day window algorithm (SEC-002):
    - No ban, 5 offenses in 7 days → 30-day ban ($set current_ban)
    - After 30-day ban expires, 2 offenses in 7 days → 90-day ban
    - After 90-day ban expires, 2 offenses in 7 days → indefinite ban
  → CF step 6: update current_ban + ban_history if threshold met

Banned client makes subsequent request:
  → Cloud Armor (passes if under rate limit)
  → Cloud Run middleware step 3: ban check
    → LRU cache lookup by IP (60s TTL, 1000 entries)
    → Cache miss → Firestore DM-009 query by identifier
    → Read current_ban → check if active (not expired)
    → 30-day ban → return 403 (with blocked_until)
    → 90-day/indefinite ban → return 404 (indistinguishable from real 404)

Lifting indefinite ban:
  → Owner deletes document from DM-009
  → LRU cache TTL expires (≤60s)
  → IP unblocked (AC-SEC-019)
```

Algorithm consistency verified across SEC-002, INFRA-008c, DM-009, AC-SEC-014, AC-SEC-019. ✅

### Flow 4: ETag Conditional Requests

```
First request: GET /technical/my-article-2025-01-15-1030.md
  → BE-API-003: fetch article from Firestore Enterprise (DM-002)
  → Response headers:
    - ETag: W/"<content_hash>" (from DM-002 pre-computed field)
    - Cache-Control: public, max-age=300
    - Last-Modified: date_updated formatted as HTTP date
  → CORS: ETag explicitly exposed via Access-Control-Expose-Headers (SEC-006) ✅
  → Browser caches response for 300 seconds

After 300s, browser revalidates:
  → GET /technical/my-article-2025-01-15-1030.md
    If-None-Match: W/"<content_hash_from_previous>"
  → CORS allows If-None-Match request header (SEC-006 allowed headers) ✅
  → BE-API-003: read only content_hash + date_updated from article (no full content fetch)
  → content_hash matches → return 304 Not Modified (empty body)

If-Modified-Since path:
  → Compare date_updated with If-Modified-Since date
  → Not more recent → return 304

Content CI/CD pipeline updates article:
  → Pipeline recomputes content_hash (SHA-256 of YAML front matter + content body)
  → Pipeline updates content + content_hash + date_updated in DM-002
  → Next conditional request: hash mismatch → full 200 response
```

All components verified: DM-002 `content_hash`, SEC-006 CORS headers, BE-API-003 conditional logic. ✅

### Flow 5: Embedding Sync

```
Content CI/CD pipeline pushes article to Firestore Enterprise
  → Pipeline calls sync-article-embeddings (INFRA-014):
    Auth: WIF via content-cicd@ SA #6 → roles/cloudfunctions.invoker + roles/run.invoker (SEC-010) ✅

sync-article-embeddings Cloud Function (SA #5):
  1. Read articles from technical_articles, blog_articles, others (Firestore Enterprise)
     → SA #5 has roles/datastore.viewer (Firestore Enterprise read) ✅
  2. For each article: embedding_text = title + "\n" + abstract + "\n" + category + "\n" + tags
  3. Compute SHA-256 of embedding_text
  4. Compare vs embedding_text_hash in Firestore Native DM-012:
     - Also compare model_version (CLR-152)
     - Hash unchanged + model matches → skip
     - Hash changed or model mismatch → re-embed
  5. Call Vertex AI gemini-embedding-001 (RETRIEVAL_DOCUMENT, 2048 dims, L2-normalize)
     → SA #5 has roles/aiplatform.user ✅
  6. Write to Firestore Native collection (DM-012a/b/c)
     → SA #5 has roles/datastore.user (Firestore Native vector-search DB) ✅
  7. Orphan cleanup: delete vectors without articles
  8. Count embedding_cache (DM-011) → log embedding_cache_size_check event
     → OBS-005 alert fires if count > 1,000 (AC-OBS-011) ✅

Daily safety net: Cloud Scheduler INFRA-014b (04:00 UTC)
  → cloud-scheduler-invoker@ SA #9 → roles/cloudfunctions.invoker + roles/run.invoker ✅
  → Idempotent — hash-based change detection ✅
```

All IAM roles, data flows, and idempotency guarantees verified. ✅

---

## 5. Algorithm Validation

### Progressive Banning Tiers (SEC-002)

| Condition | Action | Response | Spec 06 | INFRA-008c | DM-009 |
|-----------|--------|----------|---------|------------|--------|
| 1-4 offenses, no ban | Rate limited by Cloud Armor | 429 | ✅ | ✅ | — |
| 5 offenses in 7-day rolling window, no prior ban | 30-day ban | 403 | ✅ | ✅ step 5 | `current_ban.tier="30d"` ✅ |
| 2 offenses in 7 days after 30-day ban expires | 90-day ban | 404 | ✅ | ✅ step 5 | `current_ban.tier="90d"` ✅ |
| 2 offenses in 7 days after 90-day ban expires | Indefinite ban | 404 | ✅ | ✅ step 5 | `current_ban.tier="indefinite"` ✅ |

Offense counting: `findOneAndUpdate` with `$inc` + `$push` + `upsert: true` + `returnDocument: "after"` — eliminates race conditions (CLR-197). ✅

### Vector Search Pipeline (BE-API-002)

8-step flow verified against DM-011, DM-012, INFRA-012, INFRA-013, SEC-011. Distance threshold 0.35, findNearest limit 500, page size 10, cosine distance sorting. ✅

### ETag Generation

`W/"<content_hash>"` where `content_hash` = SHA-256 of full response body (YAML front matter + content). Pre-computed by CI/CD pipeline, stored in DM-002/DM-003. Backend reads only `content_hash` and `date_updated` for 304 responses — no full content fetch. ✅

### Ban Cache LRU

In-memory LRU cache: 60-second TTL, 1000 entries, keyed by client IP. Propagation delay ≤60s in both directions (new ban applied, expired ban lifted). Documented as accepted trade-off in SEC-002 and CLR-196. ✅

---

## 6. Remaining Gaps

### FINDING-001 — MEDIUM — Spec: 05

**INFRA-015 Access Control Table References Stale `roles/storage.objectAdmin` — Should Be `roles/storage.admin`**

**Gap**: The INFRA-015 Access Control table still references `roles/storage.objectAdmin` for the Terraform service account. v8 FINDING-001 changed this role to `roles/storage.admin` at Project scope in SEC-012 and SEC-013 SA #7. INFRA-015 was not updated to reflect this change.

**Evidence**:
- INFRA-015 Access Control table (spec 05 ~line 745): `Terraform service account (SEC-012) | roles/storage.objectAdmin | Read/write state files and lock files`
- SEC-012 (spec 06): `roles/storage.admin | Project | Create and manage Cloud Storage buckets (INFRA-019) and read/write Terraform state (INFRA-015)`
- SEC-013 SA #7 (spec 06): `roles/storage.admin (project)` — matches SEC-012, but INFRA-015 does not

**Why it matters**: Cross-spec inconsistency. An implementer reading INFRA-015 would see `roles/storage.objectAdmin` as the expected role for the Terraform SA, which contradicts the authoritative SEC-012 definition. The role name mismatch could cause confusion during IAM audit or Terraform configuration.

**Suggested fix**: Update INFRA-015 Access Control table from:

| Principal | Role | Purpose |
|-----------|------|---------|
| Terraform service account (SEC-012) | `roles/storage.objectAdmin` | Read/write state files and lock files |

To:

| Principal | Role | Purpose |
|-----------|------|---------|
| Terraform service account (SEC-012) | `roles/storage.admin` (Project scope, per SEC-012) | Read/write state files and lock files |

**Answer**
Follow Suggested Fix

---

### FINDING-002 — MEDIUM — Spec: 06

**SEC-014 App Deployer SA (#10) Missing `roles/iam.serviceAccountUser` — Reversal of v8 False Positive #1**

**Gap**: The app deployer SA (`app-deployer@`, SEC-014, SEC-013 SA #10) does not have `roles/iam.serviceAccountUser` (which provides `iam.serviceAccounts.actAs`). GCP requires `iam.serviceAccounts.actAs` on the target service account for **every** deployment (create or update) to Cloud Run and Cloud Functions Gen 2 that use a custom service account — not just when the SA is first assigned.

v8 dismissed this as False Positive #1 with the reasoning: "gcloud run deploy and gcloud functions deploy update code on existing services that already have SAs assigned (by Terraform). The deployer doesn't change or set the SA — it only pushes new code/images. iam.serviceAccounts.actAs is not required for code-only deployments to existing services."

This reasoning is incorrect. Per GCP documentation:
- Cloud Run: "deploying a new revision of a Cloud Run service that is associated with a service account also requires the iam.serviceAccounts.actAs permission on the associated service account" (regardless of whether the SA is being changed)
- Cloud Functions: "Any principal that deploys a function (create or **update**) must have the iam.serviceAccounts.actAs permission on the function's service account"

**Evidence**:
- SEC-014 Granted Roles (spec 06): `roles/artifactregistry.writer`, `roles/run.developer`, `roles/firebasehosting.admin`, `roles/cloudfunctions.developer` — no `roles/iam.serviceAccountUser`
- SEC-013 SA #10 (spec 06): Same 4 roles listed — no `iam.serviceAccountUser`
- AC-SEC-020 (spec 06): Lists only 4 roles — would need updating
- INFRA-020 deploy step: "Deploy to Cloud Run using gcloud run deploy" (Cloud Run uses SA #1), "Deploy the 4 Gen 2 Cloud Functions via gcloud functions deploy" (CFs use SAs #2–#5)
- `roles/run.developer` does NOT include `iam.serviceAccounts.actAs`
- `roles/cloudfunctions.developer` does NOT include `iam.serviceAccounts.actAs`

**Why it matters**: The application CI/CD pipeline (`INFRA-020`) would fail at the deploy stage:
- `gcloud run deploy` → permission denied (can't actAs `cloud-run-api@` SA #1)
- `gcloud functions deploy` for each Gen 2 CF → permission denied (can't actAs SAs #2–#5)

**Suggested fix**:
1. Add `roles/iam.serviceAccountUser` to SEC-014 Granted Roles table:

| Role | Scope | Purpose |
|------|-------|---------|
| `roles/iam.serviceAccountUser` | Project | Required to deploy Cloud Run and Cloud Functions with custom service accounts (actAs permission on target SAs #1–#5) |

2. Update SEC-013 SA #10 Key IAM Roles to include `roles/iam.serviceAccountUser`
3. Update AC-SEC-020 to include `roles/iam.serviceAccountUser` in the expected role list

**Answer**
For the `roles/iam.serviecAccountUser`, make the scope to all service accounts that will be used by all Cloud Functions and Cloud Runs.

---

### FINDING-003 — LOW — Spec: 06

**SEC-012 Security Constraints Stale Parenthetical Example `(e.g., roles/datastore.user)`**

**Gap**: After v8 FINDING-004 changed the Terraform SA's Firestore role from `roles/datastore.user` to `roles/datastore.owner`, a parenthetical example in the SEC-012 Security Constraints text was not updated.

**Evidence**:
- SEC-012 Security Constraints (spec 06): "The service account SHOULD minimize access to application data. Where infrastructure management roles **(e.g., `roles/datastore.user`)** inherently grant data-level access, this is an accepted trade-off documented in CLR-149. (CLR-192)"
- SEC-012 Granted Roles table (spec 06): `roles/datastore.owner` — the actual role after v8

**Why it matters**: Minor documentation inconsistency. The example role no longer matches the actual role in the table above it. An auditor reading the constraints would see `roles/datastore.user` as the example but find `roles/datastore.owner` in the table.

**Suggested fix**: Change `(e.g., roles/datastore.user)` to `(e.g., roles/datastore.owner)` in the SEC-012 Security Constraints text.

**Answer**
Follow Suggested Fix

---

### FINDING-004 — LOW — Spec: 03

**Allowed Status Code List Omits `413 Payload Too Large`**

**Gap**: The "Error Response Strategy" in spec 03 defines an exhaustive list of allowed HTTP status codes: `200`, `304`, `400`, `403`, `404`, `405`, `429`, `503`, `504`. However, `413 Payload Too Large` is not in this list despite being explicitly required by two other sections in the same spec:
- Middleware step 6: "For `POST /t` only: reject requests with `Content-Length` exceeding 100 KB with `413`."
- BE-API-009: "THE SYSTEM SHALL reject `POST /t` requests with `Content-Length` exceeding **100 KB** with HTTP `413 Payload Too Large`."

**Evidence**:
- Spec 03 Error Response Strategy (~line 47): Lists 9 allowed status codes — `413` absent
- Spec 03 middleware step 6 (~line 36): Explicitly specifies `413` for oversized POST /t bodies
- Spec 03 BE-API-009 (~line 614): Explicitly specifies `413` for oversized POST /t bodies

**Why it matters**: The status code list says "THE SYSTEM SHALL only return the following HTTP status codes" — the word "only" makes this a prescriptive constraint. An implementer or auditor could flag returning `413` as a violation of this constraint, despite it being required elsewhere. The list should be self-consistent.

**Suggested fix**: Add `413` to the allowed status code list:
```
- `413` — Payload Too Large (POST /t body exceeds 100 KB)
```

**Answer**
Follow Suggested Fix

---

## Summary

| ID | Priority | Spec | Category | Brief Description |
|----|----------|------|----------|-------------------|
| FINDING-001 | MEDIUM | 05 | Cross-spec reference | INFRA-015 still says `roles/storage.objectAdmin` — should be `roles/storage.admin` per SEC-012 |
| FINDING-002 | MEDIUM | 06 | IAM / CI/CD | App deployer SA #10 missing `roles/iam.serviceAccountUser` — GCP requires actAs for all Cloud Run/CF deployments (v8 FP #1 reversal) |
| FINDING-003 | LOW | 06 | Documentation | SEC-012 Security Constraints parenthetical example `(e.g., roles/datastore.user)` stale — should be `roles/datastore.owner` |
| FINDING-004 | LOW | 03 | Spec consistency | Allowed status code list omits `413` which is required by middleware step 6 and BE-API-009 |

**Total: 0 CRITICAL, 2 MEDIUM, 2 LOW.**

Both MEDIUM findings are stale references or missing IAM roles introduced or overlooked by prior rounds. No data flow, algorithm, or architectural gaps found. All 6 v8 fixes verified. Cross-spec references validated across all 7 specs.

---

## Cross-Reference Validation Summary

| Validation Area | Items Checked | Result |
|-----------------|---------------|--------|
| v8 fixes | 6 findings | All 6 correctly applied ✅ |
| SEC-012 ↔ SEC-013 SA #7 role count | 18 roles each | Match ✅ |
| SEC-012 ↔ INFRA-016 scope coverage | 19 scope items | All covered ✅ |
| BigQuery table names | 6 tables across specs 05, 06, 07 | Consistent ✅ |
| Service account inventory | 10 SAs | Consistent ✅ |
| API endpoints | 13 endpoints (12 GET + 1 POST) | Consistent ✅ |
| Cloud Function count | 5 (1 Firebase + 4 Gen 2) | Consistent ✅ |
| Cloud Scheduler jobs | 3 (within free tier) | Consistent ✅ |
| Log sink filters | 6 BigQuery + 1 Pub/Sub | Non-overlapping ✅ |
| DM field names | 12 key fields | Consistent ✅ |
| CORS config | SEC-006 vs FE/BE requirements | Consistent ✅ |
| Data flows | 5 end-to-end flows | No broken paths ✅ |
| Algorithms | 4 algorithms | Consistent ✅ |
| Stale references from v8 | `roles/storage.objectAdmin` in INFRA-015, `roles/datastore.user` in SEC-012 constraints | 2 found (FINDING-001, FINDING-003) |
| SEC-014 permission completeness | actAs requirement for deployment | Missing (FINDING-002) |
| Status code list completeness | 413 specified elsewhere but absent from list | Missing (FINDING-004) |
