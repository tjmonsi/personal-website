## Data Model Specifications

### Database

> **IMPORTANT**: See Clarification CLR-001 regarding the database technology choice. The initial plan mentions "Firestore Enterprise Database using MongoDB ORM," which is contradictory. The data models below are expressed in a database-agnostic format.

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
| `slug`         | string     | Yes      | URL-safe slug (format: `{title-slug}-{datetime}`) |
| `category`     | string     | Yes      | Article category                               |
| `abstract`     | string     | Yes      | Brief summary / excerpt                        |
| `tags`         | string[]   | Yes      | List of tags                                   |
| `content`      | string     | Yes      | Full article body in markdown                  |
| `citation`     | string     | Yes      | Citation text for referencing the article       |
| `changelog`    | object[]   | Yes      | Array of changelog entries                     |
| `changelog[].date`        | datetime | Yes | Date of the change                    |
| `changelog[].description` | string   | Yes | Description of what changed           |
| `date_created` | datetime   | Yes      | First publication date (UTC)                   |
| `date_updated` | datetime   | Yes      | Last update date (UTC)                         |
| `total_pages`  | integer    | No       | Number of pages if article is paginated        |

**Indexes**:

| Index Name              | Fields                          | Type     | Purpose                      |
| ----------------------- | ------------------------------- | -------- | ---------------------------- |
| `idx_slug`              | `slug`                          | Unique   | Slug lookups                 |
| `idx_category`          | `category`                      | Standard | Category filtering           |
| `idx_tags`              | `tags`                          | Multikey | Tag filtering                |
| `idx_date_created`      | `date_created`                  | Standard | Date range queries, sorting  |
| `idx_date_updated`      | `date_updated`                  | Standard | Date range queries           |
| `idx_text_search`       | `title`, `abstract`, `content`  | Text     | Full-text search             |

**Constraints**:
- `slug` must be unique.
- `slug` must match pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}T\d{2}-\d{2}-\d{2}$`
  - *See CLR-005 for confirmation of slug format.*

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

**Description**: Stores social media links.

| Field          | Type     | Required | Description                          |
| -------------- | -------- | -------- | ------------------------------------ |
| `_id`          | string   | Yes      | Document identifier                  |
| `platform`     | string   | Yes      | Platform name (e.g., "GitHub")       |
| `url`          | string   | Yes      | Full URL to the profile              |
| `display_name` | string   | Yes      | Display name on the platform         |
| `icon`         | string   | No       | Icon identifier or URL               |
| `sort_order`   | integer  | No       | Display order (ascending)            |

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
| `idx_text_search`       | `title`, `abstract`             | Text     | Full-text search             |

---

#### DM-006: `tracking`

**Description**: Stores anonymous page visit tracking data.

| Field          | Type     | Required | Description                                    |
| -------------- | -------- | -------- | ---------------------------------------------- |
| `_id`          | string   | Yes      | Document identifier                            |
| `page`         | string   | Yes      | Page URL visited                               |
| `referrer`     | string   | No       | Referrer URL                                   |
| `action`       | string   | Yes      | User action (e.g., "page_view")                |
| `browser`      | string   | Yes      | Browser name and version                       |
| `ip_address`   | string   | Yes      | Client IP address                              |
| `timestamp`    | datetime | Yes      | Visit timestamp (UTC)                          |

**Indexes**:

| Index Name         | Fields       | Type     | Purpose                |
| ------------------ | ------------ | -------- | ---------------------- |
| `idx_timestamp`    | `timestamp`  | Standard | Time-based queries     |
| `idx_page`         | `page`       | Standard | Page hit analytics     |

**Data Retention**: Consider TTL index to auto-expire tracking data after a configurable period (e.g., 90 days).

---

#### DM-007: `error_reports`

**Description**: Stores frontend error reports.

| Field             | Type     | Required | Description                             |
| ----------------- | -------- | -------- | --------------------------------------- |
| `_id`             | string   | Yes      | Document identifier                     |
| `error_type`      | string   | Yes      | Error classification                    |
| `error_message`   | string   | Yes      | Error description                       |
| `page`            | string   | Yes      | Page URL where error occurred           |
| `browser`         | string   | Yes      | Browser name and version                |
| `ip_address`      | string   | Yes      | Client IP address                       |
| `connection_speed`| string   | No       | Detected connection speed               |
| `timestamp`       | datetime | Yes      | Error timestamp (UTC)                   |

**Data Retention**: TTL index to auto-expire after configurable period (e.g., 30 days).

---

#### DM-008: `rate_limit_offenders`

**Description**: Tracks repeat rate limit offenders for progressive blocking.

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

---

### Data Relationships

```
frontpage (1 document)
    └── standalone

technical_articles (many documents)
    └── standalone, queried by slug/category/tags/date

blog_articles (many documents)
    └── standalone, identical structure to technical_articles

socials (few documents)
    └── standalone

others (many documents)
    └── standalone, links to external URLs

tracking (many documents, append-only)
    └── standalone, TTL auto-expiry

error_reports (many documents, append-only)
    └── standalone, TTL auto-expiry

rate_limit_offenders (many documents)
    └── standalone, referenced during rate limit checks
```
