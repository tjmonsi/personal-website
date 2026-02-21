# Gap Analysis v6 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07) after all v5 (4 findings) gaps were applied. This analysis verifies v5 fixes, traces cross-spec references, validates IAM role completeness, and examines deployment pipeline consistency.

**Methodology**:
- Verified all 4 v5 fixes were correctly applied across affected specs
- Traced all BigQuery log sink references across all 7 spec files (not just specs 05 and 07)
- Cross-referenced SEC-012 Terraform Builder SA IAM roles against every resource in INFRA-016 Terraform scope
- Verified INFRA-020 CI/CD pipeline covers all code deployment responsibilities defined in INFRA-016 Deployment Boundary
- Cross-referenced SEC-013 SA inventory (#1–#10) against all pipeline and service definitions
- Traced frontend error reporting retry/modal logic end-to-end
- Validated component IDs, field names, cache headers, and acceptance criteria across all specs

**v5 Fix Verification** (all 4 applied correctly):

| v5 Finding | Fix Applied | Verified In |
|------------|------------|-------------|
| FINDING-001: 6 log sinks/tables | Updated counts from "5" to "6" in specs 05 (diagram, INFRA-011, AC-INFRA-006) and spec 07 (OBS-006). Added `cloud_functions_error_logs` row to Summary of Data Stored. Updated spec 01 deployment config line. | Spec 01 line 233, Spec 05 lines 1049, 1070, 1198, 1289, Spec 07 line 448 |
| FINDING-002: CI/CD build/deploy commands | Overview table: `nuxt generate` → `nuxt build`. INFRA-020: `--only hosting` → `--only hosting,functions`. | Spec 05 lines 289, 1272 |
| FINDING-003: Modal trigger redefined | Modal condition now: "All 5 retry attempts exhausted OR a non-retryable error received, AND backend error response (not 503)." Updated FE-COMP-005-MODAL, Error Display Criteria, AC-FE-022. | Spec 02 lines 536, 553, 937 |
| FINDING-004: App deployer SA | SA #10 `app-deployer@` added to SEC-013 with deployment roles. SEC-014 created with WIF config (shared `terraform-cicd-pool`). INFRA-020 auth reference changed to "see SEC-014". AC-SEC-020 added. WIF-authenticated SAs note updated to "#6, #7, #10". | Spec 05 line 1274, Spec 06 lines 553, 559, 572–617, 628 |

**Dismissed False Positives** (6 candidates investigated and dismissed):

| # | Candidate | Reason Dismissed |
|---|-----------|-----------------|
| 1 | SEC-014 shared WIF pool with Terraform SA is a security risk | Not a risk — the WIF pool trusts the repository, per-workflow `google-github-actions/auth` specifies which SA to impersonate. `app-deployer@` and `terraform-builder@` have distinct, non-overlapping roles. Each GHA workflow declares exactly one SA. |
| 2 | INFRA-016 scope doesn't mention SEC-014 WIF SA bindings | The IAM binding for `app-deployer@` on the existing pool is a standard IAM binding, covered by INFRA-016's general "IAM bindings and service accounts (except bootstrap resources)" scope. No explicit mention needed. |
| 3 | `roles/compute.networkAdmin` purpose column doesn't mention INFRA-004 (Load Balancer) | The role does include LB resources (`compute.backendServices.*`, `compute.urlMaps.*`, `compute.targetHttpProxies.*`). Purpose column is intentionally concise. Not a functional gap — documentation nit only. |
| 4 | App deployer SA project-level scope vs service-level | `roles/run.developer` at project level is acceptable for a single-service personal website. Service-level scoping adds Terraform complexity for minimal benefit. |
| 5 | Bootstrap Resources table doesn't list app deployer WIF dependency | The `app-deployer@` SA uses the already-bootstrapped `terraform-cicd-pool`. SA itself is Terraform-managed. No additional bootstrap resources needed. |
| 6 | `roles/datastore.user` at project level for Terraform SA might not cover named `vector-search` database | Project-level roles apply to all Firestore databases in the project, including named databases. No gap. |

---

## Remaining Gaps

### FINDING-001 — LOW — Specs: 01, 06

**Residual "5 Sinks" References in Specs 01 and 06**

**Gap**: The v5 fix updated specs 05 and 07, but 3 additional locations in specs 01 and 06 still reference "5" log sinks instead of "6":

1. **Spec 01, AD-016** (line 213): "via **5** dedicated log sinks" — should be 6
2. **Spec 01, Definitions table** (line 58): "**Five** sinks route logs to dedicated tables" — should be Six
3. **Spec 06, SEC-007 Data Retention Summary** (lines 350–361): Table lists 5 BigQuery rows — missing `cloud_functions_error_logs`

**Counterpoint** (spec 01 IS partially correct):
- Spec 01 Deployment Topology (line 233) correctly says "6 tables" (updated in v5)
- Spec 01 architecture diagram correctly says "(6 Log Sinks → INFRA-010)" (updated in v5)

This creates an internal inconsistency within spec 01 itself.

**Why it matters**: SEC-007 Data Retention Summary is the primary privacy/compliance reference. Omitting `cloud_functions_error_logs` means its retention period and data classification are undocumented in the security spec.

**Suggested fix**:
- (A) Update all 3 locations: AD-016 → "6 dedicated log sinks", Definitions → "Six sinks route logs to dedicated tables", SEC-007 Data Retention Summary → add row: `BigQuery cloud_functions_error_logs | Cloud Functions error logs | 2 years | BigQuery table expiry`.

**Answer**
Update all 3 locations: AD-016 → "6 dedicated log sinks", Definitions → "Six sinks route logs to dedicated tables", SEC-007 Data Retention Summary → add row: `BigQuery cloud_functions_error_logs | Cloud Functions error logs | 2 years | BigQuery table expiry`.

---

### FINDING-002 — MEDIUM — Spec: 06

**SEC-012 Terraform SA Missing 3 IAM Roles for Resources in INFRA-016 Scope**

**Gap**: The Terraform SA (`terraform-builder@`) in SEC-012 is missing 3 IAM roles required to manage resources explicitly listed in the INFRA-016 Terraform scope:

1. **`roles/compute.securityAdmin`** — Required for managing Cloud Armor security policies (INFRA-005). The existing `roles/compute.networkAdmin` covers VPC, subnets, load balancer resources, and read-only firewall access, but does NOT include `compute.securityPolicies.*` permissions needed for Cloud Armor WAF rules. Also needed for VPC firewall rule creation (`compute.firewalls.create`) and Google-managed SSL certificates for the load balancer.

2. **`roles/cloudscheduler.admin`** — Required for managing Cloud Scheduler jobs (INFRA-008b, INFRA-008e, INFRA-014b). No existing role in the SEC-012 table includes `cloudscheduler.jobs.*` permissions.

3. **`roles/firebasehosting.admin`** — Required for creating the `google_firebase_hosting_site` resource. The INFRA-016 note states: "Terraform enables the Firebase API and creates the Firebase Hosting site resource (`google_firebase_hosting_site`)."

**Evidence**:
- SEC-012 currently has 12 roles (lines 493–504)
- INFRA-016 scope (lines 776–788) explicitly includes Cloud Armor, Cloud Scheduler, and the Firebase Hosting site resource
- `roles/compute.networkAdmin` permissions include `compute.networks.*`, `compute.subnetworks.*`, `compute.backendServices.*`, `compute.urlMaps.*` — but NOT `compute.securityPolicies.*`

**Why it matters**: Without these roles, `terraform apply` would fail when managing Cloud Armor policies, Cloud Scheduler jobs, or the Firebase Hosting site resource. The SEC-012 note claims roles are "the minimum set required" — this is incorrect while roles are missing.

**Suggested fix**:
- (A) Add 3 roles to SEC-012 Granted Roles table and SEC-013 SA #7 role list:
  - `roles/compute.securityAdmin | Project | Manage Cloud Armor security policies (INFRA-005), VPC firewall rules (INFRA-009), Google-managed SSL certificates (INFRA-004)`
  - `roles/cloudscheduler.admin | Project | Manage Cloud Scheduler jobs (INFRA-008b, 008e, 014b)`
  - `roles/firebasehosting.admin | Project | Create Firebase Hosting site resource (INFRA-001)`
  Update `roles/compute.networkAdmin` purpose to: "Manage VPC, subnets, load balancer resources (INFRA-004, INFRA-009)"

**Answer**
Do suggested Fix

---

### FINDING-003 — LOW — Spec: 05

**4 Gen 2 Cloud Functions Have No Defined CI/CD Deployment Mechanism**

**Gap**: INFRA-016 Deployment Boundary states CI/CD deploys Cloud Function code ("Deploys new function code via `gcloud functions deploy`"). However, INFRA-020 (the only CI/CD pipeline defined) only covers:
- **Backend**: Docker image → Artifact Registry → Cloud Run
- **Frontend**: `nuxt build` → `firebase deploy --only hosting,functions`

The 4 Gen 2 Cloud Functions (INFRA-008a, 008c, 008d, INFRA-014) are NOT Firebase Functions — they are Google Cloud Functions Gen 2 (Node.js 22 LTS). The `firebase deploy` command does not deploy them. No pipeline step, SA role, or deployment command is specified for these functions.

**Evidence**:
- INFRA-016 Deployment Boundary (line 811): `Cloud Functions | ... | Deploys new function code via gcloud functions deploy`
- INFRA-016 lifecycle note: "Terraform uses `lifecycle { ignore_changes }` on the source field for Cloud Functions"
- INFRA-020 Deploy step (lines 1270–1272): Only "Backend: Cloud Run" and "Frontend: Firebase Hosting+Functions"
- App deployer SA (#10) `roles/cloudfunctions.developer` is scoped to `server` function (INFRA-002) only

**Why it matters**: After Terraform creates the functions with initial code, subsequent code changes have no defined deployment path. The `lifecycle { ignore_changes }` prevents Terraform from updating code; the CI/CD pipeline doesn't deploy Gen 2 functions; the app deployer SA lacks permissions for them.

**Practical severity is LOW**: These are infrastructure support functions with simple, stable logic that change infrequently.

**Suggested fix**:
- (A) Expand INFRA-020: Add a deploy step for Gen 2 functions via `gcloud functions deploy`. Broaden app-deployer SA `roles/cloudfunctions.developer` scope to project-level (covering all 5 functions: `server` + 4 Gen 2). Add lint/test steps for Cloud Functions Node.js code.
- (B) Document manual deployment: Add a note to INFRA-016 Deployment Boundary that Gen 2 Cloud Functions are deployed manually by the owner using `gcloud` CLI, with justification (infrequent changes, simple logic).
- (C) Use Terraform for code deployment: Remove `lifecycle { ignore_changes }` on the source field. Let Terraform manage both infrastructure and code for these 4 functions.

**Answer**
(A) Expand INFRA-020: Add a deploy step for Gen 2 functions via `gcloud functions deploy`. Broaden app-deployer SA `roles/cloudfunctions.developer` scope to project-level (covering all 5 functions: `server` + 4 Gen 2). Add lint/test steps for Cloud Functions Node.js code.

---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| FINDING-001 | LOW | Cross-Spec Consistency | "5 log sinks/tables" residual in spec 01 (AD-016, Definitions) and spec 06 (SEC-007 Data Retention Summary) — 3 locations missed by v5 fix |
| FINDING-002 | MEDIUM | IAM / Security | SEC-012 Terraform SA missing `roles/compute.securityAdmin`, `roles/firebasehosting.admin`, `roles/cloudscheduler.admin` for resources in INFRA-016 scope |
| FINDING-003 | LOW | Deployment Pipeline | 4 Gen 2 Cloud Functions have no defined code deployment mechanism in INFRA-020; app deployer SA lacks roles for Gen 2 functions |

**Total: 1 MEDIUM, 2 LOW — no CRITICAL gaps remain.**

---

## Cross-Reference Validation Summary

The following cross-spec validations were performed with no issues found:

- **v5 fixes**: All 4 findings verified as correctly applied in their targeted specs
- **SEC-013 SA inventory (#1–#10)**: All 10 SAs cross-referenced against resource access patterns, WIF configs, and deployment pipelines
- **SEC-014 → INFRA-020**: Authentication reference updated correctly; deployment commands match SA permissions
- **Modal trigger logic**: FE-COMP-005-RETRY "any 4xx → no retry → direct to failure handling" now routes to modal via "OR a non-retryable error received" clause; AC-FE-022, AC-RETRY-004 consistent
- **Firebase deploy scope**: `firebase deploy --only hosting,functions` deploys Firebase Hosting (INFRA-001) + `server` Firebase Function (INFRA-002); consistent with rewrite rules
- **`nuxt build` consistency**: Overview table matches INFRA-020 detail and AD-001 SPA mode
- **BigQuery tables**: Summary of Data Stored (spec 05) now has 6 rows; AC-INFRA-006 lists 6 tables; diagram shows 6 bullets
- **Log sink filters**: INFRA-010d and INFRA-010f are complementary (`service_name="website-api"` vs `service_name!="website-api"`); verified no overlap
- **Cache-Control headers**: All 13 endpoints consistent between response format table and individual endpoint specs
- **Vector search pipeline**: 8-step flow verified end-to-end — unchanged from v5
- **Progressive banning pipeline**: Offense recording → ban evaluation → enforcement → response codes — consistent across specs 03, 05, 06
- **Category caching flow**: FE-COMP-007 per-type keys → BE-API-008 `type` parameter → DM-006 `types` array — verified end-to-end
- **Tracking data flow**: Frontend `POST /t` → JWT → GeoIP → IP truncation → structured log → BigQuery → Looker Studio — consistent
- **Acceptance criteria**: All AC-* criteria verified against parent spec sections — no orphaned or contradictory ACs
