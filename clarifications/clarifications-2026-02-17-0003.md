## Clarifications Needed

The following items require your input to finalize the specifications. These were identified during a comprehensive consistency review of all spec documents after applying the answers from `clarifications-2026-02-17-0002`.

Items marked **🔴 Blocking** need resolution before implementation can begin. Items marked **🟢 Non-blocking** can proceed with reasonable defaults — answer at your convenience.

---

### CLR-025: Sitemap URL in robots.txt Points to Frontend Domain 🔴

**Context**: `07-observability-specifications.md` (OBS-007) declares `Sitemap: https://tjmonserrat.com/sitemap.xml` in `robots.txt`. However, the sitemap is served by the **backend API** at `api.tjmonserrat.com` via `GET /sitemap.xml` (BE-API-011 in `03-backend-api-specifications.md`). There is no mechanism defined for the frontend domain (`tjmonserrat.com`) to serve `/sitemap.xml`.

Search engines fetch sitemaps from the URL declared in `robots.txt`. If that URL returns the SPA shell or a 404, the sitemap will never be indexed.

**Question**: How should the sitemap be accessible on the frontend domain?

**Options**:

- **A**: Change the `robots.txt` sitemap URL to `https://api.tjmonserrat.com/sitemap.xml` (simplest — search engines follow cross-domain sitemap references).
- **B**: Add a Firebase Hosting rewrite rule to proxy `tjmonserrat.com/sitemap.xml` to the backend API endpoint.
- **C**: Other approach.

**Answer**:


---

### CLR-026: Rate Limit Response Headers Impossible with Cloud Armor Architecture 🔴

