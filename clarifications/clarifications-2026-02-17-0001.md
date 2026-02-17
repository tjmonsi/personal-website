## Clarifications Needed

The following items require your input to finalize the specifications. Each clarification is referenced by its ID (CLR-XXX) throughout the spec documents.

---

### CLR-001: Database Technology Choice

**Context**: The initial plan states *"Firestore Enterprise Database using MongoDB ORM."* These are contradictory technologies:

- **Firestore** is a Google-native document database with its own SDK (no MongoDB ORM support).
- **"Firestore Enterprise"** is not a recognized Google Cloud product name.
- **MongoDB ORM** implies using MongoDB (possibly MongoDB Atlas on GCP).

**Question**: Which database do you want to use?

- **Option A**: MongoDB Atlas on Google Cloud — supports MongoDB Go driver and ORMs, full MongoDB query language, runs on GCP.
- **Option B**: Firestore (Native mode) — Google-native, integrates tightly with Firebase, uses Firestore Go SDK (not MongoDB ORM).
- **Option C**: Firestore in Datastore mode — legacy mode, not recommended for new projects.
- **Option D**: Something else? (please specify)

**Impact**: Affects the entire data layer, Go driver/ORM choice, query capabilities (especially text search), and infrastructure setup.

**Answer**
We will be using the Firestore Enterprise: https://firebase.google.com/docs/firestore/enterprise/overview-enterprise-edition-modes
and we will use the MongoDB compatibility mode: https://firebase.google.com/docs/firestore/enterprise/mongodb-compatibility-overview 
---

### CLR-002: GET-Only Constraint vs. Tracking & Error Reporting

**Context**: The plan states *"Only GET method is allowed on all endpoints"* but also requires:

- Sending visitor tracking data (browser, IP, action, referrer) to the backend
- Sending frontend error data (error type, connection speed, etc.) to the backend

Sending structured data via GET query parameters is possible but has limitations (URL length limits, data visible in server logs/browser history, less structured).

**Question**: How should tracking and error data be sent to the backend?

- **Option A**: Use GET with query parameters (e.g., `GET /track?page=/technical&action=view&ref=google.com`) — stays within GET-only rule but limited payload size.
- **Option B**: Allow POST for `/track` and `/report-error` endpoints only — better data structure, but breaks the GET-only rule.
- **Option C**: Use a third-party analytics service (e.g., Google Analytics, Plausible) and skip custom tracking entirely.
- **Option D**: Use a beacon/pixel approach (e.g., `GET /pixel.gif?data=...`) — common in analytics, stays GET-only.

**Answer**
You are correct, I forgot about the API endpoint to send these browser history logs. This is the only POST method url. Use the /t as the url. Make sure that /t is protected as of the moment that all call logs came from the browser using the website. Also, let's have a static id and static "secret" on the frontend and use obfuscating methods to create a JWT that makes the frontend be the only one allowed to call /t using the Bearer Token protocol.
---

### CLR-003: Article Pagination Meaning

**Context**: The plan says articles are *"paginated as well and show what page you are and left and right buttons."* This could mean:

- **Interpretation A**: A single long article is split across multiple pages (e.g., page 1 of 3 of one article).
- **Interpretation B**: Navigation between articles in a series (previous/next article).

**Question**: Does article pagination mean splitting a long article into multiple pages, or navigating between different articles?

If splitting a single article: What determines a page break? Character count? A manual marker in the markdown? Heading-based?

**Answer** 
This should be for the list of articles (title and abstracts only) that are listed on a scroll and at the bottom is a pagination. For mobile readers, there should not be a pagination but more of a endless scrollable list
---

### CLR-004: Firebase Functions Purpose

**Context**: The plan mentions Firebase Functions but also says *"no server involved on Nuxt4"* (static/SPA mode).

**Question**: What is the intended use of Firebase Functions? Options:

- **Option A**: Not needed — remove from the stack (pure static hosting + Cloud Run API).
- **Option B**: Used for specific frontend server tasks (e.g., dynamic `robots.txt`, sitemap generation, SSR for SEO).
- **Option C**: Used as middleware between frontend and backend.

**Recommendation**: Option A (remove) keeps the stack simpler. `robots.txt` and sitemap can be static files or served by the Go API.

