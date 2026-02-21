# Specification Gap Analysis Report

**Date**: 2026-02-20
**Scope**: All 7 specification documents (01–07)
**Purpose**: Identify missing algorithms, data flows, user functionality, and cross-spec inconsistencies that would block or complicate implementation.

---

## Summary

After reviewing all ~4,500+ lines of specifications, the specs are remarkably thorough. Most gaps are implementation-level details rather than fundamental design issues. Below are the findings organized by severity and category.

---

## 🔴 CRITICAL — Blocks correct implementation

### GAP-001: Progressive Ban Evaluation — Who Applies Bans?

**Specs affected**: SEC-002, INFRA-008c, DM-009

**Issue**: There is ambiguity about which component evaluates progressive ban tiers and writes the `current_ban` field to the `rate_limit_offenders` collection (DM-009).

- SEC-002 says: "The Go application checks ban status on each GET and POST request and enforces progressive banning tiers."
- INFRA-008c logging requirements include: "Ban applied (WARNING): Client IP, ban tier, expiry" — implying the Cloud Function applies bans.
- DM-009 has a `current_ban` field with ban tier and expiry.

**Question**: Does the `process-rate-limit-logs` Cloud Function (INFRA-008c) evaluate offense counts against ban tier thresholds and write `current_ban`, or does the Go backend evaluate ban tiers on each request by reading offense counts and determining if a ban should be applied?

**Impact**: If the Cloud Function applies bans, the Go backend only needs to read `current_ban`. If the Go backend evaluates ban tiers, it needs to understand the full progressive banning ruleset and may need to write `current_ban` back to Firestore.

**Recommendation**: Clarify that the Cloud Function both records offenses AND evaluates/applies ban tiers during the atomic `findOneAndUpdate`, so the Go backend is a simple read-only ban checker for `current_ban`. This keeps the Go backend stateless and the ban logic centralized.

**Answer**: The Cloud Function `process-rate-limit-logs` evaluates offense counts against ban tier thresholds and writes the `current_ban` field. The Go backend only reads `current_ban` to enforce bans.

---

### GAP-002: Offense Counting Algorithm — Rolling Window or Cumulative?

**Specs affected**: SEC-002

**Issue**: The progressive banning table says:
- "5 offenses within 7 days" → 30-day ban
- "2 offenses within 7 days after 30-day ban expires" → 90-day ban

But it doesn't specify:
1. Does "within 7 days" mean a rolling 7-day window from the current offense, or 7 days from the first offense in a group?
2. After a 30-day ban expires, does the offense counter reset? Or do the previous 5 offenses still count?
3. What if a client gets 3 offenses, waits 8 days (outside the 7-day window), then gets 3 more? Does the counter use only the most recent 7 days?
4. What constitutes the "after ban expires" period — does the 7-day window start from the ban expiry date?

**Impact**: Different interpretations lead to different ban application timing. This is the core progressive banning algorithm.

**Recommendation**: Specify precisely:
- Use a rolling 7-day window: count only offenses with timestamps within the last 7 days.
- After any ban expires, the "post-ban offense window" starts from `current_ban.end`. Count offenses with timestamps after `current_ban.end` and within 7 days of each other.
- Offenses older than 7 days from the current date (or ban expiry date for escalation checks) are excluded from tier evaluation.

**Answer**:
Follow the recommendation.

---

### GAP-003: Cross-Spec Inconsistency — `/categories` Cache-Control Header

**Specs affected**: BE-API-008, AC-API-031

**Issue**: 
- BE-API-008 specifies: `Cache-Control: public, max-age=3600` (1 hour)
- AC-API-031 specifies: `Cache-Control: public, max-age=86400` (1 day)

These are contradictory values. The frontend caches in localStorage for 24 hours (CLR-199).

**Recommendation**: Align both to `max-age=3600` (1 hour) since the backend changes categories more frequently than once a day and the frontend has its own 24-hour localStorage cache anyway. Update AC-API-031.

**Answer**: Update AC-API-031 to specify `Cache-Control: public, max-age=3600` for the `/categories` endpoint.

---

## 🟡 IMPORTANT — Requires clarification before implementation

