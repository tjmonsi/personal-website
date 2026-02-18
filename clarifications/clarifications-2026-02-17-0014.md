````markdown
# Clarifications Batch 0014

**Date**: 2026-02-17
**Scope**: CLR-105 through CLR-112
**Status**: Awaiting answers

---

## CLR-105: Acceptance Criteria Field Name and Dimension Errors in spec-04 🟡

**Files affected**: `specs/04-data-model-specifications.md` (AC-DM-002, AC-DM-003, AC-DM-004)

**Description**: Three acceptance criteria in spec-04 contain field names or values that do not match the schemas they reference:

1. **AC-DM-002** lists required fields as `slug, category, tags, title, abstract, body, citations, changelog, date_created, date_updated`. However, the DM-002 schema defines the field as `content` (not `body`), and the DM-002 notes explicitly state: *"Citations are generated dynamically by the backend from article metadata. No citation field is stored."*

2. **AC-DM-003** states vectors have a "768-dimension float array", but every other reference in the specs (DM-012, INFRA-012, INFRA-013, AD-020) specifies **2048 dimensions**.

3. **AC-DM-004** references `ban_expires_at` and `active_ban`, but DM-009 defines `current_ban.end` and `current_ban` (an object, not a boolean `active_ban`).

**Option A (Recommended)**: Fix all three to match the schemas:
- AC-DM-002: Replace `body` with `content`; remove `citations`
- AC-DM-003: Replace `768` with `2048`
- AC-DM-004: Replace `ban_expires_at` with `current_ban.end`; replace `active_ban` with `current_ban`

**Option B**: Keep current wording and update the schemas instead.

**Answer**:
Use A.
---

## CLR-106: Architecture Diagrams and Sitemap Missing New Components 🟡

**Files affected**: `specs/01-system-overview.md` (architecture diagram), `specs/05-infrastructure-specifications.md` (network diagram, INFRA-008a sitemap)

**Description**: After adding the image endpoint (BE-API-013), media bucket (INFRA-019), and `/changelog` page (FE-PAGE-010), the following references were not updated:

1. The **High-Level Architecture diagram** in spec-01 does not list `GET /images/{path}` among the Cloud Run endpoints or show the Cloud Storage media bucket as a connected component.

2. The **Network Diagram** in spec-05 does not show the Cloud Run → Cloud Storage media bucket relationship.

3. Both diagrams do not include `/changelog` in the Firebase Hosting pages list.

4. **INFRA-008a sitemap generation** (spec-05) lists static pages as: `/`, `/technical`, `/blog`, `/socials`, `/others`, `/privacy` — missing `/changelog`.

**Option A (Recommended)**: Update all four locations:
- Add `GET /images/{path}` to Cloud Run endpoint list in both diagrams
- Add Cloud Storage media bucket with a connecting arrow from Cloud Run in both diagrams
- Add `/changelog` to Firebase Hosting pages list in both diagrams
- Add `/changelog` to INFRA-008a sitemap static pages list

**Option B**: Leave diagrams as-is and add a separate "Recent Additions" note instead.

**Answer**:
Use A
---

## CLR-107: OBS-010 SLO Rolling Window Inconsistency (30-day vs 28-day) 🟡

**Files affected**: `specs/07-observability-specifications.md` (OBS-010, AC-OBS-005)

**Description**: OBS-010 defines the SLO compliance period as a **"Rolling 30-day window"**, but AC-OBS-005 references a **"rolling 28-day window"**. These must be consistent.

- **30-day** is a calendar-month approximation, simpler to understand.
- **28-day** is the industry standard used by Google SRE and Google Cloud Monitoring's default SLO evaluation period (exactly 4 weeks, avoids month-length variability).

**Option A**: Use **30-day** everywhere (update AC-OBS-005 to match OBS-010).
**Option B (Recommended)**: Use **28-day** everywhere (update OBS-010 to match AC-OBS-005). This aligns with Google Cloud Monitoring defaults and Google SRE best practices.

**Answer**:
Use B
---

## CLR-108: Image Cache Observability Metrics for BE-API-013 🟢

**Files affected**: `specs/07-observability-specifications.md` (OBS-004), `specs/03-backend-api-specifications.md` (BE-API-013)

**Description**: BE-API-013 specifies an in-memory LRU cache on the Go server with a 30-day TTL for proxied images. However, OBS-004 defines no custom metrics for this cache. Without metrics, there is no way to monitor cache effectiveness or diagnose performance issues.

**Option A (Recommended)**: Add image cache metrics to OBS-004:
- `image_cache_hits_total` (Counter) — Image cache hits
- `image_cache_misses_total` (Counter) — Image cache misses (Cloud Storage fetch)

**Option B**: Skip cache metrics — the cache is simple enough to debug without dedicated metrics.

**Answer**:
Use A
---

