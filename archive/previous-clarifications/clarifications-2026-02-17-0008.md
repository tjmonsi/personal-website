# Clarifications — 2026-02-17 — Batch 0008

## Architecture Review: Terraform/IaC Integration & Cross-Specification Consistency

The following items were identified during a comprehensive architecture review of all 7 spec documents (v1.9–v2.6) and the cost estimate (v0.5-draft), with a focus on the recently added Terraform/IaC documentation (INFRA-015, INFRA-016, SEC-012, AD-023) and cross-specification consistency.

Previously resolved clarifications (CLR-001 through CLR-058) were reviewed and excluded.

---

### CLR-059: Firebase Hosting & Functions Not in Terraform Scope — Deployment Boundary Unclear 🟡

**Context**: INFRA-016 (spec 05) lists the resources Terraform manages: Cloud Run, Cloud Functions (Gen 2), VPC, Cloud Armor, Load Balancer, Cloud Scheduler, BigQuery, Firestore Native, IAM, and Cloud DNS. However, **Firebase Hosting (INFRA-001) and Firebase Functions (INFRA-002) are not listed** in the Terraform scope.

The Application CI/CD pipeline (spec 05) shows `firebase deploy` as the frontend deployment method. Firebase resources can be partially managed by Terraform (e.g., `google_firebase_hosting_site`, `google_firebase_hosting_channel`), but actual content deployment (uploading built SPA assets) requires the Firebase CLI.

This creates an unclear boundary: does Terraform provision the Firebase Hosting site and custom domain configuration, while the CI/CD pipeline only deploys the built assets? Or is Firebase entirely outside Terraform?

**Affected specs**: Spec 01 (Technology Stack), Spec 05 (INFRA-001, INFRA-002, INFRA-016), Cost estimate.

**Question**: Should Terraform manage the Firebase Hosting site and Functions configuration (resource provisioning), with the CI/CD pipeline handling only asset deployment via `firebase deploy`?

**Options**:

- **A**: **Terraform provisions, CI/CD deploys** — Terraform creates the Firebase Hosting site, configures the custom domain (`tjmonsi.com`), and manages the `firebase.json` rewrite rules and headers (SEC-005). The CI/CD pipeline uses `firebase deploy` to upload built assets only. Add INFRA-001 and INFRA-002 to the Terraform scope list in INFRA-016.
- **B**: **Firebase CLI handles everything** — Firebase Hosting and Functions are managed entirely via `firebase deploy` and the Firebase Console. Keep them out of Terraform scope. Document this explicitly in INFRA-016 as an exclusion.
- **C**: **Hybrid approach** — Terraform enables the Firebase API and creates the hosting site; custom domain, rewrite rules, headers, and deployment are all handled by `firebase deploy` with `firebase.json`. Document the boundary.

**Recommendation**: Option C — Firebase Hosting configuration (rewrites, headers, domain) is tightly coupled to `firebase.json` and is more naturally managed via `firebase deploy`. Terraform should only ensure the Firebase project-level resources exist.

**Answer**
Use C 

---

### CLR-060: Firestore Enterprise (INFRA-006) Missing from Terraform Scope 🟡

**Context**: INFRA-016 lists "Firestore Native database (INFRA-012)" in Terraform's scope but does **not** list Firestore Enterprise (INFRA-006) — the primary application database. Firestore Enterprise in MongoDB compatibility mode requires specific provisioning (enterprise tier, MongoDB compat mode selection). The `google_firestore_database` Terraform resource can create Firestore databases, but MongoDB compatibility mode is a newer feature that may require specific API calls or console setup.

This is a significant omission — the primary database is not accounted for in the IaC scope.

**Affected specs**: Spec 05 (INFRA-006, INFRA-016).

**Question**: Should the Firestore Enterprise database (MongoDB compat mode) be provisioned by Terraform or created manually as a bootstrap resource?

**Options**:

