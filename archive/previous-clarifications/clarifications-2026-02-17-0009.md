# Clarifications — 2026-02-17 — Batch 0009

## Architecture Review: Post CLR-059–072 Gap Analysis

The following items were identified during a comprehensive architecture review of all 7 spec documents (v2.3–v2.7) and the cost estimate (v0.6-draft), after applying CLR-059 through CLR-072. The review focused on cross-reference consistency, new section integration (INFRA-017, INFRA-018, SEC-013, OBS-009), and remaining specification gaps.

Previously resolved clarifications (CLR-001 through CLR-072) were reviewed and excluded.

---

### CLR-073: INFRA-016 API Enablement List Missing WIF and Eventarc APIs 🟡

**Context**: The API enablement list in INFRA-016 (spec 05) includes many GCP APIs but omits three required for Workload Identity Federation and Cloud Functions Gen 2 Eventarc triggers:

- `sts.googleapis.com` (Security Token Service — required for WIF OIDC token exchange, used by both SEC-010 and SEC-012)
- `iamcredentials.googleapis.com` (IAM Credentials — required for WIF to generate short-lived credentials)
- `eventarc.googleapis.com` (Eventarc — used by INFRA-008c for the Pub/Sub → Cloud Function Gen 2 trigger)

Without these APIs enabled, WIF authentication and the rate-limit log processing trigger will fail at deployment.

**Affected specs**: Spec 05 (INFRA-016 API enablement list).

**Question**: Should `sts.googleapis.com`, `iamcredentials.googleapis.com`, and `eventarc.googleapis.com` be added to the INFRA-016 API enablement list?

**Options**:

- **A**: **Add all three APIs** — Add `sts.googleapis.com`, `iamcredentials.googleapis.com`, and `eventarc.googleapis.com` to the API enablement example list in INFRA-016. *(Recommended)*
- **B**: **Add only STS and IAM Credentials** — Add the WIF-required APIs but leave Eventarc out (treat it as implicitly enabled by Cloud Functions Gen 2 deployment).
- **C**: **Leave as-is** — Rely on implicit API enablement during deployment.

**Recommendation**: Option A — All three are required infrastructure dependencies. Explicit enablement is best practice for Terraform-managed environments.

**Answer**:
Use A.
---

### CLR-074: Terraform CI/CD WIF Pool/Provider Missing from Bootstrap Resources 🟡

**Context**: SEC-012 defines a WIF pool (`terraform-cicd-pool`) and provider for the Terraform CI/CD pipeline. However, the bootstrap resources table in INFRA-016 does not list this WIF pool/provider. This is a chicken-and-egg dependency: the Terraform CI/CD pipeline needs the WIF pool to authenticate, but the WIF pool must exist before Terraform CI/CD can run — making it a bootstrap resource (same logic as the Terraform SA and state bucket).

Additionally, the content CI/CD WIF pool (`content-cicd-pool`, SEC-010) is **not** a bootstrap resource (Terraform can manage it), but it's not explicitly listed in the INFRA-016 Terraform scope either. The scope says "IAM bindings and service accounts" but WIF pools/providers are distinct resource types (`google_iam_workload_identity_pool`, `google_iam_workload_identity_pool_provider`).

**Affected specs**: Spec 05 (INFRA-016 bootstrap resources table, Terraform scope), Spec 06 (SEC-012).

**Question**: Should the Terraform CI/CD WIF pool/provider be added to the bootstrap resources, and the content CI/CD WIF pool to the Terraform scope?

**Options**:

- **A**: **Add both** — Add Terraform CI/CD WIF pool/provider to the bootstrap resources table (with reason: "Must exist before Terraform CI/CD pipeline can authenticate"). Add content CI/CD WIF pool/provider (SEC-010) to the INFRA-016 Terraform scope list. *(Recommended)*
- **B**: **Bootstrap only** — Add Terraform CI/CD WIF pool/provider to bootstrap resources. Keep content CI/CD WIF pool implicit under "IAM bindings and service accounts."
- **C**: **Leave as-is** — Treat WIF setup as a pre-deployment manual step documented elsewhere.