### GAP-004: Smart Download Article Selection Algorithm

**Specs affected**: FE-COMP-008

**Issue**: The spec provides concrete parameters (30% scroll depth trigger, 3 articles, 50MB cap, 4g only) but the article selection/ranking algorithm is vague: "articles the user has viewed partially, articles in the same category as recently read articles."

**Missing details**:
- How are the 3 articles chosen? Purely by category match? Most recent in the same category?
- Are tags considered for ranking?
- What if there are fewer than 3 matching articles?
- How is "recently read" defined — the current session? Last 5 articles?
- How does the frontend know which articles exist without an extra API call? It only has the current list page data.

**Recommendation**: Define a concrete ranking algorithm, e.g.:
1. From the currently loaded list page, select articles matching the current article's category that haven't been cached yet.
2. If fewer than 3 matches, fill remaining slots with the most recent uncached articles from the current list.
3. Only prefetch articles whose data is already available in the current list response (no extra API calls).

**Answer**
Follow the recommendation.

---

### GAP-005: Cache API ↔ IndexedDB Coordination for Offline Reading

**Specs affected**: FE-COMP-008

**Issue**: The spec says both Cache API and IndexedDB are used for offline reading, but the coordination between them is underspecified:
- Cache API stores "article content HTTP responses"
- IndexedDB stores "article metadata, reading state, and article content as structured data"
- Both are said to store article content — what's the source of truth?
- When checking if an article is available offline, which store is queried?
- When the service worker intercepts a navigation, does it check Cache API or IndexedDB?

**Recommendation**: Clarify the storage responsibility split:
- **Cache API** (via service worker): Stores full HTTP responses for the `GET /technical/{slug}.md` and `GET /blog/{slug}.md` endpoints. The service worker intercepts network requests and serves from Cache API when offline. This is the primary retrieval mechanism.
- **IndexedDB** (from main thread): Stores article metadata for the offline availability indicator (checking quickly whether an article URL is cached), reading progress state, and last-accessed timestamps. Does NOT duplicate full article content.
- Source of truth for "is this article available offline?": Check if the URL exists in Cache API (via `caches.match()`).

**Answer**
Follow the recommendation.

---

### GAP-006: Service Worker ↔ Main Thread Communication

**Specs affected**: FE-COMP-008

**Issue**: The spec describes service worker update notifications and offline indicators, but doesn't specify the communication mechanism between the service worker and the Vue application.

**Missing details**:
- How does the service worker notify the main thread of a new version? (`postMessage`, `BroadcastChannel`, `navigator.serviceWorker.controller.postMessage`?)
- How does the main thread request the service worker to cache an article for offline reading?
- How does the new service worker activation (from the "Refresh now" button) work? (`skipWaiting()` + `clients.claim()` + page reload?)

**Recommendation**: Specify `postMessage` as the communication channel between the service worker and the main thread, following the standard Workbox patterns. For the "Refresh now" button flow: the main thread sends a `SKIP_WAITING` message to the waiting service worker, which calls `self.skipWaiting()`, and the `controllerchange` event triggers a page reload.

**Answer**
Follow the recommendation.

---

### GAP-007: Sitemap URL Inclusion and Format

**Specs affected**: INFRA-008a, BE-API-011, DM-010

**Issue**: The sitemap generation Cloud Function "reads article collections from Firestore Enterprise, generates a valid sitemap XML" but doesn't specify:
- Which URLs are included (just articles? static pages like `/`, `/socials`, `/privacy`, `/changelog`?)
- `<priority>` and `<changefreq>` values for each URL type
- Whether `<lastmod>` is included (presumably from `date_updated`)
- The exact XML structure

**Recommendation**: Specify:
- Include: all published `technical_articles` (`/technical/{slug}.md`), all published `blog_articles` (`/blog/{slug}.md`), and static pages (`/`, `/technical`, `/blog`, `/socials`, `/others`, `/privacy`, `/changelog`)
- `<lastmod>`: `date_updated` for articles, omit for static pages (or use a fixed date)
- `<changefreq>`: `weekly` for articles, `monthly` for static pages
- `<priority>`: `1.0` for `/`, `0.8` for article detail pages, `0.6` for list pages, `0.4` for static pages
- URL format: `https://tjmonsi.com/technical/{slug}.md` (with `.md` extension, matching the frontend routes)

