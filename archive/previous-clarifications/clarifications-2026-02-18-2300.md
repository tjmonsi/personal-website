# Clarifications — 2026-02-18-2300

Pre-development readiness review. 7 Critical, 10 Important, 11 Minor findings across all 7 spec files and cost estimate. Grouped by priority.

---

## Critical (blocks development)

---

### CLR-148 — Firestore Enterprise Authentication Mechanism

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 03, 05, 06  
**Affected IDs**: INFRA-003, INFRA-006, SEC-013 (SA #1)

**Problem**: The MongoDB wire protocol authentication method for Cloud Run → Firestore Enterprise is unresolved. SEC-013 lists the Cloud Run SA with "Firestore Enterprise access (MongoDB wire protocol, via project-level role or custom role)" but the actual IAM role is undefined. INFRA-006 says "Backend-only access" but doesn't specify whether authentication uses IAM-based auth (GCP service account) or MongoDB username/password credentials.

This determines the Cloud Run service's environment variable configuration, IAM binding, and connection string format — blocking infrastructure provisioning.

**Option A (Recommended)**: Research this during implementation and document findings as the first infrastructure task. Add a note in INFRA-006 and SEC-013 that the auth mechanism is "to be determined during infrastructure bootstrap — the implementer SHALL verify MongoDB wire protocol authentication and document the approach before proceeding with application development."

**Option B**: Pre-research now and specify the exact IAM role and auth mechanism in the specs before any development begins.

**Answer**
The IAM is roles/datastore.user and the connection using an mongodb ORM library will be found here: https://docs.cloud.google.com/firestore/mongodb-compatibility/docs/connect#cloud-run specifically the Cloud Run part where we just add the connection string.

For Development, we will be using https://docs.cloud.google.com/firestore/mongodb-compatibility/docs/connect#connect_with_a_temporary_access_token the temporary access token

---

### CLR-149 — Terraform Service Account Roles Undefined

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 06  
**Affected IDs**: SEC-012

**Problem**: The Terraform service account roles are listed as "To be determined during implementation." Terraform cannot manage infrastructure without correct IAM permissions. The INFRA-016 scope includes Cloud Run, Cloud Functions, Cloud DNS, Artifact Registry, Cloud Logging, Cloud Monitoring, IAM, Pub/Sub, BigQuery, and potentially Firestore Enterprise.

**Option A (Recommended)**: Define the minimum required roles based on INFRA-016 scope. The recommended set is: `roles/run.admin`, `roles/cloudfunctions.admin`, `roles/compute.networkAdmin`, `roles/dns.admin`, `roles/artifactregistry.admin`, `roles/logging.admin`, `roles/monitoring.admin`, `roles/iam.serviceAccountAdmin`, `roles/pubsub.admin`, `roles/bigquery.admin`, plus Firestore roles TBD (see CLR-148).

**Option B**: Leave as "TBD during implementation" and derive incrementally by running `terraform plan` and adding roles as permission errors occur.

**Answer**
Use Option A

---

### CLR-150 — Firestore Enterprise Terraform Provider Support

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 05  
**Affected IDs**: INFRA-016

**Problem**: INFRA-016 notes "Terraform provider support for [Firestore Enterprise MongoDB compat mode] should be verified during implementation." If the `google_firestore_database` resource doesn't support MongoDB compat mode, Firestore Enterprise becomes a manual bootstrap resource, requiring the Terraform scope to be adjusted.

**Option A (Recommended)**: Accept that Firestore Enterprise provisioning may require manual bootstrap (via `gcloud` or console) and is excluded from Terraform scope if the provider doesn't support it. Add a note: "INFRA-016 Terraform scope excludes Firestore Enterprise provisioning if the Google Terraform provider does not support MongoDB compat mode. In that case, Firestore Enterprise SHALL be provisioned manually and documented as a bootstrap prerequisite."

**Option B**: Research and verify now before development starts.

**Answer**
Use Option A

---

### CLR-151 — GeoIP Database Update Process Missing

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 05, 01  
**Affected IDs**: INFRA-003

**Problem**: The MaxMind GeoLite2-Country `.mmdb` file is `COPY`'d into the Docker image at build time, but GeoLite2 databases are updated weekly. No CI/CD step, scheduled process, or download mechanism is defined. Additionally, GeoLite2 requires a free license key to download.

**Option A (Recommended)**: Add a CI/CD step that downloads the latest GeoLite2-Country database before the Docker build, using a MaxMind license key stored in Secret Manager (or GitHub Actions secrets). Add a weekly scheduled workflow to rebuild the Docker image with the latest database. Specify in INFRA-003.

**Option B**: Treat the GeoIP database as a build-time artifact updated only when the backend is redeployed. Accept that the database may be stale between deployments.

**Answer**
Use Option A

---

### CLR-152 — DM-012 Vector Collections Missing `model_version` Field

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 04, 05  
**Affected IDs**: DM-012, INFRA-014, AC-DM-011-B

**Problem**: The `sync-article-embeddings` function uses `embedding_text_hash` to detect content changes and skip re-embedding. However, DM-012 vector documents have **no `model_version` field**. If the Gemini embedding model is upgraded, content hashes remain unchanged, so the sync function would skip all articles — leaving stale embeddings. AC-DM-011-B conflates the embedding cache (DM-011) with the vector collections (DM-012), saying "update `model_version` in all cache entries" when it should reference vector documents.

**Option A (Recommended)**: Add a `model_version` field to DM-012 vector document schema. The sync function compares this against the currently configured model. If mismatched, force re-embedding regardless of hash. Rewrite AC-DM-011-B to reference DM-012 vector collections (not DM-011 cache entries). DM-011 cache already self-heals via the `model_version` check in BE-API-002.

**Option B**: Skip `model_version` in DM-012. On model upgrade, manually trigger a full re-sync by clearing all `embedding_text_hash` values (or deleting all vector documents). Accept manual intervention as sufficient for the rare model-upgrade scenario.

**Answer**
Use Option A

---

### CLR-153 — `sendBeacon` vs `fetch` Conflict for Error Reports

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-004, FE-COMP-005-RETRY

**Problem**: FE-COMP-004 states "THE SYSTEM SHALL use `navigator.sendBeacon()` as the primary delivery method for **all** `POST /t` requests." However, FE-COMP-005-RETRY requires retry logic with HTTP status code checking (e.g., distinguishing 503 from other errors), which is impossible with `sendBeacon` — it's fire-and-forget and returns only a boolean. Error reports need `fetch()` to detect failures for retry decisions.

**Option A (Recommended)**: Clarify that `sendBeacon` is used only for fire-and-forget tracking events (`page_view`, `link_click`, `time_on_page`), while error reports (`error_report`) use `fetch()` to enable the retry logic defined in FE-COMP-005-RETRY. Update FE-COMP-004 to say "all tracking `POST /t` requests" instead of "all `POST /t` requests."

**Option B**: Use `fetch()` for all `POST /t` requests (both tracking and error reports), with `keepalive: true` for page-unload scenarios. Drop `sendBeacon` entirely.

**Answer**
Use Option A

---

### CLR-154 — JWT Expiry on Offline Queue Flush

**Priority**: 🔴 CRITICAL  
**Affected Specs**: 02, 06  
**Affected IDs**: FE-COMP-005-RETRY, SEC-003A

**Problem**: Error reports queued while offline include a JWT `token` with 5-minute expiry. When the device comes back online and flushes all queued reports (AC-RETRY-005), any reports queued more than 5 minutes ago will have expired JWTs, causing the backend to return `403` — a non-retryable error per AC-RETRY-004. These error reports would be permanently lost.

**Option A (Recommended)**: Store queued error reports **without** the JWT token. Generate a fresh JWT at send time when flushing the queue. This ensures all retried reports have valid tokens.

**Option B**: Extend the JWT expiry to a longer duration (e.g., 1 hour) to accommodate offline periods. Accept that very long offline periods would still lose reports.

**Answer**
Use Option A

---

## Important (needs answer before implementing that component)

---

### CLR-155 — Date Range Filter Boundary Behavior

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 03  
**Affected IDs**: BE-API-002, BE-API-004, BE-API-007

**Problem**: The `date_from` and `date_to` parameters accept ISO 8601 "date" format (e.g., `2025-01-31`). It's unclear whether boundaries are inclusive or exclusive. If `date_to=2025-01-31`, does it include articles with `date_updated` at `2025-01-31T23:59:59Z`?

**Option A (Recommended)**: Both boundaries are inclusive. `date_from` means `>= start of day UTC` and `date_to` means `< start of next day UTC` (i.e., the entire day is included).

**Option B**: `date_from` inclusive, `date_to` exclusive (standard half-open interval).

**Answer**
Use Option A

---

### CLR-156 — `visitor_id` Hash Concatenation Separator

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 03  
**Affected IDs**: BE-API-009

**Problem**: `visitor_id` is "a SHA-256 hash of the concatenation of: `visitor_session_id` + truncated IP address + raw `User-Agent` header." No separator is specified. Without a separator, collisions are possible (e.g., session `"abc"` + IP `"def"` == session `"ab"` + IP `"cdef"`).

**Option A (Recommended)**: Use `:` as a delimiter: `SHA-256(visitor_session_id + ":" + truncated_ip + ":" + user_agent)`.

**Option B**: Use `|` as a delimiter.

**Answer**
Use Option A

---

### CLR-157 — DM-009 `identifier` Field Says "IP or Fingerprint"

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 04  
**Affected IDs**: DM-009

**Problem**: The `identifier` field is described as "Client identifier (IP or fingerprint)" but SEC-007 explicitly prohibits fingerprinting, and INFRA-008c only extracts the client IP from Cloud Armor logs. The "or fingerprint" option contradicts the security spec.

**Option A (Recommended)**: Change description to "Client IP address" and remove the "or fingerprint" reference.

**Option B**: Leave as-is for future extensibility.

**Answer**
Use Option A

---

### CLR-158 — Smart Download Heuristic Parameters

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-008

**Problem**: "Smart Download (Heuristic Prefetching)" is specified at a high level but lacks concrete parameters: what constitutes "partially viewed" (scroll percentage? time threshold?), how many articles to prefetch, which categories count as "same category as recently read," and storage budget limits.

**Option A (Recommended)**: Define: "partially viewed" = scrolled past 30% of article height, prefetch up to 3 articles from the same category, cap prefetch cache at 50 MB, only prefetch on Wi-Fi or unmetered connections.

**Option B**: Remove smart download from the initial release and implement as a future enhancement. Keep only the explicit "Save for offline" action.

**Answer**
Use Option A

---

### CLR-159 — Markdown Rendering Library and Sanitization

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 02  
**Affected IDs**: FE-PAGE-001, FE-PAGE-003, FE-PAGE-005

**Problem**: The frontend renders markdown from API responses but no markdown rendering library is specified, and no XSS sanitization strategy is defined. Even though content is owner-authored, defense-in-depth requires sanitization.

**Option A (Recommended)**: Specify `markdown-it` (or equivalent) as the rendering library with `DOMPurify` sanitization of rendered HTML output. Leave exact library choice to the implementer, but require HTML sanitization as a constraint.

**Option B**: Leave as an implementation detail. Trust that the content is owner-authored and skip sanitization.

**Answer**
Use Option A but let's decide the library that is useful for our requirements and can be used in Vue3

---

### CLR-160 — Maximum Tag Count for FE-COMP-014

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-014

**Problem**: The tag chip input has no maximum tag count. A user could add dozens of tags, creating very long URLs (some browsers limit to ~2000 chars) and complex backend queries.

**Option A (Recommended)**: Maximum of 10 tags. When the limit is reached, disable the input and display a tooltip "Maximum 10 tags."

**Option B**: Maximum of 5 tags.

**Answer**
Use Option A. Also add a limit in backend for error handling if somehow more than 10 tags are sent (e.g., via direct API call).

---

### CLR-161 — Health Check Database Scope

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 03  
**Affected IDs**: BE-API-010

**Problem**: The health endpoint "SHALL check database connectivity" but doesn't specify which database(s) — Firestore Enterprise only, Firestore Native only, or both? What query? What timeout?

**Option A (Recommended)**: Check Firestore Enterprise only (primary data store) with a lightweight read (e.g., list collections or read a known document) with a 5-second timeout. Firestore Native is for search only — its unavailability degrades search but doesn't break core functionality.

**Option B**: Check both databases. Report partial health if only one is available.

**Answer**
Use Option A

---

### CLR-162 — ETag Algorithm

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 03  
**Affected IDs**: BE-API-003, BE-API-005

**Problem**: ETag generation says "Hash of the response body (e.g., MD5 or SHA-256)" without specifying which. Need a concrete decision for deterministic behavior.

**Option A (Recommended)**: Use SHA-256 (consistent with other hashing in the system) with weak ETag format: `W/"<sha256-hex>"`.

**Option B**: Use CRC32 for faster computation. ETags don't require cryptographic strength.

**Answer**
Use Option A

---

### CLR-163 — Progressive Banning Race Conditions

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 05, 04  
**Affected IDs**: INFRA-008c, DM-009

**Problem**: Multiple `process-rate-limit-logs` Cloud Function instances could process concurrent Pub/Sub events for the same IP, leading to race conditions on `offense_count` increment and ban evaluation.

**Option A (Recommended)**: Specify that the Cloud Function uses MongoDB atomic `$inc` on the `offense_count` field and evaluates the ban threshold against the incremented result (atomic read-after-write). No explicit transaction needed.

**Option B**: Use Firestore Enterprise transactions to read-increment-evaluate atomically.

**Answer**
Use Option A

---

### CLR-164 — Cache-Control Headers for List Endpoints

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 03  
**Affected IDs**: BE-API-001, BE-API-002, BE-API-004, BE-API-006, BE-API-007, BE-API-008

**Problem**: Only article detail endpoints (BE-API-003, BE-API-005) and static endpoints (sitemap, robots.txt, images) have defined `Cache-Control` headers. List endpoints and the frontpage have none specified, meaning browsers apply default/heuristic caching.

**Option A (Recommended)**: Define: `GET /` → `Cache-Control: public, max-age=300` (5 min); list endpoints (technical, blog, others) → `Cache-Control: no-cache` (always revalidate); `GET /categories` → `Cache-Control: public, max-age=3600` (1 hour, frontend caches 24h anyway); `GET /socials` → `Cache-Control: public, max-age=3600`.

**Option B**: Leave as implementation detail. The frontend manages its own caching (sessionStorage for categories) so backend Cache-Control is secondary.

**Answer**
Use Option A

---

## Minor (suggestions, non-blocking)

---

### CLR-165 — `FE-COMP-009` "Same or Similar Query" Ambiguity

**Priority**: 🟢 MINOR  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-009

**Problem**: Repeated empty search detection triggers on "same **or similar** query." The definition of "similar" is undefined.

**Option A (Recommended)**: Simplify to "same query" (exact match after lowercase + trim normalization). Remove "or similar."

**Option B**: Leave as-is, let the implementer decide.

**Answer**
Use exact match and levenshtein distance algorithm with a threshold of 2 for "similar" queries. This allows for minor typos while preventing false positives from unrelated queries. 

---

### CLR-166 — Service Worker Update Auto-Dismiss Behavior

**Priority**: 🟢 MINOR  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-008

**Problem**: The spec says auto-dismiss after 20 seconds, and "IF the user dismisses the snackbar..." It's unclear if auto-dismiss is treated identically to manual dismiss.

**Option A (Recommended)**: Add: "Auto-dismiss and manual dismiss have identical behavior — in both cases, the new service worker is activated on the next full page load or navigation."

**Option B**: Leave as-is, the behavior is implicitly the same.

**Answer**
Use Option A

---

### CLR-167 — `connection_speed` Limited Browser Support

**Priority**: 🟢 MINOR  
**Affected Specs**: 02, 07  
**Affected IDs**: FE-COMP-004, OBS-003

**Problem**: `Navigator.connection` is unavailable in Safari/iOS. The spec says `connection_speed` is "optional" but doesn't specify how analytics should handle the high percentage of null values.

**Option A (Recommended)**: Add a note: "Browser support for `Navigator.connection` is limited (unavailable in Safari/iOS as of 2026). Looker Studio dashboards SHALL display connection speed data as 'available where supported' and include a metric showing the percentage of events with connection speed data."

**Option B**: Leave as-is, null handling is an analytics concern.

**Answer**
Use Option A

---

### CLR-168 — `GET /socials` Response Schema Missing Fields

**Priority**: 🟢 MINOR  
**Affected Specs**: 03, 04  
**Affected IDs**: BE-API-006, DM-004

**Problem**: DM-004 includes `is_active`, `date_created`, and `date_updated` fields not shown in the BE-API-006 response example. Intentional omission or oversight?

**Option A (Recommended)**: Add a note to BE-API-006 that `is_active`, `date_created`, and `date_updated` are internal fields filtered/omitted from the API response.

**Option B**: Add `date_created` and `date_updated` to the API response for completeness.

**Answer**
Use Option A

---

### CLR-169 — Offline Availability Indicator Visual Format

**Priority**: 🟢 MINOR  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-008, FE-PAGE-002, FE-PAGE-004

**Problem**: List views should display an "offline availability indicator" but the visual design is unspecified (icon? badge? text?).

**Option A (Recommended)**: Use a small download icon (↓) next to unsaved articles and a checkmark icon (✓) next to saved articles. Keep it minimal.

**Option B**: Leave as an implementation/design detail.

**Answer**
Use Option A

---

### CLR-170 — Service Worker Lifecycle Details

**Priority**: 🟢 MINOR  
**Affected Specs**: 02  
**Affected IDs**: FE-COMP-008

**Problem**: The service worker update notification is well-defined, but registration timing, scope, cache versioning, and `waiting` state handling are not specified.

**Option A (Recommended)**: Leave as implementation detail. Nuxt/Nitro generates the service worker with sensible defaults. Add a brief note: "Service worker registration, scope, and caching strategy are managed by the Nuxt framework's built-in PWA/service worker support."

**Option B**: Specify full lifecycle: register on page load, scope `/`, cache names include a version hash, skip-waiting on user confirmation (the "Refresh now" button).

**Answer**
Use Option A
---

### CLR-171 — Frontpage ETag Support

**Priority**: 🟢 MINOR  
**Affected Specs**: 03  
**Affected IDs**: BE-API-001

**Problem**: The frontpage (`GET /`) has no conditional request support (no ETag, no 304), unlike article detail endpoints. The frontend can't efficiently check if the front page has changed.

**Option A (Recommended)**: Add ETag and `Last-Modified` (based on `date_updated` from DM-001) support to `GET /`. Consistent with article detail endpoints.

**Option B**: Skip — the front page is small, redownloading is cheap.

**Answer**
Use Option A

---

### CLR-172 — Embedding Sync Partial Failure Handling

**Priority**: 🟢 MINOR  
**Affected Specs**: 05  
**Affected IDs**: INFRA-014

**Problem**: The sync function doesn't specify behavior when individual Vertex AI embedding calls fail. Skip and continue? Fail the entire sync? Retry?

**Option A (Recommended)**: Continue processing remaining articles on individual failures. Log each failure with article ID. Report summary at the end (X succeeded, Y failed). Failed articles retry on next scheduled run.

**Option B**: Fail the entire sync on the first error. Retry from scratch on next run.

**Answer**
Use Option A

---

### CLR-173 — Embedding Cache Unbounded Growth

**Priority**: 🟢 MINOR  
**Affected Specs**: 04  
**Affected IDs**: DM-011

**Problem**: The embedding cache has no TTL and no storage limit. Over years, this could grow to millions of documents (16KB+ each for 2048-dimension vectors). Manual purge exists but no automated management.

**Option A (Recommended)**: Accept as a known trade-off. Add a note: "Cache growth is expected to be slow (one entry per unique search query). The manual purge procedure (DM-011) is sufficient for a personal website. Monitor collection size during operation and implement LRU eviction if growth exceeds expectations."

**Option B**: Add an automated cleanup — delete entries with `created_at` older than 6 months via a scheduled Cloud Function.

**Answer**
Use Option A

---

### CLR-174 — INFRA-005 `/health` Block Redundancy Note

**Priority**: 🟢 MINOR  
**Affected Specs**: 05  
**Affected IDs**: INFRA-005 (Rule 500)

**Problem**: Cloud Armor rule 500 blocks public access to `/health`. Cloud Run health checks bypass the load balancer, so this rule is defense-in-depth. Not documented as such.

**Option A (Recommended)**: Add a note: "This rule is defense-in-depth. Cloud Run startup and liveness probes bypass the external load balancer, but this rule prevents accidental public exposure if ingress settings are misconfigured."

**Option B**: Leave as-is — the intent is obvious.

**Answer**
Use Option A

---

### CLR-175 — Sitemap Excludes `others` Without Explanation

**Priority**: 🟢 MINOR  
**Affected Specs**: 05  
**Affected IDs**: INFRA-008a

**Problem**: Sitemap generation queries `technical_articles` and `blog_articles` but not `others`. Correct (others link to external URLs) but not explained.

**Option A (Recommended)**: Add a note to INFRA-008a: "The `others` collection is excluded because items link to external URLs and have no internal detail pages."

**Option B**: Leave as-is.

**Answer**
Use Option A