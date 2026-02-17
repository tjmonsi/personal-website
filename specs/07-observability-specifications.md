---
title: Observability Specifications
version: 2.0
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [observability, logging, monitoring, tracking, vector-search]
---

## Observability Specifications

### Overview

The observability strategy covers three pillars: logging, metrics, and tracing. All telemetry data flows into Google Cloud operations suite (Cloud Logging, Cloud Monitoring).

---

### OBS-001: Structured Logging (Backend)

**Format**: JSON structured logs to stdout (Cloud Run automatically ingests).

**Log Schema**:

```json
{
  "severity": "ERROR",
  "timestamp": "2026-02-17T10:30:00Z",
  "request_id": "uuid-v4",
  "method": "GET",
  "path": "/technical",
  "query_params": "q=example&page=1",
  "client_ip": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "status_code": 404,
  "latency_ms": 235,
  "message": "Article not found",
  "error": {
    "type": "NotFoundError",
    "message": "No article with slug 'invalid-slug'",
    "stack": "..."
  },
  "labels": {
    "masked_500": true
  },
  "log_type": "frontend_tracking",
  "cloud_run_instance": "instance-abc123"
}
```

> **Note on `labels`**: The `labels` field is included only when applicable. When an internal server error is masked as a 404 response, the log entry MUST include `"masked_500": true` in the `labels` object. This label is used by OBS-005 alerting to detect masked 500 errors.

