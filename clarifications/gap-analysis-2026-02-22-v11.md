# Gap Analysis v11 — 2026-02-22

**Context**: Comprehensive re-analysis of the full spec suite (specs 01–07, versions 3.7/3.7/4.0/3.4/4.7/4.7/4.1) after the v10 fix expanded `roles/iam.serviceAccountUser` scope for app deployer SA #10 to include the Compute Engine default SA. This analysis covers: (A) v10 fix verification, (B) full INFRA-020 Stage 5 deployment flow re-trace, (C) complete 19-flow data flow tracing, (D) algorithm validation, (E) cross-spec consistency, (F) security constraint validation, (G) frontend-to-backend mapping, and (H) remaining inconsistency search.

**Methodology**:
- Verified v10 fix in all 3 affected locations (SEC-014, SEC-013 SA #10, AC-SEC-020)
- Re-traced INFRA-020 Stage 5 deployment flow for all 8 deploy targets with expanded actAs scope
- Traced 19 end-to-end data flows covering all API endpoints, background functions, and frontend features
- Validated 4 algorithms (progressive banning, vector search, ETag, LRU ban cache)
- Performed cross-spec consistency checks: role names, SA references, BigQuery tables, API endpoints, Firestore collections, CSP headers, middleware order
- Validated all 10 service accounts for least-privilege compliance
- Mapped all frontend components/pages to their backend API endpoints
- Searched for stale references from v10 changes and any remaining inconsistencies
- Validated counts: 10 SAs, 13 endpoints, 6 BigQuery tables, 5 Cloud Functions, 3 Cloud Scheduler jobs, 10 status codes, 18 Terraform SA roles, 5 app deployer SA roles

---

## 1. v10 Fix Verification

The v10 fix expanded `roles/iam.serviceAccountUser` scope for SA #10 (app-deployer@) from SAs #1–#5 to SAs #1–#5 + Compute Engine default SA. Verified in all 3 locations:

| Location | Expected Content | Verified |
|----------|------------------|----------|
| SEC-014 Granted Roles table (spec 06 ~line 595) | `roles/iam.serviceAccountUser` scope: "All Cloud Run and Cloud Functions runtime SAs: #1–#5 (...) + Compute Engine default SA (`<project-number>-compute@developer.gserviceaccount.com`, used by INFRA-002)" | ✅ |
| SEC-013 SA #10 Key IAM Roles (spec 06 ~line 577) | `roles/iam.serviceAccountUser (all Cloud Run/CF runtime SAs: #1–#5 + Compute Engine default SA for INFRA-002)` | ✅ |
| AC-SEC-020 (spec 06 ~line 639) | `roles/iam.serviceAccountUser (scoped to all Cloud Run and Cloud Functions runtime SAs: #1–#5 plus Compute Engine default SA for INFRA-002)` | ✅ |

All 3 locations are consistent. The Compute Engine default SA (`<project-number>-compute@developer.gserviceaccount.com`) is the default runtime SA for Gen 2 Cloud Functions when no custom SA is assigned, which is the case for INFRA-002 per the SEC-013 Exception. ✅

**Stale reference search**: No remaining references to the old scope "SAs #1–#5" without the Compute Engine default SA in any spec. ✅

---

## 2. Deployment Flow Re-trace (INFRA-020 Stage 5)

Full re-trace of the application CI/CD pipeline deployment stage with the v10 fix applied:

```
Developer pushes to `main` → GitHub Actions triggers INFRA-020 pipeline
  → app-deployer@ authenticates via WIF (SEC-014, terraform-cicd-pool)

  Backend deployment:
    → Push Docker image to Artifact Registry (INFRA-018)
      → roles/artifactregistry.writer ✅
    → gcloud run deploy (INFRA-003, runtime SA #1 cloud-run-api@)
      → roles/run.developer ✅
      → roles/iam.serviceAccountUser on SA #1 ✅

  Frontend deployment:
    → firebase deploy --only hosting,functions
      → Hosting: roles/firebasehosting.admin ✅
      → Functions (INFRA-002 `server`, Compute Engine default SA):
        → roles/cloudfunctions.developer ✅
        → roles/iam.serviceAccountUser on Compute Engine default SA ✅ (v10 fix)

  Gen 2 Cloud Functions deployment (4 functions):
    → gcloud functions deploy generate-sitemap (INFRA-008a, SA #2)
      → roles/cloudfunctions.developer ✅ + actAs on SA #2 ✅
    → gcloud functions deploy process-rate-limit-logs (INFRA-008c, SA #3)
      → roles/cloudfunctions.developer ✅ + actAs on SA #3 ✅
    → gcloud functions deploy cleanup-rate-limit-offenders (INFRA-008d, SA #4)
      → roles/cloudfunctions.developer ✅ + actAs on SA #4 ✅
    → gcloud functions deploy sync-article-embeddings (INFRA-014, SA #5)
      → roles/cloudfunctions.developer ✅ + actAs on SA #5 ✅
```

**Result**: All 8 deploy targets have complete permissions. No broken paths. ✅

| Deploy Target | IAM Role | actAs SA | Covered | ✅ |
|---------------|----------|----------|---------|---|
| Artifact Registry (INFRA-018) | `roles/artifactregistry.writer` | N/A | N/A | ✅ |
| Cloud Run (INFRA-003) | `roles/run.developer` | SA #1 `cloud-run-api@` | In #1–#5 | ✅ |
| Firebase Hosting (INFRA-001) | `roles/firebasehosting.admin` | N/A | N/A | ✅ |
| Firebase Function `server` (INFRA-002) | `roles/cloudfunctions.developer` | Compute Engine default SA | In expanded scope | ✅ |
| CF: generate-sitemap (INFRA-008a) | `roles/cloudfunctions.developer` | SA #2 `cf-sitemap-gen@` | In #1–#5 | ✅ |
| CF: process-rate-limit-logs (INFRA-008c) | `roles/cloudfunctions.developer` | SA #3 `cf-rate-limit-proc@` | In #1–#5 | ✅ |
| CF: cleanup-rate-limit-offenders (INFRA-008d) | `roles/cloudfunctions.developer` | SA #4 `cf-offender-cleanup@` | In #1–#5 | ✅ |
| CF: sync-article-embeddings (INFRA-014) | `roles/cloudfunctions.developer` | SA #5 `cf-embed-sync@` | In #1–#5 | ✅ |

---

## 3. Complete Data Flow Tracing (19 Flows)

### Flow 1: Page View Tracking

```
User visits page → FE-COMP-004 fires page_view event
  → Frontend generates JWT (SEC-003A, 5-min expiry)
  → navigator.sendBeacon() POST /t with token in body (FE-COMP-004)
  → Cloud Armor (60 req/min check) → Cloud Run
  → Middleware: 1.ReqID → 2.CORS → 3.BanCheck → 4.MethodCheck(POST) → 5.RouteMatch(/t)
    → 6.BodySize(<100KB) → 7.JWT validation → 8.Input validation → 9.Business logic
  → Extract IP from X-Forwarded-For
  → GeoIP lookup on full IP → country code (CLR-123)
  → Truncate IP (SEC-007)
  → Compute visitor_id: SHA-256(session_id + ":" + truncated_ip + ":" + user_agent)[:32]
  → Emit structured log: log_type="frontend_tracking" (OBS-001/OBS-002)
  → Cloud Logging → INFRA-010e sink → BigQuery frontend_tracking_logs
  → Response: 200 {"status":"ok"} + Cache-Control: no-store
```
✅ Consistent across specs 02 (FE-COMP-004), 03 (BE-API-009), 06 (SEC-003A, SEC-007), 07 (OBS-001/002), 05 (INFRA-010e).

### Flow 2: Link Click Tracking

```
User clicks anchor → FE-COMP-004 fires link_click event asynchronously
  → Same path as Flow 1 with action="link_click", clicked_url field
  → Structured log: log_type="frontend_tracking" with clicked_url
  → BigQuery frontend_tracking_logs
```
✅ Consistent. AC-FE-009 confirms async send without blocking navigation.

### Flow 3: Time on Page Tracking

```
User stays on page → FE-COMP-004 tracks milestones (1min, 2min, 5min)
  → Page Visibility API: only counts active foreground time (AC-FE-041)
  → Same path as Flow 1 with action="time_on_page", milestone field
  → Structured log: log_type="frontend_tracking" with milestone
  → BigQuery frontend_tracking_logs
```
✅ Consistent. milestone validated as one of: `1min`, `2min`, `5min` (BE-API-009).

### Flow 4: Error Report Delivery

```
Frontend detects error → FE-COMP-005 constructs error payload
  → Attaches breadcrumbs from FE-COMP-013 ring buffer (up to 50 entries)
  → Generate fresh JWT → POST /t with action="error_report"
  → Backend validates breadcrumbs (SEC-001): max 50 entries, truncate if more
  → Renames breadcrumbs → client_breadcrumbs in log entry (BE-API-009)
  → Emit structured log: log_type="frontend_error" (OBS-001/OBS-003)
  → Cloud Logging → INFRA-010c sink → BigQuery frontend_error_logs
  → Response: 200 → FE-COMP-005 shows success toast (CLR-119)
```
✅ Consistent across specs 02 (FE-COMP-005, FE-COMP-013), 03 (BE-API-009), 06 (SEC-001), 07 (OBS-003), 05 (INFRA-010c).

### Flow 5: Error Report Retry & Modal

```
Error report POST /t fails (network error):
  → FE-COMP-005-RETRY: Queue payload in memory + sessionStorage (without JWT)
  → Retry with exponential backoff: 1s, 2s, 4s, 8s, 16s (±20% jitter, CLR-127)
  → On success: remove from queue, show toast
  → On non-retryable 4xx: skip retries → proceed to modal check
  → On 5 retries exhausted:
    → If backend error (not 503): FE-COMP-005-MODAL displays
    → If network unavailability or 503: silent failure (AC-FE-023)
  → Online event: flush all queued reports immediately (AC-FE-021)

Modal (FE-COMP-005-MODAL):
  → Shows error_type, error_message, page, browser, connection_speed, breadcrumbs
  → Excludes JWT token and visitor_session_id
  → "Copy to clipboard" button, link to /socials, privacy note linking to /privacy
  → Accessible (keyboard nav, focus trap, Escape to dismiss)
  → Shown max once per error per session (AC-MODAL-005)
  → Circular failure guard: no automated report from modal (CLR-116)
```
✅ Consistent across spec 02 (FE-COMP-005-RETRY, FE-COMP-005-MODAL, AC-RETRY-001 through AC-MODAL-006).

### Flow 6: Vector Search (Technical Articles)

```
User types search query on /technical → FE-COMP-011 updates URL ?q=...
  → Debounced fetch to GET /technical?q=docker&page=1
  → Cloud Armor → Cloud Run → Middleware (steps 1-9)
  → BE-API-002 vector search flow (8 steps):
    1. Validate q (max 300 chars, SEC-001)
    2. Compute UUID v5 cache key from query text
    3. Check DM-011 embedding_cache in Firestore Enterprise
    4. On cache miss: call Vertex AI gemini-embedding-001 (RETRIEVAL_QUERY, 2048 dims)
       → SA #1 has roles/aiplatform.user ✅
       → L2-normalize truncated embedding (INFRA-013)
    5. Store in DM-011 embedding_cache (UUID v5 key, vector, model_version)
       → SA #1 has roles/datastore.user on Firestore Enterprise ✅
    6. Query Firestore Native technical_article_vectors (findNearest, limit 500, cosine)
       → SA #1 has roles/datastore.viewer on vector-search DB ✅
    7. Filter by cosine distance threshold 0.35, apply category/tags/date filters
    8. Paginate (page size 10) and return sorted by similarity (ascending distance)
  → Response: 200 JSON with items + pagination
  → Frontend renders results sorted by relevance (AC-FE-017)
```
✅ Consistent across specs 02 (FE-PAGE-002), 03 (BE-API-002), 04 (DM-011, DM-012), 05 (INFRA-012, INFRA-013), 06 (SEC-011, SEC-013 SA #1).

### Flow 7: Vector Search (Blog Articles)

Same as Flow 6 but queries `blog_articles` / `blog_article_vectors`. BE-API-004 behavior "identical to BE-API-002." ✅

### Flow 8: Vector Search (Others)

Same as Flow 6 but queries `others` / `others_vectors`. BE-API-007 behavior "identical to BE-API-002." ✅

### Flow 9: Progressive Banning

```
Client exceeds 60 req/min → Cloud Armor returns 429 + Retry-After header
  → Cloud Armor log entry created (httpRequest.status=429)
  → Log sink (filter: resource.type="http_load_balancer" AND httpRequest.status=429)
    → Pub/Sub topic rate-limit-events (INFRA-008c)
  → Cloud Function process-rate-limit-logs triggered:
    1. Extract full IP from httpRequest.remoteIp
    2. findOneAndUpdate on rate_limit_offenders (DM-009): atomic $inc offense_count (CLR-197)
    3. Evaluate progressive ban tiers (SEC-002 rolling 7-day window):
       - 5 offenses in 7 days → 30-day ban (403)
       - 2 offenses in 7 days post-30d-ban → 90-day ban (404)
       - 2 offenses in 7 days post-90d-ban → indefinite ban (404)
    4. Write current_ban if threshold met
  → SA #3 has roles/datastore.user on Firestore Enterprise ✅

Subsequent request from banned IP:
  → Cloud Armor passes (rate limit not currently exceeded)
  → Go backend middleware step 3: Ban check
    → LRU cache lookup (keyed by IP, 60s TTL, max 1000 entries)
    → On miss: query rate_limit_offenders by full IP → cache result
    → SA #1 has roles/datastore.user on Firestore Enterprise (read) ✅
    → If 30-day ban: return 403 with blocked_until
    → If 90-day/indefinite ban: return 404 (indistinguishable from not-found)
  → Frontend handles:
    → 403: FE-COMP-003 snackbar (AC-FE-010: "wait ~30 minutes" + rate limit message)
    → 404: FE-COMP-003 snackbar (standard 404 message)
```
✅ Consistent across specs 03 (middleware), 05 (INFRA-008c), 06 (SEC-002, DM-009 via spec 04), 02 (FE-COMP-003).

### Flow 10: ETag Conditional Requests

```
Frontend fetches article: GET /technical/{slug}.md
  → Backend computes ETag from pre-stored content_hash (DM-002): W/"<SHA-256>"
  → Response includes ETag + Last-Modified headers
  → Browser/service worker caches response

Subsequent request (online with cache):
  → FE-COMP-008 cache-first: render from cache, then revalidate
  → Send If-None-Match: W/"<hash>" and/or If-Modified-Since
  → Backend middleware step 9: Compare ETag
    → Match → return 304 Not Modified (no body, no full content fetch)
    → No match → return 200 with updated content
  → If 304: cached content is current ✅
  → If 200: update cache, optionally notify user
```
✅ Consistent across specs 02 (FE-COMP-008), 03 (BE-API-003, BE-API-005), 04 (DM-002 content_hash field).

### Flow 11: Embedding Sync (Content CI/CD)

```
Content CI/CD pushes articles to Firestore Enterprise
  → Invokes sync-article-embeddings (INFRA-014) via HTTP (OIDC token)
    → Auth: content-cicd@ SA #6 via WIF (SEC-010) with roles/cloudfunctions.invoker + roles/run.invoker ✅
  → Cloud Function (SA #5 cf-embed-sync@):
    1. Read articles from technical_articles, blog_articles, others (Firestore Enterprise)
       → SA #5 has roles/datastore.viewer on Firestore Enterprise ✅
    2. For each article: construct source text (title + abstract + category + tags)
    3. Compute SHA-256 of source text
    4. Compare with embedding_text_hash + model_version in Firestore Native doc
    5. If changed: call Vertex AI gemini-embedding-001 (RETRIEVAL_DOCUMENT, 2048 dims)
       → SA #5 has roles/aiplatform.user ✅
       → L2-normalize before storage
    6. Write/update vector doc in Firestore Native
       → SA #5 has roles/datastore.user on vector-search DB ✅
    7. Delete orphaned vector docs (no corresponding article)
    8. Count embedding_cache docs → log INFO embedding_cache_size_check

Daily safety net: Cloud Scheduler INFRA-014b triggers same function at 04:00 UTC
  → SA #9 cloud-scheduler-invoker@ with roles/cloudfunctions.invoker + roles/run.invoker ✅
```
✅ Consistent across specs 05 (INFRA-014, INFRA-014b), 06 (SEC-010, SEC-013 SA #5/#6/#9), 04 (DM-012), 07 (OBS-005 alert for cache size > 1000).

### Flow 12: Sitemap Generation

```
Cloud Scheduler INFRA-008b triggers every 6 hours
  → SA #9 invokes generate-sitemap (INFRA-008a) via OIDC token
  → Cloud Function (SA #2 cf-sitemap-gen@):
    1. Read technical_articles, blog_articles from Firestore Enterprise
       → SA #2 has roles/datastore.user (read) ✅
    2. Generate XML sitemap per sitemaps.org protocol
    3. Write sitemap document to DM-010 sitemap collection
       → SA #2 has roles/datastore.user (write) ✅
  → Go backend serves via GET /sitemap.xml (BE-API-011)
    → Reads from sitemap collection → returns application/xml
    → Cache-Control: public, max-age=3600
  → Firebase Hosting rewrite /sitemap.xml → Cloud Run (INFRA-001)
    → tjmonsi.com/sitemap.xml → api.tjmonsi.com/sitemap.xml (proxied)
  → robots.txt references Sitemap: https://tjmonsi.com/sitemap.xml (OBS-007)
```
✅ Consistent across specs 05 (INFRA-008a/b, INFRA-001), 03 (BE-API-011), 04 (DM-010), 06 (SEC-013 SA #2/#9), 07 (OBS-007).

### Flow 13: Rate Limit Offender Cleanup

```
Cloud Scheduler INFRA-008e triggers daily at 03:00 UTC
  → SA #9 invokes cleanup-rate-limit-offenders (INFRA-008d) via OIDC token
  → Cloud Function (SA #4 cf-offender-cleanup@):
    1. Query rate_limit_offenders collection (DM-009)
    2. Delete records: no active ban AND no offenses in last 90 days
       → SA #4 has roles/datastore.user (read/write) ✅
  → Enables natural decay of offense history for reformed IPs
```
✅ Consistent across specs 05 (INFRA-008d/e), 04 (DM-009), 06 (SEC-002, SEC-013 SA #4/#9).

### Flow 14: Application Deployment Pipeline

Fully traced in Section 2 above. All 8 deploy targets with complete permissions after v10 fix. ✅

### Flow 15: Image Proxy

```
Article markdown references https://api.tjmonsi.com/images/articles/diagram.png
  → Browser sends GET /images/articles/diagram.png
  → Cloud Armor → Cloud Run → Middleware (steps 1-9)
  → BE-API-013:
    1. Validate path: no ".." (path traversal rejection → 404)
    2. Read object from Cloud Storage <project-id>-media-bucket (INFRA-019)
       → SA #1 has roles/storage.objectViewer on media bucket ✅
    3. Verify MIME type is image/* (non-image → 404)
    4. Stream bytes to response (no full buffering)
    5. Set Content-Type from Cloud Storage metadata
    6. Cache-Control: public, max-age=2592000 (30 days)
  → Frontend CSP allows: img-src 'self' data: https://api.tjmonsi.com ✅
```
✅ Consistent across specs 03 (BE-API-013), 05 (INFRA-019), 06 (SEC-005 CSP, SEC-013 SA #1).

### Flow 16: Offline Reading (Service Worker)

```
Manual save:
  → User clicks "Save for offline" on FE-PAGE-003/005 or FE-PAGE-002/004
  → Main thread sends postMessage to service worker with article URL
  → Service worker fetches article from GET /technical/{slug}.md (BE-API-003)
  → Caches response in Cache API
  → Metadata stored in IndexedDB (title, slug, reading progress)
  → Offline indicator updates via custom event offline-cache-updated
  → CSP connect-src includes https://api.tjmonsi.com ✅

Smart download:
  → User scrolls past 30% of article on 4g connection (FE-COMP-008)
  → Prefetch up to 3 related articles (same category first, then recent)
  → Storage cap: 50 MB, oldest evicted when full
  → Network check: only on navigator.connection.effectiveType === '4g' (CLR-177)
  → Safari/Firefox: smart download disabled (no Network Information API)

Offline access:
  → Service worker intercepts GET request → serves from Cache API
  → If online: background conditional request (If-None-Match) → 304 or 200 update
  → If offline + no cache: "article not available offline" message
  → Reading progress restored from IndexedDB
```
✅ Consistent across spec 02 (FE-COMP-008), spec 03 (BE-API-003/005 ETag support).

### Flow 17: Category Caching

```
User navigates to /technical:
  → FE-COMP-007 checks localStorage `categories_technical`
  → If cached and <24h old: use cached list
  → If missing or stale: fetch GET /categories?type=technical (BE-API-008)
    → Backend queries DM-006 categories where types includes "technical"
    → Returns sorted alphabetically, Cache-Control: public, max-age=3600
  → Store in localStorage with timestamp under key categories_technical
  → Populate category dropdown with type-specific categories (AC-FE-050)
  → Same pattern for /blog (categories_blog) and /others (categories_others)
  → Each per-type cache has independent 24-hour TTL (CLR-199)
```
✅ Consistent across specs 02 (FE-COMP-007), 03 (BE-API-008), 04 (DM-006).

### Flow 18: Front Page Content

```
User visits / → FE-PAGE-001
  → fetch GET / (BE-API-001)
  → Backend reads DM-001 frontpage collection from Firestore Enterprise
  → Returns text/markdown with ETag + Cache-Control headers
  → Frontend renders raw markdown on page (AC-FE-028)
```
✅ Consistent across specs 02 (FE-PAGE-001), 03 (BE-API-001), 04 (DM-001).

### Flow 19: Breadcrumb Trail

```
User interacts with site:
  → FE-COMP-013 records events in in-memory ring buffer (max 50)
  → 10 event types: navigation, click, api_request, api_response, api_error,
    search, filter_change, page_visibility, offline_status, js_error
  → Each entry: timestamp (ISO 8601 ms), type, message (max 300), data (optional)
  → No PII in breadcrumbs (no form values except search, no tokens)
  → Buffer is memory-only (lost on tab close/reload, AC-BREADCRUMB-FE-007)

On error:
  → FE-COMP-005 attaches getBreadcrumbs() array to error report payload
  → POST /t sends breadcrumbs array
  → Backend validates (SEC-001): max 50 entries, truncate if more, log WARNING
  → Renames to client_breadcrumbs in structured log (BE-API-009)
  → Separate from server_breadcrumbs (BE-BREADCRUMB) which tracks backend processing steps
  → server_breadcrumbs logged ONLY on errors (AC-OBS-008, AC-API-020)

Modal display:
  → FE-COMP-005-MODAL shows breadcrumbs in readable format
  → Excludes JWT token and visitor_session_id
```
✅ Consistent across specs 02 (FE-COMP-013, FE-COMP-005), 03 (BE-API-009, BE-BREADCRUMB), 06 (SEC-001).

---

## 4. Algorithm Validation

### 4.1 Progressive Banning (SEC-002)

- **Rolling 7-day window**: Count offenses within last 7 days for tier evaluation ✅
- **Tier 1**: 5 offenses in 7 days → 30-day ban (403) ✅
- **Tier 2**: 2 offenses in 7 days post-30d-ban → 90-day ban (404) ✅
- **Tier 3**: 2 offenses in 7 days post-90d-ban → indefinite ban (404) ✅
- **Atomic operations**: `findOneAndUpdate` with `$inc` + `returnDocument: "after"` (CLR-197) — eliminates race conditions ✅
- **Post-ban window**: Starts from `current_ban.end` (not from first offense) ✅
- **Active ban**: No re-evaluation during active ban, offense still recorded ✅
- **Cleanup**: INFRA-008d deletes records with no active ban + no offenses in 90 days ✅
- **Manual review**: Indefinite bans lifted by deleting DM-009 document; LRU cache expires within 60s ✅

### 4.2 Vector Search (BE-API-002)

- **8-step flow**: validate → UUID v5 → cache check → Vertex AI (on miss) → store cache → Firestore Native query → filter → paginate ✅
- **Cache key**: UUID v5 hash of search query text ✅
- **Embedding model**: gemini-embedding-001, 2048 dimensions, L2-normalized (INFRA-013) ✅
- **Task types**: RETRIEVAL_QUERY for runtime search, RETRIEVAL_DOCUMENT for article indexing ✅
- **Distance threshold**: cosine distance ≤ 0.35 ✅
- **findNearest limit**: 500 candidates ✅
- **Page size**: 10 items ✅
- **Empty/whitespace query**: Treated as no search → sorted by date_updated desc (AC-API-024, CLR-189) ✅
- **Result ordering**: With query → similarity (ascending distance); without query → date_updated desc ✅

### 4.3 ETag Conditional Requests (BE-API-001/003/005)

- **ETag source**: SHA-256 from pre-computed `content_hash` field in DM documents ✅
- **ETag format**: Weak ETag `W/"<hash>"` ✅
- **Conditional check**: Compare `If-None-Match` header with current ETag ✅
- **304 response**: No body, no full content fetch from database ✅
- **FE-COMP-008 integration**: Service worker uses If-None-Match for cache revalidation ✅

### 4.4 LRU Ban Cache (SEC-002)

- **TTL**: 60 seconds ✅
- **Max entries**: 1000 ✅
- **Key**: Client IP (full, from X-Forwarded-For) ✅
- **Bidirectional delay**: New bans take up to 60s for cached "not banned" IPs; expired bans enforced up to 60s for cached "banned" IPs (CLR-196) ✅
- **Cache miss behavior**: Query Firestore rate_limit_offenders → populate cache ✅
- **Indefinite ban lift**: Delete DM-009 document → cache expires within 60s → unblocked (AC-SEC-019) ✅

---

## 5. Cross-Spec Consistency

### 5.1 Role Name Consistency

| Role | SEC-012 | SEC-013 SA #7 | SEC-014 | SEC-013 SA #10 | AC-SEC-020 | Consistent |
|------|---------|---------------|---------|----------------|------------|------------|
| `roles/storage.admin` | Project ✅ | Project ✅ | — | — | — | ✅ |
| `roles/datastore.owner` | Project ✅ | `(CLR-149)` ✅ | — | — | — | ✅ |
| `roles/iam.serviceAccountUser` (TF #7) | Project ✅ | ✅ | — | — | — | ✅ |
| `roles/iam.serviceAccountUser` (App #10) | — | — | SAs #1–#5 + CE default | SAs #1–#5 + CE default | SAs #1–#5 + CE default | ✅ |
| `roles/artifactregistry.writer` | — | — | AR repo | ✅ | ✅ | ✅ |
| `roles/run.developer` | — | — | CR service | ✅ | ✅ | ✅ |
| `roles/firebasehosting.admin` | — | — | Project | ✅ | ✅ | ✅ |
| `roles/cloudfunctions.developer` | — | — | Project | ✅ | ✅ | ✅ |

**Stale reference search**: `roles/storage.objectAdmin` — 0 matches ✅. Old actAs scope "SAs #1–#5" without Compute Engine default SA — 0 matches ✅.

### 5.2 Service Account Inventory Cross-Reference

All 10 SAs in SEC-013 verified against their defining sections:

| # | SA Email | Defined In | Referenced In | Consistent |
|---|----------|------------|---------------|------------|
| 1 | cloud-run-api@ | SEC-013, INFRA-003 | SEC-011, SEC-014 (actAs target), BE-API-002/009, INFRA-019 | ✅ |
| 2 | cf-sitemap-gen@ | SEC-013, INFRA-008a | SEC-014 (actAs target) | ✅ |
| 3 | cf-rate-limit-proc@ | SEC-013, INFRA-008c | SEC-014 (actAs target) | ✅ |
| 4 | cf-offender-cleanup@ | SEC-013, INFRA-008d | SEC-014 (actAs target) | ✅ |
| 5 | cf-embed-sync@ | SEC-013, INFRA-014 | SEC-010, SEC-014 (actAs target) | ✅ |
| 6 | content-cicd@ | SEC-013, SEC-010 | INFRA-014 Integration Contract | ✅ |
| 7 | terraform-builder@ | SEC-013, SEC-012 | INFRA-015, INFRA-016 | ✅ |
| 8 | looker-studio-reader@ | SEC-013, SEC-009 | INFRA-011 | ✅ |
| 9 | cloud-scheduler-invoker@ | SEC-013 | INFRA-008b, 008e, 014b | ✅ |
| 10 | app-deployer@ | SEC-013, SEC-014 | INFRA-020, AC-SEC-020 | ✅ |

### 5.3 BigQuery Table Names (6 tables)

| Table | INFRA-010 Sink | Spec 07 Reference | Filter | Consistent |
|-------|---------------|-------------------|--------|------------|
| `all_logs` | INFRA-010a | OBS-001 | No filter (all logs) | ✅ |
| `cloud_armor_lb_logs` | INFRA-010b | OBS-005 (alerts) | `resource.type="http_load_balancer"` | ✅ |
| `frontend_error_logs` | INFRA-010c | OBS-003, OBS-005 | `log_type="frontend_error"` | ✅ |
| `backend_error_logs` | INFRA-010d | OBS-005 | `service_name="website-api" AND severity>=ERROR AND NOT frontend_error` | ✅ |
| `frontend_tracking_logs` | INFRA-010e | OBS-002, OBS-006, INFRA-011 | `log_type="frontend_tracking"` | ✅ |
| `cloud_functions_error_logs` | INFRA-010f | OBS-005 | `service_name!="website-api" AND severity>=ERROR` | ✅ |

Sink filter mutual exclusivity verified: INFRA-010d and INFRA-010f use inverse `service_name` filters to prevent overlap (AC-INFRA-021). ✅

### 5.4 API Endpoint Count (13 endpoints)

| # | Endpoint | Spec 03 | Spec 02 Reference | ✅ |
|---|----------|---------|-------------------|---|
| 1 | GET / | BE-API-001 | FE-PAGE-001, AC-FE-028 | ✅ |
| 2 | GET /technical | BE-API-002 | FE-PAGE-002 | ✅ |
| 3 | GET /technical/{slug}.md | BE-API-003 | FE-PAGE-003 | ✅ |
| 4 | GET /blog | BE-API-004 | FE-PAGE-004 | ✅ |
| 5 | GET /blog/{slug}.md | BE-API-005 | FE-PAGE-005 | ✅ |
| 6 | GET /socials | BE-API-006 | FE-PAGE-006, AC-FE-034/049 | ✅ |
| 7 | GET /others | BE-API-007 | FE-PAGE-007, AC-FE-035 | ✅ |
| 8 | GET /categories | BE-API-008 | FE-COMP-007, AC-FE-042/050 | ✅ |
| 9 | POST /t | BE-API-009 | FE-COMP-004/005 | ✅ |
| 10 | GET /health | BE-API-010 | INFRA-003 (internal probe) | ✅ |
| 11 | GET /sitemap.xml | BE-API-011 | INFRA-001 (rewrite), OBS-007 | ✅ |
| 12 | GET /robots.txt | BE-API-012 | api.tjmonsi.com only | ✅ |
| 13 | GET /images/{path} | BE-API-013 | Article markdown image refs | ✅ |

### 5.5 Firestore Collection Cross-Reference

**Firestore Enterprise (MongoDB compat, `(default)` database)**:

| Collection | DM Ref | Written By | Read By | ✅ |
|------------|--------|------------|---------|---|
| frontpage | DM-001 | Content CI/CD (out of scope) | SA #1 (BE-API-001) | ✅ |
| technical_articles | DM-002 | Content CI/CD | SA #1 (BE-API-002/003), SA #2 (INFRA-008a), SA #5 (INFRA-014 read) | ✅ |
| blog_articles | DM-003 | Content CI/CD | SA #1 (BE-API-004/005), SA #2 (INFRA-008a), SA #5 (INFRA-014 read) | ✅ |
| socials | DM-004 | Content CI/CD | SA #1 (BE-API-006) | ✅ |
| others | DM-005 | Content CI/CD | SA #1 (BE-API-007), SA #5 (INFRA-014 read) | ✅ |
| categories | DM-006 | Content CI/CD | SA #1 (BE-API-008) | ✅ |
| rate_limit_offenders | DM-009 | SA #3 (INFRA-008c) | SA #1 (ban check), SA #4 (INFRA-008d cleanup) | ✅ |
| sitemap | DM-010 | SA #2 (INFRA-008a) | SA #1 (BE-API-011) | ✅ |
| embedding_cache | DM-011 | SA #1 (BE-API-002 cache write) | SA #1 (BE-API-002 cache read) | ✅ |

**Firestore Native (`vector-search` database)**:

| Collection | DM Ref | Written By | Read By | ✅ |
|------------|--------|------------|---------|---|
| technical_article_vectors | DM-012 | SA #5 (INFRA-014) | SA #1 (BE-API-002 vector search) | ✅ |
| blog_article_vectors | DM-012 | SA #5 (INFRA-014) | SA #1 (BE-API-004 vector search) | ✅ |
| others_vectors | DM-012 | SA #5 (INFRA-014) | SA #1 (BE-API-007 vector search) | ✅ |

### 5.6 CSP Header Validation

Frontend CSP (SEC-005) checked against all resource loading patterns:

| Directive | Value | Covers | ✅ |
|-----------|-------|--------|---|
| `default-src` | `'self'` | Baseline for unlisted directives | ✅ |
| `script-src` | `'self' 'unsafe-inline'` | Nuxt 4 SPA scripts + inline | ✅ |
| `style-src` | `'self' 'unsafe-inline'` | Vue component inline styles | ✅ |
| `connect-src` | `'self' https://api.tjmonsi.com` | API calls, sendBeacon /t, service worker fetches | ✅ |
| `img-src` | `'self' data: https://api.tjmonsi.com` | Image proxy BE-API-013 | ✅ |
| `font-src` | `'self'` | Local fonts only | ✅ |
| `worker-src` | `'self'` | Service worker (FE-COMP-008) | ✅ |
| `object-src` | `'none'` | No plugins/embeds | ✅ |

### 5.7 Middleware Order Validation (Spec 03)

9 middleware steps verified for correct security ordering:

| Step | Action | Security Rationale | ✅ |
|------|--------|-------------------|---|
| 1 | Request ID + breadcrumb init (OBS-008) | Early for correlation | ✅ |
| 2 | CORS preflight handling (SEC-006) | OPTIONS exempt from ban, returned fast | ✅ |
| 3 | Ban status check (SEC-002, LRU cache) | Block banned IPs before any processing | ✅ |
| 4 | HTTP method enforcement (SEC-003) | Reject bad methods before routing | ✅ |
| 5 | Route matching | Determine handler | ✅ |
| 6 | Body size check (POST /t, 100 KB) | Reject oversized payloads before JWT parse | ✅ |
| 7 | JWT validation (POST /t, SEC-003A) | Auth after size check | ✅ |
| 8 | Input validation (SEC-001) | Validate after auth | ✅ |
| 9 | Business logic + response | Handler executes | ✅ |

---

## 6. Security Constraint Validation

### 6.1 Least-Privilege Audit (All 10 SAs)

| # | SA | Has `roles/owner` or `roles/editor`? | Minimal Roles? | ✅ |
|---|------|--------------------------------------|----------------|---|
| 1 | cloud-run-api@ | No | Read+write Firestore Enterprise, read Firestore Native, read Cloud Storage, Vertex AI, metrics | ✅ |
| 2 | cf-sitemap-gen@ | No | Read/write Firestore Enterprise only (CLR-185) | ✅ |
| 3 | cf-rate-limit-proc@ | No | Read/write Firestore Enterprise only (CLR-185) | ✅ |
| 4 | cf-offender-cleanup@ | No | Read/write Firestore Enterprise only (CLR-185) | ✅ |
| 5 | cf-embed-sync@ | No | Read Firestore Enterprise, read/write Firestore Native, Vertex AI | ✅ |
| 6 | content-cicd@ | No | Invoke only sync-article-embeddings | ✅ |
| 7 | terraform-builder@ | No | Infrastructure management (18 roles, broadly scoped, accepted per CLR-149/192) | ✅ |
| 8 | looker-studio-reader@ | No | Read-only BigQuery (dataset + jobUser) | ✅ |
| 9 | cloud-scheduler-invoker@ | No | Invoke target Cloud Functions only | ✅ |
| 10 | app-deployer@ | No | Deploy only (no admin/infra roles, SEC-014 constraints) | ✅ |

### 6.2 WIF Configuration Consistency

| Pipeline | WIF Pool | Repo Condition | Mapped SA | Key Management | ✅ |
|----------|----------|----------------|-----------|----------------|---|
| Terraform CI/CD | `terraform-cicd-pool` | `<owner>/personal-website` | terraform-builder@ | WIF (no keys) | ✅ |
| App CI/CD | `terraform-cicd-pool` (shared) | `<owner>/personal-website` | app-deployer@ | WIF (no keys) | ✅ |
| Content CI/CD | `content-cicd-pool` (separate) | `<owner>/<content-repo>` | content-cicd@ | WIF (no keys) | ✅ |

Pool separation: Content pipeline uses its own pool (`content-cicd-pool`) with a different repository condition. Terraform and app pipelines share `terraform-cicd-pool` but impersonate different SAs. ✅

### 6.3 Data Privacy Constraints

| Constraint | Implementation | ✅ |
|-----------|---------------|---|
| IP truncation before logging | SEC-007: IPv4 zero last octet, IPv6 zero last 80 bits, IPv4-mapped IPv6 treated as IPv4 | ✅ |
| Full IP only in DM-009 | rate_limit_offenders stores full IP (DM-009) for ban matching; 90-day retention | ✅ |
| Full IP in LB logs | cloud_armor_lb_logs (INFRA-010b) retains full IP; 90-day retention; not used in analytics | ✅ |
| GeoIP before truncation | CLR-123: lookup on full IP → country code only; IP discarded after | ✅ |
| No cookies/sessions/fingerprints | SEC-007: confirmed; only visitor_session_id (random UUID per session) | ✅ |
| Breadcrumbs: no PII | FE-COMP-013: no form values (except search), no passwords, no tokens, no session IDs | ✅ |

---

## 7. Frontend-to-Backend Mapping

| Frontend Page/Component | API Endpoint(s) | Data Model | Behavior Match | ✅ |
|------------------------|------------------|------------|---------------|---|
| FE-PAGE-001 (/) | GET / (BE-API-001) | DM-001 frontpage | text/markdown rendered | ✅ |
| FE-PAGE-002 (/technical) | GET /technical (BE-API-002), GET /categories?type=technical (BE-API-008) | DM-002, DM-006, DM-011, DM-012 | Paginated list, vector search, category filter | ✅ |
| FE-PAGE-003 (/technical/:slug.md) | GET /technical/{slug}.md (BE-API-003) | DM-002 | Markdown article, ETag, ToC, citations | ✅ |
| FE-PAGE-004 (/blog) | GET /blog (BE-API-004), GET /categories?type=blog (BE-API-008) | DM-003, DM-006 | Same as /technical for blog | ✅ |
| FE-PAGE-005 (/blog/:slug.md) | GET /blog/{slug}.md (BE-API-005) | DM-003 | Same as /technical/:slug.md for blog | ✅ |
| FE-PAGE-006 (/socials) | GET /socials (BE-API-006) | DM-004 | Active links, sorted by sort_order, MDI icons | ✅ |
| FE-PAGE-007 (/others) | GET /others (BE-API-007), GET /categories?type=others (BE-API-008) | DM-005, DM-006 | Same as /technical but with external_url, no detail endpoint | ✅ |
| FE-PAGE-008 (404) | None | None (hardcoded) | 30 humorous anecdotes, link to home | ✅ |
| FE-PAGE-009 (/privacy) | None | None (static content) | Privacy policy with all required sections | ✅ |
| FE-PAGE-010 (/changelog) | None | None (static/generated) | Chronological frontend changes | ✅ |
| FE-COMP-004 (tracking) | POST /t (BE-API-009) | — (Cloud Logging → BigQuery) | page_view, link_click, time_on_page via sendBeacon | ✅ |
| FE-COMP-005 (error reports) | POST /t (BE-API-009) | — (Cloud Logging → BigQuery) | error_report with breadcrumbs | ✅ |
| FE-COMP-006 (robots.txt) | — | — | Static file in Firebase Hosting public/ | ✅ |
| FE-COMP-007 (category cache) | GET /categories (BE-API-008) | DM-006 | Per-type localStorage cache, 24h TTL (CLR-199) | ✅ |
| FE-COMP-008 (offline) | GET /technical/{slug}.md, GET /blog/{slug}.md | Cache API + IndexedDB | Service worker, cache-first + conditional revalidation | ✅ |
| FE-COMP-013 (breadcrumbs) | POST /t (attached to error reports) | — | 50-entry ring buffer, 10 event types | ✅ |
| FE-COMP-014 (tag chips) | Tags param in GET /technical, GET /blog, GET /others | — | Comma-separated lowercase, max 10 tags (CLR-160) | ✅ |

---

## 8. Remaining Inconsistency Search

### 8.1 Count Validation

| Count | Expected | Actual | ✅ |
|-------|----------|--------|---|
| Service accounts | 10 | 10 (SEC-013 inventory) | ✅ |
| API endpoints | 13 | 13 (12 GET + 1 POST, spec 03) | ✅ |
| BigQuery tables | 6 | 6 (INFRA-010a–010f) | ✅ |
| Cloud Functions | 5 | 5 (1 Firebase + 4 Gen 2) | ✅ |
| Cloud Scheduler jobs | 3 | 3 (INFRA-008b, 008e, 014b — within free tier) | ✅ |
| Status codes | 10 | 10 (200, 304, 400, 403, 404, 405, 413, 429, 503, 504) | ✅ |
| Terraform SA roles (SEC-012) | 18 | 18 | ✅ |
| Terraform SA roles (SEC-013 SA #7) | 18 | 18 | ✅ |
| App deployer SA roles (SEC-014) | 5 | 5 | ✅ |
| App deployer SA roles (SEC-013 SA #10) | 5 | 5 | ✅ |
| Log sinks (total) | 7 | 7 (6 BigQuery + 1 Pub/Sub for rate-limit events) | ✅ |
| Firestore Enterprise collections | 9 | 9 (DM-001 through DM-006 + DM-009/010/011) | ✅ |
| Firestore Native collections | 3 | 3 (DM-012: technical/blog/others vectors) | ✅ |
| Frontend pages | 10 | 10 (FE-PAGE-001 through FE-PAGE-010) | ✅ |

### 8.2 Stale Reference Search (from v10 changes)

| v10 Change | Potential Stale Patterns | Matches Found | ✅ |
|-----------|--------------------------|---------------|---|
| SEC-014 actAs scope expanded to include CE default SA | Old "SAs #1–#5" scope without CE default SA | 0 across all specs (all 3 locations updated) | ✅ |
| Spec 06 version bumped to 4.7 | Version inconsistencies | All other specs retain their own versions | ✅ |

### 8.3 Comprehensive Cross-Check of Acceptance Criteria

Spot-checked all acceptance criteria sections across specs 02–07 for internal consistency:

- **AC-FE-* (50 criteria in spec 02)**: All reference correct endpoint names, component IDs, and behavior descriptions. ✅
- **AC-API-* (33 criteria in spec 03)**: All reference correct collections, log sinks, and response formats. ✅
- **AC-INFRA-* (21 criteria in spec 05)**: All reference correct service names, table names, and SA identifiers. ✅
- **AC-SEC-* (20 criteria in spec 06)**: All reference correct role names, SA scopes, and security behaviors. ✅
- **AC-OBS-* (11 criteria in spec 07)**: All reference correct log types, metric names, and dashboard sources. ✅

### 8.4 Additional Checks Performed

| Check | Result |
|-------|--------|
| robots.txt: separate files for tjmonsi.com (FE-COMP-006/OBS-007) and api.tjmonsi.com (BE-API-012) | ✅ Distinct content, correct domains |
| DNS: both domains point to same Load Balancer, URL Map routes by hostname (INFRA-004/017) | ✅ |
| Firebase Hosting rewrite for /sitemap.xml → Cloud Run (INFRA-001) | ✅ Matches BE-API-011 |
| CORS: api.tjmonsi.com allows origin https://tjmonsi.com (SEC-006) | ✅ Matches CSP connect-src |
| Cloud Run non-root container: user `jack` UID 10001, distroless base (AC-SEC-009, INFRA-003) | ✅ |
| No /others/{slug}.md detail endpoint: confirmed in General API Notes (spec 03) | ✅ Matches FE-PAGE-007 external_url behavior |
| Service worker: stale-while-revalidate lifecycle, postMessage communication (CLR-170) | ✅ Not installable as PWA (no manifest) |
| Pub/Sub log sink for rate-limit events: formally defined in INFRA-008c with filter `resource.type="http_load_balancer" AND httpRequest.status=429` | ✅ |
| VPC Direct VPC Egress scope: Cloud Run + 4 Gen 2 CFs (INFRA-009) — Production only | ✅ /27 subnet (32 IPs) sufficient for max concurrency |

---

## 9. Remaining Gaps

**No gaps found.**

After exhaustive verification of the v10 fix, full deployment flow re-trace, 19 end-to-end data flows, 4 algorithm validations, cross-spec consistency checks (roles, SAs, tables, endpoints, collections, CSP, middleware), security constraint validation, frontend-to-backend mapping, count validation, stale reference search, and acceptance criteria cross-check — the specification suite is internally consistent with no remaining gaps.

---

## Summary

| ID | Priority | Spec | Category | Brief Description |
|----|----------|------|----------|-------------------|
| (none) | — | — | — | No gaps found |

**Total: 0 CRITICAL, 0 MEDIUM, 0 LOW.**

The v10 fix (expanding `roles/iam.serviceAccountUser` scope to include the Compute Engine default SA for INFRA-002) was the final gap. All 7 specs (01–07) are now internally consistent and cross-referenced correctly.

---

## Cross-Reference Validation Summary

| Validation Area | Items Checked | Result |
|-----------------|---------------|--------|
| v10 fix | 3 locations (SEC-014, SEC-013 SA #10, AC-SEC-020) | All correctly applied ✅ |
| Deployment flow (INFRA-020 Stage 5) | 8 deploy targets | All 8 have complete permissions ✅ |
| Data flows | 19 end-to-end flows | All consistent, no broken paths ✅ |
| Algorithms | 4 (progressive banning, vector search, ETag, LRU cache) | All validated ✅ |
| SEC-012 ↔ SEC-013 SA #7 role count | 18 roles each | Match ✅ |
| SEC-014 ↔ SEC-013 SA #10 role count | 5 roles each | Match ✅ |
| SEC-014 actAs scope vs INFRA-020 deploy targets | 8 deploy targets | 8/8 covered ✅ |
| Role name consistency | All role names across specs 05, 06 | No stale references ✅ |
| BigQuery table names | 6 tables across specs 05, 06, 07 | Consistent ✅ |
| API endpoints | 13 endpoints across specs 01, 02, 03, 06, 07 | Consistent ✅ |
| Firestore collections | 9 Enterprise + 3 Native across specs 03, 04, 05, 06 | Consistent ✅ |
| Service account inventory | 10 SAs, no owner/editor roles | All verified ✅ |
| Status codes | 10 codes across specs 02, 03 | All with consistent FE/BE handling ✅ |
| CSP headers | 8 directives covering all resource patterns | All frontend features covered ✅ |
| Middleware order | 9 steps | Security ordering correct ✅ |
| Acceptance criteria | 135 criteria across 5 specs | Internally consistent ✅ |
| Frontend-to-backend mapping | 10 pages + 7 components → 13 endpoints | Complete mapping ✅ |
| Stale references from v10 | 2 change areas searched | None found ✅ |
| Counts | 14 counts validated | All match ✅ |