- **A**: **Terraform-managed** — Add INFRA-006 to the Terraform scope. Use `google_firestore_database` resource (if MongoDB compat mode is supported by the Terraform provider) or the Google Cloud provider's generic resource. Terraform manages the database creation, mode selection, and region.
- **B**: **Bootstrap resource (manual)** — Add Firestore Enterprise to the bootstrap resources table in INFRA-016 alongside the GCP project, state bucket, and Terraform SA. Document the manual provisioning steps (console or `gcloud`).
- **C**: **Investigate first** — The Terraform Google provider's support for Firestore Enterprise MongoDB compatibility mode needs verification. Defer the decision until the provider's capabilities are confirmed during implementation.

**Recommendation**: Option C, with a fallback to B. Firestore Enterprise with MongoDB compat mode is relatively new, and Terraform provider support should be verified before committing to managing it via IaC.

**Answer**
Use C, with a fallback to B if Terraform support is lacking.

---

### CLR-061: Cloud DNS Has No INFRA Section and INFRA-016 Cross-Reference Is Wrong 🟡

**Context**: INFRA-016 lists "Cloud DNS (INFRA-006)" in the Terraform scope. However, **INFRA-006 is the Database (Firestore Enterprise)**, not Cloud DNS. Cloud DNS appears in the architecture diagrams (spec 01), the deployment topology, and the cost estimate (as a line item with zone + query costs), but it has **no dedicated INFRA section** in spec 05.

This means:
1. Cloud DNS configuration (zone, A/AAAA records for `tjmonsi.com` and `api.tjmonsi.com`, NS records) is not formally specified.
2. The cross-reference in INFRA-016 incorrectly points to the database instead of a Cloud DNS section.

**Affected specs**: Spec 01 (High-Level Architecture), Spec 05 (INFRA-016 scope list), Cost estimate.

**Question**: Should a formal `INFRA-017: Cloud DNS` section be added to spec 05, and should the cross-reference in INFRA-016 be corrected?

**Options**:

- **A**: **Add INFRA-017: Cloud DNS** — Define the Cloud DNS zone, record types (A/AAAA for `tjmonsi.com` pointing to the Load Balancer, CNAME or A for `api.tjmonsi.com`), TTL values, and DNSSEC configuration. Update INFRA-016 to reference the new section. *(Recommended)*
- **B**: **Fix the cross-reference only** — Change "Cloud DNS (INFRA-006)" to "Cloud DNS" (no INFRA reference) in INFRA-016. Leave Cloud DNS configuration as an implementation detail.
- **C**: **Merge into INFRA-004** — Add DNS configuration as a subsection of the Load Balancer (INFRA-004), since DNS records point to the LB.

**Recommendation**: Option A — Cloud DNS is a distinct, Terraform-managed resource with its own configuration surface. It deserves a short dedicated section.

**Answer**
Use A.

---

### CLR-062: Terraform CI/CD Pipeline Authentication Method Unspecified 🟡

**Context**: Spec 05 defines a Terraform CI/CD pipeline (Format → Validate → Plan → Apply) and states it "authenticates using the Terraform service account (SEC-012)." SEC-012 specifies a service account key (JSON) that "SHALL NOT be committed to any repository" and notes: *"For CI/CD, Workload Identity Federation SHOULD be evaluated as a keyless alternative (future improvement)."*

The content CI/CD pipeline explicitly uses Workload Identity Federation (CLR-056, SEC-010, AD-022) with clear configuration (pool, provider, attribute mapping). However, the Terraform CI/CD pipeline has **no equivalent specification** for how it authenticates. The options are:
1. Service account key stored as a GitHub Actions secret (simpler but less secure)
2. Workload Identity Federation (consistent with the content CI/CD approach, more secure)

Without this decision, the Terraform CI/CD pipeline cannot be implemented.

**Affected specs**: Spec 05 (CI/CD Pipeline - Terraform), Spec 06 (SEC-012).

**Question**: How should the Terraform CI/CD pipeline authenticate to GCP?

**Options**:

- **A**: **Workload Identity Federation** — Configure a WIF pool/provider for the infrastructure repository's GitHub Actions (similar to SEC-010 for content CI/CD). The Terraform SA acts as the impersonated service account. No JSON keys needed in CI/CD. Update SEC-012 to formally adopt WIF instead of marking it as a "future improvement."
- **B**: **Service Account Key as GitHub Secret** — Store the Terraform SA key as a GitHub Actions secret. Simpler setup but requires key rotation and carries the risk of key exposure. Document this as the initial approach with WIF as a future improvement.
- **C**: **Defer to implementation** — Keep the spec intentionally vague. Document both options and decide during implementation.

**Recommendation**: Option A — this project already uses WIF for the content CI/CD (CLR-056), so applying the same pattern to Terraform CI/CD is consistent and avoids managing long-lived keys.

**Answer**
Use A.

---

### CLR-063: Development Environment — Local-Only or GCP-Deployed? Terraform Implications 🟡

**Context**: Spec 01 (Environments table) defines Development as *"Local development, testing, and pre-production validation"* with *"Firestore Emulator / Local MongoDB"*. The note says: *"No separate staging GCP project is provisioned."*

However, multiple specs reference a GCP Development environment:
- INFRA-009: *"The Development environment does NOT use a VPC to reduce cost; services connect to Google Cloud APIs directly."* — implies GCP services exist in Dev.
- INFRA-003: VPC egress is marked *"Production only"* — implies Cloud Run exists in Dev without VPC.
- CI/CD branch strategy: `dev` → *"Development environment"* — implies deploying somewhere.
- Cost estimate: Only models Production costs.

This creates ambiguity: if Dev is purely local (emulators), then the VPC exclusion statements are irrelevant and Terraform doesn't need environment handling. If Dev is a GCP deployment (same or different project, reduced resources), then Terraform needs separate state or configuration.

**Affected specs**: Spec 01 (Environments), Spec 05 (INFRA-003, INFRA-009, CI/CD branch strategy).

**Question**: Does the `dev` branch deploy to GCP resources, and if so, does Terraform manage them?

**Options**:

- **A**: **Dev is local-only** — Development uses only local emulators and local MongoDB. The `dev` branch is for code integration/testing before merging to `main`. No GCP resources are deployed for Dev. Remove or clarify the VPC exclusion language to say *"if a Development environment were provisioned on GCP, it would not use a VPC."* Terraform manages only Production.
- **B**: **Dev deploys to the same GCP project** — The `dev` branch deploys to the same GCP project with a separate Cloud Run service (e.g., `api-dev`), separate Firestore database, and no VPC. Terraform uses workspaces or variable files to manage both environments. Add Dev costs to the cost estimate.
- **C**: **Dev deploys to a separate GCP project** — A lightweight Dev project with no VPC, no Cloud Armor, no LB. Terraform uses separate state files per project. Add Dev costs to the cost estimate.
- **D**: **Defer** — Keep Dev as local-only for now, with the option to add a GCP Dev environment later. Clarify the spec language.

**Recommendation**: Option A or D — for a personal website with a single owner who can deploy immediately, a local Dev environment is sufficient. The VPC exclusion language should be clarified to avoid confusion.

**Answer**
Use A but add a Firestore Enterprise local development database and Firestore Native local development. This allows for more realistic local testing without incurring GCP costs. The `dev` branch is for local development and testing only, with no GCP deployment. The VPC exclusion language will be clarified to reflect this.

---

### CLR-064: Pub/Sub Topic for Rate Limit Log Sink Not Documented as a Resource 🟡

**Context**: INFRA-008c specifies that the `process-rate-limit-logs` Cloud Function is triggered by a *"Cloud Logging log sink (via Pub/Sub)."* The log sink section describes the filter but does not specify:
1. The **Pub/Sub topic name** that the log sink routes to.
2. The **Pub/Sub subscription** that triggers the Cloud Function.
3. Whether this Pub/Sub topic is listed in the Terraform scope.

Pub/Sub is not mentioned anywhere in INFRA-016's Terraform scope list. These are infrastructure resources that need to be created before the log sink and Cloud Function can work.

**Affected specs**: Spec 05 (INFRA-008c, INFRA-016).

**Question**: Should the Pub/Sub topic and subscription for the rate-limit log sink be formally documented and added to the Terraform scope?

**Options**:

- **A**: **Add a brief Pub/Sub subsection** to INFRA-008c — Define the topic name (e.g., `rate-limit-events`), subscription (auto-created by Cloud Functions Gen 2 trigger), and add Pub/Sub to the INFRA-016 Terraform scope list. *(Recommended)*
- **B**: **Leave implicit** — The Pub/Sub topic is an implementation detail of the Cloud Functions Gen 2 event trigger. Cloud Functions Gen 2 can auto-create Pub/Sub subscriptions. Document only the log sink filter and let Terraform handle the plumbing.
- **C**: **Switch to direct Cloud Function trigger** — Instead of log sink → Pub/Sub → Cloud Function, use a Cloud Functions Eventarc trigger on Cloud Logging directly (supported in Gen 2). This eliminates the explicit Pub/Sub topic.

**Recommendation**: Option A — explicitly documenting the Pub/Sub topic makes the infrastructure reproducible and avoids surprises during Terraform implementation.

**Answer**
Use A.

---

### CLR-065: Artifact Registry for Docker Images Not Specified 🟡

**Context**: The Application CI/CD pipeline (spec 05) includes a "Docker build" step for the Go backend and a "Cloud Run deploy" step. Cloud Run deploys container images from a registry. However, **no container image registry** is specified anywhere in the specs:
- No INFRA section for Artifact Registry or Container Registry (deprecated).
- No Terraform scope entry for the registry.
- No cost estimate line item for registry storage.
- INFRA-003 (Cloud Run) shows a Dockerfile but doesn't specify where the image is pushed.

Cloud Run on GCP requires images to be in Artifact Registry (or the deprecated Container Registry). This is a missing infrastructure component.

**Affected specs**: Spec 05 (INFRA-003, CI/CD Pipeline, INFRA-016), Cost estimate.

**Question**: Should Artifact Registry be added to the infrastructure specs and Terraform scope?

**Options**:

- **A**: **Add INFRA-018: Artifact Registry** — Define a Docker repository in `asia-southeast1`, add to Terraform scope, and add a line item to the cost estimate (likely free tier for < 500 MB). *(Recommended)*
- **B**: **Use default Container Registry** — Rely on `gcr.io/<project-id>` (auto-created, no explicit Terraform needed). Note: Container Registry is deprecated in favor of Artifact Registry.
- **C**: **Defer to implementation** — Let the developer choose during build pipeline setup.

**Recommendation**: Option A — Artifact Registry is the modern, recommended approach and should be explicitly documented as a Terraform-managed resource.

**Answer**
Use A.

---

### CLR-066: GCP API Enablement Not Documented as Bootstrap or Terraform Step 🟢

**Context**: Running Terraform on a GCP project requires various APIs to be enabled (e.g., `run.googleapis.com`, `cloudfunctions.googleapis.com`, `firestore.googleapis.com`, `compute.googleapis.com`, `cloudresourcemanager.googleapis.com`, `aiplatform.googleapis.com`, `dns.googleapis.com`, `cloudbuild.googleapis.com`, etc.). The specs don't mention API enablement anywhere:
- Not listed as a bootstrap step (like the project, state bucket, and SA).
- Not listed in the Terraform scope.
- The Terraform Google provider has a `google_project_service` resource for enabling APIs.

If APIs aren't enabled, Terraform will fail on the first resource it tries to create.

**Affected specs**: Spec 05 (INFRA-016 bootstrap resources or scope).

**Question**: Should GCP API enablement be documented?

**Options**:

- **A**: **Add to Terraform scope** — Terraform manages API enablement via `google_project_service` resources. Add a note to INFRA-016. *(Recommended)*
- **B**: **Add to bootstrap resources** — Document the required APIs in the bootstrap table. The project owner enables them manually before running Terraform.
- **C**: **Leave as implementation detail** — Experienced Terraform users know to enable APIs. No spec change needed.

**Recommendation**: Option A — managing API enablement in Terraform ensures reproducibility and is the standard IaC practice.

**Answer**
Use A.

---

### CLR-067: Terraform Operations Not Covered in Observability Spec 🟢

**Context**: Spec 07 (Observability) covers logging and alerting for Cloud Run, Cloud Functions, Cloud Armor, BigQuery, and Vertex AI. However, **Terraform operations are not covered**:
- No logging of `terraform plan` or `terraform apply` results.
- No alerting on Terraform apply failures in CI/CD.
- No mention of drift detection (comparing actual GCP state vs Terraform state).
- No audit trail for infrastructure changes beyond Git commit history.

