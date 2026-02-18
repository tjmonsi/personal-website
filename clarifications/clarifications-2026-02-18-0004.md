# Clarifications — 2026-02-18-0004

Third re-check after applying CLR-132 through CLR-143. 2 Important and 2 Minor findings.

---

## CLR-144 — GeoIP Lookup IP Source Contradiction

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 02, 03, 06, 07  
**Affected IDs**: BE-API-009, AC-API-GEO-001, SEC-007, FE-PAGE-009, OBS-002

**Problem**: There is a direct contradiction about whether GeoIP lookup uses the full or truncated IP address.

**"Full IP before truncation"** (current in BE-API-009 line ~693 and OBS-002):
> "THE SYSTEM SHALL perform a GeoIP country lookup on the full (un-truncated) client IP address... This lookup occurs before IP truncation (CLR-123)."

**"Truncated IP"** (current in AC-API-GEO-001 line ~838, SEC-007, FE-PAGE-009):
> AC-API-GEO-001: "it SHALL resolve geo_country from the client's truncated IP address"
> SEC-007: "derived from truncated IP via embedded MaxMind GeoLite2-Country database"
> FE-PAGE-009: "derived from the truncated IP address using a locally-stored geographic database"

Both versions exist *within* `03-backend-api-specifications.md` (BE-API-009 vs AC-API-GEO-001).

**Why this matters**: The privacy policy (FE-PAGE-009) tells users their country is derived from an already-truncated IP. If the implementation follows BE-API-009 (full IP lookup), the privacy policy is inaccurate. If it follows the truncated IP approach, GeoIP accuracy is significantly degraded (zeroed last octet often resolves to the wrong country).

**Option A (Recommended)**: Use **full IP before truncation** for GeoIP lookup. This matches the original CLR-123 intent and provides accurate country resolution. Update AC-API-GEO-001, SEC-007, and FE-PAGE-009 to say "derived from the client IP address prior to truncation; only the country code is retained — the full IP is discarded immediately after lookup."

**Option B**: Use **truncated IP** for GeoIP lookup. This maximizes privacy but degrades accuracy. Update BE-API-009 and OBS-002 to remove "full (un-truncated)" and "before IP truncation."

**Answer**
Use A.
---

## CLR-145 — INFRA-014 Runtime Missing "22 LTS" (CLR-135 Incomplete)

**Priority**: 🟡 IMPORTANT  
**Affected Specs**: 05  
**Affected IDs**: INFRA-014

**Problem**: The `sync-article-embeddings` Cloud Function (INFRA-014) runtime is specified as "Node.js (Cloud Functions Gen 2)" but is missing the "22 LTS" version qualifier and the "(CLR-135)" traceability tag. All three other Cloud Functions (INFRA-008a, 008c, 008d) already say "Node.js 22 LTS (Cloud Functions Gen 2) (CLR-135)."

The 01-system-overview tech stack table correctly lists all 4 Cloud Functions as "Node.js 22 LTS," confirming the intent.

**Option A (Recommended)**: Update INFRA-014 Runtime to `Node.js 22 LTS (Cloud Functions Gen 2) (CLR-135)` to match the other three functions.

**Option B**: Leave as-is and treat as an implementation-time detail.

**Answer**
Use A.

---

## CLR-146 — DM-011 Field Name `vector` vs DM-012 `embedding`

**Priority**: 🟢 MINOR  
**Affected Specs**: 04  
**Affected IDs**: DM-011, DM-012

**Problem**: Two related data model collections use different field names for conceptually similar data (float arrays produced by the same Gemini embedding model):
- DM-011 (`embedding_cache` in Firestore Enterprise): field named **`vector`**
- DM-012 (vector collections in Firestore Native): field named **`embedding`**

While these are different databases serving different purposes (query cache vs document store), the naming inconsistency could confuse implementers.

**Option A (Recommended)**: Add a brief note in DM-011 explaining the naming difference: "The field is named `vector` (rather than `embedding`) to distinguish cached search-query vectors from document embeddings stored in DM-012." No field rename needed.

**Option B**: Rename DM-011's `vector` field to `embedding` and update all references (AC-DM-003, BE-API-002 Step 3, AC-DM-011-A/B).

**Answer**
Use A.

---

## CLR-147 — INFRA-010d Log Sink Filter Uses Placeholder Service Name

**Priority**: 🟢 MINOR  
**Affected Specs**: 05  
**Affected IDs**: INFRA-010d, INFRA-003

**Problem**: The backend error log sink filter at INFRA-010d contains `resource.labels.service_name="<cloud-run-service-name>"` with a note "The actual service name is determined during implementation." No spec assigns a canonical name to the Cloud Run service.

**Option A (Recommended)**: Define the Cloud Run service name as `website-api` in INFRA-003 and use that name in INFRA-010d's filter: `resource.labels.service_name="website-api"`. Add a note that the name is illustrative and may be adjusted during implementation.

**Option B**: Leave the placeholder as-is. The service name is an implementation-time detail that does not affect the spec's correctness.

**Answer**
Use A.
