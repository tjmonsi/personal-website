## Clarifications Needed

The following items were identified during a consistency review of all spec documents after applying the answers from `clarifications-2026-02-17-0003`.

Items marked **🔴 Blocking** need resolution before implementation can begin. Items marked **🟢 Non-blocking** can proceed with reasonable defaults — answer at your convenience.

---

### CLR-041: Progressive Banning Cannot Detect Cloud Armor Rate Limit Violations 🔴

**Context**: SEC-002 defines progressive banning: after Cloud Armor returns `429` multiple times, the Go application should escalate to blocking the IP (403 → 404). However, when Cloud Armor returns `429`, the request **never reaches Cloud Run** — the Go application never sees the rate-limited request. This means:

1. The Go app cannot record "offenses" in the `rate_limit_offenders` collection (DM-009) because it never observes rate-limited requests.
2. The progressive banning escalation (offenses 1–5 → 30-day block → 90-day block → indefinite) cannot be tracked or enforced.
3. The `offense_history[].endpoint` field in DM-009 cannot be populated.

**Question**: How should the progressive banning system detect Cloud Armor rate limit violations?

**Options**:

- **A**: Add a **Cloud Function that processes Cloud Armor logs** (via a log sink). When Cloud Armor throttles a request, the Cloud Function writes an offense record to DM-009. The Go app continues to check DM-009 for active bans on each request it does receive.
- **B**: Use Cloud Armor's built-in **adaptive protection** for escalating bans (block IPs for longer periods automatically). Remove application-level progressive banning entirely. Remove DM-009 collection.
- **C**: Set Cloud Armor to a **higher threshold** (e.g., 120 req/min) as a safety net, and re-add **lightweight application-level rate counting** in the Go backend (checking Firestore per request) at 60 req/min. The Go app handles both counting and progressive banning. Cloud Armor only catches extreme abuse.
- **D**: Other approach.

**Answer**:
Use A and B.

---

### CLR-042: Cloud Armor Generates Non-JSON 429 Response Body 🔴

**Context**: SEC-002 specifies a custom JSON response for `429`:
```json
{
  "error": "Too many requests. Please try again later.",
  "retry_after": 30
}
```

However, since Cloud Armor generates the `429` response directly at the load balancer level, the response body is Cloud Armor's default format (typically HTML or plain text), not the application's custom JSON. The frontend error handler (FE-COMP-003) may attempt to parse the response as JSON and fail.

**Question**: How should 429 responses be handled given that Cloud Armor generates them?

**Options**:

- **A**: Remove the custom 429 JSON response body from the spec. Ensure the frontend error handler (FE-COMP-003) matches on HTTP status code only and does not attempt to parse the response body for 429s.
- **B**: Configure Cloud Armor to use a [custom error response](https://cloud.google.com/armor/docs/custom-error-responses) that returns the desired JSON body for 429 responses.
- **C**: Other approach.

**Answer**:
Use A and B.
---

### CLR-043: Site Footer Component Not Specified 🟢

**Context**: FE-PAGE-009 (Privacy Policy) states: "THE SYSTEM SHALL link the privacy policy from the site footer on all pages." However, there is no `FE-COMP` specification for a global site footer component. No other page spec mentions a footer. This requirement is orphaned — an implementer wouldn't know what else goes in the footer or how it should look.

**Question**: Should a global site footer component be formally specified?

**Options**:

- **A**: Add a `FE-COMP-010: Site Footer` component with a minimal spec (privacy link, copyright notice, and optionally a link to `/socials`). *(Recommended)*
- **B**: Remove the footer reference from FE-PAGE-009 and place the privacy link in the main navigation instead.
- **C**: Defer footer design to implementation — the privacy link placement is sufficient context.

**Answer**:
Use A.
---

### CLR-044: CORS Preflight Handling for Conditional Request Headers 🟢

**Context**: FE-COMP-008 sends `If-None-Match` and `If-Modified-Since` headers in conditional requests to the backend API (cross-origin from `tjmonserrat.com` to `api.tjmonserrat.com`). SEC-006 defines CORS with allowed methods `GET, POST` and allowed headers `Content-Type, Accept, Authorization`.

The `If-None-Match` and `If-Modified-Since` headers are part of the [CORS safelist](https://developer.mozilla.org/en-US/docs/Glossary/CORS-safelisted_request_header) and should NOT trigger a preflight as long as they meet value constraints. However, the `Authorization` header (used for `POST /t`) IS a non-simple header that triggers preflight for POST requests.

**Question**: Should the backend explicitly handle `OPTIONS` preflight requests?

**Options**:

- **A**: Add `OPTIONS` to the backend's accepted methods for CORS preflight handling. Return appropriate CORS headers for preflight requests (no rate limiting on OPTIONS). Add `If-None-Match, If-Modified-Since` to the `Access-Control-Allow-Headers` list for completeness. *(Recommended)*
- **B**: Rely on the Go CORS middleware to handle preflight automatically — no explicit spec change needed, just an implementation note.
- **C**: Other approach.

**Answer**:
Use A.
---

### CLR-045: Cloud Function Logging and Observability Requirements 🟢

**Context**: OBS-001 specifies structured logging requirements for the Cloud Run backend, but the Cloud Function for sitemap generation (INFRA-008a) is a separate compute unit with no logging spec. If the sitemap generation fails, there is no specification for how errors are logged or alerted.

**Question**: Should logging requirements be added for the sitemap generation Cloud Function?

**Options**:

- **A**: Add logging requirements for the Cloud Function: log generation started, number of URLs processed, Firestore write result, generation duration, and any errors. Add an alert in OBS-005 for sitemap generation failures. *(Recommended)*
- **B**: Rely on Cloud Functions' built-in logging and monitoring — no additional spec needed.

**Answer**:
Use A.
---

### CLR-046: OBS-001 Missing `masked_500` Log Label for Alerting 🟢

**Context**: OBS-005 defines an alert: "Any ERROR log with `masked_500` label." But OBS-001's log schema and mandatory event definitions never specify adding a `masked_500` label to log entries when internal errors are masked as 404. Without this label, the alert cannot fire.

**Question**: Can I add a `labels` field to the OBS-001 log schema and document that masked-500 log entries must include a `masked_500: true` label?

**Options**:

- **A**: Yes — add the label specification. *(Recommended)*
- **B**: No — use a different alerting mechanism (e.g., match on log message pattern).

**Answer**:
Use A.
---

### CLR-047: `rate_limit_offenders` Collection Missing Data Retention Policy 🟢

**Context**: DM-007 (tracking) has a 90-day TTL and DM-008 (error_reports) has a 30-day TTL, but DM-009 (`rate_limit_offenders`) has no data retention or cleanup policy. This collection could grow indefinitely, especially with indefinitely-banned entries.

**Question**: Should DM-009 have a data retention policy?

**Options**:

- **A**: Remove offense records with no active ban and no offenses in the last 90 days. Keep indefinite bans but subject them to periodic review. *(Recommended)*
- **B**: No data retention — the collection is small enough that cleanup isn't needed for a personal website.
- **C**: Other approach.

**Answer**:
Use A.
---

### CLR-048: Rate Limit Metrics and Alerts Need Cloud Armor Source 🟢

**Context**: OBS-004 defines an `api_rate_limit_hits_total` counter metric (application-level), and OBS-005 defines an alert "rate_limit_hits > 100 in 5 min". Since rate limiting is handled by Cloud Armor (not the Go app), the Go application cannot increment this counter — it never sees rate-limited requests. The alert as specified would never fire.

Similarly, OBS-005's "High rate limit triggers" alert references this metric.

**Question**: Should rate limit metrics and alerts be sourced from Cloud Armor instead?

**Options**:

- **A**: Replace the application-level `api_rate_limit_hits_total` metric with Cloud Armor's built-in metrics (e.g., `request_count` with `blocked_by_policy` dimension). Update the OBS-005 alert to use Cloud Armor metrics via Cloud Monitoring. *(Recommended)*
- **B**: Remove rate limit metrics and alerts entirely — Cloud Armor handles it, and detailed monitoring isn't needed for a personal website.

**Answer**:
Use A.


---

## Other Notes:
- use `tjmonsi.com` as the domain in all diagrams and examples instead of `tjmonserrat.com` for brevity