## CLR-109: Security Hardening — CSP `object-src` and Threat Model Terminology 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-005, threat model table)

**Description**: Two minor security-related items:

1. **CSP `object-src`**: The frontend CSP defines `default-src 'self'` but does not explicitly set `object-src 'none'`. This means `<object>`, `<embed>`, and `<applet>` elements default to `'self'`. Best practice is to explicitly block these with `object-src 'none'`.

2. **Threat model terminology**: The threat model table row says "JWT bearer token authentication on POST /t". The phrase "bearer token" may imply `Authorization: Bearer` header, but per CLR-103 the implementation uses a `token` field in the JSON request body. This could confuse a developer.

**Option A (Recommended)**: Apply both fixes:
- Add `object-src 'none'` to the frontend CSP string in SEC-005 (both table and `firebase.json` example)
- Reword threat model to: "JWT authentication via request body `token` field on POST /t"

**Option B**: Apply only the threat model terminology fix; leave CSP as-is.

**Option C**: No changes.

**Answer**:
Use A
---

## CLR-110: Media Bucket Object Versioning 🟢

**Files affected**: `specs/05-infrastructure-specifications.md` (INFRA-019)

**Description**: INFRA-019 sets versioning to `Disabled` for the media bucket. Unlike the Terraform state bucket (INFRA-015, versioning enabled), media images have no recovery mechanism if accidentally deleted or overwritten. Enabling versioning adds minimal cost (only charges for stored non-current versions).

**Option A**: Enable versioning on the media bucket for deletion/overwrite protection.
**Option B (Recommended)**: Keep versioning disabled and add a note acknowledging that images can be re-uploaded from source if needed. This keeps the bucket simple for a personal website with minimal images.

**Answer**:
Use A
---

## CLR-111: Privacy Policy Does Not Mention Server-Computed `visitor_id` 🟢

**Files affected**: `specs/02-frontend-specifications.md` (FE-PAGE-009 "Data We Collect" section)

**Description**: FE-PAGE-009's privacy policy describes the `visitor_session_id` (generated client-side) but does not mention the `visitor_id` — a SHA-256 hash computed server-side from `visitor_session_id` + truncated IP + User-Agent. While this derived value is non-reversible and session-scoped (not strictly personal data), GDPR best practices favor explicit disclosure of all processing.

**Option A (Recommended)**: Add to the "Data We Collect" section: *"A non-reversible session-scoped identifier derived from the session ID, truncated IP address, and browser information, used for unique visitor counting."*

**Option B**: No change — the `visitor_id` is a non-reversible derivative of already-disclosed data and does not require separate disclosure.

**Answer**:
Use A

---

## CLR-112: Looker Studio Missing from Cost Estimate Summary 🟢

**Files affected**: `docs/cost-estimate-draft.md` (cost breakdown summary table)

**Description**: The cost breakdown summary table does not have an explicit line item for Looker Studio. Its cost is implicitly covered by BigQuery query costs, but other free-tier services (Cloud Scheduler, Cloud Monitoring) are listed explicitly. Adding it would make the cost estimate more complete.

**Option A (Recommended)**: Add a `Looker Studio | $0.00 | $0.00 | Free (owner-operated; query costs covered under BigQuery)` row to the summary table.

**Option B**: No change — Looker Studio is not an infrastructure resource and its cost impact is already captured under BigQuery.

**Answer**:
Use A
---

## Summary

| CLR | Severity | Category | Brief Description |
|-----|----------|----------|-------------------|
| CLR-105 | 🟡 IMPORTANT | Data Model | Acceptance criteria AC-DM-002/003/004 contain wrong field names and vector dimension |
| CLR-106 | 🟡 IMPORTANT | Architecture | Diagrams and sitemap missing image endpoint, media bucket, and `/changelog` |
| CLR-107 | 🟡 IMPORTANT | Observability | SLO rolling window 30-day vs 28-day inconsistency between OBS-010 and AC-OBS-005 |
| CLR-108 | 🟢 SUGGESTION | Observability | Image cache metrics for BE-API-013 LRU cache |
| CLR-109 | 🟢 SUGGESTION | Security | CSP `object-src 'none'` hardening and threat model terminology |
| CLR-110 | 🟢 SUGGESTION | Infrastructure | Media bucket versioning decision |
| CLR-111 | 🟢 SUGGESTION | Privacy | Privacy policy missing server-computed `visitor_id` disclosure |
| CLR-112 | 🟢 SUGGESTION | Cost Estimate | Looker Studio missing from cost breakdown summary |

````

## Additional Notes:
- add a tldr section at the top of the privacy policy page for quick reference. Make it clear and concise but still playful and engaging to match the website's tone. The tldr should summarize the key points of the privacy policy in a way that's easy to understand at a glance.
- add a tldr section at the top of changelog page for quick reference. The tldr should summarize the most important recent changes in a way that's easy to understand at a glance.