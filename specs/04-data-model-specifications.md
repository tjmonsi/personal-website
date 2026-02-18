---
title: Data Model Specifications
version: 2.7
date_created: 2026-02-17
last_updated: 2026-02-18
owner: TJ Monserrat
tags: [data-model, database, firestore, mongodb, vector-search]
---

## Data Model Specifications

### Database

- **Primary Database**: Firestore Enterprise in MongoDB compatibility mode
- **Vector Database**: Firestore Native Mode (for semantic search vector storage)
- **Region**: `asia-southeast1`
- **Primary Driver**: MongoDB Go driver (standard MongoDB wire protocol)
- **Vector Driver**: Firestore Go SDK (`cloud.google.com/go/firestore`)
- **Reference (Enterprise)**: https://firebase.google.com/docs/firestore/enterprise/mongodb-compatibility-overview
- **Reference (Native Vector Search)**: https://firebase.google.com/docs/firestore/vector-search

Data models below use MongoDB-style field definitions for Firestore Enterprise collections. Firestore Native collections are documented separately (see DM-012).

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
| `tags`         | string[]   | Yes      | List of tags (all values MUST be lowercase)    |
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

> **Note**: Full-text search is handled by vector similarity search via Firestore Native (see DM-012), not by MongoDB text indexes.

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
| `url`          | string   | Yes      | Full URL to the profile or contact (e.g., `https://github.com/tjmonsi` or `mailto:tj@example.com`) |
| `display_name` | string   | Yes      | Display text shown to users (e.g., "@tjmonsi") |
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
| `tags`         | string[]   | Yes      | List of tags (all values MUST be lowercase)    |
| `external_url` | string     | Yes      | URL to the external content                    |
| `date_created` | datetime   | Yes      | Date of original content (UTC)                 |
| `date_updated` | datetime   | Yes      | Last metadata update (UTC)                     |

**Indexes**:

| Index Name              | Fields                          | Type     | Purpose                      |
| ----------------------- | ------------------------------- | -------- | ---------------------------- |
| `idx_slug`              | `slug`                          | Unique   | Slug lookups                 |
| `idx_category`          | `category`                      | Standard | Category filtering           |
| `idx_tags`              | `tags`                          | Multikey | Tag filtering                |
| `idx_date_created`      | `date_created`                  | Standard | Internal use / content management queries |
| `idx_date_updated`      | `date_updated`                  | Standard | Date range queries           |

> **Note**: Full-text search for `others` is handled by vector similarity search via Firestore Native (see DM-012), not by MongoDB text indexes.

