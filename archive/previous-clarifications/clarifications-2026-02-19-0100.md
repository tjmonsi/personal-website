# Clarifications — 2026-02-19-0100

Post-CLR-148–175 application re-check. All 28 previous CLR items have been applied.
This file contains new issues discovered during the comprehensive re-review.

Items fixed directly (no user input needed) — not listed here:
- Spec 05: INFRA-005 Cloud Armor table broken by inline CLR-174 note → moved after table
- Spec 05: INFRA-008a duplicate step number → renumbered
- Spec 05: INFRA-014 wrong database name in partial failure example → corrected to Firestore Native
- Spec 02: AC-FE-010 contradiction with CLR-166 → updated to persist error toasts
- Spec 02: FE-COMP-003 error snackbar table broken by inline notes → moved notes after table
- Spec 03: 304 status code list missing BE-API-001 → added
- Spec 03: AC-API-001 flat pagination fields → updated to nested object
- Spec 04: DM-011 growth note referencing articles instead of queries → corrected

---

## 🔴 CRITICAL (3 items)

### CLR-176 — `connection_speed` Fallback Value Type Mismatch

**Specs affected:** 02-frontend (FE-COMP-004), 03-backend-api (BE-API-009), 07-observability (OBS-003)

**Issue:** CLR-167 added that when `navigator.connection` is unavailable (Safari/Firefox), the frontend sends `connection_speed: "unknown"`. However, the backend spec defines `connection_speed` as an optional **object** with keys `effective_type`, `downlink`, `rtt`. Sending a string `"unknown"` would fail backend validation.

**Options:**
- **Option A (Recommended):** Omit the `connection_speed` field entirely when the API is unavailable. The field is already optional — the backend and BigQuery pipeline should handle its absence gracefully. Simplest and cleanest.
- **Option B:** Send `connection_speed: { "effective_type": "unknown" }` to preserve the object structure, but `"unknown"` is not in the allowed enum (`slow-2g`, `2g`, `3g`, `4g`).
- **Option C:** Add explicit backend handling: "IF `connection_speed` is not an object, THE SYSTEM SHALL treat it as null."

**Your answer:**
Use A
---

### CLR-177 — Smart Download Network Check Uses Unsupported Property

**Specs affected:** 02-frontend (FE-COMP-008)

**Issue:** CLR-158 specified `navigator.connection.type === 'wifi'` for the Wi-Fi-only smart download check. However, FE-COMP-004 (CLR-142) only extracts `effectiveType`, `downlink`, and `rtt` — **not** `type`. The `.type` property has even more limited browser support than `.effectiveType` (not available on Chrome desktop). This makes the requirement unimplementable on most desktop browsers.

**Options:**
- **Option A (Recommended):** Use `navigator.connection.effectiveType === '4g'` as a proxy for high-bandwidth connections suitable for prefetching. Available in Chrome/Edge/Opera. When unavailable (Safari/Firefox), disable smart download prefetching.
- **Option B:** Keep `navigator.connection.type === 'wifi'` but add a fallback: when `.type` is unavailable, check `.effectiveType === '4g'` instead.
- **Option C:** Remove the network condition check entirely — always prefetch if device is online.

**Your answer:**
Use A
---

### CLR-178 — Vector Search Pagination Without Filters Is Unimplementable as Described

**Specs affected:** 03-backend-api (BE-API-002, Note after Vector Search Flow)

**Issue:** The spec says: "When `q` is present without filters, the system maps the frontend page number directly to Firestore Native vector search pagination (offset/limit)." However, Firestore Native's `findNearest()` API supports `limit` but **not** `offset`. The described offset/limit pagination strategy cannot be implemented against the actual API.

Additionally, computing `total_items` for the pagination response requires knowing how many documents fall within the distance threshold, but `findNearest()` doesn't provide a count-only mode.

