# Clarifications — 2026-02-18-0003

Second re-check after applying CLR-121 through CLR-131. Found 3 critical issues, 4 important issues, and 5 minor issues across all 7 specs and the cost estimate.

**Root causes**: CLR-121 (image cache removal), CLR-123 (geo_country/GeoIP), and CLR-129 (Node.js 22) were incompletely propagated. Several cross-references and acceptance criteria are missing.

---

## CLR-132 — Remove stale image cache references (Critical)

**Priority**: 🔴 Critical — Direct contradiction with CLR-121
**Affected files**: `specs/05-infrastructure-specifications.md`, `specs/07-observability-specifications.md`, `specs/03-backend-api-specifications.md`

**Issue**: CLR-121 removed the in-memory image cache from BE-API-013, but three other locations still reference it:

1. **INFRA-019 Notes** (line ~851): "The Go backend caches images in memory for 30 days to reduce Cloud Storage read operations (see BE-API-013)."
2. **OBS-004 Custom Metrics** (lines ~365-366): `image_cache_hits_total` and `image_cache_misses_total` metrics still listed.
3. **BE-BREADCRUMB cache_lookup** (line ~77): Cache type column includes `image_cache` and `sitemap_cache`. The `image_cache` no longer exists (CLR-121). The `sitemap_cache` is also undocumented — there is no caching layer defined for sitemap data; BE-API-011 reads directly from the Firestore `sitemap` collection.

**Options**:

- **Option A (Recommended)**: Apply all three fixes:
  1. INFRA-019: Replace the caching note with "The Go backend proxies images directly from Cloud Storage on each request. Cloud Storage serves as the authoritative source with no application-level caching. The frontend relies on HTTP `Cache-Control` headers for browser-level caching (see BE-API-013)."
  2. OBS-004: Remove `image_cache_hits_total` and `image_cache_misses_total` rows from the custom metrics table.
  3. BE-BREADCRUMB: Change cache type examples to `embedding_cache` only (the only remaining cache). Remove `image_cache` and `sitemap_cache`.

**Answer**:
Use A.
---

## CLR-133 — GeoIP implementation approach and cross-spec updates (Critical)

**Priority**: 🔴 Critical — Undocumented dependency spans infrastructure, Docker, privacy, cost, and VPC
**Affected files**: `specs/01-system-overview.md`, `specs/02-frontend-specifications.md`, `specs/03-backend-api-specifications.md`, `specs/05-infrastructure-specifications.md`, `specs/06-security-specifications.md`, `specs/07-observability-specifications.md`, `docs/cost-estimate-draft.md`

**Issue**: CLR-123 added `geo_country` (resolved via GeoIP lookup from the truncated IP), but the GeoIP implementation is not documented anywhere. This creates gaps:

1. **System overview** (`01-system-overview.md`): Technology stack table has no GeoIP entry.
2. **Infrastructure** (`05-infrastructure-specifications.md`): No specification for how/where the GeoIP database is provisioned.
3. **Docker image** (`INFRA-003`): If using an embedded GeoIP database (e.g., MaxMind GeoLite2), it must be bundled in the Docker image.
4. **VPC** (`INFRA-009`): Network policy states "No outbound internet access is required" — if GeoIP uses an external API, this conflicts. An embedded database is compatible.
5. **Privacy** (`FE-PAGE-009`): The privacy policy does not disclose `geo_country` collection.
6. **Security** (`SEC-007`): No mention of GeoIP data handling.
7. **BigQuery** (`INFRA-010`): The "Summary of Data Stored in BigQuery" table does not include `geo_country` in the "Personal Data Present" column for `frontend_tracking_logs` or `frontend_error_logs`.
8. **Cost estimate**: No mention of GeoIP database or any associated cost.

**Options**:

