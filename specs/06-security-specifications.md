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

---

### SEC-001: Input Validation

**Search Query Validation**:

- THE SYSTEM SHALL reject search queries exceeding 300 characters with HTTP `400`.
- THE SYSTEM SHALL sanitize search queries to prevent injection attacks.
- THE SYSTEM SHALL strip or escape special characters that could be interpreted as query operators.

**Slug Validation**:

- THE SYSTEM SHALL validate article slugs against the prescribed regex pattern.
- THE SYSTEM SHALL reject any slug not matching the pattern with HTTP `404`.
- Regex pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}T\d{2}-\d{2}-\d{2}\.md$`

**Query Parameter Validation**:

- THE SYSTEM SHALL validate `page` as a positive integer.
- THE SYSTEM SHALL validate `date_from` and `date_to` as valid ISO 8601 dates.
- THE SYSTEM SHALL validate `category` against known categories (if categories are predefined — see CLR-006).
- THE SYSTEM SHALL validate `tags` as a comma-separated list of alphanumeric strings with hyphens.
- THE SYSTEM SHALL reject requests with unknown or malformed query parameters with HTTP `400`.

---

### SEC-002: Rate Limiting

#### Application-Level Rate Limiting

**Normal Rate Limits** (per IP address):

| Scope           | Limit                         | Window     |
| --------------- | ----------------------------- | ---------- |
| Global (all endpoints) | *TBD — See CLR-007*    | Per minute |
| Per endpoint    | *TBD*                         | Per minute |

> **Consideration**: The plan notes that a single IP may represent multiple users (shared ISP NAT). Rate limits should be generous enough to accommodate this. See CLR-007.

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

| Condition                                         | Action                      |
| ------------------------------------------------- | --------------------------- |
| Rate limit exceeded                               | Return `429` with `Retry-After` |
| 5 offenses within 7 days                          | Block for 30 days           |
| 2 offenses after 30-day ban expires               | Block for 90 days           |
| 2 offenses after 90-day ban expires               | Block indefinitely          |

**Blocking behavior**:

- WHEN a blocked client makes a request, THE SYSTEM SHALL return HTTP `403 Forbidden`.
  - *Or* HTTP `429` — see CLR-008 for preferred status code for banned clients.
- THE SYSTEM SHALL include when the ban expires (if not indefinite) in the response.
- Bans are tracked in the `rate_limit_offenders` database collection (see DM-008).

**Ban response**:

```json
{
  "error": "Access temporarily restricted.",
  "blocked_until": "2026-03-19T00:00:00Z"
}
```

---

#### Bot-Specific Rate Limiting

- THE SYSTEM SHALL identify bots via User-Agent header analysis.
- THE SYSTEM SHALL apply separate (potentially stricter) rate limits to known bot User-Agents.
- THE SYSTEM SHALL respect `robots.txt` crawl-delay directives.

---

### SEC-003: HTTP Method Enforcement

- THE SYSTEM SHALL only accept `GET` requests.
- WHEN any other HTTP method is used, THE SYSTEM SHALL return HTTP `405` with header `Allow: GET`.

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
| `Content-Security-Policy`      | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'` |
| `X-Content-Type-Options`       | `nosniff`                                                 |
| `X-Frame-Options`              | `DENY`                                                    |
| `Strict-Transport-Security`    | `max-age=31536000; includeSubDomains`                     |
| `Referrer-Policy`              | `strict-origin-when-cross-origin`                         |
| `Permissions-Policy`           | `camera=(), microphone=(), geolocation=()`                |

---

### SEC-006: CORS

- IF the frontend and backend are on different origins (e.g., subdomain routing), THE SYSTEM SHALL configure CORS headers.
- Allowed origin: `https://tjmonserrat.com` (and staging equivalents)
- Allowed methods: `GET`
- Allowed headers: `Content-Type, Accept`
- Max age: `86400` (1 day)

---

### SEC-007: Data Privacy

- Tracking data (DM-006) SHALL be anonymized to the extent possible.
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
