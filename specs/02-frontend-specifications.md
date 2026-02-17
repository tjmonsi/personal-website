---
title: Frontend Specifications
version: 2.0
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [frontend, nuxt4, vue3, spa]
---

## Frontend Specifications

### Technology

- **Framework**: Nuxt 4 (SPA mode, no SSR)
- **UI Framework**: Vue 3 (Composition API)
- **Build Tool**: Vite
- **Hosting**: Firebase Hosting (static assets) + Firebase Functions (SPA serving)
- **Rendering Mode**: Client-side only
- **Deployment Reference**: https://nuxt.com/deploy/firebase

### Definitions

| Term             | Definition |
| ---------------- | ---------- |
| SPA              | Single Page Application — all rendering happens client-side |
| Infinite Scroll  | A pattern where content loads automatically as the user scrolls down, without explicit pagination controls |
| Smart Download   | Heuristic-based prefetching of article content for offline reading based on user browsing patterns |
| sessionStorage   | Browser storage that persists for the duration of a page session |
| URL State Synchronization | The practice of reflecting UI state (search queries, filters, pagination) in the browser URL query parameters, enabling deep linking and shareability |
| Hash Fragment    | The portion of a URL after the `#` symbol (e.g., `#section-title`), used to identify and link to a specific section within a page |
| Deep Link        | A URL that encodes enough state to restore a specific view, including active search terms, applied filters, and current page |
| Heading Slug     | A URL-safe string derived from heading text by converting to lowercase, replacing spaces with hyphens, and removing special characters (e.g., "My Section Title" → `my-section-title`) |
| `visitor_session_id` | A random UUID v4 generated per browser session, stored in `sessionStorage`. Used to correlate tracking events within a single visit. Not linked to any real-world identity and not persisted across sessions. |
| `visitor_id`     | A SHA-256 hash computed server-side from `visitor_session_id` + truncated IP + User-Agent. A non-reversible, session-scoped identifier for unique visitor counting and bot discrimination. |
| Link Click Tracking | Fire-and-forget anonymous tracking of anchor element clicks, sent via `navigator.sendBeacon()` (with `fetch()` + `keepalive: true` fallback) to `POST /t`. |
| Time-on-Page Milestone | An engagement signal sent when a user remains on a page for 1, 2, or 5 minutes of active foreground time. Only foreground time counts (uses Page Visibility API). |

---

### Pages

#### FE-PAGE-001: Front Page (`/`)

**Description**: Landing page introducing TJ Monserrat with navigation links.

**Requirements**:

- WHEN the user visits the root URL, THE SYSTEM SHALL display the front page content loaded from the backend API (`GET /`).
- THE SYSTEM SHALL render the markdown content returned by the API.
- THE SYSTEM SHALL display navigation links to:
  - Technical Blog (`/technical`)
  - Opinions (`/blog`)
  - Other Links (`/socials`)
  - Notable Notes Outside (`/others`)

---

#### FE-PAGE-002: Technical Blog List (`/technical`)

**Description**: Paginated list of technical articles with search and filtering. Desktop uses traditional pagination; mobile uses infinite scroll.

**Requirements**:

- WHEN the user visits `/technical`, THE SYSTEM SHALL display a list of technical blog articles fetched from `GET /technical`.
- Each article item SHALL display:
  - Title (clickable, links to the article page)
  - Abstract with a "Read more" button (links to the article page)
  - Offline availability indicator (shows whether the article has been saved for offline reading)
- THE SYSTEM SHALL display a search bar at the top of the page.
- THE SYSTEM SHALL display filter controls for:
  - Category (single-select dropdown, populated from `GET /categories` with 24-hour sessionStorage cache)
  - Tags (multi-select)
  - Tag match mode (toggle or dropdown: "Match All" or "Match Any", corresponding to the `tag_match` API parameter with values `all` or `any`. Default: "Match Any")
  - Date range for "last updated" (date picker range, filters on `date_updated`)
- **Results are always sorted by `date_updated` descending** (most recently updated first). No sort controls are displayed to the user.
- **Desktop pagination**:
  - THE SYSTEM SHALL paginate results showing the current page number and total pages.
  - THE SYSTEM SHALL display navigation buttons:
    - "Previous" button: hidden on the first page
    - "Next" button: hidden on the last page
- **Mobile pagination**:
  - THE SYSTEM SHALL use infinite scroll (endless scrollable list) instead of traditional pagination.
  - WHEN the user scrolls to the bottom of the list, THE SYSTEM SHALL automatically fetch the next page of results.
  - THE SYSTEM SHALL display a loading indicator while fetching additional results.
  - THE SYSTEM SHALL stop fetching when all results have been loaded.
- THE SYSTEM SHALL allow the user to tap/click a "Save for offline" action on each article in the list view.
- **URL State Synchronization** (see FE-COMP-011):
  - THE SYSTEM SHALL reflect the current search and filter state in the browser URL query parameters.
  - URL parameter names SHALL match the backend API parameter names: `q`, `category`, `tags`, `tag_match`, `date_from`, `date_to`, `page`.
  - WHEN the user changes any search, filter, or pagination value, THE SYSTEM SHALL update the URL using `history.pushState()`.
  - WHEN the user loads the page with URL query parameters present, THE SYSTEM SHALL initialize the search bar, filter controls, and pagination from those parameters (deep linking).
  - WHEN the user navigates back or forward in the browser, THE SYSTEM SHALL restore the UI state from the URL parameters via the `popstate` event.
  - Default values (empty search, no category, no tags, `tag_match=any`, no date range, `page=1`) SHALL NOT appear in the URL. Only non-default values are included.
- **Copy Article Link**:
  - Each article item in the list SHALL display a link/share icon button.
  - WHEN the user clicks the copy link button, THE SYSTEM SHALL copy the full article URL (e.g., `https://tjmonsi.com/technical/slug.md`) to the clipboard.
  - THE SYSTEM SHALL display a brief confirmation message (e.g., "Link copied to clipboard") after copying.

---