> **Note on `log_type`**: The `log_type` field is included only for `POST /t` request log entries. It enables BigQuery log sink routing (INFRA-010) to separate frontend tracking and error logs into dedicated tables.
> - `"frontend_tracking"` — for `POST /t` requests with `action: "page_view"` or other tracking actions.
> - `"frontend_error"` — for `POST /t` requests with `action: "error_report"`.
> - Omitted for all other requests (standard backend request logs).
```

**Log Levels**:

| Level   | Usage                                                     |
| ------- | --------------------------------------------------------- |
| DEBUG   | Detailed execution flow (disabled in production)          |
| INFO    | Normal operations: request completed, cache hit/miss      |
| WARNING | Non-critical issues: slow query, high memory usage           |
| ERROR   | Failures: database errors, masked 500s                    |
| CRITICAL| System-level failures: database connection lost           |

**Mandatory Logging Events**:

- Every incoming request (INFO): method, path, status, latency
- Every `POST /t` tracking request (INFO): include `log_type: "frontend_tracking"` and the tracking payload fields (page, referrer, action, browser, connection_speed) in the structured log entry
- Every `POST /t` error report (INFO): include `log_type: "frontend_error"` and the error payload fields (error_type, error_message, page, browser, connection_speed) in the structured log entry
- Rate limit triggered (WARNING): client identifier, endpoint, offense count
- Rate limit ban applied (WARNING): client identifier, ban tier, expiry
- Internal error masked as 404 (ERROR): full error details
- Database query errors (ERROR): query details (sanitized), error message
- Slow queries > 1 second (WARNING): query details, duration
- Embedding cache hit (INFO): UUID v5 document ID, collection searched
- Embedding cache miss (INFO): UUID v5 document ID, Gemini API call duration in ms
- Gemini embedding API error (ERROR): error type, message, UUID v5 document ID
- Vector search executed (INFO): collection, number of candidates returned, distance range, latency in ms
- Vector search no results within threshold (INFO): collection, threshold used

---

### OBS-001a: Structured Logging (Cloud Functions)

**Purpose**: Ensure Cloud Functions (sitemap generation, rate-limit log processing, and article embedding sync) produce structured, queryable logs consistent with the backend logging approach.

**Format**: JSON structured logs to stdout (Cloud Functions automatically ingests into Cloud Logging).

**Logging Requirements for `generate-sitemap` (INFRA-008a)**:

| Event                          | Level   | Details                                    |
| ------------------------------ | ------- | ------------------------------------------ |
| Generation started             | INFO    | Trigger source, timestamp                  |
| URLs processed                 | INFO    | Count of URLs included in sitemap          |
| Firestore write result         | INFO    | Success/failure, document ID               |
| Generation duration            | INFO    | Total processing time in ms                |
| Generation error               | ERROR   | Error type, message, stack trace           |

**Logging Requirements for `process-rate-limit-logs` (INFRA-008c)**:

| Event                          | Level   | Details                                    |
| ------------------------------ | ------- | ------------------------------------------ |
| Log entry received             | INFO    | Client IP (sanitized), timestamp           |
| Offense recorded               | WARNING | Client IP, offense count, endpoint         |
| Ban applied                    | WARNING | Client IP, ban tier, expiry                |
| Processing error               | ERROR   | Error type, message, stack trace           |

**Logging Requirements for `sync-article-embeddings` (INFRA-014)**:

| Event                          | Level   | Details                                    |
| ------------------------------ | ------- | ------------------------------------------ |
| Sync started                   | INFO    | Trigger source, timestamp                  |
| Articles scanned               | INFO    | Count per collection (technical, blog, others) |
| Embeddings generated           | INFO    | Count of new/updated embeddings, Gemini API calls made |
| Embeddings skipped (unchanged) | INFO    | Count of articles with unchanged hash      |
| Orphans deleted                | INFO    | Count of vector documents removed          |
| Sync duration                  | INFO    | Total processing time in ms                |
| Gemini API error               | ERROR   | Error type, message, article ID            |
| Sync error                     | ERROR   | Error type, message, stack trace           |

---

### OBS-002: Anonymous Visitor Tracking

**Purpose**: Understand site usage patterns without identifying individual users.

**Endpoint**: `POST /t` (authenticated via JWT Bearer Token — see SEC-003A in [06-security-specifications.md](06-security-specifications.md))

**Data Collected**:

| Data Point          | Source                          | Storage                          |
| ------------------- | ------------------------------- | -------------------------------- |
| Page visited        | Frontend sends via `POST /t`    | Structured log → BigQuery        |
| Referrer URL        | Frontend sends in request body  | Structured log → BigQuery        |
| Browser + version   | Frontend sends in request body  | Structured log → BigQuery        |
| IP address          | Request source IP / `X-Forwarded-For` | Structured log → BigQuery  |
| User action         | Frontend sends action type      | Structured log → BigQuery        |
| Connection speed    | Frontend sends from Navigator.connection API | Structured log → BigQuery |
| Timestamp           | Server-side UTC timestamp       | Structured log → BigQuery        |

**Privacy Measures**:

- No cookies or session tracking
- No fingerprinting beyond IP + User-Agent
- Data auto-expires after 90 days (TTL index) in Firestore
- IP addresses SHALL be truncated before storage: zero the last octet for IPv4 (e.g., `203.0.113.42` → `203.0.113.0`) and zero the last 80 bits for IPv6. This truncation happens in the Go backend before emitting structured log entries (which flow to BigQuery via INFRA-010).
- Tracking and error data is NOT stored in Firestore. The `POST /t` handler emits structured log entries to stdout, which Cloud Logging ingests and routes to BigQuery via log sinks (INFRA-010). BigQuery is the sole persistence layer for tracking and error report data.
- Data is retained for up to 2 years in BigQuery (see INFRA-010 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)).

---

### OBS-003: Frontend Error Reporting

**Purpose**: Capture and analyze client-side errors, especially network performance issues.

**Endpoint**: `POST /t` with `action: "error_report"` (same endpoint as tracking — see BE-API-009 in [03-backend-api-specifications.md](03-backend-api-specifications.md))

**Data Collected**:

| Data Point           | Source                                    |
| -------------------- | ----------------------------------------- |
| Error type           | Frontend error handler                    |
| Error message        | Frontend error handler                    |
| Page URL             | `window.location.href`                    |
| Browser + version    | `navigator.userAgent`                     |
| IP address           | Server-side from request headers          |
| Connection speed     | `navigator.connection` API (where available) |
| Timestamp            | Server-side UTC timestamp                 |

**Connection Speed Detection**:

```javascript
// Navigator.connection API (limited browser support)
const connection = navigator.connection || navigator.mozConnection || navigator.webkitConnection;
const speedInfo = connection ? {
  effectiveType: connection.effectiveType,  // "4g", "3g", "2g", "slow-2g"
  downlink: connection.downlink,            // Mbps
  rtt: connection.rtt                       // ms
} : null;
```

**Error Categories to Track**:

| Category              | Examples                                    |
| --------------------- | ------------------------------------------- |
| Network errors        | Fetch failed, timeout, DNS resolution       |
| Slow loading          | API response > 3 seconds                   |
| HTTP errors           | Non-200 responses from API                  |
| JavaScript errors     | Uncaught exceptions, promise rejections     |

**Data Retention**: Error report data is stored in BigQuery only (`frontend_error_logs` table) and retained for up to 2 years (see INFRA-010 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)). Error data is NOT written to Firestore. The same IP truncation applies — no full IP addresses are stored.

---

### OBS-004: Cloud Monitoring Metrics

**Custom Metrics** (exported from Go application):

| Metric Name                    | Type      | Labels                    | Description                    |
| ------------------------------ | --------- | ------------------------- | ------------------------------ |
| `api_requests_total`           | Counter   | method, path, status      | Total API request count        |
| `api_request_duration_ms`      | Histogram | method, path              | Request latency distribution   |
| `api_bans_active`              | Gauge     | ban_tier                  | Currently active bans          |
| `db_query_duration_ms`         | Histogram | collection, operation     | Database query latency         |
| `db_connection_pool_active`    | Gauge     | —                         | Active DB connections          |
| `embedding_cache_hits_total`   | Counter   | collection                | Embedding cache hits           |
| `embedding_cache_misses_total` | Counter   | collection                | Embedding cache misses (Gemini API called) |
| `embedding_api_duration_ms`    | Histogram | —                         | Gemini embedding API call latency |
| `vector_search_duration_ms`    | Histogram | collection                | Firestore Native vector search latency |
| `vector_search_candidates`     | Histogram | collection                | Number of candidates returned per vector search |

**Cloud Armor Metrics** (sourced from Cloud Monitoring, not application-level):

| Metric                                          | Source          | Description                                |
| ------------------------------------------------ | --------------- | ------------------------------------------ |
| `cloud_armor/request_count` (blocked_by_policy)  | Cloud Monitoring | Rate limit blocks by Cloud Armor           |
| `cloud_armor/request_count` (allowed/denied)     | Cloud Monitoring | Overall Cloud Armor request decisions      |

> **Note**: Rate limit hit counting is handled by Cloud Armor at the infrastructure level. The application does not have visibility into rate-limited requests (they never reach Cloud Run). Use Cloud Monitoring to query Cloud Armor metrics for rate-limit monitoring and alerting.

**Built-in Cloud Run Metrics** (automatic):

- Request count
- Request latency (P50, P95, P99)
- Instance count
- Container CPU utilization
- Container memory utilization
- Startup latency

---

### OBS-005: Alerting

**Alert Policies**:

| Alert                            | Condition                                   | Notification Channel | Severity |
| -------------------------------- | ------------------------------------------- | -------------------- | -------- |
| High error rate                  | Error rate > 1% over 5 min                  | Email / Slack        | Warning  |
| High latency                    | P95 latency > 2s over 5 min                 | Email / Slack        | Warning  |
| Masked 500 errors               | Any ERROR log with "masked_500" label        | Email / Slack        | Critical |
| Database connection failure     | db_connection_pool_active = 0                | Email / Slack / PagerDuty | Critical |
| Cloud Run memory pressure       | Memory utilization > 80% over 5 min          | Email / Slack        | Warning  |
| High rate limit triggers        | Cloud Armor `blocked_by_policy` count > 100 in 5 min (via Cloud Monitoring) | Email / Slack | Warning |
| Indefinite ban applied          | Log entry for indefinite ban                 | Email               | Info     |
| Sitemap generation failure      | ERROR log from `generate-sitemap` Cloud Function | Email / Slack    | Warning  |
| Gemini embedding API failure    | ERROR log from embedding API call (Cloud Run or Cloud Function) | Email / Slack | Warning |
| Embedding sync failure          | ERROR log from `sync-article-embeddings` Cloud Function | Email / Slack | Warning |
| High embedding cache miss rate  | `embedding_cache_misses_total` > 50 in 5 min | Email               | Info     |

---

### OBS-006: Dashboards

**Operational Dashboard** (Cloud Monitoring):

- Request rate (req/min) by endpoint
- Latency percentiles (P50, P95, P99) by endpoint
- Error rate (%) by status code
- Cloud Run instance count over time
- Database query latency by collection
- Active rate limit bans
- Embedding cache hit/miss ratio
- Gemini embedding API latency (P50, P95)
- Vector search latency by collection
- Vector search candidate count distribution

**Analytics Dashboard** (Looker Studio via BigQuery — see INFRA-010, INFRA-011):

- Unique visitors (distinct IPs per time period)
- Page views over time (total and per-page)
- Top pages by visit count
- Referrer sources breakdown
- Browser distribution
- Geographic distribution (from IP)
- Connection speed analysis
- Frontend error trends (frequency, types, affected pages)
- Error-browser correlation
- Backend error trends and masked 500 tracking
- Cloud Armor activity (rate limit blocks, WAF events)
- Traffic patterns and request latency analysis

**Data Flow**: Cloud Logging → BigQuery log sinks (5 tables) → Looker Studio data connector (service account) → Analytics dashboards.

---

### OBS-007: robots.txt

**Purpose**: Instruct web crawlers on allowed pages and rate limits. Block API-only paths and tracking endpoints from crawlers.

**Content**:

```
User-agent: *
Allow: /
Disallow: /t
Crawl-delay: 10

