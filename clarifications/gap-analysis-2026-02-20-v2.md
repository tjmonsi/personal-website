# Gap Analysis v2 — New Findings

**Date**: 2026-02-20  
**Scope**: All 7 specification files (01–07)  
**Baseline**: Gaps already identified in `gap-analysis-2026-02-20.md` (GAP-001 through GAP-025) are **excluded** from this analysis.

---

## Summary

| Priority | Count |
|----------|-------|
| CRITICAL | 2     |
| IMPORTANT | 10   |
| SUGGESTION | 7   |
| **Total** | **19** |

---

## Findings

### CRITICAL

---

**NEW-GAP-001 | CRITICAL | Specs: 02, 06 | Frontend CSP blocks Google Fonts CDN**

**Issue**: The Visual Design section (02-frontend-specifications.md, line 327) specifies:
> *Font: Google Sans (served from Google Fonts CDN).*

However, the frontend Content-Security-Policy in SEC-005 (06-security-specifications.md) defines:
- `font-src 'self'`
- `style-src 'self' 'unsafe-inline'`

Loading Google Sans from Google Fonts CDN requires:
- `font-src 'self' https://fonts.gstatic.com` (for the font files)
- `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com` (for the CSS stylesheet)

With the current CSP, the browser will block the font download, and Google Sans will not load.

**Recommendation**: Either:
- **(A)** Update the CSP in SEC-005 to allow Google Fonts CDN domains (`fonts.googleapis.com` for styles, `fonts.gstatic.com` for font files), OR
- **(B)** Self-host the Google Sans font files and serve them from `tjmonsi.com` (compatible with the current `font-src 'self'`), and update the Visual Design section to say "self-hosted" instead of "served from Google Fonts CDN."

Option B provides better privacy (no third-party requests) and performance (no external DNS lookup), aligning with the site's privacy-conscious design.

**Answer**: Implement Option B — self-host Google Sans font files and update the spec accordingly.

---

**NEW-GAP-002 | CRITICAL | Specs: 04, 06 | SEC-007 contradicts DM-009 on full IP storage in Firestore**

**Issue**: SEC-007 (06-security-specifications.md) states:
> *THE SYSTEM SHALL NOT store full IP addresses in application-level logs, Firestore, or any application-managed data store.*

However, DM-009 `rate_limit_offenders` (04-data-model-specifications.md, line 206) defines:
> *`identifier`: Full (untruncated) client IP address (CLR-157, CLR-186)*

This is a direct contradiction. The exception in SEC-007 explicitly covers only Cloud Armor load balancer logs (`cloud_armor_lb_logs` in BigQuery), **not** the Firestore `rate_limit_offenders` collection.

The full IP is functionally required for ban enforcement (Cloud Armor bans operate on exact IPs, and the Go backend LRU cache checks ban status by full IP). But the exception is not documented in SEC-007.

**Recommendation**: Add an explicit exception in SEC-007 for the `rate_limit_offenders` collection, similar to the existing Cloud Armor exception. Example:

> *Exception — Rate Limit Offenders: The `rate_limit_offenders` collection (DM-009) stores full IP addresses because Cloud Armor bans are enforced on exact IPs and the Go backend must verify ban status by matching the request's source IP. The `process-rate-limit-logs` Cloud Function writes full IPs sourced from Cloud Armor load balancer logs. These records are retained for a maximum of 90 days (no active ban) and are only accessible to the Go backend and Cloud Functions service accounts.*

**Answer**: Add the above exception to SEC-007 to resolve the contradiction and clarify the rationale for storing full IPs in Firestore for ban enforcement.

---

### IMPORTANT

---

**NEW-GAP-003 | IMPORTANT | Specs: 03, 05 | No failure fallback for Vertex AI embedding API during search**

**Issue**: The vector search flow (BE-API-002, step 3 — cache miss path) calls the Vertex AI Gemini embedding API. No error handling is specified for:
- Vertex AI API returning an error (rate limit, quota exceeded, 5xx)
- Vertex AI API timeout
- Network failure between Cloud Run and Vertex AI

If the Vertex AI call fails and the embedding is not cached, the entire search query fails with no graceful degradation path described.

**Recommendation**: Specify error handling for Vertex AI failures:
- IF the Vertex AI API call fails AND no cached embedding exists, THE SYSTEM SHALL return the standard `404` (masked 500) response and log the error with full context (including server breadcrumbs).
- Consider specifying a timeout for the Vertex AI API call (e.g., 5 seconds) as a sub-timeout of the 30-second Cloud Run request timeout.