- **Option A (Recommended)**: Use an **embedded GeoIP database** (MaxMind GeoLite2-Country) bundled in the Docker image. This approach:
  - Is free (GeoLite2 requires free registration with MaxMind, CC BY-SA 4.0 license)
  - Requires no outbound internet (compatible with VPC policy)
  - Adds ~5 MB to Docker image size
  - Resolves country from the **truncated** IP (accuracy is lower with zeroed last octet, but acceptable for country-level)
  - Requires periodic database updates (monthly, via CI/CD pipeline downloading new DB during Docker build)
  - Apply cross-spec updates:
    1. System overview: Add "GeoIP | MaxMind GeoLite2-Country (embedded) | Bundled in Docker image" to tech stack
    2. INFRA-003 Docker image: Add `COPY` step for GeoLite2 database file
    3. INFRA-009: Add note "GeoIP resolution uses an embedded database file bundled in the Docker image — no external API calls required"
    4. FE-PAGE-009: Add to "Data We Collect": "approximate country (derived from the truncated IP address using a locally-stored geographic database — no external services are contacted)"
    5. SEC-007: Add note that geo_country is derived from truncated IPs and is approximate (country-level only)
    6. INFRA-010 BigQuery summary: Add `geo_country` to the "Personal Data Present" column for `frontend_tracking_logs` and `frontend_error_logs`
    7. Cost estimate: Add note "GeoIP (MaxMind GeoLite2-Country): $0.00 (free database, ~5 MB in Docker image)"

- **Option B**: Use **IP-to-country heuristic** without a GeoIP database (derive country from IP range allocation tables). Simpler but less accurate.

- **Option C**: Drop `geo_country` entirely if the complexity is not worth it for a personal website.

**Answer**:
Use A.
---

## CLR-134 — Fix AC-DM-003 field name: `vector` → `embedding` (Critical)

**Priority**: 🔴 Critical — Acceptance criterion references wrong field name
**Affected file**: `specs/04-data-model-specifications.md`

**Issue**: AC-DM-003 (line ~412) states:

> "...then each article has a corresponding vector document with a **`vector`** field containing a 2048-dimension float array."

But DM-012a defines the field as **`embedding`**, not `vector`. The AC will fail against the actual schema.

**Options**:

- **Option A (Recommended)**: Change `vector` to `embedding` in AC-DM-003:
  > "...then each article has a corresponding vector document with an **`embedding`** field containing a 2048-dimension float array."

**Answer**:
Use A.
---

## CLR-135 — Propagate Node.js 22 to all Cloud Functions and system overview (Important)

**Priority**: 🟡 Important — CLR-129 was only partially applied
**Affected files**: `specs/01-system-overview.md`, `specs/05-infrastructure-specifications.md`

**Issue**: CLR-129 specified Node.js 22 LTS, but only INFRA-014 was updated. The following still say "Node.js (Cloud Functions Gen 2)" without a version:

1. **INFRA-002** (line ~71): Firebase Functions runtime — "Node.js (as required by Nuxt/Nitro)"
2. **INFRA-008a** (line ~300): generate-sitemap — "Node.js (Cloud Functions Gen 2)"
3. **INFRA-008c** (line ~347): process-rate-limit-logs — "Node.js (Cloud Functions Gen 2)"
4. **INFRA-008d** (line ~402): cleanup-rate-limit-offenders — "Node.js (Cloud Functions Gen 2)"
5. **System overview** (line ~22-27): Technology stack — all four Cloud Functions rows say "Node.js (Cloud Functions Gen 2)"

**Options**:

- **Option A (Recommended)**: Update all five locations to "Node.js 22 LTS (Cloud Functions Gen 2)" and the system overview to "Node.js 22 LTS (Cloud Functions Gen 2)" for each Cloud Functions row. Also update INFRA-002 to "Node.js 22 LTS (as required by Nuxt/Nitro)".

- **Option B**: Keep INFRA-002 as "Node.js (as required by Nuxt/Nitro)" since Firebase Functions runtime version is dictated by the Nuxt/Nitro deploy target and may differ. Only update INFRA-008a, 008c, 008d and the system overview.

**Answer**:
Use A.
---

## CLR-136 — Add model_version check to vector search flow and acceptance criteria (Important)

