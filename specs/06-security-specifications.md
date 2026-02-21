---
title: Security Specifications
version: 4.0
date_created: 2026-02-17
last_updated: 2026-02-21
owner: TJ Monserrat
tags: [security, rate-limiting, cors, authentication, vector-search, terraform, iac, wif, service-accounts]
---

## Security Specifications

### Threat Model Summary

| Threat                        | Mitigation                                            | Layer          |
| ----------------------------- | ----------------------------------------------------- | -------------- |
| DDoS attacks                  | Cloud Armor + rate limiting                           | Infrastructure |
| SQL/NoSQL injection           | Parameterized queries, input validation               | Application    |
| XSS                           | Content Security Policy, output encoding              | Application    |
| Path traversal                | Slug pattern validation, no file system access        | Application    |
| Information disclosure        | Never return 500, sanitize error responses            | Application    |
| Brute-force API abuse         | Progressive rate limiting and banning                 | Application    |
| Bot abuse                     | robots.txt, rate limiting, Cloud Armor bot management | Both           |
| Unauthorized tracking abuse   | JWT authentication via request body `token` field on POST /t (CLR-109) | Application    |
| Analytics data exposure       | Least-privilege service account with read-only BigQuery access | Infrastructure |
| Embedding API abuse           | Embedding cache prevents redundant Gemini API calls; rate limiting on search endpoints | Application + Infrastructure |
| Vector data exfiltration       | Firestore Native IAM; vectors contain no readable text         | Infrastructure |
| Compromised CI/CD credentials  | Workload Identity Federation (keyless OIDC auth, no service account keys) | Infrastructure |
| IaC state/credential exposure   | GCS bucket with versioning, no public access; WIF for CI/CD (no SA keys in pipeline); least-privilege IAM | Infrastructure |

---

### SEC-001: Input Validation

**Search Query Validation**:

- THE SYSTEM SHALL reject search queries exceeding 300 characters with HTTP `400`.
- THE SYSTEM SHALL sanitize search queries to prevent injection attacks.
- THE SYSTEM SHALL strip or escape special characters that could be interpreted as query operators.

**Slug Validation**:

- THE SYSTEM SHALL validate article slugs against the prescribed regex pattern.
- THE SYSTEM SHALL reject any slug not matching the pattern with HTTP `404`.
- Regex pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}\.md$`
  - Example: `my-first-article-2025-01-15-1030.md`

**Query Parameter Validation**:

- THE SYSTEM SHALL validate `page` as a positive integer.
- THE SYSTEM SHALL validate `date_from` and `date_to` as valid ISO 8601 dates.
- THE SYSTEM SHALL validate `category` as a non-empty string.
- THE SYSTEM SHALL validate `tags` as a comma-separated list of alphanumeric strings with hyphens.
- Tags array: maximum **10** items per request. THE SYSTEM SHALL reject requests with more than 10 tags with HTTP `400`. (CLR-160)
- THE SYSTEM SHALL reject requests with unknown or malformed query parameters with HTTP `400`.

> **Note**: This strict query parameter validation is intentional. The backend API (`api.tjmonsi.com`) is not the URL shared on social media — the frontend URL (`tjmonsi.com`) is. UTM parameters and other tracking query strings on the frontend URL are handled client-side and forwarded to the backend via `POST /t` as tracking data. The backend API expects a strict one-to-one mapping of query parameters for search and pagination purposes only.

**POST /t Breadcrumbs Validation** (CLR-114):

- THE SYSTEM SHALL validate the `breadcrumbs` field in `POST /t` error report payloads with the following constraints:
  - `breadcrumbs`: optional array, maximum **50 entries**.
  - Each entry SHALL be an object with the following fields:
    - `timestamp` (string, required): ISO 8601 format. THE SYSTEM SHALL reject entries with non-ISO 8601 timestamps.
    - `type` (string, required): Maximum 50 characters.
    - `message` (string, required): Maximum 300 characters.
    - `data` (object, optional): Maximum 500 bytes when serialized to JSON. Values SHALL be strings or numbers only.
  - Total serialized `breadcrumbs` payload size: maximum **50 KB**.
- IF the `breadcrumbs` array contains more than 50 entries, THE SYSTEM SHALL truncate to the **last 50 entries** (keep newest) and log a WARNING. THE SYSTEM SHALL NOT reject the request.
- IF an individual breadcrumb entry has fields exceeding the size limits, THE SYSTEM SHALL truncate the field values to the maximum allowed length.
- IF `breadcrumbs` is provided for actions other than `error_report`, THE SYSTEM SHALL ignore the field.
- Breadcrumbs validation is enforced server-side to prevent memory exhaustion from oversized payloads. Client-side limits (FE-COMP-013) are a first line of defense but cannot be trusted.

---

### SEC-002: Rate Limiting

#### Rate Limiting Architecture

- **Real-time rate counting**: Enforced by **Google Cloud Armor** at the load balancer level (see INFRA-005 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)). Cloud Armor handles per-IP request counting and throttling. This avoids the need for in-memory rate counters in the Go application, which would be lost when Cloud Run instances scale down or restart.
- **Offense tracking via log sink**: A **Cloud Logging log sink** routes Cloud Armor rate-limit (429) events to a **Cloud Function** (`process-rate-limit-logs`, INFRA-008c). The Cloud Function writes offense records to the `rate_limit_offenders` Firestore collection (DM-009). This bridges the gap between Cloud Armor (which blocks requests before they reach Cloud Run) and the application's progressive banning logic.
- **Progressive banning state**: Stored in **Firestore** (`rate_limit_offenders` collection, DM-009). The `process-rate-limit-logs` Cloud Function (INFRA-008c) evaluates offense counts against progressive ban tier thresholds and writes the `current_ban` field when a ban threshold is met. The Go application reads the `current_ban` field on each `GET` and `POST` request and enforces the appropriate response code (429, 403, or 404). The Go application does NOT evaluate ban tiers or write ban state — it is a read-only ban checker. `OPTIONS` preflight requests are exempt from ban checks (consistent with rate-limiting exemption) to avoid CORS preflight failures for banned clients.
- **Ban status caching**: The Go application SHALL maintain a short-lived **in-memory LRU cache** for ban status lookups to reduce Firestore reads. Cache parameters: **60-second TTL**, **maximum 1000 entries**, keyed by client IP. On cache miss, the application reads from Firestore and populates the cache. A newly applied ban may take up to 60 seconds to take effect for already-cached "not banned" IPs. Similarly, an expired ban may continue to be enforced for up to 60 seconds for IPs whose "banned" status is still cached. Both directions of propagation delay are accepted trade-offs for avoiding per-request Firestore lookups. (CLR-196)
- **Adaptive protection**: **Cloud Armor Adaptive Protection** is enabled to provide ML-based DDoS detection with recommended rules that the owner can review and apply, complementing the application-level progressive banning (see INFRA-005).
- This multi-layer approach ensures rate limiting works correctly across Cloud Run auto-scaling events while keeping the infrastructure simple (no Redis/Memorystore needed).

#### Application-Level Rate Limiting

**Rate Limit** (per IP address):

| Scope           | Limit                         | Window     |
| --------------- | ----------------------------- | ---------- |
| All clients (all endpoints) | 60 req/min    | Per minute |

- A single rate limit of **60 requests per minute per IP** is enforced by Cloud Armor at the load balancer level for all clients and all endpoints. No differentiation between user types, bot tiers, or endpoint-specific limits.

**Rationale**:
- **60 req/min**: Generous enough to accommodate shared NAT/ISP IPs where multiple users share a single public IP, while still providing effective protection against abuse. Based on common web application rate limiting practices. Reference: [OWASP Rate Limiting Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html)

> **Note**: A single IP may represent multiple users (shared ISP NAT). This limit is set to be generous enough to accommodate this while still providing protection.

**Rate Limit Headers**:

- THE SYSTEM SHALL NOT include rate limit counting headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) on responses. Rate counting is handled by Cloud Armor at the load balancer level, and the Go application has no knowledge of per-request counts.
- Cloud Armor sends its own `429 Too Many Requests` response with `Retry-After` header when limits are exceeded.

**Rate limit response** (`429`):

- Cloud Armor returns the `429` response directly at the load balancer level.
- A [custom error response](https://cloud.google.com/armor/docs/custom-error-responses) is configured in Cloud Armor (INFRA-005) to return the following JSON body:

```json
{
  "error": "Too many requests. Please try again later.",
  "retry_after": 30
}
```

- Note: The frontend (FE-COMP-003) matches 429 responses by HTTP status code only and does not depend on the response body format. This ensures graceful handling even if Cloud Armor returns a non-JSON response.

---

#### Progressive Banning

WHEN a client exceeds the rate limit, THE SYSTEM SHALL track it as an "offense."

| Condition                                         | Action                      | Response Code |
| ------------------------------------------------- | --------------------------- | ------------- |
| Rate limit exceeded (offenses 1–5)                 | Return rate limit response  | `429`         |
| 5 offenses within 7 days                          | Block for 30 days           | `403`         |
| 2 offenses within 7 days after 30-day ban expires | Block for 90 days           | `404`         |
| 2 offenses within 7 days after 90-day ban expires | Block indefinitely          | `404`         |

**Offense Counting Algorithm — Rolling 7-Day Window**:

The `process-rate-limit-logs` Cloud Function (INFRA-008c) SHALL use the following algorithm when evaluating progressive ban tiers after each offense increment:

1. **Rolling window**: Count only offenses with timestamps within the last 7 days from the current offense timestamp. Offenses older than 7 days are excluded from tier evaluation.
2. **Initial ban evaluation**: If the offender has no `current_ban` (or `current_ban` is null), count offenses in the rolling 7-day window. If the count reaches 5, apply a 30-day ban.
3. **Post-ban escalation**: After any ban expires (ban `end` date is in the past), the "post-ban offense window" starts from `current_ban.end`. Count only offenses with timestamps after `current_ban.end` and within the rolling 7-day window.
   - After a 30-day ban expires: 2 offenses within 7 days of each other (both after ban expiry) → 90-day ban.
   - After a 90-day ban expires: 2 offenses within 7 days of each other (both after ban expiry) → indefinite ban.
4. **Active ban**: If `current_ban` is active (not expired), no tier evaluation is needed — the existing ban is still in effect. The offense is still recorded (step 4 in INFRA-008c processing logic) for historical tracking.

**Blocking behavior**:

- WHEN a client with 1–5 offenses exceeds the rate limit, THE SYSTEM SHALL return HTTP `429 Too Many Requests`.
- WHEN a client blocked for 30 days makes a request, THE SYSTEM SHALL return HTTP `403 Forbidden`.
- WHEN a client blocked for 90 days or indefinitely makes a request, THE SYSTEM SHALL return HTTP `404 Not Found` (maximum opacity — attacker does not know they are banned).
- THE SYSTEM SHALL include when the ban expires (if applicable) in `403` responses only.
- Bans are tracked in the `rate_limit_offenders` database collection (see DM-009).

**Ban response for 429** (rate limit exceeded):

```json
{
  "error": "Too many requests. Please try again later.",
  "retry_after": 30
}
```

> **Note**: The `retry_after` value is 30 seconds, but the user-facing message in FE-COMP-003 says "30 minutes." This is intentional — the inflated message discourages aggressive retrying that could accumulate offenses leading to progressive bans. (CLR-200)

**Ban response for 403** (30-day block):

```json
{
  "error": "Access temporarily restricted.",
  "blocked_until": "2026-03-19T00:00:00Z"
}
```

**Ban response for 404** (90-day / indefinite block):

Standard 404 response — indistinguishable from a normal "not found" to the client.

**Manual Review of Indefinite Bans**:

- To lift an indefinite ban, the site owner SHALL delete the corresponding document from the `rate_limit_offenders` collection (DM-009) in Firestore Enterprise. The Go backend's LRU cache will expire the stale "banned" entry within 60 seconds (the cache TTL), after which the IP will be unblocked.
- No automated review process is provided. The site owner may periodically review indefinite bans by querying the `rate_limit_offenders` collection for documents where `current_ban.type = "indefinite"`. This is a manual operational task performed via the Firebase Console or a `mongosh` session against the Firestore Enterprise MongoDB-compatible endpoint.

---

#### Bot Management

- THE SYSTEM SHALL rely on `robots.txt` crawl-delay directives to manage bot crawl rates (see OBS-007 in [07-observability-specifications.md](07-observability-specifications.md)).
- THE SYSTEM SHALL NOT apply separate rate limits to bots — the single 60 req/min rate limit applies to all clients equally.
- Cloud Armor's built-in bot management capabilities MAY be used for additional protection against malicious bots.

---

### SEC-003: HTTP Method Enforcement

- THE SYSTEM SHALL accept `GET` requests on all endpoints except `POST /t`.
- THE SYSTEM SHALL accept `POST` requests only on the `/t` endpoint.
- THE SYSTEM SHALL accept `OPTIONS` requests on all endpoints for CORS preflight handling. `OPTIONS` requests SHALL return appropriate CORS headers and SHALL NOT be subject to application-level ban checks. Cloud Armor rate limiting applies to all requests regardless of HTTP method; however, CORS preflight caching (`max-age: 86400`) makes OPTIONS rate limiting negligible in practice.
- WHEN any other HTTP method is used on any endpoint, THE SYSTEM SHALL return HTTP `405` with an appropriate `Allow` header.

---

### SEC-003A: POST /t Authentication

**Purpose**: Protect the `POST /t` endpoint so that only the legitimate frontend can submit tracking and error data.

**Mechanism**: JWT Bearer Token with static frontend credentials.

**Implementation**:

- A static **client ID** (e.g., `"tjmonsi-web"`) and a static **secret key** are embedded in the frontend code.
- The frontend SHALL obfuscate these credentials in the JavaScript bundle to make extraction non-trivial. Obfuscation methods include:
  - Splitting the secret across multiple variables
  - Using string transformation functions
  - Embedding within compiled/minified code
- The frontend generates a short-lived JWT (5-minute expiry) signed with the static secret using HMAC-SHA256.
- The JWT is included in the JSON request body as a `token` field on each `POST /t` request. This approach enables the use of `navigator.sendBeacon()`, which does not support custom HTTP headers such as `Authorization` (CLR-103).

**JWT Claims**:

```json
{
  "iss": "tjmonsi-web",
  "iat": 1708000000,
  "exp": 1708000300
}
```

**Backend Validation**:

- THE SYSTEM SHALL extract the `token` field from the JSON request body.
- THE SYSTEM SHALL verify the JWT signature using the shared secret (stored as an environment variable).
- THE SYSTEM SHALL verify `iss` matches the expected client ID.
- THE SYSTEM SHALL verify `exp` has not passed (token not expired).
- THE SYSTEM SHALL verify `iat` is not in the future.
- THE SYSTEM SHALL allow a clock skew tolerance of **60 seconds** when validating `exp` and `iat` claims, to accommodate minor clock drift between the client's device and the server.
- IF the `token` field is missing, empty, or validation fails, THE SYSTEM SHALL return HTTP `403 Forbidden`.

**Security Considerations**:

- This is a "security through obscurity" layer — it raises the bar for abuse but is not impenetrable. A determined attacker could extract the secret from the frontend bundle.
- Combined with the standard rate limiting (60 req/min per IP), this provides adequate protection for a personal website.
- The secret SHALL be rotatable via environment variable changes on both frontend build and backend deployment.

---

### SEC-004: Error Response Security

- THE SYSTEM SHALL NEVER expose internal error details, stack traces, or system information in API responses.
- THE SYSTEM SHALL NEVER return HTTP `500` to clients. Internal errors are masked as `404`.
- THE SYSTEM SHALL log all internal errors with full context for debugging in Cloud Logging.
- Error log entry SHALL include:
  - Correlation/Request ID
  - Timestamp (UTC)
  - Request path and parameters (sanitized)
  - Client IP
  - Error type and message
  - Stack trace
  - Cloud Run instance ID

---

### SEC-005: HTTP Security Headers

Security headers SHALL be applied on **both** the frontend (Firebase Hosting) and the backend (Cloud Run), with context-appropriate values for each.

#### Backend API Headers (`api.tjmonsi.com`)

The backend SHALL include the following headers on all API responses:

| Header                         | Value                                                     |
| ------------------------------ | --------------------------------------------------------- |
| `Content-Security-Policy`      | `default-src 'none'`                                      |
| `X-Content-Type-Options`       | `nosniff`                                                 |
| `X-Frame-Options`              | `DENY`                                                    |
| `Strict-Transport-Security`    | `max-age=31536000; includeSubDomains`                     |
| `Referrer-Policy`              | `strict-origin-when-cross-origin`                         |
| `Permissions-Policy`           | `camera=(), microphone=(), geolocation=()`                |

> **Note**: The backend CSP is `default-src 'none'` because API responses (JSON, markdown, XML) do not load sub-resources, execute scripts, or render HTML. A restrictive CSP on the API prevents any unexpected content execution.

#### Frontend Headers (`tjmonsi.com`)

The frontend SHALL include the following headers via Firebase Hosting `firebase.json` headers configuration:

| Header                         | Value                                                     |
| ------------------------------ | --------------------------------------------------------- |
| `Content-Security-Policy`      | `default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonsi.com; img-src 'self' data: https://api.tjmonsi.com; font-src 'self'; worker-src 'self'; object-src 'none'` |
| `X-Content-Type-Options`       | `nosniff`                                                 |
| `X-Frame-Options`              | `DENY`                                                    |
| `Strict-Transport-Security`    | `max-age=31536000; includeSubDomains`                     |
| `Referrer-Policy`              | `strict-origin-when-cross-origin`                         |
| `Permissions-Policy`           | `camera=(), microphone=(), geolocation=()`                |