**Answer**
Follow the recommendation.

---

### GAP-008: Vector Search Filter Application — Database-Level or In-Memory?

**Specs affected**: BE-API-002 (Vector Search Flow Step 5)

**Issue**: Step 5 says "query Firestore Enterprise with the candidate document IDs from step 4 AND the filter criteria applied." CLR-178 says "retrieves full documents from Firestore Enterprise by these IDs, applies any filters." The wording is ambiguous:
- Option A: Single MongoDB query `{ _id: { $in: [...ids] }, category: "DevOps", tags: { $all: ["go"] }, ... }` — database-level filtering
- Option B: Retrieve all documents by ID, then filter in Go code — application-level filtering

**Impact**: Option A is more efficient (less data transferred) but requires careful query construction. Option B is simpler but sends more data over the wire.

**Recommendation**: Clarify that filtering SHOULD happen at the database level (Option A) using a single query with `$in` on IDs and additional filter criteria. Fall back to in-memory filtering only if the MongoDB compatibility layer doesn't support complex compound queries with `$in`.

**Answer** Option A

---

### GAP-009: `process-rate-limit-logs` Atomic Update Logic

**Specs affected**: INFRA-008c, DM-009

**Issue**: The spec says the Cloud Function uses "atomic `findOneAndUpdate`" but doesn't detail:
- The exact MongoDB update operation (upsert? `$inc` on `offense_count`? `$push` on `offense_history`?)
- What the update query looks like
- How the function extracts the IP and endpoint from the Cloud Armor log entry Pub/Sub message
- The Cloud Armor log entry schema that the Cloud Function parses

**Recommendation**: Specify the `findOneAndUpdate` operation:
```javascript
db.rate_limit_offenders.findOneAndUpdate(
  { identifier: clientIP },
  {
    $inc: { offense_count: 1 },
    $push: {
      offense_history: {
        timestamp: new Date(),
        endpoint: extractedEndpoint
      }
    },
    $setOnInsert: {
      identifier: clientIP,
      ban_history: []
    }
  },
  { upsert: true, returnDocument: 'after' }
)
```
Then evaluate the returned document against progressive ban tiers and apply `current_ban` if thresholds are met.

Also specify: the Pub/Sub message schema contains a Cloud Logging log entry. The Cloud Function should extract `httpRequest.remoteIp` and `httpRequest.requestUrl` from the `jsonPayload` or `httpRequest` fields in the log entry.

**Answer**: Follow the recommendation.

---

### GAP-010: Heading Slug Generation — Edge Cases

**Specs affected**: FE-PAGE-003

**Issue**: The heading slug algorithm is described as "lowercase, spaces replaced with hyphens, special characters removed" but doesn't detail:
- What qualifies as "special characters"? (e.g., Unicode characters, parentheses, colons, periods, apostrophes?)
- Are consecutive hyphens collapsed to a single hyphen?
- Are leading/trailing hyphens stripped?
- How are non-ASCII characters handled (e.g., accented characters — transliterated or removed?)

**Recommendation**: Specify the algorithm precisely:
1. Convert to lowercase
2. Replace spaces and underscores with hyphens
3. Remove all characters that are not `[a-z0-9-]`
4. Collapse consecutive hyphens to a single hyphen
5. Strip leading and trailing hyphens
6. If the result is empty, use a fallback like `heading-N` where N is the heading's position

**Answer**: Follow the recommendation.

---

### GAP-011: BibTeX Citation Key Generation

**Specs affected**: FE-PAGE-003, BE-API-003

**Issue**: The BibTeX example shows `@online{monserrat2025articletitle, ...}` but the algorithm for generating the citation key (`monserrat2025articletitle`) is not specified.

**Recommendation**: Specify: `<author-lastname-lowercase><year><title-first-three-words-lowercased-no-spaces>`. For example, "My First Article" published 2025 → `monserrat2025myfirstarticle`. If the title has fewer than 3 words, use all available words. Remove special characters from the key.

