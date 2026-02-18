````markdown
# Clarifications Batch 0015

**Date**: 2026-02-18
**Scope**: CLR-113 through CLR-120
**Status**: Awaiting answers

---

## CLR-113: Observability Spec Not Updated for Breadcrumb Fields 🟡

**Files affected**: `specs/07-observability-specifications.md` (OBS-001, OBS-003)

**Description**: The 2026-02-18 update added breadcrumb data to the frontend and backend specs, but the observability spec (v2.9, last updated 2026-02-17) was not updated:

1. **OBS-001 (Structured Logging)**: The structured log schema does not list `server_breadcrumbs` as a field. BE-BREADCRUMB (spec-03) adds a `server_breadcrumbs` array to error log entries with entries like `request_received`, `validation_passed`, `db_query_start`, `db_query_complete`, `upstream_call`, `response_sent`. This field should appear in the OBS-001 log schema for error-severity entries.

2. **OBS-003 (Frontend Error Reporting)**: The section describes error categories but does not mention the `breadcrumbs` array added to the `error_report` POST /t payload (spec-03 BE-API-005). The `breadcrumbs` array (max 50 entries from FE-COMP-013) is now part of every error report and flows to the `frontend_error_logs` BigQuery table via the log sink (INFRA-010c).

3. **BigQuery table schemas**: Neither the observability spec nor the infrastructure spec explicitly defines the column schemas for `frontend_error_logs` or `backend_error_logs` BigQuery tables. With breadcrumbs added, these schemas now include nested/repeated fields. While BigQuery auto-infers schema from JSON log payloads, explicit documentation would prevent surprises.

**Option A (Recommended)**: Update OBS-001 and OBS-003:
- Add `server_breadcrumbs` (array, present on ERROR-severity entries only) to OBS-001 structured log schema
- Add `breadcrumbs` (array, from client error_report payload) to OBS-003 error reporting data description
- Add a note in OBS-003 that the `frontend_error_logs` BigQuery table will contain both `breadcrumbs` (client) and `server_breadcrumbs` (server) as nested JSON arrays
- Bump spec version to v3.0

**Option B**: Leave OBS-001 and OBS-003 as-is. Breadcrumb fields are documented in specs 02 and 03; the observability spec references those specs for payload details.

**Answer**:
Use A.

---

## CLR-114: Input Validation Rules for Breadcrumbs Array in POST /t 🟡

**Files affected**: `specs/06-security-specifications.md` (SEC-001), `specs/03-backend-api-specifications.md` (BE-API-005)

**Description**: SEC-001 defines input validation rules for POST /t payload fields (e.g., `url` max 2048 chars, `referrer` max 2048 chars, `user_agent` max 512 chars), but the new `breadcrumbs` field (added 2026-02-18) has no validation constraints. Without limits, a malicious client could send:

- An arbitrarily long breadcrumbs array (memory exhaustion)
- Individual breadcrumb entries with oversized fields
- Entries with unexpected types or malformed timestamps

FE-COMP-013 specifies a client-side max of 50 entries, but server-side validation is needed since client behavior cannot be trusted.

**Option A (Recommended)**: Add breadcrumbs validation to SEC-001 and BE-API-005:
- `breadcrumbs`: optional array, max 50 entries
- Each entry: object with fields:
  - `type`: string, max 50 chars, required
  - `label`: string, max 200 chars, required
  - `timestamp`: ISO 8601 string, required
  - `metadata`: optional object, max 500 bytes serialized
- If `breadcrumbs` is present but exceeds limits, truncate to 50 entries (keep newest) and log a warning; do not reject the request
- Total `breadcrumbs` payload size: max 50 KB

**Option B**: Add only array length validation (max 50 entries), no per-entry field validation. Trust that client-side code sends well-formed entries.

**Option C**: No server-side validation for breadcrumbs. It's debug data and doesn't affect business logic.

**Answer**:
Use A.
---

## CLR-115: Missing Acceptance Criteria for New Components 🟡

**Files affected**: `specs/02-frontend-specifications.md`, `specs/03-backend-api-specifications.md`

**Description**: Four components added in the 2026-02-18 update have no acceptance criteria:

1. **FE-COMP-005-RETRY** (Error Retry with Exponential Backoff) — No AC verifying retry behavior, delay calculation, or non-retryable status handling.

2. **FE-COMP-005-MODAL** (Error Modal Fallback) — No AC verifying modal display conditions, button actions, or accessibility features.

3. **FE-COMP-013** (Activity Breadcrumb Trail) — No AC verifying breadcrumb capture, max entry limit, or attachment to error reports.