**Answer** 
If I remember it right, if Nuxt4 is set to SPA only and deployed on Firebase Hosting, it still needs the Firebase Fucntions to serve the SPA while the hosting would serve the JS files. You can check it here: https://nuxt.com/deploy/firebase 

---

### CLR-005: Article Slug Format

**Context**: The plan describes slugs as *"title-slug-with-creation-date-and-time (or filename in md)."*

**Question**: Please confirm the exact slug format. Proposed:

```
{title-in-kebab-case}-{YYYY}-{MM}-{DD}T{HH}-{mm}-{SS}
```

Example: `my-first-article-2025-01-15T10-30-00`

Full URL: `GET /technical/my-first-article-2025-01-15T10-30-00.md`

Is this correct? Or do you prefer a different format (e.g., without time, or with a different separator)?

**Answer**
Example: my-first-article-2025-01-15-1030
Full URL: GET /technical/my-first-article-2025-01-15-1030.md
---

### CLR-006: Categories — Predefined or Free-form?

**Context**: The plan mentions filtering by categories but does not specify whether categories are from a fixed list or free-form.

**Question**: Are categories predefined (fixed list) or free-form (author can create any category)?

- **Option A**: Predefined list — provide the list of valid categories.
- **Option B**: Free-form — any string is valid as a category.

If predefined, what are the initial categories? (e.g., DevOps, Programming, Cloud, AI/ML, etc.)

**Answer**
Categories are free-form, and based on the categories set in all of the articles. Let's have a table in MongoDB though that holds all categories and will be called every time the frontend goes to /technical or /blog (for frontend that is the link for opinions) or /others. Then let's have a /categories as to get the list. The table for categories should have the category name and date created. The frontend will not call the categories API again because it will save in the session storage of the frontend, and will just call again after 24 hours have passed.

---

### CLR-007: Rate Limit Numbers

**Context**: The plan describes progressive rate limiting but does not specify the actual request limits.

**Question**: What rate limits should be applied?

Proposed defaults (per IP, per minute):

| Caller Type    | Limit         | Rationale                                         |
| -------------- | ------------- | ------------------------------------------------- |
| Regular user   | 60 req/min    | Generous enough for shared NAT / ISP IPs          |
| Known bots     | 10 req/min    | Reasonable crawl rate                             |
| Tracking/error | 30 req/min    | Separate bucket to avoid consuming user quota     |

Are these acceptable, or do you have different numbers in mind?

**Answer**
This will do, but put a reference on the rationale where these came from (like a link I can read)
---

### CLR-008: Ban Response Status Code

**Context**: When a banned client makes a request, what HTTP status code should be returned?

- **Option A**: `403 Forbidden` — semantically correct for "you are blocked."
- **Option B**: `429 Too Many Requests` — maintains the plan's principle of only returning 400/404/429/503.
- **Option C**: `404 Not Found` — maximum opacity, attacker doesn't know they're banned.

**Recommendation**: Option A (`403`) is clearest, but it adds a new status code outside the plan's stated set.

**Answer**
Use 429 for the initial 5 offenses, then 403 for blocked 30 days, then 404 for blocked for 90 days and indefinitely. 

---

### CLR-009: robots.txt Specific Rules

**Context**: The plan mentions a `robots.txt` but does not specify which pages/paths to allow or block for crawlers.

**Question**: Are there any pages or paths that should be blocked from crawling? Proposed:

```
User-agent: *
Allow: /
Crawl-delay: 10
```

Should any paths be disallowed (e.g., tracking endpoints, API-only paths)?

**Answer**
API-only paths and tracking endpoints should be blocked

---

### CLR-010: Deployment Region and Domain

**Context**: No region or domain is specified in the plan.

**Questions**:

1. What GCP region should the infrastructure be deployed to? (e.g., `us-central1`, `asia-southeast1`, `europe-west1`)
2. What is the domain name? (e.g., `tjmonserrat.com`)
3. Should the API be served from a path (`tjmonserrat.com/api/*`) or a subdomain (`api.tjmonserrat.com`)?

**Answer**
- GCP region of the Firestore should be asia-southeast1
- GCP region of the Cloud Run should be asia-southeast1
- API should be served through a subdomain

---

### CLR-011: Content Management — How Do Articles Get Into the Database?

