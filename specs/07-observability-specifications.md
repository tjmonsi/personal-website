---
title: Observability Specifications
version: 2.9
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [observability, logging, monitoring, tracking, vector-search, terraform, alerting, sli, slo]
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
  "client_ip": "203.0.113.0",
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
> - `"frontend_tracking"` — for `POST /t` requests with `action: "page_view"`, `"link_click"`, or `"time_on_page"`.
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
- Every `POST /t` tracking request (INFO): include `log_type: "frontend_tracking"`, the `visitor_id` (server-computed hash), and the tracking payload fields (page, referrer, action, browser, connection_speed, plus action-specific fields such as `clicked_url` for `link_click` and `milestone` for `time_on_page`) in the structured log entry
- Every `POST /t` error report (INFO): include `log_type: "frontend_error"`, the `visitor_id`, and the error payload fields (error_type, error_message, page, browser, connection_speed) in the structured log entry
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

**Logging Requirements for `cleanup-rate-limit-offenders` (INFRA-008d)**:

| Event                          | Level   | Details                                    |
| ------------------------------ | ------- | ------------------------------------------ |
| Cleanup started                | INFO    | Trigger source, timestamp                  |
| Records evaluated              | INFO    | Total records scanned                      |
| Records deleted                | INFO    | Count of expired records removed           |
| Records retained               | INFO    | Count of records kept (active bans, indefinite bans) |
| Cleanup duration               | INFO    | Total processing time in ms                |
| Cleanup error                  | ERROR   | Error type, message, stack trace           |

---

### OBS-002: Anonymous Visitor Tracking

**Purpose**: Understand site usage patterns without identifying individual users.

**Endpoint**: `POST /t` (authenticated via JWT Bearer Token — see SEC-003A in [06-security-specifications.md](06-security-specifications.md))

**Data Collected**:

| Data Point            | Source                          | Storage                          |
| --------------------- | ------------------------------- | -------------------------------- |
| Visitor ID            | Server-computed SHA-256 hash of `visitor_session_id` + truncated IP + User-Agent (see BE-API-009) | Structured log → BigQuery |
| Visitor session ID    | Frontend-generated UUID v4 per browser session (sessionStorage) | Structured log → BigQuery |
| Page visited          | Frontend sends via `POST /t`    | Structured log → BigQuery        |
| Referrer URL          | Frontend sends in request body  | Structured log → BigQuery        |
| Browser + version     | Frontend sends in request body  | Structured log → BigQuery        |
| IP address            | Request source IP / `X-Forwarded-For` | Structured log → BigQuery  |
| User action           | Frontend sends action type (`page_view`, `link_click`, `time_on_page`) | Structured log → BigQuery |
| Clicked URL           | Frontend sends target URL (for `link_click` action only) | Structured log → BigQuery |
| Time-on-page milestone| Frontend sends milestone reached: `1min`, `2min`, `5min` (for `time_on_page` action only) | Structured log → BigQuery |
| Connection speed      | Frontend sends from Navigator.connection API | Structured log → BigQuery |
| Timestamp             | Server-side UTC timestamp       | Structured log → BigQuery        |

**Privacy Measures**:

- No cookies or persistent cross-session tracking.
- The `visitor_session_id` is a random UUID v4 generated per browser session and stored in `sessionStorage`. It does not persist across sessions or tabs.
- The `visitor_id` is a SHA-256 hash of (`visitor_session_id` + truncated IP + User-Agent). It is non-reversible and cannot be used to identify real-world individuals. It exists solely to compute unique visitor counts and distinguish human browsing patterns from automated bot traffic.
- No fingerprinting beyond truncated IP + User-Agent + session-scoped random ID.
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
| `image_cache_hits_total`       | Counter   | —                         | Image cache hits (CLR-108)             |
| `image_cache_misses_total`     | Counter   | —                         | Image cache misses (Cloud Storage fetch) (CLR-108) |

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

**Notification Channels**:

| Channel  | Destination                         | Notes                                      |
| -------- | ----------------------------------- | ------------------------------------------ |
| Email    | `<alert-email>` *(obfuscated)*      | Primary notification channel for all alerts |

