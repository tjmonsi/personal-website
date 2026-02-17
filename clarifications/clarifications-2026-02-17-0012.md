# Clarifications Batch 0012

**Date**: 2026-02-17
**Scope**: CLR-096 through CLR-102
**Status**: Awaiting answers

---

## CLR-096: INFRA-010d Sink Filter Captures Cloud Function Errors in Backend Errors Table 🟡

**Files affected**: `specs/05-infrastructure-specifications.md` (INFRA-010d)

**Description**: The `sink-backend-errors` log sink filter is:
```
resource.type="cloud_run_revision" AND severity>=ERROR AND NOT jsonPayload.log_type="frontend_error"
```

Cloud Functions Gen 2 run on Cloud Run infrastructure and emit logs under `resource.type="cloud_run_revision"`. This means ERROR logs from all four Cloud Functions (`generate-sitemap`, `process-rate-limit-logs`, `cleanup-rate-limit-offenders`, `sync-article-embeddings`) would also land in the `backend_error_logs` BigQuery table — despite this table being described as for "Backend error trend analysis, masked 500 tracking."

The other sinks (INFRA-010c, 010e) avoid this because they filter by `jsonPayload.log_type`, which is only set by the Go backend's `POST /t` handler. INFRA-010d has no such discriminator.

**Option A**: Add a `resource.labels.service_name` filter to INFRA-010d to match only the Go backend Cloud Run service:
```
resource.type="cloud_run_revision" AND resource.labels.service_name="<cloud-run-service-name>" AND severity>=ERROR AND NOT jsonPayload.log_type="frontend_error"
```
(Service name determined during implementation.)

**Option B**: Accept that the table contains both backend API and Cloud Function errors. Update the table purpose description to reflect this broader scope. Filter by `resource.labels.service_name` in Looker Studio queries as needed.

**Answer**:
Use Option A
---

## CLR-097: INFRA-008b and INFRA-008e Missing `roles/run.invoker` in Authentication Description 🟢

**Files affected**: `specs/05-infrastructure-specifications.md` (INFRA-008b, INFRA-008e)

**Description**: Both Cloud Scheduler job configurations specify authentication as "OIDC token (service account with Cloud Functions invoker role)" — referencing only one role. However, Gen 2 Cloud Functions run on Cloud Run infrastructure and require **both** `roles/cloudfunctions.invoker` **and** `roles/run.invoker` for invocation.

INFRA-014b (embedding sync scheduler) correctly specifies both roles. SEC-013 #9 also correctly lists both roles for the `cloud-scheduler-invoker` SA. The inconsistency is only in INFRA-008b and INFRA-008e.

**Suggested Resolution**: Update the Authentication row in both INFRA-008b and INFRA-008e to match INFRA-014b:
> `OIDC token (service account with roles/cloudfunctions.invoker and roles/run.invoker)`

**Option A (Recommended)**: Apply the suggested resolution above.

**Option B**: Keep the current simplified wording and add a cross-reference note to SEC-013 #9 which already documents the correct dual roles.

**Answer**:
Use Option A
---

## CLR-098: OPTIONS Rate-Limiting Exemption Not Enforced at Cloud Armor Level 🟢

**Files affected**: `specs/03-backend-api-specifications.md` (Global API Rules), `specs/05-infrastructure-specifications.md` (INFRA-005), `specs/06-security-specifications.md` (SEC-002, SEC-003)

**Description**: The Global API Rules in spec-03 state: "OPTIONS requests SHALL NOT be subject to rate limiting." However, INFRA-005's Cloud Armor rate limit rule (priority 2000) applies to all requests with no HTTP method filter — Cloud Armor would count OPTIONS requests toward the per-IP rate limit.

Practically, the impact is negligible because browsers cache CORS preflights for up to 24 hours (SEC-006 `max-age: 86400`), so OPTIONS requests contribute minimally to rate counts. But the spec states an absolute "SHALL NOT."

**Option A**: Add a match condition to INFRA-005's rate limit rule to exclude OPTIONS requests using Cloud Armor's `request.method` expression.

**Option B (Recommended)**: Acknowledge that Cloud Armor rate limits all HTTP methods (including OPTIONS) and soften the statement to: "OPTIONS requests SHALL NOT be subject to application-level ban checks. Cloud Armor rate limiting applies to all requests regardless of method; however, CORS preflight caching (max-age 86400) makes OPTIONS rate limiting negligible in practice."

**Answer**:
Use Option B
---

## CLR-099: DM-005 `idx_date_created` Purpose Doesn't Match API Usage 🟢

**Files affected**: `specs/04-data-model-specifications.md` (DM-005)

**Description**: DM-005's `idx_date_created` index has purpose "Date range queries", implying it's used by the API for filtering. However, the `/others` endpoint (BE-API-007) filters on `date_updated`, not `date_created`.

DM-002 (`technical_articles`) has the same `idx_date_created` index with the correct purpose: "Internal use / content management queries". DM-005 should match this pattern.

