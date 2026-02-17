# Clarifications — 2026-02-17 — Batch 0007

## Architecture Review: Content CI/CD Pipeline & Embedding Sync Trigger

The following clarifications address gaps in how the content management CI/CD pipeline interacts with the `sync-article-embeddings` Cloud Function (INFRA-014) and the broader content ingestion flow.

---

### CLR-056: Content CI/CD Pipeline — Trigger Mechanism for `sync-article-embeddings` 🔴

**Context**: INFRA-014 specifies that `sync-article-embeddings` is an HTTP-triggered Cloud Function with `Ingress: Internal only`, invoked by the "content management CI/CD pipeline." However, the specs do not define:

1. **What the content CI/CD pipeline is** — Is it a GitHub Actions workflow in the content repository? A Cloud Build trigger? Something else?
2. **How the pipeline authenticates** to call an internal-only Cloud Function — Does it use Workload Identity Federation (recommended for GitHub Actions → GCP), a service account key, or something else?
3. **The exact invocation mechanism** — Does the pipeline call the function via `gcloud functions call`, an authenticated HTTP POST, or a Pub/Sub message?
4. **The service account identity** — What IAM roles does the CI/CD caller need? (`roles/cloudfunctions.invoker` at minimum, plus potentially `roles/run.invoker` since Gen 2 functions run on Cloud Run.)

The CI/CD section in spec 05 ([lines 258–280](specs/05-infrastructure-specifications.md#L258)) only covers the **application** CI/CD (frontend + backend). The **content** CI/CD pipeline is referenced throughout the specs (AD-008, INFRA-014, DM-005/DM-006/DM-007) but never formally specified.

**Question**: How should the content CI/CD pipeline invoke the `sync-article-embeddings` Cloud Function?

- **Option A**: **GitHub Actions + Workload Identity Federation** — The content repo's GitHub Actions workflow authenticates to GCP via [Workload Identity Federation](https://github.com/google-github-actions/auth) (no service account keys). After pushing content to Firestore Enterprise, the workflow calls the Cloud Function via `gcloud functions call sync-article-embeddings --region=asia-southeast1` or an authenticated HTTP POST.
- **Option B**: **GitHub Actions + Service Account Key** — Same as A but uses a stored service account JSON key as a GitHub secret. Simpler but less secure (key rotation burden).
- **Option C**: **Cloud Build trigger** — A Cloud Build trigger watches the content repository and runs a build that pushes content and calls the Cloud Function. Stays fully within GCP.
- **Option D**: **Pub/Sub event** — Instead of HTTP trigger, change INFRA-014 to a Pub/Sub trigger. The content pipeline publishes a message to a topic after pushing content, and the function picks it up asynchronously.
- **Option E**: **Defer to implementation** — Keep the spec intentionally vague and decide during implementation.

**Recommendation**: Option A (Workload Identity Federation) is the most secure and maintainable approach for GitHub Actions → GCP. No long-lived credentials to manage.

**Answer**
Use A.

---

### CLR-057: Content CI/CD Pipeline — Scope and Specification 🟡

**Context**: Multiple specs reference a "content management CI/CD pipeline" that:
- Pushes articles to Firestore Enterprise (AD-008, spec 01 Content Management section)
- Populates the `categories` collection (DM-005, spec 04)
- Triggers `sync-article-embeddings` (INFRA-014, spec 05)

But no spec formally defines this pipeline's stages, what it does, or where it runs. Spec 01 says *"A CI/CD pipeline in that repository processes markdown files and pushes structured content to the Firestore Enterprise database on merge"* — but the processing logic (markdown parsing, field extraction, slug generation, category derivation) is unspecified.

**Question**: Should the content CI/CD pipeline be formally specified in documentation?

- **Option A**: **Add a new section to spec 05** (e.g., `INFRA-015: Content CI/CD Pipeline`) defining the pipeline stages, trigger, authentication, and the sequence of operations (push content → call embedding sync → optionally trigger sitemap regeneration).
- **Option B**: **Defer to a separate spec** — Create a new spec file (e.g., `08-content-management-specifications.md`) dedicated to content management, including the pipeline, content format, markdown processing rules, and publishing workflow.
- **Option C**: **Defer entirely** — The content pipeline belongs to the content repository, not this project. Document only the integration points (how to call the Cloud Function, expected data format in Firestore) and leave the pipeline implementation to the content repo.

**Recommendation**: Option C — the content pipeline is a separate project. However, the integration contract (expected Firestore document schema, Cloud Function authentication requirements, and invocation format) should be clearly documented, either in INFRA-014 or as a brief addendum.

**Answer**
Opction C, with enhanced documentation of integration points in INFRA-014.

---

### CLR-058: INFRA-014 — Safety Net Cloud Scheduler Job 🟢

**Context**: INFRA-014 Notes state: *"The function MAY also be triggered by Cloud Scheduler as a periodic safety net (e.g., daily) to catch any missed syncs."* This is currently a MAY (optional), but if implemented, it would be a 3rd Cloud Scheduler job. The cost estimate currently lists 2 Cloud Scheduler jobs (within the free tier of 3).

**Question**: Should this safety-net Cloud Scheduler job be formally defined?

- **Option A**: **Yes, define it** — Add `INFRA-014b: Cloud Scheduler — trigger-sync-article-embeddings` (e.g., daily at 04:00 UTC). This uses the 3rd free Cloud Scheduler slot and ensures embeddings are always in sync even if the content CI/CD pipeline fails to trigger the function.
- **Option B**: **No, keep it as MAY** — Leave it optional and defer to implementation. The content CI/CD pipeline is the primary trigger, and a missed sync can be manually corrected.
- **Option C**: **Remove the MAY note** — Simplify the spec by removing the safety net mention entirely. If the CI/CD pipeline doesn't trigger it, the owner can manually invoke it.

**Recommendation**: Option A — it uses a free Cloud Scheduler slot, adds resilience at zero cost, and the function is idempotent so re-running is harmless.

**Answer**
Yes, define the safety net Cloud Scheduler job as `INFRA-014b`.

---

## Summary

| CLR     | Topic                                      | Priority  | Needs Decision |
| ------- | ------------------------------------------ | --------- | -------------- |
| CLR-056 | CI/CD trigger mechanism for embedding sync | 🔴 High   | Yes            |
| CLR-057 | Content pipeline scope and specification   | 🟡 Medium | Yes            |
| CLR-058 | Safety net scheduler for embedding sync    | 🟢 Low    | Yes            |