> **Note**: `'unsafe-inline'` is required for `script-src` because Nuxt 4 SPA builds may inline scripts. If a nonce-based CSP strategy becomes feasible, `'unsafe-inline'` should be replaced with nonce directives. `'unsafe-inline'` for `style-src` allows inline styles used by Vue component frameworks.

**Firebase Hosting Headers Configuration Reference**: Headers are configured in the `firebase.json` file under the `hosting.headers` array. Each entry specifies a `source` glob pattern and a list of `headers` key-value pairs. See: [Firebase Hosting — Configure hosting behavior > Headers](https://firebase.google.com/docs/hosting/full-config#headers)

**Example `firebase.json` headers configuration**:

```json
{
  "hosting": {
    "headers": [
      {
        "source": "**",
        "headers": [
          { "key": "X-Content-Type-Options", "value": "nosniff" },
          { "key": "X-Frame-Options", "value": "DENY" },
          { "key": "Strict-Transport-Security", "value": "max-age=31536000; includeSubDomains" },
          { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
          { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
          { "key": "Content-Security-Policy", "value": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonsi.com; img-src 'self' data: https://api.tjmonsi.com; font-src 'self'; worker-src 'self'; object-src 'none'" }
        ]
      }
    ]
  }
}
```

---

### SEC-006: CORS

- THE SYSTEM SHALL configure CORS headers since the frontend (`tjmonsi.com`) and backend (`api.tjmonsi.com`) are on different origins.
- Allowed origin: `https://tjmonsi.com` (and development environment equivalents)
- Allowed methods: `GET, POST, OPTIONS`
- Allowed headers: `Content-Type, Accept, If-None-Match, If-Modified-Since`
- Exposed headers: `ETag, X-Request-ID` (via `Access-Control-Expose-Headers`)
- Max age: `86400` (1 day)
- `OPTIONS` preflight requests SHALL return the CORS headers and SHALL NOT be subject to application-level ban checks. Cloud Armor rate limiting applies to all HTTP methods; CORS preflight caching (`max-age: 86400`) makes this negligible in practice.

> **Note**: `ETag` must be explicitly exposed for the frontend to read it from cross-origin responses and use it for conditional requests (`If-None-Match`). `X-Request-ID` is exposed so users or the developer can reference it when reporting or debugging issues (e.g., via browser DevTools or the socials/contact page). `Last-Modified` is a CORS-safelisted response header and does not require explicit exposure.

---

### SEC-007: Data Privacy

**Anonymization Requirements**:

- IP addresses SHALL be truncated **before** emitting structured log entries (which flow to BigQuery via log sinks):
  - IPv4: Zero the last octet (e.g., `203.0.113.42` → `203.0.113.0`)
  - IPv6: Zero the last 80 bits
  - IPv4-mapped IPv6: Addresses in the format `::ffff:a.b.c.d` SHALL be treated as IPv4 addresses for truncation purposes. The last IPv4 octet SHALL be zeroed (e.g., `::ffff:203.0.113.42` → `::ffff:203.0.113.0`).
- THE SYSTEM SHALL NOT store full IP addresses in application-level logs, Firestore, or any application-managed data store.
- **Exception — Cloud Armor Load Balancer Logs**: Infrastructure-level load balancer logs (`cloud_armor_lb_logs` BigQuery table, INFRA-010b) contain full IP addresses in the `httpRequest.remoteIp` field. These are generated by Google Cloud infrastructure at the load balancer level before requests reach the Go backend, and cannot be truncated at the source. Full IP addresses in these logs are retained for a maximum of **90 days** for DDoS analysis and security incident response. Only the website owner has access to this table. Looker Studio dashboards SHALL NOT query the raw `cloud_armor_lb_logs` table directly; analytics reports SHALL use truncated IP data from `frontend_tracking_logs` instead.
- **Exception — Rate Limit Offenders**: The `rate_limit_offenders` collection (DM-009) stores full IP addresses because Cloud Armor bans are enforced on exact IPs and the Go backend must verify ban status by matching the request's source IP. The `process-rate-limit-logs` Cloud Function writes full IPs sourced from Cloud Armor load balancer logs. These records are retained for a maximum of 90 days (no active ban) and are only accessible to the Go backend and Cloud Functions service accounts.
- THE SYSTEM SHALL NOT use cookies, session identifiers, user accounts, or device fingerprinting.
- The only personal-adjacent data collected is: truncated IP address, browser name/version, pages visited, referrer URL, connection speed information, approximate country (derived from the client IP address prior to truncation via embedded MaxMind GeoLite2-Country database — only the country code is retained; the full IP is discarded immediately after lookup) (CLR-133, CLR-144), and timestamps.

**Data Retention Summary**:

| Data Store / Table                  | Data Type                          | Retention Period | Deletion Method       |
| ----------------------------------- | ---------------------------------- | ---------------- | --------------------- |
| BigQuery `all_logs`                 | All project logs                   | 2 years          | BigQuery table expiry |
| BigQuery `cloud_armor_lb_logs`      | Load balancer & WAF logs (full IPs) | 90 days          | BigQuery table expiry |
| BigQuery `frontend_tracking_logs`   | Visitor tracking logs              | 2 years          | BigQuery table expiry |
| BigQuery `frontend_error_logs`      | Frontend error logs                | 2 years          | BigQuery table expiry |
| BigQuery `backend_error_logs`       | Backend error logs                 | 2 years          | BigQuery table expiry |
| Firestore `rate_limit_offenders` (DM-009) | Rate limit offender records  | 90 days (no active ban) | Scheduled cleanup |
| Firestore `embedding_cache` (DM-011) | Search query embeddings (no query text stored) | No expiration | Manual purge (DM-011 procedure) (CLR-143) |

> **Note**: Tracking and error report data from `POST /t` is NOT stored in Firestore. The Go backend emits structured log entries to stdout, which Cloud Logging routes to BigQuery via dedicated log sinks (INFRA-010). BigQuery is the sole persistence layer for this data.

**Privacy Compliance**:

- THE SYSTEM SHALL comply with applicable data protection regulations (GDPR if EU visitors are expected).
- THE SYSTEM SHALL include a privacy policy page on the website (see FE-PAGE-009 in [02-frontend-specifications.md](02-frontend-specifications.md)) that clearly discloses:
  - What data is collected and how it is anonymized
  - The purpose of data collection
  - Retention periods for all data stores
  - That infrastructure-level load balancer logs retain full IP addresses for up to 90 days for DDoS analysis and security incident response purposes only, and are not used in analytics reports
  - That anonymized data is exported to BigQuery for long-term analytics (up to 2 years)
  - That analytics dashboards (Looker Studio) are owner-operated and not publicly accessible
  - User rights (right to request information or deletion)
- THE SYSTEM SHALL link the privacy policy from the site footer on all pages.
- Tracking and error report data is retained in BigQuery for up to 2 years via dataset-level table expiry (see INFRA-010). No tracking or error report data is stored in Firestore.
- BigQuery data SHALL be automatically deleted after 2 years via dataset-level table expiry (see INFRA-010).
- THE SYSTEM SHALL NOT share any collected data with third parties.

---

### SEC-008: Dependency Security

- Go dependencies SHALL be audited with `govulncheck` in CI/CD.
- Frontend dependencies SHALL be audited with `npm audit` in CI/CD.
- Docker base images SHALL use minimal, hardened images (distroless preferred).
- Docker images SHALL be scanned for vulnerabilities before deployment.

---

### SEC-009: Looker Studio Service Account

**Purpose**: Provide least-privilege read-only access for Looker Studio to query BigQuery analytics data (see INFRA-011 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)).