**Options:**
- **Option A (Recommended):** Unify both cases (with and without filters). Always fetch up to 500 candidate IDs from Firestore Native via `findNearest(limit: 500)`, then paginate from that bounded result set. `total_items` = number of candidates returned (capped at 500). This matches the existing "with-filters" flow.
- **Option B:** Use cursor-based pagination instead of page numbers for vector search results. The frontend would use "Load More" instead of page numbers when `q` is present.
- **Option C:** Fetch all results up to `limit * page_number` from Firestore Native on each request and discard the first `(page - 1) * limit` results to simulate offset.

**Your answer:**
Use A
---

## 🟡 IMPORTANT (7 items)

### CLR-179 — Search Cache Key: Query Only vs. Query + Filters

**Specs affected:** 02-frontend (FE-COMP-009)

**Issue:** CLR-165 defined similar query detection (exact match + Levenshtein ≤ 2) but only mentions the `q` parameter. It does not specify whether filter state (category, tags, tag_match, date_from, date_to) is part of the cache key. A user searching "docker" with category "DevOps" and then "docker" with category "Cloud" has identical `q` but different filters — should the cache apply?

**Options:**
- **Option A (Recommended):** The cache key includes the full filter state (q + category + tags + tag_match + date_from + date_to). Only requests with the same query AND same filters return cached results.
- **Option B:** Only `q` is compared. Cached empty results apply regardless of filter state to avoid repeated search API calls.

**Your answer:**
Use A
---

### CLR-180 — Error Response Body Format Undefined

**Specs affected:** 03-backend-api (all endpoints)

**Issue:** The spec states error responses (4xx) SHALL use `Content-Type: application/json` but never defines the JSON body structure. When the backend returns 400 (e.g., "Maximum of 10 tags allowed"), the implementer won't know what JSON structure to use.

Note: The frontend handles all errors by HTTP status code only (not by parsing body), so this is not a frontend blocker. But it's needed for the Go backend implementation.

**Options:**
- **Option A (Recommended):** Define a standard error response schema:
  ```json
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "Human-readable error description"
    }
  }
  ```
- **Option B:** Simple flat structure: `{ "error": "Human-readable error description" }`
- **Option C:** Leave undefined — let the implementer decide.

**Your answer:**
Use A
---

### CLR-181 — AD-011 Contradicts Local-Only Dev Environment

**Specs affected:** 01-system-overview (AD-011, Environments section)

**Issue:** AD-011 says the Development environment "does NOT use a VPC to reduce cost; services connect to Google Cloud APIs directly." But the Environments note explicitly says: "The Development environment is local-only — it runs entirely on the developer's machine using emulators and local databases. No GCP resources are provisioned or deployed for Development." These contradict each other.

**Options:**
- **Option A (Recommended):** Remove the Development clause from AD-011. The AD should focus on Production's VPC rationale. Add a parenthetical: "(Development is local-only and has no GCP networking; see Environments.)"
- **Option B:** Keep both but reconcile — change AD-011 to say "Development uses local emulators with no GCP networking."

**Your answer:**
Use A
---

### CLR-182 — Dev Environment Database Spec Ambiguous

**Specs affected:** 01-system-overview (Environments table, Database column)

**Issue:** The Development database is listed as "Firestore Emulator / Local MongoDB (Firestore Enterprise)". The `/` is ambiguous — does it mean "or" (choose one) or "and" (use both)?

**Options:**
- **Option A (Recommended):** Use only Local MongoDB for development. This covers Firestore Enterprise via the MongoDB compatibility layer. The Firestore Emulator is not needed since all data operations go through the MongoDB wire protocol. Reword to: "Local MongoDB (for Firestore Enterprise MongoDB-compat development)".
- **Option B:** Use both — Firestore Emulator for Firestore Native (vector search) and Local MongoDB for Firestore Enterprise. Reword to: "Firestore Emulator (for Native vector search) + Local MongoDB (for Enterprise)".

**Your answer:**
Use A
---

