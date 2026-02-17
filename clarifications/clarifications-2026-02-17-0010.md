# Clarifications — 2026-02-17 — Batch 0010

## Architecture Review: Post CLR-073–081 + Alerting/SLI Gap Analysis

The following items were identified during a comprehensive architecture review of all 7 spec documents (v2.4–v2.8) and the cost estimate (v0.7-draft), after applying CLR-073 through CLR-081 and adding OBS-005 email alerting consolidation, OBS-010 SLI/SLO, and cost estimate updates.

Previously resolved clarifications (CLR-001 through CLR-081) were reviewed and excluded.

---

### CLR-082: Frontend Error Log Alert in OBS-005 Will Never Fire 🟡

**Context**: The OBS-005 alert table includes:

> `Frontend error log | Any ERROR severity log posted by frontend via Cloud Logging (OBS-003) | Email | Warning`

However, frontend error reports flow through `POST /t` → Go backend → structured log emitted at **INFO** level (per OBS-001: every `POST /t` error report is logged at INFO severity with `log_type: "frontend_error"`). The frontend never posts directly to Cloud Logging. The Go backend logs these at INFO severity, not ERROR. This alert condition (ERROR severity) doesn't match the actual log routing (INFO severity with `log_type: "frontend_error"`) and would **never fire**.

**Affected specs**: Spec 07 (OBS-005, OBS-001, OBS-003).

**Question**: How should the frontend error log alert condition be fixed?

**Options**:

- **A**: **Fix alert condition** — Change the alert condition to match on `jsonPayload.log_type="frontend_error"` (any severity), consistent with the INFRA-010c BigQuery sink filter. *(Recommended)*
- **B**: **Change log severity** — Change OBS-001 to log `POST /t` error reports at WARNING or ERROR severity instead of INFO, so the ERROR-based alert fires.
- **C**: **Remove the alert** — Frontend errors are already captured by the "Frontend error trends" BigQuery table. Remove the dedicated alert row.

**Recommendation**: Option A — The alert should match the actual log pattern without changing the established logging convention.

**Answer**:
Use Option A.
---

### CLR-083: INFRA-016 API Enablement List Missing Additional Required APIs 🟡

**Context**: The INFRA-016 API enablement list was expanded by CLR-073 to include WIF and Eventarc APIs. However, it still omits at least three additional APIs required for Terraform provisioning:

| Missing API | Required By |
|---|---|
| `cloudscheduler.googleapis.com` | Cloud Scheduler jobs (INFRA-008b, 008e, 014b) |
| `bigquery.googleapis.com` | BigQuery dataset and log sinks (INFRA-010) |
| `cloudbuild.googleapis.com` | Cloud Functions Gen 2 deployment (Gen 2 functions use Cloud Build internally) |

The list uses "e.g." and "etc." qualifiers, but providing an exhaustive list prevents deployment failures during `terraform apply`.

**Affected specs**: Spec 05 (INFRA-016 API enablement list).

**Question**: Should `cloudscheduler.googleapis.com`, `bigquery.googleapis.com`, and `cloudbuild.googleapis.com` be added to the INFRA-016 API enablement list?

**Options**:

- **A**: **Add all three APIs** — Add `cloudscheduler.googleapis.com`, `bigquery.googleapis.com`, and `cloudbuild.googleapis.com` to the API enablement list. *(Recommended)*
- **B**: **Add and make exhaustive** — Add these three APIs and remove the "e.g." and "etc." qualifiers to make the list definitive. Include a note that any additional APIs discovered during implementation should be added.
- **C**: **Leave as-is** — The "etc." qualifier covers implicit APIs.

**Recommendation**: Option A — Explicit is better than implicit for IaC configuration.

**Answer**:
Use Option A.
---

### CLR-084: OBS-005 Alert Overlap Causes Duplicate Email Notifications 🟢

**Context**: The new generic catch-all alerts in OBS-005 overlap with existing specific alerts, causing duplicate emails for the same event:

| Event | Alerts Fired |
|---|---|
| Sitemap generation fails | "Sitemap generation failure" + "Cloud Function error log" |
| Embedding sync fails | "Embedding sync failure" + "Cloud Function error log" |
| Offender cleanup fails | "Offender cleanup failure" + "Cloud Function error log" |
| Gemini API fails in Cloud Run | "Gemini embedding API failure" + "Backend error log" |
| Masked 500 error | "Masked 500 errors" (Critical) + "Backend error log" (Warning) |

For a personal website with email-only alerts, this means 2 emails per event in most failure scenarios.

**Affected specs**: Spec 07 (OBS-005).

**Question**: Should alert overlap be addressed or acknowledged?

**Options**:

- **A**: **Acknowledge overlap** — Add a note to OBS-005 acknowledging intentional overlap for defense-in-depth. The generic alerts act as a safety net in case specific alert conditions miss an error pattern. *(Recommended)*
- **B**: **Add exclusion conditions** — Add exclusion filters to the generic alerts to prevent duplicates (e.g., "Cloud Function error log" excludes errors already matched by specific alerts).
- **C**: **Remove generic alerts** — Remove "Backend error log", "Cloud Function error log", and "Frontend error log" rows. The existing specific alerts are sufficient.

**Recommendation**: Option A — Simple, transparent, and appropriate for a personal website. Overlap is intentional defense-in-depth.