**Priority**: 🟡 Important — CLR-125 cache invalidation not referenced in search flow
**Affected files**: `specs/03-backend-api-specifications.md`, `specs/04-data-model-specifications.md`

**Issue**: CLR-125 added `model_version` to the `embedding_cache` schema (DM-011) with automated invalidation logic. However:

1. **BE-API-002 Vector Search Flow Step 3** does not mention checking `model_version` when a cache hit occurs. If the cached embedding was generated by an older model version, it should be treated as a cache miss.
2. **No acceptance criteria** exist for `model_version` cache invalidation behavior.

**Options**:

- **Option A (Recommended)**: Apply both fixes:
  1. Update BE-API-002 Step 3 cache hit path to: "**Cache hit**: Verify the cached entry's `model_version` matches the current model version. If it matches, use the cached embedding vector. If it does not match, treat as a cache miss (re-generate embedding and update the cache entry)."
  2. Add acceptance criteria to DM-011 or the relevant section:
     - **AC-DM-011-A**: Given the embedding cache, when a cache hit occurs with a mismatched `model_version`, then the system SHALL treat it as a cache miss and re-generate the embedding.
     - **AC-DM-011-B**: Given a model version change, when `sync-article-embeddings` runs, then it SHALL regenerate all embeddings and update `model_version` in all cache entries.

**Answer**:
Use A.
---

## CLR-137 — Add geo_country acceptance criteria (Important)

**Priority**: 🟡 Important — No testable criteria for geo_country behavior
**Affected file**: `specs/03-backend-api-specifications.md`

**Issue**: `geo_country` was added to all four `POST /t` action log entries (CLR-123), but there are no acceptance criteria verifying that it is correctly resolved and included.

**Options**:

- **Option A (Recommended)**: Add the following acceptance criteria to BE-API-009:
  - **AC-API-GEO-001**: Given a `POST /t` request, when the backend processes the request, then it SHALL resolve `geo_country` from the client's truncated IP address using the embedded GeoIP database, and include it in the structured log entry.
  - **AC-API-GEO-002**: Given a `POST /t` request where the GeoIP lookup fails or returns no result, then the system SHALL set `geo_country` to `"unknown"` in the structured log entry.

**Answer**:
Use A.
---

## CLR-138 — Fix OBS-003 code example camelCase → snake_case note (Important)

**Priority**: 🟡 Important — CLR-130 stated snake_case convention, but OBS-003 example contradicts
**Affected file**: `specs/07-observability-specifications.md`

**Issue**: OBS-003 "Connection Speed Detection" code example (lines ~302-309) shows:

```javascript
const speedInfo = connection ? {
  effectiveType: connection.effectiveType,
  downlink: connection.downlink,
  rtt: connection.rtt
} : null;
```

This shows the raw browser API (camelCase), but CLR-130 established that the frontend converts to snake_case before sending. FE-COMP-004 already documents this conversion. The OBS-003 example could mislead implementers into thinking camelCase is what reaches the backend.

**Options**:

- **Option A (Recommended)**: Add a note below the code example: "**Note**: This example shows the raw browser API values. The frontend converts `effectiveType` to `effective_type` before sending to the backend (see FE-COMP-004, CLR-130). The backend receives and logs snake_case field names."

- **Option B**: Rewrite the code example to show the snake_case conversion inline.

**Answer**:
Use A.
---

## CLR-139 — Fix OBS-001 Example 1 inconsistency (Minor)

**Priority**: 🟢 Minor — Example is internally inconsistent but non-blocking
**Affected file**: `specs/07-observability-specifications.md`

**Issue**: OBS-001 Example 1 is titled "Standard Backend Error Log (e.g., GET /technical with a masked 500)" and includes `"masked_500": true`. However:

1. The error details show `"type": "NotFoundError"` and `"message": "No article with slug 'invalid-slug'"` — this is a genuine 404 (article not found), not an internal error masked as 404.
2. The slug `invalid-slug` doesn't match the validation pattern `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}\.md$`, yet the breadcrumb shows `"result": "pass"` for validation.