**Answer**: Follow the recommendation.

---

### GAP-012: Levenshtein Distance Detail

**Specs affected**: FE-COMP-009

**Issue**: Duplicate search detection uses "Levenshtein distance ≤ 2" for the `q` value comparison. Not specified:
- Standard Levenshtein or Damerau-Levenshtein (which also counts transpositions)?
- Case sensitivity of the comparison (presumably case-insensitive since queries are lowercased for vector search)
- Is the comparison on the full `q` value or just the trimmed/normalized version?

**Recommendation**: Specify standard Levenshtein distance on the lowercased, trimmed query string. This is consistent with the vector search flow which lowercases the query (step 1).

**Answer**: Follow the recommendation.

---

### GAP-013: Browser Name/Version Extraction Method

**Specs affected**: FE-COMP-004

**Issue**: The `browser` field is described as "Browser name and version (from User-Agent, determined client-side)." The spec doesn't specify:
- The extraction method (regex? `navigator.userAgentData`? a library?)
- The expected format (e.g., `"Chrome 120"` vs `"Chrome/120.0.6099.224"`)

**Recommendation**: Specify using the `navigator.userAgentData` API (User-Agent Client Hints) when available, falling back to User-Agent string parsing. Format: `"<Browser> <Major Version>"` (e.g., `"Chrome 120"`, `"Firefox 115"`, `"Safari 17"`). Use a lightweight UA parsing library or simple regex.

**Answer**: Follow the recommendation.

---

### GAP-014: Reading Progress Storage and Display

**Specs affected**: FE-COMP-008

**Issue**: IndexedDB is specified to store "reading state (e.g., reading progress, last accessed timestamp)" but:
- How is reading progress tracked? Scroll position percentage?
- Is reading progress displayed anywhere in the UI? (e.g., progress bar, percentage badge)
- When/how is reading progress saved (on every scroll? on page leave? throttled?)

**Recommendation**: Either specify reading progress tracking (scroll position saved on `beforeunload` and throttled during scroll, displayed as a small percentage badge on list items) or remove the "reading progress" mention if it's not planned for v1.

**Answer**: Add reading progress tracking: save scroll position as a percentage on `beforeunload` and during scroll (throttled to once every 5 seconds). Display the progress as a small badge (e.g., "45% read") on the article's list item in the category page.

---

### GAP-015: Content Helpfulness Score Algorithm

**Specs affected**: OBS-006

**Issue**: The Analytics Dashboard lists "Content helpfulness score (derived from engagement milestones and link click depth)" but no algorithm is defined.

**Recommendation**: Since this is a Looker Studio calculated metric (not application code), define the formula. Example:
- Score = (count of visitors reaching 2min milestone / total page views) × 0.5 + (count of link clicks / total page views) × 0.3 + (count of visitors reaching 5min / total page views) × 0.2
- Or defer this to a future iteration and note it's a "to be defined" Looker Studio metric.

**Answer**: Follow recommendation but put it in a manual to allow me to edit Looker Studio. Still update the spec to say "Content helpfulness score (Looker Studio calculated metric based on engagement milestones and link click depth)."

---

### GAP-016: Bot vs Human Discrimination Algorithm

**Specs affected**: OBS-006

**Issue**: The Analytics Dashboard lists "Bot vs. human traffic discrimination (visitors with no `time_on_page` events, abnormal page view velocity)" but no threshold or algorithm is specified.

**Recommendation**: Define the heuristics for the Looker Studio query:
- **Bot indicator**: `visitor_id` with ≥ N page views in T minutes AND zero `time_on_page` events
- Suggest starting thresholds (e.g., > 10 page views in 1 minute with no time_on_page events)
- Or defer this to a future iteration.

**Answer**: Follow recommendation but put it in a manual to allow me to edit Looker Studio. Still update the spec to say "Bot vs. human traffic discrimination (Looker Studio calculated metric based on visitor page view velocity and engagement events)."

---

## 🟢 SUGGESTION — Non-blocking improvements

### GAP-017: Dark Mode — Mentioned but Not Specified

**Specs affected**: FE-PAGE-010