**Suggested Resolution**: Update DM-005's `idx_date_created` index purpose from "Date range queries" to "Internal use / content management queries" to match DM-002.

**Option A (Recommended)**: Apply the suggested resolution above.

**Answer**:
Use Option A
---

## CLR-100: `X-Request-ID` Exposure Justification References Unspecified Error Report Field 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-006), `specs/02-frontend-specifications.md` (FE-COMP-005), `specs/03-backend-api-specifications.md` (BE-API-009)

**Description**: SEC-006 states: "`X-Request-ID` is exposed so the frontend can programmatically include it in error reports for debugging." However, neither the FE-COMP-005 error report payload nor the BE-API-009 `POST /t` request body schema includes a `request_id` field. The justification references a feature that isn't specified in the payload.

**Option A**: Add an optional `request_id` field (string, max 36 characters) to the FE-COMP-005 error report payload and BE-API-009 validation rules. When the frontend receives an error response with an `X-Request-ID` header, it captures and includes it in the subsequent error report.

**Option B (Recommended)**: Update the SEC-006 note to remove the error report reference. Change the justification to: "`X-Request-ID` is exposed so users or the developer can reference it when reporting or debugging issues (e.g., via browser DevTools or the socials/contact page)." No payload change needed.

**Answer**:
Use Option B
---

## CLR-101: SEC-013 Missing Firebase Functions Service Account Entry 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-013)

**Description**: SEC-013 documents 9 service accounts but does not include Firebase Functions (INFRA-002). SEC-013 states: "THE SYSTEM SHALL NOT use default service accounts for any component."

Firebase Functions deployed via `firebase deploy` typically uses the Firebase/App Engine default SA. Since INFRA-002 only serves static HTML and accesses no GCP resources, the default SA is functionally harmless — but it technically violates the SEC-013 constraint.

**Option A**: Add a dedicated SA entry (#10) for Firebase Functions with no additional IAM roles beyond default Cloud Functions execution permissions. Managed by Firebase CLI.

**Option B (Recommended)**: Add a scoped exception note to SEC-013: "Firebase Functions (INFRA-002) uses the Firebase default service account because it only serves static HTML content and does not access any GCP resources. The default SA is acceptable for this component since it has no meaningful permissions to abuse."

**Answer**:
Use Option B
---

## CLR-102: Cost Estimate May Underestimate LB Forwarding Rules with IPv6 🟡

**Files affected**: `docs/cost-estimate-draft.md`, `specs/05-infrastructure-specifications.md` (INFRA-004, INFRA-017)

**Description**: The cost estimate assumes **1 forwarding rule** at $0.025/hour (~$18/month) for the Cloud Load Balancer. However, INFRA-017 (Cloud DNS) specifies both A (IPv4) and AAAA (IPv6) records for `tjmonsi.com` and `api.tjmonsi.com`. For a global external Application Load Balancer, IPv6 may require a **separate forwarding rule** with its own IPv6 address — which would mean 2 forwarding rules (~$36/month).

Additionally, an HTTP-to-HTTPS redirect (port 80→443) at the LB level would require another forwarding rule. The spec does not clarify whether HTTP redirects are handled at the LB or only via HSTS headers.

**Option A**: Keep IPv6 support and update the cost estimate to 2 forwarding rules (~$36/month for the LB). Clarify whether HTTP→HTTPS redirect is handled at the LB (3rd rule) or solely via HSTS after first visit.

**Option B**: Remove AAAA records from INFRA-017 if IPv6 is not a priority. Keep cost at 1 forwarding rule (~$18/month). IPv6 can be added later.

**Option C (Recommended)**: Verify with current GCP documentation whether the global external Application Load Balancer can serve both IPv4 and IPv6 traffic from a single forwarding rule. The Global External Application Load Balancer automatically provides both IPv4 and IPv6 addresses with a single forwarding rule — if confirmed, document this finding and keep the cost as-is.

**Answer**:
Use Option C
---

## Summary

| CLR | Severity | Category | Brief Description |
|-----|----------|----------|-------------------|
| CLR-096 | 🟡 IMPORTANT | Data Quality | INFRA-010d sink filter captures Cloud Function errors in backend errors table |
| CLR-097 | 🟢 SUGGESTION | Consistency | INFRA-008b/008e missing `roles/run.invoker` in auth description |
| CLR-098 | 🟢 SUGGESTION | Config Gap | OPTIONS rate-limiting exemption not enforced at Cloud Armor level |
| CLR-099 | 🟢 SUGGESTION | Documentation | DM-005 `idx_date_created` purpose doesn't match API usage |
| CLR-100 | 🟢 SUGGESTION | Loose End | `X-Request-ID` exposure justification references unspecified error report field |
| CLR-101 | 🟢 SUGGESTION | Completeness | SEC-013 missing Firebase Functions service account entry |
| CLR-102 | 🟡 IMPORTANT | Cost Estimate | LB forwarding rule count may underestimate cost with IPv6 |
