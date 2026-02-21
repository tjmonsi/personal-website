# Gap Analysis v10 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07) after all v9 (4 findings) gaps were applied. This analysis verifies v9 fixes, performs cross-spec reference validation, data flow tracing, IAM completeness audit, algorithm validation, status code completeness check, stale reference search, and count validation.

**Methodology**:
- Verified all 4 v9 fixes were correctly applied across affected specs
- Audited SEC-014 / SEC-013 SA #10 role scope for deployment completeness against INFRA-020 pipeline targets
- Traced 7 end-to-end data flows (5 re-verified from v9 + 2 new: deployment pipeline, POST /t 413 handling)
- Validated all 10 status codes in spec 03 against FE and BE handling
- Searched for stale references introduced by v9 changes
- Validated counts: SEC-012 roles (18), SEC-013 SA #7 (18), SEC-013 SA #10 (5), SEC-014 (5), status codes (10)
- Validated 4 algorithms (progressive banning, vector search, ETag, LRU ban cache)

---

## 1. v9 Fix Verification

All 4 v9 fixes verified as correctly applied:

| v9 Finding | Fix Applied | Verified In |
|------------|-------------|-------------|
| FINDING-001: INFRA-015 `roles/storage.objectAdmin` → `roles/storage.admin` | INFRA-015 Access Control table updated with `roles/storage.admin` (Project scope, per SEC-012) | Spec 05 ~line 734 ✅ |
| FINDING-002: SEC-014 missing `roles/iam.serviceAccountUser` | Added to SEC-014 Granted Roles (scoped to SAs #1–#5), SEC-013 SA #10, and AC-SEC-020 | Spec 06 SEC-014 table, SEC-013 SA #10, AC-SEC-020 ✅ |
| FINDING-003: SEC-012 parenthetical stale `(e.g., roles/datastore.user)` | Updated to `(e.g., roles/datastore.owner)` | Spec 06 ~line 537 ✅ |
| FINDING-004: 413 missing from allowed status code list | Added `413 — Payload Too Large (POST /t body exceeds 100 KB)` to Error Response Strategy | Spec 03 ~line 55 ✅ |

---

## 2. Cross-Spec Reference Validation

### Role Name Consistency

| Role | SEC-012 | SEC-013 SA #7 | INFRA-015 | Other Refs | Consistent |
|------|---------|---------------|-----------|------------|------------|
| `roles/storage.admin` | Project ✅ | `(project)` ✅ | `(Project scope, per SEC-012)` ✅ | — | ✅ |
| `roles/datastore.owner` | Project ✅ | `(CLR-149)` ✅ | — | SEC-012 constraints: `(e.g., roles/datastore.owner)` ✅ | ✅ |
| `roles/iam.serviceAccountUser` (Terraform SA #7) | Project ✅ | ✅ | — | SEC-012 purpose: actAs for Cloud Run/CF resources | ✅ |
| `roles/iam.serviceAccountUser` (App Deployer SA #10) | — | — | — | SEC-014: SAs #1–#5, SEC-013 SA #10: SAs #1–#5, AC-SEC-020: SAs #1–#5 | ✅ |

**Stale reference search**: `roles/storage.objectAdmin` — 0 matches across all specs ✅

### SA Name Cross-References

All 10 SAs in SEC-013 cross-referenced against their defining sections and acceptance criteria. No orphaned or missing references. ✅

### BigQuery Table Names (6 tables)

Consistent across specs 05, 06, 07 (verified in v9, no v9 changes affect these). ✅

### API Endpoints (13 endpoints)

Consistent across specs 02, 03, 06, 07 (verified in v9, no v9 changes affect these). ✅

---

## 3. Data Flow Tracing

### Flows 1–5 (re-verified from v9)

All 5 flows traced in v9 remain consistent after v9 changes:

1. **Page View Tracking**: No v9 changes affect this flow. ✅
2. **Vector Search**: No v9 changes affect this flow. ✅
3. **Progressive Banning**: No v9 changes affect this flow. ✅
4. **ETag Conditional Requests**: No v9 changes affect this flow. ✅
5. **Embedding Sync**: No v9 changes affect this flow. ✅

### Flow 6: Application Deployment Pipeline (new — validates v9 FINDING-002 fix)

```
Developer pushes to `main` → GitHub Actions triggers INFRA-020 pipeline
  → Stage 5 (Deploy):
    → app-deployer@ authenticates via WIF (SEC-014)

    Backend deployment:
      → Push Docker image to Artifact Registry (INFRA-018)
        → roles/artifactregistry.writer ✅
      → gcloud run deploy (INFRA-003, runtime SA #1)
        → roles/run.developer ✅
        → roles/iam.serviceAccountUser on SA #1 (cloud-run-api@) ✅

    Frontend deployment:
      → firebase deploy --only hosting,functions
        → Hosting part: roles/firebasehosting.admin ✅
        → Functions part (INFRA-002 server function):
          → roles/cloudfunctions.developer ✅
          → roles/iam.serviceAccountUser on INFRA-002 runtime SA... ❌
            INFRA-002 uses Firebase default SA (SEC-013 Exception)
            Firebase default SA is NOT in SAs #1–#5
            actAs permission check would FAIL

    Gen 2 Cloud Functions deployment:
      → gcloud functions deploy (4 functions individually)
        → INFRA-008a (SA #2): roles/cloudfunctions.developer ✅ + actAs on SA #2 ✅
        → INFRA-008c (SA #3): roles/cloudfunctions.developer ✅ + actAs on SA #3 ✅
        → INFRA-008d (SA #4): roles/cloudfunctions.developer ✅ + actAs on SA #4 ✅
        → INFRA-014  (SA #5): roles/cloudfunctions.developer ✅ + actAs on SA #5 ✅
```

**Broken path found** at INFRA-002 deployment: See FINDING-001 below.

### Flow 7: POST /t Error Report with 413 (new — validates v9 FINDING-004 fix)

```
Frontend sends error report with oversized payload → POST /t
  → Request body > 100 KB
  → Cloud Armor (passes if under 60 req/min rate limit)
  → Cloud Run middleware:
    1. Request ID + breadcrumb init ✅
    2. CORS (not OPTIONS, continue) ✅
    3. Ban check (LRU cache → Firestore) ✅
    4. HTTP method check (POST on /t → allowed) ✅
    5. Route match (/t → matched) ✅
    6. Body size check: Content-Length > 100 KB → return 413 ✅ (exits here)
  → Response: 413 Payload Too Large

Frontend handling of 413:
  → FE-COMP-005-RETRY: 413 is 4xx → non-retryable ✅ (explicitly listed in spec 02 ~line 521)
  → Non-retryable 4xx, not 503 → FE-COMP-005-MODAL triggers ✅
  → Modal shows error data (without JWT/visitor_session_id) ✅
  → FE-COMP-003 snackbar: NOT triggered (POST /t errors go through FE-COMP-005 path) ✅
```

All steps verified. No broken paths. ✅

**Note**: 413 only originates from `POST /t` (middleware step 6). The FE-COMP-003 error snackbar table correctly omits a specific 413 mapping because `POST /t` tracking events use fire-and-forget `sendBeacon` (no status inspection), and error reports flow through FE-COMP-005-RETRY/MODAL (not FE-COMP-003). The "Other" catch-all in FE-COMP-003 would handle 413 if it ever reached the snackbar, but it won't in practice.

---

## 4. IAM Completeness

### Deployer SA #10 — Full Deployment Permission Audit

| Deploy Target | IAM Role | actAs Required | actAs Target | Covered by SAs #1–#5 | ✅ |
|---------------|----------|----------------|--------------|----------------------|---|
| Artifact Registry (INFRA-018) | `roles/artifactregistry.writer` | No | N/A | N/A | ✅ |
| Cloud Run (INFRA-003) | `roles/run.developer` | Yes | SA #1 `cloud-run-api@` | ✅ | ✅ |
| Firebase Hosting (INFRA-001) | `roles/firebasehosting.admin` | No | N/A | N/A | ✅ |
| Firebase Function `server` (INFRA-002) | `roles/cloudfunctions.developer` | Yes | Firebase default SA | ❌ Not in #1–#5 | **FINDING-001** |
| Gen 2 CF: generate-sitemap (INFRA-008a) | `roles/cloudfunctions.developer` | Yes | SA #2 `cf-sitemap-gen@` | ✅ | ✅ |
| Gen 2 CF: process-rate-limit-logs (INFRA-008c) | `roles/cloudfunctions.developer` | Yes | SA #3 `cf-rate-limit-proc@` | ✅ | ✅ |
| Gen 2 CF: cleanup-rate-limit-offenders (INFRA-008d) | `roles/cloudfunctions.developer` | Yes | SA #4 `cf-offender-cleanup@` | ✅ | ✅ |
| Gen 2 CF: sync-article-embeddings (INFRA-014) | `roles/cloudfunctions.developer` | Yes | SA #5 `cf-embed-sync@` | ✅ | ✅ |

7 of 8 deploy targets have complete permissions. 1 gap found (INFRA-002).

### All 10 Service Accounts

SAs #1–#9 verified in v9 (no v9 changes affect them). SA #10 verified above with one gap found.

---

## 5. Algorithm Validation

All 4 algorithms verified in v9 remain consistent:

1. **Progressive Banning** (SEC-002): Rolling 7-day window, tier escalation 429→403→404, offense counting via `findOneAndUpdate` atomic operations (CLR-197). ✅
2. **Vector Search** (BE-API-002): 8-step flow, UUID v5 cache key, cosine distance threshold 0.35, `findNearest(limit: 500)`, page size 10. ✅
3. **ETag** (BE-API-001/003/005): SHA-256 weak ETag from pre-computed `content_hash`, 304 on match, no full content fetch for conditional requests. ✅
4. **LRU Ban Cache** (SEC-002): 60s TTL, 1000 entries, keyed by client IP, bidirectional propagation delay accepted (CLR-196). ✅

---

## 6. Status Code Completeness

Spec 03 now lists 10 allowed status codes (after v9 FINDING-004 added 413):

| Code | Purpose | Origin | FE Handling | ✅ |
|------|---------|--------|-------------|---|
| 200 | Success | Step 9 (business logic) | Normal render | ✅ |
| 304 | Not Modified | Step 9 (ETag/Last-Modified) | Browser cache revalidation | ✅ |
| 400 | Bad Request | Step 8 (input validation) | FE-COMP-003 snackbar | ✅ |
| 403 | Forbidden | Step 3 (30-day ban) / Step 7 (invalid JWT) | FE-COMP-003 snackbar | ✅ |
| 404 | Not Found | Step 3 (90d/indef ban) / Step 5 (route miss) / Step 9 (resource/masked) | FE-COMP-003 snackbar | ✅ |
| 405 | Method Not Allowed | Step 4 (method enforcement) | FE-COMP-003 snackbar | ✅ |
| 413 | Payload Too Large | Step 6 (POST /t body > 100 KB) | FE-COMP-005-RETRY → modal | ✅ |
| 429 | Too Many Requests | Cloud Armor (load balancer) | FE-COMP-003 snackbar (AC-FE-010) | ✅ |
| 503 | Service Unavailable | BE-API-010 (health check fail) | FE-COMP-003 snackbar / silent retry | ✅ |
| 504 | Gateway Timeout | Cloud Run infrastructure | FE-COMP-003 snackbar (AC-FE-040) | ✅ |

All 10 codes accounted for with consistent handling across specs 02 and 03. ✅

---

## 7. Stale Reference Search (from v9 changes)

| v9 Change | Potential Stale Patterns | Matches Found | ✅ |
|-----------|--------------------------|---------------|---|
| INFRA-015 `roles/storage.admin` | `roles/storage.objectAdmin` | 0 across all specs | ✅ |
| SEC-014 `roles/iam.serviceAccountUser` | Old SEC-014/SEC-013 SA #10 without this role | All references updated | ✅ |
| SEC-012 `(e.g., roles/datastore.owner)` | `roles/datastore.user` in Terraform SA context | 0 (only appears in application SA contexts where it is correct) | ✅ |
| 413 in allowed status code list | Outdated "9 allowed status codes" references | 0 | ✅ |

No stale references from v9 changes found. ✅

---

## 8. Count Validation

| Count | Expected | Actual | ✅ |
|-------|----------|--------|---|
| SEC-012 Terraform SA roles | 18 | 18 (`roles/storage.admin`, `roles/run.admin`, `roles/cloudfunctions.admin`, `roles/compute.networkAdmin`, `roles/compute.securityAdmin`, `roles/cloudscheduler.admin`, `roles/firebasehosting.admin`, `roles/dns.admin`, `roles/artifactregistry.admin`, `roles/logging.admin`, `roles/monitoring.admin`, `roles/iam.serviceAccountAdmin`, `roles/iam.workloadIdentityPoolAdmin`, `roles/iam.serviceAccountUser`, `roles/serviceusage.serviceUsageAdmin`, `roles/pubsub.admin`, `roles/bigquery.admin`, `roles/datastore.owner`) | ✅ |
| SEC-013 SA #7 roles | 18 | 18 (same roles as SEC-012) | ✅ |
| SEC-013 SA #10 roles | 5 | 5 (`roles/artifactregistry.writer`, `roles/run.developer`, `roles/firebasehosting.admin`, `roles/cloudfunctions.developer`, `roles/iam.serviceAccountUser`) | ✅ |
| SEC-014 Granted Roles | 5 | 5 (same roles as SA #10) | ✅ |
| Spec 03 allowed status codes | 10 | 10 (200, 304, 400, 403, 404, 405, 413, 429, 503, 504) | ✅ |

All counts verified. ✅

---

## 9. Remaining Gaps

### FINDING-001 — MEDIUM — Spec: 06

**SEC-014 `roles/iam.serviceAccountUser` Scope Incomplete — Missing Firebase Default SA for INFRA-002 Deployment**

**Gap**: The v9 FINDING-002 fix added `roles/iam.serviceAccountUser` to the app deployer SA (#10), scoped to SAs #1–#5 (the 5 custom runtime service accounts). However, the INFRA-020 deployment pipeline also deploys the Firebase Function `server` (INFRA-002) via `firebase deploy --only hosting,functions`. INFRA-002 uses the Firebase default service account (SEC-013 Exception), which is NOT one of SAs #1–#5. GCP requires `iam.serviceAccounts.actAs` on a function's runtime SA for every deployment (create or update) of Cloud Functions Gen 2, regardless of whether the SA itself is being changed. Since the Firebase default SA (the Compute Engine default SA for Gen 2 functions: `<project-number>-compute@developer.gserviceaccount.com`) is not within the app deployer's actAs scope, the `firebase deploy --only hosting,functions` step in INFRA-020 Stage 5 would fail with a permission denied error.

**Evidence**:
- SEC-014 Granted Roles (spec 06): `roles/iam.serviceAccountUser | SAs #1–#5 (cloud-run-api@, cf-sitemap-gen@, cf-rate-limit-proc@, cf-offender-cleanup@, cf-embed-sync@)`
- SEC-013 Exception (spec 06): "Firebase Functions (INFRA-002) uses the Firebase default service account because it only serves static HTML content and does not access any GCP resources."
- INFRA-002 (spec 05 ~line 69): Node.js 22 LTS function serving the Nuxt 4 SPA shell — no custom SA assigned.
- Spec 01 technology table: Frontend runtime listed as "Firebase Hosting + Cloud Functions Gen 2" — confirming INFRA-002 is Gen 2 (default runtime SA = Compute Engine default SA).
- INFRA-020 Stage 5 (spec 05 ~line 1275): "Frontend: Deploy to Firebase Hosting and Functions via `firebase deploy --only hosting,functions`."
- GCP requirement: "Any principal that deploys a function (create or update) must have the `iam.serviceAccounts.actAs` permission on the function's service account" — applies to Gen 2 Cloud Functions regardless of deployment method (`gcloud functions deploy` or `firebase deploy`).

**Why it matters**: The application CI/CD pipeline (INFRA-020) would fail at the `firebase deploy --only hosting,functions` step. The app deployer can deploy the 4 Gen 2 Cloud Functions (actAs on SAs #2–#5) and Cloud Run (actAs on SA #1), but cannot deploy the Firebase Function `server` because actAs is not granted on the Compute Engine default SA.

**Impact**: Pipeline deploy stage failure. The backend and Gen 2 CFs deploy successfully, but the frontend function deployment fails.

**Affected locations**: SEC-014 Granted Roles table, SEC-013 SA #10 Key IAM Roles, AC-SEC-020.

**Suggested fix**:

1. Extend the `roles/iam.serviceAccountUser` scope in SEC-014 Granted Roles from SAs #1–#5 to include the Compute Engine default SA:

| Role | Scope | Purpose |
|------|-------|---------|
| `roles/iam.serviceAccountUser` | SAs #1–#5 + Compute Engine default SA (`<project-number>-compute@developer.gserviceaccount.com`) | Required to deploy Cloud Run and all Cloud Functions — actAs on custom runtime SAs (#1–#5) and the Firebase default SA (INFRA-002) |

2. Update SEC-013 SA #10 Key IAM Roles to reflect the expanded scope:
   - Change `roles/iam.serviceAccountUser (SAs #1–#5: actAs for Cloud Run and Cloud Functions deployment)` to `roles/iam.serviceAccountUser (SAs #1–#5 + Compute Engine default SA: actAs for Cloud Run and all Cloud Functions deployment including INFRA-002)`

3. Update AC-SEC-020 to mention the Compute Engine default SA in the actAs scope.

**Answer**
For the `roles/iam.serviecAccountUser`, make the scope to all service accounts that will be used by all Cloud Functions and Cloud Runs. As for the deployment using `firebase deploy --only hosting,functions`, use `Firebase Admin` role for it or lower (to make sure that it is just deploying for that particular hosting and function, I think it is Firebase Developer Admin, but I am not sure). 

---

## Summary

| ID | Priority | Spec | Category | Brief Description |
|----|----------|------|----------|-------------------|
| FINDING-001 | MEDIUM | 06 | IAM / CI/CD | SEC-014 actAs scope (SAs #1–#5) does not cover the Firebase/Compute Engine default SA used by INFRA-002, causing `firebase deploy` to fail in INFRA-020 |

**Total: 0 CRITICAL, 1 MEDIUM, 0 LOW.**

The v9 FINDING-002 fix (adding `roles/iam.serviceAccountUser`) was correct in principle but incomplete in scope — it covers the 5 custom runtime SAs (#1–#5) but misses the Firebase default SA used by INFRA-002. This was surfaced by tracing the INFRA-020 deployment flow end-to-end against the actAs scope. All other v9 fixes are correctly applied. No data flow, algorithm, status code, or architectural gaps found beyond this single IAM scope issue.

---

## Cross-Reference Validation Summary

| Validation Area | Items Checked | Result |
|-----------------|---------------|--------|
| v9 fixes | 4 findings | All 4 correctly applied ✅ |
| SEC-012 ↔ SEC-013 SA #7 role count | 18 roles each | Match ✅ |
| SEC-014 ↔ SEC-013 SA #10 role count | 5 roles each | Match ✅ |
| SEC-014 actAs scope vs INFRA-020 deploy targets | 8 deploy targets | 7/8 covered; INFRA-002 Firebase default SA missing ❌ |
| Role name consistency | `roles/storage.admin`, `roles/datastore.owner`, `roles/iam.serviceAccountUser` | No stale references ✅ |
| BigQuery table names | 6 tables across specs 05, 06, 07 | Consistent ✅ |
| API endpoints | 13 endpoints (12 GET + 1 POST) | Consistent ✅ |
| Service account inventory | 10 SAs | All verified ✅ |
| Status codes | 10 codes across specs 02, 03 | All accounted for with consistent FE/BE handling ✅ |
| Data flows | 7 flows (5 re-verified + 2 new) | 1 broken path at INFRA-002 deployment (FINDING-001) |
| Algorithms | 4 algorithms | Consistent ✅ |
| Stale references from v9 | 4 change areas searched | None found ✅ |
| Counts | 5 counts validated | All match ✅ |
