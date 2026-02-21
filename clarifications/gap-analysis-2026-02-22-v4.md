# Gap Analysis v4 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07, all ~5,500 lines) after all v1 (25 findings), v2 (19 findings), and v3 (7 findings + 12 dismissed false positives) gaps were reviewed. This analysis traces end-to-end user flows, data flows (frontend → API → backend → DB → response → rendering → analytics), algorithm completeness, cross-spec identifier consistency, and edge cases.

**Methodology**:
- Traced 12 user flows end-to-end (search, tracking, error reporting, offline reading, filtering, pagination, ban enforcement, sitemap, image proxy, categories, socials, front page)
- Cross-referenced all component IDs, field names, IAM roles, HTTP status codes, cache headers, and log sink filters
- Verified middleware execution order against all possible response codes
- Validated retry/error handling for all POST /t code paths
- Checked FE-COMP-007 cache strategy against BE-API-008 `type` parameter (NEW-GAP-006 answer)

**Dismissed False Positives** (7 candidates verified as already addressed):

| Candidate | Reason Dismissed |
|-----------|-----------------|
| Cloud Functions alerting | OBS-005 includes 4 dedicated CF alerts (sitemap, embed sync, offender cleanup, plus catch-all "Cloud Function error log") |
| Last-Modified header mismatch | FE-COMP-008 uses "and/or" phrasing — If-None-Match alone is sufficient; Last-Modified is aspirational, frontend handles absent header gracefully |
| Snackbar overlapping behavior | Standard component-level UX detail; implementations universally use queue or replace patterns |
| Offline front page / list pages | Accepted scope: offline feature is article-focused per FE-COMP-008; SPA shell is cached, API-dependent pages gracefully degrade |
| `www.tjmonsi.com` DNS | Not in DNS records (INFRA-017); ultra-low priority operational concern, not a spec gap |
| SVG script execution via image proxy | Backend CSP restricts script execution; `<img>` tag loading doesn't execute SVG scripts per browser security model |
| Privacy policy required sections | FE-PAGE-009 already defines 10 detailed content sections (Data We Collect, How We Collect Data, Purpose, What We Do NOT Collect, Data Retention, Analytics, Data Storage, Your Rights, Contact, Changes to This Policy) with extensive detail including breadcrumbs, IP truncation, GeoIP, and BigQuery retention |

---

## Remaining Gaps

### FINDING-001 — MEDIUM — Specs: 02, 03

**FE-COMP-007 Category Caching Does Not Use the `type` Query Parameter**

**Gap**: BE-API-008 added an optional `type` parameter (per NEW-GAP-006 answer) so the frontend can request categories filtered by article type (`?type=technical`, `?type=blog`, `?type=others`). However, FE-COMP-007 (Category Caching) was not updated to use this parameter or to cache per-type.

**Evidence**:
- Spec 03, BE-API-008: `type` query parameter defined — "Filter categories by article type. Must be one of: `technical`, `blog`, `others`."
- Spec 02, FE-COMP-007: "THE SYSTEM SHALL fetch categories from `GET /categories` and store the result in `localStorage` with a timestamp." — No `type` parameter used. Single cache key implied.

**Why it matters**: Without per-type filtering, a user on `/technical` sees blog-only or others-only categories in the dropdown. Selecting an irrelevant category returns empty results — confusing UX that contradicts the purpose of adding the `type` parameter.

**Impact**:
1. Frontend calls `GET /categories` (unfiltered) instead of `GET /categories?type=technical`
2. Single localStorage cache entry serves all three page types with identical (unfiltered) data
3. The `type` parameter added to BE-API-008 is effectively unused

**Suggested fix**: Update FE-COMP-007 to:
- Call `GET /categories?type=<page_type>` where `<page_type>` matches the current route (`technical`, `blog`, or `others`)
- Cache per type in localStorage with separate keys (e.g., `categories_technical`, `categories_blog`, `categories_others`), each with its own 24-hour TTL
- (A) Above approach — separate cache keys per type
- (B) Fetch all categories once (current behavior), filter client-side by checking the `types` array in each category object
- (C) Other

---

### FINDING-002 — LOW — Specs: 03, 06

**Middleware Step 3 (Ban Status Check) Incorrectly Lists HTTP 429 as Possible Response**