**Issue**: The changelog TL;DR example says: "Latest: Dark mode support, faster search, and a new socials page layout." But dark mode is never specified in the frontend specs. This is likely just an example, but could confuse implementers.

**Recommendation**: Change the example to reference real features (e.g., "Latest: Offline reading, improved error reporting, and funnier 404 pages.") or add a note that the example is illustrative.

**Answer**: Update the changelog example to "Latest: Offline reading, improved error reporting, and funnier 404 pages." to avoid confusion about dark mode. Also have the specs have a simple design put for both light and dark mode. We will use a minimalistic design and using Google Sans as font. The color scheme will be light with a white background and black text, with blue accents for links and buttons. For dark mode, we can use a dark gray background with white text and the same blue accents. The design should be clean and simple, focusing on readability and ease of navigation. We can also use subtle shadows and borders to create visual separation between elements without adding too much visual noise.

---

### GAP-018: GeoIP Database Update Schedule

**Specs affected**: INFRA-003

**Issue**: The spec says the Docker image is "rebuilt weekly" to pick up GeoIP database updates, but doesn't specify:
- Which day/time the weekly rebuild runs
- How the GeoIP database is obtained during Docker build (MaxMind license key? download script?)
- Whether the rebuild is a separate Cloud Scheduler job or part of the CI/CD pipeline

**Recommendation**: Specify that the weekly rebuild is a scheduled CI/CD job (e.g., GitHub Actions cron `0 0 * * 0` — Sunday midnight UTC) that rebuilds and deploys the Docker image. The MaxMind GeoLite2-Country database is downloaded during the Docker multi-stage build using a license key stored as a GitHub Actions secret.

**Answer**: Follow the recommendation.

---

### GAP-019: Firestore Enterprise Connection String

**Specs affected**: SEC-013, INFRA-003, INFRA-006

**Issue**: The Go backend connects to Firestore Enterprise via the MongoDB Go driver. The connection string format for Firestore Enterprise MongoDB compatibility mode is referenced (`https://docs.cloud.google.com/firestore/mongodb-compatibility/docs/connect#cloud-run`) but not detailed in the spec.

**Recommendation**: Add the expected connection string format to INFRA-006 or the backend spec:
```
mongodb+srv://cloud-run-api@<project-id>.iam@<database-id>.<region>.firestore.goog/?authMechanism=MONGODB-OIDC&readPreference=primary
```
(Or the exact format per the referenced documentation.)

**Answer**: Follow the recommendation.

---

### GAP-020: Mobile Table of Contents Behavior

**Specs affected**: FE-PAGE-003, Responsive Design section

**Issue**: "Table of contents SHALL collapse to a hamburger/dropdown on mobile" — but no detail on the specific UI pattern (floating action button? top-of-page dropdown? slide-out drawer?).

**Recommendation**: Specify the mobile TOC as a floating bottom-right button (similar to many documentation sites) that opens a slide-up sheet or dropdown showing the heading list. Scroll spy continues to work within the sheet.

**Answer**  Follow the recommendation.

---

### GAP-021: Offline Status Real-Time Updates

**Specs affected**: FE-COMP-008, FE-COMP-013

**Issue**: The offline availability indicator on list items needs to know which articles are cached. The spec doesn't detail:
- How the list view checks Cache API availability (asynchronous per-item check?)
- How the indicator updates if a smart download completes while viewing the list
- Performance impact of checking cache for 10 items per page

**Recommendation**: Specify that on list page load, the frontend checks `caches.match()` for each article URL in the visible list and displays the indicator based on the result. Use a Promise.all for parallel checks. Smart download completions trigger a re-check of visible items via a custom event.

**Answer**: Follow the recommendation.

---

### GAP-022: `q` Parameter with Only Whitespace

**Specs affected**: BE-API-002

**Issue**: CLR-189 specifies "IF `q` is present but empty or whitespace-only, THE SYSTEM SHALL treat it as absent." This is clear for the backend. But the frontend spec doesn't explicitly state whether the search bar should strip whitespace before adding `q` to the URL.

**Recommendation**: Add to FE-COMP-011: "WHEN the search bar value is empty or contains only whitespace, THE SYSTEM SHALL remove the `q` parameter from the URL (treat as no search)."