**Context**: `06-security-specifications.md` (SEC-002) specifies that the backend SHALL include rate limit headers on **every response** (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`). However, CLR-017 decided that real-time rate counting is handled by **Cloud Armor** at the load balancer level, not by the Go application. The Go application only reads **ban status** from Firestore — it has no knowledge of how many requests the client has made in the current window.

The backend **cannot populate `X-RateLimit-Remaining`** because it doesn't track request counts.

**Question**: How should rate limit headers be handled?

**Options**:

- **A**: Remove the rate limit response headers entirely. Cloud Armor sends its own `429` with `Retry-After` when limits are exceeded — that's sufficient.
- **B**: Include only static/informational headers (`X-RateLimit-Limit: 60`) without the `Remaining` count. Add `Retry-After` only on `429` responses.
- **C**: Add lightweight application-level request counting (e.g., check Firestore per request) to populate accurate remaining counts.
- **D**: Other approach.

**Answer**:


---

### CLR-028: Security Headers (CSP) Applied to Backend API Instead of Frontend 🔴

**Context**: `06-security-specifications.md` (SEC-005) states the **backend** SHALL include security headers including `Content-Security-Policy` with `script-src 'self'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonserrat.com`.

CSP directives are only meaningful for **HTML pages** (the frontend). API JSON responses don't execute scripts, load stylesheets, or make sub-requests. Setting CSP on `api.tjmonserrat.com` is unnecessary. Meanwhile, the frontend served by Firebase Hosting has **no CSP specified**.

Additionally, Nuxt 4's SPA build output may inline scripts, which would require `'unsafe-inline'` for `script-src` or a nonce-based CSP strategy.

**Question**: Where should security headers be applied?

**Options**:

- **A**: Move CSP, `X-Frame-Options`, and `Referrer-Policy` to the frontend (Firebase Hosting `firebase.json` headers config). Keep transport-security headers (`X-Content-Type-Options`, `Strict-Transport-Security`) on both frontend and backend.
- **B**: Apply all security headers on both frontend and backend. Adjust CSP for each context (frontend CSP restricts scripts; backend CSP is minimal like `default-src 'none'`).
- **C**: Other approach.

**Answer**:


---

### CLR-029: Granular Rate Limits (60/10/30) Not Enforceable by Current Architecture 🔴

**Context**: `06-security-specifications.md` (SEC-002) defines three distinct rate tiers: regular users (60 req/min), known bots (10 req/min), and tracking endpoint `POST /t` (30 req/min). `05-infrastructure-specifications.md` (INFRA-005) configures a **single** Cloud Armor rate limit rule at **120 req/min per IP**.

A single Cloud Armor rule at 120 req/min does not differentiate between user types or endpoints. The spec doesn't detail how Cloud Armor separates bots from regular users, or whether separate rules exist per path. The practical effect is that all clients get 120 req/min regardless of type.

**Question**: How should the rate limiting tiers be implemented?

**Options**:

- **A**: Simplify to a **single rate limit** (e.g., 60 req/min per IP for all clients) via Cloud Armor. No differentiation between user types or endpoints.
- **B**: Create **multiple Cloud Armor rules** — one for `POST /t` path (30 req/min), one for the general API (60 req/min), and one for known bot User-Agent patterns (10 req/min).
- **C**: Use the single Cloud Armor 120 req/min as a coarse limit, and add application-level rate limiting in the Go backend for the fine-grained tiers.
- **D**: Other approach.

**Answer**:


---

### CLR-030: Rejecting Unknown Query Parameters Breaks Shared URLs 🔴

**Context**: `06-security-specifications.md` (SEC-001) states: "THE SYSTEM SHALL reject requests with unknown or malformed query parameters with HTTP `400`."

When URLs are shared on social media, email campaigns, or analytics tools, they often include UTM parameters (e.g., `?utm_source=twitter&utm_medium=social`). Under this rule, any shared URL with tracking parameters would return a 400 error to the user.

**Question**: How should unknown query parameters be handled?

**Options**:

- **A**: **Ignore** unknown query parameters — the API only reads the parameters it knows about and silently discards the rest. This is the standard approach for public-facing APIs.
- **B**: **Reject** unknown parameters as currently specified (strictest security, but breaks URL sharing).
- **C**: **Allowlist** specific common parameters (UTM, fbclid, gclid, etc.) in addition to the API's own parameters.

**Answer**:


---

### CLR-024: Citation URL Format Contradiction (`.md` Extension) 🟢

**Context**: `02-frontend-specifications.md` (FE-PAGE-003) shows citation URLs with `.md` extension (e.g., `https://tjmonserrat.com/technical/slug.md`). `03-backend-api-specifications.md` (BE-API-003) explicitly states citation URLs SHALL use the frontend route format **without the `.md` extension** (e.g., `https://tjmonserrat.com/technical/article-title-2025-01-15-1030`).

**Question**: Can I update the frontend spec citation examples to remove the `.md` extension, matching the backend spec? The backend generates the citations so it should be authoritative.

**Options**:

- **A**: Yes — update frontend citation examples to match the backend (no `.md` in citation URLs). *(Recommended)*
- **B**: No — keep `.md` in citation URLs and update the backend spec instead.

**Answer**:


---

### CLR-027: Global JSON Content-Type Rule Needs Exception for Sitemap 🟢

**Context**: `03-backend-api-specifications.md` states globally: "All API responses SHALL use `Content-Type: application/json`." But BE-API-011 (`GET /sitemap.xml`) correctly uses `Content-Type: application/xml`, and `GET /health` (BE-API-010) also returns JSON but might not need to be listed. The global rule needs explicit exceptions.

**Question**: Can I add a note to the global response format rule listing exceptions (`GET /sitemap.xml` returns `application/xml`, `GET /robots.txt` returns `text/plain`)?

**Options**:

- **A**: Yes — add exceptions for non-JSON endpoints. *(Recommended)*
- **B**: No — restructure the global rule differently.

**Answer**:


---

### CLR-031: Cache-First Offline Strategy Lacks Efficient Update Detection 🟢

**Context**: `02-frontend-specifications.md` (FE-COMP-008) specifies a cache-first strategy that checks for updates in the background. But the backend has no conditional request support (`ETag`, `If-None-Match`, `Last-Modified`). Without these, the frontend must re-fetch the full article every time just to compare `date_updated`, negating the performance benefit.

**Question**: How should the frontend detect content updates efficiently?

**Options**:

- **A**: Add `ETag` and `Last-Modified` response headers to article detail endpoints (BE-API-003, BE-API-006). The frontend uses `If-None-Match`/`If-Modified-Since` — getting `304 Not Modified` if nothing changed.
- **B**: Use the list endpoint's `date_updated` field to check for updates without fetching the full article.
- **C**: Always re-fetch the full article in the background (simple, but wasteful). Leave optimization as a future improvement.
- **D**: Other approach.

**Answer**:


---

### CLR-032: Missing Privacy Page Specification 🟢

**Context**: `06-security-specifications.md` (SEC-007) states the system SHALL include a privacy notice and comply with data protection regulations. The site collects IP addresses, browser info, and connection data for analytics tracking. However, there is no `FE-PAGE-XXX` for a privacy policy page, no backend endpoint to serve privacy content, and no specification for where the notice is displayed.

**Question**: Should a privacy policy page be added to the specifications?

**Options**:

- **A**: Add a static privacy page (`/privacy`) to the frontend spec. Hardcode the content — no backend endpoint needed.
- **B**: Defer privacy page to post-launch. Add a TODO note to the specs.
- **C**: Add a privacy notice as a footer link/banner rather than a full page.

**Answer**:


---

### CLR-033: No `GET /robots.txt` Endpoint for API Subdomain 🟢

**Context**: `07-observability-specifications.md` (OBS-007) specifies a separate `robots.txt` for `api.tjmonserrat.com` with `Disallow: /`. But there is no `BE-API-XXX` endpoint defined for `GET /robots.txt`. Without a handler, the API would return a 404 JSON response for this path.

**Question**: Can I add a formal `BE-API-012: GET /robots.txt` endpoint to the backend spec that returns the API subdomain's `robots.txt` as `text/plain`?

**Options**:

- **A**: Yes — add the endpoint. *(Recommended)*
- **B**: No — handle this at the infrastructure level (e.g., Cloud Run serving a static file).

**Answer**:


---

### CLR-034: Internal Sitemap Generation Endpoint Needs Formal Spec 🟢

**Context**: `INFRA-008` references `POST /internal/generate-sitemap` as called by Cloud Scheduler, and the backend general notes mention it. However, there is no `BE-API-XXX` entry defining its request/response format, OIDC token validation, error handling, or whether it's publicly accessible through the load balancer.

**Question**: Should this internal endpoint be formally specified?

**Options**:

- **A**: Add a `BE-API-013: POST /internal/generate-sitemap` spec entry with OIDC validation, response format, and note that it's accessible via Cloud Run's service URL (bypassing the load balancer). *(Recommended)*
- **B**: Keep it as a general note — the behavior is sufficiently described in INFRA-008.
- **C**: Add a Cloud Armor rule to block `/internal/*` paths from public traffic, and add a minimal spec entry.

**Answer**:


---

### CLR-035: Error Report Example Uses `.md` in Page URL 🟢

**Context**: `03-backend-api-specifications.md` (BE-API-009) example shows `"page": "/technical/my-article-2025-01-15-1030.md"`. But frontend routes don't use `.md` extensions — the browser URL would be `/technical/my-article-2025-01-15-1030`. The `page` field comes from `window.location` which would never include `.md`.

**Question**: Can I correct the example to remove `.md` from the `page` field?

**Options**:

- **A**: Yes — fix the example. *(Recommended)*
- **B**: No — leave as-is.

**Answer**:


---

### CLR-036: `idx_date_created` Index Purpose Misaligned 🟢

**Context**: `04-data-model-specifications.md` (DM-002) defines an index `idx_date_created` with purpose "Sorting by creation date." CLR-018 clarified that sorting is **only by `date_updated`** — there is no user-facing sort by `date_created`.

**Question**: Should the `idx_date_created` index be kept?

**Options**:

- **A**: Keep the index but update its purpose to "Internal use / content management queries." *(Recommended)*
- **B**: Remove the index entirely since no user-facing feature uses `date_created` sorting.
- **C**: Other.

**Answer**:


---

### CLR-037: `/health` Endpoint Exposed Through Load Balancer 🟢

**Context**: BE-API-010 states the health endpoint "SHALL be accessible only through Cloud Run's internal health check mechanism (not exposed to public traffic via the Load Balancer)." But INFRA-004's URL map routes `api.tjmonserrat.com/*` to Cloud Run, which includes `/health`. No mechanism is specified to block public access.

**Question**: How should `/health` be protected from public access?

**Options**:

- **A**: Add a Cloud Armor rule to block `GET /health` from external traffic (returning 403). *(Recommended)*
- **B**: Accept that `/health` is publicly accessible — it only reveals that the service is running, which is low risk.
- **C**: Application-level check (inspect request source headers to verify it's from the Cloud Run health checker).

**Answer**:


---

### CLR-038: Frontend Error Snackbar Missing HTTP 403 Mapping 🟢

**Context**: `02-frontend-specifications.md` (FE-COMP-003) maps error messages for HTTP 400, 404, 405, 429, 503, and a generic catch-all. HTTP 403 is not explicitly mapped. SEC-002 progressive banning returns 403 for 30-day bans with a `blocked_until` field.

**Question**: Should a specific 403 error message be added?

**Options**:

- **A**: Add a vague 403 message like "Something went wrong. Please try again later." — don't reveal the ban mechanism.
- **B**: Add a clear 403 message like "Access to this site has been temporarily restricted." — transparent but informs the user.
- **C**: Use the existing catch-all — no specific 403 mapping needed.

**Answer**:


---

### CLR-039: Cloud Run 504 Timeout Not in Error Response Strategy 🟢

**Context**: INFRA-003 sets a Cloud Run request timeout of 30 seconds. If exceeded, Cloud Run returns HTTP 504 Gateway Timeout. But the backend error response strategy only lists 200, 400, 403, 404, 405, 429, 503 as possible codes. HTTP 504 is infrastructure-generated, not application-generated, and the frontend error snackbar has no mapping for it.

**Question**: Should 504 be documented as a possible infrastructure-generated response?

**Options**:

- **A**: Add 504 to the error response strategy as an infrastructure-generated code, and add a frontend snackbar mapping (e.g., "The server is taking too long. Please try again."). *(Recommended)*
- **B**: Leave as-is — the catch-all handles it, and 504s should be extremely rare.

**Answer**:


---

### CLR-040: `others` Collection Slugs Have No Detail Endpoint 🟢

**Context**: `04-data-model-specifications.md` (DM-005) defines a `slug` field with a unique index for the `others` collection, and the API returns slugs in list responses. However, there is no `GET /others/{slug}.md` detail endpoint — items link directly to external URLs. The slug serves no apparent purpose from the API consumer's perspective.

**Question**: What is the purpose of the `slug` field in the `others` collection?

**Options**:

- **A**: Keep slugs as internal document identifiers but **exclude them from the API response** to reduce payload.
- **B**: Keep slugs in the API response — they may be useful for future detail pages or deep linking.
- **C**: Remove the `slug` field and unique index entirely — use MongoDB's `_id` as the document identifier.

**Answer**:

