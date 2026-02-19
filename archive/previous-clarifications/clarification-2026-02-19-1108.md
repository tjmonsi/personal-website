# Clarifications — 2026-02-19 (HKT 11:08)

Post-Fiber migration full specification review. All 7 spec files reviewed for inconsistencies, ambiguities, and items needing owner input.

---

## CLR-193: BE-BREADCRUMB Requirements Text — Generic `context.Context` vs Fiber User Context

**Spec**: [03-backend-api-specifications.md](../specs/03-backend-api-specifications.md) — BE-BREADCRUMB section (lines ~66–67)

**Issue**: The BE-BREADCRUMB requirements text still uses generic Go `context.Context` language:
- *"THE SYSTEM SHALL create a new breadcrumb collector at the start of every incoming HTTP request, scoped to that request's `context.Context`."*
- *"THE SYSTEM SHALL propagate the collector through Go's `context.Context` so that any function in the request call chain can record breadcrumbs without passing an additional parameter."*

However, the acceptance criteria (AC-API-019 through AC-API-022) reference Fiber-specific patterns: `c.SetUserContext()` / `c.UserContext()`.

**Context**: Fiber's `c.UserContext()` returns a standard `context.Context`, so the statement is technically accurate. But a reader might not understand the connection between the generic `context.Context` language in the requirements and the Fiber-specific ACs.

**Options**:
- (A) Update the requirements text to explicitly reference Fiber's user context pattern (e.g., "scoped to the Fiber request's user context via `c.SetUserContext()` / `c.UserContext()`")
- (B) Keep the current wording as-is, since `context.Context` is technically correct and the ACs already clarify the Fiber implementation pattern
- (C) Add a note under the requirements referencing the Fiber-specific implementation: "In the Go Fiber framework, the request context is managed via `c.SetUserContext()` and retrieved via `c.UserContext()`."

**Answer**:
Use A — update the requirements text to explicitly reference Fiber's user context pattern for clarity and consistency with the ACs.
---

## CLR-194: OBS-001 Server Breadcrumbs Note — Generic `context.Context` Reference

**Spec**: [07-observability-specifications.md](../specs/07-observability-specifications.md) — OBS-001 `server_breadcrumbs` note (around line 162)

**Issue**: The note states: *"The `server_breadcrumbs` field... contains an array of timestamped processing steps recorded via Go `context.Context` during request handling."*

Same situation as CLR-193 — the language is technically accurate but doesn't mention Fiber's user context pattern.

**Options**:
- (A) Update to reference Fiber's user context explicitly
- (B) Keep as-is (consistent with whatever decision is made for CLR-193)

**Answer**:
Use A — update the note to explicitly reference Fiber's user context pattern for clarity and consistency with CLR-193.

---

## CLR-195: Dockerfile Go Version — `golang:1.23-alpine`

**Spec**: [05-infrastructure-specifications.md](../specs/05-infrastructure-specifications.md) — INFRA-003 Docker Image

**Issue**: The Dockerfile example pins the Go builder image to `golang:1.23-alpine`. By the time implementation begins, Go 1.23 may not be the latest stable version (Go 1.24+ may be available).

**Options**:
- (A) Pin to `golang:1.23-alpine` as specified — update only when a deliberate Go version upgrade is planned
- (B) Change to "latest stable Go version at build time" (e.g., `golang:1.24-alpine` or whatever is current during implementation)
- (C) Use a range notation like `golang:1.23+-alpine` to indicate "1.23 or later"

**Answer**:
Use B — update to the latest stable Go version at build time to ensure the application benefits from the latest features and security patches, while noting that the spec can be updated as new Go versions are released.
---

## CLR-196: Ban Status Check on Every Request — Firestore Read Performance

**Spec**: [06-security-specifications.md](../specs/06-security-specifications.md) — SEC-002 Progressive Banning

**Issue**: The spec states the Go application checks Firestore for ban status on each `GET` and `POST` request. For a scale-to-zero Cloud Run service, this adds a Firestore read to every single request. On cold starts especially, this compounds the latency.

**Question**: Should the application implement a short-lived in-memory cache for ban status lookups (e.g., cache a "not banned" result for 60 seconds per IP, or use a small LRU cache) to reduce Firestore reads? Or is the per-request Firestore check acceptable given the expected low traffic volume of a personal website?

**Trade-off**:
- **Without cache**: Every request pays the Firestore latency (~5–20ms). Simpler implementation, always current.
- **With cache**: First request per IP pays Firestore latency; subsequent requests within the cache TTL are free. Slight delay (up to TTL duration) before a newly applied ban takes effect. Adds cache eviction/sizing complexity.

**Options**:
- (A) No cache — per-request Firestore check is acceptable for this traffic volume
- (B) Add a short-lived in-memory LRU cache (e.g., 60-second TTL, max 1000 entries) for ban lookups
- (C) Defer decision to implementation — document that caching is an optimization that can be added if latency proves problematic