**Answer**
Add the above error handling specification to the vector search flow in BE-API-002 to ensure graceful degradation when the embedding API is unavailable.

---

**NEW-GAP-004 | IMPORTANT | Specs: 03 | No failure behavior for Firestore Native (vector search DB) unavailability**

**Issue**: BE-API-010 health check explicitly excludes Firestore Native and notes: "its unavailability only degrades search functionality, not core content delivery." However, the spec does not define what response the user sees when Firestore Native is unavailable during a search query:
- Does `GET /technical?q=cloud` return a `200` with empty results?
- Does it return `404` (masked 500)?
- Is there any degradation message?

**Recommendation**: Add explicit behavior to the vector search flow: "IF the Firestore Native vector search query fails due to connection or service errors, THE SYSTEM SHALL return `404` (masked internal error per SEC-004) and log the error with full server breadcrumbs including the failure step."

**Answer**
Use Recommendation

---

**NEW-GAP-005 | IMPORTANT | Specs: 02 | Dark mode toggle missing from FE-COMP-012 specification**

**Issue**: The Visual Design section (02-frontend-specifications.md, line 350) states:
> *THE SYSTEM SHALL provide a manual toggle in the site header (FE-COMP-012) to override the OS preference.*

However, FE-COMP-012 (Site Header Navigation, lines 862–897) only specifies navigation links and hamburger menu behavior. It does not mention a dark/light mode toggle, its placement, its icon, its behavior on click, or its accessibility characteristics.

**Recommendation**: Add to FE-COMP-012:
- A toggle button (e.g., sun/moon icon) in the header
- Toggle cycles between `light` → `dark` → `auto` (or just `light` ↔ `dark`, with `auto` as the initial OS-detected default)
- Accessibility: ARIA label (e.g., `aria-label="Toggle dark mode"`)
- Mobile: Include the toggle in the hamburger drawer or keep it visible alongside the hamburger icon

**Answer**
Use Recommendation

---

**NEW-GAP-006 | IMPORTANT | Specs: 02, 03 | Category dropdown shows all categories regardless of article type**

**Issue**: `GET /categories` (BE-API-008) returns all categories across all article types ("Categories are free-form and derived from the categories assigned to articles across all article types"). FE-PAGE-002, FE-PAGE-004, and FE-PAGE-007 all use the same `GET /categories` endpoint to populate their category dropdowns.

This means a user on `/technical` may see blog-only or others-only categories in the dropdown. Selecting a non-applicable category would return empty results — not an error, but a confusing UX.

**Recommendation**: Either:
- **(A)** Add an optional `type` query parameter to `GET /categories` (e.g., `?type=technical`) so the frontend can request only categories relevant to the current article type.
- **(B)** Accept the current design as intentional (categories are shared) and add a note in the frontend spec: "Some categories may yield empty results when used on a specific article list page because category assignment spans all article types."
- **(C)** Have the content pipeline ensure categories are distinct per type (out of scope per AD-022, but worth noting).

**Answer**
Implement Option A — add a `type` query parameter to `GET /categories` to allow filtering categories by article type, improving the relevance of the category dropdown and user experience. Let's have an array of types per category in the database given that categories are free-form and can be shared across types.
---

**NEW-GAP-007 | IMPORTANT | Specs: 02 | FE-COMP-009 duplicate detection excludes `page` parameter — caching ambiguity**

**Issue**: FE-COMP-009 defines duplicate search detection using the signature: `q + category + tags + tag_match + date_from + date_to`. The `page` parameter is excluded.

The spec also says: "Duplicate requests SHALL return cached results instead of making a new API call."

This creates ambiguity:
1. User searches "cloud" on page 1 → API call → results cached
2. User navigates to page 2 of same search → Different `page` but same signature → Is this a "duplicate"?
3. If yes, the cached page 1 results are returned instead of fetching page 2

Additionally, the "Duplicate requests SHALL return cached results" statement is in the FE-COMP-009 (Empty Search Results) section but reads as applying to ALL duplicate requests, not just empty results.

**Recommendation**: Clarify:
- The `page` parameter SHOULD be included in the duplicate detection signature (so different pages of the same search are NOT treated as duplicates), OR
- Explicitly scope the duplicate detection and caching to ONLY empty search results (not all results), and clarify that pagination changes always trigger API calls.

**Answer**
Use Recommendation

---

**NEW-GAP-008 | IMPORTANT | Specs: 06 | IPv4-mapped IPv6 address truncation not specified**

