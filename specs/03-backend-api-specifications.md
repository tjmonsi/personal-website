---
title: Backend API Specifications
version: 1.2
date_created: 2026-02-17
last_updated: 2026-02-17
owner: TJ Monserrat
tags: [backend, api, go, cloud-run]
---

## Backend API Specifications

### Technology

- **Language**: Go
- **Runtime**: Google Cloud Run (asia-southeast1)
- **Network**: Behind Google Cloud Load Balancer + Cloud Armor
- **Database**: Firestore Enterprise (MongoDB compatibility mode) via MongoDB Go driver
- **Domain**: `api.tjmonserrat.com`

### Global API Rules

#### Allowed HTTP Methods

- THE SYSTEM SHALL accept `GET` requests on all endpoints except `POST /t`.
- THE SYSTEM SHALL accept `POST` requests only on the `/t` endpoint (tracking and error reporting).
- WHEN a request uses any method other than the allowed method for an endpoint, THE SYSTEM SHALL return HTTP `405 Method Not Allowed` with an appropriate `Allow` header.

#### Error Response Strategy

- THE SYSTEM SHALL only return the following HTTP status codes to clients:
  - `200` — Success
  - `400` — Bad Request (invalid input)
  - `403` — Forbidden (blocked client, 30-day ban tier)
  - `404` — Not Found (resource does not exist, catch-all for masked errors, or indefinitely blocked client)
  - `405` — Method Not Allowed
  - `429` — Too Many Requests (rate limit exceeded or initial offense ban tier)
  - `503` — Service Unavailable (only for legitimate network/infrastructure failures)
- **THE SYSTEM SHALL NEVER return HTTP `500` to clients.**
  - IF an internal server error occurs, THE SYSTEM SHALL return `404` to the client.
  - THE SYSTEM SHALL log the actual error with full context to Google Cloud Logging.

#### Error Logging (Internal)

- WHEN an internal error occurs, THE SYSTEM SHALL log:
  - Timestamp (UTC)
  - Request method and path
  - Query parameters (sanitized)
  - Client IP address
  - Error message and stack trace
  - Request ID / correlation ID

#### Response Format

- All API responses SHALL use `Content-Type: application/json`.
- Article content responses SHALL include rendered content within the JSON payload.

---

### Endpoints

#### BE-API-001: `GET /`