**Service Account**: `looker-studio-reader@<project-id>.iam.gserviceaccount.com`

**Granted Roles**:

| Role                          | Scope                      | Purpose                                    |
| ----------------------------- | -------------------------- | ------------------------------------------ |
| `roles/bigquery.dataViewer`   | `website_logs` dataset     | Read-only access to all tables in dataset  |
| `roles/bigquery.jobUser`      | Project                    | Required to execute BigQuery queries       |

**Security Constraints**:

- THE SYSTEM SHALL NOT grant the service account any write, delete, or admin permissions on BigQuery or any other Google Cloud resource.
- THE SYSTEM SHALL NOT grant the service account access to Cloud Logging, Firestore, Cloud Run, or any other service beyond BigQuery.
- The service account key (JSON) SHALL be stored securely by the website owner and NOT committed to any repository.
- The service account key SHALL be rotated periodically (recommended: every 90 days).
- IF the service account key is compromised, THE SYSTEM SHALL revoke the key immediately and generate a new one.

---

### SEC-010: Workload Identity Federation (Content CI/CD)

**Purpose**: Provide secure, keyless authentication for the content management CI/CD pipeline (GitHub Actions in the content repository) to invoke the `sync-article-embeddings` Cloud Function (INFRA-014) after pushing content to Firestore Enterprise. See AD-022 in [01-system-overview.md](01-system-overview.md).