**Answer**:
Use B — implement a short-lived in-memory cache for ban status lookups to improve performance while accepting the trade-off of a potential delay in ban enforcement. This is a common pattern for rate-limiting and banning systems to reduce database load while maintaining reasonable responsiveness.
---

## CLR-197: `process-rate-limit-logs` — Race Condition Between Atomic Increment and Ban Evaluation

**Spec**: [05-infrastructure-specifications.md](../specs/05-infrastructure-specifications.md) — INFRA-008c Processing Logic

**Issue**: Step 4 uses MongoDB atomic `$inc` / `$push` to increment offense count and append to offense history. Step 5 evaluates progressive banning thresholds. If multiple Cloud Function instances process concurrent Pub/Sub messages for the same IP:
1. Instance A increments offense_count from 4 → 5 (atomically).
2. Instance B increments offense_count from 5 → 6 (atomically).
3. Both instances then read the document to evaluate thresholds.
4. Both see ≥ 5 offenses and both attempt to apply the 30-day ban.

The result is functionally correct (ban is applied), but there's redundant processing. More importantly, if the increment and threshold evaluation are not in the same atomic operation, there's a window where the count is updated but the ban hasn't been applied yet.

**Question**: Should the spec require using `findOneAndUpdate` with `returnDocument: "after"` so that the increment and the evaluation happen based on the same document state? Or is the current approach acceptable given the low probability of concurrent rate-limit events for the same IP?

**Options**:
- (A) Current approach is acceptable — concurrent processing is rare and the result is idempotent
- (B) Specify `findOneAndUpdate` with `returnDocument: "after"` to ensure the ban evaluation uses the post-increment document, eliminating the race
- (C) Note this as an implementation consideration rather than a spec requirement

**Answer**: Use B — specify `findOneAndUpdate` with `returnDocument: "after"` to ensure the ban evaluation uses the post-increment document, eliminating the race condition.

---

## CLR-198: VPC `/28` Subnet Sizing — 16 IP Addresses

**Spec**: [05-infrastructure-specifications.md](../specs/05-infrastructure-specifications.md) — INFRA-009 VPC

**Issue**: The `/28` subnet provides 16 IP addresses. Current max concurrency: Cloud Run max 5 instances + up to 4 Cloud Function types (each potentially multi-instance). The spec notes "sufficient for the expected maximum concurrency" but the headroom is tight.

**Question**: If Cloud Run max instances are increased in the future, or if multiple Cloud Function instances scale simultaneously, 16 IPs could become a constraint. Should a `/27` (32 IPs) or `/26` (64 IPs) be used instead for more headroom? (GCP reserves some addresses per subnet, reducing usable IPs further.)

**Options**:
- (A) Keep `/28` (16 IPs) — sufficient for current spec. Monitor and expand if needed.
- (B) Use `/27` (32 IPs) — modest headroom increase with negligible cost difference
- (C) Use `/26` (64 IPs) — generous headroom for future scaling

**Answer**: Use B — use a `/27` subnet to provide more headroom for scaling while keeping costs low and avoiding the tight constraints of a `/28`.

---

## CLR-199: Category Cache in `sessionStorage` — 24-Hour TTL vs Tab Lifetime

**Spec**: [02-frontend-specifications.md](../specs/02-frontend-specifications.md) — FE-COMP-007 Category Caching

**Issue**: The spec says "IF cached categories exist and are less than 24 hours old, THE SYSTEM SHALL use the cached list." However, `sessionStorage` is destroyed when the browser tab is closed. A user who opens a new tab always starts with an empty cache, making the 24-hour TTL check irrelevant in practice — it only applies if the user keeps the same tab open for more than 24 hours.

**Question**: Is `sessionStorage` the intended storage mechanism? If the goal is to avoid refetching categories on every page navigation within a session, `sessionStorage` works fine (the 24-hour TTL is just a safety net for long-running tabs). But if the goal is to cache across tab opens, `localStorage` would be more appropriate.

**Options**:
- (A) Keep `sessionStorage` — the 24-hour TTL is a safety net for long-running tabs. A fresh fetch on new tabs is acceptable.
- (B) Switch to `localStorage` — enables the 24-hour cache to persist across tab opens and browser restarts
- (C) Keep `sessionStorage` but clarify the intent in the spec (the 24-hour check only matters for tabs open longer than 24 hours)

**Answer**: Use B — switch to `localStorage` to allow the category cache to persist across tab opens, making the 24-hour TTL meaningful for all users regardless of tab usage patterns.

---

## CLR-200: 429 Error Message — "30 Minutes" vs `retry_after: 30` Seconds

**Spec**: [02-frontend-specifications.md](../specs/02-frontend-specifications.md) — FE-COMP-003 Error Snackbar + [06-security-specifications.md](../specs/06-security-specifications.md) — SEC-002

**Issue**: There is a disconnect between the user-facing 429 message and the technical retry period:
- **FE-COMP-003** (429 message): *"Take a breather and try again in about 30 minutes."*
- **SEC-002** (rate limit response): `"retry_after": 30` (30 seconds)
- **Cloud Armor** rate limit window: 60 requests per minute (1-minute window)

The user is told to wait 30 minutes, but the actual rate limit window resets in about 30 seconds to 1 minute. The 30-minute message may be intentionally inflated to discourage rapid retrying and reduce offense accumulation (since 5 offenses within 7 days triggers a 30-day ban). Or it may be a mistake.

**Question**: Is the "30 minutes" message intentionally longer than the actual retry period, or should it align with the `retry_after` value?

**Options**:
- (A) Intentional — keep "30 minutes" to discourage aggressive retrying and prevent offense accumulation leading to progressive bans
- (B) Align with reality — change the message to say "about 30 seconds" or "about a minute" to match the actual `retry_after` period
- (C) Use a middle ground — something like "a few minutes" that's less specific but not misleadingly long

**Answer**: Use Option A. The "30 minutes" message is intentionally longer than the actual retry period to discourage users from rapidly retrying and accumulating offenses that could lead to progressive bans. This approach prioritizes user education and long-term behavior change over precise technical accuracy in the error message.

---

## CLR-201: FE-PAGE-001 — "Render the markdown content returned by the API"

**Spec**: [02-frontend-specifications.md](../specs/02-frontend-specifications.md) — FE-PAGE-001

**Issue**: FE-PAGE-001 states: *"THE SYSTEM SHALL render the markdown content returned by the API."* However, BE-API-001 (`GET /`) returns JSON (like all other endpoints). The DM-001 frontpage collection has fields like `hero_text`, `hero_subtext`, `about_text`, `about_image_url`, `resume_url`.

**Question**: Does BE-API-001 return JSON with markdown-formatted text fields (e.g., `about_text` contains markdown) that the frontend extracts and renders? Or does it return raw markdown like the article detail endpoints (BE-API-003, BE-API-005)? The current phrasing is ambiguous.

**Options**:
- (A) BE-API-001 returns JSON with markdown-formatted string fields — the frontend extracts each field and renders markdown where applicable. Clarify the response schema.
- (B) BE-API-001 returns raw markdown (like BE-API-003/005) — update the spec to clarify this is a `text/markdown` response
- (C) Other — please specify the intended response format

**Answer**: Use B — update the spec to clarify that BE-API-001 returns raw markdown (similar to BE-API-003 and BE-API-005) and that the frontend should render it accordingly. This simplifies the API response and keeps it consistent with how article content is delivered.

---

## CLR-202: Citation Format — Author Name Inconsistency

**Spec**: [02-frontend-specifications.md](../specs/02-frontend-specifications.md) — FE-PAGE-003 Citation Section

**Issue**: The citation formats use different author name styles:
- **APA**: `Monserrat, T.J.` (with periods after initials)
- **MLA**: `Monserrat, TJ.` (no periods, trailing period after name)
- **Chicago**: `Monserrat, TJ.` (no periods, trailing period)
- **IEEE**: `T.J. Monserrat` (with periods, first-last order)

**Question**: Are these differences intentional (each citation format has its own convention for initials)? APA and IEEE typically use periods with initials; MLA and Chicago vary. Want to confirm these are correct per each format's standard.

**Options**:
- (A) Correct as-is — each format follows its own convention
- (B) Standardize to "T.J." (with periods) across all formats
- (C) Needs review — adjust to match the latest edition of each citation style guide

**Answer**: Use B — standardize to "T.J." (with periods) across all formats for consistency and to avoid confusion. While each citation style has its own rules, using periods for initials is common and helps maintain a consistent author name presentation across all formats in the application.

---

## CLR-203: FE-PAGE-007 (`/others`) — Displayed Fields in List Items

**Spec**: [02-frontend-specifications.md](../specs/02-frontend-specifications.md) — FE-PAGE-007

**Issue**: FE-PAGE-007 list items display: Title, Date, Abstract. But the API response (BE-API-007) also returns `category`, `tags`, `slug`, and `external_url`. Unlike FE-PAGE-002 (technical) and FE-PAGE-004 (blog) which both display an offline indicator, `/others` does not — presumably because items link to external URLs and cannot be cached offline.

**Question**: Should the `/others` list items also display `category` and/or `tags` alongside title, date, and abstract? These fields are available in the API response and could help users filter visually. Or should the list remain minimal (title, date, abstract only)?

**Options**:
- (A) Keep minimal — title, date, abstract is sufficient for /others. Filters are available for narrowing results.
- (B) Add category display to each list item (small label/badge)
- (C) Add both category and tags to each list item (matching the technical/blog list layout)

**Answer**: Use B — add category display to each list item as a small label or badge. This provides additional context for each item without overwhelming the list layout. Tags can be reserved for the detail page where there is more space to display them effectively.
