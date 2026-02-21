# Gap Analysis v3 — 2026-02-21

**Context**: After applying all 19 v2 gap answers and performing a deep re-analysis of the complete spec suite (specs 01–07), the following 7 genuine gaps remain. A subagent identified 19 candidate findings; 12 were validated as false positives (already addressed in the current specs) and dismissed. The remaining 7 are documented below.

**Dismissed False Positives** (for audit trail):

| RE-GAP | Reason Dismissed |
|--------|-----------------|
| RE-GAP-001 | Cloud Armor 429 response body IS defined in SEC-002 (lines 107-116) with custom JSON + frontend note about status-code-only matching |
| RE-GAP-003 | OPTIONS exit at middleware step 2 (before rate limiting), CORS preflight cache `max-age:86400` makes it negligible |
| RE-GAP-004 | Pub/Sub log sink IS formally defined in INFRA-008c (lines 392-409) with topic, subscription, filter, and message retention |
| RE-GAP-005 | `markdown-it` + `DOMPurify` specified at spec 02 line 125 (CLR-159) |
| RE-GAP-006 | All 5 citation format templates (APA, MLA, Chicago, BibTeX, IEEE) shown in BE-API-003 example response (spec 03 lines 433-437) |
| RE-GAP-008 | 6-step heading anchor generation algorithm specified at spec 02 lines 132-140, including duplicate suffix and empty fallback |
| RE-GAP-009 | `content_hash` already added to DM-001 (line 36), DM-002 (line 58), DM-005 (line 148) |
| RE-GAP-011 | GeoLite2 download URL, `MAXMIND_LICENSE_KEY` secret, and weekly rebuild schedule defined at spec 05 lines 135-137 |
| RE-GAP-002 | Changelog data source IS defined: "local markdown file bundled with the SPA" (CLR-187), entry structure (version/date + description), TL;DR section, no API fetch |
| RE-GAP-013 | Error report offline queuing fully defined: memory queue without JWT, `sessionStorage` persistence, `online` event flush, fresh JWT at send time (spec 02 lines 512-517) |
| RE-GAP-016 | Browser name/version extraction specified: `navigator.userAgentData` with `navigator.userAgent` fallback, format `"<Browser> <Major Version>"` (spec 02 line 450) |
| RE-GAP-018 | Frontpage data flow IS defined: FE-PAGE-001 "fetch from `GET /`" + AC-FE-028 + rendering as raw markdown (spec 02 lines 51, 940) |

---

## Remaining Gaps

### V3-GAP-001 — MEDIUM — Specs: 02, 03

**Vector Search Post-Filter Exhaustion Behavior**

**Gap**: When the vector search returns K nearest-neighbor results (step 3–4) but ALL of them are subsequently filtered out by the post-filter step (step 5: category, date range, tag filters), the spec does not explicitly document the expected behavior.

**Current state**: Step 5 applies `$in` / `$all` for tags, date range, and category filters to the Firestore Native results, then transforms and returns them. If all results are filtered out, the implicit behavior is an empty `items` array — but this is not documented.

**Why it matters**: An implementer might wonder whether to increase K and retry, return an explanatory message, or simply return empty results. Documenting the intended behavior removes ambiguity.

**Affected sections**:
- Spec 03: BE-API-002 vector search flow step 5 (and equivalents for BE-API-007)
- Spec 02: FE-PAGE-002 list view (empty state rendering)

**Suggested answer options**:
- (A) Return empty `items` array with `total_results: 0` — document as intended behavior, no retry with larger K
- (B) Retry with a larger K (e.g., 2×K) once before returning empty — adds complexity
- (C) Other

---

### V3-GAP-002 — MEDIUM — Specs: 02

**Mobile Infinite Scroll Deep Link Behavior**

**Gap**: Desktop pagination uses `?page=N` in the URL. Mobile uses infinite scroll. The URL State Synchronization spec says "WHEN the user loads the page with URL query parameters present, THE SYSTEM SHALL initialize the search bar, filter controls, and pagination from those parameters." But on mobile with infinite scroll, what happens when a shared URL contains `?page=5`?