**Answer**:
Use Option A.
---

### CLR-085: Spec-01 Deployment Topology Has Stale "TBD" for Cloud Run Max Instances 🟢

**Context**: The Deployment Topology section in spec-01 states:

> `Cloud Run: Min instances = 0 (scale to zero), Max instances = TBD based on budget`

But INFRA-003 in spec-05 defines `Max instances = 5`. The spec-01 reference is stale.

**Affected specs**: Spec 01 (Deployment Topology).

**Question**: Should the spec-01 Deployment Topology be updated to match INFRA-003's max instances value?

**Options**:

- **A**: **Update to match** — Change to `Max instances = 5` to match INFRA-003. *(Recommended)*
- **B**: **Leave as TBD** — The TBD is intentional to allow budget-driven adjustment during implementation.

**Recommendation**: Option A — INFRA-003 already specifies the value. Spec-01 should be consistent.

**Answer**:
Use Option A.
---

### CLR-086: INFRA-007 Alerting Section Duplicates OBS-005 Without Cross-Reference 🟢

**Context**: INFRA-007 (Cloud Logging) lists 4 alert policies inline (error rate, latency, memory, masked 500s). OBS-005 now defines a comprehensive 16-row alert policy table that supersedes and extends INFRA-007's inline list. An implementer may be confused about which source is authoritative or whether the alerts are separate.

**Affected specs**: Spec 05 (INFRA-007), Spec 07 (OBS-005).

**Question**: Should INFRA-007's inline alert list be replaced with a cross-reference to OBS-005?

**Options**:

- **A**: **Replace with cross-reference** — Remove the inline alert list from INFRA-007 and add: "See OBS-005 in `07-observability-specifications.md` for the complete alerting specification." *(Recommended)*
- **B**: **Keep both** — Keep INFRA-007's inline alerts as a summary and add a note: "Full alerting specification in OBS-005."
- **C**: **Leave as-is** — INFRA-007's alerts are a subset and don't conflict.

**Recommendation**: Option A — Single source of truth for alerting avoids confusion.

**Answer**:
Use Option A.
---

### CLR-087: INFRA-008c Log Sink Missing Exact Cloud Logging Filter Syntax 🟢

**Context**: The INFRA-008c Pub/Sub log sink configuration says:

> `Log sink filter: Cloud Armor logs where the response status is 429 (rate limit exceeded).`

This is human-readable but lacks the exact Cloud Logging filter expression. By contrast, all five BigQuery log sinks (INFRA-010a–010e) provide exact filter syntax (e.g., `resource.type="cloud_run_revision" AND jsonPayload.log_type="frontend_error"`). An engineer implementing the Pub/Sub log sink in Terraform would need to derive the filter expression.

**Affected specs**: Spec 05 (INFRA-008c).

**Question**: Should the exact Cloud Logging filter syntax be added to INFRA-008c?

**Options**:

- **A**: **Add exact filter** — Add the filter expression: `resource.type="http_load_balancer" AND httpRequest.status=429`. *(Recommended)*
- **B**: **Leave as-is** — The human-readable description is sufficient; the engineer can derive the filter during implementation.

**Recommendation**: Option A — Consistency with the BigQuery log sink filter specifications (INFRA-010a–010e).

**Answer**:
Use Option A.
---

## Verified Items — No Gaps Found

| Check | Result |
|-------|--------|
| CLR-073: WIF + Eventarc APIs in INFRA-016 | ✅ `sts.googleapis.com`, `iamcredentials.googleapis.com`, `eventarc.googleapis.com` present |
| CLR-074: WIF bootstrap + content CI/CD scope | ✅ Both added correctly |
| CLR-075: INFRA-011 unique visitors metric | ✅ Now says "Distinct `visitor_id` values per time period" |
| CLR-076: SA #2 article collection read access | ✅ Lists `technical_articles`, `blog_articles` read + `sitemap` read/write |
| CLR-077: INFRA-003 ↔ INFRA-018 cross-reference | ✅ Bidirectional reference present |
| CLR-078: Spec-01 diagram shows Pub/Sub | ✅ Pub/Sub (rate-limit-events) shown between Log Sink and Cloud Function |
| CLR-079: Log sinks distinguish BigQuery vs Pub/Sub | ✅ "Cloud Logging log sinks — BigQuery sinks (INFRA-010a–010e) + Pub/Sub sink for rate-limit events (INFRA-008c)" |
| CLR-080: Terraform CI/CD ↔ OBS-009 cross-ref | ✅ Observability note present in Terraform CI/CD section |
| CLR-081: Redundant invoker roles removed | ✅ SAs #2, #4, #5 no longer list `roles/cloudfunctions.invoker` |
| OBS-010 SLI/SLO section | ✅ 99.9% backend availability target with email alert |
| OBS-005 email consolidation | ✅ All channels changed to Email only; `<alert-email>` notification channel defined |
| Cost estimate v0.7-draft | ✅ Cloud Monitoring alerting and SLI/SLO at $0.00; changelog updated |
| Version numbers | ✅ spec-01 v2.6, spec-02 v1.9, spec-03 v2.0, spec-04 v1.9, spec-05 v2.8, spec-06 v2.6, spec-07 v2.4, cost v0.7-draft |