**Description**: Returns the front page content.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "content": "<markdown string of front page>"
}
```

**Behavior**:

- THE SYSTEM SHALL fetch the front page markdown content from the database.
- IF the content is not found, THE SYSTEM SHALL return `404`.

---

#### BE-API-002: `GET /technical`

**Description**: Returns a paginated list of technical blog articles.

**Query Parameters**:

| Parameter    | Type    | Required | Description                                    |
| ------------ | ------- | -------- | ---------------------------------------------- |
| `page`       | integer | No       | Page number (default: 1)                       |
| `q`          | string  | No       | Search query (max 300 characters)              |
| `category`   | string  | No       | Filter by category                             |
| `tags`       | string  | No       | Comma-separated list of tags to filter by      |
| `tag_match`  | string  | No       | Tag matching logic when multiple tags are specified: `all` (AND — article must have ALL specified tags) or `any` (OR — article must have at least ONE specified tag). Default: `any`. |
| `date_from`  | string  | No       | Start of date range for `date_updated` (ISO 8601 date, e.g. `2025-01-01`) |
| `date_to`    | string  | No       | End of date range for `date_updated` (ISO 8601 date)              |

**Validation**:

- IF `q` exceeds 300 characters, THE SYSTEM SHALL return `400`.
- IF `page` is not a positive integer, THE SYSTEM SHALL return `400`.
- IF `date_from` or `date_to` is not valid ISO 8601, THE SYSTEM SHALL return `400`.
- IF `tag_match` is provided and is not `all` or `any`, THE SYSTEM SHALL return `400`.

**Response** (`200`):

```json
{
  "items": [
    {
      "title": "Article Title",
      "slug": "article-title-2025-01-15-1030",
      "category": "DevOps",
      "abstract": "Brief summary of the article...",
      "tags": ["go", "cloud-run", "docker"],
      "date_created": "2025-01-15T10:30:00Z",
      "date_updated": "2025-06-20T14:00:00Z"
    }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 5,
    "total_items": 42,
    "page_size": 10
  }
}
```

**Behavior**:

- THE SYSTEM SHALL return exactly 10 items per page (fixed, not configurable by the client).
- THE SYSTEM SHALL sort results by `date_updated` in descending order (most recently updated first). This is the only sort order; no `sort_by` or `sort_order` parameters are accepted.
- IF the search/filter yields no results, THE SYSTEM SHALL return `200` with an empty `items` array and `pagination.total_items` of `0`. Example response:

```json
{
  "items": [],
  "pagination": {
    "current_page": 1,
    "total_pages": 0,
    "total_items": 0,
    "page_size": 10
  }
}
```

- Reserve `404` for truly missing individual resources (e.g., invalid slug in detail endpoints like `GET /technical/{slug}.md`).

---

#### BE-API-003: `GET /technical/{slug}.md`

**Description**: Returns a single technical blog article.

**Path Parameters**:

| Parameter | Type   | Description                                    |
| --------- | ------ | ---------------------------------------------- |
| `slug`    | string | Article slug in format: `{title-slug}-{ISO8601-datetime}` |

**Validation**:

- IF the URL does not end with `.md`, THE SYSTEM SHALL return `404`.
- IF the slug does not match the prescribed pattern, THE SYSTEM SHALL return `404`.
  - Slug pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}-\d{4}\.md$`
  - Example: `my-first-article-2025-01-15-1030.md`
- IF the article does not exist in the database, THE SYSTEM SHALL return `404`.

**Response** (`200`):

```json
{
  "title": "Article Title",
  "author": "TJ Monserrat",
  "date_created": "2025-01-15T10:30:00Z",
  "date_updated": "2025-06-20T14:00:00Z",
  "category": "DevOps",
  "tags": ["go", "cloud-run", "docker"],
  "changelog": [
    {
      "date": "2025-06-20T14:00:00Z",
      "description": "Updated code examples for Go 1.23"
    },
    {
      "date": "2025-01-15T10:30:00Z",
      "description": "Initial publication"
    }
  ],
  "content": "<markdown string of article body>",
  "citations": {
    "apa": "Monserrat, T.J. (2025). Article Title. tjmonserrat.com. https://tjmonserrat.com/technical/article-title-2025-01-15-1030",
    "mla": "Monserrat, TJ. \"Article Title.\" tjmonserrat.com, 15 Jan. 2025, https://tjmonserrat.com/technical/article-title-2025-01-15-1030.",
    "chicago": "Monserrat, TJ. \"Article Title.\" tjmonserrat.com. January 15, 2025. https://tjmonserrat.com/technical/article-title-2025-01-15-1030.",
    "bibtex": "@online{monserrat2025articletitle, author={Monserrat, TJ}, title={Article Title}, year={2025}, url={https://tjmonserrat.com/technical/article-title-2025-01-15-1030}}",
    "ieee": "T.J. Monserrat, \"Article Title,\" tjmonserrat.com, Jan. 15, 2025. [Online]. Available: https://tjmonserrat.com/technical/article-title-2025-01-15-1030"
  }
}
```

**Behavior**:

- THE SYSTEM SHALL generate citation strings dynamically from article metadata (title, author, date, slug).
- Citations SHALL be generated in all supported formats: APA, MLA, Chicago, BibTeX, and IEEE.
- Citation URLs SHALL use the frontend route format (`https://tjmonserrat.com/technical/{slug}`) without the `.md` extension. The `.md` extension is only used in API paths (`GET /technical/{slug}.md`), not in reader-facing URLs.
- The frontend route `/technical/:slug` receives the slug without `.md`. When calling the API, the frontend appends `.md` to form the API path (`GET /technical/{slug}.md`).

---