**Issue**: SEC-007 specifies IP truncation:
- IPv4: Zero the last octet
- IPv6: Zero the last 80 bits

However, IPv4-mapped IPv6 addresses (format `::ffff:a.b.c.d`) are commonly used in dual-stack environments and proxied connections. These represent IPv4 addresses in 128-bit IPv6 format. The spec does not define how to handle them:
- Treat as IPv4 (zero last IPv4 octet → `::ffff:a.b.c.0`)?
- Treat as IPv6 (zero last 80 bits → `::ffff:0:0`, which anonymizes the entire IPv4 portion)?

**Recommendation**: Add to SEC-007: "IPv4-mapped IPv6 addresses (format `::ffff:a.b.c.d`) SHALL be treated as IPv4 addresses for truncation purposes. The last IPv4 octet SHALL be zeroed (e.g., `::ffff:203.0.113.42` → `::ffff:203.0.113.0`)."

**Answer**
Add the above specification to SEC-007 to ensure consistent handling of IPv4-mapped IPv6 addresses

---

**NEW-GAP-009 | IMPORTANT | Specs: 03, 05 | Concurrent embedding sync execution not guarded**

**Issue**: The `sync-article-embeddings` Cloud Function (INFRA-014) can be triggered by:
1. Content CI/CD pipeline (on article publish/update)
2. Cloud Scheduler daily job (INFRA-014b)

If both trigger simultaneously (e.g., a content push happens at the same time as the daily schedule), two concurrent invocations of the same function could:
- Both detect the same changed articles
- Both call Vertex AI for the same embeddings (redundant API calls and cost)
- Race to write the same vector documents

The function is described as "idempotent" (hash-based change detection), so no data corruption occurs. But the duplicate Vertex AI API calls waste quota and cost.

**Recommendation**: Either:
- **(A)** Document this as an accepted trade-off (idempotent, occasional duplicate calls are low-cost)
- **(B)** Add a distributed lock mechanism (e.g., Firestore document-based lock with TTL) to prevent concurrent execution

**Answer**
Implement Option A — document that concurrent executions of the embedding sync function may occur but are idempotent and the cost of occasional duplicate Vertex AI calls is an accepted trade-off for simplicity.

---

**NEW-GAP-010 | IMPORTANT | Specs: 03 | No validation for inverted date range (`date_from` > `date_to`)**

**Issue**: BE-API-002 validates that `date_from` and `date_to` are valid ISO 8601 dates, but does not specify behavior when `date_from` is after `date_to` (e.g., `date_from=2025-12-01&date_to=2025-01-01`).

This inverted range would either:
- Return zero results (logical: no dates satisfy `>= Dec 1 AND < Jan 2`)
- Cause unexpected DB query behavior

**Recommendation**: Add validation: "IF `date_from` and `date_to` are both present AND `date_from` is after `date_to`, THE SYSTEM SHALL return HTTP `400` with error code `VALIDATION_ERROR` and message `'date_from must be on or before date_to'`."

**Answer**
Add the above validation to BE-API-002 to ensure clients receive clear feedback on invalid date range inputs, improving API usability and preventing confusion from zero-result responses due to inverted date ranges.

---

**NEW-GAP-011 | IMPORTANT | Specs: 03 | `tag_match=any` MongoDB operator not specified**

**Issue**: The vector search flow (step 5) specifies the `tag_match=all` operator: `tags: { $all: [...] }`. However, the MongoDB operator for `tag_match=any` is not specified.

For `any` matching (article has at least ONE specified tag), the MongoDB query should use `$in`: `tags: { $in: [...] }`. But this is left implicit.

**Recommendation**: Add to the filter application documentation: "WHEN `tag_match=any` (default), THE SYSTEM SHALL use `tags: { $in: [<tag values>] }`. WHEN `tag_match=all`, THE SYSTEM SHALL use `tags: { $all: [<tag values>] }`."

**Answer**
Use recommendation

---

**NEW-GAP-012 | IMPORTANT | Specs: 03 | ETag computation approach not specified (pre-computed vs per-request)**

**Issue**: BE-API-001 and BE-API-003/005 specify SHA-256 weak ETags of the response body. For conditional requests (`If-None-Match`), the backend must:
1. Fetch the content from Firestore
2. Compute SHA-256 of the response body
3. Compare with the client's `If-None-Match` value
4. Return 304 (match) or 200 with body (no match)

