# Gap Analysis v5 — 2026-02-22

**Context**: Deep re-analysis of the complete spec suite (specs 01–07, all ~5,500 lines) after all v4 (5 findings) gaps were applied. This analysis verifies v4 fixes, traces end-to-end data flows, validates cross-spec identifiers, and examines edge cases and deployment accuracy.

**Methodology**:
- Verified all 5 v4 fixes were correctly applied across affected specs
- Traced all 7 BigQuery log sinks and their cross-references across specs 05 and 07
- Verified CI/CD pipeline stages (overview table vs INFRA-020 detail) for consistency
- Validated FE-COMP-005-RETRY error handling flow for all HTTP status code classes
- Cross-referenced SEC-013 SA inventory against all pipeline/service definitions
- Traced frontend build → deploy → serve chain through INFRA-001, INFRA-002, INFRA-020
- Checked all FINDING IDs, field names, IAM roles, deployment commands, and count references

**v4 Fix Verification** (all 5 applied correctly):

| v4 Finding | Fix Applied | Verified In |
|------------|------------|-------------|
| FINDING-001: FE-COMP-007 per-type caching | Per-type cache keys (`categories_technical`, etc.), `GET /categories?type=<page_type>`, independent 24hr TTL | Spec 02 FE-COMP-007, AC-FE-042, AC-FE-050 |
| FINDING-002: Middleware step 3 → 429 removed | Step 3 now says "Return `403` or `404`" with Cloud Armor clarification note | Spec 03 middleware execution order |
| FINDING-003: 4xx non-retryable expansion | Changed to "any HTTP `4xx` client error response (including `400`, `403`, `404`, `405`, `413`, and `429`)" | Spec 02 FE-COMP-005-RETRY, AC-RETRY-004 |
| FINDING-004: SA #5 IAM role | Added `roles/datastore.viewer` for Firestore Enterprise | Spec 06 SEC-013 SA #5 |
| FINDING-005: Embedding cache size monitoring | INFRA-014 periodic count + OBS-005 log-based metric alert + OBS-006 dashboard panel | Specs 05, 07 |

**Dismissed False Positives** (7 candidates investigated and dismissed):

| # | Candidate | Reason Dismissed |
|---|-----------|-----------------|
| 1 | `tag_match` UI toggle not present | Intentional simplicity — `tag_match` defaults to `any`, is available via deep linking (URL parameter), and FE-COMP-011 correctly omits `tag_match=any` from URL. No UI toggle needed; the backend supports it for power users constructing URLs. |
| 2 | Looker Studio doesn't query `cloud_functions_error_logs` | Appropriate scope separation — Looker Studio is for user-facing analytics (visitors, pages, errors). Cloud Function operational errors are monitored via Cloud Monitoring alerts (OBS-005 has 4 dedicated CF alerts + a catch-all). |
| 3 | SEC-011 Cloud Run Firestore Native role inconsistency | Consistent — SEC-011 says `roles/datastore.viewer` on `vector-search` DB; SEC-013 SA #1 confirms `roles/datastore.viewer` for Firestore Native. No mismatch. |
| 4 | `others` items incorrectly included in sitemap | Already addressed — INFRA-008a step 4 explicitly says "THE SYSTEM SHALL exclude articles in the `others` category." Static page `/others` IS included at priority 0.4. |
| 5 | `Sitemap:` directive missing from robots.txt | Present — OBS-007 includes `Sitemap: https://tjmonsi.com/sitemap.xml` at the end of the Bingbot section. |
| 6 | SVG script execution via image proxy | Dismissed in v4 — backend CSP restricts script execution; `<img>` tag loading doesn't execute SVG scripts per browser security model. |
| 7 | `www.tjmonsi.com` DNS record missing | Not a spec gap — ultra-low priority operational concern. INFRA-017 defines `tjmonsi.com` and `api.tjmonsi.com` only. |

---

## Remaining Gaps

### FINDING-001 — MEDIUM — Specs: 05, 07

**INFRA-010f `cloud_functions_error_logs` Missing from 6 Cross-References**