### CLR-183 — Dual-Database Query Coordination Strategy Undefined (AD-021)

**Specs affected:** 01-system-overview (AD-021), 03-backend-api (Vector Search Flow)

**Issue:** AD-021 says when `q` is present, the system uses Firestore Native vector search, and filtering is done in Firestore Enterprise. But when `q` is present AND filters are applied simultaneously, the coordination flow is not explicitly defined in the system overview. (It IS defined in spec 03's Vector Search Flow, but the overview doesn't reference it.)

**Options:**
- **Option A (Recommended):** Add a brief forward reference in AD-021: "For the detailed query coordination strategy (vector search + filter pipeline), see BE-API-002 Vector Search Flow in the Backend API Specifications."
- **Option B:** Duplicate the full coordination description in AD-021.

**Your answer:**
Use A
---

### CLR-184 — AC-DM-005 References "Compound Indexes" That Don't Exist

**Specs affected:** 04-data-model (AC-DM-005, DM-002/DM-003/DM-005)

**Issue:** AC-DM-005 says "the defined compound indexes support the query without a full collection scan." But DM-002's index definitions only contain single-field indexes (`idx_slug`, `idx_category`, `idx_tags`, `idx_date_updated`, `idx_date_created`). No compound index is defined for multi-field queries (e.g., filter by category + sort by date_updated).

**Options:**
- **Option A (Recommended):** Add compound indexes to DM-002/DM-003/DM-005 for the expected query patterns:
  - `idx_category_date` on `{ category: 1, date_updated: -1 }` — for filtered+sorted listing
  - `idx_tags_date` on `{ tags: 1, date_updated: -1 }` — for tag-filtered listing
  And update AC-DM-005 accordingly.
- **Option B:** Reword AC-DM-005 to remove "compound" and rely on MongoDB's index intersection for multi-field queries.

**Your answer:**
Use A
---

### CLR-185 — SEC-013 Service Accounts #2–#4 Missing Specific IAM Role Names

**Specs affected:** 06-security (SEC-013)

**Issue:** SEC-013 entries #1, #5–#9 list specific GCP IAM role names (e.g., `roles/datastore.user`). But entries #2 (`cf-sitemap-gen`), #3 (`cf-rate-limit-proc`), and #4 (`cf-offender-cleanup`) use only descriptions like "Firestore Enterprise read/write". Since we confirmed Firestore Enterprise uses `roles/datastore.user` (CLR-148), these entries should specify the same role.

**Options:**
- **Option A (Recommended):** Update entries #2–#4 to explicitly list `roles/datastore.user` with collection-level notes:
  - #2: `roles/datastore.user` (read `technical_articles`, `blog_articles`; read/write `sitemap`)
  - #3: `roles/datastore.user` (read/write `rate_limit_offenders`)
  - #4: `roles/datastore.user` (read/write `rate_limit_offenders`)
- **Option B:** Leave as-is — the descriptions are clear enough for implementation.

**Your answer:**
Use A
---

## 🟢 SUGGESTION (7 items)

### CLR-186 — DM-009 `identifier`: Full vs. Truncated IP

**Specs affected:** 04-data-model (DM-009)

**Issue:** DM-009's `identifier` field says "Client IP address" (CLR-157) but doesn't specify whether this is the full IP or the privacy-truncated version (last octet zeroed). The backend truncates IPs for tracking/BigQuery but uses full IPs for rate limiting.

**Options:**
- **Option A (Recommended):** Clarify as "Full (untruncated) client IP address" since rate limiting requires exact match for ban enforcement.
- **Option B:** Leave as-is — the rate limiting context implies full IP.

**Your answer:**
Use A
---

### CLR-187 — `/privacy` and `/changelog` Pages Have No Content Source Defined

**Specs affected:** 01-system-overview, 02-frontend

**Issue:** The SPA routes `/privacy` and `/changelog` are listed as pages but have no API endpoint and no defined content source. Are they static SPA views with hardcoded content, or do they fetch from the API?

