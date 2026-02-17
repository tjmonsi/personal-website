---
title: Frontend Specifications
version: 1.2
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

---

#### FE-PAGE-003: Technical Blog Article (`/technical/:slug`)

**Description**: Full article view with table of contents, metadata, and citation format selector.

**Requirements**:

- WHEN the user visits `/technical/:slug`, THE SYSTEM SHALL fetch and display the article from `GET /technical/{slug}.md`.
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
- THE SYSTEM SHALL display a table of contents on the right side:
  - Generated from article headings (H2, H3, H4)
  - Each entry is a clickable anchor link
  - Sticky/fixed position while scrolling
- THE SYSTEM SHALL display a citation section with a format selector:
  - The user SHALL be able to select from multiple citation formats:
    - **APA**: `Monserrat, T.J. (2025). Article Title. tjmonserrat.com. https://tjmonserrat.com/technical/slug.md`
    - **MLA**: `Monserrat, TJ. "Article Title." tjmonserrat.com, 15 Jan. 2025, https://tjmonserrat.com/technical/slug.md.`
    - **Chicago**: `Monserrat, TJ. "Article Title." tjmonserrat.com. January 15, 2025. https://tjmonserrat.com/technical/slug.md.`
    - **BibTeX**: BibTeX entry for academic use
    - **IEEE**: `[N] T.J. Monserrat, "Article Title," tjmonserrat.com, Jan. 15, 2025. [Online]. Available: https://tjmonserrat.com/technical/slug.md`
  - WHEN the user clicks the citation text, THE SYSTEM SHALL copy the selected citation format to the clipboard.
  - THE SYSTEM SHALL display a confirmation message (e.g., "Citation copied to clipboard") after copying.
- THE SYSTEM SHALL provide a "Save for offline reading" button.

---

#### FE-PAGE-004: Opinions Blog List (`/blog`)

**Description**: Same structure and behavior as FE-PAGE-002 but for opinion articles. Desktop uses traditional pagination; mobile uses infinite scroll.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-002 (Technical Blog List), including desktop pagination and mobile infinite scroll.
- THE SYSTEM SHALL fetch data from `GET /blog` instead of `GET /technical`.
- THE SYSTEM SHALL fetch categories from `GET /categories` (same cache as `/technical`).

---

#### FE-PAGE-005: Opinion Blog Article (`/blog/:slug`)

**Description**: Same structure and behavior as FE-PAGE-003 but for opinion articles.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-003 (Technical Blog Article), including citation format selector.
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
| 404         | "The content you're looking for doesn't exist or has been removed."                              |
| 405         | "This action is not supported."                                                                  |
| 429         | "You're making too many requests. Please wait a moment and try again."                           |
| 503         | "The service is temporarily unavailable. Please try again later."                                |
| Other       | "Something went wrong. TJ has been notified of this issue."                                      |

- Snackbar SHALL auto-dismiss after 5 seconds or be manually dismissable.

---

#### FE-COMP-004: Visitor Tracking

**Requirements**:

- WHEN a user visits any page, THE SYSTEM SHALL send anonymous tracking data to the backend via `POST /t`.
- THE SYSTEM SHALL authenticate the request using a Bearer Token (JWT).
  - A static ID and static secret SHALL be embedded in the frontend code.
  - These credentials SHALL be obfuscated in the frontend bundle.
  - THE SYSTEM SHALL generate a JWT from these credentials to be sent as `Authorization: Bearer <token>` on each `POST /t` request.
  - The JWT SHALL include claims that identify the request as originating from the legitimate frontend.
- Tracking payload (JSON body) SHALL include:
  - `page`: Current page URL
  - `referrer`: Referrer URL (if available)
  - `action`: User action identifier (e.g., `page_view`)
  - `browser`: Browser name and version (from User-Agent, determined client-side)
  - `connection_speed`: Connection speed info from Navigator.connection API (if available)
- IP address and server-side timestamp are determined by the backend from request headers.
- Tracking SHALL NOT include any personally identifiable information beyond what the browser naturally sends.
- Tracking failures SHALL be silent (no error shown to user).

---

#### FE-COMP-005: Frontend Error Reporting

**Requirements**:

- WHEN a network error or slow loading occurs, THE SYSTEM SHALL send error data to the backend via `POST /t`.
- THE SYSTEM SHALL use the same JWT authentication mechanism as FE-COMP-004.
- Error payload (JSON body, sent to the same `POST /t` endpoint with `action: "error_report"`) SHALL include:
  - `action`: `"error_report"`
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

- THE SYSTEM SHALL serve a `robots.txt` file at the root of the domain.
- THE SYSTEM SHALL specify crawl rules and rate limits for bots.
- THE SYSTEM SHALL block crawlers from API-only paths and tracking endpoints.
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
- **Offline Retrieval Strategy (Cache-First)**:
  - WHEN navigating to a URL, THE SYSTEM SHALL first check if cached content exists (Cache API for content responses, IndexedDB for metadata).
  - IF cached content exists, THE SYSTEM SHALL render from cache immediately.
  - IF the device has an active internet connection, THE SYSTEM SHALL check for updated content in the background. IF the content has been updated (based on `date_updated`), THE SYSTEM SHALL update the cache and optionally notify the user that newer content is available.
  - IF the device is offline and no cached content exists, THE SYSTEM SHALL display a message indicating the article is not available offline.

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

### Responsive Design

- THE SYSTEM SHALL be responsive and functional on desktop, tablet, and mobile viewports.
- Table of contents (FE-PAGE-003, FE-PAGE-005) SHALL collapse to a hamburger/dropdown on mobile.
- Search bars and filters SHALL stack vertically on mobile.

### Accessibility

- THE SYSTEM SHALL meet WCAG 2.1 Level AA compliance.
- All interactive elements SHALL be keyboard navigable.
- All images SHALL have alt text.
- Color contrast SHALL meet minimum requirements.