**Constraints**:
- `slug` must be unique.
- `slug` must match pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}$`
  - Example: `external-article-2025-03-10-0800`

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

#### ~~DM-007~~ and ~~DM-008~~: Removed

**Note**: The `tracking` (DM-007) and `error_reports` (DM-008) Firestore collections have been removed. Tracking and error report data from `POST /t` is no longer stored in Firestore. Instead, the Go backend emits structured log entries to stdout, which Cloud Logging ingests and routes to BigQuery via dedicated log sinks (INFRA-010c for frontend errors, INFRA-010e for frontend tracking). BigQuery is the sole persistence layer for this data. See OBS-002 and OBS-003 in [07-observability-specifications.md](07-observability-specifications.md).

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
| `ban_history[].start` | datetime | Yes    | Ban start time                              |
| `ban_history[].end`   | datetime | No     | Ban end time (null = indefinite)            |
| `ban_history[].tier`  | string   | Yes    | Ban tier: `"30d"`, `"90d"`, `"indefinite"` |

**Indexes**:

| Index Name         | Fields        | Type   | Purpose                    |
| ------------------ | ------------- | ------ | -------------------------- |
| `idx_identifier`   | `identifier`  | Unique | Fast lookup for rate checks|

**Data Retention**:

- THE SYSTEM SHALL remove offense records where there is no active ban (`current_ban` is null or expired) and no offenses in the last 90 days.
- Offense records with indefinite bans SHALL be retained but subject to periodic manual review.
- A daily Cloud Scheduler job triggers the `cleanup-rate-limit-offenders` Cloud Function (INFRA-008d/008e) to enforce this retention policy.

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

#### DM-011: `embedding_cache` (Firestore Enterprise)

**Description**: Caches search query embedding vectors to avoid redundant calls to the Gemini embedding API. Document IDs are deterministic UUID v5 values derived from lowercased search text. **No search strings are stored** — only the UUID and the corresponding vector.

| Field          | Type       | Required | Description                                    |
| -------------- | ---------- | -------- | ---------------------------------------------- |
| `_id`          | string     | Yes      | UUID v5 of the lowercased search text (namespace: `6ba7b811-9dad-11d1-80b4-00c04fd430c8`, name: lowercased query string) |
| `vector`       | double[]   | Yes      | 2048-dimensional embedding vector from Gemini `gemini-embedding-001` (L2-normalized). The field is named `vector` (rather than `embedding`) to distinguish cached search-query vectors from document embeddings stored in DM-012. (CLR-146) |
| `model_version`| string     | Yes      | The embedding model identifier used to generate this vector (e.g., `gemini-embedding-001`). Used for automated cache invalidation on model upgrade (CLR-125). |
| `created_at`   | datetime   | Yes      | When the embedding was first cached (UTC)      |

**Indexes**:

| Index Name         | Fields   | Type   | Purpose                    |
| ------------------ | -------- | ------ | -------------------------- |
| (none)             | `_id`    | Primary | Cache lookup by UUID v5    |

**Constraints**:
- Document ID (`_id`) is the UUID v5 of the lowercased search string. The same query always produces the same UUID.
- No search strings or query text is stored anywhere in this collection.
- No TTL / no cache expiration. Cached embeddings persist indefinitely.
- Each cache document stores the `model_version` that was used to generate the embedding. The Go backend compares the `model_version` of each retrieved cache document against the currently configured model version. IF the versions do not match, the cached document is treated as a cache miss and the embedding is regenerated using the current model, then the document is overwritten with the new vector and updated `model_version` (CLR-125).
- The cache stores embeddings generated with `task_type=RETRIEVAL_QUERY`. If the task type is changed, the cache must be invalidated (cleared) to regenerate embeddings with the correct task type.

**Automated Cache Invalidation** (CLR-125):

On each cache lookup, the Go backend SHALL:
1. Retrieve the cache document by UUID v5.
2. Compare the document's `model_version` field against the backend's configured embedding model version (set via environment variable, e.g., `EMBEDDING_MODEL_VERSION=gemini-embedding-001`).
3. IF the versions match → cache hit (use the stored vector).
4. IF the versions do not match → cache miss (call Vertex AI to generate a new embedding, overwrite the cache document with the new vector, updated `model_version`, and refreshed `created_at`).

This ensures the cache self-heals incrementally when the model is upgraded, without requiring a bulk invalidation.

**Manual Cache Invalidation Procedure** (CLR-125):

For immediate full invalidation (e.g., during a model upgrade rollout), an operator can clear the entire cache:

```
// Via mongosh connected to Firestore Enterprise
use <database-name>
db.embedding_cache.deleteMany({})
```

Subsequent search queries will re-populate the cache with embeddings from the new model. This is a safe operation — the only impact is increased Vertex AI API calls until the cache is warm again.

**UUID v5 Generation**:
```
namespace = "6ba7b811-9dad-11d1-80b4-00c04fd430c8" (standard URL namespace)
name      = lowercase(search_query)
doc_id    = UUID_v5(namespace, name)
```

---

#### DM-012: Vector Collections (Firestore Native Mode)

**Description**: Three collections in Firestore Native Mode store article embedding vectors for semantic similarity search. Each collection mirrors a content collection in Firestore Enterprise.

> **Important**: These collections are in **Firestore Native Mode**, NOT Firestore Enterprise (MongoDB compat). They are accessed via the Firestore Go SDK, not the MongoDB driver.

##### DM-012a: `technical_article_vectors`

| Field                | Type       | Required | Description                                    |
| -------------------- | ---------- | -------- | ---------------------------------------------- |
| Document ID          | string     | Yes      | Same `_id` as the corresponding `technical_articles` document in Firestore Enterprise |
| `embedding`          | vector(2048)| Yes      | 2048-dimensional embedding vector (Gemini `gemini-embedding-001`, task type: `RETRIEVAL_DOCUMENT`, L2-normalized) |
| `embedding_text_hash`| string     | Yes      | SHA-256 hash of the source text used for embedding (for change detection during sync) |
| `updated_at`         | datetime   | Yes      | When the embedding was last generated (UTC)    |

**Embedding Source Text**: `title + "\n" + abstract + "\n" + category + "\n" + tags (comma-separated)`

**Vector Index**:

| Index Field  | Dimension | Distance Measure | Purpose                      |
| ------------ | --------- | ---------------- | ---------------------------- |
| `embedding`  | 2048      | COSINE           | Semantic similarity search   |

##### DM-012b: `blog_article_vectors`

Identical schema and vector index to DM-012a, but mirrors `blog_articles` in Firestore Enterprise.

##### DM-012c: `others_vectors`

Identical schema and vector index to DM-012a, but mirrors `others` in Firestore Enterprise.

**Embedding Source Text for `others`**: `title + "\n" + abstract + "\n" + category + "\n" + tags (comma-separated)`

**Constraints (all DM-012 collections)**:
- Document IDs MUST match the corresponding `_id` in Firestore Enterprise for 1:1 mapping.
- The `embedding_text_hash` field enables incremental sync — only re-embed articles whose source text has changed.
- Vector embeddings are generated by the `sync-article-embeddings` Cloud Function (INFRA-014) using Gemini `gemini-embedding-001` via Vertex AI with `task_type=RETRIEVAL_DOCUMENT` and `output_dimensionality=2048`. Vectors are L2-normalized before storage.
- When an article is deleted from Firestore Enterprise, the corresponding vector document SHOULD be deleted from Firestore Native during the next sync.

---

---

### Data Relationships

```
frontpage (1 document)
    └── standalone