**Answer**: Follow the recommendation.

---

### GAP-023: `POST /t` — `link_click` and `time_on_page` Actions Missing from Log Type Mapping

**Specs affected**: OBS-001, BE-API-009

**Issue**: OBS-001 says `log_type: "frontend_tracking"` applies to `action: "page_view"`, `"link_click"`, and `"time_on_page"`. BE-API-009 explicitly documents the behavior for all four actions. However, the OBS-001 note says: `"frontend_tracking" — for POST /t requests with action: "page_view", "link_click", or "time_on_page"`. This is consistent but should be verified that `link_click` and `time_on_page` log entries include all their action-specific fields (`clicked_url`, `milestone`) in the structured log.

**Status**: Actually well-specified in BE-API-009. Not a real gap — just worth verifying during implementation.

**Answer**: No change needed. Just verify that the frontend sends the correct fields for `link_click` and `time_on_page` actions, and that the backend logs them in structured logs.

---

### GAP-024: `others` Collection — Slug Pattern for External Content

**Specs affected**: DM-005

**Issue**: The `others` collection uses the same slug format as articles (`^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}$`). But for external content, the slug format (which includes a date) might not always make sense. The spec is clear though — just flagging that the content CI/CD pipeline needs to generate slugs in this format for external content.

**Status**: Not a gap per se — just an implementation note.

**Answer**: No change needed. Just ensure that the content management pipeline generates slugs for `others` collection documents in the specified format, even for external content.

---

### GAP-025: Missing Specification — `GET /` Frontend Page Content Source

**Specs affected**: DM-001, BE-API-001, FE-PAGE-001

**Issue**: DM-001 defines a `frontpage` collection with a single document containing markdown `content`. BE-API-001 serves this. FE-PAGE-001 renders it. The content management flow for how the frontpage content gets into Firestore is out of scope (content CI/CD pipeline), but the spec should note that the `frontpage` collection is populated by the content pipeline, consistent with other collections.

**Status**: Minor documentation note, not a blocking gap.

**Answer**: Add a note to DM-001: "The `frontpage` collection is populated by the content management CI/CD pipeline, similar to the `technical_articles` and `blog_articles` collections. The content pipeline is responsible for updating the `content` field of the single document in this collection to reflect changes to the homepage content."

---

## Cross-Spec Inconsistency Summary

| # | Issue | Location | Recommendation |
|---|-------|----------|----------------|
| 1 | `/categories` Cache-Control: 3600 vs 86400 | BE-API-008 vs AC-API-031 | Align to 3600 in AC-API-031 |
| 2 | Ban applier ambiguity | SEC-002 vs INFRA-008c | Clarify Cloud Function applies bans |

---

## Not Missing — Explicitly Out of Scope

The following areas are intentionally not covered and should NOT be added to these specs:

1. **Content management CI/CD pipeline** — Separate repository, out of scope (AD-022)
2. **Firestore Enterprise provisioning details** — May be manual if Terraform doesn't support MongoDB compat mode (INFRA-016 note)
3. **Nuxt 4 project configuration** — Implementation detail
4. **Go project structure** — Implementation detail
5. **Cloud Function code organization** — Implementation detail
6. **Looker Studio dashboard layout** — Owner-operated, design-time decision
7. **Firebase Hosting custom domain setup** — Operational task

---

## Priority Matrix

| Priority | Count | Action |
|----------|-------|--------|
| 🔴 CRITICAL | 3 | Must resolve before implementation starts |
| 🟡 IMPORTANT | 13 | Should resolve during design phase |
| 🟢 SUGGESTION | 9 | Can resolve during implementation |

## Additional notes:
Let's put these in scope
- Nuxt 4 project configuration (e.g., `nuxt.config.ts` settings for modules, build, runtime config)
- Go project structure (e.g., package organization, main.go setup)
- Cloud Function code organization (e.g., file structure, function exports)
- Looker Studio dashboard layout (e.g., how metrics are organized, filters, visualizations) as a manual
- Firebase Hosting custom domain setup (e.g., DNS records, SSL certs) as a manual