**Context**: The plan describes the read-only public API but does not mention how content (articles, social links, etc.) is created, edited, or published.

**Question**: How will content be managed?

- **Option A**: Direct database management (manual inserts via MongoDB shell/Compass or Firestore console).
- **Option B**: A private admin API (separate Cloud Run service or endpoint with authentication).
- **Option C**: Git-based CMS (markdown files in a repo, CI/CD pipeline pushes to database on merge).
- **Option D**: A third-party CMS (e.g., Strapi, Contentful) that syncs to the database.

This is a significant architectural decision that affects future workflow.

**Answer**
Content will be managed through a different project repository which closely relates to Option C.
---

### CLR-012: Citation Format

**Context**: The plan mentions *"a citation text if they want to cite it"* for technical blog articles. Citation text should be clickable.

**Question**: What citation format should be used?

- **Option A**: APA style — `Monserrat, T.J. (2025). Article Title. tjmonserrat.com. https://tjmonserrat.com/technical/slug.md`
- **Option B**: Custom format — `Monserrat, TJ. "Article Title." tjmonserrat.com, 15 Jan 2025.`
- **Option C**: BibTeX format for academic use.
- **Option D**: Multiple formats (user selects).

And what does "clickable citation" mean — clicking copies to clipboard? Opens a citation popup?

**Answer**
- Multiple formats (user selects)

---

### CLR-013: 404 Page Anecdotes — Source

**Context**: The plan says the 404 page should show *"a funny randomized anecdote about things not being found."*

**Question**: Do you have a set of anecdotes you'd like to use, or should I generate a list? How many should there be? (Recommended: 10-20 for good variety without repetition.)

**Answers**
Generate a list for now - use 30 anecdotes, preferably with references and links to where it came from

---

### CLR-014: Social Platforms List

**Context**: The plan mentions social media links but does not specify which platforms.

**Question**: Which social media platforms and links should be included? Common options:

- GitHub
- LinkedIn
- Twitter/X
- YouTube
- Mastodon
- Email
- Others?

**Answer**
The social media links will be coming from a database list. Provide for me a draft of the structure of an social item in the table and I will review.

---

### CLR-015: Date Range Filter Semantics

**Context**: The plan mentions filtering by *"date range for updated"* on list pages, but the API spec uses `date_created` for the date range.

**Question**: Should the date range filter apply to:

- **Option A**: `date_created` (when the article was first published)
- **Option B**: `date_updated` (when the article was last modified)
- **Option C**: Both (two separate date range filters)

**Answer**
Use date_updated

---

### Summary

| ID      | Topic                              | Blocking? | Impact Area             |
| ------- | ---------------------------------- | --------- | ----------------------- |
| CLR-001 | Database technology                | **Yes**   | Entire backend + data   |
| CLR-002 | GET-only vs tracking data          | **Yes**   | API design, tracking    |
| CLR-003 | Article pagination meaning         | **Yes**   | Frontend + backend      |
| CLR-004 | Firebase Functions purpose         | **Yes**   | Infrastructure          |
| CLR-005 | Slug format                        | Moderate  | API, data model         |
| CLR-006 | Categories predefined/free-form    | Moderate  | API validation          |
| CLR-007 | Rate limit numbers                 | Moderate  | Security                |
| CLR-008 | Ban response status code           | Low       | Security                |
| CLR-009 | robots.txt rules                   | Low       | SEO                     |
| CLR-010 | Region and domain                  | Moderate  | Infrastructure          |
| CLR-011 | Content management                 | **Yes**   | Architecture, workflow  |
| CLR-012 | Citation format                    | Low       | Frontend, data model    |
| CLR-013 | 404 anecdotes source               | Low       | Frontend                |
| CLR-014 | Social platforms list              | Low       | Data                    |
| CLR-015 | Date range filter field            | Moderate  | API, frontend           |


### Additional Notes from me:
- The frontend should be an SPA but not installable. It should allow the reader to read it offline once they have downloaded the page data on to the mobile. For downloading, do a "smart" download where the tracking is in frontend only and based on heuristics download the page that they will likely to read. But also take note that if they want to read the file, they can click on the article to save for read when offline or in the list view. The list view should also show if the article is good for offline reading.