# Gap Analysis v7 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07) after all v6 (3 findings) gaps were applied. This analysis verifies v6 fixes, traces cross-spec references, validates IAM role completeness against every Terraform-managed resource, and examines deployment pipeline consistency.

**Methodology**:
- Verified all 3 v6 fixes were correctly applied across affected specs
- Cross-referenced SEC-012 Terraform SA IAM roles against every resource in INFRA-016 Terraform scope
- Verified INFRA-016 scope list covers all Terraform-managed resources mentioned elsewhere in specs
- Traced Cloud Run SA (#1) permissions against every operation the Go backend performs
- Verified all 10 custom metrics in OBS-004 have a write path to Cloud Monitoring
- Cross-referenced BigQuery table names across all specs
- Validated deployment pipeline (INFRA-020) covers all deployable components
- Verified all acceptance criteria against parent spec sections

**v6 Fix Verification** (all 3 applied correctly):

| v6 Finding | Fix Applied | Verified In |
|------------|------------|-------------|
| FINDING-001: 6 log sinks residuals | Updated AD-016 from "5" to "6", Definitions from "Five" to "Six", added `cloud_functions_error_logs` row to SEC-007 Data Retention Summary | Spec 01 lines 58, 213; Spec 06 line 357 — no remaining "5 sinks" references anywhere |
| FINDING-002: SEC-012 missing 3 IAM roles | Added `roles/compute.securityAdmin`, `roles/cloudscheduler.admin`, `roles/firebasehosting.admin` to SEC-012 table and SEC-013 SA #7. Updated `roles/compute.networkAdmin` purpose. | Spec 06 lines 498–501 (SEC-012 table), line 554 (SEC-013 SA #7) |
| FINDING-003: Gen 2 CF deployment | Expanded INFRA-020 with Cloud Functions lint/test/scan/deploy steps. Broadened app-deployer SA `roles/cloudfunctions.developer` to project-level. Updated SEC-013 SA #10 and AC-SEC-020. | Spec 05 lines 1263–1278 (INFRA-020), Spec 06 lines 557 (SA #10), 589 (SEC-014 role), 636 (AC-SEC-020) |

**Dismissed False Positives** (5 candidates investigated and dismissed):

| # | Candidate | Reason Dismissed |
|---|-----------|-----------------|
| 1 | SEC-014 security constraint says "no admin-level roles" but `roles/firebasehosting.admin` is granted | `roles/firebasehosting.admin` is a deployment role (manages hosting sites/releases), not an infrastructure admin role. The constraint targets roles like `roles/run.admin`, `roles/cloudfunctions.admin`. Firebase Hosting intentionally names its deployment role `.admin`. SEC-014 constraints list explicitly excludes it from the prohibition. |
| 2 | INFRA-020 `firebase deploy --only hosting,functions` deploys Firebase Functions but app deployer SA has `roles/cloudfunctions.developer` project-wide | `firebase deploy --only functions` uses the Cloud Functions API. `roles/cloudfunctions.developer` is the correct role. Project-level scope is intentional (covers `server` function + Gen 2 functions). No excess permissions — `developer` is less than `admin`. |
| 3 | Spec 07 OBS-006 mentions "6 tables" — should it list them? | OBS-006 data flow says "6 tables" with back-reference to INFRA-010a–010f. The detailed table list is in AC-INFRA-006, Summary of Data Stored, and SEC-007 Data Retention Summary. Not a gap — appropriate cross-referencing. |
| 4 | App-deployer SA has `roles/cloudfunctions.developer` project-level but SEC-014 security constraints say "no infrastructure management roles" | `roles/cloudfunctions.developer` is a deployment role (`cloudfunctions.functions.get`, `cloudfunctions.functions.update`, `cloudfunctions.operations.get`). It cannot create, delete, or modify function configurations — only deploy code. It is not an infrastructure management role. |
| 5 | `roles/compute.networkAdmin` purpose no longer mentions "firewall rules" but `roles/compute.securityAdmin` does — is there overlap? | Correct split: `roles/compute.networkAdmin` handles VPC network resources (subnets, routes, LB). `roles/compute.securityAdmin` handles security-related compute resources (firewall rules, security policies, SSL certs). No overlap — Google intentionally separates these roles. |

---

## Remaining Gaps

### FINDING-001 — MEDIUM — Specs: 06, 07

**Cloud Run API SA (#1) Missing `roles/monitoring.metricWriter` for Custom Metrics**

**Gap**: The Go backend exports 10 custom metrics to Cloud Monitoring (OBS-004: `api_requests_total`, `api_request_duration_ms`, `api_bans_active`, `db_query_duration_ms`, `db_connection_pool_active`, `embedding_cache_hits_total`, `embedding_cache_misses_total`, `embedding_api_duration_ms`, `vector_search_duration_ms`, `vector_search_candidates`). Writing custom metrics to the Cloud Monitoring API requires `roles/monitoring.metricWriter` on the service account. The Cloud Run API SA (#1 in SEC-013) does not have this role.

**Evidence**:
- OBS-004 (spec 07 lines 338–355): 10 custom metrics defined for export from the Go application
- OBS-005 (spec 07): Alerting policies depend on these custom metrics
- OBS-006 (spec 07): Operational dashboard depends on these custom metrics
- SEC-013 SA #1 (spec 06 line 548): Lists `roles/datastore.viewer`, `roles/aiplatform.user`, `roles/storage.objectViewer`, `roles/datastore.user` — no `roles/monitoring.metricWriter`
- AC-OBS-010 (spec 07 line 576): Explicitly requires the Go backend to export all 10 custom metrics

**Why it matters**: Without `roles/monitoring.metricWriter`, the Go backend cannot write custom metrics to Cloud Monitoring. All OBS-005 alert policies and OBS-006 dashboard panels that depend on custom metrics would show no data, effectively making the observability layer non-functional beyond built-in Cloud Run platform metrics.

**Suggested fix**:
- (A) Add `roles/monitoring.metricWriter` to SEC-013 SA #1 (`cloud-run-api@`) at project level. Update the SA #1 Key IAM Roles column accordingly.

**Answer**:
Add `roles/monitoring.metricWriter` to SEC-013 SA #1 (`cloud-run-api@`) at project level. Update the SA #1 Key IAM Roles column accordingly.
---

### FINDING-002 — MEDIUM — Specs: 05, 06

**Terraform SA Missing Cloud Storage Role for Media Bucket (INFRA-019) & INFRA-016 Scope Omits INFRA-019**

**Gap**: Two related issues:

1. **INFRA-016 scope list omission**: The INFRA-016 scope list (spec 05 lines 773–793) does not include the Cloud Storage media bucket (INFRA-019), even though INFRA-019 itself states: "Terraform manages the bucket resource (INFRA-016)" (spec 05 line 929).

2. **Terraform SA missing Storage role**: SEC-012 grants `roles/storage.objectAdmin` scoped only to the Terraform state bucket (INFRA-015). No role covers creating or managing the media bucket at project level. Terraform would need `roles/storage.admin` at project level (to create buckets) or at minimum a bucket-level IAM binding after manual creation.

**Evidence**:
- INFRA-019 (spec 05 line 929): "Terraform manages the bucket resource (INFRA-016)"
- INFRA-016 scope (spec 05 lines 773–793): No mention of Cloud Storage media bucket or INFRA-019
- SEC-012 Granted Roles (spec 06 line 493): `roles/storage.objectAdmin` scoped to "Terraform state bucket (INFRA-015)" only
- The media bucket needs to be created, configured (uniform access, versioning, location), and have IAM bindings set

**Why it matters**: `terraform apply` would fail when trying to create the media bucket because the Terraform SA lacks Cloud Storage bucket creation permissions at project level.

**Suggested fix**:
- (A) Add `Cloud Storage media bucket (INFRA-019)` to the INFRA-016 scope list. Change SEC-012 `roles/storage.objectAdmin` scope from "Terraform state bucket (INFRA-015)" to "Project" to cover both the state bucket and media bucket. Update SEC-013 SA #7 accordingly.
- (B) Alternative: Change `roles/storage.objectAdmin` to `roles/storage.admin` at project level (includes bucket creation + object management). More permissive but cleaner.

**Answer**:
Add `Cloud Storage media bucket (INFRA-019)` to the INFRA-016 scope list. Change SEC-012 `roles/storage.objectAdmin` scope from "Terraform state bucket (INFRA-015)" to "Project" to cover both the state bucket and media bucket. Update SEC-013 SA #7 accordingly.
---

### FINDING-003 — MEDIUM — Specs: 05, 06

**Terraform SA Missing `roles/iam.workloadIdentityPoolAdmin` for Content CI/CD WIF Pool (SEC-010)**

**Gap**: INFRA-016 scope explicitly includes: "Workload Identity Federation pools and providers (SEC-010 content CI/CD pool only; Terraform CI/CD pool is a bootstrap resource)" (spec 05 line 791). Managing WIF pools requires `iam.workloadIdentityPools.*` and `iam.workloadIdentityPoolProviders.*` permissions. The Terraform SA's `roles/iam.serviceAccountAdmin` only covers `iam.serviceAccounts.*` — it does NOT include WIF pool/provider management.

**Evidence**:
- INFRA-016 scope (spec 05 line 791): "Workload Identity Federation pools and providers (SEC-010 content CI/CD pool only)"
- SEC-010 (spec 06 lines 413–450): Defines `content-cicd-pool` WIF pool with provider, attribute mapping, and attribute condition
- SEC-012 Granted Roles (spec 06 lines 493–504): Lists `roles/iam.serviceAccountAdmin` but no WIF-specific role
- Google Cloud IAM docs: `roles/iam.serviceAccountAdmin` grants `iam.serviceAccounts.*` permissions only; WIF pools need `roles/iam.workloadIdentityPoolAdmin`

**Why it matters**: `terraform apply` would fail when trying to create or manage the `content-cicd-pool` WIF pool and its `github-actions` provider because the Terraform SA lacks the necessary IAM permissions.

**Suggested fix**:
- (A) Add `roles/iam.workloadIdentityPoolAdmin` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Manage Workload Identity Federation pools and providers (SEC-010)".

**Answer**:
Add `roles/iam.workloadIdentityPoolAdmin` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Manage Workload Identity Federation pools and providers (SEC-010)".
---

### FINDING-004 — LOW — Specs: 03, 04

**DM-005 `content_hash` Field Has No Consumer (No `/others` Detail Endpoint)**

**Gap**: DM-005 (`others` collection, spec 04 line 148) defines a `content_hash` field described as: "Used by the backend for ETag generation and conditional request handling." However, spec 03 (line 864) explicitly states: "No detail endpoint for `/others`: The `others` collection does not have a `GET /others/{slug}.md` detail endpoint. Items in `/others` link directly to external URLs."

ETag/conditional request handling (304 Not Modified) is only used on article detail endpoints (`GET /technical/{slug}.md`, `GET /blog/{slug}.md`). Since `/others` has no detail endpoint, `content_hash` is never used by the backend for this collection.

**Evidence**:
- DM-005 (spec 04 line 148): `content_hash` described for "ETag generation and conditional request handling"
- Spec 03 line 864: "No detail endpoint for `/others`"
- BE-API-002 conditional request logic (spec 03 lines 448–456): References DM-002/DM-003 `content_hash`, not DM-005
- The list endpoint `GET /others` uses `Cache-Control` for client-side caching, not ETags per document

**Why it matters**: Minor data model inconsistency — the field is computed by the content CI/CD pipeline but never consumed by the backend. No runtime impact, but misleading documentation.

**Suggested fix**:
- (A) Update DM-005 `content_hash` description to clarify it is computed by the content CI/CD pipeline for change detection purposes (detecting which articles need re-embedding by `sync-article-embeddings`), not for backend ETag handling.
- (B) Alternative: Remove `content_hash` from DM-005 if the embedding sync function uses `date_updated` for change detection instead.

**Answer**:
Update DM-005 `content_hash` description to clarify it is computed by the content CI/CD pipeline for change detection purposes (detecting which articles need re-embedding by `sync-article-embeddings`), not for backend ETag handling.
---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| FINDING-001 | MEDIUM | IAM / Observability | Cloud Run API SA (#1) missing `roles/monitoring.metricWriter` for 10 custom metrics in OBS-004 — alerts and dashboards would have no data |
| FINDING-002 | MEDIUM | IAM / Terraform Scope | Terraform SA has `roles/storage.objectAdmin` scoped to state bucket only; INFRA-016 scope omits media bucket (INFRA-019); `terraform apply` would fail creating media bucket |
| FINDING-003 | MEDIUM | IAM / Terraform Scope | Terraform SA missing `roles/iam.workloadIdentityPoolAdmin` for content CI/CD WIF pool (SEC-010); `terraform apply` would fail managing WIF resources |
| FINDING-004 | LOW | Data Model | DM-005 `content_hash` described for ETag/conditional requests but no `/others` detail endpoint exists — field has no backend consumer |

**Total: 3 MEDIUM, 1 LOW — no CRITICAL gaps remain.**

---

## Cross-Reference Validation Summary

The following cross-spec validations were performed with no issues found:

- **v6 fixes**: All 3 findings verified as correctly applied — no residual "5 sinks" references, all 3 new Terraform SA roles present, INFRA-020 includes Gen 2 CF deployment, app-deployer SA broadened to project-level
- **BigQuery table names**: 6 tables consistent across INFRA-010a–010f, AC-INFRA-006, OBS-006, SEC-007 Data Retention Summary, Summary of Data Stored (spec 05)
- **SEC-013 SA inventory (#1–#10)**: All 10 SAs cross-referenced against resource access patterns; only SA #1 missing `roles/monitoring.metricWriter` (FINDING-001)
- **INFRA-016 scope vs SEC-012 roles**: All resources in scope have corresponding roles EXCEPT media bucket (FINDING-002) and WIF pool admin (FINDING-003)
- **INFRA-020 deployment pipeline**: Now covers Backend (Cloud Run), Frontend (Firebase Hosting+Functions), and Gen 2 Cloud Functions — all code deployment paths accounted for
- **Modal trigger logic**: FE-COMP-005-RETRY → FE-COMP-005-MODAL → AC-FE-022 consistent
- **Firebase deploy scope**: `firebase deploy --only hosting,functions` covers INFRA-001 + INFRA-002; Gen 2 functions deployed separately via `gcloud functions deploy`
- **Cache-Control headers**: All 13 endpoints consistent between response format table and individual endpoint specs
- **Vector search pipeline**: 8-step flow verified end-to-end (query → embedding cache → Vertex AI → Firestore Native → filter in FE → response)
- **Progressive banning pipeline**: Offense recording → ban evaluation → enforcement → response codes — consistent across specs 03, 05, 06
- **Category caching flow**: FE-COMP-007 per-type keys → BE-API-008 `type` parameter → DM-006 `types` array — verified
- **Tracking data flow**: Frontend `POST /t` → JWT → GeoIP → IP truncation → structured log → Cloud Logging → BigQuery — consistent
- **Acceptance criteria**: All AC-* criteria verified against parent spec sections — no orphaned or contradictory ACs
- **Endpoint count**: 13 `GET` endpoints + 1 `POST` endpoint consistent across specs 02, 03
- **Field names**: DM-001 through DM-011 field names match spec 03 backend references and spec 02 frontend consumption
- **Service account count**: "10 service accounts" consistent between SEC-013 inventory and all references
- **Log sink count**: "6 log sinks" now fully consistent across all 7 spec files