This means every conditional request still requires a full Firestore read, reducing the efficiency benefit of 304 responses (saves bandwidth but not DB load).

An alternative is to pre-compute and store the ETag hash in the article document (updated when content changes), allowing the backend to check the ETag without fetching full content.

**Recommendation**: Clarify the intended approach:
- **(A)** Per-request computation (simpler, current implicit behavior). Document: "The backend SHALL compute the ETag on each request by hashing the full response body. Conditional requests still require a database read."
- **(B)** Pre-computed ETag stored in the article document. Document: "The content pipeline SHALL compute and store a `content_hash` field. The backend SHALL compare `If-None-Match` against the stored hash before fetching full content."

**Answer**
Implement Option B. Pre-compute and store the ETag hash in the article document to optimize conditional request handling by avoiding unnecessary Firestore reads when the content has not changed.

---

### SUGGESTION

---

**NEW-GAP-013 | SUGGESTION | Specs: 06 | Ban expiry propagation delay not documented**

**Issue**: SEC-002 documents the LRU cache propagation delay for new bans: "A newly applied ban may take up to 60 seconds to take effect for already-cached 'not banned' IPs."

The reverse scenario — a ban that has expired but the LRU cache still holds the "banned" status — is not documented. A user whose ban legitimately expired could continue receiving 403/404 responses for up to 60 seconds.

**Recommendation**: Add a symmetric note: "Similarly, an expired ban may continue to be enforced for up to 60 seconds for IPs whose 'banned' status is still cached. This is an accepted trade-off for avoiding per-request Firestore lookups."

**Answer**
Add the above note to SEC-002 to clarify the expected behavior for expired bans and set user expectations regarding the propagation delay in both directions (new bans and expired bans) due to the LRU cache design.

---

**NEW-GAP-014 | SUGGESTION | Specs: 02 | No combined storage limit for manual offline saves + smart download**

**Issue**: FE-COMP-008 specifies a 50 MB cap for smart-download cached content. However, manually saved articles ("Save for offline reading") have no specified storage limit. Both use the Cache API.

A user who manually saves many large articles could exceed the browser's storage quota (typically 5–20% of device storage). No spec addresses this edge case.

**Recommendation**: Either:
- **(A)** Add a combined storage budget (e.g., 100 MB total for manual + smart download), with a warning when approaching the limit
- **(B)** Document that manual saves are unlimited and rely on browser-imposed quotas: "THE SYSTEM SHALL handle `QuotaExceededError` from the Cache API gracefully by displaying a message: 'Storage is full. Remove some offline articles to save new ones.'"

**Answer**
Implement Option B — document that manual saves are unlimited but the system will handle `QuotaExceededError` gracefully by informing the user when storage limits are reached, allowing them to manage their offline content without risking silent failures or data loss.

---

**NEW-GAP-015 | SUGGESTION | Specs: 03 | POST /t total request body size limit not specified**

**Issue**: Individual field limits are specified for POST /t (e.g., `page` max 500 chars, `error_message` max 2000 chars, `breadcrumbs` max 50 entries with max 300 chars per message). However, no overall HTTP request body size limit is specified at the application level.

Cloud Run has a default 32 MB body limit. A malicious actor could send a request with all fields at maximum size (though bounded, the total could still be several hundred KB per request × 60 req/min).

**Recommendation**: Add a note: "THE SYSTEM SHALL reject `POST /t` requests with Content-Length exceeding 100 KB with HTTP `413 Payload Too Large`." Alternatively, document that the per-field limits provide sufficient bounds and no overall limit is needed.

**Answer**
Add the above specification to BE-API-003 to enforce an overall request body size limit for POST /t, providing an additional layer of protection against abuse while still allowing ample room for valid error reports and breadcrumbs without risking excessive payloads that could impact backend performance or costs.

---

**NEW-GAP-016 | SUGGESTION | Specs: 05 | Cloud Scheduler free tier ceiling reached — no future growth path documented**

**Issue**: INFRA-008b, INFRA-008e, and INFRA-014b use all 3 Cloud Scheduler free tier jobs. The spec doesn't note this ceiling.

Any future scheduled task (e.g., periodic cache purge, analytics rollup, GeoIP DB update notification) would require upgrading to the paid tier ($0.10/job/month).

**Recommendation**: Add a note in the infrastructure spec: "All 3 Cloud Scheduler free-tier job slots are used. Additional scheduled tasks require the App Engine or paid plan. Consider consolidating future scheduled tasks into a single dispatcher function if needed."