User-agent: Googlebot
Allow: /
Disallow: /t
Crawl-delay: 5

User-agent: Bingbot
Allow: /
Disallow: /t
Crawl-delay: 5

Sitemap: https://tjmonsi.com/sitemap.xml
```

**Sitemap Generation**:
- The sitemap is generated every 6 hours by a Cloud Scheduler job (see INFRA-008 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)).
- The generated sitemap XML is stored in Firestore (`sitemap` collection, DM-010).
- The `GET /sitemap.xml` API endpoint serves the sitemap from Firestore (see BE-API-011 in [03-backend-api-specifications.md](03-backend-api-specifications.md)).

**Notes**:
- The `robots.txt` is served from the frontend domain (`tjmonsi.com`).
- API-only paths (`api.tjmonsi.com`) are on a separate subdomain. A separate `robots.txt` SHALL be served on `api.tjmonsi.com` via `BE-API-012` (see [03-backend-api-specifications.md](03-backend-api-specifications.md)):

```
User-agent: *
Disallow: /
```

- This blocks all crawlers from the API subdomain entirely.
- The tracking endpoint (`POST /t`) is blocked from the frontend `robots.txt` as well, though crawlers typically only follow GET links.

---

### OBS-008: Request Tracing

**Implementation**:

- Generate a UUID v4 `X-Request-ID` for every incoming request.
- Propagate `X-Request-ID` in all log entries for that request.
- Return `X-Request-ID` as a response header (useful for debugging with users).
- IF distributed tracing is needed later, integrate with Cloud Trace (OpenTelemetry).
