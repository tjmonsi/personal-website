---
title: Data Model Specifications
version: 1.4
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [data-model, database, firestore, mongodb]
---

## Data Model Specifications

### Database

- **Technology**: Firestore Enterprise in MongoDB compatibility mode
- **Region**: `asia-southeast1`
- **Driver**: MongoDB Go driver (standard MongoDB wire protocol)
- **Reference**: https://firebase.google.com/docs/firestore/enterprise/mongodb-compatibility-overview

Data models below use MongoDB-style field definitions, compatible with the MongoDB wire protocol exposed by Firestore Enterprise.

---

### Collections / Tables

#### DM-001: `frontpage`

**Description**: Stores the front page markdown content.

| Field         | Type     | Required | Description                          |
| ------------- | -------- | -------- | ------------------------------------ |
| `_id`         | string   | Yes      | Document identifier                  |
| `content`     | string   | Yes      | Markdown content of the front page   |
| `date_updated`| datetime | Yes      | Last update timestamp (UTC)          |

**Constraints**:
- Single document collection (only one front page).

---

#### DM-002: `technical_articles`

**Description**: Stores technical blog articles.

| Field          | Type       | Required | Description                                    |
| -------------- | ---------- | -------- | ---------------------------------------------- |
| `_id`          | string     | Yes      | Document identifier                            |
| `title`        | string     | Yes      | Article title                                  |
| `slug`         | string     | Yes      | URL-safe slug (format: `{title-slug}-{YYYY}-{MM}-{DD}-{HHMM}`) |
| `category`     | string     | Yes      | Article category (free-form)                   |
| `abstract`     | string     | Yes      | Brief summary / excerpt                        |
| `tags`         | string[]   | Yes      | List of tags                                   |
| `content`      | string     | Yes      | Full article body in markdown                  |
| `changelog`    | object[]   | Yes      | Array of changelog entries                     |
| `changelog[].date`        | datetime | Yes | Date of the change                    |
| `changelog[].description` | string   | Yes | Description of what changed           |
| `date_created` | datetime   | Yes      | First publication date (UTC)                   |
| `date_updated` | datetime   | Yes      | Last update date (UTC)                         |

**Indexes**:

| Index Name              | Fields                          | Type     | Purpose                      |
| ----------------------- | ------------------------------- | -------- | ---------------------------- |
| `idx_slug`              | `slug`                          | Unique   | Slug lookups                 |
| `idx_category`          | `category`                      | Standard | Category filtering           |
| `idx_tags`              | `tags`                          | Multikey | Tag filtering                |
| `idx_date_updated`      | `date_updated`                  | Standard | Date range queries, sorting  |
| `idx_date_created`      | `date_created`                  | Standard | Internal use / content management queries |
| `idx_text_search`       | `title`, `abstract`, `content`  | Text     | Full-text search             |

**Constraints**:
- `slug` must be unique.
- `slug` must match pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}$`
  - Example: `my-first-article-2025-01-15-1030`

**Notes**:
- Citations are generated dynamically by the backend from article metadata (title, author, dates, slug). No citation field is stored.
- Categories are free-form strings. When an article is created or updated, the CI/CD pipeline ensures the category exists in the `categories` collection.
- The `author` value (`"TJ Monserrat"`) is not stored in this collection. It is hardcoded in the backend API response since this is a single-author website.

---

#### DM-003: `blog_articles`

**Description**: Stores opinion/blog articles. Identical schema to `technical_articles`.

| Field          | Type       | Required | Description                                    |
| -------------- | ---------- | -------- | ---------------------------------------------- |
| (Same as DM-002) | | | |

**Indexes**: Same as DM-002.

> **Design Decision**: Separate collections for technical and blog articles to maintain clear data boundaries, despite identical schemas. This allows independent scaling, querying, and potential future schema divergence.

---

#### DM-004: `socials`

**Description**: Stores social media and contact links. Links are managed through the content management repository and pushed to the database.

| Field          | Type     | Required | Description                          |
| -------------- | -------- | -------- | ------------------------------------ |
| `_id`          | string   | Yes      | Document identifier                  |
| `platform`     | string   | Yes      | Platform name (e.g., "GitHub", "LinkedIn", "Twitter/X", "YouTube", "Mastodon", "Email") |
| `url`          | string   | Yes      | Full URL to the profile or contact (e.g., `https://github.com/tjmonserrat` or `mailto:tj@example.com`) |
| `display_name` | string   | Yes      | Display text shown to users (e.g., "@tjmonserrat") |
| `icon`         | string   | No       | Icon identifier for the platform (e.g., `github`, `linkedin`, `twitter`, `youtube`, `mastodon`, `email`). Used by the frontend to render the appropriate icon. |
| `sort_order`   | integer  | Yes      | Display order (ascending). Determines the order in which links appear on the socials page. |
| `is_active`    | boolean  | Yes      | Whether the link is actively displayed. Allows hiding links without deleting them. Default: `true`. |
| `date_created` | datetime | Yes      | When the social link was added (UTC)  |
| `date_updated` | datetime | Yes      | Last update timestamp (UTC)           |

