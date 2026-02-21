# Gap Analysis v8 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07) after all v7 (4 findings) gaps were applied. This analysis verifies v7 fixes, systematically audits every Terraform SA IAM role against _actual GCP permission requirements_, and validates all cross-spec references.

**Methodology**:
- Verified all 4 v7 fixes were correctly applied across affected specs
- Audited every IAM role in SEC-012 against the _actual GCP permissions_ needed for each Terraform resource in INFRA-016
- Verified every SA (#1–#10) role grants exactly the permissions needed — no more, no less
- Traced all data flows end-to-end
- Validated DM-005 `content_hash` against INFRA-014 sync logic and DM-012 `embedding_text_hash`
- Counted all role entries in SEC-012 and SEC-013 SA #7 for consistency
- Verified AC-OBS-010 metric list against OBS-004 defined metrics

**v7 Fix Verification** (all 4 applied correctly):

| v7 Finding | Fix Applied | Verified In |
|------------|------------|-------------|
| FINDING-001: SA #1 metricWriter | Added `roles/monitoring.metricWriter` to SA #1 | Spec 06 line 548 — SA #1 now lists `roles/monitoring.metricWriter` (Cloud Monitoring — export 10 custom metrics, OBS-004) |
| FINDING-002: Storage scope + INFRA-016 | Changed SEC-012 `roles/storage.objectAdmin` scope from state bucket to Project. Added `Cloud Storage media bucket (INFRA-019)` to INFRA-016 scope. Updated SEC-013 SA #7. | Spec 06 line 495 (scope: Project), Spec 05 line 787 (INFRA-019 in scope), Spec 06 line 555 (SA #7: `(project)`) |
| FINDING-003: WIF pool admin | Added `roles/iam.workloadIdentityPoolAdmin` to SEC-012 table and SEC-013 SA #7 | Spec 06 line 507 (SEC-012 table), Spec 06 line 555 (SA #7 role list) |
| FINDING-004: DM-005 content_hash | Updated description to reference sync function change detection | Spec 04 line 148 |

**Dismissed False Positives** (4 candidates investigated and dismissed):

| # | Candidate | Reason Dismissed |
|---|-----------|-----------------|
| 1 | App deployer SA (#10) needs `roles/iam.serviceAccountUser` for deploying Cloud Run/Cloud Functions | `gcloud run deploy` and `gcloud functions deploy` update code on existing services that already have SAs assigned (by Terraform). The deployer doesn't change or set the SA — it only pushes new code/images. `iam.serviceAccounts.actAs` is not required for code-only deployments to existing services. |
| 2 | SEC-012 note about "minimum set" needs updating after role additions | The note says "Additional roles may be needed if Terraform scope expands; any additions SHALL be documented with justification." Each addition (v6, v7) was documented with justification. The note remains accurate — it's a forward-looking constraint, not a static claim. |
| 3 | SEC-012 `roles/monitoring.admin` purpose doesn't mention OBS-010 SLI/SLO | `roles/monitoring.admin` includes `monitoring.services.*` needed for SLI/SLO. The purpose column is intentionally concise — "Manage alerting policies, dashboards (OBS-005, OBS-006)" is representative, not exhaustive. SLI/SLO management falls under monitoring administration. Not a functional gap. |
| 4 | `roles/storage.objectAdmin` at project level is too permissive — allows reading/writing objects in ALL buckets | For this single-project personal website with only 2 buckets (state + media), project-level scope is pragmatic. Bucket-level IAM bindings would require Terraform to manage IAM on the state bucket (a bootstrap resource), creating a chicken-and-egg issue. The Terraform SA already has broad project-level admin roles. |

---

## Remaining Gaps

### FINDING-001 — CRITICAL — Spec: 06

**SEC-012 `roles/storage.objectAdmin` Cannot Create GCS Buckets — Needs `roles/storage.admin`**

**Gap**: `roles/storage.objectAdmin` grants `storage.objects.*` permissions (read/write/delete objects) but does NOT include `storage.buckets.create`, `storage.buckets.update`, `storage.buckets.delete`, or `storage.buckets.setIamPolicy`. Creating the media bucket (INFRA-019) via `google_google_storage_bucket` Terraform resource requires `storage.buckets.create`. Setting uniform bucket-level access, versioning, and IAM bindings requires `storage.buckets.update` and `storage.buckets.setIamPolicy`.

**Evidence**:
- SEC-012 (spec 06 line 495): `roles/storage.objectAdmin | Project | Read/write Terraform state and lock files (INFRA-015) and manage Cloud Storage media bucket (INFRA-019)`
- INFRA-019 (spec 05 line 929): "Terraform manages the bucket resource (INFRA-016)"
- INFRA-019 config (spec 05 lines 905–914): Bucket creation requires setting location, storage class, access control (uniform), versioning
- GCP IAM docs: `roles/storage.objectAdmin` = `storage.objects.*` only. `roles/storage.admin` = `storage.buckets.*` + `storage.objects.*`

**Why it matters**: `terraform apply` would fail with a permissions error when creating the media bucket. The Terraform state bucket is a bootstrap resource (manually created), so `roles/storage.objectAdmin` was sufficient before INFRA-019 was added to Terraform scope. Now that Terraform must create the media bucket, the role is insufficient.

**Suggested fix**:
- (A) Change `roles/storage.objectAdmin` to `roles/storage.admin` in SEC-012 and SEC-013 SA #7. Update purpose to: "Create and manage Cloud Storage buckets (INFRA-019) and read/write Terraform state (INFRA-015)".

**Answer**:
Change `roles/storage.objectAdmin` to `roles/storage.admin` in SEC-012 and SEC-013 SA #7. Update purpose to: "Create and manage Cloud Storage buckets (INFRA-019) and read/write Terraform state (INFRA-015)"
---

### FINDING-002 — MEDIUM — Spec: 06

**SEC-012 Terraform SA Missing `roles/serviceusage.serviceUsageAdmin` for GCP API Enablement**

**Gap**: INFRA-016 scope (spec 05 line 773) lists as its FIRST item: "GCP API enablement (via `google_project_service` resources)." The `google_project_service` Terraform resource requires `serviceusage.services.enable` and `serviceusage.services.disable` permissions. These are in `roles/serviceusage.serviceUsageAdmin` or `roles/serviceusage.serviceUsageConsumer` — none of the 17 roles currently in SEC-012 include these permissions.

**Evidence**:
- INFRA-016 scope (spec 05 line 773): Lists ~15 GCP APIs to enable (run, cloudfunctions, firestore, compute, aiplatform, dns, artifactregistry, pubsub, firebase, sts, iamcredentials, eventarc, cloudscheduler, bigquery, cloudbuild)
- SEC-012 Granted Roles (spec 06 lines 493–508): 17 roles listed — none include `serviceusage.services.*`
- GCP Terraform docs: `google_project_service` requires `serviceusage.services.enable` permission

**Why it matters**: `terraform apply` would fail at the very first step — enabling GCP APIs — if the Terraform SA doesn't have this permission. This is typically the first resource Terraform manages, and all other resources depend on the APIs being enabled.

**Suggested fix**:
- (A) Add `roles/serviceusage.serviceUsageAdmin` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Enable and disable GCP APIs (INFRA-016 API enablement)".

**Answer**:
Add `roles/serviceusage.serviceUsageAdmin` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Enable and disable GCP APIs (INFRA-016 API enablement)".
---

### FINDING-003 — MEDIUM — Spec: 06

**SEC-012 Terraform SA Missing `roles/iam.serviceAccountUser` (actAs Permission)**

**Gap**: When Terraform creates a Cloud Run service or Cloud Function with a custom service account (e.g., `service_account = "cloud-run-api@..."` or `service_account_email = "cf-sitemap-gen@..."`), the Terraform SA needs `iam.serviceAccounts.actAs` permission on those target SAs. This permission is in `roles/iam.serviceAccountUser`. The existing `roles/iam.serviceAccountAdmin` grants `iam.serviceAccounts.create/delete/update/get/list` but NOT `iam.serviceAccounts.actAs`.

**Evidence**:
- SEC-013 (spec 06 lines 548–557): 8 service accounts (#1–#5, #9 are Terraform-managed) are assigned to Cloud Run and Cloud Functions resources
- SEC-012 (spec 06 line 506): `roles/iam.serviceAccountAdmin` — manages SA lifecycle but NOT actAs
- INFRA-003 (Cloud Run with SA #1), INFRA-008a (CF with SA #2), INFRA-008c (CF with SA #3), INFRA-008d (CF with SA #4), INFRA-014 (CF with SA #5) — all require Terraform to assign a specific SA
- GCP IAM docs: `iam.serviceAccounts.actAs` is in `roles/iam.serviceAccountUser`, NOT in `roles/iam.serviceAccountAdmin`

**Why it matters**: `terraform apply` would fail when creating Cloud Run services or Cloud Functions that specify a custom service account, because the Terraform SA cannot "act as" those SAs without the `actAs` permission.

**Suggested fix**:
- (A) Add `roles/iam.serviceAccountUser` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Assign service accounts to Cloud Run and Cloud Functions resources (actAs permission)".

**Answer**:
Add `roles/iam.serviceAccountUser` to SEC-012 Granted Roles table and SEC-013 SA #7. Purpose: "Assign service accounts to Cloud Run and Cloud Functions resources (actAs permission)".
---

### FINDING-004 — MEDIUM — Spec: 06

**SEC-012 `roles/datastore.user` Cannot Create Firestore Databases — Needs `roles/datastore.owner`**

**Gap**: INFRA-016 scope includes "Firestore Native database (INFRA-012)" and the `google_firestore_database` Terraform resource requires `datastore.databases.create` permission. `roles/datastore.user` grants `datastore.entities.*` and `datastore.databases.get` but does NOT include `datastore.databases.create`. The `roles/datastore.owner` role includes all `datastore.*` permissions including database creation.

**Evidence**:
- INFRA-012 (spec 05 line 542): Firestore Native Mode database with ID `vector-search` — must be created by Terraform
- INFRA-016 scope (spec 05 line 784): "Firestore Native database (INFRA-012)"
- SEC-012 (spec 06 line 508): `roles/datastore.user | Project | Manage Firestore Enterprise and Native databases (INFRA-006, INFRA-012)`
- GCP IAM docs: `roles/datastore.user` = `datastore.entities.*`, `datastore.indexes.*`, `datastore.databases.get` — NOT `datastore.databases.create`

**Why it matters**: `terraform apply` would fail when creating the `vector-search` Firestore Native database because the Terraform SA lacks `datastore.databases.create`. The purpose column says "Manage" databases but the role only provides entity/index-level access, not database lifecycle management.

**Suggested fix**:
- (A) Change `roles/datastore.user` to `roles/datastore.owner` in SEC-012 and SEC-013 SA #7. Update purpose to: "Create and manage Firestore Enterprise and Native databases, entities, and indexes (INFRA-006, INFRA-012)".

**Answer**:
Change `roles/datastore.user` to `roles/datastore.owner` in SEC-012 and SEC-013 SA #7. Update purpose to: "Create and manage Firestore Enterprise and Native databases, entities, and indexes (INFRA-006, INFRA-012)".
---

### FINDING-005 — LOW — Spec: 04

**DM-005 `content_hash` Description Incorrectly Claims Sync Function Usage**

**Gap**: The v7 fix updated DM-005 `content_hash` to say: "Used by the `sync-article-embeddings` Cloud Function (INFRA-014) for change detection — determines which documents need re-embedding." However, the sync function (INFRA-014 steps 2–4, spec 05 lines 642–648) computes SHA-256 of `title + "\n" + abstract + "\n" + category + "\n" + tags (comma-separated)` and compares against `embedding_text_hash` from DM-012 vector documents — NOT against `content_hash` from DM-005.

`content_hash` is SHA-256 of the full document content; `embedding_text_hash` is SHA-256 of the embedding source text (title + abstract + category + tags). These are different hashes of different source texts.

**Evidence**:
- DM-005 (spec 04 line 148): `content_hash` now says "Used by the `sync-article-embeddings` Cloud Function (INFRA-014) for change detection"
- INFRA-014 sync logic step 2 (spec 05 line 642): "construct the embedding source text: `title + "\n" + abstract + "\n" + category + "\n" + tags (comma-separated)`"
- INFRA-014 sync logic step 3 (spec 05 line 643): "Compute SHA-256 hash of the source text"
- INFRA-014 sync logic step 4 (spec 05 line 644): "Compare the hash with the `embedding_text_hash` field in the corresponding Firestore Native document"
- DM-012a (spec 04 line 325): `embedding_text_hash | string | SHA-256 hash of the source text used for embedding (for change detection during sync)`

**Why it matters**: The description is misleading — `content_hash` in DM-005 is NOT consumed by the sync function or the backend for this collection. It may be retained for content CI/CD pipeline internal use (e.g., detecting full-content changes), but has no in-scope consumer.

**Suggested fix**:
- (A) Correct DM-005 `content_hash` description: "Pre-computed SHA-256 hash of the document content. Computed by the content CI/CD pipeline on each update. Not consumed within this system — the `sync-article-embeddings` function uses `embedding_text_hash` (DM-012) for change detection, and no `/others` detail endpoint exists for ETag handling. Retained for content CI/CD pipeline internal use."

**Answer**:
(A) Correct DM-005 `content_hash` description: "Pre-computed SHA-256 hash of the document content. Computed by the content CI/CD pipeline on each update. Not consumed within this system — the `sync-article-embeddings` function uses `embedding_text_hash` (DM-012) for change detection, and no `/others` detail endpoint exists for ETag handling. Retained for content CI/CD pipeline internal use.
---

### FINDING-006 — LOW — Spec: 07

**AC-OBS-010 Lists 8 of 10 Custom Metrics — Missing `api_bans_active` and `db_connection_pool_active`**

**Gap**: OBS-004 defines 10 custom metrics, but AC-OBS-010 (spec 07 line 576) only lists 8: `api_requests_total`, `api_request_duration_ms`, `db_query_duration_ms`, `embedding_cache_hits_total`, `embedding_cache_misses_total`, `embedding_api_duration_ms`, `vector_search_duration_ms`, and `vector_search_candidates`. Missing: `api_bans_active` (Gauge) and `db_connection_pool_active` (Gauge).

**Evidence**:
- OBS-004 (spec 07 lines 340–355): 10 custom metrics including `api_bans_active` and `db_connection_pool_active`
- AC-OBS-010 (spec 07 line 576): Lists only 8 metrics

**Why it matters**: Incomplete acceptance criteria could cause the two Gauge metrics to be omitted from implementation and testing, leaving the ban monitoring and connection pool health dashboards without data.

**Suggested fix**:
- (A) Add `api_bans_active` and `db_connection_pool_active` to AC-OBS-010.

**Answer**:
Add `api_bans_active` and `db_connection_pool_active` to AC-OBS-010.
---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| FINDING-001 | CRITICAL | IAM / Terraform | `roles/storage.objectAdmin` cannot create GCS buckets — needs `roles/storage.admin` for INFRA-019 media bucket |
| FINDING-002 | MEDIUM | IAM / Terraform | Missing `roles/serviceusage.serviceUsageAdmin` — Terraform can't enable GCP APIs via `google_project_service` |
| FINDING-003 | MEDIUM | IAM / Terraform | Missing `roles/iam.serviceAccountUser` — Terraform can't assign custom SAs to Cloud Run/Cloud Functions (actAs) |
| FINDING-004 | MEDIUM | IAM / Terraform | `roles/datastore.user` can't create Firestore databases — needs `roles/datastore.owner` for INFRA-012 |
| FINDING-005 | LOW | Data Model | DM-005 `content_hash` incorrectly claims sync function usage; sync uses `embedding_text_hash` from DM-012 |
| FINDING-006 | LOW | Acceptance Criteria | AC-OBS-010 lists 8 of 10 custom metrics, missing `api_bans_active` and `db_connection_pool_active` |

**Total: 1 CRITICAL, 3 MEDIUM, 2 LOW — no CRITICAL data flow or user flow gaps remain. All findings are IAM/Terraform permission gaps or documentation accuracy issues.**

---

## Cross-Reference Validation Summary

The following cross-spec validations were performed with no issues found:

- **v7 fixes**: All 4 findings verified as correctly applied
- **BigQuery table names**: 6 tables consistent across all specs (INFRA-010a–010f, AC-INFRA-006, OBS-006, SEC-007 Data Retention Summary)
- **SEC-013 SA inventory**: All 10 SAs verified; SA #1 now has `roles/monitoring.metricWriter`
- **INFRA-016 scope**: Now includes INFRA-019 (media bucket); all items have corresponding SEC-012 roles EXCEPT GCP API enablement (FINDING-002)
- **INFRA-020 pipeline**: Covers Backend (Cloud Run), Frontend (Firebase Hosting+Functions), Gen 2 Cloud Functions — all deployment paths defined
- **Data flows**: All 12 user action flows traced end-to-end with no broken paths
- **Modal/retry logic**: FE-COMP-005-RETRY → FE-COMP-005-MODAL → AC-FE-022 consistent
- **Cache headers**: All 13 endpoints consistent
- **Vector search pipeline**: 8-step flow verified end-to-end
- **Progressive banning pipeline**: Consistent across specs 03, 05, 06
- **Field names**: DM-001 through DM-012 verified against backend and frontend specs
- **Service account count**: 10 SAs consistent
- **Log sink count**: 6 sinks consistent
- **Cloud Function count**: 5 functions (1 Firebase + 4 Gen 2) consistent
- **Endpoint count**: 14 endpoints (13 GET + 1 POST) consistent
- **SEC-012 role count**: Currently 17 roles; would become 20 with fixes (storage.admin replaces storage.objectAdmin, +serviceUsageAdmin, +serviceAccountUser, datastore.owner replaces datastore.user)