**Answer**
Add the above note to the infrastructure specifications to clarify the current usage of Cloud Scheduler free-tier jobs and set expectations for future growth, encouraging efficient use of scheduled tasks and consideration of cost implications when adding new scheduled functions.

---

**NEW-GAP-017 | SUGGESTION | Specs: 03 | Go backend middleware execution order not specified**

**Issue**: The Go backend applies multiple middleware layers: CORS handling, ban status check (LRU cache → Firestore), HTTP method enforcement, JWT validation (POST /t only), input validation, and request breadcrumb initialization. The execution order of these middleware layers is not specified.

Order matters for:
- Should CORS preflight (`OPTIONS`) bypass ban checks? (Yes, per SEC-003 — but the middleware must be ordered accordingly)
- Should ban check occur before or after input validation?
- Does breadcrumb initialization happen before or after authentication?

**Recommendation**: Add a middleware execution order diagram to the backend spec:
1. Request ID generation + breadcrumb init
2. CORS handling (return early for OPTIONS preflight)
3. Ban status check
4. HTTP method enforcement
5. Route matching
6. Authentication (POST /t only)
7. Input validation
8. Business logic

**Answer**
Add the above middleware execution order diagram to the backend specifications to clarify the processing flow and ensure that security checks (CORS, ban status) are applied at the appropriate stages, improving the robustness and maintainability of the backend implementation.

---

**NEW-GAP-018 | SUGGESTION | Specs: 03, 07 | Naming overlap between frontend and backend breadcrumbs in error logs**

**Issue**: When a `POST /t` request with `action: "error_report"` fails internally (masked as 404), the resulting error log entry would contain:
- `server_breadcrumbs`: The backend's BE-BREADCRUMB processing trail
- The request body's `breadcrumbs` field: The frontend's FE-COMP-013 activity trail

Both fields appear in the same structured log entry. The naming distinction (`server_breadcrumbs` vs `breadcrumbs`) could be confusing during debugging.

**Recommendation**: Either:
- **(A)** Document the naming convention explicitly: "Backend processing breadcrumbs are logged as `server_breadcrumbs`. Frontend activity breadcrumbs from `POST /t` request bodies are logged as `client_breadcrumbs` (renamed from `breadcrumbs` during log emission)."
- **(B)** Accept the current naming and add a note: "When both server and client breadcrumbs appear in the same log entry (failed POST /t error reports), `server_breadcrumbs` contains the backend processing trail and the request body `breadcrumbs` field contains the frontend activity trail."

**Answer**
Implement Option A

---

**NEW-GAP-019 | SUGGESTION | Specs: 05 | Application CI/CD pipeline lacks formal INFRA component specification**

**Issue**: The GitHub Actions CI/CD pipeline that builds and deploys the Go backend Docker image (to Artifact Registry → Cloud Run) and the Nuxt 4 frontend (to Firebase Hosting via Firebase Functions) is referenced throughout the specs (INFRA-003 Docker image, INFRA-016 deployment boundary, SEC-008 dependency scanning, SEC-012 Terraform CI/CD) but is never formally specified as its own INFRA component.

The deployment boundary table in INFRA-016 distinguishes Terraform-provisioned resources from CI/CD-deployed artifacts, but the CI/CD pipeline stages, triggers, and workflow steps are not documented.

**Recommendation**: Add an `INFRA-020: Application CI/CD Pipeline (GitHub Actions)` component specifying:
- Trigger conditions (push to main, PR merge)
- Pipeline stages (lint → test → build → scan → deploy)
- Build artifacts (Docker image tag format, Firebase deploy target)
- Dependency scanning integration (`govulncheck`, `npm audit`, Docker image scan)
- Deployment targets and rollback strategy

**Answer**
Use the recommendation

---

## Cross-Reference Consistency Check

All component IDs (FE-PAGE-001–010, FE-COMP-001–014, BE-API-001–013, BE-BREADCRUMB, DM-001–012, INFRA-001–019, SEC-001–013, OBS-001–010) were verified for cross-spec references. No broken or dangling cross-references were found beyond the gaps documented above.

## Acceptance Criteria Coverage Check

All acceptance criteria (AC-FE-001–047, AC-RETRY-001–006, AC-MODAL-001–006, AC-API-001–033, AC-API-GEO-001–002, AC-BREADCRUMB-001–008, AC-DM-001–009, AC-INFRA-001–019, AC-SEC-001–018, AC-OBS-001–010) were cross-referenced against their parent specification sections. No orphaned or ungrounded acceptance criteria were found.