#### BE-API-004: `GET /blog`

**Description**: Returns a paginated list of opinion/blog articles.

**Behavior**: Identical to BE-API-002 but queries the blog/opinions table instead of the technical table.

All query parameters, validation rules, response format, and error handling are the same as BE-API-002.

---

#### BE-API-005: `GET /blog/{slug}.md`

**Description**: Returns a single opinion/blog article.

**Behavior**: Identical to BE-API-003 but queries the blog/opinions table instead of the technical table.

All path parameters, validation rules, response format, and error handling are the same as BE-API-003.

---

#### BE-API-006: `GET /socials`

**Description**: Returns a list of social media links.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "items": [
    {
      "platform": "GitHub",
      "url": "https://github.com/tjmonserrat",
      "display_name": "@tjmonserrat",
      "icon": "github",
      "sort_order": 1
    },
    {
      "platform": "LinkedIn",
      "url": "https://linkedin.com/in/tjmonserrat",
      "display_name": "TJ Monserrat",
      "icon": "linkedin",
      "sort_order": 2
    }
  ]
}
```

**Behavior**:

- THE SYSTEM SHALL return social links filtered to only include entries where `is_active` is `true` in the database.
- THE SYSTEM SHALL return items sorted by `sort_order` in ascending order.
- IF no active social links exist, THE SYSTEM SHALL return an empty `items` array (HTTP 200).

---

#### BE-API-007: `GET /others`

**Description**: Returns a paginated list of external/notable content.

**Behavior**: Identical to BE-API-002 in terms of query parameters, pagination, validation, and error handling, but queries the "others" collection. Date range filter applies to `date_updated`.

**Response item differences**:

```json
{
  "items": [
    {
      "title": "External Article Title",
      "slug": "external-article-2025-03-10-0800",
      "category": "Open Source",
      "abstract": "Description of the external content...",
      "tags": ["python", "colab"],
      "date_created": "2025-03-10T08:00:00Z",
      "date_updated": "2025-03-10T08:00:00Z",
      "external_url": "https://github.com/tjmonserrat/project"
    }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 2,
    "total_items": 15,
    "page_size": 10
  }
}
```

> **Note**: Unlike technical/blog articles, items in `/others` link to external URLs rather than internal article pages.

---

#### BE-API-008: `GET /categories`

**Description**: Returns the list of all categories across all article types.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "categories": [
    {
      "name": "DevOps",
      "date_created": "2025-01-10T08:00:00Z"
    },
    {
      "name": "Cloud",
      "date_created": "2025-02-15T12:00:00Z"
    }
  ]
}
```

**Behavior**:

- THE SYSTEM SHALL return all categories from the `categories` collection, sorted alphabetically by name.
- Categories are free-form and derived from the categories assigned to articles across all article types.
- IF no categories exist, THE SYSTEM SHALL return an empty array.
- The frontend SHALL cache the response in sessionStorage for 24 hours.

---

#### BE-API-009: `POST /t`

**Description**: Receives anonymous visitor tracking data and frontend error reports. This is the **only POST endpoint** in the entire API.

**Authentication**:

- THE SYSTEM SHALL require a valid `Authorization: Bearer <JWT>` header.
- The JWT SHALL be generated by the frontend using a static client ID and static secret embedded (obfuscated) in the frontend code.
- THE SYSTEM SHALL validate the JWT signature, expiration, and issuer claims.
- THE SYSTEM SHALL allow a clock skew tolerance of **60 seconds** when validating `exp` and `iat` claims, to accommodate minor clock drift between client and server.
- IF the JWT is missing, invalid, or expired, THE SYSTEM SHALL return HTTP `403 Forbidden`.
- The JWT SHALL include the following claims:
  - `iss`: Static client identifier (e.g., `"tjmonserrat-web"`)
  - `iat`: Issued-at timestamp
  - `exp`: Expiration timestamp (short-lived, e.g., 5 minutes)
- The JWT secret SHALL be stored as an environment variable on the backend. The same secret is embedded (obfuscated) in the frontend bundle.