**Current state**: The spec doesn't define whether mobile should:
- Load all pages 1–5 sequentially on initial load
- Load only page 5's results and allow upward scrolling
- Ignore the `page` parameter and start from page 1

**Why it matters**: Users share URLs across devices. A desktop user on page 5 might share a link that a mobile user opens. The behavior should be defined to avoid inconsistent or confusing UX.

**Affected sections**:
- Spec 02: FE-PAGE-002 URL State Synchronization + Mobile pagination
- Spec 02: FE-PAGE-004 (same behavior)
- Spec 02: FE-PAGE-007 (same behavior)

**Suggested answer options**:
- (A) Mobile loads pages 1 through N sequentially on initial load (user sees all previous content), then infinite scroll continues from page N+1
- (B) Mobile starts from page 1 regardless, ignoring the `page` parameter on mobile viewports
- (C) Mobile loads page N only, and upward scroll fetches N-1, N-2, etc.
- (D) Other

---

### V3-GAP-003 — MEDIUM — Specs: 05

**Firebase Hosting SPA Catch-All Rewrite Missing from `firebase.json`**

**Gap**: INFRA-001's `firebase.json` example only shows the `/sitemap.xml` → Cloud Run rewrite rule. The SPA catch-all rewrite that routes all non-asset routes to the Firebase Function (INFRA-002) for serving the SPA shell is described in prose ("Firebase Functions handles SPA serving") but not included in the JSON example.

**Current state**: The text says "Firebase Functions handles SPA serving (all non-asset routes serve the SPA shell)" but the `firebase.json` rewrites array only contains the sitemap entry.

**Why it matters**: An implementer copying the firebase.json example would get a broken SPA (only `/sitemap.xml` would work). The Nuxt 4 Firebase preset may auto-generate this, but documenting it avoids confusion about whether it's intentional or an omission.

**Affected sections**:
- Spec 05: INFRA-001 firebase.json rewrite rules

**Suggested answer options**:
- (A) Add the SPA catch-all rewrite to the firebase.json example (e.g., `{"source": "**", "function": "server"}` or equivalent Nuxt preset output)
- (B) Add a note that the Nuxt 4 Firebase preset auto-generates the SPA catch-all rewrite and the firebase.json shown is partial (sitemap-only additions)
- (C) Both A and B — show the complete firebase.json and note it's auto-generated by Nuxt
- (D) Other

---

### V3-GAP-004 — LOW — Specs: 02, 07

**Frontend Slow Loading Threshold Cross-Reference**

**Gap**: The frontend error reporting component (FE-COMP-009, spec 02 line 497) mentions "slow loading" as a tracked event type but doesn't define the threshold. The 3-second threshold is defined only in the observability spec (spec 07 line 317: "API response > 3 seconds").

**Current state**: A frontend implementer reading only spec 02 would not know what constitutes "slow loading" without cross-referencing spec 07.

**Why it matters**: Self-contained sections reduce implementation errors and cross-reference hunting.

**Affected sections**:
- Spec 02: FE-COMP-009 error reporting
- Spec 07: Alerting thresholds

**Suggested answer options**:
- (A) Add a parenthetical cross-reference in spec 02: "slow loading (API response > 3 seconds, see OBS spec)" — keeps spec 07 as source of truth
- (B) Duplicate the threshold value in spec 02 with a note that spec 07 is the authoritative source
- (C) Leave as-is — the observability spec defines thresholds and the frontend just reports the event type
- (D) Other

---

### V3-GAP-005 — LOW — Specs: 02, 04

**Social Links Icon Library Unspecified**

**Gap**: DM-004 defines an `icon` string field (e.g., `github`, `linkedin`) and notes "maps to a frontend icon set (e.g., Material Design Icons, Font Awesome). The frontend handles rendering." But spec 02 (FE-PAGE-006) doesn't specify which icon library or rendering approach to use.

**Current state**: BE-API-006 returns `icon` values. The frontend is expected to render them but no library or approach is specified.

**Why it matters**: Without this, implementers must choose an icon library independently, which could affect bundle size and design consistency.