technical_articles (many documents) [Firestore Enterprise]
    └── standalone, queried by slug/category/tags/date_updated
    └── category field → referenced in categories collection
    └── vector search via technical_article_vectors [Firestore Native]

blog_articles (many documents) [Firestore Enterprise]
    └── standalone, identical structure to technical_articles
    └── category field → referenced in categories collection
    └── vector search via blog_article_vectors [Firestore Native]

categories (many documents)
    └── derived from article categories across all article types
    └── synced by content management CI/CD pipeline

socials (few documents)
    └── standalone, ordered by sort_order, filtered by is_active

others (many documents) [Firestore Enterprise]
    └── standalone, links to external URLs
    └── category field → referenced in categories collection
    └── vector search via others_vectors [Firestore Native]

~~tracking~~ (REMOVED — data flows to BigQuery via structured logs)
~~error_reports~~ (REMOVED — data flows to BigQuery via structured logs)

rate_limit_offenders (many documents)
    └── offense records written by process-rate-limit-logs Cloud Function (INFRA-008c) via Cloud Armor log sink
    └── read by Cloud Run Go API on each request for ban status checks
    └── real-time rate counting handled by Cloud Armor (see INFRA-005)

sitemap (single document)
    └── standalone, generated by Cloud Scheduler job
    └── served via GET /sitemap.xml endpoint

embedding_cache (many documents) [Firestore Enterprise]
    └── keyed by UUID v5 of lowercased search text
    └── caches Gemini embedding vectors for search queries
    └── no expiration

technical_article_vectors (many documents) [Firestore Native]
    └── 1:1 mapping to technical_articles by document ID
    └── vector indexed for cosine similarity search
    └── synced by sync-article-embeddings Cloud Function (INFRA-014)

blog_article_vectors (many documents) [Firestore Native]
    └── 1:1 mapping to blog_articles by document ID
    └── vector indexed for cosine similarity search
    └── synced by sync-article-embeddings Cloud Function (INFRA-014)

others_vectors (many documents) [Firestore Native]
    └── 1:1 mapping to others by document ID
    └── vector indexed for cosine similarity search
    └── synced by sync-article-embeddings Cloud Function (INFRA-014)
```

---

### Acceptance Criteria

- **AC-DM-001**: Given the Firestore Enterprise database, when the schema is initialized, then all collections (`technical_articles`, `blog_articles`, `others`, `categories`, `sitemap`, `rate_limit_offenders`) exist and accept documents matching their defined schemas.
- **AC-DM-002**: Given a `technical_articles` document, when it is created, then all required fields (`slug`, `category`, `tags`, `title`, `abstract`, `content`, `changelog`, `date_created`, `date_updated`) are present and correctly typed. (CLR-105)
- **AC-DM-003**: Given the Firestore Native database, when vector collections are synced, then each article has a corresponding vector document with an `embedding` field containing a 2048-dimension float array. (CLR-105, CLR-134)
- **AC-DM-004**: Given the `rate_limit_offenders` collection, when an offender document's `current_ban` is null or `current_ban.end` has passed and no offenses in the last 90 days, then the cleanup function deletes the document. (CLR-105)
- **AC-DM-005**: Given a query on `technical_articles` with filters on `category`, `tags`, and `date_updated`, when the query executes, then the defined compound indexes support the query without a full collection scan.
- **AC-DM-006**: Given the `slug` field on any article collection, when a duplicate slug is inserted, then the unique index prevents the insertion.
- **AC-DM-011-A**: Given the embedding cache, when a cache hit occurs with a mismatched `model_version`, then the system SHALL treat it as a cache miss and re-generate the embedding. (CLR-136)
- **AC-DM-011-B**: Given a model version change, when `sync-article-embeddings` runs, then it SHALL regenerate all embeddings and update `model_version` in all cache entries. (CLR-136)