A masked 500 scenario means an internal server error (e.g., database timeout) that is returned to the client as a 404 for security. The example should show that.

**Options**:

- **Option A (Recommended)**: Update the example to show a genuine masked 500:
  - Slug: `my-article-2026-01-15-1030.md` (passes validation)
  - Error type: `"DeadlineExceeded"` or `"InternalError"`
  - Error message: `"Firestore query timed out after 10000ms"`
  - Breadcrumb trail: validation passes, db_query starts, then error step with timeout
  - This correctly demonstrates `masked_500: true` (real error is 500, client sees 404)

**Answer**:
Use A.
---

## CLR-140 — Add BigQuery dataset to INFRA-016 Terraform scope (Minor)

**Priority**: 🟢 Minor — Missing Terraform-managed resource
**Affected file**: `specs/05-infrastructure-specifications.md`

**Issue**: INFRA-016 lists "Cloud Logging log sinks — BigQuery sinks (INFRA-010a–010e)" in the Terraform scope, but does not explicitly list the BigQuery `website_logs` dataset itself. The dataset must exist before log sinks can route to it.

**Options**:

- **Option A (Recommended)**: Add "BigQuery dataset (`website_logs`, INFRA-010)" to the Terraform scope list, after the log sinks entry.

**Answer**:
Use A.
---

## CLR-141 — Add CLR-121 note to cost estimate changelog (Minor)

**Priority**: 🟢 Minor — Changelog incomplete
**Affected file**: `docs/cost-estimate-draft.md`

**Issue**: The "Changes from v1.0-draft" section documents breadcrumb tracking, error retry, modal, and server-side breadcrumbs. It does not mention CLR-121 (image cache removal), even though it was part of the same clarification batch. No cost impact, but the changelog should be complete.

**Options**:

- **Option A (Recommended)**: Add a bullet to "Changes from v1.0-draft":
  > "**Image cache removed (CLR-121)**: In-memory image cache removed from BE-API-013. No cost impact — caching was in-process memory only."

**Answer**:
Use A.
---

## CLR-142 — Specify exact connection_speed keys in FE-COMP-004 (Minor)

**Priority**: 🟢 Minor — Prevents silent field dropping
**Affected file**: `specs/02-frontend-specifications.md`

**Issue**: BE-API-009 validates `connection_speed` keys (`effective_type`, `downlink`, `rtt`) and says "Unknown keys SHALL be ignored." FE-COMP-004 says the frontend converts camelCase to snake_case (CLR-130) but doesn't specify which exact keys to extract from `navigator.connection`. If the browser API adds new properties in the future, the frontend might send them and they'd be silently dropped by the backend.

**Options**:

- **Option A (Recommended)**: Add a note to FE-COMP-004 specifying the exact keys to extract:
  > "THE SYSTEM SHALL extract only the following keys from `navigator.connection` and convert to snake_case: `effectiveType` → `effective_type`, `downlink` → `downlink`, `rtt` → `rtt`. Other properties SHALL NOT be sent."

**Answer**:
Use A.
---

## CLR-143 — Add embedding_cache to SEC-007 Data Retention table (Minor)

**Priority**: 🟢 Minor — Data retention table incomplete
**Affected file**: `specs/06-security-specifications.md`

**Issue**: SEC-007 Data Retention Summary lists retention for all BigQuery tables and `rate_limit_offenders`, but does not include the `embedding_cache` collection (DM-011). This collection stores search query embeddings with **no expiration** (infinite retention) and has a manual purge procedure (CLR-125).

**Options**:

- **Option A (Recommended)**: Add a row to the SEC-007 Data Retention table:

  | Data Store / Table | Data Type | Retention Period | Deletion Method |
  | --- | --- | --- | --- |
  | Firestore `embedding_cache` (DM-011) | Search query embeddings (no query text stored) | No expiration | Manual purge (DM-011 procedure) |

**Answer**:
Use A.
---

**Summary**: 3 critical, 4 important, 5 minor — 12 items total (CLR-132 through CLR-143).