**Configuration**:

| Setting                       | Value                                              |
| ----------------------------- | -------------------------------------------------- |
| Workload Identity Pool        | `content-cicd-pool`                                |
| Workload Identity Provider    | `github-actions` (OIDC)                            |
| Issuer URI                    | `https://token.actions.githubusercontent.com`      |
| Allowed audience              | Default (project number)                           |
| Attribute mapping             | `google.subject` = `assertion.sub`, `attribute.repository` = `assertion.repository` |
| Attribute condition           | `assertion.repository == "<owner>/<content-repo>"` |
| Mapped service account        | `content-cicd@<project-id>.iam.gserviceaccount.com` |

**Service Account Roles**:

| Role                            | Scope                                              | Purpose                                    |
| ------------------------------- | -------------------------------------------------- | ------------------------------------------ |
| `roles/cloudfunctions.invoker`  | `sync-article-embeddings` function                 | Invoke the Cloud Function via HTTP          |
| `roles/run.invoker`             | `sync-article-embeddings` Cloud Run service (Gen 2)| Required because Gen 2 Cloud Functions run on Cloud Run |

**Security Constraints**:

- THE SYSTEM SHALL NOT use long-lived service account keys (JSON key files) for the content CI/CD pipeline. Workload Identity Federation provides short-lived OIDC tokens.
- THE SYSTEM SHALL restrict the Workload Identity Provider to the specific content repository using an attribute condition on the `repository` claim.
- THE SYSTEM SHALL NOT grant the WIF-mapped service account any roles beyond `roles/cloudfunctions.invoker` and `roles/run.invoker` on the embedding sync function.
- The WIF-mapped service account SHALL NOT have access to Firestore Enterprise, Firestore Native, BigQuery, Vertex AI, or any other Google Cloud service.

