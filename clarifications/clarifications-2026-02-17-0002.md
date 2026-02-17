## Clarifications Needed

The following items require your input to finalize the specifications. These were identified during a comprehensive review of all spec documents after applying the answers from `clarifications-2026-02-17-0001`.

---

### CLR-016: Empty Search Results — HTTP Status Code

**Context**: The current spec (BE-API-005, 06-security-specifications.md) states that requests returning no results should return `404 Not Found`. However, a search query that matches zero articles is not the same as requesting a resource that does not exist. Returning `404` for empty search results can confuse frontend developers and API consumers, since `404` is also used for invalid endpoints and progressive banning.

**Question**: Should article list endpoints (`GET /articles/technical`, `GET /articles/blog`) return `200` with an empty array when no articles match the query, or keep the current `404` behavior?

**Options**:

- **A**: Return `200` with `{ "articles": [], "pagination": { "total": 0, ... } }` when query returns no results. Reserve `404` for truly missing resources (e.g., invalid slug in detail endpoint).
- **B**: Keep `404` for empty results as currently specified.

---

### CLR-017: Rate Limiting State Storage

**Context**: The spec defines per-IP rate limiting (60/10/30 req/min) with progressive banning (offenses 1–5 → 429, then → 403, then → 404). Cloud Run auto-scales instances and can spin up/down at any time. In-memory rate limit state would be lost when instances scale down and would not be shared across instances.

**Question**: Where should rate limiting state be stored?

**Options**:

- **A**: Use **Cloud Armor** for rate limiting enforcement (handles per-IP tracking at the load balancer level, no application-level state needed). Progressive banning offender records still stored in Firestore (`rate_limit_offenders` collection).
- **B**: Use **Redis (Memorystore)** for real-time rate counters shared across instances. Progressive banning offender records in Firestore.
- **C**: Use **Firestore** for both rate counters and offender records (simpler stack, but adds latency per request).
- **D**: Other approach.

---

### CLR-018: Default Sort Order for Article Listings

**Context**: The article list endpoints (`BE-API-005`) accept `sort_by` and `sort_order` query parameters, but the specs do not define default values when these are omitted.

**Question**: What should the default sort behavior be?

**Options**:

- **A**: Default sort by `date_updated` descending (most recently updated first).
- **B**: Default sort by `date_created` descending (newest first).
- **C**: Other field or order.

---

### CLR-019: Tag Filter Matching Logic

**Context**: The article list endpoint accepts a `tags` query parameter for filtering. The specs do not define whether multiple tags should use AND logic (article must have ALL specified tags) or OR logic (article must have ANY of the specified tags).

**Question**: When filtering by multiple tags, which logic should be used?

**Options**:

- **A**: **AND** — articles must contain ALL specified tags.
- **B**: **OR** — articles must contain at least ONE of the specified tags.
- **C**: Let the user choose (add a `tag_match` parameter with values `all` or `any`).

---

### CLR-020: Sitemap Generation

**Context**: A `sitemap.xml` is referenced in the robots.txt spec (OBS-007) but no strategy is defined for how it gets generated or served. Since articles are stored in a database and can be added/updated at any time, the sitemap needs a generation mechanism.

**Question**: How should `sitemap.xml` be generated?

**Options**:

- **A**: **Build-time static generation** — sitemap is rebuilt and deployed whenever the content Git repo CI/CD runs.
- **B**: **Dynamic endpoint** — backend serves `/sitemap.xml` by querying the database on each request (with caching).
- **C**: **Scheduled generation** — a Cloud Scheduler job periodically regenerates the sitemap and stores it in Firebase Hosting / Cloud Storage.
- **D**: **No sitemap needed** for now — defer to a later phase.

---

### CLR-021: JWT Clock Skew Tolerance

**Context**: The POST `/t` endpoint uses HMAC-SHA256 JWTs with a 5-minute expiry (`exp` claim). The spec does not define a clock skew tolerance. If the client's clock is slightly ahead or behind the server's clock, valid tokens could be rejected.

**Question**: What clock skew tolerance should the backend allow when validating JWT `exp` and `iat` claims?

**Options**:

- **A**: **30 seconds** — standard tolerance for well-synchronized systems.
- **B**: **60 seconds** — more lenient for users with slightly off clocks.
- **C**: **0 seconds** — strict validation, no tolerance.

---

### CLR-022: Naming Consistency — "Others" vs "Other Links"

**Context**: The specs refer to a page/collection variously as "Others," "Other Links," and "other_links." The frontend page is `/others`, the data model collection is `other_links` (DM-005), and the API endpoint is `/others` (BE-API-007). The mixed naming may cause confusion.

**Question**: Should the naming be standardized, and if so, to what?

**Options**:

- **A**: Keep as-is. Page route is `/others`, collection is `other_links`, endpoint is `/others`. Accept the inconsistency.
- **B**: Rename the collection to `others` to match the route and endpoint.
- **C**: Rename the route and endpoint to `/other-links` to match the collection.

---

### CLR-023: Offline Storage Mechanism

**Context**: The offline reading spec (FE-COMP-008) mentions both Cache API and IndexedDB but does not specify which is used for what. Both have different strengths:
- **Cache API**: Best for caching HTTP responses (article HTML/markdown).
- **IndexedDB**: Best for structured data (article metadata, reading progress).

**Question**: Should the spec define the specific storage mechanism, or leave it as an implementation detail?

**Options**:

- **A**: Define it now: **Cache API** for article content responses, **IndexedDB** for metadata and reading state.
- **B**: Leave as implementation detail — the spec only requires offline reading to work via service worker. The developer decides the storage mechanism.
