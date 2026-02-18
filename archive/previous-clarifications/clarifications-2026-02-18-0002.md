# Clarifications — 2026-02-18 — Batch 2

**Source**: Cross-spec consistency review after CLR-113–CLR-120 were applied.

---

## CLR-121: Image Proxy In-Memory Cache Size Limit (CRITICAL)

**Spec**: 03-backend-api-specifications.md — BE-API-013

**Issue**: The in-memory cache for proxied images has a 30-day TTL but **no maximum size or entry count limit**. On a 1 GB Cloud Run container, unbounded cache growth could cause OOM crashes. Additionally, the spec says "stream the object bytes directly to the HTTP response **without buffering the entire image in memory**" but also says "implement an in-memory cache (LRU or similar)" — these may seem contradictory (streaming avoids memory buffering, but caching stores the entire image in memory).

**Question**: What constraints should the image proxy cache have?

**Suggested Options**:

- **(A)** Add explicit limits: maximum **200 MB** total cache size with LRU eviction by size (not just by TTL). Maximum single entry size of **10 MB** (reject caching images larger than this threshold; serve them streamed-only). This means streaming is the primary mechanism, and caching supplements it for frequently accessed, reasonably-sized images.
- **(B)** Remove the in-memory cache entirely and rely solely on Cloud Run response caching + Cloud CDN + the `Cache-Control: public, max-age=2592000` header. Simpler, no OOM risk.
- **(C)** Different limits — please specify.

**Answer**:
Let's use B.
---

## CLR-122: Missing Validation Limits for POST /t Fields

**Spec**: 03-backend-api-specifications.md — BE-API-009, Validation section

**Issue**: Several optional fields in the `POST /t` request body have **no maximum length or structure validation** specified:

| Field | Current Validation | Gap |
|---|---|---|
| `referrer` | None specified | No max length |
| `error_type` | "required when action is error_report" | No max length |
| `error_message` | Not mentioned in validation | No max length, not even marked optional |
| `connection_speed` | Not mentioned in validation | No schema validation (keys, value types, ranges) |

Without limits, an attacker could send multi-megabyte strings in these fields, inflating log storage costs and potentially causing processing issues.

**Question**: Should the following validation limits be added?

**Suggested Limits**:

| Field | Proposed Limit |
|---|---|
| `referrer` | Optional, string, max **2000 characters** |
| `error_type` | Required (when action=error_report), string, max **100 characters** |
| `error_message` | Optional (when action=error_report), string, max **2000 characters** |
| `connection_speed` | Optional, object. Allowed keys: `effective_type` (string, max 10 chars, one of: `slow-2g`, `2g`, `3g`, `4g`), `downlink` (number, 0–1000), `rtt` (integer, 0–60000). Unknown keys ignored. |

**Answer**:
Use the suggested limits. Truncate when limits are exceeded (e.g., `referrer` longer than 2000 chars is truncated to 2000 chars before processing).
---

## CLR-123: Geographic Distribution Analytics with Truncated IPs

**Spec**: 07-observability-specifications.md — OBS-006 (Analytics Dashboard), OBS-002 (Privacy Measures)

**Issue**: The Analytics Dashboard lists **"Geographic distribution (from IP)"** as a metric. However, OBS-002 specifies that IP addresses are **truncated** before storage (last octet zeroed for IPv4, last 80 bits zeroed for IPv6). Truncated IPs provide very unreliable geographic resolution — a `/24` block can span multiple cities or even regions.

**Question**: How should geographic analytics be handled given privacy-first IP truncation?

**Suggested Options**:

- **(A)** **Remove the metric** from the Analytics Dashboard. Accept that geographic data is not available with the current privacy model.
- **(B)** **Add a disclaimer** to the Analytics Dashboard spec noting that geographic data is approximate/unreliable and derived from truncated IPs.
- **(C)** **GeoIP lookup before truncation**: The Go backend performs a GeoIP lookup (e.g., using MaxMind GeoLite2) on the full IP, stores **only country and region** in the structured log (no IP or city), and then truncates the IP. This preserves geographic analytics without storing precise IP data.

**Answer**:
Use C. But if it is costly, just go up to country only based on truncated IP.
---

## CLR-124: `visitor_session_id` Storage Inconsistency

**Spec**: 07-observability-specifications.md — OBS-002 vs. 03-backend-api-specifications.md — BE-API-009

**Issue**: OBS-002's "Data Collected" table lists `Visitor session ID` with storage "Structured log → BigQuery". However, BE-API-009's behavior section lists the fields emitted in each structured log entry — and `visitor_session_id` is **not included** in any of the four action types' emitted fields. Only the derived `visitor_id` (the SHA-256 hash) is emitted.