#### FE-PAGE-003: Technical Blog Article (`/technical/:slug.md`)

**Description**: Full article view with table of contents, metadata, and citation format selector. The URL includes the `.md` extension, presenting the page as if accessing a markdown file directly.

**Requirements**:

- WHEN the user visits `/technical/:slug.md`, THE SYSTEM SHALL fetch and display the article from `GET /technical/{slug}.md`.
- THE SYSTEM SHALL parse the YAML front matter from the `text/markdown` response to extract metadata (title, author, category, tags, changelog, citations) and render the markdown body separately.
- THE SYSTEM SHALL display article metadata at the top:
  - Title
  - Author ("Written by TJ Monserrat")
  - Date of creation
  - Date of last update
  - Tags (list)
  - Category
- THE SYSTEM SHALL display a collapsible changelog table below the metadata.
  - Default state: collapsed
  - WHEN expanded, shows version history entries with date and description
- THE SYSTEM SHALL render the markdown article body.
- **Image Constraints** (CLR-104):
  - Articles SHALL NOT contain external image references except for images hosted on `tjmonsi.com` or `api.tjmonsi.com`.
  - Images in articles are limited to: inline `data:` URIs, images hosted on the frontend domain (`tjmonsi.com`), or images served via the backend image proxy endpoint (`https://api.tjmonsi.com/images/{path}`, see BE-API-013).
  - In markdown, article images SHALL use the format: `![alt text](https://api.tjmonsi.com/images/path/to/image.png)`.
  - The frontend CSP `img-src` directive restricts image loading to `'self' data: https://api.tjmonsi.com` (SEC-005).
- **Heading Anchors**:
  - THE SYSTEM SHALL generate an `id` attribute for each rendered heading (H2, H3, H4) using a heading slug derived from the heading text (lowercase, spaces replaced with hyphens, special characters removed).
  - IF duplicate heading slugs exist within the same article, THE SYSTEM SHALL append a numeric suffix (e.g., `my-heading`, `my-heading-1`, `my-heading-2`).
  - Each heading SHALL display a link icon button (e.g., a chain-link icon) on hover or focus.
  - WHEN the user clicks the heading link icon, THE SYSTEM SHALL copy the full URL with the hash fragment (e.g., `https://tjmonsi.com/technical/slug.md#heading-slug`) to the clipboard.
  - THE SYSTEM SHALL display a brief confirmation message (e.g., "Link copied to clipboard") after copying.
- **Table of Contents (Right Side Navigation)**:
  - THE SYSTEM SHALL display a table of contents on the right side, generated from article headings (H2, H3, H4).
  - Each entry SHALL be a clickable anchor link that scrolls to the corresponding heading.
  - WHEN the user clicks a table of contents entry, THE SYSTEM SHALL update the URL hash fragment to reflect the target heading (e.g., `#heading-slug`) using `history.pushState()`.
  - WHEN the page loads with a hash fragment in the URL, THE SYSTEM SHALL scroll to the corresponding heading.
  - The table of contents SHALL remain in a sticky/fixed position while scrolling.
  - THE SYSTEM SHALL visually highlight the currently active heading in the table of contents as the user scrolls through the article (scroll spy).
- THE SYSTEM SHALL display a citation section with a format selector:
  - The user SHALL be able to select from multiple citation formats:
    - **APA**: `Monserrat, T.J. (2025). Article Title. tjmonsi.com. https://tjmonsi.com/technical/slug.md`
    - **MLA**: `Monserrat, TJ. "Article Title." tjmonsi.com, 15 Jan. 2025, https://tjmonsi.com/technical/slug.md.`
    - **Chicago**: `Monserrat, TJ. "Article Title." tjmonsi.com. January 15, 2025. https://tjmonsi.com/technical/slug.md.`
    - **BibTeX**: BibTeX entry for academic use
    - **IEEE**: `[N] T.J. Monserrat, "Article Title," tjmonsi.com, Jan. 15, 2025. [Online]. Available: https://tjmonsi.com/technical/slug.md`
  - WHEN the user clicks the citation text, THE SYSTEM SHALL copy the selected citation format to the clipboard.
  - THE SYSTEM SHALL display a confirmation message (e.g., "Citation copied to clipboard") after copying.
- THE SYSTEM SHALL provide a "Save for offline reading" button.

---

#### FE-PAGE-004: Opinions Blog List (`/blog`)

**Description**: Same structure and behavior as FE-PAGE-002 but for opinion articles. Desktop uses traditional pagination; mobile uses infinite scroll.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-002 (Technical Blog List), including desktop pagination, mobile infinite scroll, URL state synchronization, and copy article link.
- THE SYSTEM SHALL fetch data from `GET /blog` instead of `GET /technical`.
- THE SYSTEM SHALL fetch categories from `GET /categories` (same cache as `/technical`).
- Article link URLs SHALL use the `/blog/` path prefix (e.g., `https://tjmonsi.com/blog/slug.md`).

---

#### FE-PAGE-005: Opinion Blog Article (`/blog/:slug.md`)

**Description**: Same structure and behavior as FE-PAGE-003 but for opinion articles. The URL includes the `.md` extension.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-003 (Technical Blog Article), including citation format selector, YAML front matter parsing, heading anchors with copy link, and table of contents with URL hash fragment synchronization.
- THE SYSTEM SHALL fetch data from `GET /blog/{slug}.md` instead of `GET /technical/{slug}.md`.

---

#### FE-PAGE-006: Other Links / Socials (`/socials`)

**Description**: List of social media and contact links.

**Requirements**:

- WHEN the user visits `/socials`, THE SYSTEM SHALL display a list of social media links fetched from `GET /socials`.
- Each item SHALL display the platform name and a clickable link.
- Links SHALL open in a new tab (`target="_blank"` with `rel="noopener noreferrer"`).

---

#### FE-PAGE-007: Notable Notes Outside (`/others`)

**Description**: Curated list of external content with search and filtering.

**Requirements**:

- WHEN the user visits `/others`, THE SYSTEM SHALL display a list of external content fetched from `GET /others`.
- Each item SHALL display:
  - Title (clickable, links to the external URL)
  - Date
  - Abstract
- THE SYSTEM SHALL display a search bar at the top.
- THE SYSTEM SHALL display filter controls for:
  - Category (single-select dropdown, populated from `GET /categories` with 24-hour sessionStorage cache)
  - Tags (multi-select)
  - Tag match mode (toggle or dropdown: "Match All" or "Match Any", corresponding to the `tag_match` API parameter. Default: "Match Any")
  - Date range (filters on `date_updated`)
- **Results are always sorted by `date_updated` descending** (most recently updated first). No sort controls are displayed to the user.
- THE SYSTEM SHALL paginate results (same behavior as FE-PAGE-002: desktop pagination, mobile infinite scroll).
- **URL State Synchronization** (see FE-COMP-011):
  - THE SYSTEM SHALL reflect the current search and filter state in the browser URL query parameters.
  - URL parameter names SHALL match the backend API parameter names: `q`, `category`, `tags`, `tag_match`, `date_from`, `date_to`, `page`.
  - WHEN the user changes any search, filter, or pagination value, THE SYSTEM SHALL update the URL using `history.pushState()`.
  - WHEN the user loads the page with URL query parameters present, THE SYSTEM SHALL initialize the search bar, filter controls, and pagination from those parameters (deep linking).
  - WHEN the user navigates back or forward in the browser, THE SYSTEM SHALL restore the UI state from the URL parameters via the `popstate` event.
  - Default values SHALL NOT appear in the URL.
- **Copy Item Link**:
  - Each item in the list SHALL display a link/share icon button.
  - WHEN the user clicks the copy link button, THE SYSTEM SHALL copy the item's external URL to the clipboard.
  - THE SYSTEM SHALL display a brief confirmation message (e.g., "Link copied to clipboard") after copying.

---

#### FE-PAGE-008: 404 Not Found

**Description**: Custom 404 page with randomized humorous content.

**Requirements**:

- WHEN the user navigates to a non-existent route, THE SYSTEM SHALL display a 404 page.
- THE SYSTEM SHALL randomly select and display one humorous anecdote about things not being found.
- THE SYSTEM SHALL provide a link back to the home page.
- The list of anecdotes SHALL be hardcoded in the frontend (no API call needed).
- THE SYSTEM SHALL include exactly 30 anecdotes, each with a source reference and link.
- Anecdote list (each entry includes the anecdote text, source, and source URL):