**Indexes**:

| Index Name         | Fields       | Type     | Purpose                |
| ------------------ | ------------ | -------- | ---------------------- |
| `idx_sort_order`   | `sort_order` | Standard | Ordered display        |
| `idx_is_active`    | `is_active`  | Standard | Filter active links    |

**Notes**:
- The `platform` field is free-form to support any social platform without code changes.
- The `icon` field maps to a frontend icon set (e.g., Material Design Icons, Font Awesome). The frontend handles rendering.
- Only entries with `is_active: true` are returned by the `GET /socials` API endpoint.

---

#### DM-005: `others`

**Description**: Stores curated external content references.

| Field          | Type       | Required | Description                                    |
| -------------- | ---------- | -------- | ---------------------------------------------- |
| `_id`          | string     | Yes      | Document identifier                            |
| `title`        | string     | Yes      | Title of the external content                  |
| `slug`         | string     | Yes      | URL-safe identifier                            |
| `category`     | string     | Yes      | Content category                               |
| `abstract`     | string     | Yes      | Brief description                              |
| `tags`         | string[]   | Yes      | List of tags                                   |
| `external_url` | string     | Yes      | URL to the external content                    |
| `date_created` | datetime   | Yes      | Date of original content (UTC)                 |
| `date_updated` | datetime   | Yes      | Last metadata update (UTC)                     |

**Indexes**:

| Index Name              | Fields                          | Type     | Purpose                      |
| ----------------------- | ------------------------------- | -------- | ---------------------------- |
| `idx_slug`              | `slug`                          | Unique   | Slug lookups                 |
| `idx_category`          | `category`                      | Standard | Category filtering           |
| `idx_tags`              | `tags`                          | Multikey | Tag filtering                |
| `idx_date_created`      | `date_created`                  | Standard | Date range queries           |
| `idx_date_updated`      | `date_updated`                  | Standard | Date range queries           |
| `idx_text_search`       | `title`, `abstract`             | Text     | Full-text search             |

---

#### DM-006: `categories`

**Description**: Stores the list of all content categories across all article types (technical, blog, others). Categories are free-form and derived from articles. This collection is managed by the content management CI/CD pipeline.

| Field          | Type     | Required | Description                          |
| -------------- | -------- | -------- | ------------------------------------ |
| `_id`          | string   | Yes      | Document identifier                  |
| `name`         | string   | Yes      | Category name (e.g., "DevOps", "Cloud", "AI/ML") |
| `date_created` | datetime | Yes      | When the category was first created (UTC) |

**Indexes**:

| Index Name         | Fields   | Type   | Purpose                |
| ------------------ | -------- | ------ | ---------------------- |
| `idx_name`         | `name`   | Unique | Category name lookups, prevent duplicates |

**Notes**:
- Categories are populated by the content management CI/CD pipeline when articles are created or updated.
- The `GET /categories` API endpoint returns all entries from this collection.
- The frontend caches the categories list in sessionStorage for 24 hours.

---

#### DM-007: `tracking`

**Description**: Stores anonymous page visit tracking data. Data is written by the `POST /t` endpoint.

| Field          | Type     | Required | Description                                    |
| -------------- | -------- | -------- | ---------------------------------------------- |
| `_id`          | string   | Yes      | Document identifier                            |
| `page`         | string   | Yes      | Page URL visited                               |
| `referrer`     | string   | No       | Referrer URL                                   |
| `action`       | string   | Yes      | User action (e.g., "page_view")                |
| `browser`      | string   | Yes      | Browser name and version                       |
| `ip_address`   | string   | Yes      | Client IP address                              |
| `connection_speed` | object | No     | Connection speed info (effective_type, downlink, rtt) |
| `timestamp`    | datetime | Yes      | Visit timestamp (UTC)                          |

**Indexes**:

| Index Name         | Fields       | Type     | Purpose                |
| ------------------ | ------------ | -------- | ---------------------- |
| `idx_timestamp`    | `timestamp`  | Standard | Time-based queries     |
| `idx_page`         | `page`       | Standard | Page hit analytics     |

**Data Retention**: TTL index SHALL auto-expire tracking data after 90 days.

---

#### DM-008: `error_reports`

**Description**: Stores frontend error reports.

