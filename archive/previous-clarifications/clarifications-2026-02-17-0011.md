# Clarifications Batch 0011

**Date**: 2026-02-17
**Scope**: CLR-088 through CLR-095
**Status**: Awaiting answers

---

## CLR-088: SEC-006 CORS Missing `Access-Control-Expose-Headers` for `ETag` and `X-Request-ID` 🟡

**Files affected**: `specs/06-security-specifications.md` (SEC-006), `specs/03-backend-api-specifications.md` (BE-API-003, BE-API-005), `specs/07-observability-specifications.md` (OBS-008)

**Description**: SEC-006 specifies CORS `Allowed headers` (request headers) but does not specify `Access-Control-Expose-Headers` (response headers the frontend JavaScript can read from cross-origin responses). Two response headers require explicit exposure:

1. **`ETag`** — BE-API-003 and BE-API-005 return `ETag` as a response header. FE-COMP-008 uses it for conditional requests (`If-None-Match`). `ETag` is NOT a CORS-safelisted response header — without `Access-Control-Expose-Headers: ETag`, the frontend cannot read it from JavaScript on cross-origin responses from `api.tjmonsi.com`.

2. **`X-Request-ID`** — OBS-008 says this header is "useful for debugging with users" and is returned on every response. Without exposing it, the frontend cannot programmatically read it (e.g., to include in error reports).

(`Last-Modified` IS a CORS-safelisted response header and does not need explicit exposure.)

**Option A (Recommended)**: Add `Access-Control-Expose-Headers` to SEC-006:
```
Access-Control-Expose-Headers: ETag, X-Request-ID
```

**Option B**: Expose only `ETag` (minimum required for offline caching to work) and document `X-Request-ID` as visible only in browser DevTools, not programmatically.

**Answer**:
Use A.
---

## CLR-089: SEC-002 Progressive Banning Tier 2 and Tier 3 Missing Explicit Time Window 🟡

**Files affected**: `specs/06-security-specifications.md` (SEC-002), `specs/05-infrastructure-specifications.md` (INFRA-008c step 5)

**Description**: The progressive banning thresholds specify an explicit time window for Tier 1 only:

| Tier | Condition | Time Window |
|------|-----------|-------------|
| 1 | 5 offenses | **within 7 days** (explicit) |
| 2 | 2 offenses after 30-day ban expires | **unspecified** |
| 3 | 2 offenses after 90-day ban expires | **unspecified** |

For Tier 2 and 3, "2 offenses after ban expires" is ambiguous — does this mean 2 offenses at any point before the 90-day cleanup window (DM-009 retention), or within a specific time window (e.g., 7 days)?

The DM-009 cleanup logic (remove records with no active ban and no offenses in 90 days) provides an implicit upper bound, but this is a side effect of the cleanup policy, not an explicit banning rule.

**Option A (Recommended)**: Add an explicit time window for all tiers:
- Tier 1: 5 offenses within 7 days → 30-day ban
- Tier 2: 2 offenses within 7 days after 30-day ban expires → 90-day ban
- Tier 3: 2 offenses within 7 days after 90-day ban expires → indefinite ban

**Option B**: Explicitly state that Tier 2 and 3 have no time window — any 2 offenses before the record is cleaned up (90 days) trigger the next tier. Document this as intentional.

**Answer**:
Use A.
---

## CLR-090: DM-009 `ban_history` Sub-Field Schema Not Defined 🟢

**Files affected**: `specs/04-data-model-specifications.md` (DM-009)

**Description**: The `ban_history` field is defined as `object[]` with description "Array of past bans" but its sub-fields are not documented. The `current_ban` field has explicit sub-fields (`start`, `end`, `tier`), but `ban_history` does not — it's implied they mirror `current_ban`, but this is not stated.