**Gap**: The middleware execution order (spec 03, step 3) states the ban status check can "Return `429`, `403`, or `404` as appropriate." However, no Go backend ban tier maps to HTTP 429. All 429 responses originate from Cloud Armor at the infrastructure level — requests rate-limited by Cloud Armor never reach Cloud Run.

**Evidence**:
- Spec 03, middleware step 3: "Return `429`, `403`, or `404` as appropriate."
- Spec 06, SEC-002 progressive banning tiers: `30d` → 403, `90d` → 404, `indefinite` → 404
- Spec 04, DM-009 `current_ban.tier` values: `"30d"`, `"90d"`, `"indefinite"` — no 429 tier
- Spec 03, error response strategy: "429 — Too Many Requests (rate limit exceeded or initial offense ban tier)" — "initial offense ban tier" has no corresponding implementation

**Why it matters**: An implementer reading the middleware spec might try to implement a 429 return path in the ban check logic. Since no ban tier produces 429, this code path would be dead code. The inconsistency between the middleware spec and the progressive banning spec creates confusion.

**Suggested fix**:
- (A) Remove 429 from middleware step 3: "Return `403` or `404` as appropriate" (matches actual ban tiers)
- (B) Add a "warning" ban tier that returns 429 from the Go backend for IPs with 1-4 offenses (adds complexity)
- (C) Clarify that 429 in step 3 refers to Cloud Armor's response (before reaching Cloud Run) and the Go backend's ban check only returns 403/404

---

### FINDING-003 — LOW — Specs: 02

**FE-COMP-005-RETRY Incomplete HTTP Status Code Classification**

**Gap**: The retry logic classifies errors into two groups: (1) non-retryable `{400, 403, 405}` and (2) retryable `{network errors, 5xx}`. HTTP status codes outside both groups (e.g., 413 Payload Too Large, 429, 404) have undefined retry behavior.

**Evidence**:
- Spec 02, FE-COMP-005-RETRY: "THE SYSTEM SHALL NOT retry errors that are clearly non-retryable: HTTP `400` (bad request), `403` (forbidden), `405` (method not allowed). Only network errors and server errors (5xx) are retryable."
- Spec 03, BE-API-009: POST /t can return 413 (Content-Length > 100 KB)
- 413 is not in `{400, 403, 405}` (non-retryable list) AND is not 5xx (retryable)

**Why it matters**: For HTTP 413, the payload size won't decrease on retry — retrying is pointless. Similarly, 404 (masked 500 or real not-found) shouldn't be retried. The current classification leaves these in a gap where the behavior is "no retry, no modal, silent drop" — which happens to be correct, but only by accident of the control flow's fallthrough.

**Suggested fix**:
- (A) Expand the non-retryable list to `{400, 403, 404, 405, 413}` (any 4xx) — simplest: "Non-retryable: any 4xx response"
- (B) Invert the logic: "Retryable: network errors and 5xx only. All other responses are non-retryable."
- (C) Leave as-is — the implicit fallthrough behavior (no retry, no modal) is acceptable for edge cases

---

### FINDING-004 — LOW — Specs: 06

**SEC-013 SA #5 (Embedding Sync) Missing Explicit IAM Role for Firestore Enterprise**