> **Note**: Email is the only notification channel. Slack and PagerDuty channels are not used for this personal website. All alerts are sent to the specified email address via Cloud Monitoring notification channels.

**Alert Policies**:

| Alert                            | Condition                                   | Notification Channel | Severity |
| -------------------------------- | ------------------------------------------- | -------------------- | -------- |
| High error rate                  | Error rate > 1% over 5 min                  | Email                | Warning  |
| High latency                    | P95 latency > 2s over 5 min                 | Email                | Warning  |
| Masked 500 errors               | Any ERROR log with "masked_500" label        | Email                | Critical |
| Database connection failure     | db_connection_pool_active = 0                | Email                | Critical |
| Cloud Run memory pressure       | Memory utilization > 80% over 5 min          | Email                | Warning  |
| High rate limit triggers        | Cloud Armor `blocked_by_policy` count > 100 in 5 min (via Cloud Monitoring) | Email | Warning |
| Indefinite ban applied          | Log entry for indefinite ban                 | Email                | Info     |
| Sitemap generation failure      | ERROR log from `generate-sitemap` Cloud Function | Email            | Warning  |
| Gemini embedding API failure    | ERROR log from embedding API call (Cloud Run or Cloud Function) | Email | Warning |
| Embedding sync failure          | ERROR log from `sync-article-embeddings` Cloud Function | Email     | Warning |
| Offender cleanup failure        | ERROR log from `cleanup-rate-limit-offenders` Cloud Function | Email | Warning |
| High embedding cache miss rate  | `embedding_cache_misses_total` > 50 in 5 min | Email                | Info     |
| Backend error log               | Any ERROR severity log from Cloud Run backend (INFRA-003) | Email   | Warning  |
| Cloud Function error log        | Any ERROR severity log from any Cloud Function (INFRA-008a, 008c, 008d, INFRA-014) | Email | Warning |
| Frontend error log              | Log entry with `jsonPayload.log_type="frontend_error"` from Cloud Run backend (any severity, matching INFRA-010c sink filter) | Email | Warning |
| SLI availability breach         | SLI availability burn rate exceeds threshold over 1 hour (see OBS-010) | Email | Critical |

> **Note on alert overlap**: Some generic catch-all alerts ("Backend error log", "Cloud Function error log", "Frontend error log") intentionally overlap with specific alerts (e.g., "Sitemap generation failure", "Masked 500 errors"). This is defense-in-depth — the generic alerts act as a safety net in case a specific alert condition misses an error pattern. For a personal website with email-only notifications, receiving 2 emails per event is an acceptable trade-off for comprehensive coverage.