**Recommendation**: Option A — The chicken-and-egg dependency is real and must be documented. Content CI/CD WIF pool should be explicitly listed in Terraform scope for clarity.

**Answer**:
Use A. 
---

### CLR-075: INFRA-011 Unique Visitors Metric Description Inconsistent with OBS-002/OBS-006 🟡

**Context**: The INFRA-011 analytics capabilities table (spec 05) describes unique visitors as:

> `Distinct IP addresses per time period`

This contradicts OBS-006 (spec 07) which correctly says:

> `Unique visitors (distinct visitor_id values per time period)`

The `visitor_id` is the intended unique visitor identifier — a SHA-256 hash of `visitor_session_id` + truncated IP + User-Agent (see BE-API-009 in spec 03 and OBS-002 in spec 07). Using raw IP addresses would count shared-IP users as one visitor and miss the session-scoping purpose of `visitor_id`.

**Affected specs**: Spec 05 (INFRA-011), Spec 07 (OBS-002, OBS-006), Spec 03 (BE-API-009).

**Question**: Should the INFRA-011 unique visitors description be corrected to match OBS-002 and OBS-006?

**Options**:

- **A**: **Fix to match OBS-002/OBS-006** — Change "Distinct IP addresses per time period" to "Distinct `visitor_id` values per time period" in INFRA-011. *(Recommended)*
- **B**: **Keep both descriptions** — INFRA-011 describes a simplified view; OBS-002/OBS-006 describe the implementation detail.
- **C**: **Defer** — Resolve during implementation.

**Recommendation**: Option A — Consistency across specs prevents implementation confusion.

**Answer**:
Use A. 
---

### CLR-076: SEC-013 SA #2 (Sitemap Gen) Firestore Access Scope Incomplete 🟡

**Context**: In the SEC-013 service account inventory (spec 06), SA #2 (`cf-sitemap-gen@`) lists its Firestore Enterprise access as:

> `Firestore Enterprise read/write (sitemap collection)`

But the sitemap generation logic in INFRA-008a (spec 05) step 1 says:

> "Query all published articles from `technical_articles` and `blog_articles` collections."

SA #2 needs **read** access to `technical_articles` (DM-002), `blog_articles` (DM-003), and potentially `others` (DM-005) to generate sitemap URLs — not just read/write on the sitemap collection.

**Affected specs**: Spec 06 (SEC-013), Spec 05 (INFRA-008a).

**Question**: Should SA #2's IAM role description be expanded to reflect its full Firestore access pattern?

**Options**:

- **A**: **Expand SA #2 access description** — Update to: "Firestore Enterprise read (article collections: `technical_articles`, `blog_articles`) + read/write (`sitemap` collection), `roles/cloudfunctions.invoker` (self, for Cloud Scheduler OIDC)." *(Recommended)*
- **B**: **Keep generic** — Use "Firestore Enterprise read/write (article and sitemap collections)" without enumerating specific collection names.
- **C**: **Defer** — Resolve during implementation.

**Recommendation**: Option A — Explicit collection-level access documentation supports least-privilege auditing.

**Answer**:
Use A.
---

### CLR-077: INFRA-003 Missing Cross-Reference to INFRA-018 (Artifact Registry) 🟢

**Context**: INFRA-003 (Cloud Run) documents the Docker image build (Dockerfile) and container security, but doesn't mention where the built image is stored. INFRA-018 (Artifact Registry) references INFRA-003, but the reverse cross-reference is missing. Adding a note to INFRA-003 about the Artifact Registry destination would complete the bidirectional reference.

**Affected specs**: Spec 05 (INFRA-003, INFRA-018).

**Question**: Should a cross-reference from INFRA-003 to INFRA-018 be added?

**Options**:

- **A**: **Add cross-reference** — Add a note to INFRA-003 under Docker Image or Configuration: "Docker images are pushed to Artifact Registry (INFRA-018) by the CI/CD pipeline. Cloud Run pulls images from `asia-southeast1-docker.pkg.dev/<project-id>/website-images/`." *(Recommended)*
- **B**: **Leave as-is** — The unidirectional reference from INFRA-018 to INFRA-003 is sufficient.