4. **BE-BREADCRUMB** (Server-Side Request Breadcrumb Trail) — No AC verifying that server breadcrumbs are added to error log entries or that the Go context carries breadcrumb data.

**Option A (Recommended)**: Add acceptance criteria for all four:
- **AC-FE-025**: Given a retryable API error (429, 500, 502, 503, 504), when the request fails, then the system retries up to 3 times with exponential backoff (1s, 2s, 4s ±20% jitter) before falling through to error handling.
- **AC-FE-026**: Given all retries are exhausted or a critical error occurs, when the error modal is shown, then it displays an error summary with Retry, Dismiss, and Report action buttons, and the modal is accessible (focus trap, Escape to close, ARIA attributes).
- **AC-FE-027**: Given a user navigates the SPA, when user actions occur (page views, link clicks, errors), then breadcrumb entries are stored in memory (max 50, FIFO), and when an error report is sent, the breadcrumbs array is attached to the POST /t payload.
- **AC-API-023**: Given an error occurs during request processing, when the error log entry is emitted, then it includes a `server_breadcrumbs` array containing timestamped processing steps (e.g., `request_received`, `validation_passed`, `db_query_start`, `response_sent`).

**Option B**: Add ACs only for the two IMPORTANT components (FE-COMP-005-RETRY and BE-BREADCRUMB). Skip modal and breadcrumb trail ACs since they are lower-risk UI components.

**Option C**: No new ACs. The existing ACs for FE-COMP-005 and BE-API-005 implicitly cover the new sub-components.

**Answer**:
Use A. But to add, can we breakdown Acceptance criteria for both Frontend and Backend specs on each functionality and then retain the existing ones as more of a holistic acceptance criteria.
---

## CLR-116: Error Modal "Report" Button Recursive Failure Risk 🟡

**Files affected**: `specs/02-frontend-specifications.md` (FE-COMP-005-MODAL)

**Description**: The error modal includes a "Report" button that sends a `POST /t` with `action: "error_report"`. However, if the original error that triggered the modal was itself a `POST /t` failure (e.g., tracking endpoint is unreachable), the "Report" button would attempt another `POST /t`, which would also fail, potentially triggering another modal — creating a recursive failure loop.

**Scenario**:
1. User visits a page → SPA sends `POST /t` (page_view) → fails (network error)
2. Retry exhausted → Error modal shown
3. User clicks "Report" → SPA sends `POST /t` (error_report) → also fails
4. New error → Should it show another modal? → Infinite loop

**Option A (Recommended)**: Add a guard to the spec:
- If the original error was from a `POST /t` request, the "Report" button SHALL be disabled or hidden (since reporting about a tracking failure via tracking is circular)
- Add a note: "Error reports about POST /t failures are fire-and-forget with no retry and no modal on failure"

**Option B**: Add a recursion depth limit: the error modal SHALL NOT trigger another error modal. If the "Report" POST /t fails, silently discard the failure. No UI feedback.

**Option C**: The "Report" button always fires POST /t as fire-and-forget (no retry, no error handling). If it fails, the failure is silently ignored. This is the simplest approach.

**Answer**:
Use A.
---

## CLR-117: Error Modal "Retry" Button Interaction with Exhausted Retry Counter 🟢

**Files affected**: `specs/02-frontend-specifications.md` (FE-COMP-005-MODAL, FE-COMP-005-RETRY)

**Description**: FE-COMP-005-MODAL includes a "Retry" action button. FE-COMP-005-RETRY defines a 3-retry limit with exponential backoff. When the modal appears (after all retries are exhausted), the "Retry" button's behavior is ambiguous:

1. Does clicking "Retry" restart the full retry sequence (3 retries with backoff)?
2. Does it make a single immediate attempt (no backoff)?
3. Does it restart with a fresh retry counter but keep the same backoff timing?

**Option A**: "Retry" makes a **single immediate attempt** (no backoff, no retry counter). If it fails, the modal re-appears with the same options.

**Option B (Recommended)**: "Retry" restarts the **full retry sequence** (3 retries with exponential backoff). This gives the best chance of recovery after a transient issue, while the user has explicitly opted in. If all retries fail again, the modal re-appears.

**Option C**: "Retry" is a single attempt with a **5-second delay** as a compromise between immediate and full backoff.

**Answer**:
Remove Retry button. The button i said for the modal was to copy the error log so that they can send it to me via contact information through my socials page. THey can also check the privacy page if they want to know what the error logs would be about. They can also see the error logs on the screen to know what they are copying. Because we know that it is currently unreliable through automated setups that we have, they are implored to share the logs to me via the socials page. 
---

## CLR-118: Privacy Policy Disclosure of Breadcrumb Data in Error Reports 🟢