**Options:**
- **Option A (Recommended):** Static SPA views with content embedded in the frontend codebase (no API fetch needed). Add a brief note: "The `/privacy` and `/changelog` pages are static views rendered from local markdown files bundled with the SPA."
- **Option B:** Add API endpoints for these pages.

**Your answer:**
Use A
---

### CLR-188 — BE-API-008 "Usage Counts If Applicable" Ambiguity

**Specs affected:** 03-backend-api (BE-API-008)

**Issue:** CLR-168 added a note saying the endpoint returns "category names (and their usage counts if applicable)." However, the response schema only has `name` and `date_created` — no count field. The parenthetical creates ambiguity.

**Options:**
- **Option A (Recommended):** Remove the parenthetical. The endpoint returns only category names and creation dates, matching the schema. Article counts can be computed client-side later if needed.
- **Option B:** Add an `article_count` field to the response schema and compute it server-side.

**Your answer:**
Use A
---

### CLR-189 — Empty `q` Parameter Behavior Unspecified

**Specs affected:** 03-backend-api (BE-API-002/004/007)

**Issue:** When `q` is present but empty or whitespace-only (`?q=` or `?q=   `), the spec doesn't say whether it triggers vector search or is treated as absent.

**Options:**
- **Option A (Recommended):** Treat empty/whitespace-only `q` as absent (no vector search, return standard date-sorted listing). Add validation: "IF `q` is present but empty or whitespace-only, THE SYSTEM SHALL treat it as absent."
- **Option B:** Return 400 Bad Request for empty `q`.

**Your answer:**
Use A
---

### CLR-190 — Front Page Has No Client-Side Caching/Conditional Request Behavior

**Specs affected:** 02-frontend (FE-PAGE-001)

**Issue:** BE-API-001 returns `Cache-Control: public, max-age=300`, `ETag`, and `Last-Modified` and supports 304. But FE-PAGE-001 doesn't specify whether the frontend should use conditional requests or cache the response. The offline reading spec (FE-COMP-008) only covers article detail endpoints.

**Options:**
- **Option A (Recommended):** The frontend relies on browser-native HTTP caching for the front page (no explicit conditional request logic needed in code). The browser automatically handles `Cache-Control`, `ETag`, and `If-None-Match` headers. Offline access to the front page is not required. Add a brief note.
- **Option B:** Add explicit conditional request logic for the front page in the SPA code.

**Your answer:**
Use A
---

### CLR-191 — CSP Missing Explicit `worker-src` for Service Worker

**Specs affected:** 06-security (SEC-005)

**Issue:** The CSP includes `script-src 'self' 'unsafe-inline'` but no `worker-src` directive. Per CSP, `worker-src` falls back to `script-src`, so service workers from `'self'` should work. However, explicitly adding `worker-src 'self'` improves clarity and forward-compatibility.

**Options:**
- **Option A (Recommended):** Add `worker-src 'self'` to the CSP in SEC-005.
- **Option B:** Leave as-is — the fallback behavior is sufficient.

**Your answer:**
Use A
---

### CLR-192 — SEC-012 Constraint Text Contradicts Granted `roles/datastore.user` Role

**Specs affected:** 06-security (SEC-012)

**Issue:** SEC-012 says the Terraform SA "SHALL NOT have access to application data" but `roles/datastore.user` at project scope grants read/write to Firestore entities (application data). The roles are correct per CLR-149 but the constraint text is misleading.

**Options:**
- **Option A (Recommended):** Soften the constraint to acknowledge the trade-off: "The service account SHOULD minimize access to application data. Where infrastructure management roles (e.g., `roles/datastore.user`) inherently grant data-level access, this is an accepted trade-off documented in CLR-149."
- **Option B:** Leave as-is — the CLR-149 note provides enough context.

**Your answer:**
Use A
---

*End of clarifications — 17 items total (3 critical, 7 important, 7 suggestions)*