GCP Cloud Audit Logs capture API calls made by the Terraform service account, but this isn't explicitly linked to the observability strategy.

**Affected specs**: Spec 07 (OBS-005 Alerting, OBS-006 Dashboards).

**Question**: Should Terraform operations be included in the observability spec?

**Options**:

- **A**: **Add OBS-009: Terraform Operations** — Document that Terraform plan/apply outputs are captured as CI/CD artifacts (GitHub Actions), that Cloud Audit Logs track the Terraform SA's API calls, and add an alert for Terraform apply failures. *(Recommended)*
- **B**: **Rely on CI/CD tooling** — GitHub Actions already logs pipeline runs. Cloud Audit Logs are automatically captured. No additional spec needed.
- **C**: **Defer to implementation** — Add Terraform observability as a future improvement.

**Recommendation**: Option B for a personal website — GitHub Actions logs and Cloud Audit Logs provide sufficient visibility. A brief note acknowledging this in spec 07 would be helpful.

**Answer**
Use B, but add a note in spec 07 acknowledging that Terraform operations are visible via CI/CD logs and Cloud Audit Logs, even if no custom dashboards or alerts are set up for them.

---

### CLR-068: Consolidated Service Account Inventory Missing 🟢

**Context**: Multiple service accounts are referenced across specs 05 and 06, but there is no single consolidated view. The accounts are:

| # | Service Account | Defined In | Purpose |
|---|----------------|------------|---------|
| 1 | Cloud Run default SA | SEC-011 (implicitly) | Cloud Run workload identity; reads Firestore Enterprise, Firestore Native, calls Vertex AI |
| 2 | `looker-studio-reader` | SEC-009, INFRA-011 | Read-only BigQuery access for Looker Studio |
| 3 | `content-cicd` | SEC-010 | WIF-mapped SA for content CI/CD → invoke embedding sync |
| 4 | `terraform-builder` | SEC-012 | Terraform infrastructure management |
| 5 | Cloud Scheduler SA | INFRA-008b, 008e, 014b (implicitly) | OIDC tokens for invoking Cloud Functions |
| 6 | Cloud Functions SAs | INFRA-008a, 008c, 008d, 014 (implicitly) | Per-function identity for Firestore/Vertex AI access |