This creates an inconsistency: OBS-002 says `visitor_session_id` is stored, but BE-API-009 doesn't emit it.

**Question**: Which is correct?

**Suggested Options**:

- **(A)** **Add `visitor_session_id` to BE-API-009 emitted fields**: Include the raw `visitor_session_id` alongside `visitor_id` in every structured log entry. This makes OBS-002 accurate but stores more data.
- **(B)** **Remove `visitor_session_id` from OBS-002's data table** (recommended): Since only the non-reversible `visitor_id` hash needs to reach BigQuery for unique visitor counting and bot discrimination, remove the `visitor_session_id` row from OBS-002. This is more privacy-preserving — the raw session ID never leaves the request processing scope.

**Answer**:
Use B. This is more privacy-preserving and aligns with the current implementation.
---

## CLR-125: Embedding Cache Invalidation Procedure

**Spec**: 04-data-model-specifications.md — DM-011

**Issue**: DM-011 states: *"If the model is upgraded, the cache must be invalidated (cleared) to regenerate embeddings with the new model"* and *"If the task type is changed, the cache must be invalidated (cleared)"*. However, **no procedure, runbook, or automated mechanism** is documented for this invalidation. An implementer or operator would not know how to perform it.

**Question**: Should a documented procedure be added? If so, which approach?

**Suggested Options**:

- **(A)** **Manual procedure** documented in a runbook: A simple `db.embedding_cache.deleteMany({})` via `mongosh` or the admin SDK, followed by a note that subsequent search queries will re-populate the cache. Document this in DM-011 under a "Cache Invalidation Procedure" subsection.
- **(B)** **Automated invalidation**: Store the model version (e.g., `gemini-embedding-001`) as a field in each cache document. On startup, the Go backend compares the current model version against cached documents and invalidates mismatches. More complex but fully automated.
- **(C)** **Both**: Document the manual procedure for immediate use, and add a tech debt note to implement automated invalidation later.

**Answer**:
Use B. Automated invalidation is more robust and reduces human error risk. The model version field can be added to the existing cache schema without issue. But also document option A so that it can be done manually.
---

## CLR-126: OBS-001 Log Schema Example Conflates Two Entry Types

**Spec**: 07-observability-specifications.md — OBS-001

**Issue**: The single example log schema in OBS-001 shows **all of the following together**:

- `"severity": "ERROR"` — backend error
- `"method": "GET"`, `"path": "/technical"` — standard backend request (not POST /t)
- `"server_breadcrumbs": [...]` — only present on ERROR entries
- `"log_type": "frontend_tracking"` — only present on POST /t entries

This conflates a standard backend error log (GET request, breadcrumbs) with a frontend tracking log entry (log_type). The notes below the example correctly explain that `log_type` is only for POST /t entries and `server_breadcrumbs` only for ERROR entries, but the example itself is misleading.

**Question**: Should the example be split into separate examples for clarity?

**Suggested Approach**: Replace the single example with 2–3 annotated examples:

1. **Standard backend error log** (GET /technical, severity ERROR, breadcrumbs, no log_type)
2. **Frontend tracking log** (POST /t, severity INFO, log_type: "frontend_tracking", no breadcrumbs)
3. *(Optional)* **Frontend error log** (POST /t, severity INFO, log_type: "frontend_error", no breadcrumbs — since these are INFO-level log entries about reported errors, not actual server errors)

**Answer**:
Follow the suggested approach and split into separate examples for clarity.
---

## CLR-127: Retry Backoff Lacks Jitter (Minor)

**Spec**: 02-frontend-specifications.md — FE-COMP-005-RETRY

**Issue**: The retry delay schedule is fixed at "1s, 2s, 4s, 8s, 16s". Without jitter, multiple clients experiencing the same failure (e.g., backend outage) will retry at identical intervals, creating synchronized retry storms (thundering herd).

**Question**: Should ±20% random jitter be added to the retry delays?

**Suggested Change**: *"Retry delay schedule: 1 second, 2 seconds, 4 seconds, 8 seconds, 16 seconds (doubling each attempt, **with ±20% random jitter applied to each delay**)."*

Example: a 4-second base delay would randomly become 3.2–4.8 seconds.

**Answer**:
Add the jitter suggestion.
---

## CLR-128: OBS-002 "JWT Bearer Token" Phrasing (Minor)

**Spec**: 07-observability-specifications.md — OBS-002, line 169

**Issue**: OBS-002 says *"authenticated via JWT Bearer Token"*. While SEC-003A uses the term "JWT Bearer Token" in its mechanism title, it explicitly clarifies that the JWT is sent **in the request body `token` field** (not as an HTTP `Authorization: Bearer` header) because `navigator.sendBeacon()` does not support custom headers (CLR-103).