**Request Body** (JSON, `Content-Type: application/json`):

For page tracking:
```json
{
  "action": "page_view",
  "page": "/technical",
  "referrer": "https://google.com",
  "browser": "Chrome 120",
  "connection_speed": {
    "effective_type": "4g",
    "downlink": 10.0,
    "rtt": 50
  }
}
```

For error reporting:
```json
{
  "action": "error_report",
  "page": "/technical/my-article-2025-01-15-1030.md",
  "browser": "Chrome 120",
  "error_type": "network",
  "error_message": "Failed to fetch article content",
  "connection_speed": {
    "effective_type": "3g",
    "downlink": 1.5,
    "rtt": 300
  }
}
```

**Validation**:

- `action` is required and must be one of: `page_view`, `error_report`.
- `page` is required (string, max 500 characters).
- `browser` is required (string, max 200 characters).
- All other fields are optional.
- IF validation fails, THE SYSTEM SHALL return HTTP `400`.

**Response** (`200`):

```json
{
  "status": "ok"
}
```

**Behavior**:

- THE SYSTEM SHALL extract the client IP from `X-Forwarded-For` or the connection source.
- THE SYSTEM SHALL add a server-side UTC timestamp.
- IF `action` is `"page_view"`, THE SYSTEM SHALL write to the `tracking` collection (DM-007).
- IF `action` is `"error_report"`, THE SYSTEM SHALL write to the `error_reports` collection (DM-008).
- THE SYSTEM SHALL apply separate rate limits to this endpoint (30 req/min per IP).

---

#### BE-API-010: `GET /health`

**Description**: Health check endpoint used by Cloud Run for liveness/readiness probes.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "status": "ok"
}
```

**Behavior**:

- THE SYSTEM SHALL check database connectivity and return `200` with `{"status": "ok"}` if healthy.
- IF the database is unreachable, THE SYSTEM SHALL return `503 Service Unavailable`.
- This endpoint SHALL NOT be subject to application-level rate limiting.
- This endpoint SHALL be accessible only through Cloud Run's internal health check mechanism (not exposed to public traffic via the Load Balancer).

---

#### BE-API-011: `GET /sitemap.xml`

**Description**: Returns the pre-generated sitemap XML. The sitemap is generated periodically by a Cloud Scheduler job and stored in Firestore.

**Request**: No parameters.

**Response** (`200`):

- Content-Type: `application/xml`
- Body: Standard XML sitemap per the [Sitemaps protocol](https://www.sitemaps.org/protocol.html)

**Behavior**:

- THE SYSTEM SHALL fetch the latest sitemap document from the `sitemap` Firestore collection.
- THE SYSTEM SHALL return the sitemap XML content with `Content-Type: application/xml`.
- THE SYSTEM SHALL set `Cache-Control: public, max-age=3600` (1 hour) to reduce database reads for repeated requests.
- IF the sitemap document does not exist in the database, THE SYSTEM SHALL return `404`.
- THE SYSTEM SHALL NOT regenerate the sitemap on each request. Regeneration is handled by a Cloud Scheduler job (see INFRA-008 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)).
- This endpoint SHALL be subject to the standard rate limiting rules.

---

### General API Notes

- **Author field**: The `author` field in article responses (BE-API-003, BE-API-005) is always hardcoded to `"TJ Monserrat"`. This is a single-author website; the author value is not stored in the database.
- **Offline availability**: The backend does not track which articles are available offline. Offline availability is a client-side concern managed by the frontend's service worker and local storage.
- **Internal endpoints**: The backend includes an internal endpoint `POST /internal/generate-sitemap` used by Cloud Scheduler to trigger sitemap regeneration (see INFRA-008 in [05-infrastructure-specifications.md](05-infrastructure-specifications.md)). This endpoint is NOT exposed to public traffic via the Load Balancer and is authenticated via OIDC token from the Cloud Scheduler service account.

---

### Rate Limiting

See [06-security-specifications.md](06-security-specifications.md) for full rate limiting specifications.