**Affected sections**:
- Spec 02: FE-PAGE-006 Socials page
- Spec 04: DM-004 `icon` field note

**Suggested answer options**:
- (A) Specify a concrete icon library (e.g., `@mdi/js` for Material Design Icons tree-shakeable SVGs, or simple inline SVGs for just the ~6 social platforms)
- (B) Use inline SVG components — since there are only ~6 social platforms, bundle custom SVGs directly (no library dependency)
- (C) Leave unspecified — implementation detail for the developer to decide based on current best practices
- (D) Other

---

### V3-GAP-006 — LOW — Specs: 06

**Indefinite Ban Manual Review Workflow**

**Gap**: SEC-007 escalation ladder includes an "indefinite" ban tier (2 offenses within 7 days after a 90-day ban expires → indefinite block). No process or mechanism is documented for reviewing or lifting indefinite bans.

**Current state**: The ban is permanent with no escape hatch. There's no admin UI, CLI tool, manual Firestore edit process, or periodic review schedule documented.

**Why it matters**: Without any documented process, indefinite bans are truly permanent. If a legitimate user's IP is affected (e.g., shared IP, CGNAT), there's no recovery path. Even a note saying "manual Firestore document deletion" would suffice.

**Affected sections**:
- Spec 06: SEC-007 Progressive Banning

**Suggested answer options**:
- (A) Document a simple manual process: "To lift an indefinite ban, delete the corresponding document from the `rate_limit_offenders` collection. The LRU cache will expire within [TTL] and the IP will be unblocked."
- (B) Add a Cloud Scheduler job that reviews indefinite bans older than N days
- (C) Leave as truly permanent — the site owner can manually intervene in Firestore if needed (no formal process)
- (D) Other

---

### V3-GAP-007 — LOW — Specs: 05

**Cloud Functions Error Log BigQuery Sink Missing**

**Gap**: INFRA-010d (Backend Error Logs) explicitly filters to `resource.labels.service_name="website-api"`, which excludes Cloud Functions Gen 2 errors. The spec even notes this intentionally (CLR-147). However, there is no dedicated BigQuery log sink for Cloud Functions errors (sitemap generation, rate-limit log processing, offender cleanup, embedding sync).

**Current state**: Cloud Functions errors only appear in INFRA-010a (All Logs) — the unfiltered catch-all. They are queryable but mixed with all other project logs, making trend analysis difficult.

**Why it matters**: Cloud Functions errors (especially in embedding sync and rate-limit processing) could indicate data pipeline failures. A dedicated sink would enable easier monitoring and alerting.

**Affected sections**:
- Spec 05: INFRA-010 BigQuery Log Sinks

**Suggested answer options**:
- (A) Add INFRA-010f: Cloud Functions Error Logs sink — filter: `resource.type="cloud_run_revision" AND resource.labels.service_name!="website-api" AND severity>=ERROR` — table: `website_logs.cloud_functions_error_logs`
- (B) Broaden INFRA-010d to include Cloud Functions by removing the service_name filter (but this mixes backend and CF errors)
- (C) Leave as-is — Cloud Functions errors are low-volume and queryable via INFRA-010a's all_logs table
- (D) Other

---

## Summary

| ID | Priority | Category | Brief Description |
|----|----------|----------|-------------------|
| V3-GAP-001 | MEDIUM | Data Flow | Vector search post-filter exhaustion → empty results behavior |
| V3-GAP-002 | MEDIUM | User Flow | Mobile infinite scroll deep link with `?page=N` |
| V3-GAP-003 | MEDIUM | Infrastructure | Firebase hosting SPA catch-all rewrite missing from firebase.json |
| V3-GAP-004 | LOW | Cross-Reference | Frontend slow loading threshold not in spec 02 |
| V3-GAP-005 | LOW | Frontend | Social links icon library unspecified |
| V3-GAP-006 | LOW | Security/Ops | Indefinite ban manual review workflow |
| V3-GAP-007 | LOW | Infrastructure | Cloud Functions error log BigQuery sink |

**Total: 3 MEDIUM, 4 LOW — no CRITICAL gaps remain.**
