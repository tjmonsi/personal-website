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
  "cloud_run_instance": "instance-abc123"
}
```

**Log Levels**:

| Level   | Usage                                                     |
| ------- | --------------------------------------------------------- |
| DEBUG   | Detailed execution flow (disabled in production)          |
| INFO    | Normal operations: request completed, cache hit/miss      |
| WARNING | Non-critical issues: slow query, approaching rate limit   |
| ERROR   | Failures: database errors, masked 500s                    |
| CRITICAL| System-level failures: database connection lost           |

**Mandatory Logging Events**:

- Every incoming request (INFO): method, path, status, latency
- Rate limit triggered (WARNING): client identifier, endpoint, offense count
- Rate limit ban applied (WARNING): client identifier, ban tier, expiry
- Internal error masked as 404 (ERROR): full error details
- Database query errors (ERROR): query details (sanitized), error message
- Slow queries > 1 second (WARNING): query details, duration

---

### OBS-002: Anonymous Visitor Tracking

**Purpose**: Understand site usage patterns without identifying individual users.

**Data Collected**:

| Data Point          | Source                          | Storage              |
| ------------------- | ------------------------------- | -------------------- |
| Page visited        | Frontend sends to tracking API  | `tracking` collection|
| Referrer URL        | `Referer` request header        | `tracking` collection|
| Browser + version   | `User-Agent` request header     | `tracking` collection|
| IP address          | Request source IP / `X-Forwarded-For` | `tracking` collection|
| User action         | Frontend sends action type      | `tracking` collection|
| Timestamp           | Server-side UTC timestamp       | `tracking` collection|

**Privacy Measures**:

- No cookies or session tracking
- No fingerprinting beyond IP + User-Agent
- Data auto-expires after 90 days (TTL index)
- IP addresses may be truncated (e.g., zero last octet for IPv4)

---

### OBS-003: Frontend Error Reporting

**Purpose**: Capture and analyze client-side errors, especially network performance issues.

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

**Data Retention**: 30 days (TTL auto-expiry).

---

### OBS-004: Cloud Monitoring Metrics

**Custom Metrics** (exported from Go application):

| Metric Name                    | Type      | Labels                    | Description                    |
| ------------------------------ | --------- | ------------------------- | ------------------------------ |
| `api_requests_total`           | Counter   | method, path, status      | Total API request count        |
| `api_request_duration_ms`      | Histogram | method, path              | Request latency distribution   |
| `api_rate_limit_hits_total`    | Counter   | path, client_type         | Rate limit triggers            |
| `api_bans_active`              | Gauge     | ban_tier                  | Currently active bans          |
| `db_query_duration_ms`         | Histogram | collection, operation     | Database query latency         |
| `db_connection_pool_active`    | Gauge     | —                         | Active DB connections          |

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
| High rate limit triggers        | rate_limit_hits > 100 in 5 min               | Email / Slack        | Warning  |
| Indefinite ban applied          | Log entry for indefinite ban                 | Email               | Info     |

---

### OBS-006: Dashboards

**Operational Dashboard** (Cloud Monitoring):

- Request rate (req/min) by endpoint
- Latency percentiles (P50, P95, P99) by endpoint
- Error rate (%) by status code
- Cloud Run instance count over time
- Database query latency by collection
- Active rate limit bans

**Analytics Dashboard** (built from tracking data):

- Page views over time
- Top pages by visit count
- Referrer sources
- Browser distribution
- Geographic distribution (from IP)

---

### OBS-007: robots.txt

**Purpose**: Instruct web crawlers on allowed pages and rate limits.

**Proposed Content**:

```
User-agent: *
Allow: /
Crawl-delay: 10

User-agent: Googlebot
Allow: /
Crawl-delay: 5

User-agent: Bingbot
Allow: /
Crawl-delay: 5

Sitemap: https://tjmonserrat.com/sitemap.xml
```

> **Note**: See Clarification CLR-009 for specific crawler rules.

---

### OBS-008: Request Tracing

**Implementation**:

- Generate a UUID v4 `X-Request-ID` for every incoming request.
- Propagate `X-Request-ID` in all log entries for that request.
- Return `X-Request-ID` as a response header (useful for debugging with users).
- IF distributed tracing is needed later, integrate with Cloud Trace (OpenTelemetry).