**Recommendation**: Option A — Bidirectional cross-references improve discoverability.

**Answer**:
Use A.
---

### CLR-078: Spec-01 Architecture Diagram Omits Pub/Sub in Log Processing Flow 🟢

**Context**: The high-level architecture diagram in spec 01 shows the rate-limit log processing flow as:

```
Cloud Armor Log Sink ───▶ Cloud Function (Rate Limit Log Processing)
```

The actual flow (documented in INFRA-008c, spec 05) goes through Pub/Sub:

```
Cloud Armor Log → Log Sink → Pub/Sub (rate-limit-events) → Cloud Function
```

The network diagram in spec 05 correctly shows the Pub/Sub intermediary. The spec-01 diagram is simplified but omits an infrastructure component that is separately costed, Terraform-managed, and defined in the tech stack table.

**Affected specs**: Spec 01 (High-Level Architecture diagram).

**Question**: Should the spec-01 architecture diagram be updated to show the Pub/Sub intermediary?

**Options**:

- **A**: **Update diagram** — Show the full flow with Pub/Sub in the spec-01 architecture diagram. *(Recommended)*
- **B**: **Add a note only** — Keep the simplified diagram but add a footnote: "Log processing routes through Pub/Sub (see INFRA-008c)."
- **C**: **Leave as-is** — Spec-01 diagram is intentionally high-level and doesn't need to show every intermediary.

**Recommendation**: Option A — Pub/Sub is now a named tech stack component with its own cost line and Terraform scope entry.

**Answer**:
Use A.
---

### CLR-079: INFRA-016 Terraform Scope Should Explicitly List the Pub/Sub Log Sink 🟢

**Context**: The INFRA-016 Terraform scope lists "Pub/Sub topics and subscriptions (INFRA-008c log sink trigger — `rate-limit-events` topic)" and "BigQuery dataset and log sinks (INFRA-010)." However, the Cloud Logging log sink that routes Cloud Armor 429 events to the Pub/Sub topic is a separate Terraform resource (`google_logging_project_sink`) from the 5 BigQuery log sinks. It's not explicitly listed in the scope. The scope currently implies all log sinks are BigQuery-bound (INFRA-010), but this one routes to Pub/Sub.

**Affected specs**: Spec 05 (INFRA-016 Terraform scope).

**Question**: Should the Pub/Sub-bound log sink be explicitly distinguished from the BigQuery log sinks in the INFRA-016 Terraform scope?

**Options**:

- **A**: **Expand log sinks entry** — Change to: "Cloud Logging log sinks — BigQuery sinks (INFRA-010a–010e) + Pub/Sub sink for rate-limit events (INFRA-008c)." *(Recommended)*
- **B**: **Leave as-is** — The Pub/Sub topic reference already implies the log sink exists. A separate entry is unnecessary.

**Recommendation**: Option A — Explicit is better than implicit for IaC scope documentation.

**Answer**:
Use A.
---

### CLR-080: Spec-05 Terraform CI/CD Section Missing Cross-Reference to OBS-009 🟢

**Context**: The Terraform CI/CD pipeline section in spec 05 describes the pipeline stages and authentication but doesn't reference OBS-009 (spec 07), which documents observability for Terraform operations (GitHub Actions logs, Cloud Audit Logs, Git history). Adding a cross-reference improves discoverability.

**Affected specs**: Spec 05 (Terraform CI/CD pipeline section), Spec 07 (OBS-009).

**Question**: Should a cross-reference from the Terraform CI/CD section to OBS-009 be added?

**Options**:

- **A**: **Add cross-reference** — Add a note to the Terraform CI/CD pipeline section: "For observability of Terraform operations, see OBS-009 in `07-observability-specifications.md`." *(Recommended)*
- **B**: **Leave as-is** — OBS-009 already exists and is discoverable via the spec-07 table of contents.