**Gap**: INFRA-010 defines 6 BigQuery log sinks (10a through 10f), but 6 locations across specs 05 and 07 still reference "5 sinks" or "5 tables," omitting the 6th sink/table (`cloud_functions_error_logs` / INFRA-010f). This 6th sink was likely added in a later clarification round, and several downstream references were not updated.

**Evidence**:
- Spec 05, INFRA-010a–010f: **6 sinks clearly defined** with individual subsections
- Spec 05, AC-INFRA-021: References INFRA-010f correctly ("`cloud_functions_error_logs` table (INFRA-010f)")
- Spec 05, INFRA-016 scope: Correctly says "BigQuery sinks (INFRA-010a–010f)"

But the following 6 locations say "5":
1. **Spec 05, Network diagram** — BigQuery section lists 5 bullet points (missing `cloud_functions_error_logs`) and the diagram label says "(5 Log Sinks)"
2. **Spec 05, INFRA-011** — "Tables connected: All **5** tables in `website_logs` dataset"
3. **Spec 05, INFRA-010 "Summary of Data Stored in BigQuery"** — Table has 5 rows, missing the `cloud_functions_error_logs` row entirely (no retention or privacy classification for CF error data)
4. **Spec 05, AC-INFRA-006** — Lists 5 tables: "`all_logs`, `cloud_armor_lb_logs`, `frontend_tracking_logs`, `frontend_error_logs`, `backend_error_logs`" — missing `cloud_functions_error_logs`
5. **Spec 07, OBS-006 Data Flow** — "BigQuery log sinks (**5** tables)"

**Why it matters**:
1. **Terraform implementation**: An implementer using the summary table or AC-INFRA-006 as reference would create only 5 sinks, missing the Cloud Functions error routing entirely
2. **Looker Studio configuration**: INFRA-011 saying "All 5 tables" means the 6th table might not be connected as a data source
3. **Privacy compliance**: The Summary of Data Stored table is the primary reference for data classification and retention. Omitting `cloud_functions_error_logs` means its data contents and retention period are undocumented in the privacy summary
4. **Acceptance testing**: AC-INFRA-006 would pass with only 5 sinks, missing validation of the 6th

**Suggested fix**:
- (A) Update all 6 locations to reference 6 sinks/tables. Add a row to the Summary of Data Stored table: `cloud_functions_error_logs | No personal data (Cloud Functions don't process user requests directly) | N/A | 2 years`. Update counts from "5" to "6" everywhere. Add `cloud_functions_error_logs` to AC-INFRA-006's table list and the network diagram.
- (B) If Cloud Functions error logs don't warrant a separate BigQuery table (they're already in `all_logs`), remove INFRA-010f and update AC-INFRA-021 to remove the reference. (Not recommended — INFRA-010f enables focused querying of CF errors without scanning the full `all_logs` table.)

**Answer**
Update all 6 locations to reference 6 sinks/tables. Add a row to the Summary of Data Stored table: `cloud_functions_error_logs | No personal data (Cloud Functions don't process user requests directly) | N/A | 2 years`. Update counts from "5" to "6" everywhere. Add `cloud_functions_error_logs` to AC-INFRA-006's table list and the network diagram.

---

### FINDING-002 — LOW — Spec: 05

**CI/CD Pipeline Frontend Build and Deploy Commands Inconsistent**

**Gap**: Two CI/CD-related inconsistencies in the frontend build/deploy commands:

1. **Build command**: The CI/CD Pipeline overview table (around line 300) lists `nuxt generate` as the frontend build command, while the detailed INFRA-020 section specifies `nuxt build`. For Nuxt 4 in SPA mode, `nuxt build` (with SPA config) is correct. `nuxt generate` performs static pre-rendering (SSG), which is a different rendering strategy.

2. **Deploy command**: INFRA-020 says "Deploy to Firebase Hosting via `firebase deploy --only hosting`." However, INFRA-001 and INFRA-002 establish that the frontend uses both Firebase Hosting (static assets) AND a Firebase Function (`server`) to serve the SPA shell. The `--only hosting` flag deploys only static assets, not the SPA-serving function. Every Nuxt build regenerates the function code alongside static assets, so both must be deployed together.