**Suggested Resolution**: Add sub-field definitions for `ban_history`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ban_history[].start` | datetime | Yes | Ban start time |
| `ban_history[].end` | datetime | No | Ban end time (null = indefinite) |
| `ban_history[].tier` | string | Yes | Ban tier: `"30d"`, `"90d"`, `"indefinite"` |

**Option A (Recommended)**: Apply the suggested resolution above — add explicit sub-field schema mirroring `current_ban`.

**Option B**: Mark `ban_history` as "same schema as `current_ban`" with a cross-reference rather than duplicating the sub-field table.

**Answer**:
Use A.
---

## CLR-091: DM-005 `others` Collection Missing Slug Pattern Constraint 🟢

**Files affected**: `specs/04-data-model-specifications.md` (DM-005)

**Description**: DM-002 (`technical_articles`) and DM-003 (`blog_articles`) both specify a slug pattern constraint:
```
^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}$
```

DM-005 (`others`) has a `slug` field with a unique index (`idx_slug`) and description "URL-safe identifier" but does not specify a regex pattern constraint. The BE-API-007 response example shows an `others` slug following the same format (`external-article-2025-03-10-0800`), implying the same pattern applies, but this is not enforced in the schema.

**Option A (Recommended)**: Add the same slug pattern constraint to DM-005:
```
slug must match pattern: ^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}$
```

**Option B**: If `others` slugs intentionally allow a different format (e.g., no datetime suffix for external links), explicitly document the allowed pattern and update the BE-API-007 response example to reflect it.

**Answer**:
Use A.
---

## CLR-092: OBS-005 vs OBS-010 SLI Availability Breach Alert Mechanism Inconsistency 🟡

**Files affected**: `specs/07-observability-specifications.md` (OBS-005, OBS-010)

**Description**: The SLI availability breach alert is described differently in two places:

- **OBS-005** (Alert table): "Backend availability SLI drops below 99.9% over 1 hour"
  — This describes a simple **threshold-based** alert.

- **OBS-010** (Implementation): "An alert policy fires when the burn rate indicates the error budget will be exhausted (**fast-burn alert** over 1 hour window)"
  — This describes a **burn-rate-based** alert, which is a different alerting strategy.

A threshold check ("availability < 99.9% in 1 hour") and a burn-rate check ("error budget being consumed faster than expected over 1 hour") are distinct mechanisms with different alert sensitivity and behavior. These should be consistent.

**Option A (Recommended — Burn Rate)**: Update OBS-005 alert description to: "SLI availability burn rate exceeds threshold over 1 hour (see OBS-010)." Burn-rate alerts are the industry standard for SLO-based alerting and are natively supported by Cloud Monitoring Service Monitoring.

**Option B (Simple Threshold)**: Update OBS-010 to describe a simple threshold: "An alert policy fires when availability drops below 99.9% measured over a 1-hour window." Simpler to understand but less nuanced.

**Answer**:
Use A.
---

## CLR-093: Cloud Armor Adaptive Protection Language vs Standard Tier Capabilities 🟡

**Files affected**: `specs/05-infrastructure-specifications.md` (INFRA-005), `specs/01-system-overview.md` (AD-015), `docs/cost-estimate-draft.md`

**Description**: The spec describes Cloud Armor Adaptive Protection as providing "**automatic**, ML-based escalating DDoS mitigation" that "requires **no manual rule updates**" (AD-015, INFRA-005). This language implies automatic rule enforcement. However:

- **Cloud Armor Standard** tier: Adaptive Protection provides **alerts and rule recommendations** only — a human must manually review and apply suggested rules. Pricing: $5/policy + $1/rule.
- **Cloud Armor Enterprise** tier: Adaptive Protection can **automatically deploy** protective rules. Pricing: ~$200/month base subscription + per-request charges.

The cost estimate uses Standard tier pricing ($12.01/month for policy + 7 rules). If the "automatic, no manual rule updates" behavior described in the spec requires Enterprise tier, the cost jumps to ~$212+/month.

**Option A (Recommended — Use Standard tier, update spec language)**: Revise INFRA-005 and AD-015 to say Adaptive Protection provides "ML-based DDoS detection with recommended rules that the owner can review and apply" — accurately reflecting Standard tier capabilities. This keeps costs at ~$12/month. For a personal website, alerting-based Adaptive Protection is sufficient.

**Option B (Use Enterprise tier, update cost estimate)**: Keep the automatic enforcement language and add Cloud Armor Enterprise subscription ($200/month) to the cost estimate.

**Answer**:
Use A.
---

## CLR-094: Content CI/CD Pipeline Firestore Enterprise Write Credentials Not Documented in SA Inventory 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-010, SEC-013), `specs/05-infrastructure-specifications.md` (INFRA-014)

**Description**: The content CI/CD pipeline writes articles to Firestore Enterprise (INFRA-014 Integration Contract step 2). SEC-010 states: "The content pipeline pushes to Firestore Enterprise using its own credentials (separate from this project's IAM)."

However:
1. Firestore Enterprise lives in THIS GCP project — any write access requires IAM bindings in this project.
2. The `content-cicd@` SA (SEC-013 #6) only has `roles/cloudfunctions.invoker` and `roles/run.invoker` on the sync function — no Firestore access.
3. No other service account is designated for the content pipeline's Firestore writes.
4. Whether authentication is via GCP IAM or MongoDB wire protocol username/password is unspecified.

**Option A (Recommended)**: Add a note to SEC-013 and SEC-010 acknowledging that the content pipeline requires Firestore Enterprise write credentials. State that the SA or credential is provisioned as part of the content pipeline project (out of scope for this spec) but document the required permissions for reference:
- Firestore Enterprise write access to `technical_articles`, `blog_articles`, `others`, `categories`
- No access to other collections or services

**Option B**: Extend the existing `content-cicd@` SA (#6) to include Firestore Enterprise write access and use it for both Firestore writes and Cloud Function invocation.

**Answer**:
Use A.
---

## CLR-095: SEC-002 Ban Check Behavior for OPTIONS Preflight Requests Not Specified 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-002, SEC-003)

**Description**: SEC-003 states that `OPTIONS` requests "SHALL NOT be subject to rate limiting." However, SEC-002 states that "the Go application checks Firestore for ban status on each request." It's unspecified whether "each request" includes `OPTIONS` preflight requests.

If a client is banned, blocking their `OPTIONS` requests would cause CORS preflight failures. The browser would see a CORS error instead of the expected 403/404 response on the actual request, leading to confusing error handling in the frontend (FE-COMP-003 maps by HTTP status code, which it wouldn't receive if the preflight fails).

**Option A (Recommended)**: Explicitly exempt `OPTIONS` requests from ban checks, consistent with the rate-limiting exemption. Add to SEC-002: "OPTIONS preflight requests SHALL NOT be subject to ban status checks. Ban enforcement applies only to GET and POST requests."

**Option B**: Apply ban checks to `OPTIONS` requests too, and document that banned clients will see CORS errors. Update frontend error handling (FE-COMP-003) to include a note about CORS error behavior for banned clients.

**Answer**:
Use A.
---

## Summary

| CLR | Severity | Category | Brief Description |
|-----|----------|----------|-------------------|
| CLR-088 | 🟡 IMPORTANT | Cross-reference / CORS | Missing `Access-Control-Expose-Headers` for `ETag` and `X-Request-ID` |
| CLR-089 | 🟡 IMPORTANT | Ambiguity | Progressive banning Tier 2/3 has no explicit time window |
| CLR-090 | 🟢 SUGGESTION | Completeness | `ban_history` sub-field schema not defined in DM-009 |
| CLR-091 | 🟢 SUGGESTION | Consistency | `others` collection missing slug pattern constraint |
| CLR-092 | 🟡 IMPORTANT | Contradiction | SLI alert described as threshold in OBS-005, burn-rate in OBS-010 |
| CLR-093 | 🟡 IMPORTANT | Cost / Contradiction | Adaptive Protection "automatic" language implies Enterprise tier |
| CLR-094 | 🟢 SUGGESTION | Missing definition | Content CI/CD Firestore write credentials undocumented in SA inventory |
| CLR-095 | 🟢 SUGGESTION | Logical gap | Ban check behavior for OPTIONS preflight requests not specified |
