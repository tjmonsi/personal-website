---
title: Security Specifications
version: 2.3
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [security, rate-limiting, cors, authentication, vector-search]
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
| Unauthorized tracking abuse   | JWT bearer token authentication on POST /t            | Application    |
| Analytics data exposure       | Least-privilege service account with read-only BigQuery access | Infrastructure |
| Embedding API abuse           | Embedding cache prevents redundant Gemini API calls; rate limiting on search endpoints | Application + Infrastructure |
| Vector data exfiltration       | Firestore Native IAM; vectors contain no readable text         | Infrastructure |
| Compromised CI/CD credentials  | Workload Identity Federation (keyless OIDC auth, no service account keys) | Infrastructure |

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
- THE SYSTEM SHALL reject requests with unknown or malformed query parameters with HTTP `400`.

> **Note**: This strict query parameter validation is intentional. The backend API (`api.tjmonsi.com`) is not the URL shared on social media — the frontend URL (`tjmonsi.com`) is. UTM parameters and other tracking query strings on the frontend URL are handled client-side and forwarded to the backend via `POST /t` as tracking data. The backend API expects a strict one-to-one mapping of query parameters for search and pagination purposes only.

---

### SEC-002: Rate Limiting

#### Rate Limiting Architecture

- **Real-time rate counting**: Enforced by **Google Cloud Armor** at the load balancer level (see INFRA-005 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)). Cloud Armor handles per-IP request counting and throttling. This avoids the need for in-memory rate counters in the Go application, which would be lost when Cloud Run instances scale down or restart.
- **Offense tracking via log sink**: A **Cloud Logging log sink** routes Cloud Armor rate-limit (429) events to a **Cloud Function** (`process-rate-limit-logs`, INFRA-008c). The Cloud Function writes offense records to the `rate_limit_offenders` Firestore collection (DM-009). This bridges the gap between Cloud Armor (which blocks requests before they reach Cloud Run) and the application's progressive banning logic.
- **Progressive banning state**: Stored in **Firestore** (`rate_limit_offenders` collection, DM-009). The Go application checks Firestore for ban status on each request and enforces progressive banning tiers.
- **Adaptive protection**: **Cloud Armor Adaptive Protection** is enabled to provide automatic, ML-based escalating DDoS mitigation that complements the application-level progressive banning (see INFRA-005).
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
| 2 offenses after 30-day ban expires               | Block for 90 days           | `404`         |
| 2 offenses after 90-day ban expires               | Block indefinitely          | `404`         |

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

**Ban response for 403** (30-day block):

```json
{
  "error": "Access temporarily restricted.",
  "blocked_until": "2026-03-19T00:00:00Z"
}
```

**Ban response for 404** (90-day / indefinite block):

Standard 404 response — indistinguishable from a normal "not found" to the client.

---

#### Bot Management

- THE SYSTEM SHALL rely on `robots.txt` crawl-delay directives to manage bot crawl rates (see OBS-007 in [07-observability-specifications.md](07-observability-specifications.md)).
- THE SYSTEM SHALL NOT apply separate rate limits to bots — the single 60 req/min rate limit applies to all clients equally.
- Cloud Armor's built-in bot management capabilities MAY be used for additional protection against malicious bots.

---

### SEC-003: HTTP Method Enforcement

- THE SYSTEM SHALL accept `GET` requests on all endpoints except `POST /t`.
- THE SYSTEM SHALL accept `POST` requests only on the `/t` endpoint.
- THE SYSTEM SHALL accept `OPTIONS` requests on all endpoints for CORS preflight handling. `OPTIONS` requests SHALL return appropriate CORS headers and SHALL NOT be subject to rate limiting.
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
- The JWT is sent as `Authorization: Bearer <token>` on each `POST /t` request.

**JWT Claims**:

```json
{
  "iss": "tjmonsi-web",
  "iat": 1708000000,
  "exp": 1708000300
}
```

**Backend Validation**:

- THE SYSTEM SHALL verify the JWT signature using the shared secret (stored as an environment variable).
- THE SYSTEM SHALL verify `iss` matches the expected client ID.
- THE SYSTEM SHALL verify `exp` has not passed (token not expired).
- THE SYSTEM SHALL verify `iat` is not in the future.
- THE SYSTEM SHALL allow a clock skew tolerance of **60 seconds** when validating `exp` and `iat` claims, to accommodate minor clock drift between the client's device and the server.
- IF any validation fails, THE SYSTEM SHALL return HTTP `403 Forbidden`.

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
| `Content-Security-Policy`      | `default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonsi.com; img-src 'self' data:; font-src 'self'` |
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
          { "key": "Content-Security-Policy", "value": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonsi.com; img-src 'self' data:; font-src 'self'" }
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
- Allowed headers: `Content-Type, Accept, Authorization, If-None-Match, If-Modified-Since`
- Max age: `86400` (1 day)
- `OPTIONS` preflight requests SHALL return the CORS headers and SHALL NOT be subject to rate limiting.

---

### SEC-007: Data Privacy

**Anonymization Requirements**:

- IP addresses SHALL be truncated **before** emitting structured log entries (which flow to BigQuery via log sinks):
  - IPv4: Zero the last octet (e.g., `203.0.113.42` → `203.0.113.0`)
  - IPv6: Zero the last 80 bits
- THE SYSTEM SHALL NOT store full IP addresses in application-level logs, Firestore, or any application-managed data store.
- **Exception — Cloud Armor Load Balancer Logs**: Infrastructure-level load balancer logs (`cloud_armor_lb_logs` BigQuery table, INFRA-010b) contain full IP addresses in the `httpRequest.remoteIp` field. These are generated by Google Cloud infrastructure at the load balancer level before requests reach the Go backend, and cannot be truncated at the source. Full IP addresses in these logs are retained for a maximum of **90 days** for DDoS analysis and security incident response. Only the website owner has access to this table. Looker Studio dashboards SHALL NOT query the raw `cloud_armor_lb_logs` table directly; analytics reports SHALL use truncated IP data from `frontend_tracking_logs` instead.
- THE SYSTEM SHALL NOT use cookies, session identifiers, user accounts, or device fingerprinting.
- The only personal-adjacent data collected is: truncated IP address, browser name/version, pages visited, referrer URL, connection speed information, and timestamps.

**Data Retention Summary**:

| Data Store / Table                  | Data Type                          | Retention Period | Deletion Method       |
| ----------------------------------- | ---------------------------------- | ---------------- | --------------------- |
| BigQuery `all_logs`                 | All project logs                   | 2 years          | BigQuery table expiry |
| BigQuery `cloud_armor_lb_logs`      | Load balancer & WAF logs (full IPs) | 90 days          | BigQuery table expiry |
| BigQuery `frontend_tracking_logs`   | Visitor tracking logs              | 2 years          | BigQuery table expiry |
| BigQuery `frontend_error_logs`      | Frontend error logs                | 2 years          | BigQuery table expiry |
| BigQuery `backend_error_logs`       | Backend error logs                 | 2 years          | BigQuery table expiry |
| Firestore `rate_limit_offenders` (DM-009) | Rate limit offender records  | 90 days (no active ban) | Scheduled cleanup |

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
- THE SYSTEM SHALL NOT grant the WIF-mapped service account any roles beyond `roles/cloudfunctions.invoker` and `roles/run.invoker` on the embedding sync function. The content pipeline pushes to Firestore Enterprise using its own credentials (separate from this project's IAM).
- The WIF-mapped service account SHALL NOT have access to Firestore Enterprise, Firestore Native, BigQuery, Vertex AI, or any other Google Cloud service.
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