| Field             | Type     | Required | Description                             |
| ----------------- | -------- | -------- | --------------------------------------- |
| `_id`             | string   | Yes      | Document identifier                     |
| `error_type`      | string   | Yes      | Error classification                    |
| `error_message`   | string   | Yes      | Error description                       |
| `page`            | string   | Yes      | Page URL where error occurred           |
| `browser`         | string   | Yes      | Browser name and version                |
| `ip_address`      | string   | Yes      | Client IP address                       |
| `connection_speed`| object   | No       | Connection speed info (effective_type, downlink, rtt) |
| `timestamp`       | datetime | Yes      | Error timestamp (UTC)                   |

**Data Retention**: TTL index SHALL auto-expire error reports after 30 days.

---

#### DM-009: `rate_limit_offenders`

**Description**: Tracks repeat rate limit offenders for progressive blocking. Offense records are written by a Cloud Function that processes Cloud Armor rate-limit (429) log entries via a log sink (see INFRA-008c). The Go application reads this collection on each request to check for active bans and enforce progressive banning tiers.

| Field             | Type       | Required | Description                                 |
| ----------------- | ---------- | -------- | ------------------------------------------- |
| `_id`             | string     | Yes      | Document identifier                         |
| `identifier`      | string     | Yes      | Client identifier (IP or fingerprint)       |
| `offense_count`   | integer    | Yes      | Total rate limit violations                 |
| `offense_history` | object[]   | Yes      | Array of offense records                    |
| `offense_history[].timestamp` | datetime | Yes | When the offense occurred          |
| `offense_history[].endpoint`  | string   | Yes | Which endpoint was hit             |
| `current_ban`     | object     | No       | Active ban details                          |
| `current_ban.start`   | datetime | Yes    | Ban start time                              |
| `current_ban.end`     | datetime | No     | Ban end time (null = indefinite)            |
| `current_ban.tier`    | string   | Yes    | Ban tier: "30d", "90d", "indefinite"        |
| `ban_history`     | object[]   | Yes      | Array of past bans                          |

**Indexes**:

| Index Name         | Fields        | Type   | Purpose                    |
| ------------------ | ------------- | ------ | -------------------------- |
| `idx_identifier`   | `identifier`  | Unique | Fast lookup for rate checks|

**Data Retention**:

- THE SYSTEM SHALL remove offense records where there is no active ban (`current_ban` is null or expired) and no offenses in the last 90 days.
- Offense records with indefinite bans SHALL be retained but subject to periodic manual review.
- A scheduled cleanup (e.g., via a Cloud Scheduler job or application-level TTL check) SHALL enforce this retention policy.

---

#### DM-010: `sitemap`

**Description**: Stores the pre-generated sitemap XML. A single document is maintained, periodically regenerated by a Cloud Function triggered by a Cloud Scheduler job (see INFRA-008).

| Field          | Type     | Required | Description                                    |
| -------------- | -------- | -------- | ---------------------------------------------- |
| `_id`          | string   | Yes      | Document identifier (fixed: `"current"`)       |
| `content`      | string   | Yes      | Full sitemap XML content                       |
| `generated_at` | datetime | Yes      | When the sitemap was last generated (UTC)      |
| `url_count`    | integer  | Yes      | Number of URLs included in the sitemap         |

**Constraints**:
- Single document collection (only one active sitemap).
- The sitemap is regenerated by a Cloud Scheduler job (see INFRA-008 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)).
- The `GET /sitemap.xml` API endpoint reads from this collection (see BE-API-011 in [03-backend-api-specifications.md](03-backend-api-specifications.md)).

---

### Data Relationships

```
frontpage (1 document)
    └── standalone

technical_articles (many documents)
    └── standalone, queried by slug/category/tags/date_updated
    └── category field → referenced in categories collection

blog_articles (many documents)
    └── standalone, identical structure to technical_articles
    └── category field → referenced in categories collection

categories (many documents)
    └── derived from article categories across all article types
    └── synced by content management CI/CD pipeline

socials (few documents)
    └── standalone, ordered by sort_order, filtered by is_active

others (many documents)
    └── standalone, links to external URLs
    └── category field → referenced in categories collection

tracking (many documents, append-only)
    └── standalone, TTL auto-expiry (90 days)

error_reports (many documents, append-only)
    └── standalone, TTL auto-expiry (30 days)

rate_limit_offenders (many documents)
    └── offense records written by process-rate-limit-logs Cloud Function (INFRA-008c) via Cloud Armor log sink
    └── read by Cloud Run Go API on each request for ban status checks
    └── real-time rate counting handled by Cloud Armor (see INFRA-005)

sitemap (single document)
    └── standalone, generated by Cloud Scheduler job
    └── served via GET /sitemap.xml endpoint
```