The phrase "JWT Bearer Token" in OBS-002 could mislead implementers into expecting an `Authorization` header.

**Question**: Should OBS-002's phrasing be updated?

**Suggested Change**: Replace *"authenticated via JWT Bearer Token"* with *"authenticated via JWT in request body — see SEC-003A"*.

**Answer**:
Phrasing should be updated for OBS-002.
---

## CLR-129: Cloud Functions Node.js Runtime Version (Minor)

**Spec**: 05-infrastructure-specifications.md — INFRA-014

**Issue**: The configuration table specifies `Runtime: Node.js (Cloud Functions Gen 2)` without specifying the Node.js version. Cloud Functions Gen 2 supports Node.js 18, 20, and 22 (as of Feb 2026). Without a specified version, the runtime could vary between deployments, and older versions may reach end-of-life.

**Question**: Which Node.js runtime version should be specified?

**Suggested Options**:

- **(A)** Node.js 22 (current LTS, active support until Oct 2027)
- **(B)** Node.js 20 (maintenance LTS until Apr 2026 — approaching EOL)

**Answer**:
Use Node.js 22 (current LTS) for Cloud Functions Gen 2.

---

## CLR-130: `connection_speed` Nested Field Naming Convention (Minor)

**Spec**: 03-backend-api-specifications.md — BE-API-009 request examples

**Issue**: The `connection_speed` object in request body examples uses `effective_type` (**snake_case**), but the browser Network Information API (`Navigator.connection`) uses `effectiveType` (**camelCase**). The spec doesn't explicitly state whether the frontend must convert field names from the browser API's camelCase to the spec's snake_case before sending.

Current example:
```json
"connection_speed": {
  "effective_type": "4g",
  "downlink": 10.0,
  "rtt": 50
}
```

Browser API returns: `navigator.connection.effectiveType`, `navigator.connection.downlink`, `navigator.connection.rtt`.

**Question**: Should the spec clarify that the frontend converts `effectiveType` → `effective_type`?

**Suggested Options**:

- **(A)** **Keep snake_case** (current examples) and add an explicit note in FE-COMP-004 or BE-API-009 stating the frontend must convert `effectiveType` to `effective_type` before sending. Consistent with the backend's snake_case convention.
- **(B)** **Use camelCase** (`effectiveType`) to match the browser API exactly, avoiding a conversion step. Update BE-API-009 examples accordingly.

**Answer**:
Use A. This maintains consistency with the backend's snake_case convention and ensures clarity for frontend developers.
---

## CLR-131: Missing Infrastructure Acceptance Criteria (Minor)

**Spec**: 05-infrastructure-specifications.md — Acceptance Criteria section

**Issue**: The acceptance criteria section has 10 ACs (AC-INFRA-001 through AC-INFRA-010) but is missing coverage for several infrastructure components:

| Component | INFRA ID | Coverage |
|---|---|---|
| Firestore Native Mode (Vector Search) | INFRA-012 | ❌ No AC |
| Vertex AI Gemini Embedding API | INFRA-013 | ❌ No AC |
| sync-article-embeddings Cloud Function | INFRA-014 | ❌ No AC |
| Artifact Registry | INFRA-018 | ❌ No AC |

**Question**: Should acceptance criteria be added for these components?

**Suggested ACs**:

- **AC-INFRA-011**: Given the Firestore Native Mode database (INFRA-012), when vector search is performed, then the `vector-search` named database exists with vector indexes on `technical_article_vectors`, `blog_article_vectors`, and `others_vectors` collections (2048 dimensions, COSINE distance).
- **AC-INFRA-012**: Given the Vertex AI Embedding API (INFRA-013), when called with a text input and `output_dimensionality=2048`, then it returns a 2048-dimensional L2-normalized embedding vector.
- **AC-INFRA-013**: Given the `sync-article-embeddings` Cloud Function (INFRA-014), when triggered, then it reads articles from Firestore Enterprise, generates embeddings via Vertex AI for new/changed articles, and writes vectors to the corresponding Firestore Native collections.
- **AC-INFRA-014**: Given the Artifact Registry repository (INFRA-018), when the CI/CD pipeline pushes a Docker image, then the image is stored at `asia-southeast1-docker.pkg.dev/<project-id>/website-images/` and Cloud Run can pull from it.

**Answer**:
Add the suggested acceptance criteria for the missing components to ensure complete test coverage of all infrastructure elements.
---

*End of clarifications batch. 11 items total: 1 critical, 5 important, 5 minor.*