**Evidence**:
- CI/CD Pipeline overview table: `| Build | nuxt generate | Docker build |`
- INFRA-020: "Build the Nuxt 4 SPA (`nuxt build`)"
- Spec 01 AD-001: Confirms SPA mode (not SSG)
- INFRA-001 rewrite config: `{"source": "**", "function": "server"}` — non-asset routes require the Firebase Function
- INFRA-002: Firebase Functions "Serve the Nuxt 4 SPA shell"
- INFRA-020 deploy: "`firebase deploy --only hosting`" — omits functions

**Why it matters**:
- An implementer referencing the overview table would use `nuxt generate` instead of `nuxt build`, producing SSG output instead of SPA output
- Using `firebase deploy --only hosting` on first deployment would leave the `server` function undeployed, causing all non-asset routes, including the entire SPA, to fail
- On subsequent deployments, the function would serve stale SPA content (previous build's `index.html`)

**Suggested fix**:
- (A) Update the overview table build command from `nuxt generate` to `nuxt build`. Update INFRA-020 deploy command from `firebase deploy --only hosting` to `firebase deploy --only hosting,functions` (or `firebase deploy` to deploy everything).
- (B) If the Nuxt 4 Firebase preset convention is just `firebase deploy`, use that instead of specifying `--only` flags.

**Answer**
Update the overview table build command from `nuxt generate` to `nuxt build`. Update INFRA-020 deploy command from `firebase deploy --only hosting` to `firebase deploy --only hosting,functions` (or `firebase deploy` to deploy everything)
---

### FINDING-003 — LOW — Spec: 02

**FE-COMP-005-RETRY: Non-Retryable 4xx Error Path Does Not Define Modal/Silent Behavior**

**Gap**: The v4 fix correctly expanded the non-retryable list to "any HTTP 4xx." AC-RETRY-004 says the system "proceeds directly to failure handling." However, the Error Display Criteria defines the modal trigger as: "All 5 retry attempts exhausted AND failure was a backend error response (any HTTP status other than 503)." A 4xx error that bypasses retries never exhausts 5 attempts, so the "all 5 retry attempts exhausted" predicate is never true, and the 4xx error falls through the decision tree without matching any defined behavior.

**Evidence**:
- FE-COMP-005-RETRY: "THE SYSTEM SHALL NOT retry errors that are clearly non-retryable: any HTTP `4xx` client error response"
- AC-RETRY-004: "Given a non-retryable error (any HTTP `4xx` response), when received, then the system does NOT retry and **proceeds directly to failure handling**."
- Error Display Criteria (modal): "All **5 retry attempts exhausted** AND failure was a backend error response (any HTTP status other than `503`)"
- A 4xx on first attempt → no retries → "5 retry attempts exhausted" is never satisfied → behavior undefined

**Why it matters**: If `POST /t` returns 400 (malformed error report payload — frontend bug) or 403 (expired JWT — clock skew), the error report cannot be delivered. The user should ideally see the fallback modal (allowing manual error sharing), but the spec's logic as written leaves this path undefined. Practical impact is LOW because these scenarios represent frontend bugs rather than typical runtime failures.

**Suggested fix**:
- (A) Redefine the modal trigger to: "All retry attempts exhausted OR a non-retryable error received, AND the failure was a backend error response (any HTTP status other than `503`)." This routes 4xx errors to the modal, matching the likely intended behavior.
- (B) Add explicit text to FE-COMP-005-RETRY: "WHEN a non-retryable `4xx` error is received, THE SYSTEM SHALL proceed directly to the modal display logic (same as if all retries are exhausted with a non-503 response)."
- (C) Define 4xx as silent failure (acceptable for a personal website — these are edge cases caused by frontend bugs)

**Answer**
Redefine the modal trigger to: "All retry attempts exhausted OR a non-retryable error received, AND the failure was a backend error response (any HTTP status other than `503`)." This routes 4xx errors to the modal, matching the likely intended behavior.

---

### FINDING-004 — MEDIUM — Specs: 05, 06

**Application CI/CD Pipeline Service Account Missing from SEC-013 Inventory**

**Gap**: INFRA-020 defines a full Application CI/CD Pipeline (GitHub Actions) that authenticates to GCP via WIF, pushes Docker images to Artifact Registry, deploys to Cloud Run, and deploys to Firebase Hosting. However, SEC-013 (Consolidated Service Account Inventory) does not include a service account for this pipeline. The 9 SAs in SEC-013 cover Cloud Run, Cloud Functions, content CI/CD, Terraform CI/CD, Looker Studio, and Cloud Scheduler — but not the application deployment pipeline. INFRA-020 says "see SEC-012" for WIF authentication, but SEC-012 only defines the Terraform Builder SA.

**Evidence**:
- INFRA-020: "The CI/CD pipeline SHALL authenticate to GCP using **Workload Identity Federation** (WIF). No long-lived service account keys are permitted (see SEC-012)."
- SEC-012: Defines only `terraform-builder@` SA — purpose: "Manage GCP infrastructure resources"
- SEC-013: 9 SAs listed — none for application deployment
  - SA #6 (`content-cicd@`): Content repo only — `roles/cloudfunctions.invoker` + `roles/run.invoker`
  - SA #7 (`terraform-builder@`): Infrastructure management — admin-level roles
  - No SA with deployment roles: `roles/artifactregistry.writer`, `roles/run.developer`, Firebase Hosting deploy permissions
- SEC-012 WIF pool (`terraform-cicd-pool`) trusts `personal-website` repo — the app CI/CD runs from the same repo, so the WIF pool can be shared, but the SA cannot (Terraform SA has admin-level roles that violate least privilege for deployments)

**Why it matters**:
1. **Least-privilege violation**: Without a dedicated SA, the implementer might reuse the Terraform SA (`terraform-builder@`) for deployments, granting the CI/CD deploy step unnecessary admin-level access to DNS, IAM, BigQuery, Pub/Sub, etc.
2. **Incomplete inventory**: SEC-013 is described as "Provide a single reference of all service accounts used across the system" — omitting the app CI/CD SA breaks this guarantee
3. **IAM role gap**: The specific IAM roles needed for application deployment (Artifact Registry push, Cloud Run revision deploy, Firebase Hosting deploy) are not documented anywhere
4. **WIF configuration gap**: If the app CI/CD uses the same WIF pool as Terraform (`terraform-cicd-pool`), the attribute mapping needs to allow the app CI/CD workflow to impersonate a *different* SA (not `terraform-builder@`). This is technically achievable but undocumented.

**Suggested fix**:
- (A) Add a new SA row (#10) to SEC-013: `app-deployer@<project-id>.iam.gserviceaccount.com` with roles: `roles/artifactregistry.writer` (Artifact Registry), `roles/run.developer` (Cloud Run), appropriate Firebase roles for hosting+functions deployment. Create a corresponding SEC section (e.g., SEC-014) detailing this SA. The SA can use the existing `terraform-cicd-pool` WIF pool (same repo) or a dedicated pool. Update INFRA-020 to reference the new SA instead of "see SEC-012."
- (B) Explicitly document that the Terraform SA is intentionally shared for both infrastructure and application deployment, with a justification for the broader-than-needed roles. (Not recommended — violates least privilege.)
- (C) Use the Terraform WIF pool with a separate SA binding — document the multi-SA WIF mapping in SEC-012.

**Answer**
Add a new SA row (#10) to SEC-013: `app-deployer@<project-id>.iam.gserviceaccount.com` with roles: `roles/artifactregistry.writer` (Artifact Registry), `roles/run.developer` (Cloud Run), appropriate Firebase roles for hosting+functions deployment. Create a corresponding SEC section (e.g., SEC-014) detailing this SA. The SA can use the existing `terraform-cicd-pool` WIF pool (same repo) or a dedicated pool. Update INFRA-020 to reference the new SA instead of "see SEC-012."
---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| FINDING-001 | MEDIUM | Cross-Spec Consistency | INFRA-010f `cloud_functions_error_logs` missing from 6 downstream references (diagram, summary table, INFRA-011, AC, OBS-006) |
| FINDING-002 | LOW | Spec Inconsistency | Frontend build command (`nuxt generate` vs `nuxt build`) and deploy command (`--only hosting` missing `,functions`) in CI/CD pipeline |
| FINDING-003 | LOW | Edge Case | FE-COMP-005-RETRY 4xx non-retryable path doesn't reach modal decision logic due to "5 retries exhausted" predicate |
| FINDING-004 | MEDIUM | Documentation / Security | Application CI/CD pipeline has no SA in SEC-013; INFRA-020 "see SEC-012" points to Terraform SA, not a deployment-specific SA |

**Total: 2 MEDIUM, 2 LOW — no CRITICAL gaps remain.**

---

## Cross-Reference Validation Summary

The following cross-spec validations were performed with no issues found:

- **v4 fixes**: All 5 v4 findings verified as correctly applied across relevant specs (FE-COMP-007 per-type caching, middleware step 3 no 429, non-retryable 4xx expansion, SA #5 `roles/datastore.viewer`, embedding cache size monitoring wired through INFRA-014 → OBS-005 → OBS-006)
- **Component IDs**: All FE-PAGE, FE-COMP, BE-API, DM, INFRA, SEC, OBS identifiers verified — no dangling or broken references
- **Cache-Control headers**: All 13 endpoints consistent between response format table and individual endpoint specs
- **Log sink filters**: INFRA-010d and INFRA-010f filters are correctly complementary (`service_name="website-api"` vs `service_name!="website-api"`) — no overlap
- **IAM roles**: 9 SAs in SEC-013 cross-referenced against resource access; SEC-011 Firestore Native `roles/datastore.viewer` for Cloud Run matches SEC-013 SA #1; SA #5 `roles/datastore.viewer` for Firestore Enterprise matches the read-only requirement of `sync-article-embeddings`
- **Data field names**: All Firestore field names (DM-001–012) verified against API response schemas and frontend display specs
- **Acceptance criteria**: All AC-* criteria verified against parent specification sections — AC-INFRA-021 correctly references INFRA-010f despite the count inconsistency elsewhere
- **Vector search flow**: 8-step pipeline verified end-to-end — `findNearest(limit: 500)`, cosine distance ≤ 0.35, 5s Vertex AI timeout, L2 normalization, post-filter exhaustion returning empty 200
- **Sitemap generation**: INFRA-008a correctly excludes `others` items, includes static pages at appropriate priorities, and URL patterns match frontend routes (`.md` extension)
- **Tracking data flow**: Frontend `POST /t` → JWT validation → GeoIP lookup → IP truncation → `visitor_id` hash → structured log → Cloud Logging → BigQuery sinks → Looker Studio — all field names and transformations consistent
- **Progressive banning**: Offense recording (INFRA-008c atomic `findOneAndUpdate`) → ban application → ban enforcement (Go backend LRU cache → Firestore fallback) → response codes (403/404) — consistent pipeline
- **robots.txt**: Frontend domain (`tjmonsi.com`) includes `Sitemap:` directive and disallows `/t`; API domain (`api.tjmonsi.com`) disallows all via BE-API-012 — consistent with OBS-007
- **Offline reading**: Cache API (primary retrieval) + IndexedDB (metadata only) + service worker lifecycle — consistent between FE-COMP-008, FE-PAGE-003/005, and backend conditional request support (ETag/304)
- **Category caching**: FE-COMP-007 per-type keys → `GET /categories?type=<page_type>` → BE-API-008 `type` parameter → DM-006 `types` array filtering — end-to-end chain verified
- **Firebase Hosting rewrites**: `/sitemap.xml` → Cloud Run `website-api` via `run` directive; `**` → Firebase Function `server` — consistent with INFRA-001, INFRA-002, and BE-API-011
