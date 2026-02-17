## Frontend Specifications

### Technology

- **Framework**: Nuxt 4 (static/SPA mode, no SSR)
- **UI Framework**: Vue 3 (Composition API)
- **Build Tool**: Vite
- **Hosting**: Firebase Hosting + Firebase Functions
- **Rendering Mode**: Client-side only

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

**Description**: Paginated list of technical articles with search and filtering.

**Requirements**:

- WHEN the user visits `/technical`, THE SYSTEM SHALL display a list of technical blog articles fetched from `GET /technical`.
- Each article item SHALL display:
  - Title (clickable, links to the article page)
  - Abstract with a "Read more" button (links to the article page)
  - Citation text (clickable, links to the article page)
- THE SYSTEM SHALL display a search bar at the top of the page.
- THE SYSTEM SHALL display filter controls for:
  - Category (single-select dropdown)
  - Tags (multi-select)
  - Date range for "last updated" (date picker range)
- THE SYSTEM SHALL paginate results showing the current page number.
- THE SYSTEM SHALL display navigation buttons:
  - "Previous" button: hidden on the first page
  - "Next" button: hidden on the last page
- THE SYSTEM SHALL show the current page and total pages.

---

#### FE-PAGE-003: Technical Blog Article (`/technical/:slug`)

**Description**: Full article view with table of contents and metadata.

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
- THE SYSTEM SHALL support article pagination (multi-page articles):
  - Show current page number
  - "Previous" button: hidden on first page
  - "Next" button: hidden on last page

---

#### FE-PAGE-004: Opinions Blog List (`/blog`)

**Description**: Same structure and behavior as FE-PAGE-002 but for opinion articles.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-002 (Technical Blog List).
- THE SYSTEM SHALL fetch data from `GET /blog` instead of `GET /technical`.

---

#### FE-PAGE-005: Opinion Blog Article (`/blog/:slug`)

**Description**: Same structure and behavior as FE-PAGE-003 but for opinion articles.

**Requirements**:

- THE SYSTEM SHALL follow the same specifications as FE-PAGE-003 (Technical Blog Article).
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
  - Category (single-select dropdown)
  - Tags (multi-select)
  - Date range
- THE SYSTEM SHALL paginate results (same behavior as FE-PAGE-002).

---

#### FE-PAGE-008: 404 Not Found

**Description**: Custom 404 page with randomized humorous content.

**Requirements**:

- WHEN the user navigates to a non-existent route, THE SYSTEM SHALL display a 404 page.
- THE SYSTEM SHALL randomly select and display one humorous anecdote about things not being found.
- THE SYSTEM SHALL provide a link back to the home page.
- The list of anecdotes SHALL be hardcoded in the frontend (no API call needed).

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

- WHEN a user visits any page, THE SYSTEM SHALL send anonymous tracking data to the backend.
- Tracking payload SHALL include:
  - Browser name and version (from User-Agent)
  - IP address (determined server-side from request headers)
  - Current page URL (action)
  - Referrer URL (if available)
  - Timestamp
- Tracking SHALL NOT include any personally identifiable information beyond IP address.
- Tracking failures SHALL be silent (no error shown to user).

> **Note**: See Clarification CLR-002 regarding GET-only constraint and tracking data transmission.

---

#### FE-COMP-005: Frontend Error Reporting

**Requirements**:

- WHEN a network error or slow loading occurs, THE SYSTEM SHALL send error data to the backend.
- Error payload SHALL include:
  - Error type and message
  - Browser name and version
  - IP address (determined server-side)
  - Detected connection speed (via Navigator.connection API where available)
  - Page URL where the error occurred
  - Timestamp
- Error reporting failures SHALL be silent (no error shown to user).

> **Note**: See Clarification CLR-002 regarding GET-only constraint and error data transmission.

---

#### FE-COMP-006: robots.txt

**Requirements**:

- THE SYSTEM SHALL serve a `robots.txt` file at the root of the domain.
- THE SYSTEM SHALL specify crawl rules and rate limits for bots.
- Specific rules: *See Clarification CLR-009*.

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