**Cost**: Cloud Monitoring alerting is free for up to 500 alert policies and up to 6 notification channels. Email notifications have no per-message cost. See [Cloud Monitoring pricing](https://cloud.google.com/stackdriver/pricing).

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

- Unique visitors (distinct `visitor_id` values per time period)
- Page views over time (total and per-page)
- Top pages by visit count
- Referrer sources breakdown
- Browser distribution
- Geographic distribution (from IP)
- Connection speed analysis
- Link click analysis (most-clicked URLs, click-through rate per page)
- Time-on-page engagement (percentage of visitors reaching 1 min, 2 min, 5 min milestones per page)
- Content helpfulness score (derived from engagement milestones and link click depth)
- Bot vs. human traffic discrimination (visitors with no `time_on_page` events, abnormal page view velocity)
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

---

### OBS-009: Terraform & Infrastructure Operations

**Purpose**: Acknowledge the observability of Terraform and infrastructure change operations.

Terraform operations (`terraform plan`, `terraform apply`) are observable through existing mechanisms without dedicated custom dashboards or alerts:

| Observability Source | What It Captures | Retention |
| -------------------- | ---------------- | --------- |
| GitHub Actions CI/CD logs | Full `terraform plan` and `terraform apply` output, including resource changes, errors, and timing | GitHub Actions default retention (90 days) |
| Cloud Audit Logs (Admin Activity) | All GCP API calls made by the Terraform service account (`terraform-builder@`), including resource creation, modification, and deletion | 400 days (auto-retained by GCP, no cost) |
| Cloud Audit Logs (Data Access) | Read operations by the Terraform SA (if enabled) | Configurable retention |
| Git commit history | Terraform configuration changes (`.tf` files in `/terraform/`) | Permanent (Git) |

> **Note**: For a personal website, GitHub Actions logs and Cloud Audit Logs provide sufficient visibility into infrastructure changes. No custom Terraform-specific dashboards or alert policies are defined at this time. If drift detection or Terraform-specific alerting becomes necessary, this section can be expanded.

---

### OBS-010: Service Level Indicators (SLI) & Objectives (SLO)

**Purpose**: Define availability targets for the backend API to measure service reliability and trigger alerts when objectives are breached.

**Backend Availability SLI**:

| Setting              | Value                                              |
| -------------------- | -------------------------------------------------- |
| Service              | Cloud Run backend API (INFRA-003)                  |
| SLI type             | Availability (request-based)                       |
| Good request         | HTTP response status < 500 (i.e., non-5xx responses) |
| Total requests       | All HTTP requests to the Cloud Run backend         |
| Measurement          | Cloud Monitoring `run.googleapis.com/request_count` metric with `response_code_class` filter |

**SLO Target**:

| Setting              | Value                                              |
| -------------------- | -------------------------------------------------- |
| Availability target  | 99.9% uptime                                       |
| Compliance period    | Rolling 28-day window (CLR-107)                    |
| Error budget         | 0.1% of requests may be 5xx (e.g., ~12 5xx errors per 12,000 requests/month in normal traffic) |

**Alert**:

| Alert                          | Condition                                      | Notification | Severity |
| ------------------------------ | ---------------------------------------------- | ------------ | -------- |
| SLI availability breach        | Burn rate exceeds threshold over 1 hour (error budget exhaustion) | Email (`<alert-email>`) | Critical |

**Implementation**:

- Use Cloud Monitoring [Service Monitoring](https://cloud.google.com/monitoring/service-monitoring) to define the SLI and SLO.
- The SLO is defined on the Cloud Run service resource.
- An alert policy fires when the burn rate indicates the error budget will be exhausted (fast-burn alert over 1 hour window).
- This is the only SLI/SLO defined for this personal website. Additional SLIs (latency, etc.) can be added later if needed.

**Cost**: Cloud Monitoring SLO monitoring is free. No additional cost beyond the existing Cloud Monitoring free tier. See [Cloud Monitoring pricing](https://cloud.google.com/stackdriver/pricing).

> **Note**: The 99.9% target is aspirational for a personal website with scale-to-zero Cloud Run. Cold starts will occasionally cause elevated latency but should not produce 5xx errors. The SLO primarily guards against persistent backend failures (e.g., misconfiguration, dependency outages).

---

### Acceptance Criteria

- **AC-OBS-001**: Given the Go backend processes a `POST /t` with `action: "page_view"`, when the request is handled, then a structured log entry is emitted to stdout with `log_type: "frontend_tracking"` containing all tracking payload fields.
- **AC-OBS-002**: Given a `POST /t` with `action: "error_report"`, when the request is handled, then a structured log entry is emitted with `log_type: "frontend_error"` routed to `frontend_error_logs` BigQuery table.
- **AC-OBS-003**: Given the `robots.txt` file at `tjmonsi.com/robots.txt`, when crawled, then it includes `Sitemap: https://tjmonsi.com/sitemap.xml` and disallows `/t`.
- **AC-OBS-004**: Given the Cloud Monitoring alerting (OBS-005), when email notification channels are configured, then alert policies fire on the defined conditions and send email notifications.
- **AC-OBS-005**: Given the SLI/SLO (OBS-010), when the backend availability drops below 99.9% over the rolling 28-day window, then a fast-burn alert fires.
- **AC-OBS-006**: Given the Looker Studio dashboard (OBS-006), when connected to BigQuery, then it displays unique visitors, page views, popular pages, referrer sources, and browser distribution from `frontend_tracking_logs`.
- **AC-OBS-007**: Given the structured logging format, when any log entry is emitted by the Go backend, then it includes a `request_id` (UUID v4) for correlation.