**Gap**: The service account inventory (SEC-013, row #5) lists "Firestore Enterprise read (articles)" as prose text but does not specify the actual IAM role. All other service accounts have explicit roles (e.g., `roles/datastore.user`, `roles/datastore.viewer`).

**Evidence**:
- SEC-013, SA #5: "Key IAM Roles: Firestore Enterprise read (articles), `roles/datastore.user` (Firestore Native `vector-search` DB), `roles/aiplatform.user` (Vertex AI)"
- All other SAs have explicit role names: SA #1 has `roles/datastore.user`, SA #2-4 have `roles/datastore.user`

**Why it matters**: A Terraform implementer copying IAM bindings from SEC-013 would have an explicit role for Firestore Native (`roles/datastore.user`) and Vertex AI (`roles/aiplatform.user`) but no concrete role for Firestore Enterprise. The sync function only READS from Enterprise (articles), so `roles/datastore.viewer` would be the correct least-privilege role.

**Suggested fix**:
- (A) Replace "Firestore Enterprise read (articles)" with `roles/datastore.viewer` (Firestore Enterprise — read `technical_articles`, `blog_articles`, `others`)
- (B) Use `roles/datastore.user` for simplicity, matching other Cloud Function SAs (but grants unnecessary write access)

---

### FINDING-005 — LOW — Specs: 04, 07 — LOW — Specs: 04, 07

**Embedding Cache Size Not Monitored**

**Gap**: DM-011 notes: "If the cached query count exceeds 1,000, consider implementing a TTL-based eviction policy." However, no metric, dashboard panel, or alert exists to track the cache size growth.

**Evidence**:
- Spec 04, DM-011: "No TTL / no cache expiration. Cached embeddings persist indefinitely." + growth warning at 1,000 entries
- Spec 07, OBS-004: Metrics include `embedding_cache_hits_total` and `embedding_cache_misses_total` — but no `embedding_cache_size` or `embedding_cache_entries` metric
- Spec 07, OBS-005: Alerts include "High embedding cache miss rate" but no alert for cache size threshold
- Spec 07, OBS-006: Dashboard includes "Embedding cache performance" panel — hit/miss ratio only, not size

**Why it matters**: Without a size metric, the operator has no visibility into cache growth. The DM-011 note about considering eviction at 1,000 entries becomes a manual Firestore inspection task instead of a proactive alert. For a personal website with low traffic, this is unlikely to be an issue in practice, but the spec already identifies the threshold — it just doesn't wire up monitoring for it.

**Suggested fix**:
- (A) Add a custom metric `embedding_cache_writes_total` (counter incremented on cache miss → write) and an INFO-level alert when estimated cache size exceeds 1,000 (heuristic: `embedding_cache_writes_total` - manual invalidation events)
- (B) Add a periodic check (e.g., in the daily `sync-article-embeddings` function) that counts `embedding_cache` documents and logs the count; alert if >1,000
- (C) Leave as-is — the expected scale (~100-500 queries) makes this a non-issue for years; manual inspection is sufficient

---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| FINDING-001 | MEDIUM | Cross-Spec Consistency | FE-COMP-007 category caching ignores `type` parameter from BE-API-008 |
| FINDING-002 | LOW | Spec Inconsistency | Middleware step 3 lists 429 as ban check response; no ban tier produces 429 |
| FINDING-003 | LOW | Edge Case | FE-COMP-005-RETRY doesn't classify HTTP 413 and other uncommon 4xx codes |
| FINDING-004 | LOW | Documentation | SEC-013 SA #5 missing explicit IAM role for Firestore Enterprise read access |
| FINDING-005 | LOW | Observability | Embedding cache size growth threshold identified but not monitored |

**Total: 1 MEDIUM, 4 LOW — no CRITICAL gaps remain.**

---

## Cross-Reference Validation Summary

The following cross-spec validations were performed with no issues found:

- **Component IDs**: All FE-PAGE, FE-COMP, BE-API, DM, INFRA, SEC, OBS identifiers verified — no dangling or broken references
- **Cache-Control headers**: All 13 endpoints verified for consistency between spec 03 response format table and individual endpoint specs
- **Log sink filters**: All 6 BigQuery sinks (INFRA-010a–010f) verified for non-overlapping coverage of Cloud Run backend vs Cloud Functions errors
- **IAM roles**: All 9 service accounts (SEC-013) cross-referenced against resource access patterns — only SA #5 has the documentation gap noted in FINDING-004
- **Data field names**: All Firestore field names (DM-001–012) verified against API response schemas (BE-API-001–013) and frontend display specs
- **Acceptance criteria**: All AC-* criteria verified against parent specification sections — no orphaned or contradicted criteria found
- **Vector search flow**: 8-step pipeline (BE-API-002) verified end-to-end for technical, blog, and others collections — database-level filtering (step 5), cosine distance threshold (0.35), K=50 limit, pagination, and post-filter exhaustion all consistent
- **Progressive banning**: Offense recording (INFRA-008c) → ban application (Cloud Function) → ban enforcement (Go backend LRU cache → Firestore) → response codes (403/404) verified as a consistent pipeline — only the middleware 429 mention is inconsistent (FINDING-002)
- **Tracking data flow**: Frontend POST /t → JWT validation → GeoIP lookup → IP truncation → visitor_id hash → structured log → Cloud Logging → BigQuery sinks → Looker Studio — all field names and transformations consistent across specs 02, 03, 05, 06, 07