**Recommendation**: Option A — Cross-references between related specs improve navigability.

**Answer**:
Use A.
---

### CLR-081: SEC-013 SAs #2, #4, #5 Potentially Redundant `roles/cloudfunctions.invoker` (Self) 🟢

**Context**: In SEC-013 (spec 06), SAs #2 (`cf-sitemap-gen`), #4 (`cf-offender-cleanup`), and #5 (`cf-embed-sync`) each list `roles/cloudfunctions.invoker` (self, for Cloud Scheduler OIDC). However, SA #9 (`cloud-scheduler-invoker`) already holds `roles/cloudfunctions.invoker` and `roles/run.invoker` on the target functions — which is the standard pattern for Cloud Scheduler OIDC invocation. The invoking SA (#9) needs the invoker role, not the target function's runtime SA.

If the self-grant on SAs #2, #4, #5 is intentional (e.g., a Cloud Functions Gen 2 / Cloud Run requirement for OIDC token audience validation), this should be documented with an explanatory note. If it's not required, it's an over-provisioning that should be removed for least-privilege compliance.

**Affected specs**: Spec 06 (SEC-013).

**Question**: Is the `roles/cloudfunctions.invoker` (self) grant on SAs #2, #4, #5 intentional or redundant with SA #9?

**Options**:

- **A**: **Remove redundant grants** — Remove `roles/cloudfunctions.invoker` (self) from SAs #2, #4, #5. SA #9 (`cloud-scheduler-invoker`) already has the required invoker role on the target functions. *(Recommended)*
- **B**: **Keep and document** — Add an explanatory note to SEC-013 clarifying why both the invoker SA and the target SA need `roles/cloudfunctions.invoker` (e.g., OIDC audience validation requirement for Cloud Functions Gen 2).
- **C**: **Defer** — Investigate during implementation and resolve based on actual Cloud Functions Gen 2 behavior.

**Recommendation**: Option A — Standard Cloud Scheduler OIDC invocation only requires the invoker SA (SA #9) to hold `roles/cloudfunctions.invoker`. The runtime SAs do not need self-invocation rights.

**Answer**:
Use A.
---

## Verified Items — No Gaps Found

| Check | Result |
|-------|--------|
| Remaining "dedicated log bucket" references (CLR-072) | ✅ None found — only correct "No dedicated custom log bucket" statement remains |
| Version numbers | ✅ All correctly bumped (spec-01 v2.5, spec-05 v2.7, spec-06 v2.5, spec-07 v2.3, cost v0.6) |
| INFRA-017 (Cloud DNS) cross-references | ✅ Present in spec-01 tech stack, definitions, deployment topology, architecture diagram, spec-05 section + network diagram, cost estimate |
| INFRA-018 (Artifact Registry) cross-references | ✅ Present in spec-01 tech stack, definitions, deployment topology, spec-05 section + INFRA-016 scope + network diagram, cost estimate |
| SEC-013 SA inventory alignment (general) | ✅ All 9 SAs map to their corresponding INFRA/SEC sections |
| OBS-009 content | ✅ Properly documents Terraform observability via existing mechanisms |
| Cost estimate coverage | ✅ All infrastructure components have line items (Cloud DNS, Artifact Registry, Pub/Sub via changelog note) |
| SEC-012 WIF ↔ spec-05 Terraform CI/CD consistency | ✅ Spec-05 references SEC-012 correctly |
| Spec-01 tech stack matches new INFRA sections | ✅ DNS, Container Registry, Pub/Sub rows present and accurate |

## Additional Notes to be added
- Add alerts to be emailed to a specified email (email is obfuscated for now) on Cloud log errors on Backend and Cloud Functions and errors posted by Frontend through Cloud logs. This will be added to the cost estimate as well (minimal cost for email notifications). Add SLI objective on availability of backend only for now (99.9% uptime target). This will be added to the SRE handbook as well. Add this on the costing as well. Alerts for SLI will be the same specified email. This will be added to the cost estimate as well (minimal cost for email notifications).