The Cloud Run default SA (#1), Cloud Scheduler SA (#5), and Cloud Functions SAs (#6) are not formally named or documented in a single place. Their IAM roles are scattered across SEC-011 (partial), INFRA-008, and INFRA-014.

**Affected specs**: Spec 05 (INFRA components), Spec 06 (SEC sections).

**Question**: Should a consolidated service account inventory be added to the specs?

**Options**:

- **A**: **Add a service account summary table** to spec 06 (Security) listing all SAs, their email patterns, IAM roles, scope, and whether they are Terraform-managed or manually created. *(Recommended)*
- **B**: **Leave as-is** — The information is distributed but complete. Consolidation is an implementation-time task.
- **C**: **Add to spec 05** (Infrastructure) as a new section (e.g., INFRA-019: IAM & Service Accounts).

**Recommendation**: Option A — a consolidated view reduces the risk of missing IAM bindings during Terraform implementation and serves as a security audit reference.

**Answer**
Use A — add a service account summary table to spec 06 (Security) that lists all service accounts, their purposes, IAM roles, and management method (Terraform vs manual). This provides a clear inventory for security review and implementation reference. Also add service accounts that will be used by the Cloud Functions and Cloud Run. Don't use default service accounts. Create dedicated service accounts for each component for better security and auditability.

---

### CLR-069: INFRA-016 Cross-Reference Error — Cloud Scheduler Lists INFRA-008d Instead of INFRA-008e 🟢

**Context**: INFRA-016 (Terraform scope) lists: *"Cloud Scheduler jobs (INFRA-008b, **INFRA-008d scheduler**, INFRA-014b)"*. However:
- INFRA-008d is the Cloud Function `cleanup-rate-limit-offenders`.
- INFRA-008e is the Cloud Scheduler job `trigger-cleanup-rate-limit-offenders`.

The reference should be INFRA-008e, not INFRA-008d.

**Affected specs**: Spec 05 (INFRA-016).

**Question**: Can this cross-reference be corrected?

**Options**:

- **A**: Correct to *"Cloud Scheduler jobs (INFRA-008b, INFRA-008e, INFRA-014b)"*. *(Recommended)*
- **B**: Leave as-is — the intent is clear enough.

**Recommendation**: Option A — straightforward documentation fix.

**Answer**
Use A — correct the reference to INFRA-008e in INFRA-016.

---

### CLR-070: Terraform vs. CI/CD Deployment Boundary for Cloud Run and Cloud Functions 🟡

**Context**: Both Terraform and the Application CI/CD pipeline can manage Cloud Run services and Cloud Functions. The specs list Cloud Run (INFRA-003) and Cloud Functions (INFRA-008a/c/d, INFRA-014) in Terraform's scope AND in the CI/CD pipeline's Deploy stage. This creates ambiguity:

- **Terraform** typically manages the _service definition_ (name, region, scaling, networking, IAM, environment variables).
- **CI/CD** typically manages the _deployment_ (pushing a new container image revision or function code).

If both tools try to manage the same resource, they can conflict — for example, Terraform reverting a CI/CD-deployed image on the next `terraform apply`.

**Affected specs**: Spec 05 (INFRA-003, INFRA-008, INFRA-014, INFRA-016, CI/CD Pipeline).

**Question**: What is the boundary between Terraform and CI/CD for Cloud Run and Cloud Functions?

**Options**:

- **A**: **Terraform provisions, CI/CD deploys** — Terraform creates the Cloud Run service and Cloud Functions with initial configuration (scaling, memory, VPC, IAM). The CI/CD pipeline deploys new revisions (container images / function code) using `gcloud run deploy` or `gcloud functions deploy`. Terraform ignores the image/code version (uses `lifecycle { ignore_changes = [template[0].spec[0].containers[0].image] }` or equivalent). Document this pattern explicitly.
- **B**: **Terraform manages everything** — Terraform sets the container image tag too. The CI/CD pipeline only builds and pushes the image; Terraform applies the new tag. This means every deployment requires a Terraform apply.
- **C**: **CI/CD manages everything** — Remove Cloud Run and Cloud Functions from Terraform scope. The CI/CD pipeline creates/updates services via `gcloud` commands. Terraform only manages supporting infrastructure (VPC, LB, Cloud Armor, IAM).
- **D**: **Defer to implementation** — Document both options and decide during implementation.

**Recommendation**: Option A — this is the most common pattern and avoids Terraform/CI/CD conflicts. It should be explicitly documented in INFRA-016.

**Answer**
Use A — Terraform provisions the Cloud Run services and Cloud Functions with their configuration, while the CI/CD pipeline handles deploying new revisions (container images and function code). Terraform will ignore changes to the image field to prevent conflicts. This pattern should be clearly documented in the specs to guide implementation.

---

### CLR-071: Vertex AI Not Listed in Terraform Scope 🟢

**Context**: INFRA-013 defines the Vertex AI Gemini Embedding API configuration (model, region, task types, normalization). Vertex AI requires the `aiplatform.googleapis.com` API to be enabled, and IAM roles (`roles/aiplatform.user`) to be granted. While IAM is in Terraform's scope, the Vertex AI API enablement and any Vertex AI-specific Terraform configuration (if applicable) are not explicitly listed in INFRA-016.

For the current use case (calling the Embeddings API), there are no Terraform-managed Vertex AI resources (no custom models, endpoints, or datasets). The only Terraform tasks are API enablement and IAM.

**Affected specs**: Spec 05 (INFRA-013, INFRA-016).

**Question**: Should Vertex AI be explicitly mentioned in the Terraform scope?

**Options**:

- **A**: **Add a note** to INFRA-016 that Vertex AI API enablement is managed by Terraform (via `google_project_service`), and IAM roles for Vertex AI are part of the IAM bindings scope. No dedicated INFRA section needed since there are no Vertex AI resources to manage.
- **B**: **Leave as-is** — API enablement and IAM are already covered by "IAM bindings and service accounts" in the scope. No change needed.

**Recommendation**: Option A — an explicit mention avoids confusion during implementation.

**Answer**
Use A — add a note to INFRA-016 that Vertex AI API enablement is managed by Terraform (via `google_project_service`), and that IAM roles for Vertex AI are included in the Terraform-managed IAM bindings. This clarifies that Vertex AI is part of the infrastructure scope, even though there are no dedicated Vertex AI resources being managed by Terraform. No separate INFRA section is needed since the usage is limited to API calls from Cloud Run and Cloud Functions, but it should be explicitly acknowledged in the Terraform scope for clarity.

---

### CLR-072: Cloud Logging Log Bucket (INFRA-007) Not in Terraform Scope 🟢

**Context**: INFRA-007 mentions: *"Log sinks: Route error logs to a dedicated log bucket with 90-day retention."* This Cloud Logging log bucket is a separate resource from the BigQuery log sinks (INFRA-010). However:
- The log bucket is not given its own INFRA section or subsection.
- It is not listed in INFRA-016's Terraform scope.
- Its name, retention policy, and access controls are not specified.

Cloud Logging log buckets can be managed by Terraform (`google_logging_project_bucket_config`). If this bucket is intended to exist alongside the BigQuery sinks, it should be documented.

**Affected specs**: Spec 05 (INFRA-007, INFRA-016).

**Question**: Is the dedicated Cloud Logging error log bucket (INFRA-007) still needed given the BigQuery log sinks (INFRA-010)?

**Options**:

- **A**: **Keep and document** — The log bucket serves a different purpose (fast operational access with 90-day retention) from BigQuery (long-term analytics with 2-year retention). Add it to INFRA-007 with a name and retention config, and add to Terraform scope.
- **B**: **Remove** — The BigQuery `backend_error_logs` table (INFRA-010d) already captures ERROR-level logs. A separate log bucket is redundant. Remove the reference from INFRA-007.
- **C**: **Use Cloud Logging's default `_Default` bucket** — Cloud Logging automatically retains logs for 30 days in the default bucket. Adjust to 90 days if needed. No custom bucket required.

**Recommendation**: Option C — Cloud Logging's default bucket already retains logs for 30 days (configurable up to 3650 days). A custom bucket may not be necessary since BigQuery handles long-term analytics.

**Answer**
Use C — rely on Cloud Logging's default bucket for error logs, with a retention policy of 90 days. Remove the reference to a dedicated log bucket in INFRA-007 and clarify that the default bucket is used. This simplifies the infrastructure and avoids unnecessary resources while still meeting the requirement for operational log access.

---

## Summary

| CLR     | Severity | Title                                                    | Category                  |
| ------- | -------- | -------------------------------------------------------- | ------------------------- |
| CLR-059 | 🟡       | Firebase Hosting/Functions deployment boundary           | Terraform/IaC gaps        |
| CLR-060 | 🟡       | Firestore Enterprise missing from Terraform scope        | Terraform/IaC gaps        |
| CLR-061 | 🟡       | Cloud DNS has no INFRA section + wrong cross-reference   | Cross-reference + IaC     |
| CLR-062 | 🟡       | Terraform CI/CD authentication method unspecified        | Terraform/IaC gaps        |
| CLR-063 | 🟡       | Development environment scope ambiguous                  | Environment strategy      |
| CLR-064 | 🟡       | Pub/Sub topic for rate-limit log sink undocumented       | Infrastructure gap        |
| CLR-065 | 🟡       | Artifact Registry for Docker images not specified        | Infrastructure gap        |
| CLR-066 | 🟢       | GCP API enablement not documented                        | Terraform/IaC gaps        |
| CLR-067 | 🟢       | Terraform operations not in observability spec           | Observability             |
| CLR-068 | 🟢       | Consolidated service account inventory missing           | Security / documentation  |
| CLR-069 | 🟢       | INFRA-016 cross-reference error (008d → 008e)            | Cross-reference           |
| CLR-070 | 🟡       | Terraform vs CI/CD deployment boundary unclear           | Terraform/IaC gaps        |
| CLR-071 | 🟢       | Vertex AI not listed in Terraform scope                  | Terraform/IaC gaps        |
| CLR-072 | 🟢       | Cloud Logging log bucket not in Terraform scope          | Infrastructure gap        |