| #  | Anecdote | Source |
|----|----------|--------|
| 1  | "In 1998, a NASA Mars orbiter was lost because one team used metric units and another used imperial. It was never found." | [NASA Mars Climate Orbiter Mishap](https://solarsystem.nasa.gov/missions/mars-climate-orbiter/in-depth/) |
| 2  | "The Library of Alexandria — humanity's greatest collection of knowledge — was lost to history. And you can't even find a web page." | [Library of Alexandria, Wikipedia](https://en.wikipedia.org/wiki/Library_of_Alexandria) |
| 3  | "Amelia Earhart disappeared over the Pacific in 1937. She's still easier to find than this page." | [Amelia Earhart, Smithsonian](https://airandspace.si.edu/explore/stories/amelia-earhart) |
| 4  | "In 1971, DB Cooper hijacked a plane, parachuted out with $200,000, and vanished forever. This page did the same, minus the money." | [DB Cooper, FBI](https://www.fbi.gov/history/famous-cases/db-cooper-hijacking) |
| 5  | "The Roanoke Colony of 115 people vanished in the 1500s, leaving only the word 'CROATOAN.' This page left even less." | [Roanoke Colony, History.com](https://www.history.com/articles/what-happened-to-the-lost-colony-of-roanoke) |
| 6  | "Jimmy Hoffa has been missing since 1975. At least he had a last known location." | [Jimmy Hoffa, Biography.com](https://www.biography.com/crime/jimmy-hoffa) |
| 7  | "The Amber Room — an entire chamber of golden amber panels — was stolen by Nazis and never recovered. This page pulled the same trick." | [Amber Room, Smithsonian](https://www.smithsonianmag.com/history/a-brief-history-of-the-amber-room-160940121/) |
| 8  | "In 1900, three lighthouse keepers on Eilean Mòr vanished without a trace. The light went out — just like this page." | [Flannan Isles Lighthouse, Wikipedia](https://en.wikipedia.org/wiki/Flannan_Isles_lighthouse) |
| 9  | "The Holy Grail has been sought for over a thousand years. This URL was sought for about 3 seconds." | [Holy Grail, Britannica](https://www.britannica.com/topic/Holy-Grail) |
| 10 | "Nikola Tesla's missing papers after his death have fueled conspiracy theories for decades. This page is fueling one right now." | [Tesla's Missing Papers, History.com](https://www.history.com/articles/nikola-tesla-fbi-files-declassified-death-ray) |
| 11 | "The Bermuda Triangle has allegedly swallowed ships and planes for centuries. Today it swallowed a URL." | [Bermuda Triangle, NOAA](https://oceanservice.noaa.gov/facts/bermudatri.html) |
| 12 | "Ancient Romans lost an entire legion — the Ninth — and historians still argue about what happened. This page isn't even that interesting." | [Ninth Legion, Wikipedia](https://en.wikipedia.org/wiki/Legio_IX_Hispana) |
| 13 | "In computing, a 'heisenbug' disappears when you try to debug it. This page is a heisenpage." | [Heisenbug, Jargon File](http://www.catb.org/jargon/html/H/heisenbug.html) |
| 14 | "Schrödinger's cat is both alive and dead until observed. This page is both here and not here — but mostly not here." | [Schrödinger's cat, Stanford Encyclopedia of Philosophy](https://plato.stanford.edu/entries/qt-issues/#SchCat) |
| 15 | "The Voynich Manuscript has baffled cryptographers for 600 years. This 404 page took about 0.6 seconds." | [Voynich Manuscript, Yale](https://beinecke.library.yale.edu/collections/highlights/voynich-manuscript) |
| 16 | "Explorer Percy Fawcett vanished in the Amazon in 1925 searching for a lost city. You vanished searching for a lost page." | [Percy Fawcett, Britannica](https://www.britannica.com/biography/Percy-Harrison-Fawcett) |
| 17 | "The entire nation of Atlantis supposedly sank into the ocean. This URL didn't even make a splash." | [Atlantis, National Geographic](https://www.nationalgeographic.com/science/article/atlantis) |
| 18 | "There are an estimated 3 million shipwrecks on the ocean floor. And now there's one more — this URL." | [Shipwrecks, UNESCO](https://www.unesco.org/en/underwater-cultural-heritage) |
| 19 | "Genghis Khan's tomb has been hidden for 800 years. This page's hiding spot? Also a mystery." | [Genghis Khan's Tomb, National Geographic](https://www.nationalgeographic.com/culture/article/genghis-khan-tomb) |
| 20 | "Van Gogh couldn't find buyers for his paintings during his lifetime. You can't find this page in your lifetime." | [Van Gogh, Van Gogh Museum](https://www.vangoghmuseum.nl/en/art-and-stories/vincent-van-gogh-faq/how-many-paintings-did-van-gogh-sell-during-his-lifetime) |
| 21 | "The Zodiac Killer's identity remained unknown for over 50 years. This page prefers anonymity too." | [Zodiac Killer, FBI](https://www.fbi.gov/history/famous-cases/zodiac-killer) |
| 22 | "In 2013, Bitcoin developer James Howells accidentally threw away a hard drive containing 8,000 BTC. This page was discarded with similar carelessness." | [James Howells Bitcoin, The Guardian](https://www.theguardian.com/uk-news/2025/jan/09/man-james-howells-who-lost-bitcoin-hard-drive-loses-high-court-claim-newport-council) |
| 23 | "The 'Wow!' signal from space in 1977 was never explained or repeated. Neither will this page." | [Wow! Signal, Wikipedia](https://en.wikipedia.org/wiki/Wow!_signal) |
| 24 | "Oak Island's Money Pit has been excavated for 200+ years with nothing found. You've been here for 20 seconds with similar results." | [Oak Island, History Channel](https://www.history.com/shows/the-curse-of-oak-island) |
| 25 | "The Sodder children vanished on Christmas Eve 1945 and were never found. This page vanished on a regular Tuesday." | [Sodder Children, Smithsonian](https://www.smithsonianmag.com/history/the-children-who-went-up-in-smoke-172429802/) |
| 26 | "The Mary Celeste was found adrift in 1872 with no crew. This page was found adrift with no content." | [Mary Celeste, Britannica](https://www.britannica.com/topic/Mary-Celeste) |
| 27 | "In 1962, three inmates escaped Alcatraz and were never found. This page has also escaped." | [Alcatraz Escape, FBI](https://www.fbi.gov/history/famous-cases/alcatraz-escape) |
| 28 | "The Dyatlov Pass incident in 1959 left 9 hikers dead under mysterious circumstances. This page's disappearance is slightly less dramatic." | [Dyatlov Pass, National Geographic](https://www.nationalgeographic.com/science/article/has-science-solved-mystery-dyatlov-pass-deaths) |
| 29 | "Dark matter makes up 27% of the universe but has never been directly observed. This page makes up 0% of this website and also can't be observed." | [Dark Matter, NASA](https://science.nasa.gov/astrophysics/focus-areas/what-is-dark-energy/) |
| 30 | "A googol is 10^100 — a number so large it's practically unreachable. So is this page." | [Googol, Merriam-Webster](https://www.merriam-webster.com/dictionary/googol) |

---

#### FE-PAGE-009: Privacy Policy (`/privacy`)

**Description**: Static privacy policy page describing what data is collected, how it is used, and user rights.

**Requirements**:

- WHEN the user visits `/privacy`, THE SYSTEM SHALL display a static privacy policy page.
- THE SYSTEM SHALL hardcode the privacy policy content in the frontend (no backend endpoint needed).
- THE SYSTEM SHALL display a "Last updated" date at the top of the privacy policy.
- THE SYSTEM SHALL include the following sections in the privacy policy:
  - **Data We Collect**: Truncated IP address (last octet zeroed for IPv4, last 80 bits zeroed for IPv6), browser name and version, pages visited, referrer URL, link clicks (which URLs you click on), time-on-page engagement milestones (whether you stay on a page for 1, 2, or 5 minutes), connection speed information, a randomized session identifier (not linked to your identity), and timestamps. No full IP addresses are stored.
  - **How We Collect Data**: Anonymous tracking via the site's internal tracking mechanism when pages are visited, links are clicked, and engagement milestones are reached. Client-side error reporting for performance monitoring. A random session identifier is generated in your browser for each visit and is not stored after you close the tab. No cookies, session tracking, or third-party scripts are used.
  - **Purpose of Data Collection**: Understanding how helpful and engaging the site's content is to readers, monitoring site performance, improving user experience, identifying which content resonates with visitors, and discriminating between real human visitors and automated bots or scrapers that collect text without reading it. The engagement data (time on page, link clicks) helps the site owner understand whether content is genuinely useful rather than just visited.
  - **What We Do NOT Collect**: No cookies, no user accounts, no personal identification, no fingerprinting beyond truncated IP and browser information, no cross-session tracking (i.e., we cannot recognize you on a return visit), no third-party tracking scripts, no advertising data.
  - **Data Retention**: Visitor tracking data and error reports are stored as anonymized log entries in Google BigQuery for long-term analytics and trend analysis. This data is automatically deleted after 2 years. The same anonymization (IP truncation) applies — no full IP addresses are stored in BigQuery. No tracking or error report data is stored in the application database. Infrastructure-level load balancer logs (used for DDoS analysis and security incident response) may retain full IP addresses for up to 90 days; these logs are not used in analytics reports.
  - **Analytics**: The website owner uses Looker Studio dashboards connected to BigQuery to view aggregate analytics (e.g., unique visitor counts, popular pages, referrer sources, browser distribution). These dashboards are private and not publicly accessible. No data is shared with third parties.
  - **Data Storage**: Data is stored in Google Cloud infrastructure in the `asia-southeast1` region.
  - **Your Rights**: Users may contact TJ Monserrat to request information about or deletion of any data associated with their IP address.
  - **Contact**: Link to the socials/contact page (`/socials`) for inquiries.
  - **Changes to This Policy**: Notice that the policy may be updated, with the last updated date displayed.
- THE SYSTEM SHALL link the privacy policy from the site footer on all pages (see FE-COMP-010).

---

#### FE-PAGE-010: Changelog (`/changelog`)

**Description**: Static page displaying a human-readable changelog of frontend updates. Linked from the service worker update snackbar (FE-COMP-008) so users can review what changed before refreshing.

**Requirements**:

- WHEN the user visits `/changelog`, THE SYSTEM SHALL display a chronological list of frontend-only changes (most recent first).
- THE SYSTEM SHALL hardcode the changelog content in the frontend (no backend endpoint needed).
- Each changelog entry SHALL include:
  - A version identifier or date
  - A brief description of changes
- THE SYSTEM SHALL display a "Last updated" date at the top of the page.

---

### Global Components and Behaviors

#### FE-COMP-001: Search Bar

**Requirements**:

- THE SYSTEM SHALL limit search bar input to 300 characters maximum.
- WHEN the user types and exceeds 300 characters, THE SYSTEM SHALL prevent further input.
- WHEN the user pastes text exceeding 300 characters, THE SYSTEM SHALL truncate or reject the paste.
- THE SYSTEM SHALL include a client-side validator as a secondary safeguard.
- IF the frontend detects an attempt to submit a search query exceeding 300 characters, THE SYSTEM SHALL display a snackbar error: *"Search query has a limit of 300 characters."*

---

#### FE-COMP-002: Loading Bar

**Requirements**:

- WHEN the frontend sends a request to the backend, THE SYSTEM SHALL display a loading bar at the top of the page.
- WHEN the request completes (success or failure), THE SYSTEM SHALL hide the loading bar.

---

#### FE-COMP-003: Error Snackbar

**Requirements**:

- WHEN the backend returns an error, THE SYSTEM SHALL display a snackbar with a human-friendly message.
- Error message mapping:

| HTTP Status | User Message                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------------ |
| 400         | "Your request couldn't be processed. Please check your input and try again."                     |
| 403         | "Access to this site has been temporarily restricted."                                            |
| 404         | "The content you're looking for doesn't exist or has been removed."                              |
| 405         | "This action is not supported."                                                                  |
| 429         | "You're making too many requests. Please wait a moment and try again."                           |

- **429 handling note**: The frontend SHALL match 429 responses by HTTP status code only. It SHALL NOT attempt to parse or depend on the response body for 429 responses, because Cloud Armor generates these responses at the load balancer level and the body format may not be JSON (see CLR-042, INFRA-005 custom error response configuration).

| 503         | "The service is temporarily unavailable. Please try again later."                                |
| 504         | "TJ has been notified about the server taking too long." + randomized funny message (see below)  |
| Other       | "Something went wrong. TJ has been notified of this issue."                                      |

- Snackbar SHALL auto-dismiss after 10 seconds or be manually dismissable.

**504 Timeout Funny Messages**:

- THE SYSTEM SHALL display one of the following randomized funny messages alongside the 504 notification ("TJ has been notified about the server taking too long."):
  1. "The server is still thinking... like that one friend who takes 20 minutes to pick a restaurant."
  2. "The server took so long, I considered sending a carrier pigeon instead."
  3. "Plot twist: the server went on a coffee break and forgot to come back."
  4. "The server is moving at the speed of a sloth on a lazy Sunday."
  5. "This request took longer than my last software update. And that's saying something."
  6. "Even dial-up internet is judging how long this is taking."
  7. "The server is experiencing what philosophers call 'existential delay.'"
  8. "I asked the server for a response and it gave me a timeout. Rude."
  9. "The server went to find your data and got lost along the way."
  10. "At this rate, you could have hand-delivered the request faster."
- THE SYSTEM SHALL select a message randomly on each 504 occurrence.

---

#### FE-COMP-004: Visitor Tracking

**Requirements**:

- WHEN a user visits any page, THE SYSTEM SHALL send anonymous tracking data to the backend via `POST /t`.
- THE SYSTEM SHALL authenticate the request using a JWT included in the request body.
  - A static ID and static secret SHALL be embedded in the frontend code.
  - These credentials SHALL be obfuscated in the frontend bundle.
  - THE SYSTEM SHALL generate a short-lived JWT (5-minute expiry) from these credentials to be included as a `token` field in the JSON request body on each `POST /t` request (CLR-103).
  - The JWT SHALL include claims that identify the request as originating from the legitimate frontend.

**Delivery Mechanism**:

- THE SYSTEM SHALL use `navigator.sendBeacon()` as the primary delivery method for all `POST /t` requests. The request body SHALL be sent as a JSON `Blob` with `Content-Type: application/json`, including the `token` field.
- IF `navigator.sendBeacon()` is not available (e.g., in very old browsers), THE SYSTEM SHALL fall back to `fetch()` with `keepalive: true` as the delivery method.
- This progressive fallback ensures reliable delivery during page unload events while maintaining broad browser compatibility.

**Visitor Session ID**:

- WHEN the frontend application initializes (first page load in a browser tab/window), THE SYSTEM SHALL check `sessionStorage` for an existing `visitor_session_id`.
- IF no `visitor_session_id` exists, THE SYSTEM SHALL generate a new UUID v4 and store it in `sessionStorage` under the key `visitor_session_id`.
- THE SYSTEM SHALL include the `visitor_session_id` in every `POST /t` request body.
- The `visitor_session_id` is purely random and not derived from any user-identifiable information. It exists only for the duration of the browser session (tab/window lifetime) and is not persisted across sessions.
- The backend uses this value (combined with the truncated IP and User-Agent) to compute a non-reversible `visitor_id` hash for unique visitor counting and bot discrimination (see BE-API-009).

**Page View Tracking**:

- Tracking payload (JSON body) SHALL include:
  - `action`: `"page_view"`
  - `visitor_session_id`: The session-scoped UUID v4 from `sessionStorage`
  - `page`: Current page URL path
  - `referrer`: Referrer URL (if available)
  - `browser`: Browser name and version (from User-Agent, determined client-side)
  - `connection_speed`: Connection speed info from Navigator.connection API (if available)
- IP address and server-side timestamp are determined by the backend from request headers.

**Link Click Tracking**:

- WHEN a user clicks any anchor (`<a>`) element on a page, THE SYSTEM SHALL send a `link_click` tracking event to the backend via `POST /t`.
- Link click payload (JSON body) SHALL include:
  - `action`: `"link_click"`
  - `visitor_session_id`: The session-scoped UUID v4 from `sessionStorage`
  - `page`: Current page URL path where the click occurred
  - `clicked_url`: The `href` value of the clicked link
  - `browser`: Browser name and version
  - `connection_speed`: Connection speed info (if available)
- THE SYSTEM SHALL use event delegation (a single listener on a common ancestor) rather than attaching individual listeners to every anchor element.
- THE SYSTEM SHALL NOT block or delay the navigation. The tracking request SHALL be sent asynchronously (fire-and-forget via `navigator.sendBeacon()` with `fetch()` + `keepalive: true` fallback; see Delivery Mechanism above).
- THE SYSTEM SHALL NOT track clicks on the same-page anchor links (hash-only links like `#section`) as they are already covered by hash fragment navigation.

**Time-on-Page Tracking**:

- WHEN a user remains on a page for 1 minute, 2 minutes, or 5 minutes, THE SYSTEM SHALL send a `time_on_page` tracking event to the backend via `POST /t` for each milestone reached.
- Time-on-page payload (JSON body) SHALL include:
  - `action`: `"time_on_page"`
  - `visitor_session_id`: The session-scoped UUID v4 from `sessionStorage`
  - `page`: Current page URL path
  - `milestone`: One of `"1min"`, `"2min"`, `"5min"`
  - `browser`: Browser name and version
  - `connection_speed`: Connection speed info (if available)
- THE SYSTEM SHALL use `setTimeout` or `setInterval` to track elapsed time on the page.
- THE SYSTEM SHALL clear all pending timers when the user navigates away from the page (SPA route change or `beforeunload`).
- THE SYSTEM SHALL only fire each milestone once per page visit. If the user navigates away and returns to the same page, the timer resets.
- THE SYSTEM SHALL pause the timer when the page is not visible (using the Page Visibility API `document.visibilityState`) and resume when the page becomes visible again. Only active foreground time counts toward milestones.
- THE SYSTEM SHALL use `navigator.sendBeacon()` as the preferred method for sending milestone events (with `fetch()` + `keepalive: true` fallback; see Delivery Mechanism above), as it is reliable even during page unload.

**General Tracking Rules**:

- Tracking SHALL NOT include any personally identifiable information beyond what the browser naturally sends.
- Tracking failures SHALL be silent (no error shown to user).

---

#### FE-COMP-005: Frontend Error Reporting

**Requirements**:

- WHEN a network error or slow loading occurs, THE SYSTEM SHALL send error data to the backend via `POST /t`.
- THE SYSTEM SHALL use the same JWT body-based authentication mechanism as FE-COMP-004 (including the `token` field in each request body).
- Error payload (JSON body, sent to the same `POST /t` endpoint with `action: "error_report"`) SHALL include:
  - `action`: `"error_report"`
  - `visitor_session_id`: The session-scoped UUID v4 from `sessionStorage` (same as FE-COMP-004)
  - `error_type`: Error classification (e.g., `network`, `timeout`, `js_error`)
  - `error_message`: Error description
  - `page`: Page URL where the error occurred
  - `browser`: Browser name and version
  - `connection_speed`: Detected connection speed (via Navigator.connection API where available)
- IP address and server-side timestamp are determined by the backend from request headers.
- Error reporting failures SHALL be silent (no error shown to user).

---

#### FE-COMP-006: robots.txt

**Requirements**:

- THE SYSTEM SHALL serve a `robots.txt` file at the root of the frontend domain (`tjmonsi.com`).
- THE SYSTEM SHALL deploy `robots.txt` as a static file in Firebase Hosting's `public/` directory, served directly without any rewrite.
- THE SYSTEM SHALL specify crawl rules and rate limits for bots.
- THE SYSTEM SHALL block crawlers from tracking endpoints.
- See [07-observability-specifications.md](07-observability-specifications.md) OBS-007 for the full `robots.txt` content.

---

#### FE-COMP-007: Category Caching

**Requirements**:

- WHEN the frontend navigates to `/technical`, `/blog`, or `/others`, THE SYSTEM SHALL check sessionStorage for a cached categories list.
- IF cached categories exist and are less than 24 hours old, THE SYSTEM SHALL use the cached list.
- IF cached categories do not exist or are older than 24 hours, THE SYSTEM SHALL fetch categories from `GET /categories` and store the result in sessionStorage with a timestamp.
- THE SYSTEM SHALL use the categories list to populate the category filter dropdown.

---

#### FE-COMP-008: Offline Reading

**Requirements**:

- THE SYSTEM SHALL NOT be installable as a Progressive Web App (PWA). No web app manifest with `display: standalone` or equivalent SHALL be provided.
- THE SYSTEM SHALL use a service worker for offline reading support. The service worker enables intercepting navigation requests when offline and serving cached content. Using a service worker does NOT make the app installable — installability requires a web app manifest with specific fields.
- THE SYSTEM SHALL support offline reading of previously downloaded articles.
- **Smart Download (Heuristic Prefetching)**:
  - THE SYSTEM SHALL track user browsing patterns on the frontend (no server calls for this tracking).
  - Based on heuristics (e.g., articles the user has viewed partially, articles in the same category as recently read articles), THE SYSTEM SHALL prefetch and cache article content in the browser for offline reading.
  - Prefetching SHALL happen in the background without affecting page performance.
- **Manual Save**:
  - WHEN the user clicks "Save for offline reading" on an article page (FE-PAGE-003, FE-PAGE-005) or the list view (FE-PAGE-002, FE-PAGE-004), THE SYSTEM SHALL download and store the full article content locally.
- **Offline Availability Indicator**:
  - In list views (FE-PAGE-002, FE-PAGE-004), each article item SHALL display an indicator showing whether the article is available for offline reading.
- **Storage**:
  - **Cache API**: THE SYSTEM SHALL use the Cache API (via service worker) to cache article content HTTP responses. This enables fast retrieval of full article content when offline or on repeat visits.
  - **IndexedDB**: THE SYSTEM SHALL use IndexedDB to store article metadata (title, slug, category, tags, dates, abstract), reading state (e.g., reading progress, last accessed timestamp), and article content as structured data for quick retrieval without requiring a network request via the Cache API.
  - THE SYSTEM SHALL manage storage limits gracefully, removing the oldest cached articles when storage is full.
- **Offline Retrieval Strategy (Cache-First with Conditional Requests)**:
  - WHEN navigating to a URL, THE SYSTEM SHALL first check if cached content exists (Cache API for content responses, IndexedDB for metadata).
  - IF cached content exists, THE SYSTEM SHALL render from cache immediately.
  - IF the device has an active internet connection, THE SYSTEM SHALL check for updated content in the background using **conditional requests**:
    - THE SYSTEM SHALL send `If-None-Match` (with the cached `ETag`) and/or `If-Modified-Since` (with the cached `Last-Modified` date) headers to the backend article detail endpoints (BE-API-003, BE-API-005).
    - IF the backend returns `304 Not Modified`, the cached content is still current — no download or cache update is needed.
    - IF the backend returns `200` with updated content, THE SYSTEM SHALL update the cache and optionally notify the user that newer content is available.
  - IF the device is offline and no cached content exists, THE SYSTEM SHALL display a message indicating the article is not available offline.

**Service Worker Update Notification**:

- WHEN a new version of the service worker is detected (e.g., after a frontend deployment), THE SYSTEM SHALL display an update snackbar informing the user that a new version of the website is available.
- The update snackbar SHALL include:
  - A message: "A new version of the website is available."
  - A link to the changelog page (`/changelog`, see FE-PAGE-010) so the user can review what changed.
  - A "Refresh now" button that activates the new service worker and reloads the page.
  - A manual dismiss button.
- The update snackbar SHALL auto-dismiss after **20 seconds** if the user does not interact with it. This is longer than the default 10-second snackbar duration to give the user time to read the update information.
- IF the user dismisses the snackbar without refreshing, the new service worker SHALL be activated on the next full page load or navigation.

---

#### FE-COMP-009: Empty Search Results Handling

**Requirements**:

- WHEN the backend returns `200` with an empty `items` array for a search or filter query, THE SYSTEM SHALL display a randomized clever message instead of a blank list.
- THE SYSTEM SHALL maintain a list of at least 10 randomized empty-result messages. Examples:
  - "Hmm, I've got nothing. Maybe try different words?"
  - "The void stares back. No articles found."
  - "Even my best search algorithms came up empty. That's... rare."
  - "Zero results. Nada. Zilch. The page is as empty as my coffee cup."
  - "I searched high and low. Mostly low. Found nothing."
  - "This search returned exactly zero results. Let that sink in."
  - "No matches found. Are you sure that's a real word?"
  - "The articles are hiding. Very successfully."
  - "Plot twist: there are no results."
  - "You stumped me. Try a different search?"
- **Repeated Empty Search Detection**:
  - THE SYSTEM SHALL track consecutive empty search results in the current session (frontend-only, no server call).
  - IF the user submits a search that returns the same empty result (same or similar query), THE SYSTEM SHALL escalate the message to a more humorous tone. Examples:
    - "What are you trying to find? I genuinely want to help."
    - "Try and try again, but the results won't change."
    - "Did I ever tell you what the definition of insanity is? Insanity is doing the exact same thing over and over again expecting things to change."
    - "You've searched for this before. The answer is still nothing."
    - "Persistence is admirable, but the database is still empty for this query."
    - "Third time's a charm? Not this time."
    - "At this point, I think you're just testing me."
  - THE SYSTEM SHALL reset the repeated search counter when the user submits a different search query that returns results.
- Empty result messages SHALL be hardcoded in the frontend (no API call needed).

---

#### FE-COMP-010: Site Footer

**Requirements**:

- THE SYSTEM SHALL display a global site footer on all pages.
- THE SYSTEM SHALL include the following elements in the footer:
  - Navigation links to all main pages (same links as the header navigation, FE-COMP-012):
    - Home (`/`)
    - Technical Blog (`/technical`)
    - Opinions (`/blog`)
    - Other Links (`/socials`)
    - Notable Notes Outside (`/others`)
    - Privacy (`/privacy`)
    - Changelog (`/changelog`)
  - A copyright notice: `© TJ Monserrat`.
- THE SYSTEM SHALL render the footer consistently across desktop, tablet, and mobile viewports.

---

#### FE-COMP-011: URL State Synchronization

**Requirements**:

- THE SYSTEM SHALL synchronize the UI state with the browser URL on all pages that have search, filter, or navigation controls (FE-PAGE-002, FE-PAGE-004, FE-PAGE-007 for query parameters; FE-PAGE-003, FE-PAGE-005 for hash fragments).
- **Query Parameter Synchronization** (list/search pages):
  - THE SYSTEM SHALL use the browser History API (`history.pushState()`) to update the URL when the user changes search, filter, or pagination state.
  - THE SYSTEM SHALL NOT trigger a full page reload when updating URL parameters.
  - THE SYSTEM SHALL listen for the `popstate` event to restore UI state when the user navigates with browser back/forward buttons.
  - THE SYSTEM SHALL parse URL query parameters on initial page load and initialize all UI controls (search bar, filter dropdowns, date pickers, pagination) to match the URL state (deep linking).
  - THE SYSTEM SHALL encode parameter values using standard URL encoding (`encodeURIComponent`).
  - THE SYSTEM SHALL omit default values from the URL to keep URLs clean:
    - Empty `q` (no search) → omit `q`
    - No `category` selected → omit `category`
    - No `tags` selected → omit `tags`
    - `tag_match=any` (default) → omit `tag_match`
    - No date range → omit `date_from` and `date_to`
    - `page=1` (first page) → omit `page`
  - WHEN the user clears all filters and search, THE SYSTEM SHALL restore the URL to the base path (e.g., `/technical`).
- **Hash Fragment Synchronization** (article pages):
  - THE SYSTEM SHALL update the URL hash fragment when the user clicks a table of contents entry or a heading link icon.
  - THE SYSTEM SHALL scroll to the target heading when the page loads with a hash fragment in the URL.
  - THE SYSTEM SHALL use `history.pushState()` for hash fragment updates to enable browser back/forward navigation between sections.
- **Copy to Clipboard**:
  - THE SYSTEM SHALL use the Clipboard API (`navigator.clipboard.writeText()`) for all copy link operations.
  - IF the Clipboard API is not available (e.g., non-HTTPS context in development), THE SYSTEM SHALL fall back to a `document.execCommand('copy')` approach.
  - THE SYSTEM SHALL display a confirmation message via a snackbar or tooltip (e.g., "Link copied to clipboard") that auto-dismisses after 2 seconds.

---

#### FE-COMP-012: Site Header Navigation

**Requirements**:

- THE SYSTEM SHALL display a global site header on all pages.
- THE SYSTEM SHALL include navigation links to all main pages:
  - Home (`/`)
  - Technical Blog (`/technical`)
  - Opinions (`/blog`)
  - Other Links (`/socials`)
  - Notable Notes Outside (`/others`)
  - Privacy (`/privacy`)
  - Changelog (`/changelog`)
- **Desktop**: THE SYSTEM SHALL display all navigation links horizontally in the header.
- **Mobile**:
  - THE SYSTEM SHALL display a hamburger menu icon on the left side of the header.
  - WHEN the user taps the hamburger icon, THE SYSTEM SHALL open a navigation drawer that slides in from the left side of the screen.
  - The navigation drawer SHALL contain all the same navigation links as the desktop header.
  - THE SYSTEM SHALL close the navigation drawer when the user taps outside of it, taps a navigation link, or taps a close button.
  - THE SYSTEM SHALL include a semi-transparent overlay behind the navigation drawer when it is open.
- THE SYSTEM SHALL visually indicate the currently active page in the navigation.

---

### Responsive Design

- THE SYSTEM SHALL be responsive and functional on desktop, tablet, and mobile viewports.
- Site header navigation (FE-COMP-012) SHALL collapse to a hamburger menu with a left-sliding navigation drawer on mobile.
- Table of contents (FE-PAGE-003, FE-PAGE-005) SHALL collapse to a hamburger/dropdown on mobile.
- Search bars and filters SHALL stack vertically on mobile.

### Accessibility

- THE SYSTEM SHALL meet WCAG 2.1 Level AA compliance.
- All interactive elements SHALL be keyboard navigable.
- All images SHALL have alt text.
- Color contrast SHALL meet minimum requirements.

---

### Acceptance Criteria

- **AC-FE-001**: Given a user visiting any page, when the page loads, then the site header navigation (FE-COMP-012) and site footer (FE-COMP-010) are visible with all navigation links.
- **AC-FE-002**: Given a mobile viewport, when the user taps the hamburger icon, then a navigation drawer slides in from the left with all page links.
- **AC-FE-003**: Given a user visiting `/technical` or `/blog`, when articles exist, then the list displays with title, abstract, offline indicator, and copy-link button.
- **AC-FE-004**: Given a user on a desktop viewport viewing `/technical`, when there are multiple pages of results, then traditional pagination controls (Previous/Next) are displayed.
- **AC-FE-005**: Given a user on a mobile viewport viewing `/technical`, when they scroll to the bottom, then additional articles load automatically via infinite scroll.
- **AC-FE-006**: Given a user visiting `/technical/:slug.md`, when the article is loaded, then the YAML front matter is parsed and metadata, changelog, body, table of contents, and citation selector are rendered.
- **AC-FE-007**: Given a rendered article with image references, when images use `https://api.tjmonsi.com/images/...` URLs, then images load successfully under the CSP `img-src` policy.
- **AC-FE-008**: Given a user visiting any page, when the page loads, then a `page_view` tracking event is sent to `POST /t` with the JWT `token` in the request body via `navigator.sendBeacon()` (or `fetch` fallback).
- **AC-FE-009**: Given a user clicking an anchor element, when the click event fires, then a `link_click` tracking event is sent asynchronously without blocking navigation.
- **AC-FE-010**: Given a backend error response, when the response status is 429, then a snackbar displays "You're making too many requests..." and auto-dismisses after 10 seconds.
- **AC-FE-011**: Given a previously visited article cached offline, when the user goes offline and navigates to that article, then the cached content is rendered.
- **AC-FE-012**: Given a new service worker version is detected, when the user is on the site, then an update snackbar appears with a changelog link and refresh button, displayed for 20 seconds.
- **AC-FE-013**: Given a user visiting `/changelog`, when the page loads, then a chronological list of frontend changes is displayed.
- **AC-FE-014**: Given a user visiting `/privacy`, when the page loads, then the static privacy policy content is displayed with all required sections.
- **AC-FE-015**: Given WCAG 2.1 Level AA requirements, when any page is audited, then all interactive elements are keyboard navigable and color contrast meets minimum requirements.