> **Note on Firestore Enterprise write access**: The content CI/CD pipeline pushes article content to Firestore Enterprise collections (`technical_articles`, `blog_articles`, `others`, `categories`) using its own credentials provisioned in the content repository project (out of scope for this spec). The credential mechanism (GCP IAM service account or MongoDB wire protocol username/password) is determined by the content pipeline's own infrastructure. The `content-cicd@` SA (#6 in SEC-013) is used only for invoking the embedding sync function after content is pushed — it does not have Firestore Enterprise write access.
- Reference: [Workload Identity Federation for GitHub Actions](https://cloud.google.com/iam/docs/workload-identity-federation), [google-github-actions/auth](https://github.com/google-github-actions/auth)

---

### SEC-011: Vector Search & Vertex AI Security

**Purpose**: Secure the vector search infrastructure, including Firestore Native Mode access, Vertex AI embedding API, and the embedding cache.

#### Firestore Native Mode Access Control

| Principal                              | Role                                   | Scope                          | Purpose                                    |
| -------------------------------------- | -------------------------------------- | ------------------------------ | ------------------------------------------ |
| Cloud Run service account              | `roles/datastore.viewer`               | `vector-search` database       | Read vector documents for search queries   |
| `sync-article-embeddings` Cloud Function SA | `roles/datastore.user`            | `vector-search` database       | Read/write vector documents during sync    |

**Constraints**:
- THE SYSTEM SHALL NOT grant Cloud Run write access to the Firestore Native database. Cloud Run only reads vectors for search.
- Only the `sync-article-embeddings` Cloud Function SHALL have write access to Firestore Native.

#### Vertex AI Embedding API Access Control

| Principal                              | Role                                   | Scope   | Purpose                                    |
| -------------------------------------- | -------------------------------------- | ------- | ------------------------------------------ |
| Cloud Run service account              | `roles/aiplatform.user`               | Project | Call Vertex AI Gemini embedding API for query vectors |
| `sync-article-embeddings` Cloud Function SA | `roles/aiplatform.user`           | Project | Call Vertex AI Gemini embedding API for article vectors |

**Constraints**:
- No other service account SHALL have access to the Vertex AI Embeddings API.
- The embedding API is called within Google Cloud infrastructure using IAM-based authentication (no external API keys).

#### Embedding Cache Security

- The `embedding_cache` collection (DM-011) in Firestore Enterprise does NOT store search strings — only UUID v5 identifiers and embedding vectors.
- Embedding vectors are numerical representations and cannot be reverse-engineered to recover the original text.
- The UUID v5 document IDs are cryptographic hashes (SHA-1 based) and cannot be reversed to the original query text.
- No additional access controls beyond the existing Cloud Run Firestore Enterprise service account are required for the cache.

---

### SEC-012: Terraform Service Account & Workload Identity Federation

**Purpose**: Provide a dedicated, least-privilege service account for Terraform to manage GCP infrastructure resources (INFRA-016). The service account is a **manually created bootstrap resource** — it is NOT managed by Terraform itself. The Terraform CI/CD pipeline authenticates via **Workload Identity Federation** (WIF), consistent with the content CI/CD approach (SEC-010). See AD-023 in [01-system-overview.md](01-system-overview.md).

**Service Account**: `terraform-builder@<project-id>.iam.gserviceaccount.com`

**Provisioning**: Manual (created by the project owner via Console or `gcloud`)

**Granted Roles**:

| Role                                | Scope     | Purpose                                           |
| ----------------------------------- | --------- | ------------------------------------------------- |
| `roles/storage.objectAdmin`         | Terraform state bucket (INFRA-015) | Read/write Terraform state and lock files |
| `roles/run.admin`                   | Project   | Manage Cloud Run services (INFRA-003)             |
| `roles/cloudfunctions.admin`        | Project   | Manage Cloud Functions (INFRA-008a, 008c, 008d, INFRA-014) |
| `roles/compute.networkAdmin`        | Project   | Manage VPC, subnets, firewall rules (INFRA-009)   |
| `roles/dns.admin`                   | Project   | Manage Cloud DNS zones and records (INFRA-017)    |
| `roles/artifactregistry.admin`      | Project   | Manage Artifact Registry repositories (INFRA-018) |
| `roles/logging.admin`               | Project   | Manage log sinks, log buckets (INFRA-007, INFRA-010) |
| `roles/monitoring.admin`            | Project   | Manage alerting policies, dashboards (OBS-005, OBS-006) |
| `roles/iam.serviceAccountAdmin`     | Project   | Manage service accounts (SEC-013)                 |
| `roles/pubsub.admin`               | Project   | Manage Pub/Sub topics and subscriptions (INFRA-008c) |
| `roles/bigquery.admin`             | Project   | Manage BigQuery datasets and log sinks (INFRA-010) |
| `roles/datastore.user`             | Project   | Manage Firestore Enterprise and Native databases (INFRA-006, INFRA-012) |

> **Note**: The IAM roles above represent the minimum set required for managing resources defined in INFRA-016. Roles follow the principle of least privilege. Additional roles may be needed if Terraform scope expands; any additions SHALL be documented with justification. (CLR-149)

**Workload Identity Federation Configuration (Terraform CI/CD)**:

| Setting                       | Value                                              |
| ----------------------------- | -------------------------------------------------- |
| Workload Identity Pool        | `terraform-cicd-pool`                              |
| Workload Identity Provider    | `github-actions` (OIDC)                            |
| Issuer URI                    | `https://token.actions.githubusercontent.com`      |
| Allowed audience              | Default (project number)                           |
| Attribute mapping             | `google.subject` = `assertion.sub`, `attribute.repository` = `assertion.repository` |
| Attribute condition           | `assertion.repository == "<owner>/personal-website"` |
| Mapped service account        | `terraform-builder@<project-id>.iam.gserviceaccount.com` |

> **Note**: The WIF pool and provider for Terraform CI/CD are separate from the content CI/CD pool (SEC-010) to maintain distinct trust boundaries. Each pool trusts only its respective repository.

**Security Constraints**:

- THE SYSTEM SHALL NOT grant the Terraform service account `roles/owner` or `roles/editor`. Only specific, granular roles SHALL be used.
- THE SYSTEM SHALL use **Workload Identity Federation** for the Terraform CI/CD pipeline (GitHub Actions). No long-lived service account keys (JSON key files) SHALL be used in CI/CD.
- For local Terraform runs (development/debugging), the project owner MAY use a service account key stored securely outside any repository. This key SHALL NOT be committed to any repository.
- The Terraform CI/CD pipeline authenticates via WIF using the `google-github-actions/auth` action, consistent with the pattern established in SEC-010.
- IF a service account key is used for local runs and is compromised, THE SYSTEM SHALL revoke the key immediately, generate a new one, and audit recent Terraform state changes.
- The service account SHOULD minimize access to application data. Where infrastructure management roles (e.g., `roles/datastore.user`) inherently grant data-level access, this is an accepted trade-off documented in CLR-149. (CLR-192)
- The Terraform state bucket (INFRA-015) SHALL NOT be publicly accessible. Only the Terraform service account SHALL have write access.
- Reference: [Workload Identity Federation for GitHub Actions](https://cloud.google.com/iam/docs/workload-identity-federation), [google-github-actions/auth](https://github.com/google-github-actions/auth)

---

### SEC-013: Consolidated Service Account Inventory

**Purpose**: Provide a single reference of all service accounts used across the system, their IAM roles, scopes, and management method. This inventory ensures no default service accounts are used — each component has a dedicated service account for security, auditability, and least-privilege enforcement.

**Service Account Inventory**:

| # | Service Account Email | Name | Purpose | Key IAM Roles | Scope | Managed By |
|---|----------------------|------|---------|---------------|-------|------------|
| 1 | `cloud-run-api@<project-id>.iam.gserviceaccount.com` | Cloud Run API SA | Identity for the Go backend API (INFRA-003) | `roles/datastore.viewer` (Firestore Native `vector-search` DB), `roles/aiplatform.user` (Vertex AI), `roles/storage.objectViewer` (media bucket `<project-id>-media-bucket`, INFRA-019), `roles/datastore.user` (Firestore Enterprise, MongoDB wire protocol auth via IAM — CLR-148) | Project / specific resources | Terraform |
| 2 | `cf-sitemap-gen@<project-id>.iam.gserviceaccount.com` | Sitemap Generator SA | Identity for the `generate-sitemap` Cloud Function (INFRA-008a) | `roles/datastore.user` (Firestore Enterprise — read `technical_articles`, `blog_articles`; read/write `sitemap`) (CLR-185) | INFRA-008a function | Terraform |
| 3 | `cf-rate-limit-proc@<project-id>.iam.gserviceaccount.com` | Rate Limit Log Processor SA | Identity for the `process-rate-limit-logs` Cloud Function (INFRA-008c) | `roles/datastore.user` (Firestore Enterprise — read/write `rate_limit_offenders`) (CLR-185) | INFRA-008c function | Terraform |
| 4 | `cf-offender-cleanup@<project-id>.iam.gserviceaccount.com` | Offender Cleanup SA | Identity for the `cleanup-rate-limit-offenders` Cloud Function (INFRA-008d) | `roles/datastore.user` (Firestore Enterprise — read/write `rate_limit_offenders`) (CLR-185) | INFRA-008d function | Terraform |
| 5 | `cf-embed-sync@<project-id>.iam.gserviceaccount.com` | Embedding Sync SA | Identity for the `sync-article-embeddings` Cloud Function (INFRA-014) | Firestore Enterprise read (articles), `roles/datastore.user` (Firestore Native `vector-search` DB), `roles/aiplatform.user` (Vertex AI) | INFRA-014 function | Terraform |
| 6 | `content-cicd@<project-id>.iam.gserviceaccount.com` | Content CI/CD SA | WIF-mapped SA for content repository GitHub Actions (SEC-010). Used only for invoking the embedding sync function — Firestore Enterprise write access for content publishing uses separate credentials provisioned in the content pipeline project (out of scope). | `roles/cloudfunctions.invoker`, `roles/run.invoker` on `sync-article-embeddings` | INFRA-014 function | Terraform |
| 7 | `terraform-builder@<project-id>.iam.gserviceaccount.com` | Terraform Builder SA | WIF-mapped SA for Terraform CI/CD pipeline (SEC-012) | `roles/storage.objectAdmin` (state bucket), `roles/run.admin`, `roles/cloudfunctions.admin`, `roles/compute.networkAdmin`, `roles/dns.admin`, `roles/artifactregistry.admin`, `roles/logging.admin`, `roles/monitoring.admin`, `roles/iam.serviceAccountAdmin`, `roles/pubsub.admin`, `roles/bigquery.admin`, `roles/datastore.user` (CLR-149) | Project | **Manual** (bootstrap) |
| 8 | `looker-studio-reader@<project-id>.iam.gserviceaccount.com` | Looker Studio Reader SA | Read-only BigQuery access for Looker Studio dashboards (SEC-009, INFRA-011) | `roles/bigquery.dataViewer` (dataset), `roles/bigquery.jobUser` (project) | `website_logs` dataset + project | Terraform |
| 9 | `cloud-scheduler-invoker@<project-id>.iam.gserviceaccount.com` | Cloud Scheduler Invoker SA | OIDC token issuer for Cloud Scheduler jobs (INFRA-008b, 008e, 014b) | `roles/cloudfunctions.invoker`, `roles/run.invoker` on target Cloud Functions | Target functions | Terraform |

**Security Constraints**:

- THE SYSTEM SHALL NOT use default service accounts (e.g., Compute Engine default SA, App Engine default SA) for any component. Each component has a dedicated SA with minimum required permissions.
- All Terraform-managed service accounts SHALL be created via `google_service_account` resources in Terraform.
- No service account SHALL have `roles/owner` or `roles/editor`.
- Service accounts SHALL NOT have cross-component access unless explicitly documented above.
- WIF-authenticated SAs (#6, #7) do not require JSON key files for CI/CD use.
- The Looker Studio SA (#8) requires a JSON key for the Looker Studio data connector. This key SHALL be stored securely by the project owner and rotated every 90 days (see SEC-009).

> **Exception**: Firebase Functions (INFRA-002) uses the Firebase default service account because it only serves static HTML content and does not access any GCP resources. The default SA is acceptable for this component since it has no meaningful permissions to abuse.

> **Note**: The Cloud Run SA (#1) uses `roles/datastore.user` for Firestore Enterprise access via IAM-based MongoDB wire protocol authentication. The Cloud Run service connects using the connection string approach documented at https://docs.cloud.google.com/firestore/mongodb-compatibility/docs/connect#cloud-run. (CLR-148)

---

### Acceptance Criteria

- **AC-SEC-001**: Given a `POST /t` request with a valid JWT `token` in the body, when the backend validates it, then signature, `iss`, `exp`, and `iat` are verified and the request is processed.
- **AC-SEC-002**: Given a `POST /t` request with a missing `token` field, when the backend processes it, then the response is `403 Forbidden`.
- **AC-SEC-003**: Given a client exceeding rate limits, when progressive banning is enforced, then responses escalate from `429` → `403` → `404` based on offense tier.
- **AC-SEC-004**: Given the frontend CSP, when a page loads, then `img-src` allows `'self' data: https://api.tjmonsi.com` and blocks all other image origins.
- **AC-SEC-005**: Given CORS configuration, when the frontend (`tjmonsi.com`) sends a cross-origin request to `api.tjmonsi.com`, then the backend returns appropriate CORS headers with allowed origin, methods, and headers (excluding `Authorization`).
- **AC-SEC-006**: Given the backend API, when any response is sent, then security headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Referrer-Policy`, `Permissions-Policy`) are present.
- **AC-SEC-007**: Given IP address processing, when the Go backend handles a request, then the IP is truncated (last octet zeroed for IPv4, last 80 bits zeroed for IPv6) before logging.
- **AC-SEC-008**: Given any service account in the system, when its IAM roles are reviewed, then no SA has `roles/owner` or `roles/editor`.
- **AC-SEC-009**: Given the Cloud Run container, when it runs, then it executes as non-root user `jack` (UID 10001) on a distroless base image.
- **AC-SEC-010**: Given a CORS preflight `OPTIONS` request, when the backend processes it, then it returns CORS headers and is not subject to application-level ban checks.
- **AC-SEC-011**: Given a search query exceeding 300 characters, when submitted to any search endpoint (`GET /technical`, `GET /blog`, `GET /others`), then the backend returns HTTP `400`.
- **AC-SEC-012**: Given an article slug not matching the regex `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}\.md$`, when requesting an article detail endpoint, then the backend returns HTTP `404`.
- **AC-SEC-013**: Given a `POST /t` error report with a `breadcrumbs` array exceeding 50 entries, when the backend validates it, then the array is truncated to the last 50 entries (newest kept) and a WARNING is logged — the request is NOT rejected.
- **AC-SEC-014**: Given the Go backend's in-memory ban status LRU cache (SEC-002), when a ban lookup is performed, then the cache uses a 60-second TTL and a maximum of 1000 entries; on cache miss, Firestore is queried and the result is cached.
- **AC-SEC-015**: Given the CI/CD pipeline, when the build runs, then Go dependencies are audited with `govulncheck` and frontend dependencies are audited with `npm audit`; Docker images are scanned for vulnerabilities before deployment.
- **AC-SEC-016**: Given the content CI/CD pipeline (SEC-010), when GitHub Actions authenticates to GCP, then Workload Identity Federation is used (no long-lived service account keys) and the WIF provider attribute condition restricts access to the specific content repository.
- **AC-SEC-017**: Given the Cloud Run service account, when accessing Firestore Native (`vector-search` database), then it has only `roles/datastore.viewer` (read-only); write access to Firestore Native is restricted to the `sync-article-embeddings` Cloud Function SA.
- **AC-SEC-018**: Given query parameters on any `GET` endpoint, when unknown or malformed parameters are present, then the backend returns HTTP `400` with a standard error response body.
- **AC-SEC-019**: Given an indefinitely banned IP whose `rate_limit_offenders` document is deleted from Firestore, when the LRU cache TTL (60 seconds) expires, then the IP is unblocked and can access the site normally.
