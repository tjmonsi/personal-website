---
title: Security Specifications
version: 1.1
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [security, rate-limiting, cors, authentication]
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

---

### SEC-002: Rate Limiting

#### Application-Level Rate Limiting

**Normal Rate Limits** (per IP address):

| Scope           | Limit                         | Window     |
| --------------- | ----------------------------- | ---------- |
| Regular user (all GET endpoints) | 60 req/min    | Per minute |
| Known bots      | 10 req/min                    | Per minute |
| Tracking/error (`POST /t`) | 30 req/min           | Per minute |

**Rationale**:
- **60 req/min for regular users**: Generous enough to accommodate shared NAT/ISP IPs where multiple users share a single public IP. Based on common web application rate limiting practices. Reference: [OWASP Rate Limiting Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html)
- **10 req/min for known bots**: Aligns with standard crawl rates recommended for polite bots. Google recommends no more than one request every few seconds. Reference: [Google Search Central - Crawl Rate](https://developers.google.com/search/docs/crawling-indexing/reduce-crawl-rate)
- **30 req/min for tracking/error**: Separate bucket prevents tracking calls from consuming user quota. Based on typical single-page application page view rates. Reference: [Cloudflare Rate Limiting Best Practices](https://developers.cloudflare.com/waf/rate-limiting-rules/best-practices/)

> **Note**: A single IP may represent multiple users (shared ISP NAT). These limits are set to be generous enough to accommodate this while still providing protection.

**Rate Limit Headers** (on every response):

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1706140800
Retry-After: 30  (only on 429 responses)
```

**Rate limit response** (`429`):

```json
{
  "error": "Too many requests. Please try again later.",
  "retry_after": 30
}
```

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

#### Bot-Specific Rate Limiting

- THE SYSTEM SHALL identify bots via User-Agent header analysis.
- THE SYSTEM SHALL apply separate (potentially stricter) rate limits to known bot User-Agents.
- THE SYSTEM SHALL respect `robots.txt` crawl-delay directives.

---

### SEC-003: HTTP Method Enforcement

- THE SYSTEM SHALL accept `GET` requests on all endpoints except `POST /t`.
- THE SYSTEM SHALL accept `POST` requests only on the `/t` endpoint.
- WHEN any other HTTP method is used on any endpoint, THE SYSTEM SHALL return HTTP `405` with an appropriate `Allow` header.

---

### SEC-003A: POST /t Authentication

**Purpose**: Protect the `POST /t` endpoint so that only the legitimate frontend can submit tracking and error data.

**Mechanism**: JWT Bearer Token with static frontend credentials.

**Implementation**:

- A static **client ID** (e.g., `"tjmonserrat-web"`) and a static **secret key** are embedded in the frontend code.
- The frontend SHALL obfuscate these credentials in the JavaScript bundle to make extraction non-trivial. Obfuscation methods include:
  - Splitting the secret across multiple variables
  - Using string transformation functions
  - Embedding within compiled/minified code
- The frontend generates a short-lived JWT (5-minute expiry) signed with the static secret using HMAC-SHA256.
- The JWT is sent as `Authorization: Bearer <token>` on each `POST /t` request.

**JWT Claims**:

```json
{
  "iss": "tjmonserrat-web",
  "iat": 1708000000,
  "exp": 1708000300
}
```

**Backend Validation**:

- THE SYSTEM SHALL verify the JWT signature using the shared secret (stored as an environment variable).
- THE SYSTEM SHALL verify `iss` matches the expected client ID.
- THE SYSTEM SHALL verify `exp` has not passed (token not expired).
- THE SYSTEM SHALL verify `iat` is not in the future.
- IF any validation fails, THE SYSTEM SHALL return HTTP `403 Forbidden`.

**Security Considerations**:

- This is a "security through obscurity" layer — it raises the bar for abuse but is not impenetrable. A determined attacker could extract the secret from the frontend bundle.
- Combined with rate limiting on `POST /t` (30 req/min), this provides adequate protection for a personal website.
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

The backend SHALL include the following headers on all responses:

| Header                         | Value                                                     |
| ------------------------------ | --------------------------------------------------------- |
| `Content-Security-Policy`      | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.tjmonserrat.com` |
| `X-Content-Type-Options`       | `nosniff`                                                 |
| `X-Frame-Options`              | `DENY`                                                    |
| `Strict-Transport-Security`    | `max-age=31536000; includeSubDomains`                     |
| `Referrer-Policy`              | `strict-origin-when-cross-origin`                         |
| `Permissions-Policy`           | `camera=(), microphone=(), geolocation=()`                |

---

### SEC-006: CORS

- THE SYSTEM SHALL configure CORS headers since the frontend (`tjmonserrat.com`) and backend (`api.tjmonserrat.com`) are on different origins.
- Allowed origin: `https://tjmonserrat.com` (and staging equivalents)
- Allowed methods: `GET, POST`
- Allowed headers: `Content-Type, Accept, Authorization`
- Max age: `86400` (1 day)

---

### SEC-007: Data Privacy

- Tracking data (DM-007) SHALL be anonymized to the extent possible.
- IP addresses SHALL be stored but MAY be hashed or truncated for privacy compliance.
- THE SYSTEM SHALL comply with applicable data protection regulations (GDPR if EU visitors are expected).
- THE SYSTEM SHALL include a privacy notice or link on the website.
- Tracking and error report data SHALL have TTL-based auto-expiry.

---

### SEC-008: Dependency Security

- Go dependencies SHALL be audited with `govulncheck` in CI/CD.
- Frontend dependencies SHALL be audited with `npm audit` in CI/CD.
- Docker base images SHALL use minimal, hardened images (distroless preferred).
- Docker images SHALL be scanned for vulnerabilities before deployment.