**Files affected**: `specs/02-frontend-specifications.md` (FE-PAGE-009)

**Description**: FE-COMP-013 captures an activity breadcrumb trail (page paths, timestamps, action types) and attaches it to error reports. While breadcrumbs contain no PII (no form inputs, no user identifiers), they do reveal user navigation patterns within the SPA session. FE-PAGE-009 mentions "error data" in its Data We Collect section, but does not specifically disclose breadcrumb collection.

GDPR and privacy best practices favor explicit disclosure of all data collection, even when the data is non-identifying.

**Option A (Recommended)**: Add to FE-PAGE-009 "Data We Collect" section under error data:
- *"When an error occurs, a summary of your recent navigation within the site (pages visited, timestamps, and general actions such as link clicks) may be included in the error report to help diagnose the issue. This data contains no personal information and is discarded after analysis."*

**Option B**: No change. Breadcrumbs are a subset of "error data" already disclosed, and contain no PII. The existing error data disclosure is sufficient.

**Answer**:
Use A.
---

## CLR-119: Error Modal vs Toast Error Display Criteria 🟢

**Files affected**: `specs/02-frontend-specifications.md` (FE-COMP-005-MODAL)

**Description**: FE-COMP-005-MODAL states the modal is displayed when *"a toast/notification system is not available or the error is critical"*. The term "critical" is not defined. Without a clear definition, developers may apply inconsistent criteria for when to show the modal vs a toast.

**Option A (Recommended)**: Define "critical error" in the spec:
- **Critical errors** (show modal): All retries exhausted, complete network failure (navigator.onLine === false), backend returned HTTP 500 on a page-load-blocking request (e.g., article detail fetch), or JavaScript runtime error that prevents page rendering.
- **Non-critical errors** (show toast): Background tracking failures, non-blocking API request failures (e.g., category prefetch), individual image load failures.

**Option B**: Remove the "critical" distinction entirely. Always show toast first; if the error persists after retries, then show the modal. The modal is always a fallback after retry exhaustion, never a first-choice display.

**Option C**: Keep the wording vague and let the developer decide per case during implementation.

**Answer**:
Use A. But to add, show toast first if we can still send error log data to backend - if and only if retries to sending error log data to backend fails, we show a modal of the error log data to the user asking them to send it via social contacts.
---

## CLR-120: Cost Estimate Version Bump for 2026-02-18 Changes 🟢

**Files affected**: `docs/cost-estimate-draft.md`

**Description**: The cost estimate is v1.0-draft (last updated 2026-02-17). The 2026-02-18 spec changes added breadcrumb tracking, error retry with exponential backoff, error modal, and server-side breadcrumbs. These features have **negligible cost impact**:

- **Breadcrumbs**: Increase error report payload size by ~2-5 KB per report. At ~100 reports/month (normal) or ~5,000/month (spike), the additional BigQuery ingestion is <1 MB/month. No cost impact.
- **Error retry**: Up to 3 additional requests per failed request. At ~100 errors/month, worst case is 300 extra requests. Negligible Cloud Run CPU cost.
- **Error modal**: Pure frontend component. No backend cost.
- **Server-side breadcrumbs**: Slight increase in log entry size for error-severity logs. Negligible at current volumes.

**Option A (Recommended)**: Bump to v1.1-draft, add a changelog entry noting the 2026-02-18 features and their negligible cost impact. No changes to cost tables or totals.

**Option B**: No version bump. The cost impact is zero, so no documentation update is needed.

**Answer**:
Use A.
---

## Summary

| CLR | Severity | Category | Brief Description |
|-----|----------|----------|-------------------|
| CLR-113 | 🟡 IMPORTANT | Observability | OBS-001/OBS-003 missing breadcrumb fields and BigQuery schema documentation |
| CLR-114 | 🟡 IMPORTANT | Security | No input validation rules for POST /t breadcrumbs array |
| CLR-115 | 🟡 IMPORTANT | Testing | Missing acceptance criteria for 4 new components |
| CLR-116 | 🟡 IMPORTANT | Frontend | Error modal "Report" button recursive POST /t failure loop |
| CLR-117 | 🟢 SUGGESTION | Frontend | Error modal "Retry" button interaction with exhausted retry counter |
| CLR-118 | 🟢 SUGGESTION | Privacy | Privacy policy doesn't disclose breadcrumb data in error reports |
| CLR-119 | 🟢 SUGGESTION | Frontend | "Critical error" criteria undefined for modal vs toast decision |
| CLR-120 | 🟢 SUGGESTION | Cost Estimate | Version bump for 2026-02-18 changes (negligible cost impact) |
````
